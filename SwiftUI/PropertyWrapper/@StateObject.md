# @StateObject
뷰가 **소유**하는 [[ObservableObject]]를 identity 수명에 묶어 한 번만 생성하고 유지하는
Property Wrapper. [[@ObservedObject]]와 갱신 동작은 완전히 같고, 다른 것은 소유 한 축뿐이다.

```swift
@available(iOS 14.0, macOS 11.0, tvOS 14.0, watchOS 7.0, *)
@frozen @propertyWrapper public struct StateObject<ObjectType>: DynamicProperty
	where ObjectType: ObservableObject {

	public init(wrappedValue thunk: @autoclosure @escaping () -> ObjectType)

	public var wrappedValue: ObjectType { get }        // get-only
	public var projectedValue: ObservedObject<ObjectType>.Wrapper { get }
}
```

`projectedValue`가 `ObservedObject<>.Wrapper`를 **그대로 재사용**한다 — "차이는 소유뿐"이
타입으로 선언돼 있는 셈이다.

### 왜 필요한가

iOS 13 SwiftUI에는 비대칭이 있었다.

```
값 타입 상태    →  @State           (뷰가 소유, identity 수명 동안 생존)
참조 타입 객체  →  @ObservedObject  (빌려 쓰기만)  ← 소유 짝이 없음
```

가장 흔한 구조인 **화면 하나 = 뷰모델 하나**를 표현할 자리가 비어 있었다.

- `@ObservedObject var vm = VM()` → **뷰가 재생성될 때 모델도 같이 초기화되어 백지 상태**가 된다
- 회피책으로 부모가 만들어 주입해도 **부모 역시 뷰 struct라 같은 문제가 재발**한다
	→ 소유권이 계속 위로 밀려 결국 앱 최상위·싱글톤까지 올라간다

@StateObject가 메운 자리는 **뷰가 직접 만들되, 뷰 값보다 오래 사는 객체**다.

증상이 드러나는 조건이 좁아 발견이 늦다 — **상위 뷰가 body를 호출해 그 뷰가 재생성될 때**만
터진다. 처음 띄우면 멀쩡하고 혼자 두면 계속 멀쩡하다.

### 내부 동작 — 저장 위치가 다르다

```
@ObservedObject : 참조가 뷰 struct 안에 저장   → 뷰 값이 재생성되면 프로퍼티도 재생성
@StateObject    : 객체가 identity 저장소에 저장 → 재생성돼도 기존 객체를 그대로 돌려받음
```

이걸 가능하게 하는 문법 장치가 `@autoclosure @escaping`이다.
**완성된 객체가 아니라 "만드는 방법"만 클로저로 전달된다.**

```
① 뷰 값 첫 생성  → 클로저만 보관, 객체는 아직 없음
② 첫 body 직전   → 클로저 호출 → 객체 생성 → identity 저장소에 보관
③ 부모 갱신      → 뷰 값 재생성 → 클로저도 새로 만들어지지만
                   저장소에 이미 있으므로 새 클로저는 버려진다
④ identity 종료  → 저장소 폐기 → 마지막 강한 참조가 사라져 deinit
```

`@ObservedObject`에는 ③이 없다. 클로저가 아니라 **이미 실행된 결과**를 받으므로
버릴 기회조차 없다.

**실측** — 뷰 값만 3번 생성하고 body는 한 번도 호출하지 않았을 때:

| 선언 | 생성 함수 실행 횟수 |
| --- | --- |
| `@StateObject private var vm = makeVM()` | **0회** — 지연됨 |
| `@ObservedObject private var vm = makeVM()` | **3회** — 매번 실행 |

구독 메커니즘은 둘이 동일하다 — `objectWillChange` 하나만 구독하고, 어느 프로퍼티가
바뀌든 그 뷰의 body를 통째로 재평가한다 ([[@Published]] 각각을 구독하지 않는다).

> identity 저장소의 실제 자료구조(AttributeGraph)와 "이미 있으면 새 클로저를 무시"하는
> 판정의 구현은 비공개다. 공개적으로 확언 가능한 선은 Apple 문서의 *"creates a new
> instance of the object only once for each instance of the structure that it declares"*까지다.

### 대안과 트레이드오프

| 방법 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| **@StateObject** (뷰가 소유) | 수명이 화면 수명과 정확히 일치, 지연 생성 | iOS 14+, 뷰가 의존성을 직접 만들어 주입·테스트가 껄끄러움 |
| 부모가 @StateObject → 자식은 [[@ObservedObject]] | 소유권이 한 곳에 명확, 주입 가능 | 계층이 깊으면 프롭 드릴링 |
| [[@EnvironmentObject]] | 중간 계층을 건너뜀 | 주입을 잊으면 **런타임 크래시**, 의존성이 암묵적 |
| [[싱글톤 패턴 (Singleton Pattern)]]으로 전역 보관 | 앱 어디서든 접근 가능 | 사이드 이펙트가 나기 쉽고, 앱 종료까지 메모리에 남아 관리가 어려움 |
| UIKit VC가 소유 + [[@ObservedObject]] | 기존 아키텍처와 통합, VC 수명 = 화면 수명 | 진실이 SwiftUI 밖에 있어 [[DataFlow]] 표준이 둘로 갈림 |
| 값 타입 모델 + [[@State]] | [[값 의미론 (Value Semantics)]], 비교 가능해 무효화가 정밀 | 공유가 필요한 도메인엔 부적합 |
| iOS 17 `@Observable` + [[@State]] | 프로퍼티 단위 무효화 — **읽은 뷰만** 다시 렌더링 | iOS 17+, 마이그레이션 필요 |

**iOS 17 이후 — 구분은 남고 이름만 바뀐다**

```swift
// iOS 13~16 (ObservableObject)      // iOS 17+ (@Observable)
@StateObject private var vm = VM()   →  @State private var vm = VM()   // 소유
@ObservedObject var vm: VM           →  let vm: VM                     // 빌림 (래퍼 없음)
```

빌림이 `let`으로 충분한 이유는 `@Observable`이 **body가 읽은 프로퍼티를 추적**하기
때문이다 — 읽는 행위 자체가 구독이라 "구독하겠다"는 선언이 필요 없다. 소유가 [[@State]]로
옮겨간 이유는, @StateObject가 하던 일이 결국 **identity 저장소 + 지연 생성** 두 가지였고
그건 원래 @State의 메커니즘이기 때문이다.

### 언제 쓰면 안 되는가

**① 외부에서 받은 객체에 쓰면 안 된다 — 반대 방향의 버그**

```swift
init(vm: DetailViewModel) {
	_vm = StateObject(wrappedValue: vm)   // 첫 값만 붙잡는다
}
```
부모가 **다른** 객체를 넘겨도 갈아타지 않는다. ③번 단계가 그대로 적용되기 때문이다.
소유가 아닌데 소유 래퍼를 쓴 대가다.

**② identity가 끊어지면 객체도 함께 죽는다**

리셋 조건은 [[@State]]와 동일하다 — if/else 분기 전환, `.id()` 변경, ForEach 항목 소멸,
트리에서 제거 ([[Identity]] · [[값으로서의 View (View as Value)]]).
로그인 세션처럼 **화면 하나보다 오래 살아야 하는 객체**를 화면 뷰의 @StateObject로 두면
화면이 사라질 때 함께 사라진다. 수명이 필요한 높이에 두어야 한다.

**③ `private`을 빼면 안 된다 — 컴파일러가 못 잡는 계약 위반이 된다**

```
@StateObject의 약속      : "이 객체의 생성과 수명은 SwiftUI가 관리한다"
memberwise init의 약속   : "이 프로퍼티의 값은 호출자가 정한다"
                           ↑ 동시에 참일 수 없다
```

컴파일러는 이 모순을 이해하지 못한다. **컴파일 에러가 나지 않기 때문에 오히려 알아채기
어렵고**, ①과 똑같은 버그로 런타임에 나타난다.

`private`이 막아주는 것은 Swift가 자동 합성하는 **memberwise initializer**다. 그 접근
수준은 가장 제한적인 저장 프로퍼티를 따라가므로, private을 붙이면 그 프로퍼티가 파라미터
목록에서 빠진다. **실측**:

| 선언 | 다른 파일에서 `V(vm: someVM)` |
| --- | --- |
| `@StateObject var vm = VM()` | **컴파일 성공** — 문이 열려 있다 |
| `@StateObject private var vm = VM()` | `error: argument passed to call that takes no arguments` |

즉 **private은 컴파일러가 못 읽는 계약 위반을, 컴파일러가 읽을 수 있는 에러로 승격시키는
장치**다. [[@State]]의 "항상 private" 규칙과 이유가 같다.

**④ 반복되는 뷰마다 무거운 객체를 소유시키면 안 된다**
`LazyVStack`·`List`의 행마다 두면 스크롤에 따라 identity가 생기고 죽으며 객체가 반복
생성·해제된다. 무거운 초기화를 품고 있으면 스크롤이 그대로 비용이 된다.

**⑤ 애초에 참조 타입이어야 하는지**
`ObservableObject` 채택을 컴파일 타임에 요구하므로 클래스여야 한다. "뷰모델이니까"라는
이유만으로 클래스를 만들면 값 의미론과 정밀한 diff를 포기하는 선택이 된다.

### 실전 적용

**기본형 — 소유는 한 곳, 나머지는 빌림**

```swift
struct ProfileScreen: View {
	@StateObject private var viewModel = ProfileViewModel()   // 소유

	var body: some View {
		ProfileDetail(viewModel: viewModel)                    // 전달
	}
}

struct ProfileDetail: View {
	@ObservedObject var viewModel: ProfileViewModel            // 빌림
}
```

**생성에 인자가 필요할 때 — 언더스코어로 래퍼를 직접 대입**

```swift
init(userID: String) {
	_viewModel = StateObject(wrappedValue: ProfileViewModel(userID: userID))
}
```

`self.viewModel = ...`는 불가하다(get-only이고 init 단계에서 래핑된 값 접근 불가).
여기에 **비대칭 함정**이 붙는다 — init은 뷰 값 재생성마다 매번 실행되지만 **그 결과는 첫
번째 것만 채택된다.** `userID`가 바뀌어도 뷰모델은 옛 것 그대로다.
→ `.task(id:)` / `.onChange(of:)`로 알리거나, `.id(userID)`로 identity를 갈아치우거나,
소유를 부모로 올린다.

**UIHostingController 구조** — 소유자가 SwiftUI 밖(VC)에 있으므로 [[@ObservedObject]]가
맞다. @StateObject를 쓰면 오히려 ①의 함정에 걸린다.

### 더 파볼 질문
- iOS 17 `@Observable`의 프로퍼티 단위 추적은 실제로 무엇을 기록하는가 (`withObservationTracking`)
- `@StateObject`의 클로저는 정확히 body 첫 호출의 어느 시점에 실행되는가
- memberwise init 경로에서 `@autoclosure` 지연이 유지되는가 (실측 미확인)

### 관련
- [[@ObservedObject]] — 빌리는 쪽. 이 노트의 짝
- [[@State]] — 값 타입에 대한 같은 메커니즘
- [[Identity]] — "한 번"의 범위를 정의하는 기준
- [[DynamicProperty]] — struct 안의 래퍼를 바깥 저장소에 잇는 규약
- [[ObservableObject]] · [[@Published]] · [[값으로서의 View (View as Value)]] · [[DataFlow]]

### 출처
- Apple Developer Documentation — [StateObject](https://developer.apple.com/documentation/swiftui/stateobject)
- WWDC20 "Data Essentials in SwiftUI" (Session 10040)
- WWDC21 "Demystify SwiftUI" (Session 10022)
- WWDC23 "Discover Observation in SwiftUI" (Session 10149)
