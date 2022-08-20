# ODL基本对象的的设计与实现
这几个基本对象是构成ODL MD-SAL框架的基础，与YANG语言有着直接的渊源。YANG语言将数据的层次结构建模为树，称为数据树。数据树中每个节点都有一个名称，以及一个值或者一组子节点。本文的基本对象就是对YANG语言里元素命名、数据树的索引和数据节点定义的抽象，也即：
1. `QName`
2. `YangInstanceIdentifier`
3. `NomalizedNode`

## QName
基本元素命名定义的抽象。简单理解就是添加了命名空间的成员名称。`QName`来源于XML，由XML的名字空间和XML元素名称组成，构成格式是`namespace:local name`，例如：

```xml
<xsl:template match="foo">
</xsl:template>
```

这里的`QName`就是`xsl:template`。ODL的`yangtools`对`QName`的定义与XML里的定义非常类似，但是又不是完全相同。ODL中的定义增加了YANG模型定义文件里面的`revision`元属性，YANG语言中用`namespace`和`revision`这两个元属性来标识一个`Module`。

## Yang Instance Identifier
YANG定义的数据模型就是一个数据树，有一个内建类型`instance-identifier`用来唯一标识数据树中的某个节点。对应的ODL中也有基本的类`YangInstanceIdentifier`，这是一个分层的、基于内容的、唯一的标识符，用来对数据树中数据项的寻址，代表了数据树中某个节点的路径。

### Path接口定义
XML中有`XPath`，是一种用类似目录树的方法来描述XMl文档中的路径，这两种路径的共同点都是用`\`来表示上下层级间的间隔，中间是节点层次的名称。在`XPath`中我们还可以用运算符来对树中的条目进行过滤和筛选。Yang语言对`XPath`的简化格式的子集。

路径的特点
1. 相对性：描述一个路径从某个节点到另一个节点的路径
2. 若干条路径拼接起来还是路径

```java
public interface Path<P extends Path<P>> { 
	boolean contains(@NonNull P other); 
}
```

### Yang Instance Identifier 定义
YII实现了Path接口，表示了数据树中的节点访问路径的定义。

```java
public abstract class YangInstanceIdentifier implements 
					Path<YangInstanceIdentifier>, 
					Immutable, Serializable { 
	private final int hash; 
	
	// ...... 
	
	public abstract List<PathArgument> getPathArguments(); 
	
	boolean contains(@Nonnull final YangInstanceIdentifier other){
		//...
	}
	
}
```

文件系统的目录路径由文件夹名称组成、XPath由XML的元素名称和谓语表达式组成。在ODL中，YII由`PathArgument`组成，具体来说就是一组有序的`PathArgument`组成一条访问路径。

```java
public interface PathArgument extends Comparable<PathArgument>
									  Immutable, 
									  Serializable { 
	// 构成路径的基本参数及时数据树中的节点名
	QName getNodeType(); 
	// 可以表示成一个包含其前驱节点路径参数的字符串
	String toRelativeString(PathArgument previous); 
}
```

## Nomalized Node
