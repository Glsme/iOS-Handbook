# Equatable
값이 동일한지 비교할 수 있는 Protocol

```swift
public protocol Equatable {
	public static func == (lhs: Self, rhs: Self) -> Bool
	public static func != (lhs: Self, rhs: Self) -> Bool
}
```

- `Equatable` Potocol을 채택 시 `==` , `!=`연산자를 이용하여 값이 동일한지 비교 가능
- Collection Type에서 요소가 `Equatable` 채택 시 `contains(_:)` 로 비교 가능


## 사용자 정의 타입에서 사용
### 구조체
`Equatable` 채택 시 만약 모든 타입이 `Equtable`을 채택하고 있는 타입이라면
`==` 메서드를 구현하지 않아도 사용 가능 (커스텀 가능)
(모든 타입이 채택하지 않는다면 직접 구현)

```swift
struct Human: Equatable {
	var name = ""
	var age = 0
}
```


### 클래스
`Equatable` 채택 시 직접 `==`메서드 구현


```swift
class Human: Equatable {
	var name = ""
	var age = 0
	
	static func == (lhs: Human, rhs: Human) -> Bool {
		return lhs.name == rhs.name && lhs.age == rhs.age
	}
}
```



### 열거형
- 연관값이 없는 열거형일 경우 `Equatable` 자동 채택

```swift
enum Gender {
	case male
	case female
}
```


- `Equatable`을 채택하고 있는 연관값이 있는 열거형일 경우 `Equatable` 채택 시 비교 가능

```swift
enum Gender: Equatable {
	case male(name: String)
	case female(name: String)
}
```


- `Equatable`을 채택하고 있지 않는 연관값이 있는 열거형일 경우 `Equtable` 채택 시 `==` 메서드 직접 구현

```swift
class Human {
    var name = ""
}

  

enum Gender: Equatable {
    case male(name: Human)
    case female(name: Human)

    static func == (lhs: Gender, rhs: Gender) -> Bool {
        switch (lhs, rhs) {
        case (.male, .male): return true
        case (.female, .female): return true
        default: return false
        }
    }
}
```

