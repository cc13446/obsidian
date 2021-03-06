转载自`https://zhuanlan.zhihu.com/p/95662364`
转载自`https://www.zhihu.com/question/26943938`
转载自`https://zhuanlan.zhihu.com/p/367591714`
# Reactor模式和Proactor模式
在web服务中，处理web请求通常有两种体系结构，分别为
	- `thread-based architecture`：基于线程的架构
	- `event-driven architecture`：事件驱动模型

## thread-based architecture（基于线程的架构）
基于线程的架构，通俗的说就是：多线程并发模式，一个连接一个线程，服务器每当收到客户端的一个请求， 便开启一个独立的线程来处理。

这种模式一定程度上极大地提高了服务器的吞吐量，由于在不同线程中，之前的请求在read阻塞以后，不会影响到后续的请求。但是，仅适用于于并发量不大的场景，因为：
-   线程需要占用一定的内存资源
-   创建和销毁线程也需一定的代价
-   操作系统在切换线程也需要一定的开销
-   线程处理I/O，在等待输入或输出的这段时间处于空闲的状态，同样也会造成cpu资源的浪费

## event-driven architecture（事件驱动模型）
事件驱动体系结构是目前比较广泛使用的一种。这种方式会定义一系列的事件处理器来响应事件的发生，并且将`服务端接受连接`与`对事件的处理`分离。其中，`事件`是一种状态的改变。

### Reactor模式
在Reactor模型中，主要有四个角色：客户端连接，`Reactor`，`Acceptor`和`Handler`。这里`Acceptor`会不断地接收客户端的连接，然后将接收到的连接交由`Reactor`进行分发，最后有具体的`Handler`进行处理。改进后的`Reactor`模型相对于传统的IO模型主要有如下优点：
-   `Reactor`模型是以事件进行驱动的，其能够将接收客户端连接、网络读和网络写，以及业务计算进行拆分，从而极大的提升处理效率；
-   `Reactor`模型是异步非阻塞模型，工作线程在没有网络事件时可以处理其他的任务，而不用像传统IO那样必须阻塞等待。

#### 单Reactor 单进程 / 线程
`Java`中的NIO模式的`Selector`网络通讯，其实就是一个简单的`Reactor`模型。可以说是单线程的`Reactor`模式。

![[Pasted image 20220504113134.png]]

进程里有 `Reactor`、`Acceptor`、`Handler` 这三个对象：
-   `Reactor` 对象的作用是监听和分发事件；
-   `Acceptor` 对象的作用是获取连接；
-   `Handler` 对象的作用是处理业务；

对象里的 `select`、`accept`、`read`、`send` 是系统调用函数，`dispatch` 和 「业务处理」是需要完成的操作，其中 `dispatch` 是分发事件操作。

接下来，介绍下「单 `Reactor` 单进程」这个方案：
1.  `Reactor` 对象通过 `select` （IO 多路复用接口） 监听事件，收到事件后通过 `dispatch` 进行分发，具体分发给 `Acceptor` 对象还是 `Handler` 对象，还要看收到的事件类型；
2.  如果是连接建立的事件，则交由 `Acceptor` 对象进行处理，`Acceptor` 对象会通过 `accept` 方法获取连接，并创建一个 `Handler` 对象来处理后续的响应事件；
3. 如果不是连接建立事件， 则交由当前连接对应的 `Handler` 对象来进行响应；
4. `Handler` 对象通过 `read` -> 业务处理 -> `send` 的流程来完成完整的业务流程。

单 `Reactor` 单进程的方案因为全部工作都在同一个进程内完成，所以实现起来比较简单，不需要考虑进程间通信，也不用担心多进程竞争。

但是，这种方案存在 2 个缺点：
-   第一个缺点，因为只有一个进程，无法充分利用多核 CPU 的性能
-   第二个缺点，`Handler` 对象在业务处理时，整个进程是无法处理其他连接的事件的，如果业务处理耗时比较长，那么就造成响应的延迟

所以，单 Reactor 单进程的方案不适用计算机密集型的场景，只适用于业务处理非常快速的场景
Redis 是由 C 语言实现的，它采用的正是「单 Reactor 单进程」的方案，因为 Redis 业务处理主要是在内存中完成，操作的速度是很快的，性能瓶颈不在 CPU 上，所以 Redis 对于命令的处理是单进程的方案。

### 单 Reactor 多线程 / 多进程

如果要克服「单 Reactor 单线程 / 进程」方案的缺点，那么就需要引入多线程 / 多进程，这样就产生单 Reactor 多线程 / 多进程 的方案。

![[Pasted image 20220504125725.png]]

1. `Reactor` 对象通过 `select` 监听事件，收到事件后通过 `dispatch` 进行分发，具体分发给 `Acceptor` 对象还是 `Handler` 对象，还要看收到的事件类型；
2. 如果是连接建立的事件，则交由 `Acceptor` 对象进行处理，`Acceptor` 对象会通过 `accept` 方法获取连接，并创建一个 `Handler` 对象来处理后续的响应事件；
3. 如果不是连接建立事件， 则交由当前连接对应的 `Handler` 对象来进行响应；
4. Handler 对象不再负责业务处理，只负责数据的接收和发送，`Handler` 对象通过 `read` 读取到数据后，会将数据发给子线程里的 `Processor` 对象进行业务处理；
5. 子线程里的 `Processor` 对象就进行业务处理，处理完后，将结果发给主线程中的 `Handler` 对象，接着由 `Handler` 通过 `send` 方法将响应结果发送给 `client`；

单 Reator 多线程的方案优势在于能够充分利用多核 CPU 的性能，那既然引入多线程，那么自然就带来了多线程竞争资源的问题。

单 Reactor 多进程相比单 Reactor 多线程实现起来很麻烦，主要因为要考虑子进程和父进程的双向通信，并且父进程还得知道子进程要将数据发送给哪个客户端。而多线程间可以共享数据，虽然要额外考虑并发问题，但是这远比进程间通信的复杂度低得多，因此实际应用中也看不到单 Reactor 多进程的模式。

单 Reactor 的模式还有个问题，因为一个 Reactor 对象承担所有事件的监听和响应，而且只在主线程中运行，在面对瞬间高并发的场景时，容易成为性能的瓶颈的地方。

### 多 Reactor 多进程 / 线程

要解决「单 Reactor」的问题，就是将「单 Reactor」实现成「多 Reactor」，这样就产生了多 Reactor 多进程 / 线程的方案。

![[Pasted image 20220504133206.png]]

1. 主线程中的 `MainReactor` 对象通过 `select` 监控连接建立事件，收到事件后通过 `Acceptor` 对象中的 `accept` 获取连接，将新的连接分配给某个子线程；
2. 子线程中的 `SubReactor` 对象将 `MainReactor` 对象分配的连接加入 `select` 继续进行监听，并创建一个 `Handler` 用于处理连接的响应事件。
3. 如果有新的事件发生时，`SubReactor` 对象会调用当前连接对应的 `Handler` 对象来进行响应
4. `Handler` 对象通过 read -> 业务处理 -> send 的流程来完成完整的业务流程。

多 Reactor 多线程的方案虽然看起来复杂的，但是实际实现时比单 Reactor 多线程的方案要简单的多，原因如下：
1. 主线程和子线程分工明确，主线程只负责接收新连接，子线程负责完成后续的业务处理。
2. 主线程和子线程的交互很简单，主线程只需要把新连接传给子线程，子线程无须返回数据，直接就可以在子线程将处理结果发送给客户端。

大名鼎鼎的两个开源软件 `Netty` 和 `Memcache` 都采用了「多 Reactor 多线程」的方案。


## Proactor

前面提到的 Reactor 是非阻塞同步网络模式，而 Proactor 是异步网络模式。

这里先给大家复习下阻塞、非阻塞、同步、异步 I/O 的概念。

### 阻塞 I/O
当用户程序执行 `read` ，线程会被阻塞，一直等到内核数据准备好，并把数据从内核缓冲区拷贝到应用程序的缓冲区中，当拷贝过程完成，`read` 才会返回。阻塞等待的是「内核数据准备好」和「数据从内核态拷贝到用户态」这两个过程。

![[Pasted image 20220504134216.png]]

### 非阻塞 I/O
非阻塞的 `read` 请求在数据未准备好的情况下立即返回，可以继续往下执行，此时应用程序不断轮询内核，直到数据准备好，内核将数据拷贝到应用程序缓冲区，`read` 调用才可以获取到结果。

![[Pasted image 20220504134252.png]]

最后一次 read 调用，获取数据的过程，是一个同步的过程，是需要等待的过程。这里的同步指的是内核态的数据拷贝到用户程序的缓存区这个过程。

### 异步 I/O
而真正的异步 I/O是「内核数据准备好」和「数据从内核态拷贝到用户态」这两个过程都不用等。当我们发起 `aio_read` （异步 I/O） 之后，就立即返回，内核自动将数据从内核空间拷贝到用户空间，这个拷贝过程同样是异步的，内核自动完成的，和前面的同步操作不一样，应用程序并不需要主动发起拷贝动作。

再来理解 Reactor 和 Proactor 的区别，就比较清晰了。
1. `Reactor` 是非阻塞同步网络模式，感知的是就绪可读写事件。在每次感知到有事件发生（比如可读就绪事件）后，就需要应用进程主动调用 `read` 方法来完成数据的读取，也就是要应用进程主动将 `socket` 接收缓存中的数据读到应用进程内存中，这个过程是同步的，读取完数据后应用进程才能处理数据。
2. `Proactor` 是异步网络模式， 感知的是已完成的读写事件。在发起异步读写请求时，需要传入数据缓冲区的地址（用来存放结果数据）等信息，这样系统内核才可以自动帮我们把数据的读写工作完成，这里的读写工作全程由操作系统来做，并不需要像 Reactor 那样还需要应用进程主动发起 `read/write` 来读写数据，操作系统完成读写工作后，就会通知应用进程直接处理数据。

因此，Reactor 可以理解为「来了事件操作系统通知应用进程，让应用进程来处理」，而 Proactor 可以理解为「来了事件操作系统来处理，处理完再通知应用进程」。这里的「事件」就是有新连接、有数据可读、有数据可写的这些 I/O 事件这里的「处理」包含从驱动读取到内核以及从内核读取到用户空间。

![[Pasted image 20220504135158.png]]

Proactor 模式的工作流程：

-   `Proactor Initiator` 负责创建 `Proactor` 和 `Handler` 对象，并将 `Proactor` 和 `Handler` 都通过 `Asynchronous Operation Processor` 注册到内核；
-   `Asynchronous Operation Processor` 负责处理注册请求，并处理 I/O 操作；
-   `Asynchronous Operation Processor` 完成 I/O 操作后通知 Proactor；
-   `Proactor` 根据不同的事件类型回调不同的 `Handler` 进行业务处理；
-   `Handler` 完成业务处理；

可惜的是，在 Linux 下的异步 I/O 是不完善的， `aio` 系列函数是由 POSIX 定义的异步操作接口，不是真正的操作系统级别支持的，而是在用户空间模拟出来的异步，并且仅仅支持基于本地文件的 aio 异步操作，网络编程中的 socket 是不支持的，这也使得基于 Linux 的高性能网络程序都是使用 `Reactor` 方案。

而 Windows 里实现了一套完整的支持 socket 的异步编程接口，这套接口就是 `IOCP`，是由操作系统级别实现的异步 I/O，真正意义上异步 I/O，因此在 Windows 里实现高性能网络程序可以使用效率更高的 `Proactor` 方案。


# Select & Poll & Epoll
##  Select

```cpp
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

`select` 函数监视的文件描述符分3类，分别是 `writefds`、`readfds`、和`exceptfds`，当用户`process`调用`select`的时候，select会将需要监控的`readfds`集合拷贝到内核空间（假设监控的仅仅是socket可读），然后遍历自己监控的`socket sk`，挨个调用`sk`的`poll`逻辑以便检查该`sk`是否有可读事件，遍历完所有的`sk`后，如果没有任何一个`sk`可读，那么`select`会调用`schedule_timeout`进入`schedule`循环，使得`process`进入睡眠。如果在`timeout`时间内某个`sk`上有数据可读了，或者等待`timeout`了，则调用select的`process`会被唤醒，接下来select就是遍历监控的sk集合，挨个收集可读事件并返回给用户了，相应的伪码如下：

```cpp
int select(
    int nfds,
    fd_set *readfds,
    fd_set *writefds,
    fd_set *exceptfds,
    struct timeval *timeout);
// nfds:监控的文件描述符集里最大文件描述符加1
// readfds：监控有读数据到达文件描述符集合，传入传出参数
// writefds：监控写数据到达文件描述符集合，传入传出参数
// exceptfds：监控异常发生达文件描述符集合, 传入传出参数
// timeout：定时阻塞监控时间，3种情况
//  1.NULL，永远等下去
//  2.设置timeval，等待固定时间
//  3.设置timeval里时间均为0，检查描述字后立即返回，轮询

// select服务端伪码
// 首先一个线程不断接受客户端连接，并把socket文件描述符放到一个list里。
while(1) {
  connfd = accept(listenfd);
  fcntl(connfd, F_SETFL, O_NONBLOCK);
  fdlist.add(connfd);
}
// select函数还是返回刚刚提交的list，应用程序依然list所有的fd，
// 只不过操作系统会将准备就绪的文件描述符做上标识
// 用户层将不会再有无意义的系统调用开销。
struct timeval timeout;
int max = 0;  // 用于记录最大的fd，在轮询中时刻更新即可
// 初始化比特位
FD_ZERO(&read_fd);
while (1) {
    // 阻塞获取 每次需要把fd从用户态拷贝到内核态
    nfds = select(max + 1, &read_fd, &write_fd, NULL, &timeout);
    // 每次需要遍历所有fd，判断有无读写事件发生
    for (int i = 0; i <= max && nfds; ++i) {
        // 只读已就绪的文件描述符，不用过多遍历
        if (i == listenfd) {
            // 这里处理accept事件
            FD_SET(i, &read_fd);//将客户端socket加入到集合中
        }
        if (FD_ISSET(i, &read_fd)) {
            // 这里处理read事件
        }
    }
}
```

原理视频：

![[349279b4-9119-11eb-85d0-1278b449b310.mp4]]

`Select`存在三个问题：
1. 每次调用select，都需要把被监控的fds集合从用户态空间拷贝到内核态空间，高并发场景下这样的拷贝会使得消耗的资源是很大的。  
2. 能监听端口的数量有限，单个进程所能打开的最大连接数有`FD_SETSIZE`宏定义，其大小是32个整数的大小，当然我们可以对宏`FD_SETSIZE`进行修改，然后重新编译内核，但是性能可能会受到影响，一般该数和系统内存关系很大，具体数目可以`cat /proc/sys/fs/file-max`察看。32位机默认1024个，64位默认2048。
3. 被监控的`fds`集合中，只要有一个有数据可读，整个`socket`集合就会被遍历一次调用`sk`的`poll`函数收集可读事件



## poll

`poll`的实现和`select`非常相似，只是描述`fd`集合的方式不同。针对`select`遗留的三个问题中，`poll`只是使用`pollfd`结构而不是`select`的`fd_set`结构，这就解决了`select`的`fds`集合大小1024限制问题。但`poll`和`select`同样存在一个性能缺点就是包含大量文件描述符的数组被整体复制于用户态和内核的地址空间之间，而不论这些文件描述符是否就绪，它的开销随着文件描述符数量的增加而线性增大。

下面是`poll`的函数原型，`poll`改变了`fds`集合的描述方式，使用了`pollfd`结构而不是`select`的`fd_set`结构，使得`poll`支持的`fds`集合限制远大于`select`的1024。`poll`虽然解决了`fds`集合大小1024的限制问题，从实现来看。很明显它并没优化大量描述符数组被整体复制于用户态和内核态的地址空间之间，以及个别描述符就绪触发整体描述符集合的遍历的低效问题。poll随着监控的socket集合的增加性能线性下降，使得poll也并不适合用于大并发场景。

```cpp
int poll(struct pollfd *ufds, unsigned int nfds, int timeout);
struct pollfd {
　　int fd;           /*文件描述符*/
　　short events;     /*监控的事件*/
　　short revents;    /*监控事件中满足条件返回的事件*/
};
int poll(struct pollfd *fds, nfds_tnfds, int timeout);

// poll服务端实现伪码：
struct pollfd fds[POLL_LEN];
unsigned int nfds=0;
fds[0].fd=server_sockfd;
fds[0].events=POLLIN|POLLPRI;
nfds++;
while {
    res=poll(fds,nfds,-1);
    if(fds[0].revents&(POLLIN|POLLPRI)) {
        //执行accept并加入fds中，nfds++
        if(--res<=0) continue
    }
    //循环之后的fds
    if(fds[i].revents&(POLLIN|POLLERR )) {
        //读操作或处理异常等
        if(--res<=0) continue
    }
}
```

## epoll
在linux的网络编程中，很长的时间都在使用`select`来做事件触发。在linux新的内核中，有了一种替换它的机制，就是`epoll`。相比于select，epoll最大的好处在于它不会随着监听fd数目的增长而降低效率。如前面我们所说，在内核中的select实现中，它是采用轮询来处理的，轮询的fd数目越多，自然耗时越多。并且，在`linux/posix_types.h`头文件有这样的声明：
```c
#define __FD_SETSIZE 1024  
```
表示select最多同时监听1024个fd，当然，可以通过修改头文件再重编译内核来扩大这个数目，但这似乎并不治本。

创建一个`epoll`的句柄，`size`用来告诉内核这个监听的数目一共有多大。这个参数不同于`select()`中的第一个参数，给出最大监听的`fd+1`的值。需要注意的是，当创建好`epoll`句柄后，它就是会占用一个`fd`值，在`linux`下如果查看`/proc/`进程`id/fd/`，是能够看到这个fd的，所以在使用完`epoll`后，必须调用`close()`关闭，否则可能导致fd被耗尽。

epoll的接口非常简单，一共就三个函数：
-   `epoll_create`：创建一个`epoll`句柄
-   `epoll_ctl`：向 `epoll` 对象中添加/修改/删除要管理的连接
-   `epoll_wait`：等待其管理的连接上的 IO 事件

### epoll_create 函数

```cpp
int epoll_create(int size);
```

1. 功能：该函数生成一个 epoll 专用的文件描述符。
2. 参数size：用来告诉内核这个监听的数目一共有多大，参数 `size` 并不是限制了 `epoll` 所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议。自从 `linux 2.6.8` 之后，`size` 参数是被忽略的，也就是说可以填只有大于 0 的任意值。
3. 返回值：如果成功，返回 `poll` 专用的文件描述符，否者失败，返回-1。

#### 源码实现：

```cpp
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
    struct eventpoll *ep = NULL;

    //创建一个 eventpoll 对象
    error = ep_alloc(&ep);
}

// struct eventpoll 的定义
// file：fs/eventpoll.c
struct eventpoll {

    //sys_epoll_wait用到的等待队列
    wait_queue_head_t wq;

    //接收就绪的描述符都会放到这里
    struct list_head rdllist;

    //每个epoll对象中都有一颗红黑树
    struct rb_root rbr;

    ......
}
static int ep_alloc(struct eventpoll **pep)
{
    struct eventpoll *ep;

    //申请 epollevent 内存
    ep = kzalloc(sizeof(*ep), GFP_KERNEL);

    //初始化等待队列头
    init_waitqueue_head(&ep->wq);

    //初始化就绪列表
    INIT_LIST_HEAD(&ep->rdllist);

    //初始化红黑树指针
    ep->rbr = RB_ROOT;

    ......
}
```

其中 `eventpoll` 这个结构体中的几个成员的含义如下：
1. `wq`：等待队列链表。软中断数据就绪时会通过 `wq` 来找到阻塞在 `epoll` 对象上的用户进程。
2. `rbr`：红黑树。为了支持对海量连接的高效查找、插入和删除，`eventpoll` 内部使用的就是红黑树。通过红黑树来管理用户主进程`accept`添加进来的所有 `socket` 连接。
3. `rdllist`：就绪的描述符链表。当有连接就绪的时候，内核会把就绪的连接放到 `rdllist` 链表里。这样应用进程只需要判断链表就能找出就绪进程，而不用去遍历红黑树的所有节点

### epoll_ctl 函数

```cpp
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 
```
1. 功能：`epoll`的事件注册函数，它不同于 `select()` 是在监听事件时告诉内核要监听什么类型的事件，而是在这里先注册要监听的事件类型。
2. 参数`epfd`：`epoll` 专用的文件描述符，`epoll_create()`的返回值
3. 参数op：表示动作，用三个宏来表示：
	- `EPOLL_CTL_ADD`：注册新的 fd 到 epfd 中；
	- `EPOLL_CTL_MOD`：修改已经注册的fd的监听事件；
	- `EPOLL_CTL_DEL`：从 epfd 中删除一个 fd；
4. 参数fd：需要监听的文件描述符
5. 参数event：告诉内核要监听什么事件，`struct epoll_event` 可以是以下几个宏的集合：
	-   `EPOLLIN`：表示对应的文件描述符可以读（包括对端 `SOCKET` 正常关闭）；
	-   `EPOLLOUT`：表示对应的文件描述符可以写；
	-   `EPOLLPRI`：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
	-   `EPOLLERR`：表示对应的文件描述符发生错误；
	-   `EPOLLHUP`：表示对应的文件描述符被挂断；
	-   `EPOLLET` ：将 EPOLL 设为边缘触发(Edge Trigger)模式，非水平触发(Level Trigger)
	-   `EPOLLONESHOT`：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个 `socket` 的话，需要再次把这个 `socket` 加入到 `EPOLL` 队列里
6. 返回值：0表示成功，-1表示失败。

### epoll_wait函数

```cpp
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout); 
```
1. 功能：等待事件的产生，收集在 `epoll` 监控的事件中已经发送的事件，类似于 `select()`
2. 参数`epfd`：epoll 专用的文件描述符，`epoll_create()`的返回值
3. 参数`events`：分配好的 `epoll_event` 结构体数组，epoll 将会把发生的事件赋值到`events` 数组中
	- events 不可以是空指针，内核只负责把数据复制到这个 events 数组中，不会去帮助我们在用户态中分配内存
4. 参数`maxevents`：`maxevents` 告之内核这个 `events` 有多少个 。
5. 参数`timeout`：超时时间，单位为毫秒，为 -1 时，函数为阻塞。
6. 返回值：
- 如果成功，表示返回需要处理的事件数目
- 如果返回0，表示已超时
- 如果返回-1，表示失败

```cpp
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <fcntl.h>
#include <stdlib.h>
#include <cassert>
#include <sys/epoll.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <string.h>
#include<iostream>
const int MAX_EVENT_NUMBER = 10000; //最大事件数
// 设置句柄非阻塞
int setnonblocking(int fd) {
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}

int main(){

    // 创建套接字
    int nRet=0;
    int m_listenfd = socket(PF_INET, SOCK_STREAM, 0);
    if(m_listenfd < 0) {
        printf("fail to socket!");
        return -1;
    }
    
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    address.sin_addr.s_addr = htonl(INADDR_ANY);
    address.sin_port = htons(6666);

    int flag = 1;
    // 设置ip可重用
    setsockopt(m_listenfd, SOL_SOCKET, SO_REUSEADDR, &flag, sizeof(flag));
    // 绑定端口号
    int ret = bind(m_listenfd,(struct sockaddr *)&address,sizeof(address));
    if(ret < 0) {
        printf("fail to bind!,errno :%d",errno);
        return ret;
    }

    // 监听连接fd
    ret = listen(m_listenfd, 200);
    if(ret < 0) {
        printf("fail to listen!,errno :%d",errno);
        return ret;
    }

    // 初始化红黑树和事件链表结构rdlist结构
    epoll_event events[MAX_EVENT_NUMBER];
    int m_epollfd = epoll_create(5);
    if(m_epollfd == -1) {
        printf("fail to epoll create!");
        return m_epollfd;
    }

    // 创建节点结构体将监听连接句柄
    epoll_event event;
    event.data.fd = m_listenfd;
    // 设置该句柄为边缘触发：数据没处理完后续不会再触发事件
    // 水平触发：不管数据有没有触发都返回事件
    event.events = EPOLLIN | EPOLLET | EPOLLRDHUP;
    // 添加监听连接句柄作为初始节点进入红黑树结构中，该节点后续处理连接的句柄
    epoll_ctl(m_epollfd, EPOLL_CTL_ADD, m_listenfd, &event);

    //进入服务器循环
    while(1) {
        int number = epoll_wait(m_epollfd, events, MAX_EVENT_NUMBER, -1);
        if (number < 0 && errno != EINTR) {
            printf( "epoll failure");
            break;
        }
        for (int i = 0; i < number; i++) {
            int sockfd = events[i].data.fd;
            // 属于处理新到的客户连接
            if (sockfd == m_listenfd) {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof(client_address);
                int connfd = accept(m_listenfd, 
	                (struct sockaddr *)&client_address, &client_addrlength);
                if (connfd < 0) {
                    printf("errno is:%d accept error", errno);
                    return false;
                }
                epoll_event event;
                event.data.fd = connfd;
                //设置该句柄为边缘触发
                event.events = EPOLLIN | EPOLLRDHUP;
                // 添加监听连接句柄作为初始节点进入红黑树结构中
                // 该节点后续处理连接的句柄
                epoll_ctl(m_epollfd, EPOLL_CTL_ADD, connfd, &event);
                setnonblocking(connfd);
            }
            else if(events[i].events & (EPOLLRDHUP | EPOLLHUP | EPOLLERR)){
                // 服务器端关闭连接，
                epoll_ctl(m_epollfd, EPOLL_CTL_DEL, sockfd, 0);
                close(sockfd);
            }
            // 处理客户连接上接收到的数据
            else if (events[i].events & EPOLLIN) {
                char buf[1024]={0};
                read(sockfd,buf,1024);
                printf("from client :%s");

                // 将事件设置为写事件返回数据给客户端
                events[i].data.fd = sockfd;
                events[i].events = EPOLLOUT | EPOLLET | 
					                EPOLLONESHOT | EPOLLRDHUP;
                epoll_ctl(m_epollfd, EPOLL_CTL_MOD, sockfd, &events[i]);
            }
            else if (events[i].events & EPOLLOUT) {
                std::string response = "server response \n";
                write(sockfd,response.c_str(),response.length());

                // 将事件设置为读事件，继续监听客户端
                events[i].data.fd = sockfd;
                events[i].events = EPOLLIN | EPOLLRDHUP;
                epoll_ctl(m_epollfd, EPOLL_CTL_MOD, sockfd, &events[i]);
            }
            //else if 可以加管道，unix套接字等等数据
        }
    }


}
```
![[346e30f4-9119-11eb-bb4a-4a238cf0c417.mp4]]

## 4 总结

`select`，`poll`，`epoll`都是IO多路复用机制，即可以监视多个描述符，一旦某个描述符就绪（读或写就绪），能够通知程序进行相应读写操作。 但`select`，`poll`，`epoll`本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

`select`，`poll`实现需要自己不断轮询所有`fd`集合，直到设备就绪，期间可能要睡眠和唤醒多次交替。而`epoll`其实也需要调用`epoll_wait`不断轮询就绪链表，期间也可能多次睡眠和唤醒交替，但是它是设备就绪时，调用回调函数，把就绪fd放入就绪链表中，并唤醒在`epoll_wait`中进入睡眠的进程。虽然都要睡眠和交替，但是`select`和`poll`在醒着的时候要遍历整个`fd`集合，而`epoll`在醒着的时候只要判断一下就绪链表是否为空就行了，这节省了大量的CPU时间。这就是回调机制带来的性能提升。  
    
`select`，`poll`每次调用都要把`fd`集合从用户态往内核态拷贝一次，并且要把`current`往设备等待队列中挂一次，而`epoll`只要一次拷贝，而且把`current`往等待队列上挂也只挂一次（在`epoll_wait`的开始，注意这里的等待队列并不是设备等待队列，只是一个`epoll`内部定义的等待队列）。这也能节省不少的开销。  

|              | select               | poll                 | epoll                     |
| ------------ | -------------------- | -------------------- | ------------------------- |
| 性能         | 连接数增加，性能下降 | 连接数增加，性能下降 | 连接数增加，性能没有变化  | 
| 连接数       | 一般1024             | 无限制               | 无限制                    |
| 内存拷贝     | 每次调用select拷贝   | 每次调用poll拷贝     | fd首次调用`epoll_ctl`拷贝 |
| 数据结构     | bitmap               | 数组                 | 红黑树                    |
| 内在处理机制 | 线性轮询             | 线性轮询             | FD挂在红黑树，事件回调    |
| 时间复杂度   | O(n)                 | O(n)                 | O(1)                      |


