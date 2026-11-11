# @Environment
뷰의 environment에 들어 있는 **값 하나를 키 패스로 읽어 오는** Property Wrapper.
[[@EnvironmentObject]]가 쓰는 전달 통로의 본체이고, 읽기 전용이다.

```swift
@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *)
@frozen @propertyWrapper public struct Environment<Value>: DynamicProperty {

	public init(_ keyPath: KeyPath<EnvironmentValues, Value>)   // 값 조회
	public init(_ objectType: Value.Type)                       // iOS 17+, @Observable 객체 조회

	public var wrappedValue: Value { get }                      // get-only
}
```

**세 이름의 관계**

| 이름 | 무엇 | 사물함 비유 |
| --- | --- | --- |
| `EnvironmentValues` | environment에 담긴 값들의 묶음 (구조체) | 사물함 전체 |
| `EnvironmentKey` | 묶음에 새 항목을 추가하는 프로토콜. **타입이 곧 키** | 칸 번호 + 비어 있을 때의 기본 물건 |
| `@Environment` | 뷰에서 항목 하나를 읽는 래퍼 | 칸을 열어 보는 사람 |

### 왜 필요한가

**상위 뷰에서 미리 값을 주입해 놓고, prop drilling 없이 하위 뷰에서 사용**하기 위함.

- 다크 모드 여부, 로케일, 글자 크기처럼 화면 곳곳에서 쓰는 값
- 없으면 모든 중간 뷰가 `init`으로 받아서 넘겨야 함
- 시스템이 넣어 주는 값(`colorScheme`, `locale`, `dismiss` 등)을 받는 유일한 통로

`.font(_:)`, `.disabled(_:)`, `.lineLimit(_:)` 같은 modifier도 내부적으로는 environment에 값을 쓰는 것이다.
`VStack`에 `.font`를 붙이면 안의 `Text`가 전부 따라오는 이유다.

### 내부 동작 — 전달 · 조회 · 갱신

```
RootView    environment = { colorScheme: light, themeAccent: blue(기본값), ... }
 │
 │  .environment(\.themeAccent, .orange)
 │   ① 물려받은 EnvironmentValues를 복사, themeAccent만 바꿔 자손에게 전달
 ▼
MiddleView  environment = { colorScheme: light, themeAccent: orange, ... }
 │           (값은 갖고 있지만 읽지 않음, 통과)
 ▼
LeafView    @Environment(\.themeAccent) private var themeAccent
             ② body 평가 직전, 자기 위치의 EnvironmentValues에서 키 패스로 조회
             ③ SwiftUI가 "LeafView는 themeAccent에 의존"을 기록
```

**① 전달**
- `EnvironmentValues`는 구조체 ([[값 의미론 (Value Semantics)]])
- 하위 뷰들에게 **값을 복사하면서** 내려감
- 적용 범위는 **modifier를 붙인 뷰와 그 자손**. 형제·부모에는 닿지 않음

**② 조회 — 가장 가까운 조상, 없으면 기본값**

```
RootView                 (주입 없음)
 └ ContainerView          → blue   (defaultValue)
    ├ CardA   .environment(\.themeAccent, .orange)
    │  └ LabelA           → orange
    └ CardB
       └ LabelB           → blue   (옆 서브트리의 orange는 보이지 않음)
```

**③ 갱신 — 값 비교 기반**

environment는 **보이지 않는 `init` 파라미터**다. 갱신 경로가 `init` 전달과 같다.

```
language = "en"                      (루트의 @State)
   │   @State가 SwiftUI에 변경을 알림            ← 처음 알아채는 것
   ▼
Root 무효화 → Root.body 재호출
   │   .environment(\.appLanguage, "en") 가 새 값으로 다시 실행
   ▼
내려가는 값이 "ko" → "en" 으로 달라짐            ← 판단 근거 (값 비교)
   ▼
\.appLanguage 를 선언한 뷰만 무효화 → body 재호출   (중간 뷰는 건너뜀)
```

**실측 — 어떤 뷰의 `body`가 재호출되는가** (macOS 27, 커스텀 키 `direction` · `other`)

```
Root            .environment(\.direction, …) .environment(\.other, …)
 └ Middle       선언 없음
    ├ DeclaredOnly   @Environment(\.direction) 선언만, body에서 읽지 않음
    ├ Reader         @Environment(\.direction) 선언 + body에서 읽음
    └ NoDecl         선언 없음
```

| 변경 | 재호출된 `body` |
| --- | --- |
| `direction` 0 → 1 | Root, **DeclaredOnly**, Reader |
| `other` 0 → 1 | Root |
| `direction` 1 → 1 (같은 값) | Root |

- **선언만 해도 재호출됨**. 기준은 "`body`에서 읽었는가"가 아니라 "그 키를 선언했는가"
- 다른 키가 바뀌거나 같은 값이 다시 들어오면 재호출 없음 → 키 단위 + 값 비교
- `Middle`은 `Root.body`가 다시 돌아도 재호출되지 않음
	- `body`를 다시 부르는 조건은 **저장 프로퍼티(입력)가 달라졌거나, 선언한 동적 프로퍼티의 값이 달라졌을 때**
	- `Middle`은 둘 다 해당 없음. environment는 뷰 구조체 안이 아니라 SwiftUI가 관리하는 바깥에 있음

- 값의 **원본은 보통 상위 뷰의 [[@State]]**. environment는 그 결과를 아래로 나를 뿐
- `objectWillChange`는 관여하지 않음. `String` · `Color` 같은 값에는 그런 신호가 없음
- 원본을 가진 뷰의 **`body`가 다시 돌아야** 새 값이 통로에 실림

**[[@EnvironmentObject]]와의 차이**

| | 변경을 알리는 것 | 무효화 범위 |
| --- | --- | --- |
| `@EnvironmentObject` | 객체의 `objectWillChange` ([[ObservableObject]]) | 그 객체를 선언한 뷰 전부 |
| `@Environment(\.key)` | 원본 `@State` 변경 → 상위 `body` 재평가 → 값 비교 | 그 키를 선언한 뷰만 |

> `EnvironmentValues`가 내부에서 복사와 상속을 처리하는 자료구조, 키별 의존성을 기록하는
> 방식은 비공개 구현이다. 공개 문서로 확인되는 선은 "값이 바뀌면 그 값에 의존하는 뷰를
> 갱신한다"까지다.

### 기반 개념 — 역할 분담

| 풀어야 할 일 | 담당 | 역할 |
| --- | --- | --- |
| **어떤 값**을 읽을지 지정 | [[KeyPath]] | `\.themeAccent`처럼 프로퍼티를 타입 안전하게 가리킴 |
| 값의 **자리와 기본값** | `EnvironmentKey` | 타입을 키로 삼고 `defaultValue`를 보장 |
| 값을 **위에서 아래로** 전달 | `EnvironmentValues` | 부모 것을 물려받고, 바꾼 사본을 자손에게 넘김 |
| `body` 평가와 **이어 주기** | [[DynamicProperty]] | `body` 전에 값을 채우고 의존성을 등록 |
| **변경 감지** | 원본을 가진 [[@State]] (또는 시스템) | 원본이 바뀌면 그 뷰의 `body`를 다시 돌림 |

- 문법 층은 [[@propertyWrapper]]
- 키 패스는 KVO가 아님. 관찰 기능 없이 "어떤 프로퍼티인지"만 가리키는 값

**SwiftUI는 그 뷰가 `@Environment`를 쓰는지 어떻게 아는가**

`@Environment(\.direction)`은 뷰 구조체 안의 **저장 프로퍼티**다. 키 패스를 품고 있다.

```
① 뷰가 트리에 처음 설치됨 → SwiftUI가 그 뷰의 노드를 만듦   (런타임)
      ▼
② 뷰 구조체의 저장 프로퍼티를 훑어 DynamicProperty 준수 필드를 찾음
      ▼
③ Environment 필드 발견 → 안에 든 키 패스 \.direction 을 꺼냄
      ▼
④ 노드에 "environment의 \.direction 에 의존"을 등록
      ▼
⑤ 이후 body를 부를 때마다, 직전에 최신 값을 필드에 채워 넣음
```

- 판단 시점은 **노드를 만들 때**. `body` 안의 코드를 들여다보지 않음 → 선언만 해도 재호출되는 이유
- 선언이 없는 중간 뷰는 ④에서 등록되는 것이 없음
- 뷰의 저장 프로퍼티로만 쓸 수 있는 이유: SwiftUI가 훑는 대상이 뷰 구조체의 필드
- `init`에서 읽을 수 없는 이유: `init`은 ①보다 앞

**노드는 언제 만들어지는가 — 컴파일 시점이 아니라 런타임**

| 시점 | 정해지는 것 |
| --- | --- |
| 컴파일 | 뷰의 **타입 구조** (어떤 뷰 타입이 어떤 필드를 갖는지) |
| 런타임, 뷰가 처음 화면 계층에 올라올 때 | 그 뷰의 **노드** 생성, 의존 등록 |
| 런타임, identity가 끝날 때 | 노드 폐기 (`if/else` 전환, `.id()` 변경, 트리에서 제거) |

- 노드의 수명은 [[Identity]]의 수명과 같음. [[@State]] · [[@StateObject]]의 저장소가 사는 곳도 여기
- `body`가 다시 호출될 때마다 새로 만들어지는 것은 뷰 **값**(구조체)이지 노드가 아님 ([[값으로서의 View (View as Value)]])

> "노드"는 동작을 설명하기 위한 모델이다. 실제 내부 구조(AttributeGraph)와, 필드를 훑어
> `DynamicProperty`를 찾는 구체적인 방식은 비공개 구현이다. 공개 문서로 확인되는 선은
> "`DynamicProperty`는 `body`를 다시 계산하기 전에 SwiftUI가 값을 갱신하는 저장 프로퍼티"까지다.

### 대안과 트레이드오프

비교 대상: **상위에서 정한 값을 prop drilling 없이 하위에서 쓰는 방법** / 판단 기준: 범위 분리, 자동 갱신, 누락 시 발견 시점

| 방법 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| `init`으로 명시 전달 | 의존성이 드러남, 누락 시 **컴파일 에러** | prop drilling |
| [[싱글톤 패턴 (Singleton Pattern)]] | 사용하려는 뷰에서 바로 사용, 뷰 밖에서도 접근 | 객체가 하나라 **뷰마다 다른 값 불가**, 테스터블하지 않음, 바뀌어도 뷰가 자동 갱신되지 않음 |
| **`@Environment(\.key)`** | 서브트리별 교체, 키 단위 갱신, 기본값 덕에 크래시 없음 | 조용한 실패, 읽기 전용, 뷰 안에서만 사용 |
| [[@EnvironmentObject]] | 하위에서 객체를 통해 값 변경 가능 | 누락 시 **런타임 크래시**, 객체 단위 무효화 |
| `@Environment(Type.self)` (iOS 17+) | 객체 공유 + 읽은 프로퍼티 단위 갱신 | iOS 17 이상, 비옵셔널이면 누락 시 크래시 |

**어느 쪽으로 넘길까**

| 값의 성격 | 선택 | 이유 |
| --- | --- | --- |
| 국소적으로 쓰는 필수 입력 (버튼 제목 등) | `init` | 빠뜨리면 컴파일러가 잡아 줌 |
| 앱 전체 여러 깊이에서 쓰는 값 (테마 색 등) | environment | 뷰 생성자마다 정의할 필요 없음, 일부 화면만 교체 가능 |

### 언제 쓰면 안 되는가

**① 주입 누락 = 조용한 실패**

| 선언 | 상위에서 안 넣었을 때 |
| --- | --- |
| `@EnvironmentObject var s: Settings` | 런타임 크래시 |
| `@Environment(\.someKey) var v` | **`defaultValue`가 조용히 들어옴** |
| `@Environment(Settings.self) var s` | 런타임 크래시 |
| `@Environment(Settings?.self) var s` | `nil` |

- 키 패스 방식은 크래시가 나지 않음. 프리뷰도 주입 없이 그냥 뜸
- 기본값으로 멀쩡히 동작하므로 **크래시보다 발견이 늦는 버그**가 될 수 있음
- 반드시 주입되어야 하는 값이면 `init` 전달이 안전

**② 읽기 전용 — "바꾼다"의 세 가지**

| 하려는 일 | 방법 | 결과 |
| --- | --- | --- |
| 읽은 값에 대입 | `baseFontSize += 1` | **컴파일 에러** (`wrappedValue`가 getter only) |
| 내 자손에게만 다른 값 | 자식에 `.environment(\.baseFontSize, baseFontSize + 1)` | 그 서브트리만 바뀜 |
| 전체 값 변경 | 루트의 원본 `@State`를 변경 | 그 키를 선언한 모든 뷰 갱신 |

- 하위에서 전체 값을 바꾸려면 **변경 수단을 같이 주입**해야 함 (액션 클로저, [[@Binding]], 관찰 가능한 객체)

**③ 뷰 밖에서는 사용 불가**
- 뷰 모델 같은 일반 객체, 뷰의 `init`
- `body`가 평가될 때에만 유효

**④ 평범한 클래스를 값으로 넣은 경우**
- 참조가 그대로라 내부 프로퍼티가 바뀌어도 SwiftUI가 알지 못함
- 객체를 공유하려면 [[@EnvironmentObject]] 또는 `@Observable`

**⑤ 숨은 의존성**
- `init`에 드러나지 않아 쓰는 쪽이 무엇을 주입해야 하는지 알기 어려움
- 범용 컴포넌트의 필수 입력, 한두 단계만 내려보낼 값에는 부적합

### 실전 적용

**커스텀 키 — 선언 3종**

```swift
// (A) 키 타입: 인스턴스를 만들지 않음. defaultValue 필수
private struct ThemeAccentKey: EnvironmentKey {
	static let defaultValue: Color = .blue
}

// (B) 연산 프로퍼티: 이것이 있어야 \.themeAccent 를 쓸 수 있음
extension EnvironmentValues {
	var themeAccent: Color {
		get { self[ThemeAccentKey.self] }
		set { self[ThemeAccentKey.self] = newValue }
	}
}

// 상위에서 주입
RootView()
	.environment(\.themeAccent, .orange)

// (C) 하위에서 키 패스로 읽기
struct LeafView: View {
	@Environment(\.themeAccent) private var themeAccent

	var body: some View {
		Text("Hello").foregroundStyle(themeAccent)
	}
}
```

| | 무엇 | 사물함 비유 |
| --- | --- | --- |
| (A) `ThemeAccentKey` | `EnvironmentKey`를 채택한 **키 타입** | 칸 번호 + 기본 물건 |
| (B) `EnvironmentValues.themeAccent` | `extension`으로 추가한 **연산 프로퍼티** | 칸을 여닫는 문 |
| (C) `\.themeAccent` | (B)를 가리키는 **키 패스** `KeyPath<EnvironmentValues, Color>` | 문의 주소 |

- `defaultValue`가 필수인 이유: 아무도 넣지 않은 뷰가 읽었을 때 돌려줄 값이 있어야 함
- (A)는 `private`으로 숨겨도 됨. 바깥에서 쓰는 것은 (C)뿐

**`@Entry` — (A)(B)를 한 줄로 (Xcode 16+)**

```swift
extension EnvironmentValues {
	@Entry var themeAccent: Color = .blue
}
```

**하위에서 값을 바꾸게 하기 — 원본은 루트, 변경 수단을 같이 주입**

```swift
extension EnvironmentValues {
	@Entry var themeAccent: Color = .blue
	@Entry var setThemeAccent: (Color) -> Void = { _ in }
}

struct RootView: View {
	@State private var accent: Color = .orange              // 원본

	var body: some View {
		ContentView()
			.environment(\.themeAccent, accent)             // 값
			.environment(\.setThemeAccent) { accent = $0 }  // 변경 수단
	}
}

struct LeafView: View {
	@Environment(\.setThemeAccent) private var setThemeAccent

	var body: some View {
		Button("빨간색으로") { setThemeAccent(.red) }
	}
}
```

- 시스템의 `dismiss`, `openURL`이 같은 방식 (액션을 environment에 넣음)

**iOS 17 이후 — 객체도 같은 래퍼로**

```swift
@Observable final class Settings { var isDarkMode = false }

RootView().environment(settings)
@Environment(Settings.self) private var settings
```

### 더 파볼 질문
- environment 값 변경의 전파 경로를 자료 없이 설명할 수 있는가 (`@State` 변경 → 원본 뷰의 `body` 재호출 → 값 비교 → 선언한 뷰만 무효화)
- `sheet` · `fullScreenCover`로 띄운 뷰는 어디까지 environment를 물려받는가
- `EnvironmentValues`는 뷰마다 복사되는데 왜 비용이 크지 않은가 (내부 자료구조)
- 키 패스는 실제로 무엇이고 KVO와 무엇이 다른가 → [[KeyPath]]

### 관련
- [[@EnvironmentObject]] — 같은 통로로 `ObservableObject` 객체를 전달하는 쪽
- [[@State]] — environment에 흘려보내는 값의 원본
- [[DynamicProperty]] · [[@propertyWrapper]] — 래퍼를 뷰에 잇는 기반
- [[뷰 업데이트 사이클 (View Update Cycle)]] — 무효화에서 `body` 재호출까지
- [[값 의미론 (Value Semantics)]] · [[DataFlow]]

### 출처
- Apple Developer Documentation — [Environment](https://developer.apple.com/documentation/swiftui/environment)
- Apple Developer Documentation — [EnvironmentValues](https://developer.apple.com/documentation/swiftui/environmentvalues)
- Apple Developer Documentation — [EnvironmentKey](https://developer.apple.com/documentation/swiftui/environmentkey)
- Apple Developer Documentation — [Entry()](https://developer.apple.com/documentation/swiftui/entry())
- WWDC19 "Data Flow Through SwiftUI" (Session 226)
- WWDC23 "Discover Observation in SwiftUI" (Session 10149)
- WWDC24 "What's new in SwiftUI" (Session 10144)
