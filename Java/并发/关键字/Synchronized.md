转载自`https://pdai.tech/`
# 对象锁和类锁
1. 对象锁：包括方法锁(默认锁对象为`this`,当前实例对象)和同步代码块锁(自己指定锁对象)
2. 类锁：指`synchronize`修饰静态的方法或指定锁对象为`Class`对象

`Synchronized` 内置锁是一种对象锁，作用粒度是对象，可以用来实现对临界资源的同步互斥访问，是可重入的。其可重入最大的作用是避免死锁，子类同步方法调用了父类同步方法，如没有可重入的特性，则会发生死锁。

# 底层实现
## Java 对象头
在JVM中，对象在内存中的布局分为三块区域：对象头、实例数据和对齐填充。
1.  实例数据：存放类的属性数据信息，包括父类的属性信息
2.  对齐填充：由于虚拟机要求对象起始地址必须是8字节的整数倍，所以要填充。
3.  对象头：
	1. 一般占有2个机器码(32位虚拟机：1个机器码4字节，64位虚拟机：1个机器码8个字节)
	2. 数组类型需要3个机器码，用一个机器码来记录数组长度。

Hotspot虚拟机的对象头主要包括两部分数据：`Mark Word`、`Class Pointer`
1. `Class Pointer`：对象指向它的类元数据的指针
2. `Mark Word`：用于存储对象自身的运行时数据，它是实现轻量级锁和偏向锁的关键

`Mark Word`用于存储对象自身的运行时数据

![[Pasted image 20220430111457.png]]

## Monitor
任何一个对象都有一个`Monitor`与之关联，当且一个`Monitor`被持有后，它将处于锁定状态。

`Synchronized`在JVM里的实现都是基于进入和退出`Monitor`对象来实现方法同步和代码块同步的，虽然具体实现细节不一样，但是都可以通过成对的`MonitorEnter`和`MonitorExit`指令来实现。 
1.  `MonitorEnter`：插入在同步代码块的开始位置，尝试获取该对象`Monitor`的所有权
2.  `MonitorExit`：插入在方法结束处和异常处，JVM保证与`MonitorEnter`相对应

在Java的设计中 ，每一个Java对象都有一把锁，即内部锁或者`Monitor`锁。也就是`Synchronized`的对象锁，此时`MarkWord`锁标识位为`10`，其中指针指向的是`Monitor`对象的起始地址。

在HotSpot虚拟机中，`Monitor`是由`ObjectMonitor`实现的：
```cpp
// HotSpot虚拟机源码ObjectMonitor.hpp文件
ObjectMonitor() {
    _header       = NULL;
    _count        = 0;    // 记录个数
    _waiters      = 0,
    _recursions   = 0;    //线程的重入次数
    _object       = NULL;
    _owner        = NULL; // 指向持有ObjectMonitor对象的线程
    _WaitSet      = NULL; // 曾经获取到锁，但是调用了wait方法，线程进入待授权区
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; // 处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

每个等待锁的线程都会被封装成`ObjectWaiter`对象，它有两个队列，`_WaitSet`和 `_EntryList`，用来保存这些`ObjectWaiter`对象：
1. `Entry List`：锁已被其他线程获取，期待获取锁的线程就进入`Monitor`对象的监控区
2. `Wait Set`：曾经获取到锁，但是调用了`wait`方法，线程进入待授权区

![](Synchronized/Pasted%20image%2020220711140741.png)

当某个线程访问一段同步代码时：
1. 被包装成`ObjectWaiter`
2. 在锁已经被其它线程拥有的时候，期待获得锁的线程进入了对象锁的`Entry List`区域
3. 当前拥有锁的线程释放掉锁的时候，处于该对象锁的`Entry List`区域的线程都会抢占该锁
4. 当线程获取到对象锁后，把`Monitor`中的`owner`变量设置为当前线程，同时计数器`count`加`1`
5. 若线程调用`wait()`方法，将释放当前持有的锁，`owner`变量恢复为`null`，`count`减`1`
6. `wait()`的线程进入`Wait Set`集合中等待被唤醒，来再次获得锁
7. `Wait Set`区域的线程被`Notify/notifyAll`的时候，线程从`Wait Set`进入`Entry List`
8. 若当前线程执行完毕，也将释放锁并复位`count`的值，以便其他线程进入获取锁

`Monitor`对象存在于每个`Java`对象的对象头`Mark Word`中，`Synchronized`锁便是通过这种方式获取锁的，也是为什么`Java`中任意对象可以作为锁的原因，同时`notify/notifyAll/wait`等方法会使用到`Monitor`锁对象，所以必须在同步代码块中使用。

### wait
```C++
// Wait 方法
void ObjectSynchronizer::wait(Handle obj, jlong millis, TRAPS) {

	// ...
	// 获得Object的monitor对象(即内置锁)
	ObjectMonitor* monitor = ObjectSynchronizer::inflate(THREAD, obj());
	DTRACE_MONITOR_WAIT_PROBE(monitor, obj(), THREAD, millis);
	// 调用monitor的wait方法
	monitor->wait(millis, true, THREAD);
	// ...
	// 然后在ObjectMonitor::exit释放锁
}
// 在wait方法中调用addWaiter方法
inline void ObjectMonitor::AddWaiter(ObjectWaiter* node) {
	
	// ...
	if (_WaitSet == NULL) {
		//_WaitSet为null，就初始化_waitSet
		_WaitSet = node;
		node->_prev = node;
		node->_next = node;
	} else {
		//否则就尾插
		ObjectWaiter* head = _WaitSet ;
		ObjectWaiter* tail = head->_prev;
		assert(tail->_next == head, "invariant check");
		tail->_next = node;
		head->_prev = node;
		node->_next = head;
		node->_prev = tail;
	}
}
```

### notify
```C++
// notify方法
void ObjectSynchronizer::notify(Handle obj, TRAPS) {
	// ...
	// 调用ObjectSynchronizer::inflate方法
	ObjectSynchronizer::inflate(THREAD, obj())->notify(THREAD);
}

// 通过inflate方法得到ObjectMonitor对象
ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {
	// ...
	if (mark->has_monitor()) {
	  ObjectMonitor * inf = mark->monitor() ;
	  assert (inf->header()->is_neutral(), "invariant");
	  assert (inf->object() == object, "invariant") ;
	  assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
	  return inf 
	}
	// ...
}
// 调用ObjectMonitor的notify方法
void ObjectMonitor::notify(TRAPS) {
	// ...
	// 调用DequeueWaiter方法移出_waiterSet第一个结点
	ObjectWaiter * iterator = DequeueWaiter() ;
	// 后面省略是将上面DequeueWaiter尾插入_EntrySet的操作
	// ...
}
```

## MonitorEnter 和 MonitorExit
在使用同步代码块的时候，`Synchronized` 会在代码块儿的前后加上 `Monitorenter` 和 `Monitorexit`
1.  `MonitorEnter`：尝试获取`Monitor`的所有权，过程如下：
    - 如果`Count`为0，则该线程进入`Monitor`，然后将`Count`设置为1，该线程为`Monitor`的所有者
    - 如果线程已经占有该`Monitor`，只是重新进入，则`count`加1
    - 如果其他线程已经占用了`Monitor`，则该线程进入阻塞状态，
    - 直到`Count`为0，再重新尝试获取`Monitor`的所有权
2.  `MonitorExit`：执行`monitorexit`的线程必须是`monitor`的所有者:
	- 指令执行时，`Count`减1，如果减到0，那线程退出`Monitor`，不再是锁的所有者
	- 其他被阻塞的线程可以尝试去获取这个锁的所有权
	- `Synchronized`在所有退出此临界区的出口前都加上了`monitorexit`

## ACC_SYNCHRONIZED 标示符
方法的同步并没有通过指令 `monitorenter` 和 `monitorexit` 来完成(理论上其实也可以通过这两条指令来实现)，不过相对于普通方法，其常量池中多了 `ACC_SYNCHRONIZED` 标示符。JVM就是根据该标示符来实现方法的同步的。

当方法调用时，调用指令将会检查方法的 `ACC_SYNCHRONIZED` 访问标志是否被设置，如果设置了，执行线程将先获取`Monitor`，获取成功之后才能执行方法体，方法执行完后再释放`Monitor`。在方法执行期间，其他任何线程都无法再获得同一个`Monitor`对象。

两种同步方式本质上没有区别，只是方法的同步是一种隐式的方式来实现，无需通过字节码来完成。两个指令的执行是JVM通过调用操作系统的互斥原语`mutex`来实现，被阻塞的线程会被挂起、等待重新调度，会导致用户态和内核态两个态之间来回切换，对性能有较大影响。

# 锁优化
`Monitor`依赖于底层的操作系统的`Mutex Lock`来实现的，但是由于使用`Mutex Lock`需要将当前线程挂起并从用户态切换到内核态来执行，这种切换的代价是非常昂贵的；然而在现实中的大部分情况下，同步方法是运行在单线程环境(无锁竞争环境)。如果每次都调用`Mutex Lock`那么将严重的影响程序的性能。在`jdk1.6`中对锁的实现引入了大量的优化，如锁粗化、锁消除、轻量级锁、偏向锁、适应性自旋等技术来减少锁操作的开销。
1. **锁粗化**：将多个连续的锁扩展成一个范围更大的锁
2. **锁消除**：通过运行时JIT编译器的逃逸分析来消除没有竞争的锁
3. **轻量级锁**：程序的大部分同步代码处于无锁竞争状态，可以避免互斥锁，只依靠CAS原子指令互斥
4. **偏向锁**：无锁竞争的情况下避免在锁获取过程中执行不必要的CAS原子指令
5. **适应性自旋**：执行CAS操作失败后，进行锁升级之前会多次尝试，尝试的时间会进行适应性调整

锁膨胀方向： 无锁 → 偏向锁 → 轻量级锁 → 重量级锁 (此过程是不可逆的)

## 自旋锁
所谓自旋锁，就是指当一个线程尝试获取某个锁时，如果该锁已被其他线程占用，就一直循环检测锁是否被释放，而不是进入线程挂起或睡眠状态。

自旋锁适用于锁保护的临界区很小的情况，临界区很小的话，锁占用的时间就很短。自旋等待不能替代阻塞，虽然它可以避免线程切换带来的开销，但是它占用了CPU处理器的时间。如果持有锁的线程很快就释放了锁，那么自旋的效率就非常好，反之，自旋的线程就会白白消耗掉处理的资源，它不会做任何有意义的工作，反而会带来性能上的浪费。所以说，自旋等待的时间/次数必须要有一个限度，如果自旋超过了定义的时间仍然没有获取到锁，则应该被挂起。

线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。反之，如果对于某个锁，很少有自旋能够成功，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。

## 锁消除
为了保证数据的完整性，在进行操作时需要对这部分操作进行同步控制，但是在有些情况下，JVM检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除。

锁消除的依据是逃逸分析的数据支持

## 锁粗化
原则上，我们都知道在加同步锁时，尽可能的将同步块的作用范围限制到尽量小的范围(只在共享数据的实际作用域中才进行同步，这样是为了使得需要同步的操作数量尽可能变小。在存在锁同步竞争中，也可以使得等待锁的线程尽早的拿到锁)。

大部分上述情况是完美正确的，但是如果存在连串的一系列操作都对同一个对象反复加锁和解锁，甚至加锁操作时出现在循环体中的，那即使没有线程竞争，频繁的进行互斥同步操作也会导致不必要的性能操作。
```java
public static String test04(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```
JVM会检测到这样一连串的操作都是对同一个对象加锁，那么JVM会将加锁同步的范围扩展到整个一系列操作的外部，使整个一连串的操作只需要加锁一次就可以了。

## 偏向锁
偏向锁是JDK6中的重要引进，因为HotSpot作者经过研究实践发现，在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一线程多次获得，为了让线程获得锁的代价更低，引进了偏向锁。

偏向锁是在单线程执行代码块时使用的机制，如果在多线程并发的环境下，则一定会转化为轻量级锁或者重量级锁。可以使用参数`-XX:-UseBiasedLocking`来禁止偏向锁。

引入偏向锁主要目的是：在没有多线程竞争的情况下尽量减少不必要的轻量级锁。因为轻量级锁的加锁解锁操作依赖多次的CAS原子指令，而偏向锁只需要在置换`ThreadID`的时候依赖一次CAS原子指令。偏向锁的想法是一旦线程第一次获得了监视对象，之后让监视对象偏向这个线程，之后的多次调用则可以避免CAS操作，说白了就是置个变量，如果发现为true则无需再走各种加锁或解锁流程。

轻量级锁是为了在线程交替执行同步块时提高性能，而偏向锁则是在只有一个线程执行同步块时进一步提高性能。

当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程进入和退出同步块时不需要花费CAS操作来争夺锁资源，只需要检查是否为偏向锁、锁标识为以及当前线程ID即可，处理流程如下：
1. 检测`Mark Word`是否为可偏向状态，即是否为偏向锁1，锁标识位为01
2. 若为可偏向状态，则测试线程ID是否为当前线程ID，如果是，则转到***5***，否则转到***3***
3. 如果测试线程ID不为当前线程ID，则通过CAS操作竞争锁
	- 竞争成功，则将`Mark Word`的线程ID替换为当前线程ID，此时没有其他线程试图获取锁
	- 竞争失败转到***4***
4. 通过CAS竞争锁失败，证明当前存在多线程竞争情况，就是之前已经有其他线程获取了偏向锁
	- 当到达全局安全点，获得偏向锁的线程被挂起，偏向锁升级为轻量级锁
	- 被阻塞在安全点的线程继续往下执行同步代码块
5.  执行同步代码块

CAS操作成功之后：
![](Synchronized/Pasted%20image%2020220711182657.png)

偏向锁的释放采用了一种只有竞争才会释放锁的机制，线程是不会主动去释放偏向锁，需要等待其他线程来竞争。偏向锁的撤销需要等待全局安全点，也就是`STOP THE WORLD`的时候。步骤如下：
1.  暂停拥有偏向锁的线程
2.  判断锁对象是否还处于被锁定状态
	- 没有被锁定则恢复到无锁状态，以允许其余线程竞争
	- 被锁定则要升级为轻量锁

升级轻量级锁的过程：
1. 通过 `MarkWord` 中已经存在的 `Thread Id` 找到成功获取了偏向锁的那个线程A
2. 挂起持有锁的线程
3. 在A线程的栈帧中补充上轻量级加锁时会保存的锁记录**Lock Record**
4. 将锁记录地址的指针放入对象头`Mark Word`，升级为轻量级锁状态
5. 恢复持有锁的线程A，进入轻量级锁的竞争模式
6. 此时锁仍然在线程A手中，只是将对象头中的线程ID变更为指向轻量锁记录地址的指针

### 偏向锁的批量再偏向机制
偏向锁这个机制很特殊，别的锁在执行完同步代码块后，都会有释放锁的操作，而偏向锁并没有直观意义上的释放锁操作。那另一个线程怎么直接重新获得偏向锁呢？ JVM 提供了批量再偏向机制

该机制的主要工作原理如下：
1. 引入概念 `epoch`，其本质是一个时间戳，代表了偏向锁的有效性，存在可偏向对象的`MarkWord`中
2. 对象所属的类 `class` 信息中，也会保存一个 `epoch` 值
3. 每当遇到一个全局安全点时，如果要对 `class C` 进行批量再偏向：
	1. 对 `class C` 中保存的 `epoch` 进行增加操作，得到一个新的 `epoch_new`
	2. 扫描所有持有 `class C` 实例的线程栈，根据线程栈的信息判断出该线程是否锁定了该对象
	3. 如果线程锁定了对象，将 `epoch_new` 的值赋给被锁定的对象的`epoch`中
4. 退出安全点后，当有线程需要尝试获取偏向锁时:
	1. 直接检查 `class C` 中存储的 `epoch` 值是否与目标对象中存储的 `epoch` 值相等
	2. 如果不相等，则说明该对象的偏向锁已经无效了，可以尝试对此对象重新进行偏向操作


## 轻量级锁
引入轻量级锁的主要目的是在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁，其步骤如下：
1. 线程尝试获取轻量锁时，虚拟机在当前线程的**栈帧**中建立**Lock Record**
2. 将对象头中的`Mark Word`拷贝到**Lock Record**，官方称之为 **Displaced Mark Word**
3. 虚拟机使用CAS尝试将对象`Mark Word`中的`Lock Word`更新为指向当前线程**Lock Record**的指针
4. 并且将`Lock record`里的`owner`指针指向`object mark word`。如果更新成功，则转到***5***，否则转到***6***
5. 如果更新动作成功，当前线程拥有该对象的锁，`Mark Word`的锁标志位设置为`00`，表示轻量级锁
6. 如果更新操作失败：
	1. 首先会检查对象`Mark Word`中的`Lock Word`是否指向当前线程的栈帧
	2. 如果是，就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行
	3. 否则说明多个线程竞争锁，自旋执行***3 4***
	4. 若自旋结束时仍未获得锁，轻量级锁就要膨胀为重量级锁
	5. 锁标志的状态值变为10，`Mark Word`中存储的就是指向重量级锁的指针，当前线程进入阻塞状态

轻量级锁相关内存的情况
![](Synchronized/Pasted%20image%2020220711184832.png)

轻量级锁的释放也是通过CAS操作来进行的，主要步骤如下：
1.  通过CAS操作尝试用线程栈中复制的`Displaced Mark Word`对象替换掉当前的`Mark Word`
2.  如果替换成功，整个同步过程就完成了，恢复到无锁状态01
3.  如果替换失败，说明有其他线程尝试过获取该锁，锁已膨胀，要在释放锁的同时，唤醒被挂起的线程


# synchronized的缺陷
1. `效率低`：
	- 锁的释放情况少，只有代码执行完毕或者异常结束才会释放锁
	- 试图获取锁的时候不能设定超时，不能中断一个正在使用锁的线程，而Lock可以中断和设置超时
2.  `不够灵活`：加锁和释放的时机单一，每个锁仅有一个单一的条件
3. `无法知道是否成功获得锁`：相对而言，Lock可以拿到状态