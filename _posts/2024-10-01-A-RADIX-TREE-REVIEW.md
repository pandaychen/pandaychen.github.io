---
layout:     post
title:      数据结构与算法回顾（十）：radix tree
subtitle:	内核中的 IDR 与 XArray
date:       2024-10-01
author:     pandaychen
header-img:
catalog: true
tags:
    - 数据结构
---

##  0x00    前言
前文[httprouter分析](https://pandaychen.github.io/2021/10/01/GOLANG-HTTPROUTER-ANALYSIS/)，分析了基于字符串的radix tree的实现，本文分析下内核的radix tree（ID Radix Tree）

-   page cache管理：如4.11.6内核版本中的[`radix_tree_root`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/radix-tree.h#L112)、[`radix_tree_node`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/radix-tree.h#L93)，被用于实现内核`task_struct`维度的page cache管理，关联结构[`address_space`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/fs.h#L381)
-   进程pid分配管理：如内核6.18版本的[`idr`](https://elixir.bootlin.com/linux/v6.18/source/include/linux/idr.h#L20)结构、[`radix_tree_node`](https://elixir.bootlin.com/linux/v6.18/source/include/linux/radix-tree.h#L26)，不过这里的`radix_tree_root	`与`radix_tree_node	`结构已经被替换为`xarray`与`xa_node`实现

旧版本的radix tree实现，在4.20内核之后，被`xarray`结构所替换

##	0x01	Radix tree VS 内核IDR

####	区别1：key的类型
-	字符串Radix Tree，通常key为字符串，支持前缀查找、最长前缀匹配等
-	内核IDR，key为整数（如进程id、文件描述符等），主要用于ID分配和查找

####	区别2：设计目标

-	字符串Radix Tree：应用于高效的字符串查找、前缀匹配（最长前缀匹配等）
-	内核IDR：实现目的是支持高效的整数ID管理和指针查找，如进程管理（新版本的内核进程管理已经改为IDR）、文件描述符、设备号分配，主要操作如ID分配（寻找空闲ID）、ID查找（通过ID找指针）、ID释放（回收ID）等

####	区别3：结构

1、字符串Radix Tree结构如下，对于字符串Radix Tree，由于需要按照字符/字节key比较，逐字符匹配，对于共享前缀（前缀压缩），共享前缀存储在节点中，此外，key是变长的

```TEXT
prefix: 共享字符串前缀，变长
edges: 子节点边（有序数组），动态数组
leaf: 叶子数据，稀疏存储，适合长字符串

+-----------+-----------------+-----------+
| 前缀指针   | edge（边）数组  |  叶子数据  |
+-----------+-----------------+-----------+
```

2、Linux内核IDR结构，关键成员如下：

-   按位分块：整数ID被分成固定大小的位块
-   固定层级：通常3-4层（取决于整数key的位数）
-   bitmap优化：使用bitmap快速查找空闲槽位

```cpp
// 基于位（bit）的Radix Tree
#define IDR_BITS 6
#define IDR_SIZE (1 << IDR_BITS)  // 64

struct idr_layer {
    DECLARE_BITMAP(bitmap, IDR_SIZE);  // 位图标记已用槽位
    struct idr_layer __rcu *slots[IDR_SIZE];  // 子节点或数据指针
    int count;  // 已用槽位计数
};
```

举例来说，IDR的分层结构与内存布局如下：

```text
层级3（高6位） -> 层级2（中间6位） -> 层级1（低6位） -> 数据指针
      63-58          57-52          51-0

+-----------+-----------+-----------+
| 位图      |  指针数组  | 计数      |
+-----------+-----------+-----------+
```

##  0x01    radix tree（4.11.6）
先看下4.11.6版本的IDR实现：

![idr-simple-1]()

####	数据结构

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/radix-tree.h#L112
struct radix_tree_root {
	gfp_t			gfp_mask;
	struct radix_tree_node	__rcu *rnode;
};

//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/radix-tree.h#L93
struct radix_tree_node {
	unsigned char	shift;		/* Bits remaining in each slot */
	unsigned char	offset;		/* Slot offset in parent */
	unsigned char	count;		/* Total entry count */
	unsigned char	exceptional;	/* Exceptional entry count */
	struct radix_tree_node *parent;		/* Used when ascending tree */
	struct radix_tree_root *root;		/* The tree we belong to */
	union {
		struct list_head private_list;	/* For tree user */
		struct rcu_head	rcu_head;	/* Used when freeing node */
	};
	void __rcu	*slots[RADIX_TREE_MAP_SIZE];
	unsigned long	tags[RADIX_TREE_MAX_TAGS][RADIX_TREE_TAG_LONGS];
};
```

典型的用法是在`struct inode`用来管理文件的page cache

```cpp
struct address_space {
	struct inode		*host;		/* owner: inode, block_device */
	struct radix_tree_root	page_tree;	/* radix tree of all pages */
	spinlock_t		tree_lock;	/* and lock protecting it */
	......
}
```

##  0x02    radix tree（6.18）
从 Linux 4.15 开始，内核将 bitmap 换为了radix tree来实现进程id分配（用于管理 `32` bit 位的 整数 ID），结构如下：

```cpp
struct idr {
	struct radix_tree_root	idr_rt;
	unsigned int		idr_base;
	unsigned int		idr_next;
};

#define radix_tree_root		xarray
#define radix_tree_node		xa_node

//https://elixir.bootlin.com/linux/v6.18/source/include/linux/xarray.h#L300
struct xarray {
	spinlock_t	xa_lock;
/* private: The rest of the data structure is not to be used directly. */
	gfp_t		xa_flags;
	void __rcu *	xa_head;
};

//https://elixir.bootlin.com/linux/v6.18/source/include/linux/xarray.h#L1168
struct xa_node {
	unsigned char	shift;		/* Bits remaining in each slot */
	unsigned char	offset;		/* Slot offset in parent */
	unsigned char	count;		/* Total entry count */
	unsigned char	nr_values;	/* Value entry count */
	struct xa_node __rcu *parent;	/* NULL at top of tree */
	struct xarray	*array;		/* The array we belong to */
	union {
		struct list_head private_list;	/* For tree user */
		struct rcu_head	rcu_head;	/* Used when freeing node */
	};
	void __rcu	*slots[XA_CHUNK_SIZE];  //核心，指针字段，有点像字符串radix的edges
    //默认XA_CHUNK_SIZE为64
	union {
        //#define XA_MAX_MARKS		3
        //#define XA_MARK_LONGS		BITS_TO_LONGS(XA_CHUNK_SIZE)
		unsigned long	tags[XA_MAX_MARKS][XA_MARK_LONGS];
		unsigned long	marks[XA_MAX_MARKS][XA_MARK_LONGS];
	};
};
```

内核实现的IDR，特点是通常它的每一层只管理一个 `6` bits 的分段（关联变量为`XA_CHUNK_SHIFT`），即其分叉数基本是固定的 `64`（`2^6=64`）（根节点除外），层数也是固定的。在基数树节点`xa_node`结构中，有几个重要成员：

-    `shift`：表示了自己在数字中表示第几段数字（内核默认的基数大小为 `6`），这种情况下最低一层的内部节点，`shift` 为 `0`，倒数第二层 `shift` 为 `6`，再上一层节点的 `shift` 为 `12`，以此类推，`shift` 从低往高， 逐层递增 `6`
-   `slots`：是一个指针数组，存储的是其指向的子节点的指针。内核默认`XA_CHUNK_SIZE` 是 `64`，也就是是一个 `*slots[64]`。每个元素都指向下一级的树节点，没有下一级子节点的话指针指向 `NULL`
-   `tags`：用来记录 `slots` 数组中每一个下标的存储状态。可以用来表示每一个 slot 是否已经分配出去的状态。它是一个 `long` 类型的数组，一个 `long` 类型的变量是 `8` 个字节，正好有 `64` 个 `bit` 位

`shift`的作用是在计算时，计算当前要查询的整数右移的bit数，比如某层的`shift`为`12`，那么就右移 `12` 位取得其结果（作为在该层`slots`指针数组中的下标）

####    IDR的简单构造
用 `16` bit 的整数（`0~65535`）来构造idr，已分配ID：100、1000、10000、50000、60000：

```text
#从尾部开始，按照 6 bit 为一组来表示
#每一个 16 bit 位的数字可以表示为一个三段数字
#下面整数：第一段分别是二进制 0000、0000、0010、1100 和 1110，转化成 10 进制后是 0、0、2、12 和 14
100: 0000,000001,100100
1000: 0000,001111,101000
10000: 0010,011100,010000
50000: 1100,001101,010000
60000: 1110,101001,100000
```

在基数树中，根节点用来存储的每个整数的第一段。如果其中某一个数字已占用，那就把 slot 对应的下标的指针指向其子节点。否则为空。在本例中根节点的 `shift` 为 `12`，那就右移 `12` 位取得其结果

再下面一层节点的 `slots` 下标是每个值中间 `6` 个 bit 位的值，其 `shift` 为 `6`，最底层树的节点的 `slots` 是每个值最后 `6` 个 bit 的值，其 `shift` 为 `0`。再将上述每一个整数按照 `6` bit 为分段表示，即每个整数，在每层里面`slots`数组对应的位置。例如 `100`按每 `6` bit 一段拆分表示后为  `0,1,36`，其第一段是 `0` ，那就在基数树的根节点的 `slots` 的 `0` 号下标存储其子节点指针；在其第二层节点的 `slots` 的 `1` 号下标存储其子节点指针；在第三层节点的 `slots` 的 `36` 号下标存储最终的值 `100`

```TEXT
100: 0,1,36
1000: 0,15,40
10000: 2,28,16
50000: 12,13,16
60000: 14,41,32
```

![idr-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/datastructure/radix-tree/idr-1.png)

对建立好的基数树进行查找（判断一个整数是否存在），或者是从基数树中分配一个新的整数，只需要分别对这三层的基数树节点进行遍历，分别查看每一层中的 `tag` 状态位，看 `slots` 对应的下标是否已经占用即可（不需要bitmap结构那样从头开始遍历）。对于内核而言， 处理32位整数，需要`6`层节点来存储（对于小型内存系统，可以使用`4`层基数树，内存更省，但是查找性能会差一些）

####    性能
对比bitmap的性能比较：[参考](https://lwn.net/Articles/735675/)

##  0x03    IDR的主要函数
管理32位整数，IDR把每一个ID分级数据进行管理，每一级维护着ID的`6`位数据，这样就可以把IDR分为`6`级进行管理（`6*6=36`，维护的数据大于`32`位）：

```text
31 30 | 29 28 27 26 25 24 | 23 22 21 20 19 18 | 17 16 15  14 13 12 | 11 10 9 8 7 6 | 5  4 3 2 1 0
```

假设对`0B 10 111111 001100 111110 011000 100001`，寻址如下：

```text
1. 第一层寻址：ary2=ary1[0b10]，得到下一级地址ary2
2. 第二层ary3 = ary2[0b111111]
3. 第三层ary4 = ary3[0b001100]
4. 第四层ary5 = ary4[0b111110]
5. 第五层ary6 = ary5[0b011000]
6. page节点地址（IDR指针） = ary6[0b100001]
```

##  0x04	应用场景

####    进程ID分配
6.18版本通过调用 `idr_alloc` [函数](https://elixir.bootlin.com/linux/v6.18/source/lib/idr.c#L79)来申请一个未使用过的PID

```cpp
//https://elixir.bootlin.com/linux/v6.18/source/kernel/pid.c#L163
struct pid *alloc_pid(struct pid_namespace *ns, pid_t *set_tid,size_t set_tid_size)
{
    ......

    // 进程可能归属多个命名空间，在每一个命令空间中都需要分配进程号
    // 实际调用 idr_alloc 来申请整数类型的进程号 
    tmp = ns;
    pid->level = ns->level;
    for (i = ns->level; i >= 0; i--) {
        ......
        nr = idr_alloc(&tmp->idr, NULL, tid,
                tid + 1, GFP_ATOMIC);
        ......
        pid->numbers[i].nr = nr;
        pid->numbers[i].ns = tmp;
        tmp = tmp->parent;
    }
    ......
}

int idr_alloc(struct idr *idr, void *ptr, int start, int end, gfp_t gfp)
{
	u32 id = start;
	int ret;

	if (WARN_ON_ONCE(start < 0))
		return -EINVAL;

	ret = idr_alloc_u32(idr, ptr, &id, end > 0 ? end - 1 : INT_MAX, gfp);
	if (ret)
		return ret;

	return id;
}

int idr_alloc_u32(struct idr *idr, void *ptr, u32 *nextid,
			unsigned long max, gfp_t gfp)
{
	struct radix_tree_iter iter;
	void __rcu **slot;
	unsigned int base = idr->idr_base;
	unsigned int id = *nextid;

    ......

	id = (id < base) ? 0 : id - base;
	radix_tree_iter_init(&iter, id);

    //核心操作：遍历radix tree的若干节点，并根据每个节点的 tag、slot 等字段找出还未被占用的整数 ID
	slot = idr_get_free(&idr->idr_rt, &iter, gfp, max - base);
	if (IS_ERR(slot))
		return PTR_ERR(slot);

	*nextid = iter.index + base;
	/* there is a memory barrier inside radix_tree_iter_replace() */
	radix_tree_iter_replace(&idr->idr_rt, &iter, slot, ptr);
	radix_tree_iter_tag_clear(&idr->idr_rt, &iter, IDR_FREE);

	return 0;
}

void __rcu **idr_get_free(struct radix_tree_root *root, ...)
{
    ......
    shift = radix_tree_load_root(root, &child, &maxindex);
    while (shift) {
        shift -= RADIX_TREE_MAP_SHIFT; //RADIX_TREE_MAP_SHIFT为6
        ......

        // 遍历 tag 状态 bitmap，寻找下一个可用的下标
        offset = radix_tree_find_next_bit(node, IDR_FREE,
            offset + 1);
        start = next_index(start, node, offset);
    }
    ......
}
```

##  0x06    参考
-	[Linux Radix Tree 详解](https://mp.weixin.qq.com/s/1cxEi7Q5BoQK5J2s-wVJdg)
-   [为什么新版内核将进程pid管理从bitmap替换成了radix-tree？](https://cloud.tencent.com/developer/article/2321478)
-	[Linux Kernel：内核数据结构之基数树（Radix Tree）](https://zhuanlan.zhihu.com/p/673217948)