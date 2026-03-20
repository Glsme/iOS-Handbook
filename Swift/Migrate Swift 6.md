https://swiftpackageindex.com/ready-for-swift-6

1. Enable complete checking
2. Enable Swift 6
3. Audit unsafe opt-outs

중요한 리팩토링과 데이터 레이스의 안정성을 동시에 해결하려고 하면 안된다. (너무 어려움)

전역변수
- 값이 변경되지 않으면 상수로 변경
- @MainActor 설정
- nonisolated(unsafe) 사용 (최후의 수단)

수정할 수 없는 패키지나 라이브러리를 사용할 때 사용
- `MainActor.assumeisolated`: Swift 컴파일러에게 ‘이건 main actor에서 실행되고 있다’고 알려주는 것.
- @preconcurrency




Tuist version: 4.48.1
