转载自`https://pdai.tech/`
# JDK7
## 概述
`ConcurrentHashMap`在对象中保存了一个`Segment`数组，即将整个`Hash`表划分为多个分段；而每个`Segment`元素，即每个分段则类似于一个`Hashtable`；这样，在执行put操作时首先根据hash算法定位到元素属于哪个`Segment`，然后对该`Segment`加锁即可。因此，`ConcurrentHashMap`在多线程并发编程中可是实现多线程put操作。

## 数据结构
`ConcurrentHashMap `有 16 个 `Segments`，所以理论上，这个时候，最多可以同时支持 16 个线程并发写，只要它们的操作分别分布在不同的 `Segment` 上。这个值可以在初始化的时候设置为其他值，但是一旦初始化以后，它是不可以扩容的。

![[Pasted image 20220501131511.png]]

## 初始化
```java
// initialCapacity: 初始容量，平均分给每个 Segment
// loadFactor: 负载因子，是给每个 Segment 内部使用的
// concurrencyLevel: Segment 数量
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0;
    int ssize = 1;
    // 要保证 Segment 数量是2的整数幂
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    // 用默认值，concurrencyLevel 为 16，sshift 为 4
    // 那么计算出 segmentShift 为 28，segmentMask 为 15，后面会用到这两个值
    this.segmentShift = 32 - sshift;
    this.segmentMask = ssize - 1;

    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;

    // initialCapacity 是设置整个 map 初始的大小，
    // 这里根据 initialCapacity 计算 Segment 数组中每个位置可以分到的大小
    // 如 initialCapacity 为 64，那么每个 Segment 或称之为"槽"可以分到 4 个
    int c = initialCapacity / ssize;
    if (c * ssize < initialCapacity)
        ++c;
    // 默认 MIN_SEGMENT_TABLE_CAPACITY 是 2，
    // 这个值也是有讲究的,因为这样的话，对于具体的槽上，
    // 插入一个元素不至于扩容，插入第二个的时候才会扩容
    // 每个 Segment 的容量也要是2的整数幂
    int cap = MIN_SEGMENT_TABLE_CAPACITY; 
    while (cap < c)
        cap <<= 1;

    // 创建 Segment 数组，
    // 并创建数组的第一个元素 segment[0]
    Segment<K,V> s0 = new Segment<K,V>(loadFactor, 
						    (int)(cap * loadFactor),
						    (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
    // 往数组写入 segment[0]
    UNSAFE.putOrderedObject(ss, SBASE, s0); 
    this.segments = ss;
}
```
初始化完成，我们得到了一个 `Segment` 数组，默认情况为:
-   `Segment` 数组长度为 16，不可以扩容
-   `Segment[i]` 的默认大小为 2，负载因子是 0.75，初始阈值为 1.5，插入第二个才扩容。
-   初始化了 `segment[0]`，其他位置还是 `null`
-   `segmentShift` 为 32 - 4 = 28，`segmentMask` 为 16 - 1 = 15

## 基本方法
### setNext
```java
//使用volatile语义写入next,保证可见性
final void setNext(HashEntry<K,V> n) {
	UNSAFE.putOrderedObject(this, nextOffset, n);
}
```
### entryAt 
```java
// 获取给定table的第i个元素,使用volatile读语义
static final <K,V> HashEntry<K,V> entryAt(HashEntry<K,V>[] tab, int i) {
	return (tab == null) ? 
		null :
		(HashEntry<K,V>) 
		UNSAFE.getObjectVolatile(tab, ((long)i << TSHIFT) + TBASE);
}
```
### setEntryAt set HashEntry
```java
//设置给定的table的第i个元素,使用volatile写语义
static final <K,V> void setEntryAt(HashEntry<K,V>[] tab, int i,
								   HashEntry<K,V> e) {
	UNSAFE.putOrderedObject(tab, ((long)i << TSHIFT) + TBASE, e);
}
```
## put 过程分析
```java
public V put(K key, V value) {
    Segment<K,V> s;
    // 不允许放入null的value
    if (value == null)
        throw new NullPointerException();
    // 1. 计算 key 的 hash 值
    int hash = hash(key);
    // 2. 根据 hash 值找到 Segment 数组中的位置 j
    //    hash 是 32 位，无符号右移 segmentShift = 28 位，剩下高 4 位，
    //    然后和 segmentMask = 15 做一次与操作
    int j = (hash >>> segmentShift) & segmentMask;
    // 初始化的时候初始化了 segment[0]，但是其他位置还是 null，
    // ensureSegment(j) 对 segment[j] 进行初始化
    if ((s = (Segment<K,V>)UNSAFE.getObject(segments, (j << SSHIFT) + SBASE)) == null) 
        s = ensureSegment(j);
    // 3. 插入新值到槽 s 中
    return s.put(key, hash, value, false);
}
```
根据 `hash` 值很快就能找到相应的 `Segment`，之后就是 `Segment` 内部的 put 操作了

`Segment` 内部是由 `数组+链表` 组成的。

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    // 在往该 segment 写入前，需要先获取该 segment 的独占锁
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        // 这个是 segment 内部的数组
        HashEntry<K,V>[] tab = table;
        // 再利用 hash 值，求应该放置的数组下标
        int index = (tab.length - 1) & hash;
        // first 是数组该位置处的链表的表头
        HashEntry<K,V> first = entryAt(tab, index);

        for (HashEntry<K,V> e = first;;) {
            if (e != null) {
                K k;
                // 链表中找到了相应的key
                if ((k = e.key) == key 
	                || (e.hash == hash && key.equals(k))) {
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        // 覆盖旧值
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                // 继续顺着链表走
                e = e.next;
            }
            else {
                // node 到底是不是 null，这个要看获取锁的过程
                // 如果不为 null，那就直接将它设置为链表表头
                // 如果是null，初始化并设置为链表表头。
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);

                int c = count + 1;
                // 如果超过了该 segment 的阈值，这个 segment 需要扩容
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node); 
                else
                    // 没有达到阈值，将 node 放到数组 tab 的 index 位置，
                    // 其实就是将新的节点设置成原链表的表头
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        // 解锁
        unlock();
    }
    return oldValue;
}
```


## 初始化槽: ensureSegment
`ConcurrentHashMap` 初始化的时候会初始化第一个槽 `segment[0]`，对于其他槽来说，在插入第一个值的时候进行初始化。这里需要考虑并发，因为很可能会有多个线程同时进来初始化同一个槽 `segment[k]`，不过只要有一个成功了就可以。

```java
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {
        // 这里看到为什么之前要初始化 segment[0] 了，
        // 使用当前 segment[0] 处的数组长度和负载因子来初始化 segment[k]
        // 为什么要用当前，因为 segment[0] 可能早就扩容过了
        Segment<K,V> proto = ss[0];
        int cap = proto.table.length;
        float lf = proto.loadFactor;
        int threshold = (int)(cap * lf);

        // 初始化 segment[k] 内部的数组
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
        // 再次检查一遍该槽是否被其他线程初始化了。
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss,u)) == null) { 
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
            // 用 while 循环，内部用 CAS，当前线程成功设值或其他线程成功设值后退出
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) {
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```
总结：
1. 初始化`segment[k]`时使用当前 `segment[0]`处的数组长度和负载因子
2. 用CAS和循环来初始化`segment[k]`

## 获取写入锁: scanAndLockForPut
前面我们看到，在往某个 `segment` 中 `put` 的时候，先进行一次 `tryLock()` 快速获取该 `segment` 的独占锁，如果失败，那么进入到 `scanAndLockForPut` 这个方法来获取锁。

```java
// 获取该 segment 的独占锁，如果需要的话顺便实例化了一下 node
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node

    // 循环获取锁
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {
            if (e == null) {
                if (node == null) // speculatively create node
                    // 进到这里说明数组该位置的链表是空的，没有任何元素
                    // 当然，进到这里的另一个原因是 tryLock() 失败
                    // 所以该槽存在并发，不一定是该位置
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))
                retries = 0;
            else
                // 顺着链表往下走
                e = e.next;
        }
        // 重试次数如果超过 MAX_SCAN_RETRIES(单核1多核64)，进入到阻塞队列等待锁
        // lock() 是阻塞方法，直到获取锁后返回
        else if (++retries > MAX_SCAN_RETRIES) {
            lock();
            break;
        }
        // 这个时候有新的元素进到了链表，成为了新的表头
        //  重新走一遍这个 scanAndLockForPut 方法
        else if ((retries & 1) == 0 
	        && (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```
这个方法有两个出口，一个是` tryLock()` 成功了，循环终止，另一个就是重试次数超过了 `MAX_SCAN_RETRIES`，进到 `lock()` 方法，此方法会阻塞等待，直到成功拿到独占锁。

## 扩容: rehash

扩容是 segment 数组某个位置内部的数组进行扩容，扩容后容量为原来的 2 倍。回顾一下触发扩容的地方，put 的时候，如果判断该值的插入会导致该 segment 的元素个数超过阈值，那么先进行扩容，再插值。该方法不需要考虑并发，因为到这里的时候，是持有该 segment 的独占锁的。

```java
// 方法参数上的 node 是这次扩容后，需要添加到新的数组中的数据。
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 2 倍
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    // 创建新数组
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    // 新的掩码，如从 16 扩容到 32，那么 sizeMask 为 31
    int sizeMask = newCapacity - 1;

    // 遍历原数组，将原数组位置 i 处的链表拆分到 新数组位置 i 和 i+oldCap 两个位置
    for (int i = 0; i < oldCapacity ; i++) {
        // e 是链表的第一个元素
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            // 计算应该放置在新数组中的位置，
            int idx = e.hash & sizeMask;
            if (next == null)   // 该位置处只有一个元素，那比较好办
                newTable[idx] = e;
            else { 
                // e 是链表表头
                HashEntry<K,V> lastRun = e;
                // idx 是当前链表的头节点 e 的新位置
                int lastIdx = idx;

                // 找到一个 lastRun 节点，这个节点之后的所有元素是将要放到一起的
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                // 将 lastRun 及其之后的所有节点放到 lastIdx 这个位置
                newTable[lastIdx] = lastRun;
                // 下面的操作是处理 lastRun 之前的节点，
                // 这些节点可能分配在另一个链表中，也可能分配到上面的那个链表中
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    // 将新来的 node 放到新数组中刚刚的 两个链表之一的头部
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```
## get 过程分析
相对于 put 来说，get 就很简单了。
1. 计算 hash 值，找到 segment 数组中的具体位置，或我们前面用的槽
2. 槽中也是一个数组，根据 hash 找到数组中具体的位置
3. 到这里是链表了，顺着链表进行查找即可

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    // 1. hash 值
    int h = hash(key);
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    // 2. 根据 hash 找到对应的 segment
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null 
	    && (tab = s.table) != null) {
        // 3. 找到segment 内部数组相应位置的链表，遍历
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

### get并发问题分析
现在我们已经说完了 put 过程和 get 过程，我们可以看到 get 过程中是没有加锁的，那自然我们就需要去考虑并发问题。

添加节点的操作 `put` 和删除节点的操作 `remove` 都是要加 `segment` 上的独占锁的，所以它们之间自然不会有问题，我们需要考虑的问题就是 `get` 的时候在同一个 `segment` 中发生了 `put` 或 `remove` 操作。

1. put 操作的线程安全性。
    - 初始化槽，使用了 CAS 来初始化 Segment 中的数组
    - 添加节点到链表的操作是插入到表头的，所以，如果这个时候 get 操作在链表遍历的过程已经到了中间，是不会影响的
    - 另一个并发问题就是 get 操作在 put 之后，需要保证刚刚插入表头的节点被读取，这个依赖于 `setEntryAt` 方法中使用的 `UNSAFE.putOrderedObject`
    - 扩容是新创建了数组，然后进行迁移数据，最后面将 `newTable` 设置给属性 `table`。所以，如果 get 操作此时也在进行，那么也没关系，如果 get 先行，那么就是在旧的 table 上做查询操作；而 put 先行，那么 put 操作的可见性保证就是 table 使用了 `volatile` 关键字。
2. remove 操作的线程安全性。
    -   get 操作需要遍历链表，但是 remove 操作会破坏链表。
    -   如果 remove 破坏的节点 get 操作已经过去了，那么这里不存在任何问题。
    -   如果 remove 先破坏了一个节点，分两种情况考虑。 
    1. 如果此节点是头节点，那么需要将头节点的 next 设置为数组该位置的元素，table 虽然使用了 volatile 修饰，但是 volatile 并不能提供数组内部操作的可见性保证，所以源码中使用了 UNSAFE 来操作数组，请看方法 setEntryAt。
    2. 如果要删除的节点不是头节点，它会将要删除节点的后继节点接到前驱节点中，这里的并发保证就是 next 属性是 volatile 的。

总结：在get过程中使用了大量的volatile关键字，保证了可见性，get只是读取操作，所以我们只需要保证读取的是最新的数据即可。

## size过程分析
```java
public int size() {
	// Try a few times to get accurate count. On failure due to
	// continuous async changes in table, resort to locking.
	final Segment<K,V>[] segments = this.segments;
	int size;
	boolean overflow; // 为true表示size溢出32位
	long sum;         // modCounts的总和
	long last = 0L;   // previous sum
	int retries = -1; // 第一次不计算次数,所以会重试三次
	try {
		for (;;) {
			// 重试次数达到3次 对所有segment加锁
			if (retries++ == RETRIES_BEFORE_LOCK) { 
				for (int j = 0; j < segments.length; ++j)
					ensureSegment(j).lock(); // force creation
			}
			sum = 0L;
			size = 0;
			overflow = false;
			for (int j = 0; j < segments.length; ++j) {
				Segment<K,V> seg = segmentAt(segments, j);
				if (seg != null) { // seg不等于空
					sum += seg.modCount; // 不变化和size一样
					int c = seg.count; // seg的size
					if (c < 0 || (size += c) < 0)
						overflow = true;
				}
			}
			if (sum == last) // 没有变化
				break;
			last = sum; // 变化,记录这一次的变化值,下次循环时对比.
		}
	} finally {
		if (retries > RETRIES_BEFORE_LOCK) {
			for (int j = 0; j < segments.length; ++j)
				segmentAt(segments, j).unlock();
		}
	}
	return overflow ? Integer.MAX_VALUE : size;
}
```
`size` 尝试3次不加锁获取sum，如果发生变化就全部加锁，`containsValue`方法的思想也是基本类似。执行流程：
1. `++retries=0`，不满足加锁条件，计算所有segment的容量和
2. `++retries=1`，不满足加锁条件，计算所有的segment，与上一次相等结束循环。
3. `++retries=2`，与第二次一样
4. `++retries=2`，满足加锁条件，给segment全部加锁求和

## remove方法
replace和remove都是用了`scanAndLock`这个方法，和put方法的`scanAndLockForPut`方法思想类似
```java
final V remove(Object key, int hash, Object value) {
	if (!tryLock()) // 获取锁
		scanAndLock(key, hash); // 自旋获取锁
	V oldValue = null;
	try {
		HashEntry<K,V>[] tab = table;
		int index = (tab.length - 1) & hash; 
		HashEntry<K,V> e = entryAt(tab, index);// 找到元素
		HashEntry<K,V> pred = null;
		while (e != null) {
			K k;
			HashEntry<K,V> next = e.next; // 下一个元素
			if ((k = e.key) == key 
				|| (e.hash == hash && key.equals(k))) {
				V v = e.value; // 获取value
				// 如果value相等
				if (value == null || value == v || value.equals(v)) { 
					// 说明是头结点,让next节点为头结点即可
					if (pred == null) 
						setEntryAt(tab, index, next);
					else 
						// 说明不是头结点
						pred.setNext(next);
					++modCount;
					--count;
					oldValue = v;
				}
				break;
			}
			pred = e;
			e = next;
		}
	} finally {
		unlock();
	}
	return oldValue;
}
```
## isEmpty方法
`isEmpty` 其实也和`size`的思想类似，不过这个始终没有加锁，提高了性能，执行流程：
1. 确定map的每个segment是否为0
	- 任何一个segment的count不为0就返回
	- 都为0就累加modCount为`sum`，`modCount`指的是操作次数
2.  判断`sum`是否为0，不为0，就代表了map之前是有数据的，被`remove`和`clean`了，再次确定map的每个segment是否为0，其中任何一个segment的`count`不为0，就返回。此过程中`sum -= seg.modCount`，最后判断sum是否为0，为0就代表没有任何变化，不为0，就代表在第一次统计过程中有线程又添加了元素，所以返回false。但是如果在第二次统计时又发生了变化了，所以这个不是那么的准确，但是不加锁提高了性能，也是一个可以接受的方案。

# JDK8
在JDK1.7之前，`ConcurrentHashMap`是通过分段锁机制来实现的，所以其最大并发度受Segment的个数限制。因此，在JDK1.8中，`ConcurrentHashMap`的实现原理摒弃了这种设计，而是选择了与HashMap类似的数组+链表+红黑树的方式实现，而加锁则采用CAS和`synchronized`实现。

## 为什么 `key` 和 `value` 不允许为 `null`
在并发编程中，`null` 值容易引来歧义， 假如先调用 `get(key)` 返回的结果是 `null`，那么我们无法确认是因为当时这个 `key` 对应的 `value` 本身放的就是 `null`，还是说这个 `key` 值根本不存在，这会引起歧义，如果在非并发编程中，可以进一步通过调用 `containsKey` 方法来进行判断，但是并发编程中无法保证两个方法之间没有其他线程来修改 `key` 值，所以就直接禁止了 `null` 值的存在。

## sizeCtl 的四个含义
1.   `sizeCtl<-1` 表示有 `N-1` 个线程正在执行扩容操作，如 `-2` 就表示有 `2-1` 个线程正在扩容。
2. `sizeCtl=-1` 占位符，表示当前正在初始化数组。
3. `sizeCtl=0` 默认状态，表示数组还没有被初始化。
4. `sizeCtl>0` 记录下一次需要扩容的大小。

## 数据结构
![[Pasted image 20220501165451.png]]

## 初始化
```java
// 这构造函数里，什么都不干
public ConcurrentHashMap() {

}
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```
初始化方法通过提供初始容量，计算了`sizeCtl =  (1.5 * initialCapacity + 1)`，然后`sizeCtl`向上取最近的 2 的 n 次方。

## put 过程分析
```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 得到 hash 值
    int hash = spread(key.hashCode());
    // 用于记录相应链表的长度
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
	    // f:hash对应桶的第一个节点
        Node<K,V> f; 
        // n:table长度
        // i:hash对应的下标
        // fh:头节点的hash
        int n, i, fh;
        
        // 如果数组空，进行数组初始化
        if (tab == null || (n = tab.length) == 0)
            // 初始化数组，后面会详细介绍
            tab = initTable();

        // 找该 hash 值对应的数组下标，得到第一个节点 f
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果数组该位置为空，
            // 用一次 CAS 操作将这个新值放入其中即可
            // 如果 CAS 失败，那就是有并发操作，进到下一个循环就好了
            if (casTabAt(tab, i, null, 
	            new Node<K,V>(hash, key, value, null)))
                break;                   
        }
        // hash 等于 MOVED，在扩容
        else if ((fh = f.hash) == MOVED)
            // 帮助数据迁移，这个等到看完数据迁移部分的介绍后，再理解这个就很简单了
            tab = helpTransfer(tab, f);
        // 到这里就是说，f 是该位置的头节点，而且不为空
        else { 

            V oldVal = null;
            // 获取数组该位置的头节点的监视器锁
            synchronized (f) {
	            // 保证获得了正确的锁
                if (tabAt(tab, i) == f) {
	                // 头节点的 hash 值大于 0，说明是链表
                    if (fh >= 0) { 
                        // 用于累加，记录链表的长度
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果发现了相等的 key，判断是否要进行值覆盖
                            // 然后也就可以结束
                            if (e.hash == hash 
	                            && ((ek = e.key) == key 
	                            || (ek != null 
	                            && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 到了链表的最末端，将这个新值放到链表的最后面
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = 
	                                new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        // 调用红黑树的插值方法插入新节点
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(
	                        hash, key, value)) != null) {
	                        
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }

            if (binCount != 0) {
                // 判断是否要将链表转换为红黑树，临界值和 HashMap 一样，也是 8
                if (binCount >= TREEIFY_THRESHOLD)
                    // 和 HashMap 中稍微有点不同，它不是一定会进行红黑树转换
                    // 如果当前数组的长度小于 64，那么会选择进行数组扩容
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```
### put 操作如何保证数组元素的可见性

`ConcurrentHashMap` 中存储数据采用的 `Node` 数组是采用了 `volatile` 来修饰的，但是这只能保证数组的引用在不同线程之间是可用的，并不能保证数组内部的元素在各个线程之间也是可见的，所以这里我们判定某一个下标是否有元素，并不能直接通过下标来访问，而是通过 `tabAt` 方法来获取元素，而 `tableAt` 方法实际上就是一个 `CAS` 操作：
```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)
	    U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```
- 如果发现当前节点元素为空，也是通过 `CAS` 操作（`casTabAt`）来初始化当前元素。
- 如果当前节点元素不为空，则会使用 `synchronized` 关键字锁住当前节点，并进行设值操作

## 初始化数组: initTable
初始化一个合适大小的数组，CAS设置 sizeCtl 来解决并发。
```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        // 初始化的"功劳"被其他线程"抢去"了
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // CAS 一下，将 sizeCtl 设置为 -1，代表抢到了锁
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    // DEFAULT_CAPACITY 默认初始容量是 16
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    // 初始化数组，长度为 16 或初始化时提供的长度
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    // 将这个数组赋值给 table，table 是 volatile 的
                    table = tab = nt;
                    // 如果 n 为 16 的话，那么这里 sc = 12
                    // 其实就是 0.75 * n
                    sc = n - (n >>> 2);
                }
            } finally {
                // 设置 sizeCtl 为 sc，我们就当是 12 吧
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```
## addCount()方法
若 put 方法元素插入成功之后，则会调用此方法，传入参数为 `addCount(1L, binCount)`。这个方法的目的很简单，就是把整个 table 的元素个数加1。基于减少竞争的目的，这里做了更好的优化。让这些竞争的线程，分散到不同的对象里边，单独操作它自己的数据(计数变量)，用这样的方式尽量降低竞争。到最后需要统计 size 的时候，再把所有对象里边的计数相加就可以了。总数量用 `baseCount` 表示。分散到的对象用 `CounterCell` 表示，对象里边的计数变量用 `value` 表示。注意这里的变量都是 volatile 修饰的。
```java
//用来计数的数组，大小为2的N次幂，默认为2
private transient volatile CounterCell[] counterCells;

@sun.misc.Contended static final class CounterCell {
        volatile long value;//存储元素个数
        CounterCell(long x) { value = x; }
    }
```
当需要修改元素数量时，线程会先去 CAS 修改 `baseCount` 加1，若成功即返回。若失败，则线程被分配到某个 `CounterCell` ，然后操作 `value` 加1。若成功，则返回。否则，给当前线程重新分配一个 `CounterCell`，再尝试给 `value` 加1。（这里简略的说，实际更复杂）

`CounterCell` 会组成一个数组，也会涉及到扩容问题。

![[Pasted image 20220501173039.png]]

```java
//用来存储线程和线程生成的随机数的对应关系
static final int getProbe() {
	return UNSAFE.getInt(Thread.currentThread(), PROBE);
}

// x为1，check代表map上的元素个数
private final void addCount(long x, int check) {
	CounterCell[] as; 
	long b, s;
	 // 此处要进入if有两种情况
	 // 1.数组不为空，说明数组已经被创建好了。
	 // 2.若数组为空，说明数组还未创建，有可能竞争的线程很少，
	 // 就直接 CAS 操作 baseCount
	 // 若 CAS 成功，则方法跳转到 (2) 处，
	 // 若失败，则需要考虑给当前线程分配一个格子（指CounterCell对象）
	if ((as = counterCells) != null 
		|| !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
		CounterCell a; long v; int m;
		//字面意思，是无竞争，这里先标记为 true，表示还没有产生线程竞争
		boolean uncontended = true;
		// 这里有三种情况，会进入 fullAddCount 方法
		// 1.若数组为空，进方法 (1)
		// 2.ThreadLocalRandom.getProbe() 方法会给当前线程生成一个随机数
		// 然后用随机数与数组长度取模，计算它所在的格子。
		// 若当前线程所分配到的格子为空，进方法 (1)。
		// 若数组不为空，且线程所在格子不为空，则尝试 CAS 将此格子对应的 value ++
		// 若修改成功，则跳转到 (3)，
		// 若失败，则把 uncontended 值设为 fasle，说明产生了竞争，然后进方法 (1)
		if (as == null 
			|| (m = as.length - 1) < 0 
			|| (a = as[ThreadLocalRandom.getProbe() & m]) == null 
			|| !(uncontended = U.compareAndSwapLong(a, CELLVALUE, 
				v = a.value, v + x))) {
			//方法(1), 目的是让当前线程一定把 1 加成功。
			fullAddCount(x, uncontended);
			return;
		}
		// (3)能走到这，说明数组不为空，且修改 baseCount失败，
		// 且线程被分配到的格子不为空，且修改 value 成功。
		// 但是这里没明白为什么小于等于1，就直接返回了
		// 这里我怀疑之前的方法漏掉了binCount=0的情况。
		//而且此处若返回了，后边怎么判断扩容？（存疑）
		if (check <= 1)
			return;
		//计算总共的元素个数
		s = sumCount();
	 }
	 // (2)这里用于检查是否需要扩容
	 if (check >= 0) {
		Node<K,V>[] tab, nt; int n, sc;
		// 若元素个数达到扩容阈值，且tab不为空，且tab数组长度小于最大容量
		while (s >= (long)(sc = sizeCtl) 
			&& (tab = table) != null 
			&& (n = tab.length) < MAXIMUM_CAPACITY) {
			// 这里假设数组长度n就为16，
			// 这个方法返回的是一个固定值，用于当做一个扩容的校验标识
			// 最后有计算过程，0000 0000 0000 0000 1000 0000 0001 1011
			int rs = resizeStamp(n);
			//若sc小于0，说明正在扩容
			if (sc < 0) {
				// sc的结构类似这样，1000 0000 0001 1011 0000 0000 0000 0001
				// sc的高16位是数据校验标识
				// 低16位代表当前有几个线程正在帮助扩容,RESIZE_STAMP_SHIFT=16
				// 因此判断校验标识是否相等，不相等则退出循环
				// sc == rs + 1,sc == rs + MAX_RESIZERS 
				// 这两个应该是用来判断扩容是否已经完成，但是计算方法存疑
			    
			    // nextTable=null 说明需要扩容的新数组还未创建完成
			    // transferIndex这个参数小于等于0，说明已经不需要其它线程帮助扩容
			    // 但是并不说明已经扩容完成，因为有可能还有线程正在迁移元素
				if ((sc >>> RESIZE_STAMP_SHIFT) != rs 
					|| sc == rs + 1 
					|| sc == rs + MAX_RESIZERS 
					|| (nt = nextTable) == null 
					|| transferIndex <= 0)
				     break;
				// 到这里说明当前线程可以帮助扩容，因此sc值加一，扩容的线程数加1
				if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
				    transfer(tab, nt);
			}
			// 当sc大于0，说明sc代表扩容阈值，
			// 因此第一次扩容之前肯定走这个分支，用于初始化新表 nextTable
			// rs<<16
			// 1000 0000 0001 1011 0000 0000 0000 0000
			// +2
			// 1000 0000 0001 1011 0000 0000 0000 0010
			// 这个值，转为十进制就是 -2145714174，用于标识，
			// 这是扩容时，初始化新表的状态，
			// 扩容时，需要用到这个参数校验是否所有线程都全部帮助扩容完成。
			else if (U.compareAndSwapInt(this, SIZECTL, sc,
										 (rs << RESIZE_STAMP_SHIFT) + 2))
				//扩容，第二个参数代表新表，传入null，说明是第一次初始化新表
				transfer(tab, null);
				s = sumCount();
		}
	}
}

//计算表中的元素总个数
final long sumCount() {
	CounterCell[] as = counterCells; CounterCell a;
	//baseCount，以这个值作为累加基准
	long sum = baseCount;
	if (as != null) {
		//遍历 counterCells 数组，得到每个对象中的value值
		for (int i = 0; i < as.length; ++i) {
			if ((a = as[i]) != null)
				//累加 value 值
				sum += a.value;
		}
	}
	//此时得到的就是元素总个数
	return sum;
} 

//扩容时的校验标识
static final int resizeStamp(int n) {
	return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}

//Integer.numberOfLeadingZeros方法的作用是返回 n 的最高位为1的前面的0的个数
//n=16，
0000 0000 0000 0000 0000 0000 0001 0000
//前面有27个0，即27
0000 0000 0000 0000 0000 0000 0001 1011
//RESIZE_STAMP_BITS为16，然后 1<<(16-1)，即 1<<15
0000 0000 0000 0000 1000 0000 0000 0000
//它们做或运算，得到 rs 的值
0000 0000 0000 0000 1000 0000 0001 1011
// 扩容戳的作用
// 高 16 位代表当前扩容的标记，可以理解为一个纪元。
// 低 16 位代表了扩容的线程数。
```

首先会判断 `CounterCell` 数组是不是为空，需要这里的是，这里的 `CAS` 操作是将 `BASECOUNT` 和 `baseCount` 进行比较，如果相等，则说明当前没有其他线程过来修改 `baseCount`（即 `CAS` 操作成功），此时则不需要使用 `CounterCell` 数组，而直接采用 `baseCount` 来计数。假如 `CounterCell` 为空且 `CAS` 失败，那么就会通过调用 `fullAddCount` 方法来对 `CounterCell` 数组进行初始化。
## fullAddCount()方法
上边的 addCount 方法还没完，别忘了有可能元素个数加 1 的操作还未成功，就走到 fullAddCount 这个方法了。看方法名，就知道了，全力增加计数值，一定要成功。
```java
// 传过来的参数分别为 1 false
private final void fullAddCount(long x, boolean wasUncontended) {
	int h;
	//如果当前线程的随机数为0，则强制初始化一个值
	if ((h = ThreadLocalRandom.getProbe()) == 0) {
		ThreadLocalRandom.localInit();      // force initialization
		h = ThreadLocalRandom.getProbe();
		//此时把 wasUncontended 设为true，认为无竞争
		wasUncontended = true;
	}
	//用来表示比 contend（竞争）更严重的碰撞，若为true，表示可能需要扩容，减少碰撞冲突
	boolean collide = false;                // True if last slot nonempty
	//循环内，外层if判断分三种情况，内层判断又分为六种情况
	for (;;) {
		CounterCell[] as; CounterCell a; int n; long v;
		// 1. 若counterCells数组不为空。  
		// 建议先看下边的2和3两种情况，再回头看这个。 
		if ((as = counterCells) != null && (n = as.length) > 0) {
			// (1) 若当前线程所在的格子（CounterCell对象）为空
			if ((a = as[(n - 1) & h]) == null) {
				if (cellsBusy == 0) {    
					//若无锁，则乐观的创建一个 CounterCell 对象。
					CounterCell r = new CounterCell(x); 
					//尝试加锁
					if (cellsBusy == 0 
						&& U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
						boolean created = false;
						//加锁成功后，再 recheck 一下数组是否不为空，且当前格子为空
						try {               
							CounterCell[] rs; int m, j;
							if ((rs = counterCells) != null 
								&& (m = rs.length) > 0 
								&& rs[j = (m - 1) & h] == null) {
								//把新创建的对象赋值给当前格子
								rs[j] = r;
								created = true;
							}
						} finally {
							//手动释放锁
							cellsBusy = 0;
						}
						//若当前格子创建成功，且上边的赋值成功，
						// 则说明加1成功，退出循环
						if (created)
							break;
						//否则，继续下次循环
						continue;           // Slot is now non-empty
					}
				}
				// 若cellsBusy=1，说明有其它线程抢锁成功。
				// 或者若抢锁的 CAS 操作失败，都会走到这里，
				// 则当前线程需跳转到(9)重新生成随机数，进行下次循环判断。
				collide = false;
			}
			// 后边这几种情况，都是数组和当前随机到的格子都不为空的情况。
			// 且注意每种情况，若执行成功，且不break，continue，则都会执行(9)
			// 重新生成随机数，进入下次循环判断
			// (2) 到这，说明当前方法在被调用之前已经 CAS 失败过一次
			// 为了减少竞争，则跳转到（9）处重新生成随机数，
			// 并把 wasUncontended 设置为true ，认为下一次不会产生竞争
			else if (!wasUncontended)       // CAS already known to fail
				wasUncontended = true;      // Continue after rehash
			// (3) 若 wasUncontended 为 true 无竞争，则尝试一次 CAS。
			// 若成功，则结束循环，若失败则判断后边的 (4)(5)(6)。
			else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
				break;
			// (4)(5)(6)都是 wasUncontended=true，且CAS修改value失败的情况。
			// 若数组有变化，或数组长度大于等于CPU的核心数，则把 collide 改为 false
			// 因为数组若有变化，说明是由扩容引起的；
			// 长度超限，则说明已经无法扩容，只能认为无碰撞。
			// 这里很有意思，认真思考一下，当扩容超限后，则会达到一个平衡，
			// 即 (4)(5) 反复执行，直到 (3) 中CAS成功，跳出循环。
			else if (counterCells != as || n >= NCPU)
				collide = false;            // At max size or stale
			// (5) 若数组无变化，且数组长度小于CPU核心数时，且 collide 为 false，
			// 就把它改为 true，说明下次循环可能需要扩容
			else if (!collide)
				collide = true;
			// (6) 若数组无变化，且数组长度小于CPU核心数时，且 collide 为 true，
			// 说明冲突比较严重，需要扩容了。
			else if (cellsBusy == 0 
				&& U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
				try {
					//recheck
					if (counterCells == as) {// Expand table unless stale
					//创建一个容量为原来两倍的数组
					CounterCell[] rs = new CounterCell[n << 1];
					//转移旧数组的值
					for (int i = 0; i < n; ++i)
						rs[i] = as[i];
					//更新数组
					counterCells = rs;
				}
			} finally {
				cellsBusy = 0;
			}
			//认为扩容后，下次不会产生冲突了，和(4)处逻辑照应
			collide = false;
			//当次扩容后，就不需要重新生成随机数了
			continue;                   // Retry with expanded table
		}
		// (9)，重新生成一个随机数，进行下一次循环判断
		h = ThreadLocalRandom.advanceProbe(h);
	}
	// 2.这里的 cellsBusy 参数是一个volatile的 int值，用来表示自旋锁的标志，
	// 可以类比 AQS 中的 state 参数，用来控制锁之间的竞争，并且是独占模式。
	// cellsBusy 若为0，说明无锁，线程都可以抢锁，
	// 若为1，表示已经有线程拿到了锁，则其它线程不能抢锁。
	else if (cellsBusy == 0 
		&& counterCells == as 
		&& U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
		boolean init = false;
		try {    
			//这里再重新检测下 counterCells 数组引用是否有变化
			if (counterCells == as) {
				//初始化一个长度为 2 的数组
				CounterCell[] rs = new CounterCell[2];
				//根据当前线程的随机数值，计算下标，只有两个结果0或1，并初始化对象
				rs[h & 1] = new CounterCell(x);
				//更新数组引用
				counterCells = rs;
				//初始化成功的标志
				init = true;
			}
		} finally {
			//别忘了，需要手动解锁。
			cellsBusy = 0;
		}
		//若初始化成功，则说明当前加1的操作也已经完成了，则退出整个循环。
		if (init)
			break;
	}
	//3.到这，说明数组为空，且 2 抢锁失败，则尝试直接去修改 baseCount 的值，
	//若成功，也说明加1操作成功，则退出循环。
	else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
		break;                          // Fall back on using base
	}
}
```

## 链表转红黑树: treeifyBin
`treeifyBin` 不一定就会进行红黑树转换，也可能是仅仅做数组扩容
```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
	// 头节点
    Node<K,V> b; 
    // n 数组长度
    int n, sc;
    if (tab != null) {
        // MIN_TREEIFY_CAPACITY 为 64
        // 如果数组长度小于 64 的时候，会进行数组扩容
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
	        // 扩容
            tryPresize(n << 1);
        // b 是头节点
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            // 加锁
            synchronized (b) {
	            // 双重校验
                if (tabAt(tab, index) == b) {
                    // 下面就是遍历链表，建立一颗红黑树
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val, null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                    // 将红黑树设置到数组相应位置中
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

## 扩容: tryPresize
这里的扩容也是做翻倍扩容的，扩容后数组容量为原来的 2 倍。

```java
// 方法参数 size 传进来的时候就已经翻倍了
private final void tryPresize(int size) {
	//c是一个大于等于（size * 1.5 + 1）的2的幂次方数
	int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
		tableSizeFor(size + (size >>> 1) + 1);
	int sc;
	//此时的sizeCtl是cap * 0.75，扩容阈值
	while ((sc = sizeCtl) >= 0) {
		Node<K,V>[] tab = table; int n;
		if (tab == null || (n = tab.length) == 0) {
			n = (sc > c) ? sc : c;
			//如果此时tab为空，则进行初始化，将sizeCtl设为-1
			if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
				try {
					if (table == tab) {
						@SuppressWarnings("unchecked")
						Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
						table = nt;
						//sc = n * 0.75
						sc = n - (n >>> 2);
					}
				} finally {
					sizeCtl = sc;
				}
			}
		}
		else if (c <= sc || n >= MAXIMUM_CAPACITY)
			break;
		//tab == table说明还没开始迁移节点
		else if (tab == table) {
			int rs = resizeStamp(n);
			//开始迁移原tab[] 中的数据，此时将sizeCtl设置为一个很大的负值
			if (U.compareAndSwapInt(this, SIZECTL, sc,
									(rs << RESIZE_STAMP_SHIFT) + 2))
				//第一个发起迁移的线程，传入nextTable为null
				transfer(tab, null);
		}
	}
}
```

这个方法的核心在于 `sizeCtl` 值的操作，首先将其设置为一个负数，然后执行 `transfer(tab, null)`，再下一个循环将 `sizeCtl` 加 1，并执行 `transfer(tab, nt)`。可能的操作就是执行 1 次 `transfer(tab, null)` + 多次 `transfer(tab, nt)`

## transfer()方法
需要说明的一点是，虽然我们一直在说帮助扩容，其实更准确的说应该是帮助迁移元素。因为扩容的第一次初始化新表（扩容后的新表）这个动作，只能由一个线程完成。其他线程都是在帮助迁移元素到新数组。

这里还是先看下迁移的示意图，帮助理解。

![[Pasted image 20220501183744.png]]

为了方便，上边以原数组长度 8 为例。在元素迁移的时候，所有线程都遵循从后向前推进的规则，即如图A线程是第一个进来的线程，会从下标为7的位置，开始迁移数据。

而且当前线程迁移时会确定一个范围，限定它此次迁移的数据范围，如图 A 线程只能迁移 `bound=6`到`i=7` 这两个数据。

此时，其它线程就不能迁移这部分数据了，只能继续向前推进，寻找其它可以迁移的数据范围，且每次推进的步长为固定值 stride（此处假设为2）。如图中 B线程发现 A 线程正在迁移6,7的数据，因此只能向前寻找，然后迁移 `bound=4` 到 `i=5` 的这两个数据。

当每个线程迁移完成它的范围内数据时，都会继续向前推进。那什么时候是个头呢？

这就需要维护一个全局的变量 `transferIndex`，来表示所有线程总共推进到的元素下标位置。如图，线程 A 第一次迁移成功后又向前推进，然后迁移2,3 的数据。此时，若没有其他线程在帮助迁移，则 transferIndex 即为2。

剩余部分等待下一个线程来迁移，或者有任何的 A 和B线程已经迁移完成，也可以推进到这里帮助迁移。直到` transferIndex=0`。（会做一些其他校验来判断是否迁移全部完成，看代码）。

```java
//这个类是一个标志，用来代表当前桶（数组中的某个下标位置）的元素已经全部迁移完成
static final class ForwardingNode<K,V> extends Node<K,V> {
	final Node<K,V>[] nextTable;
	ForwardingNode(Node<K,V>[] tab) {
	//把当前桶的头结点的 hash 值设置为 -1，表明已经迁移完成，
	//这个节点中并不存储有效的数据
	super(MOVED, null, null, null);
	this.nextTable = tab;
	}
}

//迁移数据
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
	int n = tab.length, stride;
	//根据当前CPU核心数，确定每次推进的步长，最小值为16.（为了方便我们以2为例）
	if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
		stride = MIN_TRANSFER_STRIDE; // subdivide range
	//从 addCount 方法，只会有一个线程跳转到这里，初始化新数组
	if (nextTab == null) {            // initiating
		try {
			@SuppressWarnings("unchecked")
			//新数组长度为原数组的两倍
			Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
			nextTab = nt;
		} catch (Throwable ex) {      // try to cope with OOME
			sizeCtl = Integer.MAX_VALUE;
			return;
		}
		//用 nextTable 指代新数组
		nextTable = nextTab;
		//这里就把推进的下标值初始化为原数组长度（以16为例）
		transferIndex = n;
	}
	//新数组长度
	int nextn = nextTab.length;
	//创建一个标志类
	ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
	//是否向前推进的标志
	boolean advance = true;
	//是否所有线程都全部迁移完成的标志
	boolean finishing = false; // to ensure sweep before committing nextTab
	//i 代表当前线程正在迁移的桶的下标，bound代表它本次可以迁移的范围下限
	for (int i = 0, bound = 0;;) {
		Node<K,V> f; int fh;
		//需要向前推进
		while (advance) {
			int nextIndex, nextBound;
			// (1)
			// 先看 (3) 
			// i每次自减 1，直到 bound。
			// 若超过bound范围，或者finishing标志为true，则不用向前推进。
			// 若未全部完成迁移，且 i 并未走到 bound，
			// 则跳转到 (7)，处理当前桶的元素迁移。
			if (--i >= bound || finishing)
				advance = false;
			// (2) 每次执行，都会把 transferIndex 最新的值同步给 nextIndex
			// 若 transferIndex 小于等于0，
			// 则说明原数组中的每个桶位置，都有线程在处理迁移了，
			// 于是，需要跳出while循环，并把 i设为 -1，
			// 以跳转到 (4) 判断在处理的线程是否已经全部完成。
			else if ((nextIndex = transferIndex) <= 0) {
				i = -1;
				advance = false;
			}
			// (3) 第一个线程会先走到这里，确定它的数据迁移范围。
			// (2)处会更新 nextIndex为 transferIndex 的最新值
			// 因此第一次 nextIndex=n=16，
			// nextBound代表当次迁移的数据范围下限，减去步长即可，
			// 所以，第一次时，nextIndex=16，nextBound=16-2=14。
			// 后续，每次都会间隔一个步长。
			else if (U.compareAndSwapInt
				(this, TRANSFERINDEX, nextIndex, 
				nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
				//bound代表当次数据迁移下限
				bound = nextBound;
				//第一次的i为15，因为长度16的数组，最后一个元素的下标为15
				i = nextIndex - 1;
				// 表明不需要向前推进
				// 只有当把当前范围内的数据全部迁移完成后，才可以向前推进
				advance = false;
			}
		}
		// (4)
		if (i < 0 || i >= n || i + n >= nextn) {
			int sc;
			//若全部线程迁移完成
			if (finishing) {
				nextTable = null;
				//更新table为新表
				table = nextTab;
				//扩容阈值改为原来数组长度的 3/2 
				// 即新长度的 3/4，也就是新数组长度的0.75倍
				sizeCtl = (n << 1) - (n >>> 1);
				return;
			}
			//到这，说明当前线程已经完成了自己的所有迁移（无论参与了几次迁移），
			//则把 sc 减1，表明参与扩容的线程数减少 1。
			if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
				// 在 addCount 方法最后
				// 我们强调，迁移开始时，会设置 sc=(rs << RESIZE_STAMP_SHIFT) + 2
				// 每当有一个线程参与迁移，sc 就会加 1，
				// 每当有一个线程完成迁移，sc 就会减 1。
				// 因此，这里就是去校验当前 sc 是否和初始值是否相等。
				// 相等，则说明全部线程迁移完成。
				if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
					return;
				//只有此处，才会把finishing 设置为true。
				finishing = advance = true;
				// 这里非常有意思，会把 i 从 -1 修改为16，
				// 目的就是，让 i 再从后向前扫描一遍数组，
				// 检查是否所有的桶都已被迁移完成，参看 (6)
				i = n; // recheck before commit
			}
		}
		// (5) 若i的位置元素为空，
		// 则说明当前桶的元素已经被迁移完成，就把头结点设置为fwd标志。
		else if ((f = tabAt(tab, i)) == null)
			advance = casTabAt(tab, i, null, fwd);
		// (6) 若当前桶的头结点是 ForwardingNode ，说明迁移完成，则向前推进 
		else if ((fh = f.hash) == MOVED)
			advance = true; // already processed
		//(7) 处理当前桶的数据迁移。
		else {
			synchronized (f) {  //给头结点加锁
				if (tabAt(tab, i) == f) {
					Node<K,V> ln, hn;
					//若hash值大于等于0，则说明是普通链表节点
					if (fh >= 0) {
						int runBit = fh & n;
						// 这里是 1.7 的 CHM 的 rehash 方法
						// 和 1.8 HashMap的 resize 方法的结合体。
						// 会分成两条链表，
						// 一条链表和原来的下标相同，
						// 另一条链表是原来的下标加数组长度的位置
						// 然后找到 lastRun 节点，从它到尾结点整体迁移。
						// lastRun前边的节点则单个迁移，但需要注意的是，这里是头插法
						// 另外还有一点和1.7不同，
						// 1.7 lastRun前边的节点是复制过去的，
						// 而这里是直接迁移的，没有复制操作。
						// 所以，最后会有两条链表，
						// 一条链表从 lastRun到尾结点是正序的，
						// 而lastRun之前的元素是倒序的，
						// 另外一条链表，从头结点开始就是倒叙的。看下图。
						Node<K,V> lastRun = f;
						for (Node<K,V> p = f.next; p != null; p = p.next) {
							int b = p.hash & n;
							if (b != runBit) {
								runBit = b;
								lastRun = p;
							}
						}
						if (runBit == 0) {
							ln = lastRun;
							hn = null;
						}
						else {
							hn = lastRun;
							ln = null;
						}
						for (Node<K,V> p = f; p != lastRun; p = p.next) {
							int ph = p.hash; K pk = p.key; V pv = p.val;
							if ((ph & n) == 0)
								ln = new Node<K,V>(ph, pk, pv, ln);
							else
								hn = new Node<K,V>(ph, pk, pv, hn);
						}
						setTabAt(nextTab, i, ln);
						setTabAt(nextTab, i + n, hn);
						setTabAt(tab, i, fwd);
						advance = true;
					}
					//树节点
					else if (f instanceof TreeBin) {
						TreeBin<K,V> t = (TreeBin<K,V>)f;
						TreeNode<K,V> lo = null, loTail = null;
						TreeNode<K,V> hi = null, hiTail = null;
						int lc = 0, hc = 0;
						for (Node<K,V> e = t.first; e != null; e = e.next) {
							int h = e.hash;
							TreeNode<K,V> p = new TreeNode<K,V>
								(h, e.key, e.val, null, null);
							if ((h & n) == 0) {
								if ((p.prev = loTail) == null)
									lo = p;
								else
									loTail.next = p;
								loTail = p;
								++lc;
							}
							else {
								if ((p.prev = hiTail) == null)
									hi = p;
								else
									hiTail.next = p;
								hiTail = p;
								++hc;
							}
						}
						ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
							(hc != 0) ? new TreeBin<K,V>(lo) : t;
						hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
							(lc != 0) ? new TreeBin<K,V>(hi) : t;
						setTabAt(nextTab, i, ln);
						setTabAt(nextTab, i + n, hn);
						setTabAt(tab, i, fwd);
						advance = true;
					}
				}
			}
		}
	}
}
```

迁移后的新数组链表方向示意图，以 runBit =0 为例。

![[Pasted image 20220501185406.png]]

## helpTransfer()方法

最后再看 put 方法中的这个方法，就比较简单了。

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
	Node<K,V>[] nextTab; int sc;
	//头结点为 ForwardingNode ，并且新数组已经初始化
	if (tab != null 
		&& (f instanceof ForwardingNode) 
		&& (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
		int rs = resizeStamp(tab.length);
		while (nextTab == nextTable 
			&& table == tab 
			&& (sc = sizeCtl) < 0) {
			//若校验标识失败，或者已经扩容完成，或推进下标到头，则退出
			if ((sc >>> RESIZE_STAMP_SHIFT) != rs 
				|| sc == rs + 1 
				|| sc == rs + MAX_RESIZERS 
				|| transferIndex <= 0)
				break;
			//当前线程需要帮助迁移，sc值加1
			if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
				transfer(tab, nextTab);
				break;
			}
		}
		return nextTab;
	}
	return table;
}
```

## 对比总结
-   `HashTable` : 使用了synchronized关键字对put等操作进行加锁;
-   `ConcurrentHashMap JDK1.7`: 使用分段锁机制实现;
-   `ConcurrentHashMap JDK1.8`: 则使用数组+链表+红黑树数据结构和CAS原子操作实现;
