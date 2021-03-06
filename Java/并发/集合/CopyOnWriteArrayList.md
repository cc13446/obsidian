转载自`https://pdai.tech/`
# 类的继承关系

`CopyOnWriteArrayList`实现了List接口，List接口定义了对列表的基本操作；同时实现了RandomAccess接口，表示可以随机访问(数组具有随机访问的特性)；同时实现了`Cloneable`接口，表示可克隆；同时也实现了`Serializable`接口，表示可被序列化。

```java
public class CopyOnWriteArrayList<E> implements List<E>, RandomAccess, Cloneable, java.io.Serializable {}
```

# 类的内部类
## COWIterator类
`COWIterator`表示迭代器，其也有一个`Object`类型的数组作为`CopyOnWriteArrayList`数组的快照，这种快照风格的迭代器方法在创建迭代器时使用了对当时数组状态的引用。此数组在迭代器的生存期内不会更改，因此不可能发生冲突，并且迭代器保证不会抛出异常。创建迭代器以后，迭代器就不会反映列表的添加、移除或者更改。在迭代器上进行的元素更改操作不受支持。这些方法将抛出 `UnsupportedOperationException`。

```java
static final class COWIterator<E> implements ListIterator<E> {
    /** Snapshot of the array */
    private final Object[] snapshot;
    /** Index of element to be returned by subsequent call to next.  */
    private int cursor;
    // 构造函数
    private COWIterator(Object[] elements, int initialCursor) {
        cursor = initialCursor;
        snapshot = elements;
    }
    // 是否还有下一项
    public boolean hasNext() {
        return cursor < snapshot.length;
    }
    // 是否有上一项
    public boolean hasPrevious() {
        return cursor > 0;
    }
    // next项
    @SuppressWarnings("unchecked")
    public E next() {
        if (! hasNext()) // 不存在下一项，抛出异常
            throw new NoSuchElementException();
        // 返回下一项
        return (E) snapshot[cursor++];
    }

    @SuppressWarnings("unchecked")
    public E previous() {
        if (! hasPrevious())
            throw new NoSuchElementException();
        return (E) snapshot[--cursor];
    }
    
    // 下一项索引
    public int nextIndex() {
        return cursor;
    }
    
    // 上一项索引
    public int previousIndex() {
        return cursor-1;
    }

    // 不支持remove操作
    public void remove() {
        throw new UnsupportedOperationException();
    }

    // 不支持set操作
    public void set(E e) {
        throw new UnsupportedOperationException();
    }

    // 不支持add操作
    public void add(E e) {
        throw new UnsupportedOperationException();
    }

    @Override
    public void forEachRemaining(Consumer<? super E> action) {
        Objects.requireNonNull(action);
        Object[] elements = snapshot;
        final int size = elements.length;
        for (int i = cursor; i < size; i++) {
            @SuppressWarnings("unchecked") E e = (E) elements[i];
            action.accept(e);
        }
        cursor = size;
    }
}
```

# 类的属性

属性中有一个可重入锁，用来保证线程安全访问，还有一个Object类型的数组，用来存放具体的元素。当然，也使用到了反射机制和CAS来保证原子性的修改lock域。

```java
public class CopyOnWriteArrayList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable {
    // 版本序列号
    private static final long serialVersionUID = 8673264195747942595L;
    // 可重入锁
    final transient ReentrantLock lock = new ReentrantLock();
    // 对象数组，用于存放元素
    private transient volatile Object[] array;
    // 反射机制
    private static final sun.misc.Unsafe UNSAFE;
    // lock域的内存偏移量
    private static final long lockOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = CopyOnWriteArrayList.class;
            lockOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("lock"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```
# 类的构造函数


```java
public CopyOnWriteArrayList() {
    // 设置数组
    setArray(new Object[0]);
}
```

```java
// 用于创建一个按 collection 的迭代器返回元素的顺序包含指定 collection 元素的列表
public CopyOnWriteArrayList(Collection<? extends E> c) {
    Object[] elements;
    if (c.getClass() == CopyOnWriteArrayList.class) // 类型相同
        // 获取c集合的数组
        elements = ((CopyOnWriteArrayList<?>)c).getArray();
    else { // 类型不相同
        // 将c集合转化为数组并赋值给elements
        elements = c.toArray();
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        // elements类型不为Object[]类型
        if (elements.getClass() != Object[].class) 
            // 将elements数组转化为Object[]类型的数组
            elements = Arrays.copyOf(elements, 
	            elements.length, Object[].class);
    }
    // 设置数组
    setArray(elements);
}
```
处理流程如下
1. 判断传入的集合c的类型是否为`CopyOnWriteArrayList`类型，若是，则获取该集合类型的底层数组(`Object[]`)，并且设置当前`CopyOnWriteArrayList`的数组，进入步骤***3***；否则，进入步骤***2***
2. 将传入的集合转化为数组elements，判断elements的类型是否为`Object[]`类型(toArray方法可能不会返回`Object`类型的数组)，若不是，则将elements转化为`Object`类型的数组。进入步骤***3***
3. 设置当前`CopyOnWriteArrayList`的`Object[]`为elements。
    
```java
// 该构造函数用于创建一个保存给定数组的副本的列表。
public CopyOnWriteArrayList(E[] toCopyIn) {
    // 将toCopyIn转化为Object[]类型数组，然后设置当前数组
    setArray(Arrays.copyOf(toCopyIn, toCopyIn.length, Object[].class));
}
```


# 核心函数分析


## Arrays.copyOf函数

该函数用于复制指定的数组，截取或用 null 填充(如有必要)，以使副本具有指定的长度。

```java
public static <T,U> T[] copyOf(U[] original, int newLength, 
							   Class<? extends T[]> newType) {
    @SuppressWarnings("unchecked")
    // 确定copy的类型
    // 将newType转化为Object类型，将Object[].class转化为Object类型，
    // 判断两者是否相等，若相等，则生成指定长度的Object数组
    // 否则,生成指定长度的新类型的数组)
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    // 将original数组从下标0开始，
    // 复制长度为(original.length和newLength的较小者)
    // 复制到copy数组中(也从下标0开始)
    System.arraycopy(original, 0, copy, 0,
                        Math.min(original.length, newLength));
    return copy;
}
```

## add函数
```java
public boolean add(E e) {
    // 可重入锁
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 元素数组
        Object[] elements = getArray();
        // 数组长度
        int len = elements.length;
        // 复制数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 存放元素e
        newElements[len] = e;
        // 设置数组
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```
此函数用于将指定元素添加到此列表的尾部，处理流程如下
1. 获取锁(保证多线程的安全访问)，获取当前的`Object`数组，获取`Object`数组的长度为`length`，进入步骤***2***。
2. 根据`Object`数组复制一个长度为`length+1`的`Object`数组为`newElements`
3. 将下标为length的数组元素`newElements[length]`设置为元素e，再设置当前`Object[]`为newElements，释放锁，返回。这样就完成了元素的添加。

## addIfAbsent方法
该函数用于添加元素(如果数组中不存在，则添加；否则，不添加，直接返回)，可以保证多线程环境下不会重复添加元素。

```java
private boolean addIfAbsent(E e, Object[] snapshot) {
    // 重入锁
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 获取数组
        Object[] current = getArray();
        // 数组长度
        int len = current.length;
        if (snapshot != current) { // 快照不等于当前数组，对数组进行了修改
            // Optimize for lost race to another addXXX operation
            // 取较小者
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++) // 遍历
	            // 当前数组的元素与快照的元素不相等并且e与当前元素相等
                if (current[i] != snapshot[i] && eq(e, current[i])) 
                    // 表示在snapshot与current之间修改了数组，
                    // 并且设置了数组某一元素为e，已经存在
                    // 返回
                    return false;
            // 在当前数组中找到e元素
            if (indexOf(e, current, common, len) >= 0) 
				// 返回
				return false;
        }
        // 复制数组
        Object[] newElements = Arrays.copyOf(current, len + 1);
        // 对数组len索引的元素赋值为e
        newElements[len] = e;
        // 设置数组
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```
该函数的流程如下:

1. 获取锁，获取当前数组为current，current长度为len，判断数组之前的快照snapshot是否等于当前数组current，若不相等，则进入步骤***2***；否则，进入步骤***4***
2. 不相等，表示在`snapshot`与`current`之间，对数组进行了修改，获取长度(`snapshot`与`current`之间的较小者)，对`current`进行遍历操作，若遍历过程发现`snapshot`与`current`的元素不相等并且`current`的元素与指定元素相等(可能进行了set操作)，进入步骤***5***，否则，进入步骤***3***
3. 在当前数组中索引指定元素，若能够找到，进入步骤***5***，否则，进入步骤***4***
4. 复制当前数组current为newElements，长度为len+1，此时`newElements[len]`为null。再设置`newElements[len]`为指定元素e，再设置数组，进入步骤***5***
5. 释放锁，返回。

## set函数

此函数用于用指定的元素替代此列表指定位置上的元素，也是基于数组的复制来实现的。

```java
public E set(int index, E element) {
    // 可重入锁
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 获取数组
        Object[] elements = getArray();
        // 获取index索引的元素
        E oldValue = get(elements, index);

        if (oldValue != element) { // 旧值等于element
            // 数组长度
            int len = elements.length;
            // 复制数组
            Object[] newElements = Arrays.copyOf(elements, len);
            // 重新赋值index索引的值
            newElements[index] = element;
            // 设置数组
            setArray(newElements);
        } else {
            // Not quite a no-op; ensures volatile write semantics
            // 设置数组
            setArray(elements);
        }
        // 返回旧值
        return oldValue;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

## remove函数

此函数用于移除此列表指定位置上的元素。

```java
public E remove(int index) {
    // 可重入锁
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 获取数组
        Object[] elements = getArray();
        // 数组长度
        int len = elements.length;
        // 获取旧值
        E oldValue = get(elements, index);
        // 需要移动的元素个数
        int numMoved = len - index - 1;
        if (numMoved == 0) // 移动个数为0
            // 复制后设置数组
            setArray(Arrays.copyOf(elements, len - 1));
        else { // 移动个数不为0
            // 新生数组
            Object[] newElements = new Object[len - 1];
            // 复制index索引之前的元素
            System.arraycopy(elements, 0, newElements, 0, index);
            // 复制index索引之后的元素
            System.arraycopy(elements, index + 1, newElements, 
			            index, numMoved);
            // 设置索引
            setArray(newElements);
        }
        // 返回旧值
        return oldValue;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```
处理流程如下

1. 获取锁，获取数组elements，数组长度为length，获取索引的值`elements[index]`，计算需要移动的元素个数`(length - index - 1)`，若个数为0，则表示移除的是数组的最后一个元素，复制elements数组，复制长度为length-1，然后设置数组，进入步骤***3***；否则，进入步骤***2***
2. 先复制index索引前的元素，再复制index索引后的元素，然后设置数组。
3. 释放锁，返回旧值。

# 缺陷和使用场景
## 缺点：
1. 由于写操作的时候，需要拷贝数组，会消耗内存，如果原数组的内容比较多的情况下，可能导致young gc或者full gc
2. 不能用于实时读的场景，像拷贝数组、新增元素都需要时间，所以调用一个set操作后，读取到数据可能还是旧的。
    
## 使用场景
CopyOnWriteArrayList 合适读多写少的场景，不过这类慎用