# 메서드 디스패치 (Method Dispatch)

`x.foo()`라는 호출 지점이 **어느 함수 구현으로 이어지는지**를 무엇이, 언제 결정하는가. 문법 층이 아니라 컴파일러·런타임 ABI 층의 문제이고, [[Protocol]]과 class 상속이 각각 다른 기구를 쓴다.

> 이 노트는 `@objc dynamic` / `objc_msgSend`(message dispatch) 경로를 다루지 않는다. 별도 항목으로 분리했다.

### 왜 필요한가

**없으면 다형성이 성립하지 않는다.** 호출 지점마다 "이 함수를 실행하라"를 하나로 못 박아 버리면, `Animal` 타입 변수에 `Dog`를 담아도 항상 `Animal.speak`만 불린다. "무엇이 담겼는지는 실행해 봐야 안다"를 표현할 방법이 사라진다.

Swift는 Objective-C처럼 **모든 호출을 하나의 동적 경로로 보내지 않는다.** 기본값을 정적 쪽에 두고, 동적성이 필요한 자리마다 서로 다른 기구를 배치했다. 특히 프로토콜 지향 프로그래밍은 class 상속과 무관한 다형성이라 vtable을 재사용할 수 없었고, 그래서 witness table이라는 별도 기구가 생겼다.

### 정적·동적을 가르는 축 — 표에 자리가 있는가

가장 헷갈리는 지점이 여기다. **"class면 vtable, protocol이면 witness table"은 정적·동적을 가르는 축이 아니다.** 그건 동적일 때 *어느 표*를 쓸지만 정한다.

|  | 타입 본문 · 프로토콜 요구사항에 선언 | **extension에만 선언** |
| --- | --- | --- |
| **class** | vtable 경유 → 동적 | **정적** |
| **protocol** | witness table 경유 → 동적 | **정적** |

> **표에 자리를 얻은 것만 동적으로 갈 수 있고, extension에만 있는 것은 호출부의 정적 타입으로 결정된다.**

- 가로축(표에 칸이 있는가) → **정적이냐 동적이냐**를 결정
- 세로축(class냐 protocol이냐) → **어느 표**를 쓸지만 결정

`extension`이 자리를 못 얻는 이유는 시점이다. 표는 타입 설계도가 확정되는 순간 한 번 찍히는데, extension은 그 뒤에 — 심지어 다른 모듈에서 — 얼마든지 추가된다. 이미 찍혀 나간 표에 칸을 새로 파려면 그 타입을 쓰는 모든 바이너리를 다시 찍어야 한다.

컴파일러가 이 사실을 직접 말해 준다.

```swift
extension Animal { func greet() { ... } }
class Dog: Animal { override func greet() { ... } }

// error: non-'@objc' instance method 'greet()' is declared in
//        extension of 'Animal' and cannot be overridden
```

**오버라이드라는 개념 자체가 성립하지 않는다 = 슬롯이 없다.** (Swift 6.2.1 실측)

### 내부 동작

#### 표는 어디에 있는가

vtable · witness table · 타입 메타데이터는 컴파일 시점에 만들어져 **바이너리의 읽기 전용 데이터 영역**에 놓인다. 힙에 있는 것은 *인스턴스*이고, 그 인스턴스가 데이터 영역의 표를 가리킬 뿐이다. 동적 디스패치는 힙을 뒤지는 게 아니라 **포인터를 한두 번 따라가는 일**이다.

> 예외는 제네릭 인스턴스(`Box<Int>`)의 메타데이터와 조건부 준수의 witness table로, 런타임이 만들어 캐시한다. 이는 Swift 런타임 구현 세부다.

#### 표를 누가 들고 오는가 — 배달 방식 세 가지

경로를 외우는 대신 **표가 호출 지점까지 배달되는 방식**으로 보면 세 가지뿐이다.

```
① class 인스턴스 — 표가 값에 매달려 온다
   참조 → 인스턴스 헤더 → 타입 메타데이터 → vtable 슬롯 N → 구현

② any P — 표가 값과 함께 포장돼 온다
   [ 값 3워드 ][ Value Witness Table ][ Protocol Witness Table ]
   상자에서 PWT를 꺼냄 → 슬롯 N → 구현

③ 제네릭 <T: P> — 표가 인자로 따로 온다
   소스:  render(x)
   실제:  render(x, T의 메타데이터, T가 P를 만족하는 표)
   값에는 아무것도 안 붙는다. 호출자가 T를 알기에 표를 집어서 같이 건넨다
```

**`any`는 표가 값 *안*에 있고, 제네릭은 표가 값 *밖*(인자)에 있다.** 그래서 `[any P]` 배열은 원소마다 표가 다를 수 있지만, 제네릭 함수 한 번의 호출 안에서는 표가 하나로 고정된다 — 특수화(specialization)가 가능한 이유가 이것이다.

#### 슬롯 번호는 컴파일 타임에 확정된다

vtable에서 일어나는 일은 비교가 아니라 **정해진 번호의 칸을 읽는 것**이다.

```
Animal 의 vtable          Dog 의 vtable
슬롯 0  speak → Animal.speak    슬롯 0  speak → Dog.speak   (오버라이드)
슬롯 1  run   → Animal.run      슬롯 1  run   → Animal.run  (상속분이 복사됨)
```

호출부가 컴파일하는 명령은 "이 참조의 표에서 슬롯 0을 읽어 call" 하나뿐이다. Animal이 담겼든 Dog가 담겼든 명령은 같고, **표의 내용만 다르다.** Dog는 상속받은 슬롯까지 복사된 **자기 표 한 장**을 갖는다.

witness table도 같다. 요구사항마다 칸이 하나씩 있고, 채택 타입이 구현했으면 그 구현이, 안 했으면 **프로토콜 extension의 기본 구현**이 그 칸에 채워진다. 기본 구현이라고 정적이 되는 게 아니다 — **칸이 있으니 여전히 동적이다.**

#### 왜 두 표를 하나로 합칠 수 없는가

**만들어지는 시점과 주인이 다르다.**

```swift
// 모듈 A — 남이 만들어 이미 배포한 바이너리
public struct Foo { ... }        // Foo의 타입 표는 여기서 굳었다

// 모듈 B — 내 앱, 오늘
extension Foo: Drawable { ... }  // 준수는 여기서 선언 → PWT는 내 바이너리에 생긴다
```

vtable은 그 class를 컴파일할 때 만들어지고, 그 순간 메서드 목록이 완결돼 있다. 반면 PWT는 **"X가 P를 준수한다"고 선언한 코드를 컴파일할 때** 만들어지는데, 그 선언은 X를 정의한 파일이 아니어도, X를 만든 사람이 아니어도 된다. 애플이 만든 `String`에 내가 오늘 `extension String: MyProtocol`을 붙일 수 있는데, 이미 iOS 안에 들어 있는 String의 타입 표에 칸을 팔 방법은 없다.

나머지 두 이유도 같은 뿌리다.

- **키가 다르다** — vtable의 키는 *타입*, PWT의 키는 *(타입 × 프로토콜)*. 한 타입이 프로토콜 셋을 채택하면 vtable은 여전히 한 장이지만 PWT는 세 장이다. 프로토콜 집합은 열려 있어 "몇 번 칸부터가 어느 프로토콜 몫"인지 못 박을 수 없다.
- **값 타입에는 vtable 자체가 없다** — struct 값은 필드가 그대로 인라인 저장될 뿐, **값 안에 표로 가는 손잡이가 없다.** 그런데 프로토콜 호출은 struct든 class든 똑같이 통해야 한다. (struct에도 타입 메타데이터는 존재한다. 없는 것은 vtable이고, 핵심은 값이 그 메타데이터를 가리키지 않는다는 점이다.)

#### vtable과 witness table은 선택지가 아니라 순서일 수 있다

```
요구사항 호출 → 상자에서 PWT → 슬롯 → 구현
                                └ 준수한 타입이 non-final class라면
                                  그 슬롯은 thunk를 가리키고, thunk가 다시 vtable을 탄다
```

struct가 채택했다면 **vtable은 존재조차 하지 않는다.** vtable은 PWT의 대안이 아니라 **PWT 다음 단계**로만 등장한다. 이는 [[Opaque Type]]에서 미정리로 남겨 둔 "상자에 실린 두 테이블"의 뒷부분이기도 하다 — VWT는 복사·파괴 같은 생명주기 설명서, PWT는 메서드 호출이 거쳐 가는 주소록이다.

### 대안과 트레이드오프

**비교 대상**: "호출하는 쪽은 구체 타입을 모른 채, 값에 따라 다른 구현이 실행되게 한다"는 문제.
**판단 기준**: 무엇이 자주 늘어나는가(종류냐 동작이냐), 그리고 최적화 여지를 얼마나 남기는가.

| 대안 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| [[Protocol]] 채택 (기준선) | 새 **종류** 추가가 싸다 — 파일 하나 만들어 채택하면 끝. 이종 컬렉션 가능 | 동적 경로 비용, 새 **동작**(요구사항) 추가 시 채택한 모든 타입이 컴파일 에러 |
| enum + switch | 전부 정적 → 인라인·최적화 유지, 힙 할당 없음. 새 case를 추가하면 **빠뜨린 switch를 컴파일러가 잡아준다** | 집합이 닫힌다 — 외부 모듈이 새 종류를 추가할 수 없고, case 하나 추가에 **그 enum을 쓰는 모든 switch를 고쳐야 한다** |
| 클로저를 프로퍼티로 저장 | 표를 직접 조립 → 런타임 교체·테스트 대역 주입이 쉽다 | 결국 간접 호출(손으로 만든 witness table), [[클로저 (Closure)]] 힙 할당과 ARC |
| 제네릭 + 특수화 | 호출자마다 타입이 확정돼 정적·인라인 | 이종 컬렉션 불가, 코드 크기 증가, 모듈 경계에서 특수화 불발 |

프로토콜과 enum은 우열이 아니라 **확장점을 어디에 두느냐**의 선택이다.

```
              새 "종류" 추가      새 "동작" 추가
프로토콜  :   싸다               비싸다
enum      :   비싸다             싸다
```

표준 라이브러리도 이 기준으로 갈라 쓴다 — Optional·Result는 enum(종류 고정, 동작 계속 추가), Collection·Equatable은 프로토콜(동작 고정, 채택 타입 계속 추가).

한 가지 대칭이 더 있다. **컴파일러가 잡아주는 에러를 조용히 없애는 장치**가 양쪽에 하나씩 있다.

| | 무엇을 추가할 때 | 잡아주는 것 | 조용히 넘어가게 만드는 것 |
| --- | --- | --- | --- |
| enum | 새 case | 모든 exhaustive switch가 에러 | `default` 절 |
| 프로토콜 | 새 요구사항 | 채택한 모든 타입이 에러 | extension의 **기본 구현** |

### 언제 쓰면 안 되는가

**동적 경로 쪽**

- 뜨거운 루프 안에서의 동적 호출. 비용의 본체는 간접 점프 몇 사이클이 아니라 **인라이닝을 하지 못해 최적화의 연쇄가 끊어지는 것**이다. 컴파일러는 벽 너머를 못 보니 벽 이쪽의 가정(레지스터에 든 값, 메모리 불변 가정, 루프 불변식)까지 전부 버려야 한다.
- `any P`를 습관적으로 쓰는 것 — 박싱과 타입 정체성 상실이 따라온다. 자세한 비교는 [[Opaque Type]].

**정적으로 굳히는 쪽 — 공짜가 아니다**

- `final`은 상속 가능성을 포기하는 것이고, `private`은 외부 접근을 포기하는 것이다. 성능을 위해 붙이는 게 아니라 **설계상 그래야 할 때** 붙이고 성능은 부산물로 받는다.

**설계상 함정**

- 프로토콜 extension에만 있는 메서드를 채택 타입에서 "재정의"하는 것. 표에 칸이 없으므로 그건 오버라이드가 아니라 **이름만 같은 별개의 메서드**이고, `any P`로 부르면 무시된다. 실제로 동적으로 갈아끼우고 싶다면 **요구사항으로 승격**해야 한다 — 그 순간 표에 칸이 하나 생긴다.

### 실전 적용

**"분명 구현했는데 왜 기본 구현이 불리지?" 진단 절차**

```swift
protocol Drawable { func draw() }              // draw만 요구사항
extension Drawable {
    func draw() { print("default draw") }
    func describe() { print("Drawable.describe") }
}
struct Circle: Drawable {
    func draw() { print("Circle.draw") }
    func describe() { print("Circle.describe") }
}

let d: any Drawable = Circle()
d.draw()      // Circle.draw        — 요구사항 → PWT 슬롯 경유
d.describe()  // Drawable.describe  — 요구사항 아님 → 정적 타입으로 결정
```

1. 그 메서드가 **프로토콜 요구사항에 선언돼 있는지** 본다. 없으면 원인은 그것이다.
2. 호출부의 **정적 타입**을 본다. `Circle`로 부르면 `Circle.describe`, `any Drawable`로 부르면 `Drawable.describe`가 나온다. 같은 값인데 결과가 갈리는 이유가 이것이다.
3. 고치려면 요구사항으로 올린다. 그러면 PWT에 칸이 생기고 그 칸이 `Circle.describe`를 가리킨다.

같은 함정이 class에도 있다. class extension에 정의한 메서드는 오버라이드 자체가 금지되므로, 하위 클래스에서 "재정의했다고 믿는" 코드가 애초에 컴파일되지 않는다.

**성능을 위해 정적으로 되돌릴 때**

컴파일러가 증명해야 하는 명제는 크기나 레이아웃이 아니라 **"이 호출 지점에서 실행될 구현은 유일하다"** 하나다.

| 증명을 가능하게 하는 것 | 어떻게 |
| --- | --- |
| `final` | 상속할 수 없음을 컴파일러에게 알려준다 |
| `private` / `fileprivate` | 그 범위 밖에서 오버라이드될 수 없다 |
| WMO (Whole Module Optimization) | 모듈 전체를 보고 "이 internal class를 상속한 곳이 없다"고 결론 |
| specialization | 제네릭 함수를 구체 타입 전용 사본으로 복제 → 표 인자가 사라진다 |

특수화는 늘 되는 게 아니다. Debug 빌드(`-Onone`), 모듈 경계 너머의 호출(`@inlinable`이 없으면 본문이 모듈 밖으로 나가지 않는다), 타입이 런타임에만 정해지는 경로에서는 **범용 코드 한 벌**이 쓰인다. 즉 특수화 실패는 예외 상황이 아니라 기본 상태이고, 특수화는 최적화가 잘 풀렸을 때 받는 보너스다.

### 더 파볼 질문

- devirtualization이 **원리적으로 불가능해지는 조건** — `open`, library evolution(ABI 안정 라이브러리), 모듈 경계를 넘는 `public` 호출, `[any P]` 순회. 각각 왜 증명이 서지 않는가 (미확인)
- 인라인 **이후**의 최적화 파이프라인 — 상수 전파·죽은 분기 제거·루프 불변식 이동·SROA가 서로를 어떻게 물고 이어지는가 (디스패치가 아니라 컴파일러 최적화 쪽 주제로 분리)
- `@objc dynamic`과 message dispatch — KVO·swizzling·selector가 이 경로를 요구하는 이유
- 타입 메타데이터의 실제 구조 — vtable·프로토콜 준수 레코드가 어디에 어떻게 놓이는가

### 출처

- WWDC 2016 세션 416 "Understanding Swift Performance"
- [Swift ABI — Type Metadata](https://github.com/swiftlang/swift/blob/main/docs/ABI/TypeMetadata.rst)
- [Library Evolution in Swift](https://www.swift.org/blog/library-evolution/)
- 컴파일러 실측: Swift 6.2.1 (class extension 메서드의 `override` 금지, `any P`에서 요구사항/비요구사항 호출 결과 대비)

### 기반 개념

- [[Protocol]] — 계약과 witness table
- [[struct가 상속을 지원하지 않는 이유]] — 저장 축과 디스패치 축을 나누는 지점
- [[값 의미론 (Value Semantics)]] — 값 타입에 vtable이 없는 이유의 뿌리
- [[Opaque Type]] — `some`/`any`의 디스패치 축과 existential 컨테이너
