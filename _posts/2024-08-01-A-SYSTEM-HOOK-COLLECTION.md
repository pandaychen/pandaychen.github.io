---
layout:     post
title:      Linux 系统调用追踪
subtitle:   收集常用的 ebpf 的系统调用：review && hook
date:       2024-08-01
author:     pandaychen
catalog:    true
tags:
    - eBPF
    - Golang
    - 系统编程
---

##  0x00    前言
本文汇总下开发过程中收集的一些 Linux 系统调用及用法

##  0x01    常用系统调用分类

####    进程控制

-   `fork`：创建一个新进程
-   `clone`：按指定条件创建子进程
-   `execve`：运行可执行文件
-   `exit`：中止进程
-   `_exit`：立即中止当前进程
-   `getdtablesize`：进程所能打开的最大文件数
-   `getpgid`：获取指定进程组标识号
-   `setpgid`：设置指定进程组标志号
-   `getpgrp`：获取当前进程组标识号
-   `setpgrp`：设置当前进程组标志号
-   `getpid`：获取进程标识号
-   `getppid`：获取父进程标识号
-   `getpriority`：获取调度优先级
-   `setpriority`：设置调度优先级
-   `modify_ldt`：读写进程的本地描述表
-   `nanosleep`：使进程睡眠指定的时间
-   `nice`：改变分时进程的优先级
-   `pause`：挂起进程，等待信号
-   `personality`：设置进程运行域
-   `prctl`：对进程进行特定操作
-   `ptrace`：进程跟踪
-   `sched_get_priority_max`：取得静态优先级的上限
-   `sched_get_priority_min`：取得静态优先级的下限
-   `sched_getparam`：取得进程的调度参数
-   `sched_getscheduler`：取得指定进程的调度策略
-   `sched_rr_get_interval`：取得按 RR 算法调度的实时进程的时间片长度
-   `sched_setparam`：设置进程的调度参数
-   `sched_setscheduler`：设置指定进程的调度策略和参数
-   `sched_yield`：进程主动让出处理器， 并将自己等候调度队列队尾
-   `vfork`：创建一个子进程，以供执行新程序，常与 `execve` 等同时使用
-   `wait`：等待子进程终止
-   `wait3`：参见 `wait`
-   `waitpid`：等待指定子进程终止
-   `wait4`：参见 `waitpid`
-   `capget`：获取进程权限
-   `capset`：设置进程权限
-   `getsid`：获取会晤标识号
-   `setsid`：设置会晤标识号

####    文件系统控制

1、文件读写操作

-   `fcntl`：文件控制
-   `open`：打开文件
-   `creat`：创建新文件
-   `close`：关闭文件描述字
-   `read`：读文件
-   `write`：写文件
-   `readv`：从文件读入数据到缓冲数组中
-   `writev`：将缓冲数组里的数据写入文件
-   `pread`：对文件随机读
-   `pwrite`：对文件随机写
-   `lseek`：移动文件指针
-   `_llseek`：在 `64` 位地址空间里移动文件指针
-   `dup`：复制已打开的文件描述字
-   `dup2`：按指定条件复制文件描述字
-   `flock`：文件加 / 解锁
-   `poll`：I/O 多路转换
-   `truncate`：截断文件
-   `ftruncate`：参见 `truncate`
-   `umask`：设置文件权限掩码
-   `fsync`：把文件在内存中的部分写回磁盘

2、文件系统操作

-   `access`：确定文件的可存取性
-   `chdir`：改变当前工作目录
-   `fchdir`：参见 `chdir`
-   `chmod`：改变文件方式
-   `fchmod`：参见 `chmod`
-   `chown`：改变文件的属主或用户组
-   `fchown`：参见 `chown`
-   `lchown`：参见 `chown`
-   `chroot`：改变根目录
-   `stat`：取文件状态信息
-   `lstat`：参见 `stat`
-   `fstat`：参见 `stat`
-   `statfs`：取文件系统信息
-   `fstatfs`：参见 `statfs`
-   `readdir`：读取目录项
-   `getdents`：读取目录项
-   `mkdir`：创建目录
-   `mknod`：创建索引节点
-   `rmdir`：删除目录
-   `rename`：文件改名
-   `link`：创建链接
-   `symlink`：创建符号链接
-   `unlink`：删除链接
-   `readlink`：读符号链接的值
-   `mount`：安装文件系统
-   `umount`：卸下文件系统
-   `ustat`：取文件系统信息
-   `utime`：改变文件的访问修改时间
-   `utimes`：参见 `utime`
-   `quotactl`：控制磁盘配额

####    系统控制

-   `ioctl`：I/O 总控制函数
-   `_sysctl`：读 / 写系统参数
-   `acct`：启用或禁止进程记账
-   `getrlimit`：获取系统资源上限
-   `setrlimit`：设置系统资源上限
-   `getrusage`：获取系统资源使用情况
-   `uselib`：选择要使用的二进制函数库
-   `ioperm`：设置端口 I/O 权限
-   `iopl`：改变进程 I/O 权限级别
-   `outb`：低级端口操作
-   `reboot`：重新启动
-   `swapon`：打开交换文件和设备
-   `swapoff`：关闭交换文件和设备
-   `bdflush`：控制 `bdflush` 守护进程
-   `sysfs`：取核心支持的文件系统类型
-   `sysinfo`： 取得系统信息
-   `adjtimex`： 调整系统时钟
-   `alarm`： 设置进程的闹钟
-   `getitimer`： 获取计时器值
-   `setitimer`： 设置计时器值
-   `gettimeofday`： 取时间和时区
-   `settimeofday`： 设置时间和时区
-   `stime`： 设置系统日期和时间
-   `time`： 取得系统时间
-   `times`： 取进程运行时间
-   `uname`： 获取当前 UNIX 系统的名称、版本和主机等信息
-   `vhangup`： 挂起当前终端
-   `nfsservctl`： 对 NFS 守护进程进行控制
-   `vm86`： 进入模拟 8086 模式
-   `create_module`： 创建可装载的模块项
-   `delete_module`： 删除可装载的模块项
-   `init_module`： 初始化模块
-   `query_module`： 查询模块信息
-   `get_kernel_syms`： 取得核心符号， 已被 `query_module` 代替

####    内存管理

-   `brk`： 改变数据段空间的分配
-   `sbrk`： 参见 `brk`
-   `mlock`： 内存页面加锁
-   `munlock`： 内存页面解锁
-   `mlockall`： 调用进程所有内存页面加锁
-   `munlockall`： 调用进程所有内存页面解锁
-   `mmap`： 映射虚拟内存页
-   `munmap`： 去除内存页映射
-   `mremap`： 重新映射虚拟内存地址
-   `msync`： 将映射内存中的数据写回磁盘
-   `mprotect`： 设置内存映像保护
-   `getpagesize`： 获取页面大小
-   `sync`： 将内存缓冲区数据写回硬盘
-   `cacheflush`： 将指定缓冲区中的内容写回磁盘

####    网络管理

-   `getdomainname`： 取域名
-   `setdomainname`： 设置域名
-   `gethostid`： 获取主机标识号
-   `sethostid`： 设置主机标识号
-   `gethostname`： 获取本主机名称
-   `sethostname`： 设置主机名称

####    socket 控制

-   `socketcall`： socket 系统调用
-   `socket`： 建立 socket
-   `bind`： 绑定 socket 到端口
-   `connect`： 连接远程主机
-   `accept`： 响应 socket 连接请求
-   `send`： 通过 socket 发送信息
-   `sendto`： 发送 UDP 信息
-   `sendmsg`： 参见 `send`
-   `recv`： 通过 socket 接收信息
-   `recvfrom`： 接收 UDP 信息
-   `recvmsg`： 参见 `recv`
-   `listen`： 监听 socket 端口
-   `select`： 对多路同步 I/O 进行轮询
-   `shutdown`： 关闭 socket 上的连接
-   `getsockname`： 取得本地 socket 名字
-   `getpeername`： 获取通信对方的 socket 名字
-   `getsockopt`： 取端口设置
-   `setsockopt`： 设置端口参数
-   `sendfile`： 在文件或端口间传输数据
-   `socketpair`： 创建一对已联接的无名 socket

####    用户管理

-   `getuid`： 获取用户标识号
-   `setuid`： 设置用户标志号
-   `getgid`： 获取组标识号
-   `setgid`： 设置组标志号
-   `getegid`： 获取有效组标识号
-   `setegid`： 设置有效组标识号
-   `geteuid`： 获取有效用户标识号
-   `seteuid`： 设置有效用户标识号
-   `setregid`： 分别设置真实和有效的的组标识号
-   `setreuid`： 分别设置真实和有效的用户标识号
-   `getresgid`： 分别获取真实的， 有效的和保存过的组标识号
-   `setresgid`： 分别设置真实的， 有效的和保存过的组标识号
-   `getresuid`： 分别获取真实的， 有效的和保存过的用户标识号
-   `setresuid`： 分别设置真实的， 有效的和保存过的用户标识号
-   `setfsgid`： 设置文件系统检查时使用的组标识号
-   `setfsuid`： 设置文件系统检查时使用的用户标识号
-   `getgroups`： 获取后补组标志清单
-   `setgroups`： 设置后补组标志清单

####    进程间通信

-   `ipc`： 进程间通信总控制调用

1、信号

-   `sigaction`： 设置对指定信号的处理方法
-   `sigprocmask`： 根据参数对信号集中的信号执行阻塞 / 解除阻塞等操作
-   `sigpending`： 为指定的被阻塞信号设置队列
-   `sigsuspend`： 挂起进程等待特定信号
-   `signal`： 参见 signal
-   `kill`： 向进程或进程组发信号
-   `*sigblock`： 向被阻塞信号掩码中添加信号， 已被 `sigprocmask` 代替
-   `*siggetmask`： 取得现有阻塞信号掩码， 已被 `sigprocmask` 代替
-   `*sigsetmask`： 用给定信号掩码替换现有阻塞信号掩码， 已被 `sigprocmask` 代替
-   `*sigmask`： 将给定的信号转化为掩码， 已被 `sigprocmask` 代替
-   `*sigpause`： 作用同 `sigsuspend`， 已被 `sigsuspend` 代替
-   `sigvec`： 为兼容 BSD 而设的信号处理函数， 作用类似 `sigaction`
-   `ssetmask`： ANSI C 的信号处理函数， 作用类似 `sigaction`

2、消息

-   `msgctl`： 消息控制操作
-   `msgget`： 获取消息队列
-   `msgsnd`： 发消息
-   `msgrcv`： 取消息

3、管道

-   `pipe`： 创建管道

4、信号量

-   `semctl`： 信号量控制
-   `semget`： 获取一组信号量
-   `semop`： 信号量操作

5、共享内存

-   `shmctl`：控制共享内存
-   `shmget`：获取共享内存
-   `shmat`：连接共享内存
-   `shmdt`：拆卸共享内存

##  0x02    系统调用关联的hook备份

####    fork

-   `tracepoint/sched/sched_process_fork`：进程创建事件
-   `tracepoint/sched/sched_process_exit`：进程退出事件
-   `tracepoint/sched/sched_process_free`：进程释放事件

####    execve*
-   `tp/syscalls/sys_execve`
-   `tp/syscalls/sys_exit_execve`
-   `tp/syscalls/sys_execveat`
-   `tp/syscalls/sys_exit_execveat`
-   

####    bash
-   `uretprobe/bash_readline`
-   `uretprobe/bash_retval`：`bash`返回值

####    fd
-   `tp/syscalls/sys_close`
-   `tp/syscalls/sys_exit_close`   
-   `tp/syscalls/sys_enter_creat`
-   `tp/syscalls/sys_exit_creat`
-    `tp/syscalls/sys_enter_open`
-   `tp/syscalls/sys_exit_open`
-   `tp/syscalls/sys_enter_openat`
-   `tp/syscalls/sys_exit_openat`
-   `tp/syscalls/sys_enter_openat2`
-   `tp/syscalls/sys_exit_openat2`

####    connect

-   `kprobe/tcp_v4_connect`
-   `kretprobe/tcp_v4_connect`
-   `kprobe/tcp_v6_connect`
-   `kretprobe/tcp_v6_connect`

####    socket
-   `tp/syscalls/sys_enter_getpeername`
-   `tp/syscalls/sys_exit_getpeername`
-   `tracepoint/syscalls/sys_enter_getsockname`
-   `tracepoint/syscalls/sys_exit_getsockname`

####    pipe
-   `tracepoint/syscalls/sys_enter_dup`
-   `tracepoint/syscalls/sys_enter_dup2`
-   `tracepoint/syscalls/sys_enter_dup3`
-   `tracepoint/syscalls/sys_eixt_dup`
-   `tracepoint/syscalls/sys_exit_dup2`
-   `tracepoint/syscalls/sys_exit_dup3`
-   `tracepoint:syscalls:sys_enter_pipe`
-   `tracepoint:syscalls:sys_enter_pipe2`
-   `tracepoint:syscalls:sys_exit_pipe`
-   `tracepoint:syscalls:sys_exit_pipe2`

####    安全对抗
-   `tp/syscalls/sys_enter_getdents64`
-   `tp/syscalls/sys_exit_getdents64`
-   `tp/syscalls/sys_enter_getdents`
-   `tp/syscalls/sys_exit_getdents`
-   `tracepoint:syscalls:sys_enter_memfd_create`
-   `tracepoint:syscalls:sys_exit_memfd_create`

####    可观测（tracing）

-   `tracepoint/syscalls/sys_enter_connect`
-   `tracepoint/syscalls/sys_exit_connect`
-   `tracepoint/syscalls/sys_enter_accept`
-   `tracepoint/syscalls/sys_exit_accept`
-   `tracepoint/syscalls/sys_enter_accept4`
-   `tracepoint/syscalls/sys_exit_accept4`
-   `tracepoint/syscalls/sys_enter_close`
-   `tracepoint/syscalls/sys_exit_close`
-   `tracepoint/syscalls/sys_enter_write`
-   `tracepoint/syscalls/sys_exit_write`
-   `tracepoint/syscalls/sys_enter_read`
-   `tracepoint/syscalls/sys_exit_read` 
-   `tracepoint/syscalls/sys_enter_sendmsg`
-   `tracepoint/syscalls/sys_exit_sendmsg`
-   `tracepoint/syscalls/sys_enter_recvmsg`
-   `tracepoint/syscalls/sys_exit_recvmsg`

####    数据包（协议栈流向）可观测

-   `kprobe/__skb_datagram_iter`
-   `kprobe/skb_copy_datagram_iovec`
-   `tracepoint/skb/skb_copy_datagram_iovec`
-   `tracepoint/net/netif_receive_skb`
-   `kprobe/tcp_queue_rcv`
-   `kprobe/tcp_rcv_established`
-   `kprobe/tcp_v4_do_rcv`
-   `kprobe/tcp_v6_do_rcv`
-   `kprobe/tcp_v4_rcv`
-   `kprobe/ip_rcv_core`
-   `kprobe/dev_hard_start_xmit`
-   `kprobe/dev_queue_xmit`
-   `kprobe/__ip_queue_xmit`
-   `kprobe/__ip_queue_xmit`
-   `kprobe/nf_nat_packet`
-   `kprobe/nf_nat_manip_pkt`
-   `kprobe/security_socket_sendmsg`
-   `kprobe/security_socket_recvmsg`
-   `tracepoint/syscalls/sys_enter_recvfrom`
-   `tracepoint/syscalls/sys_exit_recvfrom`
-   `tracepoint/syscalls/sys_enter_read`
-   `tracepoint/syscalls/sys_exit_read`
-   `tracepoint/syscalls/sys_enter_recvmsg`
-   `tracepoint/syscalls/sys_exit_recvmsg`
-   `tracepoint/syscalls/sys_enter_readv`
-   `tracepoint/syscalls/sys_exit_readv`
-   `tracepoint/syscalls/sys_enter_sendfile64`
-   `tracepoint/syscalls/sys_exit_sendfile64`
-   `tracepoint/syscalls/sys_enter_sendto`
-   `tracepoint/syscalls/sys_exit_sendto`
-   `tracepoint/syscalls/sys_enter_write`
-   `tracepoint/syscalls/sys_exit_write`
-   `tracepoint/syscalls/sys_enter_sendmsg`
-   `tracepoint/syscalls/sys_exit_sendmsg`
-   `tracepoint/syscalls/sys_enter_writev`
-   `tracepoint/syscalls/sys_exit_writev`
-   `tracepoint/syscalls/sys_enter_close`
-   `kprobe/sys_close`
-   `tracepoint/syscalls/sys_exit_close`
-   `tracepoint/syscalls/sys_enter_connect`
-   `tracepoint/syscalls/sys_exit_connect`
-   `tracepoint/syscalls/sys_enter_accept4`
-   `tracepoint/syscalls/sys_exit_accept4`

####    tcp/ip stack
-   `tracepoint:sock:inet_sock_set_state`：TCP 状态发生变化的时候被调用

####    VFS相关
-   `sys_enter_write`
-   `sys_enter_renameat2`
-   `sys_enter_rename`
-   `sys_enter_copy_file_range`
-   `sys_enter_unlinkat`

##  0x03    追踪系统调用

####    strace
直接使用`strace echo "abcdefg" >> test.log`跟踪系统调用日志，会发现输出里面没有 `openat`或 `open`系统调用来打开`test.log`文件，原因在于 shell 重定向（`>>`）是由 shell 本身处理的，而不是由被执行的命令 (echo) 处理的

`strace echo "abcdefg" >> test.log`的执行过程如下：

1.  shell 解析命令：shell解析命令行，识别出重定向操作符 `>>`
2.  shell 准备重定向：在启动 strace命令之前，Shell 会先自己执行 `open`系统调用来打开文件 `test.log`。如果文件不存在，shell 会创建；如果文件存在，Shell 会以追加模式打开，并将文件指针移动到末尾
3.  shell 将这个新打开的文件描述符（通常是 `fd=1`，也就是标准输出 stdout）保存起来，准备给即将启动的进程使用
4.  shell 执行命令：Shell 准备好启动命令，先 `fork` 创建子进程，然后在子进程中执行 `execve`来启动 strace
5.  strace 开始工作：strace进程启动，继承了 Shell 为其设置好的文件描述符表。这意味着它的标准输出（`fd=1`）已经指向了 `test.log`文件
6.  strace 跟踪目标：接着，strace又会 `fork/execve` 来启动 `echo`程序。`echo`进程继承了 strace的文件描述符表，因此它的标准输出（`fd=1`）同样指向 `test.log`文件
7.  `echo` 执行：`echo`程序运行，将字符串 `abcdefg\n`写入（`write`）到它的标准输出（`fd= 1`），对应strace的跟踪日志`write(1, "abcdefg\n", 9) = 9`；它无需要知道这个 `fd=1` 是终端、管道还是一个文件，因为 `fd=1` 已经被 shell 和 strace设置好了，数据自然就流入了 `test.log`

总结下：`test.log` 这个操作是由登录 shell完成的，`open`系统调用发生在 shell 进程中，strace只跟踪它直接启动的进程（这里是 `echo`）及其子进程的系统调用，它不会跟踪它的父进程（ Shell）的行为，因此在 `strace echo ...`的输出中看不到对 `test.log`的 `open`调用，因为这个调用发生在 strace之外，尝试采用下面的方法：

```BASH
# 启动一个跟踪 shell 的 strace，并让这个 shell 去执行命令
strace -f -o shell_trace.log bash -c 'echo "abcdefg" >> test.log'
```

使用`strace -f -o shell_trace.log bash -c 'echo "abcdefg" >> test.log'`跟踪`echo`写文件的过程

```bash
[root@VM-X-X-tencentos ~]# cat shell_trace.log 
415499 execve("/usr/bin/bash", ["bash", "-c", "echo \"abcdefg\" >> test.log"], 0x7ffc636511b8 /* 33 vars */) = 0
415499 brk(NULL)                        = 0x555deaba9000
415499 arch_prctl(0x3001 /* ARCH_??? */, 0x7ffc982164c0) = -1 EINVAL (Invalid argument)
415499 access("/etc/ld.so.preload", R_OK) = 0
415499 openat(AT_FDCWD, "/etc/ld.so.preload", O_RDONLY|O_CLOEXEC) = 3
415499 fstat(3, {st_mode=S_IFREG|0644, st_size=35, ...}) = 0
415499 mmap(NULL, 35, PROT_READ|PROT_WRITE, MAP_PRIVATE, 3, 0) = 0x7fddcf224000
415499 close(3)                         = 0
415499 readlink("/proc/self/exe", "/usr/bin/bash", 4096) = 13
415499 openat(AT_FDCWD, "/lib64/libtdsp.so", O_RDONLY|O_CLOEXEC) = 3
415499 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\340\16\0\0\0\0\0\0"..., 832) = 832
415499 fstat(3, {st_mode=S_IFREG|0755, st_size=48747, ...}) = 0
415499 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fddcf222000
415499 mmap(NULL, 2109392, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fddcedf3000
415499 mprotect(0x7fddcedf6000, 2093056, PROT_NONE) = 0
415499 mmap(0x7fddceff5000, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7fddceff5000
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/lib64/libonion.so", O_RDONLY|O_CLOEXEC) = 3
415499 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\20\22\0\0\0\0\0\0"..., 832) = 832
415499 fstat(3, {st_mode=S_IFREG|0755, st_size=19656, ...}) = 0
415499 mmap(NULL, 31552, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fddcf21a000
415499 mmap(0x7fddcf21b000, 8192, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1000) = 0x7fddcf21b000
415499 mmap(0x7fddcf21d000, 4096, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7fddcf21d000
415499 mmap(0x7fddcf21e000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3000) = 0x7fddcf21e000
415499 mmap(0x7fddcf220000, 6976, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fddcf220000
415499 close(3)                         = 0
415499 munmap(0x7fddcf224000, 35)       = 0
415499 openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
415499 fstat(3, {st_mode=S_IFREG|0644, st_size=29391, ...}) = 0
415499 mmap(NULL, 29391, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fddcf212000
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/lib64/libtinfo.so.6", O_RDONLY|O_CLOEXEC) = 3
415499 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0000\351\0\0\0\0\0\0"..., 832) = 832
415499 lseek(3, 165144, SEEK_SET)       = 165144
415499 read(3, "\4\0\0\0\20\0\0\0\5\0\0\0GNU\0\2\0\0\300\4\0\0\0\3\0\0\0\0\0\0\0", 32) = 32
415499 fstat(3, {st_mode=S_IFREG|0755, st_size=187552, ...}) = 0
415499 lseek(3, 165144, SEEK_SET)       = 165144
415499 read(3, "\4\0\0\0\20\0\0\0\5\0\0\0GNU\0\2\0\0\300\4\0\0\0\3\0\0\0\0\0\0\0", 32) = 32
415499 mmap(NULL, 2279808, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fddcebc6000
415499 mprotect(0x7fddcebef000, 2093056, PROT_NONE) = 0
415499 mmap(0x7fddcedee000, 20480, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x28000) = 0x7fddcedee000
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/lib64/libdl.so.2", O_RDONLY|O_CLOEXEC) = 3
415499 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0p\16\0\0\0\0\0\0"..., 832) = 832
415499 fstat(3, {st_mode=S_IFREG|0755, st_size=19128, ...}) = 0
415499 mmap(NULL, 2109600, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fddce9c2000
415499 mprotect(0x7fddce9c5000, 2093056, PROT_NONE) = 0
415499 mmap(0x7fddcebc4000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x2000) = 0x7fddcebc4000
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
415499 read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\300\250\3\0\0\0\0\0"..., 832) = 832
415499 fstat(3, {st_mode=S_IFREG|0755, st_size=2164648, ...}) = 0
415499 lseek(3, 808, SEEK_SET)          = 808
415499 read(3, "\4\0\0\0\20\0\0\0\5\0\0\0GNU\0\2\0\0\300\4\0\0\0\3\0\0\0\0\0\0\0", 32) = 32
415499 mmap(NULL, 4020448, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7fddce5ec000
415499 mprotect(0x7fddce7b9000, 2093056, PROT_NONE) = 0
415499 mmap(0x7fddce9b8000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1cc000) = 0x7fddce9b8000
415499 mmap(0x7fddce9be000, 14560, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7fddce9be000
415499 close(3)                         = 0
415499 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7fddcf210000
415499 arch_prctl(ARCH_SET_FS, 0x7fddcf210f00) = 0
415499 mprotect(0x7fddce9b8000, 16384, PROT_READ) = 0
415499 mprotect(0x7fddcebc4000, 4096, PROT_READ) = 0
415499 mprotect(0x7fddcedee000, 16384, PROT_READ) = 0
415499 mprotect(0x7fddcf21e000, 4096, PROT_READ) = 0
415499 mprotect(0x555dea9ad000, 16384, PROT_READ) = 0
415499 mprotect(0x7fddcf225000, 4096, PROT_READ) = 0
415499 munmap(0x7fddcf212000, 29391)    = 0
415499 openat(AT_FDCWD, "/dev/tty", O_RDWR|O_NONBLOCK) = 3
415499 close(3)                         = 0
415499 getrandom("\xb4\x67\xa7\xde\x7a\x5f\x55\x55", 8, GRND_NONBLOCK) = 8
415499 brk(NULL)                        = 0x555deaba9000
415499 brk(0x555deabca000)              = 0x555deabca000
415499 brk(NULL)                        = 0x555deabca000
415499 openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
415499 fstat(3, {st_mode=S_IFREG|0644, st_size=217800224, ...}) = 0
415499 mmap(NULL, 217800224, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7fddc1636000
415499 close(3)                         = 0
415499 getuid()                         = 0
415499 getgid()                         = 0
415499 geteuid()                        = 0
415499 getegid()                        = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 ioctl(-1, TIOCGPGRP, 0x7ffc982163a4) = -1 EBADF (Bad file descriptor)
415499 sysinfo({uptime=2693704, loads=[0, 0, 0], totalram=16500895744, freeram=8356208640, sharedram=294432768, bufferram=420241408, totalswap=0, freeswap=0, procs=280, totalhigh=0, freehigh=0, mem_unit=1}) = 0
415499 rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
415499 rt_sigaction(SIGCHLD, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7fddce63a5b0}, 8) = 0
415499 rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
415499 rt_sigaction(SIGINT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, 8) = 0
415499 rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
415499 rt_sigaction(SIGQUIT, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, 8) = 0
415499 rt_sigaction(SIGTSTP, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
415499 rt_sigaction(SIGTSTP, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, 8) = 0
415499 rt_sigaction(SIGTTIN, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
415499 rt_sigaction(SIGTTIN, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, 8) = 0
415499 rt_sigaction(SIGTTOU, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
415499 rt_sigaction(SIGTTOU, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, 8) = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 rt_sigaction(SIGQUIT, {sa_handler=SIG_IGN, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER, sa_restorer=0x7fddce63a5b0}, 8) = 0
415499 uname({sysname="Linux", nodename="VM-137-151-tencentos", ...}) = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 openat(AT_FDCWD, "/usr/lib64/gconv/gconv-modules.cache", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0644, st_size=26998, ...}) = 0
415499 mmap(NULL, 26998, PROT_READ, MAP_SHARED, 3, 0) = 0x7fddcf213000
415499 close(3)                         = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 stat("/root", {st_mode=S_IFDIR|0550, st_size=4096, ...}) = 0
415499 stat(".", {st_mode=S_IFDIR|0550, st_size=4096, ...}) = 0
415499 stat("/root", {st_mode=S_IFDIR|0550, st_size=4096, ...}) = 0
415499 getpid()                         = 415499
415499 getppid()                        = 415496
415499 stat(".", {st_mode=S_IFDIR|0550, st_size=4096, ...}) = 0
415499 stat("/usr/share/Modules/bin/bash", 0x7ffc98216020) = -1 ENOENT (No such file or directory)
415499 stat("/usr/local/sbin/bash", 0x7ffc98216020) = -1 ENOENT (No such file or directory)
415499 stat("/usr/local/bin/bash", 0x7ffc98216020) = -1 ENOENT (No such file or directory)
415499 stat("/usr/sbin/bash", 0x7ffc98216020) = -1 ENOENT (No such file or directory)
415499 stat("/usr/bin/bash", {st_mode=S_IFREG|0755, st_size=1163048, ...}) = 0
415499 stat("/usr/bin/bash", {st_mode=S_IFREG|0755, st_size=1163048, ...}) = 0
415499 geteuid()                        = 0
415499 getegid()                        = 0
415499 getuid()                         = 0
415499 getgid()                         = 0
415499 access("/usr/bin/bash", X_OK)    = 0
415499 stat("/usr/bin/bash", {st_mode=S_IFREG|0755, st_size=1163048, ...}) = 0
415499 geteuid()                        = 0
415499 getegid()                        = 0
415499 getuid()                         = 0
415499 getgid()                         = 0
415499 access("/usr/bin/bash", R_OK)    = 0
415499 stat("/usr/bin/bash", {st_mode=S_IFREG|0755, st_size=1163048, ...}) = 0
415499 stat("/usr/bin/bash", {st_mode=S_IFREG|0755, st_size=1163048, ...}) = 0
415499 geteuid()                        = 0
415499 getegid()                        = 0
415499 getuid()                         = 0
415499 getgid()                         = 0
415499 access("/usr/bin/bash", X_OK)    = 0
415499 stat("/usr/bin/bash", {st_mode=S_IFREG|0755, st_size=1163048, ...}) = 0
415499 geteuid()                        = 0
415499 getegid()                        = 0
415499 getuid()                         = 0
415499 getgid()                         = 0
415499 access("/usr/bin/bash", R_OK)    = 0
415499 getpid()                         = 415499
415499 getpgrp()                        = 415496
415499 ioctl(2, TIOCGPGRP, [415496])    = 0
415499 rt_sigaction(SIGCHLD, {sa_handler=0x555dea6fe560, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7fddce63a5b0}, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=SA_RESTORER|SA_RESTART, sa_restorer=0x7fddce63a5b0}, 8) = 0
415499 prlimit64(0, RLIMIT_NPROC, NULL, {rlim_cur=62800, rlim_max=62800}) = 0
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 getpeername(0, 0x7ffc982163a0, [16]) = -1 ENOTSOCK (Socket operation on non-socket)
415499 socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 3
415499 connect(3, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = 0
415499 sendto(3, "\2\0\0\0\v\0\0\0\7\0\0\0passwd\0", 19, MSG_NOSIGNAL, NULL, 0) = 19
415499 poll([{fd=3, events=POLLIN|POLLERR|POLLHUP}], 1, 5000) = 1 ([{fd=3, revents=POLLIN|POLLHUP}])
415499 recvmsg(3, {msg_name=NULL, msg_namelen=0, msg_iov=[{iov_base="passwd\0", iov_len=7}, {iov_base="\310O\3\0\0\0\0\0", iov_len=8}], msg_iovlen=2, msg_control=[{cmsg_len=20, cmsg_level=SOL_SOCKET, cmsg_type=SCM_RIGHTS, cmsg_data=[4]}], msg_controllen=20, msg_flags=MSG_CMSG_CLOEXEC}, MSG_CMSG_CLOEXEC) = 15
415499 mmap(NULL, 217032, PROT_READ, MAP_SHARED, 4, 0) = 0x7fddcf1db000
415499 close(4)                         = 0
415499 close(3)                         = 0
415499 getpid()                         = 415499
415499 openat(AT_FDCWD, "/proc/415499/sched", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
415499 read(3, "bash (415499, #threads: 1)\n-----"..., 1024) = 1024
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415499/stat", O_RDONLY) = 3
415499 read(3, "415499 (bash) R 415496 415496 41"..., 799) = 324
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415496/sched", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
415499 read(3, "strace (415496, #threads: 1)\n---"..., 1024) = 1024
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415496/cmdline", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
415499 read(3, "strace\0-f\0-o\0shell_trace.log\0bas"..., 9216) = 64
415499 read(3, "", 9216)                = 0
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415496/stat", O_RDONLY) = 3
415499 read(3, "415496 (strace) S 415298 415496 "..., 799) = 336
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415298/sched", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
415499 read(3, "bash (415298, #threads: 1)\n-----"..., 1024) = 1024
415499 close(3)                         = 0
415499 readlink("/proc/415298/exe", "/usr/bin/bash", 1023) = 13
415499 openat(AT_FDCWD, "/proc/415298/cmdline", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
415499 read(3, "-bash\0", 9216)         = 6
415499 read(3, "", 9216)                = 0
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415499/stat", O_RDONLY) = 3
415499 read(3, "415499 (bash) R 415496 415496 41"..., 799) = 324
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415496/cmdline", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
415499 read(3, "strace\0-f\0-o\0shell_trace.log\0bas"..., 1024) = 64
415499 read(3, "", 1024)                = 0
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415496/stat", O_RDONLY) = 3
415499 read(3, "415496 (strace) S 415298 415496 "..., 799) = 336
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415298/cmdline", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
415499 read(3, "-bash\0", 1024)         = 6
415499 read(3, "", 1024)                = 0
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415298/stat", O_RDONLY) = 3
415499 read(3, "415298 (bash) S 415297 415298 41"..., 799) = 344
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415297/cmdline", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
415499 read(3, "su\0-\0root\0", 1024)   = 10
415499 read(3, "", 1024)                = 0
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415297/stat", O_RDONLY) = 3
415499 read(3, "415297 (su) S 415296 415296 4152"..., 799) = 332
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415296/cmdline", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
415499 read(3, "sudo\0su\0-\0root\0", 1024) = 15
415499 read(3, "", 1024)                = 0
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415296/stat", O_RDONLY) = 3
415499 read(3, "415296 (sudo) S 415243 415296 41"..., 799) = 326
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415243/cmdline", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
415499 read(3, "-bash\0", 1024)         = 6
415499 read(3, "", 1024)                = 0
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415243/stat", O_RDONLY) = 3
415499 read(3, "415243 (bash) S 415242 415243 41"..., 799) = 344
415499 close(3)                         = 0
415499 openat(AT_FDCWD, "/proc/415242/cmdline", O_RDONLY) = 3
415499 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0
415499 read(3, "sshd: pandaychen@pts/0\0\0\0\0\0\0\0\0\0\0"..., 1024) = 1024
415499 close(3)                         = 0
415499 getcwd("/root", 512)             = 6
415499 ioctl(0, TCGETS, {B38400 opost isig icanon echo ...}) = 0
415499 fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
415499 readlink("/proc/self/fd/0", "/dev/pts/0", 255) = 10
415499 stat("/dev/pts/0", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
415499 socket(AF_UNIX, SOCK_DGRAM, 0)   = 3
415499 fcntl(3, F_GETFL)                = 0x2 (flags O_RDWR)
415499 fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK) = 0
415499 fcntl(3, F_GETFD)                = 0
415499 fcntl(3, F_SETFD, FD_CLOEXEC)    = 0
415499 sendto(3, "\0\0\0\6\0\0\0\377\0\4bash\0#bash -c echo \"ab"..., 255, 0, {sa_family=AF_UNIX, sun_path="/usr/local/sa/agent/log/agent_cmd.sock"}, 40) = 255
415499 rt_sigprocmask(SIG_BLOCK, NULL, [], 8) = 0
415499 openat(AT_FDCWD, "test.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 4
415499 fcntl(1, F_GETFD)                = 0
415499 fcntl(1, F_DUPFD, 10)            = 10
415499 fcntl(1, F_GETFD)                = 0
415499 fcntl(10, F_SETFD, FD_CLOEXEC)   = 0
415499 dup2(4, 1)                       = 1
415499 close(4)                         = 0
415499 getpid()                         = 415499
415499 getcwd("/root", 512)             = 6
415499 ioctl(0, TCGETS, {B38400 opost isig icanon echo ...}) = 0
415499 fstat(0, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
415499 readlink("/proc/self/fd/0", "/dev/pts/0", 255) = 10
415499 stat("/dev/pts/0", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
415499 sendto(3, "\0\0\0\6\0\0\0\351\0\4echo\0\recho abcdefg \0\0\4"..., 233, 0, {sa_family=AF_UNIX, sun_path="/usr/local/sa/agent/log/agent_cmd.sock"}, 40) = 233
415499 fstat(1, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
415499 write(1, "abcdefg\n", 8)         = 8
415499 dup2(10, 1)                      = 1
415499 fcntl(10, F_GETFD)               = 0x1 (flags FD_CLOEXEC)
415499 close(10)                        = 0
415499 rt_sigprocmask(SIG_BLOCK, [CHLD], [], 8) = 0
415499 rt_sigprocmask(SIG_SETMASK, [], NULL, 8) = 0
415499 exit_group(0)                    = ?
415499 +++ exited with 0 +++
```

上面的日志的细节：

```bash
415499 openat(AT_FDCWD, "test.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 4
......
415499 dup2(4, 1)                       = 1 #使 fd 1（标准输出）指向与 fd 4 相同的文件表项
415499 close(4)                         = 0 #关闭 fd 4（因为不再需要它）
......
415499 write(1, "abcdefg\n", 8)         = 8
```
shell 在处理重定向 `>>`时，使用 `dup2`系统调用来将文件描述符重定向到标准输出（`fd=1`），所以`write`写入的`fd=1`而不是`4`

当 shell 执行重定向时，它需要先打开文件 `test.log`，由于 shell 进程本身已经打开了标准输入（`fd=0`）、标准输出（`fd=1`）和标准错误（`fd=2`），可能还有其他文件描述符，所以新打开的文件通常会分配下一个可用的文件描述符，这里是 `fd=4`。此外，shell 不会直接使用新打开的 `fd=4` 进行写入，相反它使用 `dup2`系统调用来将 `fd=4` 复制到标准输出 `fd=1`

对上面的流程进行简化如下

```BASH
#shell 打开文件：
open("test.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 4
#shell 将 fd 4 复制到 fd 1：
dup2(4, 1)                          = 1
#shell 关闭 fd 4（可选，但常见）：
close(4)                            = 0
#shell 执行 echo命令（通过 execve）：
execve("/usr/bin/echo", ["echo", "abcdefg"], ...)
#echo进程写入 fd 1：
write(1, "abcdefg\n", 9)           = 9
```

##  0x04    参考
-   [Linux System Call Table](https://chromium.googlesource.com/chromiumos/docs/+/master/constants/syscalls.md)
-   [linux系统编程之进程（四）：进程退出exit，_exit区别即atexit函数](https://www.cnblogs.com/mickole/p/3186606.html)
-   [enhancedrecording](https://github.com/gravitational/teleport/tree/master/bpf/enhancedrecording)
-   [揭开 strace 命令捕获系统调用的神秘面纱](https://mp.weixin.qq.com/s/Iz4YPPmyjGrXVnZ8eSap2Q)