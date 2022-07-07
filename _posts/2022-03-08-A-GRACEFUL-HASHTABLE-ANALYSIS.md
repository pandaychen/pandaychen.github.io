---
layout: post
title: 数据结构与算法回顾（二）：一种固定 Size 的高性能 hashtable 实现
subtitle: 一种高效的 hash 存储结构分析
date: 2022-03-08
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 数据结构
  - hashmap
---

## 0x00 前言

本文介绍一种高性能的 hashtable，使用场景是结合最小堆统计单位时间窗口的 key 值计数并排序，该 hashtable 的特点是：
1.  hashtable 的整体大小是固定的，不扩容（避免问题），存储地址连续的（需要提前预估好存储上限）
2.  无指针，以 index 作为链接下一个节点
3.  存储分为 bucket 区和冲突链，冲突链上以 hop（非链表，还是以位置代替指针指向） 的形式进行查找及回收
4.  避免频繁内存申请和释放导致的内存碎片

现网中的使用场景，针对一段时间内，海量的访问 IP（请求），如何高效的查找到其中的 TOP-N？此问题的一种高效解法就是 hashtable 加最大堆排序。

## 0x01 Hashtable 简介
hashtable 的结构如下图所示：
![bmp-hashtable](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2021/hashtable/bmp-hashtable1.png)

##  0x01  结构定义

####  hash 节点

```cpp
enum HASH_TABLE_ENUM_CONSTANT{
	//IP 哈希桶数
	SERVERIP_HASH_BUCKET = 65536,//2^16
	SERVERIP_HASH_BUCKET_SHIFT = 16,
};
```

####  冲突链
```cpp
enum HASH_TABLE_TYPE{
	HASH_CONFICT_NODE_MUTI=4,
	HASH_CONFICT_NODE_MUTI_SHIFT=2,
};
```

####  管理节点
```cpp
// 通用的 hashtable 头结点定义
struct _comm_hash_table{
  void *pMemStart;    // 内存首地址
	void *pHashArray;
	void *pConflictArray;
	int32_t iCurBucketIndex;
	int32_t iNodeCount;
	int32_t iConflictCount;
	void *pCurNode;
	int32_t iCurIndexType;
	int32_t iType;
	uint32_t uUsed;
	uint32_t uReserved;
}__rte_cache_aligned;
```

####  hash-item 节点
`_server_ip` 是每个 hashtable 的 item 节点，
```cpp
//hashtable 的实例化节点 1
struct _server_ip{
	uint32_t uServerip;       //key：ip 地址
	uint32_t uServeripCount;  //value：统计数量
	uint32_t uNextIndex;   // 指向下一个节点
	uint32_t uReserved;   // 对齐用
};
```

##  0x02  操作

####  创建
hashtable 创建方法如下：
```cpp
int32_t CreateCommHashTable(CommHashTable **t_pCommHashTable, uint32_t t_uTotalTableSize, uint32_t t_uTableSize,uint32_t t_uConflictTableSize)
{
  //....
	//total buffersize
	uint32_t uTotalMemSize = sizeof(CommHashTable)+t_uTotalTableSize;
	int8_t *pMem = (uint8_t *)calloc(uTotalMemSize, 1);
  (*t_pCommHashTable)=(CommHashTable *)pMem;
	memset(pAllocBuffer, 0, uTotalMemSize);

	(*t_pCommHashTable)->pHashArray = (int8_t *)pMem + sizeof(CommHashTable);
	(*t_pCommHashTable)->pConflictArray = (int8_t *)pMem + sizeof(CommHashTable)+t_uTableSize;
	return RET_RIGHT;
}
```

####  插入


####  查找


####  删除


## 0x03 参考
