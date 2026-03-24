# Sendable
Data race 없이 임의의 동시 컨텍스트에서 값을 공유할 수 있는 thread-safe 타입의 Protocol

```swift
protocol Sendable: SendableMetatype
```
- [[SendableMetatype]]



##  Sendable 채택 가능 타입
- Value Types
- Actor
- Class (final + no mutable storage + no superclass)
	- final 키워드가 있을 때
	- 불변하거나 sendable한 프로퍼티만 포함하고 있을 때
	- Super class가 없거나 NSObject를 Super class 로 사용할 때
- Class (marked with `@MainActor`)
- Functions and closures (by marking them with `@Sendable`)
- Tuple (모든 elements 가 Sendable일 경우)
- Meta Type


