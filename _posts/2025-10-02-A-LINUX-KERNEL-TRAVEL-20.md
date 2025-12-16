---
layout:     post
title:  Linux 内核之旅（二十）：内核视角下的共享内存
subtitle:   mmap与shm在内核的实现分析
date:       2025-09-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
共享内存主要用于进程间通信，常见Shared Memory机制：

-   System V shared memory（`shmget/shmat/shmdt`）：旧
-   POSIX shared memory（`shm_open/shm_unlink`）：新

此外，内存映射`mmap`机制也可以用于跨进程间通信，参考前文：[虚拟内存管理（上）](https://pandaychen.github.io/2024/11/05/A-LINUX-KERNEL-TRAVEL-2/)，当然了`mmap`也支持私有映射

-   mmap文件共享映射
-   mmap匿名共享映射

```TEXT
Shared file mappings：Sharing between unrelated processes, backed by file in filesystem
```

本文主要关注几个问题：
1.  mmap的实现机制
2.  shm的实现机制
3.  内核是如何实现共享的？

####    tmpfs

```BASH
[root@X-X-01 corefile]# df -Th
Filesystem      Type      Size  Used Avail Use% Mounted on
devtmpfs        devtmpfs   16G  4.0K   16G   1% /dev
tmpfs           tmpfs      16G   39M   16G   1% /dev/shm
tmpfs           tmpfs      16G  266M   16G   2% /run
tmpfs           tmpfs      16G     0   16G   0% /sys/fs/cgroup
/dev/vda1       ext4       99G   18G   77G  19% /
tmpfs           tmpfs     3.2G     0  3.2G   0% /run/user/0
/dev/vdb        ext4       98G   47G   47G  50% /data
X.X.X.X:/ nfs4      2.5T  1.9T  642G  75% /data/cfs/xxx
tmpfs           tmpfs     3.2G     0  3.2G   0% /run/user/30000
```

####    共享内存与tmpfs的关系
POSIX共享内存是基于tmpfs来实现的，System V shared memory在内核也是基于tmpfs实现的。tmpfs主要有两个作用：

1.  用于SYSTEM V共享内存，还有匿名内存映射；这部分由内核管理，用户不可见（理解为特殊的文件系统）
2.  用于POSIX共享内存，由用户负责mount，而且一般mount到`/dev/shm`；依赖于`CONFIG_TMPFS`

####    内核提供的几种共享内存机制


##  0x01    mmap的实现原理
`mmap`系统调用是将一个文件或者其它对象映射到进程的虚拟地址空间，实现磁盘地址和进程虚拟地址空间一段虚拟地址的一一对应关系。通过mmap系统调用可以让进程之间通过映射到同一个普通文件实现共享内存，普通文件被映射到进程虚拟地址空间当中后，进程可以像访问普通内存一样对文件进行一系列操作，而不需要通过 I/O 系统调用来读取或写入

`mmap`系统调用会将一个文件或其他对象映射到进程的地址空间中，并返回一个指向映射区域的指针，进程可以使用指针来访问映射区域的数据，就像访问内存一样

![mmap-flow]()

####	mmap文件共享的原理（回顾）
先回顾下，mmap实现文件共享映射的过程

调用 mmap 进行内存文件映射的时候，内核首先会在进程的虚拟内存空间中创建一个新的虚拟内存区域 VMA 用于映射文件，通过 `vm_area_struct->vm_file` 将映射文件的 `struct flle` 结构与虚拟内存映射关联起来

```cpp
struct vm_area_struct {
    unsigned long vm_flags; 	//标记为 MAP_SHARED 共享映射
    struct file * vm_file;      /* File we map to (can be NULL). */
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
}
```

根据 `vm_file->f_inode` 可以关联到映射文件的 `struct inode`，近而关联到映射文件在磁盘中的磁盘块 `i_block`，这个就是 mmap 内存文件映射的本质。站在文件系统的视角，映射文件中的数据是按照磁盘块来存储的，读写文件数据也是按照磁盘块为单位进行的，磁盘块大小为 `4K`，当进程读取磁盘块的内容到内存之后，站在内存管理系统的视角，磁盘块中的数据被 DMA 拷贝到了物理内存页中，这个物理内存页就是前面提到的文件页，一个文件包含多个磁盘块，当它们被读取到内存之后，一个文件也就对应了多个文件页，这些文件页在内存中统一被一个叫做 page cache 的结构所组织

当多个进程调用 mmap 对磁盘上的同一个文件进行共享文件映射的时候，也都只是在每个进程的虚拟内存空间中，创建出一段用于共享映射的虚拟内存区域 VMA 出来，随后内核会将各个进程中的这段虚拟内存映射区与映射文件关联起来，mmap 共享文件映射的逻辑就结束了，并没有和物理内存建立任何关系

![mmap-file-share-mapping-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/3/mmap-file-share-mapping-1.png)

当任意一个进程，比如上图中的进程 1 开始访问这段映射的虚拟内存时，CPU 会把虚拟内存地址送到 MMU 中进行地址翻译，因为 mmap 只是为进程分配了虚拟内存，并没有分配物理内存，所以这段映射的虚拟内存在页表中是没有页表项 PTE 的。随后 MMU 就会触发缺页异常（page fault），进程切换到内核态，在内核缺页中断处理程序中会发现引起缺页的这段 VMA 是共享文件映射的，所以内核会首先通过 `vm_area_struct->vm_pgoff` 在文件 page cache 中查找是否有缓存相应的文件页（映射的磁盘块对应的文件页）

![mmap-file-share-mapping-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/3/mmap-file-share-mapping-3.png)

如果文件页不在 page cache 中，内核则会在物理内存中分配一个内存页，然后将新分配的内存页加入到 page cache 中，并增加页引用计数。随后会通过 `address_space_operations` 重定义的 `readpage` （如`ext4_readpage`）激活块设备驱动从磁盘中读取映射的文件内容，然后将读取到的内容填充新分配的内存页

这里CPU访问虚拟内存，送到MMU翻译，继而发现页表中无PTE导致缺页中断的逻辑，对应于内核函数[`do_page_fault`](https://elixir.bootlin.com/linux/v4.11.6/source/arch/x86/mm/fault.c#L1446)，见下文分析

```cpp
struct vm_area_struct {
    unsigned long vm_pgoff;     /* Offset (within vm_file) in PAGE_SIZE */
}

static inline struct page *find_get_page(struct address_space *mapping,
     pgoff_t offset)
{
   return pagecache_get_page(mapping, offset, 0, 0);
}
```

至此文件中映射的内容已经加载进 page cache 了，此时物理内存才正式登场，在缺页中断处理程序的最后一步，内核会为映射的这段虚拟内存在页表中创建 PTE，然后将虚拟内存与 page cache 中的文件页通过 PTE 关联起来，缺页处理就结束了（这里指定的共享文件映射，所以 PTE 中文件页的权限是读写的，后续进程 1 在对这段虚拟内存区域写入的时候不会触发缺页中断，而是直接写入 page cache 中，整个过程没有切态，没有数据拷贝），当内核处理完缺页中断之后，mmap 共享文件映射在内核中的关系图如下：

![mmap-file-share-mapping-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/3/mmap-file-share-mapping-2.png)

此时在切换到进程 2 的视角中，虽然现在文件中被映射的这部分内容已经加载进物理内存页，并被缓存在文件的 page cache 中了。但是现在进程 2 中这段虚拟映射区在进程 2 页表中对应的 PTE 仍然是空的，当进程 2 访问这段虚拟映射区的时候依然会产生缺页中断。当进程 2 切换到内核态，处理缺页中断的时候，此时进程 2 通过 `vm_area_struct->vm_pgoff` 在 page cache 查找文件页的时候，文件页已经被进程 1 加载进 page cache 了，进程 2 一下就找到了，不需要再去磁盘中读取映射内容了，内核会直接为进程 2 创建 PTE （由于是共享文件映射，所以这里的 PTE 也是可写的），并插入到进程 2 页表中，随后将进程 2 中的虚拟映射区通过 PTE 与 page cache 中缓存的文件页映射关联起来

![mmap-file-share-mapping-4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/3/mmap-file-share-mapping-4.png)

现在进程 1 和进程 2 各自虚拟内存空间中的这段虚拟内存区域 VMA，已经共同映射到了文件的 page cache 中，由于文件的 page cache 在内核中只有一份，它是和进程无关的，page cache 中的内容发生的任何变化，进程 1 和进程 2 都是可以看到的

####    mmap内存映射
核心分为三个步骤：

1、用户进程启动映射过程，并在虚拟地址空间中为映射创建虚拟映射区域

-   进程在用户空间调用`mmap`，在当前进程的虚拟地址空间中，寻找一段空闲的满足要求的连续的虚拟地址区域
-   为此虚拟内存区域分配一个 `vm_area_struct` 结构，接着对这个结构的各个域进行初始化
-   将新建的虚拟区结构 `vm_area_struct` 插入进程的虚拟地址区域链表或红黑树中

2、调用内核空间的函数 `mmap`（不同于用户空间函数），实现**文件物理地址和进程虚拟地址的一一映射关系**

-   为映射分配了新的虚拟地址区域后，通过待映射的文件指针，在文件描述符表中找到对应的文件描述符，通过文件描述符，链接到内核**已打开文件集**中该文件的文件结构体（`struct file`），每个文件结构体维护着和这个已打开文件的相关各项信息
-   通过该文件的文件结构体，链接到 `file_operations` ，调用内核函数 `mmap`（`int mmap(struct file *filp, struct vm_area_struct *vma`），不同于用户空间库函数
-   内核 mmap 函数通过虚拟文件系统 inode 模块定位到文件磁盘物理地址
-   通过 `remap_pfn_range` 函数建立页表，即实现了文件地址和虚拟地址区域的映射关系。此时，这片虚拟地址并没有任何数据关联到主存中

3、进程发起对这片映射空间的访问，引发缺页异常，实现文件内容到物理内存（主存）的拷贝

-   进程的读或写操作访问虚拟地址空间这一段映射地址，通过查询页表，发现这一段地址并不在物理页面上。因为目前只建立了地址映射，真正的硬盘数据还没有拷贝到内存中，因此引发缺页异常
-   缺页异常进行一系列判断，确定无非法操作后，内核发起请求调页过程
-   调页过程先在交换缓存空间（swap cache）中寻找需要访问的内存页，如果没有则调用 nopage 函数把所缺的页从磁盘装入到主存中
-   之后进程即可对这片主存进行读或者写的操作，如果写操作改变了其内容，一定时间后系统会自动回写脏页面到对应磁盘地址，也即完成了写入到文件的过程（共享文件映射场景）

注意：修改过的脏页面并不会立即更新回文件中，而是有一段时间的延迟，可以调用`msync`来强制同步, 将修改过的内容立即保存到文件里

从`mmap`系统调用开始：

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/arch/x86/kernel/sys_x86_64.c#L87
SYSCALL_DEFINE6(mmap, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, off)
{
	long error;
	error = -EINVAL;
	if (off & ~PAGE_MASK)
		goto out;

	error = sys_mmap_pgoff(addr, len, prot, flags, fd, off >> PAGE_SHIFT);
out:
	return error;
}

//https://elixir.bootlin.com/linux/v4.11.6/source/arch/x86/kernel/sys_x86_64.c#L87
SYSCALL_DEFINE6(mmap_pgoff, unsigned long, addr, unsigned long, len,
		unsigned long, prot, unsigned long, flags,
		unsigned long, fd, unsigned long, pgoff)
{
	struct file *file = NULL;
	unsigned long retval;

	if (!(flags & MAP_ANONYMOUS)) {
		audit_mmap_fd(fd, flags);
		file = fget(fd);
		if (!file)
			return -EBADF;
		if (is_file_hugepages(file))
			len = ALIGN(len, huge_page_size(hstate_file(file)));
		retval = -EINVAL;
		if (unlikely(flags & MAP_HUGETLB && !is_file_hugepages(file)))
			goto out_fput;
	} else if (flags & MAP_HUGETLB) {
		struct user_struct *user = NULL;
		struct hstate *hs;

		hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & SHM_HUGE_MASK);
		if (!hs)
			return -EINVAL;

		len = ALIGN(len, huge_page_size(hs));
		/*
		 * VM_NORESERVE is used because the reservations will be
		 * taken when vm_ops->mmap() is called
		 * A dummy user value is used because we are not locking
		 * memory so no accounting is necessary
		 */
		file = hugetlb_file_setup(HUGETLB_ANON_FILE, len,
				VM_NORESERVE,
				&user, HUGETLB_ANONHUGE_INODE,
				(flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
		if (IS_ERR(file))
			return PTR_ERR(file);
	}

	flags &= ~(MAP_EXECUTABLE | MAP_DENYWRITE);

	retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
out_fput:
	if (file)
		fput(file);
	return retval;
}
```

`vm_mmap_pgoff`

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/util.c#L296
unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot,
	unsigned long flag, unsigned long pgoff)
{
	unsigned long ret;
	struct mm_struct *mm = current->mm;
	unsigned long populate;
	LIST_HEAD(uf);

	ret = security_mmap_file(file, prot, flag);
	if (!ret) {
		if (down_write_killable(&mm->mmap_sem))
			return -EINTR;
        // 
		ret = do_mmap_pgoff(file, addr, len, prot, flag, pgoff,
				    &populate, &uf);
		up_write(&mm->mmap_sem);
		userfaultfd_unmap_complete(mm, &uf);
		if (populate)
			mm_populate(ret, populate);
	}
	return ret;
}

//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/mm.h#L2118
static inline unsigned long
do_mmap_pgoff(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot, unsigned long flags,
	unsigned long pgoff, unsigned long *populate,
	struct list_head *uf)
{
	return do_mmap(file, addr, len, prot, flags, 0, pgoff, populate, uf);
}
```

####    do_mmap

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/mmap.c#L1306
unsigned long do_mmap(struct file *file, unsigned long addr,
			unsigned long len, unsigned long prot,
			unsigned long flags, vm_flags_t vm_flags,
			unsigned long pgoff, unsigned long *populate,
			struct list_head *uf)
{
	struct mm_struct *mm = current->mm;
	int pkey = 0;

	*populate = 0;

	if (!len)
		return -EINVAL;

	/*
	 * Does the application expect PROT_READ to imply PROT_EXEC?
	 *
	 * (the exception is when the underlying filesystem is noexec
	 *  mounted, in which case we dont add PROT_EXEC.)
	 */
	if ((prot & PROT_READ) && (current->personality & READ_IMPLIES_EXEC))
		if (!(file && path_noexec(&file->f_path)))
			prot |= PROT_EXEC;

	if (!(flags & MAP_FIXED))
		addr = round_hint_to_min(addr);

	/* Careful about overflows.. */
	len = PAGE_ALIGN(len);
	if (!len)
		return -ENOMEM;

	/* offset overflow? */
	if ((pgoff + (len >> PAGE_SHIFT)) < pgoff)
		return -EOVERFLOW;

	/* Too many mappings? */
	if (mm->map_count > sysctl_max_map_count)
		return -ENOMEM;

	/* Obtain the address to map to. we verify (or select) it and ensure
	 * that it represents a valid section of the address space.
	 */
	addr = get_unmapped_area(file, addr, len, pgoff, flags);
	if (offset_in_page(addr))
		return addr;

	if (prot == PROT_EXEC) {
		pkey = execute_only_pkey(mm);
		if (pkey < 0)
			pkey = 0;
	}

	/* Do simple checking here so the lower-level routines won't have
	 * to. we assume access permissions have been handled by the open
	 * of the memory object, so we don't do any here.
	 */
	vm_flags |= calc_vm_prot_bits(prot, pkey) | calc_vm_flag_bits(flags) |
			mm->def_flags | VM_MAYREAD | VM_MAYWRITE | VM_MAYEXEC;

	if (flags & MAP_LOCKED)
		if (!can_do_mlock())
			return -EPERM;

	if (mlock_future_check(mm, vm_flags, len))
		return -EAGAIN;

	if (file) {
		struct inode *inode = file_inode(file);

		switch (flags & MAP_TYPE) {
		case MAP_SHARED:
			if ((prot&PROT_WRITE) && !(file->f_mode&FMODE_WRITE))
				return -EACCES;

			/*
			 * Make sure we don't allow writing to an append-only
			 * file..
			 */
			if (IS_APPEND(inode) && (file->f_mode & FMODE_WRITE))
				return -EACCES;

			/*
			 * Make sure there are no mandatory locks on the file.
			 */
			if (locks_verify_locked(file))
				return -EAGAIN;

			vm_flags |= VM_SHARED | VM_MAYSHARE;
			if (!(file->f_mode & FMODE_WRITE))
				vm_flags &= ~(VM_MAYWRITE | VM_SHARED);

			/* fall through */
		case MAP_PRIVATE:
			if (!(file->f_mode & FMODE_READ))
				return -EACCES;
			if (path_noexec(&file->f_path)) {
				if (vm_flags & VM_EXEC)
					return -EPERM;
				vm_flags &= ~VM_MAYEXEC;
			}

			if (!file->f_op->mmap)
				return -ENODEV;
			if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))
				return -EINVAL;
			break;

		default:
			return -EINVAL;
		}
	} else {
		switch (flags & MAP_TYPE) {
		case MAP_SHARED:
			if (vm_flags & (VM_GROWSDOWN|VM_GROWSUP))
				return -EINVAL;
			/*
			 * Ignore pgoff.
			 */
			pgoff = 0;
			vm_flags |= VM_SHARED | VM_MAYSHARE;
			break;
		case MAP_PRIVATE:
			/*
			 * Set pgoff according to addr for anon_vma.
			 */
			pgoff = addr >> PAGE_SHIFT;
			break;
		default:
			return -EINVAL;
		}
	}

	/*
	 * Set 'VM_NORESERVE' if we should not account for the
	 * memory use of this mapping.
	 */
	if (flags & MAP_NORESERVE) {
		/* We honor MAP_NORESERVE if allowed to overcommit */
		if (sysctl_overcommit_memory != OVERCOMMIT_NEVER)
			vm_flags |= VM_NORESERVE;

		/* hugetlb applies strict overcommit unless MAP_NORESERVE */
		if (file && is_file_hugepages(file))
			vm_flags |= VM_NORESERVE;
	}

	addr = mmap_region(file, addr, len, vm_flags, pgoff, uf);
	if (!IS_ERR_VALUE(addr) &&
	    ((vm_flags & VM_LOCKED) ||
	     (flags & (MAP_POPULATE | MAP_NONBLOCK)) == MAP_POPULATE))
		*populate = len;
	return addr;
}
```

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/mmap.c#L1588
unsigned long mmap_region(struct file *file, unsigned long addr,
		unsigned long len, vm_flags_t vm_flags, unsigned long pgoff,
		struct list_head *uf)
{
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *vma, *prev;
	int error;
	struct rb_node **rb_link, *rb_parent;
	unsigned long charged = 0;

	/* Check against address space limit. */
	if (!may_expand_vm(mm, vm_flags, len >> PAGE_SHIFT)) {
		unsigned long nr_pages;

		/*
		 * MAP_FIXED may remove pages of mappings that intersects with
		 * requested mapping. Account for the pages it would unmap.
		 */
		nr_pages = count_vma_pages_range(mm, addr, addr + len);

		if (!may_expand_vm(mm, vm_flags,
					(len >> PAGE_SHIFT) - nr_pages))
			return -ENOMEM;
	}

	/* Clear old maps */
	while (find_vma_links(mm, addr, addr + len, &prev, &rb_link,
			      &rb_parent)) {
		if (do_munmap(mm, addr, len, uf))
			return -ENOMEM;
	}

	/*
	 * Private writable mapping: check memory availability
	 */
	if (accountable_mapping(file, vm_flags)) {
		charged = len >> PAGE_SHIFT;
		if (security_vm_enough_memory_mm(mm, charged))
			return -ENOMEM;
		vm_flags |= VM_ACCOUNT;
	}

	/*
	 * Can we just expand an old mapping?
	 */
	vma = vma_merge(mm, prev, addr, addr + len, vm_flags,
			NULL, file, pgoff, NULL, NULL_VM_UFFD_CTX);
	if (vma)
		goto out;

	/*
	 * Determine the object being mapped and call the appropriate
	 * specific mapper. the address has already been validated, but
	 * not unmapped, but the maps are removed from the list.
	 */
	vma = kmem_cache_zalloc(vm_area_cachep, GFP_KERNEL);
	if (!vma) {
		error = -ENOMEM;
		goto unacct_error;
	}

	vma->vm_mm = mm;
	vma->vm_start = addr;
	vma->vm_end = addr + len;
	vma->vm_flags = vm_flags;
	vma->vm_page_prot = vm_get_page_prot(vm_flags);
	vma->vm_pgoff = pgoff;
	INIT_LIST_HEAD(&vma->anon_vma_chain);

	if (file) {
		if (vm_flags & VM_DENYWRITE) {
			error = deny_write_access(file);
			if (error)
				goto free_vma;
		}
		if (vm_flags & VM_SHARED) {
			error = mapping_map_writable(file->f_mapping);
			if (error)
				goto allow_write_and_free_vma;
		}

		/* ->mmap() can change vma->vm_file, but must guarantee that
		 * vma_link() below can deny write-access if VM_DENYWRITE is set
		 * and map writably if VM_SHARED is set. This usually means the
		 * new file must not have been exposed to user-space, yet.
		 */
        //重要：
		vma->vm_file = get_file(file);
		error = call_mmap(file, vma);
		if (error)
			goto unmap_and_free_vma;

		/* Can addr have changed??
		 *
		 * Answer: Yes, several device drivers can do it in their
		 *         f_op->mmap method. -DaveM
		 * Bug: If addr is changed, prev, rb_link, rb_parent should
		 *      be updated for vma_link()
		 */
		WARN_ON_ONCE(addr != vma->vm_start);

		addr = vma->vm_start;
		vm_flags = vma->vm_flags;
	} else if (vm_flags & VM_SHARED) {
		error = shmem_zero_setup(vma);
		if (error)
			goto free_vma;
	}

	vma_link(mm, vma, prev, rb_link, rb_parent);
	/* Once vma denies write, undo our temporary denial count */
	if (file) {
		if (vm_flags & VM_SHARED)
			mapping_unmap_writable(file->f_mapping);
		if (vm_flags & VM_DENYWRITE)
			allow_write_access(file);
	}
	file = vma->vm_file;
out:
	perf_event_mmap(vma);

	vm_stat_account(mm, vm_flags, len >> PAGE_SHIFT);
	if (vm_flags & VM_LOCKED) {
		if (!((vm_flags & VM_SPECIAL) || is_vm_hugetlb_page(vma) ||
					vma == get_gate_vma(current->mm)))
			mm->locked_vm += (len >> PAGE_SHIFT);
		else
			vma->vm_flags &= VM_LOCKED_CLEAR_MASK;
	}

	if (file)
		uprobe_mmap(vma);

	/*
	 * New (or expanded) vma always get soft dirty status.
	 * Otherwise user-space soft-dirty page tracker won't
	 * be able to distinguish situation when vma area unmapped,
	 * then new mapped in-place (which must be aimed as
	 * a completely new data area).
	 */
	vma->vm_flags |= VM_SOFTDIRTY;

	vma_set_page_prot(vma);

	return addr;

unmap_and_free_vma:
	vma->vm_file = NULL;
	fput(file);

	/* Undo any partial mapping done by a device driver. */
	unmap_region(mm, vma, prev, vma->vm_start, vma->vm_end);
	charged = 0;
	if (vm_flags & VM_SHARED)
		mapping_unmap_writable(file->f_mapping);
allow_write_and_free_vma:
	if (vm_flags & VM_DENYWRITE)
		allow_write_access(file);
free_vma:
	kmem_cache_free(vm_area_cachep, vma);
unacct_error:
	if (charged)
		vm_unacct_memory(charged);
	return error;
}

```

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/fs.h#L1736
static inline int call_mmap(struct file *file, struct vm_area_struct *vma)
{
    //对ext4系统而言
	return file->f_op->mmap(file, vma);
}
```

从`do_mmap`的实现，

##  0x0 mmap文件映射实现
以ext4文件系统继续分析，对应的回调函数如下：

```cpp
const struct file_operations ext4_file_operations = {
	......
	.mmap		= ext4_file_mmap,
	.get_unmapped_area = thp_get_unmapped_area,
    ......
};
```

```cpp
static const struct vm_operations_struct ext4_file_vm_ops = {
	.fault		= ext4_filemap_fault,
	.map_pages	= filemap_map_pages,
	.page_mkwrite   = ext4_page_mkwrite,
};
```

```cpp
static int ext4_file_mmap(struct file *file, struct vm_area_struct *vma)
{
	struct inode *inode = file->f_mapping->host;

	if (unlikely(ext4_forced_shutdown(EXT4_SB(inode->i_sb))))
		return -EIO;

	if (ext4_encrypted_inode(inode)) {
		int err = fscrypt_get_encryption_info(inode);
		if (err)
			return 0;
		if (!fscrypt_has_encryption_key(inode))
			return -ENOKEY;
	}
	file_accessed(file);
	if (IS_DAX(file_inode(file))) {
		vma->vm_ops = &ext4_dax_vm_ops;
		vma->vm_flags |= VM_MIXEDMAP | VM_HUGEPAGE;
	} else {
		vma->vm_ops = &ext4_file_vm_ops;
	}
	return 0;
}
```



##  0x0 mmap匿名映射实现

##	0x0	缺页中断的主要处理
缺页中断的核心逻辑中，涉及到的主要内核结构及成员如下：

```cpp
// vma
struct vm_area_struct {
    unsigned long vm_pgoff;        // 重要：在映射文件中的偏移（页为单位）
    struct file * vm_file;         // 映射的文件指针
    const struct vm_operations_struct *vm_ops;  // 操作集，包含fault函数
    ......
};

// vm_fault
struct vm_fault {
    pgoff_t pgoff;                 // 缺页地址在文件中的页偏移
    unsigned long address;         // 缺页的虚拟地址
    struct vm_area_struct *vma;    // 对应的VMA
    ......
};
```

当MMU触发缺页异常时，内核会进入缺页处理流程，缺页异常处理入口函数如下：

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/arch/x86/mm/fault.c#L1446
dotraplinkage void notrace
do_page_fault(struct pt_regs *regs, unsigned long error_code)
{
	unsigned long address = read_cr2(); /* Get the faulting address */
	enum ctx_state prev_state;

	prev_state = exception_enter();
	__do_page_fault(regs, error_code, address);
	exception_exit(prev_state);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/arch/x86/mm/fault.c#L1215
static noinline void
__do_page_fault(struct pt_regs *regs, unsigned long error_code,
		unsigned long address)
{
	struct vm_area_struct *vma;
	struct task_struct *tsk;
	struct mm_struct *mm;
	int fault, major = 0;
	unsigned int flags = FAULT_FLAG_ALLOW_RETRY | FAULT_FLAG_KILLABLE;

	tsk = current;
	mm = tsk->mm;
	......
	
	// 根据虚拟内存地址找到vma
	vma = find_vma(mm, address);
	if (unlikely(!vma)) {
		bad_area(regs, error_code, address);
		return;
	}

	// 合法的vma校验
	if (likely(vma->vm_start <= address))
		goto good_area;
	if (unlikely(!(vma->vm_flags & VM_GROWSDOWN))) {
		bad_area(regs, error_code, address);
		return;
	}
	......
	/*
	 * Ok, we have a good vm_area for this memory access, so
	 * we can handle it..
	 */
good_area:
	if (unlikely(access_error(error_code, vma))) {
		bad_area_access_error(regs, error_code, address, vma);
		return;
	}

	// 核心：处理缺页中断的函数
	fault = handle_mm_fault(vma, address, flags);
	major |= fault & VM_FAULT_MAJOR;

	......
	check_v8086_mode(regs, address, tsk);
}
```

而对于文件映射的缺页处理，调用链如下：

```text
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L3870
handle_mm_fault
	//https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L3787
    -> __handle_mm_fault
		//https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L3699
        -> handle_pte_fault
			//https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L3504
            -> do_fault
                -> do_shared_fault
                -> do_read_fault
```

上面流程对应的主要内核函数如下：

-	[`do_anonymous_page`](https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L2896)：处理匿名映射的缺页
-	[`do_fault`](https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L3504)：处理文件映射的缺页

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pagemap.h#L423
static inline pgoff_t linear_page_index(struct vm_area_struct *vma,
					unsigned long address)
{
	pgoff_t pgoff;
	......
	pgoff = (address - vma->vm_start) >> PAGE_SHIFT;
	pgoff += vma->vm_pgoff;
	return pgoff;
}

static int __handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
		unsigned int flags)
{
	struct vm_fault vmf = {
		.vma = vma,
		.address = address & PAGE_MASK,
		.flags = flags,
		.pgoff = linear_page_index(vma, address),	//重要：计算文件页偏移
		.gfp_mask = __get_fault_gfp_mask(vma),
	};
	struct mm_struct *mm = vma->vm_mm;
	pgd_t *pgd;
	p4d_t *p4d;
	int ret;

	pgd = pgd_offset(mm, address);
	p4d = p4d_alloc(mm, pgd, address);
	if (!p4d)
		return VM_FAULT_OOM;

	vmf.pud = pud_alloc(mm, p4d, address);
	if (!vmf.pud)
		return VM_FAULT_OOM;
	if (pud_none(*vmf.pud) && transparent_hugepage_enabled(vma)) {
		ret = create_huge_pud(&vmf);
		if (!(ret & VM_FAULT_FALLBACK))
			return ret;
	} else {
		pud_t orig_pud = *vmf.pud;

		barrier();
		if (pud_trans_huge(orig_pud) || pud_devmap(orig_pud)) {
			unsigned int dirty = flags & FAULT_FLAG_WRITE;

			/* NUMA case for anonymous PUDs would go here */

			if (dirty && !pud_write(orig_pud)) {
				ret = wp_huge_pud(&vmf, orig_pud);
				if (!(ret & VM_FAULT_FALLBACK))
					return ret;
			} else {
				huge_pud_set_accessed(&vmf, orig_pud);
				return 0;
			}
		}
	}

	vmf.pmd = pmd_alloc(mm, vmf.pud, address);
	if (!vmf.pmd)
		return VM_FAULT_OOM;
	if (pmd_none(*vmf.pmd) && transparent_hugepage_enabled(vma)) {
		ret = create_huge_pmd(&vmf);
		if (!(ret & VM_FAULT_FALLBACK))
			return ret;
	} else {
		pmd_t orig_pmd = *vmf.pmd;

		barrier();
		if (pmd_trans_huge(orig_pmd) || pmd_devmap(orig_pmd)) {
			if (pmd_protnone(orig_pmd) && vma_is_accessible(vma))
				return do_huge_pmd_numa_page(&vmf, orig_pmd);

			if ((vmf.flags & FAULT_FLAG_WRITE) &&
					!pmd_write(orig_pmd)) {
				ret = wp_huge_pmd(&vmf, orig_pmd);
				if (!(ret & VM_FAULT_FALLBACK))
					return ret;
			} else {
				huge_pmd_set_accessed(&vmf, orig_pmd);
				return 0;
			}
		}
	}

	return handle_pte_fault(&vmf);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L3699
static int handle_pte_fault(struct vm_fault *vmf)
{
	......
	if (!vmf->pte) {
		if (vma_is_anonymous(vmf->vma)){
			// 匿名映射的缺页
			return do_anonymous_page(vmf);
		}
		else{
			// 文件映射的缺页
			return do_fault(vmf);
		}
	}
	......
}

static int do_fault(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
	int ret;

	/* The VMA was not fully populated on mmap() or missing VM_DONTEXPAND */
	if (!vma->vm_ops->fault)
		ret = VM_FAULT_SIGBUS;
	else if (!(vmf->flags & FAULT_FLAG_WRITE))	//只读
		ret = do_read_fault(vmf);
	else if (!(vma->vm_flags & VM_SHARED))
		ret = do_cow_fault(vmf);
	else
		ret = do_shared_fault(vmf);	// 共享+读写

	/* preallocated pagetable is unused: free it */
	if (vmf->prealloc_pte) {
		pte_free(vma->vm_mm, vmf->prealloc_pte);
		vmf->prealloc_pte = NULL;
	}
	return ret;
}
```

继续，以`do_read_fault->__do_fault`调用链路，`__do_fault`中的`vma->vm_ops->fault`，这里的`fault`在ext4文件系统中就对应着`ext4_file_vm_ops`的`fault`成员，即`ext4_filemap_fault`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/inode.c#L5959)

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L3397
static int do_read_fault(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
	int ret = 0;

	......
	//
	ret = __do_fault(vmf);
	if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
		return ret;

	ret |= finish_fault(vmf);
	unlock_page(vmf->page);
	if (unlikely(ret & (VM_FAULT_ERROR | VM_FAULT_NOPAGE | VM_FAULT_RETRY)))
		put_page(vmf->page);
	return ret;
}

//https://elixir.bootlin.com/linux/v4.11.6/source/mm/memory.c#L3006
static int __do_fault(struct vm_fault *vmf)
{
	struct vm_area_struct *vma = vmf->vma;
	int ret;

	//CALL ext4_filemap_fault
	ret = vma->vm_ops->fault(vmf);
	......

	return ret;
}


int ext4_filemap_fault(struct vm_fault *vmf)
{
	struct inode *inode = file_inode(vmf->vma->vm_file);
	int err;

	down_read(&EXT4_I(inode)->i_mmap_sem);
	// 最终调用filemap_fault实现
	err = filemap_fault(vmf);
	up_read(&EXT4_I(inode)->i_mmap_sem);

	return err;
}
```

最终`ext4_filemap_fault`还是会调用`filemap_fault`完成缺页中断最后的部分，`filemap_fault`是通用文件系统缺页处理函数，主要任务是在page cache中查找或加载文件页，以满足文件内存映射的缺页需求

注意在`filemap_fault`中会调用`find_get_page`函数对文件页进行查找操作，本质上是调用page cache IDR的查找[方法](https://elixir.bootlin.com/linux/v4.11.6/source/mm/filemap.c#L1190)`radix_tree_lookup_slot`

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/filemap.c#L2174
int filemap_fault(struct vm_fault *vmf)
{
	int error;
	struct file *file = vmf->vma->vm_file;		// 获取mmap映射的文件
	struct address_space *mapping = file->f_mapping;	// 文件对应的地址空间（找到radix树即page cache管理入口）
	struct file_ra_state *ra = &file->f_ra;	 	// 预读状态
	struct inode *inode = mapping->host;		// 文件inode
	pgoff_t offset = vmf->pgoff;	// 这就是从vm_area_struct获取的偏移
	struct page *page;
	loff_t size;
	int ret = 0;

	// 边界检查，检查请求的页偏移是否超出文件大小
	size = round_up(i_size_read(inode), PAGE_SIZE);
	if (offset >= size >> PAGE_SHIFT)
		return VM_FAULT_SIGBUS;

	/*
	 * Do we have something in the page cache already?
	 */
	// 1. 首先在page cache中查找（第一次查找），首先尝试在page cache中查找指定偏移的页
	page = find_get_page(mapping, offset);
	if (likely(page) && !(vmf->flags & FAULT_FLAG_TRIED)) {
		/*
		 * We found the page, so try async readahead before
		 * waiting for the lock.
		 */
		// 页面在cache中找到，页面命中处理
		// 执行异步预读，预读后续页面
		do_async_mmap_readahead(vmf->vma, ra, file, page, offset);
	} else if (!page) {
		// 2. 页面不在cache中，进行同步预读（页面未命中处理）
		/* No page in the page cache at all */
		do_sync_mmap_readahead(vmf->vma, ra, file, offset);

		//记录主要缺页事件（PGMAJFAULT）
		count_vm_event(PGMAJFAULT);
		mem_cgroup_count_vm_event(vmf->vma->vm_mm, PGMAJFAULT);
		
		//设置返回码为VM_FAULT_MAJOR（表示需要磁盘I/O）
		ret = VM_FAULT_MAJOR;
retry_find:
		// 3. 再次查找（重新尝试查找页面）
		page = find_get_page(mapping, offset);
		if (!page){
			// 4. 如果仍然没有，从磁盘读取
			goto no_cached_page;
		}
	}

	// 5. 等待页面就绪（尝试锁定页面），防止并发修改
	if (!lock_page_or_retry(page, vmf->vma->vm_mm, vmf->flags)) {
		put_page(page);
		return ret | VM_FAULT_RETRY;
	}

	// 6. 页面一致性检查
	//检查页面是否仍然属于同一个address_space，验证页面索引是否匹配请求的偏移
	if (unlikely(page->mapping != mapping)) {
		unlock_page(page);
		put_page(page);
		goto retry_find;	//如果不匹配，回退跳转到重新查找
	}
	VM_BUG_ON_PAGE(page->index != offset, page);

	// 7. 检查页面是否最新（是否与磁盘同步），如果不是最新，跳转到重新读取逻辑
	if (unlikely(!PageUptodate(page)))
		goto page_not_uptodate;

	/*
	 * Found the page and have a reference on it.
	 * We must recheck i_size under page lock.
	 */

	// 8. 再次边界检查，在获取页面锁后再次检查文件大小，防止在锁定期间文件被截断
	size = round_up(i_size_read(inode), PAGE_SIZE);
	if (unlikely(offset >= size >> PAGE_SHIFT)) {
		unlock_page(page);
		put_page(page);
		return VM_FAULT_SIGBUS;
	}

	// 成功情况下返回，将找到的页面存入vmf->page，另外返回VM_FAULT_LOCKED表示页面已锁定
	vmf->page = page;
	return ret | VM_FAULT_LOCKED;

	//下面是缺页处理的错误路径：
	//1、页面不在缓存中
	//2、页面非最新
no_cached_page:
	/*
	 * We're only likely to ever get here if MADV_RANDOM is in
	 * effect.
	 */
	// 如果页面不在page cache，调用page_cache_read分配新页并触发读取
	// 如果成功，跳回retry_find重新查找
	// page_cache_read的主要功能是申请page，并把page加入全局链表
	error = page_cache_read(file, offset, vmf->gfp_mask);

	/*
	 * The page we want has now been added to the page cache.
	 * In the unlikely event that someone removed it in the
	 * meantime, we'll just come back here and read it again.
	 */
	if (error >= 0)
		goto retry_find;

	/*
	 * An error return from page_cache_read can result if the
	 * system is low on memory, or a problem occurs while trying
	 * to schedule I/O.
	 */
	if (error == -ENOMEM)
		return VM_FAULT_OOM;
	return VM_FAULT_SIGBUS;

page_not_uptodate:
	/*
	 * Umm, take care of errors if the page isn't up-to-date.
	 * Try to re-read it _once_. We do this synchronously,
	 * because there really aren't any performance issues here
	 * and we need to check for errors.
	 */
	//如果页面不是最新，需要调用文件系统的readpage方法重新读取
	ClearPageError(page);
	error = mapping->a_ops->readpage(file, page);
	if (!error) {
		//等待读取完成
		//如果读取成功，重新查找页面（retry_find的部分）
		wait_on_page_locked(page);
		if (!PageUptodate(page))
			error = -EIO;
	}
	put_page(page);

	if (!error || error == AOP_TRUNCATED_PAGE)
		goto retry_find;

	/* Things didn't work out. Return zero to tell the mm layer so. */
	shrink_readahead_size_eio(file, ra);
	return VM_FAULT_SIGBUS;
}
```

整体的缺页流程处理基本结束，这里再说明下两个细节：

1、当页面不在page cache时，`page_cache_read`的逻辑是什么？`page_cache_read`的主要功能是于页缓存page cache中分配、插入和读取文件页

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/filemap.c#L2074
/*
file: 指向打开的文件结构体指针
offset: 文件中的页偏移（页为单位）
gfp_mask: 内存分配标志
*/
static int page_cache_read(struct file *file, pgoff_t offset, gfp_t gfp_mask)
{
	struct address_space *mapping = file->f_mapping;
	struct page *page;
	int ret;

	do {
		//使用__page_cache_alloc分配一个页面
		page = __page_cache_alloc(gfp_mask|__GFP_COLD);
		if (!page)
			return -ENOMEM;

		//1. 将页面插入页缓存的radix树
		//2. 同时将页面添加到LRU链表中
		ret = add_to_page_cache_lru(page, mapping, offset, gfp_mask & GFP_KERNEL);
		if (ret == 0){
			//情况1：插入成功 (ret == 0)时，调用文件系统的readpage方法读取文件数据，这是一个异步操作，启动磁盘I/O
			//页面在I/O完成前保持锁定状态
			ret = mapping->a_ops->readpage(file, page);
		}
		else if (ret == -EEXIST)	//竞态，其他线程已成功插入同一页面
			ret = 0; /* losing race to add is OK */

		put_page(page);

	} while (ret == AOP_TRUNCATED_PAGE);

	return ret;
}
```

2、`filemap_fault`中的性能优化


TODO

##  0x01   页面类型及共享原理

####  共享内存框架
![shm-arch]()

####  共享内存原理
![shm-principle]()

##  0x0  参考
-   [深入理解Linux内核共享内存机制- shmem&tmpfs](https://aijishu.com/a/1060000000451498)
-   [浅析Linux的共享内存与tmpfs文件系统](http://hustcat.github.io/shared-memory-tmpfs/)
-   [从内核世界透视 mmap 内存映射的本质（源码实现篇）](https://www.cnblogs.com/binlovetech/p/17754173.html)
-   [Linux 内核之 mmap 内存映射的原理及源码解析](https://zhuanlan.zhihu.com/p/709372792)
-	[从内核世界透视 mmap 内存映射的本质（原理篇）](https://www.cnblogs.com/binlovetech/p/17712761.html)