## chan的数据结构
`chan`的具体结构如下
```go
type hchan struct {
	qcount uint        // 当前队列中剩余元素个数
	dataqsiz uint      // 环形队列长度，即可以存放的元素个数
	buf unsafe.Pointer // 环形队列指针
	elemsize uint16    // 每个元素的大小
	closed uint32      // 标识关闭状态
	elemtype *_type    // 元素类型
	sendx uint         // 队列下标，指示元素写入时存放到队列中的位置
	recvx uint         // 队列下标，指示元素从队列的该位置读出
	recvq waitq        // 等待读消息的goroutine队列
	sendq waitq        // 等待写消息的goroutine队列
	lock mutex         // 互斥锁，chan不允许并发读写
}
```
### 数据队列
`chan`内部实现了一个环形队列作为其数据的缓冲区，长度由创建的时候指定。

### 等待队列
从`channel`中读写数据的时候，如果缓冲区为空(即读不到)或者当前缓冲区满了(即写不了)，当前的`goroutine`会被阻塞，挂起在等待队列中。

## chan的使用
### 创建
创建`channel`的过程实际上是初始化`hchan`结构。其中类型信息和缓冲区长度由`make`语句传入，`buf`的大小则与元素大小和缓冲区长度共同决定。

```go
func makechan(t *chantype, size int) *hchan {
	var c *hchan
	c = new(hchan)
	c.buf = malloc(元素类型大小*size)
	c.elemsize = 元素类型大小
	c.elemtype = 元素类型
	c.dataqsiz = size
	return c
}
```

### 写入
具体流程如下：
1. 如果等待接收队列不为空，从等待接收队列中取出一个线程`G`，数据写入`G`并将其唤醒
2. 如果缓冲区有空余位置，写入缓冲区
3. 如果缓冲区没有空余位置，将待发送的数据写入当前线程`G`，将`G`写入等待发送队列，挂起`G`

### 读取
简单过程如下：
1. 如果等待发送队列不为空且没有缓冲区，从等待发送队列中取出一个线程`G`，读出数据并唤醒`G`
2. 如果等待发送队列不为空即缓冲区已满，从缓冲区首部读数据，发送队列`G`中数据写入尾部，唤醒`G`
3. 如果缓冲区中有数据，则从缓冲区取出数据
4. 如果缓冲区为空将当前`goroutine`加入等待发送队列，进入睡眠，等待被写`goroutine`唤醒

### 关闭
关闭`channel`时会把等待接收队列中的`G`全部唤醒，本该写入`G`的数据位置为`nil`。把等待发送队列中的`G`全部唤醒，但这些`G`会`panic`

除此之外，`panic`出现的常见场景还有：
1. 关闭值为`nil`的`channel`
2. 关闭已经被关闭的`channel`
3. 向已经关闭的`channel`写数据

### 单向channel
通过形参进行使用限制
```go
func readChan(chanName <-chan int) {
	<- chanName
}

func writeChan(chanName chan<- int) {
	chanName <- 1
}

func main() {

	var mychan = make(chan int, 10)
	writeChan(mychan)
	readChan(mychan)
	
}
```
