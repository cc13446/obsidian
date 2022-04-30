# CAS
## 概述
CAS的全称为Compare-And-Swap，直译就是对比交换，是一条CPU的原子指令，其作用是让CPU先进行比较两个值是否相等，然后原子地更新某个位置的值，其实现方式是基于硬件平台的汇编指令，就是说CAS是靠硬件实现的，JVM只是封装了汇编调用。

简单解释：CAS操作需要输入两个数值，一个旧值(期望操作前的值)和一个新值，在操作期间先比较下在旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换。

## 问题
### ABA问题
因为CAS需要在操作值的时候，检查值有没有发生变化，比如没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时则会发现它的值没有发生变化，但是实际上却变化了。

ABA问题的解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么A->B->A就会变成1A->2B->3A。

### 循环时间长开销大
自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销。

### 只能保证一个共享变量的原子操作
当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。

还有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如，有两个共享变量`i = 2, j = a`，合并一下`ij = 2a`，然后用CAS来操作ij。

JDK提供了AtomicReference类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。

# UnSafe类详解
Unsafe是位于sun.misc包下的一个类，主要提供一些用于执行低级别、不安全操作的方法，如直接访问系统内存资源、自主管理内存资源等，这些方法在提升Java运行效率、增强Java语言底层资源操作能力方面起到了很大的作用。但由于Unsafe类使Java语言拥有了类似C语言指针一样操作内存空间的能力，这无疑也增加了程序发生相关指针问题的风险。在程序中过度、不正确使用Unsafe类会使得程序出错的概率变大，使得Java这种安全的语言变得不再安全，因此对Unsafe的使用一定要慎重。这个类尽管里面的方法都是 public 的，但是并没有办法使用它们。

![[Pasted image 20220430162035.png]]

一些反编译出来的代码：
```java
public final int getAndAddInt(Object paramObject, long paramLong, 
							  int paramInt) {
    int i;
    do
      i = getIntVolatile(paramObject, paramLong);
    while (!compareAndSwapInt(paramObject, paramLong, i, i + paramInt));
    return i;
  }

  public final long getAndAddLong(Object paramObject, long paramLong1, 
							  long paramLong2) {
    long l;
    do
      l = getLongVolatile(paramObject, paramLong1);
    while (!compareAndSwapLong(paramObject, paramLong1, l, l + paramLong2));
    return l;
  }

  public final int getAndSetInt(Object paramObject, long paramLong, 
							  int paramInt) {
    int i;
    do
      i = getIntVolatile(paramObject, paramLong);
    while (!compareAndSwapInt(paramObject, paramLong, i, paramInt));
    return i;
  }

  public final long getAndSetLong(Object paramObject, long paramLong1, 
							  long paramLong2) {
    long l;
    do
      l = getLongVolatile(paramObject, paramLong1);
    while (!compareAndSwapLong(paramObject, paramLong1, l, paramLong2));
    return l;
  }

  public final Object getAndSetObject(Object paramObject1, long paramLong,
							  Object paramObject2) {
    Object localObject;
    do
      localObject = getObjectVolatile(paramObject1, paramLong);
    while (!compareAndSwapObject(paramObject1, paramLong, localObject, paramObject2));
    return localObject;
  }
```
从源码中发现，内部使用自旋的方式进行CAS更新。原子操作其实只支持下面三个方法。

```java
public final native boolean compareAndSwapObject(Object paramObject1, long paramLong, Object paramObject2, Object paramObject3);

public final native boolean compareAndSwapInt(Object paramObject, long paramLong, int paramInt1, int paramInt2);

public final native boolean compareAndSwapLong(Object paramObject, long paramLong1, long paramLong2, long paramLong3);
```
## Unsafe底层
不妨再看看Unsafe的`compareAndSwapXXX`方法来实现CAS操作，它是一个本地方法，实现位于unsafe.cpp中。
```cpp
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END
```
它通过 `Atomic::cmpxchg` 来实现比较和替换操作。其中x是即将更新的值，e是原内存的值。`Atomic::cmpxchg`通过汇编代码来实现，不同平台实现方式不同。

## Unsafe其它功能
Unsafe 提供了硬件级别的操作，比如说获取某个属性在内存中的位置，比如说修改对象的字段值，即使它是私有的。不过 Java 本身就是为了屏蔽底层的差异，对于一般的开发而言也很少会有这样的需求。

例如：
```java
public native long staticFieldOffset(Field paramField);
```

可以用来获取给定的 paramField 的内存地址偏移量，这个值对于给定的 field 是唯一的且是固定不变的。
```java
public native int arrayBaseOffset(Class paramClass);
public native int arrayIndexScale(Class paramClass);
```

前一个方法是用来获取数组第一个元素的偏移地址，后一个方法是用来获取数组的转换因子即数组中元素的增量地址的。

```java
public native long allocateMemory(long paramLong);
public native long reallocateMemory(long paramLong1, long paramLong2);
public native void freeMemory(long paramLong);
```
分别用来分配内存，扩充内存和释放内存的。

# 原子类
JDK中提供了13个原子操作类。

## 原子更新基本类型

使用原子的方式更新基本类型，Atomic包提供了以下3个类。
-   `AtomicBoolean`: 原子更新布尔类型。
-   `AtomicInteger`: 原子更新整型。
-   `AtomicLong`: 原子更新长整型。

## 原子更新数组

通过原子的方式更新数组里的某个元素，Atomic包提供了以下的3个类：
-   `AtomicIntegerArray`: 原子更新整型数组里的元素。
-   `AtomicLongArray`: 原子更新长整型数组里的元素。
-   `AtomicReferenceArray`: 原子更新引用类型数组里的元素。

## 原子更新引用类型
Atomic包提供了以下3个类：
-   `AtomicReference`: 原子更新引用类型。
-   `AtomicStampedReference`: 原子更新引用类型, 内部使用Pair来存储元素值及其版本号。
-   `AtomicMarkableReferce`: 原子更新带有标记位的引用类型。

## 原子更新字段类

Atomic包提供了四个类进行原子字段更新：
-   `AtomicIntegerFieldUpdater`: 原子更新整型的字段的更新器。
-   `AtomicLongFieldUpdater`: 原子更新长整型字段的更新器。
-   `AtomicStampedFieldUpdater`: 原子更新带有版本号的引用类型。
-   `AtomicReferenceFieldUpdater`: 上面已经说过此处不在赘述。

这四个类的使用方式都差不多，是基于反射的原子更新字段的值。