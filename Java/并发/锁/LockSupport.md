# 概述
LockSupport用来创建锁和其他同步类的基本线程阻塞原语。简而言之，当调用`park`时，表示当前线程将会等待，直至获得许可，当调用`unpark`时，必须把等待获得许可的线程作为参数进行传递，好让此线程继续运行。

# 底层实现
## 定义
```java
public class LockSupport {
    // Hotspot implementation via intrinsics API
    private static final sun.misc.Unsafe UNSAFE;
    private static final long parkBlockerOffset;
    private static final long SEED;
    private static final long PROBE;
    private static final long SECONDARY;
    
    static {
        try {
            // 获取Unsafe实例
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            // 线程类类型
            Class<?> tk = Thread.class;
            // 获取Thread的parkBlocker字段的内存偏移地址
            parkBlockerOffset = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("parkBlocker"));
            // 获取Thread的threadLocalRandomSeed字段的内存偏移地址
            SEED = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSeed"));
            // 获取Thread的threadLocalRandomProbe字段的内存偏移地址
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
            // 获取Thread的threadLocalRandomSecondarySeed字段的内存偏移地址
            SECONDARY = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
        } catch (Exception ex) { throw new Error(ex); }
    }
    
	// 私有构造函数，无法被实例化
	private LockSupport() {}
}
```
Lock Support的实现要依赖于Unsafe类中的park和unpark函数
```java
public native void park(boolean isAbsolute, long time);
// 阻塞线程，并且该线程在下列情况发生之前都会被阻塞: 
// 1. 调用unpark函数，释放该线程的许可
// 2.该线程被中断
// 3. 设置的时间到了
// isAbsolute表示是否为绝对时间
// 当time为0时，表示无限等待，直到unpark发生

public native void unpark(Thread thread);
// unpark函数，释放线程的许可，即激活调用park后阻塞的线程。
// 这个函数不是安全的，调用这个函数时要确保线程依旧存活。
```

### park函数

park函数有两个重载版本

```java
public static void park() {
    // 获取许可，设置时间为无限长，直到可以获取许可
    UNSAFE.park(false, 0L);
}

public static void park(Object blocker) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 设置Blocker
    setBlocker(t, blocker);
    // 获取许可
    UNSAFE.park(false, 0L);
    // 重新可运行后再此设置Blocker
    setBlocker(t, null);
}

private static void setBlocker(Thread t, Object arg) {
    // 设置线程t的parkBlocker字段的值为arg
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}
```
调用了park函数后，会禁用当前线程。在以下三种情况之一发生之前，当前线程都将处于休眠状态，即下列情况发生时，当前线程会获取许可，可以继续运行。
-   其他某个线程将当前线程作为目标调用 unpark。
-   其他某个线程中断当前线程。
-   该调用不合逻辑地(即毫无理由地)返回。

### parkNanos函数

此函数表示在许可可用前禁用当前线程，并最多等待指定的等待时间。
```java
public static void parkNanos(Object blocker, long nanos) {
    if (nanos > 0) { // 时间大于0
        // 获取当前线程
        Thread t = Thread.currentThread();
        // 设置Blocker
        setBlocker(t, blocker);
        // 获取许可，并设置了时间
        UNSAFE.park(false, nanos);
        // 设置许可
        setBlocker(t, null);
    }
}
```
### parkUntil函数

此函数表示在指定的时限前禁用当前线程
```java
public static void parkUntil(Object blocker, long deadline) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 设置Blocker
    setBlocker(t, blocker);
    UNSAFE.park(true, deadline);
    // 设置Blocker为null
    setBlocker(t, null);
}
```
### unpark函数
此函数表示如果给定线程的许可尚不可用，则使其可用。如果线程在 park 上受阻塞，则它将解除其阻塞状态。否则，***保证下一次调用 park 不会受阻塞***。如果给定线程尚未启动，则无法保证此操作有任何效果。具体函数如下:
```java
public static void unpark(Thread thread) {
    if (thread != null) // 线程为不空
        UNSAFE.unpark(thread); // 释放该线程许可
}
```
