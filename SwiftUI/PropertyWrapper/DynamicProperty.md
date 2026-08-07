# DynamicProperty
SwiftUI가 설치해 주는 타입(View·App·Scene·ViewModifier)의 저장 프로퍼티를
SwiftUI의 저장소·업데이트 사이클에 참여시키는 프로토콜.
모든 상태 래퍼([[@State]], [[@Binding]], [[@ObservedObject]], @FetchRequest…)가 채택한다.
역할은 **"표식 하나, 훅 하나"**.

### 왜 필요한가
- 뷰는 struct라 프로퍼티가 뷰와 함께 버려진다([[값으로서의 View (View as Value)]]). 값은 **SwiftUI 소유 저장소**에 산다 — 저장의 주인은 개발자가 아니라 SwiftUI
- 이 프로토콜이 없으면 SwiftUI는 **"이 프로퍼티가 상태 값인지, 순전히 그냥 값인지"** 구별할 수 없다 — 무엇을 저장소에 연결(설치)할지 알 길이 없다
- 채택이 주는 것은 둘뿐: ① 설치 심사 때 struct 안을 들여다봐 주는 표식 ② body 직전 update() 훅. **저장소 개설은 목록에 없다** — 그 능력은 [[@State]] 같은 SwiftUI 제공 래퍼에게만 있다(개설 API 비공개)

### 내부 동작 — 래퍼의 일생
```
t1 뷰 struct 생성 — _count는 초깃값 든 상자일 뿐, 아무 데도 연결 안 됨
t2 설치 — identity 자리에 앉히며 저장소와 연결
   유지: 기존 저장소에 재연결 (값 유지, 초깃값 무시)
   신규: 저장소 개설 (초깃값은 이때만 쓰인다)
   소멸: 저장소 파괴 (상태 리셋)
t3 update() — SwiftUI가 body 직전에 호출
t4 body — count 읽기 = wrappedValue getter = 연결을 타고 저장소 읽기
```
- **연결(t2)이 update(t3)보다 앞인 이유**: 연결이 없으면 update()가 읽어올 원본이 없다 — 초깃값밖에 못 읽는다
- **update()는 방아쇠가 아니다.** `count += 1`은 setter가 저장소에 쓰고 **dirty 표시만** 남긴다([[뷰 업데이트 사이클 (View Update Cycle)]] ①~③). update()는 다음 프레임 틱, body 직전에 SwiftUI가 부르는 **읽기 준비 훅**. @State는 여기서 할 일이 거의 없고(기본 구현), **외부 시스템과 동기화하는 래퍼**(@FetchRequest의 Core Data 쿼리 실행)가 실질적으로 쓴다
- **이층 구조**: 언어(@propertyWrapper, SE-0258)는 컴파일 타임에 `count` → `_count.wrappedValue` **바꿔치기만** 한다. 설치·연결·update 호출·무효화는 전부 SwiftUI가 런타임에. `nonmutating set`이 가능한 이유 = 쓰기가 struct가 아니라 **힙의 SwiftUI 저장소**로 가기 때문
- **기반 네 다리**: 뷰가 struct라서 필요했고(문제) → 값을 SwiftUI 저장소에 두고(어디) → 프로퍼티 래퍼 문법으로 쓰고(어떻게) → 리플렉션으로 찾는다(누가)

> 설치 대상을 찾는 수단(리플렉션)과 저장소의 실체(AttributeGraph + [[Identity]])는 비공개 구현 추정.
> 공개 계약은 "채택하면 내부 dynamic property들을 SwiftUI가 관리 + body 전 update() 호출"까지 — Apple 문서

### 대안과 트레이드오프
| 대안 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| plain `let` + init 주입 (부모가 상태 소유) | 단순·명시적 | 로컬 상태 불가, init 시점과 렌더 시점의 어긋남 |
| React hooks — 호출 **순서**로 슬롯 매칭 | 언어 확장 불필요 | "조건문 안 훅 금지"를 사람이 지켜야 |
| UIKit — 뷰가 참조 타입이라 상태 직접 소유 | 저장 문제 자체가 없음 | 값↔화면 동기화 전부 수동 |

### 언제 쓰면 안 되는가 — 설치 기준 "아직 / 영영 / 이미"
| | 상황 | 왜 죽나 |
| --- | --- | --- |
| 아직 | 뷰 `init`에서 값 읽기/쓰기 | 설치 전이라 연결 없음 — 런타임 경고(아래) |
| 영영 | plain class·전역에 선언 | SwiftUI가 설치하는 그래프 밖이라 심사가 닿지 않는다 |
| 이미 | 뷰 소멸 후 escaping closure에서 접근 | 저장소가 이미 파괴됨 |

**"영영"의 기준은 "뷰인가"가 아니다.** View 말고 App·Scene·ViewModifier도 SwiftUI가 설치해 주고, 설치된 것이 품은 DynamicProperty도 재귀적으로 따라 들어간다(아래 Capped가 그 경우다). 죽는 건 **SwiftUI가 설치하는 그래프에 아예 매달리지 않은** 곳 — plain class, 전역 변수다

> 런타임 경고 원문 (검색용):
> `Accessing State's value outside of being installed on a View. This will result in a constant Binding of the initial value and will not update.`

단 `_count = State(initialValue: 10)`은 init에서도 된다 — **상자 만들기는 연결을 안 거친다.** 안 되는 건 wrappedValue 접근(연결 필수)

### 실전 적용 — 커스텀 상태 래퍼
**"저장 능력은 State에만 있다. 그래서 State를 품는다. 설치는 재귀다."**
```swift
@propertyWrapper
struct Capped: DynamicProperty {     // 채택 → 설치 심사 대상
    @State private var value: Int    // 품는 것은 이것 — 저장·갱신을 통째로 위임

    init(wrappedValue: Int) {        // @Capped var count = 5 의 `= 5`가 여기로 온다
        _value = State(initialValue: min(wrappedValue, 100))  // 상자 만들기 — 연결 전이라 가능
    }

    var wrappedValue: Int {
        get { value }
        nonmutating set { value = min(newValue, 100) }  // 내 로직은 창구에만
    }
}
```
@State 없이 만들면: 뷰 재생성마다 값 리셋 + 쓰기가 dirty를 못 일으켜 화면이 안 바뀐다 — 채택했어도 안에 설치할 물건이 없으면 그냥 지나간다

`init(wrappedValue:)`가 **없으면 컴파일되지 않는다.** 초깃값을 줄 통로가 그것뿐이고, `@State private var value`가 합성해 주는 memberwise init은 `private`이라 같은 파일 밖에서는 쓸 수도 없다. 그리고 이 init이 `_value`에 직접 대입할 수 있는 근거가 위의 **"상자 만들기는 연결을 안 거친다"**

### 더 파볼 질문
- 커스텀 래퍼에서 State의 나머지 반쪽 — "쓰기 → dirty → 화면 갱신"까지 State가 대신 해준다는 것 (재인출 약점)
- [[@propertyWrapper]] (SE-0258) — 언어 층의 합성 규칙 (백로그)
- [[Mirror]] / 리플렉션 — "심사" 다리의 실체 (백로그)
- projectedValue와 `$` (백로그 기존 항목)

### 관련
[[PropertyWrapper]] · [[@State]] · [[값으로서의 View (View as Value)]] · [[뷰 업데이트 사이클 (View Update Cycle)]]

### 출처
- Apple 문서 [DynamicProperty](https://developer.apple.com/documentation/swiftui/dynamicproperty) · [update()](https://developer.apple.com/documentation/swiftui/dynamicproperty/update())
- Swift Evolution [SE-0258 Property Wrappers](https://github.com/apple/swift-evolution/blob/main/proposals/0258-property-wrappers.md)
- WWDC21 "Demystify SwiftUI" · WWDC20 "Data Essentials in SwiftUI"
