# 학습 백로그

> 작업 중 만난 모르는 개념·질문을 쌓는 학습 지도. deep-dive 스킬이 관리한다.

## 미착수

> 맥락: 2026-08-06, SwiftUI 핵심 개념 + 상태 관리 전체 키워드 리스트업.
> Tier 0 → 7 순서가 권장 학습 순서다. Tier 0~1이 상태 관리의 본체이고, 나머지는 필요할 때 꺼내 쓴다.

### Tier 0 — 토대 (이걸 모르면 상태 관리가 이해되지 않는다)

- [ ] **불투명 반환 타입 (Opaque Return Type)** — `some View`가 제네릭·`any View`와 다른 이유는?
  - 맥락: 2026-08-08 [[View]] 딥다이브에서 `any`(existential) 자체가 처음 만난 개념으로 드러남 —
    some/any/제네릭 3자 비교부터 시작할 것. "상자 비유"(밀봉 상자 vs 안 정해진 상자)까지는 정착
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

~~### Tier 2 — 저장·특수 목적 상태~~

~~- [ ] **@AppStorage / @SceneStorage** — UserDefaults·상태 복원과 뷰를 잇는 래퍼. 언제 쓰면 안 되는가?~~
~~- [ ] **@FocusState** — 키보드 포커스를 상태로 다루기~~
~~- [ ] **@GestureState** — 제스처가 끝나면 자동으로 초기화되는 상태~~
~~- [ ] **@Namespace** — `matchedGeometryEffect`의 식별 공간~~
~~- [ ] **@ScaledMetric** — Dynamic Type에 따라 스케일되는 값~~
~~- [ ] **@FetchRequest / @Query** — Core Data / SwiftData를 뷰에 직접 잇는 래퍼~~
~~- [ ] **@UIApplicationDelegateAdaptor** — SwiftUI 앱에서 AppDelegate를 살리는 통로~~

### Tier 3 — 렌더링과 성능 (핸드북에 일부 있음)

- [ ] **onChange(of:)** — iOS 17에서 시그니처와 호출 시점이 바뀐 이유는?
- [ ] **Lazy 컨테이너와 상태 수명** — `LazyVStack`에서 화면 밖으로 나간 뷰의 [[@State]]는 어떻게 되는가?
- [ ] **@State 남용과 body 재호출 비용** — 상태를 어느 높이에 두느냐가 성능에 미치는 영향
  - 연관: [[Identity]], [[Diffing]], [[EquatableView]]

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

### Tier 6 — 애니메이션

~~- [ ] **withAnimation / .animation(_:value:)** — 암시적·명시적 애니메이션의 차이~~
~~- [ ] **Transaction** — 애니메이션 컨텍스트가 뷰 트리를 타고 흐르는 방식~~
~~- [ ] **Animatable / animatableData** — 커스텀 값을 애니메이션 가능하게 만들기~~
~~`- [ ] **matchedGeometryEffect** — 두 뷰 사이의 전환 연결`~~
~~- [ ] **transition / PhaseAnimator / KeyframeAnimator** — 등장·퇴장과 다단계 애니메이션~~

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

## 진행 중

## 완료

- [x] **View 프로토콜 / body / `some View`** → [[View]] (2026-08-08, 딥다이브 6문항 중 ②만 부분 통과 후 전면 설명 · 재인출 5라운드 25%→50%→40%→67%→100%. 약점 메모: **컴파일 타임 질문에 identity(런타임 개념)로 답하는 패턴이 3회 재발** — "두 세계" 구분(컴파일 세계: 타입·준수·포장 / 런타임 세계: identity·diff·teardown)으로 교정, @propertyWrapper 때 "언어 층에 SwiftUI로 답하기"의 변형이라 취약 지대. AnyView teardown 기준이 최장 4라운드 — "값이 다르면 도배(업데이트·@State 유지), 타입이 다르면 철거(teardown·@State 소멸)"로 정착. Never "무한 호출" 오개념은 완전 교정("값 0개 → 반환 불가 → 즉시 크래시"). any(existential)는 처음 만난 개념 — 불투명 반환 타입 항목에 맥락 승계. 근소 통과로 남은 것: AnyView 설계의 "얻는 것" 쪽, _makeView 이후 픽셀 경로)
- [x] **@StateObject / @StateObject vs @ObservedObject** → [[@StateObject]] (2026-08-08, 딥다이브 6문항 **전부 미인출** 후 3단계 전면 설명 · 재인출 1라운드 71%(10/14)에서 본인 요청으로 중단. 인출 성공: 저장 위치 차이(struct 안 vs identity 저장소), "만드는 방법만 클로저로 전달", 객체가 죽는 시점(identity 종료), 외부 주입 함정, 기반 개념 Identity·DynamicProperty. 미통과 4건 — ① iOS 17 대체 경로를 @Query로 오답 ② init 인자 주입의 비대칭 함정("init은 매번 실행되나 결과는 첫 번째만 채택") ③ "충돌"의 정체가 두 계약의 모순이라는 점 ④ private이 막는 것이 memberwise initializer라는 점. ①은 Observation 항목에, ③④는 신규 "memberwise initializer 합성 규칙" 항목으로 승격)
- [x] **@propertyWrapper (SE-0258)** → [[@propertyWrapper]] (2026-08-07, 딥다이브 6문항 완주 · 재인출 4라운드 73%→67%→67%→100%. 약점 메모: 여섯 문항 중 다섯에서 언어 층 질문에 SwiftUI로 답하는 패턴이 나와 백로그의 진단이 그대로 확증됨 — "컴파일은 언어 층, 실행은 프레임워크 층"이 4라운드에야 정착. 매크로와의 경계(프로퍼티 한 칸 vs 타입 전체)는 1라운드 완전 미답 후 회복, plain class 사례는 마지막 라운드에야 인출. 코드 작성 문항은 본인 요청으로 스킵 — 신규 백로그 "커스텀 래퍼 직접 작성 실습" 참고)
- [x] **DynamicProperty** → [[DynamicProperty]] (2026-08-07, 딥다이브 6문항 완주 · 재인출 5라운드 10%→22%→43%→50%→100%. 약점 메모: update() 방향 오개념("값 변경 시 호출")을 잡는 데 2라운드, 커스텀 래퍼 "State를 품는다"는 5라운드에야 정착 — State의 나머지 반쪽(쓰기→dirty→갱신)은 끝까지 미인출, 기반 4개는 목록 암기 대신 인과 사슬("문제→어디→어떻게→누가")로 정착, 세션 중 wrappedValue·리플렉션 기초 질문이 등장해 언어 층이 약함이 드러남 — 노트 "더 파볼 질문"과 신규 백로그 @propertyWrapper·Mirror 참고)
- [x] **뷰 업데이트 사이클** → [[뷰 업데이트 사이클 (View Update Cycle)]] (2026-08-07, 딥다이브 6문항 완주 · 재인출 4라운드 33%→50%→0%→100%. 약점 메모: "몰아서 갱신"의 주사율 상한 이득 재인출 누락, 위반 4종 정착에 4라운드(특히 백그라운드 위반 = 상태 변경 자체라는 정의), 트랜잭션은 본인 질문으로 뚫림. React·Flutter 비교는 본인 선택으로 범위 제외 — 노트 "더 파볼 질문" 참고)
- [x] **View는 값 타입(struct)이다** → [[값으로서의 View (View as Value)]] (2026-08-07, 딥다이브 6문항 완주 · 재인출 4라운드 37.5%→60%→50%→100%. 약점 메모: identity 리셋 케이스 중 ForEach id 소멸, ".task는 body 재호출에 재시작 안 됨" 명시, @StateObject의 해결 메커니즘, 언어 강제력 vs 관례 — 노트 "더 파볼 질문" 참고)
- [x] **값 의미론 (Value Semantics)** → [[값 의미론 (Value Semantics)]] (2026-08-06, 딥다이브 6문항 완주 · 재인출 4라운드 62.5%→66.7%→부분 통과→100%. 약점 메모: Obj-C 방어적 복사 사고 시나리오 재현, "관찰 가능한"이라는 수식어 — 노트의 "더 파볼 질문" 참고)
- [x] **선언형 UI (Declarative UI)** → [[선언형 UI (Declarative UI)]] (2026-08-06, 딥다이브 6문항 완주 · 같은 날 재인출 재검증 3라운드 통과 50%→80%→100%)
- [x] **DataFlow** → [[DataFlow]] (2026-08-05)
- [x] **@State** → [[@State]] (2026-05-14)
- [x] **@Binding** → [[@Binding]] (2026-07-15)
- [x] **@ObservedObject** → [[@ObservedObject]] (2026-08-05)
- [x] **ObservableObject** → [[ObservableObject]] (2026-08-05)
- [x] **@Published** → [[@Published]] (2026-08-05)
- [x] **Identity** → [[Identity]] (2026-08-05)
- [x] **Diffing** → [[Diffing]] (2026-08-05)
- [x] **EquatableView** → [[EquatableView]] (2026-08-05)
