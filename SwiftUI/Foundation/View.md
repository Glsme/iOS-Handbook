# View 프로토콜 (View Protocol)
"화면 조각"의 계약. 규칙만 준수하면 되는 **청사진**이라 물려받기(상속)가 없고,
그래서 class를 요구하지 않는다 — struct 뷰와 [[값 의미론 (Value Semantics)]]이 여기서 가능해진다.

```swift
// iPhoneOS 26.2 SDK, SwiftUICore.swiftinterface:13891 실측
@MainActor public protocol View {
    associatedtype Body : View                      // 재귀 제약 — body의 산출물도 View
    @ViewBuilder @MainActor var body: Self.Body { get }
    // + 언더스코어 요구사항 3개 (_makeView 등) — extension View의 기본 구현이 채운다
}
```

### 왜 필요한가

- **프로토콜이라서 struct가 가능하다.** 상속은 "물려받기"이고 물려받기는 class 전용.
  프로토콜은 물려주는 게 없어 신분(struct/enum/class)을 묻지 않는다.
  class-only였다면 **이전 값과 비교할 수 없어 [[Diffing]]이 근본적으로 무너진다**
  ([[값으로서의 View (View as Value)]]).
- **View는 "그릴 수 있다"가 아니라 "환원할 수 있다"는 계약이다.** `associatedtype Body: View`
  한 줄이 "모든 뷰는 더 원시적인 뷰로 풀린다"를 타입 시스템으로 보장한다.
- **타입에 구조 지도가 실린다.** `VStack<TupleView<(Text, Button<Text>)>>` — 이 지도가
  [[Identity]](structural identity)와 [[Diffing]]의 재료다.

### 내부 동작 — 환원의 사다리

body는 그리는 함수가 아니라 **설명서를 내놓는 함수**다. SwiftUI가 body를 불러
한 단계 원시적인 뷰를 얻고, 또 부르고 — **`Body = Never`인 뷰(primitive)에서 멈춘다.**

```
MyView ──body──▶ VStack<TupleView<(Text, Button<Text>)>>   Body = Never ← 멈춤
                          │
                          ▼
              _makeView(view:inputs:)   ← 실제 렌더 진입점 (언더스코어 API)
```

- `Text`·`Color`·`VStack`·`TupleView` 전부 `typealias Body = Swift.Never` (SDK 실측).
  `extension Never: View`도 실재한다 — 재귀가 자기 자신으로 닫힌다.
- **Never는 값이 0개인 enum**(Bool 2, Void 1, Never 0). `var body: Never`는 "존재하지
  않는 값을 반환하라"라서, 억지로 호출하면 무한루프가 아니라 **한 발짝도 못 가고 즉시 크래시**다.
  "여기가 재귀의 바닥"을 타입으로 선언한 트릭.
- body의 호출 시점·횟수는 **위로도 아래로도 비보장** — 무수히 불릴 수도, 안 불릴 수도.
  부작용 금지의 진짜 이유 ([[뷰 업데이트 사이클 (View Update Cycle)]]).
- `@MainActor`가 프로토콜에 박혀 있어 body는 항상 메인 액터에서 호출된다.

> 일반 뷰가 기본 구현 `_makeView`에서 "body를 꺼내 재귀한다"는 것은 구조로부터의
> 재구성이며, 기본 구현의 본문은 비공개다. 요구사항·기본 구현·primitive의 개별 구현
> 존재까지가 실측 사실.

### 컴파일러가 나 대신 채우는 것

`struct MyView: View { var body: some View { Text("hi") } }` 한 줄로 충분한 이유 — 세 조력자:

| 빈칸 | 채우는 자 |
| --- | --- |
| `associatedtype Body` | **컴파일러**가 body 반환 타입에서 역추론 (개발자는 typealias를 쓴 적 없다) |
| `@ViewBuilder` `@MainActor` | **프로토콜 요구사항 선언에 이미 인쇄**되어 있어 자동 적용 |
| `_makeView` 3형제 | **`extension View`의 기본 구현** — 프로토콜 본체엔 구현이 못 실린다 |

**@ViewBuilder의 핵심 트릭**: if/else 두 갈래를 **"Text 또는 Image"라는 하나의 타입**
(`_ConditionalContent<Text, Image>` — Optional처럼 케이스 둘, 타입 하나)으로 접는다.
`some`의 "단 하나의 구체 타입" 규칙이 봉투 타입 하나로 만족된다.
**`return`을 쓰면 포장이 꺼진다** — 두 타입이 맨몸으로 나와 충돌, 컴파일 에러.

### 대안과 트레이드오프

| 설계 | 예 | 얻는 것 | 잃는 것 |
| --- | --- | --- | --- |
| 기반 클래스 상속 | UIKit `UIView` | identity 내장, 정밀 제어 | class 강제 → 이전 값 소멸 → diff 불가 |
| 추상 클래스 + `build()` | Flutter `Widget` | 선언형 + 환원 (구조는 SwiftUI와 동형, `RenderObjectWidget` = primitive) | 참조 타입이라 매 build 힙 할당 + GC |
| 함수 = 컴포넌트 | React | 계약 최소 | 타입에 구조가 안 실림 → 짝짓기 전부 런타임 |
| `body: AnyView`로 설계 | (가상) | 배열·분기 자유, `some`·빌더 불필요 | **타입 속 지도 소멸** → identity 연결 불가, 통째 재생성 |
| associatedtype 프로토콜 | SwiftUI | 지도 → diff 재료, struct 허용 | existential 불가, 타입 폭발 |

**some vs any (상자 비유)**: `some` = 내용물이 하나로 정해진 밀봉 상자(컴파일러는 안다),
`any` = 정말로 안 정해진 상자(existential). `any View`는 "네 Body가 뭐냐"는 면접 질문에
답할 수 없어 **View 준수 자체가 안 된다** → body 자리에 못 온다. 수동 탈출구가 `AnyView`.

### 언제 한계에 닿는가

- **바닥 뷰를 직접 만들 수 없다** — primitive가 되려면 `_makeView`(비공개 SPI)가 필요.
  새 그리기 원시 요소는 `Canvas`·`Shape`·`UIViewRepresentable`로.
- **타입 폭발** — 지도를 타입에 싣는 대가. body가 크면 타입 추론이 감당 못 해
  `unable to type-check this expression in reasonable time`.
  **뷰 쪼개기는 가독성 문제이기 전에 컴파일러 문제다.**
- **`[any View]`·`func f() -> View` 불가** — associatedtype의 대가. 유동적 타입 조합이 막힌다.
- **AnyView의 철거 조건** — 비교 시점에 **내부 런타임 타입이 바뀌었는가**가 기준이다.
  값만 다르면 **업데이트**(도배, @State 유지), 타입이 다르면 **철거 후 신축**(@State 소멸).
  상태 변경은 비교를 일으키는 방아쇠일 뿐 철거를 결정하지 않는다.

### 기반 개념

**① 불투명 반환 타입(some, SE-0244)** · **② associatedtype 프로토콜** — 지도의 뿌리이자
existential 불가의 뿌리 (이득과 대가가 같은 한 줄) · **③ Result Builder(SE-0289)** ·
**④ Never(uninhabited type)** · **⑤ [[값 의미론 (Value Semantics)]]** · **⑥ 프로토콜 익스텐션** — 기본 구현 배달

### 더 파볼 질문
- `some View`의 언어 메커니즘 — 제네릭·`any`와의 정확한 차이 → [[Backlog]] (Tier 0 기존 항목)
- `@ViewBuilder`의 변환 규칙 — buildBlock·buildEither가 실제로 하는 일 → [[Backlog]] (Tier 0 기존 항목)
- Never / uninhabited type — SwiftUI 밖에서의 쓰임 (`Result<T, Never>`) → [[Backlog]]
- 프로토콜 익스텐션 기본 구현의 디스패치 — 요구사항 유무에 따라 정적/동적이 갈리는 규칙
- `_makeView` 이후의 세계 — 렌더 트리·CoreAnimation 백킹까지 실제 픽셀 경로

### 관련
- [[선언형 UI (Declarative UI)]] — `UI = f(state)`의 f가 사는 곳
- [[값으로서의 View (View as Value)]] — 이 계약이 struct에 허락한 것
- [[Identity]] · [[Diffing]] — 타입 속 지도의 소비자
- [[@State]] · [[DynamicProperty]]

### 출처
- SDK 실측 — iPhoneOS 26.2, SwiftUICore.swiftinterface (프로토콜 :13891, 기본 구현 :4839,
  Never:View :15817, VStack :1156, TupleView :2657, _ConditionalContent :16873)
- WWDC19 "SwiftUI Essentials" (216) · WWDC21 "Demystify SwiftUI" (10022)
- SE-0244 Opaque Result Types · SE-0289 Result Builders · SE-0309 Unlock Existentials
- Apple Documentation — View, View.body · Flutter Docs — Widget, RenderObjectWidget
