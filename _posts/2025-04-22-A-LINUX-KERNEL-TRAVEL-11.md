---
layout:     post
title:  Linux 内核之旅（十一）：追踪 open 系统调用
subtitle:   VFS
date:       2025-04-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

用户进程在能够读 / 写一个文件之前必须要先 open 这个文件。对文件的读 / 写从概念上说是一种进程与文件系统之间的一种有连接通信，所谓打开文件实质上就是在进程与文件之间建立起链接。在文件系统的处理中，每当一个进程重复打开同一个文件时就建立起一个由 `struct file` 结构代表的独立的上下文。通常一个 `file` 结构，即一个读 / 写文件的上下文，都由一个打开文件号（fd）加以标识。从 VFS 的层面来看，open 操作的实质就是根据参数指定的路径去获取一个该文件系统（比如 `ext4`）的 inode（硬盘上），然后触发 VFS 一系列机制（如生成 dentry、加载 inode 到内存 inode 以及将 dentry 指向内存 inode 等等），然后去填充 VFS 层的 `struct file` 结构体，这样就可以让上层使用了

用户态程序调用 `open` 函数时，会产生一个中断号为 `5` 的中断请求，其值以该宏 `__NR__open` 进行标示，而后该进程上下文将会被切换到内核空间，待内核相关操作完成后，就会从内核返回至用户态，此时还需要一次进程上下文切换，本文就以内核视角追踪下 `open` 的内核调用过程

##  0x01   ftrace open 系统调用
内核调用链如下（简化）

```BASH
 1) open1-3789488  |               |    __x64_sys_openat() {
 1) open1-3789488  |               |      do_sys_openat2() {
 1) open1-3789488  |               |        do_filp_open() {
 1) open1-3789488  |               |          path_openat() {
 1) open1-3789488  |               |              lookup_open.isra.0() {
 1) open1-3789488  |               |            do_open() {
 1) open1-3789488  |               |              may_open() {
 1) open1-3789488  |               |              vfs_open() {
 1) open1-3789488  |               |                do_dentry_open() {
 1) open1-3789488  |               |                  security_file_open() {
 1) open1-3789488  |               |                    selinux_file_open() {
 1) open1-3789488  |               |                    bpf_lsm_file_open() {
 1) open1-3789488  |               |                  ext4_file_open() {
 1) open1-3789488  |               |                    dquot_file_open() {
 1) open1-3789488  |   0.172 us    |                      generic_file_open();
```

完整的函数调用链 [可见]()

##  0x02    再看文件系统缓存

####    VFS的hash表：dentry cache
一个 `struct dentry` 结构代表文件系统中的一个目录或文件，VFS dentry 结构的意义，即需要建立文件名 filename 到 inode 的映射关系，目的为了提高系统调用在操作、查找目录/文件操作场景下的效率，且 dentry 是仅存在于内存的数据结构，所以内核使用了 `dentry_hashtable`（dentry 树）来管理整个系统的目录树结构。在 Linux 可通过下面方式查看 dentry cache 的指标：

```BASH
[root@VM-X-X-tencentos ~]# cat /proc/sys/fs/dentry-state
1212279 1174785 45      0       598756  0
```

前文已经描述过 [`dentry_hashtable`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L105) 数据结构的意义，即为了提高目录查找效率，内核使用 `dentry_hashtable` 对 dentry 进行管理，在 `open` 等系统调用进行路径查找时，用于快速定位某个目录 dentry 下面的子 dentry（哈希表的搜索效率显然更好）

`dentry_hashtable`用来保存对应一个dentry的`hlist_bl_node`，使用拉链法来解决哈希冲突。它的好处是能根据路径分量（component）名称的hash值，快速定位到对应的`hlist_bl_node`，最后通过`hlist_bl_node`（`container_of`）获得对应的dentry


1、`dentry` 及 `dentry_hashtable` 的结构（搜索 & 回收）

对 `dentry` 而言，对于加速搜索关联了两个关键数据结构 `d_hash` 及 `d_lru`，dentry 回收关联的重要成员是 `d_lockref`，`d_lockref` 内嵌一个自旋锁（`spinlock_t`），用于保护 dentry 结构的并发修改
```CPP
struct dentry {
    /* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */

    struct hlist_bl_node d_hash;       //hashtable 的节点
    strcut list_head d_lru;     //LRU 链上的节点
}
```

![dcache-pic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/review/dentry-hashtable.png)

`dentry_hashtable`是一个指向`hlist_bl_head`的指针的数组。通过name的hash值，调用[`d_hash()`](1https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L107)即可获取`dentry_hashtable`中对应的`hlist_bl_head`（即拉链的头）

```CPP
//dentry hashtable定义
static struct hlist_bl_head *dentry_hashtable __read_mostly;

// hlist_bl_head的头
struct hlist_bl_head {
	struct hlist_bl_node *first;
};

// 链表的节点（双向链表）
struct hlist_bl_node {
	struct hlist_bl_node *next, **pprev;
};

static inline struct hlist_bl_head *d_hash(unsigned int hash)
{
	return dentry_hashtable + (hash >> (32 - d_hash_shift));
}
```

如上，结构体`hlist_bl_node`类似一个双向链表，采用的是头插法（`hlist_bl_add_head()`）,前文[]()已经介绍过此类内核链表实现。`pprev`这个二级指针，指向的不是前面节点，而是指向前面节点的next指针的存储地址（如果是头节点，`pprev`会指向`hlist_bl_head`的`first`的存储地址）。这样的好处是从链表中移除自身的时候，只需要将`next->pprev = pprev`即可,关联内核函数为`__hlist_bl_del()`，对`pprev`解引用操作即`(*pprev)`这个值为当前链表节点`hlist_bl_node`的地址，下面介绍关于`dentry_hashtable`的几类典型操作：

-	判断节点是否在链表中，只需要判断`pprev`是否为`NULL`（`hlist_bl_unhashed()`）
-	`dentry_hashtable`的初始化：通过`alloc_large_system_hash()`分配一个容量是2的整数次幂的内存块，即一个指针数组
-	`hlist_bl_node`中并没有dentry相关信息，那么查询`dentry_hashtable`，如何通过`hlist_bl_node`指针获取对应的dentry呢？dentry里有一个对象（非指针成员）`struct hlist_bl_node d_hash`，所以可以用`hlist_bl_node`指针，通过`container_of()`机制获得对应的dentry
-	链表遍历：`hlist_bl_for_each_entry_rcu()`，链表里面包含了冲突节点
-	如何区分是否是目标dentry? `dentry_hashtable`是采用拉链法解决冲突的。由于是根据路径分量名称+父parent dentry指针地址来做hash的，所以如何找到需要的节点？
	-	当hash相同，路径分量名称不同，比较一下路径分量名称的字符串
	-	当hash相同，路径分量名称相同，通过`hlist_bl_node`获取对应的dentry，判断一下`dentry->d_parent`是否是它的`parent`即可。因为同一目录下不可能有两个同名的文件，所以名称相同的，parent肯定不同

2、`dentry_hashtable` 的创建过程

在文件系统初始化时，调用 `vfs_caches_init->dcache_init` 为 dcache 进行初始化，先创建一个 dentry 的 slab，用于后续 dentry 对象的分配，同时还初始化了 `dentry_hashtable` 这个用于管理 dentry 的全局 hashtable

```CPP
static struct hlist_head *C __read_mostly;

static void __init dcache_init(void)
{
    dentry_cache = KMEM_CACHE_USERCOPY(dentry,
        SLAB_RECLAIM_ACCOUNT|SLAB_PANIC|SLAB_MEM_SPREAD|SLAB_ACCOUNT,
        d_iname);

    /* Hash may have been set up in dcache_init_early */
    if (!hashdist)
        return;

    //dentry_hashtable 初始化
    dentry_hashtable =
        alloc_large_system_hash("Dentry cache",
                    sizeof(struct hlist_bl_head),
                    dhash_entries,			//	bucket size
                    13,
                    HASH_ZERO,
                    &d_hash_shift,
                    NULL,
                    0,
                    0);
    d_hash_shift = 32 - d_hash_shift;
}
```

3、基于`dentry-hashtable`的 RCU-walk 和 REF-walk 查找机制

RCU-walk 和 REF-walk 是路径查找（Path Lookup）中两种核心的并发控制机制，分别针对高频读取和写操作场景优化，简单介绍下此二种机制：

一、RCU-walk：无锁路径查找，设计目标是在无写入干扰的场景下，避免所有内存写操作（无锁、无引用计数增减），通过 RCU（Read-Copy-Update）和序列锁（Seqlock）保证数据一致性，相关实现原理如下：

1.	无锁遍历
	-	使用 `hlist_bl_for_each_entry_rcu()` 遍历 dentry 哈希链表（bucket），不获取任何锁
	-	依赖 RCU 宽限期（Grace Period）确保被释放的 dentry 不会被访问
2.	seqlock验证一致性
	-	通过 `raw_seqcount_begin()` 读取 dentry 的序列号 `d_seq`
	-	关键操作后调用 `read_seqcount_retry()` 检测序列号是否变化
3.	快速回退机制，若遇到以下情况，立即切换到 REF-walk：
	-	Dentries 不在缓存中（如冷数据）
	-	检测到并发修改（序列号变化频繁）
	-	路径包含符号链接或挂载点等

```CPP
// seqlock 验证模式
seq = raw_seqcount_begin(&dentry->d_seq); 
// 读取 dentry 字段
if (read_seqcount_retry(&dentry->d_seq, seq)) 
    goto retry; // 序列号变化则重试
```

二、REF-walk：基于引用计数的安全路径查找，设计目标是处理高频写入或复杂路径（如符号链接、权限校验），通过引用计数和锁保证强一致性，相关实现原理如下：

1.	引用计数保护
	-	对每个 dentry 和 vfsmount 调用 `dget()/mntget()` 增加引用计数，防止并发释放
	-	退出时通过 `dput()/mntput()` 减少计数
2.	锁机制同步
	-	自旋锁：保护 dentry 哈希桶（`dentry->d_lock`）
	-	读写信号量：处理 inode 数据更新（`inode->i_rwsem`）
3.	安全处理复杂路径
	-	解析符号链接时递归调用 `link_path_walk()`
	-	挂载点检查：通过 `__lookup_mnt()` 验证 vfsmount 有效性


##  0x03   基础知识
一个进程需要读/写一个文件，必须先通过 filename 建立和文件 inode 之间的通道，方式是通过 `open()` 函数，该函数的参数是文件所在的路径名 `pathname`，如何根据 `pathname` 找到对应的 inode？这就要依靠 dentry 结构了

####	系统调用

```CPP
int open (const char *pathname, int flags, mode_t mode);
int openat(int dirfd, const char *pathname, int flags, mode_t mode);
```

-	参数 `pathname`：文件路径，可以是相对路径或绝对路径（以`/` 开头）
-	参数 `dirfd`： 是打开一个目录后得到的文件描述符，作为相对路径的基准目录。如果文件路径是相对路径，那么在函数 `openat` 中解释为相对文件描述符 `dirfd` 引用的目录，`open` 函数中解释为相对调用进程的当前工作目录。如果文件路径是绝对路径, `openat` 会忽略参数 `dirfd`
-	参数 `flags`：必须包含一种访问模式: `O_RDONLY/O_ WRONLY/O_RDWR`，`flags` 可以包含多个文件创建标志和文件状态标志，区别是文件创建标志只影响打开操作, 文件状态标志影响后面的读写操作

####	文件创建标志
-	`O_CLOEXEC`：开启 close-on-exc标志，使用系统调用 `execve()` 装载程序的时候关闭文件
-	`CREAT`：如果文件不存在，创建文件
-	`ODIRECTORY`：参数 `pathname` 必须是一个日录
-	`EXCL`：通常和标志位 `CREAT` 联合使用，用来创建文件。如果文件已经存在，那么 `open()` 失败,返回错误号 `EEXIST`
-	`NOFOLLOW`：不允许参数 `pathname` 是符号链接（最后一个分量不能是符号链接，其他分量可以是符号链接）。如果参数 `pathname` 是符号链接，那么打开失败，返回错误号 `ELOOP`
-	`O_TMPFILE`：创建没有名字的临时普通文件，参数 `pathname` 指定目录关闭文件的时候，自动删除文件
-	`O_TRUNC`：如果文件已经存在，是普通文件并且访问模式允许写，那么把文件截断到长度为`0`

####	文件状态标志
-	`APPEND`：使用追加模式打开文件，每次调用 `write` 写文件的时候写到文件的末尾
-	`O_ASYNC`：启用信号驱动的输入输出，当输入或输出可用的时候,发送信号通知进程，默认的信号是 `SIGIO`
-	`O_DIRECT`：直接读写存储设备，不使用内核的页缓存。虽然会降低读写速度，但是在某些情况下有用处，例如应用程序使用自己的缓冲区，不需要使用内核的页缓存文件
-	`DSYNC`：调用 `write` 写文件时，把数据和检索数据所需要的元数据写回到存储设备
-	`LARGEFILE`：允许打开长度超过 `4GB` 的大文件（`64` 位内核会强制设置）
-	`NOATIME`：调用 `read` 读文件时，不要更新文件的访问时间
-	`O_NONBLOCK`：使用非阻塞模式打开文件， `open` 和以后的操作不会导致调用进程阻塞
-	`PATH`：获得文件描述符有两个用处，指示在目录树中的位置以及执行文件描述符层次的操作。不会真正打开文件，不能执行读操作和写操作
-	`O_SYNC`：调用 `write` 写文件时，把数据和相关的元数据写回到存储设备

####	mode参数
`mode` 指定创建新文件时的文件模式，当参数 `flags` 指定标志位 `O_CREAT` 或 `O_TMPFILE` 的时候，必须指定参数 `mode`，其他情况下忽略参数 `mode`，组合如下：

-	`S_IRWXU`：`0700`，用户（文件拥有者）有读、写和执行权限
-	`S_IRUSR`：`00400`，用户有读权限
-	`S_IWUSR`：`00200`，用户有写权限
-	`S_IXUSR`：`00100`，用户有执行权限
-	`S_IRWXG`：`00070`，文件拥有者所在组的其他用户有读、写和执行权限
-	`S_IRGRP`：`00040`，文件拥有者所在组的其他用户有读权限
-	`S_IWGRP`：`00020`，文件拥有者所在组的其他用户有写权限
-	`S_IXGRP`：`0010`，文件拥有者所在组的其他用户有执行权限
-	`S_IRWXO`：`0007`，其他组的用户有读、写和执行权限
-	`S_IROTH`：`0004`，其他组的用户有读权限
-	`S_IWOTH`：`00002`，其他组的用户有写权限
-	`S_IXOTH`：`00001`，其他组的用户有执行权限

####	AT_FDCWD
`AT_FDCWD`表明当 filename 为相对路径的情况下，将当前进程的工作目录设置为起始路径

####	flags && mode 解析

-	`flags`：控制打开一个文件
-	`mode`：新建文件的权限

`build_open_flags`函数将`open`的参数转换为内核结构`open_flags`：

```CPP
struct open_flags {
        int open_flag;
        umode_t mode;
        int acc_mode;
        int intent;
        int lookup_flags;
};
```

```CPP
static inline int build_open_flags(int flags, umode_t mode, struct open_flags *op)
{
	int lookup_flags = 0;
    //O_CREAT 或者 `__O_TMPFILE*` 设置了，acc_mode 才有效。
	int acc_mode;

	// Clear out all open flags we don't know about so that we don't report
	// them in fcntl(F_GETFD) or similar interfaces.
	// 只保留当前内核支持且已被设置的标志，防止用户空间乱设置不支持的标志
	flags &= VALID_OPEN_FLAGS;

	if (flags & (O_CREAT | __ O_TMPFILE))
		op->mode = (mode & S_IALLUGO) | S_IFREG;
	else
        //如果 O_CREAT | __ O_TMPFILE 标志都没有设置，那么忽略 mode
		op->mode = 0;

	// Must never be set by userspace
	flags &= ~FMODE_NONOTIFY & ~O_CLOEXEC;


	// O_SYNC is implemented as __ O_SYNC|O_DSYNC.  As many places only
	// check for O_DSYNC if the need any syncing at all we enforce it's
	// always set instead of having to deal with possibly weird behaviour
	// for malicious applications setting only __ O_SYNC.
	if (flags & __ O_SYNC)
		flags |= O_DSYNC;

	//如果是创建一个没有名字的临时文件，参数 pathname 用来表示一个目录，
	//会在该目录的文件系统中创建一个没有名字的 iNode
	if (flags & __ O_TMPFILE) {
		if ((flags & O_TMPFILE_MASK) != O_TMPFILE)
			return -EINVAL;
		acc_mode = MAY_OPEN | ACC_MODE(flags);
		if (!(acc_mode & MAY_WRITE))
			return -EINVAL;
	} else if (flags & O_PATH) {

		// If we have O_PATH in the open flag. Then we
		// cannot have anything other than the below set of flags
		// 如果设置了 O_PATH 标志，那么 flags 只能设置以下 3 个标志
		flags &= O_DIRECTORY | O_NOFOLLOW | O_PATH;
		acc_mode = 0;
	} else {
		acc_mode = MAY_OPEN | ACC_MODE(flags);
	}

	op->open_flag = flags;

    // O_TRUNC implies we need access checks for write permissions
    // 如果设置了，那么写之前可能需要清空内容
	if (flags & O_TRUNC)
		acc_mode |= MAY_WRITE;

	// Allow the LSM permission hook to distinguish append
	// access from general write access.
    // 让 LSM 有能力区分 追加访问和普通访问
	if (flags & O_APPEND)
		acc_mode |= MAY_APPEND;

	op->acc_mode = acc_mode;

    //设置意图，如果没有设置 O_PATH，表示此次调用有打开文件的意图
	op->intent = flags & O_PATH ? 0 : LOOKUP_OPEN;

	if (flags & O_CREAT) {
        //是否有创建文件的意图
		op->intent |= LOOKUP_CREATE;
		if (flags & O_EXCL)
			op->intent |= LOOKUP_EXCL;
	}
        //判断查找的目标是否是目录
	if (flags & O_DIRECTORY)
		lookup_flags |= LOOKUP_DIRECTORY;
        //判断当发现符号链接时是否继续跟下去
	if (!(flags & O_NOFOLLOW))
		lookup_flags |= LOOKUP_FOLLOW; //查找标志设置了 LOOKUP_FOLLOW 表示会继续跟下去

    //设置查找标志，lookup_flags 在路径查找时会用到
	op->lookup_flags = lookup_flags;
	return 0;
}
```

####	chroot
`chroot` 改变当前进程所在的内核空间 `current->root` 的全局变量， 之后该进程所有的文件系统操作的路径，都以新的 path 作为根目录。关联`open*`系统调用中的`set_fs_root`函数的处理过程

####   nameidata/path/vfsmount 结构
[`nameidata`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L506)，`nameidata` **用来存储遍历路径的中间结果**（临时性存放），在路径搜索时常用到。这个结构体在路径查找中非常重要，它记录了查找信息、保存了查找起始路径。在路径 `/a/b/c/d` 的每一个分量的查找中，它会保存当前的结果。对于一般路径名查找，在查找结束时，它会包含查询结果的信息；对于父路径名查找，在查找结束时，它会包含最后一个分量所在目录的信息。最重要的成员是 `nameidata.path`（记住在 VFS 中只有 `path` 才能唯一标识一个路径）

如`open()`、`mkdir()`、`rename()`等系统调用，可以使用文件的路径名作为参数，VFS将解析路径名并把它拆分成一个文件序列（分量），除了最后一个文件之外，所有的文件都必须是目录。为了识别目录文件，VFS将沿着路径逐层查找，并且使用`nameidata`边查找边缓存

-	`last`：存储需要解析的文件路径的分量，不仅包字符串,还包含长度和散列值
-	`path`：存储解析得到的挂载描述符和目录项
-	`inode`：存储目录项对应的索引节点（`path.dentry.d_inode`）

1、`path` 会保存已经成功解析到的信息，`last` 用来存放当前需要解析的信息，如果 `last` 解析成功那么就会更新 `path`

2、如果文件路径的分量是一个符号链接，那么接下来需要解析符号链接的目标，那么`stack`用来保存文件路径还未被解析的部分，`depth` 表示深度。假设目录`b`是符号链接（目标是 `e/f`），解析文件路径 `a/b/c/d`，那么解析到 `b`，发现 `b` 是符号链接，接下来要解析符号链接 `b` 的目标 `e/f`，需要把文件路径中没有解析的部分 `c/d` 保存到`stack`中，等解析完符号链接后继续解析

3、`nameidata`中`inode`与`link_inode`成员的作用是什么？

TODO

```CPP
struct nameidata {
	struct path	path;   //path 保存当前搜索到的路径（包含了 vfsmount 及在该 mount 下的 dentry）
	struct qstr	last;   //last 保存当前子路径名及其散列值
	struct path	root;   //root 用来保存根目录的path信息
	struct inode	*inode; // path.dentry.d_inode
    // inode 指向当前找到的目录项的 inode 结构

	unsigned int	flags;  //flags 是一些和查找（lookup）相关的标志位
    unsigned	seq, m_seq;
	int		last_type;  //last_type 表示当前节点类型
	unsigned	depth;  // depth 用来记录在解析符号链接过程中的递归深度
	int		total_link_count;
    
	struct saved {
		struct path link;
		struct delayed_call done;
		const char *name;
		unsigned seq;
	} *stack, internal[EMBEDDED_LEVELS];

	struct filename	*name;
	struct nameidata *saved;    // 保存上一个 nameidata 的指针
	struct inode	*link_inode;
	unsigned	root_seq;
	int		dfd;
};

//set_nameidata
static void set_nameidata(struct nameidata *p, int dfd, struct filename *name)
{
	struct nameidata *old = current->nameidata;
	p->stack = p->internal;
	p->dfd = dfd;
	p->name = name;
	p->total_link_count = old ? old->total_link_count : 0;
	p->saved = old;
}
```

![nameidata-path-vfsmount]()

`last_type`的五种类型：

-	`LAST_NORM`：最后一个分量是普通文件名
-	`LAST_ROOT`：最后一个分量是`/`（即整个路径名为`/`）
-	`LAST_DOT`：最后一个分量是`.`
-	`LAST_DOTDOT`：最后一个分量是`..`
-	`LAST_BIND`：最后一个分量是链接到特殊文件系统的符号链接

####	VFS中的目录查找

####	VFS mount：review
-	`struct path`是一个`<vfsmount,dentry>`二元组，`path`中的`vfsmount`记录的是当前所在文件系统的根目录信息，而`dentry`是当前路径行走所在的分量


VFS mount机制中的hashtable

TODO

####	VFS 的 path walk：简单介绍
VFS的path walk是`open()`的内核调用的核心设计，这里以访问`/dev/binderfs/binder`为例，`/`是根文件系统`rootfs`的根目录，`dev`是根文件系统的根目录下的一个普通目录，非挂载点，`binderfs`根文件系统下的位于`/dev`里的目录，同时也是[Binder文件系统](https://source.android.com/docs/core/architecture/ipc/binder-overview?hl=zh-cn)的挂载点，`binder`是Binder文件系统的挂载点下的一个代表binder设备的文件，下面是vfs的path walk过程：

1.	walk path起点是在根文件系统`rootfs`的根目录。所以一开始，`path1`记录的是根文件系统的`vfsmount1`和代表根文件系统根目录`/`的`dentry1`
2.	进入到`dev`分量时，`path2`记录的dentry将是代表路径分量`dev`的dentry，由于此时分量仍属于根文件系统，所以记录的`vfsmount`仍是`vfsmount1`
3.	当进入到`binderfs`时，`path3`记录的dentry将是代表分量`binderfs`的dentry，由于还是在根文件系统，所以记录的`vfsmount`仍是`vfsmount1`。但是检查dentry时，发现其`DCACHE_MOUNTED`的flag，即表示它是一个挂载点。这时就要利用`path`里的信息，即父`mount1`的`vfsmount1`和挂载点的dentry，从`mount_hashtable`获得Binder文件系统的`mount2`（关联内核函数是`__lookup_mnt()`），最终获得`vfsmount2`里的`dentry4`，成功切换到Binder文件系统的根目录（Binder文件系统的根目录在访问的路径上是看不出来的，因为VFS屏蔽了用户的感知）
4.	在整个VFS的路径行走中，需要先进入根文件系统，最后再进入Binder文件系统，注意 `dentry3`和`dentry4`是有区别的，`dentry3`是由根文件系统创建，而`dentry4`是由Binder文件系统，在挂载的时候创建的

![binder_vfs_open_flow_short()

####	RCU lock
RCU（Read-copy_update）是一种数据同步机制，允许读写同时进行。读操作不存在睡眠、阻塞、轮询，不会形成死锁，相比读写锁效率更高。写操作时先拷贝一个副本，在副本上进行修改、发布，并在合适时间释放原来的旧数据

##	0x04	内核实现走读：open

####	open的内核调用链（主要）

```BASH
SYSCALL_DEFINE4(openat, int, dfd,...)
  |- do_sys_open()
    |- do_sys_openat2()
      |- do_filp_open() //初始化结构体nameidata的实例，并将open()相关参数保存在nameidata的实例中
        |- path_openat()
          |- path_init() //获取根文件系统的vfsmount、根目录的dentry，并保存在nameidata中
          |- link_path_walk() //路径行走：逐个解析文件路径分量（除了最后一个分量），获取对应的dentry、inode
            |- walk_component()
			  |- lookup_fast()
				|- __d_lookup_rcu() //实现了rcu-walk
				  |- d_hash() //根据hash值，从dentry_hashtable中获取对应的hlist_bl_head
				  |- hlist_bl_for_each_entry_rcu() //遍历链表hlist_bl_head，寻找对应的dentry
				|- __d_lookup() //实现了ref-walk
			  |- lookup_slow() //在lookup_fast()中没有找到dentry，会获取父dentry对应的inode，通过inode->i_op->lookup去查找、创建
			    |- __lookup_slow()
				  |- d_alloc_parallel() //创建一个新的dentry，并用in_lookup_hashtable检测、处理并发的创建操作
				  |- inode->i_op->lookup // 通过父dentry的inode去查找对应的dentry：通过其文件系统，获取信息及创建对应的dentry
			  |- step_into() //处理dentry是一个链接或者挂载点的情况
			    |- handle_mounts() //处理一个挂载点的情况，获取最后一个挂载在挂载点的文件系统信息
				  |- __follow_mount_rcu() //轮询调用__lookup_mnt()，处理重复挂载（查找标记有LOOKUP_RCU时调用）
				    |- __lookup_mnt()	//寻找挂载点
				  |- traverse_mounts() //作用与__follow_mount_rcu()类似
				|- pick_link() //处理是一个链接的情况，获取对应的真实路径
		  |- do_last() //分析最后的路径分量，部分代码与link_path_walk()类似
			|- lookup_fast()
			|- step_into()
		  |- do_open() //根据最后路径分量的inode，调用binder设备的binder_open()
			|- vfs_open()
			|- do_dentry_open() //将最后的路径分量对应的inode，将inode支持的file_operations，保存在file中
								//最后调用file中的file_operations里的open函数指针，最终会调用binder_open()
				|- binder_open() //打开binder设备，进行相关初始化
```

####	系统调用入口

`open`系统调用如下：
```CPP
SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
	//64位机器下，force_o_largefile 将展开为 true 并且 O_LARGEFILE 标志将被添加到 open 系统调用的 flags 参数中
    if (force_o_largefile())
        flags |= O_LARGEFILE;

    return do_sys_open(AT_FDCWD, filename, flags, mode);
}
```

![open-call-flow]()

[`do_sys_open`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/open.c#L1036)主要流程如下：

1.	检查&&解析传入的`flags`与`mode`
2.	复制文件名到内核空间，调用 `getname(filename)` 从用户空间复制文件名，避免直接访问用户内存引发安全问题
3.	分配文件描述符fd，调用 `get_unused_fd_flags(flags)` 从当前进程的文件描述符表`files_struct->fdt`中分配空闲的 fd。若分配失败，返回 `EMFILE` 错误
4.	打开文件对象 `struct file`，调用 `do_filp_open(dfd, tmp, &op)` 执行路径解析和文件打开操作
5.	关联 fd 与 file 对象，若`do_filp_open`成功，通过 `fd_install(fd, f)` 将 `struct file` 指针存入进程的 fd 数组，完成映射

```cpp
long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
	struct open_flags op;
	//检查、解析传入标志位
	int fd = build_open_flags(flags, mode, &op);
	struct filename *tmp;

	if (fd)
		return fd;

	// 将用户空间的路径名复制到内核空间
	tmp = getname(filename);
	if (IS_ERR(tmp))
		return PTR_ERR(tmp);

    // 获取一个空闲的文件描述符
	fd = get_unused_fd_flags(flags);
	if (fd>= 0) {
        // 调用 do_filp_open() 函数打开文件，返回打开文件的 struct file 结构
		struct file *f = do_filp_open(dfd, tmp, &op);
		if (IS_ERR(f)) {
			//若传参有误，则 do_filp_open 执行失败，并使用 put_unused_fd 释放文件描述符
			put_unused_fd(fd);
			fd = PTR_ERR(f);
		} else {
            // 通知 fsnotify 机制
			fsnotify_open(f);
            // 调用 fd_install() 函数把文件描述符 fd 与 file 结构关联起来
			// 即将struct file *f加入到fd索引位置处的数组中
			// 如果后续过程中，有对该文件描述符的操作的话，就会通过查找该数组得到对应的文件结构，而后在进行相关操作
			fd_install(fd, f);
		}
	}
	//释放已分配的 filename 结构体
	putname(tmp);
    // 返回文件描述符，也就是 open() 系统调用的返回值
	return fd;
}

void fd_install(unsigned int fd, struct file *file)
{
	__fd_install(current->files, fd, file);
}
```

可以看到，`open` 系统调用的目的就是建立新建 fd 与 `struct file` 的绑定关系，但是实际上，间接建立的 VFS 内存映射关系可能如下图（假设路径 `b` 文件并不在 dcache 中）

![dcache-build](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/open/open-2-dentry-build.png)

####	getname：复制路径名到内核

`__getname` 用于在内核缓冲区专用队列里申请一块内存用来放置路径名，作用如下：

TODO

##	0x05	do_filp_open->path_openat实现
[`do_filp_open`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L3515)，主要流程如下：

1.	初始化路径查找上下文，通过 `path_openat->path_init(dfd, pathname, flags, &nd)` 初始化 `struct nameidata nd`，确定起始目录（当前目录 `AT_FDCWD` 或根目录）
2.	逐级遍历路径分量，通过`link_path_walk(pathname, &nd)` 解析每一级路径：
	-	查找目录项（dentry）并检查权限
	-	处理挂载点：若当前 dentry 是挂载点（`d_mountpoint` 标志），切换到新文件系统的根目录（`vfsmount->mnt_root`）
	-	解析符号链接（symbol link）：递归处理链接目标路径
3.	处理最终路径分量，通过`do_last(&nd, file, ...)` 处理最后一个路径分量，大致流程为：
	-	若文件不存在且指定 `O_CREAT`，则创建新文件
	-	检查访问权限（`may_open()`）
	-	调用 `vfs_open(&path, file)` 执行实际文件系统的打开操作

```CPP
/*
参数 dfd：相对路径的基准目录对应的文件描述符
参数 pathname：指向文件完整路径
参数 op：查找标志
*/
struct file *do_filp_open(int dfd, struct filename *pathname,
		const struct open_flags *op)
{
	struct nameidata nd;
	int flags = op->lookup_flags;
	struct file *filp;

    // 将 pathname 保存到 nameidata 里
	set_nameidata(&nd, dfd, pathname);
    // 调用 path_openat，此时是 flags 是带了 LOOKUP_RCU flag

	//RCU 模式（首先尝试RCU快速路径）
	filp = path_openat(&nd, op, flags | LOOKUP_RCU);
	if (unlikely(filp == ERR_PTR(-ECHILD)))
		//正常模式（回退到慢速路径）
		filp = path_openat(&nd, op, flags);
	if (unlikely(filp == ERR_PTR(-ESTALE)))	
		//NFS模式，这里不讨论
		filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
	restore_nameidata();
	return filp;
}
```

函数 `do_filp_open` 三次调用函数 `path_openat`以解析文件路径的目的是什么？简单说明下

1.	第一次解析传入标志 `LOOKUP_RCU`，该模式 `LOOKUP_RCU` 即rcu-walk方式，在 dcache 哈希表中根据`{父目录, 名称}`查找目录的过程中，使用 RCU机制保护dcache桶上的链表，使用序列号保护目录，其他处理器可以并行地修改目录, RCU 查找方式速度最快
2.	如果在第一次解析的过程中发现其他处理器修改了正在查找的目录（问题：内核如何发现？），返回错误号`-ECHILD`，那么第二次使用引用查找（ref-walk）即REF 方式，在dcache中根据`{父目录, 名称}`查找目录的过程中，使用 RCU 保护散列桶的链表，使用自旋锁保护目录，并且把目录的引用计数加`1`，引用查找方式速度较慢
3.	网络文件系统的文件在网络的服务器上，本地上次查询得到的信息可能过期,和服务器的当前状态不一致。如果第二次解析发现信息过期，返回错误号 `-ESTALE`，那么第三次解析传入标志 `LOOKUP_REVAL`，表示需要重新确认信息是否有效

####	do_filp_open->path_openat
[`path_openat`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L3457)，在 `path_openat` 中，先调用 `get_empty_filp` 方法分配一个空的 `struct file` 实例，再调用 `path_init`、`link_path_walk`、`do_last` 等方法执行后续的 open 操作，如果都成功了，则返回 `struct file` 给上层

`path_openat`的主要功能是尝试寻找一个与路径相符合的 dentry 目录数据结构，核心方法是 `path_init`、`link_path_walk`、`do_last`，其中 `path_init` 和 `link_path_walk` 通常合在一起调用，作用是 **可以根据给定的文件路径名称在内存中找到或者建立代表着目标文件或者目录的 dentry 结构和 inode 结构**

注意`while (!(error = link_path_walk(s, nd)) && (error = do_last(nd, file, op, &opened)) > 0)` 这里的循环的作用是什么？通过后续的`trailing_symlink`字面意思不难看出，是为了递归处理路径解析过程中可能存在的末尾符号链接（trailing symlink）

此外，这里阅读代码要区别`flags`带不带`LOOKUP_RCU`，即快速/慢速查找模式

```CPP
static struct file *path_openat(struct nameidata *nd,
			const struct open_flags *op, unsigned flags)
{
	const char *s;
	struct file *file;
	int opened = 0;
	int error;

	file = get_empty_filp();
	if (IS_ERR(file))
		return file;

	file->f_flags = op->open_flag;

	......

	if (unlikely(file->f_flags & O_PATH)) {
		error = do_o_path(nd, flags, file);
		if (!error)
			opened |= FILE_OPENED;
		goto out2;
	}

    // 路径初始化，确定查找的起始目录，初始化结构体 nameidata 的成员 path
    // 调用 path_init() 设置 nameidata 的 path 结构体
    // 对于常规文件来说，如 /data/test/testfile，设置 path 结构体指向根目录 /
	// 即设置 path.mnt/path.dentry 指向根目录 /
	// 为后续的目录解析做准备
    // path_init() 的返回值即指向 open file 的完整路径名字串开头
	s = path_init(nd, flags);
	if (IS_ERR(s)) {
		put_filp(file);
		return ERR_CAST(s);
	}
    // 核心方法
    // 调用函数 link_path_walk 解析文件路径的每个分量，最后一个分量除外
  	// 调用函数 do_last，解析文件路径的最后一个分量，并且打开文件
	while (!(error = link_path_walk(s, nd)) &&
		(error = do_last(nd, file, op, &opened)) > 0) {
		nd->flags &= ~(LOOKUP_OPEN|LOOKUP_CREATE|LOOKUP_EXCL);
        // 如果最后一个分量是符号链接，调用 trailing_symlink 函数进行处理
    	// 读取符号链接文件的数据，新的文件路径是符号链接链接文件的数据，然后继续 while
    	// 循环，解析新的文件路径
		s = trailing_symlink(nd);
		if (IS_ERR(s)) {
			error = PTR_ERR(s);
			break;
		}
	}
	terminate_walk(nd);
out2:
	if (!(opened & FILE_OPENED)) {
		BUG_ON(!error);
		put_filp(file);
	}
	if (unlikely(error)) {
		if (error == -EOPENSTALE) {
			if (flags & LOOKUP_RCU)
				error = -ECHILD;
			else
				error = -ESTALE;
		}
		file = ERR_PTR(error);
	}
	return file;
}
```

这里稍微整理一下`while(....)`中的实现过程和部分关键代码，对于路径 `/a/b/c/d/e`的处理，在进入 `do_last`之前，路径中除最后一个分量外的所有目录分量（`a`、`b`、`c`、`d`）都已经成功解析，并且它们的 dentry 通常已经存在于 dcache 中，最后一个分量（`e`）的处理在 `do_last`中完成

1、目录分量缓存机制，在路径解析过程中遵循如下规则：

-	缓存优先：每个目录分量首先通过 `lookup_fast`尝试从 dcache 获取
-	缓存未命中：如果未命中缓存dcache，通过 `lookup_slow`和文件系统查找
-	缓存填充：新找到的目录分量通过 `d_add`加入 `dcache`（`d_alloc_parallel`）
-	路径更新：`path_to_nameidata`更新当前路径

TODO

####	path_openat->path_init
`path_init` 方法主要是用来初始化 `struct nameidata` 实例中的 `path`、`root`、`inode` 等字段。当 `path_init` 函数执行成功后，就会在 `nameidata` 结构体的成员 `nd->path.dentry` 中指向搜索路径的起点，接下来就使用 `link_path_walk` 函数顺着路径进行搜索

```CPP
static const char *path_init(struct nameidata *nd, unsigned flags)
{
	int retval = 0;
	const char *s = nd->name->name;
	// 如果路径名为空，清除 LOOKUP_RCU 标志
	if (!*s)
		flags &= ~LOOKUP_RCU;

	nd->last_type = LAST_ROOT;

	// 重要：path_openat的参数flags保存在nameidata的flags成员中
	nd->flags = flags | LOOKUP_JUMPED | LOOKUP_PARENT;
	nd->depth = 0;

	// 如果设置 `LOOKUP_ROOT`
	// 表示 nameidata 中的 root 字段是由调用者提供的
	if (flags & LOOKUP_ROOT) {
		struct dentry *root = nd->root.dentry;
		struct inode *inode = root->d_inode;
		if (*s) {
			if (!d_can_lookup(root))
				return ERR_PTR(-ENOTDIR);
			retval = inode_permission(inode, MAY_EXEC);
			if (retval)
				return ERR_PTR(retval);
		}
		nd->path = nd->root;
		nd->inode = inode;
		// 如果是 RCU 快速模式，则保存序列锁（处理竞争）
		if (flags & LOOKUP_RCU) {
			rcu_read_lock();
			nd->seq = __read_seqcount_begin(&nd->path.dentry->d_seq);
			nd->root_seq = nd->seq;
			nd->m_seq = read_seqbegin(&mount_lock);
		} else {
			// 不是RCU模式
			// 如果是 REF 模式，则获取 path 的计数引用（处理竞争）
			path_get(&nd->path);
		}
		return s;
	}

	nd->root.mnt = NULL;
	nd->path.mnt = NULL;
	nd->path.dentry = NULL;

	// mount_lock 是一个全局 seqlock，有点像 rename_lock
	// 它可以用来检查任何挂载点的任何修改
	nd->m_seq = read_seqbegin(&mount_lock);

	// CASE1
	// 如果是以 / 开头，也就是说明是绝对路径
	// 获取文件系统的根目录，并保存在nameidata中
	if (*s == '/') {
		if (flags & LOOKUP_RCU)
			rcu_read_lock();	//RCU模式加锁
		set_root(nd);
		if (likely(!nd_jump_root(nd)))
			return s;
		nd->root.mnt = NULL;
		rcu_read_unlock();	//RCU模式解锁
		return ERR_PTR(-ECHILD);
	} else if (nd->dfd == AT_FDCWD) {
		//如果为相对路径，且未指定相对路径的基准目录
		if (flags & LOOKUP_RCU) {
			//current是一个结构体task_struct，当前正在运行进程的进程描述符
			struct fs_struct *fs = current->fs;
			unsigned seq;

			rcu_read_lock();
			//在RCU 快速模式下，通过RCU锁初始化nameidata的成员（from：当前进程的目录信息）
			do {
				seq = read_seqcount_begin(&fs->seq);
				nd->path = fs->pwd;
				nd->inode = nd->path.dentry->d_inode;
				nd->seq = __read_seqcount_begin(&nd->path.dentry->d_seq);
			} while (read_seqcount_retry(&fs->seq, seq));
		} else {
			// 非RCU模式（REF模式），使用自旋锁保证并发安全
			// 见下
			get_fs_pwd(current->fs, &nd->path);
			nd->inode = nd->path.dentry->d_inode;
		}
		return s;
	} else {
		//如果为相对路径，且已经指定了相对路径的基准目录
		/* Caller must check execute permissions on the starting path component */
		struct fd f = fdget_raw(nd->dfd);
		struct dentry *dentry;

		if (!f.file)
			return ERR_PTR(-EBADF);

		dentry = f.file->f_path.dentry;

		if (*s) {
			if (!d_can_lookup(dentry)) {
				fdput(f);
				return ERR_PTR(-ENOTDIR);
			}
		}

		nd->path = f.file->f_path;
		if (flags & LOOKUP_RCU) {
			rcu_read_lock();
			nd->inode = nd->path.dentry->d_inode;
			nd->seq = read_seqcount_begin(&nd->path.dentry->d_seq);
		} else {
			path_get(&nd->path);
			nd->inode = nd->path.dentry->d_inode;
		}
		fdput(f);
		return s;
	}
}

static inline void get_fs_pwd(struct fs_struct *fs, struct path *pwd)
{
	spin_lock(&fs->lock);
	*pwd = fs->pwd;
	path_get(pwd);
	spin_unlock(&fs->lock);
}
```

```CPP
static int nd_jump_root(struct nameidata *nd)
{
	if (!nd->root.mnt) {
        //获取根目录的Path信息，保存到nd->root中
		int error = set_root(nd);
	}
	if (nd->flags & LOOKUP_RCU) {
        //将根文件系统的path、inode记录到nameidata中
		struct dentry *d;
		nd->path = nd->root;
		d = nd->path.dentry;
		nd->inode = d->d_inode;
		nd->seq = nd->root_seq;
	} else {
		path_put(&nd->path);
		nd->path = nd->root;
		path_get(&nd->path);
		nd->inode = nd->path.dentry->d_inode;
	}
	nd->state |= ND_JUMPED;
	return 0;
}
        
static int set_root(struct nameidata *nd)
{
    //获取当前进程的fs_struct
	struct fs_struct *fs = current->fs;

	if (nd->flags & LOOKUP_RCU) {
		unsigned seq;
		do {
			seq = read_seqcount_begin(&fs->seq);
            //从fs中获取根目录"/"的Path，里面有它对应的dentry，保存在nd->root中
			nd->root = fs->root;
			nd->root_seq = __read_seqcount_begin(&nd->root.dentry->d_seq);
		} while (read_seqcount_retry(&fs->seq, seq));
	}
	return 0;
}
```

####	path_openat->link_path_walk
[`link_path_walk`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L2042)函数用于实现path walk的核心功能，逐个解析文件路径分量（除了最后一个分量），获取对应的dentry、inode（查找分量对应的dentry/inode通过函数`walk_component`实现）

```CPP
// 计算name中除根路径组件外的第一个路径组件的hash值和名字长度
// 返回值hash_len是64位的无符号整数
// 高32位记录name的长度len，低32位记录name的hash值
static inline u64 hash_name(const void *salt, const char *name)
{
	unsigned long hash = init_name_hash(salt);
	unsigned long len = 0, c;

	c = (unsigned char)*name;
	do {
		len++;
		hash = partial_name_hash(c, hash);
		c = (unsigned char)name[len];
	} while (c && c != '/');	// 截断/
	return hashlen_create(end_name_hash(hash), len);
}

// link_path_walk：循环查找最后一个分量之前的所有分量
static int link_path_walk(const char *name, struct nameidata *nd)
{
	int err;
 	int depth = 0; // depth <- nd->depth
    // 跳过开始的 / 字符（根目录）
	// 指针移动，/dev/binder变成 dev/binder
	while (*name=='/'){
		name++;
	}

    // 如果路径只包含 /，搜索完成，返回
	if (!*name)
		return 0;

	/* At this point we know we have a real path component. */
	//轮询处理各个文件路径名
	for(;;) {
		u64 hash_len;
		int type;

        //https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1667
        //may_lookup 检查是否拥有中间目录的权限，需要有执行权限 MAY_EXEC
		err = may_lookup(nd);
		if (err)
			return err;

        //static inline u64 hash_name(const vo id *salt, const char *name)
        // 用父 dentry 的地址 + 当前 denty 的 name 计算 hash 值
        // 逐个字符的计算出，当前节点名称的哈希值，遇到 '/' 或者 '\0' 退出
        // 计算当前目录的 hash_len，这个变量高 4 byte 是当前目录 name 字串长度，低 4byte 是当前目录（路径）的 hash 值，hash 值的计算是基于当前目录的父目录 dentry（nd->path.dentry）来计算的，所以它跟其目录（路径）dentry 是关联的
		hash_len = hash_name(nd->path.dentry, name);

		type = LAST_NORM;

        // 如果目录的第一个字符是.，当前节点长度只能为 1 或者 2
		// `.` 或者 `..`
		if (name[0] == '.') switch (hashlen_len(hash_len)) {
			case 2:
				if (name[1] == '.') {
                    // 如果是 2，第二个字符也是.
					type = LAST_DOTDOT;
                    //.. 需要查找当前目录的父目录
					nd->flags |= LOOKUP_JUMPED;
				}
				break;
			case 1:
                // 回到 for 循环开始，继续下一个节点
				type = LAST_DOT;
		}
		if (likely(type == LAST_NORM)) {
            // LAST_NORM：普通目录
			// 第一次循环parent代表根目录"/"（在path_init中获取到）
            // 第二次循环parent代表目录"dev"
			// ......
			struct dentry *parent = nd->path.dentry;
			nd->flags &= ~LOOKUP_JUMPED;
			if (unlikely(parent->d_flags & DCACHE_OP_HASH)) {
                // 当前目录项需要重新计算一下 hash 值
				struct qstr this;	//初始化并设置this的值
                // 调用 parent 这个 dentry 的 parent->d_op->d_hash 方法计算 hash 值
				err = parent->d_op->d_hash(parent, &this);
				if (err < 0)
					return err;
				hash_len = this.hash_len;
				name = this.name;
			}
		}

        // 更新 nameidata last 结构体（解析完成）
		nd->last.hash_len = hash_len;
		nd->last.name = name;
		nd->last_type = type;
        // 这里使 name 指向下一级目录
		// 指针移动，第一次循环时，dev/binder 变成了 /binder
		name += hashlen_len(hash_len);
	    //注意通常循环处理到最后一个分量时，if (!*name)是为true的
		if (!*name)
			goto OK;
		/*
		 * If it wasn't NUL, we know it was'/'. Skip that
		 * slash, and continue until no more slashes.
		 */
		//指针移动，第一次循环时，/binder 变成了 binder	
		do {
			name++;
		} while (unlikely(*name == '/'));
		if (unlikely(!*name)) {
            // 假设 open file 文件名路径上没有任何 symlink，则如果这个条件满足，说明整个路径都解析完了
			// 还剩最后的 filename 留给 do_last() 解析，此函数将从下面的!nd->depth 条件处返回
OK:
			// pathname body, done
			if (!nd->depth)
				// 路径最后一个分量，不管是否是symlink，
                // 都会走到这里，不会执行walk_component()
                // 此时已经到达了最终目标，路径行走任务完成
                 // 如果 open file 完整路径上没有任何 symlink，nd->depth 等于 0
				return 0;
			name = nd->stack[nd->depth - 1].name;
			/* trailing symlink, done */
			if (!name)
                // 此时已经到达了最终目标，路径行走任务完成
				return 0;
			/* last component of nested symlink */
            // symlink case
			// symlink即软链接
			// 只有嵌套在文件路径中的symlink才可能走这里
            // 如果是位于文件路径末尾的symlink，都会先视为最后一个组件，不会走这里
			err = walk_component(nd, WALK_FOLLOW);
		} else {
			/* not the last component */
            // 常规目录 case，非 symlink 的 case
			// 不是最后一个分量，查找对应的dentry、inode
			// 检查是否是挂载点
			err = walk_component(nd, WALK_FOLLOW | WALK_MORE);
		}
		if (err < 0)
			return err;

		if (err) {
			const char *s = get_link(nd);

			if (IS_ERR(s))
				return PTR_ERR(s);
			err = 0;
			// s指symlink对应的真实路径
			if (unlikely(!s)) {
				/* jumped */
				put_link(nd);
			} else {
				//只有嵌套在文件路径中的symlink，才会走到这里
                //如"/a/b/c"，检查组件a时，发现它其实是个嵌套symlink，代表文件路径"e/f"
                //在执行walk_component->get_link后，返回的link就是"e/f"
                //这时候，name就是"b/c"，将它保存在nd->stack中，并自增depth
                //walk_component完symlink对应的真实路径后，才会取出来"b/c"
				//对它执行walk_component()
				nd->stack[nd->depth - 1].name = name;
				name = s;
				// 继续处理
				//将"e/f"赋值给name，继续循环，执行walk_component()
				continue;
			}
		}
		if (unlikely(!d_can_lookup(nd->path.dentry))) {
			if (nd->flags & LOOKUP_RCU) {
				if (unlazy_walk(nd))
					return -ECHILD;
			}
			return -ENOTDIR;
		}
	}
}
```

`link_path_walk` 函数中会调用 `walk_component` [函数](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1763)来进行路径搜索（向上或者向下），这个是较为复杂的逻辑，有很多场景，典型的如：

-   `.` 或 `..`
-   普通的目录，这里又会检测当前目录是否为其他文件系统的挂载点
-   符号链接

对于本文前面提到的vfs path walk的例子，对于Binder文件系统挂载而言，`/dev/binder`是`/binderfs/binder`的symlink，不过由于`binder`是路径`/dev/binder`的最后一个分量，所以它在第一次执行`link_path_walk()`，不会对其用`walk_component()`进行查找。整体的处理逻辑，得看调用link_path_walk()的地方path_openat()
TODO

```CPP
static struct file *path_openat(struct nameidata *nd,
			const struct open_flags *op, unsigned flags)
{
        
        //s即"/dev/binder"
        const char *s = path_init(nd, flags);
        //在第一次循环时，s是"/dev/binder"，执行link_path_walk()，"binder"是最后一个组件，
        //跳出link_path_walk()的循环处理，调用open_last_lookups()处理，但发现"binder"是一个symlink。
        //open_last_lookups()会返回"binder"对应的"binderfs/binder"（又或者是"dev/binderfs/binder"?）。
        
        //第二次循环时，s已经是"binderfs/binder", 再次执行link_path_walk()，"binder"是最后一个组件，
        //不过这个"binder"不同于上次循环那个，不是symlink，而是一个真实存在的路径组件
        //最后调用open_last_lookups()即可获得最后一个组件"binder"对应的dentry、inode等信息
        while (!(error = link_path_walk(s, nd)) &&
		(error = do_last(nd, file, op, &opened)) > 0) {
			......
			s = trailing_symlink(nd);
			......
		}
}
```

####	path_openat->link_path_walk->walk_component
[`walk_component`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1763) 方法对 `nd`（中间结果）中的目录进行遍历，当前的子路径一定是一个中间节点（目录OR符号链接）

-   优先使用 `lookup_fast`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1537)：如果当前的目录是一个普通目录，路径行走有两个策略：先在效率高的 rcu-walk 模式 [`__d_lookup_rcu`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1554) 下遍历，如果失败了就在效率较低的 ref-walk 模式 [`__d_lookup`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1600) 下遍历
-   如果 `lookup_fast` 查找失败，则调用 `lookup_slow` 函数。在 ref-walk 模式下会 **首先在内存缓冲区查找相应的目标（lookup_fast），如果找不到就启动具体文件系统（如 `ext4`）自己的 `lookup` 进行查找（lookup_slow）**
-   在 dcache 里找到了当前目录对应的 dentry 或者是通过 `lookup_slow` 寻找到当前目录对应的 dentry，这两种场景都会去设置 `path` 结构体里的 `dentry`、`mnt` 成员，并且将当前路径更新到 `path` 结构体（对dcache的分析参考后文）
-   当 `path` 结构体更新后，最后调用 `step_info->path_to_nameidata` 将 `path` 结构体更新到 `nd.path`，[参考](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1803)，这样 `nd` 里的 `path` 就指向了当前目录了，至此完成一级目录的解析查找，返回 `link_path_walk()` 将基于 `nd.path` 作为父目录解析下一级目录，继续 `link_path_walk` 的循环查找直至退出

TODO：快速模式慢速模式下的调用顺序，从`walk_component`中`lookup_fast`的返回值`err`可知：

-	`err>0`：
-	`err == 0`：
-	`err<0`：直接返回错误（如经典的`-ECHILD`）


```CPP
static int walk_component(struct nameidata *nd, int flags)
{
	struct path path;
	struct inode *inode;
	unsigned seq;
	int err;

	// 如果不是 LAST_NORM 类型，就交由 handle_dots 处理
	if (unlikely(nd->last_type != LAST_NORM)) {
		err = handle_dots(nd, nd->last_type);
		if (!(flags & WALK_MORE) && nd->depth)
			put_link(nd);
		return err;
	}

	// 快速查找
	err = lookup_fast(nd, &path, &inode, &seq);
	// 重点：当lookup_fast返回1时，说明快速路径查找成功
	if (unlikely(err <= 0)) {
		if (err < 0){
			return err;
		}
		// 如果快速查找模式失败，则进行慢速查找模式
		path.dentry = lookup_slow(&nd->last, nd->path.dentry,
					  nd->flags);
		if (IS_ERR(path.dentry))
			return PTR_ERR(path.dentry);

		path.mnt = nd->path.mnt;
		err = follow_managed(&path, nd);
		if (unlikely(err < 0))
			return err;

		if (unlikely(d_is_negative(path.dentry))) {
			path_to_nameidata(&path, nd);
			return -ENOENT;
		}

		seq = 0;	/* we are already out of RCU mode */
		inode = d_backing_inode(path.dentry);
	}
	//检查是否是链接、挂载点
    //是链接，则获取到对应的真实路径
    //是挂载点，则获取最后一个挂载在挂载点的文件系统信息
	return step_into(nd, &path, flags, inode, seq);
}
```

接下来看下`walk_component`中最核心的两个函数：`lookup_fast`与`lookup_slow`

####	路径查找的核心：walk_component->lookup_fast
`lookup_fast()`函数根据路径分量的名称，快速找到对应的dentry、inode的实现，主要分为rcu-walk和ref-walk，二者都是从`dentry_hashtable`中查询，但是在并发实现上有差异

-	rcu-walk：实现是`__d_lookup_rcu()`
-	ref-walk：实现是`__d_lookup()`

**即在快速模式，慢速模式都会调用`lookup_fast`，快速模式中的`lookup_fast`对应的实现是`__d_lookup_rcu`，而慢速模式下的`lookup_fast`对应的是`__d_lookup`**

```CPP
|- lookup_fast()
	|- __d_lookup_rcu() //实现了rcu-walk
		|- d_hash() 	 //根据hash值，从dentry_hashtable中获取对应的hlist_bl_head
		|- hlist_bl_for_each_entry_rcu() //遍历链表hlist_bl_head，寻找对应的dentry
	|- __d_lookup() 	 //实现了ref-walk
```

```CPP
static int lookup_fast(struct nameidata *nd,
		       struct path *path, struct inode **inode,
		       unsigned *seqp)
{
	struct vfsmount *mnt = nd->path.mnt;
	struct dentry *dentry, *parent = nd->path.dentry;
	int status = 1;
	int err;

	//flags里有LOOKUP_RCU标记，则执行rcu-walk，否则执行ref-walk
	if (nd->flags & LOOKUP_RCU) {
		unsigned seq;
		bool negative;
		// rcu-walk
		dentry = __d_lookup_rcu(parent, &nd->last, &seq);
		if (unlikely(!dentry)) {
			// 移除flags里的LOOKUP_RCU标记，尝试切换到ref-walk
			// 成功则在下一个分量的lookup中，会采用ref-walk
			// 当前的分量，看流程，不会换到ref-walk，而是用lookup_slow进行查找
			if (unlazy_walk(nd))
				return -ECHILD;
			return 0;
		}

		*inode = d_backing_inode(dentry);
		negative = d_is_negative(dentry);
		if (unlikely(read_seqcount_retry(&dentry->d_seq, seq)))
			return -ECHILD;

		if (unlikely(__read_seqcount_retry(&parent->d_seq, nd->seq)))
			return -ECHILD;

		*seqp = seq;
		status = d_revalidate(dentry, nd->flags);
		if (likely(status > 0)) {
			/*
			 * Note: do negative dentry check after revalidation in
			 * case that drops it.
			 */
			if (unlikely(negative))
				return -ENOENT;
			path->mnt = mnt;
			path->dentry = dentry;
			if (likely(__follow_mount_rcu(nd, path, inode, seqp))){
				// 快速模式下查找成功
				return 1;
			}
		}
		if (unlazy_child(nd, dentry, seq))
			return -ECHILD;
		if (unlikely(status == -ECHILD))
			/* we'd been told to redo it in non-rcu mode */
			status = d_revalidate(dentry, nd->flags);
	} else {
		//ref-walk
		dentry = __d_lookup(parent, &nd->last);
		if (unlikely(!dentry))
			return 0;
		status = d_revalidate(dentry, nd->flags);
	}
	if (unlikely(status <= 0)) {
		if (!status)
			d_invalidate(dentry);
		dput(dentry);
		return status;
	}
	if (unlikely(d_is_negative(dentry))) {
		dput(dentry);
		return -ENOENT;
	}

	path->mnt = mnt;
	path->dentry = dentry;
	err = follow_managed(path, nd);
	if (likely(err > 0))
		*inode = d_backing_inode(path->dentry);
	return err;
}
```

注意`lookup_fast`中的`unlazy_walk`函数：

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L683
static int unlazy_walk(struct nameidata *nd)
{
	struct dentry *parent = nd->path.dentry;

	BUG_ON(!(nd->flags & LOOKUP_RCU));

	// 清除nd->flags中的LOOKUP_RCU标志
	nd->flags &= ~LOOKUP_RCU;
	if (unlikely(!legitimize_links(nd)))
		goto out2;
	if (unlikely(!legitimize_path(nd, &nd->path, nd->seq)))
		goto out1;
	if (nd->root.mnt && !(nd->flags & LOOKUP_ROOT)) {
		if (unlikely(!legitimize_path(nd, &nd->root, nd->root_seq)))
			goto out;
	}
	rcu_read_unlock();
	BUG_ON(nd->inode != parent->d_inode);
	return 0;

out2:
	nd->path.mnt = NULL;
	nd->path.dentry = NULL;
out1:
	if (!(nd->flags & LOOKUP_ROOT))
		nd->root.mnt = NULL;
out:
	rcu_read_unlock();
	return -ECHILD;
}
```

继续，这里看下rcu-walk即`__d_lookup_rcu()`的实现细节， `__d_lookup_rcu()`遍历查找目标dentry的时候，使用了顺序锁`seqlock`，读操作不会被写操作阻塞，写操作也不会被读操作阻塞。但读操作可能会反复读取相同的数据：当它发现sequence发生了变化，即它执行期间，有写操作更改了数据。此外`seqlock`要求进入临界区的写操作只有一个，多个写操作之间仍然是互斥的

在`__d_lookup_rcu`函数中，`dentry->d_seq`的类型是`seqcount_spinlock_t`，`seqcount_spinlock_t`经过一些复杂的宏定义包含了`seqcount_t`，可以简单认为`seqcount_spinlock_t`就是一个`int`序列号

```CPP
struct dentry *__d_lookup_rcu(const struct dentry *parent,
				const struct qstr *name,
				unsigned *seqp)
{
	u64 hashlen = name->hash_len;
	const unsigned char *str = name->name;
	struct hlist_bl_head *b = d_hash(hashlen_hash(hashlen));
	struct hlist_bl_node *node;
	struct dentry *dentry;

	hlist_bl_for_each_entry_rcu(dentry, node, b, d_hash) {
		unsigned seq;

seqretry:
		seq = raw_seqcount_begin(&dentry->d_seq);
		if (dentry->d_parent != parent)
			continue;
		if (d_unhashed(dentry))
			continue;

		if (unlikely(parent->d_flags & DCACHE_OP_COMPARE)) {
			int tlen;
			const char *tname;
			if (dentry->d_name.hash != hashlen_hash(hashlen))
				continue;
			tlen = dentry->d_name.len;
			tname = dentry->d_name.name;
			/* we want a consistent (name,len) pair */
			if (read_seqcount_retry(&dentry->d_seq, seq)) {
				cpu_relax();
				goto seqretry;
			}
			if (parent->d_op->d_compare(dentry,
						    tlen, tname, name) != 0)
				continue;
		} else {
			if (dentry->d_name.hash_len != hashlen)
				continue;
			if (dentry_cmp(dentry, str, hashlen_len(hashlen)) != 0)
				continue;
		}
		*seqp = seq;
		return dentry;
	}
	return NULL;
}
```


接着看下慢速模式即`__d_lookup()`的[实现](https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L2203)，`__d_lookup`遍历查找目标dentry的时候，使用了自旋锁`spin_lock`，多个读操作会发生锁竞争。它还会更新查找到的dentry的引用计数（因此叫做ref-walk）。RCU仍然用于ref-walk中的dentry哈希查找，但不是在整个ref-walk过程中都使用，频繁地加减reference count可能造成cacheline的刷新，这也是ref-walk开销更大的原因之一

```CPP
struct dentry *__d_lookup(const struct dentry *parent, const struct qstr *name)
{
	struct hlist_bl_head *b = d_hash(hash);
	rcu_read_lock();
    //遍历链表b里的dentry，查找目标dentry
    hlist_bl_for_each_entry_rcu(dentry, node, b, d_hash) {
		
		if (dentry->d_name.hash != hash)
			continue;	//name_hash不对，不是要找的
        //spin_lock加锁
        spin_lock(&dentry->d_lock);
		if (dentry->d_parent != parent)
			goto next;	//parent 不对，不是要找的
		if (d_unhashed(dentry))
			goto next;

		if (!d_same_name(dentry, parent, name))
			goto next;	//full name 比对，不是要找的
        ......
        //查找到目标dentry，增加引用计数
        dentry->d_lockref.count++;
		found = dentry;	// 找到了，可以返回了
		//解锁
		spin_unlock(&dentry->d_lock);
		break;
next:
        //没有查找到目标dentry会跳转到next这里，解锁，继续查找
		spin_unlock(&dentry->d_lock);  
    }
	rcu_read_unlock();
    ...
}
```

####	路径查找：walk_component->lookup_slow
与上述方法不同，`lookup_fast`的两种模式，都是查询的`dentry_hashtable`，`lookup_slow`是兜底方案，即当`lookup_fast`失败后，才会调用。`lookup_slow()`是通过当前所在的文件系统，获取对应的信息，创建对应的dentry和inode，将新的dentry添加到`dentry_hashtable`中

```BASH
|- lookup_slow() //在lookup_fast()中没有找到dentry，会获取父dentry对应的inode，通过inode->i_op->lookup去查找、创建
	|- __lookup_slow()
        |- d_alloc_parallel() //创建一个新的dentry，并用in_lookup_hashtable检测、处理并发的创建操作
        |- inode->i_op->lookup 通过父dentry的inode去查找对应的dentry：其实就是通过它所在的文件系统，获取对应的信息，创建对应的dentry
```

[`__lookup_slow`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1625) 实现如下，首先调用 `d_alloc_parallel` 给当前路径分配一个新的 dentry，然后调用 `inode->i_op->lookup()`，注意这里的 inode 是当前路径的父路径 dentry 的 `d_inode` 成员。`inode->i_op` 是具体的文件系统 inode operations 函数集，如对 ext4 文件系统就是 ext4 fs 的 inode operations 函数集 `ext4_dir_inode_operations`，其 lookup 函数是 `ext4_lookup()`

```CPP
/* Fast lookup failed, do it the slow way */
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1625
static struct dentry *lookup_slow(const struct qstr *name,
				  struct dentry *dir,
				  unsigned int flags)
{
	struct dentry *dentry = ERR_PTR(-ENOENT), *old;
	struct inode *inode = dir->d_inode; // 指向当前路径的父路径
	DECLARE_WAIT_QUEUE_HEAD_ONSTACK(wq);

	inode_lock_shared(inode);
	/* Don't go there if it's already dead */
	if (unlikely(IS_DEADDIR(inode)))
		goto out;
again:
    // 新建 dentry 节点，并初始化相关关联
	dentry = d_alloc_parallel(dir, name, &wq);
	if (IS_ERR(dentry))
		goto out;
	if (unlikely(!d_in_lookup(dentry))) {
		if (!(flags & LOOKUP_NO_REVAL)) {
			int error = d_revalidate(dentry, flags);
			if (unlikely(error <= 0)) {
				if (!error) {
					d_invalidate(dentry);
					dput(dentry);
					goto again;
				}
				dput(dentry);
				dentry = ERR_PTR(error);
			}
		}
	} else {
        // 在 ext4 fs 中，会调用 ext4_lookup 寻找，此函数涉及到 IO 操作，性能较 dcache 会低
		old = inode->i_op->lookup(inode, dentry, flags);
		d_lookup_done(dentry);
		if (unlikely(old)) {
            // 如果 dentry 是一个目录的 dentry，则有可能 old 是有效的；否则如果 dentry 是文件的 dentry 则 old 是 null
			dput(dentry);
			dentry = old;
		}
	}
out:
	inode_unlock_shared(inode);
	return dentry;
}
```

这里简单介绍下 `ext4_lookup()` 的实现，其原型为 `static struct dentry *ext4_lookup(struct inode *dir, struct dentry *dentry, unsigned int flags)`，参数 `dir` 为当前目录 dentry 的父目录，参数 `dentry` 为需要查找的当前目录。`ext4_lookup()` 首先调用了 `ext4_lookup_entry()`，此函数根据当前路径的 dentry 的 `d_name` 成员在当前目录的父目录文件（用 inode 表示）里查找，这个会 open 父目录文件会涉及到 IO 读操作（具体可以分析 `ext4_bread` 函数的 [实现](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/inode.c#L994)）。查找到后，得到当前目录的 `ext4_dir_entry_2`，此结构体里有当前目录的 inode number，然后根据此 inode number 调用 `ext4_iget()` 函数获得这个 inode number 对应的 inode struct，得到这个 inode 后调用 `d_splice_alias()` 将 dentry 和 inode 绑定，即将 inode 赋值给 dentry 的 `d_inode` 成员

当 ext4 文件系统的 lookup 完成后，此时的 dentry 已经有绑定的 inode 了，即已经设置了其 `d_inode` 成员了，然后调用 `d_lookup_done()` 将此 dentry 从 lookup hash 链表上移除（它是在 `d_alloc_parallel` 里被插入 lookup hash 的），这个 lookup hash 链表的作用是避免其它线程也同时来查找当前目录造成重复 alloc dentry 的问题

####	walk_component->step_into
step_into()主要是处理dentry是一个链接或者挂载点的情况

```BASH
|- step_into()
	|- handle_mounts() //处理一个挂载点的情况，获取最后一个挂载在挂载点的文件系统信息
		|- __follow_mount_rcu() //轮询调用__lookup_mnt()，处理重复挂载（查找标记有LOOKUP_RCU时调用）
			|- __lookup_mnt()
		|- traverse_mounts() //作用与__follow_mount_rcu()类似
	|- pick_link() //处理是一个链接的情况，获取对应的真实路径
```

```CPP
static const char *step_into(struct nameidata *nd, int flags,
		     struct dentry *dentry, struct inode *inode, unsigned seq)
{
	struct path path;
    //处理组件是挂载点的情况
	int err = handle_mounts(nd, dentry, &path, &inode, &seq);
    //检查是否是链接
	if (likely(!d_is_symlink(path.dentry)) ||
	   ((flags & WALK_TRAILING) && !(nd->flags & LOOKUP_FOLLOW)) ||
	   (flags & WALK_NOFOLLOW)) {
		/* not a symlink or should not follow */
		if (!(nd->flags & LOOKUP_RCU)) {
			dput(nd->path.dentry);
			if (nd->path.mnt != path.mnt)
				mntput(nd->path.mnt);
		}
    	//更新path
		nd->path = path;
		nd->inode = inode;
		nd->seq = seq;
		return NULL;
	}
    //处理组件是链接的情况
	return pick_link(nd, &path, inode, seq, flags);
}
```

`pick_link`：

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1692
static int pick_link(struct nameidata *nd, struct path *link,
		     struct inode *inode, unsigned seq)
{
	int error;
	struct saved *last;
	if (unlikely(nd->total_link_count++ >= MAXSYMLINKS)) {
		path_to_nameidata(link, nd);
		return -ELOOP;
	}
	if (!(nd->flags & LOOKUP_RCU)) {
		if (link->mnt == nd->path.mnt)
			mntget(link->mnt);
	}
	error = nd_alloc_stack(nd);
	if (unlikely(error)) {
		if (error == -ECHILD) {
			if (unlikely(!legitimize_path(nd, link, seq))) {
				drop_links(nd);
				nd->depth = 0;
				nd->flags &= ~LOOKUP_RCU;
				nd->path.mnt = NULL;
				nd->path.dentry = NULL;
				if (!(nd->flags & LOOKUP_ROOT))
					nd->root.mnt = NULL;
				rcu_read_unlock();
			} else if (likely(unlazy_walk(nd)) == 0)
				error = nd_alloc_stack(nd);
		}
		if (error) {
			path_put(link);
			return error;
		}
	}

	last = nd->stack + nd->depth++;
	last->link = *link;
	clear_delayed_call(&last->done);
	nd->link_inode = inode;
	last->seq = seq;
	return 1;
}
```

####	path_openat->trailing_symlink

```CPP
static const char *trailing_symlink(struct nameidata *nd)
{
	const char *s;
	int error = may_follow_link(nd);
	if (unlikely(error))
		return ERR_PTR(error);
	nd->flags |= LOOKUP_PARENT;
	nd->stack[0].name = NULL;
	//https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1019
	s = get_link(nd);
	return s ? s : "";
}
```

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1019
static __always_inline
const char *get_link(struct nameidata *nd)
{
	struct saved *last = nd->stack + nd->depth - 1;
	struct dentry *dentry = last->link.dentry;
	struct inode *inode = nd->link_inode;
	int error;
	const char *res;

	if (!(nd->flags & LOOKUP_RCU)) {
		touch_atime(&last->link);
		cond_resched();
	} else if (atime_needs_update_rcu(&last->link, inode)) {
		if (unlikely(unlazy_walk(nd)))
			return ERR_PTR(-ECHILD);
		touch_atime(&last->link);
	}

	error = security_inode_follow_link(dentry, inode,
					   nd->flags & LOOKUP_RCU);
	if (unlikely(error))
		return ERR_PTR(error);

	nd->last_type = LAST_BIND;
	res = inode->i_link;
	if (!res) {
		const char * (*get)(struct dentry *, struct inode *,
				struct delayed_call *);
		get = inode->i_op->get_link;
		if (nd->flags & LOOKUP_RCU) {
			res = get(NULL, inode, &last->done);
			if (res == ERR_PTR(-ECHILD)) {
				if (unlikely(unlazy_walk(nd)))
					return ERR_PTR(-ECHILD);
				res = get(dentry, inode, &last->done);
			}
		} else {
			res = get(dentry, inode, &last->done);
		}
		if (IS_ERR_OR_NULL(res))
			return res;
	}
	if (*res == '/') {
		if (!nd->root.mnt)
			set_root(nd);
		if (unlikely(nd_jump_root(nd)))
			return ERR_PTR(-ECHILD);
		while (unlikely(*++res == '/'))
			;
	}
	if (!*res)
		res = NULL;
	return res;
}
```

##	0x06	do_last的实现
至此基本走读完 `link_path_walk` 函数的实现，这时已经处于路径中的最后一个分量（只是沿着路径走到最终分量所在的目录），该分量可能是：



有可能是个常规的目录 OR 符号链接 OR 根本不存在，接下来调用 `do_last`函数解析最后一个分量，厘清`do_last`的逻辑需要关注：

-	各类标志位flag的检查（打开模式、属性等、部分flag还存在互斥关系）
-	对`..`、挂载点、链接的处理
-	建议通过指定flag进行分析

```cpp
struct file * path_openat(struct nameidata * nd,
	const struct open_flags * op, unsigned flags) {
	......
	while (! (error = link_path_walk(s, nd)) && (error = do_last(nd, file, op, &opened)) > 0) {
		nd->flags &= ~(LOOKUP_OPEN | LOOKUP_CREATE | LOOKUP_EXCL);
		s = trailing_symlink(nd);

		if (IS_ERR(s)) {
			error = PTR_ERR(s);
			break;
		}
	}
	......
}
```

####	path_openat->do_last
`do_last` 方法[实现](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L3259)中，先调用 `lookup_fast`，寻找路径中的最后一个 component，如果成功，就会跳到 `finish_lookup` 对应的 label，然后执行 `step_into` 方法，更新 `nd` 中的 `path`、`inode` 等信息，使其指向目标路径。然后调用 `vfs_open` 方法，继续执行 open 操作

本文内核版本中，`do_last`还是存在较多与`link_path_walk`相同的逻辑，因为最后一个路径分量可能仍然是一个链接或者挂载点，这里只考虑最后一个分量为正常文件的情况，接下来基于几种典型场景分析下`do_last`方法

####	case1：只读打开文件
如`open(pathname, O_RDONLY)`，使用只读方式打开（先查找到然后再打开）一个文件，在`do_last`中的运作路径：

```CPP
static inline int build_open_flags(int flags, umode_t mode, struct open_flags *op)
{
	//非创建文件， mode 设置为 0
	op->mode = 0; 
	// acc_mode：权限检查 MAY_OPEN | MAY_READ
	// 对于目标文件我们至少需要打开和读取的权限
	acc_mode = MAY_OPEN | ACC_MODE(flags);
	op->open_flag = flags;
	// intent 用来标记我们对最终目标想要做什么操作（至少LOOKUP_OPEN）
	op->intent = flags & O_PATH ? 0 : LOOKUP_OPEN;
	op->lookup_flags = lookup_flags;
	return 0;
}
```

只读打开模式下`do_last`主要实现：

```CPP
static int do_last(struct nameidata *nd,
		   struct file *file, const struct open_flags *op,
		   int *opened)
{
	struct dentry *dir = nd->path.dentry;
	int open_flag = op->open_flag;
	bool will_truncate = (open_flag & O_TRUNC) != 0;
	bool got_write = false;
	int acc_mode = op->acc_mode;
	unsigned seq;
	struct inode *inode;
	struct path path;
	int error;

	nd->flags &= ~LOOKUP_PARENT;
	nd->flags |= op->intent;

	......

	if (!(open_flag & O_CREAT)) {
		// 当前非创建文件，进入此流程
		// 判断 last.name 是否以 / 结尾
		if (nd->last.name[nd->last.len])
			nd->flags |= LOOKUP_FOLLOW | LOOKUP_DIRECTORY;
		/* we _can_ be in RCU mode here */
		/*
			三种返回值？：
			-	如果返回 0，则表示成功找到
			-	返回 1，表示内存中没有，需要 lookup_real
			-	返回小于 0，则表示需要从当前 rcu-walk 转到 ref-walk
		*/
		error = lookup_fast(nd, &path, &inode, &seq);
		if (likely(error > 0))
			goto finish_lookup;

		if (error < 0)
			return error;

		......
	} else {
		......
	}

	......
	
	if (open_flag & O_CREAT)
		inode_lock(dir->d_inode);
	else
		inode_lock_shared(dir->d_inode);

	//
	error = lookup_open(nd, &path, file, op, got_write, opened);
	if (open_flag & O_CREAT)
		inode_unlock(dir->d_inode);
	else
		inode_unlock_shared(dir->d_inode);

	if (error <= 0) {
		if (error)
			goto out;

		if ((*opened & FILE_CREATED) ||
		    !S_ISREG(file_inode(file)->i_mode))
			will_truncate = false;

		audit_inode(nd->name, file->f_path.dentry, 0);
		goto opened;
	}

	if (*opened & FILE_CREATED) {
		/* Don't check for write permission, don't truncate */
		open_flag &= ~O_TRUNC;
		will_truncate = false;
		acc_mode = 0;
		path_to_nameidata(&path, nd);
		goto finish_open_created;
	}

	/*
	 * If atomic_open() acquired write access it is dropped now due to
	 * possible mount and symlink following (this might be optimized away if
	 * necessary...)
	 */
	if (got_write) {
		mnt_drop_write(nd->path.mnt);
		got_write = false;
	}

	error = follow_managed(&path, nd);
	if (unlikely(error < 0))
		return error;

	if (unlikely(d_is_negative(path.dentry))) {
		path_to_nameidata(&path, nd);
		return -ENOENT;
	}

	/*
	 * create/update audit record if it already exists.
	 */
	audit_inode(nd->name, path.dentry, 0);

	if (unlikely((open_flag & (O_EXCL | O_CREAT)) == (O_EXCL | O_CREAT))) {
		path_to_nameidata(&path, nd);
		return -EEXIST;
	}

	seq = 0;	/* out of RCU mode, so the value doesn't matter */
	inode = d_backing_inode(path.dentry);
finish_lookup:
	error = step_into(nd, &path, 0, inode, seq);
	if (unlikely(error))
		return error;
finish_open:
	/* Why this, you ask?  _Now_ we might have grown LOOKUP_JUMPED... */
	error = complete_walk(nd);
	if (error)
		return error;
	audit_inode(nd->name, nd->path.dentry, 0);
	error = -EISDIR;
	if ((open_flag & O_CREAT) && d_is_dir(nd->path.dentry))
		goto out;
	error = -ENOTDIR;
	if ((nd->flags & LOOKUP_DIRECTORY) && !d_can_lookup(nd->path.dentry))
		goto out;
	if (!d_is_reg(nd->path.dentry))
		will_truncate = false;

	if (will_truncate) {
		error = mnt_want_write(nd->path.mnt);
		if (error)
			goto out;
		got_write = true;
	}
finish_open_created:
	// 准备好打开文件操作
	// may_open：权限和标志位的检查
	error = may_open(&nd->path, acc_mode, open_flag);
	if (error)
		goto out;
	BUG_ON(*opened & FILE_OPENED); /* once it's opened, it's opened */

	// 核心：终于可以做打开文件的操作了
	error = vfs_open(&nd->path, file, current_cred());
	if (error)
		goto out;
	*opened |= FILE_OPENED;
opened:
	error = open_check_o_direct(file);
	if (!error)
		error = ima_file_check(file, op->acc_mode, *opened);
	if (!error && will_truncate)
		error = handle_truncate(file);
out:
	if (unlikely(error) && (*opened & FILE_OPENED))
		fput(file);
	if (unlikely(error > 0)) {
		WARN_ON(1);
		error = -EINVAL;
	}
	if (got_write)
		mnt_drop_write(nd->path.mnt);
	return error;
}
```

`lookup_open`负责处理路径查找的最终分量（final component）并协调文件创建与打开操作，`lookup_open`的实现机制类似于`lookup_slow`，先使用 `d_lookup` 在内存中找（快速查找），如果未命中就启动 `d_alloc_parallel->` 在具体文件系统里面去找（慢速查找），当它成功返回时会将 `path` 指向找到的目标，`lookup_open`的实现流程描述如下：

1、缓存优先：通过 `d_lookup` 在父目录的哈希链中查找目标 dentry，若命中则跳过后续分配

2、若缓存未命中，则通过[`d_alloc_parallel`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L2405)并行分配 dentry，避免重复创建（避免多线程重复分配同一 dentry）

-	无锁设计：使用 RCU 和等待队列机制同步，仅首个线程执行实际分配，后续线程等待结果
-	状态标记：通过`d_in_lookup(dentry)` 检测 `DCACHE_PAR_LOOKUP` 标志，表示该 dentry 正在通过具体文件系统的 `->lookup()` 方法初始化（如磁盘 I/O）。通过 `DCACHE_PAR_LOOKUP` 标志协调并发初始化，避免全局锁
`d_alloc_parallel` 用于高并发环境下并行分配和查找目录项（dentry），其设计目标是减少锁竞争并优化多线程场景下的性能，在父目录下并行查找或分配指定名称的 dentry，避免多线程重复创建同一 dentry，并确保状态同步

-	若 dentry 已存在：返回现有 dentry，避免重复分配；此外，若其他线程正在初始化同一 dentry，当前线程阻塞直至其完成（触发等待队列机制，通过 `d_wait` 队列唤醒），关联`d_alloc_parallel->d_wait_lookup`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L2475)
-	若不存在：分配新 dentry 并加入**正在查找哈希链（`in_lookup_hash`）**，并将此dentry标记为 `DCACHE_PAR_LOOKUP`，后续线程可等待其初始化完成

3、dentry 验证与重试，需要保证缓存一致性，即通过`d_revalidate` 检查 dentry 是否过期（如被其他进程删除），若失效则调用 `d_invalidate` 将其移出缓存并重试

4、`O_CREAT` 标志处理与权限校验，`O_CREAT` 需检查文件是否存在，但检查与创建需保证原子性；同时检查权限是否满足，`may_o_create` 验证父目录的写权限及文件系统支持，若进程无写权限（`!got_write`），返回 `-EROFS`（只读）

5、文件系统协作模式，优先高性能路径，如支持 `atomic_open` 的文件系统（如 Ext4）可合并查找与打开操作，减少锁竞争；其次再以传统模式进行兜底。如`O_CREAT + O_EXCL` 模式组合通过文件系统钩子（`create/atomic_open`）确保创建的唯一性

6、结果返回与错误处理

-	成功路径：返回 `1` 通知上层调用 `f_op->open`（如 `ext4_file_open`）可以打开文件
-	错误处理：释放 dentry 引用并返回错误码（如 `-ENOENT`、`-EACCES`等）

小结下，`lookup_open`函数通过缓存查找`d_lookup`、并行分配`d_alloc_parallel`以及原子操作 `atomic_open`形成三级加速路径

```CPP
//do_sys_open->do_sys_openat2->do_filp_open->path_openat->do_last->lookup_open
// 主要功能：查找或创建目标文件的 dentry（目录项）
// 这里只考虑传统 lookup + create 流程
static int lookup_open(struct nameidata *nd, struct path *path,
			struct file *file,
			const struct open_flags *op,
			bool got_write, int *opened)
{
	struct dentry *dir = nd->path.dentry;	// 父目录的 dentry
	struct inode *dir_inode = dir->d_inode;
	int open_flag = op->open_flag;
	struct dentry *dentry;
	int error, create_error = 0;
	umode_t mode = op->mode;
	DECLARE_WAIT_QUEUE_HEAD_ONSTACK(wq);

	if (unlikely(IS_DEADDIR(dir_inode)))
		return -ENOENT;
	*opened &= ~FILE_CREATED;
	// fast查找，从缓存中查找dentry
	dentry = d_lookup(dir, &nd->last);
	for (;;) {
		if (!dentry) {
			// 并行分配 dentry
			// d_alloc_parallel 见后文分析
			dentry = d_alloc_parallel(dir, &nd->last, &wq);
			if (IS_ERR(dentry))
				return PTR_ERR(dentry);
		}
		/*
			如果检测到目前是慢速模式
			即当 d_in_lookup(dentry) 返回 true 时
			说明此 dentry 正在被其他线程初始化（例如通过文件系统的 ->lookup() 执行磁盘 I/O）
			此时立即 break 跳出循环，避免重复操作
		*/
		if (d_in_lookup(dentry))	
			break;
		
		//如果非慢速查找
		//验证 dentry 有效性
		error = d_revalidate(dentry, nd->flags);
		if (likely(error > 0))
			break;	//有效则退出循环
		if (error)
			goto out_dput;	//错误处理
		d_invalidate(dentry);	//无效则丢弃并重试
		dput(dentry);
		dentry = NULL;
	}
	if (dentry->d_inode) {
		/* Cached positive dentry: will open in f_op->open */
		goto out_no_open;
	}

	if (open_flag & O_CREAT) {
		if (!IS_POSIXACL(dir->d_inode))
			mode &= ~current_umask();
		if (unlikely(!got_write)) {
			create_error = -EROFS;
			open_flag &= ~O_CREAT;
			if (open_flag & (O_EXCL | O_TRUNC))
				goto no_open;
			/* No side effects, safe to clear O_CREAT */
		} else {
			// 检查创建权限
			create_error = may_o_create(&nd->path, dentry, mode);
			if (create_error) {
				// 权限不足时清除 O_CREAT
				open_flag &= ~O_CREAT;
				// O_EXCL 依赖 O_CREAT
				if (open_flag & O_EXCL)
					goto no_open;
			}
		}
	} else if ((open_flag & (O_TRUNC|O_WRONLY|O_RDWR)) &&
		   unlikely(!got_write)) {
		/*
		 * No O_CREATE -> atomicity not a requirement -> fall
		 * back to lookup + open
		 */
		goto no_open;
	}

	// 原子打开模式（优先）
	if (dir_inode->i_op->atomic_open) {
		error = atomic_open(nd, dentry, path, file, op, open_flag,
				    mode, opened);
		if (unlikely(error == -ENOENT) && create_error)
			error = create_error;
		return error;
	}

no_open:
	// 传统模式（兜底）
	/*
	两步操作：
	1.	先通过 ->lookup 完成磁盘查找，再通过 ->create 创建文件
	2.	状态同步：d_lookup_done 清除 DCACHE_PAR_LOOKUP 标志，表示慢速查找完成
	*/
	if (d_in_lookup(dentry)) {
		// d_in_lookup的作用是什么？
		// 如果没有找到，调用文件系统的lookup 方法进行查找
		struct dentry *res = dir_inode->i_op->lookup(dir_inode, dentry,nd->flags);
		// 由于->lookup已经完成了
		// 清除 DCACHE_PAR_LOOKUP 标志
		d_lookup_done(dentry);
		if (unlikely(res)) {
			if (IS_ERR(res)) {
				error = PTR_ERR(res);
				goto out_dput;
			}
			dput(dentry);
			dentry = res;
		}
	}

	/* Negative dentry, just create the file */
	if (!dentry->d_inode && (open_flag & O_CREAT)) {
		*opened |= FILE_CREATED;
		audit_inode_child(dir_inode, dentry, AUDIT_TYPE_CHILD_CREATE);
		if (!dir_inode->i_op->create) {
			error = -EACCES;
			goto out_dput;
		}
		// 如果没有找到且设置了O_CREAT， 调用文件系统的create方法进行创建
		// 对应ext4文件系统，ext4_create方法
		// 正常情况下，ext4_create会成功创建inode并且和dentry建立关联关系（前文）
		error = dir_inode->i_op->create(dir_inode, dentry, mode,
						open_flag & O_EXCL);
		if (error)
			goto out_dput;
		fsnotify_create(dir_inode, dentry);
	}
	if (unlikely(create_error) && !dentry->d_inode) {
		error = create_error;
		goto out_dput;
	}
out_no_open:
	// 找到了，设置path的值
	path->dentry = dentry;	// 返回最终 dentry
	path->mnt = nd->path.mnt;
	return 1;	// 表示需执行 f_op->open

out_dput:
	dput(dentry);	// 错误时释放 dentry
	return error;
}
```

在`do_last`函数的标签`finish_open_created`中通过调用`vfs_open`最终完成了对文件打开操作，简单分析下基于ext4文件系统的运作过程

首先在`vfs_open->do_dentry_open` 方法，该方法中看到了熟悉的 `inode->i_fop` 成员，该成员的值是在 [`init_special_inode`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/inode.c#L1973) 中设置的，由于这里讨论的文件系统是 `ext4`，所以会命中 `S_ISBLK(mode)` 的逻辑

```CPP
void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev)
{
        inode->i_mode = mode;
        if (S_ISCHR(mode)) {
                inode->i_fop = &def_chr_fops;
                inode->i_rdev = rdev;
        } else if (S_ISBLK(mode)) {
                //https://elixir.bootlin.com/linux/v4.11.6/source/fs/block_dev.c#L2142
                inode->i_fop = &def_blk_fops;
                inode->i_rdev = rdev;
        }
        //...
}
```

当用户调用 `open()` 系统调用时，VFS 层会初始化一个 `struct file` 对象 `f`，`f->f_op` 被赋值为目标文件所属文件系统的 `file_operations` 结构体，此处为 `ext4_file_operations`，因此，执行 `f->f_op->open` 实际调用的是 `ext4_file_open()`。这里体现了 VFS 的协作流程，即 **VFS 通过路径解析找到文件的 dentry 和 inode，根据 inode 关联的文件系统类型（如 `ext4`），将 file 结构 `f->f_op` 绑定到 `ext4_file_operations`，那么执行 `open` 操作时，路由到具体文件系统的实现函数 `ext4_file_open`**

```CPP
// https://elixir.bootlin.com/linux/v4.11.6/source/fs/open.c#L855
int vfs_open(const struct path *path, struct file *file,
	     const struct cred *cred)
{
	struct dentry *dentry = d_real(path->dentry, NULL, file->f_flags);

	if (IS_ERR(dentry))
		return PTR_ERR(dentry);

	file->f_path = *path;
	return do_dentry_open(file, d_backing_inode(dentry), NULL, cred);
}

// fs/open.c
static int do_dentry_open(struct file *f,
                          struct inode *inode,
                          int (*open)(struct inode *, struct file *))
{
        ...
        f->f_inode = inode;
        ...
        f->f_op = fops_get(inode->i_fop);
        ...
        if (!open)
				//ext4_file_open
                open = f->f_op->open;
        if (open) {
                error = open(inode, f);
                ...
        }
        f->f_mode |= FMODE_OPENED;
        ...
        return 0;
        ...
}
```

最终通过[`ext4_file_open`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/file.c#L365) 方法完成了打开文件操作，在 `ext4` 文件系统中，`f->f_op->open` 对应的实际函数是 `ext4_file_open()`，这一关联通过 `ext4` 定义的 [`file_operations`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/file.c#L718) 结构体实现。`ext4_file_open` 最终完成这些工作：

-   初始化文件状态：检查文件是否加密（fscrypt）、是否启用日志（jbd2）等
-   处理大文件标志：若文件超过 `2GB`，设置 `O_LARGEFILE` 标志
-   调用通用逻辑：最终通过 `generic_file_open()` 完成 VFS 层的通用文件打开流程

```CPP
static int ext4_file_open(struct inode * inode, struct file * filp)
```

####	case2：文件创建场景
模式如下，创建一个新文件（`O_CREAT`），并赋予该文件用户可读（`S_IRUSR`）和用户可写（`S_IWUSR`）权限，然后以只写（`O_WRONLY`）的方式打开这个文件。`O_EXCL` 在这里保证该文件必须被创建，如果该文件已经存在则失败返回

-	mode：
-	acc_mode：
-	intent：

```CPP
open(pathname, O_WRONLY | O_CREAT | O_EXCL, S_IRUSR | S_IWUSR)

static inline int build_open_flags(int flags, umode_t mode, struct open_flags *op)
{
	int lookup_flags = 0;
	int acc_mode;

	flags &= VALID_OPEN_FLAGS;

	if (flags & (O_CREAT | __O_TMPFILE))
		//mode：S_IRUSR | S_IWUSR
		op->mode = (mode & S_IALLUGO) | S_IFREG;
	...
	/* Must never be set by userspace */
	flags &= ~FMODE_NONOTIFY & ~O_CLOEXEC;
	...
	// acc_mode：MAY_OPEN | MAY_WRITE
	acc_mode = MAY_OPEN | ACC_MODE(flags);
	op->open_flag = flags;
	...
	op->acc_mode = acc_mode;

	//intent：LOOKUP_OPEN
	op->intent = flags & O_PATH ? 0 : LOOKUP_OPEN;

	if (flags & O_CREAT) {
		op->intent |= LOOKUP_CREATE;
		if (flags & O_EXCL)
			op->intent |= LOOKUP_EXCL;
	}
	...
	return 0;
}
```

对应的`do_last`核心实现：

```CPP
static int do_last(struct nameidata *nd,
		   struct file *file, const struct open_flags *op,
		   int *opened)
{
	struct dentry *dir = nd->path.dentry;
	int open_flag = op->open_flag;
	bool will_truncate = (open_flag & O_TRUNC) != 0;
	bool got_write = false;
	int acc_mode = op->acc_mode;
	unsigned seq;
	struct inode *inode;
	struct path path;
	int error;

	......

	if (!(open_flag & O_CREAT)) {
		......
	} else {
		// O_CREAT模式
		//complete_walk：告别 rcu-walk 模式
		error = complete_walk(nd);
		if (error)
			return error;

		audit_inode(nd->name, dir, LOOKUP_PARENT);
		/* trailing slashes? */
		// 判断这个最终目标是不是以 / 结尾
		// 如果是的话就表示最终目标是一个目录那就返回 EISDIR
		// 并报错 Is a directory
		if (unlikely(nd->last.name[nd->last.len]))
			return -EISDIR;
	}

	if (open_flag & (O_CREAT | O_TRUNC | O_WRONLY | O_RDWR)) {
		// 尝试取得当前文件系统的写权限（有可能写入）
		// 最终会让lookup_open决定
		error = mnt_want_write(nd->path.mnt);
		if (!error)
			got_write = true;	//不退出，先打个标记
		/*
		 * do _not_ fail yet - we might not need that or fail with
		 * a different error; let lookup_open() decide; we'll be
		 * dropping this one anyway.
		 */
	}
	if (open_flag & O_CREAT)
		inode_lock(dir->d_inode);
	else
		inode_lock_shared(dir->d_inode);
	// lookup_open：核心处理逻辑
	error = lookup_open(nd, &path, file, op, got_write, opened);
	if (open_flag & O_CREAT)
		inode_unlock(dir->d_inode);
	else
		inode_unlock_shared(dir->d_inode);

	if (error <= 0) {
		if (error)
			goto out;

		if ((*opened & FILE_CREATED) ||
		    !S_ISREG(file_inode(file)->i_mode))
			will_truncate = false;

		audit_inode(nd->name, file->f_path.dentry, 0);
		goto opened;
	}

	if (*opened & FILE_CREATED) {
		/* Don't check for write permission, don't truncate */
		// 如果这个文件就是本次新建的
		// 那么就要清除 O_TRUNC 位
		open_flag &= ~O_TRUNC;
		will_truncate = false;
		// 已经成功的创建了新文件
		// 那么写权限也没必要检查了
		acc_mode = 0;
		path_to_nameidata(&path, nd);

		//直接跳转到标号 finish_open_created 处完成打开
		goto finish_open_created;
	}

	/*
	 * If atomic_open() acquired write access it is dropped now due to
	 * possible mount and symlink following (this might be optimized away if
	 * necessary...)
	 */
	if (got_write) {
		mnt_drop_write(nd->path.mnt);
		got_write = false;
	}

	error = follow_managed(&path, nd);
	if (unlikely(error < 0))
		return error;

	if (unlikely(d_is_negative(path.dentry))) {
		path_to_nameidata(&path, nd);
		return -ENOENT;
	}

	/*
	 * create/update audit record if it already exists.
	 */
	audit_inode(nd->name, path.dentry, 0);

	if (unlikely((open_flag & (O_EXCL | O_CREAT)) == (O_EXCL | O_CREAT))) {
		// 如果这个文件本来就存在
		// 检查 O_EXCL 和 O_CREAT 这两个标志位
		// 因为 O_EXCL 标志位要求必须创建成功
		// 所以这里就会返回File exists”错误
		path_to_nameidata(&path, nd);
		return -EEXIST;
	}

	seq = 0;	/* out of RCU mode, so the value doesn't matter */
	inode = d_backing_inode(path.dentry);
finish_lookup:
	error = step_into(nd, &path, 0, inode, seq);
	if (unlikely(error))
		return error;
finish_open:
	/* Why this, you ask?  _Now_ we might have grown LOOKUP_JUMPED... */
	error = complete_walk(nd);
	if (error)
		return error;
	audit_inode(nd->name, nd->path.dentry, 0);
	error = -EISDIR;
	if ((open_flag & O_CREAT) && d_is_dir(nd->path.dentry))
		goto out;
	error = -ENOTDIR;
	if ((nd->flags & LOOKUP_DIRECTORY) && !d_can_lookup(nd->path.dentry))
		goto out;
	if (!d_is_reg(nd->path.dentry))
		will_truncate = false;

	if (will_truncate) {
		error = mnt_want_write(nd->path.mnt);
		if (error)
			goto out;
		got_write = true;
	}
finish_open_created:
	// 最后 may_open 检查相应的权限
	// 然后 vfs_open 完成打开
	error = may_open(&nd->path, acc_mode, open_flag);
	if (error)
		goto out;
	......
	error = vfs_open(&nd->path, file, current_cred());
	if (error)
		goto out;
	*opened |= FILE_OPENED;
opened:
	error = open_check_o_direct(file);
	if (!error)
		error = ima_file_check(file, op->acc_mode, *opened);
	if (!error && will_truncate)
		error = handle_truncate(file);
out:
	if (unlikely(error) && (*opened & FILE_OPENED))
		fput(file);
	if (unlikely(error > 0)) {
		WARN_ON(1);
		error = -EINVAL;
	}
	if (got_write)
		mnt_drop_write(nd->path.mnt);
	return error;
}
```

```CPP
static int lookup_open(struct nameidata *nd, struct path *path,
			struct file *file,
			const struct open_flags *op,
			bool got_write, int *opened)
{
	struct dentry *dir = nd->path.dentry;
	struct inode *dir_inode = dir->d_inode;
	int open_flag = op->open_flag;
	struct dentry *dentry;
	int error, create_error = 0;
	umode_t mode = op->mode;

	if (unlikely(IS_DEADDIR(dir_inode)))
		return -ENOENT;
	
	//先清除改变量的 FILE_CREATED 位
	//如果该文件不存在的话会被重新赋上 FILE_CREATED，表示文件是这次创建的
	*opened &= ~FILE_CREATED;
	dentry = d_lookup(dir, &nd->last);
	for (;;) {
		if (!dentry) {
			dentry = d_alloc_parallel(dir, &nd->last, &wq);
			if (IS_ERR(dentry))
				return PTR_ERR(dentry);
		}
		if (d_in_lookup(dentry))
			break;

		error = d_revalidate(dentry, nd->flags);
		if (likely(error > 0))
			break;
		if (error)
			goto out_dput;
		d_invalidate(dentry);
		dput(dentry);
		dentry = NULL;
	}
	if (dentry->d_inode) {
		/* Cached positive dentry: will open in f_op->open */
		goto out_no_open;
	}


	if (open_flag & O_CREAT) {
		if (!IS_POSIXACL(dir->d_inode))
			mode &= ~current_umask();
		if (unlikely(!got_write)) {
			create_error = -EROFS;
			open_flag &= ~O_CREAT;
			if (open_flag & (O_EXCL | O_TRUNC))
				goto no_open;
			/* No side effects, safe to clear O_CREAT */
		} else {
			create_error = may_o_create(&nd->path, dentry, mode);
			if (create_error) {
				open_flag &= ~O_CREAT;
				if (open_flag & O_EXCL)
					goto no_open;
			}
		}
	} else if ((open_flag & (O_TRUNC|O_WRONLY|O_RDWR)) &&
		   unlikely(!got_write)) {
		/*
		 * No O_CREATE -> atomicity not a requirement -> fall
		 * back to lookup + open
		 */
		goto no_open;
	}

	if (dir_inode->i_op->atomic_open) {
		error = atomic_open(nd, dentry, path, file, op, open_flag,mode, opened);
		if (unlikely(error == -ENOENT) && create_error)
			error = create_error;
		return error;
	}

no_open:
	if (d_in_lookup(dentry)) {
		struct dentry *res = dir_inode->i_op->lookup(dir_inode, dentry,nd->flags);
		d_lookup_done(dentry);
		if (unlikely(res)) {
			if (IS_ERR(res)) {
				error = PTR_ERR(res);
				goto out_dput;
			}
			dput(dentry);
			dentry = res;
		}
	}

	/* Negative dentry, just create the file */
	if (!dentry->d_inode && (open_flag & O_CREAT)) {
		// 重新补入FILE_CREATED位
		*opened |= FILE_CREATED;
		audit_inode_child(dir_inode, dentry, AUDIT_TYPE_CHILD_CREATE);
		if (!dir_inode->i_op->create) {
			error = -EACCES;
			goto out_dput;
		}
		// struct dentry * dir = nd->path.dentry; 指的是当前要创建 dentry 的父目录的 dentry
        // 所以这里是调用父目录 dentry 的 inode_operations.create
		// create 会调用文件系统的 inode_operations.create函数真正创建这个文件
		error = dir_inode->i_op->create(dir_inode, dentry, mode,
						open_flag & O_EXCL);
		if (error)
			goto out_dput;
		fsnotify_create(dir_inode, dentry);
	}
	if (unlikely(create_error) && !dentry->d_inode) {
		error = create_error;
		goto out_dput;
	}
out_no_open:
	path->dentry = dentry;
	path->mnt = nd->path.mnt;
	return 1;

out_dput:
	dput(dentry);
	return error;
}
```

![do_last]()

##	0x07	附录

####	小结：do_filp_open中的查找模式
`do_filp_open`对`path_openat`的调用模式与`path_openat`内部的`lookup_fast/lookup_slow`组合，这些策略的设计目的是在效率（无锁RCU模式）和兼容性（带锁的ref-walk模式）之间动态平衡，实际组合共`4`种场景（RCU独立成功、RCU回退、标准模式、REVAL模式），核心策略如下：

1、RCU模式（`LOOKUP_RCU`）

-	触发条件：`do_filp_open`首次调用`path_openat`时设置`LOOKUP_RCU | LOOKUP_FOLLOW`
-	核心流程：通过`lookup_fast->__d_lookup_rcu`在无锁状态下查询dcache，若dcache命中且通过RCU校验（如`read_seqcount_retry`），直接返回成功
-	失败场景：dcache未命中或校验失败，返回`-ECHILD`，触发回退至ref-walk模式

2、RCU回退至ref-walk模式

-	触发条件：RCU模式返回`-ECHILD`后，`do_filp_open`重新调用`path_openat`（无`LOOKUP_RCU`标志）
-	核心流程
	-	Step 1：`lookup_fast`（ref-walk版），在dcache中带锁查找（`lookup_fast->__d_lookup`），若成功则更新`nameidata`
	-	Step 2：`lookup_slow`（文件系统级查找），若`lookup_fast`失败，调用具体文件系统的`->lookup`钩子（如`ext4_lookup`）从磁盘加载inode

3、标准ref-walk模式

-	触发条件：RCU模式未启用或已回退后
-	核心流程：同RCU回退模式（`lookup_fast->lookup_slow`），但无RCU前置尝试
-	典型场景：非首次查找、或进程已持有锁无法使用RCU

4、 REVAL模式（`LOOKUP_REVAL`）

-	触发条件：标准模式返回`-ESTALE`（inode可能过期），在`lookup_slow`中强制重新验证inode有效性（如NFS场景）
-	核心流程：与标准ref-walk模式一致，但增加有效性校验逻辑

####	handle_dots
TODO

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1679
static inline int handle_dots(struct nameidata *nd, int type)
{
	if (type == LAST_DOTDOT) {
		if (!nd->root.mnt)
			set_root(nd);
		if (nd->flags & LOOKUP_RCU) {
			// RCU
			return follow_dotdot_rcu(nd);
		} else
			return follow_dotdot(nd);
	}
	return 0;
}
```

####	operations 成员
针对ext4系统，相应注册的inode实例化方法[如下](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/namei.c#L3903)

```CPP
const struct inode_operations ext4_dir_inode_operations = {
	.create		= ext4_create,
	.lookup		= ext4_lookup,
	.link		= ext4_link,
	.unlink		= ext4_unlink,
	.symlink	= ext4_symlink,
	.mkdir		= ext4_mkdir,
	.rmdir		= ext4_rmdir,
	.mknod		= ext4_mknod,
	.tmpfile	= ext4_tmpfile,
	.rename		= ext4_rename2,
	.setattr	= ext4_setattr,
	.getattr	= ext4_getattr,
	.listxattr	= ext4_listxattr,
	.get_acl	= ext4_get_acl,
	.set_acl	= ext4_set_acl,
	.fiemap     = ext4_fiemap,
};
```

以inode的创建函数`ext4_create`为例，核心调用关系如下：

```CPP
ext4_create
  --ext4_new_inode_start_handle
  --ext4_add_nondir
	--ext4_add_nondir
		--d_instantiate
```

-	`ext4_new_inode_start_handle`：为新创建的文件分配一个inode结构，接着为该文件找一个有空闲inode和空闲block的块组group，然后在该块组的inode bitmap找一个空闲inode编号，最后把该inode编号赋值给`inode->i_ino`
-	`ext4_add_nondir`： 把dentry和inode对应的文件或目录添加到它父目录的`ext4_dir_entry_2`结构里

绑定dentry与inode的方法[`__d_set_inode_and_type`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L280)：

```CPP
static inline void __d_set_inode_and_type(struct dentry *dentry,
					  struct inode *inode,
					  unsigned type_flags)
{
	unsigned flags;

	dentry->d_inode = inode;
	flags = READ_ONCE(dentry->d_flags);
	flags &= ~(DCACHE_ENTRY_TYPE | DCACHE_FALLTHRU);
	flags |= type_flags;
	WRITE_ONCE(dentry->d_flags, flags);
}
```

####    对 dcache 的操作（lookup_slow）
通过上面可以知道对路径的查找过程，也是对 dcache 树的不断的追加、修改过程（即对于不存在于 dcache 的 dentry 节点，需要先通过 filename 找到其文件系统磁盘的 inode，然后建立内存 inode 及 dentry 结构，然后建立内存 dentry 到 inode 的关系），对于 `lookup_slow` 逻辑会涉及到如下操作 dcache 的逻辑：

-   分配 dentry，从 slab 分配器分配内存，初始化 dentry 对象
-   绑定 inode，关联 dentry 与 inode，并加入 dcache 树
-   处理别名，解决硬链接冲突
-   链接到父目录，将 dentry 加入父目录的 `d_subdirs` 链表

1、[`d_alloc_parallel->d_alloc->__d_alloc`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L2405)

```CPP
struct dentry *d_alloc(struct dentry * parent, const struct qstr *name)
{
	struct dentry *dentry = __d_alloc(parent->d_sb, name);
	if (!dentry)
		return NULL;
	dentry->d_flags |= DCACHE_RCUACCESS;
	spin_lock(&parent->d_lock);
	/*
	 * don't need child lock because it is not subject
	 * to concurrency here
	 */
	__dget_dlock(parent);
	dentry->d_parent = parent;  // 将新建节点的 dentry->d_parent 设置为 parent
	list_add(&dentry->d_child, &parent->d_subdirs); // 将 dentry 加入父目录的 d_subdirs 链表
	spin_unlock(&parent->d_lock);

	return dentry;
}

struct dentry *__d_alloc(struct super_block *sb, const struct qstr *name)
{
	struct dentry *dentry;
	char *dname;
	int err;

	dentry = kmem_cache_alloc(dentry_cache, GFP_KERNEL);
	if (!dentry)
		return NULL;

	dentry->d_iname[DNAME_INLINE_LEN-1] = 0;
	if (unlikely(!name)) {
		static const struct qstr anon = QSTR_INIT("/", 1);
		name = &anon;
		dname = dentry->d_iname;
	} else if (name->len > DNAME_INLINE_LEN-1) {
		size_t size = offsetof(struct external_name, name[1]);
		struct external_name *p = kmalloc(size + name->len,
						  GFP_KERNEL_ACCOUNT);
		if (!p) {
			kmem_cache_free(dentry_cache, dentry);
			return NULL;
		}
		atomic_set(&p->u.count, 1);
		dname = p->name;
		if (IS_ENABLED(CONFIG_DCACHE_WORD_ACCESS))
			kasan_unpoison_shadow(dname,
				round_up(name->len + 1,	sizeof(unsigned long)));
	} else  {
		dname = dentry->d_iname;
	}

	dentry->d_name.len = name->len;
	dentry->d_name.hash = name->hash;
	memcpy(dname, name->name, name->len);
	dname[name->len] = 0;

	/* Make sure we always see the terminating NUL character */
	smp_wmb();
	dentry->d_name.name = dname;

	dentry->d_lockref.count = 1;
	dentry->d_flags = 0;
	spin_lock_init(&dentry->d_lock);
	seqcount_init(&dentry->d_seq);
	dentry->d_inode = NULL;
	dentry->d_parent = dentry;
	dentry->d_sb = sb;
	dentry->d_op = NULL;
	dentry->d_fsdata = NULL;
	INIT_HLIST_BL_NODE(&dentry->d_hash);
	INIT_LIST_HEAD(&dentry->d_lru);
	INIT_LIST_HEAD(&dentry->d_subdirs);
	INIT_HLIST_NODE(&dentry->d_u.d_alias);
	INIT_LIST_HEAD(&dentry->d_child);
	d_set_d_op(dentry, dentry->d_sb->s_d_op);

	if (dentry->d_op && dentry->d_op->d_init) {
		err = dentry->d_op->d_init(dentry);
		if (err) {
			if (dname_external(dentry))
				kfree(external_name(dentry));
			kmem_cache_free(dentry_cache, dentry);
			return NULL;
		}
	}

	this_cpu_inc(nr_dentry);

	return dentry;
}
```

2、`d_splice_alias->__d_instantiate->__d_set_inode_and_type`：将 dentry 和 inode 绑定，即将 inode 赋值给 dentry 的 `d_inode` 成员

```CPP
void d_instantiate(struct dentry *entry, struct inode * inode)
{
	BUG_ON(!hlist_unhashed(&entry->d_u.d_alias));
	if (inode) {
		security_d_instantiate(entry, inode);
		spin_lock(&inode->i_lock);
		__d_instantiate(entry, inode);
		spin_unlock(&inode->i_lock);
	}
}

static void __d_instantiate(struct dentry *dentry, struct inode *inode)
{
	unsigned add_flags = d_flags_for_inode(inode);
	WARN_ON(d_in_lookup(dentry));

	spin_lock(&dentry->d_lock);
	hlist_add_head(&dentry->d_u.d_alias, &inode->i_dentry);
	raw_write_seqcount_begin(&dentry->d_seq);
	__d_set_inode_and_type(dentry, inode, add_flags);
	raw_write_seqcount_end(&dentry->d_seq);
	fsnotify_update_flags(dentry);
	spin_unlock(&dentry->d_lock);
}

static inline void __d_set_inode_and_type(struct dentry *dentry,
					  struct inode *inode,
					  unsigned type_flags)
{
	unsigned flags;

    // 设置 dentry 与 inode 的绑定关系
	dentry->d_inode = inode;
	flags = READ_ONCE(dentry->d_flags);
	flags &= ~(DCACHE_ENTRY_TYPE | DCACHE_FALLTHRU);
	flags |= type_flags;
	WRITE_ONCE(dentry->d_flags, flags);
}
```

####	`d_alloc_parallel`的实现
`d_alloc_parallel` 函设计核心是通过无锁并发机制优化目录项（dentry）的分配与查找，完全在内存中操作，属于快速路径（fast path）的一部分

-	并行查找：通过 RCU（Read-Copy-Update）机制无锁遍历哈希桶，直接检查目标 dentry 是否已存在于父目录的哈希链表中
-	条件分配：若 dentry 不存在，则分配一个新 dentry 并尝试将其插入哈希链；若已存在，则直接返回现有 dentry（通过 `res` 参数返回）
-	无磁盘 I/O：整个过程仅操作内存中的 dentry 缓存，不触发文件系统底层的磁盘读取或复杂逻辑（如加载 inode）

####	小结：path walk的策略
前文描述了内核实现path walk的策略，首先 Kernel 会在 rcu-walk 模式下进入 lookup_fast 进行尝试，如果失败了那么就尝试就地转入 ref-walk，如果还是不行就回到 do_filp_open 从头开始。Kernel 在 ref-walk 模式下会首先在内存缓冲区查找相应的目标（lookup_fast），如果找不到就启动具体文件系统自己的 lookup 进行查找（lookup_slow）。注意，在 rcu-walk 模式下是不会进入 lookup_slow 的。如果这样都还找不到的话就一定是是出错了，那就报错返回吧，这时屏幕就会出现 No such file or directory

##  0x08 参考
-   [open 系统调用（一）](https://www.kerneltravel.net/blog/2021/open_syscall_szp1/)
-   [走马观花： Linux 系统调用 open 七日游](http://blog.chinaunix.net/uid-20522771-id-4419678.html)
-   [Open() 函数的内核追踪](https://blog.csdn.net/blue95wind/article/details/7472350)
-   [Linux Kernel 文件系统写 I/O 流程代码分析（二）](https://www.cnblogs.com/jimbo17/p/10491223.html)
-   [linux 内核 open 文件流程](https://zhuanlan.zhihu.com/p/471175983)
-   [文件系统缓存](https://www.laumy.tech/776.html)
-   [Linux 的 VFS 实现 - 番外 [一] - dcache](https://zhuanlan.zhihu.com/p/261669249)
-   [Linux Kernel 文件系统写 I/O 流程代码分析（一）](https://www.cnblogs.com/jimbo17/p/10436222.html)
-   [vfs dentry cache 模块实现分析](https://zhuanlan.zhihu.com/p/457005511)
-	[open 系统调用实现](https://xinqiu.gitbooks.io/linux-insides-cn/content/SysCall/linux-syscall-5.html)
-	[open(2) — Linux manual page](https://man7.org/linux/man-pages/man2/open.2.html)
-	[Linux Open系统调用 篇二](https://juejin.cn/post/6844903926735568904)
-	[Linux Open系统调用 篇三](https://juejin.cn/post/6844903937032585230)
-	[Linux Open系统调用 篇四](https://juejin.cn/post/6844903937036779533)
-	[