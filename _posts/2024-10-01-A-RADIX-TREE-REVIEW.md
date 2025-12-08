---
layout:     post
title:      数据结构与算法回顾（十）：radix tree
subtitle:	
date:       2024-10-01
author:     pandaychen
header-img:
catalog: true
tags:
    - 数据结构
---

##  0x00    前言
前文[httprouter分析](https://pandaychen.github.io/2021/10/01/GOLANG-HTTPROUTER-ANALYSIS/)，分析了基于字符串的radix tree的实现，本文分析下内核的radix tree（ID Radix Tree）

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

##  0x06    参考
-	[Linux Radix Tree 详解](https://mp.weixin.qq.com/s/1cxEi7Q5BoQK5J2s-wVJdg)