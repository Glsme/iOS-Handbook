# @dynamicMemberLookup
존재하지 않는 프로퍼티에 점(dot) 문법으로 접근하면 컴파일러가 이를 `subscript(dynamicMember:)` 호출로 바꿔주는 Attribute

```swift
@dynamicMemberLookup
struct Wrapper {
    subscript(dynamicMember member: String) -> Any? { ... }
}
```

### 핵심 개념
- `wrapper.member` 코드는 컴파일 타임에 `wrapper[dynamicMember: "member"]` 호출로 치환됨
- 타입에 `member`라는 실제 프로퍼티/메서드가 있으면 그게 우선 — 진짜 멤버가 없을 때만 동적 조회로 넘어감
- `struct`, `class`, `enum`, `protocol`에 붙일 수 있음 (프로토콜에 붙이면 채택 타입마다 subscript 구현을 강제)

### subscript(dynamicMember:) 두 가지 형태
- String 기반
	- `subscript(dynamicMember member: String) -> Any?` 처럼 문자열을 그대로 받는 방식
	- 어떤 이름이든 다 받아버려서 오타가 나도 컴파일 에러 없이 런타임에 nil로 새어나갈 수 있음
	- Xcode 자동완성이 동작하지 않음
- KeyPath 기반 (타입 세이프)
	- `subscript<T>(dynamicMember keyPath: KeyPath<Value, T>) -> T` 처럼 KeyPath를 받는 방식
	- 실제 존재하는 프로퍼티만 넘길 수 있어 컴파일 타임 체크와 자동완성이 그대로 유지됨
	- `WritableKeyPath`를 쓰면 값 쓰기(set)도 위임 가능

### 특징
- SwiftUI의 [[@Binding]]이 이 속성으로 구현되어 있음
	- `@frozen @propertyWrapper @dynamicMemberLookup struct Binding<Value>`
	- 덕분에 `$user.name`처럼 내부 프로퍼티 하나만 골라서 바인딩 생성 가능
- 하나의 타입에 String 기반 / KeyPath 기반 subscript를 오버로드로 동시에 정의 가능
- Python(PythonKit), JavaScriptCore 등 다른 언어의 동적 객체를 Swift에서 점 문법으로 다루는 브릿징 용도로 자주 쓰임
- 동적 JSON 접근, 프록시/위임 타입처럼 "내부 값의 프로퍼티를 그대로 노출"하고 싶을 때도 사용

### 예시

**1. String 기반 — 동적 JSON 래퍼**
```swift
@dynamicMemberLookup
struct JSON {
    let value: [String: Any]

    subscript(dynamicMember member: String) -> JSON? {
        guard let raw = value[member] else { return nil }
        return JSON(value: raw as? [String: Any] ?? [:])
    }
}
```

**2. KeyPath 기반 — 타입 세이프 프록시**
```swift
@dynamicMemberLookup
struct Proxy<Value> {
    var value: Value

    subscript<T>(dynamicMember keyPath: KeyPath<Value, T>) -> T {
        value[keyPath: keyPath]
    }
}

struct Point { var x: Int; var y: Int }

let p = Proxy(value: Point(x: 1, y: 2))
p.x // p.value.x 로 위임, 자동완성/타입 체크 그대로 유지
```

### 주의할 점
- String 기반은 컴파일 타임 안정성을 포기하는 트레이드오프 — 외부 시스템과의 인터롭처럼 진짜 동적인 경우가 아니라면 남용하지 않는 게 좋음
- 문법적으로 어떤 멤버 이름이든 통과되기 때문에 실제 어떤 속성이 존재하는지 코드만 봐서는 파악하기 어려워질 수 있음

### 참고 링크
- [SE-0195](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0195-dynamic-member-lookup.md)