# 概述
堆分为最大堆和最小堆，即中每一个节点的值都必须大于等于（或小于等于）其子树中每个节点的值。堆在逻辑上通过二叉树来实现，并拥有以下特点
-   堆总是一棵完全二叉树
-   堆中每一个节点的值都必须大于等于（或小于等于）其子树中每个节点的值

由于完全二叉树适合利用数组来存储，所以堆在物理上可以通过数组来存储。

# 插入操作
![[Heap_insert.png]]
对于插入操作，我们要保证在插入元素之后，堆依然要保证其两个性质。我们先将元素添加到数组的末尾，也就是完全二叉树的最后一层的空节点位置，然后再跟父元素递归调整。

# 删除操作
![[Heap_delete.png]]
由于堆的特殊性质，堆中每一个节点的值都必须大于等于（或小于等于）其子树中每个节点的值，所以堆中的最大值（最小值）必在堆顶，现在我们把堆顶元素删除了，就不是二叉树了。一个解决的方式是：将数组末尾的元素移动到堆顶，然后递归的调整堆顶和子女的关系。即对于不满足父子节点大小关系的结点，元素应该和子节点比较，如果大于等于子节点或者没有子节点，停止比较；否则，选择子节点中最大的元素，进行交换。这个过程被称为自上而下的堆化过程。