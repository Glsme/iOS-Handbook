# SwiftUI Property Wrapper MOC

> SwiftUI 상태 관리 Property Wrapper 노트를 모아놓은 허브입니다.

# 값 타입 상태
- [[@State]]
- [[@Binding]]

# 참조 타입 상태
- [[@ObservedObject]]
- [[@StateObject]]
- [[@EnvironmentObject]]

관찰당하는 쪽은 [[ObservableObject]] + [[@Published]]

# 환경
- [[@Environment]]

# 기반
- [[DynamicProperty]] — 모든 상태 래퍼가 채택하는 "표식 하나, 훅 하나"의 참여 통로
