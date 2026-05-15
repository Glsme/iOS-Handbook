# Identifiable
객체에 고유한 식별자(id)를 제공하여 객체를 고유하게 식별할 수 있게 해주는 Protocol

```swift
public protocol Identifiable {
	associatedtype ID: Hashable
	
	var id: Self.ID { get }
}
```



## 특징
- Class 타입에 대해 [[ObjectIdentifier]]를 사용해서 기본 구현을 제공
- `id` 의 타입은 개발자가 자유롭게 결정 가능

### ID 특성
- 항상 고유하도록 보장
- 한 환경 안에서는 영구적으로 고유
- 한 프로세스가 살아있는 동안만 고유
- 한 객체가 살아있는 동안만 고유
- 현재 컬렉션 안에서만 고유

