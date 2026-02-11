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

本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

####    Operating System Kernel
操作系统内核（Operation System Kernel）本质上也是一种软件，可以看作是普通应用程序与硬件之间的一层中间层，其主要作用便是调度系统资源、控制 IO 设备、操作网络与文件系统等，并为上层应用提供便捷、抽象的应用接口

操作系统内核实际上是抽象出来的概念，本质上与用户进程一般无二，都是位于物理内存中的代码 加数据，不同之处在于**当 CPU 执行操作系统内核代码时通常运行在高权限，拥有着完全的硬件访问能力，而 CPU 在执行用户态代码时通常运行在低权限环境，只拥有部分 / 缺失硬件访问能力**

这两种不同权限的运行状态实际上是通过硬件（分级保护环）来实现的

![os](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/os_kernel_arch.png)

####    linux 的开机过程
计算机通电后，首先执行 BIOS 的自检，用于检查外围关键设备是否正常。BIOS 根据设置的启动顺序，搜索用于启动系统的驱动器，并将其 MBR 加载到内存，然后执行 MBR（BIOS 并不关心 MBR 中是什么内容，它只负责读取并执行），此时控制权被交到了 MBR，boot loader 找到和加载 Linux 内核到内存中，并将控制权移交给内核。linux 启动之后，**内核只是存在于内存中的程序，它和其他程序一样，都是被 cpu 调度执行的**。那么，cpu 的使用权什么情况下，会被交给内核，然后内核进行进程管理的呢？

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
-	Linux中的线程（即轻量级的进程）
	-	从内核角度，以线程为单位，一个线程对应一个`task_struct`，对应一个`pid`
	-	从用户角度，以进程为单位，一个进程是多个线程组成的线程组，对应一个`tgid`（thread group id）
-   从Kernel的角度来看，不会区分进程（PID）、线程（TID），最终都会对应到内核对象`task_struct`，这里的**线程等同于轻量级进程**
-   `TGID`：若进程以 `CLONE_THREAD` 标志调用 `clone` 方法，创建与该进程共享资源的线程。线程有独立的`task_struct`，but其 `files_struct`、`fs_struct` 、`sighand_struct`、`signal_struct`和`mm_struct` 成员仅仅是对进程相应数据结构的引用；由进程创建的所有线程都有相同的线程组ID（即TGID），线程有自己的 PID，它的TGID 就是进程的主线程的 PID；如果进程没有使用线程，则其 PID 和 TGID 相同，此外，当 `task_struct` 代表一个线程时，`task_struct->group_leader` 指向主线程的 `task_struct`
-   `PGID`： 如果 shell 具有作业管理能力，则它所创建的相关进程构成一个进程组，同一进程组的进程都有相同的 `PGID`（如用管道连接的进程包含在同一个进程组中），进程组简化了向组的所有成员发送信号的操作，信号可以发送给组内的所有进程，这使得作业控制变得简单。当 `task_struct` 代表一个进程，且该进程属于某一个进程组，则 `task_struct->group_leader` 指向组长进程的 `task_struct`。PGID 保存在`task_struct->signal->pids[PIDTYPE_PGID].pid`中
-   `SID`：一系列进程组的组合，一般看到的 tty(比如键盘、屏幕的 `tty1~tty7`，或者网络连接的虚拟 tty 即 pty)，一个 tty 对应一个 session。在当前 tty 中创建的所有进程都共享一个 sid(即 leader 的 pid)
-   `PID`：是kernel内部对进程的一个标识符，用来唯一标识一个进程（task）、进程组（process group）、会话（session）。PID 及其对应进程存储在一个哈希表中，方便依据 PID 快速访问进程 `task_struct`

![basic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/sid_struct.png)

小结下就是，Linux 只有轻量级进程的概念，如果一个进程它和其他进程共享进程空间 `mm` 和文件句柄 `fd` 等一些资源，那它就是轻量级进程（相当于线程 thread），多个轻量级进程组成了一个线程组（thread group），该线程组中第一个创建的轻量级进程称之为 group_leader，其中的每一个轻量级进程（thread）拥有自己独立的 `pid`，所有的轻量级进程共享同一个线程组 `tgid`（即 group_leader 的 `pid`）；而`pgid`则是指进程组（process group）的所有进程都共享一个 pgid（也即 leader 的`pid`），二者是不同的概念

####    进程描述符基础结构：task_struct

本文只讨论内核态的进（线）程，本质上Linux 内核中进程/线程都是用 [`task_struct`](https://elixir.free-electrons.com/linux/v4.11.6/source/include/linux/sched.h#L483)（任务） 来表示的，结构如下：**在用户态调用`getpid`实际上返回的是`task_struct`的`tgid`字段**，而`pid`每个线程都是不同的（就同一个进程生成的不同线程而言），`task_struct`也是CPU调度的实体，是进程 process 是最小的调度单位

```cpp
//file:include/linux/sched.h
struct task_struct {
 //2.1 进程状态 
 volatile long state;

 //2.2 进程线程的pid
 pid_t pid;         //本质上int（每个轻量级进程都不同，算是唯一标识）
 pid_t tgid;        //本质上int（getpid实际返回的是这个字段）

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

```cpp
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
	struct hlist_node node;	// 链表节点
	struct pid *pid;	// 指向关联的 pid 结构
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

此外，在ebpf中常用的函数[`bpf_get_current_pid_tgid`](https://elixir.bootlin.com/linux/v4.15.18/source/kernel/bpf/helpers.c#L119)实现也是获取了`tgid`和`pid`（通过`bpf_get_current_pid_tgid() >> 32`获取用户空间可见的pid字段）

```cpp
BPF_CALL_0(bpf_get_current_pid_tgid)
{
	struct task_struct *task = current;

	if (unlikely(!task))
		return -EINVAL;

	return (u64) task->tgid << 32 | task->pid;
}
```

关于id的几个成员如下：

|  表头   | 表头  |	
|  ----  | ----  |	
| 轻量级进程 process （`pid`）  | `task_struct->pid //pid_t` |
|	|`task_struct->pids[PIDTYPE_PID]->pid //struct pid *`|
| 线程组 thread group （`tgid`）  | `task_struct->tgid //pid_t` |
|   | `task_struct->group_leader //struct task_struct *` |
|   | `task_struct->signal->leader_pid //struct pid *` |
| 进程组 process group （`pgid`）| `task_struct->pids[PIDTYPE_PGID]->pid //struct pid *`|

####    PID：进程ID（进程的抽象）
内核结构[`pid`](https://elixir.free-electrons.com/linux/v4.11.6/source/include/linux/pid.h#L57)定义如下，在`task_struct`中，通过`task->pids[PIDTYPE_PID].pid`可以定位到（`task_pid`[函数](https://elixir.free-electrons.com/linux/v4.11.6/source/include/linux/sched.h#L1056)），**虽然名字是pid，不过实际上该结构抽象了不仅是一个thread ID或者process ID，实际上还包括了进程组ID和session ID**，要点如下：

-	**`struct pid`是内核对进程（用户态）的PID的表示（即`tgid`的抽象，注意与线程的`pid_t pid`区别）**
-	一个`struct pid`可以对应多个`task_struct`，即一个（用户态）进程包含多个线程
-	一个PID可以对应多个的命名空间（pid_namespace）

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

```cpp
struct pid
{
	atomic_t count;
	unsigned int level;	//指定的level（ns）
	/* lists of tasks that use this pid */
	 /* 使用该pid的进程的列表*/
	struct hlist_head tasks[PIDTYPE_MAX];
	struct rcu_head rcu;
	struct upid numbers[1];		// 重要：包含了namespace信息
};

struct upid {
	  /* Try to keep pid_chain in the same cacheline as nr for find_vpid */
	  int nr;		//是`pid`的值， 即 `task_struct` 中 `pid_t pid` 域的值
	  struct pid_namespace *ns; // 所属的pid namespace（归属，其中包含了在每个namespace管理进程分配的bitmap）
	  struct hlist_node pid_chain;
};

enum pid_type
{
	PIDTYPE_PID,		//0
	PIDTYPE_PGID,		//1
	PIDTYPE_SID,		//2
	PIDTYPE_MAX
};
```

所以，内核对PID的管理其实就是围绕两个数据结构展开

-	`struct pid`：是内核对PID的内部表示
-	`struct upid`：**是表示特定的命名空间中可见的信息**


关于`pid.numbers[1]`这个成员，本质上一个柔性数组，每次在分配`struct pid`时，`numbers`会被扩展到`level`个元素，它用来容纳在每一层pid namespace中的 id（`upid`）

`struct pid`与`struct task_struct`之间是什么关系？答案是一对多，由于`struct pid`本身就是轻量级进程的抽象结构，一个进程可以有多个线程（轻量级进程），每个线程也有一个对应的`task_struct`实例，多个`task_struct`通过`task_struct->pids[x]->pid`指向其对应的`struct pid`结构（参考下图，`pids`一共有`PIDTYPE_MAX`种类型）

```cpp
struct task_struct{
 /* PID/PID hash table linkage. */
 struct pid_link pids[PIDTYPE_MAX];
};

//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pid.h#L57
struct pid
{
	//......
	struct hlist_head tasks[PIDTYPE_MAX];
};

struct pid_link
{
	struct hlist_node node;
	struct pid *pid;		//包含了ns信息：level与 numbers[]数组
};
```

不过，在较新版本内核如[6.15.1](https://elixir.bootlin.com/linux/v6.15.1/source/include/linux/sched.h#L1090)中，这里的成员略有调整（更简洁了，参考下图的指向关系）：

```cpp
struct task_struct{
	//......
	/* PID/PID hash table linkage. */
	struct pid			*thread_pid;	//thread_pid指向struct pid结构
	struct hlist_node		pid_links[PIDTYPE_MAX];
	struct list_head		thread_node;
}

struct pid
{
	refcount_t count;
	unsigned int level;
	spinlock_t lock;
	struct dentry *stashed;
	u64 ino;
	struct rb_node pidfs_node;
	/* lists of tasks that use this pid */
	struct hlist_head tasks[PIDTYPE_MAX];
	struct hlist_head inodes;
	/* wait queue for pidfd notifications */
	wait_queue_head_t wait_pidfd;
	struct rcu_head rcu;
	struct upid numbers[];
};
```

这里以`PIDTYPE_PGID`说明，为了方便查找，同属于一个进程组的所有进程对应的`task_struct`都被链接到同一个hash链表上：

![PIDTYPE_PGID_kernel_new](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/PIDTYPE_PGID.png)

####	signal_struct（tty处理相关）
`struct signal_struct *signal`成员中的`struct pid *pids[PIDTYPE_MAX]`这个成员的意义是什么？


####    pid_namespace：进程命名空间
`pid` 命名空间 `pid_namespace` 的[定义](https://elixir.free-electrons.com/linux/v4.11.6/source/include/linux/pid_namespace.h#L30)如下，关联`upid`的`ns`成员：

```cpp
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

下面可以看到 `pid_namespace` 的组织结构，`pid_namespace` 使用父子关系组成了树形结构，在分配新 pid 结构，会从当前的 pid ns(`task_struct->nsproxy->pid_ns_for_children->pid_cachep`) 中分配一个 `struct pid` 结构，`struct pid` 的`numbers`成员，该数组包含了向上的所有 pid ns，在每个 pid ns 中都分配了一个 pid number

![namespace](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_ns/struct_pid.png)

####    pid之间的关系（重要）

-   如何快速地根据进程的 `task_struct`、ID 类型、命名空间找到 PID ？
-   如何快速地根据 PID、pid namespace、ID 类型找到对应进程的 `task_struct` ？
-   如何快速地给新进程在可见的命名空间内分配一个唯一的 PID ？

下图描述了这种关系，在level `2` 的某个pid namespace上新建了一个进程，分配给它的 pid 为`45`，映射到 level `1` 的pid namespace，分配给它的 pid 为 `134`；再映射到 level `0` 的pid namespace，分配给它的 pid 为`289`（注意`numbers`这个柔性数组），此外，图中标识了某个位于namespace内的进程，其`level=2,pid=45`的pid结构（在`level1`pid为`134`，`level0`pid为`289`）

![relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_namespace.png)

![relation-final](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/relation-final-cn.png-modify.png)

上面这张图中有若干个细节：
1.	`struct pid`结构中的`tasks`数组里`3`个不同的`task`类型对`task`散列，`tasks[0]`、`tasks[1]`、`tasks[2]`的`0/1/2`代表`PIDTYPE_PID/PIDTYPE_PGID/PIDTYPE_SID`，通过这个链表结构可以快速定位到该属性的pid被哪些`task_struct`结构所引用
2.	`task_struct`的`pid_link.pid`成员指向它关联的`struct pid`结构（由`task_struct.tgid`决定），多个`task_struct`指向一个`struct pid`
3.	`pid_hash`表是由pid（`tgid`）加上namespace两个属性hash之后形成的hashtable，用于提高检索性能
4. 一个`struct pid`会属于多个命名空间，通过柔性数组`struct upid numbers[1]`扩展，每个`numbers[i]->nr`存储了对应namespace上可见的pid的值，`numbers[i]->ns`指向了对应的namespace

![pid_pidnamespace_upid_kernel_new](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_pidnamespace_upid.png)


####	pid_hash表（全局）
上文提到`pid_hash`表是由pid（`tgid`）加上命名空间namespace两个属性hash之后形成的hashtable，用于提高检索性能，其结构如下：

![pid_hash](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_hash.png)

定位过程如下：
1.	获取`struct upid`：根据 PID（参数`nr`） 以及指定命名空间（参数`ns`）计算在 `pid_hash` 数组中的索引，然后遍历散列表找到所要的 `upid`
2.  获得 pid 实体（`struct pid`）：根据内核的 `container_of` 机制找到 pid 实例
3.	根据`struct pid`的`tasks`链表及PID 类型（上文的`enums`），找到对应的`struct task_struct`

`pid_hash`中存放的是链表头，指向`upid`中的`pid_chain`，对于不同`level`但`pid`号相同的`upid`可能会被挂到同一串`pid_chain`，所以通过`pid`号查找`struct pid`的情况下需要指定`namespace`

注意到**pid_hash是全局唯一的，它包含所有命名空间中的PID实例（即`struct upid`），所有PID namespace共享同一个全局哈希表**

```cpp
// 1.	内核中的定义

// 全局唯一的 PID 哈希表，所有命名空间共享
static struct hlist_head *pid_hash;  

// 用于保护哈希表的锁也是全局的
static DEFINE_SPINLOCK(pidmap_lock);         // 全局锁

//2. 初始化代码
// 在系统启动时初始化，全局唯一
void __init pidhash_init(void)
{
    unsigned int i, pidhash_size;
    
    pid_hash = alloc_large_system_hash("PID", 
                                      sizeof(*pid_hash), 
                                      0, 18,  // 哈希表大小
                                      HASH_EARLY | HASH_ZERO,
                                      &pidhash_shift, 
                                      &pidhash_size, 
                                      0, 4096);
    ......
}
```

思考一下，为何`pid_hash`要设计为全局的hashtable呢？从下面几方面来分析下

1、支持**高效的跨namespace命名空间查找**，如`find_pid_ns`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L365)，传入参数为`ns`与该namespace中的pid值（`nr`），这样无论从哪个namespace进行查找，都使用同一个hashtable结构

```cpp
struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
{
	struct upid *pnr;

	// 计算哈希值：结合 PID 数值和namespace
    //int hash = pid_hashfn(nr, ns);

	//在全局哈希表中查找
	hlist_for_each_entry_rcu(pnr,
			&pid_hash[pid_hashfn(nr, ns)], pid_chain)
		if (pnr->nr == nr && pnr->ns == ns)
			return container_of(pnr, struct pid,
					numbers[ns->level]);	//返回struct pid（通过宏机制）

	return NULL;
}
```

2、hash function设计

hash值由 **(PID数值 + 命名空间层级)**计算得出，确保同一进程在不同命名空间中的不同 PID 值映射到不同的hash bucket，不同进程在不同命名空间中的相同 PID 值也映射到不同的hash bucket

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L43
#define pid_hashfn(nr, ns)	\
	hash_long((unsigned long)nr + (unsigned long)ns, pidhash_shift)
```

####    查询PID：分类讨论
根据上图，来看下相关的函数实现

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

3、获取 `pid` 实例中的 PID，参数如下

-   `pid`：指向 struct pid 结构体的指针，代表了一个进程的PID
-   `ns`：指向 struct pid_namespace 结构体的指针，代表了一个PID命名空间

```cpp
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

```cpp
static inline pid_t pid_nr(struct pid *pid)
{
 	pid_t nr = 0;
 	if (pid)
 		nr = pid->numbers[0].nr;
 	return nr;
}
```

5、当前命名空间对应 PID

```cpp
pid_t pid_vnr(struct pid *pid)
{
 	return pid_nr_ns(pid, task_active_pid_ns(current));
}
```

####   核心逻辑：分配PID
以 PID namespace为例，现在需要为新的任务分配一个新的任务id（`fork`进程/线程），调用链开始于 `clone` -> `do_fork` -> `copy_process` -> `alloc_pid`，`copy_process`函数会给此线程alloc一个`struct pid`结构体，核心代码[如下](https://github.com/torvalds/linux/blob/master/kernel/fork.c#L2147)：

```C
/*
 * This creates a new process as a copy of the old one,
 * but does not actually start it yet.
 *
 * It copies the registers, and all the appropriate
 * parts of the process environment (as per the clone
 * flags). The actual kick-off is left to the caller.
 */
__latent_entropy struct task_struct *copy_process(
					struct pid *pid,
					int trace,
					int node,
					struct kernel_clone_args *args)
{
	struct task_struct *p;
	//...
	p = dup_task_struct(current, node);
	//...
	retval = copy_namespaces(clone_flags, p);
	//...
	if (pid != &init_struct_pid) {
		//alloc_pid：p->nsproxy->pid_ns_for_children表示当前分配pid的namespace
		//申请pid
		pid = alloc_pid(p->nsproxy->pid_ns_for_children, args->set_tid,
				args->set_tid_size);
		if (IS_ERR(pid)) {
			retval = PTR_ERR(pid);
			goto bad_fork_cleanup_thread;
		}
		// set pid
		p->pid = pid_nr(pid);
		p->tgid = p->pid;
	}

	//...
}
```

1、`alloc_pid`函数：为新进程 `task_struct` 分配 `pid`，比如在`level=2`的namespace中运行一个`top`进程（`level=1`是`level=2`的parent，`level=0`是`level=1`的parent），该`level=2`的线程`fork`时，会从`level=2`开始alloc pid，一直到`level=0`，它会创建`3`个pid namespace的pid number（即`struct pid`的`level`成员的值为`2`），其中`level=0`是全局的，在通过`pid_nr()`函数设置`task_struct.pid_t`成员时，其就是取的`level=0` pid_namespace的pid number

所以这里也说明了，在pid namespace场景中，针对指定的微线程，其`struct pid`和`struct task_struct`都是唯一的，而不同namespace上的局部性差异则是由结构`upid`体现

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L296
// 参数传递的是新进程的 pid namespace
struct pid *alloc_pid(struct pid_namespace *ns)
{
 	struct pid *pid;
 	enum pid_type type;
 	int i, nr;
 	struct pid_namespace *tmp;
 	struct upid *upid;
 	int retval = -ENOMEM;
    
     // 从命名空间分配一个 pid 结构体（非整数）
 	pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
 	if (!pid)
 		return ERR_PTR(retval);
    
     // 初始化进程在各级命名空间的 PID，直到全局命名空间（level 为0）为止
	// 调用到alloc_pidmap来分配一个空闲的pid编号
 	// 注意，在每一个命令空间中都需要分配进程号
 	tmp = ns;
 	pid->level = ns->level;	//ns->level保存了当前所在的namespace层级

	// 从当前所在的namespace逆序遍历，已知到level==0层级，构建每一个level的pid值，并且保存在pid->numbers数组中（upid类型）
 	for (i = ns->level; i >= 0; i--) {
		// 注意：每个空间都调用alloc_pidmap，tmp为当前namespace的pid_namespace结构！
 		nr = alloc_pidmap(tmp);  //分配一个局部PID
 		if (nr < 0) {
 			retval = nr;
 			goto out_free;
 		}
    
 		pid->numbers[i].nr = nr;
 		pid->numbers[i].ns = tmp;
 		tmp = tmp->parent; //tmp更新为其上一级的namespace指针
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

```cpp
//alloc_pidmap： 在 pid 命名空间中申请一个 pid 号
static int alloc_pidmap(struct pid_namespace *pid_ns)
 {
 	int i, offset, max_scan, pid, last = pid_ns->last_pid;
 	struct pidmap *map;
    
 	pid = last + 1;
 	// 默认最大值在 include/linux/threads.h 中定义为 (CONFIG_BASE_SMALL ? 0x1000 : 0x8000)
 	if (pid >= pid_max)  
 		pid = RESERVED_PIDS;  // RESERVED_PIDS = 300
 	offset = pid & BITS_PER_PAGE_MASK;
	// 获取bitmap所在数组的下标
 	map = &pid_ns->pidmap[pid/BITS_PER_PAGE];
 	/*
 	 * If last_pid points into the middle of the map->page we
 	 * want to scan this bitmap block twice, the second time
 	 * we start with offset == 0 (or RESERVED_PIDS).
 	 */
 	max_scan = DIV_ROUND_UP(pid_max, BITS_PER_PAGE) - !offset;
	// 扫描bitmap，找到合适的未使用的 bit
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
					// 每个命名空间维护自己的 last_pid，加速分配
					// 缓存最后分配的 PID，性能优化
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

```cpp
 static void free_pidmap(struct upid *upid)
 {
 	int nr = upid->nr;
 	struct pidmap *map = upid->ns->pidmap + nr / BITS_PER_PAGE;
 	int offset = nr & BITS_PER_PAGE_MASK;
    
 	clear_bit(offset, map->page);
 	atomic_inc(&map->nr_free);
 }
```

####	进程号 pid 的管理
在`4.11.6`版本中，每个pid namespace（不同的level级别）都会有自己独立的bitmap来保存这个空间中的pid号，其中每一个 bit 位的 `0`或`1` 的状态来表示当前序号的 pid 是否被占用。在上面介绍的 `alloc_pidmap` 函数中就是以 bit 的方式来遍历整个 bitmap，找到合适的未使用的 bit，将其设置为已使用，然后返回

```cpp
#define BITS_PER_PAGE  (PAGE_SIZE * 8)
#define PIDMAP_ENTRIES  ((PID_MAX_LIMIT+BITS_PER_PAGE-1)/BITS_PER_PAGE)

struct pid_namespace {
	struct pidmap pidmap[PIDMAP_ENTRIES];
	//...
}
```

####    查询task_struct

1、获得 pid 实体（`struct pid`）

大致过程是，根据 PID（参数`nr`） 以及指定命名空间（参数`ns`）计算在 `pid_hash` 数组中的索引，然后遍历散列表找到所要的 `upid`， 再根据内核的 `container_of` 机制找到 pid 实例

```cpp
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

2、根据当前命名空间下的局部 PID 获取对应的 pid实例（方法二）

```cpp
struct pid *find_vpid(int nr)
{
  	return find_pid_ns(nr, task_active_pid_ns(current));
}
```

3、`pid_task`：根据 `pid` 及 PID 类型获取 `task_struct`（重要）


```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/rculist.h#L453
#define hlist_first_rcu(head)	(*((struct hlist_node __rcu **)(&(head)->first)))
#define hlist_next_rcu(node)	(*((struct hlist_node __rcu **)(&(node)->next)))
#define hlist_pprev_rcu(node)	(*((struct hlist_node __rcu **)((node)->pprev)))
```

```cpp
// https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L434
// 在哈希桶（pid->tasks[type]）中找到第一个节点，并将其转换回 task_struct 结构体
struct task_struct *pid_task(struct pid *pid, enum pid_type type)
{
 	struct task_struct *result = NULL;
 	if (pid) {
 		struct hlist_node *first;
		//hlist_first_rcu(head)：获取哈希桶（bucket）的第一个节点（并标记为__rcu）
 		first = rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),
 					      lockdep_tasklist_lock_is_held());
 		if (first){
			//hlist_entry：利用偏移量+container_of计算出包含这个节点的 task_struct 的起始地址
			//type：指定了 PID 的类型（如 PIDTYPE_PID 进程, PIDTYPE_PGID 进程组, PIDTYPE_SID 会话）
			//pids[type].node：task_struct 内部嵌入的链表节点
 			result = hlist_entry(first, struct task_struct, pids[(type)].node);
		}
 	}
 	return result;
}
```

先分析下`pid_task`函数的实现，作用是利用RCU机制配合`rcu_dereference_check`无锁读取hashtable的链表头，根据 `struct pid` 获取对应任务（`task_struct`），其实现完美的体现了**尽可能通过 RCU 避免获取全局锁（`tasklist_lock`）**的理念。`pid_task`中的核心关键宏`rcu_dereference_check`，这里反应内核中一个非常普遍的并发保护场景，如下：

当在一个 CPU 上通过 `pid_task` 查找某个进程组的组长时，另一个 CPU 可能正在把该组长对应的`task_struct`从链表中摘除（[`detach_pid`](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L414)）：

-	RCU机制：保证了读取者（Reader）不需要加锁
-	即便链表节点正在被删除，读取者拿到的指针依然是有效的内存地址（直到 RCU 宽限期结束才会真正释放），这保证了内核不会因为并发访问 PID 链表而崩溃（虽然RCU机制拿到的数据可能已经过期）

```cpp
//rcu_dereference_check：宏
rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),
 					      lockdep_tasklist_lock_is_held());
```

-	参数`hlist_first_rcu(...)`：获取链表首节点的指针
-	参数`lockdep_tasklist_lock_is_held()`：给内核调试工具 `Lockdep` 用的断言，有两个含义：
	-	读取：该断言告诉内核，要以 RCU 的方式读取这个指针。它会处理内存屏障，确保读取到的指针是有效的
    -	安全性检查：它要求调用者要么处于 RCU 读端临界区（`rcu_read_lock`），要么已经持有了 `tasklist_lock` 锁。如果两个条件都不满足，内核在调试模式下会直接报Warning警告

还有一个细节问题，为什么这个函数只取 `first`（链表头），而不需要循环遍历（for_each）获取呢？结合前面对结构体的描述，内核这里也使用了一个小技巧，即链表头成员一般是组长或者是最新的成员

稍微回顾下，在内核 PID 管理中，一个特定的 `struct pid`（代表一个具体的数值）对应一种 `type` 时，其 `tasks`（成员）链表通常代表的是进程领头者或归属于该 PID 的进程组/会话成员。对于 `PIDTYPE_PID` 类型，链表通常只有一个节点，对于进程组（`PGID`）类型，通过会关联多个进程，但 `pid_task` 返回的是链表头部的那个`task_struct`（组长或最新的成员）

`pid_task`函数（读函数）的典型的调用方场景如下，可以看到下面两个函数有共同点：

-	`rcu_read_lock()`：进入 RCU（Read-Copy-Update）临界区，由于进程列表和 `task_struct` 可能会动态变化，使用 RCU 可以安全地读取这些数据结构（不需要重量级的锁）
-	`pid_task(.....)`：通过 `struct pid`对象获取目标进程的 `task_struct` 结构体指针
-	`rcu_read_unlock()`： 退出 RCU 临界区

为何要如此设计呢？`rcu_read_lock` 机制（轻量级）保证了在读操作期间，所引用的内存对象不会被其他 CPU 释放掉，结合下面两个函数来看就很容易理解了

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L475
struct task_struct *get_pid_task(struct pid *pid, enum pid_type type)
{
	struct task_struct *result;
	rcu_read_lock();
	result = pid_task(pid, type);
	if (result){
		// 由于在调用层还需要使用result，所以这里需要增加task_struct的引用计数
		get_task_struct(result);
	}

	//增加完result的计数，才关闭rcu_read_unlock，允许其他CPU进行调度
	rcu_read_unlock();
	return result;
}

//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/fd.c#L282
// 用于检测进程访问`/proc/[pid]/fd`文件是否有权限
int proc_fd_permission(struct inode *inode, int mask)
{
	struct task_struct *p;
	int rv;

	rv = generic_permission(inode, mask);
	if (rv == 0)
		return rv;

	rcu_read_lock();
	p = pid_task(proc_pid(inode), PIDTYPE_PID);
	
	//same_thread_group：发起访问的进程（current）是否与目标进程（p）属于同一个线程组（Thread Group）？
	if (p && same_thread_group(p, current)){
		rv = 0;
	}

	//使用完p，才允许其他CPU调度
	rcu_read_unlock();

	return rv;
}
```

这里再列举下`get_pid_task`的上层调用方都做了哪些事情，以`proc_single_show`函数为例：

1.	调用`get_pid_task`获取 `task_struct`
2.	使用`task_struct`
3.	调用`put_task_struct`释放`task_struct`

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/proc/base.c#L2183
static int proc_single_show(struct seq_file *m, void *v)
{
	struct inode *inode = m->private;
	struct pid_namespace *ns;
	struct pid *pid;
	struct task_struct *task;
	int ret;

	ns = inode->i_sb->s_fs_info;
	pid = proc_pid(inode);

	//调用get_pid_task，拿到task_struct，并使用
	task = get_pid_task(pid, PIDTYPE_PID);
	if (!task)
		return -ESRCH;

	ret = PROC_I(inode)->op.proc_show(m, ns, pid, task);

	//释放task_struct
	put_task_struct(task);
	return ret;
}
```

所以，上面列举的基于`get_pid_task`前后的典型的场景，体现了指针安全以及对象生存两个特性，即`pid_task` 内部的 RCU 是为了查找时的内存安全；而 `get_pid_task` 外部的 RCU 结合引用计数，是为了跨函数调用的对象持久性

-	`pid_task` 的 RCU：仅仅保证在遍历链表时，拿到的 `task_struct` 指针是有效的，且在该 RCU 临界区内，这个结构体不会被释放内存
-	`get_task_struct`：RCU 只能保证在 `rcu_read_unlock()` 之前对象不消失。如果想在 `rcu_read_unlock()` 之后继续使用这个 `task_struct`，就必须增加它的引用计数（Reference Count）
-	如果 `get_pid_task` 不调用 `get_task_struct` 就返回指针，那么一旦函数返回，调用者就处于 RCU 临界区之外了。此时，如果该进程退出，内核可能会立即回收其内存，导致调用者持有一个野指针。所以通过 `get_pid_task` 获取的任务，调用者后续必须显式地调用 `put_task_struct` 来平衡引用计数

####	进程与线程
![process_and_thread](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/process_vs_thread.png)

####    小结
tid/pid/ppid/tgid/pgid/seesion id小结：

| ID | 解释 | task_struct 中的对应变量 | 系统调用|
| :-----:| :----: | :----: |:----: |
| PID （Process ID） | 实际上是线程 ID，内核中进程、线程都使用 `task_struct` 结构表示 | `task_struct->pid` |`pid_t gettid(void)` |
| TGID (Thread Group ID) | 线程组 ID，即线程组组长的 PID，真正的进程 ID，如果进程只有一个线程则他的 PID 和 TGID 相同 | `task_struct->tgid` | `pid_t getpid(void)` |
|PGID （Process Group ID）|进程组 ID，多个进程可以组合为进程组，方便向所有成员发送信号，进程组组长的 PID 即为PGID|`task_struct->signal->__pgrp`|`pid_t getpgrp(void)`|
|SID（Session ID）|会话 ID，多个进程组可以组合为会话，会话的组长PGID 即为 SID|`task_struct->signal->__session`|`pid_t getsid(pid_t pid)`|
|PPID （Parent Process ID）	|父进程 ID|`task_struct->parent->pid`|`pid_t getppid(void)`|

##	0x02	relation
本小节汇总下`task_struct`中几个id成员的关系：

####	process
每一个进程 process（包括轻量级进程、thread）创建时，都会创建一个对应的 `struct pid`，`struct pid` 创建了一个 `pid number` 数组，在每一层名空间中都分配了一个 `pid number`

process 的 `struct task_struct` 和 `struct pid` 之间的双向查询关系如下：

-	`task_struct --> pid`：通过 `task->pids[PIDTYPE_PID].pid` 指针指向 `struct pid`
-	`pid --> task_struct`：通过 `pid->tasks[PIDTYPE_PID]` 链表找到 `task_struct`（理论上该链表只有一个成员）

![process](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_ns/pid.png)

####	thread group
轻量级进程（线程）和线程组leader线程之间的双向查询关系如下：

-	`thread-->group leader`：普通线程 `task->group_leader` 存放线程组 leader 线程的 `task_struct` 结构
-	普通线程 `task->signal->leader_pid` 指向线程组 leader 线程的 `struct pid`
-	`group leader-->thread`：线程组 leader 线程的 `task->thread_group` 链表，链接了本线程组所有线程的 `task_struct`

![thread_group](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_ns/tgid.png)

####	process group
进程组也是使用 `pid_link` 来进行链接的，每个进程的进程组 `pgid` 指向同一个 leader，需要注意反向由进程组 leader 查询进程时的只能查询到线程组 leader，因为只把线程组 leader 链接到一起，而线程组下的普通线程由线程组 leader 自己来组织

线程组 leader 和进程组 leader 之间的双向查询关系：

-	`thread group leader --> process group leader`：线程组learder的`task->pids[PIDTYPE_PGID].pid`指针指向进程组leader的`struct pid`
-	`process group leader --> thread group leader`：进程组leader（对应的`struct pid`）的`pid->tasks[PIDTYPE_PGID]`链表链接了所有线程组learder的`task_struct`结构

![Pg](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_ns/pgid.png)

####	session
会话也是使用 `pid_link` 进行链接，每个进程的会话 `sid` 指向同一个 leader，需要注意反向由会话 leader 查询进程时的只能查询到线程组 leader，因为只把线程组 leader 链接到一起，而线程组下的普通线程由线程组 leader 自己来组织（同上）

线程组 leader 和会话 leader 之间的双向查询关系：

-	`thread group leader --> process group leader`：线程组learder的`task->pids[PIDTYPE_SID].pid`指针指向会话leader的`struct pid`结构
-	`process group leader --> thread group leader`：会话leader的`pid->tasks[PIDTYPE_SID]`链表链接了所有线程组learder的`task_struct`结构

![SESSION](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_ns/sid.png)

##  0x03    Linux进程虚拟地址空间
`task_struct`成员，内存描述符 `mm_struct`（memory descriptor）表示了整个进程的虚拟地址空间部分。进程运行时，在用户态其所需要的代码、全局变量以及 mmap 内存映射等全部都是通过 `mm_struct` 来进行内存查找和寻址的， `mm_struct` 关联的地址空间、页表、物理内存的关系如下图：

```cpp
struct mm_struct {
 struct vm_area_struct * mmap;  /* list of VMAs */
 struct rb_root mm_rb;

 unsigned long mmap_base;  /* base of mmap area */
 unsigned long task_size;  /* size of task vm space */
 unsigned long start_code, end_code, start_data, end_data;
 unsigned long start_brk, brk, start_stack;
 unsigned long arg_start, arg_end, env_start, env_end;


	/* store ref to file /proc/<pid>/exe symlink points to */
	struct file __rcu *exe_file;
}
```

![mm_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/task-struct-4-mm-modify.png)

-	**在内核内存区域，可以通过直接计算得出物理内存地址，并不需要复杂的页表计算**。而且最重要的是所有内核进程、以及用户进程的内核态，这部分内存都是共享的
-	`mm_struct`表示的是虚拟地址空间，而对于内核线程来说，是没有用户态的虚拟地址空间的，其value为`NULL`
-	`mm_struct.exe_file`：（指向 `struct file`） 引用可执行文件（[较新内核](https://elixir.bootlin.com/linux/v6.2-rc1/source/include/linux/mm_types.h#L732)增加此字段）

##  0x04    进程权限

####    进程权限凭证（credential）
在内核结构`task_struct`中有下面的字段标识了进程权限凭证，`real_cred`是指可以操作本任务的对象，而`cred`是指本任务可以操作的对象
```cpp
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

```cpp
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

-	`uid`和 `gid`（ real user/group id）：一般情况下，谁启动的进程，就是谁的 ID
-	`euid` 和 `egid`（effective user/group id）：实际起作用的字段。当这个进程要操作消息队列、共享内存、信号量等对象的时候，其实就是在比较这个用户和组是否有权限
-	`fsuid` 和`fsgid`（filesystem user/group id）：这个是对文件操作会审核的权限

在Linux中可以通过`chmod u+s program`更改`euid`和`fsuid`来获取权限

##  0x05 关联文件系统
`task_struct`亦关联了进程文件系统信息（如：当前目录等）以及当前进程打开文件的信息

```cpp
struct task_struct{
	// ...
    struct fs_struct *fs; 　　　　//文件系统信息，fs保存了进程本身与VFS（虚拟文件系统）的关系信息
    struct files_struct *files;　//打开文件信息（记录了所有process打开文件的句柄数组）
	// ...
}
```

![relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/task_struct.png)

#### 进程文件系统信息（fs_struct）

```cpp
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

在 `fs_struct` 中包含了两个 `path` 对象，而每个 `path` 中都指向了一个 `struct dentry`。在 Linux 内核中，`dentry` 结构是对一个目录项的描述。以 `pwd` 为例，该指针指向的是进程当前目录所处的 `dentry` 目录项。如在 shell 进程中执行 `pwd`命令，或用户进程查找当前目录下的配置文件的时候，都是通过访问 `pwd` 这个对象，进而找到当前目录的 `dentry`

####  进程打开的文件信息（files）
每个进程用一个 `files_struct` 结构（用户打开文件表）来记录文件描述符的使用情况

```cpp
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

与此，vfs主要数据结构的关联关系如下图所示：

![vfs](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/vfs.png)

这里埋一个有趣的问题，假设某个`struct task_struct`的父进程打开了socket连接（如bash反连场景），那么如何通过当前的`task_struct`获取到该socket（本质上也是fd）关联的四元组详情？

##	0x06	namespaces
`task_struct`中成员`struct nsproxy *nsproxy;`指向命名空间namespaces的指针（通过 namespace 可以让一些进程只能看到与自己相关的一部分资源，而另外一些进程也只能看到与它们自己相关的资源，这两类进程无法感知对方的存在）。实现方式是把一个或多个进程的相关资源指定在同一个 namespace 中，而进程究竟是属于哪个 namespace 由 `*nsproxy` 指针表明了归属关系

```cpp
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

##	0x07	进程线程状态及状态图
![state](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/scheduler/process-state.png)

进程描述符中的`state`成员描述了进程的当前状态：

```cpp
struct task_struct {
	volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
    /*...*/
};
```

内核中主要的状态字段定义如下（注释已经说明了应用在`task_struct`的字段）：

```cpp
/* Used in tsk->state: */
#define TASK_RUNNING			0x0000
#define TASK_INTERRUPTIBLE		0x0001
#define TASK_UNINTERRUPTIBLE		0x0002

/* Used in tsk->exit_state: */
#define EXIT_DEAD			0x0010
#define EXIT_ZOMBIE			0x0020
#define EXIT_TRACE			(EXIT_ZOMBIE | EXIT_DEAD)

/* Used in tsk->state again: */
#define TASK_PARKED			0x0040
#define TASK_DEAD			0x0080
#define TASK_WAKEKILL			0x0100
#define TASK_WAKING			0x0200
#define TASK_NOLOAD			0x0400
#define TASK_NEW			0x0800
#define TASK_STATE_MAX			0x1000

/* Convenience macros for the sake of set_current_state: */
#define TASK_KILLABLE			(TASK_WAKEKILL | TASK_UNINTERRUPTIBLE)
#define TASK_STOPPED			(TASK_WAKEKILL | __TASK_STOPPED)
#define TASK_TRACED			(TASK_WAKEKILL | __TASK_TRACED)

#define TASK_IDLE			(TASK_UNINTERRUPTIBLE | TASK_NOLOAD)
```

系统中的每个内核线程都必然处于五种进程状态中的一种，与进程状态和进程的运行、调度有关系：

-	`TASK_RUNNING`（运行）：进程是可执行的，它或者正在执行，或者在运行队列中等待执行。这是进程在用户空间中执行的唯一可能的状态（表示进程线程处于就绪状态或者是正在执行）
-	`TASK_INTERRUPTIBLE`（可中断）：进程正在睡眠（被阻塞），等待某些条件的达成。一旦这些条件达成，内核就会把进程状态设置为运行，处于此状态的进程也会因为接收到信号而提前被唤醒并随时准备投入运行
-	`TASK_UNINTERRUPTIBLE` （不可中断）：除了就算是接收到信号也不会被唤醒或准备投入运行外，这个状态与可打断状态相同。这个状态通常在进程必须在等待时不受干扰或等待事件很快就会发生时出现。由于处于此状态的任务对信号不做响应，所以较之`TASK_INTERRUPTIBLE`可中断状态，使用得较少（如`ps`时看到被标为`D`状态而又不能被杀死的进程的原因。由于类任务将不响应信号，因此不可能给它发送`SIGKILL`信号）
-	`__TASK_TRACED`：被其他进程跟踪的进程（如通过`ptrace`对程序进行跟踪）
-	`__TASK_TSTOPPED`（停止）：进程停止执行；进程没有投入运行也不能投入运行，通常这种状态发生在接收到`SIGSTOP`、`SIGTSTP`、`SIGTTIN`、`SIGTTOU` 等信号的时候。此外，调试期间接收到任何信号，都会使进程进入这种状态

举例来说，一个任务（进程或线程）刚创建出来的时候是 `TASK_RUNNING` 就绪状态，等待调度器的调度，调度器执行 `schedule` 后，任务获得 CPU 后进入执行（运行）。当需要等待某个事件的时候，例如阻塞式 `read` 某个 socket 上的数据，但是数据还没有到达时，任务会进入 `TASK_INTERRUPTIBLE` 或 `TASK_UNINTERRUPTIBLE` 状态，任务被阻塞掉

当等待的事件到达以后，如 socket 上的数据到达，内核在收到数据后会查看 socket 上阻塞的等待任务队列，然后将之唤醒，使得任务重新进入 `TASK_RUNNING` 就绪状态；任务如此往复地在各个状态之间循环，直到退出

注意`TASK_RUNNING`是表示进程在时刻准备运行的状态（可能不一定在运行），当处于这个状态的进程获得时间片的时候，就是在运行中；如果没有获得时间片，就说明它被其他进程抢占了，在等待再次分配时间片。在运行中的进程，一旦要进行一些 I/O 操作，需要等待 I/O 完毕，这个时候会释放 CPU，进入睡眠（阻塞）状态

-	若进入`TASK_INTERRUPTIBLE`可中断的睡眠状态，虽然在睡眠状态等待 I/O 完成，但是这时一个信号来的时候，进程还是要被唤醒。只不过唤醒后不是继续刚才的操作，而是进行信号处理。当然程序员可以根据自己的意愿，来写信号处理函数，例如收到某些信号，就放弃等待这个 I/O 操作完成，直接退出；或者收到某些信息，继续等待。
-	若进入 `TASK_UNINTERRUPTIBLE`不可中断的睡眠状态时，不可被信号唤醒，只能死等 I/O 操作完成。一旦 I/O 操作因为特殊原因不能完成，这个时候谁也叫不醒这个进程了，当然也无法被 `kill`（本质也是信号，除非重启）

于是就有了一种新的进程睡眠状态，`TASK_KILLABLE`，可以终止的新睡眠状态，若进程处于这种状态中，它的运行原理类似 `TASK_UNINTERRUPTIBLE`，只不过可以响应致命信号。由于`TASK_WAKEKILL` 用于在接收到致命信号时唤醒进程，因此`TASK_KILLABLE`即在`TASK_UNINTERUPTIBLE`的基础上增加一个`TASK_WAKEKILL`标记位即可

`TASK_STOPPED`是在进程接收到 `SIGSTOP`、`SIGTTIN`、`SIGTSTP`或者 `SIGTTOU` 信号之后进入该状态

`TASK_TRACED` 表示进程被 `debugger` 等进程监视，进程执行被调试程序所停止。当一个进程被另外的进程所监视，每一个信号都会让进程进入该状态

一旦一个进程要结束，先进入的是 `EXIT_ZOMBIE` 状态，但是这个时候它的父进程还没有使用`wait()` 等系统调用来获知其终止信息，此时进程就成了僵尸进程。`EXIT_DEAD` 是进程的最终状态，`EXIT_ZOMBIE` 和 `EXIT_DEAD` 也可以用于 `exit_state`

![switch-state](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/TASK_STATE.png)

小结下，状态变迁：
-	`INIT--->TASK_RUNNING[READY]`：当前任务调用`fork`创建一个新进程
-	`TASK_RUNNING[READY]--->TASK_RUNNING[RUNNING]`：内核调度，通过`schedule-->context_switch`将新任务放入CPU运行
-	`TASK_RUNNING[RUNNING]--->TASK_RUNNING[READY]`：当前任务被高优先任务抢占
-	`TASK_RUNNING[RUNNING]--->TASK_INTERRUPTIBLE/TASK_UNINTERRUPTIBLE`：为了等待特定事件，任务在等待队列上阻塞
-	`TASK_RUNNING[RUNNING]--->EXIT`：任务通过`do_exit`函数退出
-	`TASK_INTERRUPTIBLE/TASK_UNINTERRUPTIBLE-->TASK_RUNNING[READY]`：	等待的事件发生后任务被唤醒，并且被重新置入运行队列中

##	0x08	进程亲缘关系
全部进程构成了一颗进程树（`0`号进程是根节点）
```cpp
struct task_struct {
	struct task_struct __rcu *real_parent; /* real parent process */
	struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */
	struct list_head children;      /* list of my children */
	struct list_head sibling;       /* linkage in my parent's children list */
}
```
-	`real_parent`：表示进程的原始创建者（即最初调用`fork()`或`clone()`的进程）。始终指向进程的原始父进程，不受调试或进程关系变更的影响
-	`parent`：指向其父进程。当它终止时，必须向它的父进程发送信号
-	`children`：指向子进程链表的头部。链表中的所有元素都是它的子进程
-	`sibling`：用于把当前进程插入到兄弟链表中

注意：通常情况下，`real_parent` 和 `parent` 是一样的，例外如 `bash` 创建一个进程，那进程的 `parent` 和 `real_parent` 就都是 `bash`。如果在 `bash` 上使用 `GDB` 来 `debug` 一个进程，这个时候 `GDB` 是 `parent`，`bash` 则是这个进程的 `real_parent`；当用于审计、统计或需要追溯进程真实来源的场景时，使用`real_parent`比较合适

##	0x09	操作汇总
本节根据上面这张图，总结下内核实现的常用的查找函数的实现

![kernel-process-relation]()

TODO

####	`struct pid`相关
1、分配pid与回收pid


2、查询pid


####	`struct task_struct`：安全的引用/解引用
`task_struct` 结构体在内核中被很多地方（如调度器、进程间通信、文件系统等）引用。为了防止它被提前销毁，内核采用了引用计数机制

1、查找`task_struct`

2、`get_task_struct/put_task_struct`的应用

这里主要介绍`put_task_struct`函数，其核心功能是通过原子操作减少进程描述符的引用计数，并在计数归零时触发真正的内存释放

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/sched/task.h#L87
#define get_task_struct(tsk) do { atomic_inc(&(tsk)->usage); } while(0)

extern void __put_task_struct(struct task_struct *t);

static inline void put_task_struct(struct task_struct *t)
{
    // 1. 原子减 1，并测试结果是否为 0
    if (atomic_dec_and_test(&t->usage)){
		// 如果atomic_dec_and_test结果是 0，返回 true（意味着这是最后一个使用者）
        // 2. 如果（原子计数）减到了 0，说明没有任何地方再使用这个 task 了
        __put_task_struct(t);
	}
	//如果atomic_dec_and_test结果 大于 0，返回 false（意味着还有其他人在使用）
}

//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/fork.c#L393
void __put_task_struct(struct task_struct *tsk)
{
	WARN_ON(!tsk->exit_state);
	WARN_ON(atomic_read(&tsk->usage));
	WARN_ON(tsk == current);

	cgroup_free(tsk);
	task_numa_free(tsk);
	security_task_free(tsk);
	exit_creds(tsk);
	delayacct_tsk_free(tsk);
	put_signal_struct(tsk->signal);

	if (!profile_handoff_task(tsk))
		free_task(tsk);
}
```

`__put_task_struct`是底层清理函数，主要完成释放 `task_struct` 占用的内存，清理与该进程相关的内核栈和其他附带资源等，在应用层视角来看，当进程主动执行`exit`退出，父进程调用`wait`系统调用获取进程退出码时，`task_struct`，只有当调度器不再需要它（比如最后一次切换完成）、父进程也不再需要它、以及所有持有该 task 引用的内核模块都释放了它，`put_task_struct` 才会真正把这块内存还给系统

####	`struct upid`相关


####	示例应用：kill的实现
比如，在容器中执行`kill xxx`（ `kill` 系统调用只在当前的namespace中生效），这里涉及到哪些process相关的操作？从kill的内核[实现](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/signal.c#L2865)：

```cpp
SYSCALL_DEFINE2(kill, pid_t, pid, int, sig)
{
	struct siginfo info;

	info.si_signo = sig;
	info.si_errno = 0;
	info.si_code = SI_USER;
	info.si_pid = task_tgid_vnr(current);
	info.si_uid = from_kuid_munged(current_user_ns(), current_uid());

	//kill_something_info函数：向当前namespace中的所有进程发送信号
	return kill_something_info(sig, &info, pid);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/signal.c#L1385
/*
sig: 要发送的信号编号
info: 信号附加信息（可为NULL）
pid: 目标进程标识符，取值有特殊含义
*/
static int kill_something_info(int sig, struct siginfo *info, pid_t pid)
{
	int ret;

	if (pid > 0) {
		//pid > 0 的情况：向单个（指定）进程发信号
		/*
		使用RCU读锁保护进程查找
		find_vpid(pid)：在当前命名空间查找对应的pid结构
		调用kill_pid_info发送信号
		*/
		rcu_read_lock();
		ret = kill_pid_info(sig, info, find_vpid(pid));
		rcu_read_unlock();
		return ret;
	}

	//pid <= 0 的情况：向进程组发信号
	//需要获取任务列表锁来保护进程组操作

	read_lock(&tasklist_lock);
	if (pid != -1) {
		/*
		pid < -1： 向进程组ID为-pid的整个进程组发信号
		pid = 0：向当前进程所在进程组发信号
		pid = -1：不进入这个分支（有特殊处理）
		*/
		ret = __kill_pgrp_info(sig, info,
				pid ? find_vpid(-pid) : task_pgrp(current));
	} else {
		//pid = -1 的情况：向所有进程发信号（除特殊进程）
		int retval = 0, count = 0;
		struct task_struct * p;

		for_each_process(p) {
			//task_pid_vnr(p) > 1：跳过init进程（PID=1）
			//!same_thread_group(p, current)：跳过当前线程组的进程
			if (task_pid_vnr(p) > 1 &&
					!same_thread_group(p, current)) {
				int err = group_send_sig_info(sig, info, p);
				++count;
				if (err != -EPERM)
					retval = err;
			}
		}
		ret = count ? retval : -ESRCH;
	}
	read_unlock(&tasklist_lock);

	return ret;
}
```

上面代码中涉及到的几个查询函数&&宏，实现了两种非常典型的链表遍历的场景

-	`find_vpid->find_pid_ns->task_active_pid_ns`与`task_pgrp`
-	`for_each_process`

```cpp
struct pid *find_vpid(int nr)
{
	// 熟悉的find_pid_ns
	return find_pid_ns(nr, task_active_pid_ns(current));
}

#define for_each_process(p) \
	for (p = &init_task ; (p = next_task(p)) != &init_task ; )
```

链表遍历1：基于`struct pid`的链表头开始的遍历，在进程组发送信号的情况下（`pid < 0` 且 `pid!=-1`）：

```cpp
......
if (pid != -1) {
    ret = __kill_pgrp_info(sig, info,
            pid ? find_vpid(-pid) : task_pgrp(current));
}
......
//在__kill_pgrp_info会遍历进程组的所有线程，这里才会使用pid链表
int __kill_pgrp_info(int sig, struct siginfo *info, struct pid *pgrp)
{
	struct task_struct *p = NULL;
	......
	//遍历进程组的所有线程，基于struct pid的成员tasks[PIDTYPE_PGID]链表头的遍历，遍历元素为task_struct
	do_each_pid_task(pgrp, PIDTYPE_PGID, p) {
		......
	} while_each_pid_task(pgrp, PIDTYPE_PGID, p);
	return success ? 0 : retval;
}
```

链表遍历2：基于全局链表`init_task`的遍历，`pid = -1`（向所有进程发信号），使用`for_each_process`遍历所有`task_struct`。准确的说，这里的遍历是需要加上namespace限制的，即只过滤当前namespace下的所有`task_struct`，这个逻辑是在`task_pid_vnr`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/sched.h#L1106)中实现的，核心是`__task_pid_nr_ns`函数

**在`pid_nr_ns`函数中会校验`current`所在的命名空间是否与遍历`init_task`中的`task_struct`对象所在的命名空间是否一致（可见），需要满足前者的`level`小于或者等于后者**

```cpp
......
for_each_process(p) {
			//task_pid_vnr(p) > 1：跳过init进程（PID=1）
			//!same_thread_group(p, current)：跳过当前线程组的进程

			//task_pid_vnr->__task_pid_nr_ns
			if (task_pid_vnr(p) > 1 &&
					!same_thread_group(p, current)) {
				int err = group_send_sig_info(sig, info, p);
				++count;
				if (err != -EPERM)
					retval = err;
			}
		}
......

static inline pid_t task_pid_vnr(struct task_struct *tsk)
{
	return __task_pid_nr_ns(tsk, PIDTYPE_PID, NULL);
}

//隐藏的命名空间过滤 - task_pid_vnr(p)
pid_t __task_pid_nr_ns(struct task_struct *task, enum pid_type type, struct pid_namespace *ns)
{
	pid_t nr = 0;

	rcu_read_lock();
	if (!ns){
		// 获取当前进程的命名空间！
		ns = task_active_pid_ns(current);
	}
	if (likely(pid_alive(task))) {
		if (type != PIDTYPE_PID)
			task = task->group_leader;
		
		//注意这里的type是PIDTYPE_PID
		//pid_nr_ns中会校验current所在的命名空间是否与遍历init_task中的task_struct对象所在的命名空间是否一致（可见）
		nr = pid_nr_ns(rcu_dereference(task->pids[type].pid)/*init_task*/, ns/*current*/);
	}
	rcu_read_unlock();

	return nr;
}

//返回PID在指定命名空间中的数值
pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)
{
	struct upid *upid;
	pid_t nr = 0;

	if (pid && ns->level <= pid->level) {
		upid = &pid->numbers[ns->level];
		if (upid->ns == ns)
			nr = upid->nr;
	}
	return nr;
}
```

上面代码有几处细节，首先是`__task_pid_nr_ns`中的`nr = pid_nr_ns(rcu_dereference(task->pids[type].pid), ns)`，其中的`type`为`PIDTYPE_PID`，`task->pids[PIDTYPE_PID].pid`即获取该`task_struct`以`PIDTYPE_PID`身份对应的`struct pid`对象，然后在函数`pid_nr_ns`中，使用对应的`struct pid`对象与namespace命名空间对象，确认该pid是否在这个namespace中：

```cpp
pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)
{
	struct upid *upid;
	pid_t nr = 0;

	//如果ns->level<=pid->level，说明pid肯定在namespace中，但不一定是在同一个level上
	if (pid && ns->level <= pid->level) {
		// 获取在当前（与传入的task_struct对象一直）level（ns->level）的upid对象
		upid = &pid->numbers[ns->level];
		if (upid->ns == ns){	//二次校验
			nr = upid->nr;		//返回该namespace上的pid数值
		}
	}

	//返回0（不满足）或者真实的nr
	return nr;
}
```

##	0x0A	一些有趣的示例

####	思考1：fs_struct的cwd/root 与 nsproxy 的关系

![mnt_namespace](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/nsproxy_mntnamespace.png)

从`0`号进程开始，每个`task_struct`都会指向一个`nsproxy`结构，用于提供namespace下的挂载点的所有拓扑信息（默认内核系统中只有一棵 mount 树，为了支持 `mnt_namespace`机制内核把 mount 树扩展成了多棵。每个 `mnt_namespace` 拥有一棵独立的 mount 树），如上图中`mnt_namespace`结构中的`root`成员指向本命名空间mount树的根（挂载）节点，此外，该namespace内所有的独立挂载点都通过 `mnt_namespace.list`链表串接起来

因此，`task_struct`中的`fs_struct`结构的`root`成员（进程的根目录），其`root.vfsmount`结构指向和`task_struct->nsproxy->mnt_namespace->root`就是一致的（不使用`chroot`等场景），总结如下：

1、`fs_struct`与`mnt_namespace`的核心成员

`fs_struct`代表了**进程级文件系统视图**，定义了进程对文件系统的个性化视图（如隔离的根目录或工作路径），管理进程独立的文件系统上下文

-	`struct path root`：进程的根目录（可通过 `chroot` 修改）
-	`struct path pwd`：进程的当前工作目录（通过 `chdir` 修改）

​而`mnt_namespace`[结构](https://elixir.bootlin.com/linux/v4.11.6/source/fs/mount.h#L7)定义了挂载命名空间全局视图，**管理整个挂载命名空间的全局状态**，维护命名空间内所有挂载点的拓扑结构（如 `mount/umount` 操作的生效范围），如在容器内运行`mount`新的挂载，在宿主机是看不到的（仅在当前的`mnt_namespace`内生效）

-	`struct mount *root`：命名空间的挂载树根节点（描述所有挂载点）

`mnt_namespace` 限定物理资源范围，`fs_struct` 定义进程在范围内的工作起点

2、场景交互关系

-	进程访问文件（如`open`操作）：`fs_struct`提供`root`或者`pwd`作为路径解析起点，而`mnt_namespace`提供本命名空间内的全局挂载树，将路径映射到实际文件系统；此外，若进程通过 `fs_struct->root` 或`pwd` 解析相对路径时，内核需结合 `mnt_namespace->root` 指向的挂载树，找到文件对应的物理存储位置
-	修改进程根目录 (`chroot`调用)：更新 `fs_struct->root` 为进程新的根路径，这里隐含了新的根路径需在当前挂载命名空间内存在，特别注意`fs_struct->root` 仅改变进程的根目录视图，但无法突破 `mnt_namespace` 的隔离。若新根目录不在当前挂载命名空间内，操作将失败
-	挂载操作 (`mount`)，涉及到更新挂载树，影响所有共享该命名空间的进程

3、生命周期与共享机制

-	对于`fs_struct`而言，一般通过写时复制（COW）机制，进程修改根目录或工作目录时会复制 `fs_struct`，避免影响其他进程
-	对于`mnt_namespace`而言，是命名空间内所有进程共享的，即同一挂载命名空间的所有进程共享相同的 `mnt_namespace`，仅当调用 `unshare(CLONE_NEWNS)` 或创建新容器时，才会生成新副本（回忆一下创建进程时内核调用`copy_namespaces`方法来为子进程复制独立的命名空间操作）

![mnt_namespace_vs_fsstruct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/namespace/mnt/mnt_tree_ns.png)

####	思考2：进程创建的过程
以系统调用[`fork`](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/fork.c#L2063)为例，其调用链为`fork()->_do_fork()->copy_process()`

![copy_process](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/scheduler/task_fork_flow.jpeg)

1、`_do_fork`的实现，其中有三个关键节点

-	`copy_process`：以拷贝父进程的方式来生成一个新的 `task_struct` 
-	`trace_sched_process_fork`
-	`wake_up_new_task`：子任务加入到CPU就绪队列（runqueue）中去，等待调度器调度

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/fork.c#L1491
long _do_fork(unsigned long clone_flags,
	      unsigned long stack_start,
	      unsigned long stack_size,
	      int __user *parent_tidptr,
	      int __user *child_tidptr,
	      unsigned long tls)
{
	struct task_struct *p;
	int trace = 0;
	long nr;

	//......

	p = copy_process(clone_flags, stack_start, stack_size,
			 child_tidptr, NULL, trace, tls, NUMA_NO_NODE);
	add_latent_entropy();
	/*
	 * Do this prior waking up the new thread - the thread pointer
	 * might get invalid after that point, if the thread exits quickly.
	 */
	if (!IS_ERR(p)) {
		struct completion vfork;
		struct pid *pid;

		// tracepoint with tracepoint:sched:sched_process_fork
		trace_sched_process_fork(current, p);

		pid = get_task_pid(p, PIDTYPE_PID);
		nr = pid_vnr(pid);

		if (clone_flags & CLONE_PARENT_SETTID)
			put_user(nr, parent_tidptr);

		if (clone_flags & CLONE_VFORK) {
			p->vfork_done = &vfork;
			init_completion(&vfork);
			get_task_struct(p);
		}

		// 底层调度（参考CFS调度）
		wake_up_new_task(p);

		// ......
	} else {
		nr = PTR_ERR(p);
	}
	return nr;
}
```

2、`copy_process`的实现，对于进程创建的场景其中地址空间 `mm_struct`、挂载点 `fs_struct`、打开文件列表 `files_struct` 都要是独立复制一份（子进程指向单独的空间），而命名空间`nsproxy` 是父子共享的。但对于线程而言，上述结构都是共享的。共同点是对于`task_struct`结构而言，内核会调用`dup_task_struct`单独复制一份，针对fork进程场景的核心调用如下：

-	`security_task_create`
-	`dup_task_struct`：复制进程 `task_struct` 结构体，传入的参数是 `current`，它表示的是当前进程。会申请一个新的 `task_struct` 内核对象，然后将当前进程复制给新的结构，注意此时成员指针的指向还未变化
-	`copy_files`：由于进程之间都是独立的，所以新进程需要copy一份独立的 `files` 成员
-	`copy_fs`：新进程也需要一份独立的文件系统信息，即 copy 一份`fs_struct` 成员
-	`copy_mm`： copy 独立的进程地址空间（进程之间地址空间也必须是隔离的），`copy_mm`操作复制进程的页表，但是由于Linux系统采用了写时复制（Copy-On-Write）技术，因此这里仅为新进程设置自己的页目录表项和页表项，而没有实际为新进程分配物理内存页面，此时新进程与其父进程共享所有物理内存页面
-	`copy_namespaces`：默认情况下，父子进程复用同一套命名空间对象
-	`alloc_pid`：为子进程申请pid，前文已分析

此外，还需要注意的虽然`task_struct.files_struct.fdtable`在子进程中单独复制了一份，但是这里的`fdtable`数组每个元素的指针指向是和父进程一致的，即**子进程也会继承父进程的文件描述符，也就是子进程会将父进程的文件描述符表项都复制一份**，也就是说如果父进程和子进程同时写一个文件的话可能产生并发写的问题，导致写入的数据错乱，所以在开发上就会有各种技巧来应对（参考下一小节）

![fork-2-vfs](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/fork_2_vfs_relation.png)

多说一句，这里的fork场景还有一个经典的地方使用到，那就是网络编程中通过`SO_REUSEPORT`技术解决惊群问题

```cpp
static __latent_entropy struct task_struct *copy_process(
					unsigned long clone_flags,
					unsigned long stack_start,
					unsigned long stack_size,
					int __user *child_tidptr,
					struct pid *pid,
					int trace,
					unsigned long tls,
					int node)
{
	struct task_struct *p;
	retval = security_task_create(clone_flags);

	// 复制进程 task_struct 结构体
	p = dup_task_struct(current, node);

	retval = copy_creds(p, clone_flags);

	/* Perform scheduler related setup. Assign this task to a CPU. */
	retval = sched_fork(clone_flags, p);

	// 拷贝 files_struct
	retval = copy_files(clone_flags, p);

	// 拷贝 fs_struct
	retval = copy_fs(clone_flags, p);

	retval = copy_sighand(clone_flags, p);

	retval = copy_signal(clone_flags, p);

	// 拷贝 mm_struct
	retval = copy_mm(clone_flags, p);

	// 拷贝进程的命名空间 nsproxy
	retval = copy_namespaces(clone_flags, p);
	
	retval = copy_io(clone_flags, p);
	
	retval = copy_thread_tls(clone_flags, stack_start, stack_size, p, tls);

	if (pid != &init_struct_pid) {
		// 申请 pid && 设置进程号
		pid = alloc_pid(p->nsproxy->pid_ns_for_children);
		if (IS_ERR(pid)) {
			retval = PTR_ERR(pid);
			goto bad_fork_cleanup_thread;
		}
	}

	// ......
}
```

3、`wake_up_new_task`：当 `copy_process` 执行完毕的时候，表示新进程的一个新的 `task_struct` 对象创建完成，接下来内核会调用此方法将这个新创建出来的子进程添加到就绪队列中等待调度

```cpp
void wake_up_new_task(struct task_struct *p)
{
	struct rq_flags rf;
	struct rq *rq;

	raw_spin_lock_irqsave(&p->pi_lock, rf.flags);
	p->state = TASK_RUNNING;

	rq = __task_rq_lock(p, &rf);
	update_rq_clock(rq);
	post_init_entity_util_avg(&p->se);

	activate_task(rq, p, 0);
	p->on_rq = TASK_ON_RQ_QUEUED;
	trace_sched_wakeup_new(p);
	check_preempt_curr(rq, p, WF_FORK);

	task_rq_unlock(rq, p, &rf);
}
```

4、补充：**写时复制（Copy-On-Write）技术**

写时复制的本质是共享直到修改，是一种延迟拷贝的资源优化策略，通常用在拷贝复制操作中，如果一个资源只是被拷贝但是没有被修改，那么这个资源并不会真正被创建，而是和原数据共享。COW 将复制操作推迟到第一次写入时进行

当`fork` 函数调用之后（因为Copy-On-Write 机制），存在父子进程实际上是共享物理内存的，内核把会共享的所有的内存页的权限都设为 read-only。当父子进程都只读内存，然后执行 exec 函数时就可以省去大量的数据复制开销。当其中某个进程写内存时，内存管理单元 MMU 检测到内存页是 read-only 的，于是触发缺页异常（page-fault），处理器会从中断描述符表（IDT）中获取到对应的处理程序。在中断程序中，内核就会把触发的异常的页复制一份，于是父子进程各自持有独立的一份，之后进程再修改对应的数据

Copy-On-Write的好处是显而易见的，同时也有相应的缺点，如果父子进程都需要进行大量的写操作，会产生大量的缺页异常（page-fault）。缺页异常不是没有代价的，它会处理器会停止执行当前程序或任务转而去执行专门用于处理中断或异常的程序。处理器会从中断描述符表（IDT）中获取到对应的处理程序，当异常或中断执行完毕之后，会继续回到被中断的程序或任务继续执行

Copy-on-write 实现原理如下图，当`fork()` 之后，内核会把父进程的所有内存页都标记为只读。一旦其中一个进程尝试写入某个内存页，就会触发一个保护故障（缺页异常），此时会陷入内核。内核将拦截写入，并为尝试写入的进程创建这个页面的一个新副本，恢复这个页面的可写权限，然后重新执行这个写操作，这时就可以正常执行了。内核会保留每个内存页面的引用数。每次复制某个页面后，该页面的引用数减少一；如果该页面只有一个引用，就可以跳过分配，直接修改。这种分配过程对于进程来说是透明的，能够确保一个进程的内存更改在另一进程中不可见

![COW](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/cow.jpg)

####	思考3：fork+dup(dup2)
在前文[主机入侵检测系统 Elkeid：设计与分析（一）](https://pandaychen.github.io/2024/08/19/A-ELKEID-STUDY-1/#0x01----agent)介绍进程通信的例子，父子进程+两组pipe实现全双工通信的模型，如下（one pipe）：

![pipe](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/system-pro/fork-pipe-dup.png)

-	父进程调用`pipe()`创建管道，得到两个fd（`fd[0]`、`fd[1]`）指向管道的两端
-	父进程调用`fork()`创建子进程，那么子进程也有两个文件描述符fd指向同一管道
-	父进程关闭管道读端（`close(fd[1])`），子进程关闭管道写端（`close(fd[0]`），父进程可以往管道里写数据，子进程可以从管道里读数据（单向）

从vfs视角，上面的流程如下图：
![vfs-pipe](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/163/fork-dup-dup2.jpg)

为什么父进程关闭管道的`fd[0]`后子进程还能读取管道的数据？从上图可见内核会维护文件的fd被引用计数（标识该表项被多少个fd引用），父/子进程都各自有指向相同文件的fd，当关闭一个fd时，相应计数减一，当这个计数减到`0`时，文件就被关闭，虽然父进程关闭了其fd，但是该文件的fd计数还没等于`0`，所以子进程可以读取（父进程和子进程都有各自的文件描述符，因此虽然父进程中关闭了`fd[0]`，但是对子进程中的`fd[0]`没有影响）

在linux的pipe管道下，写端进行写数据时，不需要关闭读端的缓冲文件（即不需要读端的fd计数为`0`），但是读端进行读数据时必须先关闭写端的缓冲文件（即写端的文件描述符计数为`0`），然后才能读取数据

##  0x0B 总结

####     内核态的意义
CPU 为了进行指令权限管控，引入了特权级的概念，CPU 工作在不同的特权级下能够执行的指令和可访问的内存区域是不一样的。计算机上电启动之处，CPU 运行在高特权级下，操作系统（Linux 内核）率先获得了执行权限，在内存设置一块固定区域并将自己的程序代码（内核代码）放了进去，并设定了这一部分内存只有高特权级才能访问。随后，操作系统在创建进程的时候，都会把自己所在的这块内存区域映射到每一个进程地址空间中，这样所有进程都能看到自己的进程空间中被映射的内核的区域，这一块区域是无法直接访问的。通常进入内核态是指：当中断、异常、系统调用等情况发生的时候，CPU 切换工作模式到高特权级模式 `Ring0`，并转而执行位于内核地址空间处的代码

####	全局结构 OR namespace结构
对本文中涉及到的几个进程核心的数据结构做一个小结：

-	`struct upid`：表示进程在特定namespace中的ID（结构）。一个进程在每个可见的namespace中都有一个`upid`（**从内核视角，一个进程必定最终归属于某个namespace，参考`struct pid`的`level`值必然是确定的**），该结构的本质是**进程在特定命名空间中的身份凭证**
-	`struct pid`：是进程的全局标识，一个 `struct pid`代表一个全局唯一的进程实体，包含一个`upid`数组，数组中的每个元素对应一个命名空间。`struct pid`是全局唯一的，但它在不同namespace中的ID（即`upid`中的`nr`）可能不同。`struct pid`结构的本质是**跨命名空间的进程身份凭证**
-	`struct task_struct`：代表一个进程或线程的全局结构，每个进程/线程只有一个task_struct，它是全局的
-	`pid_hash`：是一个全局的哈希表，用于通过（PID数值，namespace）快速查找对应的`upid`，进而找到`struct pid`
-	`struct pid_link`：用于进程与 PID 的关联桥梁，在内核中一个进程可以有多个 PID（PID、TGID、PGID、SID）身份，此外线程组场景下，一个 PID 可以关联多个进程。`pid_link`的本质是`task_struct`与 PID 的多对多关系连接器

TODO

![kernel-process]()

1、每个PID namespace独立的结构

```cpp
// 每个PID命名空间独立拥有的结构
struct pid_namespace;              // 每个命名空间一个独立实例
struct pidmap pidmap[PIDMAP_ENTRIES]; // 每个命名空间独立的位图（进程）

// 每个进程在每个命名空间中的视图
struct upid;                        // 进程在特定命名空间中的PID实例
```

2、全局唯一的结构与全局变量

```cpp
//1. 真正的全局单例（系统级）
// 整个系统只有一个实例
pid_hash;                          // 全局PID查找哈希表
init_pid_ns;                       // 初始PID命名空间（特殊的全局实例）

// 整个内核中只有一个实例（全局变量）
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L45
pid_hash;                          // 全局PID哈希表（系统级单例）

// 每个进程一个实例，但所有进程的实例都在全局链表中
struct task_struct;                // 每个进程一个，全局链表管理
struct pid;                        // 每个进程一个，全局哈希表管理
```

3、每个进程一个，但被全局（结构）管理

```cpp
// 每个进程都有独立的实例，但所有实例在全局数据结构中管理
struct task_struct;                // 通过init_task.tasks全局链表管理
struct pid;                        // 通过pid_hash全局哈希表管理
struct nsproxy;                    // 每个进程的命名空间代理
```

4、进程-namespace映射关系

```cpp
// 描述进程在特定命名空间中的身份（多对多关系）
struct upid;                        // 一个进程在多个命名空间有多个upid
```

对上面列举的结构进行一些实例化的补充：

5、`pid_namespace` namespace维度的结构与操作

`init_pid_ns`是默认内核初始化时就创建的全局，初始namespace

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L71
struct pid_namespace init_pid_ns = {
	.kref = KREF_INIT(2),
	.pidmap = {
		[ 0 ... PIDMAP_ENTRIES-1] = { ATOMIC_INIT(BITS_PER_PAGE), NULL }
	},
	.last_pid = 0,
	.nr_hashed = PIDNS_HASH_ADDING,
	.level = 0,
	.child_reaper = &init_task,
	.user_ns = &init_user_ns,
	.ns.inum = PROC_PID_INIT_INO,
#ifdef CONFIG_PID_NS
	.ns.ops = &pidns_operations,
#endif
};
```

每个 PID namespace在创建时都会分配自己独立的位图（PID在不同namespace中的独立性/进程隔离），这样每个namespace都可以独立维护自己的PID 分配状态，`pid_namespace`自身会形成树状结构（上图）

```cpp
struct pid_namespace {
    ......
    struct pidmap pidmap[PIDMAP_ENTRIES];  // 独立的 PID 位图（数组）
    int last_pid;                          // 最后分配的 PID
    struct task_struct *child_reaper;      // 该命名空间的 init 进程
    ......
    unsigned int level;                     // 命名空间层级
    struct pid_namespace *parent;          // 父命名空间
    ......
};

//struct pidmap表示一个位图页，用于跟踪一组 PID 的分配情况
struct pidmap {
    atomic_t nr _free;
    void *page;
};
```

在`create_pid_namespace`函数中可以观察到上述行为：

```cpp
// 创建新的 PID 命名空间
// https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid_namespace.c#L95
static struct pid_namespace *create_pid_namespace(struct user_namespace *user_ns,
	struct pid_namespace *parent_pid_ns)
{
    struct pid_namespace *ns;
    ......
    // 分配命名空间结构
	ns = kmem_cache_zalloc(pid_ns_cachep, GFP_KERNEL);
	if (ns == NULL)
		goto out_dec;
    
    // 初始化独立的 PID 位图    
	......
	atomic_set(&ns->pidmap[0].nr_free, BITS_PER_PAGE - 1);

	for (i = 1; i < PIDMAP_ENTRIES; i++)
		atomic_set(&ns->pidmap[i].nr_free, BITS_PER_PAGE);

	return ns;
	......
}
```

在每次创建新进程（假设在容器中创建）的时候，该容器所在的namespace level以及在此之上的所有层级都会使用本层级的进行位图用来分配pid，参考上面的`alloc_pid`函数

```cpp
struct pid *alloc_pid(struct pid_namespace *ns)
 {
 	struct pid *pid;
 	struct pid_namespace *tmp;
 	struct upid *upid;
    
    // 从命名空间分配一个 pid 结构体（非整数）
 	pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
	......
    
     // 初始化进程在各级命名空间的 PID，直到全局命名空间（level 为0）为止
	// 调用到alloc_pidmap来分配一个空闲的pid编号
 	// 注意，在每一个命令空间中都需要分配进程号
 	tmp = ns;
 	pid->level = ns->level;	//ns->level保存了当前所在的namespace层级

	// 遍历向上的namespace level
 	for (i = ns->level; i >= 0; i--) {
		// 在该命名空间的PID位图中查找空闲 PID
 		nr = alloc_pidmap(tmp);  
 		if (nr < 0) {
 			retval = nr;
 			goto out_free;
 		}
    
 		pid->numbers[i].nr = nr;
 		pid->numbers[i].ns = tmp;
 		tmp = tmp->parent; //tmp更新为其上一级的namespace指针
 	}
    
	.......
 }
```

跨命名空间的 PID 映射：**注意到`alloc_pid`传入的参数是`struct pid_namespace *ns`，说明创建进程时一定要指定该进程位于哪个PID namespace中，对于该进程对应的`struct pid`结构，其`level`成员是一个固定的值**，那么前文也描述了，一个进程在不同PID namespace下的表示，主要依赖`struct pid`中的`struct upid numbers[1]`这个柔性数组成员：

```cpp
struct pid {
    ......
    unsigned int level;
    struct upid numbers[1];  // 柔性数组，大小为 level+1

	......
};

struct upid {
    int nr;                      // 在该命名空间中的 PID 数值
    struct pid_namespace *ns;    // 对应的命名空间
    struct hlist_node pid_chain; // 哈希链表
};
```

这样，一个进程在`level=3`的namespace中的可能表示如下：

```cpp
// 进程在三个命名空间中的 PID（实例化）
struct pid {
    .level = 2,  // 三级命名空间（0,1,2）
    .numbers = {
        [0] = { .nr = 1000, .ns = &init_pid_ns },     // 全局：PID=1000
        [1] = { .nr = 100,  .ns = &container_pid_ns }, // 容器：PID=100  
        [2] = { .nr = 1,    .ns = &nested_pid_ns }     // 嵌套：PID=1
    }
};
```

关于PID 进程位图的操作，简单描述下查找`find_ge_pid`及遍历`next_pidmap`函数：

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L555
struct pid *find_ge_pid(int nr, struct pid_namespace *ns)
{
	struct pid *pid;

	do {
		// 在指定命名空间中查找精确的 PID
		pid = find_pid_ns(nr, ns);
		if (pid)
			break;	//说明找不到或者已经寻找完成
		// 在该命名空间的位图中查找下一个存在的 PID
		nr = next_pidmap(ns, nr);
	} while (nr > 0);

	return pid;
}

// 遍历指定命名空间的位图
int next_pidmap(struct pid_namespace *pid_ns, unsigned int last)
{
	int offset;
	struct pidmap *map, *end;

	if (last >= PID_MAX_LIMIT)
		return -1;

	offset = (last + 1) & BITS_PER_PAGE_MASK;
	map = &pid_ns->pidmap[(last + 1)/BITS_PER_PAGE];
	end = &pid_ns->pidmap[PIDMAP_ENTRIES];
	for (; map < end; map++, offset = 0) {
		// 跳过未分配的位图页
		if (unlikely(!map->page))
			continue;
		offset = find_next_bit((map)->page, BITS_PER_PAGE, offset);
		if (offset < BITS_PER_PAGE)
			return mk_pid(pid_ns, map, offset);
	}
	return -1;
}
```

6、全局的`task_struct`结构

```cpp
struct task_struct {
	......
	struct pid_link pids[PIDTYPE_MAX]; // 链接到不同类型的 PID 结构
	struct nsproxy *nsproxy; // 指向命名空间代理的指针
	......
};
```

`task_struct`是全局唯一的，每个进程/线程在内核中都有且只有一个 `task_struct`结构，无论它属于哪个namespace，存在于内核的全局链表中，理解`task_struct`的全局性，先看下其创建过程`copy_process`函数（每个进程/线程创建时都会分配一个独立的 `task_struct`）

-	`task_struct->nsproxy`成员：标识了这个`task_struct`是属于哪个namespace的

```cpp
//1. 内核中的定义和分配

static __latent_entropy struct task_struct *copy_process(
					unsigned long clone_flags,
					unsigned long stack_start,
					unsigned long stack_size,
					int __user *child_tidptr,
					struct pid *pid,
					int trace,
					unsigned long tls,
					int node)
{
	int retval;
	struct task_struct *p;
	......
	// 分配新的 task_struct
	p = dup_task_struct(current, node);
	if (!p)
		goto fork_out;

	// 设置命名空间关系
	retval = copy_namespaces(clone_flags, p);
	......
	if (pid != &init_struct_pid) {
		pid = alloc_pid(p->nsproxy->pid_ns_for_children);
		if (IS_ERR(pid)) {
			retval = PTR_ERR(pid);
			goto bad_fork_cleanup_thread;
		}
	}


	if (likely(p->pid)) {
		ptrace_init_task(p, (clone_flags & CLONE_PTRACE) || trace);

		init_task_pid(p, PIDTYPE_PID, pid);
		......
		// 设置 PID link
		attach_pid(p, PIDTYPE_PID);
		nr_threads++;
	}

	......	
}
```

内核为全局`task_struct`维护了全局任务列表即`init_task`，可通过 `init_task` 遍历

```cpp
// 内核维护全局的任务列表（可通过 init_task 遍历）
struct task_struct init_task = INIT_TASK(init_task);

// 所有任务通过 task_list 链接在一起
struct task_struct {
    ......
    struct list_head tasks;        // 全局任务链表节点
    struct list_head thread_group;  // 线程组链表
    ......
};
```

通过namespace（特别是 PID namespace），可以控制进程的可见性，使得每个namespace只能看到一部分进程（即该命名空间内的进程），如何理解？也就是说，**当在容器内的视角查找`task_struct`时，内核会根据当前的namespace进行过滤，只能看到属于本namespace的`task_struct`**

为什么内核要将`task_struct`定义为全局的？从本质上来说，由于CPU的最小调度单元就是`task_struct`，**内核需要使用全局视图进行调度，调度器可以公平调度（调度算法）所有任务，无论其所属于的namespace是哪个**

```cpp
// 调度器需要看到所有任务，无论它们属于哪个命名空间
void schedule(void)
{
    // 从所有可运行任务中选择下一个要运行的任务
    next = pick_next_task(rq);
    // ...
}
```

那么再说明下`task_struct`与namespace命名空间的关系

```cpp
// 1. 通过 nsproxy 关联命名空间
struct task_struct {
    // ...
    struct nsproxy *nsproxy;  // 命名空间代理，指向当前任务的命名空间视图
    // ...
};

struct nsproxy {
    atomic_t count;
    struct uts_namespace *uts_ns;    // UTS 命名空间
    struct ipc_namespace *ipc_ns;    // IPC 命名空间  
    struct mnt_namespace *mnt_ns;    // 挂载命名空间
    struct pid_namespace *pid_ns;    // PID 命名空间（内部包含了本namespace的进程位图关联结构、level层级等关键成员）
    struct net *net_ns;              // 网络命名空间
    // ...
};

//2. 多命名空间支持的实现

// 一个任务可以同时属于多个不同类型的命名空间
struct task_struct container_task = {
    .nsproxy = &container_nsproxy,  // 包含各种命名空间的指针
};

struct nsproxy container_nsproxy = { isValid := re.MatchString(domain)
    .uts_ns = &container_uts_ns,    // 独立的主机名域
    .pid_ns = &container_pid_ns,    // 独立的 PID 空间
    .net_ns = &container_net_ns,    // 独立的网络栈
    .mnt_ns = &container_mnt_ns,    // 独立的文件系统视图
    // ...
};
```

7、`struct upid`：namespace维度的结构

上文已经提到过，`upid`中的 `pid_chain`这个成员是挂载在全局的 `pid_hash`哈希表上的，梳理下这个过程

```cpp
//全局哈希表定义
static struct hlist_head *pid_hash;  // 全局PID哈希表数组

struct upid {
    int nr;                      // 在该命名空间中的 PID 数值
    struct pid_namespace *ns;    // 对应的命名空间
    struct hlist_node pid_chain; // 哈希链表
};
```

那么，`upid`创建及插入哈希表的过程，在`alloc_pid`函数中，当创建新的 `struct pid` 时，将 `upid` 插入全局哈希表：

```cpp
struct pid *alloc_pid(struct pid_namespace *ns)
{
	struct pid *pid;
	enum pid_type type;
	int i, nr;
	struct pid_namespace *tmp;
	struct upid *upid;

	//1、这里已经对 `pid`中的柔性数组成员numbers[]进行内存初始化了
	pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
	......

	tmp = ns;
	pid->level = ns->level;
	for (i = ns->level; i >= 0; i--) {
		nr = alloc_pidmap(tmp);
		if (nr < 0) {
			retval = nr;
			goto out_free;
		}

		// 2、这里本质是初始化upid的成员（nr与namespace）
		pid->numbers[i].nr = nr;
		pid->numbers[i].ns = tmp;
		tmp = tmp->parent;
	}

	......

	//3、upid指针直接指向pid的位置
	upid = pid->numbers + ns->level;

	......
	for ( ; upid >= pid->numbers; --upid) {
		//4、对每个upid，将其插入到全局hashtable pid_hash中
		// key 为：通过namespace与nr符合计算得出
		// pid_hashfn函数计算 upid 应该挂载到哪个哈希桶
		/*
		struct hlist_head *bucket = &pid_hash[hash_bucket];
		hlist_add_head_rcu(&upid->pid_chain, bucket);
		*/
		// 将 upid 挂载到对应哈希桶的链表头部
		hlist_add_head_rcu(&upid->pid_chain,
				&pid_hash[pid_hashfn(upid->nr, upid->ns)]);
		upid->ns->nr_hashed++;
	}

	return pid;
	.......
}
```

这里有个细节是，内核的hashtable 桶链表插入（`hlist_add_head_rcu`函数）是头插法，即新元素会插入到链表的头部，这里后文VFS重复挂载场景会用到

`find_pid_ns`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L365)用于根据`upid`对象获取到其对应的`struct pid`结构：

```cpp
//查找特定命名空间中的PID
struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
{
	struct upid *pnr;

	//遍历hashtable桶的链表（冲突链）
	hlist_for_each_entry_rcu(pnr,
			&pid_hash[pid_hashfn(nr, ns)], pid_chain)
		if (pnr->nr == nr && pnr->ns == ns){
			// 冲突检测（精确匹配 nr 和 ns）
			return container_of(pnr, struct pid,
					numbers[ns->level]);
		}

	return NULL;
}
```

所以，可以利用`find_pid_ns`在容器内遍历查找所有的进程（如在容器内执行 `ps aux` 时），抽象代码如下：

```cpp
// 1. 获取当前任务的 PID 命名空间
struct pid_namespace *ns = current->nsproxy->pid_ns;

// 2. 遍历该命名空间的所有 PID
for (pid_num = 1; pid_num < PID_MAX; pid_num++) {
    // 在全局哈希表中查找该命名空间的 PID
    struct pid *pid = find_pid_ns(pid_num, ns);
    if (pid) {
        // 显示进程信息
        show_process_info(pid);
    }
}
```

8、`init_task`：全局`task_struct`链表

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/fork.c#L1856
static __latent_entropy struct task_struct *copy_process(
					unsigned long clone_flags,
					unsigned long stack_start,
					unsigned long stack_size,
					int __user *child_tidptr,
					struct pid *pid,
					int trace,
					unsigned long tls,
					int node)
{
	......
	if (likely(p->pid)) {
		init_task_pid(p, PIDTYPE_PID, pid);
		if (thread_group_leader(p)) {
			init_task_pid(p, PIDTYPE_PGID, task_pgrp(current));
			init_task_pid(p, PIDTYPE_SID, task_session(current));

			if (is_child_reaper(pid)) {
				ns_of_pid(pid)->child_reaper = p;
				p->signal->flags |= SIGNAL_UNKILLABLE;
			}

			p->signal->leader_pid = pid;
			p->signal->tty = tty_kref_get(current->signal->tty);
		
			p->signal->has_child_subreaper = p->real_parent->signal->has_child_subreaper ||
							 p->real_parent->signal->is_child_subreaper;
			list_add_tail(&p->sibling, &p->real_parent->children);

			// 如果本进程是groupleader，那么会插入到全局的init_task中
			list_add_tail_rcu(&p->tasks, &init_task.tasks);
			attach_pid(p, PIDTYPE_PGID);
			attach_pid(p, PIDTYPE_SID);
			__this_cpu_inc(process_counts);
		} else {
			current->signal->nr_threads++;
			atomic_inc(&current->signal->live);
			atomic_inc(&current->signal->sigcnt);
			list_add_tail_rcu(&p->thread_group,
					  &p->group_leader->thread_group);
			list_add_tail_rcu(&p->thread_node,
					  &p->signal->thread_head);
		}
		// 重要：所有的进程，都会把自己的task_struct挂到自己对应的struct pid的PIDTYPE_PID类型上面
		attach_pid(p, PIDTYPE_PID);
		nr_threads++;
	}
	......
}
```

####	`struct pid`与`struct task_struct`中的`PIDTYPE_MAX`链表数组

这里解释下`pid.tasks`与`task_struct.pids`这两个链表的意义是什么？

1.	一个`task_struct`有多个不同类型的进程标识符（PID、TGID、PGID、SID），用于不同的管理目的（线程控制、进程组控制、会话控制）
2.	一个PID关联多个进程（`task_struct`），如一个TGID（线程组ID）关联多个线程，实现了多线程进程的模型，所有线程共享相同的TGID

```cpp
enum pid_type {
    PIDTYPE_PID,    // 进程的PID（每个任务唯一）
    PIDTYPE_TGID,   // 线程组ID
    PIDTYPE_PGID,   // 进程组ID
    PIDTYPE_SID,    // 会话ID
	PIDTYPE_MAX
};

struct pid {
	......
    atomic_t count;                          // 引用计数
    unsigned int level;                      // 命名空间层级深度
    struct hlist_head tasks[PIDTYPE_MAX];   // 不同类型的任务链表（4个链表头）
    struct rcu_head rcu;                    // RCU 回调
    struct upid numbers[1];                 // 柔性数组，多命名空间映射
	......
};

struct pid_link {
    struct hlist_node node;   // 链表节点
    struct pid *pid;         // 指向pid结构
};

struct task_struct {
    ......
    struct pid_link pids[PIDTYPE_MAX];
    ......
};
```

1、`pid.tasks`的作用

先看下`struct pid`中的`struct hlist_head tasks[PIDTYPE_MAX]`这个链表头的作用，回顾下如下进/线程类型：

-	`PIDTYPE_PID`：进程ID（线程ID），内核中每个`task_struct`（包括线程）都有一个唯一的PID
-	`PIDTYPE_TGID`：线程组ID（通常在用户态呈现的进程ID，即主线程的PID），在一个多线程应用程序中，所有线程共享同一个TGID，即主线程的PID。用户态进程的PID通常指的是TGID
-	`PIDTYPE_PGID`：进程组ID，一个进程组包含一个或多个进程（或线程），它们可以接收同一个信号。通常，一个进程组由shell管道中的多个进程组成
-	`PIDTYPE_SID`：会话ID，一个会话包含一个或多个进程组，通常代表一个用户登录会话

直观上看，`pid.tasks`的作用是将使用同一个pid（在不同类型下）的所有`task_struct`链接起来，考虑到内核对进程的如下管理场景：

-	通过线程组ID（`TGID`）查找组内所有线程：当内核向整个线程组发送信号时，需要找到该线程组的所有线程。这时，通过TGID类型的`pid`结构中的`tasks[PIDTYPE_TGID]`链表，可以快速遍历所有线程
-	进程组和会话管理：通过进程组ID（`PGID`）可以找到进程组中的所有进程，通过会话ID（`SID`）可以找到会话中的所有进程
-	对于普通的PID，通常一个pid只对应一个`task_struct`（因为每个线程有唯一的PID），但设计上仍然使用链表，以便统一处理（上面描述的`attach_pid(p, PIDTYPE_PID)`方法）
-	当将一个`task_struct`与一个pid关联时，会将`task_struct`的`pid_link[type]`节点插入到其对应的`pid`的`tasks[type]`链表中，参考[`copy_process`](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/fork.c#L1857)中的对`attach_pid`调用

所以，本质上`struct pid`中的 `tasks[PIDTYPE_MAX]`链表头用于实现反向映射，即通过 PID 值快速找到所有关联的`task_struct`

```cpp
void attach_pid(struct task_struct *task, enum pid_type type)
{
	struct pid_link *link = &task->pids[type];
	hlist_add_head_rcu(&link->node, &link->pid->tasks[type]);
}
```

上述四种链表的意义如下：
-	`tasks[PIDTYPE_PID]`：特定线程的`task_struct`，一个 PID 值对应一个具体的线程；对于单线程进程，此链表只有一个`task_struct`；对于多线程进程，每个线程有独立的 PID，各自对应一个`task_struct`
-	`tasks[PIDTYPE_TGID]`：线程组的所有`task_struct`，一个 TGID 对应一个线程组的所有线程。对多线程进程而言，此链表包含该进程的所有线程
-	`tasks[PIDTYPE_PGID]`：进程组的所有`task_struct`，即一个 PGID 对应一个进程组的所有进程
-	`tasks[PIDTYPE_SID]`：会话的所有`task_struct`，一个 SID 对应一个会话的所有进程

2、`task_struct.pids`的作用，`pid_link`的`node`成员的作用上面已经介绍（作为链表的节点），而`task_struct`中的 `pid_link`的 `pid`指针成员的作用，指向的正是该`task_struct`在不同场景下所表现的特定 PID 身份，即一个有着多重身份的执行实体

```cpp
struct task_struct {
    // ...
    struct pid_link pids[PIDTYPE_MAX];  // 4个不同的身份标识
    // ...
};
```

-	`pids[PIDTYPE_PID]`：个体身份，指向该`task_struct`作为独立线程的身份，如`thread->pids[PIDTYPE_PID].pid = &individual_pid`，可通过`ps -eLf`中的 `LWP`列（轻量级进程ID）查看，是在系统中唯一标识这个特定的执行线程
-	`pids[PIDTYPE_TGID]`：线程组身份，指向该`task_struct`所属线程组（进程）的身份，如`thread->pids[PIDTYPE_TGID].pid = &thread_group_pid`，可通过`ps aux` 中的 PID 查看，多线程进程的所有线程共享相同的 TGID，用于标识这个线程属于哪个进程
-	`pids[PIDTYPE_PGID]`：进程组身份，指向该`task_struct`所属进程组的身份，如`thread->pids[PIDTYPE_PGID].pid = &process_group_pid`，常用于作业控制中的进程组，用于 `kill -STOP -PGID` 等操作，标识这个线程属于哪个作业
-	`pids[PIDTYPE_SID]`：会话身份，指向该`task_struct`所属会话的身份，如`thread->pids[PIDTYPE_SID].pid = &session_pid`，常用于终端会话管理，用于控制终端关联，标识这个线程属于哪个登录会话

这样设计使得内核可以从不同维度来获取到某个`task_struct`对应的`struct pid`结构信息，如有如下不同场景：

-	调度器视角：个体身份（PID），用于时间片分配、CPU亲和性等，`task_struct`是个体线程
-	进程管理器视角：线程组身份（TGID），用于进程树显示、进程信号传递，`task_struct`是进程成员
-	作业控制视角：进程组身份（PGID），用于后台作业管理、终端控制，`task_struct`是作业组成员
-	会话管理器视角：会话身份（SID），用于登录会话、终端关联，`task_struct`是会话成员

内核提供了`task_struct`的身份查询函数：

```cpp
// 获取任务在不同场景下的 PID 值
pid_t task_pid(struct task_struct *task) {
    return pid_nr(task->pids[PIDTYPE_PID].pid);    // 个体身份
}

pid_t task_tgid(struct task_struct *task) {
    return pid_nr(task->pids[PIDTYPE_TGID].pid);   // 线程组身份
}

pid_t task_pgrp(struct task_struct *task) {
    return pid_nr(task->pids[PIDTYPE_PGID].pid);  // 进程组身份
}

pid_t task_session(struct task_struct *task) {
    return pid_nr(task->pids[PIDTYPE_SID].pid);   // 会话身份
}
```

再次强调一下，`task_struct`中的 `pid_link`的 `pid`指针指向的正是该 `task_struct`本身的身份标识，`struct pid_link pids[PIDTYPE_MAX]`应该理解为这个`task_struct`拥有的身份标识，是这个`task_struct`自身在不同维度下的身份

既然存在上面描述的这种关系，那么很容易想到，可以通过`task_struct`的`pid_link`找到对应类型的`pid`对象（如`PIDTYPE_PGID`），找到这个`pid`对象之后，再使用该`pid`对象的`struct hlist_head tasks[PIDTYPE_MAX]`，遍历链表，就可以找到该进程组的所有的`task_struct`了，这也是 Linux 进程组管理的核心机制，即通过 `task_struct`的 `pid_link`找到对应的 `struct pid`，然后使用该 `pid`的 `tasks[PIDTYPE_PGID]`链表，确实可以找到该进程组的所有 `task_struct`，参考内核函数`kill_orphaned_pgrp`的[实现](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/exit.c#L392)

小结一下，**一个`struct pid`的链表包含多个`task_struct`，这些`task_struct`各自的`pids`数组指向多个不同的`struct pid`**

3、几个例子

-	`kill_pgrp`
-	`disassociate_ctty`

先看`kill_pgrp`的实现，`kill_pgrp`传入的参数`pid`，就是`PIDTYPE_PGID`类型对应的`struct pid`对象，其实现也很明显，遍历`PIDTYPE_PGID`链表（`tasks`），对每个链表上的`task_struct`都执行`group_send_sig_info`操作

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/signal.c#L1470
// 向整个进程组发送信号
int kill_pgrp(struct pid *pid, int sig, int priv)
{
	int ret;

	read_lock(&tasklist_lock);
	ret = __kill_pgrp_info(sig, __si_special(priv), pid);
	read_unlock(&tasklist_lock);

	return ret;
}

int __kill_pgrp_info(int sig, struct siginfo *info, struct pid *pgrp)
{
	struct task_struct *p = NULL;
	int retval, success;

	success = 0;
	retval = -ESRCH;
	//遍历进程组的所有线程
	do_each_pid_task(pgrp, PIDTYPE_PGID, p) {
		int err = group_send_sig_info(sig, info, p);
		success |= !err;
		retval = err;
	} while_each_pid_task(pgrp, PIDTYPE_PGID, p);
	return success ? 0 : retval;
}

#define do_each_pid_task(pid, type, task)				\
	do {								\
		if ((pid) != NULL)					\
			hlist_for_each_entry_rcu((task),		\
				&(pid)->tasks[type], pids[type].node) {
#define while_each_pid_task(pid, type, task)				\
				if (type == PIDTYPE_PID)		\
					break;				\
			}						\
	} while (0)

#define do_each_pid_thread(pid, type, task)				\
	do_each_pid_task(pid, type, task) {				\
		struct task_struct *tg___ = task;			\
		for_each_thread(tg___, task) {

#define while_each_pid_thread(pid, type, task)				\
		}							\
		task = tg___;						\
	} while_each_pid_task(pid, type, task)
```

注意上面这几个宏定义用于遍历进/线程，稍微拆解一下：

-	`do_each_pid_task`用于遍历PID对应的任务，先检查pid是否为空，不为空时使用RCU安全的方式遍历哈希链表中的任务，`type`参数指定任务类型（进程、线程组、进程组等）
-	`while_each_pid_task`用于结束遍历，如果是进程ID类型（`PIDTYPE_PID`），直接跳出循环，因为一个PID只对应一个任务，不需要继续遍历
-	`do_each_pid_thread`用于遍历PID对应的所有线程，先找到线程组领导者（主线程），然后遍历该线程组的所有线程
-	`while_each_pid_thread`用于结束线程遍历，恢复`task`变量为线程组领导者，最后调用结束线程遍历宏

再看下`disassociate_ctty`的[实现](https://elixir.bootlin.com/linux/v4.11.6/source/drivers/tty/tty_io.c#L864)，终端会话管理

```cpp
//// 当终端断开时，向整个会话发送 SIGHUP
void disassociate_ctty(int on_exit)
{
	struct tty_struct *tty;

	......
	if (tty) {
		if (on_exit && tty->driver->type != TTY_DRIVER_TYPE_PTY) {
			tty_vhangup_session(tty);
		} else {
			// 获取tty所在的pgrp的struct pid对象
			struct pid *tty_pgrp = tty_get_pgrp(tty);
			if (tty_pgrp) {
				// 遍历组进程，并发送SIGHUP信号
				kill_pgrp(tty_pgrp, SIGHUP, on_exit);
				if (!on_exit)
					kill_pgrp(tty_pgrp, SIGCONT, on_exit);
				put_pid(tty_pgrp);
			}
		}
		tty_kref_put(tty);

	} else if (on_exit) {
		struct pid *old_pgrp;
		spin_lock_irq(&current->sighand->siglock);
		old_pgrp = current->signal->tty_old_pgrp;
		current->signal->tty_old_pgrp = NULL;
		spin_unlock_irq(&current->sighand->siglock);
		if (old_pgrp) {
			kill_pgrp(old_pgrp, SIGHUP, on_exit);
			kill_pgrp(old_pgrp, SIGCONT, on_exit);
			put_pid(old_pgrp);
		}
		return;
	}

	......

	// 获取current对应的PIDTYPE_SID的pid对象
	session_clear_tty(task_session(current));
	read_unlock(&tasklist_lock);
}

static inline struct pid *task_session(struct task_struct *task)
{
	return task->group_leader->pids[PIDTYPE_SID].pid;
}

static void session_clear_tty(struct pid *session)
{
	struct task_struct *p;
	// 遍历pid对象的PIDTYPE_SID链表头，执行proc_clear_tty操作
	do_each_pid_task(session, PIDTYPE_SID, p) {
		proc_clear_tty(p);
	} while_each_pid_task(session, PIDTYPE_SID, p);
}
```

4、数量（关系）

从`struct pid`的视角来看，`tasks`这个hash表（链表头）

```cpp
enum pid_type {
	PIDTYPE_PID, // 进程的PID（每个任务唯一）
	PIDTYPE_TGID, // 线程组ID
	PIDTYPE_PGID, // 进程组ID
	PIDTYPE_SID, // 会话ID
	PIDTYPE_MAX
};

struct pid {
	......
	struct hlist_head tasks[PIDTYPE_MAX]; // 不同类型的任务链表（4个链表头）
	
	......
};
```

现在讨论下如上`struct pid`结构中，tasks数组中的`PIDTYPE_MAX`个元素（链表头），一般会对应多少个`task_struct`？这四个代表了以该 PID 为中心的不同角色的集合，每个链表头通常对应的 `task_struct` 数量如下：

-	`PIDTYPE_PID`（进程 ID），对应数量为`1` 个。在内核中，每个 `task_struct`（无论是进程/线程）都有一个全局唯一的 PID。对于一个具体的 `struct pid` 结构体，它代表了一个具体的数值。在这个层级上，一个 PID 数值只能指向一个唯一的 `task_struct`。正常情况下，这个链表里永远只有一个元素
-	`PIDTYPE_TGID`（线程组 ID），对应数量为`1` 个（线程组的主线程），TGID 即常说的进程 ID。虽然一个线程组内有很多线程，但内核规定**只有线程组的 Leader（主线程）**会挂载到 `struct pid` 的 `PIDTYPE_TGID` 链表上，如此设计是为了通过该 PID 快速找到整个进程的代表
-	`PIDTYPE_PGID`（进程组 ID），对应数量可能多个（属于该进程组的所有进程），由于内核中一个进程组可以包含多个进程（如通过`|`连接的命令 `ls | grep`），当一个进程成为组长（Process Group Leader）时，所有属于该组的进程的 `task_struct->pids[PIDTYPE_PGID]` 都会挂载到组长所关联的这个 `struct pid` 的 `tasks[PIDTYPE_PGID]` 链表上
-	`PIDTYPE_SID`（会话 ID），对应数量可能多个（属于该会话的所有进程），一个会话（Session）包含多个进程组，类似于 PGID，所有属于该会话的进程都会将其 `task_struct` 挂载到会话首进程（Session Leader）对应的 `struct pid` 的 `tasks[PIDTYPE_SID]` 链表上

然后，从`struct task_struct`视角来看，`pid_link`的链表节点包含的`node`与`pid`成员如下：

```cpp
struct pid_link {
	struct hlist_node node; // 链表节点
	struct pid *pid; // 指向pid结构，每个 link 指向一个 struct pid 实例
}

struct task_struct {
	......
	// 在 task_struct 中，pids[PIDTYPE_MAX] 并不是存放 PID 数字
	// 而是存放了 struct pid_link
	// 每个 link 指向一个 struct pid 实例
	struct pid_link pids[PIDTYPE_MAX];
	......
};
```

在 `task_struct.pids` 数组中，每个元素（链表节点）必然且只能对应 `1` 个 `struct pid`。由于 `task_struct.pids` 是一个长度为 `PIDTYPE_MAX`（通常为 `4`）的数组，这意味着一个`task_struct`（线程/进程）同时关联着 `4` 个不同的身份标志：

-	`pids[PIDTYPE_PID]`：对应`task_struct`自身的标识，指向一个唯一的 `struct pid`，其数值就是在 `top/ps` 中看到的 LWP（线程ID）
-	`pids[PIDTYPE_TGID]`：对应`task_struct`所属的线程组（即传统意义上的进程），如果是主线程，它指向的 `struct pid` 与上面的 PID 类型相同；如果是子线程，它指向主线程的那个 `struct pid`
-	`pids[PIDTYPE_PGID]`：对应任务所属的进程组，指向进程组组长（Process Group Leader）的那个 `struct pid`
-	`pids[PIDTYPE_SID]`：对应任务所属的会话，指向会话首进程（Session Leader）的那个 `struct pid`

注意，上面四个类型数量都是一个，为什么数组里是 `4` 个 link呢？虽然一个进程有四种身份，但这四种身份可能指向同一个 `struct pid` 实例，也可能指向不同的。内核只需通过一个 `struct pid` 就能管理成百上千个属于同一组的任务，极大地节省了内存并提高了查找效率

最后一个问题，`task_struct.pids[xxxx].node`这个成员到底是什么作用？从其定义`struct hlist_node node`来看，非指针，是一个结构体对象，大概率是通过某些关系定位到这个`node`成员后，然后使用`container_of`机制获取到`task_struct`的指针地址

回到`struct pid`的定义，瞬间就清晰了，**`struct hlist_head tasks[PIDTYPE_MAX]`这里面的链表头，其链表成员就是`task_struct.pids[xxxx].node`（类型为`struct hlist_node`）**，参考下图：

![pid-relation-with-task_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/1/pid-relation-task_struct.png)

内核这个设计非常精妙，可以理解为`struct pid` 是master，`task_struct` 是worker，worker的 `node` 成员放置在master的 `tasks` 插槽里。内存中的链接关系如下（链表头指向 node时）

-	`struct pid` 结构体对象，它里面有一个 `tasks` 数组
-	其成员`tasks[PIDTYPE_PGID]` 是一个 `struct hlist_head`（指针）
-	这个指针存储的地址，正是某个 `task_struct` 内部嵌套的 `pids[PIDTYPE_PGID].node` 的地址

内核如此设计的另一个目的是解决了**一个进程拥有多重身份**的问题，思考下，一个 `task_struct` 代表一个进程：

-	它想当自己：它把 `pids[PIDTYPE_PID].node` 挂在它自己的 `struct pid` 上
-	它想加入进程组：它把 `pids[PIDTYPE_PGID].node` 挂在组长进程的 `struct pid` 上

```cpp
struct pid {
	......
	struct hlist_head tasks[PIDTYPE_MAX]; // 不同类型的任务链表（4个链表头）
	...... 
};
```

-	`sturct pid`的`pid.tasks[PID_TYPE]`这个链表头，实际上是指向 `task_struct.pids[PID_TYPE].node`
-	对于进程组而言，`sturct pid`的`pid.tasks[PGID_TYPE]`这个链表头，实际上是指向该`struct pid`对应的pgid的 `task_struct.pids[PGID_TYPE].node`
-	`sturct pid`的`pid.tasks[PGID_TYPE]`这个链表头，指向该`struct pid`对应的pgid的 `task_struct.pids[PGID_TYPE].node`，对于`struct pid`与`task_struct.pids[PGID_TYPE].pid`，这两个`struct pid`可能不是一个pid

在上文介绍过的`pid_task`函数实现，就利用了上面的技巧

```cpp
struct task_struct *pid_task(struct pid *pid, enum pid_type type)
{
 	struct task_struct *result = NULL;
 	if (pid) {
 		struct hlist_node *first;
		// 1. 从 pid->tasks[type] 拿到第一个 node 的地址
		// struct hlist_node *first = pid->tasks[type].first;
		// 2. 重点：根据 node 的地址，减去它在 task_struct 里的偏移量，还原出 task 的首地址
		// struct task_struct *task = container_of(first, struct task_struct, pids[type].node);
 		first = rcu_dereference_check(hlist_first_rcu(&pid->tasks[type]),
 					      lockdep_tasklist_lock_is_held());
 		if (first){
 			result = hlist_entry(first, struct task_struct, pids[(type)].node);
		}
 	}
 	return result;
}
```

####	应用：task_tgid_vnr(current)
`task_tgid_vnr`就是通过上面的`struct pid`对应的链条，一步步获取到最终的pid数字，该函数是用于获取TGID（线程组 ID，即用户态看到的进程 PID），核心代码如下：

```cpp
static inline pid_t task_tgid_vnr(struct task_struct *tsk)
{
    // 第一步：拿到该任务 TGID 对应的 struct pid
    // 第二步：将这个 struct pid 转换成当前命名空间下的数字
    return pid_vnr(task_tgid(tsk));
}

// 偷懒的方法：直接通过group_leader对应的task_struct结构，拿到对应的struct pid对象指针
// group_leader->pids[PIDTYPE_PID].pid：表示group_leader对应的PIDTYPE_PID的pid结构
static inline struct pid *task_tgid(struct task_struct *task)
{
	return task->group_leader->pids[PIDTYPE_PID].pid;
}

//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L513
pid_t pid_vnr(struct pid *pid)
{
	return pid_nr_ns(pid, task_active_pid_ns(current));
}

//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L544
struct pid_namespace *task_active_pid_ns(struct task_struct *tsk)
{
	return ns_of_pid(task_pid(tsk));
}

//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/sched.h#L1056
static inline struct pid *task_pid(struct task_struct *task)
{
	//获取task_struct本身的那个pid结构
	return task->pids[PIDTYPE_PID].pid;
}

// ns_of_pid：这个也很重要
// 直接根据pid所在的level，返回所属level的那个pid_namespace结构体
static inline struct pid_namespace *ns_of_pid(struct pid *pid)
{
	struct pid_namespace *ns = NULL;
	if (pid)
		ns = pid->numbers[pid->level/*这里肯定是最深层级的那里的*/].ns;
	return ns;
}
```

上面实现了转换的第一步，定位 `struct pid`，内核通过 `task_pid_ptr` 这种内部逻辑，直接访问 `pids` 数组。对于 TGID，它会去找 `task->pids[PIDTYPE_TGID].pid`

-	如果是主线程：这个指针指向它自己的 `struct pid`
-	如果是子线程：这个指针指向它所属主线程的 `struct pid`

下面是第二步，即数字转换（关键的命名空间处理）

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/kernel/pid.c#L499
pid_t pid_nr_ns(struct pid *pid, struct pid_namespace *ns)
{
	struct upid *upid;
	pid_t nr = 0;
	// 只要 level 没超，且 namespace 匹配
	if (pid && ns->level <= pid->level) {
		// 关键：柔性数组 numbers 登场
		upid = &pid->numbers[ns->level];
		if (upid->ns == ns)
			nr = upid->nr;	// 拿到数字
	}
	return nr;
}
```

从`task_tgid_vnr`的实现，内核设计了三层解耦：
    
-	身份解耦（Role Separation）： 通过 `pids[PIDTYPE_MAX]` 数组，一个 `task_struct` 可以同时扮演四个角色。内核不需要在 `task_struct` 里存四个数字，只需要存四个链接
-	空间解耦（Namespace Isolation）： `struct pid` 里的 `numbers[1]` 是个柔性数组。这意味着同一个进程在宿主机看是 PID 1024，在 Docker 容器里看可能是 PID 1。通过 `ns->level` 作为下标，一次内存寻址就能找到对应环境下的数字
-	生存期解耦（Lifetime Management）：`struct pid` 拥有自己的引用计数（atomic_t count）。即便进程退出了（`task_struct` 释放），只要还有进程组里的其他进程指向这个 PID，这个 `struct pid` 结构就不会消失

##  0x0C  参考
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
-	[Linux Namespace分析——mnt namespace的实现与应用](https://hustcat.github.io/namespace-implement-1/)
-	[容器技术原理(五)：文件系统的隔离和共享](https://waynerv.com/posts/container-fundamentals-filesystem-isolation-and-sharing/#contents:%E4%BD%BF%E7%94%A8-pivot_root-%E6%88%96-chroot-%E5%88%87%E6%8D%A2%E6%A0%B9%E7%9B%AE%E5%BD%95)
-	[写时复制 Copy-on-write](https://imageslr.com/2020/copy-on-write.html)