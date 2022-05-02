# Fork/Join框架简介
`Fork/Join`框架是Java并发工具包中的一种可以将一个大任务拆分为很多小任务来异步执行的工具，自`JDK1.7`引入。

# 三个模块及关系

Fork/Join框架主要包含三个模块:

-   任务对象: `ForkJoinTask` (包括`RecursiveTask`、`RecursiveAction` 和 `CountedCompleter`)
-   执行Fork/Join任务的线程: `ForkJoinWorkerThread`
-   线程池: `ForkJoinPool`

`ForkJoinPool`可以通过池中的`ForkJoinWorkerThread`来处理`ForkJoinTask`任务。

```java
// from 《A Java Fork/Join Framework》Dong Lea
Result solve(Problem problem) {
	if (problem is small)
 		directly solve problem
 	else {
 		split problem into independent parts
 		fork new subtasks to solve each part
 		join all subtasks
 		compose result from subresults
	}
}
```

`ForkJoinPool` 只接收 `ForkJoinTask` 任务( `Runnable/Callable` 被封装成 `ForkJoinTask` 类型的任务)，`RecursiveTask` 是 `ForkJoinTask` 的子类，是可以递归执行的 `ForkJoinTask`，`RecursiveAction` 是一个无返回值的 `RecursiveTask`，`CountedCompleter` 在任务完成执行后会触发执行一个自定义的回调函数。在实际运用中，我们一般都会继承 `RecursiveTask` 、`RecursiveAction` 或 `CountedCompleter` 来实现我们的业务需求，而不会直接继承 `ForkJoinTask` 类。

# 核心思想: 分治算法
分治算法(Divide-and-Conquer)把任务递归的拆分为各个子任务，这样可以更好的利用系统资源，尽可能的使用所有可用的计算能力来提升应用性能。

![[Pasted image 20220502112421.png]]

# 核心思想: 工作窃取算法
工作窃取(work-stealing)算法: 线程池内的所有工作线程都尝试找到并执行已经提交的任务，或者是被其他活动任务创建的子任务(如果不存在就阻塞等待)。这种特性使得 `ForkJoinPool` 在运行多个可以产生子任务的任务、或者是提交的许多小任务时效率更高。尤其是构建异步模型的 `ForkJoinPool` 时，对不需要合并`join`的事件类型任务也非常适用。

在 `ForkJoinPool` 中，线程池中每个工作线程(`ForkJoinWorkerThread`)都对应一个任务队列(`WorkQueue`)，工作线程优先处理来自自身队列的任务(LIFO或FIFO顺序，参数 mode 决定)，然后以FIFO的顺序随机窃取其他队列中的任务。

具体思路如下:
-   每个线程都有自己的一个WorkQueue，该工作队列是一个双端队列。
-   队列支持三个功能`push`、`pop`、`poll`
-   push/pop只能被队列的所有者线程调用，而poll可以被其他线程调用。
-   划分的子任务调用fork时，都会被push到自己的队列中。
-   默认情况下，工作线程从自己的双端队列获出任务并执行。
-   当自己的队列为空时，线程随机从另一个线程的队列末尾调用poll方法窃取任务。

# 执行流程

`ForkJoinPool` 中的任务执行分两种:
-   直接通过 FJP 提交的外部任务，存放在 workQueues 的偶数槽位
-   通过内部 fork 分割的子任务(Worker task)，存放在 workQueues 的奇数槽位。

![[Pasted image 20220502123840.png]]