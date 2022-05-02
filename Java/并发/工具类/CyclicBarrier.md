# CyclicBarrier简介
`CyclicBarrier`也叫同步屏障，在JDK1.5被引入，可以让一组线程达到一个屏障时被阻塞，直到最后一个线程达到屏障时，所以被阻塞的线程才能继续执行。  `CyclicBarrier`好比一扇门，默认情况下关闭状态，堵住了线程执行的道路，直到所有线程都就位，门才打开，让所有线程一起通过。

总结：持有重入锁和`Condition`，并记录阻塞在条件上线程的数量，线程都阻塞在条件上，当数量达到记录时就唤醒所有的线程，然后进入下一代，也就是再次重复这一过程。

# CyclicBarrier源码分析
## 类的继承关系
CyclicBarrier没有显示继承哪个父类或者实现哪个父接口, 所有AQS和重入锁不是通过继承实现的，而是通过组合实现的。
```java
public class CyclicBarrier {}
```
## 类的内部类
`CyclicBarrier`类存在一个内部类`Generation`，每一次使用的`CyclicBarrier`可以当成`Generation`的实例，其源代码如下

```java
private static class Generation {
	// 表示当前屏障是否被损坏
    boolean broken = false;
}
```
## 类的属性
```java
public class CyclicBarrier {

    // 可重入锁
    private final ReentrantLock lock = new ReentrantLock();
    // 条件队列
    private final Condition trip = lock.newCondition();
    // 参与的线程数量
    private final int parties;
    // 由最后一个进入 barrier 的线程执行的操作
    private final Runnable barrierCommand;
    // 当前代
    private Generation generation = new Generation();
    // 正在等待进入屏障的线程数量
    private int count;
}
```
说明: 该属性有一个为`ReentrantLock`对象，有一个`Condition`对象，而`Condition`对象又是基于`AQS`的，所以，归根到底，底层还是由AQS提供支持。

## 构造函数
```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    // 参与的线程数量小于等于0，抛出异常
    if (parties <= 0) throw new IllegalArgumentException();
    // 设置parties
    this.parties = parties;
    // 设置count
    this.count = parties;
    // 设置barrierCommand
    this.barrierCommand = barrierAction;
}

public CyclicBarrier(int parties) {
    // 调用含有两个参数的构造函数
    this(parties, null);
}
```
## 核心函数 - dowait函数
```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException, TimeoutException {
    // 保存当前锁
    final ReentrantLock lock = this.lock;
    // 锁定
    lock.lock();
    try {
        // 保存当前代
        final Generation g = generation;
        
        if (g.broken) // 屏障被破坏，抛出异常
            throw new BrokenBarrierException();

        if (Thread.interrupted()) { // 线程被中断
            // 损坏当前屏障，并且唤醒所有的线程，只有拥有锁的时候才会调用
            breakBarrier();
            // 抛出异常
            throw new InterruptedException();
        }
        
        // 减少正在等待进入屏障的线程数量
        int index = --count;
        if (index == 0) {  // 正在等待进入屏障的线程数量为0，所有线程都已经进入
            // 运行的动作标识
            boolean ranAction = false;
            try {
                // 保存运行动作
                final Runnable command = barrierCommand;
                if (command != null) // 动作不为空
                    // 运行
                    command.run();
                // 设置ranAction状态
                ranAction = true;
                // 进入下一代
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction) // 没有运行的动作
                    // 损坏当前屏障
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed) // 没有设置等待时间
                    // 等待
                    trip.await(); 
                else if (nanos > 0L) // 设置了等待时间，并且等待时间大于0
                    // 等待指定时长
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) { 
	            // 等于当前代并且屏障没有被损坏
                if (g == generation && ! g.broken) { 
                    // 损坏当前屏障
                    breakBarrier();
                    // 抛出异常
                    throw ie;
                } else { // 不等于当前带后者是屏障被损坏
                    // 中断当前线程
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken) // 屏障被损坏，抛出异常
                throw new BrokenBarrierException();

            if (g != generation) // 不等于当前代
                // 返回索引
                return index;

            if (timed && nanos <= 0L) { // 设置了等待时间，并且等待时间小于0
                // 损坏屏障
                breakBarrier();
                // 抛出异常
                throw new TimeoutException();
            }
        }
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```
1. 每当线程执行await，内部变量count减1，如果`count != 0`，说明有线程还未到屏障处，则在锁条件变量trip上等待。
2. 当`count == 0`时，说明所有线程都已经到屏障处，执行条件变量的`signalAll`方法唤醒等待的线程。(`nextGeneration`)

## 核心函数 - nextGeneration函数
此函数在所有线程进入屏障后会被调用，即生成下一个版本，所有线程又可以重新进入到屏障中，其源代码如下

```java
private void nextGeneration() {
    // 唤醒所有线程
    trip.signalAll();
    // set up next generation
    // 恢复正在等待进入屏障的线程数量
    count = parties;
    // 新生一代
    generation = new Generation();
}
// condition支持
public final void signalAll() {
    if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
        throw new IllegalMonitorStateException();
    // 保存condition队列头节点
    Node first = firstWaiter;
    if (first != null) // 头节点不为空
        // 唤醒所有等待线程
        doSignalAll(first);
}

private void doSignalAll(Node first) {
    // condition队列的头节点尾结点都设置为空
    lastWaiter = firstWaiter = null;
    // 循环
    do {
        // 获取first结点的nextWaiter域结点
        Node next = first.nextWaiter;
        // 设置first结点的nextWaiter域为空
        first.nextWaiter = null;
        // 将first结点从condition队列转移到sync队列
        transferForSignal(first);
        // 重新设置first
        first = next;
    } while (first != null);
}

final boolean transferForSignal(Node node) {
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
    Node p = enq(node);
    int ws = p.waitStatus;
    // 返回的节点是Cancel状态，原来的尾节点被取消了，cas再次判断也是一样
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
	    // 这里为什么放入sync队列之后又进行一次unpark呢？
	    // 这里没有逻辑也是可以的，但是可以加快并发
        LockSupport.unpark(node.thread);
    return true;
}
// 确保结点能够成功入队列，返回前一个节点
private Node enq(final Node node) {
    for (;;) { // 无限循环，
        // 保存尾结点
        Node t = tail;
        if (t == null) { // 尾结点为空，即还没被初始化
	        // 头节点为空，并设置头节点为新生成的结点
            if (compareAndSetHead(new Node())) 
                tail = head; // 头节点与尾结点都指向同一个新生结点
        } else { // 尾结点不为空，即已经被初始化过
            // 将node结点的prev域连接到尾结点
            node.prev = t; 
            // 比较结点t是否为尾结点，若是则将尾结点设置为node
            if (compareAndSetTail(t, node)) { 
                // 设置尾结点的next域为node
                t.next = node; 
                return t; // 返回尾结点
            }
        }
    }
}
```
![[Pasted image 20220502170947.png]]

![](https://pdai-1257820000.cos.ap-beijing.myqcloud.com/pdai.tech/public/_images/thread/java-thread-x-cyclicbarrier-2.png)

## breakBarrier函数
```java
// 此函数的作用是损坏当前屏障，会唤醒所有在屏障中的线程
private void breakBarrier() {
    // 设置状态
    generation.broken = true;
    // 恢复正在等待进入屏障的线程数量
    count = parties;
    // 唤醒所有线程
    trip.signalAll();
}
```