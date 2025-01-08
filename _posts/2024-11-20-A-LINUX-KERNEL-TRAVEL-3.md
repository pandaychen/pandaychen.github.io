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
Linux 支持多种文件系统格式（如 `ext2`、`ext3`、`reiserfs`、`FAT`、`NTFS`、`iso9660` 等），不同的磁盘分区或其它存储设备都有不同的文件系统格式，然而这些文件系统都可以 `mount` 到某个目录下，使开发者看到一个统一的目录树，各种文件系统上的目录和文件，读写操作用起来也都是一样的。Linux 内核在各种不同的文件系统格式之上做了一个抽象层，使得文件、目录、读写访问等概念成为抽象层的概念，因此各种文件系统看起来用起来都一样，这个抽象层称为虚拟文件系统（VFS，Virtual Filesystem）

![arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/linux_vfs_arch.png)

####	目录树
![linuxdirectorytree](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/linuxdirectorytree.png)

####	mount
linux 目录是以根目录 `/` 为起点的树状结构（目录树），在访问磁盘分区之前需要先将磁盘分区挂载（mount）到这棵树上。可以挂载设备的目录称为挂载点，通过 `mount` 命令可以将 `/dev/sda1` 挂载到 `/` 根目录下，`/dev/sda2` 挂载到 `/home` 目录下，`/dev/sda3` 挂载到 `/boot` 目录下。注意不是所有目录都适合作为挂载点使用的，比如根目录下的 `/etc`、`/bin`、`/dev`、`/lib`、`/sbin`，这些目录都不能作为挂载点使用，需要和 `/` 根目录放在同一个分区中

```BASH
mount /dev/sda1 /
mount /dev/sda2 /home
mount /dev/sda3 /boot
```

![MOUNT](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/directorytreepartition.png)

通过反向追踪来判断某个文件在哪个 partition 下，如查询 `/home/vbird/test` 这个文件在哪个 partition 时，由 `test`-->`vbird`-->`home`-->`/`， 看哪个进入点先被查到那就是使用的进入点了，所以 `test` 使用的是 `/home` 这个进入点而不是 `/`

Linux 亦支持将同一个分区挂载到不同的目录下面：
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

如果还有另外一块磁盘 vdc，那么格式化后完全可以挂载在 `/data/test/vdc` 这个路径下面。一个有趣的问题是，从 `dentry` 视角来看，在 `./subdir/a_file` 如何知道其挂载到那个分区？后面会讨论这个问题，目前有三种可能（最简单的办法 `pwd`，但这不是 `dentry` 视角）：

-	`/formount/subdir/a_file`
-	`/data/subdir/a_file`
-	`/data/test/subdir/a_file`

####	基础结构
![vfs-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/file-system/task_struct_fs.png)

-	[`inode`](https://elixir.bootlin.com/linux/v3.4/source/include/linux/fs.h#L761)：与磁盘上真实文件的一对一映射，inode号是唯一的，表示不同的文件，在Linux内部访问文件都是通过inode号来进行的，所谓文件名仅仅是给用户容易使用的
-	`super_block`：文件系统的控制块，有整个文件系统信息，一个文件系统所有的`inode`都要连接到超级块上
-	[`dentry`](https://elixir.bootlin.com/linux/v3.4/source/include/linux/dcache.h#L88)：**VFS中负责维护目录树的数据结构就是`dentry`，通过把`inode`绑定到`dentry`来实现目录树的访问和管理**。Linux中一个路径字符串，在内核中会被解析为一组路径节点object，`dentry`就是路径中的一个节点。`dentry`被用来索引和访问`inode`，每个`dentry`对应一个`inode`，通过`dentry`可以找到并操作其所对应的`inode`。与`inode`和`super block`不同，`dentry`在磁盘中并没有实体储存，`dentry`链表都是在运行时通过解析`path`字符串构造出来的。此外，多个 `dentry` 可以指向同一个 `inode`（符号链接）

-	`dentry cache`：通过一个 `path` 查找对应的 `dentry`，如果每次都从磁盘中去获取的话会比较耗资源，所以内核提供了一个 LRU 缓存用于加速查找，比如查找 `/usr/bin/java` 这个文件的目录项的时候，先需要找到 `/` 的目录项，然后 `/bin`，依次类推直到找到 `path` 的结尾，这样中间的查找过程中涉及到的目录项`dentry`就会被缓存起来，方便下次查找
-	[`file`](https://elixir.bootlin.com/linux/v3.4/source/include/linux/fs.h#L976)：文件除了`dentry`和`inode`描述信息外，还需要有如读写位置等上下文操作信息，每个进程必须各自保存自己的读写上下文，因为同一个文件可以同时被多个进程读写，如果放在`dentry`和`inode`这种公共位置，会暴露给其它进程。所以一个 `file` 结构体代表一个物理文件的上下文，不同的进程，甚至同一个进程可以多次打开一个文件，因此具有多个 `file struct`

1、文件路径的本质

对于路径`/dir1/dir2/file3`，包含了`4`个文件，其中`/`、`dir1/`、`dir2/`为目录（本质上也是文件，专门存储子目录或文件的信息，而不是存储最终的用户数据），`file3`可能为目录，也可能为普通文件。以`ext2`文件系统为例，存于硬盘时，每个目录和普通文件，都对应一个`ext2_dir_entry_2`和`ext2_inode`结构，当加载到内存时，每个文件又会对应一个dentry（包含文件名）和inode（包含文件内容的位置信息）结构。同一个文件的信息，使用两种结构描述，是因为同一个文件inode，可能会有多个文件名（比如链接文件）

2、`dentry` 与 `inode` 关系
举例来说，对于`/dir1/dir2/x3`路径中的`dir2`目录，为何可以通过`/dir1/dir2`找到它？因为`dir2`的dentry，包含在`dir1`的文件内容中

```BASH
/
 |── file1
 |── dir1
   ├── dir2
       ├── dir3
       └── file3
   └── file2
```
上面的目录对应如下关系：
![dentry-inode-relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/practice/dentry_inode_relation.jpg)

3、process 与 vfs主要结构的关系

![process-relation](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/kernel/file-system/task_struct_fs-1.png?raw=true)

#### 内核数据结构（基础）
先介绍 VFS 的四个基础结构在内核中的定义

-	超级块（super block）
-	索引节点（inode）
-	目录项（dentry）
-	文件对象（file）

1、`struct super_block`：代表了整个文件系统，是文件系统的控制块，有整个文件系统信息，一个文件系统所有的 inode 都要连接到超级块上（为了便于理解，就认为一个分区就是一个 super block），实际上假设有一个 `100GB` 的硬盘，并将其划分为两个 `50GB` 的分区：`/dev/sda1` 和 `/dev/sda2`。每个分区都可以单独格式化为不同的文件系统（如 `/dev/sda1` 使用 `ext4`，`/dev/sda2` 使用 `NTFS`），对于每个格式化后的分区，操作系统都会在其内部创建一个或多个 super block 来管理该文件系统的所有操作

数据结构 [定义](https://elixir.bootlin.com/linux/v6.12.6/source/include/linux/fs.h#L1253)，列举几个重要成员：

```CPP
//struct super_block 表示一个文件系统的超级块，包含该文件系统的元数据，如文件系统的类型、挂载信息等。VFS 通过超级块来访问和管理文件系统的整体状态
struct super_block {
	struct file_system_type	*s_type;	// 文件系统类型
	struct dentry		*s_root;
	struct block_device	*s_bdev;	/* can go away once we use an accessor for @s_bdev_file */
}
```
-	 `s_type`：标识当前超级快对应的文件系统类型，也就是当前这个文件系统属于哪个类型？（例如 `ext2` 还是 `fat32`）
-	`s_bdev`：指向文件系统被安装的块设备
-	`s_root`：指向该具体文件系统安装目录的目录项（参考下图）

文件系统类型结构为 `struct file_system_type`，每个文件系统都要实现一套自己的文件操作函数，这些函数定义在 `struct file_operations` 和 `struct inode_operations` 结构体中。例如 `read` 和 `write` 函数会在不同的文件系统中有不同的实现，每种文件系统类型通过 `struct file_system_type` 来注册到 VFS

```CPP
struct file_system_type {
    char *name;                           // 文件系统名称
    int (*mount) (struct super_block *, const char *, int, void *); // 挂载操作
    // 其他字段...
};
```

2、`struct inode`：代表文件的 **元数据**，包含文件的属性，如文件的大小、权限、所有者、文件类型、设备标识符，用户标识符，用户组标识符，文件模式，扩展属性，文件读取 `/` 修改的时间戳，链接数量，指向存储该内容的磁盘区块的指针，文件分类等。VFS 通过 inode 来定位文件，并执行相应的文件操作。每个文件系统都会定义一个自己的 inode 结构，都会通过 VFS 接口来进行交互

inode 有两种：一种是 VFS 的 inode，一种是具体文件系统的 inode。前者在内存中，后者在磁盘中。所以每次其实是将磁盘中的 inode 调进填充内存中的 inode，这样才算使用了磁盘文件 inode。inode 号是唯一的，表示不同的文件。Linux 内核定位文件都是依靠 inode 号进行，当 `open` 一个文件时，首先系统找到这个文件名 `filename` 对应的 inode 号，然后通过 inode 号获取 inode 信息，最后由 inode 定位到文件数据所在的 block 后，就可以处理文件数据

inode 和文件的关系是当创建一个文件的时候，就给文件分配了一个 inode。一个 inode 只对应一个实际文件，一个文件也会只有一个 inode，inodes 最大数量就是文件的最大数量，Linux可通过`df -i`查询inode使用情况

```CPP
struct inode {
	umode_t i_mode;                 // 文件的类型和权限
    unsigned long i_ino;            // 文件的 inode 号（在文件系统中的偏移）
    struct super_block *i_sb;       // 指向文件系统超级块的指针
    unsigned long i_size;           // 文件大小
	/* Stat data, not accessed from path walking */
	//unsigned long        i_ino;
	union {
		struct list_head    i_dentry;
		struct rcu_head        i_rcu;
	};
}
```

-	`i_sb`：inode 所属文件系统的超级块指针
-	`i_ino`：索引节点号，每个 inode 都是唯一的
-	`i_dentry`：指向目录项链表指针，注意一个 `inodes` 可以对应多个 `dentry`，因为一个实际的文件可能被链接到其他的文件，那么就会有另一个 `dentry`，这个链表就是将所有的与本 `inode` 有关的 `dentry` 都link到一起

3、`struct dentry`：目录项是描述文件的逻辑属性，只存在于内存中，并没有实际对应的磁盘上的描述，更确切的说是**存在于内存的目录项缓存，为了提高查找性能而设计**。文件或目录（本质也是文件）都对应于`dentry`，即属于目录项，所有的目录项在一起构成一颗目录树（dentry tree）。例如`open` 一个文件 `/home/xxx/yyy`，那么`/`、`home`、`xxx`、`yyy` 都是一个`dentry`，VFS 在查找的时候，根据一层一层的`dentry`定位到对应的每个目录项的 inode，那么沿着目录项进行操作就可以找到最终的文件

一个有效的 `dentry` 结构必定有一个 `inode` 结构，这是因为一个目录项要么代表着一个文件，要么代表着一个目录，而目录实际上也是文件。所以，只要 `dentry` 结构是有效的，则其指针 `d_inode` 必定指向一个 `inode` 结构。那么问题来了，dentry的意义是什么？

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

-	`d_name`：dentry名称（相对，见图）
-	`d_parent`：指向父目录的`dentry`结构
-	`d_inode`：与该dentry关联的 inode
-	`d_iname`：存放短的文件名（和 `d_name` 的区别），为了节省内存用
-	`d_sb`：该目录项所属的文件系统的超级块（注意：与`dentry->d_sb`与`dentry->d_inode->i_sb`都是指向同一个`super_block`）
-	`d_subdirs`：本目录的所有孩子目录链表头

4、[`file`](https://elixir.bootlin.com/linux/v6.5/source/include/linux/fs.h#L961)：文件结构体代表一个打开的文件，系统中的每个打开的文件在内核空间都有一个关联的 `struct file`，它由内核在打开文件时创建，并传递给在文件上进行操作的任何函数。在文件的所有实例都关闭后，内核负责释放此数据结构。**注意文件对象描述的是进程已经打开的文件，因为一个文件可以被多个进程打开，所以一个文件可以存在多个文件对象**，但是由于文件是唯一的，那么 inode 就是唯一的，dentry也是确定的（针对一个指定路径的文件）

需要关注 `struct file` 的这几个重要成员：

-	`f_inode`：
-	`f_list`：所有的打开的文件形成的链表！注意一个文件系统所有的打开的文件都通过这个链接到 super_block 中的 s_files 链表中！
-	`f_path.dentry`：类型为`struct dentry *`，与该文件相关的 `dentry`
-	`f_path.mnt`：类型为`struct vfsmount *`，该文件在这个文件系统中的挂载点（参考下面的mnt图）
-	`f_path`：**通过`f_path`可以定位到该`file`在文件系统中的唯一绝对路径**
-	`f_flags`、`f_mode` 和 `f_pos`：代表进程当前操作这个文件的控制信息（因为对于一个文件，可以被多个进程同时打开，对于每个进程来说，操作该文件是异步的）
-	`f_count`：引用计数，当进程关闭某一个文件描述符fd时候，其实并不是真正的关闭文件，仅仅是将 `f_count` 计数减一，当 `f_count=0` 时候，才会真的去关闭它（`dup`，`fork`等多进程的场景）
-	`f_op`：涉及到所有的文件的操作结构，例如用户使用 `read`函数，最终都会调用 `file_operations` 中的读操作，而 `file_operations` 结构体是区分不同的文件系统的

记住`file->f_path->mnt`这个成员，它指向的是mount树中节点`struct mount`的`struct vfsmount`成员，可以通过内核特殊的`container_of`宏获取到外层mount树节点的指针地址

开发场景：
-	如何获取当前进程对应的运行二进制的绝对路径？
-	如何获取当前进程打开的文件的路径？

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

####	其他数据结构

1、[`path`](https://elixir.bootlin.com/linux/v6.5/source/include/linux/path.h#L8)，成员 `dentry` 对应于图中指向 dentry 树节点，成员 `vfsmount` 表示挂载的分区信息等，`path` 成员非常重要：

-	`struct vfsmount *mnt`：该 `path` 指向哪个挂载点（重要）
-	`struct dentry *dentry`：该 `path` 指向哪个 `dentry` 结构（dentry 树上的某个子节点）

为什么说 `struct path` 结构很重要呢？内核里用来表达路径的结构体 `path`，本质就是 `vfsmount` + `dentry`，这二者才能唯一确定文件的绝对路径，解释如下：

对于唯一的绝对路径，只依靠 `dentry` 是不行的，还需要 `vfsmount` 才能可靠地获取到，目录项 `dentry` 只能够获取向上直至承载它的文件系统根的全路径，如 `/home` 目录上挂了一个盘 `/dev/sda1`，那 `/home/dir1/dir2` 这个文件通过 `dentry` 指针（如指向`dir2`的`dentry`结构）最多只能获取到 `/dir1/dir2` 这一部分，它无法知道外层mount到哪里了，比如这里 `/home` 的部分，如果同一个盘再挂出第二个目录例如 `/data`，那么完全相同的 `dentry` 就可以通过 `/home/dir1/dir2` 和 `/data/dir1/dir2` 两个挂载点来访问，即它们对应同一个 `dentry` 但对应不同的 `vfsmount`，`vfsmount` 的作用就是指定具体的挂载点，所以内核里用来表达路径的结构体 `path` 就是 `vfsmount` 加上 `dentry`

```cpp
struct path {
	struct vfsmount *mnt;	/* 指向这个文件系统的根的dentry */
	struct dentry *dentry;	  /* 指向这个文件系统的超级块对象 */
	int mnt_flags;                  /* 此文件系统的挂载标志 */
} __randomize_layout;
```

2、`mount`[结构](https://elixir.bootlin.com/linux/v6.5/source/fs/mount.h#L39)：`struct mount`代表着一个mount实例（一次真正挂载对应一个mount实例），其中`struct vfsmount`定义的`mnt`成员是它最核心的部分（旧版本`mount`和`vfsmount`的成员都在`vfsmount`里，现在内核将`vfsmount`改作`mount`结构体，并将`mount`中`mnt_root`, `mnt_sb`, `mnt_flags`成员移到`vfsmount`结构体中）

-	上文已说，`mount`结构包含了`struct vfsmount mnt`成员
-	为了实现Linux系统的多挂载点机制，系统中所有的`mount`也仿造Linux目录树构建了一颗mount tree，用来管理mount依赖等
-	重要！由于**一个文件系统可以挂装载到不同的挂载点，所以文件系统树的一个位置要由`<mount, dentry>`二元组（或者说`<vfsmount, dentry>`）来确定**，还是参考本文开头列举的例子
-	特别注意：`mnt_mountpoint`和`mnt.mnt_root`这两个成员的区别？二者都是`struct dentry *`指针

```BASH
/formount  --> /dev/vdb
/data	   --> /dev/vdb
/data/test --> /dev/vdb
```

```CPP
struct mount {
	struct hlist_node mnt_hash;	/* 用于链接到全局已挂载文件系统的链表 */
	struct mount *mnt_parent;	/* 指向此文件系统的挂载点所属的文件系统，即父文件系统 */
	struct dentry *mnt_mountpoint;	/* 指向此文件系统的挂载点的dentry */
	struct vfsmount mnt;		/* 指向此文件系统的vfsmount实例 */
	union {
		struct rcu_head mnt_rcu;
		struct llist_node mnt_llist;
	};

	//......
	#ifdef CONFIG_SMP
	struct mnt_pcp __percpu *mnt_pcp;
#else
	int mnt_count;
	int mnt_writers;
#endif
	struct list_head mnt_mounts;	/* list of children, anchored here */
	struct list_head mnt_child;	/* and going through their mnt_child */
	struct list_head mnt_instance;	/* mount instance on sb->s_mounts */
	const char *mnt_devname;	/* Name of device e.g. /dev/dsk/hda1 */
	struct list_head mnt_list;
	struct list_head mnt_expire;	/* link in fs-specific expiry list */
	struct list_head mnt_share;	/* circular list of shared mounts */
	struct list_head mnt_slave_list;/* list of slave mounts */
	struct list_head mnt_slave;	/* slave list entry */
	struct mount *mnt_master;	/* slave is on master->mnt_slave_list */
	struct mnt_namespace *mnt_ns;	/* containing namespace */
	struct mountpoint *mnt_mp;	/* where is it mounted */
	union {
		struct hlist_node mnt_mp_list;	/* list mounts with the same mountpoint */
		struct hlist_node mnt_umount;
	};
	struct list_head mnt_umounting; /* list entry for umount propagation */
#ifdef CONFIG_FSNOTIFY
	struct fsnotify_mark_connector __rcu *mnt_fsnotify_marks;
	__u32 mnt_fsnotify_mask;
#endif
	int mnt_id;			/* mount identifier */
	int mnt_group_id;		/* peer group identifier */
	int mnt_expiry_mark;		/* true if marked for expiry */
	struct hlist_head mnt_pins;
	struct hlist_head mnt_stuck_children;
} __randomize_layout;
```

举例来说，

3、`vfsmount`[结构](https://elixir.bootlin.com/linux/v6.5/source/include/linux/mount.h#L70)：新版本`vfsmount`的成员大部分都移动到`mount`结构中了

```CPP
struct vfsmount {
     struct dentry *mnt_root;    /* root of the mounted tree */
     struct super_block *mnt_sb; /* pointer to superblock */
     int mnt_flags;
};
```

`vfsmount`结构描述的是**一个独立文件系统的挂载信息，每个不同挂载点对应一个独立的`vfsmount`结构，属于同一文件系统的所有目录和文件隶属于同一个`vfsmount`，该`vfsmount`结构对应于该文件系统顶层目录，即挂载目录**

举个例子，运行`mount /dev/sdb1 /media/dir1`后，挂载点为`/media/dir1`，对于`dir1`这个目录，其产生新的`vfsmount`，独立于根文件系统挂载点`/`所在的`vfsmount`，对于挂载点`/media/dir1`而言，其`vfsmount->mnt_root->f_dentry->d_name.name = '/'`，而`vfsmount->mnt_mountpoint->f_dentry->d_name.name = 'dir1'`，这对于`/media/dir1`下的所有目录和文件而言，都是如此（为了方便举例，这里就借用`mount`结构了），所以，在`/media/dir1`下的所有目录和文件而言，通过dentry树进行向上遍历，只能看到`vfsmount->mnt_root->f_dentry->d_name.name = '/'`这里，如果要获取完整的绝对路径，就需要继续沿着``vfsmount->mnt_mountpoint->f_dentry`向上遍历，这样才能获取绝对路径

![]()

-	所有的`vfsmount`挂载点通过`mnt_list`双链表挂载于`mnt_namespace->list`链表中，该`mnt`命名空间可以通过任意进程获得
-	子`vfsmount`挂载点结构通过`mnt_mounts`挂载于父`vfsmount`的`mnt_child`链表中，并且`mnt_parent`直接指向父亲fs的`vfsmount`结构，从而形成树层次结构
-	`vfsmount`的`super_block->statfs`函数可以获得该文件系统中空间的使用情况

4、[`fs_struct`](https://elixir.bootlin.com/linux/v6.12.4/source/include/linux/fs_struct.h#L9)结构，用来表示对于进程本身信息的描述

```CPP
struct fs_struct {
	int users;
	spinlock_t lock;
	seqcount_spinlock_t seq;
	int umask;	//打开文件时候默认的文件访问权限
	int in_exec;
	struct path root, pwd;
} __randomize_layout;
```

-	`root`：进程的根目录
-	`pwd`：进程当前的执行目录

注意，实际运行时，`root`，`pwd`目录不一定都在同一个文件系统中。例如进程的根目录通常是安装于`/`节点上的`ext`文件系统，而当前工作目录可能是安装于`/etc`的一个文件系统

5、`files_struct`[结构](https://elixir.bootlin.com/linux/v6.5/source/include/linux/fdtable.h#L49)：用户打开文件表，对于一个进程 (用户) 来说，可以同时处理多个文件，所以需要一个结构来管理所有的 `files`

```cpp
/*
 * Open file table structure
 */
struct files_struct {
  /*
   * read mostly part
   */
	atomic_t count;
	bool resize_in_progress;
	wait_queue_head_t resize_wait;

	struct fdtable __rcu *fdt;
	struct fdtable fdtab;
  /*
   * written part on a separate cache line in SMP
   */
	spinlock_t file_lock ____cacheline_aligned_in_smp;
	unsigned int next_fd;
	unsigned long close_on_exec_init[1];
	unsigned long open_fds_init[1];
	unsigned long full_fds_bits_init[1];
	struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};
```

-	`fd_array`：已打开文件对象指针的初始化数组

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

####	设计dentry的意义
dentry是目录项缓存，是一个存放在内存里的缩略版的磁盘文件系统目录树结构，思考几个问题：

1.	由于文件系统内的文件可能非常庞大，目录树结构可能很深，该树状结构中，可能存在几千万、几亿的文件，如何快速的定位到某个路径`/a/b/c`？（不可能按照inode一层层定位下去）
2.	Linux提供了page cache页高速缓存，很多文件的内容已经缓存在内存里，如果没有dentry，文件名无法快速地关联到inode，即使文件的内容已经缓存在页高速缓存，但是每一次不得不重复地从磁盘上找出来文件名到VFS inode的关联
3.	需要将文件系统所有文件名到VFS inode的关联都纪录下来，但是这么做并不现实，首先并不是所有磁盘文件的inode都会纪录在内存中，其次磁盘文件数字可能非常庞大，无法简单地建立这种关联，耗尽所有的内存也做不到将文件树结构照搬进内存

所以，dentry存在的意义就是要**建立文件名filename到inode的mapping关系，加速文件操作（基于路径）时的搜索速度**，这里再列举下dentry的核心定义：

```cpp
#ifdef __LITTLE_ENDIAN
#define HASH_LEN_DECLARE u32 hash; u32 len;
#else
#define HASH_LEN_DECLARE u32 len; u32 hash;
#endif

struct qstr { //quick string
	union {
		struct {
			HASH_LEN_DECLARE;  //注意此结构体中有hash，还有一个len变量隐藏在struct HASH_LEN_DECLARE中
		};
		u64 hash_len;
	};
	const unsigned char *name;  //qstr中的name只存放路径的最后一个分量，即basename，/usr/bin/vim 只会存放vim这个名字
};
```

由于每个dentry的父目录是唯一的，所以dentry 有成员变量`d_parent`，也就是说根据dentry很容易找到其父目录。但是dentry也会有子目录对应的dentry，所以提供了`d_subdirs`即链表成员，所有子目录对应的dentry都会挂在该链表上。根据`d_subdirs`已经可以查找某个目录是否在内存的dentry中，但是用链表查找性能不佳，所以内核提供了hash表即`d_hash`会放置到hash表某个头节点所在的链表

既然是hash表，那么key的取值就很关键，要尽可能避免冲突（当然不能根据目录的`basename`来hash，重复概率过高），因此计算hash的时候，将父dentry的地址也放入了hash计算当中，影响最后的结果，这大大降低了发生碰撞的机会，即一个dentry的hash值，取决于两个值：父dentry的地址和该dentry路径的basename，参考`d_hash`函数：

```cpp
static inline struct hlist_bl_head *d_hash(const struct dentry *parent,
                                         unsigned int hash)
{
         hash += (unsigned long) parent / L1_CACHE_BYTES;
         return dentry_hashtable + hash_32(hash, d_hash_shift);
}
```

-	注意如果一个目录book，但是每一次都要计算该basename的hash值，就会每次查找不得不计算一次book的hash，那效率就低了，因此， qstr结构体中有一个字段hash，一次算好，再也不算了。此处是稍微牺牲空间效率来提升时间效率，用空间来换时间
-	一开始可能某个目录对应的dentry根本就不在内存中，所以会有d_lookup函数，以父dentry和qstr类型的name为依据，来查找内存中是否已经有了对应的dentry，当然，如果没有，就需要分配一个dentry，这是d_alloc函数负责分配dentry结构体，初始化相应的变量，建立与父dentry的关系

##	0x0	Mnt Namespace 详解
本小节引用自 [Mnt Namespace 详解](https://tinylab.org/mnt-namespace/#down) 一文

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
3.	内核为了辅助 `struct inode` 的使用设计了 `struct dentry` 结构（即 dentry cache），通常情况下一个 `struct dentry` 对应一个 `struct inode`，也有少数情况下多个 `struct dentry` 对应一个 `struct inode`（如硬链接）。`struct dentry` 中缓存了更多的文件信息，类如文件名、层次结构，成员 `->d_parent` 指向同一块设备内的父节点 `struct dentry` ，成员 `->d_subdirs` 链接了所有的子节点 `struct dentry`

单独块设备的主要成员的联系如图：

![1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/bdev_tree.png)


####	多个块设备（重点）
Linux 使用父子树的形式来构造，父设备树中的一个文件夹 `struct dentry` 可以充当子设备树的挂载点 mountpoint（满足要求），比如上面的例子，可以将 `/dev/vdb` 设备挂载到不同的目录，如 `/data/test`、`/formount` 等，下面引入若干概念：

1、`mount`（包含成员 `vfsmount`），内核定义了一个 `struct mount` 结构来负责对一个设备内子树的引用（对于每个装载的文件系统，都对应于一个`vfsmount`结构的实例），这里再回顾下`struct vfsmount`的三个成员：

-	`mnt_root`：类型为`struct dentry *`，文件系统本身的相对根目录所对应的`dentry`保存在`mnt_root`中
-	`mnt_sb`：类型为`struct super_block *`，`mnt_sb`指针建立了与相关的超级块之间的关联（对每个装载的文件系统而言，都有且只有一个超级块实例）
-	`mnt_flags`：在`nmt_flags`可以设置各种独立于文件系统的标志，[参考](https://github.com/novelinux/linux-4.x.y/tree/master/include/linux/mount.h/struct_vfsmount.mnt_flags.md)

![2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mount_struct.png)

2、mount tree：内核引入树形结构来关联 mount 关系（思考下前文，一个合法的子目录可以成为任意一个块设备的挂载点），`struct mount` 结构之间也构成了树形结构（问题：mount tree 构造的原则是什么？）

![3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree.png)

-	`mnt_parent`：指向其父节点（表示当前挂载点的父挂载点），通过跟踪每个挂载点的父挂载点，内核可以确保文件系统按照正确的顺序进行挂载和卸载，从而避免出现不一致的状态。在 unmount 一个文件系统之前，内核需要检查是否有其他挂载点依赖于它，确保只有在没有子挂载点的情况下才能卸载该文件系统，防止数据丢失或不一致

```BASH
/ (rootfs)
├── /mnt (ext4)	 #/mnt 的 mnt_parent 指向 /（rootfs）
│   └── /mnt/sub (vfat)	#/mnt/sub 的 mnt_parent 指向 / mnt
|   └── /mnt/sub1/file1 # 继承 ext4 文件结构
└── /proc (procfs)
```

1、上图的挂载点

如上图，可以看到通过一个 `struct mount` 结构负责引用一颗 ** 子设备树 **，把这颗子设备树挂载到父设备树的其中一个 `dentry` 节点上；如果 `dentry` 成为了挂载点 `mountpoint`，会给其标识成 `DCACHE_MOUNTED`。** 在查找路径的时候同样会判断 `dentry` 的 `DCACHE_MOUNTED` 标志，一旦置位就变成了 `mountpoint`，挂载点目录下原有的内容就不能访问了，转而访问子设备树根节点下的内容 **

3、`path`：因为 Linux 提供的灵活的挂载规则，所以如果要标识一个路径 `struct path` 的话需要两个元素：`vfsmount` 和 `dentry`

![path](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_path.png)

特别要注意 `struct path`->`struct vfsmount *mnt`->`struct dentry *mnt_mountpoint`，计算绝对路径要用到，指向了挂载点的 `dentry`

从上图，可以看到两个路径 `struct path` 最后引用到了同一 `inode`，但是路径 `path` 是不一样的，因为 `path` 指向的 `vfsmount` 是不一样的（很少的说明的本文开头列举的例子）

4、`chroot`：Linux 支持每个进程拥有不同的根目录，使用 `chroot()` 系统调用可以把当前进程的根目录设置为整棵文件系统树中的任何 path

![chroot](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_chroot.png)

上图，`current->fs->root` 就是 `task_struct` 中的成员指向：

```CPP
//file:include/linux/sched.h
struct task_struct {
 //2.6 进程文件系统信息（当前目录等）
 struct fs_struct *fs;
}
```

####	mount 理解（两个规则）
mount 的过程就是把设备的文件系统加入到 vfs 框架中，以 `mount -t fstpye devname pathname` 命令来进行挂载子设备的操作为例：

1、规则一，一个设备可以被挂载多次（本文开头的例子），如下图可以看到同一个子设备树，同时被两个 `struct mount` 结构所引用，被挂载到父设备树的两处不同的 `dentry` 处。特别说明 ** 虽然子设备树被挂载两次并且通过两处路径都能访问，但子设备的 `dentry` 和 `inode` 只保持一份 **

![RULE1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_mmp.png)

2、规则二，一个挂载点可以挂载多个设备，即可以对父设备树的同一个文件夹 `dentry` 进行多次挂载，最后路径查找时生效的是最后一次挂载的子设备树

![RULES2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_mdev.png)


####	多名空间的层次化（mnt_namespace）

##  0x0	 VFS 关联 task_struct

##	0x0	files_struct
![files_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/fs.vfs.png)

##	0x0	部分核心源码摘要

####	mount
`mount()` 系统调用是理解文件系统层次化的核心，它主要包含 3 个关键步骤：

1、解析 `mount` 系统调用中的参数挂载点路径 `pathname` ，返回对应的 `struct path` 结构

```BASH
SYSCALL_DEFINE5(mount) → do_mount() → user_path() → user_path_at_empty() → filename_lookup() → path_lookupat() → link_path_walk() → walk_component() → follow_managed()
```

2、解析 `mount` 系统调用中的参数文件系统类型 `-t type` 和设备路径 `devname` ，建立起子设备的树形结构（如果之前已经创建过引用即可），建立起新的 `struct mount` 结构对其引用

```BASH
SYSCALL_DEFINE5(mount) → do_mount() → do_new_mount() → vfs_kern_mount() → mount_fs() → type->mount()
```

3、将新建立的 `struct mount` 结构挂载到查找到的 `struct path` 结构上

```BASH
SYSCALL_DEFINE5(mount) → do_mount() → do_new_mount() → do_add_mount() → graft_tree() → attach_recursive_mnt() → commit_tree()
```

##	0x0	用户态视角

####	open then write

以文件写入为例，先 `open` 再 `write`：

![vfsops](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/vfsops.png)

-	`open`：工作流程大致为，系统调用将创建一个 `file` 对象（首先通过查找 dentry cache 来确定 `file` 存在的位置），并且在 open files tables 中（即 `task_struct` 的 fd table）分配一个索引
-	`write`：由于 block I/O 非常耗时，所有 linux 会使用 page cache 来缓存每次 read file 的内容， 当 write system call 时，系统将这个 page 标记为 dirty，并且将这个 page 移动到 dirty list 上， 系统会定时将这些 page flush 到磁盘上

##  0x0  参考
-   [VFS](https://akaedu.github.io/book/ch29s03.html)
-   [vfs](https://hushi55.github.io/2015/10/19/linux-kernel-vfs)
-	[File Descriptor Table](https://chenshuo.com/notes/kernel/data-structures/)
-	[Linux 的虚拟文件系统 VFS](https://sq.sf.163.com/blog/article/218133060619399168)
-   [linux vfs 轮廓](https://qiankunli.github.io/2018/05/19/linux_file_system.html)
-   [从文件 I/O 看 Linux 的虚拟文件系统](https://jarsonfang.github.io/Kernel/%E5%86%85%E6%A0%B8%E6%96%87%E4%BB%B6%E7%B3%BB%E7%BB%9F/linux-vfs/)
-   [VFS 中的 file，dentry 和 inode](https://bean-li.github.io/vfs-inode-dentry/)
-	[Mnt Namespace 详解](https://tinylab.org/mnt-namespace/)
-	[Virtual File System](https://hushi55.github.io/2021/05/11/linux-kernel-vfs)
-	[bash 的 pwd 命令实现](https://github.com/wertarbyte/coreutils/blob/master/src/pwd.c)
-	[深入理解 Linux 文件系统之文件系统挂载 (下)](https://cloud.tencent.com/developer/article/1857533)
-	[Linux 内核－虚拟文件系统 (VFS)](https://bbs.kanxue.com/article-20845.htm)
-	[Virtual File System](https://myaut.github.io/dtrace-stap-book/kernel/fs.html)
-	[What is a Superblock, Inode, Dentry and a File?](https://unix.stackexchange.com/questions/4402/what-is-a-superblock-inode-dentry-and-a-file)