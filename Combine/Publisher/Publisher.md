---
tags:
---
# Publisher
: 특정 타입이 시간에 따라 값을 보낼 수 있음을 선언

```swift
protocol Publisher<Output, Failure>
```



Apple 공식 문서 링크
https://developer.apple.com/documentation/combine/publisher


## Publisher 종류
- [[Just]]
- [[Future]]
- [[Deferred]]
- [[Empty]]
- [[Record]]
- [[Fail]]


## `eraseToAnyPublisher()`

^382016

하위 구독자에게 직접 타입을 노출하는 대신 추상화된 Publisher를 전달하기 위해 타입을 소거


