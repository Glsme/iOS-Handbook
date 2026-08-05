# Identity
SwiftUI가 "이 뷰가 이전의 그 뷰와 같은 것인가"를 판단하는 개념

### 핵심 개념
- **Identity = "누구냐"**, [[Diffing]] = "뭐가 달라졌냐" — 서로 다른 층
- Identity가 먼저 판정되고, **같을 때만** 값 비교로 넘어감
	- 같다 → 상태 유지 + [[Diffing]]으로 달라진 부분만 갱신
	- 다르다 → 옛 뷰 **파괴** + 새 뷰 **생성** (`@State` 초기화, transition 발생)
- 뷰가 스스로 들고 있는 값이 아니라 **SwiftUI가 뷰 트리를 보고 부여**하는 것

### View는 [[Identifiable]]을 채택하지 않음
```swift
public protocol View {
	associatedtype Body: View
	@ViewBuilder var body: Self.Body { get }
}
```
`id` 프로퍼티가 없다. [[Identifiable]]은 **모델(데이터)** 에 붙는 프로토콜이지 View의 것이 아니다.

### Identity가 생기는 두 경로

#### 1. 구조적 identity (암묵적, 기본값)
뷰 트리의 **위치와 타입**으로 결정. 개발자가 아무것도 안 해도 붙는다.

```swift
// identity 2개 — 서로 다른 뷰로 취급 (@State 초기화, transition 발생)
if isOn {
	Text("A")
} else {
	Text("B")
}

// identity 1개 — 같은 Text의 내용만 바뀜
Text(isOn ? "A" : "B")
```

`if/else`는 `_ConditionalContent<A, B>`라는 **다른 타입**을 만들어 두 분기가 별개 identity가 된다.
삼항 연산자는 `Text` 하나라 identity가 유지된다.

#### 2. 명시적 identity
- `.id(_:)` — `Hashable` 값을 넘겨 직접 지정
	- 이 값이 바뀌면 완전히 다른 뷰로 보고 새로 만듦
	- `@State`를 강제로 리셋하는 트릭이자, 의도치 않게 상태가 날아가는 흔한 버그 원인
- `ForEach`의 `id:` 파라미터

```swift
// 모델이 Identifiable이면 id: 생략 가능
ForEach(items) { item in ... }

// 아니면 KeyPath로 지정
ForEach(items, id: \.avoidNo) { item in ... }
ForEach(Array(items.enumerated()), id: \.offset) { _, item in ... }
```

### 두 identity는 우선순위가 아니라 스코프 관계다
```
최종 identity ≈ (구조적 경로) + (그 자리에 얹힌 explicit id)
```

`.id()`는 구조적 위치를 **대체하지 않고 그 안에 얹힌다.**
그래서 위치가 갈리면 뒤에 같은 id를 붙여도 앞부분이 이미 다르다.

```
[_ConditionalContent.true]  + id("same")   ← 앞부분이 다름
[_ConditionalContent.false] + id("same")
```

`ForEach`가 예외처럼 보이는 건 explicit이 structural을 이겨서가 아니라,
**`ForEach`가 자기 스코프 안에서 id를 매칭해주는 컨테이너**이기 때문이다.
그 매칭 주체가 없는 `if/else`에서는 비교 자체가 일어나지 않는다.

| 구조적 위치 | explicit id | 컨테이너 | 결과 |
|---|---|---|---|
| 같음 | **바뀜** | — | 다른 뷰 (상태 리셋) |
| 다름 | 같음 | `ForEach` **밖** | 다른 뷰 |
| 다름 | 같음 | `ForEach` **안** | **같은 뷰** |

"구조적이 명시적보다 우선"으로 외우면 세 번째 줄에서 예측이 어긋난다.
**"explicit이 structural을 덮어쓰지 않는다"**로 기억할 것.

#### explicit identity의 두 방향
구조적 identity는 위치 기반이라 두 방향으로 오판한다. 교정 도구가 각각 다르다.

| 상황 | 구조적 판단 | 실제 | 교정 |
|---|---|---|---|
| 리스트 항목 순서가 바뀜 | "다른 뷰" | 같은 항목 | `ForEach`의 `id:` → "같다" |
| 같은 자리에 다른 대상이 옴 | "같은 뷰" | 다른 것 | `.id()` 모디파이어 → "다르다" |

`.id()`로 identity를 **다르게** 만드는 건 되지만, **같게** 만드는 건 안 된다.

### 실험 — 두 분기에 같은 `.id()`를 주면?
Apple 문서에 없어 직접 확인함 (macOS SwiftUI, 2026-08-05).
첫 `onAppear`에서 `count`를 42로 올린 뒤 조건을 토글해, `onAppear`가 다시 불리고
`count`가 0부터 시작하는지를 봤다.

```swift
struct Probe: View {
	let label: String
	@State private var count = 0

	var body: some View {
		let _ = print("body [\(label)] count=\(count)")
		return Text("\(label) \(count)")
			.onAppear {
				print("APPEAR [\(label)] count=\(count)")
				if count == 0 { count = 42 }
			}
	}
}

// A. 두 분기에 같은 id  ← 검증 대상
if isOn { Probe(label: "A/true").id("same") } else { Probe(label: "A/false").id("same") }

// B. id 없음  ← 대조군
if isOn { Probe(label: "B/true") } else { Probe(label: "B/false") }

// C. 분기 없음  ← 대조군
Probe(label: isOn ? "C/true" : "C/false")
```

토글 직후 로그:
```
=========== TOGGLE ===========
  body   [C:no-branch/true]  count=42     ← 상태 유지, onAppear 없음
  body   [A:same-id/true]    count=0      ← 상태 파괴
  body   [B:no-id/true]      count=0      ← 상태 파괴
  APPEAR [B:no-id/true]      count=0
  APPEAR [A:same-id/true]    count=0
```

| 케이스 | 구성 | 결과 | 판정 |
|---|---|---|---|
| A | `if/else` + 두 분기 같은 `.id()` | count 0, `onAppear` 재호출 | identity 다름 |
| B | `if/else`, id 없음 | count 0, `onAppear` 재호출 | identity 다름 |
| C | 분기 없이 값만 교체 | count 42 유지, `onAppear` 없음 | identity 같음 |

**A와 B가 완전히 동일하게 동작한다.** 같은 `.id()`를 줘도 안 준 것과 결과가 같다 —
분기 사이에서 `.id()`가 아무 일도 하지 않았다.

분기를 넘어 상태를 지키는 방법은 분기를 없애거나(C), 상태를 분기 바깥 상위로 올리는 것뿐이다.

### ForEach id 선택이 중요한 이유
id를 **인덱스**로 잡으면 행의 identity가 곧 위치가 된다.
중간 항목을 삭제하면 아래 행들의 identity가 밀려서 SwiftUI 입장에선
"같은 행인데 내용이 바뀐 것"으로 보인다.

- 삭제/삽입 애니메이션이 어색해짐
- 각 행에 `@State`가 있으면 엉뚱한 행으로 상태가 옮겨감

`avoidNo` 같은 **안정적인 키**로 잡으면 "그 행이 사라진 것"으로 정확히 인식한다.
애니메이션이 없고 리스트가 짧으면 인덱스도 실무상 문제되지 않지만 기본은 안정적인 키.

### 관련
- [[Diffing]] — identity가 같을 때 하는 값 비교
- [[EquatableView]] — 그 값 비교를 직접 지정
- [[@State]] — 저장소가 View 인스턴스가 아니라 **identity**에 종속
- [[Identifiable]]

### 근거
WWDC21 "Demystify SwiftUI" — Identity / Lifetime / Dependencies 세 축으로 설명하며
"identity가 바뀌면 상태가 유지되지 않는다"를 명시적으로 다룬다.

스코프 모델과 "두 분기에 같은 `.id()`" 동작은 Apple이 명시한 적이 없다.
위 실험으로 확인한 결과이며, 구현 세부사항이라 버전에 따라 바뀔 수 있다.
