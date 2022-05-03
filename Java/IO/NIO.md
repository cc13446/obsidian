转载自`https://pdai.tech/`
转载自`https://zhuanlan.zhihu.com/p/78869158`
# 流与块
- 面向流的 I/O 一次处理一个字节数据
- 面向块的 I/O 一次处理一个数据块

# 通道与缓冲区

## 通道

通道 Channel 是对原 I/O 包中的流的模拟，可以通过它读取和写入数据。

通道与流的不同之处在于，流只能在一个方向上移动，而通道是双向的，可以用于读、写或者同时用于读写。

通道包括以下类型:
-   `FileChannel`: 从文件中读写数据；
-   `DatagramChannel`: 通过 UDP 读写网络中数据；
-   `SocketChannel`: 通过 TCP 读写网络中数据；
-   `ServerSocketChannel`: 可以监听新进来的 TCP 连接;

## 缓冲区

发送给一个通道的所有数据都必须首先放到缓冲区中，同样地，从通道中读取的任何数据都要先读到缓冲区中。也就是说，不会直接对通道进行读写数据，而是要先经过缓冲区。

缓冲区实质上是一个数组，但它不仅仅是一个数组。缓冲区提供了对数据的结构化访问，而且还可以跟踪系统的读/写进程。

缓冲区包括以下类型:
-   `ByteBuffer`
-   `CharBuffer`
-   `ShortBuffer`
-   `IntBuffer`
-   `LongBuffer`
-   `FloatBuffer`
-   `DoubleBuffer`

### 缓冲区状态变量
-   `capacity`: 最大容量；
-   `position`: 当前已经读写的字节数；
-   `limit`: 还可以读写的字节数。


# 文件 NIO 实例

以下展示了使用 NIO 快速复制文件的实例:

```java
public static void fastCopy(String src, String dist) throws IOException {

    /* 获得源文件的输入字节流 */
    FileInputStream fin = new FileInputStream(src);

    /* 获取输入字节流的文件通道 */
    FileChannel fcin = fin.getChannel();

    /* 获取目标文件的输出字节流 */
    FileOutputStream fout = new FileOutputStream(dist);

    /* 获取输出字节流的通道 */
    FileChannel fcout = fout.getChannel();

    /* 为缓冲区分配 1024 个字节 */
    ByteBuffer buffer = ByteBuffer.allocateDirect(1024);

    while (true) {

        /* 从输入通道中读取数据到缓冲区中 */
        int r = fcin.read(buffer);

        /* read() 返回 -1 表示 EOF */
        if (r == -1) {
            break;
        }

        /* 切换读写 */
        buffer.flip();

        /* 把缓冲区的内容写入输出文件中 */
        fcout.write(buffer);
        
        /* 清空缓冲区 */
        buffer.clear();
    }
}
```
  

# 选择器

NIO 常常被叫做非阻塞 IO，主要是因为 NIO 在网络通信中的非阻塞特性被广泛使用。

NIO 实现了 IO 多路复用中的 `Reactor` 模型，一个线程 `Thread` 使用一个选择器 `Selector` 通过轮询的方式去监听多个通道 `Channel` 上的事件，从而让一个线程就可以处理多个事件。

通过配置监听的通道 `Channel` 为非阻塞，那么当 `Channel` 上的 IO 事件还未到达时，就不会进入阻塞状态一直等待，而是继续轮询其它 `Channel`，找到 IO 事件已经到达的 `Channel` 执行。

因为创建和切换线程的开销很大，因此使用一个线程来处理多个事件而不是一个线程处理一个事件具有更好的性能。

## 1. 创建选择器
```java
Selector selector = Selector.open();
```

## 2. 将通道注册到选择器上
```java
ServerSocketChannel ssChannel = ServerSocketChannel.open();
ssChannel.configureBlocking(false);
ssChannel.register(selector, SelectionKey.OP_ACCEPT);
```

通道必须配置为非阻塞模式，否则使用选择器就没有任何意义了，因为如果通道在某个事件上被阻塞，那么服务器就不能响应其它事件，必须等待这个事件处理完毕才能去处理其它事件，显然这和选择器的作用背道而驰。

在将通道注册到选择器上时，还需要指定要注册的具体事件，主要有以下几类:
-   `SelectionKey.OP_CONNECT`
-   `SelectionKey.OP_ACCEPT`
-   `SelectionKey.OP_READ`
-   `SelectionKey.OP_WRITE`

它们在 SelectionKey 的定义如下:

```java
public static final int OP_READ = 1 << 0;
public static final int OP_WRITE = 1 << 2;
public static final int OP_CONNECT = 1 << 3;
public static final int OP_ACCEPT = 1 << 4;
```

可以看出每个事件可以被当成一个位域，从而组成事件集整数。例如:

```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;
```

## 3. 监听事件

```java
int num = selector.select();
```

使用 select() 来监听到达的事件，它会一直阻塞直到有至少一个事件到达。

## 4. 获取到达的事件

```java
Set<SelectionKey> keys = selector.selectedKeys();
Iterator<SelectionKey> keyIterator = keys.iterator();
while (keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if (key.isAcceptable()) {
        // ...
    } else if (key.isReadable()) {
        // ...
    }
    keyIterator.remove();
}
```

## 5. 事件循环

因为一次 `select()` 调用不能处理完所有的事件，并且服务器端有可能需要一直监听事件，因此服务器端处理事件的代码一般会放在一个死循环内。

```java
while (true) {
    int num = selector.select();
    Set<SelectionKey> keys = selector.selectedKeys();
    Iterator<SelectionKey> keyIterator = keys.iterator();
    while (keyIterator.hasNext()) {
        SelectionKey key = keyIterator.next();
        if (key.isAcceptable()) {
            // ...
        } else if (key.isReadable()) {
            // ...
        }
        keyIterator.remove();
    }
}
```

# 套接字 NIO 实例

```java
public class NIOServer {

    public static void main(String[] args) throws IOException {

        Selector selector = Selector.open();

        ServerSocketChannel ssChannel = ServerSocketChannel.open();
        ssChannel.configureBlocking(false);
        ssChannel.register(selector, SelectionKey.OP_ACCEPT);

        ServerSocket serverSocket = ssChannel.socket();
        InetSocketAddress address 
	        = new InetSocketAddress("127.0.0.1", 8888);
        serverSocket.bind(address);

        while (true) {

            selector.select();
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> keyIterator = keys.iterator();

            while (keyIterator.hasNext()) {

                SelectionKey key = keyIterator.next();

                if (key.isAcceptable()) {

                    ServerSocketChannel ssChannel1 
	                    = (ServerSocketChannel) key.channel();

                    // 服务器会为每个新连接创建一个 SocketChannel
                    SocketChannel sChannel = ssChannel1.accept();
                    sChannel.configureBlocking(false);

                    // 这个新连接主要用于从客户端读取数据
                    sChannel.register(selector, SelectionKey.OP_READ);

                } else if (key.isReadable()) {

	                SocketChannel sChannel = (SocketChannel) key.channel();
	                System.out.println(
			            readDataFromSocketChannel(sChannel));
                    sChannel.close();
                }
                
                keyIterator.remove();
            }
        }
    }

    private static String readDataFromSocketChannel(SocketChannel sChannel)
					throws IOException {

        ByteBuffer buffer = ByteBuffer.allocate(1024);
        StringBuilder data = new StringBuilder();

        while (true) {

            buffer.clear();
            int n = sChannel.read(buffer);
            if (n == -1) {
                break;
            }
            buffer.flip();
            int limit = buffer.limit();
            char[] dst = new char[limit];
            for (int i = 0; i < limit; i++) {
                dst[i] = (char) buffer.get(i);
            }
            data.append(dst);
            buffer.clear();
        }
        return data.toString();
    }
}
```

```java
public class NIOClient {

    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("127.0.0.1", 8888);
        OutputStream out = socket.getOutputStream();
        String s = "hello world";
        out.write(s.getBytes());
        out.close();
    }
}
```


# NIO 零拷贝实现
`Java NIO` 中的通道(Channel)就相当于操作系统的内核空间(kernel space)的缓冲区，而缓冲区(Buffer)对应的相当于操作系统的用户空间(user space)中的用户缓冲区(user buffer)。
-   `Channel`是全双工的，它既可能是读缓冲区，也可能是网络缓冲区。
-   `Buffer`分为堆内存和堆外内存，这是通过 `malloc()` 分配出来的用户态内存。
 
堆外内存`DirectBuffer`在使用后需要应用程序手动回收，而堆内存`HeapBuffer`的数据在 GC 时可能会被自动回收。因此，在使用 `HeapBuffer` 读写数据时，为了避免缓冲区数据因为 GC 而丢失，`NIO` 会先把 `HeapBuffer` 内部的数据拷贝到一个临时的 `DirectBuffer` 中的本地内存`native memory`，这个拷贝涉及到 `sun.misc.Unsafe.copyMemory()` 的调用，背后的实现原理与 `memcpy()` 类似。 最后，将临时生成的 `DirectBuffer` 内部的数据的内存地址传给 I/O 调用函数，这样就避免了再去访问 Java 对象处理 I/O 读写。

## 内存映射文件
### MappedByteBuffer
`MappedByteBuffer` 是 NIO 基于内存映射方式的提供的一种实现，继承自 `ByteBuffer`。`FileChannel` 定义了一个 `map()` 方法，它可以把一个文件从 `position` 位置开始的 `size` 大小的区域映射为内存映像文件。

`FileChannel.map()`方法其实就是采用了操作系统中的内存映射方式，底层就是调用`Linux mmap()`实现的。

抽象方法 `map()` 方法在 `FileChannel` 中的定义如下：
```java
public abstract MappedByteBuffer map(MapMode mode, long position, 
	long size) throws IOException;
```
-   `mode`：限定内存映射区域对内存映像文件的访问模式
	- 只可读`READ_ONLY`
	- 可读可写`READ_WRITE`
	- 写时拷贝`PRIVATE`
-   `position`：文件映射的起始地址，对应内存映射区域的首地址。
-   `size`：文件映射的字节长度，从 `position` 往后的字节数，对应内存映射区域的大小

`MappedByteBuffer` 相比 `ByteBuffer` 新增了三个重要的方法：

-   `fore()`：对于处于 `READ_WRITE` 模式下的缓冲区，把对缓冲区内容的修改强制刷新到本地文件
-   `load()`：将缓冲区的内容载入物理内存中，并返回这个缓冲区的引用。
-   `isLoaded()`：如果缓冲区的内容在物理内存中，则返回 true，否则返回 false。

#### 使用示例：

```java
private final static String CONTENT = "Zero copy implemented by MappedByteBuffer";
private final static String FILE_NAME = "/mmap.txt";
private final static String CHARSET = "UTF-8";
```
##### 写文件数据
打开文件通道 `fileChannel` 并提供读权限、写权限和数据清空权限，通过 `fileChannel` 映射到一个可写的内存缓冲区 `mappedByteBuffer`，将目标数据写入 `mappedByteBuffer`，通过 `force()` 方法把缓冲区更改的内容强制写入本地文件。

```java
@Test
public void writeToFileByMappedByteBuffer() {
    Path path = Paths.get(getClass().getResource(FILE_NAME).getPath());
    byte[] bytes = CONTENT.getBytes(Charset.forName(CHARSET));
    // 打开文件通道
    try (FileChannel fileChannel 
		    = FileChannel.open(path, StandardOpenOption.READ,
	        StandardOpenOption.WRITE, 
	        StandardOpenOption.TRUNCATE_EXISTING)) {
		// 映射到一个可写的内存缓冲区
        MappedByteBuffer mappedByteBuffer 
	        = fileChannel.map(READ_WRITE, 0, bytes.length);
        if (mappedByteBuffer != null) {
            mappedByteBuffer.put(bytes);
            // 把缓冲区更改的内容强制写入本地文件
            mappedByteBuffer.force();
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

##### 读文件数据
打开文件通道 `fileChannel` 并提供只读权限，通过 `fileChannel` 映射到一个只可读的内存缓冲区 `mappedByteBuffer`，读取 `mappedByteBuffer` 中的字节数组即可得到文件数据。

```java
@Test
public void readFromFileByMappedByteBuffer() {
    Path path = Paths.get(getClass().getResource(FILE_NAME).getPath());
    int length = CONTENT.getBytes(Charset.forName(CHARSET)).length;
    try (FileChannel fileChannel 
	    = FileChannel.open(path, StandardOpenOption.READ)) {
        MappedByteBuffer mappedByteBuffer 
	        = fileChannel.map(READ_ONLY, 0, length);
        if (mappedByteBuffer != null) {
            byte[] bytes = new byte[length];
            mappedByteBuffer.get(bytes);
            String content = new String(bytes, StandardCharsets.UTF_8);
            assertEquals(content, "Zero copy implemented by MappedByteBuffer");
        }
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```
#### 特点和不足
1. `MappedByteBuffer` 使用是堆外的虚拟内存，因此分配的内存大小不受 JVM 的 -Xmx 参数限制，但是也是有大小限制的。 如果当文件超出 `Integer.MAX_VALUE` 字节限制时，可以通过 `position` 参数重新 `map` 文件后面的内容。
2. `MappedByteBuffer` 在处理大文件时性能的确很高，但也存内存占用、文件关闭不确定等问题，被其打开的文件只有在垃圾回收的才会被关闭，而且这个时间点是不确定的。
3. `MappedByteBuffer` 提供了文件映射内存的 `mmap()` 方法，也提供了释放映射内存的 `unmap()` 方法。然而 `unmap()` 是 `FileChannelImpl` 中的私有方法，无法直接显示调用。因此，用户程序需要通过 Java 反射的调用 `sun.misc.Cleaner` 类的 clean() 方法手动释放映射占用的内存区域。

### FileChannel

 `transferTo()`：通过 `FileChannel` 把文件的源数据写入一个 `WritableByteChannel` 的目的通道。

```java
public abstract long transferTo(long position, long count, 
								WritableByteChannel target)
							    throws IOException;
```

`transferFrom()`：把一个源通道 `ReadableByteChannel` 中的数据读取到当前 `FileChannel` 的文件

```java
public abstract long transferFrom(ReadableByteChannel src, 
								  long position, long count)
							      throws IOException;
```

`FileChannel.transferTo()`方法直接将当前通道内容传输到另一个通道，没有涉及到`Buffer`的任何操作，`transferTo()`的实现方式就是通过系统调用`sendfile()` (当然这是Linux中的系统调用)



```java
//使用sendfile:读取磁盘文件，并网络发送
FileChannel sourceChannel 
	= new RandomAccessFile(source, "rw").getChannel();
SocketChannel socketChannel = SocketChannel.open(sa);
sourceChannel.transferTo(0, sourceChannel.size(), socketChannel);
```

`ZeroCopyFile`实现文件复制
```java
class ZeroCopyFile {

    public void copyFile(File src, File dest) {
        try (FileChannel srcChannel 
		        = new FileInputStream(src).getChannel();
			FileChannel destChannel 
	             = new FileInputStream(dest).getChannel()) {
            srcChannel.transferTo(0, srcChannel.size(), destChannel);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
`Java NIO`提供的`FileChannel.transferTo` 和 `transferFrom` 并不保证一定能使用零拷贝。实际上是否能使用零拷贝与操作系统相关，如果操作系统提供 `sendfile` 这样的零拷贝系统调用，则这两个方法会通过这样的系统调用充分利用零拷贝的优势，否则并不能通过这两个方法本身实现零拷贝。