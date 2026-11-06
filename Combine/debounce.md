# debounce
입력이 일정 시간 멈춘 뒤 마지막 값 하나만 내보내는 [[Combine]]의 시간 기반 연산자. 짝이 되는 연산자는 [[throttle]].

```swift
func debounce<S: Scheduler>(
    for dueTime: S.SchedulerTimeType.Stride,
    scheduler: S,
    options: S.SchedulerOptions? = nil
) -> Publishers.Debounce<Self, S>
```

### 왜 필요한가

연달아 쏟아지는 이벤트를 전부 처리하면 처리량이 늘어난다. 그게 곧 앱의 부하로 이어진다.

검색창에서 키 입력마다 API를 호출하면 생기는 문제:

| 문제 | 내용 |
| --- | --- |
| 불필요한 요청과 비용 | 키 입력마다 요청. 서버 비용·배터리·데이터 낭비, rate limit 위험 |
| 응답 순서 경합 | "ab" 응답이 "abc"보다 늦게 오면 옛 결과가 화면을 덮음 |
| UI 깜빡임 | 결과가 짧은 간격으로 계속 바뀜 |

- debounce가 줄이는 것은 **이벤트의 개수**뿐
- 첫째·셋째 문제는 해결
- 둘째 문제(순서)는 해결 못 함 → `switchToLatest`의 몫

> 어원은 하드웨어 용어. 물리 스위치를 누르면 접점이 몇 ms 동안 떨리며(bounce) 신호가 여러 번 들어감. 이를 한 번으로 거르는 처리가 debounce.

### 내부 동작

- 이벤트가 올 때마다 타이머를 **처음부터 다시** 걺
- 마지막 이벤트 뒤로 설정 시간 동안 입력이 없으면 그 값을 방출
- 입력이 이어지는 동안에는 아무것도 안 나감

예: 간격 0.3초, 0.1초마다 2초 동안 입력

- 타이머가 끝나기 전에 계속 리셋됨
- 마지막 입력(2.0초) + 0.3초 = **2.3초에 한 번** 방출
- 키 입력 21번 → 값 1개

```
input             a─b─c─d─e───────────f─g─────────
debounce(0.3)     ──────────────e─────────────g───
                  0.0   0.3   0.6   0.9   1.2   1.5
```

- `e`(0.4초) 뒤 0.3초 동안 조용함 → 0.7초에 `e`
- `g`(1.1초) 뒤 0.3초 동안 조용함 → 1.4초에 `g`

#### 한 단계 아래: Scheduler

`scheduler` 인자의 정체는 `Scheduler` 프로토콜이다. debounce는 벽시계를 직접 보지 않는다. 스케줄러가 **debounce가 보는 유일한 시계**다.

| 측면 | 요구사항 | 예 |
| --- | --- | --- |
| 언제 (시간) | `now`, `schedule(after:tolerance:options:_:)` | "지금 몇 시야?", "0.3초 뒤에 실행해 줘" |
| 어디서 (실행 장소) | 채택 타입 자체 | `DispatchQueue.main` → 메인 큐, `RunLoop.main` → 메인 런루프 |

debounce 내부에서 반복되는 흐름:

```
① 새 값 도착        → "최신 값 번호"를 올림 (이전 예약은 낡은 예약이 됨)
② 스케줄러에 예약    → schedule(after: now + 0.3)
③ 예약 시각 도착     → 내 번호가 아직 최신인가?
                       ├─ 아니면 버림
                       └─ 맞으면 다운스트림으로 방출
```

Task로 직접 짠 debounce와 같은 구조:

| Task로 직접 짠 debounce | Combine debounce 내부 |
| --- | --- |
| `searchTask?.cancel()` | ① 최신 값 번호 올림 |
| `Task.sleep(for: .milliseconds(300))` | ② `schedule(after:)`로 예약 |
| `guard !Task.isCancelled` | ③ 번호가 최신인지 확인 |
| `await search(q)` | ③ 최신이면 방출 |

> 한 번만 실행하는 `schedule(after:tolerance:options:_:)`의 반환 타입은 `Void`. 예약을 취소할 핸들이 없음. `Cancellable`을 돌려주는 것은 반복 실행용 `schedule(after:interval:tolerance:options:_:)`뿐. 그래서 ①·③의 "번호 확인"으로 취소를 흉내 낸다고 봄. Apple 구현은 비공개라 OpenCombine 재구현을 근거로 한 **추정**.

#### 시간 주입과 테스트

- 시계가 스케줄러뿐이라 스케줄러를 바꿔 끼우면 시간 주입이 가능
- 가상 시간 스케줄러(`TestScheduler`)는 두 측면 중 **"언제"를 바꿔치기**함
- 실제로 기다리지 않고 `advance(by:)`로 시간을 앞당겨 검증

```swift
vm.query = "s"                                // 0.0초: 1번, 0.3초 예약
scheduler.advance(by: .milliseconds(200))
vm.query = "sw"                               // 0.2초: 2번, 0.5초 예약
scheduler.advance(by: .milliseconds(200))     // 0.4초: 0.3초 예약 실행 → 1번은 낡음 → 버림. 출력 없음
scheduler.advance(by: .milliseconds(100))     // 0.5초: 0.5초 예약 실행 → 2번 최신 → "sw" 방출
```

### debounce와 throttle의 차이

> 이 섹션은 [[debounce]]와 [[throttle]] 노트에 똑같이 들어 있음. 고칠 때는 두 곳 모두 수정.

간격 0.3초, 한 칸 = 0.05초:

```
input             a─b─c─d─e───────────f─g─────────
debounce(0.3)     ──────────────e─────────────g───
throttle(true)    ──────c─────e───────────g───────
throttle(false)   ──────a─────d───────────f───────
                  0.0   0.3   0.6   0.9   1.2   1.5
```

- debounce: 입력이 멈춘 뒤에만 방출
- throttle: 0.3초 구간마다 하나씩 방출. 입력 없는 구간(0.6~0.9초)은 건너뜀
- throttle 그림은 구간을 0초부터 고정한 개념도. 실제 Combine throttle은 첫 값을 바로 내보내고 그 시점부터 구간을 셈 (문서 미명시, 관찰된 동작)

| | debounce | throttle |
| --- | --- | --- |
| 방출 시점 | 입력이 **멈춘 뒤** | 입력 **중에도 주기적으로** |
| 0.1초 간격 2초 연속 입력 | 1번 (2.3초) | 약 7번 |
| 끝없이 이어지는 입력 | 영원히 안 나감 | 구간마다 계속 나감 |
| 반응 지연 | 항상 설정 시간만큼 늦음 | 첫 값은 즉시 (관찰된 동작) |
| 마지막 입력값 | 항상 전달 | `latest: false`면 유실 가능 |
| 묻는 질문 | "다 끝났어?" | "지금 어디쯤이야?" |

고르는 기준은 이벤트 종류가 아니라 **결과가 언제 필요한가**다.

- 입력이 끝난 뒤에만 결과가 의미 있음 → debounce
- 입력 도중의 중간 상태가 필요함 → throttle

| 상황 | 연산자 | `latest` |
| --- | --- | --- |
| 검색어 자동완성, 입력값 검증, 자동 저장 | debounce | — |
| 스크롤 이벤트 (무한 스크롤 판단, 위치 로깅, 헤더 축소) | throttle | `true` |
| 슬라이더·드래그 실시간 미리보기 | throttle | `true` (최종 위치 반영 필수) |
| 다운로드·업로드 진행률 표시 | throttle | `true` (100% 누락 불가) |
| 채팅 "입력 중…" 표시 전송 | throttle | 무관 |
| 위치·센서 업데이트 전송 | throttle | `true` |
| 버튼 연타 방지 | throttle | `false` (첫 탭 즉시 처리) |

### 대안과 트레이드오프

- 비교 대상: **연달아 쏟아지는 입력 이벤트 때문에 처리량이 늘어나는 문제**의 해법
- 판단 기준: 요청 수 감소, 응답 순서 보장, 이벤트 보존, 구현 비용

| 대안 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| **debounce** | 요청 수 최소화, 선언형 한 줄 | 마지막 값 외에는 버림. 응답 순서는 못 정리함 |
| **Task cancel / `switchToLatest`** | 항상 최신 결과, 경합 해결 | 요청 수는 그대로. cancel은 취소 협조 요청이라 이미 나간 API 호출은 못 막음 |
| **`DispatchWorkItem` 취소 / `Timer` 재설정** (수동 debounce) | 프레임워크 의존 없음 | 상태·스레드 직접 관리, 보일러플레이트 |
| **명시적 트리거** (리턴 키·검색 버튼) | 가장 단순, 낭비 0 | 입력하며 결과를 보는 UX 포기 |
| **`removeDuplicates`, 최소 글자 수 `filter`** | 비용 거의 없음 | 빈도 자체는 제한 못 함 |
| **결과 캐싱** (`NSCache` 등) | 같은 검색어는 공짜 | 메모리 사용, 데이터가 낡을 수 있음 |
| **버퍼링** (`collect(.byTime)`, 배열·큐에 쌓아 주기 전송) | 이벤트를 하나도 안 버림 + 요청 수 감소 | 전송 지연, 앱 종료 시 유실 위험, 메모리 사용 |
| **서버 측 rate limit** | 서버 보호 | 클라이언트 낭비는 그대로 |

- 실무에서는 하나만 쓰지 않음
- 검색창 기본 조합: **debounce(횟수) + `switchToLatest`(순서)**

### 언제 쓰면 안 되는가

#### 이벤트 하나하나가 중요할 때

- 실시간 통신처럼 값 하나하나가 의미 있는 스트림
- debounce는 애초에 값을 다 받지 않음
- 분석 로그처럼 누락 불가 + 요청 수 감소가 필요하면 → 배열·큐에 쌓아 주기적으로 전송(버퍼링)

#### 끝없이 이어지는 입력 (기아, starvation)

- 입력이 멈추지 않으면 **영원히 값이 안 나감**
- 스크롤 위치 스트림에 걸면 스크롤이 멈출 때까지 대기 → 화면이 어색하게 멈춘 것처럼 보임
- 대신 [[throttle]] 사용

```
input             a─b─c─d─e─f─g─h─i─j─k─l───…   0.1초마다 쉬지 않고 입력
debounce(0.3)     ──────────────────────────…   입력이 멈추지 않으면 영원히 안 나감
throttle(true)    ──────c─────f─────i─────l─…   0.3초마다 중간 상태 전달
```

#### 즉각 반응이 필요한 입력

- 반응이 항상 설정 시간만큼 늦음
- "좋아요" 버튼에 걸면 탭마다 굼뜸
- 연타하면 계속 대기 → 앱이 동작하지 않는 것처럼 보임
- 대신 [[throttle]] 사용

#### 응답 순서가 중요할 때 (debounce만으로는 부족)

- UI 반영까지는 비동기라서, 이전 결과가 늦게 도착하면 새 결과를 덮음
- 원인: `flatMap`이 안쪽 구독을 전부 살려 둠
- 해결: `map` + `switchToLatest()` → 새 검색이 오면 이전 검색 구독 취소

```
query             A─────B───────────────
search(A)         ──────────────────a   느린 응답
search(B)               ──────b
flatMap           ────────────b─────a───   늦게 온 a가 b를 덮음
switchToLatest    ────────────b─────────   B가 오는 순간 search(A) 구독 취소
```

#### `RunLoop.main`을 스케줄러로 쓸 때

- 증상: 스크롤하는 동안 결과가 안 바뀌다가, 스크롤이 멈춘 뒤에야 갱신됨
- 런루프가 멈춘 것은 아님
- 스크롤 중 메인 런루프는 `.tracking` 모드로 돎
- 이 스케줄러의 예약은 `.default` 모드에 등록됨 → 그동안 실행 안 됨
- [[throttle]]로 바꿔도 같은 스케줄러면 똑같이 멈춤
- 해결: 런루프 모드와 무관한 `DispatchQueue.main` 사용

> `RunLoop` 스케줄러가 예약을 `.default` 모드에 등록한다는 점은 공식 문서에 명시되지 않음. 커뮤니티에서 관찰된 동작.

### 실전 적용

#### 검색 체인

```swift
$query                                    // @Published var query: String
    .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
    .removeDuplicates()                   // 멈췄을 때 직전 검색어와 같으면 무시
    .filter { $0.count >= 2 }             // 너무 짧은 검색어 거르기
    .map { query in
        api.search(query)
            .replaceError(with: [])       // 에러는 이 요청 안에서 처리
    }
    .switchToLatest()                     // 옛 요청 결과 버리기
    .receive(on: DispatchQueue.main)
    .sink { [weak self] in self?.results = $0 }
    .store(in: &cancellables)
```

- **`removeDuplicates`는 debounce 뒤.** debounce가 마지막 값을 내보내도 직전 검색어와 중복일 수 있음. 예: "ab" → "abc" → "ab"로 되돌린 경우
- **출발점**: `UITextField`에는 기본 Combine Publisher가 없음
  - `UITextField.textDidChangeNotification`을 `NotificationCenter.default.publisher(for:object:)`로 수신
  - 또는 `addAction(_:for: .editingChanged)`에서 [[@Published]] 값에 대입
  - `searchBar(_:textDidChange:)`는 `UISearchBarDelegate` 메서드. `UITextFieldDelegate`에는 없음
- **에러 처리는 안쪽에서.** 아래 참고

#### 에러 처리 위치

Publisher는 한 번 완료되면 다시 살아나지 않는다.

- failure나 finished로 한 번 완료되면 이후 값 방출 없음
- 바깥 체인에 `catch`/`replaceError`를 붙이면, 요청 하나 실패 시 체인 전체가 `Just([])`로 교체된 뒤 완료
- 이미 끝나고 종료됐으므로 이후 입력은 sink까지 안 감
- 증상: 비행기 모드에서 한 번 실패한 뒤로 검색 먹통

```
query             A─────B─────C─────D───
search            a     X     c     d   B 요청만 실패
catch(outer)      a─────[]|   체인 전체가 Just([])로 교체되고 완료
catch(inner)      a─────[]────c─────d───   그 요청만 []로 대체, 스트림 유지
```

컨베이어 벨트 비유:

- 바깥 체인 = 벨트, 안쪽 Publisher = 벨트 위 상자 하나
- 상자 안에서 난 불(에러)은 상자 안에서 꺼야 함
- sink의 `receiveCompletion`에서 처리 = 공장이 멈춘 뒤 화재 보고서를 받는 것. 해결책 아님

컴파일러의 보증:

- `catch`/`replaceError`를 `map` 안으로 옮기면 바깥 체인의 `Failure`가 `Never`
- completion 핸들러 없는 `sink(receiveValue:)`는 `Failure == Never`일 때만 사용 가능
- 이 형태가 컴파일됨 = "에러로 스트림이 끝날 일 없음"의 보증
- 대체 값으로 [[Just]]를 쓰든 [[Empty]]를 쓰든 동일

#### Swift Concurrency로 직접 짠 debounce

- 이전 Task를 cancel만 하면 → switch-to-latest
- 새 Task에서 **먼저 sleep한 뒤** 일을 하면 → debounce

```swift
private var searchTask: Task<Void, Never>?

func queryDidChange(_ q: String) {
    searchTask?.cancel()
    searchTask = Task {
        try? await Task.sleep(for: .milliseconds(300))
        guard !Task.isCancelled else { return }   // try?가 CancellationError를 삼키므로 다시 확인
        await search(q)
    }
}
```

- 함정: `Task.sleep`은 취소되면 `CancellationError`를 던짐
- `try?`로 삼키면 sleep만 일찍 끝나고 아래 코드가 그대로 실행됨
- 그래서 `isCancelled` 재확인 필요
- SwiftUI: `.task(id: query) { try? await Task.sleep(...); ... }`. `id`가 바뀌면 이전 task 자동 취소
- `AsyncSequence`: Swift Async Algorithms의 `debounce(for:)`. 표준 라이브러리에는 없음

### 더 파볼 질문

- `RunLoop` 스케줄러가 예약을 어느 런루프 모드에 등록하는지에 대한 공식 근거
- Swift Async Algorithms의 `debounce`는 `Clock` 기반. Combine의 `Scheduler` 기반과 동작·테스트 방식의 차이
- 버퍼링한 로그를 앱 종료 시 잃지 않는 방법 (디스크 영속화, 백그라운드 작업)

### 관련 노트

- [[throttle]]: 짝이 되는 시간 기반 연산자
- [[Combine]]: 연산자가 속한 프레임워크
- [[Publisher]]: 연산자가 붙는 주체. 완료 후 재방출 없음
- [[Just]], [[Empty]]: 에러 대체용 Publisher
- [[@Published]]: 검색 체인의 출발점
- [[Swift_Concurrency]]: Task cancel과 `Task.sleep`으로 짠 debounce
- [[RunLoop]]: 런루프 모드와 스케줄러 (미정리)
- [[기본기 8주 로드맵]]: 1~2주차 ★ 항목

### 출처

- Apple Docs, [`debounce(for:scheduler:options:)`](https://developer.apple.com/documentation/combine/publisher/debounce(for:scheduler:options:))
- Apple Docs, [`Scheduler`](https://developer.apple.com/documentation/combine/scheduler)
- Apple Docs, [`switchToLatest()`](https://developer.apple.com/documentation/combine/publisher/switchtolatest()), [`removeDuplicates()`](https://developer.apple.com/documentation/combine/publisher/removeduplicates()), [`collect(_:options:)`](https://developer.apple.com/documentation/combine/publisher/collect(_:options:))
- Apple Docs, [`catch(_:)`](https://developer.apple.com/documentation/combine/publisher/catch(_:)), [`replaceError(with:)`](https://developer.apple.com/documentation/combine/publisher/replaceerror(with:)), [`sink(receiveValue:)`](https://developer.apple.com/documentation/combine/publisher/sink(receivevalue:)), [`Subscribers.Completion`](https://developer.apple.com/documentation/combine/subscribers/completion)
- Apple Docs, [`RunLoop.Mode.tracking`](https://developer.apple.com/documentation/foundation/runloop/mode/tracking)
- Apple Docs, [`Task.sleep(for:tolerance:clock:)`](https://developer.apple.com/documentation/swift/task/sleep(for:tolerance:clock:)), [`task(id:priority:_:)`](https://developer.apple.com/documentation/swiftui/view/task(id:priority:_:))
- WWDC 2019, *Combine in Practice*: debounce·removeDuplicates로 입력을 검증하는 예제
- GitHub, [apple/swift-async-algorithms](https://github.com/apple/swift-async-algorithms): Guides/Debounce.md
- GitHub, [OpenCombine](https://github.com/OpenCombine/OpenCombine): 비공식 재구현, 내부 구조 추정 근거
- GitHub, [pointfreeco/combine-schedulers](https://github.com/pointfreeco/combine-schedulers): `TestScheduler`
- SwiftLee, *RunLoop.main vs DispatchQueue.main*: RunLoop 스케줄러와 스크롤 관찰 사례
