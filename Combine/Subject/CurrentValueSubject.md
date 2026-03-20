# CurrentValueSubject
현재 값을 저장하고 있으면서 값이 바뀔 때 마다 구독자에세 방출하는 Subject
```swift
final class CurrentValueSubject<Output, Failure> where Failure: Error
```

- Buffer에 가장 최근 발행된 요소를 유지
- `send` 메서드로 현재 값을 업데이트 (값을 직접 업데이트 하는 것과 같음)
- 현재 값을 가지고 있어 `value`로 접근 가능
- `value`에 새로운 값을 주입하는 것으로도 값 방출 가능 (단, 완료 이벤트는 주입 불가능)
- 구독 즉시 현재 값을 방출
- 상태(State)를 관리할 때 적합
- 채택하는 Protocol
	- [[Publisher]]
	- [[Subject]]
- 동작 방식
```swift
let subject = CurrentValueSubject<String, Never>("초기값")

// 현재 값 확인
print(subject.value)  // 초기값

// 구독하면 현재 값을 즉시 받음
subject.sink { print($0) }
// 출력: 초기값

// 새 값 전송
subject.send("두번째")  // 출력: 두번째
print(subject.value)    // 두번째
subject.value = "세번째" // 출력: 세번째
print(subject.value) // 세번째
```
