# 학습 백로그

> 작업 중 만난 모르는 개념·질문을 쌓는 학습 지도. deep-dive 스킬이 관리한다.

## 미착수

> 맥락: 2026-08-06, SwiftUI 핵심 개념 + 상태 관리 전체 키워드 리스트업.
> Tier 0 → 7 순서가 권장 학습 순서다. Tier 0~1이 상태 관리의 본체이고, 나머지는 필요할 때 꺼내 쓴다.

### Tier 0 — 토대 (이걸 모르면 상태 관리가 이해되지 않는다)

- [ ] **View 프로토콜 / body / `some View`** — body는 언제, 누가, 몇 번 호출하는가?
- [ ] **불투명 반환 타입 (Opaque Return Type)** — `some View`가 제네릭·`any View`와 다른 이유는?
- [ ] **@ViewBuilder / Result Builder** — 중괄호 안에 나열한 뷰들이 어떻게 하나의 타입으로 합쳐지는가?
- [ ] **View는 값 타입(struct)이다** — 뷰 값과 실제 렌더링 트리는 어떻게 분리되는가? 뷰가 매번 새로 만들어져도 상태가 유지되는 이유는?
- [ ] **뷰 업데이트 사이클** — 상태 변경 → 무효화 → body 재호출 → diff → 렌더의 전체 흐름은?
  - 연관: [[Diffing]]
- [ ] **DynamicProperty** — 모든 상태 래퍼가 채택하는 이 프로토콜이 하는 일은? 래퍼가 SwiftUI 업데이트에 참여하는 통로
- [ ] **Source of Truth / Derived Value** — 어떤 데이터를 상태로 두고 어떤 걸 계산으로 둘 것인가?
  - 연관: [[DataFlow]]
- [ ] **AttributeGraph** — SwiftUI 내부의 의존성 그래프. 비공개 구현이라 어디까지가 확인된 사실인가?
- [ ] **body의 순수성과 부작용** — 호출 시점·횟수가 보장되지 않는다는 계약을 어기면 실제로 어떤 버그로 나타나는가? 부작용은 어디에 둬야 하는가(`.task`, `.onChange(of:)`)?
  - 맥락: 2026-08-06, 선언형 UI 딥다이브 중 등장
  - 연관: [[선언형 UI (Declarative UI)]]

### Tier 1 — 상태 관리 코어

- [ ] **@StateObject** — [[@ObservedObject]]와 무엇이 다르고, 왜 iOS 14에서 따로 추가됐나? (핸드북 링크 있으나 노트 없음)
  - 연관: [[@ObservedObject]], [[DataFlow]]
- [ ] **@StateObject vs @ObservedObject** — 소유와 전달의 구분. `@ObservedObject`에 객체를 직접 생성하면 왜 상태가 날아가는가?
- [ ] **@EnvironmentObject** — 뷰 계층을 건너뛴 주입. 주입을 잊으면 왜 런타임 크래시인가? (링크 있으나 노트 없음)
- [ ] **@Environment / EnvironmentValues / EnvironmentKey** — 환경 값은 어떻게 아래로 전파되는가? 커스텀 키 정의와 iOS 18 `@Entry` 매크로 (링크 있으나 노트 없음)
- [ ] **Observation / @Observable** — iOS 17의 새 관찰 방식. [[ObservableObject]]의 객체 단위 무효화 문제를 어떻게 프로퍼티 단위로 바꿨나? (링크 있으나 노트 없음)
  - 연관: [[ObservableObject]], [[@Published]], [[Diffing]]
- [ ] **withObservationTracking** — `@Observable` 매크로가 실제로 만들어내는 추적 메커니즘은?
- [ ] **@Bindable** — `@Observable` 객체에 양방향 바인딩을 만드는 래퍼. [[@Binding]]과 어떻게 다른가?
- [ ] **objectWillChange** — [[ObservableObject]]의 진짜 갱신 신호. willChange인 이유는?
- [ ] **projectedValue와 `$`** — `$count`가 실제로 반환하는 것은 무엇인가?
- [ ] **Binding 직접 만들기** — `Binding(get:set:)`, `.constant(_:)`를 언제 쓰는가?

### Tier 2 — 저장·특수 목적 상태

- [ ] **@AppStorage / @SceneStorage** — UserDefaults·상태 복원과 뷰를 잇는 래퍼. 언제 쓰면 안 되는가?
- [ ] **@FocusState** — 키보드 포커스를 상태로 다루기
- [ ] **@GestureState** — 제스처가 끝나면 자동으로 초기화되는 상태
- [ ] **@Namespace** — `matchedGeometryEffect`의 식별 공간
- [ ] **@ScaledMetric** — Dynamic Type에 따라 스케일되는 값
- [ ] **@FetchRequest / @Query** — Core Data / SwiftData를 뷰에 직접 잇는 래퍼
- [ ] **@UIApplicationDelegateAdaptor** — SwiftUI 앱에서 AppDelegate를 살리는 통로

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
- [ ] **List / ForEach와 Identifiable** — 잘못된 id가 어떤 버그로 나타나는가?
  - 연관: [[Identifiable]], [[Identity]]
- [ ] **ScrollView / scrollPosition / scrollTargetBehavior** — 스크롤 상태 제어
- [ ] **task / onAppear / onDisappear** — 뷰 생명주기 훅과 비동기 작업 취소
- [ ] **searchable / refreshable / toolbar** — 시스템 제공 인터랙션 modifier

### Tier 6 — 애니메이션

- [ ] **withAnimation / .animation(_:value:)** — 암시적·명시적 애니메이션의 차이
- [ ] **Transaction** — 애니메이션 컨텍스트가 뷰 트리를 타고 흐르는 방식
- [ ] **Animatable / animatableData** — 커스텀 값을 애니메이션 가능하게 만들기
- [ ] **matchedGeometryEffect** — 두 뷰 사이의 전환 연결
- [ ] **transition / PhaseAnimator / KeyframeAnimator** — 등장·퇴장과 다단계 애니메이션

### Tier 7 — UIKit 연동과 아키텍처

- [ ] **UIViewRepresentable / UIViewControllerRepresentable / Coordinator** — UIKit 뷰를 SwiftUI에 얹기. `updateUIView` 호출 시점은?
- [ ] **UIHostingController** — SwiftUI를 UIKit에 얹기. 데이터 흐름 표준이 없는 영역
  - 연관: [[DataFlow]]
- [ ] **단방향 데이터 흐름 (Unidirectional Data Flow)** — MVVM·MV·TCA가 나뉘는 지점
  - 연관: [[DataFlow]]

### Swift 언어 (Tier 외)

- [ ] **배타적 접근 (Law of Exclusivity, SE-0176)** — "값을 바꾸는 경로는 그 변수뿐"을 컴파일 타임/런타임에서 각각 무엇으로 어떻게 강제하는가?
  - 맥락: 2026-08-06, 값 의미론 딥다이브에서 기반 개념(언어 차원의 받침)으로 등장
  - 연관: [[값 의미론 (Value Semantics)]]
- [ ] **CoW 직접 구현** — `isKnownUniquelyReferenced`로 내 타입에 값 의미론 + 지연 복사를 어떻게 구현하는가? "복사 조건 = 변경 × 공유"를 코드로 체화하기
  - 맥락: 2026-08-06, 값 의미론 딥다이브에서 표준 라이브러리의 CoW 메커니즘을 배우고 남은 실습 과제
  - 연관: [[값 의미론 (Value Semantics)]]

## 진행 중

## 완료

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
