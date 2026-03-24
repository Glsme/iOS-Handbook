# Hashable
`Hasher`를 통해 정수 hash 값으로 해싱될 수 있는 Protocol

```swift
protocol Hashable: Equatable {
	var hashValue: Int { get } // 더이상 사용하지 않음
	func hash(into hasher: inout Hasher)
}
```

```mermaid
flowchart LR
	Hashable --> Hasher --> Hash

```
- Hasher를 통하여 Hash (int type)으로 변환


### 특징
- `Hashable` 채택 시 Set, Dictionary 키로 사용 가능
- 채택하고 있는 Protocol
	- [[Equatable]]

## `Equatable` 채택 이유
Hash 충돌이 발생할 경우 실제로 같은 값인지 `==` 메서드로 확인 필요

## 사용자 정의 타입에서 사용
### 구조체
구조체 내 프로퍼티가 모두 `Hashble` 타입이면 채택만으로 사용 가능

```swift
struct Human: Hashable {
    let name: String
    let age: Int
}
```


### 클래스
`hash(into:)`와 `==` 메서드 직접 구현

```swift
class Human: Hashable {
	let name = ""
	let age = 0
	
	static func == (lhs: Human, rhs: Human) -> Bool {
        return lhs.name == rhs.name && lhs.age == rhs.age
    }
	
	hash(into hasher: inout Hasher) {
		hasher.combine(name)
		hasher.combine(age)
	}
}
```



### 열거형
연관값이 없는 열겨형일 경우 자동 구현

```swift
enum Gender {
    case male
}
```


연관값이 있는 열거형의 경우 연관깂이 모두 `Hashable`을 채택

```swift
enum Gender: Hashable {
    case male(age: Int)
}
```
