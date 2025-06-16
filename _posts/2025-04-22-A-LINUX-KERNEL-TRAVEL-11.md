---
layout:     post
title:  Linux 内核之旅（十一）：追踪 open/write 系统调用
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

用户态程序调用 `open` 函数时，会产生一个中断号为 `5` 的中断请求，其值以该宏 `__NR__open` 进行标示，而后该进程上下文将会被切换到内核空间，待内核相关操作完成后，就会从内核返回至用户态，此时还需要一次进程上下文切换，本文就以内核视角追踪下 `open`（`write`）的内核调用过程

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

####    dentry cache
一个 `struct dentry` 结构代表文件系统中的一个目录或文件，VFS dentry 结构的意义，即需要建立文件名 filename 到 inode 的映射关系，目的为了提高系统调用在操作、查找目录 / 文件操作场景下的效率，且 dentry 是仅存在于内存的数据结构，所以内核使用了 `dentry_hashtable`（dentry 树）来管理整个系统的目录树结构。在 Linux 可通过下面方式查看 dentry cache 的指标：

```BASH
[root@VM-X-X-tencentos ~]# cat /proc/sys/fs/dentry-state
1212279 1174785 45      0       598756  0
```

前文已经描述过 [`dentry_hashtable`]() 数据结构的意义，即为了提高目录，内核使用 `dentry_hashtable` 对 dentry 进行管理，在 `open` 等系统调用进行路径查找时，用于快速定位某个目录 dentry 下面的子 dentry（哈希表的搜索效率显然更好）


1、`dentry` 及 `dentry_hashtable` 的结构（搜索 & 回收）

对 `dentry` 而言，对于加速搜索关联了两个关键数据结构 `d_hash` 及 `d_lru`，dentry 回收关联的重要成员是 `d_lockref`，`d_lockref` 内嵌一个自旋锁（spinlock_t），用于保护 dentry 结构的并发修改
```CPP
struct dentry {
    /* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */

    struct hlist_bl_node d_hash;       //hashtable 的节点
    strcut list_head d_lru;     //LRU 链上的节点
}
```

![dcache]()

2、`dentry_hashtable` 的创建过程

在文件系统初始化时，调用 `vfs_caches_init->dcache_init` 为 dcache 进行初始化，先创建一个 dentry 的 slab，用于后续 dentry 对象的分配，同时还初始化了 `dentry_hashtable` 这个用于管理 dentry 的全局 hashtable

```CPP
static struct hlist_head *dentry_hashtable __read_mostly;

static void __init dcache_init(void)
{
    /*
     * A constructor could be added for stable state like the lists,
     * but it is probably not worth it because of the cache nature
     * of the dcache.
     */
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
                    dhash_entries,
                    13,
                    HASH_ZERO,
                    &d_hash_shift,
                    NULL,
                    0,
                    0);
    d_hash_shift = 32 - d_hash_shift;
}
```

##  0x03 追踪 Open 流程
一个进程需要读 / 写一个文件，必须先通过 filename 建立和文件 inode 之间的通道，方式是通过 `open()` 函数，该函数的参数是文件所在的路径名 `pathname`，如何根据 `pathname` 找到对应的 inode？这就要依靠 dentry 结构了

```CPP
int open (const char *pathname, int flags, mode_t mode);
int openat(int dirfd, const char *pathname, int flags, mode_t mode);
```

####   nameidata/path/vfsmount 结构
[`nameidata`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L506)，`nameidata` 用来存储遍历路径的中间结果（临时性存放），在路径搜索时常用到
这个结构体在路径查找中非常重要，它记录了查找信息、保存了查找起始路径。在路径 `/a/b/c/d` 的每一个分量的查找中，它会保存当前的结果。对于一般路径名查找，在查找结束时，它会包含查询结果的信息；对于父路径名查找，在查找结束时，它会包含最后一个分量所在目录的信息。最重要的成员是 `nameidata.path`（思考下在 VFS 中只有 `path` 才能唯一标识一个路径）

```CPP
struct nameidata {
	struct path	path;   //path 保存当前搜索到的路径（包含了 vfsmount 及在该 mount 下的 dentry）
	struct qstr	last;   //last 保存当前子路径名及其散列值
	struct path	root;   //root 用来保存根目录的信息
	struct inode	*inode; /* path.dentry.d_inode */
    // inode 指向当前找到的目录项的 inode 结构

	unsigned int	flags;  //flags 是一些和查找（lookup）相关的标志位
    unsigned	seq, m_seq;
	int		last_type;  //last_type 表示当前节点类型
	unsigned	depth;  // depth 用来记录在解析符号链接过程中的递归深度
	int		total_link_count;
    //...

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

####   核心流程
1、[`do_sys_open`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/open.c#L1036)

```cpp
long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
	struct open_flags op;
	int fd = build_open_flags(flags, mode, &op);
	struct filename *tmp;

	if (fd)
		return fd;

	tmp = getname(filename);
	if (IS_ERR(tmp))
		return PTR_ERR(tmp);

    // 获取一个空闲的文件描述符
	fd = get_unused_fd_flags(flags);
	if (fd>= 0) {
        // 调用 do_filp_open() 函数打开文件，返回打开文件的 struct file 结构
		struct file *f = do_filp_open(dfd, tmp, &op);
		if (IS_ERR(f)) {
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

2、[`do_filp_open`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L3515)

```CPP
//pathname 是 open file 的完整路径名
struct file *do_filp_open(int dfd, struct filename *pathname,
		const struct open_flags *op)
{
	struct nameidata nd;
	int flags = op->lookup_flags;
	struct file *filp;

    // 将 pathname 保存到 nameidata 里
	set_nameidata(&nd, dfd, pathname);
    // 调用 path_openat，此时是 flags 是带了 LOOKUP_RCU flag
	filp = path_openat(&nd, op, flags | LOOKUP_RCU);
	if (unlikely(filp == ERR_PTR(-ECHILD)))
		filp = path_openat(&nd, op, flags);
	if (unlikely(filp == ERR_PTR(-ESTALE)))
		filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
	restore_nameidata();
	return filp;
}
```

3、[`path_openat`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L3457)，在 `path_openat` 中，先调用 `get_empty_filp` 方法分配一个空的 `struct file` 实例，再调用 `path_init`、`link_path_walk`、`do_last` 等方法执行后续的 open 操作，如果都成功了，则返回 `struct file` 给上层。核心方法是 `path_init`、`link_path_walk`、`do_last`，其中 `path_init` 和 `link_path_walk` 通常合在一起调用，作用是 **可以根据给定的文件路径名称在内存中找到或者建立代表着目标文件或者目录的 dentry 结构和 inode 结构**

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

	if (unlikely(file->f_flags & __O_TMPFILE)) {
		error = do_tmpfile(nd, flags, op, file, &opened);
		goto out2;
	}

	if (unlikely(file->f_flags & O_PATH)) {
		error = do_o_path(nd, flags, file);
		if (!error)
			opened |= FILE_OPENED;
		goto out2;
	}

    // 路径初始化，确定查找的起始目录，初始化结构体 nameidata 的成员 path
    // 调用 path_init() 设置 nameidata 的 path 结构体
    // 对于常规文件来说，如 / data/test/testfile，设置 path 结构体指向根目录 /，即设置 path.mnt/path.dentry 指向根目录 /，为后续的目录解析做准备
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

4、`path_init` 方法，`path_init` 方法主要是用来初始化 `struct nameidata` 实例中的 `path`、`root`、`inode` 等字段。当 `path_init` 函数执行成功后，就会在 `nameidata` 结构体的成员 `nd->path.dentry` 中指向搜索路径的起点，接下来就使用 `link_path_walk` 函数顺着路径进行搜索

```CPP
static const char *path_init(struct nameidata *nd, unsigned flags)
{
	int retval = 0;
	const char *s = nd->name->name;

	if (!*s)
		flags &= ~LOOKUP_RCU;

	nd->last_type = LAST_ROOT; /* if there are only slashes... */
	nd->flags = flags | LOOKUP_JUMPED | LOOKUP_PARENT;
	nd->depth = 0;
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
		if (flags & LOOKUP_RCU) {
			rcu_read_lock();
			nd->seq = __read_seqcount_begin(&nd->path.dentry->d_seq);
			nd->root_seq = nd->seq;
			nd->m_seq = read_seqbegin(&mount_lock);
		} else {
			path_get(&nd->path);
		}
		return s;
	}

	nd->root.mnt = NULL;
	nd->path.mnt = NULL;
	nd->path.dentry = NULL;

	nd->m_seq = read_seqbegin(&mount_lock);
	if (*s == '/') {
		if (flags & LOOKUP_RCU)
			rcu_read_lock();
		set_root(nd);
		if (likely(!nd_jump_root(nd)))
			return s;
		nd->root.mnt = NULL;
		rcu_read_unlock();
		return ERR_PTR(-ECHILD);
	} else if (nd->dfd == AT_FDCWD) {
		if (flags & LOOKUP_RCU) {
			struct fs_struct *fs = current->fs;
			unsigned seq;

			rcu_read_lock();

			do {
				seq = read_seqcount_begin(&fs->seq);
				nd->path = fs->pwd;
				nd->inode = nd->path.dentry->d_inode;
				nd->seq = __read_seqcount_begin(&nd->path.dentry->d_seq);
			} while (read_seqcount_retry(&fs->seq, seq));
		} else {
			get_fs_pwd(current->fs, &nd->path);
			nd->inode = nd->path.dentry->d_inode;
		}
		return s;
	} else {
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
```

5、[`link_path_walk`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L2042)

```CPP
static int link_path_walk(const char *name, struct nameidata *nd)
{
	int err;

    // 跳过开始的 / 字符（根目录）
	while (*name=='/')
		name++;

    // 如果路径只包含 /，搜索完成，返回
	if (!*name)
		return 0;

	/* At this point we know we have a real path component. */
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
			struct dentry *parent = nd->path.dentry;
			nd->flags &= ~LOOKUP_JUMPED;
			if (unlikely(parent->d_flags & DCACHE_OP_HASH)) {
                // 当前目录项需要重新计算一下 hash 值
				struct qstr this = {{ .hash_len = hash_len}, .name = name };
                // 调用 parent 这个 dentry 的 parent->d_op->d_hash 方法计算 hash 值
				err = parent->d_op->d_hash(parent, &this);
				if (err < 0)
					return err;
				hash_len = this.hash_len;
				name = this.name;
			}
		}

        // 更新 nameidata last 结构体
		nd->last.hash_len = hash_len;
		nd->last.name = name;
		nd->last_type = type;
        // 这里使 name 指向下一级目录
		name += hashlen_len(hash_len);
		if (!*name)
			goto OK;
		/*
		 * If it wasn't NUL, we know it was'/'. Skip that
		 * slash, and continue until no more slashes.
		 */
		do {
			name++;
		} while (unlikely(*name == '/'));
		if (unlikely(!*name)) {
            // 假设 open file 文件名路径上没有任何 symlink，则如果这个条件满足，说明整个路径都解析完了，还剩最后的 filename 留给 do_last() 解析，此函数将从下面的! nd->depth 条件处返回
OK:
			// pathname body, done
			if (!nd->depth)
                // 此时已经到达了最终目标，路径行走任务完成
                 // 如果 open file 完整路径上没有任何 symlink，nd->depth 等于 0
				return 0;
			name = nd->stack[nd->depth - 1].name;
			/* trailing symlink, done */
			if (!name)
                // 此时已经到达了最终目标，路径行走任务完成
				return 0;
			/* last component of nested symlink */
            //symlink case
			err = walk_component(nd, WALK_FOLLOW);
		} else {
			/* not the last component */
            // 常规目录 case，非 symlink 的 case
			err = walk_component(nd, WALK_FOLLOW | WALK_MORE);
		}
		if (err < 0)
			return err;

		if (err) {
			const char *s = get_link(nd);

			if (IS_ERR(s))
				return PTR_ERR(s);
			err = 0;
			if (unlikely(!s)) {
				/* jumped */
				put_link(nd);
			} else {
				nd->stack[nd->depth - 1].name = name;
				name = s;
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

`link_path_walk` 函数中会调用 `walk_component` 函数来进行路径搜索（向上或者向下），这个是较为复杂的逻辑，有很多场景，典型的如：

-   `.` 或 `..`
-   普通的目录，这里又会检测当前目录是否为其他文件系统的挂载点
-   符号链接

6、`do_last` 方法，先调用 `lookup_fast`，寻找路径中的最后一个 component，如果成功，就会跳到 `finish_lookup` 对应的 label，然后执行 `step_into` 方法，更新 `nd` 中的 `path`、`inode` 等信息，使其指向目标路径。然后调用 `vfs_open` 方法，继续执行 open 操作

```CPP
// fs/namei.c
static int do_last(struct nameidata *nd,
                   struct file *file, const struct open_flags *op)
{
        //...
        if (!(open_flag & O_CREAT)) {
                //...
                error = lookup_fast(nd, &path, &inode, &seq);
                if (likely(error> 0))
                        goto finish_lookup;
                //...
        } else {
                //...
        }
        //...

		error = lookup_open(nd, &path, file, op, got_write, opened);

		//...

finish_lookup:
        error = step_into(nd, &path, 0, inode, seq);
        //...
        error = vfs_open(&nd->path, file);
        //...
        return error;
}
```

7、`lookup_open`

```CPP
//do_sys_open->do_sys_openat2->do_filp_open->path_openat->do_last->lookup_open
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
	DECLARE_WAIT_QUEUE_HEAD_ONSTACK(wq);

	if (unlikely(IS_DEADDIR(dir_inode)))
		return -ENOENT;

	*opened &= ~FILE_CREATED;
	dentry = d_lookup(dir, &nd->last);	//从缓存中查找dentry
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
		error = atomic_open(nd, dentry, path, file, op, open_flag,
				    mode, opened);
		if (unlikely(error == -ENOENT) && create_error)
			error = create_error;
		return error;
	}

no_open:
	if (d_in_lookup(dentry)) {
		//如果没有找到， 调用文件系统的lookup 方法进行查找
		struct dentry *res = dir_inode->i_op->lookup(dir_inode, dentry,
							     nd->flags);
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
		//如果没有找到且设置了O_CREAT， 调用文件系统的create方法进行创建
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
	path->dentry = dentry;
	path->mnt = nd->path.mnt;
	return 1;

out_dput:
	dput(dentry);
	return error;
}
```

8、`vfs_open->do_dentry_open` 方法，该方法中看到了熟悉的 `inode->i_fop` 成员，该成员的值是在 [`init_special_inode`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/inode.c#L1973) 中设置的，由于笔者的文件系统是 `ext4`，所以会命中 `S_ISBLK(mode)` 的逻辑

当用户调用 `open()` 系统调用时，VFS 层会初始化一个 `struct file` 对象 `f`，`f->f_op` 被赋值为目标文件所属文件系统的 `file_operations` 结构体，此处为 `ext4_file_operations`，因此，执行 `f->f_op->open` 实际调用的是 `ext4_file_open()`。这里体现了 VFS 的协作流程，即 **VFS 通过路径解析找到文件的 dentry 和 inode，根据 inode 关联的文件系统类型（如 `ext4`），将 file 结构 `f->f_op` 绑定到 `ext4_file_operations`，那么执行 `open` 操作时，路由到具体文件系统的实现函数 `ext4_file_open`**

```CPP
// fs/open.c
int vfs_open(const struct path *path, struct file *file)
{
        file->f_path = *path;
        return do_dentry_open(file, d_backing_inode(path->dentry), NULL);
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

8、[`ext4_file_open`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/file.c#L365) 方法，在 `ext4` 文件系统中，`f->f_op->open` 对应的实际函数是 `ext4_file_open()`，这一关联通过 `ext4` 定义的 [`file_operations`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/file.c#L718) 结构体实现。`ext4_file_open` 最终完成这些工作：

-   初始化文件状态：检查文件是否加密（fscrypt）、是否启用日志（jbd2）等
-   处理大文件标志：若文件超过 `2GB`，设置 `O_LARGEFILE` 标志
-   调用通用逻辑：最终通过 `generic_file_open()` 完成 VFS 层的通用文件打开流程

```CPP
static int ext4_file_open(struct inode * inode, struct file * filp)
```

####	补充：operations成员
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

####    补充：`walk_component` 的细节
[`walk_component`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1763) 方法对 `nd`（中间结果）中的目录进行遍历

-   优先使用 `lookup_fast`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1537)：如果当前的目录是一个普通目录，路径行走有两个策略：先在效率高的 rcu-walk 模式 [`__d_lookup_rcu`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1554) 下遍历，如果失败了就在效率较低的 ref-walk 模式 [`__d_lookup`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1600) 下遍历
-   如果 `lookup_fast` 查找失败，则调用 `lookup_slow` 函数。在 ref-walk 模式下会 **首先在内存缓冲区查找相应的目标（lookup_fast），如果找不到就启动具体文件系统（如 `ext4`）自己的 `lookup` 进行查找（lookup_slow）**
-   在 dcache 里找到了当前目录对应的 dentry 或者是通过 `lookup_slow` 寻找到当前目录对应的 dentry，这两种场景都会去设置 `path` 结构体里的 `dentry`、`mnt` 成员，并且将当前路径更新到 `path` 结构体
-   当 `path` 结构体更新后，最后调用 `step_info->path_to_nameidata` 将 `path` 结构体更新到 `nd.path`，[参考](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1803)，这样 `nd` 里的 `path` 就指向了当前目录了，至此完成一级目录的解析查找，返回 `link_path_walk()` 将基于 `nd.path` 作为父目录解析下一级目录，继续 `link_path_walk` 的循环查找直至退出

```CPP
static int walk_component(struct nameidata *nd, int flags)
{
	struct path path;
	struct inode *inode;
	unsigned seq;
	int err;
	/*
	 * "." and ".." are special - ".." especially so because it has
	 * to be able to know about the current root directory and
	 * parent relationships.
	 */
	if (unlikely(nd->last_type != LAST_NORM)) {
		err = handle_dots(nd, nd->last_type);
		if (!(flags & WALK_MORE) && nd->depth)
			put_link(nd);
		return err;
	}
	err = lookup_fast(nd, &path, &inode, &seq);
	if (unlikely(err <= 0)) {
		if (err < 0)
			return err;
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

	return step_into(nd, &path, flags, inode, seq);
}
```

[`__lookup_slow`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L1625) 实现如下，首先调用 `d_alloc_parallel` 给当前路径分配一个新的 dentry，然后调用 `inode->i_op->lookup()`，注意这里的 inode 是当前路径的父路径 dentry 的 `d_inode` 成员。此外，`inode->i_op` 是具体的文件系统 inode operations 函数集。以 ext4 文件系统为例，就是 ext4 fs 的 inode operations 函数集 `ext4_dir_inode_operations`，其 lookup 函数是 `ext4_lookup()`

```CPP
/* Fast lookup failed, do it the slow way */
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

####    对 dcache 的操作
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

##  0x04  write 流程
[`vfs_write`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/read_write.c#L542) 实现：

```cpp
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
	ssize_t ret;

	if (!(file->f_mode & FMODE_WRITE))
		return -EBADF;
	if (!(file->f_mode & FMODE_CAN_WRITE))
		return -EINVAL;
	if (unlikely(!access_ok(VERIFY_READ, buf, count)))
		return -EFAULT;

	ret = rw_verify_area(WRITE, file, pos, count);
	if (!ret) {
		if (count> MAX_RW_COUNT)
			count =  MAX_RW_COUNT;
		file_start_write(file);
		ret = __vfs_write(file, buf, count, pos);
		if (ret> 0) {
			fsnotify_modify(file);
			add_wchar(current, ret);
		}
		inc_syscw(current);
		file_end_write(file);
	}

	return ret;
}
```

##  0x05 参考
-   [open 系统调用（一）](https://www.kerneltravel.net/blog/2021/open_syscall_szp1/)
-   [走马观花： Linux 系统调用 open 七日游](http://blog.chinaunix.net/uid-20522771-id-4419678.html)
-   [Open() 函数的内核追踪](https://blog.csdn.net/blue95wind/article/details/7472350)
-   [Linux Kernel 文件系统写 I/O 流程代码分析（二）](https://www.cnblogs.com/jimbo17/p/10491223.html)
-   [linux 内核 open 文件流程](https://zhuanlan.zhihu.com/p/471175983)
-   [文件系统缓存](https://www.laumy.tech/776.html)
-   [Linux 的 VFS 实现 - 番外 [一] - dcache](https://zhuanlan.zhihu.com/p/261669249)
-   [Linux Kernel 文件系统写 I/O 流程代码分析（一）](https://www.cnblogs.com/jimbo17/p/10436222.html)
-   [vfs dentry cache 模块实现分析](https://zhuanlan.zhihu.com/p/457005511)