# side table
Swift 런타임이 힙 객체 하나에 대해 **객체 밖에 따로 할당하는 작은 장부**. 참조 카운트 세 개와 객체 포인터를 담고, [[weak]] 참조가 "객체가 살아 있는가"를 묻는 곳

- 언어 문법이 아니라 런타임의 참조 카운팅 구현 세부. 코드에서 직접 만질 수 없음
- Swift 4에서 도입
- `weak` 참조 쪽에서 본 동작은 [[weak#내부 동작]]

### 왜 필요한가

`weak` 참조는 객체가 죽은 뒤에도 "죽었다"는 답을 들어야 한다.

장부가 객체 헤더 안에 있을 때
- Strong RC · Unowned RC가 0이어도 Weak RC가 남아 있으면 객체 메모리 블록을 반납할 수 없음
- 내용은 죽었는데 메모리만 남는 **Zombie Object** 현상
- Swift 3 이전이 이 방식

장부를 객체 밖으로 뺐을 때
- 객체 메모리는 `deinit` 직후 반납
- 죽은 뒤 남는 것은 작은 장부 하나

**이유는 자리가 아니라 수명**
- 헤더를 넓혀 Weak RC 자리를 만들어도 해결되지 않음
- 헤더는 객체 메모리의 일부라 객체와 같이 사라짐
- 판정 정보는 객체보다 오래 살아야 하므로 별도 블록이어야 함

### 내부 동작

**구조**

```
 strongRef ──────────▶ ┌─ object ─────────────────────┐
 unownedRef ─────────▶ │ metadata pointer             │
                       │ refcount 칸 [flag=1]         │
                       │   side table 주소 ────────────┼──┐
                       │ stored properties            │  │
                       └──────────────────────────────┘  │
                                   ▲                     ▼
                                   │   ┌─ side table (별도 힙 블록) ───────┐
                                   └───┤ object pointer                 │
 weakRef ─────────────────────────────▶│ Strong   Unowned   Weak        │
                                       └────────────────────────────────┘
```

- strong · [[unowned]] 변수는 객체를 직접 가리킴. side table이 생겨도 그대로
- `weak` 변수만 side table을 가리킴
- 객체와 side table은 서로를 가리킴

**만들어지는 때**
- 첫 `weak` 참조가 생길 때
- 인라인 카운트가 헤더 칸을 넘칠 때 (Strong 또는 Unowned 오버플로)
- 64비트 기기에서 오버플로는 한 객체에 강한 참조 약 10억 개가 필요. 사실상 `weak` 때문에만 생김

**헤더 칸 하나를 두 가지로 읽는 방법**
- 헤더의 refcount 칸은 64비트 한 칸
- 그 안의 **flag 비트 하나**가 구분자
- flag 꺼짐: 나머지 비트는 Strong · Unowned 카운트 (인라인)
- flag 켜짐: 나머지 비트는 side table 주소

**카운트를 통째로 옮기는 이유**
- `weak` 읽기가 확인하는 것은 Strong 쪽 상태
- Strong이 객체 헤더에 있으면 확인하러 객체 메모리에 가야 함
- 객체 메모리는 이미 반납됐을 수 있음
- 그래서 Strong도 side table에 있어야 함
- 카운트가 두 곳에 나뉘면 "확인하고 +1"을 한 번에 할 수 없으므로 전부 한 칸에 모음

**`weak` 변수에 하는 네 가지 일**

| 동작 | 코드 | 변하는 카운트 |
| --- | --- | --- |
| 만들기 | `weak var w = a` | Weak +1. 첫 `weak`이면 side table 할당 |
| 읽기 | `w?.speak()`, `if let p = w` | 살아 있으면 Strong +1, 다 쓰면 -1 |
| 쓰기 | `w = other`, `w = nil` | 이전 대상 Weak -1, 새 대상 Weak +1 |
| 없애기 | 스코프 종료, 담고 있던 객체 해제 | Weak -1 |

- 읽기에서 Weak RC는 변하지 않음
- Weak RC가 하는 일은 하나. **side table을 언제 반납할지** 정하는 것

**읽기 경로**

```
w 읽기 → side table의 Strong 쪽 상태 확인
           ├─ 살아 있음 → Strong +1 → object pointer를 따라 객체 반환
           └─ 죽었음   → nil 반환. object pointer는 쓰지 않음
```

- 죽은 경우 객체 메모리에는 가지 않음. 판정이 side table에서 끝남
- Strong을 올리는 이유: 받은 참조를 쓰는 동안 객체가 죽으면 안 되고, 객체를 살려 두는 것은 Strong뿐
- 죽은 뒤에도 side table을 읽을 수 있는 이유: 그 `weak` 변수 자신이 Weak RC 1을 들고 있음

**기본 몫**

각 카운트는 "그 대상을 쓰고 있는 쪽의 수"다. 참조 변수 말고도 쓰는 쪽이 하나씩 더 있다.

```
Strong  = strong 변수 수          (기본 몫 없음)
Unowned = unowned 변수 수 + 1     (객체가 살아 있는 동안)
Weak    = weak 변수 수    + 1     (객체 메모리 블록이 남아 있는 동안)
```

| 카운트 | 기본 1의 주인 | 그 1이 빠지는 때 | 없으면 |
| --- | --- | --- | --- |
| Strong | 없음 | - | - |
| Unowned | 살아 있는 객체 자신 | `deinit`이 끝날 때 | 생성과 동시에 메모리 블록이 반납 대상이 됨 |
| Weak | 객체 메모리 블록 | 메모리 블록이 반납될 때 | `weak` 변수가 사라지는 순간 side table이 반납됨 |

- 객체도 side table의 사용자. 자기 Strong · Unowned 카운트를 거기에 보관 중
- 그래서 `weak` 참조가 전부 사라져도 객체 자신이 side table을 참조하고 있어 반납되지 않음
- 한 번 side table을 가진 객체는 **인라인으로 돌아가지 않음**
- Strong에 기본 몫이 없는 이유: 빼 줄 윗 단계 사건이 없음. 있으면 Strong이 0이 되지 않아 `deinit`이 돌지 않음
- 기본 몫은 변수를 지운다고 빠지지 않음. 위 표의 사건이 일어나야 빠짐

**숫자로 따라가기**

강한 참조 `a` · `b`, weak 참조 `w1` · `w2`. unowned 참조 없음.

| 사건 | Strong | Unowned | Weak | 반납 |
| --- | --- | --- | --- | --- |
| 시작 | 2 | 0 + 1 = 1 | 2 + 1 = 3 | |
| `w1 = nil` | 2 | 1 | 2 | 없음 |
| `a = nil` | 1 | 1 | 2 | 없음 |
| `b = nil` | 0 | 0 | 1 | 객체 메모리 |
| `w2 = nil` | 0 | 0 | 0 | side table |

- `b = nil` 한 줄에서 연쇄 발생
  - Strong 0 → `deinit` 끝 → Unowned 기본 몫 빠짐
  - Unowned 0 → 메모리 블록 반납 → Weak 기본 몫 빠짐
  - Weak 2 → 1. 남은 1은 `w2` 몫
- unowned 참조까지 섞인 예는 [[weak#내부 동작]]의 표

**읽기와 해제가 동시에 일어날 때**

Strong 1인 상태에서 스레드 A는 `weak` 읽기, 스레드 B는 마지막 강한 참조 해제.

| 먼저 성공한 쪽 | Strong | A가 받는 것 | 객체 |
| --- | --- | --- | --- |
| A (읽기) | 1 → 2, 이후 B가 놓아 1 | 강한 참조 | 살아 있음 |
| B (해제) | 1 → 0 | `nil` | `deinit` 시작 |

- "확인"과 "+1"이 따로면, 그 사이에 B가 끼어들어 죽고 있는 객체를 받게 됨
- 런타임은 락으로 줄을 세우지 않음
- 확인과 +1을 **CPU 명령 하나로 묶음** (원자적 연산, compare-and-swap)
- 진 쪽은 실패를 통보받고 다시 읽음

> 카운트 배치, flag 비트, 기본 몫 규칙은 Swift 오픈소스 런타임(`RefCount.h`)의 구현 세부다.
> 공개 API가 아니므로 버전에 따라 달라질 수 있다. 표의 숫자는 논리적인 값이고 실제 비트 인코딩은 다르다.
> 오버플로 기준(약 10억)은 64비트 헤더의 Strong 비트 수에서 나온 대략적인 값이다.

### 대안과 트레이드오프

비교 대상: **객체가 죽은 뒤에도 `weak` 참조가 안전하게 `nil`을 판정하는 방법** / 판단 기준: 판정 정보를 어디에 두는가, 누가 언제 일하는가

| 방식 | 판정 방법 | 얻는 것 | 잃는 것 |
| --- | --- | --- | --- |
| **side table** (Swift 4+) | 객체 밖 장부에 읽을 때 물음 | 객체 메모리 즉시 반납, 죽은 뒤 남는 것은 작은 블록 하나 | 추가 힙 할당 1회, 이후 모든 카운트 연산이 한 단계 더 거침 |
| 헤더에 Weak RC (Swift 3 이전) | `deinit` 뒤에도 객체 메모리를 남겨 헤더에 물음 | 추가 할당 없음, 간접 참조 없음 | `weak` 참조가 남는 한 객체 메모리 전체가 반납되지 않음 |
| 전역 weak 테이블 (Objective-C `__weak`) | 객체가 죽을 때 런타임이 `weak` 변수들을 찾아가 `nil`로 바꿈 | 객체 메모리 즉시 반납, 죽은 뒤 남는 장부 없음 | `weak` 변수 조작마다 전역 테이블 조회와 락, 해제 비용이 `weak` 참조 수에 비례 |
| [[unowned]] | 판정하지 않음. 죽은 객체 접근 시 트랩 | 옵셔널 아님, side table 불필요 | `nil` 판정 없음, 참조가 남는 동안 메모리 블록이 남음 |

- Objective-C: 객체가 **죽을 때** 런타임이 찾아다니며 지움
- Swift: 아무도 찾아다니지 않음. **읽을 때** 장부를 보고 판정

### 언제 쓰면 안 되는가

side table은 직접 고르는 것이 아니라 `weak`을 쓰면 따라온다. "side table 비용을 내지 말아야 할 때"로 본다.

**side table이 생긴 객체가 내는 비용**
- 한 번: 힙 할당
- 계속: side table이 차지하는 메모리
- 계속: retain/release마다 한 단계 추가. 헤더 읽기 → flag 확인 → side table로 이동 → 카운트 수정
- `weak` 참조가 전부 사라져도 객체가 살아 있는 한 사라지지 않음
- 객체가 죽은 뒤에도 마지막 `weak` 참조가 사라질 때까지 side table 메모리는 남음

**피할 만한 경우**
- 수가 많고 retain/release가 잦은 객체를 습관적으로 `weak`으로 잡는 것
- 대상이 나보다 먼저 죽을 수 없는 관계. [[unowned]]면 side table이 생기지 않음
- 순환이 생기지 않는 관계. `weak` 자체가 필요 없음 ([[weak#언제 쓰면 안 되는가]])

**side table이 해결하지 않는 것**
- `weak` 읽기는 매번 원자적 Strong +1과 -1
- `nil` 가능성은 그대로. 옵셔널 처리 필요

> 위 비용은 구조상 존재하는 비용이다. 실제 크기는 측정하지 않았다.

### 실전 적용

**side table을 만드는 코드**

```swift
final class Loader {
	weak var delegate: LoaderDelegate?      // delegate로 지정된 객체에 side table
}

loader.load { [weak self] result in         // self에 side table
	guard let self else { return }
	self.render(result)
}
```

**읽기 횟수 줄이기**

```swift
{ [weak self] in
	self?.stepOne()                // 읽기 절차 1회
	self?.stepTwo()                // 읽기 절차 1회
}

{ [weak self] in
	guard let self else { return } // 읽기 절차 1회
	self.stepOne()
	self.stepTwo()
}
```

- `self?.`는 줄마다 side table 확인 → Strong +1 → 호출 → Strong -1
- `guard let self`는 한 번만 확인하고 스코프 끝까지 강한 참조로 유지

### 더 파볼 질문
- 기본 몫 공식을 자료 없이 다시 쓸 수 있는가. 특히 Weak 쪽
- compare-and-swap이 실패했을 때 재시도하는 구조
- 헤더 refcount 칸의 정확한 비트 배치
- Objective-C 전역 weak 테이블의 실제 비용

### 관련
- [[weak]] — side table을 만드는 참조. 읽기와 해제 흐름
- [[unowned]] — side table 없이 객체 헤더에 묻는 참조
- [[ARC (Automatic Reference Counting)]] — 카운트가 저장되는 헤더, retain/release
- [[힙과 스택]] — side table이 별도 힙 블록인 이유
- [[강한 참조 순환 (Strong Reference Cycle)]] — `weak`이 푸는 문제
- [[Swift]]

### 출처
- Swift 저장소 `stdlib/public/SwiftShims/RefCount.h` (객체 수명 상태, 세 카운트, side table, flag 비트)
- Swift 런타임 함수 `swift_weakInit` · `swift_weakLoadStrong` · `swift_weakAssign` · `swift_weakDestroy`
- Mike Ash, "Friday Q&A 2017-09-22: Swift 4 Weak References" (side table 도입, Swift 3 방식과의 비교)
- Objective-C 런타임 저장소 `objc-weak.h` (전역 weak 테이블)
