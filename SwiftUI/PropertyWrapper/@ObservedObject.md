# @ObservedObject
외부에서 소유한 [[ObservableObject]]를 구독해 뷰를 갱신하는 Property Wrapper

```swift
@propertyWrapper
struct ObservedObject<ObjectType> where ObjectType: ObservableObject
```

```swift
@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *)
@frozen @propertyWrapper public struct ObservedObject<ObjectType>: DynamicProperty
	where ObjectType: ObservableObject {

	public init(wrappedValue: ObjectType)
	public init(initialValue: ObjectType)

	public var wrappedValue: ObjectType
	public var projectedValue: ObservedObject<ObjectType>.Wrapper { get }
}
```

### 핵심 개념
- [[ObservableObject]]가 "관찰당하는 쪽"이라면, @ObservedObject는 "관찰하는 쪽"
- **객체를 소유하지 않음** — 생명주기는 외부(부모 뷰, UIKit ViewController 등)가 책임짐
- 소유가 필요하면 [[@StateObject]]
- 애초에 객체를 넘길지 값을 넘길지의 판단은 [[DataFlow]]

### 특징
- 제네릭 제약으로 [[ObservableObject]] 채택을 **컴파일 타임에 강제**
- `objectWillChange`를 구독해 신호가 오면 해당 뷰를 무효화
	- [[@Published]] 각각의 퍼블리셔를 구독하는 게 아님 — **`objectWillChange` 하나만** 구독
	- 따라서 어떤 프로퍼티가 바뀌든 그 뷰의 `body`는 전부 재평가됨 ([[Diffing]])
- `$`를 붙이면 내부 프로퍼티의 [[@Binding]] 생성 가능
	- `$viewModel.name` → `Binding<String>`
	- 단, 해당 프로퍼티가 `private(set)`이면 불가
- 커스텀 `init`에서는 언더스코어로 래퍼 인스턴스를 직접 대입
	- `self.viewModel = viewModel`은 래핑된 값 대입이라 `init` 단계에서 사용 불가

```swift
init(viewModel: MyViewModel) {
	self._viewModel = ObservedObject(wrappedValue: viewModel)
}
```

### 소유권 비교

| 래퍼 | 객체 소유 | 언제 쓰나 |
|---|---|---|
| [[@StateObject]] | **O** — 뷰가 만들고 생명주기를 책임짐 | 이 뷰가 뷰모델의 주인일 때 |
| @ObservedObject | **X** — 외부에서 받아 보기만 함 | 부모나 UIKit이 주인일 때 |
| [[@EnvironmentObject]] | X — environment에서 주입받음 | 깊은 계층에 전달할 때 |

### 대표적인 함정 — 뷰가 직접 생성하면 안 됨
```swift
struct BadView: View {
	@ObservedObject var vm = VM()   // 위험
}
```
소유하지 않으므로 SwiftUI가 뷰 struct를 다시 만들 때마다 `VM()`이 새로 생성되어
**상태가 초기화되는 버그**가 된다. 뷰가 직접 만드는 경우엔 [[@StateObject]]를 쓸 것
(한 번만 생성됨).

### UIKit 호스팅 구조에서의 사용
`UIHostingController`로 SwiftUI를 올릴 때는 ViewModel의 주인이 ViewController다.
뷰 struct가 다시 만들어져도 ViewController가 붙잡고 있으므로 위 함정에 걸리지 않는다.

```swift
// ViewController가 소유
final class MyViewController: BaseHostingViewController<MyView> {
	private let viewModel: MyViewModel

	init(viewModel: MyViewModel) {
		self.viewModel = viewModel
		super.init(rootView: MyView(viewModel: viewModel))   // 주입만
	}
}

// SwiftUI는 관찰만
struct MyView: View {
	@ObservedObject private var viewModel: MyViewModel
}
```

공식 문서
https://developer.apple.com/documentation/swiftui/observedobject
