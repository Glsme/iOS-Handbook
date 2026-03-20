# PassthroughSubject
값을 저장하지 않고 그 순간 구독 중인 구독자에게만 값을 전달하는 Subject

```swift
final class PassthroughSubject<Output, Failure> where Failure: Error
```

- 여러 구독자에게 동시에 값 방출
- 구독 시점 이후부터 값을 방출
- 완료 이벤트 방출 후 더이상 이벤트를 방출하지 않음
- 채택하는 Protocol
	- [[Publisher]]
	- [[Subject]]
- 동작 방식
```swift
let subject = PassthroughSubject<String, Never>()

// 구독 전에 보낸 값은 유실
subject.send("A")  // 아무도 못 받음

// 구독
subject.sink { print($0) }

// 구독 후에 보낸 값만 받음
subject.send("B")  // 출력: B
subject.send("C")  // 출력: C
```
