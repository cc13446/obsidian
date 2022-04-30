# 概述
`AbstractQueuedSynchronizer`是一个用来构建锁和同步器的框架，使用AQS能简单且高效地构造出应用广泛的大量的同步器，比如我们提到的`ReentrantLock`，`Semaphore`，`其他的诸如ReentrantReadWriteLock`，`SynchronousQueue`，`FutureTask`等等皆是基于AQS的。

# 核心思想
如果被请求的共享资源空闲，那么就将当前请求资源的线程设置为有效的工作线程，将共享资源设置为锁定状态；如果共享资源被占用，就需要一定的阻塞等待唤醒机制来保证锁分配。这个机制主要用的是CLH队列的变体实现的，将暂时获取不到锁的线程加入到队列中。

> CLH(Craig, Landin, and Hagersten)队列是一个虚拟的双向队列(即不存在队列实例，仅存在结点之间的关联关系)。AQS将每条请求共享资源的线程封装成一个CLH锁队列的一个结点来实现锁的分配。

# 数据结构
## 父类
```java
public abstract class AbstractOwnableSynchronizer implements java.io.Serializable {
    
    // 版本序列号
    private static final long serialVersionUID = 3737899427754241961L;
    // 构造方法
    protected AbstractOwnableSynchronizer() { }
    // 独占模式下的线程
    private transient Thread exclusiveOwnerThread;
    
    // 设置独占线程 
    protected final void setExclusiveOwnerThread(Thread thread) {
        exclusiveOwnerThread = thread;
    }
    
    // 获取独占线程 
    protected final Thread getExclusiveOwnerThread() {
        return exclusiveOwnerThread;
    }
}
```
## 主要字段
```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer
    implements java.io.Serializable {    
    // 版本号
    private static final long serialVersionUID = 7373984972572414691L;    
    // 头节点
    private transient volatile Node head;    
    // 尾结点
    private transient volatile Node tail;    
    // 状态
    private volatile int state;    
    // 自旋时间
    static final long spinForTimeoutThreshold = 1000L;
    
    // Unsafe类实例
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // state内存偏移地址
    private static final long stateOffset;
    // head内存偏移地址
    private static final long headOffset;
    // state内存偏移地址
    private static final long tailOffset;
    // tail内存偏移地址
    private static final long waitStatusOffset;
    // next内存偏移地址
    private static final long nextOffset;
    // 静态初始化块
    static {
        try {
            stateOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
            headOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
            tailOffset = unsafe.objectFieldOffset
                (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
            waitStatusOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("waitStatus"));
            nextOffset = unsafe.objectFieldOffset
                (Node.class.getDeclaredField("next"));
        } catch (Exception ex) { throw new Error(ex); }
    }
}
```
## 同步状态
AQS使用一个int成员变量来表示同步状态，通过内置的FIFO队列来完成获取资源线程的排队工作。AQS使用CAS对该同步状态进行原子操作实现对其值的修改。

``` java
private volatile int state;//共享变量，使用volatile修饰保证线程可见性
```

状态信息通过procted类型的getState，setState，compareAndSetState进行操作

``` java
//返回同步状态的当前值
protected final int getState() {  
    return state;
}
 // 设置同步状态的值
protected final void setState(int newState) { 
    state = newState;
}
// CAS将同步状态值设置为给定值update 如果当前同步状态的值等于expect(期望值)
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```
## Node 内部类
### Node 属性
| 属性             | 解释                 |
| ---------------- | -------------------- |
| `int waitStatus` | 等待状态             | 
| `Node prev`        | 前驱节点             |
| `Node next`        | 后继节点             |
| `Thread thread`    | 节点中的线程         |
| `Node nextWaiter`  | 等待队列中的后继节点 |
等待状态 
1. CANCELLED，值为1，表示线程已取消 
2. SIGNAL，值为-1，表示后继节点处于等待状态，锁释放或者被取消会通知后继节点去运行 
3. CONDITION，值为-2，表示下一次共享模式同步状态将会无条件地被传播下去

### Node 源码
```java
static final class Node {
    /** 共享锁标记 */
    static final Node SHARED = new Node();
    /** 独占多标记 */
    static final Node EXCLUSIVE = null;

    /** 表示线程已取消 */
    static final int CANCELLED =  1;
    /** 表示后继节点处于等待状态，锁释放或者被取消，会通知后继节点去运行 */
    static final int SIGNAL    = -1;
    /** 表示线程正在等待状态：即节点在等待队列中，节点线程等待在Condition上 */
    static final int CONDITION = -2;
    /** 表示下一次共享模式同步状态获取将会无条件地被传播下去 */
    static final int PROPAGATE = -3;
	// 节点的状态
    volatile int waitStatus;
	// 前驱节点
    volatile Node prev;
	// 后继节点
    volatile Node next;
	// 节点中的线程
    volatile Thread thread;

    Node nextWaiter;

    final boolean isShared() {
        return nextWaiter == SHARED;
    }
	// 获取前驱节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```

## ConditionObject 内部类
`ConditionObject`内部类用于维护一个或者多个`Condition queue`，它是一个单向链表，只有当使用`Condition`时，才会存在。
```java
public class ConditionObject implements Condition, java.io.Serializable {
    // 版本号
    private static final long serialVersionUID = 1173984872572414699L;

    // condition队列的头节点
    private transient Node firstWaiter;

    // condition队列的尾结点
    private transient Node lastWaiter;

    // 构造方法
    public ConditionObject() { }

    // 添加新的waiter到wait队列
    private Node addConditionWaiter() {
        // 保存尾结点
        Node t = lastWaiter;
        // 尾结点不为空，并且尾结点的状态不为CONDITION
        if (t != null && t.waitStatus != Node.CONDITION) { 
            // 清除状态为CONDITION的结点
            unlinkCancelledWaiters(); 
            // 将最后一个结点重新赋值给t
            t = lastWaiter;
        }
        // 新建一个结点
        Node node = new Node(Thread.currentThread(), Node.CONDITION);
        // 尾结点为空
        if (t == null) 
            // 设置condition队列的头节点
            firstWaiter = node;
        else 
            // 设置为节点的nextWaiter域为node结点
            t.nextWaiter = node;
        
        // 更新condition队列的尾结点
        lastWaiter = node;
        return node;
    }

	// 唤醒一个节点, first是条件队列的第一个节点
    private void doSignal(Node first) {
        // 循环
        do {
	        // 该节点的nextWaiter为空
            if ( (firstWaiter = first.nextWaiter) == null) 
                // 设置尾结点为空
                lastWaiter = null;
            // 设置first结点的nextWaiter域
            first.nextWaiter = null;
        } while (!transferForSignal(first) && (first = firstWaiter) != null); 
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

    // 从condition队列中清除状态为CANCEL的结点
    private void unlinkCancelledWaiters() {
        // 保存condition队列头节点
        Node t = firstWaiter;
        Node trail = null;
        while (t != null) { 
            // 下一个结点
            Node next = t.nextWaiter;
            // t结点的状态不为CONDTION状态
            if (t.waitStatus != Node.CONDITION) { 
                // 设置t节点的nextWaiter域为空
                t.nextWaiter = null;
                if (trail == null) 
                    // 重新设置condition队列的头节点
                    firstWaiter = next;
                else 
                    // 设置trail结点的nextWaiter域为next结点
                    trail.nextWaiter = next;
                if (next == null) 
                    // 设置condition队列的尾结点
                    lastWaiter = trail;
            }
            else 
                // 设置trail结点
                trail = t;
            // 设置t结点
            t = next;
        }
    }

    // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒。
    public final void signal() {
        if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
            throw new IllegalMonitorStateException();
        // 保存condition队列头节点
        Node first = firstWaiter;
        if (first != null) // 头节点不为空
            // 唤醒一个等待线程
            doSignal(first);
    }

    // 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程。
    public final void signalAll() {
        if (!isHeldExclusively()) // 不被当前线程独占，抛出异常
            throw new IllegalMonitorStateException();
        // 保存condition队列头节点
        Node first = firstWaiter;
        if (first != null) // 头节点不为空
            // 唤醒所有等待线程
            doSignalAll(first);
    }

    // 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
    public final void awaitUninterruptibly() {
        // 添加一个结点到等待队列
        Node node = addConditionWaiter();
        // 获取释放的状态
        int savedState = fullyRelease(node);
        boolean interrupted = false;
        while (!isOnSyncQueue(node)) { 
            // 阻塞当前线程
            LockSupport.park(this);
            if (Thread.interrupted()) // 当前线程被中断
                // 设置interrupted状态
                interrupted = true; 
        }
        if (acquireQueued(node, savedState) || interrupted) 
            selfInterrupt();
    }

    // 阻塞中被中断抛出异常还是重新中断
    /** Mode meaning to reinterrupt on exit from wait */
    private static final int REINTERRUPT =  1;
    /** Mode meaning to throw InterruptedException on exit from wait */
    private static final int THROW_IE    = -1;

    /**
	 * Checks for interrupt, returning THROW_IE if interrupted
	 * before signalled, REINTERRUPT if after signalled, or
	 * 0 if not interrupted.
	 */
    private int checkInterruptWhileWaiting(Node node) {
        return Thread.interrupted() ?
            (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
            0; 
    }

    /**
	 * Throws InterruptedException, reinterrupts current thread, or
	 * does nothing, depending on mode.
	 */
    private void reportInterruptAfterWait(int interruptMode)
        throws InterruptedException {
        if (interruptMode == THROW_IE)
            throw new InterruptedException();
        else if (interruptMode == REINTERRUPT)
            selfInterrupt();
    }

    // 等待，当前线程在接到信号或被中断之前一直处于等待状态
    public final void await() throws InterruptedException {
        if (Thread.interrupted()) // 当前线程被中断，抛出异常
            throw new InterruptedException();
        // 在wait队列上添加一个结点
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            // 阻塞当前线程
            LockSupport.park(this);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null) // clean up if cancelled
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }

    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态 
    public final long awaitNanos(long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        final long deadline = System.nanoTime() + nanosTimeout;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (nanosTimeout <= 0L) {
                transferAfterCancelledWait(node);
                break;
            }
            if (nanosTimeout >= spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
            nanosTimeout = deadline - System.nanoTime();
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return deadline - System.nanoTime();
    }


    // 等待，当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态
    public final boolean awaitUntil(Date deadline)
            throws InterruptedException {
        long abstime = deadline.getTime();
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        boolean timedout = false;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (System.currentTimeMillis() > abstime) {
                timedout = transferAfterCancelledWait(node);
                break;
            }
            LockSupport.parkUntil(this, abstime);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return !timedout;
    }

  
    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。
    // 此方法在行为上等效于: awaitNanos(unit.toNanos(time)) > 0
    public final boolean await(long time, TimeUnit unit)
            throws InterruptedException {
        long nanosTimeout = unit.toNanos(time);
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();
        int savedState = fullyRelease(node);
        final long deadline = System.nanoTime() + nanosTimeout;
        boolean timedout = false;
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {
            if (nanosTimeout <= 0L) {
                timedout = transferAfterCancelledWait(node);
                break;
            }
            if (nanosTimeout >= spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
            nanosTimeout = deadline - System.nanoTime();
        }
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
        return !timedout;
    }

    //  support for instrumentation

    /**
	 * Returns true if this condition was created by the given
	 * synchronization object.
	 *
	 * @return {@code true} if owned
	 */
    final boolean isOwnedBy(AbstractQueuedSynchronizer sync) {
        return sync == AbstractQueuedSynchronizer.this;
    }

    //  查询是否有正在等待此条件的任何线程
    protected final boolean hasWaiters() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION)
                return true;
        }
        return false;
    }


    // 返回正在等待此条件的线程数估计值
    protected final int getWaitQueueLength() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int n = 0;
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION)
                ++n;
        }
        return n;
    }

    // 返回包含那些可能正在等待此条件的线程集合
    protected final Collection<Thread> getWaitingThreads() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        ArrayList<Thread> list = new ArrayList<Thread>();
        for (Node w = firstWaiter; w != null; w = w.nextWaiter) {
            if (w.waitStatus == Node.CONDITION) {
                Thread t = w.thread;
                if (t != null)
                    list.add(t);
            }
        }
        return list;
    }
}
```
## Condition接口

ConditionObject实现了Condition接口，Condition接口定义了条件操作规范
```java
public interface Condition {

    // 等待，当前线程在接到信号或被中断之前一直处于等待状态
    void await() throws InterruptedException;
    
    // 等待，当前线程在接到信号之前一直处于等待状态，不响应中断
    void awaitUninterruptibly();
    
    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态 
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    
    // 等待，当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态
    // 此方法在行为上等效于: awaitNanos(unit.toNanos(time)) > 0
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    
    // 等待，当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态
    boolean awaitUntil(Date deadline) throws InterruptedException;
    
    // 唤醒一个等待线程。如果所有的线程都在等待此条件，则选择其中的一个唤醒
    // 在从 await 返回之前，该线程必须重新获取锁。
    void signal();
    
    // 唤醒所有等待线程。如果所有的线程都在等待此条件，则唤醒所有线程
    // 在从 await 返回之前，每个线程都必须重新获取锁。
    void signalAll();
}
```

## 同步队列
AQS中内部使用同步双向队列来完成同步状态的管理，当前线程获取同步状态失败时，同步器会将当前线程和状态构造成一个Node节点拼接到队列的尾部，并堵塞当前线程。当首节点的同步状态释放后，将唤醒后继节点，后继节点再尝试获取同步状态。

![[Pasted image 20220430201014.png]]

同步器包含了两个引用节点：
- head指向头部节点
- tail指向尾部节点

为了保证队列的线程安全，同步器使用CAS操作：`compareAndSetTail(Node expect, Node update)`设置队列的尾部节点

操作队列的源码：
```java
/**
 * cas操作，设置队列的头节点
 */
private final boolean compareAndSetHead(Node update) {
	return unsafe.compareAndSwapObject(this, headOffset, null, update);
}

/**
 * cas操作，设置队列尾节点
 */
private final boolean compareAndSetTail(Node expect, Node update) {
	return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}

/**
 * cas操作，设置队列中节点的状态
 */
private static final boolean compareAndSetWaitStatus(Node node, int expect, int update) {
	return unsafe.compareAndSwapInt(node, waitStatusOffset, expect, update);
}

/**
 * cas操作，设置节点的下一个节点
 */
private static final boolean compareAndSetNext(Node node, Node expect, Node update) {
	return unsafe.compareAndSwapObject(node, nextOffset, expect, update);
}
```

# AQS 对资源的共享方式
AQS定义两种资源共享方式
-   Exclusive(独占)：只有一个线程能执行，如ReentrantLock。
    -   公平锁：按照线程在队列中的排队顺序，先到者先拿到锁
    -   非公平锁：当线程要获取锁时，无视队列顺序直接去抢锁，谁抢到就是谁的
-   Share(共享)：多个线程可同时执行

`ReentrantReadWriteLock` 可以看成是两种共享方式的组合，因为读写锁允许多个线程同时对某一资源进行读，只允许一个线程进行写。

不同的自定义同步器争用共享资源的方式也不同。自定义同步器在实现时只需要实现共享资源 `state` 的获取与释放方式即可，至于具体线程等待队列的维护(如获取资源失败入队/唤醒出队等)，AQS已经在上层已经帮我们实现好了

# 模板方法
使用者继承`AbstractQueuedSynchronizer`并重写指定的方法。这样将AQS组合在自定义同步组件的实现中，并调用其模板方法，而这些模板方法会调用使用者重写的方法。

自定义同步器时需要重写下面几个AQS提供的模板方法：

```java
//该线程是否正在独占资源。只有用到condition才需要去实现它。
protected boolean isHeldExclusively()
//独占方式。尝试获取资源，成功则返回true，失败则返回false。
protected boolean tryAcquire(int)
//独占方式。尝试释放资源，成功则返回true，失败则返回false。
protected boolean tryRelease(int)
//共享方式。尝试获取资源。负数表示失败；大于等于0表示成功，以及资源的数量。
protected int tryAcquireShared(int)
//共享方式。尝试释放资源，成功则返回true，失败则返回false。
protected boolean tryReleaseShared(int)
```

默认情况下，每个方法都抛出 `UnsupportedOperationException`。 这些方法的实现必须是内部线程安全的，并且通常应该简短而不是阻塞。AQS类中的其他方法都是final ，所以无法被其他类使用，只有这几个方法可以被其他类使用。

以`ReentrantLock`为例，`state`初始化为0，表示未锁定状态。A线程lock()时，会调用`tryAcquire()`独占该锁并将`state+1`。此后，其他线程再`tryAcquire()`时就会失败，直到A线程`unlock()`到`state=0`(即释放锁)为止，其它线程才有机会获取该锁。当然，释放锁之前，A线程自己是可以重复获取此锁的(`state`会累加)，这就是可重入的概念。但要注意，获取多少次就要释放多么次，这样才能保证`state`是能回到零态的。


# 类的核心方法 - acquire方法
该方法以独占模式获取资源，忽略中断，即线程在aquire过程中，中断此线程是无效的。
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

static void selfInterrupt() {
	// 未获取到同步状态 && 线程中断状态是true，中断当前线程
	Thread.currentThread().interrupt();
}
```
调用方法流程如下：
1. 首先调用`tryAcquire`方法，调用此方法的线程会试图在独占模式下获取对象状态。
2. 若`tryAcquire`失败，则调用`addWaiter`方法，将此线程封装成一个结点并放入`Sync queue`。 
3. 调用`acquireQueued`方法，此方法完成的功能是`Sync queue`中的结点不断尝试获取资源，若成功，则返回true，否则，返回false。  

### addWaiter方法 
```java
// 添加等待者
private Node addWaiter(Node mode) {
    // 新生成一个结点，默认为独占模式
    Node node = new Node(Thread.currentThread(), mode);
    // 保存尾结点
    Node pred = tail;
    // 尝试快速添加尾结点
    if (pred != null) { // 尾结点不为空，即已经被初始化
        // 将node结点的prev域连接到尾结点
        node.prev = pred; 
        // 比较pred是否为尾结点，是则将尾结点设置为node 
        if (compareAndSetTail(pred, node)) { 
            // 设置尾结点的next域为node
            pred.next = node;
            return node; 
        }
    }
    // 尾结点为空(即还没有被初始化过)，或者是compareAndSetTail操作失败，则入队列
    // 队列中没有元素 或者已经被其他线程修改了
    // 如果上面添加失败，这里循环尝试添加，直到添加成功为止
    enq(node); 
    return node;
}
private Node enq(final Node node) {
    for (;;) { // 无限循环，确保结点能够成功入队列
        // 保存尾结点
        Node t = tail;
        if (t == null) { 
	        // 双端链表的头结点是一个无参构造函数的头结点
            if (compareAndSetHead(new Node())) // 初始化
                tail = head; 
        } else { 
            // 将node结点的prev域连接到尾结点
            node.prev = t; 
            // CAS设置尾结点
            if (compareAndSetTail(t, node)) { 
                // 设置尾结点的next域为node
                t.next = node; 
                return t; // 返回尾结点
            }
        }
    }
}
```
### acquireQueue方法
一个线程获取锁失败了，被放入等待队列，acquireQueued会让放入队列中的线程不断去获取锁，直到获取成功或者不再需要获取（中断）
```java
// sync队列中的结点在独占且忽略中断的模式下获取(资源)
final boolean acquireQueued(final Node node, int arg) {
    // 标记是否成功拿到资源
    boolean failed = true;
    try {
        // 中断标志
        boolean interrupted = false;
        // 开始自旋，要么获取锁，要么中断
        for (;;) {
            // 获取node节点的前驱结点
            final Node p = node.predecessor(); 
            // 如果p是头结点，说明当前节点在真实数据队列的首部
            // 就尝试获取锁（别忘了头结点是虚节点
            if (p == head && tryAcquire(arg)) { 
                setHead(node); // 设置头节点
                p.next = null; // help GC
                failed = false; // 设置标志
                return interrupted; 
            }
            // 说明p为头节点且当前没有获取到锁或者是p不为头结点
            // 这个时候就要判断当前node是否要被阻塞防止无限循环浪费资源
            if (shouldParkAfterFailedAcquire(p, node) 
	            && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
	        // 取消获取同步状态
            cancelAcquire(node);
    }
}
// setHead方法是把当前节点置为虚节点，
// 但并没有修改waitStatus，因为它是一直需要用的数据。
private void setHead(Node node) { 
	head = node; 
	node.thread = null; 
	node.prev = null; 
}
  
// 靠前驱节点判断当前线程是否应该被阻塞
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    // 获取前驱结点的状态
    int ws = pred.waitStatus;
    // 说明头结点处于唤醒状态
    if (ws == Node.SIGNAL)
        // 可以进行park操作
        // 前置节点的状态是signal，当prev释放了同步状态或者取消了，会通知当前节点
        // 所以当前节点可以安心的阻塞了（相当睡觉会有人叫醒他）
        return true; 
    // 状态为CANCELLED
    if (ws > 0) { 
	    // 找到pred结点前面最近的一个状态不为CANCELLED的结点
	    // 把取消节点从队列中剔除
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0); 
        // 赋值pred结点的next域
        pred.next = node; 
    } else { 
	    // 为PROPAGATE -3 或者是0 表示无状态,
	    // 为CONDITION -2 时，表示此节点在condition queue中
        // CAS设置前驱结点的状态为SIGNAL
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL); 
    }
    // 不能进行park操作
    return false;
}
// 进行park操作并且返回该线程是否被中断
private final boolean parkAndCheckInterrupt() {
    // 在许可可用之前禁用当前线程，并且设置了blocker
    LockSupport.park(this);
    return Thread.interrupted(); // 当前线程是否已被中断，并清除中断标记位
}
```
通过`cancelAcquire`方法，将Node的状态标记为`CANCELLED`
```java
private void cancelAcquire(Node node) {
	// 将无效节点过滤
	if (node == null)
		return;
	// 设置该节点不关联任何线程，也就是虚节点
	node.thread = null;
	Node pred = node.prev;
	// 通过前驱节点，跳过取消状态的node
	while (pred.waitStatus > 0)
		node.prev = pred = pred.prev;
	// 获取过滤后的前驱节点的后继节点
	Node predNext = pred.next;
	// 把当前node的状态设置为CANCELLED
	node.waitStatus = Node.CANCELLED;
	// 如果当前节点是尾节点，将从后往前的第一个非取消状态的节点设置为尾节点
	// 更新失败的话，则进入else，如果更新成功，将tail的后继节点设置为null
	if (node == tail && compareAndSetTail(node, pred)) {
		compareAndSetNext(pred, predNext, null);
	} else {
		int ws;
		// 如果当前节点不是head的后继节点，
		// 1:判断当前节点前驱节点的是否为SIGNAL，
		// 2:如果不是，则把前驱节点设置为SINGAL看是否成功
	    // 如果1和2中有一个为true，再判断当前节点的线程是否为null
	    // 如果上述都满足，把当前节点的前驱节点的后继指针指向当前节点的后继节点
		if (pred != head 
			&& ((ws = pred.waitStatus) == Node.SIGNAL 
			|| (ws <= 0 
			&& compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) 
			&& pred.thread != null) {
			Node next = node.next;
			if (next != null && next.waitStatus <= 0)
				compareAndSetNext(pred, predNext, next);
		} else {
	      // 如果当前节点是head的后继节点，或上述不满足，就唤醒当前节点的后继节点
			unparkSuccessor(node);
		}
		node.next = node; // help GC
	}
}
// 唤醒后继结点
private void unparkSuccessor(Node node) {
	// 获取当前结点waitStatus
	int ws = node.waitStatus;
	if (ws < 0)
		// 如果当前节点状态 < 0，将状态设置成0，当前节点已经没用了
		compareAndSetWaitStatus(node, ws, 0);
	// 获取当前节点的下一个节点
	Node s = node.next;
	// 如果下个节点是null或者下个节点被cancelled
	// 就找到队列最开始的非cancelled的节点
	if (s == null || s.waitStatus > 0) {
		s = null;
		// 就从尾部节点开始找，到队首，找到队列第一个waitStatus<0的节点。
		for (Node t = tail; t != null && t != node; t = t.prev)
			if (t.waitStatus <= 0)
				s = t;
	}
	// 如果当前节点的下个节点不为空，而且状态<=0，就把当前节点unpark
	if (s != null)
		// 唤醒后继节点
		LockSupport.unpark(s.thread);
}
```
acquireQueued方法的整个的逻辑如下: 
1. 判断结点的前驱是否为head并且是否成功获取资源。 
2. 若步骤1均满足，则设置结点为head，之后会判断是否finally模块，然后返回。 
3. 若步骤2不满足，则判断是否需要park当前线程，是否需要park当前线程的逻辑是判断结点的前驱结点的状态是否为SIGNAL，若是，则park当前结点，否则，不进行park操作。 
4. 若park了当前线程，之后某个线程对本线程unpark后，并且本线程也获得机会运行。那么，将会继续进行***1***的判断。
## 类的核心方法 - release方法 
以独占模式释放对象
```java

public final boolean release(int arg) {
    if (tryRelease(arg)) { // 释放成功
        // 保存头节点
        Node h = head; 
        if (h != null && h.waitStatus != 0) // 头节点不为空并且头节点状态不为0
            unparkSuccessor(h); //释放头节点的后继结点
        return true;
    }
    return false;
}
```
其中，tryRelease的默认实现是抛出异常，需要具体的子类实现，如果tryRelease成功，那么如果头节点不为空并且头节点的状态不为0，则释放头节点的后继结点，unparkSuccessor方法已经分析过，不再累赘。

## 共享式同步状态过程
共享式与独占式的最主要区别在于同一时刻独占式只能有一个线程获取同步状态，而共享式在同一时刻可以有多个线程获取同步状态。例如读操作可以有多个线程同时进行，而写操作同一时刻只能有一个线程进行写操作，其他操作都会被阻塞。 
```java
// 获取同步状态
public final void acquireShared(int arg) {
	// 尝试去获取共享锁，获取成功返回true，否则返回false。
	// 该方法由继承AQS的子类自己实现。采用了模板方法设计模式。
	if (tryAcquireShared(arg) < 0)
		// 获取失败，自旋获取同步状态
		doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
	// 添加共享模式节点到队列中
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
		boolean interrupted = false;
		// 自旋获取同步状态
		for (;;) {
			// 当前节点的前驱
			final Node p = node.predecessor();
			// 如果前驱节点是head节点
			if (p == head) {
				// 尝试去获取共享同步状态
				int r = tryAcquireShared(arg);
				if (r >= 0) {
					// 将当前节点设置为头结点，并且释放也是共享模式的后继节点
					setHeadAndPropagate(node, r);
					p.next = null; // help GC
					if (interrupted)
						selfInterrupt();
					failed = false;
					return;
				}
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				interrupted = true;
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}

private void setHeadAndPropagate(Node node, int propagate) {
	Node h = head; // Record old head for check below
	setHead(node);
	if (propagate > 0 || h == null || h.waitStatus < 0 
		|| (h = head) == null || h.waitStatus < 0) {
		Node s = node.next;
		if (s == null || s.isShared())
			// 真正的释放共享同步状态，并唤醒下一个节点
			doReleaseShared();
	}
}

private void doReleaseShared() {
	// 自旋释放共享同步状态
	for (;;) {
		Node h = head;
		// 如果头结点不为空 && 头结点不等于尾结点，说明存在有效的node节点
		if (h != null && h != tail) {
			int ws = h.waitStatus;
			// 如果头结点的状态为signal，说明存在需要唤醒的后继节点
			if (ws == Node.SIGNAL) {
				// 将头结点状态更新为0（初始值状态），因为此时头结点已经没用了                    
				// continue为了保证替换成功
				if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
					continue;
				// 唤醒后继节点
				unparkSuccessor(h);
			}
			// 如果状态为初始值状态0，那么设置成PROPAGATE状态
			// 确保在释放同步状态时能通知后继节点
			else if (ws == 0 &&
					 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
				continue;                // loop on failed CAS
		}
		if (h == head)                   // loop if head changed
			break;
	}
}
```