# @Binding
다른 곳의 값을 읽고 쓰게 연결해 주는 Property Wrapper (부모의 @State를 자식과 공유할 때 사용)

```swift
@frozen @propertyWrapper @dynamicMemberLookup 
struct Binding<Value>
```

```swift
@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *) 
@frozen @propertyWrapper @dynamicMemberLookup 
public struct Binding<Value> {
	public var transaction: Transaction
	
	public init(get: @escaping () -> Value, set: @escaping (Value) -> Void) 
	public init(get: @escaping () -> Value, set: @escaping (Value, Transaction) -> Void) 
	
	public static func constant(_ value: Value) -> Binding<Value>
	
	public var wrappedValue: Value { get nonmutating set } 
	public var projectedValue: Binding<Value> { get } 
	
	public init(projectedValue: Binding<Value>) 
	public subscript<Subject>(dynamicMember keyPath: WritableKeyPath<Value, Subject>) -> Binding<Subject> { get }
}
```

### 핵심 개념
- [[@State]]가 값의 **주인**이라면, @Binding은 그 값을 **빌려 쓰는 통로**
- 값을 복사하지 않고 원본을 가리키는 "리모컨"을 넘겨받는 셈 — 자식이 바꾸면 원본도 함께 변경

### 특징
- 값을 소유하지 않고 원본을 가리키기만 하는 참조(reference) 타입
	- 실제 값(Source of Truth)은 보통 부모의 [[@State]]에 있고, @Binding은 그걸 가리키기만 함
	- 그래서 `wrappedValue`가 `nonmutating set` — 값을 바꿔도 원본 저장소가 바뀔 뿐 @Binding 자신은 그대로
- 읽기뿐 아니라 쓰기도 가능한 양방향 바인딩(two-way binding)
	- 자식 뷰가 부모의 상태를 직접 읽고 고칠 수 있음
	- `TextField`, `Toggle`, `Slider` 등 입력 뷰가 값을 원본에 되돌려 쓰려고 요구
- 이름 앞에 `$`를 붙여서 전달
	- @State, @StateObject 앞에 `$`를 붙이면 @Binding으로 변환 (이 값이 projectedValue)
	- 예: 부모의 `@State var count`를 `$count`로 넘기면 자식은 `@Binding var count`로 받아 값 공유
- 값 안의 특정 프로퍼티만 골라서 바인딩 가능
	- `$user.name`처럼 내부 프로퍼티 하나만 가리키는 @Binding도 바로 생성 가능 
	  ([[@dynamicMemberLookup]] 덕분)
- 커스텀 Binding을 직접 생성 가능
	- `get`/`set` 클로저로 동작을 직접 정의. 변환·검증 로직을 끼워 넣고 싶을 때 사용
- `Binding.constant(_:)`로 고정값 바인딩 생성
	- 항상 같은 값만 주고 쓰기는 무시 — Preview나 테스트에서 사용
- @Binding만으로는 값이 유지되지 않음
	- 통로일 뿐 스스로 값을 보관하지 못함 — 상위 어딘가에 원본([[@State]] 등)이 반드시 존재해야 함
- iOS 17+에서 `@Observable`과 함께 쓸 때는 `@Bindable` 사용
	- `@Observable` 객체는 @Binding 대신 `@Bindable`로 바인딩

### 예시
```swift
struct ParentView: View {
    @State private var isOn = false        // 값의 주인 (원본)

    var body: some View {
        // $isOn → 자식에게 전달
        ChildView(isOn: $isOn)
    }
}

struct ChildView: View {
    @Binding var isOn: Bool                // 부모의 원본을 빌려 씀

    var body: some View {
        Toggle("알림 받기", isOn: $isOn)   // 바꾸면 부모의 isOn도 함께 변경
    }
}
```
