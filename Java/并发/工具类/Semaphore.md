# 简介
`Semaphore`也叫信号量，在JDK1.5被引入，可以用来控制同时访问特定资源的线程数量，通过协调各个线程，以保证合理的使用资源。它内部维护了一组虚拟的许可，许可的数量可以通过构造函数的参数指定。
1. 访问特定资源前，必须使用acquire方法获得许可，如果许可数量为0，该线程则一直阻塞，直到有可用许可。
2. 访问资源后，使用release释放许可。

`Semaphore`和`ReentrantLock`类似，获取许可有公平策略和非公平许可策略，默认情况下使用非公平策略。

# 应用场景
`Semaphore`可以用来做流量分流，特别是对公共资源有限的场景，比如数据库连接。  
假设有一个需求，读取几万个文件的数据到数据库中，由于文件读取是IO密集型任务，可以启动几十个线程并发读取，但是数据库连接数只有10个，这时就必须控制最多只有10个线程能够拿到数据库连接进行操作。这个时候，就可以使用`Semaphore`做流量控制。

# Semaphore源码分析
总结：通过AQS的共享锁实现，初始化许可为count，存在AQS的state中；`acquire`函数将检查当前资源是否够用，不够用的话就放到阻塞队列里面，够用的话就减少相应数量的资源(`count -= n`)；`release`函数增加资源(`count += n`)。

## 类的继承关系
```java
public class Semaphore implements java.io.Serializable {}
```

## 类的内部类
![[Pasted image 20220502181300.png]]

`Semaphore`与`ReentrantLock`的内部类的结构相同，类内部总共存在`Sync`、`NonfairSync`、`FairSync`三个类，`NonfairSync`与`FairSync`类继承自`Sync`类，`Sync`类继承自`AbstractQueuedSynchronizer`抽象类。下面逐个进行分析。

## 类的内部类 - Sync类

```java
// 内部类，继承自AQS
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 版本号
    private static final long serialVersionUID = 1192457210091910933L;
    
    // 构造函数
    Sync(int permits) {
        // 设置状态数
        setState(permits);
    }
    
    // 获取许可
    final int getPermits() {
        return getState();
    }

    // 共享模式下非公平策略获取
    // 返回正数表示获取共享锁成功，不会阻塞，返回负数代表获取共享锁失败，阻塞
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) { // 无限循环
            // 获取许可数
            int available = getState();
            // 剩余的许可
            int remaining = available - acquires;
            if (remaining < 0 ||
	            // 许可小于0或者比较并且设置状态成功
                compareAndSetState(available, remaining)) 
                return remaining;
        }
    }
    
    // 共享模式下进行释放
    protected final boolean tryReleaseShared(int releases) {
        for (;;) { // 无限循环
            // 获取许可
            int current = getState();
            // 可用的许可
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next)) // 比较并进行设置成功
                return true;
        }
    }

    // 根据指定的缩减量减小可用许可的数目
    final void reducePermits(int reductions) {
        for (;;) { // 无限循环
            // 获取许可
            int current = getState();
            // 可用的许可
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next)) // 比较并进行设置成功
                return;
        }
    }

    // 获取并返回立即可用的所有许可
    final int drainPermits() {
        for (;;) { // 无限循环
            // 获取许可
            int current = getState();
            // 许可为0或者比较并设置成功
            if (current == 0 || compareAndSetState(current, 0)) 
                return current;
        }
    }
}
```

## 类的内部类 - NonfairSync类
NonfairSync类继承了Sync类，表示采用非公平策略获取资源，其只有一个tryAcquireShared方法，重写了AQS的该方法，其源码如下:

```java
static final class NonfairSync extends Sync {
    // 版本号
    private static final long serialVersionUID = -2694183684443567898L;
    
    // 构造函数
    NonfairSync(int permits) {
        super(permits);
    }
    // 共享模式下获取
    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

## 类的内部类 - FairSync类

FairSync类继承了Sync类，表示采用公平策略获取资源，其只有一个tryAcquireShared方法，重写了AQS的该方法，其源码如下。

```java
protected int tryAcquireShared(int acquires) {
    for (;;) { // 无限循环
        if (hasQueuedPredecessors()) // 同步队列中存在其他节点
            return -1;
        // 获取许可
        int available = getState();
        // 剩余的许可
        int remaining = available - acquires;
        // 剩余的许可小于0或者比较设置成功
        if (remaining < 0 ||
            compareAndSetState(available, remaining)) 
            return remaining;
    }
}
```
## 类的属性
```java
public class Semaphore implements java.io.Serializable {
    // 版本号
    private static final long serialVersionUID = -3222578661600680210L;
    // 属性
    private final Sync sync;
}
```
说明: Semaphore自身只有两个属性，最重要的是sync属性，基于`Semaphore`对象的操作绝大多数都转移到了对sync的操作。

## 类的构造函数
```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
## 核心函数分析 - acquire函数
此方法从信号量获取一个锁，在提供一个许可前一直将线程阻塞，或者线程被中断
```java
public void acquire() throws InterruptedException {
	// aqs 获取一个共享锁
    sync.acquireSharedInterruptibly(1);
}
```
![[Pasted image 20220502190024.png]]

## 核心函数分析 - release函数
此方法释放一个许可，将其返回给信号量
```java
public void release() {
	// aqs 释放一个共享锁
    sync.releaseShared(1);
}
```
![[Pasted image 20220502190153.png]]
