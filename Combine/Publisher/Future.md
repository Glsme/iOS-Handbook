# Future
비동기적으로 값 하나를 방출하거나 실패하는 Publisher

```swift
final class Future<Output, Failure> where Failure: Error
```

- 채택하는 Protocol
	- [[Publisher]]
- 클로저를 통해 초기화
- Swift Concurrency의 Continuation으로 교체 가능

- 동작 방식
```swift
let future = Future<String, Never> { promise in
	promise(.success("success 1"))
	promise(.success("success 2")) // 단일값 전달이기 때문에 success 2는 전달되지 않음
}

future.sink { completion in 
	print(completion)
} receiveValue: { value in
	print(value)
}
.cancel()

// success 1
// finished
```

1. `promise 1` 방출
2. 방출 후 `.finsihed` 수행
3. 만약 `promise`를 한 번도 호출하지 않으면 영원히 대기 상태 지속