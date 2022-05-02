# 为什么要有线程池

线程池能够对线程进行统一分配，调优和监控:
-   降低资源消耗(线程无限制地创建，然后使用完毕后销毁)
-   提高响应速度(无须创建线程)
-   提高线程的可管理性

# 原理
其实java线程池的实现原理很简单，说白了就是一个线程集合`workerSet`和一个阻塞队列`workQueue`。当用户向线程池提交一个任务(也就是线程)时，线程池会先将任务放入`workQueue`中。`workerSet`中的线程会不断的从`workQueue`中获取线程然后执行。当`workQueue`中没有任务的时候，`worker`就会阻塞，直到队列中有任务了就取出来继续执行。

# 参数
```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              RejectedExecutionHandler handler)
```

-   `corePoolSize` 线程池中的核心线程数
	- 当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于corePoolSize,
	- 如果当前线程数为`corePoolSize`，继续提交的任务被保存到阻塞队列中，等待被执行；
	- 执行线程池的`prestartAllCoreThreads()`方法可以提前创建并启动所有核心线程
    
-   `workQueue` 用来保存等待被执行的任务的阻塞队列. 
    -   `ArrayBlockingQueue`
    -   `LinkedBlockingQuene`
    -   `SynchronousQuene`
    -   `PriorityBlockingQuene`: 具有优先级的无界阻塞队列；

>`LinkedBlockingQueue`比`ArrayBlockingQueue`在插入删除节点性能方面更优，但是二者在`put()`, `take()`任务的时均需要加锁，`SynchronousQueue`使用无锁算法，根据节点的状态判断执行，而不需要用到锁，其核心是`Transfer.transfer()`.

-   `maximumPoolSize` 线程池中允许的最大线程数
	- 如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，
	- 前提是当前线程数小于`maximumPoolSize`
	- 当阻塞队列是无界队列, 则`maximumPoolSize`则不起作用
	- 因为无法提交至核心线程池的线程会一直持续地放入`workQueue`

-   `keepAliveTime` 线程空闲时的存活时间，
	- 当线程没有任务执行时，该线程继续存活的时间
	- 默认情况下，该参数只在线程数大于`corePoolSize`时才有用
	- 超过这个时间的空闲线程将被终止
    
-   `unit` `keepAliveTime`的单位
    
-   `threadFactory` 创建线程的工厂，
	- 通过自定义的线程工厂可以给每个新建的线程设置一个具有识别度的线程名
	- 默认为DefaultThreadFactory
    
-   `handler` 线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4种策略:
    -   `AbortPolicy`: 直接抛出异常，默认策略；
    -   `CallerRunsPolicy`: 用调用者所在的线程来执行任务；
    -   `DiscardOldestPolicy`: 丢弃阻塞队列中靠最前的任务，并执行当前任务；
    -   `DiscardPolicy`: 直接丢弃任务；
    -  可以根据应用场景实现`RejectedExecutionHandler`接口，自定义饱和策略

# 三种类型
## newFixedThreadPool
```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>());
}
```
1. 线程池的线程数量达corePoolSize后，即使线程池没有可执行任务时，也不会释放线程。
2. 工作队列为无界队列`LinkedBlockingQueue`(队列容量为`Integer.MAX_VALUE`)
3. `FixedThreadPool`永远不会拒绝, 即饱和策略失效

## newSingleThreadExecutor
```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```
1. 只有一个线程，如果该线程异常结束，会重新创建一个新的线程继续执行任务
2. 唯一的线程可以保证所提交任务的顺序执行
3. 饱和策略失效

## newCachedThreadPool
```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>());
}
```
1. 线程池的线程数可达到`Integer.MAX_VALUE`，即2147483647
2. 内部使用`SynchronousQueue`作为阻塞队列
3. 自动释放线程资源和创建新线程执行任务

执行过程：
1. 主线程调用`SynchronousQueue`的`offer()`方法放入task, 倘若此时线程池中有空闲的线程尝试读取 `SynchronousQueue`的task, 即调用了`SynchronousQueue`的`poll()`, 那么主线程将该task交给空闲线程. 否则执行***2***
2. 当线程池为空或者没有空闲的线程, 则创建新的线程执行任务.
3. 执行完任务的线程倘若在60s内仍空闲, 则会被终止

# 源码
## 数据结构
```java
// 这个属性是用来存放 当前运行的worker数量以及线程池状态的
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
// 存放任务的阻塞队列
private final BlockingQueue<Runnable> workQueue;
// worker的集合,用set来存放
private final HashSet<Worker> workers = new HashSet<Worker>();
// 历史达到的worker数最大值
private int largestPoolSize;
// 当队列满了并且worker的数量达到maxSize的时候,执行具体的拒绝策略
private volatile RejectedExecutionHandler handler;
// 超出coreSize的worker的生存时间
private volatile long keepAliveTime;
// 常驻worker的数量
private volatile int corePoolSize;
// 最大worker的数量,一般当workQueue满了才会用到这个参数
private volatile int maximumPoolSize;
```

## 内部状态
```java
// 低29位表示线程池中线程数，通过高3位表示线程池的运行状态:
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
// 111 该状态的线程池会接收新任务，并处理阻塞队列中的任务
private static final int RUNNING    = -1 << COUNT_BITS;
// 000 该状态的线程池不会接收新任务，但会处理阻塞队列中的任务
// shutdown方法
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 001 该状态的线程不会接收新任务，也不会处理阻塞队列中的任务，会中断正在运行的任务
// shutdownNow 方法
private static final int STOP       =  1 << COUNT_BITS;
// 010 所有的任务都已经终止
private static final int TIDYING    =  2 << COUNT_BITS;
// 011 terminated()方法已经执行完成
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```
![[Pasted image 20220502103616.png]]