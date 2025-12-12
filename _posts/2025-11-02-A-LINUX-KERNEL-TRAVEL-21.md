---
layout:     post
title:  Linux 内核之旅（二十一）：page cache
subtitle:   内核中的page cache管理
date:       2025-09-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
本文主要梳理下page cache与管理的若干知识，本文基于[v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include)的源码

页高速缓存（page cache），它是一种对完整的数据页进行操作的磁盘高速缓存，即把磁盘的数据块缓存在页高速缓存中。page cache是内核为文件创建的内存缓存，用以加速相关的文件操作。当应用程序需要读取文件中的数据时，操作系统先分配一些内存，将数据从存储设备读入到这些内存中，然后再将数据分发给应用程序；当需要往文件中写数据时，操作系统先分配内存接收用户数据，然后再将数据从内存写到磁盘上

![linux_page_cache]()

本文涉及到read/write讨论，不考虑`O_DIRECT`的情况（如MySQL）

##  0x01    基础数据结构
前面在介绍VFS的时候提到了`struct inode`中的一个重要成员：`address_space`，`address_space`对象是文件系统中管理内存页page cache的核心数据结构

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/fs.h#L554
struct inode {
	umode_t			i_mode;
	unsigned short		i_opflags;
	kuid_t			i_uid;
	kgid_t			i_gid;
	unsigned int		i_flags;
    ......
	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;
    ......
}

//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/fs.h#L379
struct address_space {
	struct inode		*host;		/* owner: inode, block_device */
	struct radix_tree_root	page_tree;	/* radix tree of all pages */
	spinlock_t		tree_lock;	/* and lock protecting it */
	atomic_t		i_mmap_writable;/* count VM_SHARED mappings */
	struct rb_root		i_mmap;		/* tree of private and shared mappings */
	struct rw_semaphore	i_mmap_rwsem;	/* protect tree, count, list */
	/* Protected by tree_lock together with the radix tree */
	unsigned long		nrpages;	/* number of total pages */
	/* number of shadow or DAX exceptional entries */
	unsigned long		nrexceptional;
	pgoff_t			writeback_index;/* writeback starts here */
	const struct address_space_operations *a_ops;	/* methods */
	unsigned long		flags;		/* error bits */
	spinlock_t		private_lock;	/* for use by the address_space */
	gfp_t			gfp_mask;	/* implicit gfp mask for allocations */
	struct list_head	private_list;	/* ditto */
	void			*private_data;	/* ditto */
} __attribute__((aligned(sizeof(long))));

//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/mm_types.h#L40
struct page {
	/* First double word block */
	unsigned long flags;		/* Atomic flags, some possibly
					 * updated asynchronously */
	union {
		struct address_space *mapping;	/* If low bit clear, points to
						 * inode address_space, or NULL.
						 * If page mapped as anonymous
						 * memory, low bit is set, and
						 * it points to anon_vma object:
						 * see PAGE_MAPPING_ANON below.
						 */
		void *s_mem;			/* slab first object */
		atomic_t compound_mapcount;	/* first tail page */
		/* page_deferred_list().next	 -- second tail page */
	};

	/* Second double word */
	union {
		pgoff_t index;		/* Our offset within mapping. */
		void *freelist;		/* sl[aou]b first free object */
	};
    ......
}
```

每一个文件（一个inode指向的文件）都对应着一个`address_space`对象，`inode`中有一个`i_mmaping`成员，该成员即指向该文件对应的`address_space`对象，而`address_space`中的成员`page_tree`，这个指针指向的就是文件对应的基数树的根，而这棵radix树的叶子节点就是page cache

注意到每个`struct page` 描述符都包括把page链接到page cache的两个重要字段`mapping`和`index`。其中`mapping`成员指向拥有该页的索引节点的`address_space`对象，`index`成员表示在所有者的地址空间中以页大小为单位的偏移量，也就是在所有者的磁盘映射中页中数据的位置

`address_space`中的`host`成员指向其所属的`struct inode`对象，也就是`address_space`中的`host`字段与对应inode中的 `i_data`成员互相指向。对于普通文件而言，inode和address_space的相应指针的指向关系如下：

![inode-to-address_space](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/21/inode2addressspace2page.png)

####    内核中的radix tree
因为文件位于慢速的块设备上，如果没有缓存，每一次对文件的读写都要走到块设备，这样的访问速度是无法容忍的。在Linux上操作，如果一次读某个文件慢的话，紧接着读这个文件第二次，速度会有明显的提升。原因是Linux已经将文件的部分内容或全部内容缓存到了page cache，先列举几个问题：

1.  page cache在内核是通过radix树管理的，为何要使用该数据结构？
2.  从应用层的文件描述符fd，到`struct file`，从`struct file` 到 `struct dentry`，再从`struct dentry` 到`struct inode`，再`struct inode` 到 `struct address_space`, 只要知道文件的偏移量offset（如系统调用`read`参数），就能从radix_tree中查找对应的页面是否在页高速缓存，这里的整体过程是如何的？
3.  若A用户的a进程操作文件，将文件带入缓存，那么稍后B用户的b进程操作通一个文件时，同样可以享受文件内容在页高速缓存带来的福利
4.  page cache预读的原理是什么？
5.  要读某文件的第`N`个页面，内核是如何判断该页面是否在页高速缓存？如果在，如何找到该页的内容？如果不在，内核是如何处理的？


`ext4`支持到PB级文件存储，如此页的量级是巨大的。访问大文件时，page cache中存在着有关该文件巨大数量的页，所以内核提供了radix树来加快查找（一个address_space对象对应一个radix树）。`struct address_space`中成员`page_tree`指向是基树的根`radix_tree_root`，基树根中的`struct radix_tree_node *rnode`指向基树的最高层节点`radix_tree_node`，radix树节点都是`radix_tree_node`结构，节点中存放的都是指针，叶子节点的指针指向page描述符，radix树中上层节点指向存放其他节点的指针。一般一个`radix_tree_node`最多可以有`64`个指针，字段`count`表示该`radix_tree_node`已用节点数

```cpp
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

![radix-tree](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/21/radix-tree-with-page-cache-1.jpg)

如图，radix树的叶子节点对应的就是`struct page`

再回顾下radix树的查询过程，[]()

##  0x0    struct page的本质
前文已经描述了内核中虚拟内存主要分为两种类型的页，即匿名页与文件页，此外还介绍了虚拟内存地址到物理内存地址的翻译过程、页表体系等

-   [Linux 内核之旅（三）：虚拟内存管理（上）](https://pandaychen.github.io/2024/11/05/A-LINUX-KERNEL-TRAVEL-2/)
-   [Linux 内核之旅（十四）：Linux内核的页表体系](https://pandaychen.github.io/2025/04/25/A-LINUX-KERNEL-TRAVEL-14/)


####    基础结构
```cpp

```

需要注意的一点是：**`struct page`中是不包含存储数据的，其成员仅包含元数据，每个page对应的实际的页面内容放在物理内存中，需要通过虚拟地址访问**


####    文件页与匿名页

| 类别 | 内存来源 | 常用场景 |
| :-----:| :----: | :----: |
| 匿名页 | 从伙伴系统分配的新页面 | 进程堆、栈、通过`brk/sbrk`分配的内存，mmap匿名映射如`mmap(MAP_ANONYMOUS)`等 |
| 文件页 | Page Cache中的页面 | mmap文件映射，文件读写缓存等 |

如何区分`struct page`是哪种类型？

```cpp
struct page {
    union {
        struct address_space *mapping;  // 文件页：指向address_space
        //void *s_mem;
        void *anon_vma;                  // 匿名页：反向映射
    };
    pgoff_t index;  // 偏移量
    // ...
};
```

####    文件页（File-backed Pages）
TODO

####    匿名页

####    struct page 存储的位置
`struct page`结构体本身也需要内存存储，这些内存位于内核的虚拟地址空间，即直接映射区域（线性映射）。`mem_map`是全局数组，该存储在内核的虚拟地址空间中，位于直接映射区域（Direct Map）

```cpp
struct page *mem_map;  // 全局数组，指向所有struct page

// 系统启动时，内核计算需要多少内存来存储struct page
unsigned long nr_pages = total_physical_pages;
size_t page_struct_size = sizeof(struct page) * nr_pages;

// 为struct page数组分配内存
// 注意：这个内存本身也是物理内存，也需要用struct page描述！
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/page_alloc.c#L6094
TODO
```

以x86_64为例，虚拟内存地址布局如下，这里**直接映射区域是物理地址 + 固定偏移 = 虚拟地址**，偏移量是`PAGE_OFFSET`（x86_64通常是`0xffff880000000000`），这种`1:1`映射使得物理地址和虚拟地址可以快速转换

```text
0xffff 8000 0000 0000 ┬───────────────────┐
                      │  vmalloc区域      │
0xffff 8800 0000 0000 ┼───────────────────┤ <- 这里是struct page数组所在
                      │  直接映射区域      │   (PAGE_OFFSET = 0xffff880000000000)
                      │  1:1映射物理内存   │
0xffff 8800 0000 0000 ┼───────────────────┤
                      │  struct page数组  │
                      │  物理页描述符      │
0xffff 8800 0010 0000 ┼───────────────────┤
                      │  其他内核数据      │
0xffff c900 0000 0000 ┴───────────────────┘
```

####    page cache相关的操作函数


##  0x0 内核的预读机制

##  0x0 文件读与page cache


##  0x0 文件写与page cache

##  0x0 总结

####    page cache

-   page cache的内存在内核中是匿名的物理页（不与用户进程的逻辑地址进行映射），由`struct page`表示，在内核中page cache使用 LRU 管理，当用户进行mmap映射文件时，内核创建对应的vma，在访问到mmap的内存区域时，触发page fault，在page fault回调中page cache内存所属的物理页与用户进程的虚拟地址vma进行映射
-   每个文件的page cache元数据存储于对应的`struct inode->address_space`中，因此进程之间可以共享同一个文件的page cache，同一个文件多次open不会影响其page cache
-   文件的page cache是延时分配的，当有读写命令时，才会按需创建缓存页
-   page cache的脏页是单线程回写的，因此当一个文件大量写入时，写入的性能与单 CPU 的性能有相当的关系
-   对于`struct page`结构体，其存储在直接映射区域，虚拟地址=物理地址+固定偏移，而`struct page`数组占用的（物理）物理页，也用`struct page`描述。通过简单的加减运算就能在`struct page`和物理地址间转换

##  0x0  参考
-   [Linux中的Page Cache [一]](https://zhuanlan.zhihu.com/p/68071761)
-   [Linux中的Page Cache [二]](https://zhuanlan.zhihu.com/p/71217136)
-   [Linux 内核源码分析-Page Cache 刷脏源码分析](https://www.leviathan.vip/2019/06/01/Linux%E5%86%85%E6%A0%B8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-Page-Cache%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)
-   [文件IO系统调用内幕](https://lrita.github.io/2019/03/13/the-internal-of-file-syscall/)
-   [Linux Kernel：物理内存模型](https://zhuanlan.zhihu.com/p/704170214)
-   [图解匿名反向映射](https://richardweiyang-2.gitbook.io/kernel-exploring/nei-cun-guan-li/00-index/01-anon_rmap_history/06-anon_rmap_usage)