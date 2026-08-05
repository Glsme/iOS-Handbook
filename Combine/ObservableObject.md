# ObservableObject
객체가 변경되기 직전에 알림을 방출하는 Protocol (SwiftUI가 뷰 갱신 시점을 잡는 통로)

```swift
public protocol ObservableObject: AnyObject {
	associatedtype ObjectWillChangePublisher: Publisher = ObservableObjectPublisher
		where Self.ObjectWillChangePublisher.Failure == Never

	var objectWillChange: Self.ObjectWillChangePublisher { get }
}
```

### 핵심 개념
- 관찰**당하는** 쪽(모델·ViewModel)의 프로토콜
- 관찰**하는** 쪽은 SwiftUI의 [[@ObservedObject]] / [[@StateObject]] / [[@EnvironmentObject]]
- 둘 다 있어야 동작 — 프로토콜만 채택하면 "알릴 준비"만 된 상태

### 특징
- `AnyObject` 제약 — **클래스 전용** (struct 불가)
- 요구사항은 `objectWillChange` 프로퍼티 하나
	- 메서드가 아니라 **Publisher** — 실행되는 건 `objectWillChange.send()`
- [[@Published]] 프로퍼티가 있으면 직접 구현할 필요 없음
	- 기본 구현이 `ObservableObjectPublisher`를 제공
	- 각 [[@Published]]의 `willSet`이 여기에 `send()`를 대신 호출
- **방출 시점이 `willSet`** — 값이 실제로 갱신되기 전에 신호
	- 다만 SwiftUI는 신호를 받아 뷰를 무효화만 하고 `body`는 다음 업데이트 주기에 호출
	- 그때는 값이 이미 갱신돼 있어 실무에서 문제되지 않음
- **알림이 객체 단위** — 어떤 프로퍼티가 바뀌었는지 정보가 없음
	- 프로퍼티 하나만 바뀌어도 그 객체를 관찰하는 뷰 전체가 무효화됨
	- 프로퍼티 단위 추적이 필요하면 iOS 17+의 `@Observable`([[Observation]]) 사용
- `objectWillChange`를 **직접 선언하면 자동 연결이 끊김**
	- 기본 구현을 쓰지 않게 되므로 [[@Published]] 변경 시 직접 `send()` 호출 필요
	- 특별한 이유가 없으면 선언하지 않는 편이 안전
- [[@Published]]가 아닌 일반 `var`는 감지되지 않음

### 언제 채택하나
판단 기준은 "`objectWillChange`가 필요한가"가 아니라 **"이 객체를 SwiftUI 뷰가 관찰하는가"**다.
`objectWillChange`를 개발자가 직접 다룰 일은 거의 없고, 구독하는 쪽은 SwiftUI다.

```
@ObservedObject / @StateObject / @EnvironmentObject 로 받나?
├─ 예     → 채택 필요 (안 붙이면 컴파일 에러)
└─ 아니오 → 불필요
      └─ @Published + sink 로 UIKit에 바인딩 → @Published만으로 충분
```

- SwiftUI를 쓰는 화면이어도 모든 클래스에 붙이는 게 아님
	- 네트워크 클라이언트·계산 서비스·Coordinator처럼 뷰가 직접 관찰하지 않는 객체는 제외
	- 붙는 건 뷰가 [[@ObservedObject]]로 받는 그 객체 하나
- 같은 ViewModel이라도 관찰 주체가 UIKit → SwiftUI로 바뀌면 그때 필요해짐
	- UIKit 시절 `$foo.sink`만 쓰던 코드에 채택이 없던 건 누락이 아니라 정상
- 프레임워크 소속은 Combine이지만 실사용은 사실상 SwiftUI 전용
	- SwiftUI 초기 설계가 Combine 위에 얹혀 있던 흔적
	- 순수 Combine 코드에서는 [[@Published]]의 `$foo`가 더 정밀해 쓸 이유가 없음
- 뷰에 객체를 넘길지 값을 넘길지는 [[DataFlow]] 참고
	- 뷰가 읽는 상태가 적고 객체 생성 비용이 크면 값 주입이 유리
	- 값 주입을 택하면 이 프로토콜 자체가 필요 없어짐

`$foo.sink`로 받는 목적이라면 [[@Published]]만으로 충분하고 채택할 이유가 없다.
SwiftUI의 [[@ObservedObject]]가 **제네릭 제약으로 요구**하기 때문에 필요해진다.

```swift
@propertyWrapper
public struct ObservedObject<ObjectType> where ObjectType: ObservableObject { ... }
```

채택하지 않으면 "갱신이 안 온다" 이전에 빌드가 실패한다.
```
Generic struct 'ObservedObject' requires that 'MyViewModel' conform to 'ObservableObject'
```

### 예외 — SwiftUI 없이 채택하는 경우
`objectWillChange`를 직접 구독하면 SwiftUI 없이도 채택이 필요하다.
"이 객체의 뭐가 됐든 바뀌면 알려줘"를 스트림 하나로 받는 용도.

```swift
viewModel.objectWillChange
	.sink { _ in print("뭔가 바뀜") }
	.store(in: &cancellables)
```

어떤 프로퍼티가 바뀌었는지 알 수 없고 `willSet` 시점이라 값도 이전 것이라, 실무에서 쓸 일은 드물다.

### iOS 17+ — @Observable이 대체
`@Observable`([[Observation]])로 넘어가면 `ObservableObject`도 [[@Published]]도 사라진다.

```swift
// iOS 16
final class VM: ObservableObject {
	@Published var count = 0
}
struct V: View { @ObservedObject var vm: VM }

// iOS 17+
@Observable final class VM {
	var count = 0                    // @Published 불필요
}
struct V: View { let vm: VM }        // 래퍼도 불필요
```

프로퍼티 단위 추적이라 [[Diffing]]의 객체 단위 무효화 문제도 함께 해결된다.
"SwiftUI = ObservableObject" 등식은 deployment target이 17로 올라가면 깨진다.

### 예시
```swift
final class CounterViewModel: ObservableObject {
	@Published private(set) var count = 0        // 변경 시 objectWillChange 자동 발화
	let title = "카운터"                          // 감지 안 됨 (@Published 아님)

	func increase() {
		count += 1
	}
}

struct CounterView: View {
	@ObservedObject var viewModel: CounterViewModel

	var body: some View {
		Text("\(viewModel.count)")
	}
}
```

### 갱신 흐름
```
① @Published 대입 (같은 값이어도)
   ↓ willSet
② objectWillChange.send()          ← 어느 프로퍼티인지 정보 없음
   ↓
③ @ObservedObject 수신 → 뷰 무효화  ← 즉시 body 호출 아님
   ↓ 다음 업데이트 주기
④ body 재평가 → 새 뷰 트리 생성
   ↓
⑤ Identity 비교 → 값 비교(diffing) → 달라진 부분만 화면 반영
```

⑤ 단계는 [[Identity]]·[[Diffing]] 참고.

공식 문서
https://developer.apple.com/documentation/combine/observableobject
