# struct가 상속을 지원하지 않는 이유

Swift의 **struct**는 레이아웃이 확정된 값으로 복사되며, 클래스 상속처럼 하위 타입이 저장 프로퍼티를 덧붙이는 다형성을 제공하지 않는다.

### 왜 필요한가

struct 값은 다른 값 안에 직접 저장될 수 있으므로, 컴파일러가 그 타입의 크기·정렬·필드 오프셋을 계산할 수 있어야 한다. 만약 **Vehicle** struct를 상속한 **Car**가 저장 프로퍼티를 더할 수 있다면, Vehicle 자리에 Car를 넣을 때 필요한 저장 공간이 달라진다.

이 문제의 선택지는 셋이다.

- 값 전체를 잘라 내는 object slicing
- 모든 값을 박싱해 참조로 다루기
- 처음부터 고정 레이아웃을 지키고, 재사용은 [[Protocol]]과 합성으로 표현하기

Swift struct는 세 번째를 택한다. 따라서 “상속이 물리적으로 절대 불가능하다”기보다, **값 의미론·예측 가능한 복사·고정 레이아웃을 지키기 위한 언어 설계**다.

### 내부 동작

#### 저장 축 — 상속과 직접 재귀

상속과 직접 재귀는 모두 값의 크기 상한을 깨뜨린다.

~~~
상속
Base 값의 크기 < Child가 저장 프로퍼티를 추가한 뒤의 크기

직접 재귀
size(ValueNode) = size(Optional<ValueNode>) + 나머지 필드
→ size(ValueNode)를 계산하려면 다시 size(ValueNode)가 필요함
~~~

`struct ValueNode { var next: ValueNode? }`처럼 값 전체를 다시 품으면 크기 계산이 끝나지 않는다. Optional로 감싸도 Optional 역시 값 타입이므로 재귀가 끊기지 않는다.

여기서 **값 타입 = 항상 스택 또는 인라인 저장**은 아니다. struct는 클래스의 프로퍼티, 힙 버퍼, 클로저 캡처 등 여러 위치에 놓일 수 있다. 필요한 것은 저장 위치가 아니라 **해당 값의 레이아웃을 계산할 수 있는가**다.

#### class와 간접 참조

class 변수·프로퍼티가 저장하는 것은 실제 인스턴스 전체가 아니라 고정 크기의 참조다. 하위 클래스가 필드를 더해 실제 인스턴스의 크기가 커져도, Base를 가리키는 참조 칸의 크기는 변하지 않는다.

**final class ReferenceNode { var next: ReferenceNode? }**가 재귀를 허용받는 것도 같은 이유다. next에는 ReferenceNode 전체가 아니라 다음 인스턴스를 가리키는 참조 하나가 들어가므로, 재귀가 간접 참조에서 멈춘다. indirect enum도 필요한 경우를 박싱해 같은 방식으로 재귀를 표현한다.

#### 디스패치 축 — 저장 문제와 별개

상속을 허용한 class는 오버라이드된 메서드를 실제 타입에 맞게 골라야 한다. 개념적으로 흐름은 다음과 같다.

~~~
클래스 참조
→ 인스턴스
→ 인스턴스의 타입 메타데이터
→ 해당 타입의 vtable 슬롯
→ 선택된 함수 구현
~~~

vtable의 슬롯에는 함수 주소가 들어 있다. Dog가 speak()를 오버라이드했다면 Dog의 해당 슬롯은 Dog 구현을, 오버라이드하지 않았다면 상속받은 Animal 구현을 가리킨다. 호출 때마다 “오버라이드했는가?”를 비교하는 것이 아니다.

> 인스턴스 헤더·메타데이터·vtable의 정확한 물리적 레이아웃은 Swift ABI 구현 세부다. 핵심 언어 모델은 “non-final class의 메서드 호출은 실제 런타임 타입에 따라 동적 디스패치될 수 있다”는 점이다. final이면 컴파일러가 직접 호출로 최적화할 수 있다.

### 대안: 프로토콜과 합성

[[Protocol]]은 “어떤 속성·메서드를 제공해야 하는가”라는 계약이고, 저장 프로퍼티를 상속·공유하지는 않는다. 상태 재사용은 합성으로 한다.

~~~
VehicleCore  → 실제 speed 저장 공간                 (상태 재사용)
Vehicle      → speed를 읽고 쓸 수 있다는 요구사항      (계약)
Car.speed    → core.speed로 읽기·쓰기를 전달           (위임)
~~~

~~~swift
struct VehicleCore {
    var speed = 0
}

protocol Vehicle {
    var speed: Int { get set }
}

struct Car: Vehicle {
    var core = VehicleCore()

    var speed: Int {
        get { core.speed }
        set { core.speed = newValue }
    }
}

struct Bike: Vehicle {
    var core = VehicleCore()

    var speed: Int {
        get { core.speed }
        set { core.speed = newValue }
    }
}
~~~

Car와 Bike는 VehicleCore를 **포함**해 같은 상태 구조를 재사용한다. Vehicle은 두 타입이 지켜야 할 인터페이스만 선언하며, 상태를 제공하지는 않는다.

프로토콜의 요구사항은 **any P** 같은 existential에서 witness table을 통해 동적으로 선택될 수 있다. 반대로 프로토콜 익스텐션에만 있고 요구사항에는 없는 메서드는 witness table에 슬롯이 없으며, 호출 지점의 정적 타입을 기준으로 선택된다.

### 대안과 트레이드오프

비교 대상: 여러 타입에서 상태와 행동을 재사용하는 문제.
판단 기준: identity·공유 가변 상태가 필요한가, 아니면 값의 독립성과 고정 레이아웃이 중요한가.

| 대안 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| class 상속 | 공유 identity, 저장 프로퍼티·오버라이드, 런타임 다형성 | 참조 의미론, ARC·순환 참조 부담, 계층 결합 |
| [[Protocol]] + 합성 | 값 타입 유지, 필요한 계약만 조합, 명시적인 상태 소유 | 위임 코드, 공유 저장 프로퍼티는 직접 구성해야 함 |
| indirect enum | 트리·연결 리스트 같은 닫힌 재귀 모델을 값처럼 표현 | 새 case를 외부에서 확장할 수 없고, 박싱 비용이 생길 수 있음 |

### 언제 class를 선택해야 하는가

- 여러 곳이 **같은 인스턴스**를 공유하며 함께 변화해야 할 때
- 객체 그래프의 identity가 중요할 때. 예를 들어 UIKit의 UIView는 부모 뷰·제약·gesture recognizer 등이 특정 인스턴스를 참조한다.
- Objective-C 상호 운용, deinit, subclassing·오버라이드 자체가 요구사항일 때

반대로 단순한 데이터 모델, 독립적인 복사, 예측 가능한 상태 소유가 목적이면 struct가 기본 선택이다. 행동만 공통화하고 싶다면 class 계층을 만들기보다 [[Protocol]] + 합성을 먼저 검토한다.

### 실전 적용

SwiftUI의 View 값은 “어떻게 화면을 그릴지”를 나타내는 일시적인 설명이므로 새로 만들어져도 된다. **@State** 같은 지속 상태는 View 값 바깥의 SwiftUI 저장소가 관리한다. 반면 UIKit의 UIView는 화면 계층에 붙어 지속되는 객체라, 변수에 새 UIView를 대입해도 기존 superview·제약·responder chain의 연결은 자동으로 새 객체로 옮겨가지 않는다.

이 차이는 [[값으로서의 View (View as Value)]]와 [[값 의미론 (Value Semantics)]]을 이해하는 실제 적용 사례다.

### 기반 개념

- [[값 의미론 (Value Semantics)]] — 복사본의 독립성과 값 타입의 선택 기준
- [[Protocol]] — 계약과 witness table
- 간접 참조 — 고정 크기 참조로 재귀·확장 가능한 객체 그래프를 표현하는 방법
- 정적 디스패치와 동적 디스패치 — 저장 축과 섞지 말아야 할 별도 축

### 더 파볼 질문

- 라이브러리 배포 시 class의 레이아웃 변경이 ABI 호환성에 어떤 영향을 주는가?
- existential의 인라인 버퍼와 힙 박싱은 어떤 조건에서 갈리는가?

### 출처

- [Swift Language Guide — Structures and Classes](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/classesandstructures/)
- [Swift Language Reference — Declarations](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/declarations/)
- [Swift Language Guide — Inheritance](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/inheritance/)
- [Swift ABI — Type Metadata](https://github.com/swiftlang/swift/blob/main/docs/ABI/TypeMetadata.rst)
- [Swift ABI Stability Manifesto](https://github.com/swiftlang/swift/blob/main/docs/ABIStabilityManifesto.md)
