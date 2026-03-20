# Fail
Error와 함께 즉시 종료되는 Publisher

```swift
struct Fail<Output, Failure> where Failure: Error
```

- 채택하는 Protocol
	- [[Copyable]]
	- [[Equatable]]
	- [[Escapable]]
	- [[Publisher]]
- `Empty` Publisher와 마찬가지로 즉시 종료 (Error와 Finished 차이)
- 동작 방식
```swift
enum MyError: Error { case somethingWrong }
let fail = Fail<String, MyError>(error: .somethingWrong)

fail.sink { completion in
	print(completion)  // failure(MyError.somethingWrong)
} receiveValue: { value in
	print(value)       // 절대 호출 안 됨
}
```

1. 구독 시 바로 저장되어있는 `error` 프로퍼티에 저장되어있는 Error를 `.failure`로 수행
2. 종료