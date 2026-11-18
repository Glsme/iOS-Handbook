# unowned
참조 타입 인스턴스를 **강한 참조 카운트를 올리지 않고** 가리키되, **대상이 반드시 살아 있다고 가정**하는 참조. 대상이 해제된 뒤에 읽으면 크래시가 난다.

- 타입은 옵셔널이 아님. 언래핑 없이 사용
- `let`으로 선언 가능
- 쓰는 자리: 프로퍼티 · 변수 선언, 클로저 캡처 리스트 `[unowned self]`
- 짝이 되는 수식어는 [[weak]]

### 왜 필요한가

[[weak]]만으로도 순환 참조는 끊을 수 있다. `unowned`는 **대상이 나보다 먼저 죽을 수 없는 관계**를 그대로 표현하기 위해 있다.

`weak`만 있을 때 생기는 문제
- 절대 `nil`이 될 수 없는 관계에도 옵셔널 처리가 붙음
- "없을 수도 있다"는 틀린 정보가 타입에 실림
- 가정이 깨졌을 때 `guard`로 조용히 빠져나가 버그가 숨음

`unowned`가 주는 것
- 옵셔널 없이 `let`으로 선언
- 객체의 메모리가 해제되면 그 객체를 사용할 수 없어야 한다는 가정을 코드에 박음
- 가정이 깨지면 그 자리에서 멈춤
- [[side table]]이 필요 없음

### 내부 동작

객체가 죽은 뒤에 "너 죽었니?"를 누구에게 묻는가. `unowned`는 **객체 자신**에게 묻는다.

```
 [stack]                [heap]
 strongRef ──────────▶ ┌─ object ─────────────────────┐
                       │ metadata                     │
 unownedRef ─────────▶ │ refcount: Strong   Unowned   │  ← 헤더 안에 인라인
                       │ stored properties            │
                       └──────────────────────────────┘
```

- 객체를 직접 가리킴
- Unowned RC는 객체 헤더 안에 Strong RC와 같이 저장됨
- `weak` 참조가 한 번도 없었던 객체라면 side table은 존재하지 않음
- 카운트의 전체 그림은 [[weak#내부 동작]]

**Unowned RC가 붙잡는 것**

| 카운트 | 붙잡는 것 | 0이 되면 |
| --- | --- | --- |
| Strong RC | 객체의 내용 (살아 있는 상태) | `deinit` 실행, 프로퍼티 파괴 |
| Unowned RC | 객체의 메모리 블록 | 객체 메모리 반납 |

- `unowned` 참조는 죽은 뒤에도 객체 헤더를 들여다봐야 하므로 Unowned RC가 메모리 블록을 붙잡음

**해제 흐름**

마지막 strong 참조가 사라졌을 때

```
unowned 참조가 남아 있음
    deinit → 객체 메모리 그대로 남음 (빈 껍데기)
           → unowned 참조가 사라질 때 객체 메모리 반납
```

- Strong 0, Unowned 1인 상태 자체로는 크래시가 나지 않음
- 아무도 읽지 않으면 `unowned` 참조가 사라질 때 조용히 반납됨
- 크래시는 그 상태에서 **읽는 순간**에만 발생

**읽을 때 일어나는 일**

- 객체 헤더에 가서 "Strong이 아직 살아 있나"를 확인
- 살아 있으면 쓰는 동안 **Strong을 +1**
- 이미 `deinit`됐으면 런타임이 의도적으로 트랩

```
Fatal error: Attempted to read an unowned reference but object 0x... was already deallocated
```

- 값이 없어서 우연히 죽는 것이 아니라, 런타임이 검사해서 확정적으로 멈추는 것
- 읽는다고 Unowned RC가 변하지 않음. Unowned RC는 `unowned` 참조 자체가 생기거나 사라질 때만 변함

**죽은 것을 읽었을 때 런타임이 고를 수 있는 길**

| 선택 | 결과 | 채택 |
| --- | --- | --- |
| 껍데기를 그대로 돌려줌 | 파괴된 프로퍼티를 읽음, 데이터 오염 | `unowned(unsafe)` |
| `nil`을 돌려줌 | 안전하게 계속 진행 | 불가능. 타입이 옵셔널이 아님. 이것이 필요하면 [[weak]] |
| 그 자리에서 멈춤 | 크래시 | **`unowned`** |

**빈 껍데기를 남기는 이유**

메모리가 남아 있어서 크래시가 나는 것이 아니다. **안전하게 크래시를 낼 수 있도록** 메모리를 남겨 둔다.

- 런타임은 그 참조가 실제로 읽힐지 미리 알 수 없음
- 껍데기를 치우면 그 자리가 재사용되어 "죽었는지"조차 확인할 수 없음
- `unowned` 참조가 사라질 때 Unowned RC를 -1 하려면 헤더가 있어야 함
- 원칙대로 쓰면 둘이 거의 동시에 정리되므로 껍데기는 잠깐만 존재
- 껍데기가 오래 남는다면 수명 가정이 틀렸다는 신호. 그 자리는 `weak`

**껍데기가 없을 때: use-after-free**

해제된 메모리를 그대로 읽는 것. 결과가 정해져 있지 않은 미정의 동작이다.

| 그 자리의 상태 | 겉으로 보이는 것 |
| --- | --- |
| 아직 재사용 안 됨 | 아무 일 없는 것처럼 보임. 테스트는 통과 |
| 다른 객체가 들어옴 | 틀린 값을 읽거나, 무관한 객체의 데이터를 망가뜨림 |
| 망가진 객체를 나중에 누가 사용 | 원인과 무관한 곳에서 한참 뒤 크래시 |
| 접근 불가 영역 | 즉시 크래시. 운이 좋은 경우 |
| 공격자가 채운 데이터 | 보안 취약점 |

| | 확정 크래시 (`unowned`) | 미정의 동작 (`unowned(unsafe)`) |
| --- | --- | --- |
| 증상 | 매번 같은 자리, 같은 메시지 | 매번 다름, 안 나타나기도 함 |
| 데이터 | 망가지기 전에 멈춤 | 조용히 망가질 수 있음 |
| 디버깅 | 스택 트레이스가 원인을 가리킴 | 원인과 증상이 떨어져 있음 |

- 크래시는 나쁜 결과지만 가장 덜 나쁜 결과
- 배열 범위를 벗어나면 쓰레기 값 대신 크래시를 내는 것과 같은 설계 원칙

> Unowned RC의 위치, 빈 껍데기 유지, 트랩 검사는 Swift 오픈소스 런타임(`RefCount.h`)의 구현 세부다.
> 언어가 보장하는 것은 "해제된 인스턴스를 `unowned`로 접근하면 런타임 에러"까지다.

### 대안과 트레이드오프

비교 대상: **순환 참조를 끊으면서 대상을 가리키는 방법** / 판단 기준: 수명 가정이 틀렸을 때의 결과, 타입 부담, 남는 메모리

| 방법 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| **`unowned`** | 옵셔널 아님, `let` 가능, side table 불필요, 가정 위반 시 즉시 드러남 | 가정이 틀리면 크래시, 죽은 객체의 메모리 블록이 참조가 남은 동안 유지됨 |
| [[weak]] | 대상이 먼저 죽어도 안전하게 `nil` | 옵셔널 처리, 작업이 조용히 누락될 수 있음, side table 비용 |
| `unowned(unsafe)` | 카운트 · 검사 비용 없음 | 해제된 메모리를 읽는 미정의 동작 |
| 강한 참조 + 바깥 사건에서 수동으로 끊기 | 끊는 지점이 코드에 보임 | 놓치면 그대로 누수 |

`weak`/`unowned` 없이 푸는 방법은 [[weak#대안과 트레이드오프]] 참고.

### weak과 unowned 중 고르기

**대상이 나보다 먼저 죽을 수 있는가**로 고른다.

| 질문 | 답 | 선택 |
| --- | --- | --- |
| 대상이 나보다 먼저 죽을 수 있는가 | 그렇다 / 모르겠다 | [[weak]] |
| | 아니다. 같이 죽거나 대상이 더 오래 삶 | `unowned` |

| | `weak` | `unowned` |
| --- | --- | --- |
| 타입 | 옵셔널 | 옵셔널 아님 |
| 누구에게 묻나 | side table | 객체 자신 |
| 죽은 뒤 남는 것 | 작은 side table | 객체 메모리 블록 (빈 껍데기) |
| 죽은 것을 읽으면 | `nil` | 크래시 |
| 잘못 골랐을 때 | 크래시 없이 아무 일도 일어나지 않음 | 즉시 크래시 |

- 생명 주기를 명확하게 알 때만 `unowned`. 애매하면 `weak`
- TSPL의 클로저 기준
  - 클로저와 인스턴스가 항상 서로를 참조하고 항상 함께 해제되면 `unowned`
  - 캡처한 참조가 언젠가 `nil`이 될 수 있으면 `weak`

### 언제 쓰면 안 되는가

- **대상이 나보다 먼저 죽을 수 있을 때**
  - 네트워크 응답 콜백, 지연 실행처럼 실행 시점을 내가 정하지 못하는 비동기 경로
  - 화면이 닫힌 뒤에 콜백이 도착하면 크래시
- **생명 주기를 모르는 상태에서** 옵셔널 처리가 귀찮다는 이유로 쓸 때
- **죽은 대상을 오래 가리킬 때**. 참조가 남은 동안 대상의 메모리 블록이 반납되지 않음
- 값 타입. 참조 카운트 자체가 없음

`unowned var x: T?`의 `?`는 "애초에 없을 수도 있다"는 뜻이다. 해제되면 `nil`이 된다는 뜻이 아니다.

### 실전 적용

**소유 관계가 한 방향으로 묶인 두 객체**

```swift
final class Customer {
	var card: CreditCard?
}

final class CreditCard {
	unowned let customer: Customer
	init(customer: Customer) { self.customer = customer }
}
```

- 카드는 고객 없이 존재할 수 없음. 소유권이 명확하고 카드는 고객이 살아 있을 때만 동작
- `weak`이었다면 `customer`가 옵셔널 `var`가 되어, 있을 수 없는 "주인 없는 카드"가 타입에 생김

**자기 프로퍼티로 저장한 클로저**

```swift
final class Element {
	let name = "p"
	lazy var asHTML: () -> String = { [unowned self] in
		"<\(self.name)>"
	}
}
```

- 클로저가 `self`의 프로퍼티. `self`가 죽으면 클로저도 같이 죽음
- 클로저가 `self`보다 오래 살 수 없으므로 `unowned`가 안전
- 이 클로저를 밖으로 꺼내 다른 곳에 저장하면 가정이 깨짐

### 더 파볼 질문
- `unowned`를 쓸 자리를 실제 코드에서 짚고 예시를 직접 쓸 수 있는가
- Strong 0 · Unowned 1 상태에서 "읽지 않으면 크래시가 나지 않는다"를 자료 없이 설명할 수 있는가
- `unowned(unsafe)`와 `Unmanaged`가 실제로 쓰이는 곳
- 빈 껍데기 구간을 메모리 그래프 디버거로 관찰할 수 있는가

### 관련
- [[weak]] — 같은 문제의 다른 해법. 세 카운트와 side table의 전체 그림
- [[ARC (Automatic Reference Counting)]] — 카운트가 저장되는 헤더, retain/release
- [[강한 참조 순환 (Strong Reference Cycle)]] — `unowned`가 푸는 문제
- [[클로저 (Closure)]] — 캡처 리스트
- [[side table]]
- [[Swift]]

### 출처
- The Swift Programming Language — [Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/) (Unowned References, Unowned Optional References, `unowned(unsafe)`)
- WWDC21 "ARC in Swift: Basics and beyond" (Session 10216)
- Swift 저장소 `stdlib/public/SwiftShims/RefCount.h` (객체 수명 상태, Unowned RC)
- Mike Ash, "Friday Q&A 2017-09-22: Swift 4 Weak References"
