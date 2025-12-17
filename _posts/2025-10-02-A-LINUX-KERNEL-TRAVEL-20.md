---
layout:     post
title:  Linux 内核之旅（二十）：内核视角下的共享内存
subtitle:   mmap与shm在内核的实现分析与区别
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

本文主要基于[v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include)版本进行分析

####    tmpfs
tmpfs文件系统，其文件数据都在内存中，掉电会丢失，主要特点：

-	内存文件系统，所有的文件数据都在内存中，掉电丢失
-	数据在内存，数据访问速度很快
-	内存不足，回收到swap中
-	读的时候，不分配物理页面，读取的数据都是`0`

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

| 场景 | 说明 | 内核调用链（简） |
| :-----| :---- | :---- |
| 匿名（文件）共享映射 | 父子进程间通信 | `mmap_region->shmem_zero_setup` |
| ipc共享内存 | 任意进程间共享内存 | `newseg->shmem_kernel_file_setup->__shmem_file_setup` |
| tmpfs| 实现内存文件系统 | `shmem_file_operations.mmap->->shmem_mmap->vma->vm_ops = &shmem_vm_ops` |
| memfd | 创建共享匿名文件 | `memfd_create->shmem_file_setup` | 

####	共享内存页？
前文描述了匿名页和文件页，文件页会关联文件系统中的文件，而匿名页不关联任何文件。而共享内存页同时具备文件页和匿名页的的一些特征（如会关联文件、存在page cache等，同时也具备swap功能）

##  0x01    mmap的实现原理
`mmap`系统调用是将一个文件或者其它对象映射到进程的虚拟地址空间，实现磁盘地址和进程虚拟地址空间一段虚拟地址的一一对应关系。通过mmap系统调用可以让进程之间通过映射到同一个普通文件实现共享内存，普通文件被映射到进程虚拟地址空间当中后，进程可以像访问普通内存一样对文件进行一系列操作，而不需要通过 I/O 系统调用来读取或写入

`mmap`系统调用会将一个文件或其他对象映射到进程的地址空间中，并返回一个指向映射区域的指针，进程可以使用指针来访问映射区域的数据，就像访问内存一样

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

从`mmap`系统调用开始，核心参数如下：

-	`addr`：待映射的虚拟内存区域在进程虚拟内存空间中的起始地址（**虚拟内存地址**），通常设置成 `NULL`，意思就是完全交由内核来决定虚拟映射区的起始地址（要按照 PAGE_SIZE（`4K`） 对齐）
-	`length`：待申请映射的内存区域的大小，如果是匿名映射，则是要映射的匿名物理内存有多大，如果是文件映射，则是要映射的文件区域有多大（要按照 PAGE_SIZE（`4K`） 对齐）
-	`prot`：映射区域的保护模式，有 `PROT_READ`、`PROT_WRITE`、`PROT_EXEC`等
-	`flags`：标志位，可以控制映射区域的特性。常见的有 `MAP_SHARED` 和 `MAP_PRIVATE` 等
-	`fd`：文件描述符，用于指定映射的文件 
-	`offset`：映射的起始位置，表示被映射对象 (即文件) 从那里开始对映，通常设置为 `0`，该值应该为大小为PAGE_SIZE（`4K`）的整数倍

其中常用的`flags`取值：

-	`MAP_SHARED`：共享映射（用于多进程之间的通信），对映射区域的写入操作直接反映到文件当中
-	`MAP_PRIVATE`：私有映射，对映射区域的写入操作只反映到缓冲区当中不会写入到真正的文件
-	`MAP_ANONYMOUS`：匿名映射将虚拟地址映射到物理内存而不是文件（忽略fd、offset）

![mmap-result]()

####	虚拟内存地址与vma


![mmap-kernel-function-flow]()

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

	if (!(flags & MAP_ANONYMOUS)) {	// 预处理文件映射
		audit_mmap_fd(fd, flags);
		// 通过文件 fd 获取映射文件的 struct file 结构
		// 从而获取 inode 信息，关联磁盘文件，后面关闭 fd，仍然可以用 mmap 操作
		file = fget(fd);
		if (!file)
			return -EBADF;
		if (is_file_hugepages(file))
			len = ALIGN(len, huge_page_size(hstate_file(file)));
		retval = -EINVAL;
		if (unlikely(flags & MAP_HUGETLB && !is_file_hugepages(file)))
			goto out_fput;
	} else if (flags & MAP_HUGETLB) {
		//MAP_HUGETLB 只能支持 MAP_ANONYMOUS 匿名映射的方式使用 HugePage
		struct user_struct *user = NULL;
		struct hstate *hs;	// 内核中的大页池（预先创建）
		// 选取指定大页尺寸的大页池（内核中存在不同尺寸的大页池）
		hs = hstate_sizelog((flags >> MAP_HUGE_SHIFT) & SHM_HUGE_MASK);
		if (!hs)
			return -EINVAL;
		
		// 映射长度 len 必须与大页尺寸对齐
		len = ALIGN(len, huge_page_size(hs));
		/*
		 * VM_NORESERVE is used because the reservations will be
		 * taken when vm_ops->mmap() is called
		 * A dummy user value is used because we are not locking
		 * memory so no accounting is necessary
		 */
		// 在 hugetlbfs 中创建 anon_hugepage 文件，并预留大页内存（禁止其他进程申请）
		file = hugetlb_file_setup(HUGETLB_ANON_FILE, len,
				VM_NORESERVE,
				&user, HUGETLB_ANONHUGE_INODE,
				(flags >> MAP_HUGE_SHIFT) & MAP_HUGE_MASK);
		if (IS_ERR(file))
			return PTR_ERR(file);
	}

	flags &= ~(MAP_EXECUTABLE | MAP_DENYWRITE);
	//核心：开始内存映射
	retval = vm_mmap_pgoff(file, addr, len, prot, flags, pgoff);
out_fput:
	if (file)
		fput(file);
	return retval;
}
```

`vm_mmap_pgoff`函数的核心流程如下：

1.	获取进程虚拟内存空间 mm_struct，用于在开始 mmap 内存映射之前，对进程虚拟内存空间加写锁保护，防止多线程并发修改，映射完成后，再释放写锁。
2.	调用 do_mmap_pgoff 函数开始 mmap 内存映射，在进程虚拟内存空间中分配一段 vma，并建立相关映射关系。
3.	如果设置了 MAP_POPULATE 或者 MAP_LOCKED 属性，则调用 mm_populate 函数，提前为 [ret , ret + populate] 这段虚拟内存立即分配物理内存页面，后续访问不会发生缺页中断异常

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/util.c#L296
unsigned long vm_mmap_pgoff(struct file *file, unsigned long addr,
	unsigned long len, unsigned long prot,
	unsigned long flag, unsigned long pgoff)
{
	unsigned long ret;
	// 获取进程虚拟内存空间
	struct mm_struct *mm = current->mm;
	// 是否需要为映射的 vma，提前分配物理内存页，避免后续的缺页
	// 取决于 flag 是否设置了 MAP_POPULATE 或者 MAP_LOCKED
	// 这里的 populate 表示需要分配物理内存的大小
	unsigned long populate;
	// 初始化 userfaultfd 链表
	LIST_HEAD(uf);

	// security钩子
	ret = security_mmap_file(file, prot, flag);
	if (!ret) {
		// 对进程虚拟内存空间加写锁保护，防止多线程并发修改
		if (down_write_killable(&mm->mmap_sem))
			return -EINTR;
        // 开始 mmap 内存映射，在进程虚拟内存空间中分配一段 vma，并建立相关映射关系
        // 返回值 ret 为映射虚拟内存区域的起始地址
		ret = do_mmap_pgoff(file, addr, len, prot, flag, pgoff,
				    &populate, &uf);
		
		// 释放写锁
		up_write(&mm->mmap_sem);
		// 等待 userfaultfd 处理完成
		userfaultfd_unmap_complete(mm, &uf);
		if (populate)	// 提前分配物理内存页面，后续访问不会缺页，为 [ret , ret + populate] 这段虚拟内存立即分配物理内存
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

####    do_mmap：映射的核心实现
`do_mmap`核心功能如下：

1.	调用 `get_unmapped_area` 函数用于在进程地址空间中寻找出一段长度为 `len`，并且还未映射的虚拟内存区域 vma 出来，返回值 `addr` 表示这段虚拟内存区域的起始地址。之后根据不同的文件打开方式设置不同的 vm 标志位 `flag`
2.	调用 `mmap_region` 函数，首先会为刚才选取出来的映射虚拟内存区域分配 vma 结构，并根据映射信息进行初始化，以及建立 vma 与相关映射文件的关系，最后将这段 vma 插入到进程的虚拟内存空间中（链表或红黑树进行管理）

TODO
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

####	追踪get_unmapped_area：寻找VMA

这里先看下`get_unmapped_area`函数的实现，即如何寻找到合适长度的虚拟内存区域

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/mmap.c#L2053

unsigned long
get_unmapped_area(struct file *file, unsigned long addr, unsigned long len,
		unsigned long pgoff, unsigned long flags)
{
	// 在进程虚拟空间中寻找还未被映射的 VMA 这段核心逻辑是被内核实现在特定于体系结构的函数中
    // 该函数指针用于指向真正的 get_unmapped_area 函数，在经典布局下，真正的实现函数为 arch_get_unmapped_area
	unsigned long (*get_area)(struct file *, unsigned long,
				  unsigned long, unsigned long, unsigned long);

	unsigned long error = arch_mmap_check(addr, len, flags);
	if (error)
		return error;

	/* Careful about overflows.. */
	// 映射的虚拟内存区域长度不能超过进程的地址空间
	if (len > TASK_SIZE)
		return -ENOMEM;
	
	// 如果是匿名映射，则采用 mm_struct 中保存的特定于体系结构的 arch_get_unmapped_area 函数
	get_area = current->mm->get_unmapped_area;
	if (file) {
		// 如果是文件映射，则需要使用 file->f_op 中的 get_unmapped_area 指向的函数来为文件映射申请虚拟内存
		// file->f_op 保存的是特定于文件系统中文件的相关操作，如 ext4 文件系统下的 thp_get_unmapped_area 函数
		if (file->f_op->get_unmapped_area)
			get_area = file->f_op->get_unmapped_area;
	} else if (flags & MAP_SHARED) {
		/*
		 * mmap_region() will call shmem_zero_setup() to create a file,
		 * so use shmem's get_unmapped_area in case it can be huge.
		 * do_mmap_pgoff() will clear pgoff, so match alignment.
		 */
		pgoff = 0;
		// 共享匿名映射是通过在 tmpfs 中创建的匿名文件实现的，所以这里也有其专有的 get_unmapped_area 函数
		// 共享匿名映射的情况下 get_unmapped_area 指向 shmem_get_unmapped_area 函数
		get_area = shmem_get_unmapped_area;
	}

	// 在进程虚拟内存空间中，根据指定的 addr，len 查找合适的 vma
	addr = get_area(file, addr, len, pgoff, flags);
	if (IS_ERR_VALUE(addr))
		return addr;
	
	// vma 区域不能超过进程地址空间
	if (addr > TASK_SIZE - len)
		return -ENOMEM;
	// addr 需要与 page size 对齐
	if (offset_in_page(addr))
		return -EINVAL;

	error = security_mmap_addr(addr);
	return error ? error : addr;
}
```

文件页与内存页映射的函数调用如下：

![get_unmapped_area]()

arch_get_unmapped_area 函数的核心作用如下：

1.	调用 `find_vma` 函数，根据指定的映射起始地址 addr，在进程地址空间中查找出符合 addr < vma->vm_end 条件的第一个 vma，然后在进程地址空间 mm_struct 中 mmap 指向的 vma 链表中，找出它的前驱节点 pprev。
2.	如果明确指定起始地址 addr ，但是指定的虚拟内存范围有一段无效的区域或者已经存在映射关系，内核就不能按照指定的addr开始映射，此时调用vm_unmapped_area函数，内核会自动在文件映射与匿名映射区中按照地址的增长方向寻找一段len大小的虚拟内存范围出来。注意：此时找到的虚拟内存范围的起始地址就不是指定的addr

unmapped_area函数的核心任务就是在管理进程地址空间这些vma的红黑树mm_struct-> mm_rb中查找出一个满足条件的地址间隙gap用于内存映射。如果能够找到符合条件的地址间隙 gap 则直接返回，否者就从进程地址空间中最后一个 vma->vm_end 开始映射

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/mmap.c#L1966
unsigned long
arch_get_unmapped_area(struct file *filp, unsigned long addr,
		unsigned long len, unsigned long pgoff, unsigned long flags)
{
	struct mm_struct *mm = current->mm;
	struct vm_area_struct *vma;
	struct vm_unmapped_area_info info;

	// 进程虚拟内存空间的末尾 TASK_SIZE

	// 映射区域长度是否超过进程虚拟内存空间
	if (len > TASK_SIZE - mmap_min_addr)
		return -ENOMEM;
	
	// 如果指定了 MAP_FIXED 表示必须要从指定的 addr 开始映射 len 长度的区域
    // 如果这块区域已经存在映射关系，那么后续内核会把旧的映射关系覆盖掉
	if (flags & MAP_FIXED)
		return addr;
	
	// 没有指定 MAP_FIXED，但指定了 addr，内核从指定的 addr 地址开始映射，内核这里会检查指定的这块虚拟内存范围是否有效
	if (addr) {
		// addr 先保证与 page size 对齐
		addr = PAGE_ALIGN(addr);
		// 内核这里需要确认一下指定的 [addr, addr+len] 这段虚拟内存区域是否存在已有的映射关系
		// 若[addr, addr+len] 地址范围内已经存在映射关系，则不能按照指定的 addr 作为映射起始地址
		// 在进程地址空间中查找第一个符合 addr < vma->vm_end  条件的 vma
		// 如果不存在这样一个 vma（!vma）, 则表示 [addr, addr+len] 这段范围的虚拟内存是可以使用的，内核将会从指定的 addr 开始映射
        // 如果存在这样一个 vma ，则表示  [addr, addr+len] 这段范围的虚拟内存区域目前已经存在映射关系了，不能采用 addr 作为映射起始地址
        // 这里还有一种情况是 addr 落在 prev 和 vma 之间的一块未映射区域
        // 如果这块未映射区域的长度满足 len 大小，那么这段未映射区域可以被本次使用，内核也会从指定的 addr 开始映射
		vma = find_vma(mm, addr);
		if (TASK_SIZE - len >= addr && addr >= mmap_min_addr &&
		    (!vma || addr + len <= vma->vm_start))
			return addr;
	}
	// 如果明确指定 addr 但是指定的虚拟内存范围是一段无效的区域或者已经存在映射关系
    // 那么内核会自动在地址空间中寻找一段合适的虚拟内存范围出来，这段虚拟内存范围的起始地址就不是指定的 addr
	info.flags = 0;
	// vma 区域长度
	info.length = len;
	// 定义从哪里开始查找 vma, mmap_base 表示从文件映射与匿名映射区开始查找
	info.low_limit = mm->mmap_base;
	// 查找结束位置为进程地址空间的末尾 TASK_SIZE
	info.high_limit = TASK_SIZE;
	info.align_mask = 0;

	//见下
	return vm_unmapped_area(&info);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/mm/mmap.c#L2097
struct vm_area_struct *find_vma(struct mm_struct *mm, unsigned long addr)
{
	struct rb_node *rb_node;
	struct vm_area_struct *vma;

	/* Check the cache first. */
	// 进程地址空间中缓存了最近访问过的 vma，首先从进程地址空间中 vma 缓存中开始查找，缓存命中率通常大约为 35%
    // 查找条件为：vma->vm_start <= addr && vma->vm_end > addr
	vma = vmacache_find(mm, addr);
	if (likely(vma))
		return vma;
	
	// 进程地址空间中的所有 vma 被组织在一颗红黑树中，为了方便内核在进程地址空间中快速查找特定的 vma
    // 这里首先需要获取红黑树的根节点，内核会从根节点开始查找
	rb_node = mm->mm_rb.rb_node;

	while (rb_node) {
		struct vm_area_struct *tmp;
		// 获取位于根节点的 vma
		tmp = rb_entry(rb_node, struct vm_area_struct, vm_rb);
		
		if (tmp->vm_end > addr) {
			vma = tmp;
			// 判断 addr 是否恰好落在根节点 vma 中： vm_start <= addr < vm_end
			if (tmp->vm_start <= addr)
				break;
			rb_node = rb_node->rb_left;	// 如果不存在，则继续到左子树中查找
		} else{
			// 如果根节点的 vm_end <= addr，说明 addr 在根节点 vma 的后边，这种情况则到右子树中继续查找
			rb_node = rb_node->rb_right;
		}
	}

	if (vma)	// 更新 vma 缓存
		vmacache_update(addr, vma);
	// 返回查找到的 vma，如果没有查找到，则返回 null，表示进程空间中目前还没有这样一个 vma，后续需要新建
	return vma;
}

//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/mm.h#L2169
static inline unsigned long
vm_unmapped_area(struct vm_unmapped_area_info *info)
{	
	// 按照进程虚拟内存空间中文件映射与匿名映射区的地址增长方向分为两个函数，用来在进程地址空间中查找未映射的 vma
	if (info->flags & VM_UNMAPPED_AREA_TOPDOWN)
		// 当文件映射与匿名映射区的地址增长方向是从上到下逆向增长时（新式布局），采用 topdown 查找
		return unmapped_area_topdown(info);
	else
		// 地址增长方向为从下倒上正向增长（经典布局），采用该函数查找
		return unmapped_area(info);
}

unsigned long unmapped_area(struct vm_unmapped_area_info *info)
{
	/*
	 * We implement the search by looking for an rbtree node that
	 * immediately follows a suitable gap. That is,
	 * - gap_start = vma->vm_prev->vm_end <= info->high_limit - length;
	 * - gap_end   = vma->vm_start        >= info->low_limit  + length;
	 * - gap_end - gap_start >= length
	 */

	struct mm_struct *mm = current->mm;
	// 寻找未映射区域的参考 vma (该区域已存在映射关系)
	struct vm_area_struct *vma;
	// 未映射区域产生在 vma->vm_prev 与 vma 这两个虚拟内存区域中的间隙 gap 中，length 表示本次映射区域的长度
    // low_limit ，high_limit 表示在进程地址空间中哪段地址范围内查找，一个地址下限（mm->mmap_base），另一个标识地址上限（TASK_SIZE）
    // gap_start, gap_end 表示 vma->vm_prev 与 vma 之间的 gap 范围，unmapped_area 将会在这里产生
	unsigned long length, low_limit, high_limit, gap_start, gap_end;

	/* Adjust search length to account for worst case alignment overhead */
	// 调整搜索长度以考虑最坏情况下的对齐开销
	length = info->length + info->align_mask;
	if (length < info->length)
		return -ENOMEM;

	/* Adjust search limits by the desired length */
	// 根据需要的长度调整搜索限制
	if (info->high_limit < length)
		return -ENOMEM;

	// gap_start 需要满足的条件：gap_start =  vma->vm_prev->vm_end <= info->high_limit - length
    // 否则 unmapped_area 将会超出 high_limit 的限制
	high_limit = info->high_limit - length;

	if (info->low_limit > high_limit)
		return -ENOMEM;

	// gap_end 需要满足的条件：gap_end = vma->vm_start >= info->low_limit + length
    // 否则 unmapped_area 将会超出 low_limit 的限制
	low_limit = info->low_limit + length;

	/* Check if rbtree root looks promising */
	// 首先将 vma 红黑树的根节点作为 gap 的参考 vma，检查根节点是否符合
	if (RB_EMPTY_ROOT(&mm->mm_rb))
		goto check_highest;
	
	// 获取红黑树根节点的 vma
	vma = rb_entry(mm->mm_rb.rb_node, struct vm_area_struct, vm_rb);

	// rb_subtree_gap 为当前 vma 及其左右子树中所有 vma 与其对应 vm_prev 之间最大的虚拟内存地址 gap
    // 最大的 gap 如果都不能满足映射长度 length 则跳转到 check_highest 处理
	if (vma->rb_subtree_gap < length)
		goto check_highest;	// 从进程地址空间最后一个 vma->vm_end 地址处开始映射

	while (true) {
		/* Visit left subtree if it looks promising */
		// 左子树，获取当前 vma 的 vm_start 起始虚拟内存地址作为 gap_end
		gap_end = vma->vm_start;
		// gap_end 需要满足：gap_end >= low_limit，否则 unmapped_area 将会超出 low_limit 的限制
        // 如果存在左子树，则需要继续到左子树中去查找，因为需要按照地址从低到高的优先级来查看合适的未映射区域
		if (gap_end >= low_limit && vma->vm_rb.rb_left) {
			struct vm_area_struct *left =
				rb_entry(vma->vm_rb.rb_left,
					 struct vm_area_struct, vm_rb);
			// 如果左子树中存在合适的 gap，则继续左子树的查找
            // 否则查找结束，gap 为当前 vma 与其 vm_prev 之间的间隙
			if (left->rb_subtree_gap >= length) {
				vma = left;
				continue;
			}
		}

		// 获取当前 vma->vm_prev 的 vm_end 作为 gap_start
		gap_start = vma->vm_prev ? vma->vm_prev->vm_end : 0;
check_current:
		/* Check if current node has a suitable gap */
		// gap_start 需要满足：gap_start <= high_limit，否则 unmapped_area 将会超出 high_limit 的限制
		if (gap_start > high_limit)
			return -ENOMEM;
		if (gap_end >= low_limit && gap_end - gap_start >= length)
			goto found;	// 找到了合适的 unmapped_area 跳转到 found 处理

		/* Visit right subtree if it looks promising */
		// 当前 vma 与其左子树中的所有 vma 均不存在一个合理的 gap，那么从 vma 的右子树中继续查找
		if (vma->vm_rb.rb_right) {
			struct vm_area_struct *right =
				rb_entry(vma->vm_rb.rb_right,
					 struct vm_area_struct, vm_rb);
			if (right->rb_subtree_gap >= length) {
				vma = right;
				continue;
			}
		}

		/* Go back up the rbtree to find next candidate node */
		// 如果在当前 vma 以及它的左右子树中均无法找到一个合适的 gap
        // 那么这里会从当前 vma 节点向上回溯整颗红黑树，在它的父节点中尝试查找是否有合适的 gap
        // 因为这时候有可能会有新的 vma 插入到红黑树中，可能会产生新的 gap
		while (true) {
			struct rb_node *prev = &vma->vm_rb;
			if (!rb_parent(prev))
				goto check_highest;
			vma = rb_entry(rb_parent(prev),
				       struct vm_area_struct, vm_rb);
			if (prev == vma->vm_rb.rb_left) {
				gap_start = vma->vm_prev->vm_end;
				gap_end = vma->vm_start;
				goto check_current;
			}
		}
	}

check_highest:
	/* Check highest gap, which does not precede any rbtree node */
	// 流程走到这里表示在当前进程虚拟内存空间的所有 vma 中都无法找到一个合适的 gap 来作为 unmapped_area
    // 那么就从进程地址空间中最后一个 vma->vm_end 开始映射
    // mm->highest_vm_end 表示当前进程虚拟内存空间中，地址最高的一个 vma 的结束地址位置
	gap_start = mm->highest_vm_end;
	gap_end = ULONG_MAX;  /* Only for VM_BUG_ON below */
	if (gap_start > high_limit)	// 这里最后需要检查剩余虚拟内存空间是否满足映射长度
		return -ENOMEM;

found:
	/* We found a suitable gap. Clip it with the original low_limit. */
	// 流程走到这里表示已经找到了一个合适的 gap 来作为 unmapped_area，直接返回 gap_start（需要与 4K 对齐）作为映射的起始地址
	if (gap_start < info->low_limit)
		gap_start = info->low_limit;

	/* Adjust gap address to the desired alignment */
	// 调整间隙地址到所需的对齐方式
	gap_start += (info->align_offset - gap_start) & info->align_mask;

	VM_BUG_ON(gap_start + info->length > info->high_limit);
	VM_BUG_ON(gap_start + info->length > gap_end);
	return gap_start;	// 返回找到的地址间隙 gap 
}
```

####	do_mmap->mmap_region：创建虚拟内存区域
在上一节`get_unmapped_area`函数结束时，内核已在进程地址空间中找出一段地址范围为`[addr,addr + len]`的虚拟内存区域供mmap进行映射。接下来追踪下`mmap_region`及后续函数具体是如何初始化vma并建立映射关系的,`mmap_region`负责创建虚拟内存区域，其核心流程如下：

1.	调用 `may_expand_vm` 函数以检查进程在本次 mmap 映射之后申请的虚拟内存是否超过限制，检查（进程的虚拟内存总数+申请的页数）是否超过地址空间限制，如果是私有的可写映射，并且不是栈，则检查（进程的虚拟内存总数+申请的页数）是否超过最大数据长度
2.	调用 `find_vma_links` 函数查找当前进程地址空间中是否存在与指定映射区域 `[addr, addr+len]` 重叠的部分，如果有重叠则需调用 `do_munmap` 函数将这段重叠的映射部分解除掉，后续会重新映射这部分
3.	调用`vma_merge`函数，内核先尝试看能不能将待映射的vma和地址空间中已有的vma进行合并，如果可以合并，则不用创建新的vma结构，节省内存的开销。如果不能合并，则从 slab 中取出一个新的 vma 结构，并根据要映射的虚拟内存区域属性初始化 vma 结构中的相关字段
4.	调用 `vma_link` 函数把虚拟内存区域 vma 插入到链表和红黑树中。如果 vma 关联文件，那么把虚拟内存区域添加到文件的区间树中，文件的区间树用来跟踪文件被映射到哪些虚拟内存区域
5.	调用 `vma_set_page_prot` 函数更新地址空间 `mm_struct` 中的相关统计变量，根据虚拟内存标志（`vma->vm_flags`）计算页保护位（`vma->vm_page_prot`），如果共享的可写映射想要把页标记为只读，其目的是跟踪写事件，那么从页保护位删除可写位

![do_mmap-mmap_region-flow]()

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
本小节以ext4文件系统的mmap文件映射过程为例，分析缺页中断的处理过程。缺页中断的核心逻辑中，涉及到的主要内核结构及成员如下：

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
	/*
	参数：vmf->pgoff是单个页面的偏移，只返回这一个页面的指针
	函数返回值vmf->page也指向单个页面
	*/
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

	// 注意：这里仅设置单个页面
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

2、`filemap_fault`中的性能优化，虽然`filemap_fault`函数主要处理单page缺页，但会通过预读（`readahead`）机制连续加载多页。从其调用入口`page = find_get_page(mapping, offset)`来看，这里只查找指定偏移的一页，但通过预读机制实际上在优化连续的内存访问模式。即**预读后续页面，减少未来缺页**


##  0x0   shmem基础知识

####	页面类型
对shmem类型的页面而言，其既有匿名页的特点（如`page->flags`设置`PG_swapbacked`，具有swap功能），也有文件页的特点（如`inode->i_mapping->a_ops = &shmem_aops`关联文件inode，有page cache）

![shm-page-type](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-page-type.png)

####  共享内存框架
shmem的整体框架如下：

![shm-arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-arch.png)

####	LRU with shmem
内核中，用户态进程使用的物理页面会放入LRU链表中进行老化/回收，其中匿名页面会加入匿名LRU，而文件页会加入文件LRU。虽然shmem页面既有匿名页和文件页的特点，但是由于它有swap特性，它会加入到匿名页LRU

![shm-lru-type](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-lru-type.png)

##	0x0	共享内存原理（shmem）及操作梳理

####  共享内存原理
基于shmem的内存共享原理如下，可以看到和上文描述的mmap共享机制非常类似：

![shm-principle](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-principle.png)

以memfd为例，实现共享内存的步骤：

1.	通过`memfd`系统调用等方式创建文件描述符（fd），如本例子send进程会通过`memfd`系统调用来获得一个unused fd，并将fd关联文件实例（file），这个file就会关联一片共享内存
2.	将文件描述符传递给其他进程来实现共享，如可通过unix socket传递文件描述符，实际上传递文件描述符是在接收方申请一个unused fd，然后关联共享内存对应的`struct file`对象，如图所示send进程的文件描述符`fd=4`会关联共享内存对应的file，recv进程的文件描述符`fd=5`也会关联共享内存对应的file
3.	send/recv进程通过mmap映射共享内存到进程虚拟地址空间
4.	send进程首次写访问数据，此时会发生缺页异常，page cache查询不到物理页面PAGE1，会申请物理页面（`__page_cache_alloc`）并加入文件实例（`struct inode`）对应的page cache（`add_to_page_cache_lru`），并通过页表映射PAGE1到send进程的虚拟地址空间，缺页返回后，将数据写入PAGE1（这里都是`'a'`），注意，**这里写入不会切态，因为用户态的函数如`memcpy`可以直接操作本进程的虚拟内存地址（本质上是直接操作page）**
5.	recv进程首次读访问数据时，同样也会发生缺页异常，但是会首先查询page cache，发现PAGE1，然后通过页表映射物理页PAGE1到recv进程的虚拟地址空间，如此缺页返回后，从PAGE1读出数据（都是`'a'`），于是实现了内存共享

####	缺页中断的处理流程
缺页中断的内核调用链基本如下图：

![shm-page-fault-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-page-fault-flow.png)

缺页处理步骤框图如下：

![shm-page-fault-flow-arch.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-page-fault-flow-arch.png)

shmem共享内存页面的缺页处理步骤如下，当缺页发生时

1.	查找或分配物理页面
	-	先从page cache中查找，相关的物理页面可能已经被其他线程加入了page cache，所以首先从page cache查找   
	-	若找不到从swap cache中查找，页面有可能在回收等场景被加入了swap cache，所以在这里也查找下
	-	找不到如果之前有swap out 则swap in，之前如果由于内存回收等场景相关页面被swap out到swap device，那么相关的swap cache对应的位置会被替换为swap entry, 这个时候根据swap entry从swap device中读取物理页面内容
	-	否则分配新的folio：上面都尝试了查找但是没有找到，那么有可能是第一次访问这个页面，这个时候需要分配新的物理页面，既是folio
2.	新的page加入全局lru，shmem会被加入匿名的lru中，以便内存回收都场景回收到swap device
3.	页表映射，将相关的物理页面通过页表映射到进程的虚拟地址空间，这样后面进程就可以正常访问页面数据了

####	回收shmem页

![shm-recycle-kernel-function-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-kernel-function-flow.png)

shmem页面回收逻辑如下：

1、从页面从lru中隔离

![shm-recycle-flow-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-1.png)

2、申请页面的page lock

![shm-recycle-flow-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-2.png)

3、rmap反向映射查找解除这个页面的所有页表映射，如send/recv进程

![shm-recycle-flow-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-3.png)

4、分配swap entry,页面加入swap cache

![shm-recycle-flow-4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-4.png)

5、替换页面的page cache为swap entry

![shm-recycle-flow-5](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-5.png)

6、页面从swap cache中删除

![shm-recycle-flow-6](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-6.png)

7、页面内容写入swap device

![shm-recycle-flow-7](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-7.png)

8、释放页面的page lock

![shm-recycle-flow-8](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-8.png)

9、页面还给buddy

![shm-recycle-flow-9](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-recycle-flow-9.png)

这里需要注意一点的是：shmem页面回收时保存swap entry的方式跟匿名页完全不一样，匿名页在回收时，会将相应的swap entry替换为原来的页表项，而shmem页面会直接清掉原来的页表项，会将swap entry替换为对应的swap cache的位置

##	0x0	tmpfs的读写实现跟踪

####	基于tmpfs的读过程
主要涉及的内核调用链如下：

![shm-tmpfs-read-kernel-function-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-tmpfs-read-kernel-function-flow.png)


1.	按照page cache -> swap cache -> swap device顺序查找文件页面
2.	查找并拷贝页面内容到用户空间缓冲区
	-	如果找到，则拷贝文件页面数据到用户空间缓冲区
	-	如果没有找到，则直接往用户空间缓冲区拷贝`0`
3.	更新文件读写位置

这里需要注意的是：对于tmpfs文件系统中的文件的读操作来说，按照page cache -> swap cache -> swap device顺序如果查找不到页面，则不会分配新的页面，只会往用户空间缓冲区拷贝0（这有点类似匿名页的第一次读，一般会映射到`0`页），这种情况也说明了相关文件偏移的页面从来没有被人写访问过

####	基于tmpfs的写过程
主要涉及的内核调用链如下：

![shm-tmpfs-write-kernel-function-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/20/shm-tmpfs-write-kernel-function-flow.png)

写tmpfs文件的主要步骤如下：

1.	查找或分配文件页面，按照page cache -> swap cache -> swap device顺序查找，如果找到，继续下一步；如果没找到，则分配新的页面
2.	从用户空间缓冲区拷贝数据到文件页面
3.	标记页面为脏

##	0x0	System V共享内存实现分析

####	shmget：创建共享内存

```text
shmget() 系统调用
  -> newseg()          
    -> shmem_kernel_file_setup() 
        -> __shmem_file_setup()
            -> shmem_get_inode()  // 创建 inode
```

从调用链的实现跟踪，最终会生成一个inode（因此具有page cache的功能），该inode对应的`address_space_operations`如下：

```cpp
static const struct address_space_operations shmem_aops = {
	.writepage	= shmem_writepage,
	.set_page_dirty	= __set_page_dirty_no_writeback,
#ifdef CONFIG_TMPFS
	.write_begin	= shmem_write_begin,
	.write_end	= shmem_write_end,
#endif
#ifdef CONFIG_MIGRATION
	.migratepage	= migrate_page,
#endif
	.error_remove_page = generic_error_remove_page,
};
```

####	shmget：内核实现跟踪

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/ipc/shm.c#L657
SYSCALL_DEFINE3(shmget, key_t, key, size_t, size, int, shmflg)
{
	struct ipc_namespace *ns;
	static const struct ipc_ops shm_ops = {
		.getnew = newseg,	//newseg
		.associate = shm_security,
		.more_checks = shm_more_checks,
	};
	struct ipc_params shm_params;

	ns = current->nsproxy->ipc_ns;

	shm_params.key = key;
	shm_params.flg = shmflg;
	shm_params.u.size = size;

	return ipcget(ns, &shm_ids(ns), &shm_ops, &shm_params);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/ipc/shm.c#L522
static int newseg(struct ip c_namespace *ns, struct ipc_params *params)
{
	key_t key = params->key;
	int shmflg = params->flg;
	size_t size = params->u.size;
	int error;
	struct shmid_kernel *shp;
	size_t numpages = (size + PAGE_SIZE - 1) >> PAGE_SHIFT;
	struct file *file;
	char name[13];
	int id;
	vm_flags_t acctflag = 0;

	if (size < SHMMIN || size > ns->shm_ctlmax)
		return -EINVAL;

	if (numpages << PAGE_SHIFT < size)
		return -ENOSPC;

	if (ns->shm_tot + numpages < ns->shm_tot ||
			ns->shm_tot + numpages > ns->shm_ctlall)
		return -ENOSPC;

	shp = ipc_rcu_alloc(sizeof(*shp));
	if (!shp)
		return -ENOMEM;

	shp->shm_perm.key = key;
	shp->shm_perm.mode = (shmflg & S_IRWXUGO);
	shp->mlock_user = NULL;

	shp->shm_perm.security = NULL;
	error = security_shm_alloc(shp);
	if (error) {
		ipc_rcu_putref(shp, ipc_rcu_free);
		return error;
	}

	sprintf(name, "SYSV%08x", key);
	if (shmflg & SHM_HUGETLB) {
		......
	} else {
		/*
		 * Do not allow no accounting for OVERCOMMIT_NEVER, even
		 * if it's asked for.
		 */
		if  ((shmflg & SHM_NORESERVE) &&
				sysctl_overcommit_memory != OVERCOMMIT_NEVER)
			acctflag = VM_NORESERVE;
		file = shmem_kernel_file_setup(name, size, acctflag);
	}
	error = PTR_ERR(file);
	if (IS_ERR(file))
		goto no_file;

	shp->shm_cprid = task_tgid_vnr(current);
	shp->shm_lprid = 0;
	shp->shm_atim = shp->shm_dtim = 0;
	shp->shm_ctim = get_seconds();
	shp->shm_segsz = size;
	shp->shm_nattch = 0;
	shp->shm_file = file;
	shp->shm_creator = current;

	id = ipc_addid(&shm_ids(ns), &shp->shm_perm, ns->shm_ctlmni);
	if (id < 0) {
		error = id;
		goto no_id;
	}

	list_add(&shp->shm_clist, &current->sysvshm.shm_clist);

	/*
	 * shmid gets reported as "inode#" in /proc/pid/maps.
	 * proc-ps tools use this. Changing this will break them.
	 */
	file_inode(file)->i_ino = shp->shm_perm.id;

	ns->shm_tot += numpages;
	error = shp->shm_perm.id;

	ipc_unlock_object(&shp->shm_perm);
	rcu_read_unlock();
	return error;
	......
}

struct file *shmem_kernel_file_setup(const char *name, loff_t size, unsigned long flags)
{
	return __shmem_file_setup(name, size, flags, S_PRIVATE);
}

//https://elixir.bootlin.com/linux/v4.11.6/source/mm/shmem.c#L4133
static struct file *__shmem_file_setup(const char *name, loff_t size,
				       unsigned long flags, unsigned int i_flags)
{
	struct file *res;
	struct inode *inode;
	struct path path;
	struct super_block *sb;
	struct qstr this;

	if (IS_ERR(shm_mnt))
		return ERR_CAST(shm_mnt);

	if (size < 0 || size > MAX_LFS_FILESIZE)
		return ERR_PTR(-EINVAL);

	if (shmem_acct_size(flags, size))
		return ERR_PTR(-ENOMEM);

	res = ERR_PTR(-ENOMEM);
	this.name = name;
	this.len = strlen(name);
	this.hash = 0; /* will go */
	sb = shm_mnt->mnt_sb;
	path.mnt = mntget(shm_mnt);
	path.dentry = d_alloc_pseudo(sb, &this);
	if (!path.dentry)
		goto put_memory;
	d_set_d_op(path.dentry, &anon_ops);

	res = ERR_PTR(-ENOSPC);
	// shmem_get_inode：核心
	inode = shmem_get_inode(sb, NULL, S_IFREG | S_IRWXUGO, 0, flags);
	if (!inode)
		goto put_memory;

	inode->i_flags |= i_flags;
	d_instantiate(path.dentry, inode);
	inode->i_size = size;
	clear_nlink(inode);	/* It is unlinked */
	res = ERR_PTR(ramfs_nommu_expand_for_mapping(inode, size));
	if (IS_ERR(res))
		goto put_path;

	res = alloc_file(&path, FMODE_WRITE | FMODE_READ,
		  &shmem_file_operations);
	if (IS_ERR(res))
		goto put_path;

	return res;

put_memory:
	shmem_unacct_size(flags, size);
put_path:
	path_put(&path);
	return res;
}
```

`shmem_get_inode`：

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/mm/shmem.c#L2135
static struct inode *shmem_get_inode(struct super_block *sb, const struct inode *dir,
				     umode_t mode, dev_t dev, unsigned long flags)
{
	struct inode *inode;
	struct shmem_inode_info *info;
	struct shmem_sb_info *sbinfo = SHMEM_SB(sb);

	if (shmem_reserve_inode(sb))
		return NULL;

	inode = new_inode(sb);
	if (inode) {
		inode->i_ino = get_next_ino();
		inode_init_owner(inode, dir, mode);
		inode->i_blocks = 0;
		inode->i_atime = inode->i_mtime = inode->i_ctime = current_time(inode);
		inode->i_generation = get_seconds();
		info = SHMEM_I(inode);
		memset(info, 0, (char *)inode - (char *)info);
		spin_lock_init(&info->lock);
		info->seals = F_SEAL_SEAL;
		info->flags = flags & VM_NORESERVE;
		INIT_LIST_HEAD(&info->shrinklist);
		INIT_LIST_HEAD(&info->swaplist);
		simple_xattrs_init(&info->xattrs);
		cache_no_acl(inode);

		switch (mode & S_IFMT) {
		default:
			inode->i_op = &shmem_special_inode_operations;
			init_special_inode(inode, mode, dev);
			break;
		case S_IFREG:
			inode->i_mapping->a_ops = &shmem_aops;	//核心：关联shmem_aops
			inode->i_op = &shmem_inode_operations;
			inode->i_fop = &shmem_file_operations;
			mpol_shared_policy_init(&info->policy,
						 shmem_get_sbmpol(sbinfo));
			break;
		case S_IFDIR:
			inc_nlink(inode);
			/* Some things misbehave if size == 0 on a directory */
			inode->i_size = 2 * BOGO_DIRENT_SIZE;
			inode->i_op = &shmem_dir_inode_operations;
			inode->i_fop = &simple_dir_operations;
			break;
		case S_IFLNK:
			/*
			 * Must not load anything in the rbtree,
			 * mpol_free_shared_policy will not be called.
			 */
			mpol_shared_policy_init(&info->policy, NULL);
			break;
		}
	} else
		shmem_free_inode(sb);
	return inode;
}
```

##	0x0	总结


##  0x0  参考
-   [深入理解Linux内核共享内存机制- shmem&tmpfs](https://aijishu.com/a/1060000000451498)
-   [浅析Linux的共享内存与tmpfs文件系统](http://hustcat.github.io/shared-memory-tmpfs/)
-   [从内核世界透视 mmap 内存映射的本质（源码实现篇）](https://www.cnblogs.com/binlovetech/p/17754173.html)
-   [Linux 内核之 mmap 内存映射的原理及源码解析](https://zhuanlan.zhihu.com/p/709372792)
-	[从内核世界透视 mmap 内存映射的本质（原理篇）](https://www.cnblogs.com/binlovetech/p/17712761.html)