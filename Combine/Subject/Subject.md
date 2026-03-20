# Subject
외부 호출자가 요소를 발행할 수 있는 방법을 제공하는 Publisher

```swift
protocol Subject<Output, Failure>: AnyObject, Publisher
```

- `send(:)` 로 Stream으로 값을 주입 가능
- 채택하는 Protocol
	- [[Publisher]]

공식 문서
https://developer.apple.com/documentation/combine/subject


## Subject 종류
- [[CurrentValueSubject]]
- [[PassthroughSubject]]

