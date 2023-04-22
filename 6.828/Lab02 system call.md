实验不难，只需要理清xv6系统调用的过程即可。

`user.h`中定义系统调用的函数
```c
// system call
// ...
int sleep(int);
int uptime(void);
int trace(int);
int sysinfo(struct sysinfo *);
```

`usys.pl`中定义汇编入口
```perl
sub entry {
    my $name = shift;
    print ".global $name\n";
    print "${name}:\n";
    print " li a7, SYS_${name}\n";
    print " ecall\n";
    print " ret\n";
}

# other entry
# ...
entry("sleep");
entry("uptime");
entry("trace");
entry("sysinfo");
```

`usys.pl`会用perl生成汇编代码
```
# 大概逻辑为
# 定义一个函数，与user.h中一致
# a7寄存器中存入一个整数，此整数定义在syscall.h中
# ecall进入中断

sleep:
 li a7, SYS_sleep
 ecall
 ret
.global uptime
uptime:
 li a7, SYS_uptime
 ecall
 ret
.global trace
trace:
 li a7, SYS_trace
 ecall
 ret
.global sysinfo
sysinfo:
 li a7, SYS_sysinfo
 ecall
 ret
```

中断会进入`trampoline.S`，进行上下文的切换，以及将用户空间的传参映射到内核空间中

然后会调用`trap.c/usertrap`，后续会执行`syscall.c/syscall`
```c
struct syscall
{
  char* name;
  syscall_t func;
};

static struct syscall syscalls[] = {
[SYS_fork]    {"fork", sys_fork},
[SYS_exit]    {"exit", sys_exit},
[SYS_wait]    {"wait", sys_wait},
[SYS_pipe]    {"pipe", sys_pipe},
[SYS_read]    {"read", sys_read},
[SYS_kill]    {"kill", sys_kill},
[SYS_exec]    {"exec", sys_exec},
[SYS_fstat]   {"fstat", sys_fstat},
[SYS_chdir]   {"chdir", sys_chdir},
[SYS_dup]     {"dup", sys_dup},
[SYS_getpid]  {"getpid", sys_getpid},
[SYS_sbrk]    {"sbrk", sys_sbrk},
[SYS_sleep]   {"sleep", sys_sleep},
[SYS_uptime]  {"uptime", sys_uptime},
[SYS_open]    {"open", sys_open},
[SYS_write]   {"write", sys_write},
[SYS_mknod]   {"mknod", sys_mknod},
[SYS_unlink]  {"unlink", sys_unlink},
[SYS_link]    {"link", sys_link},
[SYS_mkdir]   {"mkdir", sys_mkdir},
[SYS_close]   {"close", sys_close},
[SYS_trace]   {"trace", sys_trace},
[SYS_sysinfo] {"sysinfo", sys_sysinfo},
};

void
syscall(void)
{
  int num;
  struct proc *p = myproc();

  num = p->trapframe->a7;
  if(num > 0 && num < NELEM(syscalls) && syscalls[num].func) {
    p->trapframe->a0 = syscalls[num].func();
    // print trace info
    if(p->trace & (1 << num)) {    
	    printf("%d: syscall %s -> %d\n", 
				p->pid, syscalls[num].name, p->trapframe->a0);
	}
  } else {
    printf("%d %s: unknown sys call %d\n",
            p->pid, p->name, num);
    p->trapframe->a0 = -1;
  }
}
```

这样就完成了系统调用。

如果返回值为整数的话可以简单存入`a0`寄存器返回，如果需要将内核内存中的信息复制到用户空间中进程的内存中得话，则需要`copyout`函数，大概原理是：
- 利用进程的页表对进程的虚拟内存和物理内存做好映射
- 然后将内存信息复制到物理内存中