# 뷰 업데이트 사이클 (View Update Cycle)
상태 변경이 화면 픽셀 변화로 이어지기까지 SwiftUI가 밟는 단계들의 반복 주기.
갱신의 **시점과 순서를 프레임워크가 소유한다**. [[Diffing]]·[[Identity]]가 이 사이클의 조각.

### 왜 필요한가 — 즉시 렌더가 아니라 "몰아서"인 이유
1. **일관성** — 핸들러가 상태를 여러 번 바꿔도 코얼레스를 거쳐 기다렸다가 한 번에 반영하므로 중간 상태가 화면에 새지 않는다
2. **병합** — 같은 턴의 N번 변경 = 1번 body. 변경마다 렌더하지 않아 비용 감소
3. **주사율 상한** — 120Hz면 최대 1초 120번. 가변 주사율에선 무효화가 없으면 0번 — 주기가 시계처럼 도는 게 아니라 **무효화가 있을 때만 프레임이 만들어진다**

### 내부 동작 — 파이프라인 7단계 (이 노트가 단계 번호의 정본)
```
① count += 1  (@State setter는 뷰 struct가 아니라 SwiftUI 소유 저장소에 쓴다)
② 의존성 조회 — "이 상태를 읽는 뷰"를 이미 안다 (이전 body 실행 때 기록)
③ 무효화 — dirty 표시만 남기고 끝. 실행이 아니라 예약
   ↓ 같은 턴의 다른 변경들이 여기 합류 = 병합
④ body 재호출 — 무효화된 뷰만. 새 뷰 값 트리 생성
⑤ identity 매칭 — 새 결과를 이전 트리의 누구와 비교할지 (구조적 위치 + 타입)
⑥ diffing — 이전 뷰와 최신 뷰 비교, 뭐가 달라졌나
⑦ 렌더 반영 — 달라진 View만 다시 렌더 (Core Animation 커밋 → vsync)
```
- **무효화 = dirty 표시.** 같은 것의 두 이름이다 (스프레드시트의 "재계산 필요" 표시)
- **②의 추적 단위는 상태 종류마다 다르다** — [[@State]]와 `@Observable`(iOS 17+)은 **읽은 프로퍼티** 단위지만,
  [[ObservableObject]]는 `objectWillChange` 신호 하나뿐이라 **객체** 단위다.
  후자는 뷰가 읽지도 않은 프로퍼티가 바뀌어도 그 객체를 가진 뷰가 통째로 무효화된다 ([[Diffing]])
- 세 질문의 담당: **전 = 누구(의존성 ②③) · 사이 = 짝(identity ⑤) · 후 = 뭐(diffing ⑥)**.
  "누구"는 body가 돌기 전에 답해야 하고(안 그러면 전부 돌려야 함), "뭐"는 body 산출물이 있어야 답할 수 있다
- 병합 타이밍은 고정 주기가 아니다 — **런루프 턴 끝에 몰아서, 단 프레임 데드라인(120Hz ≈ 8.3ms) 안에**.
  핸들러는 동기로 끝까지 돌고 body는 그 뒤라서, 업데이트 패스는 언제나 그 턴의 최종 상태만 본다 (중간 상태를 "감지"하는 게 아니라 구조적으로 볼 수 없다)

> 의존성 그래프의 실제 자료구조(AttributeGraph)와 vsync 연동 방식은 비공개 구현.
> 공개 확언 가능한 선은 "변경은 하나의 업데이트로 병합되고, 메인 스레드에서 프레임 데드라인 안에 돈다"까지
> — WWDC20 "Data Essentials", WWDC23 "Demystify SwiftUI performance"

### 트랜잭션 — 파이프라인의 동행자
트랜잭션은 파이프라인의 한 단계가 아니라 **한 번의 업데이트에 동행하는 맥락 정보**다.
①에서 태어나 파이프라인을 관통하며 값을 갖고 있다가 ⑦ 렌더 시점에 효력을 낸다.
- **모든 업데이트는 트랜잭션을 갖는다.** `withAnimation`은 그 봉투의 animation 속성을 채울 뿐
- 애니메이션이 뷰도 상태도 아닌 트랜잭션에 붙는 이유: 같은 뷰라도 탭엔 애니메이션, 초기 로드엔 즉시 반영 — **애니메이션 여부는 "이번 변경"의 속성**이다
- 세밀 제어: `withTransaction`, `.transaction { }` modifier

### 계약과 위반 — 언제 실패하는가
사이클의 소유물 4개와 침해:

| 소유물 | 위반 | 증상 |
| --- | --- | --- |
| 언제 (턴 끝에) | body **도중** 상태 변경 | "Modifying state during view update" 보라색 경고, 비결정적 화면 |
| 어디서 (메인) | 백그라운드 스레드에서 **상태 변경** (작업 밀어넣기가 아니라 변경 자체) | "Publishing changes from background threads" 경고, 갱신 누락 |
| 얼마나 (프레임 예산) | body 안 무거운 **동기** 계산 | 프레임 드랍, 스크롤 버벅임 |
| 누구로 (identity) | `.id(UUID())` 매번 새 id | 매 사이클 파괴+재생성 → [[@State]] 리셋, 애니메이션 끊김 |

- UB의 이유: body 도중 변경하면 **상태 변경 이전의 body와 이후의 body가 나뉜다** — 트리 앞쪽은 옛 스냅샷, 뒤쪽은 새 스냅샷. 결과가 끼어든 지점에 달려 실행마다 다르다 = "보장할 수 없다"
- 합법/불법의 기준은 "누가"가 아니라 **"시점"** — 핸들러·`.onAppear`·`.task`·`.onChange`는 전부 body 재호출 **전/밖**이라 합법
- 백그라운드의 올바른 패턴: **일은 백그라운드에서, 상태 반영은 메인에서** (`@MainActor` ViewModel). Swift 6 엄격 동시성에선 컴파일 타임에 잡히는 방향

### 대안과 트레이드오프
| 방식 | 갱신 트리거 | 얻는 것 | 잃는 것 |
| --- | --- | --- | --- |
| UIKit | 개발자가 직접 갱신하거나 `setNeedsLayout`으로 다음 주기에 지연 | 정밀 제어 | 상태→화면 동기화가 수동 책임 |
| SwiftUI | 상태 관찰로 **자동 트리거** + diff로 변경분만 | 갱신 누락·중복이 구조적으로 차단 | 갱신 시점·순서의 제어권 위임 |

"지연·병합"은 UIKit 디스플레이 사이클에도 있다. SwiftUI가 그 위에 얹은 것은 **상태 관찰(자동 무효화)과 diff**.
React·Flutter 비교는 [[값으로서의 View (View as Value)]]의 트레이드오프 표 참조.

### 실전 적용
도구를 파이프라인 위치로 기억:
- ④ 입구의 질문 "왜 다시 돌았어?" → `Self._printChanges()` ([[Diffing]]에 스니펫)
- 사이클 전체의 스톱워치 → Instruments **SwiftUI 템플릿** (View Body 트랙). 계측 전엔 손대지 않는다
- ①을 감싸는 봉투 → `withAnimation { }`

### 더 파볼 질문
- Transaction의 전파와 `.transaction` modifier의 우선순위 규칙
- React·Flutter의 갱신 모델 (이번 세션 범위 제외 — 선택)
- [[DynamicProperty]] — 상태 래퍼가 사이클 ②에 참여를 등록하는 통로 (백로그 기존 항목)

### 관련
- [[Diffing]] · [[Identity]] · [[값으로서의 View (View as Value)]] · [[선언형 UI (Declarative UI)]] · [[DataFlow]]

### 출처
- WWDC21 "Demystify SwiftUI" (Session 10022) — 의존성 그래프, identity
- WWDC23 "Demystify SwiftUI performance" (Session 10160) — 업데이트 주기, 프레임 데드라인
- WWDC20 "Data Essentials in SwiftUI" (Session 10040) — 변경 병합
- WWDC21 "Add rich graphics to your SwiftUI app" — 트랜잭션·애니메이션
- Apple 문서 `Transaction` · `withAnimation(_:_:)` · Core Animation Programming Guide (커밋 타이밍)
