---
layout:     post
title:  Linux 内核之旅（二十二）：内核视角下的IO读写（三）
subtitle:   内核视角下的读文件过程（续）
date:       2025-12-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
先回顾一下，调用read系统调用之后，内核的调用路径是什么？

![read-syscall-kernel-trace]()

再回顾一下，前文介绍的`splice`实现零拷贝中的内核读操作，是怎么实现的？

本文主要梳理下基于page cache的内核read实现的若干细节，基于[v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include)的源码

##  0x01    generic_file_read_iter的实现细节
通常大部分文件系统的读取read实现，都是将`read_iter`置为`generic_file_read_iter`，如本文分析的ext4系统
![generic_file_read_iter-flow]()

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/filemap.c#L2015
/**
 * generic_file_read_iter - generic filesystem read routine
 * @iocb:	kernel I/O control block
 * @iter:	destination for the data read
 *
 * This is the "read_iter()" routine for all filesystems
 * that can use the page cache directly.
 */
ssize_t
generic_file_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
	struct file *file = iocb->ki_filp;
	ssize_t retval = 0;
	size_t count = iov_iter_count(iter);

	if (!count)
		goto out; /* skip atime */

	if (iocb->ki_flags & IOCB_DIRECT) {
		//非page cache读
        ......
	}

    // buffered read
	retval = do_generic_file_read(file, &iocb->ki_pos, iter, retval);
out:
	return retval;
}
```

上面函数中`iov_iter_count`函数，就是**用户请求读取的字节数，也就是用户缓冲区的剩余容量**，回顾下前文的内容，`iov_iter`本质是一个迭代器，它的特点是：

1.	初始化时，`count`被设置为用户调用 `read(fd, buf, count)`时的 `count`参数
2.	随着数据拷贝（`page` copy到`iovec`），`count`逐渐减少（变化）
3.	当 `i->count`变为 `0` 时，表示用户缓冲区已满

```cpp
static inline size_t iov_iter_count(const struct iov_iter *i)
{
	return i->count;
}
```

继续分析`do_generic_file_read`的实现，`do_generic_file_read` 是内核处理缓冲 I/O（Buffered I/O）的核心函数，通过 Page Cache 机制读取文件数据。主要流程如下：

1.	计算页索引和偏移：将文件偏移转换为页号
2.	循环处理每一页：逐页从 Page Cache 读取
3.	页缓存命中/未命中处理
4.	数据拷贝到用户空间
5.	预读机制：优化顺序读取性能

在调用`do_generic_file_read`之前，主要注意`iov_iter`中保存了用户期望copy的长度以及分段`iovec`（缓冲区）

```cpp
//filp：要读取的文件对象
//ppos： 指向当前文件偏移量的指针
//iter：用户空间缓冲区的迭代器，支持分散/聚集 I/O
//written：已读取的字节数（用于部分读取的继续）
static ssize_t do_generic_file_read(struct file *filp, loff_t *ppos,
		struct iov_iter *iter, ssize_t written)
{
	struct address_space *mapping = filp->f_mapping;	// 文件的页缓存映射
	struct inode *inode = mapping->host;				// 文件的 inode（反向）
	struct file_ra_state *ra = &filp->f_ra;				// 预读状态
	pgoff_t index;			// 当前页索引
	pgoff_t last_index;		// 最后页索引
	pgoff_t prev_index;		
	unsigned long offset;    // 页内偏移  /* offset into pagecache page */	
	unsigned int prev_offset;
	int error = 0;

	if (unlikely(*ppos >= inode->i_sb->s_maxbytes))
		return 0;
	iov_iter_truncate(iter, inode->i_sb->s_maxbytes);

	// 重要：偏移量转换计算
	index = *ppos >> PAGE_SHIFT;		 // 计算当前页索引（注意：是当前文件对象）
	prev_index = ra->prev_pos >> PAGE_SHIFT;	
	prev_offset = ra->prev_pos & (PAGE_SIZE-1);
	last_index = (*ppos + iter->count + PAGE_SIZE-1) >> PAGE_SHIFT;	// 计算最后一页索引
	offset = *ppos & ~PAGE_MASK;		//计算页内偏移

	// 重要：page操作的核心循环流程
	for (;;) {	// 无限循环，处理多个页面
		struct page *page;
		pgoff_t end_index;
		loff_t isize;
		unsigned long nr, ret;

		// 条件调度：允许调度器中断
		cond_resched();
find_page:
		if (fatal_signal_pending(current)) {
			error = -EINTR;
			goto out;
		}

		// find_get_page：从页缓存查找
		// step1：查找页缓存
		page = find_get_page(mapping, index);
		if (!page) {
			// 页不在缓存（radix树）中，触发同步预读
			page_cache_sync_readahead(mapping,
					ra, filp,
					index, last_index - index);
			// 再次尝试查找
			page = find_get_page(mapping, index);
			if (unlikely(page == NULL))
				goto no_cached_page;	// 分配新页
		}

		// 重要：预读机制（step2）
		if (PageReadahead(page)) {
			// 异步预读：当读取到标记为预读的页面时触发
			page_cache_async_readahead(mapping,
					ra, filp, page,
					index, last_index - index);
		}

		//step3：页面状态检查
		if (!PageUptodate(page)) {	// 页面数据不是最新的
			/*
			 * See comment in do_read_cache_page on why
			 * wait_on_page_locked is used to avoid unnecessarily
			 * serialisations and why it's safe.
			 */
			error = wait_on_page_locked_killable(page);
			if (unlikely(error))
				goto readpage_error;
			if (PageUptodate(page))	// 等待后，页面已更新
				goto page_ok;

			// 检查是否为部分更新页（某些文件系统支持）
			if (inode->i_blkbits == PAGE_SHIFT ||
					!mapping->a_ops->is_partially_uptodate)
				goto page_not_up_to_date;
			/* pipes can't handle partially uptodate pages */
			// 管道不支持部分更新页
			if (unlikely(iter->type & ITER_PIPE))
				goto page_not_up_to_date;
			// 检查部分更新的具体状态
			if (!trylock_page(page))
				goto page_not_up_to_date;
			/* Did it get truncated before we got the lock? */
			if (!page->mapping)	// 页面已被截断
				goto page_not_up_to_date_locked;
			if (!mapping->a_ops->is_partially_uptodate(page,
							offset, iter->count))
				goto page_not_up_to_date_locked;
			unlock_page(page);
		}

		//step5：重要，page已经准备好，数据拷贝到用户空间
page_ok:
		/*
		 * i_size must be checked after we know the page is Uptodate.
		 *
		 * Checking i_size after the check allows us to calculate
		 * the correct value for "nr", which means the zero-filled
		 * part of the page is not copied back to userspace (unless
		 * another truncate extends the file - this is desired though).
		 */

		//检查文件大小边界
		isize = i_size_read(inode);
		end_index = (isize - 1) >> PAGE_SHIFT;
		if (unlikely(!isize || index > end_index)) {
			put_page(page);
			goto out;	// 已经读到文件末尾
		}

		/* nr is the maximum number of bytes to copy from this page */
		// 计算本页可拷贝的字节数
		nr = PAGE_SIZE;
		if (index == end_index) {	// 最后一页
			nr = ((isize - 1) & ~PAGE_MASK) + 1;	 // 计算文件在最后一页的字节数
			if (nr <= offset) {	// 偏移已超过文件末尾
				put_page(page);
				goto out;
			}
		}
		nr = nr - offset;	// 减去页内偏移

		/* If users can be writing to this page using arbitrary
		 * virtual addresses, take care about potential aliasing
		 * before reading the page on the kernel side.
		 */
		// 处理缓存一致性（写时拷贝等情况）
		if (mapping_writably_mapped(mapping))
			flush_dcache_page(page);

		/*
		 * When a sequential read accesses a page several times,
		 * only mark it as accessed the first time.
		 */
		// 标记页面访问（用于页面回收算法）
		if (prev_index != index || offset != prev_offset)
			mark_page_accessed(page);
		prev_index = index;

		/*
		 * Ok, we have the page, and it's up-to-date, so
		 * now we can copy it to user space...
		 */

		// copy_page_to_iter：拷贝数据到用户空间
		//负责从内核页拷贝数据到用户空间缓冲区，处理页边界、部分拷贝等情况，返回实际拷贝的字节数
		ret = copy_page_to_iter(page, offset, nr, iter);
		//offset：页面内的字节偏移
		//index：当前处理的页面索引
		//prev_offset：上一次访问的偏移（用于顺序性检测）
		//written：总共已读取的字节数
		offset += ret;
		index += offset >> PAGE_SHIFT;
		offset &= ~PAGE_MASK;
		prev_offset = offset;

		put_page(page);
		written += ret;
		if (!iov_iter_count(iter))
			goto out;
		if (ret < nr) {
			error = -EFAULT;
			goto out;
		}

		//继续下一页处理
		continue;

		//step4：从磁盘读取页面的过程
page_not_up_to_date:
		/* Get exclusive access to the page ... */
		error = lock_page_killable(page);
		if (unlikely(error))
			goto readpage_error;

page_not_up_to_date_locked:
		/* Did it get truncated before we got the lock? */
		if (!page->mapping) {	// 页面已被截断
			unlock_page(page);
			put_page(page);
			continue;	// 重新开始
		}

		/* Did somebody else fill it already? */
		if (PageUptodate(page)) {	 // 其他进程已更新页面
			unlock_page(page);
			goto page_ok;
		}

readpage:
		/*
		 * A previous I/O error may have been due to temporary
		 * failures, eg. multipath errors.
		 * PG_error will be set again if readpage fails.
		 */
		ClearPageError(page);	// 清除之前的错误
		/* Start the actual read. The read will unlock the page. */

		//重要：调用文件系统的 readpage 方法（实际磁盘读取）
		//比如对于ext4系统：调用ext4_readpage
		//此调用会提交 BIO 请求到底层块设备
		//https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/inode.c#L3224
		error = mapping->a_ops->readpage(filp, page);

		if (unlikely(error)) {
			if (error == AOP_TRUNCATED_PAGE) {
				put_page(page);
				error = 0;
				goto find_page;
			}
			goto readpage_error;
		}

		if (!PageUptodate(page)) {
			error = lock_page_killable(page);
			if (unlikely(error))
				goto readpage_error;
			if (!PageUptodate(page)) {
				if (page->mapping == NULL) {
					/*
					 * invalidate_mapping_pages got it
					 */
					unlock_page(page);
					put_page(page);
					goto find_page;
				}
				unlock_page(page);
				shrink_readahead_size_eio(filp, ra);
				error = -EIO;
				goto readpage_error;
			}
			unlock_page(page);
		}

		goto page_ok;

		//step7：错误处理
readpage_error:
		/* UHHUH! A synchronous read error occurred. Report it */
		put_page(page);
		goto out;


		//step 6：页面分配（缓存未命中）
no_cached_page:
		/*
		 * Ok, it wasn't cached, so we need to create a new
		 * page..
		 */
		// 分配cold页面，cold页面更适合一次性的读取操作，同时加入到页面缓存和 LRU 列表
		page = page_cache_alloc_cold(mapping);	
		if (!page) {
			error = -ENOMEM;
			goto out;
		}
		error = add_to_page_cache_lru(page, mapping, index,
				mapping_gfp_constraint(mapping, GFP_KERNEL));
		if (error) {
			put_page(page);
			if (error == -EEXIST) {	 // 其他进程已添加
				error = 0;
				goto find_page;
			}
			goto out;
		}
		goto readpage;	// 读取新分配的页面
	}

	//step8：完成
out:	
	ra->prev_pos = prev_index;
	ra->prev_pos <<= PAGE_SHIFT;
	ra->prev_pos |= prev_offset;

	*ppos = ((loff_t)index << PAGE_SHIFT) + offset;
	file_accessed(filp);	 // 更新文件访问时间
	return written ? written : error;
}
```

####	offset的意义
关于参数`pgoff_t offset`，`offset`是页索引（page index），表示文件被划分为页面大小的块后，从`0`开始计数的页面序号，来自文件偏移量。如在文件读取/写入操作中，`offset`从用户的文件偏移量计算而来（注意`offset`	代表页面个数，非字节）

```cpp
// 用户系统调用：read(fd, buf, count)
// 内核处理时：
loff_t pos = file->f_pos;  // 当前文件偏移（字节）
pgoff_t index = pos >> PAGE_SHIFT;  // 转换为页索引
unsigned int page_offset = pos & ~PAGE_MASK;  // 页内偏移

//对于类型pgoff_t

// offset 是 pgoff_t 类型，通常定义为 unsigned long
typedef unsigned long pgoff_t;

// 对于大于 4GB 的文件：
loff_t file_size = 10LL * 1024 * 1024 * 1024;  // 10GB
pgoff_t max_index = file_size >> PAGE_SHIFT;    // 2621440 个页面

// 在32位系统上：
// pgoff_t 是 32 位
// 最大文件大小 = 2^32 * 4096 = 16TB（实际上受文件系统和其他限制）
```

此外，在回顾一下，在IDR树中，key就是`offset`（页索引），而value 就是`struct page*`即页面指针


####	page_ok标签
`page_ok`标签处表示当前页面已准备就绪，可以进行用户空间拷贝。这个标签处理单个页面的读取完成，包括：

1.	检查文件边界
2.	计算可拷贝字节数
3.	处理缓存一致性
4.	执行实际拷贝
5.	更新状态并决定是否继续（检查退出状态）

```cpp
{
	......
page_ok:
		//检查文件大小边界
		isize = i_size_read(inode);
		end_index = (isize - 1) >> PAGE_SHIFT;
		if (unlikely(!isize || index > end_index)) {
			put_page(page);
			goto out;	// 已经读到文件末尾
		}

		/* nr is the maximum number of bytes to copy from this page */
		// 计算本页可拷贝的字节数
		nr = PAGE_SIZE;
		if (index == end_index) {	// 最后一页
			nr = ((isize - 1) & ~PAGE_MASK) + 1;	 // 计算文件在最后一页的字节数
			if (nr <= offset) {	// 偏移已超过文件末尾
				put_page(page);
				goto out;
			}
		}
		nr = nr - offset;	// 减去页内偏移

		// 处理缓存一致性（写时拷贝等情况）
		if (mapping_writably_mapped(mapping))
			flush_dcache_page(page);

		/*
		 * When a sequential read accesses a page several times,
		 * only mark it as accessed the first time.
		 */
		// 标记页面访问（用于页面回收算法）
		// 这里是执行page的顺序访问检测，只有第一次访问时才标记页面为"已访问"，这里影响页面回收算法的决策
		if (prev_index != index || offset != prev_offset)
			mark_page_accessed(page);
		prev_index = index;

		/*
		 * Ok, we have the page, and it's up-to-date, so
		 * now we can copy it to user space...
		 */

		// copy_page_to_iter：拷贝数据到用户空间
		//负责从内核页拷贝数据到用户空间缓冲区，处理页边界、部分拷贝等情况，返回实际拷贝的字节数
		ret = copy_page_to_iter(page, offset, nr, iter);
		offset += ret;					// 更新页内偏移
		index += offset >> PAGE_SHIFT;	// 如果offset跨页，index增加（下一次需要访问后面的page了）
		offset &= ~PAGE_MASK;			// 将offset限制在当前页内
		prev_offset = offset;			// 记录偏移用于下次访问检测

		put_page(page);	// 在每次循环结束时释放页面引用
		/*
		引用计数管理机制， 确保页面在使用期间不会被回收
		1、find_get_page()增加引用计数
		2、使用完毕后必须 put_page()
		*/
		written += ret;	// 累计已读取字节数
		if (!iov_iter_count(iter))	// 用户缓冲区用完了，退出
			goto out;
		if (ret < nr) {		// 拷贝失败，退出
			error = -EFAULT;
			goto out;
		}

		// 继续下一个循环，处理下一个页面
		continue;
	......
}
```

细节一：如何检测copy完成（退出）

从上面代码分析可知在`copy_page_to_iter`执行完对本page的copy动作完成之后，会依次检查这些（退出）条件是否满足：

-	通常情况下，用户缓冲区的size等于本次要copy的文件page的总字节大小，但实际跨越的page页数，需要由offset来决定，可能跨越多页
-	copy大文件，本次copy未到达文件的尾部，用户缓冲区已经耗尽
-	copy小文件，本次copy到达了文件尾部（到达了文件最后一页，但未占满最后一页），用户缓冲区还有剩余空间

1、检查用户缓冲区是否已满，代码片段：

```cpp
if (!iov_iter_count(iter))  // 用户缓冲区已空
    goto out;               // 退出循环，返回
```

2、检查`copy_page_to_iter`（参数`nr`为希望copy的字节数、返回值为实际copy的字节数）是否拷贝失败，代码片段：

```cpp
if (ret < nr) {             // 实际拷贝字节数小于预期拷贝字节数
    error = -EFAULT;        // 设置错误
    goto out;               // 退出循环
}
```

细节二：对边界条件处理，片段如下：

1、在copy前发现已经到达了文件的末尾，如下：

```cpp
isize = i_size_read(inode);
end_index = (isize - 1) >> PAGE_SHIFT;

if (unlikely(!isize || index > end_index)) {
    put_page(page);
    goto out;  // 说明到达了文件末尾
}
```

2、对最后一页的特殊处理，片段如下：

```cpp
nr = PAGE_SIZE;
if (index == end_index) {	 // 当前页面是最后一页
	nr = ((isize - 1) & ~PAGE_MASK) + 1;	 
	if (nr <= offset) {	// 如果要读取的偏移已超过有效数据，说明已经读取完成了，可以退出
		put_page(page);
		goto out;
	}
}
```

####	如何计算本次处理的页面数目
注意到`do_generic_file_read`中，使用到了`for(;;)`，那么这个循环的退出条件是什么？或者说本次`do_generic_file_read`处理（读取）了多少页是如何计算出来的？考虑下面几个关键因子：

```cpp
//参数ppos：对应要读取的文件偏移（指向当前文件偏移量的指针）

//用户请求的数据量
// 原始请求来自用户空间的 read() 调用
// 在 __vfs_read -> generic_file_read_iter -> do_generic_file_read
size_t count = iov_iter_count(iter);  // 用户请求的字节数

//	实际读取边界计算
//  计算要读取的页面范围
index = *ppos >> PAGE_SHIFT;                      // 起始页索引
offset = *ppos & ~PAGE_MASK;                      // 页内偏移
last_index = (*ppos + iter->count/*待读的总数*/ + PAGE_SIZE-1) >> PAGE_SHIFT;  // 最后一页索引
```

上面的片段中，使用了 `iter->count`（用户请求的字节数）来计算需要读取的页面范围，但这个计算只是估算，不是限制。此外，对于系统调用`read`而言，参数`ppos`来自于`struct file`结构体中的`f_pos`字段：

```cpp
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		// 获取文件偏移
		loff_t pos = file_pos_read(f.file);
		ret = vfs_read(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}
	return ret;
}

static inline loff_t file_pos_read(struct file *file)
{
	return file->f_pos;
}
```

所以，这里了解到如下几个关键信息：
-	（要读取）文件page的起始页面位置
-	起始页面从哪个offset开始读
-	最后一页的位置`last_index`
-	估算的读取页数（`[last_index,index]`）

实际读取中，还需要考虑当前文件的大小（即当前读指针指向的page内容实质已经不满一页），关联如下代码：

```cpp
......
// 在 page_ok 标签处：
// 1. 根据文件大小计算当前页可用的字节数 (nr)
if (index == end_index) {
    nr = ((isize - 1) & ~PAGE_MASK) + 1;  // 最后一页的有效字节
    if (nr <= offset) {
        put_page(page);
        goto out;  // 文件结束
    }
}
nr = nr - offset;  // 页内从 offset 开始的有效字节

// 2. copy_page_to_iter 会读取 min(nr, iov_iter_count(iter))
// 即会取 nr和 iov_iter_count(iter)的较小值进行拷贝
ret = copy_page_to_iter(page, offset, nr, iter);

......
```

当然针对大文件，最普遍的情况还是一直copy pages到用户缓冲区已经耗尽

此外，这里内核使用`for(;;)`，主要考虑到一个 `read` 系统调用可能跨越多个page内存页面（处理跨页读取），其中每个page都需要完成下面的操作：

1.	单独查找/分配
2.	单独从磁盘读取（如果不在page cache中）
3.	单独拷贝到用户空间

```cpp
for (;;) {
	......
	cond_resched();	// 允许调度器中断（防止无限循环）
find_page:	//提供直接跳转的标签
    ......
	// 处理某一页（page）
	copy_page_to_iter(......)
    // 在 page_ok 标签后：
    if (!iov_iter_count(iter))  // 用户缓冲区已满
        goto out;
    if (ret < nr) {  // copy_page_to_iter 拷贝不完整
        error = -EFAULT;
        goto out;
    }
    continue;  // 继续处理下一页
}
```

几个问题：

接下来继续将`do_generic_file_read`的实现拆解分析，首先看下对单个页page的处理逻辑

##	0x02	do_generic_file_read：计算页索引和偏移
本节主要分析下`find_get_page`的实现过程，注意到其入参`pgoff_t offset`，来源于`index = *ppos >> PAGE_SHIFT`， 即page在文件中的索引index，这让人很容易联想到内核IDR结构的key

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pagemap.h#L245
static inline struct page *find_get_page(struct address_space *mapping,
					pgoff_t offset)
{	
	//用于从页缓存中获取页面，支持多种获取模式
	return pagecache_get_page(mapping, offset, 0, 0);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/mm/filemap.c#L1269
//mapping：	页缓存所属的地址空间（文件映射）
//offset：	页面在文件中的页索引
//fgp_flags：获取页面的标志位，控制函数行为
//gfp_mask：内存分配的 GFP 标志
struct page *pagecache_get_page(struct address_space *mapping, pgoff_t offset,
	int fgp_flags, gfp_t gfp_mask)
{
	struct page *page;

repeat:
	page = find_get_entry(mapping, offset);
	if (radix_tree_exceptional_entry(page))
		page = NULL;
	if (!page)
		goto no_page;

	if (fgp_flags & FGP_LOCK) {
		if (fgp_flags & FGP_NOWAIT) {
			if (!trylock_page(page)) {
				put_page(page);
				return NULL;
			}
		} else {
			lock_page(page);
		}

		/* Has the page been truncated? */
		if (unlikely(page->mapping != mapping)) {
			unlock_page(page);
			put_page(page);
			goto repeat;
		}
		VM_BUG_ON_PAGE(page->index != offset, page);
	}

	if (page && (fgp_flags & FGP_ACCESSED)){
		//标记页面访问
		//将页面标记为活跃，避免被快速回收，同时更新页面在 LRU 链表中的位置
		mark_page_accessed(page);
	}

	// 未在radix树中查找到相关的文件页，需要新建
no_page:
	if (!page && (fgp_flags & FGP_CREAT)) {
		......

		// 分配页面
		page = __page_cache_alloc(gfp_mask);
		if (!page)
			return NULL;

		if (WARN_ON_ONCE(!(fgp_flags & FGP_LOCK)))
			fgp_flags |= FGP_LOCK;

		/* Init accessed so avoid atomic mark_page_accessed later */
		if (fgp_flags & FGP_ACCESSED)
			__SetPageReferenced(page);
		// 重要：添加到page cache和全局LRU链表
		// 将这个新申请的page根据index插入到普通文件的address_space对应的radix树中
		err = add_to_page_cache_lru(page, mapping, offset,
				gfp_mask & GFP_RECLAIM_MASK);
		if (unlikely(err)) {
			put_page(page);
			page = NULL;
			if (err == -EEXIST)	// 竞争条件：其他线程已添加
				goto repeat;
		}
	}

	return page;
}
```

`find_get_entry`函数用于从页缓存中查找页面，它无锁查找页缓存，并使用 RCU 机制确保并发安全，这里会调用`radix_tree_lookup_slot`在文件的IDR树中进行查找

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/filemap.c#L1169
//mapping：文件的页缓存地址空间
//offset：页面在文件中的索引
struct page *find_get_entry(struct address_space *mapping, pgoff_t offset)
{
	void **pagep;
	struct page *head, *page;

	rcu_read_lock();
repeat:
	page = NULL;
	/*
	radix_tree_lookup_slot：返回指向存储页面指针的槽位的指针，槽位存储的是 void *，可能是 struct page *；如果没有对应的条目，返回 NULL
	*/
	pagep = radix_tree_lookup_slot(&mapping->page_tree, offset);
	if (pagep) {
		page = radix_tree_deref_slot(pagep);
		if (unlikely(!page))
			goto out;
		if (radix_tree_exception(page)) {
			if (radix_tree_deref_retry(page))
				goto repeat;
			/*
			 * A shadow entry of a recently evicted page,
			 * or a swap entry from shmem/tmpfs.  Return
			 * it without attempting to raise page count.
			 */
			goto out;
		}

		head = compound_head(page);
		if (!page_cache_get_speculative(head))
			goto repeat;

		/* The page was split under us? */
		if (compound_head(page) != head) {
			put_page(head);
			goto repeat;
		}

		/*
		 * Has the page moved?
		 * This is part of the lockless pagecache protocol. See
		 * include/linux/pagemap.h for details.
		 */
		if (unlikely(page != *pagep)) {
			put_page(head);
			goto repeat;
		}
	}
out:
	rcu_read_unlock();

	return page;
}
```

`add_to_page_cache_lru`函数

```cpp
int add_to_page_cache_lru(struct page *page, struct address_space *mapping,
				pgoff_t offset, gfp_t gfp_mask)
{
	void *shadow = NULL;
	int ret;

	__SetPageLocked(page);
	// __add_to_page_cache_locked：像page cache增加radix节点（page）
	ret = __add_to_page_cache_locked(page, mapping, offset,
					 gfp_mask, &shadow);
	if (unlikely(ret))
		__ClearPageLocked(page);
	else {
		if (!(gfp_mask & __GFP_WRITE) &&
		    shadow && workingset_refault(shadow)) {
			SetPageActive(page);
			workingset_activation(page);
		} else
			ClearPageActive(page);

		// 向全局的page链表中增加节点
		lru_cache_add(page);
	}
	return ret;
}
```

所以，这里有个细节是，虽然每个文件（inode）对应的`address_space`有自己的私有radix树，但所有的页面page都会链接到全局的LRU链表中，使得内核可以进行全局回收

##	0x0	循环处理每一页

##	0x0	页缓存命中/未命中处理

##	0x0	预读机制
预读状态结构体如下：

```cpp
struct file_ra_state {
    pgoff_t start;                  // 预读窗口起始页
    unsigned int size;               // 当前预读窗口大小（页面数）
    unsigned int async_size;         // 异步预读大小
    unsigned int ra_pages;           // 最大预读页面数
    unsigned int mmap_miss;          // mmap 缓存未命中
    loff_t prev_pos;                 // 上次读取位置
};
```

####	readahead 状态

####	page_cache_sync_readahead VS page_cache_async_readahead

####	page_cache_sync_readahead的实现原理

####	page_cache_async_readahead的实现原理
[`page_cache_async_readahead`](https://elixir.bootlin.com/linux/v4.11.6/source/mm/readahead.c#L530)

##	0x0	数据拷贝到用户空间
[`iov_iter`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/uio.h#L30)结构的作用：

```cpp
struct iovec {
    void __user *iov_base;  // 用户空间缓冲区地址
    __kernel_size_t iov_len;  // 缓冲区长度
};

struct iov_iter {
    const struct iovec *iov;  // 当前 iovec
    unsigned long nr_segs;    // 剩余段数
    size_t iov_offset;        // 在当前 iovec 中的偏移
    size_t count;             // 剩余总字节数
	......
};
```

`copy_page_to_iter`：当page cache中的数据准备好之后，内核调用此函数将数据从内存copy至用户空间的缓冲区

`i->type`对应四种不同的迭代器类型：

```cpp
// include/linux/uio.h
#define ITER_IOVEC     0  // 用户空间 iovec，常用于readv/writev, sendmsg/recvmsg
#define ITER_KVEC      1  // 内核空间 kvec，常用于内核内部 I/O
#define ITER_BVEC      2  // 块 I/O 向量，直接 I/O, 块设备操作
#define ITER_PIPE      3  // 管道，关联pipe, splice 系统调用
```

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/lib/iov_iter.c#L642
//copy_page_to_iter：在read*调用中是内核通过 struct page访问文件内容并拷贝到用户空间的核心实现

/*参数
page：要拷贝的源页面（struct page*）
offset：页面内的字节偏移量
bytes：要拷贝的字节数
i：目标迭代器，表示用户空间缓冲区
*/
size_t copy_page_to_iter(struct page *page, size_t offset, size_t bytes,
			 struct iov_iter *i)
{
	if (i->type & (ITER_BVEC|ITER_KVEC)) {
		// 处理 bio_vec 或 kvec 类型的迭代器

		// 通过kmap_atomic映射页面到内核虚拟地址
		void *kaddr = kmap_atomic(page);

		// 基于内核的虚拟地址完成copy
		size_t wanted = copy_to_iter(kaddr + offset, bytes, i);
		// 解除映射
		kunmap_atomic(kaddr);
		return wanted;
	} else if (likely(!(i->type & ITER_PIPE)))
		return copy_page_to_iter_iovec(page, offset, bytes, i);
	else
		return copy_page_to_iter_pipe(page, offset, bytes, i);
}
```

##	0x0	copy_page_to_iter的实现

####	分支1：copy_page_to_iter对ITER_KVEC类型的处理
关联代码如下：
```cpp
void *kaddr = kmap_atomic(page);
// 基于内核的虚拟地址完成copy
size_t wanted = copy_to_iter(kaddr + offset, bytes, i);
// 解除映射
kunmap_atomic(kaddr);
```

在Linux内核中，`kmap_atomic`函数**用于将给定的页面（page）临时映射到内核虚拟地址空间，主要用于高端内存（High Memory）的映射**。在x86架构中，对于32位系统，由于虚拟地址空间有限（通常只有`4GB`），所以需要特殊处理高端内存。而对于64位系统，由于虚拟地址空间巨大（`128TB`用户空间 + `128TB`内核空间），理论上可以不需要高端内存映射，但为了兼容性和优化，内核仍然保留了相关机制（TODO）

```cpp
size_t copy_to_iter(const void *addr, size_t bytes, struct iov_iter *i)
{
	const char *from = addr;
	if (unlikely(i->type & ITER_PIPE))
		return copy_pipe_to_iter(addr, bytes, i);
	iterate_and_advance(i, bytes, v,
		__copy_to_user(v.iov_base, (from += v.iov_len) - v.iov_len,
			       v.iov_len),
		memcpy_to_page(v.bv_page, v.bv_offset,
			       (from += v.bv_len) - v.bv_len, v.bv_len),
		memcpy(v.iov_base, (from += v.iov_len) - v.iov_len, v.iov_len)
	)

	return bytes;
}
```

这里有一个细节问题，为何`copy_to_iter`的`addr`参数需要先把page转为内核虚拟地址呢？思考前文介绍的内核页表转换机制，涉及到内核内存管理和虚拟地址映射，内核需要虚拟地址访问物理内存

`copy_to_iter`需要内核虚拟地址作为源数据指针，而 `struct page`本身不提供这个地址，所以必须先用`kmap_atomic(page)`将物理页面映射到内核虚拟地址空间，然后将映射后的地址传递给 `copy_to_iter`

先回顾下CPU 工作原理，CPU 只能通过虚拟地址访问内存。当内核需要读写物理内存中的数据时，必须先建立虚拟地址到物理地址的映射

```text
// CPU访问流程
CPU发出指令：mov [0xffff888012345678], eax
    |
MMU查找页表：虚拟地址 0xffff888012345678 -> 物理地址 0x12345678
    |
内存控制器访问物理内存地址 0x12345678
```

####	kmap_atomic的作用
`kmap_atomic`是原子操作，作用是建立临时映射，从而使内核能访问物理页面，映射过程如下：

```text
物理页面 (PFN 0x12345)
    | 通过 page_to_pfn(page) 获取页帧号
物理地址 = 0x12345000
    | 通过 kmap_atomic 建立映射
内核虚拟地址 = 0xffffc90001234500
```

```cpp
//for x86_64
void *kmap_atomic(struct page *page)
{
    // 如果是低端内存，直接计算地址
    if (!PageHighMem(page))
        return page_address(page);	// 页面地址可以直接计算
    
    // 高端内存：分配固定映射槽位
    idx = type + KM_TYPE_NR * smp_processor_id();
    vaddr = __fix_to_virt(FIX_KMAP_BEGIN + idx);
    
    // 建立页表项：虚拟地址->物理页面
    set_pte_at(&init_mm, vaddr, mk_pte(page, kmap_prot));
    
    return (void *)vaddr;
}
```

TODO：低端内存 AND 高端内存的处理

####	copy_to_iter的实现
继续回到`copy_to_iter(kaddr + offset, bytes, i)`的调用，这里入参`kaddr + offset`也说明了从这个虚拟地址开始读取数据，其内部实现操作都需要使用虚拟内存地址

```cpp
//addr： 内核虚拟地址，指向源数据
//bytes： 要拷贝的字节数
//i： 目标迭代器
size_t copy_to_iter(const void *addr, size_t bytes, struct iov_iter *i)
{
	const char *from = addr;	//虚拟内存地址
	if (unlikely(i->type & ITER_PIPE))
		//用于从页缓存拷贝到管道的场景，非零拷贝
		return copy_pipe_to_iter(addr, bytes, i);
	iterate_and_advance(i, bytes, v,
		//下面函数都需要源地址作为参数，而这个源地址必须是可寻址的内核虚拟地址

		// 对于 ITER_IOVEC（用户空间缓冲区）
		__copy_to_user(v.iov_base, (from += v.iov_len) - v.iov_len,
			       v.iov_len),

		// 对于 ITER_BVEC（块I/O向量）
		memcpy_to_page(v.bv_page, v.bv_offset,
			       (from += v.bv_len) - v.bv_len, v.bv_len),

		// 对于 ITER_KVEC（内核空间缓冲区） 
		memcpy(v.iov_base, (from += v.iov_len) - v.iov_len, v.iov_len)
	)

	return bytes;
}
```

上面[`iterate_and_advance`](https://elixir.bootlin.com/linux/v4.11.6/source/lib/iov_iter.c#L94)是一个宏定义，简化定义如下：

```cpp
#define iterate_and_advance(i, n, v, I, B, K)           \
    while (n) {                                         \
        // 获取当前的段                            \
        v = i->current_segment();                       \
                                                        \
        // 计算这次可拷贝的长度                     \
        size_t len = min(n, v.len);                     \
                                                        \
        if (iov_iter_is_iovec(i)) {                     \
            // 对于iovec：用户空间拷贝                 \
            __copy_to_user(v.iov_base, from, len);      \
        } else if (iov_iter_is_bvec(i)) {               \
            // 对于bvec：页面拷贝                     \
            memcpy_to_page(v.bv_page, v.bv_offset,      \
                           from, len);                  \
        } else {                                        \
            // 对于kvec：内核空间拷贝                 \
            memcpy(v.iov_base, from, len);              \
        }                                               \
                                                        \
        from += len;                                    \
        n -= len;                                       \
        i->advance(len);                                \
    }
```

####	copy_page_to_iter_iovec的实现
`copy_page_to_iter_iovec`将一个内存页面的数据拷贝到由 `iovec` 数组描述的用户空间缓冲区

```cpp
//page：源页面
//offset：页面内的偏移量
//bytes：要拷贝的字节数
//i：迭代器，包含 iovec 数组信息
//https://elixir.bootlin.com/linux/v4.11.6/source/lib/iov_iter.c#L133
static size_t copy_page_to_iter_iovec(struct page *page, size_t offset, size_t bytes,
			 struct iov_iter *i)
{
	size_t skip, copy, left, wanted;
	const struct iovec *iov;
	char __user *buf;
	void *kaddr, *from;

	if (unlikely(bytes > i->count))
		bytes = i->count;	// 不能超过迭代器中剩余字节数

	if (unlikely(!bytes))
		return 0;	// 没有要拷贝的字节

	wanted = bytes;	// 保存原始请求的字节数（用户请求的字节数）
	iov = i->iov;	 // 获取当前 iovec（当前 iovec 结构指针）
	skip = i->iov_offset;	// 当前 iovec 中的偏移（在当前 iovec 中已处理的字节数）
	buf = iov->iov_base + skip;	// 目标用户空间地址（目标用户空间缓冲区地址）
	copy = min(bytes, iov->iov_len - skip);	// 本次可拷贝的字节数（本次调用可拷贝的字节数，取最小）

	//针对高端内存优化路径
	//fault_in_pages_writeable：由于在快速路径中，__copy_to_user_inatomic不处理缺页
	//的情况，所以调用此函数预故障处理，提前触发可能的缺页异常，如果用户空间地址不可写，会返回非0

	//fault_in_pages_writeable：提前触发可能的缺页异常，避免在原子上下文中处理
	if (IS_ENABLED(CONFIG_HIGHMEM) && !fault_in_pages_writeable(buf, copy)) {
		kaddr = kmap_atomic(page);	// 原子映射页面（很眼熟）
		from = kaddr + offset;		// 源地址

		/* first chunk, usually the only one */
		// 第一个块，通常是唯一的一个块

		// __copy_to_user_inatomic：原子的拷贝到用户空间，不会休眠，适合原子上下文；返回未能拷贝的字节数（0表示成功）
		left = __copy_to_user_inatomic(buf, from, copy);
		copy -= left;	// 减去未能拷贝的字节
		skip += copy;	// 更新当前 iovec 的偏移
		from += copy;	// 更新源地址偏移
		bytes -= copy;	 // 更新剩余字节数

		//case2：处理多个 iovec 的情况
		//这里循环处理的原因：用户缓冲区由多个 iovec 组成
		//第一个 iovec 填满后，继续填充下一个
		//直到所有字节拷贝完成或遇到错误
		while (unlikely(!left && bytes)) {
			iov++;	// 移动到下一个 iovec
			buf = iov->iov_base;	 // 新的目标地址
			copy = min(bytes, iov->iov_len);	// 计算本次可拷贝的字节数
			left = __copy_to_user_inatomic(buf, from, copy);
			copy -= left;	// 减去失败的字节
			skip = copy;	// 更新跳过字节数
			from += copy;	 // 更新源地址
			bytes -= copy;	// 更新剩余字节
		}
		if (likely(!bytes)) {
			// 所有字节已拷贝，跳转到结束处理
			kunmap_atomic(kaddr);
			goto done;
		}
		offset = from - kaddr;	 // 计算新的偏移
		buf += copy;			 // 更新目标地址
		kunmap_atomic(kaddr);	 // 解除原子映射
		copy = min(bytes, iov->iov_len - skip);	// 重新计算可拷贝字节
	}
	/* Too bad - revert to non-atomic kmap */
	//回退到非原子映射，为什么要回退呢？原因如下：
	//1. 原子映射只适合短时间持有
	//2. 如果拷贝没有完成，需要回退到普通映射
	kaddr = kmap(page);		//注意：这里更换为普通映射
	from = kaddr + offset;
	left = __copy_to_user(buf, from, copy);
	copy -= left;
	skip += copy;
	from += copy;
	bytes -= copy;
	while (unlikely(!left && bytes)) {
		iov++;
		buf = iov->iov_base;
		copy = min(bytes, iov->iov_len);
		left = __copy_to_user(buf, from, copy);
		copy -= left;
		skip = copy;
		from += copy;
		bytes -= copy;
	}
	kunmap(page);	// 解除普通映射

	//更新迭代器状态
done:
	if (skip == iov->iov_len) {	// 当前 iovec 已满
		iov++;	 	 // 移动到下一个
		skip = 0;	 // 重置偏移
	}
	i->count -= wanted - bytes;
	i->nr_segs -= iov - i->iov;
	i->iov = iov;
	i->iov_offset = skip;
	return wanted - bytes;
}
```

在上面实现中，`__copy_to_user_inatomic`用于原子拷贝，而`__copy_to_user`是非原子的，此外前者原子操作，不处理缺页；而后者可能休眠，可处理缺页

##	0x0	read系统调用到

TODO

##	0x0	零拷贝splice中的page读
`copy_page_to_iter_pipe`用于`splice`函数即零拷贝技术的底层实现，用于从管道读出数据。`splice`直接将页面从一个文件描述符的页缓存"移动"到另一个文件描述符的缓冲区，不经过用户空间，也不复制页面数据

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/lib/iov_iter.c#L342
static size_t copy_page_to_iter_pipe(struct page *page, size_t offset, size_t bytes,
			 struct iov_iter *i)
{
	struct pipe_inode_info *pipe = i->pipe;
	struct pipe_buffer *buf;
	size_t off;
	int idx;
	......
	off = i->iov_offset;
	idx = i->idx;
	buf = &pipe->bufs[idx];
	if (off) {
		if (offset == off && buf->page == page) {
			/* merge with the last one */
			buf->len += bytes;
			i->iov_offset += bytes;
			goto out;
		}
		idx = next_idx(idx, pipe);
		buf = &pipe->bufs[idx];
	}
	if (idx == pipe->curbuf && pipe->nrbufs)
		return 0;
	pipe->nrbufs++;
	buf->ops = &page_cache_pipe_buf_ops;
	// 零拷贝技术的核心：不是真正拷贝数据，而是添加页面引用
	get_page(buf->page = page);
	buf->offset = offset;
	buf->len = bytes;
	i->iov_offset = offset + bytes;
	i->idx = idx;
out:
	i->count -= bytes;
	//copy done
	return bytes;
}
```

####	do_generic_file_read中的页表转换


##  0x0 参考
-   [VFS：读文件的过程中发生了什么](https://zhuanlan.zhihu.com/p/268375848)
-   [read 文件一个字节实际会发生多大的磁盘IO？](https://cloud.tencent.com/developer/article/1964473)
-   [Linux内核中跟踪文件PageCache预读](https://mp.weixin.qq.com/s/8GIeK8C3bz8nbLcwmk1vcA?from=singlemessage&isappinstalled=0&scene=1&clicktime=1646449253&enterid=1646449253)
-   [The iov_iter interface](https://lwn.net/Articles/625077/)
-	[Linux readahead文件预读分析](https://zhuanlan.zhihu.com/p/690066876)
-	<<Linux 内核文件 Cache 机制>>
-	[Linux 内核的 IO 预读算法](https://www.bluepuni.com/archives/kernel-readahead/)