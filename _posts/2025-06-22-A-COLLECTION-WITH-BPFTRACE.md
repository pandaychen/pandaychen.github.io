---
layout:     post
title:      bpftrace 常用code收集
subtitle:   
date:       2025-06-22
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
    - bpftrace
---

##  0x00    前言
本文的bpftrace工具版本：`bpftrace v0.19.1`

####    bpftrace语法
bpftrace语法（类似awk），`{`前的部分相当于awk中的condition，`{}`中的部分相当于awk的action，不过bpftrace执行actions的条件是触发probe名称指定的事件

probe的格式为`namespace:function_name`，如`tracepoint:timer:tick_stop`、`kprobe:do_sys_open`，好处是便于查找函数及定位不同模块中的同名函数。此外，还有两个特殊的probe即`BEGIN`与`END`，这与awk类似，它们分别在bpftrace程序执行开始、结束时，无条件的执行一些操作，比如完成一些初始化、清理工作等

bpftrace亦可使用ebpf支持的probe：`hardware/iter/kfunc/kprobe/software/tracepoint/uprobe`，如本机内核`6.6.30-5.tl4.x86_64`

```BASH
[root@VM-X-X-tencentos ~]# bpftrace -l  | awk -F ":" '{print $1}' | sort | uniq -c | sort  -nr -k 1
  54112 kprobe
  47345 kfunc
   2364 tracepoint
   1676 rawtracepoint
     16 iter
     14 software
     12 hardware
```

1、内置变量

`pid/tid`：tid是thread id的缩写，在task中的成员是`task_sruct.pid`。所以对于Linux内核，线程=轻量级进程。而`pid`实际上指的是内核中进程组，由task中的`task_sruct.tgid`成员表示。也就是说，进程=线程组
-   `uid/gid`：执行函数的用户ID、组ID
-   `nsecs`：时间戳，纳秒
-   `elapsed`：ebpfs 启动后的纳秒数
-   `cpu`：当前 cpu 编号，从 `0` 开始
-   `comm`：进程名称，通常为进程可执行文件名
-   `kstack`：内核栈
-   `ustack`: 用户栈
-   `arg0~arg1~argN`：函数参数
-   `retval`：返回值
-   `func`：函数名，可以在可执行文件的符号表中这个函数名
-   `probe`：探针的完整名称，也就是 bpftrace 中如 `kprobe:do_nanosleep`
-   `curtask`：当前 `task_struct`
-   `rand`：一个无符号 `32` 位随机数
-   `cgroup`：当前进程的 Cgroup，内核资源组，类似 namespace，docker 等虚拟化技术即基于内核提供的这一基础设施

2、全局变量`@name`

bpftrace中，全局变量对所有的probe actions可见，bpftrace生命周期内可见，bpftrace支持两种变量形式：
-   简单变量`@name = value`：单纯的变量名和值
-   map：`@name[key] = value`

```TEXT
#map例子
kprobe:do_nanosleep {
    @start[tid] = nsecs;
}

kretprobe:do_nanosleep /@start[tid] != 0/ {
    printf("slept for %d ms\n", (nsecs - @start[tid]) / 1000000);
    delete(@start[tid]);
}
```

3、临时变量`$name`， 只在当前action中有效

4、内置函数，参考[文档](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md)，常用`print/time/system`

5、控制/循环类语法，参考[文档](https://github.com/iovisor/bpftrace/blob/master/docs/reference_guide.md)

####    运行方式
-   `bpftrace -e 'cmds`：执行单行指令
-   `bpftrace filename`：执行脚本文件

```BASH
利用 shell 换行：
[root@localhost ~]# bpftrace -e '
BEGIN{
printf("Hello, World!\n");
}

END{
printf("Bye, World!\n");
}'
#在脚本尾部的单引号之前，按照需要换行即可
```

##  0x01    基础示例

1、helloworld

```BASH
bpftrace -e 'BEGIN {printf("hello world\n");}'
```

2、查询hook点

```BASH
[root@VM-X-X-tencentos ~]# bpftrace -l | grep vfs_read
kfunc:vmlinux:vfs_read
kfunc:vmlinux:vfs_readlink
kfunc:vmlinux:vfs_readv
kprobe:vfs_read
kprobe:vfs_readlink
kprobe:vfs_readv

[root@VM-X-X-tencentos ~]# bpftrace -l 'tracepoint:syscalls:sys_enter_*'
```

3、打印`pid`、`comm`

```BASH
bpftrace -e 'kprobe:vfs_read {printf("pid = %d, comm = %s\n",pid,comm);}'
```

4、bpftrace的map使用：进程级系统调用计数

```BASH
[root@VM-119-175-tencentos ~]# bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
Attaching 1 probe...

@[base]: 1
@[server]: 2
@[TsysAgent]: 4
@[bksecbeat]: 4
@[sshd]: 11
@[nscd]: 12
@[TsysProxy]: 15
@[tagentV1.0]: 26
@[bpftrace]: 34
@[agent]: 44
@[nslcdagent]: 60
@[bkmonitorbeat]: 74
@[containerd]: 80
@[os_monitor]: 95
@[sap1015]: 149
@[tmanager-servic]: 150
@[tdsp-filescan]: 157
@[bkhostc]: 183
@[secu-tcs-agent]: 193
@[in:imjournal]: 204
@[tdsp-taskchanne]: 298
@[gse_agent]: 627
@[sshlogd]: 1036
@[barad_agent]: 1358
@[tokio-runtime-w]: 1459
@[sh]: 18838
@[rm]: 51944
```

-   `@`：定义为map，可以在`@`后添加可选的变量名（如`@num`），用来增加可读性或者区分不同的map。`[]`可选的中括号允许设置map的key，比如这里就设置`comm`为key
-   `count()`：map函数用于记录被调用次数。因为调用次数根据`comm`保存在map里，输出结果是进程执行系统调用的次数统计

##  0x02    常用实例

####    静态探针：tracepoint

1、在文件打开的时候打印进程名和文件名

```BASH
[root@VM-119-175-tencentos ~]# bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args.filename)); }'
sshlogd /var/log/sshlog/sessions/ssh_2960655.log
rm /usr/lib64/gconv/gconv-modules.cache
rm /usr/lib/locale/en_US.utf8/LC_MEASUREMENT
rm /usr/lib/locale/en_US.utf8/LC_TELEPHONE
rm /usr/lib/locale/en_US.utf8/LC_ADDRESS
sh /dev/null
rm /usr/lib/locale/en_US.utf8/LC_NAME   
```

-   `comm`是内建变量，代表当前进程的名字。其它类似的变量还有`pid`和`tid`等，分别表示进程标识和线程标识
-   `args`是一个包含所有tracepoint参数的结构。这个结构是由bpftrace根据tracepoint信息自动生成的。这个结构的成员可以通过命令`bpftrace -vl tracepoint:syscalls:sys_enter_openat`查询
-   `args.filename`用来获取`args`的成员变量`filename`的值
-   `str()`：将字符串指针转换成字符串

2、捕获进程调用参数

```BASH
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s ", comm); join(args->argv); }'

sh rm -f /tmp/test/697425390.tmp
sh rm -f /tmp/test/694688366.tmp1
sh rm -f /tmp/test/694688367.tmp1
bash readlink /proc/3545309/exe
sh rm -f /tmp/test/694688368.tmp1
bash basename /usr/bin/bash
sh rm -f /tmp/test/694688369.tmp1
```

3、`sys_enter_finit_module`

```BASH
#!/usr/bin/env bpftrace

BEGIN {
    printf("Tracing init_module and finit_module syscalls... Hit Ctrl+C to stop.\n");
}

tracepoint:syscalls:sys_enter_finit_module {
    printf("Syscall executed: %s (PID: %d, UID: %d, Command: %s),umod: %d\n", probe, pid, uid, comm, args->fd);
}
```

4、块级I/O跟踪，print块I/O请求字节数的直方图

```BASH
#运行内核版本：6.6.30-5.tl4.x86_64
bpftrace -e 'tracepoint:block:block_rq_issue { @ = hist(args.bytes); }'
```

-   `tracepoint:block`：表示块类别的跟踪点跟踪块级I/O事件
-   `block_rq_issue`：跟踪点，当I/O提交到块设备时触发
-   `args.bytes`：跟踪点`block_rq_issue`的参数成员`bytes`，表示提交I/O请求的字节数

注意这里的`tracepoint:block:block_rq_issue`，它在I/O请求被提交给块设备时触发。这个hook点通常发生在进程上下文，此时通过内核的`comm`可以得到进程名；也可能发生在内核上下文，（如`readahead`），此时不能显示预期的进程号和进程名信息，特别解释下：

关联进程上下文的场景，当一个用户空间进程直接发起一个块设备 I/O 请求时，该请求最终会通过内核的系统调用路径（如 `read/write/pread/pwrite`等）到达块设备层，此时上下文特点是内核代码是在代表某个特定的用户空间进程执行
内核可以访问该进程的完整信息，因为它是由该进程的系统调用带入内核的，此时内核处于该进程的上下文中，`block_rq_issue`的表现为当 I/O 请求最终被提交到块设备队列时，触发 `block_rq_issue` hook调用，此时内核可以轻松地获取并记录发起这个 I/O 请求的进程的信息。通过 `comm`字段可以获取到发起系统调用的那个用户空间进程的名字

关联内核上下文的场景是，由于存在内核的预读机制，当 I/O 请求的发起完全源于内核内部的操作，没有直接关联到一个特定的、当前正在执行系统调用的用户空间进程。常见场景如下：

-   预读 (Read-ahead)：内核预测进程接下来可能需要读取更多数据，提前从磁盘读取到页缓存。这个预测和读取动作是由内核的预读算法触发的，并非由进程的当前 `read`调用直接发起（虽然最初的 `read`可能触发了预读机制的启动）
-   页面回收/交换 (Page Reclaim/Swapping)：当内存压力大时，内核的 `kswapd`线程或直接回收路径可能需要将脏页写回磁盘（如 `pdflush`或其继承者如 `kworker`线程）
-   文件系统元数据操作： 文件系统为了维护自身结构（如写日志、更新 inode）而发起的 I/O
-   设备映射/DM/ MD RAID 等：一些虚拟块设备层内部的操作可能触发额外的 I/O
-   内核线程直接发起的 I/O：某些内核线程（如负责数据刷新）可能直接操作块设备

与进程上下文不同的是，内核代码的执行不是代表某个特定的用户空间进程。它可能是在一个内核线程（如 `kworker`等）的上下文中运行。它也可能是在一个中断处理程序的上下文中，内核无法关联到一个具体的用户空间进程。此时`block_rq_issue`的表现为当这种源于内核内部的 I/O 请求被提交时，同样会触发 `block_rq_issue`hook调用，这时获取的`comm`值结果是不可靠的，可能是当前正在 CPU 上执行的内核线程的名字

输出如下：
```
@: 
[16, 32)               6 |@                                                   |
[32, 64)               0 |                                                    |
[64, 128)              0 |                                                    |
[128, 256)             0 |                                                    |
[256, 512)             0 |                                                    |
[512, 1K)              0 |                                                    |
[1K, 2K)               0 |                                                    |
[2K, 4K)               0 |                                                    |
[4K, 8K)             233 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)             22 |@@@@                                                |
[16K, 32K)            25 |@@@@@                                               |
[32K, 64K)            13 |@@                                                  |
[64K, 128K)            4 |                                                    |
[128K, 256K)           4 |                                                    |
```

5、调度器跟踪：统计进程上下文切换次数，主要涉及到hook点为`tracepoint:sched:sched_switch`，根据前文分析CPU切换的文章可知，该hook点主要是当线程主动释放（让出）cpu资源，当前不运行时触发。这里可能的阻塞事件:如等待I/O，定时器，分页/交换，锁等

```BASH
#kstack：内核堆栈跟踪，打印调用栈
bpftrace -e 'tracepoint:sched:sched_switch { @[kstack] = count(); }'

#采样输出如下：
@[
    __traceiter_sched_switch+71
    __traceiter_sched_switch+71
    __schedule+719
    schedule_idle+42
    do_idle+196
    cpu_startup_entry+42
    start_secondary+276
    secondary_startup_64_no_verify+404
]: 2954
```

`sched_switch`在线程切换的时候触发，打印的调用栈是被切换出cpu的那个线程。注意这里的上下文，例如`comm/pid/kstack`等等，并不一定反映了探针的目标的状态（和块IO追踪类似）

6、统计进程级别的事件

```BASH
#直接统计
bpftrace -e 'tracepoint:sched:sched* { @[probe] = count(); }'

#统计5秒内进程级的事件并打印
[root@VM-X-X-tencentos ~]# bpftrace -e 'tracepoint:sched:sched* { @[probe] = count(); } interval:s:5 { exit(); }'
Attaching 28 probes...
@[tracepoint:sched:sched_wake_idle_without_ipi]: 598
@[tracepoint:sched:sched_process_exec]: 1386
@[tracepoint:sched:sched_process_exit]: 1406
@[tracepoint:sched:sched_process_free]: 1411
@[tracepoint:sched:sched_process_fork]: 1411
@[tracepoint:sched:sched_wakeup_new]: 1481
@[tracepoint:sched:sched_process_wait]: 2963
@[tracepoint:sched:sched_migrate_task]: 4450
@[tracepoint:sched:sched_wakeup]: 21631
@[tracepoint:sched:sched_waking]: 21722
@[tracepoint:sched:sched_stat_runtime]: 42597
@[tracepoint:sched:sched_switch]: 44724
```

-   `sched*`：sched探针可以探测调度器的高级事件和进程事件如`fork/exec`和上下文切换等
-   `probe`：内置变量，作为统计map的key，探针的完整名称
-   `interval:s:5`：每`5`秒在每个CPU上触发一次的探针，它用来创建脚本级别的间隔或超时时间
-   `exit()`：退出bpftrace

7、基于系统调用`read()`返回值分布统计

```BASH
[root@VM-X-X-tencentos ~]# bpftrace -e 'tracepoint:syscalls:sys_exit_read  { @bytes = hist(args.ret); }'
Attaching 1 probe...

@bytes: 
(..., 0)             252 |@@@@@@                                              |
[0]                 2106 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[1]                    4 |                                                    |
[2, 4)                 1 |                                                    |
[4, 8)                16 |                                                    |
[8, 16)               43 |@                                                   |
[16, 32)             573 |@@@@@@@@@@@@@@                                      |
[32, 64)              60 |@                                                   |
[64, 128)           1260 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                     |
[128, 256)           129 |@@@                                                 |
[256, 512)           619 |@@@@@@@@@@@@@@@                                     |
[512, 1K)           1266 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                     |
[1K, 2K)             429 |@@@@@@@@@@                                          |
[2K, 4K)             674 |@@@@@@@@@@@@@@@@                                    |
[4K, 8K)               1 |                                                    |
```

可以使用filter进行过滤，如：

```BASH
#统计进程号为18644的进程执行内核函数sys_read()的返回值，并打印出直方图
bpftrace -e 'tracepoint:syscalls:sys_exit_read /pid == 18644/ { @bytes = hist(args.ret); }'
```

-   `/.../`：设置一个过滤条件，满足该过滤条件时才执行`{}`里面的动作
-   `ret`：表示函数的返回值。对于`sys_read()`，取值可能是`-1`（错误）或者成功读取的字节数
-   `hist()`：与`@`配合使用，一个map函数，用来描述直方图的参数。输出行以`2`次方的间隔开始，如`[128, 256)`表示值大于等于`128`且小于`256`。后面跟着位于该区间的参数个数统计，最后是ascii码表示的直方图

####    动态探针：kprobe
1、定时调用 `ps` 查看当前进程

```BASH
[root@VM-X-X-centos ~]#  bpftrace --unsafe -e 'kprobe:do_nanosleep { system("ps -p %d\n", pid); }'
Attaching 1 probe...
    PID TTY          TIME CMD
  27511 ?        08:49:56 xx
    PID TTY          TIME CMD
  27511 ?        08:49:56 xx
```

2、内核结构成员跟踪（打印），打印`vfs_open`函数第一个参数对应的dentry name，函数原型为`int vfs_open(const struct path *path, struct file *file)`

-   `arg0` 是一个内建变量，表示探针的第一个参数，其含义由探针类型决定。对于kprobe类型探针，它表示函数的第一个参数。其它参数使用`arg1~argvN`表示
-   `((struct path *)arg0)->dentry->d_name.name`：这里`arg0`作为`struct path *`并引用dentry
-   `include`代码：在没有BTF (BPF Type Format) 的情况下,包含必要的path和dentry类型声明的头文件

重要：bpftrace对内核结构跟踪的支持和bcc是一样的，允许使用内核头文件。这意味着大多数结构是可用的，但是并不是所有的，有时需要手动增加某些结构的声明

```BASH
#cat path.bt    -- remove this line
#ifndef BPFTRACE_HAVE_BTF
#include <linux/path.h>
#include <linux/dcache.h>
#endif

kprobe:vfs_open
{
    printf("open path: %s\n", str(((struct path *)arg0)->dentry->d_name.name));
}
```

3、基于线程id `tid`，统计`read()`系统调用（内核层面`vfs_read`）调用的时间

```BASH
#根据进程名，以直方图的形式显示read()调用花费的时间，时间单位为ns
[root@VM-X-X-tencentos ~]# bpftrace -e 'kprobe:vfs_read { @start[tid] = nsecs; } kretprobe:vfs_read /@start[tid]/ { @ns[comm] = hist(nsecs - @start[tid]); delete(@start[tid]); }'

Attaching 2 probes...

@ns[dockerd]: 
[8K, 16K)              1 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@ns[base]: 
[64K, 128K)            1 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@ns[sshd]: 
[8K, 16K)              1 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[16K, 32K)             0 |                                                    |
[32K, 64K)             1 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

......

@ns[crond]: 
[2K, 4K)               5 |                                                    |
[4K, 8K)             330 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)            174 |@@@@@@@@@@@@@@@@@@@@@@@@@@@                         |
[16K, 32K)             9 |@                                                   |
[32K, 64K)             2 |                                                    |
[64K, 128K)            0 |                                                    |
[128K, 256K)           1 |                                                    |
[256K, 512K)           0 |                                                    |
[512K, 1M)             0 |                                                    |
[1M, 2M)               0 |                                                    |
[2M, 4M)               0 |                                                    |
[4M, 8M)               0 |                                                    |
[8M, 16M)              0 |                                                    |
[16M, 32M)             1 |                                                    |
[32M, 64M)             0 |                                                    |
[64M, 128M)            0 |                                                    |
[128M, 256M)           1 |                                                    |
[256M, 512M)           4 |                                                    |

@ns[sshlogd]: 
[2K, 4K)             325 |@@@@@@@@@@@@@@@@                                    |
[4K, 8K)             998 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)             13 |                                                    |
[16K, 32K)             6 |                                                    |

@ns[ps]: 
[2K, 4K)             805 |@@@@@@@@@@@@@@@@@@                                  |
[4K, 8K)            1970 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@       |
[8K, 16K)           1393 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                    |
[16K, 32K)          1492 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                  |
[32K, 64K)          2228 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[64K, 128K)           92 |@@                                                  |
[128K, 256K)          22 |                                                    |
[256K, 512K)           5 |                                                    |

@ns[rm]: 
[2K, 4K)             982 |@@@@@@                                              |
[4K, 8K)            7476 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)            148 |@                                                   |
[16K, 32K)            26 |                                                    |
[32K, 64K)            16 |                                                    |
[64K, 128K)            5 |                                                    |
[128K, 256K)           1 |                                                    |

......

@start[2023558]: 28263891995592926
@start[2973970]: 28263891996321400
@start[436182]: 28263891997194846
@start[434562]: 28263891998296840
```

这是一个比较有趣的示例：

-   `@start[tid]`：需求是希望为每个调用记录一个起始时间戳，使用线程ID（`tid`）作为key，由于内核线程一次只能执行一个系统调用，可以使用线程ID作为上述标识符
-   `nsecs`：内置变量，自系统启动到现在的纳秒数。这是一个高精度时间戳，可以用来对事件计时
-   `/@start[tid]/`：该过滤条件检查起始时间戳是否被记录。程序可能在某次`read`调用中途被启动，如果没有这个过滤条件，这个调用的时间会被统计为`now-zero`，而不是`now-start`
-   `delete(@start[tid])`：释放变量

也可以把这里的`tid`修改为`pid`，结果如下：

```BASH
[root@VM-X-X-tencentos ~]# bpftrace -e 'kprobe:vfs_read { @start[pid] = nsecs; } kretprobe:vfs_read /@start[pid]/ { @ns[comm] = hist(nsecs - @start[pid]); delete(@start[pid]); }'
Attaching 2 probes...

@ns[sshd]: 
[4K, 8K)               1 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)              0 |                                                    |
[16K, 32K)             0 |                                                    |
[32K, 64K)             1 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@ns[nscd]: 
[8K, 16K)              2 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@ns[in:imjournal]: 
[4K, 8K)               2 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@ns[containerd]: 
[8K, 16K)              4 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|

@ns[base]: 
[4K, 8K)               4 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)              0 |                                                    |
[16K, 32K)             0 |                                                    |
[32K, 64K)             0 |                                                    |
[64K, 128K)            1 |@@@@@@@@@@@@@                                       |
[128K, 256K)           0 |                                                    |
[256K, 512K)           0 |                                                    |
[512K, 1M)             0 |                                                    |
[1M, 2M)               0 |                                                    |
[2M, 4M)               0 |                                                    |
[4M, 8M)               0 |                                                    |
[8M, 16M)              0 |                                                    |
[16M, 32M)             0 |                                                    |
[32M, 64M)             1 |@@@@@@@@@@@@@                                       |

@ns[df]: 
[4K, 8K)               7 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8K, 16K)              1 |@@@@@@@                                             |
[16K, 32K)             0 |                                                    |
[32K, 64K)             1 |@@@@@@@                                             |
[64K, 128K)            1 |@@@@@@@                                             |
[128K, 256K)           1 |@@@@@@@                                             |

@start[2973908]: 28264627263193561
@start[642691]: 28264627265250099
```

4、内核动态跟踪`read()`返回的字节数，基于`kretprobe:vfs_read`

```BASH
#使用内核动态跟踪技术显示read()返回字节数的直方图
#lhist：线性直方图函数，参数分别是value，最小值，最大值，步进值
#第一个参数（retval）：表示系统调用sys_read()返回值，即成功读取的字节数
[root@VM-119-175-tencentos ~]# bpftrace -e 'kretprobe:vfs_read { @bytes = lhist(retval, 0, 2000, 200); }'
Attaching 1 probe...

@bytes: 
(..., 0)             806 |@@@                                                 |
[0, 200)           10879 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[200, 400)          1401 |@@@@@@                                              |
[400, 600)            97 |                                                    |
[600, 800)          2315 |@@@@@@@@@@@                                         |
[800, 1000)         3493 |@@@@@@@@@@@@@@@@                                    |
[1000, 1200)         895 |@@@@                                                |
[1200, 1400)           0 |                                                    |
[1400, 1600)          12 |                                                    |
[1600, 1800)           6 |                                                    |
[1800, 2000)           1 |                                                    |
[2000, ...)         2035 |@@@@@@@@@                                           |
```

####    profile
1、分析内核实时函数栈（注意`profile:hz:99`的意义），采样频率太大会影响性能

`@[kstack] = count()`的用法是`kstack`作为返回内核调用栈作为map的key，可以跟踪次数（value）

```BASH
#以99赫兹的频率分析内核调用栈并打印次数统计
[root@VM-119-175-tencentos ~]# bpftrace -e 'profile:hz:99 { @[kstack] = count(); }'
@[
    next_uptodate_folio+191
    filemap_map_pages+436
    do_read_fault+264
    do_fault+330
    handle_pte_fault+265
    __handle_mm_fault+818
    handle_mm_fault+410
    do_user_addr_fault+360
    exc_page_fault+105
    asm_exc_page_fault+39
]: 39
@[
    do_user_addr_fault+220
    exc_page_fault+105
    asm_exc_page_fault+39
]: 97
@[
    finish_task_switch.isra.0+154
    __schedule+561
    schedule_idle+42
    do_idle+196
    cpu_startup_entry+42
    start_secondary+276
    secondary_startup_64_no_verify+404
]: 109
@[]: 143
@[]: 656
@[
    default_idle+11
    default_idle_call+44
    cpuidle_idle_call+291
    do_idle+144
    cpu_startup_entry+42
    __pfx_kernel_init+0
    arch_call_rest_init+14
    start_kernel+913
    x86_64_start_reservations+24
    x86_64_start_kernel+146
    secondary_startup_64_no_verify+404
]: 2527
@[
    default_idle+11
    default_idle_call+44
    cpuidle_idle_call+291
    do_idle+144
    cpu_startup_entry+42
    start_secondary+276
    secondary_startup_64_no_verify+404
]: 17471
```

2、分析用户级函数栈

```bash
[root@VM-X-X-tencentos ~]# bpftrace -e 'profile:hz:99 { @[ustack] = count(); }'
@[
    internal/runtime/syscall.Syscall6+14
    runtime.netpoll+210
    runtime.findRunnable+2245
    runtime.schedule+177
    runtime.park_m+491
    runtime.mcall+78
    runtime.gopark+206
    runtime.chanrecv+1052
    runtime.chanrecv1+18
    github.com/gogf/gf/os/gtimer.(*Timer).loop.func1+105
    runtime.goexit.abi0+1
]: 1
@[
    __clock_nanosleep+101
    void std::__invoke_impl<void, void (sshlog::FailedLoginWatcherThread::*)(), sshlog::FailedLoginWatcherThread*>(std::__invoke_memfun_deref, void (sshlog::FailedLoginWatcherThread::*&&)(), sshlog::FailedLoginWatcherThread*&&)+106
    std::__invoke_result<void (sshlog::FailedLoginWatcherThread::*)(), sshlog::FailedLoginWatcherThread*>::type std::__invoke<void (sshlog::FailedLoginWatcherThread::*)(), sshlog::FailedLoginWatcherThread*>(void (sshlog::FailedLoginWatcherThread::*&&)(), sshlog::FailedLoginWatcherThread*&&)+59
    void std::thread::_Invoker<std::tuple<void (sshlog::FailedLoginWatcherThread::*)(), sshlog::FailedLoginWatcherThread*> >::_M_invoke<0ul, 1ul>(std::_Index_tuple<0ul, 1ul>)+71
    std::thread::_Invoker<std::tuple<void (sshlog::FailedLoginWatcherThread::*)(), sshlog::FailedLoginWatcherThread*> >::operator()()+28
    std::thread::_State_impl<std::thread::_Invoker<std::tuple<void (sshlog::FailedLoginWatcherThread::*)(), sshlog::FailedLoginWatcherThread*> > >::_M_run()+32
    0x7fa2ca8f4de4
    0x27a9428
    std::thread::_State_impl<std::thread::_Invoker<std::tuple<void (sshlog::FailedLoginWatcherThread::*)(), sshlog::FailedLoginWatcherThread*> > >::~_State_impl()+0
    0xf87d894810ec8348
]: 2
```

为什么要采用`hz:99`呢？这里涉及到一类采样应用的技巧，profile是 bpftrace 提供的一种定时采样探针类型。它会在所有 CPU 核心上以固定的时间间隔触发，类似于性能分析工具（如 perf）的采样机制，`hz`表示采样频率的单位是 Hertz（赫兹），即每秒采样的次数，而`99`表示采样频率为 `99` 次/秒。`hz:99`的含义是以每秒 `99` 次的频率在所有 CPU 核心上进行定时采样

为什么是 `99` 而不是 `100`呢？使用非整数频率（如 `99Hz`）是一种常见技巧，目的是避免采样周期与系统定时器或其他周期性事件同步。如果使用 `100Hz`（或其他整数频率等），采样点可能与某些固定周期事件（如系统定时器中断）对齐，导致采样结果偏向某些特定代码路径，产生观测偏差。选择 `99Hz` 这种略低于标准值的频率，可以使采样点随机化，获得更均匀的统计分布

如上面引入的示例`bpftrace -e 'profile:hz:99 { @[ustack] = count(); }'`，该命令会每秒 `99` 次捕获所有 CPU 上正在执行的用户空间程序的调用栈（ustack），通过统计不同调用栈的出现次数（`count()`），可以分析出哪些代码路径消耗的 CPU 时间最多，用于可视化分析性能热点

####    uprobe


####    USDT



##  0x03    参考
-   [bpftrace一行教程](https://eunomia.dev/zh/tutorials/bpftrace-tutorial/)
-   [基于ebpf的性能工具-bpftrace脚本语法](https://cloud.tencent.com/developer/article/2322518)
-   [The bpftrace One-Liner Tutorial](https://github.com/bpftrace/bpftrace/blob/master/docs/tutorial_one_liners.md)