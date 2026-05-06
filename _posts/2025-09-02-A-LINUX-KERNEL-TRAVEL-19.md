---
layout:     post
title:  Linux 内核之旅（十九）：
subtitle:   
date:       2025-09-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
本文主要结合linux dentry（dentry cache）的实现及机制，讨论下相关的坑，基于内核版本5.4.241

1.  dentry cache过高导致的卡顿及OOM问题
2.  dentry cache与fsnotify文件监控机制的问题

```c
struct dentry {
	/* RCU 查找涉及的字段 */
	unsigned int d_flags;		/* 由 d_lock 保护 */
	seqcount_t d_seq;		/* 每个目录项的序列锁 */
	struct hlist_bl_node d_hash;	/* 查找哈希列表 */
	struct dentry *d_parent;	/* 父目录 */
	struct qstr d_name;        /* 目录项名称 */
	struct inode *d_inode;	    /* 名称所属的位置 - 为 NULL 表示负 */
	unsigned char d_iname[DNAME_INLINE_LEN];	/* 短名称 */
//..

	union { 
		struct list_head d_lru;		/* LRU 列表 */
		wait_queue_head_t *d_wait;	/* 仅用于查找中的等待队列 */
	};
	struct list_head d_child;	/* 父列表中的子项 */
	struct list_head d_subdirs;	/* 我们的子目录 */
	/*
	 * d_alias 和 d_rcu 可以共享内存
	 */
	union {
		struct hlist_node d_alias;	/* inode 别名列表 */
		struct hlist_bl_node d_in_lookup_hash;	/* 仅用于查找中的列表 */
		struct rcu_head d_rcu;
	} d_u;
} __randomize_layout; // 目录项结构
```

##  0x01    dcache 介绍


##  0x0 dentry cache 过高的典型场景分析

####    相关内核代码

TODO

##  0x0 fsnotify与dentry cache的坑

####    相关内核代码

TODO

##  0x0  参考
-   [Linux的VFS实现 - 番外[一] - dcache](https://zhuanlan.zhihu.com/p/261669249)