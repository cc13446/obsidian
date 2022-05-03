转载自`https://pdai.tech/`
# CountDownLatch介绍
底层由AQS提供支持，所以其数据结构可以参考AQS的数据结构，而AQS的数据结构核心就是两个虚拟队列: 同步队列`sync queue` 和条件队列`condition queue`，不同的条件会有不同的条件队列。`CountDownLatch`典型的用法是将一个程序分为n个互相独立的可解决任务，并创建值为n的`CountDownLatch`。当每一个任务完成时，都会在这个锁存器上调用`countDown`，等待问题被解决的任务调用这个锁存器的`await`，将他们自己拦住，直至锁存器计数结束。

总结：利用了AQS的共享锁，初始设置AQS共享锁`A`的state为`count`，`await`函数获取共享锁`B`， `countDown`方法减共享锁`state`

# 源码分析
## 类的内部类
```java
private static final class Sync extends AbstractQueuedSynchronizer {
    // 版本号
    private static final long serialVersionUID = 4982264981922014374L;
    
    // 构造器
    Sync(int count) {
        setState(count);
    }
    
    // 返回当前计数
    int getCount() {
        return getState();
    }

    // 试图在共享模式下获取对象状态
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    // 试图设置状态来反映共享模式下的一个释放
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        // 无限循环
        for (;;) {
            // 获取状态
            int c = getState();
            if (c == 0) // 没有被线程占有
                return false;
            // 下一个状态
            int nextc = c-1;
            if (compareAndSetState(c, nextc)) // 比较并且设置成功
                return nextc == 0;
        }
    }
}
```
## 类的属性
```java
public class CountDownLatch {
    // 同步队列
    private final Sync sync;
}
```
## 核心函数 - await函数

此函数将会使当前线程在锁存器倒计数至零之前一直等待，除非线程被中断。其源码如下

```java
public void await() throws InterruptedException {
    // 转发到sync对象上
    sync.acquireSharedInterruptibly(1);
}
```
```java
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

// 简单的判断AQS的state是否为0，为0则返回1，不为0则返回-1
// 为0的时候之前的线程都减完了，也就不会阻塞
// 返回1表示获取共享锁成功，不会阻塞，返回-1代表获取共享锁失败，阻塞
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

/**
 * Acquires in shared interruptible mode.
 * 在共享可中断模式下请求锁
 */
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 为当前线程和给定模式创建节点并插入队列尾部，addWaiter方法前文讲解过
    final Node node = addWaiter(Node.SHARED);
    // 操作是否失败
    boolean failed = true;
    try {
        for (;;) {
        	// 自旋
        	// 获取当前节点的前驱节点
            final Node p = node.predecessor();
            if (p == head) {
            	// 如果前驱节点是头节点，以共享的方式请求获取锁
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                	// 成功获取锁，设置头节点和共享模式传播
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 如果前驱节点不是头节点或者没有获取锁
                // 判断当前线程是否需要被阻塞
                // 阻塞线程并且检测线程是否被中断
                // 没抢到锁的线程需要被阻塞，避免一直去争抢锁，浪费CPU资源
                throw new InterruptedException();
        }
    } finally {
        if (failed)
        	// 自旋异常退出，取消正在进行锁争抢
            cancelAcquire(node);
    }
}
// AQS的方法
// 设置头节点并且释放头节点后面的满足条件的结点
private void setHeadAndPropagate(Node node, int propagate) {
    // 获取头节点
    Node h = head; // Record old head for check below
    // 设置头节点
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        // 获取节点的后继
        Node s = node.next;
        if (s == null || s.isShared()) // 后继为空或者为共享模式
            // 以共享模式进行释放
            doReleaseShared();
    }
}
// AQS的方法
private void doReleaseShared() {
    // 无限循环
    for (;;) {
        // 保存头节点
        Node h = head;
        if (h != null && h != tail) { // 头节点不为空并且头节点不为尾结点
            // 获取头节点的等待状态
            int ws = h.waitStatus; 
            if (ws == Node.SIGNAL) { // 状态为SIGNAL
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 释放后继结点
                unparkSuccessor(h);
            }
            else if (ws == 0 
	            &&!compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) 
                continue;                // loop on failed CAS
        }
        if (h == head) // 若头节点改变，继续循环  
            break;
    }
}
```
如下的调用链：
![[Pasted image 20220502131215.png]]


## 核心函数 - countDown函数

此函数将递减锁存器的计数，如果计数到达零，则释放所有等待的线程

```java
public void countDown() {
    sync.releaseShared(1);
}

// 如果共享锁释放到0，也就是countdown到0，释放所有的线程
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    // 无限循环
    for (;;) {
        // 获取状态
        int c = getState();
        if (c == 0) // 没有被线程占有
            return false;
        // 下一个状态
        int nextc = c-1;
        if (compareAndSetState(c, nextc)) // 比较并且设置成功
            return nextc == 0;
    }
}
```