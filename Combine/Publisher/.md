### Empty
아무 값도 방출하지 않는 Publisher

```swift
struct Empty<Output, Failure> where Failure: Error
```

- 채택하는 Protocol
	- [[Equatable]]
	- [[Publisher]]
- 조건에 따라 아무 값도 보내지 않고 깔끔하게 종료하고 싶을 때 사용
- 동작 방식
```swift
let empty = Empty<String, Never>()

empty.sink(receiveCompletion: { c in
	print(c)
}, receiveValue: { v in
	print(v)
})
.cancel()
```
1. empty 생성
2. 구독 시 바로 `.finished` 수행