# 값으로서의 View (View as Value)
SwiftUI의 View는 화면에 사는 객체가 아니라 "화면이 이래야 한다"는 기술(description)을
담은 일시적 값이다. 만들어지고, [[Diffing]]으로 비교되고, 버려지는 것이 정상 수명.
[[값 의미론 (Value Semantics)]]을 UI 프레임워크가 소비하는 접점.

### 왜 필요한가 — class 뷰의 두 가지 죄 + 보너스

1. **과거 보존 (diff의 전제)** — class는 값이 변경되면 복사 없이 **원본에 곧바로
   변경해버려서** 이전 값이 소멸한다 → 비교 대상이 없어 diff 불가.
   struct는 이전 뷰 트리가 독립된 값으로 공짜로 살아남는다
2. **상태 추방 (Source of Truth 강제)** — class 뷰는 오래 살아 상태를 담기 쉽고,
   진실이 뷰들에 사본으로 흩어져 **Source of Truth를 위반하기 쉽다** → 수동 동기화,
   놓치면 화면 불일치. struct 뷰는 상태가 살아남을 수 없는 그릇이라 가변 상태가
   [[@State]]라는 명시적 통로로 강제 격리된다 (body는 nonmutating — plain 프로퍼티
   변경은 컴파일조차 안 된다)
3. **공짜 재생성 (보너스)** — 참조 카운팅 없이 인라인으로 다뤄져 초당 수십 번
   재생성이 성립. "싸야 하니 struct"가 아니라 **"struct라 싸서 매번 버리는 설계가 성립"**

### 내부 동작 — 죽는 값, 사는 상태

```
일시적: 뷰 값 트리 (body의 산출물, 찰나)
지속:   렌더 트리 + @State 저장소 (SwiftUI 소유)
연결:   identity = 뷰 트리의 구조적 위치 + 타입 (structural identity)
        또는 .id()·ForEach의 id (explicit identity)
```

상태 변경 → body 재호출 → 새 값 트리 → 이전 값 트리와 diff → 지속 트리에 차이만
반영. 새로 태어난 뷰 값은 자기 identity 앞으로 보관된 저장소를 돌려받는다.

**상태의 수명은 뷰 값의 수명이 아니라 identity의 수명을 따른다.**
@State는 identity에 세 들어 산다.

> 저장소의 실제 자료구조(AttributeGraph)는 비공개 구현. 공개적으로 확언 가능한
> 선은 "SwiftUI가 identity 단위로 관리하는 저장소"까지다.
> — WWDC21 "Demystify SwiftUI"

### 대안과 트레이드오프

diff가 요구하는 성질은 struct 자체가 아니라 **"이전 값이 변경의 영향을 받지
않는다"**는 것. 달성 방법은 두 가지 — 복사해 두거나(값), 아무도 못 건드리게
하거나(불변).

| 설계 | 예 | 얻는 것 | 잃는 것 |
| --- | --- | --- | --- |
| 오래 사는 class 뷰 | UIKit UIView | 정밀 제어, identity 기반 API | 상태가 뷰에 흩어짐 → 수동 동기화 책임, 이전 소멸 → diff 불가 |
| 일시적 기술 + 불변 class | Flutter 위젯, React 엘리먼트 | 선언형 + diff — 변경이 아예 불가능해 과거가 보존됨 | 매 렌더 힙 할당 + GC 압력, 불변성이 관례·린트 수준 |
| 일시적 기술 + struct | SwiftUI | + 값 의미론을 언어가 강제, 할당·ARC 최소 | 선언형 공통의 제어권 위임 |

[[값 의미론 (Value Semantics)]]의 "불변 참조 타입(Java String)" 행과 같은 원리다.

### 언제 실패하는가 — 뷰의 두 가지 수명을 헷갈릴 때

**값의 수명**(한 번의 body, 찰나) vs **identity의 수명**(그 자리에 같은 뷰로
붙어 있는 동안).

| 착각 | 깨지는 코드 | 증상 |
| --- | --- | --- |
| 찰나를 길게 본다 | plain var에 상태 저장 / init에서 API 호출·무거운 생성 | 재생성마다 초기화 / 부작용이 초당 수십 번 실행 |
| 〃 | `@ObservedObject var vm = ViewModel()` | 부모 갱신마다 **백지 상태 → 유저에게 초기 화면** ([[@StateObject]]가 생성을 identity에 묶어 해결) |
| 영원으로 본다 | 분기·`.id()` 너머로 상태 유지 기대 / init을 화면 등장 시점으로 사용 | @State 통째 리셋 / 시점 어긋남 |

교정 원칙: **일은 자기 수명에 맞는 곳에.** 상태는 [[@State]], 생성은
[[@StateObject]](identity에 한 번), 비동기는 `.task`(등장 시 시작, 퇴장 시
자동 취소 — body 재호출에는 재시작되지 않는다).

### @State 리셋의 실체 — 변신이 아니라 사망과 출생

struct 값에는 identity가 없어, SwiftUI의 유일한 '같음' 기준은 자리(위치+타입)다.
if↔else 전환을 SwiftUI는 "수정"이 아니라 **"제거 + 삽입"**으로 본다.
identity 종료는 UIKit의 뷰 컨트롤러 deinit과 같다 — 주인이 죽었는데 짐(@State)을
보관하면 누수일 뿐이라 함께 폐기되고, 새 identity는 초기값으로 빈손 시작한다.

리셋 조건 = 좌표가 달라지는 모든 경우:
① if/else 분기 전환 (같은 타입이라도 남남)
② `.id(값)` 변경 (뒤집으면 의도적 초기화 도구)
③ ForEach에서 그 id의 항목 소멸
④ 트리에서 제거 후 재등장

**"같은 자리 + 같은 타입이면 같은 뷰. 자리나 이름표가 바뀌면 남남."**

### 실전 적용

뷰 안 데이터의 3분류 — 안 바뀌는 입력은 plain `let` / 이 뷰가 소유한 가변 상태는
[[@State]]·[[@StateObject]] / 빌려온 상태는 [[@Binding]]·[[@ObservedObject]].
init은 싸고 순수하게.

### 더 파볼 질문
- `.task(id:)`의 재실행 조건 — id가 바뀌면 기존 작업은 취소되고 새로 시작되는가?
- "언어 규칙 vs 관례"라는 강제력 차이가 실제로 어떤 버그를 차단하는가?
- ForEach의 id를 잘못 주면 상태가 엉뚱한 행에 붙는 시나리오 재현

### 관련
- [[값 의미론 (Value Semantics)]] — 언어 차원의 받침
- [[Identity]] — 일시적 값에 연속성을 부여하는 장치
- [[Diffing]] · [[선언형 UI (Declarative UI)]] · [[DataFlow]]

### 출처
- WWDC21 "Demystify SwiftUI" (Session 10022)
- WWDC20 "Data Essentials in SwiftUI" (Session 10040)
- WWDC19 "SwiftUI Essentials" (Session 216)
- React 공식 문서 "Reconciliation" · Flutter 공식 문서 "Widgets are immutable"
