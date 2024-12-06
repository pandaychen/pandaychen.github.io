---
layout:     post
title:  Linux 内核之旅（一）：进程
subtitle:
date:       2024-10-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言

####    Operating System Kernel
操作系统内核（Operation System Kernel）本质上也是一种软件，可以看作是普通应用程序与硬件之间的一层中间层，其主要作用便是调度系统资源、控制 IO 设备、操作网络与文件系统等，并为上层应用提供便捷、抽象的应用接口

操作系统内核实际上是抽象出来的概念，本质上与用户进程一般无二，都是位于物理内存中的代码 加数据，不同之处在于**当 CPU 执行操作系统内核代码时通常运行在高权限，拥有着完全的硬件访问能力，而 CPU 在执行用户态代码时通常运行在低权限环境，只拥有部分 / 缺失硬件访问能力**

这两种不同权限的运行状态实际上是通过硬件（分级保护环）来实现的

![os](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/os_kernel_arch.png)

####    linux 的开机过程
计算机通电后，首先执行 BIOS 的自检，用于检查外围关键设备是否正常。BIOS 根据设置的启动顺序，搜索用于启动系统的驱动器，并将其 MBR 加载到内存，然后执行 MBR（BIOS 并不关心 MBR 中是什么内容，它只负责读取并执行），此时控制权被交到了 MBR，boot loader 找到和加载 Linux 内核到内存中，并将控制权移交给内核。linux 启动之后，** 内核只是存在于内存中的程序，它和其他程序一样，都是被 cpu 调度执行的 **。那么，cpu 的使用权什么情况下，会被交给内核，然后内核进行进程管理的呢？

####    CPU 特权指令（分级）
涉及到操作系统管理计算机资源的指令（敏感指令），只能被操作系统能够执行。x86 CPU 提供了 `RING0`（最高权限模式，可以执行所有指令）~`RING3`（最低权限模式，仅能执行指令集中的一部分指令）的特权分级，让 CPU 在执行操作系统代码的时候运行在 `Ring0` 模式，在执行普通应用程序代码的时候运行在 `Ring3` 模式，这样就解决了特权指令的问题

![CPU-RING0](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/Priv_rings.svg.png)
-   用户态：CPU 运行在 `ring3` + 用户进程运行环境上下文
-   内核态：CPU 运行在 `ring0` + 内核代码运行环境上下文

CPU 在不同的特权级间进行切换主要有两个途径：

-   中断与异常（interrupt & exception）：当 CPU 收到一个中断 / 异常时，会切换到 `ring0`，并根据中断描述符表索引对应的中断处理代码以执行
-   特权级相关指令：当 CPU 运行这些指令时会发生运行状态的改变，例如 `iret` 指令（`ring0`->`ring3`）或是 `sysenter` 指令（`ring3`->`ring0`）

基于特权级切换的方式，现代操作系统的开发者包装出了系统调用（syscall），作为由用户态切换到内核态的入口，从而执行内核代码来完成用户进程所需的一些功能。当用户进程想要请求更高权限的服务时，便需要通过由系统提供的应用接口，使用系统调用以陷入内核态，再由操作系统完成请求

####    内核地址空间
除了指令增加特权分级以外，在内存的访问也得加上特权级。由于 x86 架构的 CPU 是基于分段 + 分页式相结合的内存管理方式，通过给不同的内存段限定了不同的访问模式，并把它记录到了段的描述符中，在访问内存的时候，CPU 就会拿当前段寄存器中标示的权限和要访问的目标内存所在段段访问权限进行对比，符合要求才能访问，否则会抛出异常

1.  操作系统（Kernel）在内存中设定一块区域，将 OS Kernel 代码放在这块区域中，并设置了访问权限，非 `Ring0` 权限禁止访问
2.  操作系统又将该内存区域映射到了每一个进程的虚拟地址空间中，这样所有进程都可以看到该进程地址空间被 OS 占用，无法访问及修改

从而衍生出下面的若干概念：
-   操作系统设定的这段内存区域，一般称为内核地址空间
-   位于内核地址空间中的代码叫做操作系统的内核代码
-   位于应用程序代码所活动的区域叫做用户地址空间
-   CPU 执行内核代码的模式称为内核态
-   CPU 执行用户程序时的模式称为用户态

CPU 执行代码的过程，就是不断游走于用户态和内核态的过程（切换）

####    用户态与内核态的切换方式
CPU 提供了专门的入口，用来从用户态进入内核态（CPU 使用权移交）

- 中断：当硬件设备有消息来了之后，会通过中断通知 CPU，比如（移动鼠标，敲下键盘，收到了一个数据包等），当中断发生时，CPU 会将当前执行的上下文保存到栈中，转入内核执行中断处理程序。通过中断进入内核（入口是记录在中断描述符表 IDT）
- 异常：当 CPU 执行过程中发现一些异常情况，比如执行除法指令的除数是 0，访问的内存地址无效，或者访问的内存地址属于特权页面等这些情况，CPU 都会触发异常；同中断处理类似，遇到异常时，CPU 也会将执行的上下文保存在栈中，转入内核执行中断处理程序
-   系统调用：在系统编程中调用操作系统提供的 API 函数，比如文件操作、内存操作、网络操作等等，这些函数都是操作系统封装出来的应用程序编程 API，真正的底层实现是位于内核中的系统调用。应用层上的 API 通过 CPU 专门的指令进入内核来完成对应的功能

####    CPU 中断
中断是当系统中出现了一个必须由 CPU 立即处理的情况时，CPU 需要暂停当前正在执行的程序，转而处理这个新的情况，分为硬件中断和软中断

-   硬件中断：硬件中断是一个异步信号，它是由与系统相连的外设（如网卡，硬盘，键盘等）产生的。每个设备或设备集都有自己的 IRQ（中断请求），cpu 根据 IRQ 将中断请求分发给相应的中断处理程序。比如当网卡收到一个数据包的时候，就会发出一个中断请求。需要注意的是硬件中断是可屏蔽的。当发生硬件中断时，cpu 会暂停当前程序，转而执行中断代码，中断代码本身也可以被其他硬件中断中断
-   软中断：软中断是由正在运行的程序发出的，不会中断 cpu。软中断是一种需要内核为当前正在运行的进程做一些事情（通常是 I/O）的请求
-   时钟中断：linux 的 `0` 号中断即时钟中断，操作系统利用时钟中断，维持系统时间更新 cpu 计数，也就是调用 `scheduler_tick` 递减进程的时间片，若进程的时间片递减到 `0`，则进程被调度出去而放弃 CPU 的使用权。本质上说，时钟中断只是一个周期性的信号，完全是硬件行为，该信号触发 cpu 执行一个中断服务程序（ISR）

##  0x01    进程的基础概念

本节主要讨论下面几个问题

-   `tid`/`pid`/`ppid`/`tgid`/`pgid`/`seesion id`在内核的表示
-   pid namesapce：对pid的影响
-   如何创建一个pid namespace以及如何进入一个已存在的pid namespace
-   内核结构`task_struct`与这一系列id之间的关联

####    基础概念
-   从Kernel的角度来看，不会区分进程（PID）、线程（TID），最终都会对应到内核对象`task_struct`上
-   `TGID`：若进程以 `CLONE_THREAD` 标志调用 `clone` 方法，创建与该进程共享资源的线程。线程有独立的`task_struct`，but其 `files_struct`、`fs_struct` 、`sighand_struct`、`signal_struct`和`mm_struct` 成员仅仅是对进程相应数据结构的引用；由进程创建的所有线程都有相同的线程组ID（即TGID），线程有自己的 PID，它的TGID 就是进程的主线程的 PID；如果进程没有使用线程，则其 PID 和 TGID 相同，此外，当 `task_struct` 代表一个线程时，`task_struct->group_leader` 指向主线程的 `task_struct`
-   `PGID`： 如果 shell 具有作业管理能力，则它所创建的相关进程构成一个进程组，同一进程组的进程都有相同的 `PGID`（如用管道连接的进程包含在同一个进程组中），进程组简化了向组的所有成员发送信号的操作，信号可以发送给组内的所有进程，这使得作业控制变得简单。当 `task_struct` 代表一个进程，且该进程属于某一个进程组，则 `task_struct->group_leader` 指向组长进程的 `task_struct`。PGID 保存在`task_struct->signal->pids[PIDTYPE_PGID].pid`中
-   `SID`：一系列进程组的组合，一般看到的 tty(比如键盘、屏幕的 `tty1~tty7`，或者网络连接的虚拟 tty 即 pty)，一个 tty 对应一个 session。在当前 tty 中创建的所有进程都共享一个 sid(即 leader 的 pid)
-   `PID`：是kernel内部对进程的一个标识符，用来唯一标识一个进程（task）、进程组（process group）、会话（session）。PID 及其对应进程存储在一个哈希表中，方便依据 PID 快速访问进程 `task_struct`

![basic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/sid_struct.png)

####    进程描述符基础结构：task_struct

本文只讨论内核态的进（线）程，本质上Linux 内核中进程/线程都是用 [`task_struct`](https://elixir.free-electrons.com/linux/v4.11.6/source/include/linux/sched.h#L483)（任务） 来表示的，结构如下：**在用户态调用`getpid`实际上返回的是`task_struct`的`tgid`字段**，而`pid`每个线程都是不同的（都一个进程生成的不同线程而言），`task_struct`也是CPU调度的实体

```CPP
//file:include/linux/sched.h
struct task_struct {
 //2.1 进程状态 
 volatile long state;

 //2.2 进程线程的pid
 pid_t pid;         //本质上int
 pid_t tgid;        //本质上int

 //2.3 进程树关系：父进程、子进程、兄弟进程
 struct task_struct __rcu *parent;
 struct list_head children; 
 struct list_head sibling;
 struct task_struct *group_leader; 

 //2.4 进程调度优先级
 int prio, static_prio, normal_prio;
 unsigned int rt_priority;

 //2.5 进程地址空间
 struct mm_struct *mm, *active_mm;

 //2.6 进程文件系统信息（当前目录等）
 struct fs_struct *fs;

 //2.7 进程打开的文件信息
 struct files_struct *files;

 //2.8 namespaces 
 struct nsproxy *nsproxy;

 // relation
 /* PID/PID hash table linkage. */
 struct pid_link			pids[PIDTYPE_MAX];

 //
 struct task_struct   *group_leader;

 //namespace相关
 struct nsproxy       *nsproxy;
}
```

`task_struct` 是 linux 内核中最重要的概念之一，与 `pid` 有关的成员结构定义如下：

```CPP
struct task_struct {
	//···
	pid_t                 pid;
	pid_t                 tgid;
	struct task_struct   *group_leader;
	struct pid_link       pids[PIDTYPE_MAX];
	struct nsproxy       *nsproxy;
	//···
};

struct pid_link
{
	struct hlist_node node;
	struct pid *pid;
};

struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net 	     *net_ns;
    struct cgroup_namespace *cgroup_ns;
};
```

-   `pid`：内核进程的 id，使用 `fork`/`clone` 系统调用时产生的进程均会由内核分配一个新的唯一的PID值
-   `tgid`：线程组 id，在一个进程中，如果以 `CLONE_THREAD` 标志来调用 `clone` 建立的进程就是该进程的一个线程，它们处于一个线程组。处于相同的线程组中的所有进程都有相同的 `TGID`；线程组组长的 `TGID` 与其 `PID` 相同；一个进程没有使用线程，则其 `TGID` 与 `PID` 也相同
-   `group_leader`：除了在多线程的模式下指向主线程外， 当一些进程组成一个群组时（`PIDTYPE_PGID`）， 该成员指向该群组的leader
-   `pids[PIDTYPE_MAX]`：指向了和该 `task_struct` 相关的 `pid` 结构体
-   `nsproxy`：指向namespace相关的结构，与其它命名空间不同。此处 `pid_ns_for_children` 指向该进程的子进程会使用的 `pid_namespace`，该进程本身所属的 `pid_namespace` 可以通过 `task_active_pid_ns` 方法获得；`nsproxy` 被所有共享命名空间的 `task_struct` 共享，随命名空间的复制而复制（namespace原理）

![task-struct-basic.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/task-struct-basic.png)

####    PID：进程ID
内核结构[`pid`](https://elixir.free-electrons.com/linux/v4.11.6/source/include/linux/pid.h#L57)定义如下，在`task_struct`中，通过`task->pids[PIDTYPE_PID].pid`可以定位到（`task_pid`[函数](https://elixir.free-electrons.com/linux/v4.11.6/source/include/linux/sched.h#L1056)），**虽然名字是pid，不过实际上该结构抽象了不仅是一个thread ID或者process ID，实际上还包括了进程组ID和session ID**

`pid`重要成员如下：

-   `count`：由于多个`task struct`会共享`pid`（例如一个session中的所有的`task_struct`都会指向同一个表示该session ID的`struct pid`数据对象），表示该数据对象的引用计数
-   `level`：该 `pid` 在 `pid_namespace` 中所处层级，当 `level=0` 时表示是 global namespace（最高层）
-   `tasks[i]`：指向 PID 对应的 `task_struct`。`PIDTYPE_MAX` 是 `pid` 的类型数（枚举）。一个或多个进程可以组成一个进程组，进程组ID（PGID）为进程组领导进程 （process group leader）的PID；一个或多个进程组可以组成一个会话，会话ID（SID）为会话领导进程（session leader）的PID
-   `rcu`：用于保证数据同步
-   `numbers[1]`：是一个可扩展 `upid` 结构体。 一个 PID 可以属于不同的 namespace ， `numbers[0]` 表示 global namespace，`numbers[i]` 表示第 `i` 层 namespace，`i` 越大所在层级越低，下文详细说明

前文[Linux Namespace && Cgroup](https://pandaychen.github.io/2023/10/06/A-CGROUP-ANALYSIS/)描述过pid namespace hierarchy基础，任何一个系统分配的PID都是隶属于某一个namespace的，而这个namespace又是位于整个pid namespace hierarchy的某个层次上，`pid->level`指明了该PID所属的namespace的level。由于pid对其parent pid namespace也是可见的，因此`level`值其实也就表示了这个pid对象在多少个pid namespace中可见。在多少个pid namespace中可见，就会有多少个（pid namespace，pid number）对，`numbers`就是这样的一个数组，每个成员都指向了各个`level`上的pid number


`upid` 结构的成员如下（**在Kernel中一个task的ID由两个元素唯一确定 `[pid namespace, processid id]`，在内核中用upid表示**）
-   `nr`：是`pid`的值， 即 `task_struct` 中 `pid_t pid` 域的值（重要）
-   `ns`：指向该 `pid` 所处的 `namespace`
-   `pid_chain`： 是 `pid_hash` 哈希表节点。linux内核将所有进程的`upid`都存放在一个哈希表（`pid_hash`）中，以方便查找和统一管理。通过 `pid_chain` 能够找到该 `upid` 所在 `pid_hash` 中的位置

`pid_chain`这个概念是有点绕的，下文详细说明

```CPP
struct pid
{
	atomic_t count;
	unsigned int level;
	/* lists of tasks that use this pid */
	struct hlist_head tasks[PIDTYPE_MAX];
	struct rcu_head rcu;
	struct upid numbers[1];
};

struct upid {
	  /* Try to keep pid_chain in the same cacheline as nr for find_vpid */
	  int nr;
	  struct pid_namespace *ns; // 所属的pid namespace
	  struct hlist_node pid_chain;
};

enum pid_type
{
	PIDTYPE_PID,
	PIDTYPE_PGID,
	PIDTYPE_SID,
	PIDTYPE_MAX
};
```

关于`pid.numbers[1]`这个成员，本质上一个柔性数组，每次在分配struct pid时，`numbers`会被扩展到`level`个元素，它用来容纳在每一层pid namespace中的 id（`upid`）

####    pid_namespace：进程命名空间
`pid` 命名空间 `pid_namespace` 的[定义](https://elixir.free-electrons.com/linux/v4.11.6/source/include/linux/pid_namespace.h#L30)如下，关联`upid`的`ns`成员：

```CPP
struct pid_namespace {
	struct kref kref;
	struct pidmap pidmap[PIDMAP_ENTRIES];
	struct rcu_head rcu;
	int last_pid;
	unsigned int nr_hashed;
	struct task_struct *child_reaper;
	struct kmem_cache *pid_cachep;
	unsigned int level;
	struct pid_namespace *parent;
	struct user_namespace *user_ns;
	struct ucounts *ucounts;
	struct work_struct proc_work;
	kgid_t pid_gid;
	int hide_pid;
	int reboot;	/* group exit code if this pidns was rebooted */
	struct ns_common ns;
};

struct pidmap {
       atomic_t nr_free;
       void *page;
};

#define BITS_PER_PAGE		(PAGE_SIZE * 8)
#define BITS_PER_PAGE_MASK	(BITS_PER_PAGE-1)
#define PIDMAP_ENTRIES		((PID_MAX_LIMIT+BITS_PER_PAGE-1)/BITS_PER_PAGE)
  
// include/linux/threads.h
#define PID_MAX_DEFAULT (CONFIG_BASE_SMALL ? 0x1000 : 0x8000)
/*
 * A maximum of 4 million PIDs should be enough for a while.
 * [NOTE: PID/TIDs are limited to 2^29 ~= 500+ million, see futex.h.]
 */
#define PID_MAX_LIMIT (CONFIG_BASE_SMALL ? PAGE_SIZE * 8 : \
	(sizeof(long) > 4 ? 4 * 1024 * 1024 : PID_MAX_DEFAULT))
```

-   `kref`： 表示指向 `pid_namespace` 的个数
-   `pidmap` 结构体表示分配pid的bitmap，`pidmap[PIDMAP_ENTRIES]` 域存储了该 `pid_namespace` 下 pid 已分配情况
-   `rcu`：同样用于保证数据同步
-	`last_pid`：最后一个已分配的 pid
-	`nr_hashed`：统计该命名空间已分配PID个数
-	`child_reaper`：指向的是一个进程，该进程的作用是当子进程结束时为其收尸（回收空间），global namespace 中`child_reaper` 指向 `init_task`
-	`pid_cachep`：域指向分配 pid 的 slab 的地址
-	`level`：该命名空间所处层级（重要）
-	`parent`：指向该命名空间的父命名空间

`pidmap` 结构体定义如下：
-	`nr_free`：表示还能分配的 pid 的数量
-	`page`：指向的是存放 pid 的物理页

####    pid之间的关系（重要）

-   如何快速地根据进程的 `task_struct`、ID 类型、命名空间找到 PID ？
-   如何快速地根据 PID、pid namespace、ID 类型找到对应进程的 `task_struct` ？
-   如何快速地给新进程在可见的命名空间内分配一个唯一的 PID ？

下图描述了这种关系，在level `2` 的某个pid namespace上新建了一个进程，分配给它的 pid 为`45`，映射到 level `1` 的pid namespace，分配给它的 pid 为 `134`；再映射到 level `0` 的pid namespace，分配给它的 pid 为`289`（注意`numbers`这个柔性数组），此外，图中只标识了`level=2,pid=45`的pid结构

![relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_namespace.png)

####    查询PID：分类讨论
根据上图，来看下相关的函数

1、获取与 `task_struct` 相关的 pid 结构体实例：`pid`/`tgid`/`pgrp`/`session`

```cpp
static inline struct pid *task_pid(struct task_struct *task)
{
 	return task->pids[PIDTYPE_PID].pid;
}
    
static inline struct pid *task_tgid(struct task_struct *task)
{
 	return task->group_leader->pids[PIDTYPE_PID].pid;
}
    
static inline struct pid *task_pgrp(struct task_struct *task)
{
 	return task->group_leader->pids[PIDTYPE_PGID].pid;
}
    
static inline struct pid *task_session(struct task_struct *task)
{
 	return task->group_leader->pids[PIDTYPE_SID].pid;
}
```

2、获取与 `task_struct` 相关的 PID 命名空间

```cpp
struct pid_namespace *task_active_pid_ns(struct task_struct *tsk)
{
     return ns_of_pid(task_pid(tsk));
}

static inline struct pid_namespace *ns_of_pid(struct pid *pid)
{
    struct pid_namespace *ns = NULL;
    if (pid)
        ns = pid->numbers[pid->level].ns;
    return ns;
}
```

3、获取 `pid` 实例中的 PID

参数：
-   `pid`：指向 struct pid 结构体的指针，代表了一个进程的PID
-   `ns`：指向 struct pid_namespace 结构体的指针，代表了一个PID命名空间

```CPP
pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)
{
 	struct upid *upid;
 	pid_t nr = 0;
    
    //由于 PID 命名空间的层次性，父命名空间能看到子命名空间的内容，反之则不能
    // 确保当前命名空间的 level 小于等于产生局部 PID 的命名空间的 level
 	if (pid && ns->level <= pid->level) {
 		upid = &pid->numbers[ns->level];
 		if (upid->ns == ns){
            // 检查传入的参数ns地址是否与upid的level指向的ns地址一致
 			nr = upid->nr;
        }
 	}

    //违反规则时返回0
 	return nr;
}
```


4、直接获取初始命名空间

```CPP
static inline pid_t pid_nr(struct pid *pid)
{
 	pid_t nr = 0;
 	if (pid)
 		nr = pid->numbers[0].nr;
 	return nr;
}
```

5、当前命名空间对应 PID

```CPP
pid_t pid_vnr(struct pid *pid)
{
 	return pid_nr_ns(pid, task_active_pid_ns(current));
}
```

####    分配PID
1、为新进程 `task_struct` 分配 `pid`

```CPP
 struct pid *alloc_pid(struct pid_namespace *ns)
 {
 	struct pid *pid;
 	enum pid_type type;
 	int i, nr;
 	struct pid_namespace *tmp;
 	struct upid *upid;
 	int retval = -ENOMEM;
    
     // 从命名空间分配一个 pid 结构体
 	pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
 	if (!pid)
 		return ERR_PTR(retval);
    
     // 初始化进程在各级命名空间的 PID，直到全局命名空间（level 为0）为止
 	tmp = ns;
 	pid->level = ns->level;
 	for (i = ns->level; i >= 0; i--) {
 		nr = alloc_pidmap(tmp);  //分配一个局部PID
 		if (nr < 0) {
 			retval = nr;
 			goto out_free;
 		}
    
 		pid->numbers[i].nr = nr;
 		pid->numbers[i].ns = tmp;
 		tmp = tmp->parent;
 	}
    
     // 若为命名空间的初始进程
 	if (unlikely(is_child_reaper(pid))) {
 		// 0
 		if (pid_ns_prepare_proc(ns))
 			goto out_free;
 	}
    
 	get_pid_ns(ns);
 	atomic_set(&pid->count, 1);
 	for (type = 0; type < PIDTYPE_MAX; ++type)
 		INIT_HLIST_HEAD(&pid->tasks[type]); // // 初始化 pid->task[] 结构体，值为NULL
    
 	upid = pid->numbers + ns->level;
 	spin_lock_irq(&pidmap_lock);
 	if (!(ns->nr_hashed & PIDNS_HASH_ADDING))
 		goto out_unlock;
 	for ( ; upid >= pid->numbers; --upid) {
 		// 将每个命名空间经过哈希之后加入到散列表中
 		hlist_add_head_rcu(&upid->pid_chain,
 				&pid_hash[pid_hashfn(upid->nr, upid->ns)]);
 		upid->ns->nr_hashed++;
 	}
 	spin_unlock_irq(&pidmap_lock);
    
 	return pid;
    
 out_unlock:
 	spin_unlock_irq(&pidmap_lock);
 	put_pid_ns(ns);
    
 out_free:
 	while (++i <= ns->level)
 		free_pidmap(pid->numbers + i);
    
 	kmem_cache_free(ns->pid_cachep, pid);
 	return ERR_PTR(retval);
 }
```

2、从指定命名空间中分配唯一PID

```CPP
 static int alloc_pidmap(struct pid_namespace *pid_ns)
 {
 	int i, offset, max_scan, pid, last = pid_ns->last_pid;
 	struct pidmap *map;
    
 	pid = last + 1;
 	// 默认最大值在 include/linux/threads.h 中定义为 (CONFIG_BASE_SMALL ? 0x1000 : 0x8000)
 	if (pid >= pid_max)  
 		pid = RESERVED_PIDS;  // RESERVED_PIDS = 300
 	offset = pid & BITS_PER_PAGE_MASK;
 	map = &pid_ns->pidmap[pid/BITS_PER_PAGE];
 	/*
 	 * If last_pid points into the middle of the map->page we
 	 * want to scan this bitmap block twice, the second time
 	 * we start with offset == 0 (or RESERVED_PIDS).
 	 */
 	max_scan = DIV_ROUND_UP(pid_max, BITS_PER_PAGE) - !offset;
 	for (i = 0; i <= max_scan; ++i) {
 		if (unlikely(!map->page)) {
 			void *page = kzalloc(PAGE_SIZE, GFP_KERNEL);
 			/*
 			 * Free the page if someone raced with us
 			 * installing it:
 			 */
 			spin_lock_irq(&pidmap_lock);
 			if (!map->page) {
 				map->page = page;
 				page = NULL;
 			}
 			spin_unlock_irq(&pidmap_lock);
 			kfree(page);
 			if (unlikely(!map->page))
 				return -ENOMEM;
 		}
 		if (likely(atomic_read(&map->nr_free))) {
 			for ( ; ; ) {
 				if (!test_and_set_bit(offset, map->page)) {
 					atomic_dec(&map->nr_free);
 					set_last_pid(pid_ns, last, pid);
 					return pid;
 				}
 				offset = find_next_offset(map, offset);
 				if (offset >= BITS_PER_PAGE)
 					break;
 				pid = mk_pid(pid_ns, map, offset);
 				if (pid >= pid_max)
 					break;
 			}
 		}
 		if (map < &pid_ns->pidmap[(pid_max-1)/BITS_PER_PAGE]) {
 			++map;
 			offset = 0;
 		} else {
 			map = &pid_ns->pidmap[0];
 			offset = RESERVED_PIDS;
 			if (unlikely(last == offset))
 				break;
 		}
 		pid = mk_pid(pid_ns, map, offset);
 	}
 	return -EAGAIN;
 }
```

3、回收PID

```CPP
 static void free_pidmap(struct upid *upid)
 {
 	int nr = upid->nr;
 	struct pidmap *map = upid->ns->pidmap + nr / BITS_PER_PAGE;
 	int offset = nr & BITS_PER_PAGE_MASK;
    
 	clear_bit(offset, map->page);
 	atomic_inc(&map->nr_free);
 }
```

####    查询task_struct

1、获得 pid 实体（`struct pid`）

大致过程是，根据 PID（参数`nr`） 以及指定命名空间（参数`ns`）计算在 `pid_hash` 数组中的索引，然后遍历散列表找到所要的 `upid`， 再根据内核的 `container_of` 机制找到 pid 实例

```CPP
 struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
 {
 	  struct upid *pnr;
    
 	  hlist_for_each_entry_rcu(pnr,
 			  &pid_hash[pid_hashfn(nr, ns)], pid_chain)
 		  if (pnr->nr == nr && pnr->ns == ns)
 	  		  return container_of(pnr, struct pid,
 					  numbers[ns->level]);
       
 	       return NULL;
 }
```

2、根据当前命名空间下的局部 PID 获取对应的 pid实例：

```CPP
 struct pid *find_vpid(int nr)
 {
   	return find_pid_ns(nr, task_active_pid_ns(current));
 }
```

####    小结
tid/pid/ppid/tgid/pgid/seesion id小结：

| ID | 解释 | task_struct 中的对应变量 | 系统调用|
| :-----:| :----: | :----: |:----: |
| PID （Process ID） | 实际上是线程 ID，内核中进程、线程都使用 `task_struct` 结构表示 | `task_struct->pid` |`pid_t gettid(void)` |
| TGID (Thread Group ID) | 线程组 ID，即线程组组长的 PID，真正的进程 ID，如果进程只有一个线程则他的 PID 和 TGID 相同 | `task_struct->tgid` | pid_t getpid(void) |
|PGID （Process Group ID）|进程组 ID，多个进程可以组合为进程组，方便向所有成员发送信号，进程组组长的 PID 即为PGID|`task_struct->signal->__pgrp`|pid_t getpgrp(void)|
|SID（Session ID）|会话 ID，多个进程组可以组合为会话，会话的组长PGID 即为 SID|`task_struct->signal->__session`|pid_t getsid(pid_t pid);|
|PPID （Parent Process ID）	|父进程 ID|task_struct->parent->pid|pid_t getppid(void)|

##  0x02    Linux进程虚拟地址空间
`task_struct`成员，内存描述符 `mm_struct`（memory descriptor）表示了整个进程的虚拟地址空间部分。进程运行时，在用户态其所需要的代码、全局变量以及 mmap 内存映射等全部都是通过 `mm_struct` 来进行内存查找和寻址的， `mm_struct` 关联的地址空间、页表、物理内存的关系如下图：

```CPP
struct mm_struct {
 struct vm_area_struct * mmap;  /* list of VMAs */
 struct rb_root mm_rb;

 unsigned long mmap_base;  /* base of mmap area */
 unsigned long task_size;  /* size of task vm space */
 unsigned long start_code, end_code, start_data, end_data;
 unsigned long start_brk, brk, start_stack;
 unsigned long arg_start, arg_end, env_start, env_end;
}
```

![mm_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/task-struct-4-mm-modify.png)

-	**在内核内存区域，可以通过直接计算得出物理内存地址，并不需要复杂的页表计算**。而且最重要的是所有内核进程、以及用户进程的内核态，这部分内存都是共享的
-	`mm_struct`表示的是虚拟地址空间，而对于内核线程来说，是没有用户态的虚拟地址空间的，其value为`NULL`

##  0x03    进程权限

####    进程权限凭证（credential）
在内核结构`task_struct`中有下面的字段标识了进程权限凭证：
```CPP
struct task_struct {
    // ...
    /* Process credentials: */

    /* Tracer's credentials at attach: */
    const struct cred __rcu        *ptracer_cred;

    /* Objective and real subjective task credentials (COW): */
    const struct cred __rcu        *real_cred;

    /* Effective (overridable) subjective task credentials (COW): */
    const struct cred __rcu        *cred;
    //...
}
```

其中，结构体 `cred` 用以管理一个进程的权限，如下所示。一个 `cred` 结构体中记载了一个进程`4`种不同的用户 ID，在通常情况下这几个 ID 应当都是相同的，以`*uid`为例：
-   真实用户 ID（real UID）：标识一个进程启动时的用户 ID
-   保存用户 ID（saved UID）：标识一个进程最初的有效用户 ID
-   有效用户 ID（effective UID）：标识一个进程正在运行时所属的用户 ID，一个进程在运行途中是可以改变自己所属用户的，因而权限机制也是通过有效用户 ID 进行认证的，内核通过 `euid` 来进行特权判断；为了防止用户一直使用高权限，当任务完成之后，`euid` 会与 `suid` 进行交换，恢复进程的有效权限
-   文件系统用户 ID（UID for VFS ops）：标识一个进程创建文件时进行标识的用户 ID

```CPP
/*
 * The security context of a task
 *
 * The parts of the context break down into two categories:
 *
 *  (1) The objective context of a task.  These parts are used when some other
 *  task is attempting to affect this one.
 *
 *  (2) The subjective context.  These details are used when the task is acting
 *  upon another object, be that a file, a task, a key or whatever.
 *
 * Note that some members of this structure belong to both categories - the
 * LSM security pointer for instance.
 *
 * A task has two security pointers.  task->real_cred points to the objective
 * context that defines that task's actual details.  The objective part of this
 * context is used whenever that task is acted upon.
 *
 * task->cred points to the subjective context that defines the details of how
 * that task is going to act upon another object.  This may be overridden
 * temporarily to point to another security context, but normally points to the
 * same context as task->real_cred.
 */
struct cred {
    atomic_long_t   usage;
    kuid_t      uid;        /* real UID of the task */
    kgid_t      gid;        /* real GID of the task */
    kuid_t      suid;       /* saved UID of the task */
    kgid_t      sgid;       /* saved GID of the task */
    kuid_t      euid;       /* effective UID of the task */
    kgid_t      egid;       /* effective GID of the task */
    kuid_t      fsuid;      /* UID for VFS ops */
    kgid_t      fsgid;      /* GID for VFS ops */
    unsigned    securebits; /* SUID-less security management */
    kernel_cap_t    cap_inheritable; /* caps our children can inherit */
    kernel_cap_t    cap_permitted;  /* caps we're permitted */
    kernel_cap_t    cap_effective;  /* caps we can actually use */
    kernel_cap_t    cap_bset;   /* capability bounding set */
    kernel_cap_t    cap_ambient;    /* Ambient capability set */
#ifdef CONFIG_KEYS
    unsigned char   jit_keyring;    /* default keyring to attach requested
                     * keys to */
    struct key  *session_keyring; /* keyring inherited over fork */
    struct key  *process_keyring; /* keyring private to this process */
    struct key  *thread_keyring; /* keyring private to this thread */
    struct key  *request_key_auth; /* assumed request_key authority */
#endif
#ifdef CONFIG_SECURITY
    void        *security;  /* LSM security */
#endif
    struct user_struct *user;   /* real user ID subscription */
    struct user_namespace *user_ns; /* user_ns the caps and keyrings are relative to. */
    struct ucounts *ucounts;
    struct group_info *group_info;  /* supplementary groups for euid/fsgid */
    /* RCU deletion */
    union {
        int non_rcu;            /* Can we skip RCU deletion? */
        struct rcu_head rcu;        /* RCU deletion hook */
    };
} __randomize_layout;
```

##  0x04 关联文件系统
`task_struct`亦关联了进程文件系统信息（如：当前目录等）以及当前进程打开文件的信息

```CPP
struct task_struct{
	// ...
    struct fs_struct *fs; 　　　　//文件系统信息，fs保存了进程本身与VFS（虚拟文件系统）的关系信息
    struct files_struct *files;　//打开文件信息（记录了所有process打开文件的句柄数组）
	// ...
}
```

![relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/task_struct.png)

#### 进程文件系统信息（fs_struct）

```CPP
//file:include/linux/fs_struct.h
struct fs_struct {
 //...
 struct path root, pwd;
};

//file:include/linux/path.h
struct path {
 struct vfsmount *mnt;
 struct dentry *dentry;
};
```

![fs_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/task-struct-5-files.png)

在 `fs_struct` 中包含了两个 `path` 对象，而每个 `path` 中都指向了一个 `struct dentry`。在 Linux `内核中，denty` 结构是对一个目录项的描述。以 `pwd` 为例，该指针指向的是进程当前目录所处的 `denty` 目录项。如在 shell 进程中执行 `pwd`命令，或用户进程查找当前目录下的配置文件的时候，都是通过访问 `pwd` 这个对象，进而找到当前目录的 `denty`

####  进程打开的文件信息（files）
每个进程用一个 `files_struct` 结构（用户打开文件表）来记录文件描述符的使用情况

```CPP
//file:include/linux/fdtable.h
struct files_struct {
  //......
 //下一个要分配的文件句柄号
 int next_fd; 

 //fdtable
 struct fdtable __rcu *fdt;
}

struct fdtable {
 //当前的文件数组
 struct file __rcu **fd;
 //......
};
```

![files_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/task-struct-5-files-1.png)

如上图，在 `files_struct` 中，最重要的是在 `fdtable` 中包含的 `file **fd` 这个二维数组，此数组的下标就是文件描述符，其中 `0`、`1`、`2` 三个描述符总是默认分配给标准输入、标准输出和标准错误。此外，其他数组元素中记录了当前进程打开的每一个文件的指针。这个文件是 Linux 中抽象的文件，可能是真的磁盘上的文件，也可能是一个 socket（考虑下管道pipe的场景）

##	0x05	namespaces
`task_struct`中成员`struct nsproxy *nsproxy;`指向命名空间namespaces的指针（通过 namespace 可以让一些进程只能看到与自己相关的一部分资源，而另外一些进程也只能看到与它们自己相关的资源，这两类进程无法感知对方的存在）。实现方式是把一个或多个进程的相关资源指定在同一个 namespace 中，而进程究竟是属于哪个 namespace 由 `*nsproxy` 指针表明了归属关系

```CPP
struct nsproxy {
 atomic_t count;
 struct uts_namespace *uts_ns;
 struct ipc_namespace *ipc_ns;
 struct mnt_namespace *mnt_ns;
 struct pid_namespace *pid_ns;
 struct net       *net_ns;
};
```

![task_struct_2_namespaces](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/task-struct-6-namespace.png)

##	0x06	进程线程状态及状态图


##  0x0 总结

####     内核态的意义
CPU 为了进行指令权限管控，引入了特权级的概念，CPU 工作在不同的特权级下能够执行的指令和可访问的内存区域是不一样的。计算机上电启动之处，CPU 运行在高特权级下，操作系统（Linux 内核）率先获得了执行权限，在内存设置一块固定区域并将自己的程序代码（内核代码）放了进去，并设定了这一部分内存只有高特权级才能访问。随后，操作系统在创建进程的时候，都会把自己所在的这块内存区域映射到每一个进程地址空间中，这样所有进程都能看到自己的进程空间中被映射的内核的区域，这一块区域是无法直接访问的。通常进入内核态是指：当中断、异常、系统调用等情况发生的时候，CPU 切换工作模式到高特权级模式 `Ring0`，并转而执行位于内核地址空间处的代码

##  0x0  参考
-   [Linux 进程是如何创建出来的？](https://cloud.tencent.com/developer/article/2187989)
-   [Linux 内核进程管理](http://timd.cn/kernel-process-management/)
-   <<深入理解 Linux 进程与内存>>
-   [CPU 进入内核，是什么意思？](https://mp.weixin.qq.com/s?__biz=MzkxNjE3NTAyNQ==&mid=2247485492&idx=1&sn=d3196f90d31ff19060e0105eba66723a&chksm=c152a9eaf62520fc87d306e42766ede75fc0457718adcc98acb1e15cf3497d1ef2e71619be5e#rd)
-   [进程ID及进程间的关系](https://cloud.tencent.com/developer/article/2363228)
-   [Linux PID 一网打尽](https://cloud.tencent.com/developer/article/1682890)
-   [Linux开启动过程详解](https://handerfly.github.io/linux/2019/04/02/Linux%E5%BC%80%E5%90%AF%E5%8A%A8%E8%BF%87%E7%A8%8B%E8%AF%A6%E8%A7%A3/)
-   [linux内核PID管理](https://carecraft.github.io/basictheory/2017/03/linux-pid-manage/)
-   [Linux 内核进程管理之进程ID](https://www.cnblogs.com/hazir/p/linux_kernel_pid.html)
-   [Pid Namespace 详解](https://tinylab.org/pid-namespace/)
-   [linuxkerneltravel](https://github.com/linuxkerneltravel)
-   [基础知识](https://ctf-wiki.org/pwn/linux/kernel-mode/basic-knowledge/)