# 概述
`ArrayList`实现了`List`接口，是顺序容器，即元素存放的数据与放进去的顺序相同，允许放入`null`元素。每个`ArrayList`都有一个`capacity`，表示底层数组的实际大小。

# 并发支持
`ArrayList`没有实现同步，如果需要多个线程并发访问，用户可以手动同步，也可使用Vector替代。

# 底层数据实现
Java泛型只是编译器提供的语法糖，所以这里的数组是一个`Object`数组，以便能够容纳任何类型的对象。
```java
transient Object[] elementData; 
private int size
```

## `transient` 关键字
当需要对某个对象进行序列化的时候，只需要实现`Serilizable`接口就可以实现，但是对于某些属性可能涉及到用户的敏感信息不希望进行序列化，这些信息对应的变量就可以加上`transient`关键字，这样这个字段的生命周期仅存于调用者的内存中而不会写到磁盘里持久化。

# 自动扩容
当向容器中添加元素时，如果容量不足，容器会自动增大底层数组的大小。数组进行扩容时，会将老数组中的元素重新拷贝一份到新的数组中，每次数组容量的增长大约是其原容量的1.5倍。这种操作的代价是很高的，因此在实际使用时，我们应该尽量避免数组容量的扩张。当我们可预知要保存的元素的多少时，要在构造`ArrayList`实例时，就指定其容量，以避免数组扩容的发生。或者根据实际需求，通过调用`ensureCapacity`方法来手动增加`ArrayList`实例的容量。数组扩容通过一个公开的方法`ensureCapacity(int minCapacity)`来实现。
```java
public void ensureCapacity(int minCapacity) {
	// 避免重复初始化
	int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) 
		? 0 : DEFAULT_CAPACITY; 
	if (minCapacity > minExpand) {
		ensureExplicitCapacity(minCapacity); 
	} 
}

private void ensureCapacityInternal(int minCapacity) { 
	if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) { 
		minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity); 
	} 
	ensureExplicitCapacity(minCapacity); 
} 

private void ensureExplicitCapacity(int minCapacity) { 
	// 迭代器会在创建的时候复制一个modCount的副本，本体修改的时候可以抛出并发错误
	modCount++; 
	// 进行扩容
	if (minCapacity - elementData.length > 0) 
		grow(minCapacity); 
}

// 最长数组限制 创建更大的数组可能会导致 OutOfMemoryError
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8; 

private void grow(int minCapacity) { 
	// overflow-conscious code 
	int oldCapacity = elementData.length; 
	// 增加1.5倍
	int newCapacity = oldCapacity + (oldCapacity >> 1); 
	if (newCapacity - minCapacity < 0) 
		newCapacity = minCapacity;
	if (newCapacity - MAX_ARRAY_SIZE > 0) 
		newCapacity = hugeCapacity(minCapacity);
	elementData = Arrays.copyOf(elementData, newCapacity); 
} 

private static int hugeCapacity(int minCapacity) { 
	if (minCapacity < 0) // overflow 
		throw new OutOfMemoryError(); 
	return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE; 
}
```

# `Array.copyOf()`
`Array.copyOf()` ⽤于复制指定的数组内容以达到扩容的⽬的，该⽅法对不同的基本数据类型都有对应的重载⽅法。
```java
public static <T> T[] copyOf(T[] original, int newLength) {
        return (T[]) copyOf(original, newLength, original.getClass());
}

public static <T,U> T[] copyOf(U[] original, int newLength, 
							   Class<? extends T[]> newType) {
	// 类型相等直接创建，类型不等则需要用反射去创建
	// getComponentType 本地方法，返回数组内的元素类型，不是数组时，返回null
    T[] copy = ((Object)newType == (Object)Object[].class)
        ? (T[]) new Object[newLength]
        : (T[]) Array.newInstance(newType.getComponentType(), newLength);
    System.arraycopy(original, 0, copy, 0, 
					 Math.min(original.length, newLength));
    return copy;
}

@param      src      the source array.
@param      srcPos   starting position in the source array.
@param      dest     the destination array.
@param      destPos  starting position in the destination data.
@param      length   the number of array elements to be copied.
public static native void arraycopy(Object src,  int  srcPos,
                                    Object dest, int destPos,
                                    int length);
```


# Fail-Fast机制:
`ArrayList`也采用了快速失败的机制，通过记录`modCount`参数来实现。在面对并发的修改时，迭代器很快就会完全失败，而不是冒着在将来某个不确定时间发生任意不确定行为的风险。