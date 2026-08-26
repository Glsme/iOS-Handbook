# 학습 백로그

> 작업 중 만난 모르는 개념·질문을 쌓는 학습 지도. deep-dive 스킬이 관리한다.

## 미착수

> 맥락: 2026-08-06, SwiftUI 핵심 개념 + 상태 관리 전체 키워드 리스트업.
> Tier 0 → 7 순서가 권장 학습 순서다. Tier 0~1이 상태 관리의 본체이고, 나머지는 필요할 때 꺼내 쓴다.

### Tier 0 — 토대 (이걸 모르면 상태 관리가 이해되지 않는다)

- [ ] **@ViewBuilder / Result Builder** — 중괄호 안에 나열한 뷰들이 어떻게 하나의 타입으로 합쳐지는가?
  - 맥락: 2026-08-08 [[View]] 딥다이브에서 _ConditionalContent 포장("Text 또는 Image라는 하나의 타입",
    return이 포장을 끈다)까지 인출 완료 — buildBlock·buildEither 변환 규칙이 남은 몫
- [ ] **Source of Truth / Derived Value** — 어떤 데이터를 상태로 두고 어떤 걸 계산으로 둘 것인가?
  - 연관: [[DataFlow]]
- [ ] **AttributeGraph** — SwiftUI 내부의 의존성 그래프. 비공개 구현이라 어디까지가 확인된 사실인가?
- [ ] **body의 순수성과 부작용** — 호출 시점·횟수가 보장되지 않는다는 계약을 어기면 실제로 어떤 버그로 나타나는가? 부작용은 어디에 둬야 하는가(`.task`, `.onChange(of:)`)?
  - 맥락: 2026-08-06, 선언형 UI 딥다이브 중 등장
  - 연관: [[선언형 UI (Declarative UI)]]

### Tier 1 — 상태 관리 코어

- [ ] **@EnvironmentObject** — 뷰 계층을 건너뛴 주입. 주입을 잊으면 왜 런타임 크래시인가? (링크 있으나 노트 없음)
  - 맥락: 2026-08-08 [[@StateObject]] 딥다이브 대안 표에서 세 번째 소유 방식으로 등장 — 소유/빌림 축을 완성하는 자리
- [ ] **@Environment / EnvironmentValues / EnvironmentKey** — 환경 값은 어떻게 아래로 전파되는가? 커스텀 키 정의와 iOS 18 `@Entry` 매크로 (링크 있으나 노트 없음)
- [ ] **Observation / @Observable** — iOS 17의 새 관찰 방식. [[ObservableObject]]의 객체 단위 무효화 문제를 어떻게 프로퍼티 단위로 바꿨나? (링크 있으나 노트 없음)
  - 맥락: 2026-08-08 [[@StateObject]] 딥다이브 재인출에서 "프로퍼티 단위 무효화"는 인출 성공했으나
    **대체 경로(@StateObject→@State, @ObservedObject→plain `let`)는 미통과** — @Query로 오답. 여기서 이어서
  - 연관: [[ObservableObject]], [[@Published]], [[Diffing]], [[@StateObject]]
- [ ] **withObservationTracking** — `@Observable` 매크로가 실제로 만들어내는 추적 메커니즘은?
- [ ] **@Bindable** — `@Observable` 객체에 양방향 바인딩을 만드는 래퍼. [[@Binding]]과 어떻게 다른가?
- [ ] **objectWillChange** — [[ObservableObject]]의 진짜 갱신 신호. willChange인 이유는?
- [ ] **projectedValue와 `$`** — `$count`가 실제로 반환하는 것은 무엇인가?
- [ ] **Binding 직접 만들기** — `Binding(get:set:)`, `.constant(_:)`를 언제 쓰는가?

### Tier 3 — 렌더링과 성능 (핸드북에 일부 있음)

- [ ] **onChange(of:)** — iOS 17에서 시그니처와 호출 시점이 바뀐 이유는?
- [ ] **Lazy 컨테이너와 상태 수명** — `LazyVStack`에서 화면 밖으로 나간 뷰의 [[@State]]는 어떻게 되는가?
- [ ] **@State 남용과 body 재호출 비용** — 상태를 어느 높이에 두느냐가 성능에 미치는 영향
  - 연관: [[Identity]], [[Diffing]], [[EquatableView]]
- [ ] **뷰 트리의 인라인 평탄화와 AnyView의 비용** — `VStack { Text; Button }`이 런타임에 `VStack<TupleView<(Text, Button<Text>)>>` **값 하나**로 평탄화되는 것이 "재생성이 싼" 실체다. 그런데 [[값으로서의 View (View as Value)]]에는 "공짜 재생성"으로 적어 뒀는데, 뷰 struct는 String·클로저 같은 참조 필드를 품고 비교를 위해 프레임워크 저장소로 escape한다 — 어디까지가 공짜인가?
  - 맥락: 2026-08-13, struct/class 메모리 비용 문답이 "그럼 SwiftUI가 View를 struct로 쓰는 이유가 이건가"로 이어짐
  - 파볼 질문: ① 노드당 힙 할당이 0이 되는 근거는 수명이 아니라 **크기·타입의 정적 확정**(제네릭 특수화)이라는 점 확인 ② `AnyView`가 한 번에 깨뜨리는 것 — existential 박싱(힙 할당) + 정적 타입 소실로 인한 diff·identity 손실 ③ body 결과가 다음 비교를 위해 보관된다면 "스택에서 pop되며 사라진다"는 모델은 어디서 틀리는가
  - 연관: [[값으로서의 View (View as Value)]], [[Diffing]], [[Identity]], [[View]] · [[Opaque Type]](②의 existential 박싱은 여기서 이어짐), "타입 소거"·"제네릭" 항목(Tier 외)

### Tier 4 — 앱 구조와 레이아웃

- [ ] **App / Scene / WindowGroup / @main** — SwiftUI 앱의 진입점과 씬 구조
- [ ] **ScenePhase** — active/inactive/background 전환을 상태로 받기
- [ ] **레이아웃 협상 3단계** — 부모가 크기를 제안 → 자식이 결정 → 부모가 배치. UIKit의 프레임 계산과 무엇이 다른가?
- [ ] **ViewModifier와 modifier 순서** — `.padding().background()`와 `.background().padding()`이 다른 이유는?
- [ ] **GeometryReader** — 크기를 읽는 대가로 무엇을 잃는가?
- [ ] **PreferenceKey** — 자식에서 부모로 값을 거슬러 올리는 통로
- [ ] **Layout 프로토콜** — iOS 16의 커스텀 레이아웃
- [ ] **alignment guide** — 정렬 기준선을 직접 정의하기

### Tier 5 — 내비게이션·리스트·이벤트

- [ ] **NavigationStack / NavigationPath / navigationDestination** — 내비게이션을 상태로 다루기. `NavigationView`에서 왜 갈아엎었나?
- [ ] **sheet / fullScreenCover / alert / confirmationDialog** — 모달을 Bool·Optional 상태로 띄우는 방식
- [ ] **List / ForEach와 Identifiable** — 잘못된 id가 어떤 버그로 나타나는가? 항목의 id가 소멸하면 그 행의 @State는? 잘못된 id로 상태가 엉뚱한 행에 붙는 시나리오 재현
  - 맥락: 2026-08-07 [[값으로서의 View (View as Value)]] 딥다이브 — identity 리셋 케이스 중 유일하게 끝까지 미인출된 약점
  - 연관: [[Identifiable]], [[Identity]]
- [ ] **ScrollView / scrollPosition / scrollTargetBehavior** — 스크롤 상태 제어
- [ ] **task / onAppear / onDisappear** — 뷰 생명주기 훅과 비동기 작업 취소. `.task(id:)`의 재실행 조건은? (id가 바뀌면 기존 작업은 취소되는가)
  - 맥락: 2026-08-07 [[값으로서의 View (View as Value)]] 딥다이브에서 ".task = 무한 호출 위험" 오개념 교정 — "body 재호출에 재시작되지 않는다"를 명시적으로 인출하는 게 숙제
- [ ] **searchable / refreshable / toolbar** — 시스템 제공 인터랙션 modifier

- [ ] **Transaction 전파와 `.transaction` modifier** — 봉투가 뷰 트리를 타고 흐를 때 우선순위 규칙은? `withTransaction`과의 관계는?
  - 맥락: 2026-08-07, 뷰 업데이트 사이클 딥다이브에서 "트랜잭션은 어느 단계냐"는 질문으로 뚫린 갈래
  - 연관: [[뷰 업데이트 사이클 (View Update Cycle)]]

### Tier 7 — UIKit 연동과 아키텍처

- [ ] **UIViewRepresentable / UIViewControllerRepresentable / Coordinator** — UIKit 뷰를 SwiftUI에 얹기. `updateUIView` 호출 시점은?
- [ ] **UIHostingController** — SwiftUI를 UIKit에 얹기. 데이터 흐름 표준이 없는 영역
  - 연관: [[DataFlow]]
- [ ] **단방향 데이터 흐름 (Unidirectional Data Flow)** — MVVM·MV·TCA가 나뉘는 지점
  - 연관: [[DataFlow]]

### Swift 언어 (Tier 외)

- [ ] **배타적 접근 (Law of Exclusivity, SE-0176)** — 겹치는 접근 금지를 컴파일 타임/런타임에서 각각 무엇으로 어떻게 강제하는가? 그리고 이것이 값 의미론과 정확히 어떤 관계인가(근거인가 보조 받침인가)?
  - 맥락: 2026-08-06, 값 의미론 딥다이브에서 기반 개념(언어 차원의 받침)으로 등장.
    2026-08-07 노트 교정에서 "값을 바꾸는 경로는 그 변수뿐을 강제"라는 요약이 과했다고 보아 본문을 "보조 받침"으로 낮춤 — 이 항목에서 확정할 것
  - 연관: [[값 의미론 (Value Semantics)]]
- [ ] **CoW 직접 구현** — `isKnownUniquelyReferenced`로 내 타입에 값 의미론 + 지연 복사를 어떻게 구현하는가? "복사 조건 = 변경 × 공유"를 코드로 체화하기
  - 맥락: 2026-08-06, 값 의미론 딥다이브에서 표준 라이브러리의 CoW 메커니즘을 배우고 남은 실습 과제
  - 연관: [[값 의미론 (Value Semantics)]]
- [ ] **래퍼 합성 `@A @B var x`** — A의 `wrappedValue` 타입이 B여야 한다는 타입 체인 규칙. 중첩 래핑은 어디까지 되는가?
  - 맥락: 2026-08-07, @propertyWrapper 딥다이브에서 컴파일러 실측 중 등장(`composed wrapper type does not match`)
  - 연관: [[@propertyWrapper]]
- [ ] **`init(projectedValue:)`와 SE-0293** — `$`로 초기화하는 경로는 언제 쓰는가? 함수 파라미터에 붙는 property wrapper는 무엇을 합성하는가?
  - 맥락: 2026-08-07, @propertyWrapper 딥다이브 — 파라미터·지역 변수에 붙는 것이 실측으로 확인됐으나 용도는 미확인
  - 연관: [[@propertyWrapper]], projectedValue와 `$`(Tier 1 기존 항목)
- [ ] **커스텀 래퍼 직접 작성 실습** — `@Clamped`·`@Capped`를 자료 없이 처음부터 코드로 쓰기. 창고(백킹 스토리지)와 창구(wrappedValue) 분리를 손으로 체화
  - 맥락: 2026-08-07, @propertyWrapper 딥다이브에서 코드 작성 문항을 스킵해 미실습으로 남음
  - 연관: [[@propertyWrapper]], [[DynamicProperty]]
- [ ] **제네릭 (Generics)** — 타입을 정하는 쪽은 왜 호출자인가? 특수화(specialization)는 무엇을 하는가?
  - 맥락: 2026-08-09 [[Opaque Type]] 딥다이브에서 결정권 축("컴파일러가 정한다" 오답)이 2라운드 연속 미통과 — 역방향 제네릭을 지탱하는 기반
  - 연관: [[Opaque Type]]
- [ ] **타입 소거 (Type Erasure)** — AnyView·AnyPublisher가 손으로 하는 일은 무엇인가? `any`(컴파일러 제공 상자)와 수동 소거는 무엇이 다른가?
  - 맥락: 2026-08-09 [[Opaque Type]] 딥다이브에서 existential 컨테이너와 함께 등장
  - 연관: [[Opaque Type]], [[View]]
- [ ] **Mirror / 리플렉션** — Swift 런타임 메타데이터는 무엇을 알고 있고, Mirror는 어디까지 보여주는가? SwiftUI의 래퍼 "심사"가 서는 기반
  - 맥락: 2026-08-07, DynamicProperty 딥다이브 중 "리플렉션이 뭐야?" 질문에서 등장
  - 연관: [[DynamicProperty]]
- [ ] **Never / uninhabited type** — 값이 0개인 타입은 무엇을 표현하는 도구인가?
  `Result<T, Never>`·`fatalError()`의 반환 타입·SwiftUI primitive의 `Body = Never`를 관통하는 원리
  - 맥락: 2026-08-08 [[View]] 딥다이브에서 "무한 호출로 멈춘다" 오개념을 교정한 지점 —
    "값 0개 → 반환 불가 → 즉시 크래시"는 정착, SwiftUI 밖의 쓰임이 남은 몫
  - 연관: [[View]]
- [ ] **memberwise initializer 합성 규칙** — property wrapper가 붙은 저장 프로퍼티는 파라미터로 어떻게 노출되는가? 합성된 initializer의 접근 수준은 무엇이 정하는가? `@autoclosure` 지연은 그 경로에서도 유지되는가?
  - 맥락: 2026-08-08, [[@StateObject]] 딥다이브에서 "private을 빼면 왜 위험한가"를 컴파일러 실측으로 확인했으나
    재인출 미통과 — 언어 층 규칙 자체가 빈칸이었다("충돌"의 정체 = 두 계약의 모순, private이 막는 것 = 합성 initializer)
  - 연관: [[@StateObject]], [[@propertyWrapper]], [[@State]]
- [ ] **struct/class의 메모리 비용** — struct가 싼 진짜 이유는 "스택을 써서"가 아니라 **수명이 컴파일 타임에 확정돼 런타임 부기(ARC)가 필요 없어서**다. 그렇다면 참조 필드를 품은 struct는 언제 class보다 비싸지는가?
  - 맥락: 2026-08-13, "struct가 class보다 생성·해제 비용이 저렴하다"는 명제가 참인지 따지다 등장
  - 파볼 질문: ① 스택 할당(sp 이동 1회) vs malloc(size class 판별·존·free list·락·시스템 콜)의 실제 비용 차이 ② 원자적 retain/release가 비싼 이유는 명령어 수인가, 캐시라인 배타 소유권과 주변 컴파일러 최적화 차단인가 ③ 참조 필드 N개짜리 struct 복사(retain N회)가 class 참조 복사(retain 1회)를 역전하는 지점 ④ escaping 클로저 캡처·existential 3워드 초과가 struct를 힙으로 끌고 가는 경계 ⑤ SROA·레지스터 전달로 할당 자체가 소멸하는 조건 ⑥ Instruments에서 `swift_allocObject`·`swift_retain` 비중을 실측하는 법
  - 연관: [[값 의미론 (Value Semantics)]] — "저장 위치는 구현 세부사항"이라며 남겨 둔 지점이 정확히 여기.
    ④는 [[Opaque Type]]의 existential 컨테이너와 같은 메커니즘 — "타입 소거" 항목과 붙여서 볼 것

---

### ▍2026-08-14 배치 — Swift/iOS 기본기 (14항목)

> 2026-08-14, Swift/iOS 기본기를 점검하며 한 번에 리스트업했다.
> 위 SwiftUI 배치(2026-08-06)와 출처가 달라 따로 묶어 둔다.
> 언어·디스패치를 뺀 나머지 6개 영역은 **핸드북에 노트가 하나도 없다** — 각 항목의 첫 질문이 곧 진입점이다.
> 권장 순서: 언어 → 메모리 관리 → Swift Concurrency → 나머지(독립적이라 필요할 때 꺼내 쓴다).

#### 언어 — 타입과 디스패치

- [ ] **`@objc dynamic`과 message dispatch** — objc_msgSend 경로는 언제 *필요*한가? KVO·swizzling·selector가 이 경로를 요구하는 이유는?
  - 맥락: 2026-08-25, 메서드 디스패치 딥다이브에서 범위를 좁히려고 의도적으로 제외한 갈래
  - 연관: [[메서드 디스패치 (Method Dispatch)]]

- [ ] **vtable (Virtual Method Table)** — non-final class 인스턴스의 실제 타입에서 오버라이드된 메서드 구현을 어떤 경로로 찾는가?
  - 맥락: 2026-08-20, struct 상속 제약과 class 디스패치를 구분하는 딥다이브에서 발견
  - **2026-08-26: 경로·슬롯 확정은 [[메서드 디스패치 (Method Dispatch)]]에서 정리 완료.** 남은 몫은 표의 물리적 레이아웃(타입 메타데이터 항목과 함께 볼 것)
  - 연관: [[Protocol]], [[메서드 디스패치 (Method Dispatch)]]
- [ ] **witness table** — 프로토콜 요구사항과 요구사항 밖 extension 메서드의 호출 경로가 왜 갈리는가?
  - 맥락: 2026-08-20, struct 상속 제약과 class 디스패치를 구분하는 딥다이브에서 발견
  - **2026-08-26: 이 질문 자체는 [[메서드 디스패치 (Method Dispatch)]]에서 해소됨**(요구사항만 표에 칸을 얻는다). 남은 몫은 PWT의 실제 레코드 구조와 조건부 준수의 런타임 인스턴스화
  - 연관: [[Protocol]], [[메서드 디스패치 (Method Dispatch)]]

- [ ] **Swift Intermediate Language (SIL)** — 타입 검사를 마친 Swift 소스는 어떤 단계를 거쳐 SIL로 낮아지며, SILGen·정규화/최적화 SIL·LLVM IR은 각각 무엇을 담당하는가?
  - 맥락: 2026-08-20, Swift 컴파일러가 타입·디스패치·ARC 결정을 표현하는 중간 단계를 이해하기 위해 추가
  - 연관: 메서드 디스패치 항목, [[Opaque Type]], ARC 항목

- [ ] **재귀 enum과 `indirect`** — 직접·상호 재귀 enum의 크기 계산을 indirect가 어떻게 박싱해 끊는가? class 재귀와 어떤 트레이드오프인가?
  - 맥락: 2026-08-20, struct의 직접 재귀와 간접 참조를 구분하는 딥다이브에서 발견
  - 연관: [[값 의미론 (Value Semantics)]], 힙과 스택 항목
- [ ] **타입 메타데이터 (Type Metadata)** — 컴파일러와 런타임은 타입의 크기·정렬·필드·프로토콜 준수 정보를 각각 어디에 두고 언제 조회하는가?
  - 맥락: 2026-08-20, class 인스턴스에서 vtable로 가는 경로를 학습하며 발견
  - 연관: vtable 항목, witness table 항목, Mirror / 리플렉션 항목

- [ ] **컴파일러 최적화 파이프라인 — 인라인 다음에 오는 것들** — 인라인이 열어주는 최적화들은 서로를 어떻게 물고 이어지는가? 그리고 devirtualization이 *원리적으로* 불가능해지는 조건은?
  - 맥락: 2026-08-26, 메서드 디스패치 딥다이브에서 "동적 디스패치 비용의 본체 = 최적화 연쇄가 끊기는 것"까지는 정착했으나, 그 연쇄의 내용물은 디스패치가 아니라 최적화 쪽 주제라 분리
  - 파볼 질문: ① 상수 전파·죽은 분기 제거·루프 불변식 이동·SROA가 인라인 뒤에 이어지는 순서 ② 증명이 서지 않는 조건 — `open`, library evolution, 모듈 경계를 넘는 `public` 호출, `[any P]` 순회 ③ `@inlinable`이 모듈 경계에서 여는 것과 그 대가(ABI 고정) ④ Debug/Release에서 실제로 달라지는 것을 SIL로 확인하는 법
  - 연관: [[메서드 디스패치 (Method Dispatch)]], Swift Intermediate Language (SIL) 항목

#### 메모리 관리

- [ ] **ARC (Automatic Reference Counting)** — 참조 카운트를 올리고 내리는 코드는 누가 언제 넣는가? 카운트가 0이 되면 정확히 무슨 일이 일어나는가?
  - 파볼 질문: ① retain/release 삽입이 컴파일 타임 결정이라는 점과 그 함의 ② strong·weak·unowned 카운트가 각각 어디에 기록되는가(객체 헤더 vs side table) ③ 추적형 GC와 비교해 얻는 것(결정적 해제, 일시정지 없음)과 잃는 것(순환 참조를 스스로 못 끊음) ④ `deinit` 호출 시점과 프로퍼티 해제 순서 ⑤ 힙 객체 헤더에 실제로 무엇이 들어 있는가
  - 연관: [[값 의미론 (Value Semantics)]], struct/class의 메모리 비용 항목(Swift 언어)
- [ ] **강한 참조 순환 (Strong Reference Cycle)** — 두 객체가 서로를 강하게 참조하면 왜 ARC가 손을 못 대는가? 클로저가 낄 때 고리는 어디서 닫히는가?
  - 파볼 질문: ① 순환의 최소 조건은 "캡처했다"가 아니라 "서로 붙잡는 고리가 닫혔다"라는 구분 ② escaping 클로저에서 completion을 호출하지 않으면 왜 영구 미해제인가 ③ Timer·NotificationCenter처럼 시스템이 붙잡는 경우는 순환인가 단순 수명 연장인가 ④ 캡처 리스트 `[weak self]`가 실제로 하는 일과 캡처가 확정되는 시점 ⑤ Memory Graph Debugger·Leaks로 고리를 찾는 법
  - 연관: [[클로저 (Closure)]]
- [ ] **weak / unowned** — 둘 다 카운트를 올리지 않는데, 대상이 먼저 사라졌을 때 무엇이 달라지는가? unowned를 굳이 고를 이유는?
  - 파볼 질문: ① weak가 자동으로 nil이 되는 메커니즘(side table, zeroing) ② `unowned`와 `unowned(unsafe)`의 실패 방식 차이 ③ weak 접근이 unowned보다 비싼 이유 ④ 수명 포함 관계(부모-자식)를 unowned로 표명해도 되는 판별 기준 ⑤ 크래시를 의도적으로 남기는 것이 설계 선택이 되는 경우 — required 의존성을 `fatalError` 맥락으로 강제하기
- [ ] **힙과 스택** — 할당·해제 비용 차이는 어디서 오는가? Swift에서 어떤 값이 어느 쪽에 놓이는지는 무엇이 결정하는가?
  - 파볼 질문: ① 스택 프레임의 sp 이동 1회 vs malloc의 전체 경로 ② struct가 힙으로 끌려가는 조건(클래스 프로퍼티에 담김, escaping 클로저 캡처, existential 3워드 초과) ③ "힙은 선형적이지 않다"의 실체 — free list, size class, 단편화 ④ iOS의 메모리 압박·jetsam과 힙 사용량의 관계
  - 연관: [[가상 메모리(Virtual Memory)의 개념과 동작 원리]], struct/class의 메모리 비용 항목(Swift 언어)

#### Swift Concurrency

- [ ] **Structured Concurrency** — 부모-자식 관계를 언어가 보장하면 무엇이 공짜로 따라오는가?
  - 파볼 질문: ① `async let`과 `withTaskGroup`이 각각 맞는 상황 ② "자식이 끝나기 전에 스코프를 벗어날 수 없다"는 규칙이 취소·에러 전파를 어떻게 단순하게 만드는가 ③ 상속되는 것의 정확한 목록 — 우선순위, actor 컨텍스트, task-local 값 ④ 취소가 협조적(cooperative)인 이유와 `Task.checkCancellation()`을 넣지 않았을 때의 결과
  - 연관: [[Sendable]]
- [ ] **Unstructured Concurrency — Task vs Task.detached** — `Task {}`가 상속하는 것과 `Task.detached`가 버리는 것의 정확한 목록은?
  - 파볼 질문: ① Task는 actor 컨텍스트·우선순위·task-local을 상속하는데도 왜 unstructured인가(수명이 스코프에 묶이지 않음) ② 취소를 직접 관리한다는 것의 실무적 의미 — 핸들 보관, 소멸 시 cancel ③ `.task` modifier가 뷰 수명에 자동으로 묶이는 이유 ④ detached가 정당한 좁은 사례는 무엇인가
- [ ] **MainActor와 actor 격리** — UI 상태 갱신이 안전한 근거가 "내부 직렬 큐"가 아니라 격리라면, 격리는 컴파일 타임에 무엇을 강제하는가?
  - 파볼 질문: ① actor 격리와 DispatchQueue 직렬화의 차이 — 재진입(reentrancy), 큐를 점유하지 않고 await로 양보 ② `@MainActor`가 타입·함수·프로퍼티에 붙을 때 각각 달라지는 것 ③ `nonisolated`·`MainActor.assumeIsolated`가 필요해지는 지점 ④ Swift 6 엄격 동시성에서 런타임 경고가 컴파일 에러로 옮겨간 범위
  - 연관: [[Sendable]], [[뷰 업데이트 사이클 (View Update Cycle)]], [[Migrate Swift 6]]
- [ ] **프로퍼티 래퍼의 스레드 안전성** — 래퍼 자체는 아무것도 보장하지 않는다면, 안전을 붙이는 방법마다 무엇을 희생하는가?
  - 파볼 질문: ① 직렬 큐 / 락(`os_unfair_lock`) / actor / atomic 네 구현의 비용과 한계 ② `wrappedValue`의 get-modify-set이 원자적이지 않아 생기는 read-modify-write 경합 ③ 래퍼를 actor로 만들 수 없는 이유(get/set이 동기 요구사항이라는 점) ④ Swift 6에서 래퍼가 Sendable을 만족하려면 무엇이 필요한가
  - 연관: [[@propertyWrapper]], [[Singleton Multi Thread 전략]], [[Sendable]]

#### Combine · 시간 기반 연산자

- [ ] **debounce / throttle** — 둘 다 이벤트를 줄이는데 "무엇을 기준으로" 줄이는가? 어떤 입력에 어느 쪽이 맞는가?
  - 파볼 질문: ① debounce는 마지막 이벤트 이후 침묵 구간을 기다리고, throttle은 고정 주기당 최대 1회 — 타임라인을 직접 그려 구분하기 ② throttle의 `latest` 옵션이 바꾸는 것(첫 값 vs 최신 값) ③ 이벤트가 끊기지 않고 계속 들어오면 debounce는 영원히 방출하지 않는가 ④ Combine과 RxSwift의 동작·기본값 차이 ⑤ `removeDuplicates`, `collect(.byTime:)`와 무엇이 다른가
  - 연관: [[Combine]], [[Publisher]]

#### 빌드·링킹

- [ ] **정적 링크 vs 동적 링크** — 라이브러리가 앱에 합쳐지는 시점이 다르면 무엇이 연쇄적으로 달라지는가?
  - 파볼 질문: ① 정적은 링커가 최종 실행 파일에 심볼을 합치고, 동적은 dyld가 실행 시 로드·바인딩한다는 구조 ② 바이너리 크기 / 런치 시간 / 프로세스 간 메모리 공유의 트레이드오프 ③ 정적 라이브러리를 여러 모듈이 참조할 때의 중복 심볼 문제 ④ `.framework`·`.xcframework`·static framework·Swift Package의 linkage 설정이 실제로 고르는 것 ⑤ 동적 라이브러리 개수가 앱 런치 시간에 미치는 영향(dyld 작업량)
  - 맥락: 2026-08-26, 모듈이 여러 개로 나뉜 앱에서 링크 방식을 바꿔가며 Release 빌드를 실측 비교하던 중 ①②⑤를 부분 인출.
    확인한 것 — 동적 라이브러리는 export 심볼을 전부 살려야 해서 **dead stripping이 모듈 경계를 넘지 못한다**는 점,
    dylib당 고정 오버헤드(Mach-O 헤더·export trie·코드서명·페이지 정렬 패딩)가 개수에 비례해 쌓인다는 점,
    **출하 실행 파일이 하나뿐이면**(앱 확장이 없으면) 동적의 코드 공유 이점이 아예 성립하지 않는다는 점.
    남은 몫 — ③ 중복 심볼, ④ linkage 설정이 실제로 고르는 것
  - 새로 열린 질문: 링크 방식이 **리소스 조회를 바꾼다**. `Bundle(for:)`는 코드가 실제로 어디 있느냐를 따라가
    정적/병합이면 앱 번들을, 동적이면 프레임워크 번들을 반환한다 — 리소스 조회 코드가 링크 방식에 묶이는 구조
  - 반증된 추정: `-all_load`가 정적 링크의 dead stripping 이득을 무력화한다고 짐작했으나 실측에서 기각.
    `-dead_strip`이 이미 켜져 있으면 링커가 전부 적재한 뒤 회수하므로 결과물이 같다 — `-all_load`가 지불하게 하는 건 크기가 아니라 링크 시간
  - 연관: Mergeable Libraries

- [ ] **Mergeable Libraries** — 동적으로 빌드해두고 링크 시점에 앱 바이너리로 접어 넣는 방식(WWDC 2023). 정적/동적의 트레이드오프를 정말 없앤 것인가, 아니면 옮긴 것인가?
  - 파볼 질문: ① 일반 dylib은 왜 다시 합칠 수 없는가 — 링크가 끝나면 무엇을 잃어버리고, `-make_mergeable`이 남기는 `LC_ATOM_INFO`가 그중 무엇을 복원해주는가 ② "1차 링크는 모듈별, 2차 링크만 전체"라는 구조가 Debug 증분 빌드와 Release 런타임을 동시에 챙기는 원리 ③ 병합되면 dead stripping이 모듈 경계를 넘게 되는 이유(닫힌 세계를 링커가 알게 됨) ④ 병합 후에도 남는 stub 라이브러리는 왜 필요한가 ⑤ Debug와 Release의 런타임 의미가 갈라지는 지점 — `Bundle(for:)` 반환값, `+load`·static initializer 실행 순서, 심볼 충돌 ⑥ prebuilt 바이너리(xcframework)는 왜 병합 대상이 될 수 없는가
  - 맥락: 2026-08-26, 모듈 링크 방식을 실측 비교하며 정적 전환의 세 번째 대안으로 적용해봄.
    실측으로 본 것 — mergeable로 빌드한 중간 산출물은 **+47~67% 커지고 그 증가분 대부분이 `LC_ATOM_INFO`**(한 모듈에서 전체 크기의 절반 이상),
    병합 후 남는 stub은 `__TEXT` 1페이지에 export 심볼 0개이며 앱의 로드 커맨드에서도 빠진다,
    병합 시 회수되는 크기는 약 18%(dylib 합계 대비 실행 파일 증가분의 차액)
  - 남은 의문: 이 방식의 **본래 명분인 Debug 증분 빌드 이득을 측정하지 못했다**(Release만 측정).
    Debug에서 reexport stub으로 심볼 해석을 Release와 맞춘다고 알고 있으나 미확인
  - 연관: 정적 링크 vs 동적 링크

- [ ] **Static Framework vs Dynamic Framework vs Mergeable Library** — 셋 중 무엇을 언제 고르는가? 판단 축은 무엇인가?
  - 파볼 질문: ① 판단을 가르는 1차 축 — 출하되는 실행 파일이 몇 개인가(앱 확장 유무)가 왜 다른 모든 고려보다 먼저인가 ② 런치 타임 / 앱 크기 / 증분 빌드 시간 중 무엇을 최적화하려느냐에 따라 답이 어떻게 갈리는가 ③ 세 방식의 `Bundle(for:)`·초기화 순서·심볼 격리 차이가 코드에 거는 제약 ④ 리소스 보유, prebuilt 여부, SwiftUI Preview 요구처럼 모듈별 사정이 선택을 강제하는 경우 ⑤ 한 프로젝트 안에서 방식을 섞는 것이 정당한 조건과 그 유지 비용
  - 맥락: 2026-08-26, 위 두 항목이 "어떻게 동작하는가"라면 이 항목은 "그래서 어떻게 고르는가" — 메커니즘이 아니라 결정 기준 쪽
  - 연관: 정적 링크 vs 동적 링크, Mergeable Libraries

- [ ] **dyld와 앱 런치 시퀀스** — 앱이 실행되기까지 dyld는 무엇을 하고, 이미지가 하나 늘면 정확히 어떤 비용이 붙는가?
  - 파볼 질문: ① 이미지 로드 → rebase → bind → Obj-C 런타임 등록 → initializer 실행까지의 순서 ② chained fixups(iOS 13.4+)가 bind 비용을 어떻게 줄였고, 그래서 "dylib 개수 = 런치 시간"의 기울기가 얼마나 완만해졌는가 ③ dyld shared cache에 있는 시스템 라이브러리와 앱이 임베드한 dylib의 비용 차이 ④ pre-main과 post-main의 경계, 그리고 각 구간에서 개발자가 손댈 수 있는 것 ⑤ 실제 측정 수단(Instruments App Launch, `DYLD_PRINT_STATISTICS`)
  - 맥락: 2026-08-26, 링크 방식 비교에서 파생. "동적 라이브러리 개수가 런치 시간을 늘린다"를 **전제로 깔고 판단했는데 정작 그 메커니즘 자체는 확인하지 않았다** — 전제를 검증하는 자리
  - 연관: 정적 링크 vs 동적 링크, Mergeable Libraries, Mach-O 구조

- [ ] **Mach-O 구조 (세그먼트·섹션·로드 커맨드)** — 실행 파일과 dylib은 어떤 형식이며, 무엇을 읽으면 링크·로드 동작을 남의 설명 없이 검증할 수 있는가?
  - 파볼 질문: ① 헤더 → 로드 커맨드 → 세그먼트/섹션의 계층 구조 ② `__TEXT`·`__DATA`·`__DATA_CONST`·`__LINKEDIT`이 각각 담는 것과 페이지 보호 속성 ③ `LC_LOAD_DYLIB`·`LC_RPATH`·`LC_ID_DYLIB`이 경로 해석에 쓰이는 방식(`@rpath`, `@executable_path`) ④ export trie·symtab·function starts가 담는 것과 strip이 지우는 것 ⑤ Obj-C 런타임 섹션(`__objc_classlist`, `__objc_catlist`)으로 클래스·카테고리 생존을 확인하는 법 ⑥ `otool`·`nm`·`size`·`dyld_info` 실사용
  - 맥락: 2026-08-26, 링크 방식을 실측 검증하며 파생. 도구 출력을 읽어 결론을 냈지만 **형식 자체에 대한 이해는 빌려 쓴 상태** — 6번(기반 개념) 질문에 해당
  - 연관: dyld와 앱 런치 시퀀스, 정적 링크 vs 동적 링크

- [ ] **번들과 리소스 조회 (`Bundle.main` / `Bundle(for:)` / `Bundle.module`)** — 리소스를 찾는 세 경로는 각각 무엇을 기준으로 번들을 고르는가?
  - 파볼 질문: ① `Bundle(for:)`가 "그 클래스의 코드가 담긴 이미지"를 반환한다는 정의, 그래서 링크 방식에 종속되는 이유 ② `Bundle.main`은 항상 앱 번들인가 — 앱 확장이나 테스트 번들에서는 무엇을 가리키는가 ③ SwiftPM의 `Bundle.module`이 생성되는 방식과, 위치에 독립적이게 만드는 후보 탐색 전략 ④ 정적 프레임워크의 리소스가 별도 `.bundle`로 분리되어야 하는 이유 ⑤ 전형적 실패 모드 — 테스트 타깃·프리뷰·앱 확장에서 번들을 못 찾는 상황
  - 맥락: 2026-08-26, 링크 방식을 바꾸면 `Bundle(for:)` 반환값이 달라진다는 걸 발견하며 파생. 리소스 조회 코드가 링크 방식에 묶여버리는 구조가 출발점
  - 연관: 정적 링크 vs 동적 링크, Mergeable Libraries

#### 테스트

- [ ] **유닛 테스트와 UI 테스트** — 실행되는 프로세스부터 다르다면 속도·안정성·검증 범위는 어떻게 갈리는가?
  - 파볼 질문: ① XCUITest가 앱과 별도 프로세스에서 접근성 계층을 통해 조작한다는 구조 ② 거기서 오는 flaky의 근원(타이밍·애니메이션)과 대기 전략 ③ 테스트 피라미드에서 UI 테스트를 얇게 유지하는 근거 ④ 회귀 검증을 UI 테스트로 잡을 때 시나리오 선정 기준 ⑤ CI에 물릴 때 실행 시간과 병렬화

#### 아키텍처

- [ ] **TCA (The Composable Architecture)** — State/Action/Reducer/Effect가 단방향 흐름을 어떻게 강제하는가? 그 대가는 무엇인가?
  - 파볼 질문: ① Reducer가 순수 함수여야 하는 이유와 부작용을 Effect로 밀어내는 구조 ② Store·ViewStore가 뷰 갱신 범위를 좁히는 방식 ③ 테스트가 강력해지는 근거 — 상태 전이가 값 비교로 환원됨 ④ 보일러플레이트·컴파일 시간·학습 비용이라는 대가와 도입 판단 기준 ⑤ MVVM과 갈라지는 정확한 지점
  - 연관: [[DataFlow]], 단방향 데이터 흐름 항목(Tier 7)

---

## 완료

- [x] **메서드 디스패치** → [[메서드 디스패치 (Method Dispatch)]] (2026-08-26)
  - 4단계 재인출: 라운드 1 `1/10` → 3 `6/10` → 4 `7/8` → 5 `2/2`로 게이트 통과
  - 정착한 축: **표에 자리가 있는가**(정적/동적) vs **class냐 protocol이냐**(어느 표) — 두 축을 섞던 오개념이 교정됨
