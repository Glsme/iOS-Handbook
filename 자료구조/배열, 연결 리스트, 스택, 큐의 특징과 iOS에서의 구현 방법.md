# 배열
### 특징 
- 원소가 모두 동일한 데이터 타입
- 각 요소 접근 시간이 O(1)
- 연속된 메모리 공간에 데이터를 저장하여 메모리 관리에 효율적

### 구현 방법
```swift
let array = [1, 2, 3, 4]
```

- Swift에서 Array 선언 시, 배열 (값 자체)은 Stack에 저장되고, 실제 요소들이 있는 Buffer는 Heap에 저장




# 연결 리스트
- 노드들이 서로 연결된 선형 자료 구조
- 메모리 상으로는 연속된 위치에 존재하지 않음.
- 각 요소 접근 시간이 최악의 경우 O(n)
- 삽입 / 삭제의 경우 주변 노드들의 Link만 수정하면 되기 때문에 O(1)

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
```



# 스택
- 후입선출(LIFO) 구조

### 구현 방법
```swift
struct Stack<T> {
    private var storage: [T] = []

    var isEmpty: Bool {
        storage.isEmpty
    }
    
    var peek: T? {
        storage.last
    }

    mutating func push(_ element: T) {
        storage.append(element)
    }

    @discardableResult
    mutating func pop() -> T? {
        storage.popLast()
    }
}
```


# 큐
- 선입선출(FIFO) 구조

### 구현 방법
```swift
struct Queue<T> {
    private var array: [T] = []
    private var head: Int = 0

    var isEmpty: Bool {
        head >= array.count
    }

    var peek: T? {
        isEmpty ? nil : array[head]
    }

    mutating func enqueue(_ element: T) {
        array.append(element)
    }

    mutating func dequeue() -> T? {
        guard !isEmpty else { return nil }
        let element = array[head]
        head += 1
        
        // 메모리 정리
        if head > 50 {
            array.removeFirst(head)
            head = 0
        }

        return element
    }
}
```

