转载自`https://pdai.tech/`
# 例子
```java
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.SQLException;

public class ConnectionManager {

    private static final ThreadLocal<Connection> dbConnectionLocal 
	    = new ThreadLocal<Connection>() {
        @Override
        protected Connection initialValue() {
            try {
                return DriverManager.getConnection("", "", "");
            } catch (SQLException e) {
                e.printStackTrace();
            }
            return null;
        }
    };

    public Connection getConnection() {
        return dbConnectionLocal.get();
    }
}
```


# ThreadLocal原理
主要是用到了`Thread`对象中的一个`ThreadLocalMap`类型的变量`threadLocals`, 负责存储当前线程的关于`Connection`的对象, `dbConnectionLocal`这个变量为`Key`, 以新建的`Connection`对象为`Value`; 这样的话, 线程第一次读取的时候如果不存在就会调用`ThreadLocal`的`initialValue`方法创建一个`Connection`对象并且返回;
```java
public T get() {
    // 获得当前线程
    Thread t = Thread.currentThread();
    // 每个线程 都有一个自己的ThreadLocalMap，
    // ThreadLocalMap里就保存着所有的ThreadLocal变量
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // ThreadLocalMap的key就是当前ThreadLocal对象实例，
        // 多个ThreadLocal变量都是放在这个map中的
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 从map里取出来的值就是我们需要的这个ThreadLocal变量
            T result = (T)e.value;
            return result;
        }
    }
    // 如果map没有初始化，那么在这里初始化一下
    return setInitialValue();
}
```

1. 首先获取当前线程对象t, 然后从线程t中获取到`ThreadLocalMap`的成员属性`threadLocals`
2. 如果当前线程的`threadLocals`已经初始化并且存在以当前`ThreadLocal`对象为`Key`的值，则直接返回当前线程要获取的对象
3. 如果当前线程的`threadLocals`已经初始化，但是不存在以当前`ThreadLocal`对象为`Key`的的对象, 那么重新创建一个`Connection`对象, 并且添加到当前线程的`threadLocalsMap`中，并返回
4. 如果当前线程的`threadLocals`属性还没有被初始化, 则重新创建一个`ThreadLocalMap`对象, 并且创建一个`Connection`对象并添加到`ThreadLocalMap`对象中并返回。
```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```
1. 首先调用我们上面写的重载过后的`initialValue`方法
2. 继续查看当前线程的`threadLocals`是不是空的，如果`ThreadLocalMap`已被初始化，那么直接将产生的对象添加到`ThreadLocalMap`中, 如果没有初始化, 则创建并添加对象到其中

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}  
```

# ThreadLocalMap对象是什么
本质上来讲, 它就是一个Map
1. 它没有实现Map接口;
2. 它没有public的方法, 他的方法仅仅在`ThreadLocal`类中调用, 属于静态内部类
3. `ThreadLocalMap`的`Entry`实现继承了`WeakReference<ThreadLocal<?>>`
4. 仅仅用了一个`Entry`数组来存储`Key`, `Value`; `Entry`并不是链表形式, 而是每个`bucket`里面仅仅放一个`Entry`

```java
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
	// 从i开始往后一直遍历到数组最后一个Entry
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
         
        ThreadLocal<?> k = e.get();
        //如果key相等，覆盖value
        if (k == key) {
            e.value = value;
            return;
        }
        //如果key为null,用新key、value覆盖，同时清理历史key=null的陈旧数据
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
    // 如果清理完无用条目(ThreadLocal被回收的条目)
    // 并且数组中的数据大小 > 阈值的时候对当前的Table进行重新哈希 
    // 所以, 该HashMap是处理冲突检测的机制是向后移位, 
    // 清除过期条目 最终找到合适的位置
}
```

```java
// 先找到ThreadLocal的索引位置,
// 如果索引位置处的entry不为空并且键与threadLocal是同一个对象, 则直接返回; 
// 否则去后面的索引位置继续查找。
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```


## hreadLocal造成内存泄露的问题

网上有这样一个例子：

```java
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadLocalDemo {
    static class LocalVariable {
        private Long[] a = new Long[1024 * 1024];
    }

    // (1)
    final static ThreadPoolExecutor poolExecutor 
	    = new ThreadPoolExecutor(5, 5, 1, TimeUnit.MINUTES,
            new LinkedBlockingQueue<>());
    // (2)
    final static ThreadLocal<LocalVariable> localVariable 
	    = new ThreadLocal<LocalVariable>();

    public static void main(String[] args) throws InterruptedException {
        // (3)
        Thread.sleep(5000 * 4);
        for (int i = 0; i < 50; ++i) {
            poolExecutor.execute(new Runnable() {
                public void run() {
                    // (4)
                    localVariable.set(new LocalVariable());
                    // (5)
                    System.out.println("use local varaible" 
	                    + localVariable.get());
                    localVariable.remove();
                }
            });
        }
        // (6)
        System.out.println("pool execute over");
    }
}
```

如果用线程池来操作 `ThreadLocal `对象确实会造成内存泄露，因为对于线程池里面不会销毁的线程，里面会存在着`<ThreadLocal, LocalVariable>`的强引用，因为`final static` 修饰的 `ThreadLocal` 并不会释放, 而`ThreadLocalMap` 对于 `Key` 虽然是弱引用, 但是强引用不会释放, 弱引用当然也会一直有值, 同时创建的`LocalVariable`对象也不会释放, 就造成了内存泄露。如果`LocalVariable`对象不是一个大对象的话, 其实泄露的并不严重, `泄露的内存 = 核心线程数 * LocalVariable`对象的大小;

所以, 为了避免出现内存泄露的情况, `ThreadLocal`提供了一个清除线程中对象的方法, 即 `remove`, 其实内部实现就是调用 `ThreadLocalMap` 的`remove`方法:

```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```
找到`Key`对应的`Entry`，并且清除`Entry`的`Key`(ThreadLocal)置空, 随后清除过期的`Entry`即可避免内存泄露。