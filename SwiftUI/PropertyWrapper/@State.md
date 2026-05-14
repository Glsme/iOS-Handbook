# @State
SwiftUI가 관리하는 값을 읽고 쑬 수 있는 Property Wrapper

```swift
@frozen @propertyWrapper 
struct State<Value>
```

```swift
@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *) 
@frozen @propertyWrapper public struct State<Value> : DynamicProperty {
	 public init(wrappedValue value: Value) 
	 public init(initialValue value: Value) 
	 
	 public var wrappedValue: Value { get nonmutating set } 
	 public var projectedValue: Binding<Value> { get } 
}
```


### 특징
- SwiftUI에 의해 관리됨
	- @State로 선언한 프로퍼티는 값이 변경되면 뷰 계층 구조의 부분을 업데이트
	- @State를 자식 뷰에 전달하면 부모에서 값이 변경될 때마다 자식을 업데이트
	- 단, 자식 뷰에서 값을 수정하려면, 부모에서 자식으로 Binidng을 전달하여 자식 뷰에서 값을 수정이 가능
- @State 프로퍼티는 힙 영역에 저장
- @State 프로퍼티가 저장되어있는 Storage는 View 인스턴스가 아닌 View itentity에 종속
- @State 프로퍼티가 `Equatable` 채택 시 같은 값 대입은 변경이 없다고 판단하여 body 재호출 스킵
- @State 프로퍼티는 항상 `private`으로 선언하고 가장 상위 뷰에서 @State를 관리할 것
    - 뷰 초기화 시 @State 프로퍼티 값도 같이 초기화하게 되면 SwiftUI에서 @State 프로퍼티를 관리하는 공간인 Storage에서 conflict 발생
	- 만약 상위 뷰에서의 State값을 하위 뷰에서도 접근할 수 있게하려면 [[@Binding]]을 하위 뷰에 넘겨주거나 상위 뷰에서 read-only property로 설정하여 값을 공유