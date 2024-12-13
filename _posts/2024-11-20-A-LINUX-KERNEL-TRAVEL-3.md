---
layout:     post
title:  Linux 内核之旅（二）：VFS
subtitle:
date:       2024-11-20
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
Linux支持多种文件系统格式（如ext2、ext3、reiserfs、FAT、NTFS、iso9660等），不同的磁盘分区或其它存储设备都有不同的文件系统格式，然而这些文件系统都可以`mount`到某个目录下，使开发者看到一个统一的目录树，各种文件系统上的目录和文件，读写操作用起来也都是一样的。Linux内核在各种不同的文件系统格式之上做了一个抽象层，使得文件、目录、读写访问等概念成为抽象层的概念，因此各种文件系统看起来用起来都一样，这个抽象层称为虚拟文件系统（VFS，Virtual Filesystem）
![arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/linux_vfs_arch.png)

####	目录树
![linuxdirectorytree](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/linuxdirectorytree.png)

####	mount
linux目录是以根目录`/`为起点的树状结构（目录树），在访问磁盘分区之前需要先将磁盘分区挂载（mount）到这棵树上。可以挂载设备的目录称为挂载点，通过`mount`命令可以将`/dev/sda1`挂载到`/`根目录下，`/dev/sda2`挂载到`/home`目录下，`/dev/sda3`挂载到`/boot`目录下。注意不是所有目录都适合作为挂载点使用的，比如根目录下的`/etc`、`/bin`、`/dev`、`/lib`、`/sbin`，这些目录都不能作为挂载点使用，需要和`/`根目录放在同一个分区中

```BASH
mount /dev/sda1 /
mount /dev/sda2 /home
mount /dev/sda3 /boot
```

![MOUNT](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/directorytreepartition.png)

通过反向追踪来判断某个文件在哪个partition下，如查询`/home/vbird/test`这个文件在哪个partition时，由`test`-->`vbird`-->`home`-->`/`， 看哪个进入点先被查到那就是使用的进入点了，所以`test`使用的是`/home`这个进入点而不是`/`

Linux亦支持将同一个分区挂载到不同的目录下面：
```BASH
mount /dev/vdb /formount/
mount /dev/vdb /data/
mount /dev/vdb /data/test
```

详情如下：

```BASH
[root@VM-X-X-tencentos X]# lsblk -f
NAME   FSTYPE FSVER LABEL UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
vda                                                                           
└─vda1 ext4   1.0         7d369bd4-a580-43e2-a67c-ca6d44f82c7b     85G     9% /
vdb    ext4   1.0         5ab378b4-f1c3-48ce-9cf1-dff36890668e   46.1G     1% /formount
                                                                              /data/test
                                                                              /data

[root@VM-119-175-tencentos pandaychen]# ls /formount/
build  docker  home  lost+found  test  subdir
[root@VM-119-175-tencentos pandaychen]# ls /data/
build  docker  home  lost+found  test  subdir
[root@VM-119-175-tencentos pandaychen]# ls /data/test/
build  docker  home  lost+found  test  subdir
```

有个有趣的问题是，从`dentry`视角来看，在`./subdir/a_file`如何知道其挂载到那个分区？后面会讨论这个问题，目前有三种可能（最简单的办法`pwd`，但这不是`dentry`视角）：

-	`/formount/subdir/a_file`
-	`/data/subdir/a_file`
-	`/data/test/subdir/a_file`

####	基础结构
![vfs-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/file-system/task_struct_fs.png)

-	`inode`：[定义](https://elixir.bootlin.com/linux/v3.4/source/include/linux/fs.h#L761)，与磁盘上真实文件的一对一映射
-	`file`：[定义](https://elixir.bootlin.com/linux/v3.4/source/include/linux/fs.h#L976)，一个 `file` 结构体代表一个物理文件的上下文，不同的进程，甚至同一个进程可以多次打开一个文件，因此具有多个 `file struct`
-	`dentry`：[定义](https://elixir.bootlin.com/linux/v3.4/source/include/linux/dcache.h#L88)，多个 `dentry` 可以指向同一个 `inode`
-	`super_block`：

-	`dentry cache`：通过一个`path`查找对应的`dentry`，如果每次都从磁盘中去获取的话会比较耗资源，所以内核提供了一个lru缓存用于加速查找，比如查找 `/usr/bin/java`这个文件的目录项的时候，先需要找到 `/` 的 目录项，然后`/bin`，依次类推直到找到`path`的结尾，这样中间的查找过程中涉及到的目录项就会被缓存起来，方便下次查找

#### 内核数据结构

1、[`file`](https://elixir.bootlin.com/linux/v6.5/source/include/linux/fs.h#L961)

```cpp
/*
 * f_{lock,count,pos_lock} members can be highly contended and share
 * the same cacheline. f_{lock,mode} are very frequently used together
 * and so share the same cacheline as well. The read-mostly
 * f_{path,inode,op} are kept on a separate cacheline.
 */
struct file {
	union {
		struct llist_node	f_llist;
		struct rcu_head 	f_rcuhead;
		unsigned int 		f_iocb_flags;
	};

	/*
	 * Protects f_ep, f_flags.
	 * Must not be taken from IRQ context.
	 */
	spinlock_t		f_lock;
	fmode_t			f_mode;
	atomic_long_t		f_count;
	struct mutex		f_pos_lock;
	loff_t			f_pos;
	unsigned int		f_flags;
	struct fown_struct	f_owner;
	const struct cred	*f_cred;
	struct file_ra_state	f_ra;
	struct path		f_path;
	struct inode		*f_inode;	/* cached value */
	const struct file_operations	*f_op;

	u64			f_version;
#ifdef CONFIG_SECURITY
	void			*f_security;
#endif
	/* needed for tty driver, and maybe others */
	void			*private_data;

#ifdef CONFIG_EPOLL
	/* Used by fs/eventpoll.c to link all the hooks to this file */
	struct hlist_head	*f_ep;
#endif /* #ifdef CONFIG_EPOLL */
	struct address_space	*f_mapping;
	errseq_t		f_wb_err;
	errseq_t		f_sb_err; /* for syncfs */
} __randomize_layout
  __attribute__((aligned(4)));	/* lest something weird decides that 2 is OK */
```

2、[`path`](https://elixir.bootlin.com/linux/v6.5/source/include/linux/path.h#L8)，成员`dentry`对应于图中指向dentry树节点，成员`vfsmount`表示挂载的分区信息等

```cpp
struct path {
	struct vfsmount *mnt;
	struct dentry *dentry;
} __randomize_layout;
```

3、

```CPP
struct dentry {
	/* RCU lookup touched fields */
	unsigned int d_flags;		/* protected by d_lock */
	seqcount_t d_seq;		/* per dentry seqlock */
	struct hlist_bl_node d_hash;	/* lookup hash list */
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */
	unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

	/* Ref lookup also touches following */
	unsigned int d_count;		/* protected by d_lock */
	spinlock_t d_lock;		/* per dentry lock */
	const struct dentry_operations *d_op;
	struct super_block *d_sb;	/* The root of the dentry tree */
	unsigned long d_time;		/* used by d_revalidate */
	void *d_fsdata;			/* fs-specific data */

	struct list_head d_lru;		/* LRU list */
	/*
	 * d_child and d_rcu can share memory
	 */
	union {
		struct list_head d_child;	/* child of parent list */
	 	struct rcu_head d_rcu;
	} d_u;
	struct list_head d_subdirs;	/* our children */
	struct list_head d_alias;	/* inode alias list */
};
```

4、`vfsmount`

```CPP
struct vfsmount {
     struct dentry *mnt_root;    /* root of the mounted tree */
     struct super_block *mnt_sb; /* pointer to superblock */
     int mnt_flags;
};
```

5、https://elixir.bootlin.com/linux/v6.12.4/source/include/linux/fs_struct.h#L9

```CPP
struct fs_struct {
	int users;
	spinlock_t lock;
	seqcount_spinlock_t seq;
	int umask;
	int in_exec;
	struct path root, pwd;
} __randomize_layout;
```

####	vfs 相关 API
文件系统相关的一些对象，和对应的数据结构：

-	文件对象：`file`
-	挂载文件系统：
	-	`vfsmount`：挂载点
	-	`super_block`：文件系统
-	文件系统操作：`super_operations`
-	文件或者目录：`dentry`
-	文件或者目录操作：
	-	`file_operations`：文件操作
	-	`inode_operaions`：inode 操作
	-	`address_space_operations`：数据和 page cache 操作

##	0x0	Mnt Namespace 详解
本小节引用自[Mnt Namespace 详解](https://tinylab.org/mnt-namespace/#down)一文

Linux 文件系统，是由多种设备、多种文件系统组成的一个混合的树形结构。本小节尝试从简单到复杂介绍树形结构构造：
-	单独的块设备
-	多个块设备
-	多个命名空间的层次化

####	单独的块设备
对一个块设备来说要想构造文件系统树形结构（目录树），最重要的两个全局因素是：
-	块设备（`block_device`）
-	文件系统（`file_system_type`）

1.	内核使用数据结构 `struct super_block` 把这二者结合起来，用来标识一个块设备。确定了 `super_block` 以后，就可使用文件系统提供的方法来解析块设备的内容，形成一个块设备内部的树形结构（即目录、文件的层次结构）
2.	内核使用 `struct inode` 结构来标识块设备内部的一个文件夹或者文件，`struct inode` 结构中最重要的成员是 `->i_ino`，这个记录了 inode 在块设备中的偏移
3.	内核为了辅助 `struct inode` 的使用设计了 `struct dentry` 结构（即dentry cache），通常情况下一个 `struct dentry` 对应一个 `struct inode`，也有少数情况下多个 `struct dentry` 对应一个 `struct inode`（如硬链接）。`struct dentry` 中缓存了更多的文件信息，类如文件名、层次结构，成员 `->d_parent` 指向同一块设备内的父节点 `struct dentry` ，成员 `->d_subdirs` 链接了所有的子节点 `struct dentry`

单独块设备的主要成员的联系如图：

![1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/bdev_tree.png)


####	多个块设备（重点）
Linux 使用父子树的形式来构造，父设备树中的一个文件夹 `struct dentry` 可以充当子设备树的挂载点 mountpoint（满足要求），比如上面的例子，可以将`/dev/vdb`设备挂载到不同的目录，如`/data/test`、`/formount`等，下面引入若干概念：

1、`mount`（包含成员`vfsmount`），内核定义了一个 `struct mount` 结构来负责对一个设备内子树的引用

-	`mnt_root`：
-	`mnt_sb`：

![2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mount_struct.png)

2、mount tree：内核引入树形结构来关联mount关系（思考下前文，一个合法的子目录可以成为任意一个块设备的挂载点），`struct mount` 结构之间也构成了树形结构

![3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree.png)

-	`mnt_parent`：指向其父节点

如上图，可以看到通过一个 `struct mount` 结构负责引用一颗**子设备树**，把这颗子设备树挂载到父设备树的其中一个 `dentry` 节点上；如果 `dentry` 成为了挂载点 `mountpoint`，会给其标识成 `DCACHE_MOUNTED`。在查找路径的时候同样会判断 `dentry` 的 `DCACHE_MOUNTED` 标志，一旦置位就变成了 `mountpoint`，挂载点目录下原有的内容就不能访问了，转而访问子设备树根节点下的内容

3、`path`：因为 Linux 提供的灵活的挂载规则，所以如果要标识一个路径 `struct path` 的话需要两个元素：`vfsmount` 和 `dentry`

![path](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_path.png)

特别要注意`struct path`->`struct vfsmount *mnt`->`struct dentry *mnt_mountpoint`，计算绝对路径要用到，指向了挂载点的`dentry` 

从上图，可以看到两个路径 `struct path` 最后引用到了同一 `inode`，但是路径 `path` 是不一样的，因为 `path` 指向的 `vfsmount` 是不一样的（很少的说明的本文开头列举的例子）

4、`chroot`：Linux 支持每个进程拥有不同的根目录，使用 `chroot()` 系统调用可以把当前进程的根目录设置为整棵文件系统树中的任何 path

![chroot](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_chroot.png)

上图，`current->fs->root`就是`task_struct`中的成员指向：

```CPP
//file:include/linux/sched.h
struct task_struct {
 //2.6 进程文件系统信息（当前目录等）
 struct fs_struct *fs;
}
```

####	mount理解
mount的过程就是把设备的文件系统加入到 vfs 框架中



##  0x0	 VFS 关联task_struct

##	0x0	files_struct
![files_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/fs.vfs.png)

##	0x	

![vfsops]()

-	open：open system call 将创建一个 file 对象，并且在 open files tables 中分配一个索引。 在这个之前，还会做一些其他的工作，首先通过查找 dentry cache 来确定这个 file 存在的位置。 其次还包括一些鉴权工作
-	write：由于 block I/O 非常耗时，所有 linux 会使用 page cache 来缓存每次 read file 的内容， 当 write system call 时，系统将这个 page 标记为 dirty，并且将这个 page 移动到 dirty list 上， 系统会定时将这些 page flush 到磁盘上

##  0x0  参考
-   [VFS](https://akaedu.github.io/book/ch29s03.html)
-   [vfs](https://hushi55.github.io/2015/10/19/linux-kernel-vfs)
-	[File Descriptor Table](https://chenshuo.com/notes/kernel/data-structures/)
-	[Linux的虚拟文件系统VFS](https://sq.sf.163.com/blog/article/218133060619399168)
-   [linux vfs轮廓](https://qiankunli.github.io/2018/05/19/linux_file_system.html)
-   [从文件 I/O 看 Linux 的虚拟文件系统](https://jarsonfang.github.io/Kernel/%E5%86%85%E6%A0%B8%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/linux-vfs/)
-   [VFS中的file，dentry和inode](https://bean-li.github.io/vfs-inode-dentry/)
-	[Mnt Namespace 详解](https://tinylab.org/mnt-namespace/)
-	[Virtual File System](https://hushi55.github.io/2021/05/11/linux-kernel-vfs)
-	[bash的pwd命令实现](https://github.com/wertarbyte/coreutils/blob/master/src/pwd.c)
-	[深入理解Linux文件系统之文件系统挂载(下)](https://cloud.tencent.com/developer/article/1857533)
-	[Linux内核－虚拟文件系统(VFS)](https://bbs.kanxue.com/article-20845.htm)