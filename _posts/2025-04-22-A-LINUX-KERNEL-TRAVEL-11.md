---
layout:     post
title:  Linux 内核之旅（十）：追踪open/write系统调用
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

用户进程在能够读/写一个文件之前必须要先open这个文件。对文件的读/写从概念上说是一种进程与文件系统之间的一种有连接通信，所谓打开文件实质上就是在进程与文件之间建立起链接。在文件系统的处理中，每当一个进程重复打开同一个文件时就建立起一个由`struct file`结构代表的独立的上下文。通常一个`file`结构，即一个读/写文件的上下文，都由一个打开文件号（fd）加以标识。从VFS的层面来看，open操作的实质就是根据参数指定的路径去获取一个该文件系统（比如`ext4`）的inode（硬盘上），然后触发VFS一系列机制（如生成dentry、加载inode到内存inode以及将dentry指向内存inode等等），然后去填充VFS层的`struct file`结构体，这样就可以让上层使用了

用户态程序调用`open`函数时，会产生一个中断号为`5`的中断请求，其值以该宏`__NR__open`进行标示，而后该进程上下文将会被切换到内核空间，待内核相关操作完成后，就会从内核返回至用户态，此时还需要一次进程上下文切换，本文就以内核视角追踪下`open`（`write`）的内核调用过程

##  0x01   ftrace open系统调用
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

完整的函数调用链[可见]()

##  0x02    再看文件系统缓存

####    dentry cache
一个`struct dentry`结构代表文件系统中的一个目录或文件，VFS dentry结构的意义，即需要建立文件名filename到inode的映射关系，目的为了提高系统调用在操作、查找目录/文件操作场景下的效率，且dentry是仅存在于内存的数据结构，所以内核使用了`dentry_hashtable`（dentry树）来管理整个系统的目录树结构。在Linux可通过下面方式查看dentry cache的指标：

```BASH
[root@VM-X-X-tencentos ~]# cat /proc/sys/fs/dentry-state
1212279 1174785 45      0       598756  0
```

前文已经描述过[`dentry_hashtable`]()数据结构的意义，即为了提高目录，内核使用`dentry_hashtable`对dentry进行管理，在`open`等系统调用进行路径查找时，用于快速定位某个目录dentry下面的子dentry（哈希表的搜索效率显然更好）


1、`dentry`及`dentry_hashtable`的结构（搜索&回收）

对`dentry`而言，对于加速搜索关联了两个关键数据结构`d_hash`及`d_lru`，dentry回收关联的重要成员是`d_lockref`，`d_lockref`内嵌一个自旋锁（spinlock_t），用于保护 dentry 结构的并发修改
```CPP
struct dentry {
    /* Ref lookup also touches following */
	struct lockref d_lockref;	/* per-dentry lock and refcount */

    struct hlist_bl_node d_hash;       //hashtable的节点
    strcut list_head d_lru;     //LRU链上的节点
}
```

![dcache]()

2、`dentry_hashtable`的创建过程

在文件系统初始化时，调用`vfs_caches_init->dcache_init`为dcache进行初始化，先创建一个dentry的slab，用于后续dentry对象的分配，同时还初始化了`dentry_hashtable`这个用于管理dentry的全局hashtable

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

##  0x03 open流程
一个进程需要读/写一个文件，必须先通过filename建立和文件inode之间的通道，方式是通过`open()`函数，该函数的参数是文件所在的路径名`pathname`，如何根据`pathname`找到对应的inode？这就要依靠dentry结构了

```CPP
int open (const char *pathname, int flags, mode_t mode);
int openat(int dirfd, const char *pathname, int flags, mode_t mode);
```

####   nameidata/path/vfsmount结构
[`nameidata`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L506)

```CPP
struct nameidata {
	struct path	path;
	struct qstr	last;
	struct path	root;
	struct inode	*inode; /* path.dentry.d_inode */
	unsigned int	flags;

    //...

	struct filename	*name;
	struct nameidata *saved;
	struct inode	*link_inode;
	unsigned	root_seq;
	int		dfd;
};
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
	if (fd >= 0) {
        //调用 do_filp_open() 函数打开文件，返回打开文件的struct file结构
		struct file *f = do_filp_open(dfd, tmp, &op);
		if (IS_ERR(f)) {
			put_unused_fd(fd);
			fd = PTR_ERR(f);
		} else {
            // 通知fsnotify机制
			fsnotify_open(f);
            //调用 fd_install() 函数把文件描述符fd与file结构关联起来
			fd_install(fd, f);
		}
	}
	putname(tmp);
    //返回文件描述符，也就是 open() 系统调用的返回值
	return fd;
}

void fd_install(unsigned int fd, struct file *file)
{
	__fd_install(current->files, fd, file);
}
```

可以看到，`open`系统调用的目的就是建立新建fd与`struct file`的绑定关系，但是实际上，间接建立的VFS内存映射关系可能如下图（假设路径`b`文件并不在dcache中）

![dcache-build](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/open/open-2-dentry-build.png)

2、[`do_filp_open`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L3515)

```CPP
struct file *do_filp_open(int dfd, struct filename *pathname,
		const struct open_flags *op)
{
	struct nameidata nd;
	int flags = op->lookup_flags;
	struct file *filp;

	set_nameidata(&nd, dfd, pathname);
	filp = path_openat(&nd, op, flags | LOOKUP_RCU);
	if (unlikely(filp == ERR_PTR(-ECHILD)))
		filp = path_openat(&nd, op, flags);
	if (unlikely(filp == ERR_PTR(-ESTALE)))
		filp = path_openat(&nd, op, flags | LOOKUP_REVAL);
	restore_nameidata();
	return filp;
}
```

3、[`path_openat`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/namei.c#L3457)，在`path_openat`中，先调用`get_empty_filp`方法分配一个空的`struct file`实例，再调用`path_init`、`link_path_walk`、`do_last`等方法执行后续的open操作，如果都成功了，则返回`struct file`给上层。核心方法是`link_path_walk`、`do_last`

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

	s = path_init(nd, flags);
	if (IS_ERR(s)) {
		put_filp(file);
		return ERR_CAST(s);
	}
    // 核心方法
	while (!(error = link_path_walk(s, nd)) &&
		(error = do_last(nd, file, op, &opened)) > 0) {
		nd->flags &= ~(LOOKUP_OPEN|LOOKUP_CREATE|LOOKUP_EXCL);
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

4、`path_init`方法，`path_init`方法主要是用来初始化`struct nameidata`实例中的`path`、`root`、`inode`等字段

TODO

5、`link_path_walk`

```CPP
/*
 * Name resolution.
 * This is the basic name resolution function, turning a pathname into
 * the final dentry. We expect 'base' to be positive and a directory.
 *
 * Returns 0 and nd will have valid dentry and mnt on success.
 * Returns error and drops reference to input namei data on failure.
 */
static int link_path_walk(const char *name, struct nameidata *nd)
{
	int err;

    //跳过开始的/字符
	while (*name=='/')
		name++;
	if (!*name)
		return 0;

	/* At this point we know we have a real path component. */
	for(;;) {
		u64 hash_len;
		int type;

		err = may_lookup(nd);
		if (err)
			return err;

		hash_len = hash_name(nd->path.dentry, name);

		type = LAST_NORM;
		if (name[0] == '.') switch (hashlen_len(hash_len)) {
			case 2:
				if (name[1] == '.') {
					type = LAST_DOTDOT;
					nd->flags |= LOOKUP_JUMPED;
				}
				break;
			case 1:
				type = LAST_DOT;
		}
		if (likely(type == LAST_NORM)) {
			struct dentry *parent = nd->path.dentry;
			nd->flags &= ~LOOKUP_JUMPED;
			if (unlikely(parent->d_flags & DCACHE_OP_HASH)) {
				struct qstr this = { { .hash_len = hash_len }, .name = name };
				err = parent->d_op->d_hash(parent, &this);
				if (err < 0)
					return err;
				hash_len = this.hash_len;
				name = this.name;
			}
		}

		nd->last.hash_len = hash_len;
		nd->last.name = name;
		nd->last_type = type;

		name += hashlen_len(hash_len);
		if (!*name)
			goto OK;
		/*
		 * If it wasn't NUL, we know it was '/'. Skip that
		 * slash, and continue until no more slashes.
		 */
		do {
			name++;
		} while (unlikely(*name == '/'));
		if (unlikely(!*name)) {
OK:
			/* pathname body, done */
			if (!nd->depth)
				return 0;
			name = nd->stack[nd->depth - 1].name;
			/* trailing symlink, done */
			if (!name)
				return 0;
			/* last component of nested symlink */
			err = walk_component(nd, WALK_FOLLOW);
		} else {
			/* not the last component */
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

`link_path_walk`函数中会调用`walk_component`函数来进行路径搜索（向上或者向下），这个是较为复杂的逻辑，有很多场景，典型的如：

-   `.`或`..`
-   普通的目录，这里又会检测当前目录是否为其他文件系统的挂载点
-   符号链接

6、`do_last`方法，先调用`lookup_fast`，寻找路径中的最后一个component，如果成功，就会跳到`finish_lookup`对应的label，然后执行`step_into`方法，更新`nd`中的`path`、`inode`等信息，使其指向目标路径。然后调用`vfs_open`方法，继续执行open操作

```CPP
// fs/namei.c
static int do_last(struct nameidata *nd,
                   struct file *file, const struct open_flags *op)
{
        ...
        if (!(open_flag & O_CREAT)) {
                ...
                error = lookup_fast(nd, &path, &inode, &seq);
                if (likely(error > 0))
                        goto finish_lookup;
                ...
        } else {
                ...
        }
        ...
finish_lookup:
        error = step_into(nd, &path, 0, inode, seq);
        ...
        error = vfs_open(&nd->path, file);
        ...
        return error;
}
```

7、`vfs_open->do_dentry_open`方法，该方法中看到了熟悉的`inode->i_fop`成员，该成员的值是在[`init_special_inode`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/inode.c#L1973)中设置的，由于笔者的文件系统是`ext4`，所以会命中`S_ISBLK(mode)`的逻辑

当用户调用 `open()` 系统调用时，VFS 层会初始化一个 `struct file` 对象`f`，`f->f_op` 被赋值为目标文件所属文件系统的 `file_operations` 结构体，此处为 `ext4_file_operations`，因此，执行 `f->f_op->open` 实际调用的是 `ext4_file_open()`。这里体现了VFS的协作流程，即**VFS 通过路径解析找到文件的 dentry 和 inode，根据 inode 关联的文件系统类型（如 `ext4`），将file结构 `f->f_op` 绑定到 `ext4_file_operations`，那么执行 `open` 操作时，路由到具体文件系统的实现函数`ext4_file_open`**

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

8、[`ext4_file_open`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/file.c#L365)方法，在 `ext4` 文件系统中，`f->f_op->open` 对应的实际函数是 `ext4_file_open()`，这一关联通过 `ext4` 定义的 [`file_operations`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/ext4/file.c#L718) 结构体实现。`ext4_file_open`最终完成这些工作：

-   初始化文件状态：检查文件是否加密（fscrypt）、是否启用日志（jbd2）等
-   处理大文件标志：若文件超过 `2GB`，设置 `O_LARGEFILE` 标志
-   调用通用逻辑：最终通过 `generic_file_open()` 完成 VFS 层的通用文件打开流程

```CPP
static int ext4_file_open(struct inode * inode, struct file * filp)
```

####    补充：`walk_component`的细节

##  0x04  write流程
[`vfs_write`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/read_write.c#L542)实现：

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
		if (count > MAX_RW_COUNT)
			count =  MAX_RW_COUNT;
		file_start_write(file);
		ret = __vfs_write(file, buf, count, pos);
		if (ret > 0) {
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
-   [open系统调用（一）](https://www.kerneltravel.net/blog/2021/open_syscall_szp1/)
-   [走马观花： Linux 系统调用 open 七日游](http://blog.chinaunix.net/uid-20522771-id-4419678.html)
-   [Open()函数的内核追踪](https://blog.csdn.net/blue95wind/article/details/7472350)
-   [Linux Kernel文件系统写I/O流程代码分析（二）](https://www.cnblogs.com/jimbo17/p/10491223.html)
-   [linux 内核open文件流程](https://zhuanlan.zhihu.com/p/471175983)
-   [文件系统缓存](https://www.laumy.tech/776.html)
-   [Linux的VFS实现 - 番外[一] - dcache](https://zhuanlan.zhihu.com/p/261669249)
-   [Linux Kernel文件系统写I/O流程代码分析（一）](https://www.cnblogs.com/jimbo17/p/10436222.html)
-   [vfs dentry cache 模块实现分析](https://zhuanlan.zhihu.com/p/457005511)