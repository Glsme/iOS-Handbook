# EquatableView
[[Diffing]]의 기본 비교 대신 내가 구현한 `==`를 쓰게 만드는 View

```swift
@frozen struct EquatableView<Content> where Content: Equatable, Content: View
```

```swift
extension View where Self: Equatable {
	public func equatable() -> EquatableView<Self>
}
```

### 핵심 개념
- SwiftUI의 기본 [[Diffing]]은 [[Equatable]]을 쓰지 않는 **내부 비교**
- [[Equatable]]을 개입시키는 **유일하게 보장된 경로**가 `.equatable()` / `EquatableView`
- 클로저처럼 비교 불가능한 프로퍼티를 비교 대상에서 **제외**할 때 사용

### 사용법
```swift
struct SectionView: View, Equatable {
	let type: SectionType
	let items: [Item]
	let onRegister: () -> Void          // 비교에서 제외
	let onRemove: (Item) -> Void        // 비교에서 제외

	static func == (lhs: SectionView, rhs: SectionView) -> Bool {
		return lhs.type == rhs.type && lhs.items == rhs.items
	}

	var body: some View { ... }
}

// 호출부
SectionView(type: type, items: items, onRegister: ..., onRemove: ...)
	.equatable()
```

`.equatable()`을 붙이지 않으면 `Equatable` 채택만으로는 개입이 보장되지 않는다.

### 주의점
- **비교에서 뺀 값이 실제로 바뀌면 stale 상태가 된다**
	- 위 예시에서 클로저가 다른 상태를 캡처하면 뷰가 옛날 클로저를 계속 들고 있게 됨
	- 클로저가 고정된 객체 참조(EventHandler 등)만 캡처하면 안전
	- 나중에 캡처 대상이 늘어나는 순간 이 최적화가 **버그로 바뀜**
- `==` 구현이 비싸면 오히려 손해
	- 큰 배열 전체 비교보다 body 재평가가 쌀 수 있음
- 계측 없이 예방적으로 붙이지 말 것 — [[Diffing]]의 측정 방법 참고

### 언제 쓰나
- 부모가 자주 갱신되는데 자식은 실제로 바뀌는 일이 드물 때
- 자식이 콜백 클로저를 프로퍼티로 받아 기본 비교가 항상 실패할 때
- `Self._printChanges()`나 Instruments로 **불필요한 재평가를 확인한 뒤**

### 관련
- [[Diffing]]
- [[Identity]]
- [[Equatable]]

공식 문서
https://developer.apple.com/documentation/swiftui/equatableview
