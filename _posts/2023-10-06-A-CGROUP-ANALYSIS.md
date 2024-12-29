---
layout:     post
title:      Linux Namespace && Cgroup
subtitle:   Linux 系统资源隔离与管理机制介绍
date:       2023-10-06
author:     pandaychen
catalog:    true
tags:
    - Cgroup
    - Namespace
---

##  0x00    前言
Linux Namespace 和 Cgroup 都是 Linux 内核的功能，用于隔离和管理系统资源，但它们的目标和方法有所不同

```TEXT
cgroup = limits how much you can use;
namespaces = limits what you can see (and therefore use)
```

-   Linux Namespace 主要用于隔离系统资源，在一个 Namespace 中的进程看不到另一个 Namespace 中的资源。例如一个进程可能有自己的网络堆栈，与其他进程的网络堆栈完全隔离。Namespace 是实现容器技术的基础，如 Docker， Namespace 是将内核的全局资源做封装，使得每个 Namespace 都有一份独立的资源，因此不同的进程在各自的 Namespace 内对同一种资源的使用不会互相干扰
-   Cgroup（Control Group）则主要用于管理和限制系统资源的使用。它可以限制一个进程组使用的 CPU、内存等资源的数量。如可以设置一个 Cgroup，使得其中的进程最多只能使用 `50%` 的 CPU 时间，Cgroup 是内核提供的一种资源隔离的机制，可以实现对进程所使用的 cpu、内存物理资源、及网络带宽等进行限制。还可以通过分配的 CPU 时间片数量及磁盘 IO 宽带大小控制任务运行的优先级

Cgroups 可以理解为是房子的土地面积，限制了房子的大小 ，而 Namespace 是房子的墙，与邻居互相隔离

![namespace-and-cgroup](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/linux/namespace-vs-cgroup.png)

####    Linux CFS （Completely Fair Scheduler：完全公平调度器） 
针对普通进程，在一般的抢占式调度实现中，当该进程用完内核为其所分配的固定 CPU 时间片（进程使用的 CPU 配额时间）时，内核将会停止一个进程的运行（该进程让出CPU），转而运行其它的进程。CFS 算法中的时间片的称为virtual runtime（vruntime），使⽤ vruntime 来记录进程的虚拟执⾏时间，vruntime的计算公式如下（单位：`ns`）：

![vruntime](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/process-cfs/compute.png)

vruntime是通过进程的优先级（如 `nice` 值）与实际进程执⾏时间加权所得到的⼀个值，进程的 `nice` 值越⾼，表⽰该进程越友好，就更乐意把 cpu 让给别⼈，即进程优先级越低。当决定下⼀个调度进程时，调度器将选择最⼩虚拟运⾏时间（vruntime）的任务；当调整进程的 `nice` 值时，会直接影响进程的 vruntime 的计算，更小数值的 vruntime会以更高的优先级被调度

以具有相同优先级的 I/O 密集型任务和 CPU 密集型任务为例。I/O 密集型任务通常在运⾏很短的时间以后就开始等待 I/O 事件并让出 CPU。⽽ CPU 密集型任务只要能拿到 CPU，就可能⼀直运⾏。因此⼀段时间以后，I/O 密集型任务的 vruntime 将⼩于 CPU 密集型任务的 vruntime，从⽽将拥有更⾼的调度优先级，那么也就有了更低的响应延迟

##  0x01    Namespace

当前 Linux 内核总共支持以下几种 Namespace：

-   IPC：隔离 System V IPC 和 POSIX 消息队列
-   Network：隔离网络资源
-   Mount：隔离文件系统挂载点
-   PID：隔离进程 ID
-   UTS：隔离主机名和域名
-   User：隔离用户 ID 和组 ID

####    PID namespace
Linux PID 命名空间用于隔离进程 ID，在一个 PID 命名空间中，每个进程拥有独立的进程 ID，这样在不同的命名空间中可以有相同的进程 ID，而不会产生冲突。每个子 PID 命名空间中都有 PID 为 `1` 的 `init` 进程，对应父命名空间中的进程，父命名空间对子命名空间运行状态是不隔离的，但是每一个子命名空间是互相隔离的

在子命名空间 A 和 B 中都有一个进程 `ID=1` 的 `init` 进程，这两个进程实际上是父命名空间的 `55` 号和 `66` 号进程 ID, 虚拟化出来的空间而已

![pid-namespace](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/linux/pid-namespace.png)

##  0x02  Cgroup
先介绍下Cgroup机制中相关名称的概念：

-   任务（task）： 在cgroup中，任务相当于是一个进程，可以属于不同的cgroup组，但是所属的cgroup不能同属一层级
-   任务/控制组： 资源控制是以控制组的方式实现的，进程可以加入到指定的控制组中，类似于Linux中user和group的关系。控制组为树状结构的上下父子关系，子节点控制组会继承父节点控制组的属性，如资源配额等
-   层级（hierarchy）： 一个大的控制组群树，归属于一个层级中，不同的控制组以层级区分开
-   子系统（subsystem）： 一个的资源控制器，比如cpu子系统可以控制进程的cpu使用率，子系统需要附加（attach）到某个层级，然后该层级的所有控制组，均受到该子系统的控制

####    核心组件
-   子系统（subsystem）：子系统是用于管理特定类型资源的模块，每个子系统负责管理一类资源，例如 CPU、内存、磁盘 I/O、网络带宽等
-   Cgroup Hierarchies（Cgroup 层级树）：Hierarchies 是用于组织和管理 Cgroup 的结构。一个 Hierarchies 由一个或多个 subsystem 组成。Hierarchies 形成树状结构，每个节点都是一个 Cgroup，同一层级中的 Cgroup 之间是平级的。Hierarchies 允许 Cgroup 在不同的 subsystem 中进行组合和嵌套，形成多层的资源管理结构。系统会默认为每个 subsystem 创建一个默认的 Hierarchy
-   Cgroups（控制组）：Cgroup 是最终的资源管理单元，它是一组进程的集合，被组织在一个特定的 Cgroup 层级中，并受到该层级中控制器的资源限制和管理

核心组件关系如下图：

![cgroups](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/linux/cgroups.png)

关系说明如下：

-  默认系统会为所有的 subsystem 创建 Hierarchy 树，默认 cgroup 根路径在`/sys/fs/cgroup` 目录下，在`/sys/fs/cgroup` 下可以看到全部的 subsystem 对应的 Hierarchy 树
-  系统创建了新的 Hierarchy 后，默认所有进程都会加入到树中根节点的 cgroup 中，例如可以到`/sys/fs/cgroup/cpu/tasks` 文件中看到所有的进程列表
-  一个 subsystem 只能附加到一个 Hierarchy 树上
-   一个 Hierarchy 树可以对应多个 subsystem
-   一个进程可以作为多个 cgroup 的成员，但是这些 cgroup 必须在不同的 Hierarchy 树中
-   一个进程 fork 子进程时，子进程和父进程默认是在同一个 cgroup 中的（继承父进程的cgroup），也可以自行移动到其他 cgroup 中

####    cpu subsystem
用于管理 CPU 资源。它允许设置进程的 CPU 使用率和时间片配额，从而限制进程的 CPU 占用，cpu 子系统的一些常见控制文件如下：

-   `cpu.cfs_quota_us`：用于设置 cgroup 中进程的 CPU 配额（μs），表示在每个 `cpu.cfs_period_us` 微秒内，cgroup 中的进程可以使用的 CPU 时间量（-1表示 cgroup 中的进程无 CPU 使用限制）
-   `cpu.cfs_period_us`：用于设置 cgroup 中进程的 CPU 时间周期（μs），表示在一个周期内，cgroup 中的进程可以使用的 CPU 时间总量

####    memory subsystem
用于管理和限制进程组中的内存资源使用。memory 子系统允许为每个 cgroup 设置内存限制，并控制进程在 cgroup 中使用的内存量

-   `memory.limit_in_bytes`：用于设置 cgroup 中进程的内存限制。可以设置一个整数值，表示 cgroup 中所有进程可使用的内存上限，单位为字节。超过该限制的内存请求将被拒绝，进程可能会受到 OOM（Out of Memory）事件的影响
-   `memory.soft_limit_in_bytes`：用于设置 cgroup 中进程的软内存限制。软内存限制是一个较低的限制值，当系统内存不足时，它可以防止进程抢占过多的内存资源，但通常不会导致 OOM 事件
-   `memory.memsw.limit_in_bytes`：用于设置 cgroup 中进程的内存+交换空间（swap）的限制。可以设置一个整数值，表示 cgroup 中所有进程可使用的内存+swap 的上限
-   `memory.memsw.usage_in_bytes`：记录了 cgroup 中所有进程当前的内存+swap 使用量

##  0x03    CGROUP 限制配置 && 实现
正如前文描述，在共享的机器上，进程相互隔离，互不影响，对其它进程是种保护。对于可能存在内存泄漏的进程，可以设置内存限制，通过系统 OOM 触发的 Kill 信号量来实现重启。本小节给出一个基于`/sys/fs/cgroup/memory`控制内存使用的case

1、使用 `mkdir /sys/fs/cgroup/memory/climits` 来创建属于自己的内存组 `climits`，此时系统已经在目录 `/sys/fs/cgroup/memory/climits` 下生成了内存相关的所有配置

```BASH
cgroup.procs #使用该组配置的进程列表
memory.limit_in_bytes #内存使用限制
memory.memsw.limit_in_bytes #内存和交换分区总计限制
memory.swappiness #交换分区使用比例
memory.usage_in_bytes #当前进程内存使用量
memory.stat #内存使用统计信息
memory.oom_control #OOM 控制参数
```

2、设置内存限制，为进程 pid `1234`设置内存限制为 `10MB`

```BASH
echo 10M > /sys/fs/cgroup/memory/climits/memory.limit_in_bytes  #limit_in_bytes 设置为 10MB
echo 0 > /sys/fs/cgroup/memory/climits/memory.swappiness  #禁用交换分区，实际生产中可以配置合适的比例
echo 1234 > /sys/fs/cgroup/memory/climits/cgroup.procs #当进程 1234 使用内存超过 10MB 的时候，默认进程 1234 会触发 OOM，被系统 Kill 掉
```

####    Cgroup的工作原理
1、配置动态即时生效：Linux 内核通过虚拟文件的方式对外暴露 Cgroup 的相关实时动态配置接口（参考上文），修改即时生效，会直接修改运行时的内核状态。内核通过 `poll()` 机制来监听 Cgroup 所挂载目录下的全部文件读写（如 `cpu.cfs_quota_us`、`cpu.shares`等），Cgroup 文件被修改时将会调用 `cgroup_file_write()` 函数，一方面将数据写入到虚拟文件中，另一方面则是修改内核运行时的一些数据结构

2、限制进程使用 CPU 资源上限

还是以 `cpu.cfs_quota_us` 为例，该配置限制某个进程在 `cpu.cfs_period_us` 时间内能够使用的最大 CPU 配额，默认值 `-1`表示该进程没有使用 CPU 资源的限制，`cfs_period_us` 的默认值为 `100000us`，思考下面这个场景：

当 `cfs_period_us` 为 `100000`，而 `cfs_quota_us` 为 `50000` 时，表示当前进程可以在 `100` 毫秒（`100ms=100000us`）内使用 `50` 毫秒的 CPU，也就是 `0.5` 个 CPU；同样，当 `cfs_period_us` 为 `100000`，而 `cfs_quota_us` 为 `200000` 时，表示当前进程可以使用 `2` 个 CPU，那么既然需要通过 `cfs_quota_us` 和 `cfs_period_us` 的比例来决定进程能使用多少个 CPU 的话，那么 `100:1000` 和 `1000:10000` 有何区别？

还是从`cfs_period_us`的定义即**使用最大CPU的配额**来看：

1.  更大的`cfs_period_us`周期将会使得进程有更大的突发能力，但是无法保证延迟响应， 这里的突发能力指的是允许应用程序短暂地突破他们的配额限制
2.  更小的`cfs_period_us`周期将会以牺牲突发容量为代价来确保稳定的延迟响应，因为这样一来内核就必须要较小的时间内对进程进行调度，这自然会有更稳定的延迟响应

3、`cpu.cfs_period_us`的生效过程

当修改`cpu.cfs_period_us`时，内核会调用 `cpu_cfs_period_write_u64()` 函数来修改对应 task_group 的 CPU 带宽，该值的修改最终会导致 `cfs_rq.runtime_enabled` 和 `cfs_rq.runtime_remaining` 两个值发生变化，从而直接影响 CFS 的调度。当一个 CGroup 的 `runtime_remaining<=0` 时，CFS 直接对其进行限流（对应内核函数`check_enqueue_throttle`）

```CPP
static int tg_set_cfs_period(struct task_group *tg, long cfs_period_us)
{
	u64 quota, period, burst;

	if ((u64)cfs_period_us > U64_MAX / NSEC_PER_USEC)
		return -EINVAL;

	period = (u64)cfs_period_us * NSEC_PER_USEC;
	quota = tg->cfs_bandwidth.quota;
	burst = tg->cfs_bandwidth.burst;

	return tg_set_cfs_bandwidth(tg, period, quota, burst);
}
```

4、 设置进程在满负载情况下的优先级

Cgroup 提供了设置进程在满负载情况下的优先级接口（`cpu.shares`），`cpu.shares` 默认值为 `1024`，可认为进程的默认优先级就是 `1024`。当 Group A 的 `cpu.shares` 设置为 `2048`，Group B 的 `cpu.shares` 设置为 `1024` 时，在系统满负载的情况下 Group A 和 Group B 获得的 CPU 资源比例为 `2:1`，`cpu.shares` 是一个相对值，并不会和 `cfs_period_us` 一样存在突发能力，`2048:1024` 和 `2:1` 的结果是一致的。修改 `cpu.shares` 将会调用 `sched_group_set_shares()` 函数，最后会调用 `update_load_avg()` 去修改 task_group 的 load_avg，同样会直接影响 CFS 对进程的调度

##  0x04    一些细节

1、查看 pid 进程对应的 cgroup 信息

子进程默认也会集成父进程的 cgroup 限制（分组）

```BASH
[root@VM-130-44-centos ~]# cat /proc/12965/cgroup
12:pids:/collector-XXX-f75df986656263e86157c4df672b81c4
11:perf_event:/collector-XXX-f75df986656263e86157c4df672b81c4
10:cpu,cpuacct:/collector-XXX-f75df986656263e86157c4df672b81c4
9:freezer:/collector-XXX-f75df986656263e86157c4df672b81c4
8:blkio:/collector-XXX-f75df986656263e86157c4df672b81c4
7:cpuset:/collector-XXX-f75df986656263e86157c4df672b81c4
6:misc:/
5:memory:/collector-bkmonitorbeat-f75df986656263e86157c4df672b81c4
4:net_cls,net_prio:/collector-XXX-f75df986656263e86157c4df672b81c4
3:hugetlb:/collector-XXX-f75df986656263e86157c4df672b81c4
2:devices:/collector-XXX-f75df986656263e86157c4df672b81c4
1:name=systemd:/collector-XXX-f75df986656263e86157c4df672b81c4
```

2、查看对应 cgroup 的 CPU 限制

```BASH
[root@VM-130-44-centos ~]# cat /sys/fs/cgroup/cpu/XXX-bkmonitorbeat-f75df986656263e86157c4df672b81c4/cpu.cfs_period_us
100000
[root@VM-130-44-centos ~]# cat /sys/fs/cgroup/cpu/XXX-bkmonitorbeat-f75df986656263e86157c4df672b81c4/cpu.cfs_quota_us
1000000
```

3、查看对应 cgroup 的 memory 限制

```BASH
cat /sys/fs/cgroup/memory/system.slice/XXX-bkmonitorbeat-f75df986656263e86157c4df672b81c4/memory.limit_in_bytes
```

4、查看默认 cgroup 挂载分组

```BASH
[root@VM-130-44-centos ~]# mount -t cgroup
cgroup on /sys/fs/cgroup/systemd type cgroup (rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,nosuid,nodev,noexec,relatime,devices)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,nosuid,nodev,noexec,relatime,hugetlb)
cgroup on /sys/fs/cgroup/net_cls,net_prio type cgroup (rw,nosuid,nodev,noexec,relatime,net_cls,net_prio)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,nosuid,nodev,noexec,relatime,memory)
cgroup on /sys/fs/cgroup/misc type cgroup (rw,nosuid,nodev,noexec,relatime,misc)
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,nosuid,nodev,noexec,relatime,cpuset)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,nosuid,nodev,noexec,relatime,blkio)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,nosuid,nodev,noexec,relatime,freezer)
cgroup on /sys/fs/cgroup/cpu,cpuacct type cgroup (rw,nosuid,nodev,noexec,relatime,cpu,cpuacct)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,nosuid,nodev,noexec,relatime,perf_event)
cgroup on /sys/fs/cgroup/pids type cgroup (rw,nosuid,nodev,noexec,relatime,pids)
```

##  0x05    pid namespace：详解
Kernel为了实现资源隔离和虚拟化，引入了Namespace机制，即可以创建多个pid相互隔离的namespace：
-   每个namespace里都有自己的`1`号init进程，来负责启动时初始化和接收孤儿进程
-   不同pid namespace中的各进程`pid`可以相同
-   pid namespace可以层级嵌套创建，**下一级pid namespace中进程对其以上所有层的pid namespace都是可见的**，同一个task（内核里进程，线程统一都用`task_struct`表示）且在不同层级namespace中`pid`可以不同
-   Kernel中一个task的ID由两个元素可唯一确定：`[pid-namespace, pid]`

如下图所示

-   `PN1-PID2`、 `PN2-PID10`、`PN4-PID1`，它们都指向的是同一个task `1`
-   `PN1-PID3`、`PN2-PID8`、`PN5-PID1`，它们都指向的是同一个task `2`

![PIDNAMESPACE](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/linux/pid_namespace_1.png)

##  0x06    参考
-   [Linux Namespace 和 Cgroup](https://segmentfault.com/a/1190000009732550)
-   [如何在 Go 中使用 CGroup 实现进程内存控制](https://cloud.tencent.com/developer/article/2005471)
-   [探索 Linux 命名空间和控制组：实现资源隔离与管理的双重利器](https://cloud.tencent.com/developer/article/2367949)
-   [Linux PID 一网打尽](https://cloud.tencent.com/developer/article/1682890)
-   [关于 Linux Cgroup 的一些个人理解](https://smartkeyerror.com/Linux-Cgroup)