# 클로저 (Closure)
변수에 저장하거나 다른 함수에 전달할 수 있는 **기능 블록**. 만들어진 자리의 변수를 **캡처**해서 들고 다닌다.

| 형태 | 이름 | 캡처 |
| --- | --- | --- |
| 전역 함수 | 있음 | 없음 |
| 중첩 함수 | 있음 | 감싼 함수의 값 |
| 클로저 표현식 `{ }` | 없음 | 주변 문맥의 값 |

- "이름 없는 함수"는 클로저 표현식만 가리키는 말. Swift에서는 `func`로 만든 함수도 클로저의 한 종류
- Objective-C 블록의 후계. 타입 추론 · `$0` · 암시적 반환 · 후행 클로저로 문법을 줄임

### 왜 필요한가

**동작을 값처럼 넘기면서, 그 동작에 필요한 주변 값까지 함께 들고 가기** 위함.

없으면 생기는 문제
- 추가 코드가 늘어남. delegate 프로토콜 선언, 구현 메서드, 연결 코드가 따로 필요
- 요청하는 곳과 응답을 처리하는 곳이 떨어짐
- 함수가 끝나면 사라지는 값은 **외부(프로퍼티)에 따로 저장**해 둬야 함

```swift
// 클로저 없이: delegate
final class SearchScreen: SearchDelegate {
	private var pendingKeyword = ""             // 응답 시점까지 살려 두려고 옮겨 적음

	func search(keyword: String) {
		pendingKeyword = keyword
		api.delegate = self                     // 연결
		api.request()
	}

	func didReceive(count: Int) {               // 처리는 다른 메서드
		label.text = "\(pendingKeyword) 검색 결과 \(count)건"
	}
}

// 클로저
func search(keyword: String) {
	api.request { count in
		label.text = "\(keyword) 검색 결과 \(count)건"
	}
}
```

**클로저에 값이 들어오는 두 길**

| 길 | 누가 주는가 | 위 예시 |
| --- | --- | --- |
| 파라미터 | 클로저를 **호출하는 쪽** | `count` (api가 줌) |
| 캡처 | 클로저가 **만들어진 자리** | `keyword` (선언도 전달도 없이 그냥 씀) |

- 함수를 인자로 넘길 수 있는 이유는 일급 함수
- `keyword`가 따라가는 이유는 캡처. 해야 할 일은 본문에서 쓰는 것이 전부

### 내부 동작 — 캡처

```
스택                     힙
┌──────────────┐
│ 클로저 값      │
│  함수 포인터   ─┼──▶ 코드 영역 (컴파일된 본문)
│  문맥 포인터   ─┼──▶ ┌────────────────┐
└──────────────┘    │ 문맥 객체        │
                    │  참조 카운트      │
                    │  count 박스 ────┼──▶ ┌──────────────┐
                    └────────────────┘    │ 박스          │
                                          │  참조 카운트    │
                                          │  count = 2   │
                                          └──────────────┘
```

- 클로저 값 = **함수 포인터 + 문맥 포인터**
- 코드는 컴파일 때 한 번 만들어짐. 런타임에 새로 생기는 것은 **문맥**
- 캡처는 기본적으로 **값이 아니라 변수 자체**를 붙잡음
- 캡처된 `var`는 스택이 아니라 힙의 **박스**에 놓임 → 함수가 끝나도 살아 있음 ([[힙과 스택]])

```swift
func makeCounter() -> () -> Int {
	var count = 0
	return { count += 1; return count }
}

let x = makeCounter()
let y = makeCounter()
let z = y

x()   // 1
y()   // 1
y()   // 2
z()   // 3   y와 같은 박스
x()   // 2   x는 다른 박스
```

- `makeCounter()`를 부를 때마다 **새 박스**
- 클로저는 참조 타입. `z = y`는 같은 문맥(같은 박스)을 공유 ([[값 의미론 (Value Semantics)]]과 대비)
- "참조 타입"의 뜻은 대입하면 공유한다는 것. 모든 클로저가 저장소 하나를 쓴다는 뜻이 아님

**박스가 생기는 경우**

| 캡처 대상 | 문맥에 들어가는 것 | 박스 |
| --- | --- | --- |
| 변경되는 `var` 지역 변수 | 박스를 가리키는 참조 | 있음 |
| `let` 상수 · 함수 파라미터 | 값 복사본 | 없음 |
| 클래스 메서드 안의 `self` | 참조의 복사본 (강한 참조) | 없음 |
| 캡처 리스트 `[x]` · `[weak self]` | 생성 시점의 복사본 | 없음 |

- 박스는 **변수**를 담는 저장소, 객체는 참조가 **가리키는 대상**. 층이 다름
- 본문에 `self.title`이라고 써도 캡처되는 것은 `title`이 아니라 **`self`**
- 메모리 릭을 따질 때는 박스가 아니라 "참조 타입을 강하게 잡았는가"만 보면 됨

**수명 — [[ARC (Automatic Reference Counting)]]**

| 시점 | 일어나는 일 |
| --- | --- |
| 클로저가 **만들어질 때** | 문맥 객체 생성, 캡처한 객체의 카운트 +1 |
| 클로저를 호출할 때 | 카운트 변화 없음 |
| 클로저가 **버려질 때** | 문맥 객체 해제, 캡처한 객체의 카운트 -1, 박스 해제 |

- 박스는 그 박스를 가리키는 클로저가 하나도 남지 않을 때 사라짐
- "버려진다"는 클로저를 담은 변수 · 프로퍼티가 없어지는 것. 실행이 끝나는 것과 다름

> 함수 포인터 + 문맥 포인터 구성, 힙 박스(`alloc_box`), 스택 승격은 컴파일러 구현 수준의
> 설명이다. 언어가 보장하는 것은 "클로저는 주변 변수를 캡처하고, 변경되지 않는 값은 복사본을
> 캡처할 수 있다"까지다. 캡처가 하나뿐이면 박스가 곧 문맥이 되는 식으로 배치가 달라질 수 있다.

### 캡처 리스트

기본 캡처는 **창문**, 캡처 리스트는 **사진**이다.

```swift
var score = 1
let live = { print(score) }               // 변수 자체에 연결
let snap = { [score] in print(score) }    // 지금 값 1을 복사
score = 9
snap()   // 1
live()   // 9
```

| | 클로저 안의 `score` | 값이 정해지는 때 |
| --- | --- | --- |
| `{ print(score) }` | 바깥 변수 그 자체 | 실행할 때마다 읽음 |
| `{ [score] in print(score) }` | 클로저 전용 새 상수 | 클로저를 만들 때 한 번 |

- `[score]`는 `[score = score]`의 줄임. 클로저 안에 **새 상수**를 만들어 지금 값으로 초기화
- 기본값이 연결인 이유: 그래야 `makeCounter`처럼 클로저가 바깥 변수를 바꾸고 바깥의 변경도 볼 수 있음
- 참조 타입을 `[obj]`로 잡으면 복사되는 것은 객체가 아니라 참조

**`weak` / `unowned`**

| | 카운트 | 타입 | 객체가 해제된 뒤 |
| --- | --- | --- | --- |
| `[weak self]` | 올리지 않음 | 옵셔널 | `nil` |
| `[unowned self]` | 올리지 않음 | 옵셔널 아님 | 접근하면 **크래시** |

- `weak`: 사용할 때 값이 없을 수도 있다는 뜻
- `unowned`: 값이 무조건 있다는 가정. **생명 주기를 확실히 알 때만** 사용

### @escaping

**함수가 종료된 이후에** 클로저가 실행될 수 있으면 붙인다.

```swift
func first(_ work: () -> Void) {
	print("A"); work(); print("B")
}

func second(_ work: @escaping () -> Void) {
	print("C")
	DispatchQueue.main.asyncAfter(deadline: .now() + 1) { work() }
	print("D")
}

first { print("E") }
second { print("F") }
// A E B C D (1초 뒤) F
```

| | non-escaping (기본) | `@escaping` |
| --- | --- | --- |
| 실행 시점 | 함수 반환 전 | 함수 반환 후 가능 |
| 저장 · 비동기 전달 | 컴파일 에러 | 가능 |
| 클래스에서 `self` | `self.` 생략 가능 | `self.` 또는 `[self]` 명시 |
| 순환 참조 | 생길 수 없음 | 저장되면 생길 수 있음 |
| 문맥 메모리 | 스택 가능 | 힙 |
| 예 | `map` · `filter` · `forEach` · `sorted` | 완료 콜백, 프로퍼티 저장 |

- 탈출 경로는 두 가지: **외부에 저장**, **다른 escaping 파라미터에 전달**
- `self.` 명시 요구는 "이 클로저는 `self`를 붙잡고 오래 살 수 있다"를 코드에 드러내는 장치
- 옵셔널 클로저 파라미터는 `@escaping` 없이도 escaping 취급

### withoutActuallyEscaping

non-escaping 클로저를 **블록 안에서만 잠깐 escaping인 것처럼** 쓰게 해 주는 표준 라이브러리 함수.

```swift
func withoutActuallyEscaping<ClosureType, ResultType>(
	_ closure: ClosureType,
	do body: (ClosureType) throws -> ResultType
) rethrows -> ResultType
```

필요한 상황
- 내 함수는 클로저를 non-escaping으로 받음
- 그런데 안에서 부르려는 API가 `@escaping`을 요구함 → 그대로 넘기면 컴파일 에러
- 실제로는 내 함수가 끝나기 전에 클로저 사용이 끝남

```swift
func allValues(in array: [Int], match predicate: (Int) -> Bool) -> Bool {
	// array.lazy.filter(predicate)   ❌ lazy의 filter는 @escaping을 요구
	return withoutActuallyEscaping(predicate) { escapable in
		array.lazy.filter { !escapable($0) }.isEmpty
	}
}
```

- `lazy.filter`는 조건을 저장해 두었다가 나중에 평가하므로 `@escaping`을 요구함
- 위 코드는 `isEmpty`에서 바로 평가가 끝남. 저장된 조건이 블록 밖으로 나가지 않음
- 이 사실을 컴파일러는 모르므로 **"실제로는 탈출하지 않는다"를 개발자가 보증**하는 것

| 방법 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| 파라미터를 `@escaping`으로 변경 | 간단함 | 호출하는 쪽 전부에 `self.` 명시 · 순환 참조 검토 부담이 전파됨 |
| `withoutActuallyEscaping` | 함수 시그니처는 non-escaping 유지 | 보증이 틀리면 런타임 문제 |

주의
- `escapable`은 **블록이 끝나기 전까지만** 유효
- 블록이 끝난 뒤 저장 · 참조 · 실행하면 미정의 동작 (공식 문서 표현)
- 비동기 작업에 넘겼다면 블록을 벗어나기 전에 그 작업이 끝났음을 보장해야 함

> 현재 컴파일러는 블록이 끝날 때 클로저가 아직 붙잡혀 있는지 검사해 런타임 에러로 멈춘다.
> 이 검사는 구현 동작이고, 문서가 보장하는 것은 "블록 밖 사용은 미정의 동작"까지다.

### 후행 클로저 (Trailing Closure)

파라미터의 마지막이 클로저일 때 **소괄호를 먼저 닫고 중괄호를 뒤에 쓰는 표기법**. `})`처럼 보이지 않게 한다.

```swift
names.sorted(by: { $0 < $1 })                 // 괄호 안
names.sorted { $0 < $1 }                      // 후행 클로저. 위와 같은 코드
```

| 용어 | 묻는 것 | 판별법 |
| --- | --- | --- |
| 후행 (trailing) | 어디에 **썼는가** | `{ }`가 `)` 밖에 있는가 |
| 탈출 (escaping) | 언제 **실행되는가** | 함수가 끝난 뒤에도 불릴 수 있는가 |

- "후행"의 뒤는 시간이 아니라 **괄호의 뒤**. 둘은 독립적

| 호출 | 후행 | escaping |
| --- | --- | --- |
| `names.sorted { $0 < $1 }` | O | X |
| `loader.load(id: 1) { user in ... }` | O | O |
| `loader.load(id: 1, completion: handle)` | X | O |

### @autoclosure

인자로 넘긴 **표현식을 컴파일러가 자동으로 클로저로 감싸 주는** 파라미터 속성. 호출하는 쪽은 `{ }` 없이 값처럼 쓴다.

```swift
// 없이: 호출부에 중괄호가 필요
func log(_ message: () -> String) {
	if isEnabled { print(message()) }
}
log({ makeReport() })

// @autoclosure: 값처럼 넘기지만 실제로는 { makeReport() } 로 감싸짐
func log(_ message: @autoclosure () -> String) {
	if isEnabled { print(message()) }
}
log(makeReport())
```

얻는 것은 **지연 평가**다.
- `isEnabled`가 `false`면 `makeReport()`는 **호출되지 않음**
- 일반 파라미터 `String`이었다면 `log`를 부르기 전에 무조건 먼저 계산됨

| 파라미터 | 호출부 | 인자가 평가되는 때 |
| --- | --- | --- |
| `String` | `log(makeReport())` | 호출 전에 항상 |
| `() -> String` | `log({ makeReport() })` | 함수 안에서 부를 때 |
| `@autoclosure () -> String` | `log(makeReport())` | 함수 안에서 부를 때 |

표준 라이브러리의 사용처

| API | 감싸는 인자 | 효과 |
| --- | --- | --- |
| `a ?? b` | `b` | `a`가 `nil`일 때만 `b` 계산 |
| `a && b` · `a \|\| b` | `b` | 단락 평가 |
| `assert(조건, 메시지)` | 조건, 메시지 | 릴리스 빌드에서는 계산하지 않음 |

주의
- 호출부만 보면 **즉시 계산되는 값처럼 보임**. 실제로는 안 불리거나 여러 번 불릴 수 있음
- 부수 효과가 있는 표현식을 넘기면 실행 여부를 읽기 어려움
- 남용하면 코드 이해가 어려워진다는 것이 공식 문서의 경고. 지연 평가가 꼭 필요한 곳에만 사용
- 파라미터가 없는 클로저(`() -> T`)에만 붙일 수 있음
- 기본은 non-escaping. 저장하거나 나중에 부르려면 `@autoclosure @escaping`

### 대안과 트레이드오프

비교 대상: **나중에 실행할 동작을 다른 객체 · 함수에 넘겨주는 방법** / 판단 기준: 코드 위치, 이벤트 개수, 수명 관리

| 방법 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| **클로저** | 구현 코드를 따로 만들고 연결할 필요 없이 바로 작성, 문맥 자동 캡처 | 순환 참조 위험, 중첩 시 가독성 저하, 호출 누락을 컴파일러가 못 잡음 |
| [[Delegate 패턴]] | 관련 이벤트 여러 개를 한 묶음으로, `weak` 관례로 수명이 명확 | 보일러플레이트, 처리 코드가 호출 지점과 멀어짐 |
| Target-Action | UIKit 컨트롤과 바로 연결 | 타입 안전성 약함, 문맥 전달 불가 |
| NotificationCenter | **1:N** 가능 | 관리가 어려움, 테스터블하지 않음 |
| async/await ([[Swift_Concurrency]]) | 클로저를 쓰지 않아 가독성이 좋음, 반환 누락을 컴파일러가 잡음 | 함수에 `async`를 붙여야 하고 async 문맥에서만 호출 가능, 여러 번 오는 이벤트에는 부적합 |

- `return`으로 결과를 돌려주는 것은 대안이 아님. 결과가 **즉시** 준비될 때만 성립
- `Result`는 대안이 아니라 콜백에 담는 **내용물** (`(Result<Data, Error>) -> Void`)

**어느 쪽으로**

| 상황 | 선택 | 이유 |
| --- | --- | --- |
| 한 번 끝나는 비동기 작업 | async/await | 위에서 아래로 읽힘 |
| 짧은 일회성 콜백, 고차 함수 | 클로저 | 호출 지점에 로직이 붙음 |
| 진행률 · 완료 · 실패처럼 묶인 이벤트 여러 개 | delegate | 한 프로토콜로 묶음 |
| 서로 모르는 여러 수신자 | NotificationCenter | 1:N |

### 언제 쓰면 안 되는가

**① 순환 참조 — 강한 캡처만으로는 릭이 아님**

릭의 조건은 **서로** 붙잡는 것이다. 클로저가 객체를 강하게 캡처하고, 그 객체가 클로저를 쥐고 있어야 한다.

같은 클로저 `{ print(self.total) }`을 어디에 넣느냐에 따른 결과

| 넣는 곳 | 쥔 쪽 | 결과 |
| --- | --- | --- |
| 지역 상수 `let work = { ... }` | 함수의 실행 구간 | 함수가 끝나면 풀림 |
| 함수 인자 `prices.forEach { ... }` | 실행되는 동안만 | 릭 없음 |
| 남의 객체 `logger.onLog = { ... }` | `logger` | logger가 사라질 때까지 수명 **연장** |
| 자기 프로퍼티 `onChange = { ... }` | 자기 자신 | **순환** |
| 자기가 소유한 객체 `self.button.action = { ... }` | `button` ← 자기 자신 | **순환** |

```
직접   Cart ──▶ 클로저 ──▶ Cart
간접   Cart ──▶ button ──▶ 클로저 ──▶ Cart
```

판별 절차
1. "X가 해제되면 이 객체가 해제된다"를 쓴다 (X = 클로저를 쥔 쪽)
2. "X는 언제 해제되나?"를 묻는다
3. 답이 "이 객체가 해제될 때"면 순환. 이 객체와 무관한 시점이면 순환 아님
4. 답이 또 다른 객체 Y면 Y에 대해 2를 반복

- 단서: X가 클래스 **안에 프로퍼티로 선언**됐는지, **밖에서 만들어져 인자로** 들어왔는지
- 인스턴스를 자기가 관리하면 순환, 밖에서 받았으면 수명 연장
- 해결: 고리 한 곳을 `[weak self]`로 끊음 ([[강한 참조 순환 (Strong Reference Cycle)]])

**② 클로저 중첩**
- 비동기 단계를 클로저로 이으면 들여쓰기와 에러 분기가 늘어 가독성이 떨어짐
- completion 호출 누락 · 중복 호출을 컴파일러가 검사하지 못함

**③ 참조 캡처의 함정**
- 클로저를 만든 뒤 바뀐 값을 실행 시점에 보게 됨
- 만든 시점의 값이 필요하면 캡처 리스트

**④ 동시성**
- 스레드를 넘는 클로저가 가변 상태를 캡처하면 데이터 레이스
- `@Sendable` 검사 대상 ([[Sendable]])

**⑤ 묶인 이벤트가 여러 개**
- 클로저 프로퍼티를 여러 개 두는 것보다 delegate가 나음

### 실전 적용

```swift
final class ProfileScreen {
	let loader = UserLoader()
	var name = ""

	func show(id: Int) {
		loader.load(id: id) { [weak self] user in
			guard let self else { return }
			self.name = user.name
		}
	}
}
```

- `loader`가 완료 클로저를 저장하는 구현이면 `self → loader → 클로저 → self`가 되므로 `[weak self]`
- 저장하지 않고 한 번 호출 후 버리는 구현이면 순환은 아님. 다만 응답까지 화면의 수명이 연장됨
- `guard let self` 이후는 강한 참조. 클로저 실행 구간에만 유효하므로 고리가 되지 않음

### 더 파볼 질문
- 간접 순환 참조를 코드만 보고 바로 짚을 수 있는가 (쥔 쪽을 찾은 뒤 "그 쥔 쪽은 누가 쥐나"까지)
- 캡처된 객체가 풀리는 시점은 클로저 실행이 끝날 때가 아니라 **클로저가 버려질 때**라는 것을 예시로 설명할 수 있는가
- 클로저 값의 구성(함수 포인터 + 문맥)을 자료 없이 말할 수 있는가
- 완료 클로저 기반 API를 async/await으로 감싸는 방법 (Continuation)

### 관련
- [[ARC (Automatic Reference Counting)]] — 문맥 객체와 캡처된 참조의 수명 관리
- [[값 의미론 (Value Semantics)]] — 참조 타입인 클로저와의 대비
- [[Sendable]] — 스레드를 넘는 클로저의 제약
- [[Swift_Concurrency]] — 완료 클로저의 대안
- [[Swift]]

### 출처
- The Swift Programming Language — [Closures](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/)
- The Swift Programming Language — [Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- The Swift Programming Language, Language Reference — [Expressions (Capture Lists)](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/expressions/)
- Apple Developer Documentation — [withoutActuallyEscaping(_:do:)](https://developer.apple.com/documentation/swift/withoutactuallyescaping(_:do:))
- SE-0103 "Make non-escaping closures the default"
- SE-0269 "Increase availability of implicit self in @escaping closures when reference cycles are unlikely to occur"
- SE-0279 "Multiple Trailing Closures"
- SE-0296 "Async/await"
- WWDC21 "Meet async/await in Swift" (Session 10132)
- Swift 저장소 `docs/SIL.rst` (`alloc_box`, `partial_apply`)
