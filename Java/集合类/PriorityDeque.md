转载自`https://pdai.tech/`
# 概述
优先队列的作用是能保证每次取出的元素都是队列中权值最小的。Java的优先队列每次取最小元素，C++的优先队列每次取最大元素。这里牵涉到了大小关系，元素大小的评判可以通过元素本身的自然顺序(natural ordering)，也可以通过构造时传入的比较器来实现。Java中`PriorityQueue`实现了`Queue`接口，不允许放入`null`元素。

# 底层实现
`PriorityQueue` 采用树形结构来描述元素的存储，具体说是通过完全二叉树实现一个小顶堆，在物理存储方面，PriorityQueue 底层通过数组来实现元素的存储。
![[PriorityQueue_struct.png]]
```java
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable {
	
	/**默认容量为11*/
	private static final int DEFAULT_INITIAL_CAPACITY = 11;

	/**队列容器*/
    transient Object[] queue;

	/**队列长度*/
    private int size = 0;

	/**比较器,为null使用自然排序*/
    private final Comparator<? super E> comparator;
	
	public PriorityQueue() {
		//默认数组长度为11，传入比较器为null
		this(DEFAULT_INITIAL_CAPACITY, null);
	}
	
	public PriorityQueue(int initialCapacity, Comparator<? super E> comparator) {
		//初始化容量小于 1，抛异常
		if (initialCapacity < 1)
			throw new IllegalArgumentException();
		this.queue = new Object[initialCapacity];
		this.comparator = comparator;
	}

	public PriorityQueue(Comparator<? super E> comparator) {
		//传入比较器 comparator
		this(DEFAULT_INITIAL_CAPACITY, comparator);
	}

}
```

#  扩容
当数组空间不足时，会进行扩容，扩容函数`grow()`类似于`ArrayList`里的`grow()`函数，就是再申请一个更大的数组，并将原数组的元素复制过去。
```java
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
	//如果旧数组容量小于64，新容量为 oldCapacity *2 + 2
	//如果大于64，新容量为 oldCapacity + oldCapacity * 0.5
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :
                                     (oldCapacity >> 1));
    //判断是否超过最大容量值，设置最高容量值
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
	//复制数组元素
    queue = Arrays.copyOf(queue, newCapacity);
}
```
# 插入和删除逻辑
参见[[Heap]]