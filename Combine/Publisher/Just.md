# Just
각 구독자에게 한 번만 출력을 내보내고 종료하는 Publisher

```swift
struct Just<Output>
```

- 채택하는 Protocol
	- [[Copyable]]
	- [[Equatable]]
	- [[Escapable]]
	- [[Publisher]]
- 오류 발생 불가능 (내부적으로 `Failure = Never` 로 구현)

- 동작 방식
```swift
let just = Just("Hi")

publisher.sink { completion in 
	print(completion)
} receiveValue: { value in
	print(value)
}
.cancel()

// 출력
// Hi
// finished
```
1. Just Publisher를 구독 시 값 방출
2. 방출 후 바로 `.finished`를 수행