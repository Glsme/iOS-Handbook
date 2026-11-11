# @EnvironmentObject
상위 뷰가 `.environmentObject(_:)`로 넣은 [[ObservableObject]]를 하위 뷰가 **타입만 선언해서** 꺼내 쓰는 Property Wrapper.
**environment를 통한 전달** + **[[@ObservedObject]]식 구독**의 합성이다.

```swift
@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *)
@frozen @propertyWrapper public struct EnvironmentObject<ObjectType>: DynamicProperty
	where ObjectType: ObservableObject {

	public init()                                   // 초기값 없음

	public var wrappedValue: ObjectType { get }     // get-only
	public var projectedValue: EnvironmentObject<ObjectType>.Wrapper { get }
}
```

- `init()`에 인자 없음 → 값은 호출자가 아니라 **environment가 채움**
- `where ObjectType: ObservableObject` → 변경 통지 수단을 컴파일 타임에 요구

### 왜 필요한가

없으면 **prop drilling**이 생긴다.

- 객체를 생성하는 뷰부터 실제로 사용하는 뷰까지, **뷰 계층마다 계속 선언하면서 들고 있어야 함**
- 중간 뷰는 쓰지도 않는 객체를 받아서 넘기기만 함
- 계층이 바뀌면 중간 뷰의 `init`을 전부 수정해야 함

```
[@ObservedObject로 명시 전달]          [@EnvironmentObject]

Root     settings 생성                 Root     settings 생성 · 주입
 │  settings 전달                       │
Middle1  받아서 넘기기만                Middle1  (선언 없음)
 │  settings 전달                       │
Middle2  받아서 넘기기만                Middle2  (선언 없음)
 │  settings 전달                       │
Leaf     실제 사용                      Leaf     타입만 선언해 사용
```

### 내부 동작 — 주입 · 조회 · 구독

```
App
 │  @StateObject settings ──────────────► [ Settings 객체 ]  (힙에 1개)
 │                                              ▲    │
 ▼                                              │    │
RootView  .environmentObject(settings)          │    │
 │   ① 주입: 서브트리의 environment에               │    │
 │      { Settings 타입 → 객체 참조 } 등록 ──────────┘   │
 ▼                                                   │
MiddleView        (선언 없음, 통과)                     │
 ▼                                                   │
DeepChildView                                        │
     @EnvironmentObject var settings: Settings       │
     ② 조회: body 평가 직전, Settings 타입으로 참조 획득    │
     ③ 구독: 객체의 objectWillChange 구독 ◄─────────────┘
```

**① 주입**
- environment에 들어가는 것은 **객체 참조 하나** (프로퍼티가 따로 들어가지 않음)
- 키는 **객체의 타입** → 하위 뷰가 이름 없이 타입만으로 조회 가능
- 적용 범위는 **주입한 뷰와 그 자손**까지. 전역 저장소가 아님

**② 조회**
- 시점은 뷰가 트리에 설치되어 `body`가 평가될 때
- 뷰의 `init`에서는 접근 불가

**③ 구독과 갱신** — 여기부터는 [[@ObservedObject]]와 동일

```
settings.fontSize = 20
      │
      ▼
@Published가 objectWillChange.send()     (바뀌기 직전)
      │
      ▼
그 타입을 선언해 구독 중인 뷰만 무효화
      │
      ▼
다음 업데이트 사이클에 body 재호출
      │
      ▼
이전 결과와 비교해 달라진 부분만 화면 반영
```

- 무효화 → `body` 재호출 → 비교의 흐름은 [[뷰 업데이트 사이클 (View Update Cycle)]] · [[Diffing]] 참고

**같은 타입을 두 번 주입하면 — 가장 가까운 조상이 이긴다**

```
RootView            ← settingsA 주입
 └ MiddleView
    ├ LeftView      ← settingsB 주입
    │  └ LeafL      → settingsB
    └ RightView
       └ LeafR      → settingsA   (옆 서브트리의 B는 보이지 않음)
```

- 변수 가리기(shadowing)와 같은 구조
- "타입이 키"여도 나중에 넣은 값이 전체를 덮어쓰지 않음
- [[싱글톤 패턴 (Singleton Pattern)]]은 객체가 하나라서 이런 서브트리별 교체가 불가능

> 타입을 키로 `EnvironmentValues` 안에 저장하는 구체적인 방식은 비공개 구현이다.
> 공개 문서로 확인되는 선은 "타입으로 조회한다", "조상에서 주입해야 한다"까지다.

### 기반 개념 — 역할 분담

한 번의 갱신이 성립하려면 네 가지가 각각 해결되어야 한다.

| 풀어야 할 일 | 담당 | 역할 |
| --- | --- | --- |
| 객체를 **어디서** 찾나 | Environment ([[@Environment]]) | 뷰 트리를 따라 내려오는 전달 통로 |
| **언제** 바뀌는지 어떻게 아나 | [[ObservableObject]]의 `objectWillChange` | 바뀌기 직전에 보내는 신호 |
| 둘을 `body` 평가에 **이어 주는** 것 | [[DynamicProperty]] | `body` 전에 래퍼를 채우고 구독을 거는 연결 고리 |
| 객체를 **유지**하는 것 | [[@StateObject]] | 소유와 수명 |

- 문법 층은 [[@propertyWrapper]] (`@` 표기, `wrappedValue`, `$`)
- `@EnvironmentObject` 자신은 **전달과 구독만** 담당. 소유하지 않음

**수명과 통지는 다른 문제다**

| 문제 | 증상 | 담당 |
| --- | --- | --- |
| 수명 | 객체가 새로 만들어져 값이 초기화됨 | [[@StateObject]] |
| 통지 | 값은 바뀌었는데 화면이 옛값 그대로 | [[ObservableObject]] |

**`ObservableObject`를 요구하는 이유** — 평범한 클래스가 들어갈 수 있었다면:

```swift
class Counter { var count = 0 }        // objectWillChange 없음

Text("\(counter.count)")
Button("+") { counter.count += 1 }     // 3번 탭
```

- 객체 안의 `count`: **3** (객체는 멀쩡히 살아 있음)
- 화면: **0** (`objectWillChange`가 없어 `body`가 재호출되지 않음)
- `struct`는 참조가 아니라서 여러 뷰가 같은 하나를 공유할 수 없음

### 대안과 트레이드오프

비교 대상: **prop drilling을 푸는 방법들** / 판단 기준: 의존성의 명시성, 누락 시 발견 시점, 갱신 범위

| 방법 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| `init`으로 명시 전달 ([[@ObservedObject]]) | 의존성이 시그니처에 드러남, 누락 시 **컴파일 에러** | prop drilling 그대로 |
| [[싱글톤 패턴 (Singleton Pattern)]] | 계층에 넣지 않고 바로 사용 | 테스터블하지 않음, 관리 어려움, 서브트리별 교체 불가 |
| **@EnvironmentObject** | 중간 계층이 깔끔함, 서브트리별 교체 가능 | 누락 시 **런타임 크래시**, 의존성이 숨음 |
| `@Observable` + `@Environment(Type.self)` (iOS 17+) | **읽은 프로퍼티 단위** 갱신, 옵셔널 조회 가능 | iOS 17 이상 필요, 마이그레이션 |

### 언제 쓰면 안 되는가

**① 주입 누락 = 런타임 크래시**
- 조상에 `.environmentObject`가 없으면 `body`에서 접근하는 순간 fatal error
	- `No ObservableObject of type Settings found. A View.environmentObject(_:) for Settings may be missing as an ancestor of this view.`
- 컴파일러가 못 잡는 이유: 뷰 트리(누가 조상인지)는 **런타임에** 정해짐
- 프리뷰·테스트에서도 매번 주입 필요

**② 객체 단위 무효화**

```
Settings { @Published isDarkMode, @Published fontSize }

ViewA: 선언 O, body에서 isDarkMode만 읽음
ViewB: 선언 O, body에서 fontSize만 읽음
ViewC: 선언 X

fontSize 변경 → ViewA 재호출 (읽지 않는데도)
              → ViewB 재호출
              → ViewC 영향 없음
```

- `objectWillChange`는 "무언가 바뀐다"만 알림. 어느 프로퍼티인지는 모름
- 큰 전역 객체 하나를 여러 뷰가 물고 있으면 불필요한 `body` 재호출 증가
- `@Observable`이면 위 경우 `ViewB`만 무효화

**③ environment가 이어지지 않는 경계**
- 별도의 [[UIHostingController]], 새 윈도우처럼 뷰 트리의 루트가 따로 생기는 곳
- 해당 루트에서 다시 주입 필요

**④ 타입당 하나**
- 같은 타입의 객체 두 개를 한 서브트리에서 동시에 꺼낼 수 없음

**⑤ 뷰 밖에서는 사용 불가**
- 뷰 모델 같은 일반 객체, 뷰의 `init`

**⑥ 재사용 컴포넌트**
- 의존성이 `init`에 드러나지 않아 쓰는 쪽이 무엇을 주입해야 하는지 알기 어려움
- 범용 컴포넌트, 한두 단계만 내려보내는 경우에는 명시 전달이 나음

### 실전 적용

**기본형 — 소유는 최상위, 주입은 한 번, 중간 뷰는 할 일 없음**

```swift
final class Settings: ObservableObject {
	@Published var isDarkMode = false
}

@main
struct MyApp: App {
	@StateObject private var settings = Settings()      // ① 생성 · 소유

	var body: some Scene {
		WindowGroup {
			RootView()
				.environmentObject(settings)            // ② 주입
		}
	}
}

struct DeepChildView: View {
	@EnvironmentObject var settings: Settings           // ③ 타입만 선언

	var body: some View {
		Toggle("다크 모드", isOn: $settings.isDarkMode)  // ④ $로 Binding 파생
	}
}
```

- 소유를 `@ObservedObject`로 두면 뷰 구조체가 재생성될 때 객체도 다시 만들어짐 → **소유는 [[@StateObject]]**
- `$settings.isDarkMode`는 `projectedValue`(`Wrapper`)에서 [[@Binding]]을 꺼내는 문법 ([[@dynamicMemberLookup]])

**프리뷰 — 직접 주입**

```swift
#Preview {
	DeepChildView()
		.environmentObject(Settings())
}
```

**iOS 17 이후 대응**

```swift
// iOS 13~16                                  // iOS 17+
.environmentObject(settings)              →   .environment(settings)
@EnvironmentObject var settings: Settings →   @Environment(Settings.self) private var settings
```

### 더 파볼 질문
- environment는 뷰 트리를 따라 실제로 어떻게 전파되는가 (`EnvironmentValues` · `EnvironmentKey`)
- `sheet` · `fullScreenCover`로 띄운 뷰는 어디까지 environment를 물려받는가
- 선언만 하고 `body`에서 한 번도 읽지 않는 뷰도 무효화되는가 (실측 미확인)

### 관련
- [[@ObservedObject]] — 구독 메커니즘이 같은 쪽. 전달 경로만 다름
- [[@StateObject]] — 주입할 객체를 소유하는 쪽
- [[@Environment]] — 같은 전달 통로를 쓰는 값 조회 래퍼
- [[ObservableObject]] · [[@Published]] — 변경 통지
- [[DynamicProperty]] · [[@propertyWrapper]] · [[DataFlow]]

### 출처
- Apple Developer Documentation — [EnvironmentObject](https://developer.apple.com/documentation/swiftui/environmentobject)
- Apple Developer Documentation — [environmentObject(_:)](https://developer.apple.com/documentation/swiftui/view/environmentobject(_:))
- Apple Developer Documentation — [Migrating from the Observable Object protocol to the Observable macro](https://developer.apple.com/documentation/swiftui/migrating-from-the-observable-object-protocol-to-the-observable-macro)
- WWDC19 "Data Flow Through SwiftUI" (Session 226)
- WWDC21 "Demystify SwiftUI" (Session 10022)
- WWDC23 "Discover Observation in SwiftUI" (Session 10149)
