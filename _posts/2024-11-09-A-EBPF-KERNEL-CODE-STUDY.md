---
layout:     post
title:      EBPF 内核态代码学习（一）：进程调度延时计算
subtitle:   runqslower/runqlat 等CPU性能工具实现分析
date:       2024-11-09
author:     pandaychen
header-img:
catalog: true
tags:
    - eBPF
---

##  0x00    前言
调度延迟是指一个任务`task_struct`具备运行的条件（进入 CPU 的 runqueue），到真正执行（获得 CPU 的执行权）的这段等待调度的时间。延迟是因为 CPU 还被其他任务占据，而且可能还有其他在 runqueue 中排队的任务（见前文），排队的任务越多，调度延迟就可能越长，所以这也是间接衡量 CPU 负载的一个指标（CPU 负载通过计算各个时刻 runqueue 上的任务数量获得）。常用的CPU调度分析命令如下：

-   `runqlat`：统计CPU运行队列的延迟信息，常用于当CPU资源处于饱和状态时，识别和量化问题的严重性
-   `runqlen`：统计CPU运行队列的长度，用来定位负载不均衡问题
-   `runqslower`：运行队列latency超过阈值时打印信息，常用于CPU繁忙系统中，定位哪些进程受到调度延迟

####    runqlat
`runqlat` 常用于分析 Linux 系统的调度性能。`runqlat` 用于测量一个任务在被调度到 CPU 上运行之前在运行队列中等待的时间。这些信息对于识别性能瓶颈和提高 Linux 内核调度算法的整体效率非常有用。`runqlat`提供了直方图统计调度器的runqueue的延时latency，利用对CPU调度器的线程（`task_struct`）唤醒事件和线程上下文切换事件的跟踪来计算线程从唤醒到运行之间的时间间隔。`runqlat`主要追踪两类runqueue latency：

-   进程从被加入队列到上下文切换以及开始执行的时间，追踪路径`ttwu_do_wakeup() --> wake_up_new_task() --> finish_task_switch()`
-   进程从被动上下文切换且仍处于可执行状态（仍在runqueue中）开始到它的下一次开始执行的间隔时间，这是从`finish_task_switch()`开始监测

```BASH
[root@VM-x-x-centos tools]# bpftrace runqlat.bt 
Attaching 5 probes...
Tracing CPU scheduler... Hit Ctrl-C to end.

@usecs: 
[0]                   54 |                                                    |
[1]                  506 |@@@@@                                               |
[2, 4)              2430 |@@@@@@@@@@@@@@@@@@@@@@@@@                           |
[4, 8)              4874 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[8, 16)             3058 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@                    |
[16, 32)            1351 |@@@@@@@@@@@@@@                                      |
[32, 64)             568 |@@@@@@                                              |
[64, 128)            384 |@@@@                                                |
[128, 256)           358 |@@@                                                 |
[256, 512)           420 |@@@@                                                |
[512, 1K)            363 |@@@                                                 |
[1K, 2K)             344 |@@@                                                 |
[2K, 4K)             185 |@                                                   |
[4K, 8K)             107 |@                                                   |
[8K, 16K)             48 |                                                    |
[16K, 32K)            11 |                                                    |
[32K, 64K)             1 |                                                    |
```

也可以参考bpftrace的[实现](https://github.com/bpftrace/bpftrace/blob/master/tools/runqlat.bt)，如下：

```TEXT
tracepoint:sched:sched_wakeup,
tracepoint:sched:sched_wakeup_new
{
	@qtime[args.pid] = nsecs;
}

tracepoint:sched:sched_switch
{
	if (args.prev_state == TASK_RUNNING) {
		@qtime[args.prev_pid] = nsecs;
	}

	$ns = @qtime[args.next_pid];
	if ($ns) {
		@usecs = hist((nsecs - $ns) / 1000);
	}
	delete(@qtime, args.next_pid);
}
```

####    runqlen
`runqlen`：以直方图统计队列中有多少进程正在等待（runqlen是定时采样，采样频率是`99Hz`），通常用来定位CPU负载不均衡问题，比如某些进程同时绑定在同一个CPU上，而其他CPU却空闲，导致进程在等待。此外，相比较于`runqlat`通过跟踪CPU调度器事件来说，`runqlen`定时采样的消耗就小的很多，可忽略不计。对于长时间监控的环境下，优先使用`runqlen`来定位问题

```BASH
[root@VM-218-158-centos tools]# bpftrace runqlen.bt 
Attaching 2 probes...
Sampling run queue length at 99 Hertz... Hit Ctrl-C to end.

@runqlen: 
[0, 1)              1685 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[1, 2)                97 |@@                                                  |
[2, 3)                31 |                                                    |
[3, 4)                13 |                                                    |
[4, 5)                12 |                                                    |
[5, 6)                 7 |                                                    |
[6, 7)                 2 |                                                    |
[7, 8)                 0 |                                                    |
[8, 9)                 1 |                                                    |
```

####    runqslower
linux内核代码提供了`runqslower`的工具，该工具用于展示在CPU runqueue队列中停留的时间大于某一值的任务（哪些进程的调度延迟超过了特定的阈值），有两个版本：

-   [bcc基于libbpf的版本](https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqslower.c)
-   [内核实现的版本](https://github.com/torvalds/linux/blob/master/tools/bpf/runqslower/runqslower.bpf.c)：有若干新特性，比如`v5.11`版本的提供的helper方法：[`bpf_task_storage_get`](https://docs.ebpf.io/linux/helper-function/bpf_task_storage_get/)

主要涉及到如下hook点：
-   `tp_btf/sched_wakeup`：用于处理 `sched_wakeup` 事件，当一个进程从睡眠状态被唤醒时触发
-   `tp_btf/sched_switch`：用于处理 `sched_switch` 事件，当调度器选择一个新的进程运行时触发
-   `tp_btf/sched_wakeup_new`：用于处理 `sched_wakeup_new` 事件，当一个新创建的进程被唤醒时触发

####    runslower工具

1、`runqslower 10000`：检测哪些进程的run delay超过`10ms`

```BASH
[root@VM-X-X-tencentos libbpf-tools]# ./runqslower 10000
Tracing run queue latency higher than 10000 us
TIME     COMM             TID           LAT(us)
17:39:01 process1          262259          27431
17:39:01 process2          262260          27366
17:39:01 process3          262262          27255
```

2、`./runqslower 10000 -P`：增加`-P`选项，可以知道当前进程的调度延迟是由前面执行的哪个任务导致的

```BASH
[root@VM-X-X-tencentos libbpf-tools]# ./runqslower 10000 -P
Tracing run queue latency higher than 10000 us
TIME     COMM             TID           LAT(us) PREV COMM        PREV TID
18:40:23 heartbeat        1974944          60934 swapper/1        0     
18:40:24 heartbeat        1974944          79923 swapper/1        0     
18:40:24 cpulimit         1775688          35520 swapper/4        0     
18:40:39 awk              1975065          79925 swapper/7        0     
18:40:39 grep             1975064          79925 swapper/2        0     
18:40:39 ps               1975061          74938 swapper/3        0     
18:40:39 tmanager-servic  1491             53124 swapper/1        0     
18:40:39 tagentV1.0       1775693          73167 swapper/5        0     
18:40:40 readlink         1975066          84938 swapper/3        0     
18:40:40 ps               1975068          90918 swapper/1        0     
18:40:40 tmanager-servic  1491            49719 runqslower       1974929
```


3、利用`stress`压测下`runqslower`的CPU抢占情况

```BASH
#让一个CPU同时跑两个线程，就可以造成它们互相抢占的情况，所以可以看到两个TID互相抢占的情况
[root@VM-X-X-tencentos]#  taskset -c 1 stress -c 2
stress: info: [2662976] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
```

观察`runqslower`运行结果可以看出这两个线程在互相抢占CPU1：

```BASH
[root@VM-X-X-tencentos libbpf-tools]# ./runqslower 5000 -P
Tracing run queue latency higher than 5000 us
TIME     COMM             TID           LAT(us) PREV COMM        PREV TID
11:09:33 stress           2662978           6995 stress           2662977
11:09:33 stress           2662977           7000 stress           2662978
11:09:33 stress           2662978           5997 stress           2662977
11:09:33 stress           2662978           6997 stress           2662977
11:09:33 stress           2662977           6999 stress           2662978
11:09:33 stress           2662978           7000 stress           2662977
11:09:33 stress           2662977           7000 stress           2662978
11:09:33 stress           2662978           7000 stress           2662977
11:09:33 stress           2662977           7000 stress           2662978
11:09:33 stress           2662978           7000 stress           2662977
11:09:33 stress           2662977           7000 stress           2662978
11:09:33 stress           2662978           7000 stress           2662977
11:09:33 stress           2662977           6000 stress           2662978
11:09:33 stress           2662978           5996 stress           2662977
```


大致原理如下，在`runqslower`发现调度延迟的情况下，必然是由于其他的task的抢占CPU导致。那么可能是某一个task或者某几个task，那么在抓到了调度延迟的时候，把前面的task也dump出来，那么大概率是可以发现是哪个task抢占的CPU时间，这样就可以发现是由于哪个进程的影响导致了延迟

大致流程如下图：

![runqslower](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/scheduler/run-sleep-wakeup.png)


##  0x01    调度器框架和性能：tracepoint相关

####    tracepoint工作原理
Tracepoint是静态插桩，会和内核源码一起编译。默认情况下 Tracepoint是关闭的，因此在插桩点，Tracepoint的实际指令为`nop`，即表示什么都不做。在内核运行时，若用户enable某一Tracepoint，Tracepoint处的`nop`指令会被动态改写为跳转指令`jmp`。`jmp`指令会跳转到当前函数的末尾，这里存放了一个数组，记录了当前Tracepoint的回调函数。用户开启Tracepoint时，探针函数也会以RCU的形式注册到这个数组中。当Tracepoint被关闭后，跳转指令再次覆盖为`nop`，同时用户的探针函数被移除

以`sched_process_exec`为例，对应的内核[定义](https://github.com/torvalds/linux/blob/v6.13/include/trace/events/sched.h#L400)如下：

```CPP
/*
 * Tracepoint for exec:
 */
TRACE_EVENT(sched_process_exec,

	TP_PROTO(struct task_struct *p, pid_t old_pid,
		 struct linux_binprm *bprm),

	TP_ARGS(p, old_pid, bprm),

	TP_STRUCT__entry(
		__string(	filename,	bprm->filename	)
		__field(	pid_t,		pid		)
		__field(	pid_t,		old_pid		)
	),

	TP_fast_assign(
		__assign_str(filename);
		__entry->pid		= p->pid;
		__entry->old_pid	= old_pid;
	),

	TP_printk("filename=%s pid=%d old_pid=%d", __get_str(filename),
		  __entry->pid, __entry->old_pid)
);
```

对应的参数如下（内核提供了tracefs 伪文件系统）
```BASH
$ cat /sys/kernel/debug/tracing/events/sched/sched_process_exec/format 
name: sched_process_exec
ID: 316
format:
    field:unsigned short common_type; offset:0; size:2; signed:0;
    field:unsigned char common_flags; offset:2; size:1; signed:0;
    field:unsigned char common_preempt_count; offset:3; size:1; signed:0;
    field:int common_pid; offset:4; size:4; signed:1;

    field:__data_loc char[] filename; offset:8; size:4; signed:1;
    field:pid_t pid; offset:12; size:4; signed:1;
    field:pid_t old_pid; offset:16; size:4; signed:1;
```

当`sched_process_exec`发生时，代码中也会执行到对应的Tracepoint语句`trace_sched_process_exec`：

```CPP
static int exec_binprm(struct linux_binprm *bprm)
{
    // ...
    audit_bprm(bprm);
    trace_sched_process_exec(current, old_pid, bprm);
    ptrace_event(PTRACE_EVENT_EXEC, old_vpid);
    proc_exec_connector(current);
    return 0;
}
```

####    schedule相关的静态hook点

1、静态跟踪点tracepoint

根据上一小节可知，静态跟踪点的入口是在每个要跟踪的位置埋下`trace_xxx`函数，比如`tracepoint:sched:sched_switch`这个tracepoint静态跟踪点对应的hook[位置](https://github.com/torvalds/linux/blob/v6.13/kernel/sched/core.c#L6753)就在CFS的周期性调度核心函数`__schedule`中：

```CPP
static void __schedule notrace __schedule(bool preempt){
    //...
    trace_sched_switch(preempt, prev, next);
}
```

此外，内核源码中`register_trace_sched_switch`在该静态跟踪点上注册了一些钩子函数，每当内核执行到`__schedule`中的`trace_sched_switch`时，就会调用所注册的`xx_probe_xx` 等函数来完成整个静态跟踪过程

2、`trace_sched_switch`的实现


####    调度相关

##  0x02    runqlat 实现
以bcc的[实现](https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqlat.bpf.c)为例

##	0x03	runqlen 实现
[实现](https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqlen.bpf.c)

##	0x04	runslower 实现
[实现](https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqslower.bpf.c)

##  0x05  参考
-   [透过Tracepoint理解内核 - 调度器框架和性能](https://zhuanlan.zhihu.com/p/143320517)
-   [runqslower实现：kernel](https://github.com/torvalds/linux/blob/master/tools/bpf/runqslower/runqslower.bpf.c)
-   [runqslower实现：bcc](https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqslower.bpf.c)
-   [Helper function bpf_task_storage_get](https://docs.ebpf.io/linux/helper-function/bpf_task_storage_get/)
-   [高性能：7-可用于CPU分析的BPF工具【bpf performance tools读书笔记】](https://cloud.tencent.com/developer/article/1595327)