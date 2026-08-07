# Diffing
[[Identity]]가 같은 뷰의 내용이 바뀌었는지 비교해 갱신 범위를 정하는 과정

### 핵심 개념
- **기본 비교는 [[Equatable]]이 아니다** — SwiftUI 내부의 리플렉션 기반 비교이고 공개 API가 아님
- `body` 재평가와 실제 화면 갱신은 **다른 단계**
- [[Identity]]가 "누구냐"를 판정한 뒤, diffing이 "뭐가 달라졌냐"를 판정

### 전체 흐름
```
① @Published 대입 (같은 값이어도 발생)
   ↓ willSet
② objectWillChange.send()
   ↓
③ @ObservedObject 수신 → 뷰 무효화
   ↓ 다음 업데이트 주기
④ body 재평가 → 새 뷰 값 트리 생성          ← 무효화된 뷰는 무조건 실행
   ↓
⑤ Identity 비교 → 다르면 파괴 후 재생성
   ↓ 같으면
⑥ 값 비교(diffing) → 같으면 자식 body 재평가 스킵
   ↓ 다르면
⑦ 최종 트리 대조 → 실제로 달라진 노드만 화면 반영
```

> 이 흐름은 [[ObservableObject]] 경로에 특화된 것이다. 일반 파이프라인의 단계 번호는
> [[뷰 업데이트 사이클 (View Update Cycle)]]이 정본이므로, **번호가 아니라 이름으로** 대조할 것.

- ④는 **무조건 실행**된다. [[@ObservedObject]]가 있는 뷰의 `body`는 값이 실제로 안 바뀌어도 돈다.
- 스킵이 일어날 수 있는 건 ⑥의 **자식**들.
- ⑦은 항상 동작한다. body가 다 돌아도 결과가 같으면 화면은 안 건드린다.

### 기본 비교의 동작
View의 **저장 프로퍼티들을 런타임에 훑어서** 비교한다고 알려져 있다.

- `Int`, `String`, enum 같은 단순 값 → 메모리 수준에서 빠르게 비교
- [[Equatable]] 준수 타입 → 런타임에 `==`를 찾아 사용
- **클로저, 함수, 일부 existential → 비교 불가 → 무조건 "달라졌다"로 처리**

> 이 메커니즘은 Apple이 문서로 보장한 API가 아니다. WWDC에서 "뷰를 비교해 body 재실행 여부를
> 정한다"는 개념과 POD 타입이 빠르다는 점까지는 설명하지만, 정확한 알고리즘은 구현 세부사항이며
> 버전에 따라 바뀔 수 있다.

### 클로저를 들고 있는 뷰는 스킵되지 않는다
```swift
SectionView(
	type: type,
	items: items,
	onRegister: { ... },        // 비교 불가 → 항상 "변경됨"
	onRemove: { item in ... }
)
```
콜백 클로저를 프로퍼티로 받는 자식 뷰는 부모가 재평가될 때마다 같이 재평가된다.
막으려면 [[EquatableView]]로 비교 기준을 직접 지정.

### 왜 프로퍼티 단위 추적이 안 되나
[[ObservableObject]]의 `objectWillChange`는 **객체 단위 신호 하나**다.
어떤 [[@Published]]가 바뀌었는지 정보 자체가 실려 있지 않아 SwiftUI는 그 객체를 관찰하는 뷰를
통째로 무효화할 수밖에 없다.

iOS 17+의 `@Observable`([[Observation]])은 **뷰가 실제로 읽은 프로퍼티만 추적**해서 이 문제를 해결한다.

### 측정 방법
```swift
var body: some View {
	let _ = Self._printChanges()   // 무엇 때문에 재평가됐는지 콘솔 출력
	ScrollView { ... }
}
```
또는 Instruments의 **SwiftUI 템플릿 → View Body** 항목으로 body 실행 횟수와 시간 확인.

### 최적화 판단 기준
- 뷰 트리가 작고 갱신이 드물면 body 재평가 비용은 무시 가능
- 문제가 되는 건 리스트가 수십~수백 행이거나, 스크롤·타이핑처럼 초당 여러 번 갱신되는 화면
- **계측으로 확인하기 전에는 손대지 않는 편이 낫다**

줄이는 수단
1. [[EquatableView]] — 비교 기준을 직접 지정해 클로저를 제외
2. ViewModel 분할 — 관심사별로 [[ObservableObject]]를 나누면 서로 영향 없음
3. `@Observable`([[Observation]]) — iOS 17+

### 관련
- [[Identity]]
- [[EquatableView]]
- [[ObservableObject]]

### 근거
WWDC21 "Demystify SwiftUI", WWDC23 "Demystify SwiftUI performance"
