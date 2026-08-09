# Opaque Type
불투명 타입 — 구체 타입을 숨긴 채 반환하되, 컴파일러는 그 정체를 정확히 아는 타입. `some P` (SE-0244, Swift 5.1).
[[Protocol]]을 반환 자리에서 쓰지 못하던 문제를 "역방향 제네릭"으로 풀었다.

### 왜 필요한가
- **프로토콜 쪽 문제**: associatedtype·`Self` 요구가 있는 프로토콜은 그 타입이 메서드·프로퍼티
  시그니처에 사용되기 때문에, 구체 타입을 모르면 멤버를 타입 체크할 수 없다 → 반환 타입 자리에서
  아예 금지였다 ("protocol can only be used as a generic constraint", ~Swift 5.6).
- **구체 타입 쪽 문제**: ① 호출부가 구체 타입에 의존하게 되어 타입을 바꾸면 호출부를 전부
  수정해야 한다(은닉 실패). ② 하위 뷰가 많아질수록 타입이 기하급수적으로 늘어나
  (`ModifiedContent<VStack<TupleView<(Text, Image)>>, _PaddingLayout>`) 손으로 쓸 수 없고,
  본문 한 줄 수정 = 시그니처 변경이 된다.
- `some`은 그 사이의 빈자리: **숨기되, 컴파일러는 알게 한다.**

### 내부 동작
- 컴파일러가 **컴파일 시점에 함수 본문을 보고** 구체 타입을 추론한다.
- 그래서 본문에 제약이 걸린다: **모든 return 경로가 하나의 구체 타입**이어야 한다.
- 이름은 숨겨도 정체는 보존되므로 **타입 정체성(type identity)** 이 유지된다:

```
some Equatable을 반환하는 f(), g()가 둘 다 실제로는 Int를 반환할 때
① f()의 결과끼리  == 비교  → ✅ 같은 함수 = 같은 (비공개) 타입
② f()와 g()의 결과 == 비교 → ❌ 같은 Int일지라도 함수마다 다른 타입 취급
   (함수마다 자기만의 봉인 도장 — 호출부가 "f와 g는 같다"에 의존하는 순간 은닉이 새기 때문)
```

- `any`와의 결정적 차이: **`any`가 런타임으로 미루는 것은 저장 방식과 디스패치뿐, 타입 검사는
  여전히 100% 컴파일 타임이다.** `any Equatable` 두 개의 `==`는 크래시가 아니라 컴파일 에러다.
- 비유: `some` = 밀봉 상자(내용물은 하나로 정해져 있고 겉면만 가림) / `any` = 빈 상자(실행 중
  무엇이든 담기고 바뀔 수 있음).

### 대안과 트레이드오프
| 대안 | 타입 정하는 쪽 | 시점 | 얻는 것 | 잃는 것 |
| --- | --- | --- | --- | --- |
| 구체 타입 반환 | 구현부 (공개) | 컴파일 | 최대 성능, 전체 API | 은닉 실패, 변경 시 호출부 전파 |
| `some` | 구현부 (비공개) | 컴파일 | 은닉 + 타입 정체성 + 정적 디스패치 | 함수당 한 타입만 |
| `any` | 아무나, 실행 중 변경 가능 | 런타임 | 분기·이종 컬렉션의 유연성 | 박싱 + 동적 디스패치 비용, 타입 정체성 상실 |
| 제네릭 파라미터 | **호출자** | 컴파일 | 호출자 유연성 + 정적 디스패치 | 반환 은닉에는 부적합 |

- (미정리) 결정권 축이 재인출 미통과 — "컴파일러가 정한다"로 반복 오답. 컴파일러는 결정권자가
  아니라 받아 적는 쪽이고, 결정은 항상 구현부 아니면 호출자에 있다. 파라미터 위치의
  `some Animal`(SE-0341)이 `<T: Animal>`의 축약(정방향 제네릭)이라는 것도 미정착.
- 디스패치 축 (재인출 통과): `some`은 구체 타입을 알아 컴파일 시점에 결정(정적) → 특수화·인라이닝
  가능. `any`는 어떤 타입이 올지 몰라 **런타임에 어느 곳을 실행할지 결정**(동적) → 런타임에 타입이
  결정되므로 특수화가 원리상 불가능, 속도·메모리 손해.
- `any`의 실물 = **existential 컨테이너**: 어떤 타입이 와도 겉 크기가 같은 규격 상자. 작은 값은
  인라인 버퍼에 직접, 큰 값은 힙에 두고 포인터만 담는다(박싱 비용).
  - (미정리) 상자에 실린 두 테이블 — Value Witness Table(복사·파괴 = 생명주기 설명서) /
    Protocol Witness Table(메서드 호출이 거쳐 가는 주소록) — 재인출 미통과.

> existential 컨테이너의 버퍼 크기·레이아웃 세부는 컴파일러 구현 사항이다 (WWDC 2016 공개 설명 기준 모델).

### 언제 쓰면 안 되는가
- **"하나의 타입" 전제가 깨질 때.** 분기에서 서로 다른 타입 반환(Text or Image)은 `some`만으로
  구현할 수 없다 → AnyView로 지우거나 [[@ViewBuilder]]로 포장. ([[View]]의 _ConditionalContent 참고)
- 이종 컬렉션: `[some Shape]`은 한 가지 타입의 배열이라 `[Circle(), Square()]`를 못 담는다 →
  `[any Shape]`의 자리.

### 실전 적용
- SwiftUI 밖: RegexBuilder(`some RegexComponent`), Swift Charts(`some ChartContent`),
  `some Collection<Int>`(SE-0346 primary associated type — 정체는 숨기고 원소 타입만 계약).
- (미정리) **`body: some View`인 이유** — 재인출 미통과. 구체 타입은 괴물 타입이라 탈락,
  `any View`는 타입 구조(= 뷰 트리 설계도)가 지워져 [[Diffing]]의 "값이 다르면 도배, 타입이
  다르면 철거" 판정이 무너지고 철거가 헤퍼진다([[@State]] 소멸, [[Identity]] 문제의 전면화).
  `some View`만 "개발자에게 커튼, 컴파일러에게 유리창"을 만족.

### 기반 개념
- (미정리) 사슬로 재인출 미통과: **추방 → 반전 → 공짜 → 상자** —
  [[Protocol]]이라는 무대에서, 계약이 자기 타입에 의존하면(associatedtype/`Self`) 반환 자리에서
  추방당했고 → 제네릭의 결정권을 뒤집어(`some`) 해결했으며 → 디스패치 축 덕에 최적화가 공짜고 →
  반대편 `any`의 실물이 existential 컨테이너다.

### 더 파볼 질문
- 결정권 축: 제네릭 파라미터의 타입은 왜 "호출자"가 정하는가? `feed(some Animal)` = 제네릭 축약 체화 (2회 미통과)
- SE-0309는 금지의 단위를 타입 전체 → 멤버 하나하나로 어떻게 바꿨나 (미통과)
- existential 컨테이너의 두 witness table 각각의 역할 (미통과)
- 기반 사슬 네 박자 재인출 (미통과)

### 출처
- [SE-0244 Opaque Result Types](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0244-opaque-result-types.md) · [SE-0309](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0309-unlock-existential-types-for-all-protocols.md) · [SE-0335 existential any](https://github.com/swiftlang/swift-evolution/blob/main/proposals/0335-existential-any.md) · SE-0341 · SE-0346
- [Swift 공식 문서 — Opaque and Boxed Protocol Types](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/opaquetypes/)
- WWDC 2022 [Embrace Swift generics](https://developer.apple.com/videos/play/wwdc2022/110352/) · WWDC 2016 [Understanding Swift Performance](https://developer.apple.com/videos/play/wwdc2016/416/) (existential 컨테이너) · WWDC 2021 [Demystify SwiftUI](https://developer.apple.com/videos/play/wwdc2021/10022/) (타입 구조 = 설계도)
