# @propertyWrapper
프로퍼티 하나를 **숨은 저장소 + 접근자 쌍**으로 바꿔치기하는 컴파일 타임 코드 합성 기능 (SE-0258, Swift 5.1).
SwiftUI 전용이 아니라 **순수 언어 기능**이다 — [[DynamicProperty]]·[[@State]]는 이 위에 얹힌 소비자일 뿐이다.

### 왜 필요한가
- 없으면 **저장 프로퍼티 + 계산 프로퍼티 쌍을 프로퍼티 수만큼 손으로 복제**해야 한다. 클램프 로직이 20개 프로퍼티에 20번 복사되고, 상한을 바꾸려면 20군데를 고치며 한 군데를 빠뜨리면 조용히 틀린다
- **함수로 안 되는 이유**: 함수는 로직을 모을 뿐 *저장 프로퍼티와 접근자의 짝*을 만들어 주지 못한다. 선언 두 줄은 여전히 손으로 쓴다
- **프로토콜로 안 되는 이유**: 프로토콜은 타입에 붙지 프로퍼티에 붙지 않고, 무엇보다 **프로토콜(확장)은 저장 공간을 추가할 수 없다**
- 동기는 특권의 개방이다 — `lazy`와 `@NSCopying`은 **컴파일러에 하드코딩된 기능**이었다. "같은 종류를 사용자도 라이브러리로 정의하게 하자"가 SE-0258

### 내부 동작 — 저장소(창고)와 wrappedValue(창구)는 다른 물건
```
@Clamped var volume = 150
        ↓ 컴파일러가 만드는 것 (치환 규칙은 셋뿐)
private var _volume = Clamped(wrappedValue: 150)   ① 선언 자체
volume   → _volume.wrappedValue                     ② 값 접근
$volume  → _volume.projectedValue                   ③ 투영
```
- **`wrappedValue`는 값을 담는 곳이 아니라 값이 드나드는 창구다.** 창고(백킹 스토리지)를 따로 두지 않으면 getter가 돌려줄 것이 없다 — `get { self }` 류의 실패가 여기서 나온다
- **`_volume`은 래퍼 인스턴스 자체(`Clamped`), `volume`은 그 안의 값(`Int`).** 이름이 비슷할 뿐 타입이 다르다
- **`$`는 Binding이 아니라 `projectedValue`다.** [[@State]]가 `Binding`을 주는 것은 그 래퍼가 그렇게 만들어서일 뿐, 언어 규칙은 "무엇이든 래퍼 작성자가 정한 타입"이다 (`Bool`도 된다)
- **`_volume`은 자동으로 `private`이다.** 원본 프로퍼티보다 접근 수준이 낮아져 타입 밖에서는 보이지 않는다
- `= 150`은 **오직 `init(wrappedValue:)`로만** 간다. 다른 통로가 없다

### 언어 층과 프레임워크 층의 경계
> **컴파일은 언어 층, 실행은 프레임워크 층.**

```
@State private var count = 0

 swiftc의 일 ─────────────────────────────┐
   _count = State<Int>(wrappedValue: 0)   │ 상자를 만들어 struct 안에 놓는다
   count  → _count.wrappedValue           │ ← 컴파일 타임에 전부 끝. 여기까지가 국경
   $count → _count.projectedValue         │
 ─────────────────────────────────────────┘
 SwiftUI의 일 ────────────────────────────┐
   _count를 찾아내 저장소에 설치·연결       │ ← 런타임. DynamicProperty 도장이 있어야 시작
   body 직전 update() 호출                 │ update()는 국경이 아니라 한참 안쪽
   wrappedValue 쓰기 → dirty → 무효화      │
 ─────────────────────────────────────────┘
```

**`@propertyWrapper`는 상자를 만들고, [[DynamicProperty]]는 그 상자를 SwiftUI에 신고한다.**
앞이 없으면 상자가 안 생기고, 뒤가 없으면 상자가 아무 데도 연결되지 않는다.

`@State`의 실제 선언에 도장이 둘 찍혀 있는 것이 이 구조의 실물이다:
```swift
@frozen @propertyWrapper public struct State<Value>: DynamicProperty { ... }
```

### 대안과 트레이드오프
| 대안 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| 손으로 쓴 저장 + 계산 프로퍼티 | 프로퍼티마다 다른 로직, 마법 없음 | 프로퍼티 수만큼 복제, 수정 시 누락 |
| **`@propertyWrapper`** | 로직이 한 군데 — N개 프로퍼티가 공유, `$` 투영 | 변환 모양이 고정, 타입이 래퍼에 묶임, `_x` 간접층 |
| 매크로 (Swift 5.9+) | 임의 선언 생성 — 프로토콜 준수·형제 프로퍼티·타입 전체 | 구현·디버깅 복잡도, 별도 모듈, 컴파일 시간 |

**매크로는 상위 호환이 아니라 다른 층이다.** property wrapper의 변환은 **"프로퍼티 한 칸 → 저장소 + 접근자"** 한 가지 모양으로 고정돼 있다. `@Observable`이 매크로인 이유가 이것 — 클래스에 프로토콜 준수를 추가하고 레지스트라를 심고 모든 프로퍼티의 접근자를 고쳐야 하는데, **프로퍼티 한 칸 안에서 끝나는 일이 아니다.** 프로퍼티 한 칸을 고치는 게 wrapper, 타입 전체를 고치는 게 매크로.

### 언제 쓰면 안 되는가
`swiftc`로 전수 확인한 결과다.

| 위치 | 결과 | 컴파일러 메시지 |
| --- | --- | --- |
| 저장 프로퍼티 / 지역 변수 / 함수 파라미터 | **OK** | — |
| `let` 상수 | X | `can only be applied to a 'var'` |
| 계산 프로퍼티 | X | `cannot be applied to a computed property` |
| `lazy`와 병용 | X | `with a wrapper cannot also be lazy` |
| `weak`와 병용 | X | `with a wrapper cannot also be weak` |
| 프로토콜 요구사항 | X | `declared inside a protocol cannot have a wrapper` |
| 전역 변수 | X | `not yet supported in top-level code` |
| extension의 인스턴스 프로퍼티 | X | `declared inside an extension cannot have a wrapper` |

**공통 논리 하나: 저장 공간이 없거나, 이미 다른 방식으로 쓰고 있는 자리에는 붙지 않는다.**
계산 프로퍼티는 저장이 없고, `lazy`·`weak`는 자기만의 저장 전략이 있고, 프로토콜·extension은 저장을 추가할 수 없고, `let`은 `set`을 못 쓴다.

- 래퍼 타입에 **`wrappedValue`가 없으면** 선언 자체가 컴파일 에러다
- **가장 위험한 실패는 컴파일이 통과하는 경우다** — DynamicProperty 래퍼를 plain class에 선언하면 `swiftc`는 조용히 통과시킨다(언어는 DynamicProperty에 관심이 없다). 죽는 것은 실행 시점이고, [[@State]]는 설치되지 않아 초깃값에 고정된다:
  `Accessing State's value outside of being installed on a View. This will result in a constant Binding of the initial value and will not update.`

### 실전 적용 — SwiftUI 없이 성립하는 래퍼
```swift
@propertyWrapper
struct Clamped {
    private var storage: Int = 0          // ① 창고 — 진짜 값이 사는 곳
    var wrappedValue: Int {               // ② 창구 — 드나들 때 로직이 걸린다
        get { storage }
        set { storage = min(max(newValue, 0), 100) }
    }
    var projectedValue: Bool { storage == 100 }        // $volume — Binding이 아니어도 된다
    init(wrappedValue: Int) { self.wrappedValue = wrappedValue }   // 초기값도 창구를 통과
}

struct Config { @Clamped var volume: Int = 150 }
// volume == 100, $volume == true   ← import SwiftUI 없이 동작한다
```

`init(wrappedValue:)`를 빼면 **초깃값을 주는 형태(`= 150`)가 깨지고, 타입만 지정하는 형태(`var volume: Int`)는 여전히 된다.** [[DynamicProperty]] 노트의 `Capped`가 걸렸던 지점이 정확히 이것이다 — 게다가 저장 프로퍼티가 `private`이면 합성 memberwise init도 `private`이 되어 타입 밖에서는 보이지도 않는다.

반대로 `Capped` 안의 `@State`가 View가 아닌 곳에서 사는 이유는 **설치가 재귀이기 때문이다** — SwiftUI가 View에서 시작해 매달린 DynamicProperty를 계속 타고 들어가 끝까지 설치한다. View라서가 아니라 **View에 매달려서** 산다.

### 더 파볼 질문
- 래퍼 합성 `@A @B var x` — A의 `wrappedValue` 타입이 B여야 한다는 타입 체인 규칙
- `init(projectedValue:)` — `$`로 초기화하는 경로는 언제 쓰는가
- 함수 파라미터에 붙는 property wrapper(SE-0293)는 무엇을 합성하는가
- 커스텀 래퍼를 코드로 직접 작성해 보기 (이번 세션 미실습)

### 관련
[[DynamicProperty]] · [[@State]] · [[@Binding]] · [[PropertyWrapper]] · [[@dynamicMemberLookup]]

### 출처
- Swift Evolution [SE-0258 Property Wrappers](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0258-property-wrappers.md)
- [The Swift Programming Language — Property Wrappers](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/properties/#Property-Wrappers)
- WWDC19 "Modern Swift API Design" (Session 415)
- 위 제약 표·치환 동작은 Swift 6.2.3 / iOS 17 SDK `swiftc`로 직접 검증 (2026-08-07)
