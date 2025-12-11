---
layout:     post
title:  Linux 内核之旅：基础知识
subtitle:
date:       2024-10-05
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
-   双向链表
-   hash 链表
-   队列：Priority sorted lists used for mutexes, drivers, etc
-   rbtree：Red-Black trees are used for scheduling, virtual memory management, to track file descriptors and directory entries,etc

本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

####    offset_of 机制

```cpp
// 说明：获得结构体 (TYPE) 的变量成员 (MEMBER) 在此结构体中的偏移量
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

![offset_of](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/datastructure/offset_of.png)

`TYPE` 是结构体，代表整体，而 `MEMBER` 是结构体成员，`offsetof` 的本质是已知整体和该整体中某一个部分，计算该部分在整体中的偏移，详细步骤如下：

1. `((TYPE *)0 )`：将零转型为 `TYPE` 类型指针，即 `TYPE` 类型的指针的地址是 `0`
2. `((TYPE *)0)->MEMBER`：访问结构中的数据成员
3. `&(( (TYPE *)0 )->MEMBER )`：取出数据成员的地址，由于 `TYPE` 的地址是 `0`，这里获取到的地址就是相对 `MEMBER` 在 `TYPE` 中的偏移
4. `(size_t)(&(((TYPE*)0)->MEMBER))`：结果转换类型。对于 `32` 位系统而言，`size_t` 是 `unsigned int` 类型；对于 `64` 位系统而言，`size_t` 是 `unsigned long` 类型

####    container_of 机制

```cpp
// 说明：根据结构体 (type) 变量中的域成员变量 (member) 的指针 (ptr) 来获取指向整个结构体变量的指针
#define container_of(ptr, type, member) ({          \
    const typeof(((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

同样的，已知成员 `MEMBER` 及 `MEMBER` 的指针地址，推算出结构体的开始地址，即已知整体和该整体中某一个部分，要根据该部分的地址，计算出整体的地址

![container_of](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/datastructure/container_of.png)

1.  `typeof(( (type *)0)->member )`：取出 `member` 成员的变量类型
2.  `const typeof(((type *)0)->member ) *__mptr = (ptr)`：定义变量 `__mptr` 指针，并将 `ptr` 赋值给 `__mptr`。经过这一步，`__mptr` 为 `member` 数据类型的常量指针，其指向 `ptr` 所指向的地址
3.  `(char *)__mptr`：将 `__mptr` 转换为字节型指针
4.  `offsetof(type,member))`：就是获取 `member` 成员在结构体 `type` 中的位置偏移
5.  `(char *)__mptr - offsetof(type,member))`：就是用来获取结构体 `type` 的指针的起始地址（为 `char *` 型指针）
6.  `(type *)( (char *)__mptr - offsetof(type,member) )`：就是将 `char *` 类型的结构体 `type` 的指针转换为 `type *` 类型的结构体 `type` 的指针

##  0x01    双向链表
内核实现的 double-linklist 和普通的 linklist 有差异，普通 linklist 实现缺点就是通用性不够好，而内核的 linklist 巧妙解决了这个问题，不是将用户数据保存在 linklist 节点中，而是将 linklist 节点保存在用户数据中。linux 的 linklist 节点只有 `2` 个指针成员 (`prev` 和 `next`)，因此 linklist 的节点将独立于用户数据之外，便于实现 linklist 的共同操作

内核 linklist 中的最大问题是怎样通过链表的节点来取得用户数据？linux 的 linklist 节点 (node) 中没有包含用户的用户如 `data1`，`data2` 等，所以 Linux 中双向链表的使用思想也变成了将双向链表节点嵌套在其它的结构体（比如 linux 进程管理的 `struct task_struct`[结构](https://github.com/torvalds/linux/blob/master/include/linux/sched.h#L1055)）中；在遍历链表的时候，根据双链表节点的指针获取 **它所在结构体的指针**（使用 `container_of` 宏），从而再获取数据或者该结构体的其他成员

```cpp
struct list_head {
	struct list_head *next, *prev;
};
```

![linklist](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/datastructure/linklist2.png)

####    使用
1、结构定义

普通 linklist：

```cpp
struct student{// 结构体 student 本身就是一个链表节点
	struct student *prev;
	struct student *next;
	char name[64];
	char id[64];
}
```

内核 linklist：

```cpp
struct student{
	struct list_head list_node;// 链表节点
	char name[64];
	char id[64];
};
```

2、遍历链表

参考 [`linklist.c`](https://github.com/pandaychen/ebpf-allinone/blob/main/kernel/linklist.c#L56)

####	内核的相关应用
-	[`rt_prio_array`](https://github.com/torvalds/linux/blob/master/kernel/sched/sched.h#L313)：Linux CPU调度中用来表示实时调度实体所属的实时运行队列

##  0x02    hash 链表
`hlist_head` 和 `hlist_node` 用于内核 hash 表，分别表示列表头（数组中的一项）和列表头所在双向链表中的某项，结构定义及示意图如下:

```cpp
struct hlist_head {
	struct hlist_node *first;
};

struct hlist_node {
	struct hlist_node *next, **pprev;
};
```

![hlist](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/datastructure/list_head-and-hlist_node-2.png)

如上图 hash 表（数组）的元素类型为 `struct hlist_head`，以 `hlist_head` 为链表头的链表为冲突链，节点类型为 `struct hlist_node`。hash_table 的列表头仅存放一个 `first` 指针，指向的是对应冲突链表的头结点，不设计 `tail` 指针是为了减少内存空间（当数组的 size 很大时）。

此外，注意到 `hlist_node.pprev` 是一个指向指针的指针（保存前一个节点的 `next` 成员地址），目的是在删除链表头结点的时候，`pprev` 这个设计无需判断删除的节点是否为头结点（如果是普通双向 linklist 的设计，那么删除头结点之后，`hlist_head` 中的 `first` 指针需要指向新的头结点），参考 `__hlist_del` 的实现

```cpp
static inline void hlist_add_head(struct hlist_node *n, struct hlist_head *h)
{
	struct hlist_node *first = h->first;
	n->next = first; // 新节点的 next 指针指向原头结点
	if (first)
		first->pprev = &n->next;// 原头结点的 pprev 指向新节点的 next 字段
	WRITE_ONCE(h->first, n);//first 指针指向新的节点（更换了头结点）
	n->pprev = &h->first;// 此时 n 是链表的头结点, 将它的 pprev 指向 list_head 的 first 字段
}

static inline void __hlist_del(struct hlist_node *n)
{
	struct hlist_node *next = n->next;
	struct hlist_node **pprev = n->pprev;

	WRITE_ONCE(*pprev, next); // pprev 指向的是前一个节点的 next 指针, 当该节点是头节点时指向 hlist_head 的 first, 两种情况下不论该节点是一般的节点还是头结点都可以通过这个操作删除掉所需删除的节点
	if (next)
		next->pprev = pprev;// 使删除节点的后一个节点的 pprev 指向删除节点的前一个节点的 next 字段，节点成功删除
}
```

和 `list_head` 一样，`hlist_node` 通常也是嵌套在其他结构中的，即 `hlist_entry` 函数用于根据 `hlist_node` 找到其外层结构体

```cpp
#define hlist_entry(ptr, type, member) container_of(ptr,type,member)
```

####    内核的应用
内核中用 `hlist` 来实现 hash table，如下：

```BASH
[root@VM-X-X-tencentos kernel]# dmesg | grep "hash table entries"
[0.039306] PV qspinlock hash table entries: 256 (order: 0, 4096 bytes, linear)
[0.041316] Dentry cache hash table entries: 2097152 (order: 12, 16777216 bytes, linear)
[0.042294] Inode-cache hash table entries: 1048576 (order: 11, 8388608 bytes, linear)
[0.118822] Mount-cache hash table entries: 32768 (order: 6, 262144 bytes, linear)
[0.118822] Mountpoint-cache hash table entries: 32768 (order: 6, 262144 bytes, linear)
[0.129682] futex hash table entries: 2048 (order: 5, 131072 bytes, linear)
[0.298401] VFS: Dquot-cache hash table entries: 512 (order 0, 4096 bytes)
[0.306529] IP idents hash table entries: 262144 (order: 9, 2097152 bytes, linear)
[0.309459] tcp_listen_portaddr_hash hash table entries: 8192 (order: 5, 131072 bytes, linear)
[0.309480] Table-perturb hash table entries: 65536 (order: 6, 262144 bytes, linear)
[0.309509] TCP established hash table entries: 1048576 (order: 11, 8388608 bytes, vmalloc hugepage)
[0.310554] TCP bind hash table entries: 65536 (order: 9, 2097152 bytes, linear)
[0.311066] MPTCP token hash table entries: 16384 (order: 6, 393216 bytes, linear)
[0.311098] UDP hash table entries: 8192 (order: 6, 262144 bytes, linear)
[0.311134] UDP-Lite hash table entries: 8192 (order: 6, 262144 bytes, linear)
```

####	进程管理：upid hash table
回想下，在 linux 进程管理 `struct pid/upid`[结构](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pid.h#L50) 中就使用了 hash linklist

```cpp
struct upid {
	/* Try to keep pid_chain in the same cacheline as nr for find_vpid */
	int nr;
	struct pid_namespace *ns;
	struct hlist_node pid_chain;        //hash list 中的 node
};

struct pid
{
	atomic_t count;
	unsigned int level;
	/* lists of tasks that use this pid */
	struct hlist_head tasks[PIDTYPE_MAX];   // 数组头节点
	struct rcu_head rcu;
	struct upid numbers[1];		//柔性数组，可扩展为多个
};
```

![pid_hash](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/pid_hash.png)

如上图展示了当`pid.upid`长度为`1`时的内存布局，`upid.pid_chain`作为hash list中的node节点

![pid.tasks_hlist_head](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/linux-process/struct_pid_hlist_head.png)

上图展示了，`struct pid`结构体中的`struct hlist_head tasks[PIDTYPE_MAX]`成员，这是一个hash list 头节点，其冲突链上的每个`struct hlist_node`节点，对应着`struct task_struct`结构体中的`struct pid_link pids[PIDTYPE_MAX]`成员，而`pid_link`定义为如下：

```cpp
struct pid_link
{
	struct hlist_node node;
	struct pid *pid;
};
```

####	常用操作函数/宏

1、`hlist_for_each_entry_rcu`：用于在 RCU 读锁的保护下，无锁地遍历该哈希桶中的单向链表

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/rculist.h#L584
#define hlist_for_each_entry_rcu(pos, head, member)			\
	for (pos = hlist_entry_safe (rcu_dereference_raw(hlist_first_rcu(head)),\
			typeof(*(pos)), member);			\
		pos;							\
		pos = hlist_entry_safe(rcu_dereference_raw(hlist_next_rcu(\
			&(pos)->member)), typeof(*(pos)), member))
```

`hlist_for_each_entry_rcu`宏的三个入参含义如下：

`pos`：当前位置指针，指向包含`hlist_node`的结构体的指针，用于在遍历过程中指向当前链表元素，注意点：

-	必须是有效的指针变量
-	类型应该与链表元素的实际类型匹配
-	在宏展开后会被自动初始化和更新

`head`：链表头指针，类型为`struct hlist_head *`，作用是指向要遍历的哈希链表的头节点

`member`：节点成员名称，即结构体成员的名称（标识符），用来指定`hlist_node`在包含结构体中的字段名（链表节点在包含结构体中的字段名称）

列举几个例子：

`find_pid_ns`，用于遍历全局upid hashtable，第一个参数`pnr`表示 upid hashtable中每个hash链表桶元素（类型为`struct upid *`）；第二个参数`&pid_hash[pid_hashfn(nr, ns)]`，直接定位到upid hashtable的bucket 头指针，类型为`struct hlist_head *`；第三个参数为`pid_chain`，对应于`upid`结构体的node成员`pid_chain`，类型为`struct hlist_node`

```cpp
static struct hlist_head *pid_hash;	//全局hashtable

struct upid {
	/* Try to keep pid_chain in the same cacheline as nr for find_vpid */
	int nr;
	struct pid_namespace *ns;
	struct hlist_node pid_chain;
};

struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
{
	struct upid *pnr;
	//
	hlist_for_each_entry_rcu(pnr,
			&pid_hash[pid_hashfn(nr, ns)], pid_chain)
		if (pnr->nr == nr && pnr->ns == ns)
			return container_of(pnr, struct pid,
					numbers[ns->level]);
	
		return NULL;
}
```

再看一个宏定义宏的例子：

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pid.h#L178
#define do_each_pid_task(pid, type, task)				\
	do {								\
		if ((pid) != NULL)					\
			hlist_for_each_entry_rcu((task),		\
				&(pid)->tasks[type], pids[type].node) {
```

调用`do_each_pid_task`的代码：

```cpp
static bool has_stopped_jobs(struct pid *pgrp)
{
	struct task_struct *p;

	do_each_pid_task(pgrp, PIDTYPE_PGID, p) {
		if (p->signal->flags & SIGNAL_STOP_STOPPED)
			return true;
	} while_each_pid_task(pgrp, PIDTYPE_PGID, p);

	return false;
}

struct pid {
    // ...
    struct hlist_head tasks[PIDTYPE_MAX];  // 每个pid有自己的任务链表
    // ...
};

struct task_struct {
    // ...
    struct pid_link pids[PIDTYPE_MAX];     // 链接到对应pid的tasks链表
    // ...
};

struct pid_link {
    struct hlist_node node;   // ← 这就是member参数指向的节点
    struct pid *pid;
};
```

在`do_each_pid_task`调用`hlist_for_each_entry_rcu`宏，第一个参数为`task`（`p`），是`struct task_struct *`类型；第二个参数为`&(pid)->tasks[type]`，调用展开为`&(pgrp)->tasks[PIDTYPE_PGID]`,即`struct pid`中的hashtable头；第三个参数为`pids[type].node`，展开为`pids[PIDTYPE_PGID].node`，对应`task_struct`中的成员`pids[PIDTYPE_MAX].node`，其类型正为`struct hlist_node`，简化为如下：

```cpp
struct task_struct *task;                    // pos: 遍历时的当前任务
struct hlist_head *head = &pid->tasks[type]; // head: PID的任务链表头
                                            // member: pids[type].node

hlist_for_each_entry_rcu(task, head, pids[type].node) {
    // 处理每个task
	......
}
```

##	0x03	rbtree
[rbtree](https://elixir.bootlin.com/linux/v6.13.1/source/lib/rbtree.c)
经典数据结构，性能稳定，插入/删除/查找的时间复杂度接近`O(logN)`，而且中序遍历的结果是从小到大有序，内核中有如下应用：
-	linux高精度定时器的实现（`hrtimer`）：以rbtree的形式组织，树的最左边的节点就是最快到期的事件节点
-	Ext3文件系统，通过rbtree组织目录项
-	linux内存中内存的管理：分配和回收。用rbtree组织已经分配的内存块，当应用程序调用free释放内存的时候，可以根据内存地址在rbtree中快速找到目标内存块
-	linux内核进程调度算法通过rbtree组织管理，便于快速插入、删除、查找进程的`task_struct`

此外，这里再补充关于rbtree的几个细节：
1.	rbtree的键值可以重复么？
2.	rbtree必须有键值么？
3.	rbtree的精髓在于其动态的高度调整，即在增、删、改的过程中，为了避免rbtree退化成单链表，rbtree的特性会动态地调整树的高度，让树高不超过`2lg(n+1)`

![rb-tree](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/datastructure/rb_node_shili1.png)

参考上面这棵树的创建过程，对于键值相同的节点是有时间顺序的，插入晚的默认为大值，放在后面，也就是说rbtree自动实现了按时间轴存储键值的功能。即使到期时间相等（键值Key相等），也可以根据其插入红黑树的时间顺序来取出最小到期事件去执行

####	结构体定义
与其他的内核数据结构相同，为了通用性，`rb_node`只保留了红黑树节点自身的必须属性，即左孩子、右孩子、节点颜色

![rb-node](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/datastructure/rb_node.jpg)

```cpp
struct rb_node {
    unsigned long  __rb_parent_color;
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
    /* The alignment might seem pointless, but allegedly CRIS needs it */
```

TODO

####	核心代码解析

####	rbtree的应用

1、CFS 非实时任务调度

在Linux CFS调度算法（`2.6.24` 版本之后）中，所有非实时可运行进程都以虚拟运行时间（`vruntime`）为键值（注意，`vruntime`是红黑树的key非value）用一棵红黑树进行维护，以完成更公平高效地调度所有任务。CFS 算法弃用 `active/expired` 数组和动态计算优先级，不再跟踪任务的睡眠时间和区别是否交互任务，而是在调度中采用基于时间计算键值的红黑树来选取下一个任务，根据所有任务占用 CPU 时间的状态来确定调度任务优先级，优点如下：

1. 高效的查找、插入和删除操作：红黑树是一种自平衡二叉搜索树，能够在`O(logN)`时间内完成查找、插入和删除操作，适合调度器频繁调整任务的需求
2.	自平衡特性：红黑树通过颜色标记和旋转操作保持平衡，避免树结构退化为链表，确保操作效率稳定
3. 支持快速获取最小节点：CFS每次调度需要选择`vruntime`最小的任务，红黑树的最左节点即为最小节点，可以在`O(logN)`时间内快速定位
4. 动态任务管理：CFS需要频繁插入新任务和删除已完成任务，红黑树的高效动态操作能力满足了这一需求
5. 内存效率：红黑树每个节点只需额外存储一个颜色标记，内存开销较小，适合内核这种对内存敏感的环境

##	0x04		等待队列
**等待（waiter） - 唤醒（waker）**模型是 Linux 中的一种基础机制。当进程要获取某些资源（例如从网卡读取数据）的时候，但资源并没有准备好（例如网卡还没接收到数据），这时候内核必须切换到其他进程运行，直到资源准备好再唤醒进程。**waitqueue 机制就是内核用于管理等待资源的进程，当某个进程获取的资源没有准备好的时候，可以通过调用 `add_wait_queue()` 函数把进程添加到 waitqueue 中，然后切换到其他进程继续执行。当资源准备好，由资源提供方通过调用 `wake_up()` 函数来唤醒等待的进程**

####	等待队列结构
和linklist类似，waitqueue 本质上是一个链表，waitqueue包含了`wait_queue_head_t`（头）以及`__wait_queue`（节点）两个基础结构

```cpp
struct __wait_queue_head {
    spinlock_t lock;	//lock 字段用于保护等待队列在多核环境下数据被破坏
    struct list_head task_list;	//task_list 字段用于保存等待资源的进程列表
};

struct __wait_queue {
    unsigned int flags;	// 重要：可以设置为 WQ_FLAG_EXCLUSIVE，表示等待的进程应该独占资源（解决惊群现象）
    void *private;	//一般用于保存等待进程的进程描述符 task_struct，也可以设置为NULL（epoll下的acceptfd）
    wait_queue_func_t func;	//唤醒函数，一般设置为 default_wake_function() 函数，当然也可以设置为自定义的唤醒函数
    struct list_head task_list;	//用于链接其他等待资源的进程
};
```

等待同一事件的任务（由`private` 指向）通过双向链表串接（对应`__wait_queue` 节点），形成一个等待队列，在等待的事件发生后，waitqueue上的任务被唤醒，并执行`func` 回调函数

重点指出`flags`、`private`及`func`成员，在后面的文章中会有不同的用途

![waitqueue](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/datastructure/waitqueue-1.jpg)

####	等待队列操作

1、waitqueue 初始化：通过调用 `init_waitqueue_head()` 函数来初始化 `wait_queue_head_t` 结构，首先调用 `spin_lock_init()` 来初始化自旋锁 lock，然后调用 `INIT_LIST_HEAD()` 来初始化进程链表

```cpp
void init_waitqueue_head(wait_queue_head_t *q)
{
    spin_lock_init(&q->lock);
    INIT_LIST_HEAD(&q->task_list);
}
```

2、向waitqueue添加等待进程，要向 waitqueue 添加等待进程，首先要声明一个 `wait_queue_t` 结构的变量即等待队列成员，通过`init_waitqueue_entry`实现。初始化完 `wait_queue_t` 结构变量后，可以通过调用 `add_wait_queue()` 函数把等待进程添加到等待队列，其中`add_wait_queue()` 函数的实现逻辑为首先通过调用 `spin_lock_irqsave()` 上锁，然后调用 `list_add()` 函数把节点添加到等待队列链表即可

```cpp
// init_waitqueue_entry 初始化为系统默认的等待唤醒函数
static inline void init_waitqueue_entry(wait_queue_t *q, struct task_struct *p)
{
    q->flags = 0;
    q->private = p;
    q->func = default_wake_function;
}

//可以通过调用 init_waitqueue_func_entry() 函数来初始化为自定义的唤醒函数
static inline void init_waitqueue_func_entry(wait_queue_t *q, wait_queue_func_t func)
{
    q->flags = 0;
    q->private = NULL;
    q->func = func;
}

void add_wait_queue(wait_queue_head_t *q, wait_queue_t *wait)
{
    unsigned long flags;

    wait->flags &= ~WQ_FLAG_EXCLUSIVE;
    spin_lock_irqsave(&q->lock, flags);
    __add_wait_queue(q, wait);
    spin_unlock_irqrestore(&q->lock, flags);
}

static inline void __add_wait_queue(wait_queue_head_t *head, wait_queue_t *new)
{
    list_add(&new->task_list, &head->task_list);
}
```

3、休眠等待进程，当把进程添加到等待队列后，就可以休眠当前进程，让出CPU给其他进程运行。通常按照下面的代码可以完成**休眠当前进程->让出当前CPU的操作**：

```cpp
set_current_state(TASK_INTERRUPTIBLE);	//把当前进程运行状态设置为可中断休眠状态`TASK_INTERRUPTIBLE`
schedule();	//调用 schedule() 函数可以使当前进程让出CPU，切换到其他进程执行
```

关联的函数有如下：

```cpp
wait_event(wq_head, condition)
wait_event_timeout(wq_head, condition, timeout) 
wait_event_interruptible(wq_head, condition)
wait_event_interruptible_timeout(wq_head, condition, timeout)
```

4、唤醒等待队列，当资源准备好后，就可以唤醒等待队列中的进程，可以通过 `wake_up()->__wake_up_common()` 函数来唤醒等待队列中的进程。唤醒等待队列就是遍历等待队列的等待进程（即`list_for_each_entry_safe`方法），然后调用唤醒函数来唤醒它们

关联的函数有如下：

```cpp
wake_up(&wq_head)
wake_up_interruptible(&wq_head)
wake_up_nr(&wq_head, nr)
wake_up_interruptible_nr(&wq_head, nr)
wake_up_interruptible_all(&wq_head)
```

```cpp
static void __wake_up_common(wait_queue_head_t *q, 
    unsigned int mode, int nr_exclusive, int sync, void *key)
{
    wait_queue_t *curr, *next;

    list_for_each_entry_safe(curr, next, &q->task_list, task_list) {
        unsigned flags = curr->flags;

        if (curr->func(curr, mode, sync, key) &&
                (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
            break;
    }
}
```

####    一些细节
1、`wait_event`的实现细节

Linux 内核的 `___wait_event` 宏通过**循环检查条件（`condition`）+ 主动让出 CPU（`cmd`） + 唤醒后重新检查条件**的机制实现了等待队列的核心逻辑

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/wait.h#L266
#define ___wait_event(wq, condition, state, exclusive, ret, cmd)	\
({									\
	__label__ __out;						\
	wait_queue_t __wait;						\
	long __ret = ret;	/* explicit shadow */			\
									\
	init_wait_entry(&__wait, exclusive ? WQ_FLAG_EXCLUSIVE : 0);	\
	for (;;) {							\
		long __int = prepare_to_wait_event(&wq, &__wait, state);\
									\
        // 被唤醒后再次检查condition是否满足
		if (condition)						\
			break;						\
									\
		if (___wait_is_interruptible(state) && __int) {		\
			__ret = __int;					\
			goto __out;					\
		}							\
									\
        //内置schedule实现，让出CPU
		cmd;							\
	}								\
	finish_wait(&wq, &__wait);					\
__out:	__ret;								\
})

// finish_wait ...
void finish_wait(wait_queue_head_t *q, wait_queue_t *wait)
{
	unsigned long flags;

	__set_current_state(TASK_RUNNING);

	if (!list_empty_careful(&wait->task_list)) {
		spin_lock_irqsave(&q->lock, flags);
		list_del_init(&wait->task_list);
		spin_unlock_irqrestore(&q->lock, flags);
	}
}
```

1、在`for`主循环里，`condition` 是一个`true/false`的标志位，通常由 waker 设置（需要保证并发安全），当 waiter 检测到其值为 `true`时，表明事件发生，通过`break`跳出循环

2、跳出循环后，调用 `finish_wait()`将状态重置为运行态`TASK_RUNNING`，并从 waitqueue 移除，等待的过程彻底结束

3、如果没有收到信号，`condition` 条件也没满足，那么任务应该已经在 waitqueue 上面了，接下来调用`cmd`执行任务切换，`cmd`是由上一层的宏或者函数传入的参数，通常为`schedule`/`schedule_timeout`，底层的调用链为`__schedule() --> deactivate_task() --> dequeue_task()` ，最后`dequeue_task()` 将作为 waiter 的任务从所在CPU的 runqueue 移除，正式让出 CPU 的控制权，当前进程由运行态切换为睡眠态

4、当发生上下文切换时，内核保存了进程的寄存器状态、栈指针、程序计数器（PC）等，恢复时直接从 `schedule()` 的下一条指令开始，当进程再次被调度时，确实从 `schedule()`后的代码继续执行

5、当从 `schedule`/`schedule_timeout` 函数返回时，意味着事件到了或者信号来了或者超时了，即被唤醒了，继续回到`for`循环的开头，再次判断`condition`是否为`true`，这里引出一个问题，内核为何要执行先唤醒后重新检查条件这个策略呢？

TODO

2、关于`wait_event*`函数中`condition`的理解，`condition`是谁负责更新的？

3、内核的典型用法一：一个队列等待多个事件

如果一个 page 正在经历 I/O 换出，为了避免同时访问造成的数据不一致，需要上锁，之后试图使用这个 page 的任务将调用 `wait_on_page_locked()` 排队。当 I/O 操作完成，page 解锁，等待任务被唤醒

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pagemap.h#L497
static inline void wait_on_page_locked(struct page *page)
{
	if (PageLocked(page))
		wait_on_page_bit(compound_head(page), PG_locked);
}
```

直觉上应该每个 page 有一个 waitqueue，但这样开销太大了，内核的做法是将 page 按照一定的 hash 规则分组（一共 `256` 个分组），等待同一组内的 page 的任务都将被挂接在同一个 waitqueue 上。当分组内的某一个 page 解锁时，通过 hash 比对，查找 waitqueue 中等待该 page 的所有任务并唤醒

```cpp
#define PAGE_WAIT_TABLE_SIZE (1 << 8)
wait_queue_head_t page_wait_table[PAGE_WAIT_TABLE_SIZE];

wait_queue_head_t *page_waitqueue(struct page *page)
{
    return &page_wait_table[hash_ptr(page, PAGE_WAIT_TABLE_BITS)];
}

//https://elixir.bootlin.com/linux/v4.11.6/source/mm/filemap.c#L912
void add_page_wait_queue(struct page *page, wait_queue_t *waiter)
{
	wait_queue_head_t *q = page_waitqueue(page);
	unsigned long flags;

	spin_lock_irqsave(&q->lock, flags);
	__add_wait_queue(q, waiter);
	SetPageWaiters(page);
	spin_unlock_irqrestore(&q->lock, flags);
}
EXPORT_SYMBOL_GPL(add_page_wait_queue);
```

4、内核的典型用法二：一个任务等待多个队列

I/O 多路复用场景下，一个任务可以同时等待多个事件，这里的同时等待就利用了等待队列的机制来实现。以`epoll`机制为例，通过accept获取的fd关联的`eppoll_entry` 结构体就包含了等待队列的相关数据结构，如下

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L230
struct eppoll_entry {
	/* List header used to link this structure to the "struct epitem" */
	struct list_head llink;

	/* The "base" pointer is set to the container "struct epitem" */
	struct epitem *base;

	/*
	 * Wait queue item that will be linked to the target file wait
	 * queue head.
	 */
	wait_queue_t wait;

	/* The wait queue head that linked the "wait" wait queue item */
	wait_queue_head_t *whead;
};
```

而通过`epoll_ctl()` 增加监听fd及事件的过程，也包含了加入 waitqueue 的动作（注册socket就绪时的`wait_queue_t`结构，包含了就绪回调函数）等，主要内核函数调用链为`epoll_ctl(EPOLL_CTL_ADD) --> ep_insert() --> ep_ptable_queue_proc() --> add_wait_queue()`，调用它的任务会被挂在多个 waitqueue 上面，当其中一个 waitqueue 上的事件到来时（比如文件有可读的内容，或者腾出了可写的空闲区域），创建 waitqueue 节点时填入的回调函数 `ep_poll_callback()` 会将这个 entry 移入 ready list（或者 ovflist），等待任务唤醒后处理

##	0x05	小结

1、链表 VS hashtable，内核更偏爱后者

对比内核中二者实现，哈希链表的结构`struct hlist_head`和 `struct hlist_node`。它与普通的 `struct list_head`双链表有一个关键区别：

-	`struct list_head`有 `prev`和 `next`两个指针，构成一个完美的双向链表
-	`struct hlist_head`只有一个指向第一个节点的 `first`指针
-	`struct hlist_node`有 `next`和 `pprev`（指向指向上一个节点的指针的指针）

hashtable这样设计主要是为了节省内存，哈希表的表头是一个数组，每个桶都是一个链表头。如果使用普通的 `list_head`，每个链表头都会有两个指针（`prev`/`next`），但在表头数组中，`prev`指针对于桶头来说是浪费的，因为桶头之间并不需要相互链接。`hlist`的这种设计将表头的开销减少了一半，对于拥有大量桶的哈希表来说，节省的内存非常可观

内核中双链表更多用于需要严格顺序或集合管理的场景，例如调度器中的就绪队列、等待某个事件发生的进程队列或者
需要按顺序处理的软中断或工作队列等

##  0x06  参考
-   [Linux 内核中的数据结构](https://blog.csdn.net/Rong_Toa/article/details/115110811)
-   [内核基础设施——hlist_head/hlist_node 结构解析](https://linux.laoqinren.net/kernel/hlist/)
-   [《Linux 内核设计与实现》读书笔记（六）- 内核数据结构](https://www.cnblogs.com/wang_yb/archive/2013/04/16/3023892.html)
-   [linux kernel 中 HList 和 Hashtable](https://liujunming.top/2018/03/12/linux-kernel%E4%B8%ADHList%E5%92%8CHashtable/)
-   [Linux 内核中双向链表的经典实现](https://www.cnblogs.com/skywang12345/p/3562146.html)
-	[rbtree](https://oi-wiki.org/ds/rbtree/)
-	[linux源码解读（十五）：红黑树在内核的应用——CFS调度器](https://www.cnblogs.com/theseventhson/p/15799832.html)
-	[linux源码解读（十四）：红黑树在内核的应用——红黑树原理和api解析](https://www.cnblogs.com/theseventhson/p/15798449.html)
-	[数据结构可视化](http://www.u396.com/wp-content/collection/data-structure-visualizations/)
-	[Linux 中的等待队列机制](https://zhuanlan.zhihu.com/p/97107297)
-	[等待队列原理与实现](https://github.com/liexusong/linux-source-code-analyze/blob/master/waitqueue.md)
-	[如何阅读内核源码](https://cloud.tencent.com/developer/article/1963196)