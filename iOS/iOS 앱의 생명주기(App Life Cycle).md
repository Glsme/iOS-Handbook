**2. iOS 앱의 생명주기(App Life Cycle)에 대해 설명해주세요.**  
- 앱의 각 상태(`Not Running`, `Inactive`, `Active`, `Background`, `Suspended`)에서 가능한 작업은 무엇인가요?  
- 상태 변화에 따라 호출되는 `AppDelegate` 또는 `SceneDelegate` 메서드는 무엇인가요?  
- 백그라운드에서 작업을 완료하기 위한 방법은 어떤 것이 있나요?


# iOS 앱의 생명주기(App Life Cycle)
: 앱이 최초 실행부터 완전 종료되기까지의 Cycle

앱 상태 종류
- Not Running: 앱이 아직 실행되지 않았거나, 완전히 종료되어 동작하지 않는 상태
- Foreground
	- Inactive: 앱이 실행중이지만 사용자로부터 이벤트를 받을 수 없는 상태
	- Active: 앱이 실제 실행중이고 사용자 이벤트를 받아서 상호작용할 수 있는 상태
- Background
	- Running: 홈 화면으로 나가거나 다른 앱으로 전환되어 실질적인 동작을 하지 않는 상태
	- Suspended: 앱을 다시 실행했을 때 최근 작업을 빠르게 로드하기 위해 메모리에 관련 데이터만 저장되어있는 상태



## 앱의 각 상태에서 가능한 작업
Not Running
- 없음

Inactive
- UI 업데이트를 중단하고 중요한 데이터를 보존할 준비

Active
- 일반적인 앱 작업 수행, 사용자 입력 처리 등

Background
- 음악 재생, 데이터 다운로드, 위치 추적

Suspend
- 없음


## 상태 변화에 따라 호출되는 `AppDelegate` 또는 `SceneDelegate` 메서드

### 앱 실행

```swift
application(_: didFinishLaunchingWithOptions:) 
```


### Scene 연결

```swift
application(:configurationForConnecting: options: )
```
- 새로운 Scene을 만들고 UIKit와 연결하기 위한 configuration을 지정


```swift
scene(_:willConnectTo:options: )
```
- Scene이 연결될 것임을 delegate에 알려줌


```swift
sceneDidBecomeActive(_:)
```
- 앱이 Inactive -> Active 상태로 전환 시 호출

### Active -> Inactive -> Background
```swift
sceneWillResignActive(_:)
```
- Active -> Inactive 상태로 전환 시 호출

```swift
sceneDidEnterBackground(_:)
```
- 앱이 Background 상태로 전환되었을 때 호출


### Background -> Inactive -> Active
```swift
sceneWillEnterForeground(_:)
```
- 앱이 Background -> Inactive 상태로 전환 시 호출

```swift
sceneDidBecomeActive(_:)
```
- 앱이 Inactive에서 Active 상태로 전환 시 호출


### Scene 연결 해제
```swift
sceneDidDisconnected(_:)
```
- delegate에 연결된 Scene의 연결 해제요청

```swift
application(_: didDiscardSceneSessions:)
```
- 사용자가 멀티태스킹 창 (app Switcher)에서 1개 이상의 Scene을 종료시켰을 때 호출

```swift
applicationWillTerminate(_:)
```
- 앱이 사용자에 의해 종료될 때 호출


### 백그라운드 작업을 완료하기 위한 방법
- 백그라운드 작업 실행
	- 앱이 백그라운드 전환 시 제한 시간(약 30초) 동안 작업 수행 가능
	- `beginBackgroundTask(expirationHandler:)` 에서 수행
- 백그라운드 모드 사용
	- 특정 작업(음악 재생, 위치 추적 등)은 Background Modes 설정으로 계속 실행 가능
	- Info.plist에서 UIBackgroundModes 키를 추가하여 사용
- Push Notification 처리
	- 원격 푸시 알림을 통해 백그라운드 작업 트리거
	- ex: 콘텐츠 업데이트
- URLSession
	- Background Transfer Service를 사용하여 데용량 데이터를 백그라운드에 다운로드 & 업로드 가능
	- `URLSession(configuration:delegate:delegateQueue:)` 에서 백그라운드 세션 구성
- Core Location
	- 위치 기반 서비스앱은 Background Modes를 활성화하여 지속적 위치 추적 가능
