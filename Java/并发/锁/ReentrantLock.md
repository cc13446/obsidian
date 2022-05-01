# ReentrantLock源码分析
## 类的继承关系

ReentrantLock实现了Lock接口，Lock接口中定义了lock与unlock相关操作，并且还存在newCondition方法，表示生成一个条件。

```java
public class ReentrantLock implements Lock, java.io.Serializable
```
## 类的内部类
ReentrantLock总共有三个内部类，并且三个内部类是紧密相关的，下面先看三个类的关系。
![[Pasted image 20220430220158.png]]

### Sync类
```java
abstract static class Sync extends AbstractQueuedSynchronizer {
    // 序列号
    private static final long serialVersionUID = -5179523762034025860L;
    
    // 获取锁
    abstract void lock();
    
    // 非公平方式获取
    final boolean nonfairTryAcquire(int acquires) {
        // 当前线程
        final Thread current = Thread.currentThread();
        // 获取状态
        int c = getState();
        if (c == 0) { // 表示没有线程正在竞争该锁
	        // 比较并设置状态成功，状态0表示锁没有被占用
            if (compareAndSetState(0, acquires)) { 
                // 设置当前线程独占
                setExclusiveOwnerThread(current); 
                return true; // 成功
            }
        }
        // 当前线程拥有该锁
        else if (current == getExclusiveOwnerThread()) { 
            int nextc = c + acquires; // 增加重入次数
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            // 设置状态
            setState(nextc); 
            // 成功
            return true; 
        }
        // 失败
        return false;
    }
    
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        // 当前线程不为独占线程
        if (Thread.currentThread() != getExclusiveOwnerThread()) 
            throw new IllegalMonitorStateException(); // 抛出异常
        // 释放标识
        boolean free = false; 
        if (c == 0) {
            free = true;
            // 已经释放，清空独占
            setExclusiveOwnerThread(null); 
        }
        // 设置标识
        setState(c); 
        return free; 
    }
    
    // 判断资源是否被当前线程占有
    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    // 新生一个条件
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // Methods relayed from outer class
    // 返回资源的占用线程
    final Thread getOwner() {        
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }
    
    // 返回状态
    final int getHoldCount() {            
        return isHeldExclusively() ? getState() : 0;
    }

    // 资源是否被占用
    final boolean isLocked() {        
        return getState() != 0;
    }

    // 自定义反序列化逻辑
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}　　
  ```
### NonfairSync类
NonfairSync类继承了Sync类，表示采用非公平策略获取锁，其实现了Sync类中抽象的lock方法，每一次都尝试获取锁，而并不会按照公平等待的原则进行等待。
```java
// 非公平锁
static final class NonfairSync extends Sync {
    // 版本号
    private static final long serialVersionUID = 7316153563782823691L;

    // 获得锁
    final void lock() {
	    // 比较并设置状态成功，状态0表示锁没有被占用
        if (compareAndSetState(0, 1)) 
            // 把当前线程设置独占了锁
            setExclusiveOwnerThread(Thread.currentThread());
        else // 锁已经被占用，或者set失败
            // 以独占模式获取对象，忽略中断
            acquire(1); 
    }

    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```
### FairSyn类
FairSync类也继承了Sync类，表示采用公平策略获取锁
```java
// 公平锁
static final class FairSync extends Sync {
    // 版本序列化
    private static final long serialVersionUID = -3000897897090466540L;

    final void lock() {
        // 以独占模式获取对象，忽略中断
        acquire(1);
    }

    // 尝试公平获取锁, AQS会调用这个方法
    protected final boolean tryAcquire(int acquires) {
        // 获取当前线程
        final Thread current = Thread.currentThread();
        // 获取状态
        int c = getState();
        if (c == 0) { // 状态为0
	        // 不存在已经等待更久的线程并且比较并且设置状态成功
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) { 
                // 设置当前线程独占
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) { 
	        // 当前线程占据锁
            // 下一个状态
            int nextc = c + acquires;
            if (nextc < 0) // 超过了int的表示范围
                throw new Error("Maximum lock count exceeded");
            // 设置状态
            setState(nextc);
            return true;
        }
        return false;
    }
}
```
调用过程如下：
![[Pasted image 20220430221448.png]]

# ReentrantReadWriteLock
## 类的继承关系

```java
public class ReentrantReadWriteLock 
		implements ReadWriteLock, java.io.Serializable {}
```
ReentrantReadWriteLock实现了ReadWriteLock接口，ReadWriteLock接口定义了获取读锁和写锁的规范，具体需要实现类去实现；同时其还实现了Serializable接口，表示可以进行序列化，在源代码中可以看到ReentrantReadWriteLock实现了自己的序列化逻辑。

```java
public interface ReadWriteLock {    
	Lock readLock();    
	Lock writeLock();
}
```
内部类的继承、聚合关系如下图所示

![[Pasted image 20220501005001.png]]

## 内部类
### Sync类
Sync类利用AQS单个state字段，来同时表示读状态和写状态，源码如下：
```java
abstract static class Sync extends AbstractQueuedSynchronizer {

    private static final long serialVersionUID = 6317671515068378041L;
    
    /*
    * Read vs write count extraction constants and functions.
    * Lock state is logically divided into two unsigned shorts:
    * The lower one representing the exclusive (writer) lock hold count,
    * and the upper the shared (reader) hold count.
    */
 
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

	// 本地线程计数器
    private transient ThreadLocalHoldCounter readHolds;
    // 缓存的计数器，加快代码执行速度
    private transient HoldCounter cachedHoldCounter;
    // 第一个读线程
    private transient Thread firstReader = null;
    // 第一个读线程的计数
    private transient int firstReaderHoldCount;	
     
    /** Returns the number of shared holds represented in count  */
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    /** Returns the number of exclusive holds represented in count  */
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
    
};
```
根据上面源码可以看出：
- `SHARED_SHIFT`表示AQS中的state的高16位作为读状态，低16位作为写状态
- `SHARED_UNIT`二级制为2^16，读锁加1，state加SHARED_UNIT
- `MAX_COUNT`就是写或读资源的最大数量，为2^16-1
- 使用`sharedCount`方法获取读状态，使用`exclusiveCount`方法获取获取写状态

#### Sync类的内部类：HoldCounter
```java
// 内部类，用于记录当前线程重入读锁的次数
// 目的是便于线程释放读锁时进行合法性判断
// 线程在不持有读锁的情况下释放锁是不合法的，需要抛出异常
static final class HoldCounter {
    // 计数
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    // 获取当前线程的TID属性的值
    final long tid = getThreadId(Thread.currentThread());
}
```
#### Sync类的内部类：ThreadLocalHoldCounter
```java
// 该类型的变量是每个线程各自保存一份，其中保存的是HoldCounter对象
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    // 重写初始化方法，在没有进行set的情况下，获取的都是该HoldCounter值
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```
#### 获取锁
无论是公平锁还是非公平锁，它们获取锁的逻辑都是相同的，因此`Sync`类在这一层就提供了统一的实现，但是，获取写锁和获取读锁的逻辑不相同：
-  写锁是**互斥**资源，获取写锁的逻辑主要在`tryAcquire`方法
-  读锁是**共享**资源，获取读锁的逻辑主要在`tryAcquireShared`方法

#### 释放锁
无论是公平锁还是非公平锁，它们释放锁的逻辑都是相同的，因此`Sync`类在这一层就提供了统一的实现，但是，释放写锁和释放读锁的逻辑不相同：
-   写锁是**互斥**资源，释放写锁的逻辑主要在`tryRelease`方法
-   读锁是**共享**资源，释放读锁的逻辑主要在`tryReleaseShared`方法
#### 写锁实现
##### 申请写锁
```java
// 构造函数
public static class WriteLock implements Lock, java.io.Serializable {

	private static final long serialVersionUID = -4992448646407690164L;
	private final Sync sync;     
	
	// 构造方法注入Sync类对象    
	protected WriteLock(ReentrantReadWriteLock lock) {        
		sync = lock.sync;    
	}        
	// 实现了Lock接口的所有方法
	// ...
}

// 获取写锁
// WriteLock使用lock方法获取写锁，一次获取一个写锁
public void lock() {    
	sync.acquire(1);
}
```
`lock`方法内部实际调用的是AQS的`acquire`方法：

```java
public final void acquire(int arg) {
	if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))       
		selfInterrupt();
}
```

而`acquire`方法会调用子类`Sync`实现的`tryAcquire`方法：

```java
protected final boolean tryAcquire(int acquires) {
    // 获取当前线程
    Thread current = Thread.currentThread();
    // 获取状态
    int c = getState();
    // 写线程数量
    int w = exclusiveCount(c);
    if (c != 0) {
        // 如果是读锁被获取中，或写锁被获取但不是本线程获取的，则获取失败
        if (w == 0 || current != getExclusiveOwnerThread()) 
            return false;
        // 判断是否超过最高写线程数量
        if (w + exclusiveCount(acquires) > MAX_COUNT) 
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        // 设置AQS状态
        setState(c + acquires);
        return true;
    }
    // 如果根据公平性判断此时写线程需要被阻塞
    // 或在获取过程中发生竞争且竞争失败，则获取失败
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires)) // 写线程是否应该被阻塞
        return false;
    // 设置独占线程
    setExclusiveOwnerThread(current);
    return true;
}
```
分为三步：  
1. 如果读锁正在被获取中，或者写锁被获取中但不是本线程持有，则获取失败  
2. 如果获取写锁达到饱和，则抛出错误  
3. 如果上面两个都不成立，说明此线程可以请求写锁。但需要先根据公平策略来判断是否应该先阻塞。如果不用阻塞，且CAS成功，则获取成功。否则获取失败

`tryAcquire`体现的读写锁的特征：
-   互斥关系：
    -   写锁和写锁之间是**互斥**的：如果是别的线程持有写锁，那么直接返回false
    -   读锁和写锁之间是**互斥**的。当有线程持有读锁时，写锁不能获得
-   可重入性：如果当前线程持有写锁，就不用进行公平性判断，请求锁一定会获取成功
-   不允许锁升级

##### 释放写锁
`WriteLock`使用`unlock`方法释放写锁：
```java
public void unlock() { sync.release(1);}
```

`unlock`内部实际上调用的是AQS的`release`方法，源码如下：
```java
public final boolean release(int arg) {    
	if (tryRelease(arg)) {        
		Node h = head;        
		if (h != null && h.waitStatus != 0)            
			unparkSuccessor(h);        
		return true;    
	}    
	return false;
}
```

而该方法会调用子类`Sync`实现的`tryAcquire`方法，源码如下：
```java
protected final boolean tryRelease(int releases) {
    // 如果并不持有锁就释放，会抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 如果释放锁之后锁空闲，那么需要将锁持有者置为null
    int nextc = getState() - releases;
    boolean free = exclusiveCount(nextc) == 0; // 是否释放成功
    if (free)
        setExclusiveOwnerThread(null); // 设置独占线程为空
    setState(nextc); // 设置状态
    return free;
}
```
 
> 任何锁的释放都需要判断是否是在持有锁的情况下。如果不持有锁就释放，会抛出异常。对于写锁来说，判断是否持有锁很简单，只需要调用`isHeldExclusively`方法进行判断即可；而对于读锁来说，判断是否持有锁比较复杂，需要根据每个线程各自保存的持有读锁数来判断，即`readHolds`中保存的变量

##### 尝试获取写锁
`WriteLock`使用`tryLock`来尝试获取写锁：
```java
public boolean tryLock( ) { return sync.tryWriteLock(); }
```

`tryLock`内部实际调用的是`Sync`类定义并实现的`tryWriteLock`方法。该方法是一个`final`方法，不允许子类重写。其源码如下：
```java
final boolean tryWriteLock() {    
	Thread current = Thread.currentThread();    
	int c = getState();    
	if (c != 0) {        
		int w = exclusiveCount(c);        
		if (w == 0 || current != getExclusiveOwnerThread())            
			return false;        
		if (w == MAX_COUNT)            
			throw new Error("Maximum lock count exceeded");    
	}
	// 相比于tryAcquire方法，这里缺少对公平性判断（writerShouldBlock）        
	if (!compareAndSetState(c, c + 1))		
		return false;    
	setExclusiveOwnerThread(current);    
	return true;
}
```
其实除了缺少对公平策略判断`writerShouldBlock`的调用以外，和`tryAcquire`方法基本上是一样的
##### Lock接口其他方法的实现
```java
// 支持中断响应的lock方法，实际上调用的是AQS的acquireInterruptibly方法
public void lockInterruptibly() throws InterruptedException {    
	sync.acquireInterruptibly(1);
} 
// 实际上调用的是AQS的方法tryAcquireNanos方法
public boolean tryLock(long timeout, TimeUnit unit) 
		throws InterruptedException {    
	return sync.tryAcquireNanos(1, unit.toNanos(timeout));
} 
// 实际上调用的是Sync类实现的newCondition方法
public Condition newCondition() {    
	return sync.newCondition();
}
```

写锁支持创建条件变量，因为写锁是独占锁，而条件变量在`await`时会释放掉所有锁资源。写锁能够保证所有的锁资源都是本线程所持有，所以可以放心地去释放所有的锁

而读锁不支持创建条件变量，因为读锁是共享锁，可能会有其他线程持有读锁。如果调用`await`，不仅会释放掉本线程持有的读锁，也会释放掉其他线程持有的读锁，这是不被允许的。因此读锁不支持条件变量

#### 读锁实现
读锁是由内部类`ReadLock`实现的，其实现了`Lock`接口，获取锁、释放锁的逻辑都委托给了`Sync`类实例`sync`来执行。`ReadLock`的基本结构如下：

```java
public static class ReadLock implements Lock, java.io.Serializable { 
	private static final long serialVersionUID = -5992448646407690164L;
	private final Sync sync;     // 构造方法注入Sync类对象    
	protected ReadLock(ReentrantReadWriteLock lock) {        
		sync = lock.sync;    
	}
	// 实现了Lock接口的所有方法
	// ...
}
```
##### 获取读锁

`ReadLock`使用`lock`方法获取读锁，一次获取一个读锁：
```java
public void lock() { sync.acquireShared(1);}
```

`lock`方法内部实际调用的是AQS的`acquireShared`方法：
```java
public final void acquireShared(int arg) {    
	if (tryAcquireShared(arg) < 0)        
		doAcquireShared(arg);
}
```

该方法会调用`Sync`类实现的`tryAcquireShared`方法，源码如下：

```java
// 共享模式下获取资源
protected final int tryAcquireShared(int unused) {

    // 获取当前线程
    Thread current = Thread.currentThread();
    // 获取状态
    int c = getState();
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current) 
        // 写线程数不为0并且占有资源的不是当前线程
        // 如果写锁被获取，且并不是由本线程持有写锁，那么获取失败
        return -1;
    // 读锁数量
    int r = sharedCount(c);
    // 先进行公平性判断是否应该让步，这可能会导致重入读锁的线程获取失败
    // CAS失败可能也会导致本能获取成功的线程获取失败
    if (!readerShouldBlock() 
	    && r < MAX_COUNT 
	    && compareAndSetState(c, c + SHARED_UNIT)) { 
	    // 如果此时读锁没有被获取，则该线程是第一个获取读锁的线程，记录相应信息
        if (r == 0) {
            // 设置第一个读线程
            firstReader = current;
            // 读线程占用的资源数为1
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { // 当前线程为第一个读线程
            // 占用资源数加1
            firstReaderHoldCount++;
        } else { 
	        // 该线程不是首个获取读锁的线程，需要记录到readHolds中
            // 获取计数器
            // 通常当前获取读锁的线程就是最近获取到读锁的线程，所以直接用缓存
            HoldCounter rh = cachedHoldCounter;
            // 还是需要判断一下是不是最近获取到读锁的线程。
            // 如果不是，则调用get创建一个新的局部HoldCounter变量
            if (rh == null || rh.tid != getThreadId(current)) 
                // 获取当前线程对应的计数器
                cachedHoldCounter = rh = readHolds.get();
            // 之前最近获取读锁的线程如果释放完了读锁
            // 而导致其局部HoldCounter变量被remove了，这里重新获取就重新set
            else if (rh.count == 0) 
                // 设置
                readHolds.set(rh);
            rh.count++;
        }
        // 如果公平性判断无需让步，且读锁数未饱和，且CAS竞争成功，则说明获取成功
        return 1;
    }
    return fullTryAcquireShared(current);
}
```
 `tryAcquireShared`的返回值说明：
-   负数：获取失败，线程会进入同步队列阻塞等待
-   0：获取成功，但是后续以共享模式获取的线程都不可能获取成功（这里暂时用不上）
-   正数：获取成功，且后续以共享模式获取的线程也可能获取成功

在读写锁中，`tryAcquireShared`没有返回0的情况，只会返回正数或负数

前面“`Sync`类中的变量：
-   `firstReader`、`firstReaderHoldCount`记录第一个获取到锁的线程及其持有读锁的数量
-   `cachedHoldCounter`记录最近获取到锁的线程持有读锁的数量
-   `readHolds`是一个线程局部变量，用于保存每个获得读锁的线程各自持有的读锁数量

`tryAcquireShared`的流程如下：  
1. 如果其他线程持有写锁，那么获取失败（返回-1）  
2. 否则，根据公平策略判断是否应该阻塞。如果不用阻塞且读锁数量未饱和，则CAS请求读锁。如果CAS成功，获取成功（返回1），并记录相关信息
3. 如果根据公平策略判断应该阻塞，或者读锁数量饱和，或者CAS竞争失败，那么交给完整版本的获取方法`fullTryAcquireShared`去处理

其中上述***步骤2***如果发生了重入读，**但根据公平策略判断该线程需要阻塞等待，而导致重入读失败**。按照正常逻辑，重入读不应该失败。不过，`tryAcquireShared`并没有处理这种情况，而是将其放到了`fullTryAcquireShared`中进行处理。此外，CAS竞争失败而导致获取读锁失败，也交给`fullTryAcquireShared`去处理。

`fullTryAcquireShared`方法是尝试获取读锁的完全版本，用于处理`tryAcquireShared`方法未处理的：  
1. CAS竞争失败  
2. 因公平策略判断应该阻塞而导致的重入读失败
3. 超过最大值

这三种情况。其源码如下：

```java
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) { 
        int c = getState();
        // 有独占线程持有锁
        if (exclusiveCount(c) != 0) { 
            if (getExclusiveOwnerThread() != current) // 不为当前线程
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock.
        } else if (readerShouldBlock()) { 
            // Make sure we're not acquiring read lock reentrantly
            // 如果当前线程就是firstReader，那么它一定是重入读，
            // 不让它失败，而是重新loop直到公平性判断不阻塞为止
            if (firstReader == current) { 
                // assert firstReaderHoldCount > 0;
            } else { 
	            // 当前线程不为第一个读线程
                if (rh == null) { 
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) { 
                        rh = readHolds.get();
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                // 当前线程既不持有读锁（不是重入读），
                // 并且被公平性判断为应该阻塞，那么就获取失败
                if (rh.count == 0)
                    return -1;
            }
        }
        // 下面的逻辑基本上和tryAcquire中差不多
        // 不过这里的CAS如果失败，会重新loop直到成功为止
        if (sharedCount(c) == MAX_COUNT) // 读锁数量为最大值，抛出异常
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) { // 比较并且设置成功
            if (sharedCount(c) == 0) { // 读线程数量为0
                // 设置第一个读线程
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```
`tryAcquireShared`和`fullTryAcquireShared`体现的读写锁特征：
-   互斥关系：
    -   读锁和读锁之间是共享的：即使有其他线程持有了读锁，当前线程也能获取读锁
    -   读锁和写锁之间是互斥的。当有其他线程持有写锁，读锁不能获得：
-   可重入性：如果当前线程获取了读锁，那么它再次申请读锁一定能成功。
-   支持锁降级：如果当前线程持有写锁，那么它申请读锁一定会成功。

##### 释放读锁

`ReadLock`使用`unlock`方法释放读锁：

```java
public void unlock() { sync.releaseShared(1);}
```

`unlock`方法实际调用的是AQS的`releaseShared`方法：

```java
public final boolean releaseShared(int arg) {    
	if (tryReleaseShared(arg)) {        
		doReleaseShared();        
		return true;    
	}    
	return false;
}
```

而该方法会调用`Sync`类实现的`tryReleaseShared`方法，源码如下：

```java
protected final boolean tryReleaseShared(int unused) {    
	Thread current = Thread.currentThread();    
	if (firstReader == current) {        
		if (firstReaderHoldCount == 1)            
			firstReader = null;        
		else            
			firstReaderHoldCount--;    
	} else {        
		HoldCounter rh = cachedHoldCounter;		
		// 一般释放锁的都是最后获取锁的那个线程        
		if (rh == null || rh.tid != getThreadId(current))            
			rh = readHolds.get(); 
		       
		int count = rh.count;        
		if (count <= 1) {            
			readHolds.remove();		
			// 如果释放读锁后不再持有锁，
			// 那么移除readHolds保存的线程局部HoldCounter变量 
			if (count <= 0)                
				throw unmatchedUnlockException();	
		}        
		--rh.count;    
	}    
	// 循环CAS保证修改state成功    
	for (;;) {        
		int c = getState();        
		int nextc = c - SHARED_UNIT;        
		if (compareAndSetState(c, nextc))   
			// 如果释放后锁空闲，那么返回true，否则返回false             
			return nextc == 0;		
	}
}
```

如果返回true，说明锁是空闲的，`releaseShared`方法会进一步调用`doReleaseShared`方法，`doReleaseShared`方法会唤醒后继线程并确保传播，**保证被唤醒的线程可以执行唤醒其后续线程的逻辑**

##### 尝试申请读锁

`ReadLock`使用`tryLock`方法尝试申请读锁，源码如下：

```java
public boolean tryLock() {    return sync.tryReadLock();}
```

`tryLock`内部实际调用的是`Sync`类定义并实现的`tryReadLock`方法。该方法是一个`final`方法，不允许子类重写。其源码如下：

```java
final boolean tryReadLock() {    
	Thread current = Thread.currentThread();    
	for (;;) {        
		int c = getState();        
		if (exclusiveCount(c) != 0 
			&& getExclusiveOwnerThread() != current)           
			return false;        
		int r = sharedCount(c);        
		if (r == MAX_COUNT)            
			throw new Error("Maximum lock count exceeded");        
		if (compareAndSetState(c, c + SHARED_UNIT)) {            
			if (r == 0) {                
				firstReader = current;                
				firstReaderHoldCount = 1;            
			} else if (firstReader == current) {                
				firstReaderHoldCount++;            
			} else {                
				HoldCounter rh = cachedHoldCounter;                
				if (rh == null || rh.tid != getThreadId(current))   
					cachedHoldCounter = rh = readHolds.get();                
				else if (rh.count == 0)                    
					readHolds.set(rh);                
				rh.count++;            
			}            
			return true;        
		}    
	}
}
```
除了缺少对公平策略判断方法`readerShouldBlock`的调用以外，和`tryAcquireShared`方法基本上是一样的

## Lock接口其他方法的实现

```java
// 支持中断响应的lock方法，实际上调用的是AQS的acquireSharedInterruptibly方法
public void lockInterruptibly() throws InterruptedException {    
	sync.acquireSharedInterruptibly(1);
} 
// 实际上调用的是AQS的方法tryAcquireSharedNanos方法
public boolean tryLock(long timeout, TimeUnit unit)    
	throws InterruptedException {    
	return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
} 
// 读锁不支持创建条件变量
public Condition newCondition() {    
	throw new UnsupportedOperationException();
}
```
和写锁的区别在于，读锁不支持创建条件变量。如果调用`newCondition`方法，会直接抛出`UnsupportedOperationException`异常。


## 读写锁的公平策略

`ReentrantReadWriteLock`默认构造方法如下：

```java
public ReentrantReadWriteLock() {
    this(false);
}

public ReentrantReadWriteLock(boolean fair) {
    // 公平策略或者是非公平策略
    sync = fair ? new FairSync() : new NonfairSync();
    // 读锁
    readerLock = new ReadLock(this);
    // 写锁
    writerLock = new WriteLock(this);
}
```

说明其**默认**创建的是非公平读写锁。如果要创建公平读写锁，需要使用**有参**构造函数，**参数fair设置为true**

## 公平读写锁
公平读写锁依赖于`Sync`的子类`FairSync`来实现，其源码如下：
```java
static final class FairSync extends Sync {    
	private static final long serialVersionUID = -2274990926593161451L;  
	final boolean writerShouldBlock() {        
		return hasQueuedPredecessors();    
	}
	final boolean readerShouldBlock() {        
		return hasQueuedPredecessors();    
	}
}
```

### writerShouldBlock
`writerShouldBlock`实际上调用的是AQS的`hasQueuedPredecessors`方法，该方法会检查是否有线程在同步队列中等待，源码如下：

```java
public final boolean hasQueuedPredecessors() {    
	Node t = tail;    
	Node h = head;    
	Node s;    
	// 如果head等于tail，说明是空队列    
	// 如果队首的thread域不是当前线程，说明有别的线程先于当前线程等待获取锁  
	return h != t 
		&& ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

`writerShouldBlock`只有在`tryAcquire`中被调用。如果当前线程请求写锁时发现已经有线程（读线程or写线程）在同步队列中等待，则让步

### readerShouldBlock
`readerShouldBlock`和`writerShouldBlock`一样，都是调用AQS的`hasQueuedPredecessors`方法，`readerShouldBlock`只有在`tryAcquireShared`中被调用。如果当前线程请求读锁时发现已经有线程（读线程or写线程）在同步队列中等待，则让步

## 非公平读写锁
非公平读写锁依赖于`Sync`的子类`NonfairSync`来实现，其源码如下：

```java
static final class NonfairSync extends Sync {    
	private static final long serialVersionUID = -8159625535654395037L;    
	final boolean writerShouldBlock() {        
		return false; // writers can always barge    
	}    
	final boolean readerShouldBlock() {        
		return apparentlyFirstQueuedIsExclusive();    
	}
}
```

### writerShouldBlock
`writerShouldBlock`直接返回false，只有在`tryAcquire`中被调用，返回false表示在非公平模式下，不管是否有线程在同步队列中等待，请求写锁都不会让步，而是直接上去竞争

### readerShouldBlock
`readerShouldBlock`实际调用的是AQS的`apparentlyFirstQueuedIsExclusive`方法。其源码如下：
```java
final boolean apparentlyFirstQueuedIsExclusive() {    
	Node h, s;    
	return (h = head) != null 
		&& (s = h.next) != null 
		&& !s.isShared()        
		&& s.thread != null;
}
```
`readerShouldBlock`只有在`tryAcquireShared`中被调用。如果当前线程请求读锁时发现同步队列队首线程是写线程，则让步。这么做的目的是为了防止写线程被饿死。因为如果一直有读线程前来请求锁，且读锁是有求必应，就会使得在同步队列中的写线程一直不能被唤醒。不过，`apparentlyFirstQueuedIsExclusive`只是一种启发式算法，并不能保证写线程一定不会被饿死。因为写线程有可能不在同步队列队首，而是排在其他读线程后面。

### 读写锁的公平策略总结
1. 公平模式：  无论当前线程请求写锁还是读锁，只要发现此时还有别的线程在同步队列中等待，都一律选择让步

2. 非公平模式：
	-   请求写锁时，当前线程会选择直接竞争，不会做丝毫的让步
	-   请求读锁时，如果发现同步队列队首线程在等待获取写锁，则会让步。不过这是一种启发式算法，因为写线程可能排在其他读线程后面

> 参考：[cnblogs](https://www.cnblogs.com/frankiedyz/p/15655865.html)  


