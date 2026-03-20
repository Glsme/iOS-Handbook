# Record
미리 정해둔 값들과 completion을 그대로 재생하는 Publisher

```swift
strcut Record<Output, Failure> where Failure: Error
```
- 채택하는 Protocol
	- [[Copyable]]
	- [[Decodable]]
	- [[Encodable]]
	- [[Escapable]]
	- [[Publisher]]
- 동작 방식
```swift
let record = Record<String, Error> { recording in
	print("recording start")
	recording.receive("1")
	recording.receive("2")
	recording.receive(completion: .failure(NSError(domain: "test", code: 1)))
	// recording.receive("3") // 만약 여기서 recieve를 또 한다면 크래시 발생
	// recording.receive(completion: .finished)
}

record.sink { completion in
	print(completion)
} receiveValue: { value in
	print(value)
}
.cancel()
```
1. `Recording` 구조체 안에 `output` 배열에 저장
2. 구독 시 저장되어있는 값들 방출
3. `.finsihed` or `error` 수행
4. 만약 `recording.receive(completion:)`을 방출 후 이후 값을 방출하려고 할 경우 앱 크래시 발생