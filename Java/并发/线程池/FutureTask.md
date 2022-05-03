转载自`https://pdai.tech/`
# FutureTask 简介

FutureTask 为 Future 提供了基础实现，如获取任务执行结果(get)和取消任务(cancel)等。如果任务尚未完成，获取任务执行结果时将会阻塞。一旦执行结束，任务就不能被重启或取消(除非使用`runAndReset`执行计算)。FutureTask 常用来封装 `Callable` 和 `Runnable`，也可以作为一个任务提交到线程池中执行。除了作为一个独立的类之外，此类也提供了一些功能性函数供我们创建自定义 task 类使用。FutureTask 的线程安全由CAS来保证。

# FutureTask 类关系

![[Pasted image 20220501213259.png]]

FutureTask既能当做一个Runnable直接被Thread执行，也能作为Future用来得到Callable的计算结果。

# FutureTask源码解析

## Callable接口

Callable是个泛型接口，泛型V就是要call()方法返回的类型。对比Runnable接口，Runnable不会返回数据也不能抛出异常。

```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

## Future接口

Future接口代表异步计算的结果，通过Future接口提供的方法可以查看异步计算是否执行完成，或者等待执行结果并获取执行结果，同时还可以取消执行。Future接口的定义如下:

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

1. `cancel()`:用来取消异步任务的执行
	- 如果异步任务已经完成或者已经被取消，或者由于某些原因不能取消，则会返回false
	- 如果任务还没有被执行，则会返回true并且异步任务不会被执行。
	- 如果任务已经开始执行了但是还没有执行完成：
		- 若mayInterruptIfRunning为true，则会立即中断执行任务的线程并返回true
		- 若mayInterruptIfRunning为false，则会返回true且不会中断任务执行线程。
2. `isCanceled()`:判断任务是否被取消
	- 如果任务在结束(正常执行结束或者执行异常结束)前被取消则返回true
	- 否则返回false。
3. `isDone()`:判断任务是否已经完成
	- 如果完成则返回true，否则返回false。
	- 需要注意的是：任务执行过程中发生异常、任务被取消也属于任务已完成，也会返回true。
4. `get()`:获取任务执行结果
	- 如果任务还没完成则会阻塞等待直到任务执行完成。
	- 如果任务被取消则会抛出CancellationException异常
	- 如果任务执行过程发生异常则会抛出ExecutionException异常，
	- 如果阻塞等待过程中被中断则会抛出InterruptedException异常。
5. `get(long timeout,Timeunit unit)`:带超时时间的get()版本
	- 如果阻塞等待过程中超时则会抛出TimeoutException异常。

## 核心属性

```java
//内部持有的callable任务，运行完毕后置空
private Callable<V> callable;

//从get()中返回的结果或抛出的异常
private Object outcome; 

//运行callable的线程
private volatile Thread runner;

//使用Treiber栈保存等待线程
private volatile WaitNode waiters;

//任务状态
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```
7种状态具体表示：

1. `NEW`:表示是个新的任务或者还没被执行完的任务。
2. `COMPLETING`:任务已经执行完成或者执行任务的时候发生异常
	- 但是任务执行结果或者异常原因还没有保存到outcome字段
	- 这个状态会时间会比较短，属于中间状态。
3. `NORMAL`:任务已经执行完成并且任务执行结果已经保存到outcome字段
4. `EXCEPTIONAL`:任务执行发生异常并且异常原因已经保存到outcome字段
5. `CANCELLED`:任务还没开始执行或者已经开始执行但是还没有执行完成的时候，用户调用cancel(false)方法取消任务且不中断任务执行线程
6. `INTERRUPTING`: 任务还没开始执行或者已经执行但是还没有执行完成的时候，用户调用了cancel(true)方法取消任务并且要中断任务执行线程，但是还没有中断任务执行线程之前
7. `INTERRUPTED`:调用`interrupt()`中断任务执行线程之后状态会从INTERRUPTING转换到INTERRUPTED。

有一点需要注意的是，所有值大于`COMPLETING`的状态都表示任务已经执行完成(任务正常执行完成，任务执行异常或者任务被取消)。

各个状态之间的可能转换关系如下图所示:

![[Pasted image 20220501214152.png]]

## 内部类
```java
static final class WaitNode {
        //被阻塞的线程,当前线程
        volatile Thread thread;
        //此节点的下一个节点
        volatile WaitNode next;
        //构造函数，将当前线程赋值于thread属性
        WaitNode() { thread = Thread.currentThread(); }
}
```
## 构造函数
``` java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    // 用来保存底层的调用，在被执行完成以后会指向null
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
	// 把传入的Runnable封装成一个Callable对象保存在callable字段中
	// 同时如果任务执行成功的话就会返回传入的result
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}

public static <T> Callable<T> callable(Runnable task, T result) {
    if (task == null)
       throw new NullPointerException();
    // 采用的是适配器模式
    return new RunnableAdapter<T>(task, result);
}

// 简单的实现了Callable接口
// 在call()实现中调用Runnable.run()方法，然后把传入的result作为任务的结果返回。
static final class RunnableAdapter<T> implements Callable<T> {
    final Runnable task;
    final T result;
    RunnableAdapter(Runnable task, T result) {
        this.task = task;
        this.result = result;
    }
    public T call() {
        task.run();
        return result;
    }
}
```
## 核心方法 
``` java
//任务顺利执行：NEW -> COMPLETING -> NORMAL
//任务执行异常：NEW -> COMPLETING -> EXCEPTIONAL
//任务取消：NEW -> CANCELLED
//任务中断：NEW -> INTERRUPTING -> INTERRUPTED
public void run() {
	// 如果状态不是初始化状态，直接返回，避免重复执行
	// 或者使用UnSafe的cas将FutureTask实例的runner属性设置为当前线程，
	// 即将要执行该任务的线程，如果设置失败也直接返回，
	// 表明此任务有线程正在执行，避免重复执行
	if (state != NEW ||
		!UNSAFE.compareAndSwapObject(this, runnerOffset,
									 null, Thread.currentThread()))
		return;
	try {
		Callable<V> c = callable;
		//当前callable任务不为空，并且当前FutureTask状态为初始化状态
		if (c != null && state == NEW) {
			//callable执行后的返回结果
			V result;
			//任务执行是否成功
			boolean ran;
			try {
				//调用callable的call方法，获取返回结果
				result = c.call();
				//如果callable方法的call方法正常执行，将ran标志位设置为true
				ran = true;
			} catch (Throwable ex) {
				// 如果线程在执行callable的call方法出现异常，
				// 将callable的返回结果的存储对象置为空
				result = null;
				// 将标志位设置为false，表示任务执行失败
				ran = false;
				// 将异常的对象赋值给FutureTask的outcome属性
				setException(ex);
			}
			//如果任务执行成功
			if (ran)
				// 返回结果赋值给FutureTask的outcome属性
				set(result);
		}
	} finally {
		// 最后将记录执行当前任务的线程的FutureTask的属性runner置为空
		runner = null;
		// 获取当前FutureTask实例的属性state状态值 
		int s = state;
		// 如果当前状态属于中断中，或者已中断
		// 执行handlePossibleCancellationInterrupt方法
		if (s >= INTERRUPTING)
			handlePossibleCancellationInterrupt(s);
	}
}
```

```java
// 设置任务的执行结果给outcome属性
// 并且state从NEW->COMPLETING->NORMAL
// 唤醒waiter队列中所有节点对应的阻塞线程，删除其队列中的所有节点
protected void set(V v) {
	// 首先使用UnSafe的比较内存的旧值，替换成新的值，
	// 首先比较FutureTask实例的state属性内存值是否为NEW,
	// 如果是就将其替换为完成中COMPLETING的中间状态
	if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
		// 将NEW状态替换为COMPLETING状态成功
		// 将其传入进来的返回值赋值给FutureTask实例的outcome属性
		outcome = v;
		// 然后将其state的属性值重新设置为NORMAL状态，任务正常结束状态，终态
		UNSAFE.putOrderedInt(this, stateOffset, NORMAL);
		// 删除所有的等待节点，并且唤醒所有节点对应的线程
		finishCompletion();
	}
}
```

```java
// 设置任务的执行异常给outcome属性，
// 并且属性state从NEW->COMPLETING->EXCEPTIONAL
// 唤醒waiter队列中所有节点对应的阻塞线程，删除其队列中的所有节点
protected void setException(Throwable t) {
	// 首先使用UnSafe实例比较FutureTask实例的state属性的内存旧值是否是NEW值
	// 如果是将其内存重新赋值为COMPLETING状态值
	if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
		// 如果从内存中将NEW状态值替换为COMPLETING状态成功
		// 将其执行任务中出现的异常对象赋值给outcome对象 
		outcome = t;
		// 然后将其state的属性值重新设置为EXCEPTIONAL状态值，
		// 任务执行出现异常，终态
		UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL);
		// 删除所有的等待节点，并且唤醒所有节点对应的线程，
		finishCompletion();
	}
}
```

```java
// 在任务执行完成（包括任务取消、正常结束、发生异常），
// 将等待队列置空，唤醒其所有等待队列中的所有节点对应的阻塞线程，同时让任务执行体置空
private void finishCompletion() {
	//从等待队列的头结点开始
	for (WaitNode q; (q = waiters) != null;) {
		//使用UnSafe的cas将其FutureTask的waiters的等待队列置为空
		if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
			//循环唤醒等待队列节点对应的线程
			for (;;) {
				//获取节点对应的线程
				Thread t = q.thread;
				//如果线程不为空
				if (t != null) {
					//将节点对应的thread属性线程置为空
					q.thread = null;
					// 使用LockSupport唤醒线程
					// Lock的底层也是使用LockSupport进行线程的阻塞和唤醒
					LockSupport.unpark(t);
				}
				//获取当前节点的下一个节点
				WaitNode next = q.next;
				//如果下一节点为空，表明队列中没有其余节点，退出循环
				if (next == null)
					break;
				//将当前节点的下一节点置为空，加快gc
				q.next = null; 
				//将下一节点设置为当前节点 
				q = next;
			}
			//退出循环
			break;
		}
	}
	// 预留下的模板方法，
	// 如果有需要在任务执行完成，不管是任务取消、正常完成、还是出现异常，
	// 做一些处理，可以自定义一个类实现FutureTask类，重写此done方法    
	done();
	//任务执行体置空
	callable = null;
}
```

如果在 run 期间被中断，此时需要调用`handlePossibleCancellationInterrupt`方法来处理中断逻辑，确保任何中断只停留在当前`run`或`runAndReset`的任务中

```java
// 如果有线程调用FutureTask任务的cancel方法，
// 并且执行任务的线程还未调用interrupt方法，
// 此时的state的状态为INTERRUPTING状态值
private void handlePossibleCancellationInterrupt(int s) {
	//如果FutureTask的属性state状态值是INTERRUPTING值
	if (s == INTERRUPTING)
		// 如果当前任务的状态还是INTERRUPTING，让其执行任务的线程让步，
		// 让出cpu，循环等到cancel方法将其state属性更改为INTERRUPTED
		while (state == INTERRUPTING)
			//线程让步
			Thread.yield();

}
```

```java
// 相对上面介绍的run方法，此方法不会将任务正常的执行结果赋值给outcome属性
// 即不会调用上面介绍的set方法，将callable任务置为空，也不会修改FutureTask的状态，
// 为此任务可以重复被执行
// 但是如果任务执行出现异常，还会调用上面介绍的setException方法，
// 如果想要使用此方法，因为是protect，可以写个类继承于FutureTask类，
// 这样的话任务可以重复的被执行，但是有个问题是获取不到任务的结果
protected boolean runAndReset() {
	// 如果状态不是初始化状态，直接返回，避免重复执行，
	// 或者使用UnSafe的cas将FutureTask实例的runner属性设置为当前线程，
	// 即将要执行该任务的线程，如果设置失败也直接返回，
	// 表明此任务有线程正在执行，避免重复执行
	if (state != NEW ||
		!UNSAFE.compareAndSwapObject(this, runnerOffset,
									 null, Thread.currentThread()))
		//状态不是NEW或者设置执行线程失败，直接退出
		return false;
	//设置初始的执行状态为false
	boolean ran = false;
	//将FutureTask的属性state赋值给局部变量s
	int s = state;
	try {
		//传入进来的callable对象赋值给局部变量c
		Callable<V> c = callable;
		//传入进来的callable对象不为空并且FutureTask的状态为NEW 
		if (c != null && s == NEW) {
			try {
				//调用callable类的call方法
				c.call(); // don't set result
				//如果callable类的call方法正常执行，将其ran标志位设置为true
				ran = true;
			} catch (Throwable ex) {
				// 将异常的对象赋值给FutureTask的outcome属性
				setException(ex);
			}
		}
	} finally {
		// 最后将记录执行当前任务的线程的FutureTask的属性runner置为空
		runner = null;
		// 获取当前FutureTask实例的属性state状态值 
		s = state;
		// 当前状态属于中断中，或者已中断
		if (s >= INTERRUPTING)
			handlePossibleCancellationInterrupt(s);
	}
	// 因为runAndReset方法不会修改FutureTask的状态，
	// 如果当前任务正常执行完成并且当前FutureTask的属性state状态为NEW状态值
	return ran && s == NEW;
}

//判断FutureTask任务是否已经取消，state不管是取消、还是中断中、已中断，任务都是已取消
public boolean isCancelled() {
    // 如果FutureTask的属性state状态值大于等于CANCELLED
    // 不管任务是被取消还是中断（INTERRUPTEING、INTERRUPTED值）
    return state >= CANCELLED;
}

//判断FutureTask任务是否已经完成，state不是NEW初始化状态，任务都是已完成
public boolean isDone() {
    //state不等于初始化状态
    return state != NEW;
}

// 取消当前任务，如果mayInterruptIfRunning参数是true的话
// 此方法中断当前任务，不管取消当前任务还是中断当前任务，
// 只有FutureTask任务的属性state为NEW，才能取消、中断任务
public boolean cancel(boolean mayInterruptIfRunning) {
	// 如果当前任务的状态state不是NEW状态值，
	// 或者根据mayInterruptIfRunning参数，
	// 使用UnSafe的cas将状态NEW替换为INTERRUPTING或者CANCELLED状态值失败
	// 直接退出 
	if (!(state == NEW &&
		  UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
			  mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
		return false;
	try {
		// 如果传入进来的mayInterruptIfRunning的标志位是true
		// 表明要中断执行任务的线程
		if (mayInterruptIfRunning) {
			try {
				//获取执行任务的线程
				Thread t = runner;
				//执行任务的线程不为空
				if (t != null)
					//中断执行任务的线程
					t.interrupt();
			} finally { // final state
				// 将FutureTask的状态state设置为INTERRUPTED
				// 表明FutureTask的状态为已中断
				UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
			}
		}
	} finally {
		// 在任务执行完成（包括任务取消、正常结束、发生异常），
		// 将等待队列置空，唤醒其所有等待队列中的所有节点对应的阻塞线程，
		// 同时让任务执行体置空，
		finishCompletion();
	}
	//根据mayInterruptIfRunning参数取消或者中断任务成功
	return true;
}

// 其他线程调用FutureTask任务的get方法获取任务的执行结果，
// 比如main线程调用t线程执行的任务FutureTask的get方法，
// 如果没法马上获取到值，阻塞到任务执行完成，
// get方法可能抛出执行异常，中断异常、或者取消异常    
public V get() throws InterruptedException, ExecutionException {
	//获取FutureTask的属性state值
	int s = state;
	//如果当前状态小于等于COMPLETING，调用下面介绍的awaitDone方法
	if (s <= COMPLETING) 
		//awaitDone方法看下面对此方法的介绍
		s = awaitDone(false, 0L);
	// 根据任务执行完成的状态，判断是直接返回结果，
	// 还是抛出执行异常，或者取消异常，看下面对此report方法的介绍
	return report(s);
}
    
// 和上面get方法一样，只是此方法是支持超时获取结果
// 如果在超时FutureTask的状态还是小于等于COMPLETING,则会抛出超时异常，
// 还会有其他异常中断异常、执行异常、取消异常    
public V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException {
	//如果传入unit是空，抛出空指针异常
	if (unit == null)
		throw new NullPointerException();
	//获取FutureTask的属性state值 
	int s = state;
	// 如果当前状态小于等于COMPLETING，并且调用awaitDone()方法，
	// 超时，状态还是COMPLETING，抛出超时异常
	if (s <= COMPLETING &&
		(s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
		throw new TimeoutException();
	// 根据任务执行完成的状态，判断是直接返回结果，还是抛出执行异常，
	// 或者取消异常，看下面对此report方法的介绍
	return report(s);
}    

//等待任务完成，返回任务执行完成的状态
private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
	// 如果传入的timed是true支持超时，deadline为当前时间加上超时时间，
	// 或者阻塞直到唤醒，即调用LockSupport.unpark方法
	final long deadline = timed ? System.nanoTime() + nanos : 0L;
	WaitNode q = null;
	//将其要阻塞的当前节点设置为等待队列的起始节点的标志位
	boolean queued = false;
	for (;;) {
		//如果当前要阻塞的线程，被中断 
		if (Thread.interrupted()) {
			//移除当前节点，看下面removeWaiter方法的介绍
			removeWaiter(q);
			//抛出中断异常
			throw new InterruptedException();
		}
		//获取FutureTask的属性state值
		int s = state;
		//如果当前任务已经完成，不管是任务正常执行完成，
		//还是已取消、中断，都是已完成
		if (s > COMPLETING) {
			//如果当前要阻塞的节点不为空
			if (q != null)
				//将其要阻塞节点的对应线程置为空
				q.thread = null;
			//返回任务的状态
			return s;
		}
		//如果当前任务已经在完成中，不阻塞当前节点
		else if (s == COMPLETING) // cannot time out yet
			//让其当前节点对应的线程，让步
			Thread.yield();
		//要阻塞的节点为空
		else if (q == null)
			//构造要阻塞的节点
			q = new WaitNode();
		else if (!queued)
			//将其当前要阻塞的节点设置为等待队列的起始节点
			//下一节点为上一次节点的起始节点
			queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
												 q.next = waiters, q);
		//如果当前传入进来的标志位timed是true，即支持超时
		else if (timed) {
			//得到要阻塞的时间 
			nanos = deadline - System.nanoTime(); 
			//如果当前要阻塞的超时时间小于等于0，表明已超时
			if (nanos <= 0L) {
				//移除当前要等待的节点
				removeWaiter(q);
				//返回状态 
				return state;
			}
			//超时的阻塞当前要获取任务结果的线程
			LockSupport.parkNanos(this, nanos);
		}
		else
			//否则的话阻塞当前要获取任务结果的线程
			//除非调用LockSupport.unpark方法才能唤醒
			LockSupport.park(this);
	}
}

@SuppressWarnings("unchecked")
private V report(int s) throws ExecutionException {
	Object x = outcome;
	//如果当前传入进来的状态是NORMAL，即任务正常执行完成，返回结果
	if (s == NORMAL)
		//返回任务正常执行完成的结果
		return (V)x;
	//如果当前传入进来的状态是已取消或者被中断中、已中断
	if (s >= CANCELLED)
		//抛出取消异常
		throw new CancellationException();
	//否则抛出执行异常 
	throw new ExecutionException((Throwable)x);
}

//移除传入的节点，如果有多个线程调用FutureTask的get方法，队列有很多节点，会影响性能
private void removeWaiter(WaitNode node) {
	//如果传入的节点不为空
	if (node != null) {
		//将其传入节点的对应属性thread线程置为空
		node.thread = null;
		retry:
		//重新开始移除节点
		for (;;) {          // restart on removeWaiter race
			//从FutureTask队列的起始节点开始，如果节点不为空，开始循环
			for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
				//获取当前节点的下一节点
				s = q.next;
				//如果当前节点对应的线程不为空
				if (q.thread != null)
					//将其当前节点设置为前置节点
					pred = q;
				//如果当前节点对应的线程为空，
				// 将其当前节点的下一节点设置为前置节点，即上一个节点，移除当前节点
				else if (pred != null) {
					pred.next = s;
					//如果上一次节点的线程也被置为空，重新循环移除节点
					if (pred.thread == null) // check for race
						continue retry;
				}
				//只有两个节点，将当前节点的下一节点设置为队列的起始节点
				else if (!UNSAFE.compareAndSwapObject(this, waitersOffset,
													  q, s))
					//设置失败重新循环获取节点
					continue retry;
			}
			//退出循环
			break;
		}
	}
}
```
