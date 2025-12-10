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

![]()

##  0x01    generic_file_read_iter的实现细节
通常大部分文件系统的读取read实现，都是将`read_iter`置为`generic_file_read_iter`，如本文分析的ext4系统

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

继续分析`do_generic_file_read`的实现，`do_generic_file_read` 是内核处理缓冲 I/O（Buffered I/O）的核心函数，通过 Page Cache 机制读取文件数据。主要流程如下：

1.	计算页索引和偏移：将文件偏移转换为页号
2.	循环处理每一页：逐页从 Page Cache 读取
3.	页缓存命中/未命中处理
4.	数据拷贝到用户空间
5.	预读机制：优化顺序读取性能

```cpp
//filp：要读取的文件对象
//ppos： 指向当前文件偏移量的指针
//iter：用户空间缓冲区的迭代器，支持分散/聚集 I/O
//written：已读取的字节数（用于部分读取的继续）
static ssize_t do_generic_file_read(struct file *filp, loff_t *ppos,
		struct iov_iter *iter, ssize_t written)
{
	struct address_space *mapping = filp->f_mapping;
	struct inode *inode = mapping->host;
	struct file_ra_state *ra = &filp->f_ra;
	pgoff_t index;
	pgoff_t last_index;
	pgoff_t prev_index;
	unsigned long offset;      /* offset into pagecache page */
	unsigned int prev_offset;
	int error = 0;

	if (unlikely(*ppos >= inode->i_sb->s_maxbytes))
		return 0;
	iov_iter_truncate(iter, inode->i_sb->s_maxbytes);

	index = *ppos >> PAGE_SHIFT;
	prev_index = ra->prev_pos >> PAGE_SHIFT;
	prev_offset = ra->prev_pos & (PAGE_SIZE-1);
	last_index = (*ppos + iter->count + PAGE_SIZE-1) >> PAGE_SHIFT;
	offset = *ppos & ~PAGE_MASK;

	for (;;) {
		struct page *page;
		pgoff_t end_index;
		loff_t isize;
		unsigned long nr, ret;

		cond_resched();
find_page:
		if (fatal_signal_pending(current)) {
			error = -EINTR;
			goto out;
		}

		page = find_get_page(mapping, index);
		if (!page) {
			page_cache_sync_readahead(mapping,
					ra, filp,
					index, last_index - index);
			page = find_get_page(mapping, index);
			if (unlikely(page == NULL))
				goto no_cached_page;
		}
		if (PageReadahead(page)) {
			page_cache_async_readahead(mapping,
					ra, filp, page,
					index, last_index - index);
		}
		if (!PageUptodate(page)) {
			/*
			 * See comment in do_read_cache_page on why
			 * wait_on_page_locked is used to avoid unnecessarily
			 * serialisations and why it's safe.
			 */
			error = wait_on_page_locked_killable(page);
			if (unlikely(error))
				goto readpage_error;
			if (PageUptodate(page))
				goto page_ok;

			if (inode->i_blkbits == PAGE_SHIFT ||
					!mapping->a_ops->is_partially_uptodate)
				goto page_not_up_to_date;
			/* pipes can't handle partially uptodate pages */
			if (unlikely(iter->type & ITER_PIPE))
				goto page_not_up_to_date;
			if (!trylock_page(page))
				goto page_not_up_to_date;
			/* Did it get truncated before we got the lock? */
			if (!page->mapping)
				goto page_not_up_to_date_locked;
			if (!mapping->a_ops->is_partially_uptodate(page,
							offset, iter->count))
				goto page_not_up_to_date_locked;
			unlock_page(page);
		}
page_ok:
		/*
		 * i_size must be checked after we know the page is Uptodate.
		 *
		 * Checking i_size after the check allows us to calculate
		 * the correct value for "nr", which means the zero-filled
		 * part of the page is not copied back to userspace (unless
		 * another truncate extends the file - this is desired though).
		 */

		isize = i_size_read(inode);
		end_index = (isize - 1) >> PAGE_SHIFT;
		if (unlikely(!isize || index > end_index)) {
			put_page(page);
			goto out;
		}

		/* nr is the maximum number of bytes to copy from this page */
		nr = PAGE_SIZE;
		if (index == end_index) {
			nr = ((isize - 1) & ~PAGE_MASK) + 1;
			if (nr <= offset) {
				put_page(page);
				goto out;
			}
		}
		nr = nr - offset;

		/* If users can be writing to this page using arbitrary
		 * virtual addresses, take care about potential aliasing
		 * before reading the page on the kernel side.
		 */
		if (mapping_writably_mapped(mapping))
			flush_dcache_page(page);

		/*
		 * When a sequential read accesses a page several times,
		 * only mark it as accessed the first time.
		 */
		if (prev_index != index || offset != prev_offset)
			mark_page_accessed(page);
		prev_index = index;

		/*
		 * Ok, we have the page, and it's up-to-date, so
		 * now we can copy it to user space...
		 */

		ret = copy_page_to_iter(page, offset, nr, iter);
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
		continue;

page_not_up_to_date:
		/* Get exclusive access to the page ... */
		error = lock_page_killable(page);
		if (unlikely(error))
			goto readpage_error;

page_not_up_to_date_locked:
		/* Did it get truncated before we got the lock? */
		if (!page->mapping) {
			unlock_page(page);
			put_page(page);
			continue;
		}

		/* Did somebody else fill it already? */
		if (PageUptodate(page)) {
			unlock_page(page);
			goto page_ok;
		}

readpage:
		/*
		 * A previous I/O error may have been due to temporary
		 * failures, eg. multipath errors.
		 * PG_error will be set again if readpage fails.
		 */
		ClearPageError(page);
		/* Start the actual read. The read will unlock the page. */
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

readpage_error:
		/* UHHUH! A synchronous read error occurred. Report it */
		put_page(page);
		goto out;

no_cached_page:
		/*
		 * Ok, it wasn't cached, so we need to create a new
		 * page..
		 */
		page = page_cache_alloc_cold(mapping);
		if (!page) {
			error = -ENOMEM;
			goto out;
		}
		error = add_to_page_cache_lru(page, mapping, index,
				mapping_gfp_constraint(mapping, GFP_KERNEL));
		if (error) {
			put_page(page);
			if (error == -EEXIST) {
				error = 0;
				goto find_page;
			}
			goto out;
		}
		goto readpage;
	}

out:
	ra->prev_pos = prev_index;
	ra->prev_pos <<= PAGE_SHIFT;
	ra->prev_pos |= prev_offset;

	*ppos = ((loff_t)index << PAGE_SHIFT) + offset;
	file_accessed(filp);
	return written ? written : error;
}
```

##	0x0	计算页索引和偏移

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pagemap.h#L245
static inline struct page *find_get_page(struct address_space *mapping,
					pgoff_t offset)
{
	return pagecache_get_page(mapping, offset, 0, 0);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/mm/filemap.c#L1269
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

	if (page && (fgp_flags & FGP_ACCESSED))
		mark_page_accessed(page);

no_page:
	if (!page && (fgp_flags & FGP_CREAT)) {
		int err;
		if ((fgp_flags & FGP_WRITE) && mapping_cap_account_dirty(mapping))
			gfp_mask |= __GFP_WRITE;
		if (fgp_flags & FGP_NOFS)
			gfp_mask &= ~__GFP_FS;

		page = __page_cache_alloc(gfp_mask);
		if (!page)
			return NULL;

		if (WARN_ON_ONCE(!(fgp_flags & FGP_LOCK)))
			fgp_flags |= FGP_LOCK;

		/* Init accessed so avoid atomic mark_page_accessed later */
		if (fgp_flags & FGP_ACCESSED)
			__SetPageReferenced(page);

		err = add_to_page_cache_lru(page, mapping, offset,
				gfp_mask & GFP_RECLAIM_MASK);
		if (unlikely(err)) {
			put_page(page);
			page = NULL;
			if (err == -EEXIST)
				goto repeat;
		}
	}

	return page;
}
```

##	0x0	循环处理每一页

##	0x0	页缓存命中/未命中处理

##	0x0	预读机制

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

##	0x0	总结

####	零拷贝splice
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


##  0x0 参考
-   [VFS：读文件的过程中发生了什么](https://zhuanlan.zhihu.com/p/268375848)
-   [read 文件一个字节实际会发生多大的磁盘IO？](https://cloud.tencent.com/developer/article/1964473)
-   [Linux内核中跟踪文件PageCache预读](https://mp.weixin.qq.com/s/8GIeK8C3bz8nbLcwmk1vcA?from=singlemessage&isappinstalled=0&scene=1&clicktime=1646449253&enterid=1646449253)
-   [The iov_iter interface](https://lwn.net/Articles/625077/)