---
layout: post
title: 数据结构与算法回顾（四）：环形内存缓冲区 ringbuffer
subtitle: 一种高效进程间通信的机制：环形内存缓冲区
date: 2022-04-05
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 环形队列
  - Ring Buffer
  - Circular Buffer
---

## 0x00 前言
环形队列是一种 FIFO 数据结构，适合内存 / 共享内存的存储场景，是项目中解耦模块间（进程间）通信的可用手段之一。通常也称为 Ring Buffer、Circular Buffer。下图描绘了一个 A 24-byte keyboard circular buffer：

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/queue/Circular_Buffer_Animation.gif)

## 0x01 实现
ringbuffer 的实现主要依赖于读写指针的移动（head-ReadIndex/tail-WriteIndex）：
- 初始化一块定长的内存（或共享内存）作为存储空间，长度即为 `m_size`
- 通过 `mod m_size` 得出 ReadIndex/WriteIndex 的相对位置，进而实现 "环形" 的机制
- 写入 / 读取操作时需要考虑边界情况，写入需要移动 `tail` 指针，读取需要移动 `head` 指针

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/queue/ringbuffer-2.png)

##  0x02  现网应用
基于 ringbuffer 可以实现无锁的通信，在现网项目中，会遇到进程间通信的场景，及两个进程（进程 `A` 和进程 `B`）需要进行双向数据通信，如何无锁化实现呢？这就可以借用 ringbuffer，实现思路如下：
1.  创建两块固定大小的共享内存（共享内存 `X` 和共享内存 `Y`），每个共享存储单元的数据头信息结构为 `DataHead`，数据体信息为 `DataUnit`
2.  当进程 `A` 想传送数据体信息给进程 `B` 时，进程 `A` 向共享内存 `X` 写入数据体信息，变更共享内存 A 中的 `DataHead` 中的 `tail` 信息，进程 `B` 从共享内存 `X` 读出数据体信息，变更共享内存 `X` 中的 `DataHead` 中的 `head` 信息
3.  同样的方式，当进程 `B` 想传送数据体信息给进程 `A` 时，进程 `B` 向共享内存 `Y` 写入数据体信息，变更共享内存 `Y` 中的 `DataHead` 中的 `tail` 信息，进程 `A` 从共享内存 `Y` 读出数据体信息，变更共享内存 `Y` 中的 `DataHead` 中的 `head` 信息
4.  整个交互过程中，进程 `A` 与进程 `B` 无缝的通过 ringbuffer 进行通信，实现了进程间通信效率的最大化


流程图如下：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/queue/ring-buffer-shm-2-process.png)

##  0x03  代码实现
整个存储结构如下：
![ring-buffer-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/queue/ringbuffer-3.png)

####  存储数据节点

每个存储在内存中的数据都包括如下两个结构 `DataHead` 和 `DataUnit`：
- `DataHead`：内存数据头结构
- `DataUnit`：内存数据体结构（变长）

```cpp
struct NodeDataHead
{
	int iSize;
	int iTail;    // 写入数据更新位置
	int iHead;  // 读取数据更新位置
	int iOffset;
};

struct NodeDataUnit
{
	int iLen;   // 可变长
  	char *pData;
};
```

####	管理节点
`ShmRingQueue`是管理结构：
-	`NodeDataHead`：指向管理头节点
-	`m_pBuff`：指向数据区首地址

```cpp
class ShmRingQueue
{
	//...
private:
	NodeDataHead *m_pDataHead;
	char *m_pBuff; //指向首地址
};

// 初始化ringqueue的成员
ShmRingQueue::ShmRingQueue(char *pShmBuff)
{
	m_pDataHead = (NodeDataHead *)pShmBuff;
	m_pBuff = pShmBuff + sizeof(NodeDataHead);
}
```

####	功能方法
主要是利用`iWrite`、`iRead`计算出当前的一些指标（基于环形结构）：
1.	`GetLeftSize`：获取当前ringbuffer中还有多少可写的size
2.	`IsFull`：判断ringbuffer是否容量已满（包含了减掉`BUFFER_RESERVE_LENGTH`的部分）
3.	`GetUsedSize`：调用`GetLeftSize`，计算已占用的size

```cpp
// 获取当前ringqueue的剩余可用大小
int ShmRingQueue::GetLeftSize()
{
	int iRetSize = 0;
	int iWritePos = -1;
	int iReadPos = -1;

	iWritePos = m_pDataHead->iWrite;
	iReadPos = m_pDataHead->iRead;

	//首尾相等，无数据
	if (iReadPos == iWritePos)
	{
		iRetSize = m_pDataHead->iSize; //
	}
	//首大于尾，一般情况，iWritePos始终在"前"
	else if (iWritePos > iReadPos)
	{
		iRetSize = iWritePos - iReadPos;
	}
	//首小于尾，分开计算
	else
	{
		iRetSize = m_pDataHead->iSize - iWritePos + iReadPos;
	}

	//注意：最大长度减去预留部分长度，保证首尾不会相接
	iRetSize -= BUFFER_RESERVE_LENGTH;

	return iRetSize;
}
```

####	读操作
读取操作时需要移动 `iRead` 指针，需要考虑边界情况：
```cpp
int ShmRingQueue::GetDataUnit(char *pOut, int *pnOutLen)
{
	int iLeftSize = 0;
	int iReadPos = -1;
	int iWritePos = -1;
	char *pbyCodeBuf = m_pBuff;
	char *pTempSrc = NULL;
	char *pTempDst = NULL;

	//参数判断
	if ((NULL == pOut) || (NULL == pnOutLen))
	{
		return -1;
	}

	if (m_pDataHead->iOffset <= 0 || m_pDataHead->iSize <= 0)
	{
		return -1;
	}

	//取读写指针
	iReadPos = m_pDataHead->iRead; //
	iWritePos = m_pDataHead->iWrite;

	//无数据
	if (iReadPos == iWritePos)
	{
		*pnOutLen = 0;
		return 0;
	}

	//剩余缓冲大小,小于包长度字节数,错误返回
	iLeftSize = GetLeftSize();
	if (iLeftSize < sizeof(int))
	{
		//异常情况，重置首尾，返回错误
		*pnOutLen = 0;
		m_pDataHead->iRead = 0;
		m_pDataHead->iWrite = 0;
		return READ_INDEX_INVALID;
	}

	// copy data
	pTempDst = (char *)pnOutLen;
	pTempSrc = (char *)&pbyCodeBuf[0];

	//包长度编码
	for (int i = 0; i < sizeof(int); i++)
	{
		pTempDst[i] = pTempSrc[iReadPos];
		iReadPos = (iReadPos + 1) % m_pDataHead->iSize;
	}

	//数据包长度非法
	if (((*pnOutLen) > GetUsedSize()) || (*pnOutLen < 0))
	{
		*pnOutLen = 0;
		m_pDataHead->iRead = 0;
		m_pDataHead->iWrite = 0;
		return DATA_UNIT_INDEX_INVALID;
	}

	pTempDst = pOut;

	//首小于尾，未跨越终点
	if (iReadPos < iWritePos)
	{
		memcpy((void *)pTempDst, (const void *)&pTempSrc[iReadPos], (size_t)(*pnOutLen));
	}
	else
	{
		//首大于尾且出现分段，则需要分段拷贝
		int iRightLeftSize = m_pDataHead->iSize - iReadPos; //查看当前要读取的数据是否被分段了
		if (iRightLeftSize < *pnOutLen)
		{
			//分段拷贝
			memcpy((void *)pTempDst, (const void *)&pTempSrc[iReadPos], iRightLeftSize);
			pTempDst += iRightLeftSize;
			memcpy((void *)pTempDst, (const void *)&pTempSrc[0], (size_t)(*pnOutLen - iRightLeftSize));
		}
		//否则，直接拷贝（临界情况），待拷贝的数据长度没有跨越分段
		else
		{
			memcpy((void *)pTempDst, (const void *)&pTempSrc[iReadPos], (size_t)(*pnOutLen));
		}
	}

	//变更读指针
	iReadPos = (iReadPos + (*pnOutLen)) % m_pDataHead->iSize;
	//更新iRead
	m_pDataHead->iRead = iReadPos;

	return iReadPos;
}
```

####	写操作
与读取操作不同的是，写入需要移动 `iWrite` 指针
```cpp
//写入数据
int ShmRingQueue::PutDataUnit(const char *pIn, int nInLen)
{
	int iLeftSize = 0;
	int iRead = -1;
	int iWrite = -1;

	//参数判断
	if ((NULL == pIn) || (nInLen <= 0))
	{
		return -1;
	}

	if (m_pDataHead->iOffset <= 0 || m_pDataHead->iSize <= 0)
	{
		return -1;
	}

	//首先判断是已满
	if (IsFull())
	{
		return WRITE_INDEX_FULL;
	}

	//取首、尾
	iRead = m_pDataHead->iRead;
	iWrite = m_pDataHead->iWrite;

	//缓冲区异常判断处理
	if (iRead < 0 || iRead >= m_pDataHead->iSize || iWrite < 0 || iWrite >= m_pDataHead->iSize)
	{
		//非法的index，重置
		m_pDataHead->iWrite = 0;
		m_pDataHead->iRead = 0;
		return WRITE_INDEX_INVALID;
	}

	//剩余缓冲大小小于新来的数据,溢出了,返回错误
	iLeftSize = GetLeftSize();
	if ((int)(nInLen + sizeof(int)) > iLeftSize)
	{
		//空闲不够，无法写入
		return WRITE_INDEX_FULL;
	}

	//数据首指针
	char *pbyCodeBuf = m_pBuff;

	char *pTempSrc = NULL;
	char *pTempDst = NULL;

	pTempDst = &pbyCodeBuf[0];
	pTempSrc = (char *)&nInLen;

	//包的长度编码
	for (int i = 0; i < sizeof(nInLen); i++)
	{
		pTempDst[iWrite] = pTempSrc[i];
		iWrite = (iWrite + 1) % m_pDataHead->iSize;
	}

	//首大于尾，直接写入（说明W-R之间可写，且一定不会跨越分段，一旦跨越分段iRead必然小于iWrite）
	if (iRead > iWrite)
	{
		memcpy((void *)&pbyCodeBuf[iWrite], (const void *)pIn, (size_t)nInLen);
	}
	else
	{
		//首小于尾,本包长大于右边剩余空间,需要分两段循环放到buff存放
		if ((int)nInLen > (m_pDataHead->iSize - iWrite))
		{
			//右边剩余buff
			int iRightLeftSize = m_pDataHead->iSize - iWrite;
			memcpy((void *)&pbyCodeBuf[iWrite], (const void *)&pIn[0], (size_t)iRightLeftSize);
			memcpy((void *)&pbyCodeBuf[0], (const void *)&pIn[iRightLeftSize], (size_t)(nInLen - iRightLeftSize));
		}
		//右边剩余buff够了，直接写入即可
		else
		{
			memcpy((void *)&pbyCodeBuf[iWrite], (const void *)&pIn[0], (size_t)nInLen);
		}
	}

	//更新尾偏移
	iWrite = (iWrite + nInLen) % m_pDataHead->iSize;
	m_pDataHead->iWrite = iWrite;

	return iWrite;
}
```

##  0x04  总结
代码实现[在此](https://github.com/pandaychen/d-ringbuffer)

##  0x05  参考
- [Circular buffer](https://en.wikipedia.org/wiki/Circular_buffer)