# ConcurrentLinkedQueue数据结构

`ConcurrentLinkedQueue`的数据结构与`LinkedBlockingQueue`的数据结构相同，都是使用的链表结构，并且包含有一个头节点和一个尾结点。

# ConcurrentLinkedQueue源码分析

## 类的继承关系

```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable {}
```
说明: `ConcurrentLinkedQueue`继承了抽象类`AbstractQueue`，`AbstractQueue`定义了对队列的基本操作；同时实现了`Queue`接口，Queue定义了对队列的基本操作，同时，还实现了`Serializable`接口，表示可以被序列化。

## 类的内部类
```java
private static class Node<E> {
    // 元素
    volatile E item;
    // next域
    volatile Node<E> next;
    
    // 构造函数
    Node(E item) {
        // 设置item的值
        UNSAFE.putObject(this, itemOffset, item);
    }
    // 比较并替换item值
    boolean casItem(E cmp, E val) {
        return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
    }
    
    void lazySetNext(Node<E> val) {
        // 设置next域的值，并不会保证修改对其他线程立即可见
        UNSAFE.putOrderedObject(this, nextOffset, val);
    }
    // 比较并替换next域的值
    boolean casNext(Node<E> cmp, Node<E> val) {
        return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
    }

    // Unsafe mechanics
    // 反射机制
    private static final sun.misc.Unsafe UNSAFE;
    // item域的偏移量
    private static final long itemOffset;
    // next域的偏移量
    private static final long nextOffset;

    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = Node.class;
            itemOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("item"));
            nextOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

说明: Node类表示链表结点，用于存放元素，包含item域和next域，item域表示元素，next域表示下一个结点，其利用反射机制和CAS机制来更新item域和next域，保证原子性。

## 类的属性
```java
public class ConcurrentLinkedQueue<E> extends AbstractQueue<E>
        implements Queue<E>, java.io.Serializable {
    // 版本序列号        
    private static final long serialVersionUID = 196745693267521676L;
    // 反射机制
    private static final sun.misc.Unsafe UNSAFE;
    // head域的偏移量
    private static final long headOffset;
    // tail域的偏移量
    private static final long tailOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = ConcurrentLinkedQueue.class;
            headOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("head"));
            tailOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("tail"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
    
    // 头节点
    private transient volatile Node<E> head;
    // 尾结点
    private transient volatile Node<E> tail;
}
```

说明: 属性中包含了head域和tail域，表示链表的头节点和尾结点，同时，也使用了反射机制和CAS机制来更新头节点和尾结点，保证原子性。

## 类的构造函数

```java
public ConcurrentLinkedQueue() {
    // 初始化头节点与尾结点
    head = tail = new Node<E>(null);
}
// 用于创建一个最初包含给定 collection 元素的 ConcurrentLinkedQueue，
// 按照此 collection 迭代器的遍历顺序来添加元素。
public ConcurrentLinkedQueue(Collection<? extends E> c) {
    Node<E> h = null, t = null;
    for (E e : c) { // 遍历c集合
        // 保证元素不为空
        checkNotNull(e);
        // 新生一个结点
        Node<E> newNode = new Node<E>(e);
        if (h == null) // 头节点为null
            // 赋值头节点与尾结点
            h = t = newNode;
        else {
            // 直接头节点的next域
            t.lazySetNext(newNode);
            // 重新赋值头节点
            t = newNode;
        }
    }
    if (h == null) // 头节点为null
        // 新生头节点与尾结点
        h = t = new Node<E>(null);
    // 赋值头节点
    head = h;
    // 赋值尾结点
    tail = t;
}
```
## 核心函数分析

### offer函数
```java
public boolean offer(E e) {
    // 不能添加空元素
    checkNotNull(e);
    // 新节点
    final Node<E> newNode = new Node<E>(e);

    // 入队到链表尾
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        // 如果没有next，说明到链表尾部了，就入队
        if (q == null) {
            // CAS更新p的next为新节点
            // 如果成功了，就返回true
            // 如果不成功就重新取next重新尝试
            if (p.casNext(null, newNode)) {
                // 如果p不等于t，说明有其它线程先一步更新tail
                // 也就不会走到q==null这个分支了
                // p取到的可能是t后面的值
                // 把tail原子更新为新节点
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  // Failure is OK.
                // 返回入队成功
                return true;
            }
        }
        else if (p == q)
            // 如果p的next等于p，说明p已经被删除了（已经出队了）
            // 重新设置p的值
            p = (t != (t = tail)) ? t : head;
        else
            // t后面还有值，重新设置p的值
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

### poll函数

```java
public E poll() {
    restartFromHead:
    for (;;) {
        // 尝试弹出链表的头节点
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;
            // 如果节点的值不为空，并且将其更新为null成功了
            if (item != null && p.casItem(item, null)) {
                // 如果头节点变了，则不会走到这个分支
                // 会先走下面的分支拿到新的头节点
                // 这时候p就不等于h了，就更新头节点
                // 在updateHead()中会把head更新为新节点
                // 并让head的next指向其自己
                if (p != h) // hop two nodes at a time
                    updateHead(h, ((q = p.next) != null) ? q : p);
                // 上面的casItem()成功，就可以返回出队的元素了
                return item;
            }
            // 下面三个分支说明头节点变了
            // 且p的item肯定为null
            else if ((q = p.next) == null) {
                // 如果p的next为空，说明队列中没有元素了
                // 更新h为p，也就是空元素的节点
                updateHead(h, p);
                // 返回null
                return null;
            }
            else if (p == q)
                // 如果p等于p的next，说明p已经出队了，重试
                continue restartFromHead;
            else
                // 将p设置为p的next
                p = q;
        }
    }
}
// 更新头节点的方法
final void updateHead(Node<E> h, Node<E> p) {
    // 原子更新h为p成功后，延迟更新h的next为它自己
    // 这里用延迟更新是安全的，因为head节点已经变了
    // 只要入队出队的时候检查head有没有变化就行了，跟它的next关系不大
    if (h != p && casHead(h, p))
        h.lazySetNext(h);
}
```

出队的整个逻辑也是比较清晰的：
1. 定位到头节点，尝试更新其值为null；
2. 如果成功了，就成功出队；
3. 如果失败或者头节点变化了，就重新寻找头节点，并重试；
4. 整个出队过程没有一点阻塞相关的代码，所以出队的时候不会阻塞线程，没找到元素就返回null

## remove函数

```java
public boolean remove(Object o) {
    // 元素为null，返回
    if (o == null) return false;
    Node<E> pred = null;
    // 获取第一个存活的结点
    for (Node<E> p = first(); p != null; p = succ(p)) { 
        // 第一个存活结点的item值
        E item = p.item;
        // 找到item相等的结点，并且将该结点的item设置为null
        if (item != null 
	        && o.equals(item) 
	        && p.casItem(item, null)) { 
            // p的后继结点
            Node<E> next = succ(p);
            if (pred != null && next != null) // pred不为null并且next不为null
                // 比较并替换next域
                pred.casNext(p, next);
            return true;
        }
        // pred赋值为p
        pred = p;
    }
    return false;
}

private Node<E> first() {
    restartFromHead:
    for (;;) { // 无限循环，确保成功
        for (Node<E> h = head, p = h, q;;) {
            // p结点的item域是否为null
            boolean hasItem = (p.item != null);
            // item不为null或者next域为null
            if (hasItem || (q = p.next) == null) { 
                // 更新头节点
                updateHead(h, p);
                // 返回结点
                return hasItem ? p : null;
            }
            else if (p == q) // p等于q
                // 继续从头节点开始
                continue restartFromHead;
            else
                // p赋值为q
                p = q;
        }
    }
}
final Node<E> succ(Node<E> p) {
    // p结点的next域
    Node<E> next = p.next;
    // 如果next域为自身，则返回头节点，否则，返回next
    return (p == next) ? head : next;
}
```

## size函数
```java
public int size() {
    // 计数
    int count = 0;
    // 从第一个存活的结点开始往后遍历
    for (Node<E> p = first(); p != null; p = succ(p)) 
        if (p.item != null) // 结点的item域不为null
            // Collection.size() spec says to max out
            // 增加计数，若达到最大值，则跳出循环
            if (++count == Integer.MAX_VALUE) 
                break;
    // 返回大小
    return count;
}
```

# HOPS(延迟更新的策略)的设计

通过上面对offer和poll方法的分析，我们发现tail和head是延迟更新的，两者更新触发时机为：

-   ***tail更新触发时机***：当tail指向的节点的下一个节点不为null的时候，会执行定位队列真正的队尾节点的操作，找到队尾节点后完成插入之后才会通过casTail进行tail更新；当tail指向的节点的下一个节点为null的时候，只插入节点不更新tail。
    
-   ***head更新触发时机***：当head指向的节点的item域为null的时候，会执行定位队列真正的队头节点的操作，找到队头节点后完成删除之后才会通过updateHead进行head更新；当head指向的节点的item域不为null的时候，只删除节点不更新head。
    

并且在更新操作时，源码中会有注释为：`hop two nodes at a time`。所以这种延迟更新的策略就被叫做HOPS的大概原因是这个，head和tail的更新是跳着的，即中间总是间隔了一个。那么这样设计的意图是什么呢?

如果让`tail`永远作为队列的队尾节点，实现的代码量会更少，而且逻辑更易懂。但是，这样做有一个缺点，如果大量的入队操作，每次都要执行CAS进行`tail`的更新，汇总起来对性能也会是大大的损耗。如果能减少CAS更新的操作，无疑可以大大提升入队的操作效率，所以每间隔1次(`tail`和队尾节点的距离为1)进行才利用CAS更新`tail`。对`head`的更新也是同样的道理，虽然，这样设计会多出在循环中定位队尾节点，但总体来说读的操作效率要远远高于写的性能，因此，多出来的在循环中定位尾节点的操作的性能损耗相对而言是很小的。

# ConcurrentLinkedQueue适合的场景

`ConcurrentLinkedQueue`通过无锁来做到了更高的并发量，是个高性能的队列，但是使用场景相对不如阻塞队列常见，毕竟取数据也要不停的去循环，不如阻塞的逻辑好设计，但是在并发量特别大的情况下，是个不错的选择，性能上好很多，而且这个队列的设计也是特别费力，尤其的使用的改良算法和对哨兵的处理。整体的思路都是比较严谨的，这个也是使用了无锁造成的，我们自己使用无锁的条件的话，这个队列是个不错的参考。