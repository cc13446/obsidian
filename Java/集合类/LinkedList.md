# 概述
`LinkedList`同时实现了`List`接口和`Deque`接口，也就是说它既可以看作一个顺序容器，又可以看作一个队列，同时又可以看作一个栈。Java官方已经声明不建议使用`Stack`类。而且，Java里根本没有一个叫做`Queue`的类(它是个接口名字)。关于栈或队列，现在的首选是`ArrayDeque`，它有着比`LinkedList`(当作栈或队列使用时)有着更好的性能。

# 并发支持
为追求效率`LinkedList`没有实现同步(synchronized)，如果需要多个线程并发访问，可以先采用`Collections.synchronizedList()`方法对其进行包装。

# 底层实现
`LinkedList`底层通过双向链表实现，双向链表的每个节点用内部类`Node`表示。`LinkedList`通过`first`和`last`引用分别指向链表的第一个和最后一个元素。注意这里没有所谓的哑元，当链表为空的时候`first`和`last`都指向`null`。
```java
transient int size = 0;
/** 
* Pointer to first node. 
* Invariant: (first == null && last == null) || 
* (first.prev == null && first.item != null) 
*/ 
transient Node<E> first; 

/** 
* Pointer to last node. 
* Invariant: (first == null && last == null) || 
* (last.next == null && last.item != null) 
*/ 
transient Node<E> last;

// 内部节点
private static class Node<E> { 
	E item; 
	Node<E> next; 
	Node<E> prev; 
	Node(Node<E> prev, E element, Node<E> next) { 
		this.item = element; 
		this.next = next; 
		this.prev = prev; 
	} 
}
```

# Queue 方法
| 方法    | 抛出异常  | 返回特殊值 |
| ------- | --------- | ---------- |
| insert  | add(e)    | offer(e)   |
| remove  | remove()  | poll(e)    |
| examine | element() | peek(e)    | 

# Deque 方法
| 方法    | 抛出异常      | 返回特殊值    |
| ------- | ------------- | ------------- |
| insert  | addFirst(e)   | offerFirst(e) |
| remove  | removeFirst() | pollFirst(e)  |
| examine | getFirst()    | peekFirst(e)  |
| insert  | addLast(e)    | offerLast(e)  |
| remove  | removeLast()  | pollLast(e)   |
| examine | getLast()     | peekLast(e)   |
