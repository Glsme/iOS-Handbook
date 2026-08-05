# @Published
프로퍼티 값이 바뀔 때 Publisher로 방출해주는 Property Wrapper

```swift
@propertyWrapper struct Published<Value>
```

```swift
@available(iOS 13.0, macOS 10.15, tvOS 13.0, watchOS 6.0, *)
@propertyWrapper public struct Published<Value> {
	public init(wrappedValue value: Value)
	public init(initialValue value: Value)

	public struct Publisher: Publisher {
		public typealias Output = Value
		public typealias Failure = Never
	}

	public var projectedValue: Published<Value>.Publisher { mutating get set }
}
```

### 특징
- 이름 앞에 `$`를 붙이면 `Published.Publisher`를 얻어 구독 가능
	- `$count.sink { ... }` 형태로 [[Publisher]]처럼 사용
- Failure 타입이 `Never`로 고정 — 에러를 흘릴 수 없음
- 구독 즉시 현재 값을 방출 → [[CurrentValueSubject]]와 성격이 같음
- **클래스의 프로퍼티에만 사용 가능**
	- struct에서 접근하면 `@Published is only available on properties of classes` 컴파일 에러
- **방출 시점이 `willSet`** — 프로퍼티가 실제로 갱신되기 **전**에 방출
	- sink 클로저 안에서 원본 프로퍼티를 다시 읽으면 **이전 값**이 나옴
	- [[CurrentValueSubject]]는 반대로 `value` 갱신 후 방출
- **이전 값과 비교하지 않음** — `Equatable` 제약이 없어 같은 값을 대입해도 방출
	- 불필요한 방출을 막으려면 대입 전에 직접 거르거나 `removeDuplicates()` 사용
- 선언한 클래스가 [[ObservableObject]]를 채택하면 `objectWillChange`도 함께 발화
	- 채택하지 않으면 `$foo` 구독만 동작하고 SwiftUI 갱신은 일어나지 않음
- 스트림을 명시적으로 종료할 수 없음 — `send(completion:)`에 해당하는 API가 없음
- 퍼블리셔에 `value` 같은 현재 값 접근자가 없음 — 원본 프로퍼티를 직접 읽어야 함

### Subject와 비교

| | @Published | [[CurrentValueSubject]] | [[PassthroughSubject]] |
|---|---|---|---|
| 현재 값 보유·재생 | O | O | X |
| Failure 타입 | `Never` 고정 | 지정 가능 | 지정 가능 |
| 방출 시점 | `willSet` (갱신 전) | 갱신 후 | 즉시 |
| 값 직접 읽기 | 원본 프로퍼티 | `.value` | 불가 |
| 값 주입 | 프로퍼티 대입 | `.send(_:)` | `.send(_:)` |
| 스트림 종료 | 불가 | `.send(completion:)` | `.send(completion:)` |
| [[ObservableObject]] 연동 | 자동 | 수동 | 수동 |

### willSet 타이밍 예시
```swift
final class VM {
	@Published var count = 0
	let subject = CurrentValueSubject<Int, Never>(0)
}

let vm = VM()

vm.$count.sink { new in
	print("published:", new, vm.count)          // 1, 0  ← 프로퍼티는 아직 옛날 값
}
vm.count = 1

vm.subject.sink { new in
	print("subject:", new, vm.subject.value)    // 1, 1  ← 이미 갱신됨
}
vm.subject.send(1)
```

### 선택 기준
- **화면 상태를 SwiftUI에 바인딩** → @Published + [[ObservableObject]]
- **sink 안에서 다른 프로퍼티를 같이 읽어야 함** → [[CurrentValueSubject]] (willSet 함정 회피)
- **에러 타입이 필요하거나 스트림을 끝내야 함** → [[CurrentValueSubject]] / [[PassthroughSubject]]
- **한 번 일어나고 끝나는 이벤트** → [[PassthroughSubject]]
	- @Published는 재생(replay)되므로 재구독 시 이벤트가 다시 발화함
	- 굳이 @Published로 처리하려면 "소비 후 초기값으로 리셋" 패턴 필요

공식 문서
https://developer.apple.com/documentation/combine/published
