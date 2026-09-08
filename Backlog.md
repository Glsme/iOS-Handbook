# 학습 백로그

> 작업 중 만난 모르는 개념·질문을 쌓는 학습 지도. deep-dive 스킬이 관리한다.

## 미착수

### Tier 0 — 토대

- [ ] **@ViewBuilder / Result Builder**
- [ ] **Source of Truth / Derived Value**
- [ ] **AttributeGraph**
- [ ] **body의 순수성과 부작용**

### Tier 1 — 상태 관리 코어

- [ ] **@EnvironmentObject**
- [ ] **@Environment / EnvironmentValues / EnvironmentKey**
- [ ] **Observation / @Observable**
- [ ] **withObservationTracking**
- [ ] **@Bindable**
- [ ] **objectWillChange**
- [ ] **projectedValue와 `$`**
- [ ] **Binding 직접 만들기**

### Tier 3 — 렌더링과 성능

- [ ] **onChange(of:)**
- [ ] **Lazy 컨테이너와 상태 수명**
- [ ] **@State 남용과 body 재호출 비용**
- [ ] **뷰 트리의 인라인 평탄화와 AnyView의 비용**

### Tier 4 — 앱 구조와 레이아웃

- [ ] **App / Scene / WindowGroup / @main**
- [ ] **ScenePhase**
- [ ] **레이아웃 협상 3단계**
- [ ] **ViewModifier와 modifier 순서**
- [ ] **GeometryReader**
- [ ] **PreferenceKey**
- [ ] **Layout 프로토콜**
- [ ] **alignment guide**

### Tier 5 — 내비게이션·리스트·이벤트

- [ ] **NavigationStack / NavigationPath / navigationDestination**
- [ ] **sheet / fullScreenCover / alert / confirmationDialog**
- [ ] **List / ForEach와 Identifiable**
- [ ] **ScrollView / scrollPosition / scrollTargetBehavior**
- [ ] **task / onAppear / onDisappear**
- [ ] **searchable / refreshable / toolbar**
- [ ] **Transaction 전파와 `.transaction` modifier**

### Tier 7 — UIKit 연동과 아키텍처

- [ ] **UIViewRepresentable / UIViewControllerRepresentable / Coordinator**
- [ ] **UIHostingController**
- [ ] **단방향 데이터 흐름 (Unidirectional Data Flow)**

### Swift 언어 (Tier 외)

- [ ] **배타적 접근 (Law of Exclusivity, SE-0176)**
- [ ] **CoW 직접 구현**
- [ ] **래퍼 합성 `@A @B var x`**
- [ ] **`init(projectedValue:)`와 SE-0293**
- [ ] **커스텀 래퍼 직접 작성 실습**
- [ ] **제네릭 (Generics)**
- [ ] **타입 소거 (Type Erasure)**
- [ ] **Mirror / 리플렉션**
- [ ] **Never / uninhabited type**
- [ ] **memberwise initializer 합성 규칙**
- [ ] **struct/class의 메모리 비용**

---

### ▍2026-08-14 배치 — Swift/iOS 기본기

#### 언어 — 타입과 디스패치

- [ ] **`@objc dynamic`과 message dispatch**
- [ ] **vtable (Virtual Method Table)**
- [ ] **witness table**
- [ ] **Swift Intermediate Language (SIL)**
- [ ] **재귀 enum과 `indirect`**
- [ ] **타입 메타데이터 (Type Metadata)**
- [ ] **컴파일러 최적화 파이프라인**

#### 메모리 관리

- [ ] **ARC (Automatic Reference Counting)**
- [ ] **강한 참조 순환 (Strong Reference Cycle)**
- [ ] **weak / unowned**
- [ ] **힙과 스택**

#### Swift Concurrency

- [ ] **Structured Concurrency**
- [ ] **Unstructured Concurrency / Task vs Task.detached**
- [ ] **MainActor와 actor 격리**
- [ ] **프로퍼티 래퍼의 스레드 안전성**

#### Combine · 시간 기반 연산자

- [ ] **debounce / throttle**

#### 빌드·링킹

- [ ] **정적 링크 vs 동적 링크**
- [ ] **Mergeable Libraries**
- [ ] **Static Framework vs Dynamic Framework vs Mergeable Library**
- [ ] **dyld와 앱 런치 시퀀스**
- [ ] **Mach-O 구조 (세그먼트·섹션·로드 커맨드)**
- [ ] **번들과 리소스 조회 (`Bundle.main` / `Bundle(for:)` / `Bundle.module`)**

#### 테스트

- [ ] **유닛 테스트와 UI 테스트**

#### 아키텍처

- [ ] **TCA (The Composable Architecture)**
