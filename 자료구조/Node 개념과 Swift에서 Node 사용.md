# Node

데이터와 다음 데이터의 주소 (Pointer)를 저장하는 단위

## Swift 관점의 Node

- 다음 데이터의 주소를 저장해야하기 때문에 `Class` Type 사용
- Pointer 에 실제 메모리 주소 저장 대신 Node 참조 사용

```swift
class Node<T> {
    var data: T?
    var next: Node?
    
    init(data: T, next: Node?) {
        self.data = data
        self.next = next
    }
}
```


# 연결 리스트
데이터와 다음 데이터의 주소(포인터, Pointer)를 함께 가진 노드(Node)들이 연결된 자료 구조
- 노드들이 서로 연결된 선형 자료 구조
- 메모리 상으로는 연속된 위치에 존재하지 않음.
- 각 요소 접근 시간이 최악의 경우 O(n)
- 삽입 / 삭제의 경우 주변 노드들의 Link만 수정하면 되기 때문에 O(1)


- 장점
	- 필요에 따라 메모리를 할당하여 크기를 동적으로 조절 가능
	- 삽입 / 삭제가 빠름 (O(1), 위치만 알면 포인터만 변경)
	- 메모리 낭비 감소 (배열처럼 남는 공간이 발생하지 않음)
- 단점
	- 인덱스 접근이 불가능하여 탐색 속도가 느림 (O(n))
	- 데이터와 포인터를 저장하여 추가적인 메모리 공간이 필요
	- 메모리 위치가 흩어져있어 캐시 효율이 낮음 (CPU 캐시 활용이 어려움)
- 종류
	- 단일 연결 리스트 (Singly Linked List): 다음 노드만 참조
	- 이중 연결 리스트 (Doubly Linked List): 이전 + 다음 노드 참조
	- 원형 연결 리스트 (Circular Linked List): 마지막 노드가 처음을 가리킴


### 구현 방법
```swift
struct LinkedList<T: Equatable> {
	private var head: Node<T>?
	private var tail: Node<T>?
	
	mutating func append(data: T?) {
		if head == nil {
			head = Node(data: data)
			return
		}
		
		var node = head
		
		while node?.next != nil {
			node = nodex?.next
		}
		
		node?.next = Node(data: data)
	}
	
	mutating func insert(data: T?, at index: Int) {
		if head == nil {
			head = Node(data: data)
			return
		}
		
		var node = head
		
		for _ in 0..<(index - 1) {
			if node?.next == nil { break }
			node = node?.next
		}
		
		let nextNode = node?.next
		node?.next = Node(data: data)
		node?.next?.next = nextNode
	}
	
	mutating func removeLast() {
		if head == nil { return }
		if head?.next == nil {
			head = nil
			return
		}
		
		var node = head
		
		while node?.next?.next != nil {
			node = node?.ext
		}
		
		node?.next = node?.next?.next
	}
	
	mutating func remove(at index: Int) {
		if head == nil { return }
		if index == 0 || head?.next == nil {
			head = head?.next
			return
		}
		
		var node = head
		
		for _ in 0..<(index - 1) {
			if node?.next?.next == nil { break }
			node = node?.next
		}
		
		node?.next = node?.next?.next
	}
	
	func searchNode(from data: T?) -> Node<T>? {
		if head == nil { return nil }
		
		var node = head
		
		while node?.next != nil {
			if node?.data == data { break }
			node = node?.next
		}
		
		return node
	}
}

// Node
class Node<T> {
	var data: T?
	var next: Node?
	
	init(data: T?, next: Node? = nil) {
		self.data = data
		self.next = next
	}
}
