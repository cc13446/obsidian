## pingpong
实验要求我们利用`fork`系统调用创建两个进程，两个进程之间通过管道进行通信，并分别打印出
- `4: received ping`
- `3: received pong`

实验本身不难，但是值得注意到的是管道是否是**全双工**的

### 第一版实现
```c
#include<kernel/types.h>
#include<user/user.h>
int main(int argc, char* argv[]) {
    int status, p[2];
    pipe(p);
    int f = fork();
    int pid = getpid();

    if (f == 0) {
        // child
        close(p[0]);
        
        // write to p[1]
        write(p[1], "ping", 4);

        // read from p[1]
        char buffer[8];
        read(p[1], buffer, 4);
        printf("%d: received %s\n", pid, buffer);
        
        close(p[1]);
        exit(0);
    } else {
        // parent 
        close(p[1]);

        // read from p[0]
        char buffer[8];
        read(p[0], buffer, 4);
        
        // write to p[0]
        write(p[0], "pong", 4);
        close(p[0]);
        
        wait(&status);
        printf("%d: received %s\n", pid, buffer);
        exit(0);
    }
}
```

但是xv6的管道不是全双工的，只能从`p[0]`读取，从`p[1]`写入，所以这种实现是有问题的。

### 第二版实现
```c
#include<kernel/types.h>
#include<user/user.h>
int main(int argc, char* argv[]) {
    int status, p1[2], p2[2];
    pipe(p1);
    pipe(p2);
    int f = fork();
    int pid = getpid();

    if (f == 0) {
        // child
        close(p1[1]);
        close(p2[0]);
        
        // write to p2[1]
        write(p2[1], "ping", 4);
        close(p2[1]);

        // read from p1[0]
        char buffer[8];
        read(p1[0], buffer, 4);
        printf("%d: received %s\n", pid, buffer);
        close(p1[0]);

        exit(0);
    } else {
        // parent 
        close(p1[0]);
        close(p2[1]);

        // read from p2[0]
        char buffer[8];
        read(p2[0], buffer, 4);
        printf("%d: received %s\n", pid, buffer);
        close(p2[0]);

        // write to p1[1]
        write(p1[1], "pong", 4);
        close(p1[1]);

        wait(&status);
        exit(0);
    }
}
```

### 管道的实现 
管道本质上是由内存管理的一个环形缓冲区，一般4k大小。当管道中没有消息的时候，读管道的进程会阻塞，当管道信息满的时候，写管道的进程会阻塞。

![](Lab_01_utilities/Pasted%20image%2020230416133700.png)

在Linux系统中，管道并没有专门的数据结构来实现，而是借助了文件系统的`file`结构和VFS的索引节点`inode`。将两个`file`结构指向同一个临时的VFS索引节点，这样两个进程就可以像操作文件一样操作管道了。这个临时的`inode`通过内存文件的方式进行实现，当管道都被关闭时，此内存文件被释放。

![](Lab_01_utilities/Pasted%20image%2020230416133731.png)

