# DataFlow
뷰에 무엇을 어떻게 넘길지에 대한 SwiftUI의 데이터 흐름 규칙

### Apple은 MVVM을 표준으로 제시하지 않았다
SwiftUI 문서에 `ViewModel`이라는 용어는 나오지 않는다. Apple이 제시하는 건 아키텍처 이름이 아니라
데이터 흐름 원칙 세 가지다 (WWDC20 "Data Essentials in SwiftUI").

1. **Source of Truth는 하나만 둔다** — 같은 데이터를 두 곳에 복제하지 않는다
2. **뷰는 상태의 함수다** — 이벤트의 나열이 아니라 상태를 받아 화면을 그리는 함수
3. **뷰에는 필요한 최소한의 데이터만 전달한다**

Apple 공식 튜토리얼(Scrumdinger)과 샘플 앱(Fruta, Food Truck)에는 ViewModel 레이어가 없다.
모델 객체 + [[@State]] / [[@Binding]]으로 직접 간다.

### 버전별 정석

#### iOS 13~16
참조 타입 상태는 [[ObservableObject]]로 만들고 **소유권에 따라 래퍼를 나눈다**.

```swift
struct RootView: View {
	@StateObject private var model = Model()      // 소유 — 한 번만 생성

	var body: some View {
		DetailView(model: model)
	}
}

struct DetailView: View {
	@ObservedObject var model: Model              // 전달 — 소유하지 않음
}
```

[[@StateObject]]가 iOS 14에서 추가된 이유 자체가 [[@ObservedObject]]에 객체를 직접 생성해
상태가 날아가는 문제를 막기 위해서였다.

#### iOS 17+
`@Observable`([[Observation]])이 나오면서 규칙이 단순해졌다.

```swift
@Observable
final class Model {
	var count = 0                                 // @Published 불필요
}

struct RootView: View {
	@State private var model = Model()            // 소유는 @State

	var body: some View {
		DetailView(model: model)                  // 전달은 래퍼 없이
	}
}

struct DetailView: View {
	let model: Model

	var body: some View {
		Stepper("\(model.count)", value: $model.count)   // 양방향은 @Bindable
	}
}
```

- 소유 → [[@State]]
- 전달 → 래퍼 없음
- 양방향 → `@Bindable`

### 모델과 ViewModel은 다른 물건이다
이름이 비슷해 섞이기 쉽지만 뷰 경계에서의 성질이 다르다.

| | 도메인/앱 모델 | ViewModel |
|---|---|---|
| 담는 것 | 데이터 그 자체 | 화면 상태 + 화면 로직 |
| 수명·범위 | 앱 또는 기능 단위, 여러 뷰가 공유 | 화면 1개 전용 |
| 노출하는 것 | 데이터, 도메인 연산 | **액션** (네비게이션, 저장, 삭제) |
| 생성 | 대개 인자 없이 가능 | UseCase·Repository·Actions 주입 필요 |

Apple이 "모델 객체를 그냥 넘겨라"라고 할 때의 대상은 **전자**다.
후자를 넘겨도 된다는 뜻이 아니라, 애초에 ViewModel 레이어를 두지 않는다는 뜻에 가깝다.

### 객체 vs 값 — 뷰에 무엇을 넘기나

#### 주입 깊이로 보면 명확하다
```
값 주입      → View(list: [...])                깊이 0. 데이터만 있으면 끝
객체 주입    → View(viewModel: VM(useCaseA:...  깊이 N. 화면에 안 쓰는 의존성까지 채워야 함
                                  useCaseB:...
                                  actions:...))
```

Feathers의 seam 관점에서 양쪽 다 seam은 있다 — `init` 파라미터가 enabling point다.
차이는 **통과 비용**이다. 리스트 하나 보여주려고 UseCase 여섯 개를 목으로 만들어야 한다면
seam이 있어도 사실상 닫힌 것에 가깝다.

#### 판단은 "무엇"이 아니라 "얼마나 무거운가"
- 인자 없이 생성되는 모델 → 들고 있어도 무해
- UseCase 다수를 물고 있는 ViewModel → 뷰가 읽는 게 적을수록 값 주입이 유리

ViewModel의 생성 비용이 큰 건 설계 실수가 아니라 역할의 결과다. UseCase를 호출하는 게 일이니
UseCase를 받아야 한다. "가볍게 만들면 된다"는 선택지가 아니다.

#### ViewModel에만 있는 리스크
객체를 들고 있으면 **액션도 함께 손 닿는 거리에 놓인다.**
`viewModel.showDetail()`을 뷰에서 직접 부를 수 있게 되고, 그 순간 네비게이션과 에러 처리가 뷰로 샌다.
콜백/EventHandler로 우회하게 만들어도 컴파일러가 막아주지 않아 **규율에 의존**하게 된다.

모델에는 이 리스크가 없다. 데이터를 읽는 게 전부이기 때문이다.

#### 비용의 실체는 테스트보다 Preview
SwiftUI View 자체를 단위 테스트하는 경우는 실무에서 드물다 — `ViewInspector` 같은 서드파티 없이는
body를 검사할 방법이 없다. 그래서 주입 깊이의 손실은 대개 **Preview가 막히는 형태**로 나타난다.
값만 받는 하위 뷰로 쪼개면 Preview는 거기서 커버된다.

### 커뮤니티에서 실제로 쓰이는 것

| 패턴 | 특징 | 주로 쓰는 곳 |
|---|---|---|
| **MVVM** ([[ObservableObject]] ViewModel) | 가장 널리 쓰임. UIKit MVVM 경험 이식 | UIKit에서 넘어온 팀 |
| **MV** (ViewModel 없이 모델 직접) | Apple 스타일. 레이어가 적음 | 신규 SwiftUI 전용 앱 |
| **TCA** | 단방향 흐름, `Store` 보유. 테스트 강력 | 대규모·복잡 상태 |

"SwiftUI에 MVVM이 필요한가"는 몇 년째 결론이 안 난 논쟁이고, Apple은 명시적으로 답한 적이 없다.

### 표준이 없는 영역 — UIKit 하이브리드
`UIHostingController`로 SwiftUI를 얹을 때의 데이터 흐름은 Apple 가이드가 사실상 없다.
브릿지 도구로만 문서화돼 있고 데이터를 어떻게 흘릴지는 다루지 않는다.
표준을 찾을 게 아니라 팀 규약으로 정할 영역이다.

값 주입으로 갈 경우 갱신은 `rootView` 교체로 처리한다.

```swift
viewModel.$items
	.receive(on: DispatchQueue.main)
	.sink { [weak self] items in
		self?.hostingController.rootView = ContentView(items: items)
	}
	.store(in: cancelBag)
```

이러면 ViewModel이 SwiftUI를 전혀 모르게 되고, [[ObservableObject]] 채택도 필요 없어진다.

### 관련
- [[ObservableObject]] · [[@ObservedObject]] · [[@Published]]
- [[Diffing]] — 객체 단위 무효화가 왜 문제인지
- [[Identity]]

### 근거
WWDC19 "Data Flow Through SwiftUI", WWDC20 "Data Essentials in SwiftUI",
WWDC21 "Demystify SwiftUI", WWDC23 "Discover Observation in SwiftUI", Apple SwiftUI Tutorials
