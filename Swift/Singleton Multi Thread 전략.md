# Serial DispatchQueue로 보호
```swift
final class Singleton {
	static let shared = Singleton()
	private init() { }

	private let queue = DispatchQueue(label: "Singleton.queue")
	private var _token: String?
	
	var token: String? {
		get { queue.sync { _token } }
		set { queue.async(flags: .barrier) { _token = newValue } }
	}
}
```


# Actor 
```swift
actor Singleton {
	static let shared = Singleton()
	private init() { }
	
	private var token: String?
	
	func getToken() -> String? { token }
	func setToken(_ t: String?) { token = t }
}
```
