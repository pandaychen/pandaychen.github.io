---
layout:     post
title:  Linux 内核之旅（二十）：page cache
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

页高速缓存（page cache），它是一种对完整的数据页进行操作的磁盘高速缓存，即把磁盘的数据块缓存在页高速缓存中

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

##  0x0  参考
-   [Linux中的Page Cache [一]](https://zhuanlan.zhihu.com/p/68071761)
-   [Linux中的Page Cache [二]](https://zhuanlan.zhihu.com/p/71217136)
-   [Linux 内核源码分析-Page Cache 刷脏源码分析](https://www.leviathan.vip/2019/06/01/Linux%E5%86%85%E6%A0%B8%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90-Page-Cache%E5%8E%9F%E7%90%86%E5%88%86%E6%9E%90/)