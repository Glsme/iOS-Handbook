# weak
참조 타입 인스턴스를 **강한 참조 카운트를 올리지 않고** 가리키는 참조. 대상이 해제된 뒤에 읽으면 `nil`이 나온다.

- 타입은 반드시 옵셔널 ([[Optional (옵셔널)]])
- 클래스 타입과 클래스 전용 프로토콜(`AnyObject`)에만 붙음. struct · 클로저 타입에 붙이면 컴파일 에러
- 쓰는 자리: 프로퍼티 · 변수 선언, 클로저 캡처 리스트 `[weak self]`
- 짝이 되는 수식어는 [[unowned]]. 둘 중 고르는 기준은 [[unowned#weak과 unowned 중 고르기]]

### 왜 필요한가

[[ARC (Automatic Reference Counting)]]는 강한 참조 수만 센다. **소유하지 않고 가리키기만 하는 관계**를 표현할 수단이 없으면 순환 참조에서 메모리 릭이 발생한다. ([[강한 참조 순환 (Strong Reference Cycle)]])

고리가 닫히는 대표 구조
- **delegate**: 부모가 자식을 소유하고, 자식이 `delegate`로 부모를 다시 잡음
- **저장된 escaping 클로저**: `self → 클로저 프로퍼티 → self` ([[클로저 (Closure)]])
- **부모–자식 역참조**: 자식이 `parent` 프로퍼티로 부모를 잡음

`weak`이 주는 것
- 카운트를 올리지 않으므로 고리가 닫히지 않음
- "대상이 나보다 먼저 사라질 수 있다"는 사실이 옵셔널 타입으로 드러남

### 내부 동작

핵심 질문은 하나다. **객체가 죽은 뒤에 "너 죽었니?"를 누구에게 묻는가.**

- 죽은 객체에게는 물을 수 없음. 메모리가 반납되면 그 자리에 쓰레기 값이나 다른 객체가 들어옴
- `weak`은 객체가 아니라 객체 밖의 장부인 **[[side table]]** 에게 물음

```
 [stack]                [heap]
 strongRef ──────────▶ ┌─ object ─────────────────────┐
                       │ metadata                     │
                       │ refcount: ───────────────────┼──┐  카운트 대신 주소
                       │ stored properties            │  │
                       └──────────────────────────────┘  │
                                   ▲                     ▼
                                   │   ┌─ side table (별도 힙 블록) ────┐
                                   └───┤ object pointer                 │
 weakRef ─────────────────────────────▶│ Strong   Unowned   Weak        │
                                       └────────────────────────────────┘
```

- side table은 객체와 **별개의 힙 할당**. 객체 메모리가 반납된 뒤에도 혼자 남을 수 있음
- 첫 `weak` 참조가 생기는 순간 만들어짐
- 그때 헤더에 있던 Strong · Unowned 카운트가 side table로 옮겨가고, 헤더 자리에는 side table 주소가 들어감
- `weak` 변수는 객체가 아니라 side table을 가리킴

**세 카운트가 붙잡는 것**

각 참조는 **자기가 들여다볼 곳**을 붙잡는다.

| 카운트 | 붙잡는 것 | 0이 되면 |
| --- | --- | --- |
| Strong RC | 객체의 내용 (살아 있는 상태) | `deinit` 실행, 프로퍼티 파괴 |
| Unowned RC | 객체의 메모리 블록 | 객체 메모리 반납 |
| Weak RC | side table | side table 반납 |

- `weak` 참조는 죽은 뒤에도 side table을 들여다봐야 하므로 Weak RC가 side table을 붙잡음
- Weak RC가 아무것도 막지 않는다면 `weak` 참조가 남아 있는데 side table이 사라져, 물어볼 곳이 없어짐

**읽을 때 일어나는 일**

- side table에 가서 "Strong이 아직 살아 있나"를 확인
- 살아 있으면 **Strong을 +1** 한 강한 참조를 돌려줌
- 죽었으면 `nil`
- 읽는다고 Weak RC가 변하지 않음. Weak RC는 `weak` 참조 자체가 생기거나 사라질 때만 변함
- 런타임이 `weak` 변수들을 찾아다니며 `nil`로 바꾸는 것이 아님. **읽을 때 판정**

**해제 흐름**

마지막 strong 참조가 사라졌을 때

```
strong만 있었음       deinit → 객체 메모리 즉시 반납
weak이 남아 있음      deinit → 객체 메모리 즉시 반납, side table만 남음
                             → weak 참조가 사라질 때 side table 반납
```

- 죽은 뒤 남는 것은 **작은 side table뿐**. 객체 블록 전체가 남는 [[unowned]]와 다른 점
- 순서가 지켜지는 이유는 연쇄 잠금
  - strong 참조 전체가 묶어서 Unowned RC +1
  - unowned 참조 전체가 묶어서 Weak RC +1
  - 앞 단계가 끝나야 다음 단계의 +1이 풀림

**숫자로 따라가기**

```swift
var a: Obj? = Obj()
unowned let u = a!
weak var w = a
```

| 사건 | Strong | Unowned | Weak | 일어나는 일 |
| --- | --- | --- | --- | --- |
| `a` 생성 | 1 | 1 | 없음 | 헤더에 인라인. Unowned 1은 strong 전체 몫 |
| `u` 생성 | 1 | 2 | 없음 | `u` 몫 +1 |
| `w` 생성 | 1 | 2 | 2 | side table 할당. Weak 2는 unowned 전체 몫과 `w` 몫 |
| `a = nil` | 0 | 1 | 2 | `deinit` 실행. strong 전체 몫이던 Unowned +1 반납 |
| `u` 소멸 | 0 | 0 | 1 | 객체 메모리 반납. unowned 전체 몫이던 Weak +1 반납 |
| `w` 소멸 | 0 | 0 | 0 | side table 반납 |

**Weak RC만 객체 밖에 있는 이유**

- 객체 메모리는 `deinit` 직후 반납됨
- 그 뒤에도 `weak` 참조는 "죽었다"는 답을 들어야 함
- 장부가 객체 헤더 안에 있으면 객체와 함께 사라져 읽을 수 없음

**side table을 처음부터 만들지 않는 이유**

- 만드는 것 자체가 비용. 힙 할당 한 번, 유지하는 동안의 메모리
- 카운트를 만질 때마다 헤더에서 side table로 한 단계 더 거침
- 대부분의 객체는 `weak`으로 잡히지 않음 → `weak`으로 잡히는 객체만 비용을 냄

> 카운트 배치, side table 구조, 연쇄 +1 규칙은 Swift 오픈소스 런타임(`RefCount.h`)의 구현 세부다.
> 공개 API가 아니므로 버전에 따라 달라질 수 있다. 표의 숫자는 논리적인 값이고 실제 비트 인코딩은 다르다.

> Swift 3까지는 side table이 없었다. `weak` 참조가 객체를 직접 가리켰고, Strong이 0이 되어도
> `weak` 참조가 남아 있으면 객체 메모리를 반납하지 못해 좀비 오브젝트로 남았다. 남은 `weak` 참조가
> 읽히거나 사라져 카운트가 0이 되어야 반납됐다. 지금의 `unowned`와 같은 구조다. 큰 객체가 오래
> 남는 낭비와 동시 읽기 경쟁 상태를 해결하려고 Swift 4에서 side table이 도입됐다. (Mike Ash의 글 기준)

### 대안과 트레이드오프

비교 대상: **순환 참조로 생기는 메모리 릭을 푸는 방법** / 판단 기준: 누가 끊는가, 실수했을 때 어떻게 되는가

| 방법 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| **`weak`** | 고리가 닫히지 않음, 대상이 먼저 죽어도 안전 | 옵셔널 처리, 작업이 조용히 누락될 수 있음, side table 비용 |
| [[unowned]] | 옵셔널 아님, `let` 가능, side table 불필요 | 수명 가정이 틀리면 크래시 |
| 바깥 사건에서 수동으로 끊기 (`nil` 대입, `invalidate()`) | 끊는 지점이 코드에 보임 | 휴먼 에러로 놓치면 그대로 누수 |
| `self` 대신 필요한 값만 캡처 | 고리가 생기지 않음 | 나중에 값이 바뀌어도 클로저 안의 값은 그대로 |
| 소유 구조 재설계 (값 타입, 단방향 소유) | 수식어도 정리 코드도 필요 없음 | 설계 비용 |
| 추적형 GC (언어 차원) | 순환을 자동 회수 | 해제 시점이 비결정적, 일시 정지 |

- `nil`을 대입하는 것 자체가 카운트 -1. MRC가 아니어도 참조를 없애면 ARC가 카운트를 내림
- 수동으로 끊는 코드는 **`deinit` 안에 넣으면 실행되지 않음**
  - `deinit`은 Strong RC가 0이 되어야 실행됨
  - 고리가 있는 한 0이 되지 않음
  - 끊는 코드는 바깥 사건(화면 닫힘, 작업 완료, 취소)에 둬야 함
- 수동 끊기의 실제 사례: `Timer.invalidate()`, `URLSession.invalidateAndCancel()`

```swift
let id = self.id
onTap = { print(id) }     // self를 잡지 않음 → 고리 없음
```

### 언제 쓰면 안 되는가

**쓰면 안 됨**
- 그 객체를 참조하는 곳이 `weak`뿐일 때. 만들자마자 해제됨

```swift
weak var cache = Cache()   // 경고: 즉시 해제됨. cache는 nil
```

- 반드시 끝나야 하는 작업(저장, 로깅)을 `[weak self]` 뒤에 둘 때. `self`가 먼저 사라지면 작업이 누락됨

**쓸 필요 없음**
- 순환 참조가 생기지 않을 때
  - 즉시 실행되고 끝나는 non-escaping 클로저 (`map` · `forEach`)
  - `self`가 그 클로저를 저장하지 않을 때. 순환이 아니라 수명 연장일 뿐
- 값 타입. 참조 카운트 자체가 없음

**실패 양상**
- 크래시 없이 **조용히 아무 일도 일어나지 않음**
- 크래시 로그가 남지 않아 누락을 알아채기 어려움
- [[unowned]]의 실패는 반대로 즉시 크래시

### 실전 적용

**delegate**

```swift
protocol LoaderDelegate: AnyObject {
	func didLoad()
}

final class Loader {
	weak var delegate: LoaderDelegate?
}
```

- 프로토콜에 `AnyObject` 제약이 있어야 `weak`을 붙일 수 있음
- 소유 방향은 부모 → 자식 하나만 남김

**비동기 클로저 캡처**

```swift
loader.load { [weak self] result in
	guard let self else { return }
	self.render(result)
}
```

- 클로저가 실행될 때 화면 인스턴스가 이미 해제됐을 수 있으므로 `weak`
- `guard let self`는 `self`를 지역 강한 참조로 승격. 그 스코프 동안 Strong +1, 클로저가 끝나면 반납

**`self?.` 와 `guard let self`의 차이**

```swift
{ [weak self] in
	self?.stepOne()
	self?.stepTwo()        // stepOne만 실행되고 여기는 건너뛸 수 있음
}

{ [weak self] in
	guard let self else { return }
	self.stepOne()
	self.stepTwo()         // 둘 다 실행되거나 둘 다 실행되지 않음
}
```

- `self?.`는 줄마다 따로 확인. 두 줄 사이에 `self`가 해제될 수 있음
- 단계들이 한 묶음이어야 하면 `guard let self`

**캡처 방식 고르기**

| 코드 | 선택 | 이유 |
| --- | --- | --- |
| `users.map { self.format($0) }` | 강한 참조 | 메서드 안에서 즉시 실행되고 끝남. 생명 주기가 확실 |
| `api.fetchProfile { self.render($0) }` | `weak` | 응답 시점에 화면 인스턴스가 해제됐을 수 있음 |

### 더 파볼 질문
- 읽을 때 변하는 카운트는 Strong뿐이라는 것을 자료 없이 설명할 수 있는가 (Unowned · Weak RC는 참조가 생기고 사라질 때만 변함)
- side table의 정확한 구조와 헤더 비트 배치 ([[side table]])
- `weak let`(SE-0481)은 어느 버전부터 쓸 수 있고, 왜 필요해졌는가
- Objective-C의 전역 weak 테이블 방식과 Swift side table 방식의 비용 차이
- `Timer`의 target 기반 API와 블록 기반 API에서 고리가 생기는 위치

### 관련
- [[unowned]] — 같은 문제의 다른 해법. 선택 기준 포함
- [[ARC (Automatic Reference Counting)]] — 카운트가 저장되는 헤더, retain/release
- [[강한 참조 순환 (Strong Reference Cycle)]] — `weak`이 푸는 문제
- [[클로저 (Closure)]] — 캡처 리스트, `@escaping`, 순환 판별 절차
- [[side table]] — Weak RC가 사는 곳
- [[Delegate 패턴]] — 대표 사용처
- [[Optional (옵셔널)]]
- [[Swift]]

### 출처
- The Swift Programming Language — [Automatic Reference Counting](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/)
- WWDC21 "ARC in Swift: Basics and beyond" (Session 10216)
- Swift 저장소 `stdlib/public/SwiftShims/RefCount.h` (객체 수명 상태, 세 카운트, side table)
- Mike Ash, "Friday Q&A 2015-12-11: Swift Weak References" (Swift 3 이전 구현)
- Mike Ash, "Friday Q&A 2017-09-22: Swift 4 Weak References" (side table 도입)
- Apple Developer Documentation — [URLSession](https://developer.apple.com/documentation/foundation/urlsession) (delegate를 무효화 시점까지 강하게 유지)
