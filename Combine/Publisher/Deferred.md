# Deferred
구독할 때까지 Publisher 생성을 미루는 Publisher

```swift
struct Deferred<DeferredPublisher> where DeferredPublisher: Publisher
```

- 채택하는 Protocol
	- [[Publisher]]
- 동작 방식
```swift
let deferred = Deferred {
      Future<String, Error> { promise in
          print("실행됨")  // ← 구독해야 출력됨
          promise(.success("Hi"))
      }
  }

// 아직 아무것도 출력 안 됨
  
deferred.sink { print($0) }
// 이제 "실행됨" 출력
```
1. 구독 시 인스턴스 생성
2. 값 방출
3. `.finsihed` 수행