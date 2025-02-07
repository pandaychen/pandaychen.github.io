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
-   队列
-   rbtree

本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

####    offset_of 机制

```c
// 说明：获得结构体 (TYPE) 的变量成员 (MEMBER) 在此结构体中的偏移量
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

![offset_of]()

`TYPE` 是结构体，代表整体，而 `MEMBER` 是结构体成员，`offsetof` 的本质是已知整体和该整体中某一个部分，计算该部分在整体中的偏移，详细步骤如下：

1. `((TYPE *)0 )`：将零转型为 `TYPE` 类型指针，即 `TYPE` 类型的指针的地址是 `0`
2. `((TYPE *)0)->MEMBER`：访问结构中的数据成员
3. `&(( (TYPE *)0 )->MEMBER )`：取出数据成员的地址，由于 `TYPE` 的地址是 `0`，这里获取到的地址就是相对 `MEMBER` 在 `TYPE` 中的偏移
4. `(size_t)(&(((TYPE*)0)->MEMBER))`：结果转换类型。对于 `32` 位系统而言，`size_t` 是 `unsigned int` 类型；对于 `64` 位系统而言，`size_t` 是 `unsigned long` 类型

####    container_of 机制

```CPP
// 说明：根据结构体 (type) 变量中的域成员变量 (member) 的指针 (ptr) 来获取指向整个结构体变量的指针
#define container_of(ptr, type, member) ({          \
    const typeof(((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

同样的，已知成员 `MEMBER` 及 `MEMBER` 的指针地址，推算出结构体的开始地址，即已知整体和该整体中某一个部分，要根据该部分的地址，计算出整体的地址

![container_of]()

1.  `typeof(( (type *)0)->member )`：取出 member 成员的变量类型
2.  `const typeof(((type *)0)->member ) *__mptr = (ptr)`：定义变量 `__mptr` 指针，并将 `ptr` 赋值给 `__mptr`。经过这一步，`__mptr` 为 `member` 数据类型的常量指针，其指向 `ptr` 所指向的地址
3.  `(char *)__mptr`：将 `__mptr` 转换为字节型指针
4.  `offsetof(type,member))`：就是获取 `member` 成员在结构体 `type` 中的位置偏移
5.  `(char *)__mptr - offsetof(type,member))`：就是用来获取结构体 `type` 的指针的起始地址（为 `char *` 型指针）
6.  `(type *)( (char *)__mptr - offsetof(type,member) )`：就是将 `char *` 类型的结构体 `type` 的指针转换为 `type *` 类型的结构体 `type` 的指针

##  0x01    双向链表
内核实现的 double-linklist 和普通的 linklist 有差异，普通 linklist 实现缺点就是通用性不够好，而内核的 linklist 巧妙解决了这个问题，不是将用户数据保存在 linklist 节点中，而是将 linklist 节点保存在用户数据中。linux 的 linklist 节点只有 `2` 个指针成员 (`prev` 和 `next`)，因此 linklist 的节点将独立于用户数据之外，便于实现 linklist 的共同操作

内核 linklist 中的最大问题是怎样通过链表的节点来取得用户数据？linux 的 linklist 节点 (node) 中没有包含用户的用户如 `data1`，`data2` 等，所以 Linux 中双向链表的使用思想也变成了将双向链表节点嵌套在其它的结构体（比如 linux 进程管理的 `struct task_struct`[结构](https://github.com/torvalds/linux/blob/master/include/linux/sched.h#L1055)）中；在遍历链表的时候，根据双链表节点的指针获取 ** 它所在结构体的指针 **（使用 `container_of` 宏），从而再获取数据或者该结构体的其他成员

```CPP
struct list_head {
	struct list_head *next, *prev;
};
```

![]()

####    使用
1、结构定义

普通 linklist：

```CPP
struct student{// 结构体 student 本身就是一个链表节点
	struct student *prev;
	struct student *next;
	char name[64];
	char id[64];
}
```

内核 linklist：

```CPP
struct student{
	struct list_head list_node;// 链表节点
	char name[64];
	char id[64];
};
```

2、遍历链表

参考 []()

##  0x02    hash 链表
`hlist_head` 和 `hlist_node` 用于内核 hash 表，分别表示列表头（数组中的一项）和列表头所在双向链表中的某项，结构定义及示意图如下:

```CPP
struct hlist_head {
	struct hlist_node *first;
};

struct hlist_node {
	struct hlist_node *next, **pprev;
};
```

![hlist]()

回想下，在 linux 进程管理 `struct pid/upid`[结构](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pid.h#L50) 中就使用了 hash linklist

```CPP
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
	struct upid numbers[1];
};
```

![hlist]()

如上图 hash 表（数组）的元素类型为 `struct hlist_head`，以 `hlist_head` 为链表头的链表为冲突链，节点类型为 `struct hlist_node`。hash_table 的列表头仅存放一个 `first` 指针，指向的是对应冲突链表的头结点，不设计 `tail` 指针是为了减少内存空间（当数组的 size 很大时）。

此外，注意到 `hlist_node.pprev` 是一个指向指针的指针（保存前一个节点的 `next` 成员地址），目的是在删除链表头结点的时候，`pprev` 这个设计无需判断删除的节点是否为头结点（如果是普通双向 linklist 的设计，那么删除头结点之后，`hlist_head` 中的 `first` 指针需要指向新的头结点），参考 `__hlist_del` 的实现

```CPP
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

```CPP
#define hlist_entry(ptr, type, member) container_of(ptr,type,member)
```

####    内核的应用
内核中用 `hlist` 来实现 hash table，如下：

```BASH
[root@VM-119-175-tencentos kernel]# dmesg | grep "hash table entries"
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

##  0x03  参考
-   [Linux 内核中的数据结构](https://blog.csdn.net/Rong_Toa/article/details/115110811)
-   [内核基础设施——hlist_head/hlist_node 结构解析](https://linux.laoqinren.net/kernel/hlist/)
-   [《Linux 内核设计与实现》读书笔记（六）- 内核数据结构](https://www.cnblogs.com/wang_yb/archive/2013/04/16/3023892.html)
-   [linux kernel 中 HList 和 Hashtable](https://liujunming.top/2018/03/12/linux-kernel%E4%B8%ADHList%E5%92%8CHashtable/)
-   [Linux 内核中双向链表的经典实现](https://www.cnblogs.com/skywang12345/p/3562146.html)