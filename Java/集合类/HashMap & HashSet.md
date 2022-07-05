转载自`https://pdai.tech/`
# HashSet
`HashSet`和`HashMap`在Java中有相同的实现，前者仅仅是对后者做了一层包装，也就是说`HashSet`里面有一个`HashMap`(适配器模式)。
```java
//HashSet是对HashMap的简单包装
public class HashSet<E>
{
	private transient HashMap<E, Object> map;
	
    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
	
    public HashSet() {
        map = new HashMap<>();
    }
	
	//简单的方法转换
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

}
```
# HashMap
## Java7
### 概述
`HashMap`实现了Map接口，即允许放入`key`为`null`的元素，也允许插入`value`为`null`的元素；除该类未实现同步外，其余跟`Hashtable`大致相同；跟`TreeMap`不同，该容器不保证元素顺序，根据需要该容器可能会对元素重新哈希，元素的顺序也会被重新打散，因此不同时间迭代同一个`HashMap`的顺序可能会不同。 

根据对冲突的处理方式不同，哈希表有两种实现方式，一种开放地址方式，另一种是冲突链表方式。Java7中`HashMap`采用的是冲突链表方式。

有两个参数可以影响`HashMap`的性能: 初始容量(inital capacity)和负载系数(load factor)。初始容量指定了初始`table`的大小，负载系数用来指定自动扩容的临界值。当`entry`的数量超过`capacity*load_factor`时，容器将自动扩容并重新哈希。对于插入元素较多的场景，将初始容量设大可以减少重新哈希的次数。

---
将对象放入到`HashMap`或`HashSet`中时，有两个方法需要特别关心: `hashCode()`和`equals()`。`hashCode()`方法决定了对象会被放到哪个`bucket`里，当多个对象的哈希值冲突时，`equals()`方法决定了这些对象是否是同一个对象。所以，如果要将自定义的对象放入到`HashMap`或`HashSet`中，需要覆盖`hashCode()`和`equals()`方法。

### hash扰动
`HashMap`为对象分配槽的时候使用`hashCode & (length - 1)` 这个操作来进行取余操作，所以`hashCode` 只有低位跟`(length — 1)`发生 & 操作，参与到槽位的分配，高位参与不到，可能导致槽位分配不均匀，所以需要进行`hash`扰动。
```java
final int hash(Object k) {
	
	//获得hash种子，默认为0
	int h = hashSeed;
	if (0 != h && k instanceof String) {
		return sun.misc.Hashing.stringHash32((String) k);
	}
	//0 异或 key的hashcode ，结果不变
	h ^= k.hashCode();
	//h >>> 的过程就是让高位参与hash槽计算
	//异或操作扰动hash
	h ^= (h >>>  20) ^ (h >>> 12);
	return h ^ (h >>> 7) ^ (h >>> 4);
}

```
### `get`方法
```java
final Entry<K,V> getEntry(Object key) {
	
	...
	
	int hash = (key == null) ? 0 : hash(key);
	//依次遍历冲突链表中的每个entry
    for (Entry<K,V> e = table[hash & (table.length - 1)]; e != null; e = e.next) {
        Object k;
        //依据equals()方法判断是否相等
        if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```
### `put`方法
```java
static int indexFor(int h, int length) { 
	return h & (length-1); 
}

public V put(K key, V value) {
	// 刚开始没有设置数组大小，就会进入inflateTable方法初始化数组大小
	// threshold 扩容的阈值 默认16
	if (table == EMPTY_TABLE) {
		inflateTable(threshold);
	}
	if (key == null)
		//hashmap是可以传入null作为key的
		//key = null hash为0 放到数组的第一位
		return putForNullKey(value);
	
	//根据key获得到经过扰动的hashcode            
	int hash = hash(key);
	//根据hashcode和数组长度获取hash槽位        
	int i = indexFor(hash, table.length);
	//遍历所在的hash槽位上的链表，如果存在相同的key就替代   
	//什么才算是相同的key，这就涉及到为什么重写hashCode之后还要重新equals
	//通过比较hashCode是否相等，之后还需要比较==或者通过equals比较是否相同
	for (Entry<K,V> e = table[i]; e != null; e = e.next) {
		Object k;
		if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
			V oldValue = e.value;
			e.value = value;
			e.recordAccess(this);
			return oldValue;
		}
	}
	modCount++;
	//添加元素        
	addEntry(hash, key, value, i);
	return null;
}

void addEntry(int hash, K key, V value, int bucketIndex) {
	//数组扩容条件  (size >= threshold) && (null != table[bucketIndex])
	//数组元素大小大于等于阈值同时需要put的头节点不为空
	if ((size >= threshold) && (null != table[bucketIndex])) {
		//扩容大小是原来数组大小的两倍
		resize(2 * table.length);
		//扩容之后需要重新计算hash值    
		hash = (null != key) ? hash(key) : 0;
		//扩容之后重新计算槽位    
		bucketIndex = indexFor(hash, table.length);
	}
	//新建新的节点
	createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
	//头插法插入数据
	Entry<K,V> e = table[bucketIndex];
	table[bucketIndex] = new Entry<>(hash, key, value, e);
	size++;
}
```
在向链表中插入新的`entry`的时候，有两种选择：头插法和尾插法。Java7为了简单省时选择了头插法这样也就意味着带来了多线程扩容问题：循环链表，导致后面如果有`get`或者`put`需要遍历链表的时候就会出现死循环。

### 扩容
```java
void resize(int newCapacity) {
	Entry[] oldTable = table;
	int oldCapacity = oldTable.length;
	if (oldCapacity == MAXIMUM_CAPACITY) {
		threshold = Integer.MAX_VALUE;
		return;
	}
	//新建一个数组
	Entry[] newTable = new Entry[newCapacity];
	//进行数据迁移        
	transfer(newTable, initHashSeedAsNeeded(newCapacity));
	//新数组赋值给老数组        
	table = newTable;
	//扩容之后重新设置阈值        
	threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}

void transfer(Entry[] newTable, boolean rehash) {
	int newCapacity = newTable.length;
	//两层for循环
	//第一层针对数组
	//第二层针对链表
	for (Entry<K,V> e : table) {
		while(null != e) {
			Entry<K,V> next = e.next;
			if (rehash) {
				e.hash = null == e.key ? 0 : hash(e.key);
			}
			//重新计算操作
			int i = indexFor(e.hash, newCapacity);
			e.next = newTable[i];
			newTable[i] = e;
			e = next;
		}
	}
}
```
总结：
1. 扩容数组长度为原来的两倍 
2. 扩容是新建一个新的数组，hashmap直接引用新的数组对象 
3. 扩容之后需要重新设置阈值 
4. 数组扩容之后，迁移数据，槽位要么是在原来的地方，要么是原来的槽位+原来数组长度
5. 扩容多线程会产生循环链表，再次遍历的时候就会死循环

### 死循环问题
```java
void transfer(Entry[] newTable, boolean rehash) {
	int newCapacity = newTable.length;
	for (Entry<K,V> e : table) {
		while(null != e) {
			Entry<K,V> next = e.next;
			// 对于线程 T1 和 T2 都在这里挂起
			// 等T1推出循环之后，T2再执行
			if (rehash) {
				e.hash = null == e.key ? 0 : hash(e.key);
			}
			//重新计算操作
			int i = indexFor(e.hash, newCapacity);
			e.next = newTable[i];
			newTable[i] = e;
			e = next;
		}
	}
}
```
#### 1.都挂起
![](HashMap%20&%20HashSet/Pasted%20image%2020220705223414.png)
#### 2. T1执行结束
![](HashMap%20&%20HashSet/Pasted%20image%2020220705223451.png)
#### 3. T2开始执行
##### 第一个循环
```java
if (rehash) {
	e.hash = null == e.key ? 0 : hash(e.key);
}
//重新计算操作
int i = indexFor(e.hash, newCapacity);
// A 的 next 指向null，因为这里 newTable 是T2建立的，不是T1的
e.next = newTable[i];
// T2 的 newTable[i] = A
newTable[i] = e;
// e 变成 B
e = next;
```
![](HashMap%20&%20HashSet/Pasted%20image%2020220705225803.png)
##### 第二个循环
```java
// e 为 B 
// next 为 A
Entry<K,V> next = e.next;
if (rehash) {
	e.hash = null == e.key ? 0 : hash(e.key);
}
//重新计算操作
int i = indexFor(e.hash, newCapacity);
// B 的 next 设置为 A
e.next = newTable[i];
// T2 的 newTable[i] = B
newTable[i] = e;
// e 变成 A
e = next;
```
![](HashMap%20&%20HashSet/Pasted%20image%2020220705230358.png)
##### 第三轮循环
```java
// e 为 A
// next 为 null
Entry<K,V> next = e.next;
if (rehash) {
	e.hash = null == e.key ? 0 : hash(e.key);
}
//重新计算操作
int i = indexFor(e.hash, newCapacity);
// A 的 next 设置为 B 循环生成
e.next = newTable[i];
// T1 的 newTable[i] = A
newTable[i] = e;
// e 变成 null 结束循环
e = next;
```
![](HashMap%20&%20HashSet/Pasted%20image%2020220705230820.png)

## Java8
### 概述
![[HashMap_java8_struct.png]]

Java8 对 `HashMap` 进行了一些修改，最大的不同就是利用了红黑树，所以由数组+链表+红黑树组成。
```java
// 序列化Id,版本号
private static final long serialVersionUID = 362498820763181265L; 
// 默认Map容量为16，移位运算，向左移4位，所以是16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16
// 最大容量，向左移30位
static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认的填充因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 当桶(bucket)上的结点数大于这个值时会转成红黑树
static final int TREEIFY_THRESHOLD = 8;
// 当桶(bucket)上的结点数小于这个值时红黑树将还原回链表
static final int UNTREEIFY_THRESHOLD = 6;
// 桶中结构转化为红黑树对应的数组的最小大小，如果当前容量小于它，就不会将链表转化为红黑树，而是用resize()代替	
static final int MIN_TREEIFY_CAPACITY = 64;
// 存储元素的数组，大小总是2的幂
transient java.util.HashMap.Node<K,V>[] table;
// 存放具体元素的集合
transient Set<Map.Entry<K,V>> entrySet;
// 集合目前的大小,并非数组的length,即当前Map存储了多少个元素
transient int size;
// 计数器，版本号，每次修改就会+1
transient int modCount;
// 临界值,当实际节点个数超过临界值(容量*填充因子)时，就会进行扩容
int threshold;
// 填充因子
final float loadFactor;
```
### hash扰动
```java
static final int hash(Object key) {
	int h;
	/**
	 * 1. key为null,则hash值为0
	 * 2. key不为null,执行key的hashcode方法(得到的hashcode值需要进行异或计算)
	 */
	return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
	/**
	 * 1. h = key.hashCode() 第一步：取hashcode值  
	 * 2. h^(h>>>16)         第二步：高位参与运算
	 */
}
```
### `put`过程分析
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// 第四个参数 onlyIfAbsent 如果是 true，那么只有在不存在该 key 时才会进行 put 操作
// 第五个参数 evict 我们这里不关心
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
	// tab:hashMap的数组
	// p:key对应的具体Node
	// n:hashMap数组长度
	// i:hash值
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 第一次 put 值的时候，会触发下面的 resize()
    // 类似 java7 的第一次 put 也要初始化数组长度
    // 第一次 resize 和后续的扩容有些不一样
    // 这次是数组从 null 初始化到默认的 16 或自定义的初始容量
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 找到具体的数组下标，如果此位置没有值，
    // 那么直接初始化一下 Node 并放置在这个位置就可以了
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 首先，判断该位置的第一个数据和我们要插入的数据，key 是不是相等
        // 如果是，取出这个节点
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果该节点是代表红黑树的节点，调用红黑树的插值方法
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 到这里，说明数组该位置上是一个链表
            for (int binCount = 0; ; ++binCount) {
                // 插入到链表的最后面(Java7 是插入到链表的最前面)
                // 也就避免了死循环
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // TREEIFY_THRESHOLD 为 8，如果新插入的值是链表中的第 8 个
                    // 会触发下面的 treeifyBin，也就是将链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果在该链表中找到了相等的 key (== 或 equals)
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 此时 e 为链表中与要插入的新值的key相等的 node
                    break;
                p = e;
            }
        }
        // e!=null 说明存在旧值的key与要插入的key相等
        // 对于我们分析的put操作，下面这个 if 其实就是进行值覆盖，然后返回旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 如果 HashMap 由于新插入这个值导致 size 已经超过了阈值，需要进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
### 扩容
```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
	// 对应数组扩容
    if (oldCap > 0) { 
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 将数组大小扩大一倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 将阈值也扩大一倍
            newThr = oldThr << 1; // double threshold
    }
	// 对应使用 new HashMap(int initialCapacity) 初始化后，第一次 put 的时候
    else if (oldThr > 0) 
        newCap = oldThr;
	// 对应使用 new HashMap() 初始化后，第一次 put 的时候
    else {
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }

    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;

    // 用新的数组大小初始化新的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab; // 如果是初始化数组，到这里就结束了，返回 newTab 即可

    if (oldTab != null) {
        // 开始遍历原数组，进行数据迁移。
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果该数组位置上只有单个元素，那就简单了，简单迁移这个元素就可以了
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果是红黑树
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                    // 这块是处理链表的情况，
                    // 需要将此链表拆成两个链表，放到新的数组中，并且保留原来的先后顺序
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        // 第一条链表
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 第二条链表的新的位置是 j + oldCap，这个很好理解
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

# LinkedHashMap & LinkedHashSet
`LinkedHashMap`是`HashMap`的直接子类，二者唯一的区别是`LinkedHashMap`在`HashMap`的基础上，采用双向链表的形式将所有`entry`连接起来，这样是为保证元素的迭代顺序跟插入顺序相同。

`LinkedHashMap`除了可以保证迭代顺序外，还有一个非常有用的用法: 可以轻松实现一个采用了FIFO替换策略的缓存。具体说来，`LinkedHashMap`有一个子类方法`protected boolean removeEldestEntry(Map.Entry<K,V> eldest)`，该方法的作用是告诉Map是否要删除最早插入的`Entry`，如果该方法返回`true`，最老的那个元素就会被删除。在每次插入新元素的之后`LinkedHashMap`会自动询问removeEldestEntry()是否要删除最老的元素。这样只需要在子类中重载该方法，当元素个数超过一定数量时让removeEldestEntry()返回true，就能够实现一个固定大小的FIFO策略的缓存。

```java
/** 一个固定大小的FIFO替换策略的缓存 */
class FIFOCache<K, V> extends LinkedHashMap<K, V>{
    private final int cacheSize;
    public FIFOCache(int cacheSize){
        this.cacheSize = cacheSize;
    }

    // 当Entry个数超过cacheSize时，删除最老的Entry
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
       return size() > cacheSize;
    }
}
```

# TreeSet & TreeMap
`TreeMap`底层通过红黑树实现，也就意味着`containsKey()`, `get()`, `put()`, `remove()`都有着`log(n)`的时间复杂度。出于性能原因，`TreeMap`是非同步的，如果需要在多线程环境使用，需要程序员手动同步；或者通过如下方式将`TreeMap`包装成同步的:
`SortedMap m = Collections.synchronizedSortedMap(new TreeMap());`