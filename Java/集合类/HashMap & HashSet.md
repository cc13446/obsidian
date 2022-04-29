# HashSet
`HashSet`和`HashMap`在Java中有相同的实现，前者仅仅是对后者做了一层包装，也就是说`HashSet`里面有一个`HashMap`(适配器模式)。
```java
//HashSet是对HashMap的简单包装
public class HashSet<E>
{
	private transient HashMap<E, Object> map;
	
    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();
	
    public HashSet() {
        map = new HashMap<>();
    }
	
	//简单的方法转换
    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

}
```
# HashMap
## 概述
