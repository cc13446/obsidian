转载自`https://pdai.tech/`
# 概述
`ArrayDeque`和`LinkedList`是`Deque`的两个通用实现，由于官方更推荐使用`ArrayDeque`用作栈和队列。

# 并发支持
`ArrayDeque`是非线程安全的(not thread-safe)，当多个线程同时使用的时候，需要程序员手动同步。

# 底层实现
底层通过循环数组实现，所以该容器不允许加入`null`元素。
```java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable{
	/**用于存放数组元素*/
	transient Object[] elements;

	/**用于指向数组中头部下标*/
	transient int head;

	/**用于指向数组中尾部下标*/
	transient int tail;

	/**最小容量，必须为2的幂次方*/
	private static final int MIN_INITIAL_CAPACITY = 8;
}
```

# 初始化方法解析
```java
public ArrayDeque() {
	// 默认初始化数组大小为 16
    elements = new Object[16];
}

public ArrayDeque(int numElements) {
	// 指定容量
	allocateElements(numElements);
}


private void allocateElements(int numElements) {
	elements = new Object[calculateSize(numElements)];
}
// 这个方法会将容量向上取2的倍数
private static int calculateSize(int numElements) {
	// 最小容量为 8
	int initialCapacity = MIN_INITIAL_CAPACITY;
    // 如果容量大于8，比如是2的倍数
    if (numElements >= initialCapacity) {
        initialCapacity = numElements;
        initialCapacity |= (initialCapacity >>>  1);
        initialCapacity |= (initialCapacity >>>  2);
        initialCapacity |= (initialCapacity >>>  4);
        initialCapacity |= (initialCapacity >>>  8);
        initialCapacity |= (initialCapacity >>> 16);
        initialCapacity++;

		// 容量超出 int 型最大范围，直接扩容到最大容量到 2 ^ 30
        if (initialCapacity < 0)
            initialCapacity >>>= 1;
    }
    return initialCapacity;
}
```

# 扩容
![[ArrayDeque_doubleCapacity.png]]
```java
private void doubleCapacity() {
	//扩容时头部索引和尾部索引肯定相等
    assert head == tail;
    int p = head;
    int n = elements.length;
	//计算头部索引到数组末端(length-1处)共有多少元素
    int r = n - p;
	//容量翻倍，相当于 2 * n
    int newCapacity = n << 1;
	//容量过大，溢出了
    if (newCapacity < 0)
        throw new IllegalStateException("Sorry, deque too big");
	//分配新空间
    Object[] a = new Object[newCapacity];
	//复制头部索引至数组末端的元素到新数组的头部
    System.arraycopy(elements, p, a, 0, r);
	//复制其余元素
    System.arraycopy(elements, 0, a, r, p);
    elements = a;
    head = 0;
    tail = n;
}
```

# 性能测试
![[ArrayDeque_compare.png]]
为什么`ArrayDeque`性能比`LinkedList`在数据量大的时候性能更好呢？
1. 在同等的数据量情况下，链表的内存开销要明显大于数组
2. 同时因为 `ArrayDeque` 底层是数组结构，天然在查询方面有优势，在插入、删除方面，只需要移动一下头部或者尾部变量，而`LinkedList`要频繁修改节点的前驱、后继变量。