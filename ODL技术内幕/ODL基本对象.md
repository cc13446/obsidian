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
数据树节点的抽象定义，有多种类型：
1. `Normalized Node`：基础类型，其他类型继承该基础类型，包含一个`idetifier`和一个`value`
2. `DataContainerNode`：所有可包含子节点的节点
3. `ContainerNode`：非重复的，可包含多个子节点的节点，对应Yang的`container`
4. `MapEntryNode`：可多次出现的节点，通过key进行唯一标识，对应Yang中`list`的一个实例
5. `ChoiceNode`：非重复出现，但可能包含不同类型的值节点，对应Yang中`choice`和`case`
6. `AugmentationNode`：Yang中`augment`
7. `LeafNode`：不含子节点，对应Yang中的`leaf`
8. `LeafSetEntryNode`：可以多次重复的叶子结点，对应Yang中`leaf-list`结点的一个实例
9. `LeafSetNode`：对应Yang中的`leaf-list`
10. `MapNode`：包含`MapEntryNode`，对应Yang中的`list`

```java
public interface NormalizedNode<K extends PathArgument, V> extends Idebtifiable<K> {
	// 名字
	QName getNodeType();
	@Override
	// 唯一标识
	@Nonnull K getIdentifier();
	@Nonnull V getValue();
}
```

## 数据树的设计与实现
在ODL中，被控制的网络被看作一个由YANG语言建模的巨大的状态机，YANG语言设计的数据模型就是一个层次化的树状数据模型。以下是几个基本概念：
1. 数据树：一种实例化的树形结构，表示所建模的问题域的配置或操作状态数据
2. 状态数据树：由网络/系统发布、上报的状态，表示应用程序持续观察到的网络/系统的状态
3. 配置数据树：系统或网络的预期状态，是由用户给出的配置，表达其想要网络达到的期望状态
4. 数据树标识：数据树中特定子树的唯一标识符，它由一个类型和子树根节点的实例标识符组成
5. 数据树快照：只读快照
6. 数据树变更：基于快照对数据树做的变更，可以根据快照和变更获得新的快照
7. 数据树候选者：已验证且合法有效的数据树变更，该候选者可被原子提交到数据树
8. MCVV：多版本并发控制

ODL MD-SAL中没有对配置和状态数据树分别单独设计，他们遵循完全一致的设计接口和实现逻辑，只是被初始化为两个单独的实例，并且都可以通过数据树标识完全寻址。协议插件或者应用程序访问的是配置树还是状态树是在调用接口时由入参指定的，系统接收到用户的预期配置到系统真正达到期望的状态是一个异步的过程。

数据树标识不包含资源的访问机制，MD-SAL提供了对数据树位置的透明访问，只需要知道传入的是配置树还是状态树并且加上YII即可。通过YII查找数据项需要分两步执行：
1. 执行最长前缀匹配以定位改标识符的存储后端实例
2. 掩码路径元素由存储引擎解析

MVCC可以避免并发访问下的一部分加锁操作，每个对数据的操作都会看到一个一致性的快照，在操作时对数据的操作并不是直接生效的，而是生成数据的不同版本，最后在正式提交前，执行冲突检测，如果不存在冲突，则新版本数据正式被提交生效。
