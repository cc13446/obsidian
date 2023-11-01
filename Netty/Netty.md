转载自[多图详解-Netty](https://anye3210.github.io/2021/08/22/%E5%A4%9A%E5%9B%BE%E8%AF%A6%E8%A7%A3-Netty/)

## Netty 线程模型

Netty 线程模型是基于主从 Reactor 多线程模型优化而来的，整体架构如下图所示

![[Pasted image 20220505131143.png]]

Netty 的线程模型主要分为两部分，分别是 `BossGroup` 和 `WorkerGroup`，它们都分别管理一个或多个 `NioEventLoop`。每个 `NioEventLoop` 对应着一个线程，一个 `Selector`，一个 `Executor` 和一个 `TaskQueue`。

`NioEventLoop` 可以理解成一个事件循环，当程序启动后每个 `NioEventLoop` 都会通过 `Executor` 启动一个线程，开始执行事件循环，在循环中 `Selector` 会通过 `select` 方法阻塞并监听就绪事件，当有事件到来时通过 `processSeelectedKeys` 方法处理 `Selector` 事件，之后再通过 `runAllTasks` 方法处理其他的任务。

与前面介绍的 主从 `Reactor` 多线程模型类似，`BossGoup` 负责连接事件，当建立连接之后会生成一个 `NioSocketChannel` 并注册到 `WorkGroup` 其中一个 `NioEventLoop` 的 `Selector` 上。`WokerGroup` 中的 `NioEventLoop` 负责处理数据请求，当请求到来时会调用 `processSelectedKeys` 方法，其中的业务处理会依次经过 `Pipeline` 中的多个 `Handler`。