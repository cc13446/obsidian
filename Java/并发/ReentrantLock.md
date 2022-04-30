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

#### 类的内部类：HoldCounter
```java
// 计数器
static final class HoldCounter {
    // 计数
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    // 获取当前线程的TID属性的值
    final long tid = getThreadId(Thread.currentThread());
}
```
