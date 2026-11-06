# throttle
일정 구간마다 값 하나씩만 내보내는 [[Combine]]의 시간 기반 연산자. 입력이 이어지는 동안에도 주기적으로 방출함. 짝이 되는 연산자는 [[debounce]].

```swift
func throttle<S: Scheduler>(
    for interval: S.SchedulerTimeType.Stride,
    scheduler: S,
    latest: Bool
) -> Publishers.Throttle<Self, S>
```

### 왜 필요한가

스크롤, 드래그, 진행률처럼 끝없이 이어지는 이벤트가 있다. 전부 처리하면 처리량이 늘고 앱 부하로 이어진다. 그렇다고 입력이 끝날 때까지 기다릴 수도 없다.

- 스크롤 위치: `scrollViewDidScroll`이 거의 프레임마다 호출됨
- 진행률 콜백: 매우 자주 옴. UI는 0.1초마다 갱신해도 충분함
- 슬라이더: 손을 떼기 전에도 미리보기가 따라와야 함

[[debounce]]로는 이 문제를 못 푼다.

- debounce는 입력이 멈춰야 방출
- 끝없는 스트림에서는 영원히 값이 안 나감 (기아, starvation)
- throttle은 **처리 횟수를 줄이면서도 중간 상태를 주기적으로 전달**함

### 내부 동작

- 설정한 구간마다 값 하나를 방출
- 입력이 이어지는 동안에도 구간마다 꾸준히 나감
- 입력이 없는 구간은 건너뜀
- 어떤 값을 보낼지는 `latest`가 결정

| `latest` | 구간에서 보내는 값 | 마지막 입력값 |
| --- | --- | --- |
| `true` | 가장 최근 값 | 반드시 전달 |
| `false` | 첫 값 | 유실 가능 |

예: 간격 0.3초, 0.1초마다 2초 동안 입력

- 0.3초마다 하나씩, 약 7번 방출
- `latest: true` → 각 구간의 마지막 값. 마지막 입력(2.0초)도 전달됨
- `latest: false` → 각 구간의 첫 값. 마지막 입력이 구간의 첫 값이 아니면 버려짐

```
input             a─b─c─d─e───────────f─g─────────
throttle(true)    ──────c─────e───────────g───────
throttle(false)   ──────a─────d───────────f───────
                  0.0   0.3   0.6   0.9   1.2   1.5
```

- 구간 [0.0, 0.3): `a b c` → `true`면 `c`, `false`면 `a`
- 구간 [0.3, 0.6): `d e` → `true`면 `e`, `false`면 `d`
- 구간 [0.6, 0.9): 입력 없음 → 방출 없음
- 구간 [0.9, 1.2): `f g` → `true`면 `g`, `false`면 `f`

> 위 그림은 구간을 0초부터 고정한 개념도. 실제 Combine throttle은 **첫 값을 기다리지 않고 바로 내보내고**, 그 시점부터 구간을 셈. 문서에 명시되지 않은 관찰된 동작이며, iOS 버전에 따라 차이가 보고된 적 있음.

#### 한 단계 아래: Scheduler

- 구간을 재는 시계와 실행 장소는 `scheduler` 인자가 담당
- `Scheduler` 프로토콜의 두 측면: **언제**(시간) + **어디서**(실행 장소)
- 예약·확인 흐름과 테스트 시 시간 주입은 [[debounce#한 단계 아래: Scheduler]] 참고

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

- 비교 대상: **끝없이 이어지는 이벤트를 매번 처리하면 부하가 생기지만, 중간 상태는 보여 줘야 하는 문제**의 해법
- 판단 기준: 중간 상태 전달 여부, 이벤트 보존, 중복 실행 차단의 확실성, 구현 비용

| 대안 | 얻는 것 | 잃는 것 |
| --- | --- | --- |
| **throttle** | 처리 횟수 감소 + 중간 상태 주기적 전달 | 구간 내 나머지 값은 버림. 시간만 보므로 요청 상태는 모름 |
| **[[debounce]]** | 처리 횟수 최소화 | 입력 중 중간 상태 없음. 끝없는 스트림에서는 영원히 안 나감 |
| **in-flight 플래그·버튼 비활성화** | 요청 상태 기준이라 중복 실행을 확실히 차단 | 플래그 해제 관리 필요. 연속 스트림에는 부적합 |
| **버퍼링** (`collect(.byTime)`, 배열·큐에 쌓아 주기 전송) | 값을 하나도 안 버림 | 전송 지연, 배치 처리 로직 필요, 앱 종료 시 유실 위험 |
| **소스 측 필터** (`CLLocationManager.distanceFilter` 등) | 이벤트 발생 자체를 줄여 가장 저렴 | 해당 API에만 존재. 시간 기준이 아닌 경우가 많음 |
| **수동 타임스탬프 비교** (마지막 처리 시각 기록) | 의존성 없음 | 마지막 값 처리를 직접 구현해야 함, 스레드 관리 |

### 언제 쓰면 안 되는가

#### 이벤트 하나하나가 중요할 때

- 실시간 통신처럼 값 하나하나가 의미 있는 스트림
- throttle도 구간 내 나머지 값은 버림
- 누락 불가 + 처리 횟수 감소가 필요하면 → 버퍼링

#### 최종 상태가 중요한데 `latest: false`일 때

- 입력이 끝난 구간의 마지막 값이 버려질 수 있음
- 증상: 필터 강도 슬라이더를 빠르게 끌다 놓으면, 손을 뗀 위치와 미리보기가 어긋남
- 해결: `latest: true`

#### 절대 두 번 실행되면 안 될 때

- throttle은 **시간만 봄**
- 결제 API가 3~5초 걸리고 throttle 간격이 1초면 → 1초 뒤 재탭 가능 → 중복 결제
- 해결: flag로 막거나 버튼 터치 차단. 응답까지 기다리는 UI 표시
- 시간이 아니라 **요청 상태**를 보는 방식

#### debounce와 공통인 한계

- **응답 순서 경합**: throttle도 이벤트 개수만 줄임. 순서는 `switchToLatest`의 몫
- **`RunLoop.main` 스케줄러**: 스크롤 중 `.tracking` 모드에서 예약이 실행 안 됨. `DispatchQueue.main` 사용
- 자세한 내용은 [[debounce#언제 쓰면 안 되는가]] 참고

### 실전 적용

#### 슬라이더 실시간 미리보기

```swift
$intensity                                // @Published var intensity: Double (Slider와 바인딩)
    .throttle(for: .milliseconds(100), scheduler: DispatchQueue.main, latest: true)
    .sink { [weak self] in self?.renderPreview(intensity: $0) }
    .store(in: &cancellables)
```

- `latest: true` → 손을 뗀 최종 위치가 반드시 반영됨
- 100ms마다 렌더링 → 드래그 중에도 미리보기가 따라옴

#### 버튼 연타 방지

```swift
let tap = PassthroughSubject<Void, Never>()
button.addAction(UIAction { _ in tap.send() }, for: .touchUpInside)

tap
    .throttle(for: .seconds(1), scheduler: DispatchQueue.main, latest: false)
    .sink { [weak self] in self?.submit() }
    .store(in: &cancellables)
```

- `UIButton`에는 기본 Combine Publisher가 없음 → [[PassthroughSubject]]로 탭을 스트림으로 만듦
- `latest: false` → 첫 탭을 처리하고 1초 안의 추가 탭은 무시
- 첫 탭이 바로 나간다는 관찰된 동작에 기대는 패턴
- 결제처럼 절대 두 번 나가면 안 되는 동작이면 throttle 대신 flag나 버튼 비활성화

#### Swift Concurrency

- `AsyncSequence`: Swift Async Algorithms의 `throttle(for:latest:)`. 표준 라이브러리에는 없음

### 더 파볼 질문

- Combine throttle의 첫 값 즉시 방출과 구간 경계 처리: 문서화되지 않은 동작이 iOS 버전별로 어떻게 달라졌는가
- Swift Async Algorithms의 `throttle`은 `Clock` 기반. Combine의 `Scheduler` 기반과 동작 차이

### 관련 노트

- [[debounce]]: 짝이 되는 시간 기반 연산자. Scheduler 내부 동작, 공통 한계
- [[Combine]]: 연산자가 속한 프레임워크
- [[Publisher]]: 연산자가 붙는 주체
- [[@Published]]: 슬라이더 값 스트림의 출발점
- [[PassthroughSubject]]: 버튼 탭을 스트림으로 만드는 출발점
- [[기본기 8주 로드맵]]: 1~2주차 ★ 항목

### 출처

- Apple Docs, [`throttle(for:scheduler:latest:)`](https://developer.apple.com/documentation/combine/publisher/throttle(for:scheduler:latest:))
- Apple Docs, [`Scheduler`](https://developer.apple.com/documentation/combine/scheduler)
- Apple Docs, [`collect(_:options:)`](https://developer.apple.com/documentation/combine/publisher/collect(_:options:))
- Apple Docs, [`scrollViewDidScroll(_:)`](https://developer.apple.com/documentation/uikit/uiscrollviewdelegate/scrollviewdidscroll(_:))
- Apple Docs, [`CLLocationManager.distanceFilter`](https://developer.apple.com/documentation/corelocation/cllocationmanager/distancefilter)
- GitHub, [apple/swift-async-algorithms](https://github.com/apple/swift-async-algorithms): Guides/Throttle.md
