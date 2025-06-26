---
layout:     post
title:  Linux 内核之旅（二）：VFS（基础篇）
subtitle:	VFS的基本数据结构及关系
date:       2024-11-20
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
---

##  0x00    前言
Linux 支持多种文件系统格式（如 `ext2`、`ext3`、`reiserfs`、`FAT`、`NTFS`、`iso9660` 等），不同的磁盘分区或其它存储设备都有不同的文件系统格式，然而这些文件系统都可以 `mount` 到某个目录下，使开发者看到一个统一的目录树，各种文件系统上的目录和文件，读写操作用起来也都是一样的。Linux 内核在各种不同的文件系统格式之上做了一个抽象层，使得文件、目录、读写访问等概念成为抽象层的概念，因此各种文件系统看起来用起来都一样，这个抽象层称为虚拟文件系统（VFS，Virtual Filesystem）

本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

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

[root@VM-X-X-tencentos pandaychen]# ls /formount/
build  docker  home  lost+found  test  subdir
[root@VM-X-X-tencentos pandaychen]# ls /data/
build  docker  home  lost+found  test  subdir
[root@VM-X-X-tencentos pandaychen]# ls /data/test/
build  docker  home  lost+found  test  subdir
```

如果还有另外一块磁盘 vdc，那么格式化后完全可以挂载在 `/data/test/vdc` 这个路径下面。一个有趣的问题是，从 `dentry` 视角来看，在 `./subdir/a_file` 如何知道其挂载到那个分区？后面会讨论这个问题，目前有三种可能（最简单的办法 `pwd`，但这不是 `dentry` 视角）：

-	`/formount/subdir/a_file`
-	`/data/subdir/a_file`
-	`/data/test/subdir/a_file`

####	基础结构
![vfs-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/file-system/task_struct_fs.png)

-	[`inode`](https://elixir.bootlin.com/linux/v3.4/source/include/linux/fs.h#L761)：与磁盘上真实文件的一对一映射，inode号是唯一的，表示不同的文件，在Linux内部访问文件都是通过inode号来进行的，所谓文件名仅仅是给用户容易使用的。代表一个特定的文件（包括目录）
-	`super_block`：文件系统的控制块，有整个文件系统信息，一个文件系统所有的`inode`都要连接到超级块上，代表一个已挂载的文件系统
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
进一步说，由于有hard link机制的存在，对一个inode每增加一个hard link，该inode的路径指向就增加一个（即一个inode可以被多个dentry所指向）

3、process 与 vfs主要结构的关系

![process-relation](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/kernel/file-system/task_struct_fs-1.png?raw=true)

4、dcache的作用

-	VFS实现`open`、`stat`、`chmod`等类似的系统调用，都会传递一个`pathname`参数给VFS
-	VFS根据文件路径`pathname`搜索directory entry cache（高速目录项缓存，用于映射文件路径和`dentry`）获取对应的`dentry`。由于内存限制，并不是所有`dentry`都能在缓存命中，当根据`pathname`找不到对应`dentry`时，VFS调用`lookup`接口向底层文件系统查找获取`inode`信息，以此建立`dentry`和其对应的`inode`结构的关联

#### 内核数据结构（基础）
先介绍 VFS 的四个基础结构在内核中的定义

-	超级块（super block）：存储挂载的文件系统的相关信息
-	索引节点（inode）：存储一个特定文件的相关信息
-	目录项（dentry）：存储目录和文件的链接信息
-	文件对象（file）：存储进程中一个打开的文件的交互相关的信息

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

inode 和文件的关系是当创建一个文件的时候，就给文件分配了一个 inode。一个 inode 只对应一个实际文件，一个文件也会只有一个 inode，inodes 最大数量就是文件的最大数量，Linux可通过`df -i`查询inode使用情况。此外，一个inode可以代表一个普通的文件，也可以代表管道或者设备文件等这样的特殊文件

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

	//...
	union {
		struct pipe_inode_info	*i_pipe;	//如果inode所代表的文件是一个管道，则使用该字段
		struct block_device	*i_bdev;		//如果inode所代表的文件是一个块设备，则使用该字段
		struct cdev		*i_cdev; 			//如果inode所代表的文件是一个字符设备，则使用该字段
	};
}
```

-	`i_sb`：inode 所属文件系统的超级块指针
-	`i_ino`：索引节点号，每个 inode 都是唯一的
-	`i_dentry`：指向目录项链表指针，注意一个 `inodes` 可以对应多个 `dentry`，因为一个实际的文件可能被链接到其他的文件（硬链接hard link），那么就会有另一个 `dentry`，这个链表就是将所有的与本 `inode` 有关的 `dentry` 都link到一起（对一个inode每增加一个hard link，该inode的路径指向就增加一个）。**因此一个inode会对应多个dentry，通过`i_dentry`链表组织，而一个dentry只会对应一个inode，且这种关系仅存在于内存中（在磁盘中是不直接存在）**

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
-	`d_inode`：与该dentry关联的 inode
-	`d_iname`：存放短的文件名（和 `d_name` 的区别），为了节省内存用
-	`d_sb`：该目录项所属的文件系统的超级块（注意：与`dentry->d_sb`与`dentry->d_inode->i_sb`都是指向同一个`super_block`）
-	`d_parent`：指向父目录的`dentry`结构，对`..`表示的上级目录将借助`dentry->d_parent`进行遍历
-	`d_child`：
-	`d_subdirs`：
-	`d_alias`：

先描述下dentry与inode的关系，即`dentry->d_alias`/`inode->i_dentry`/`dentry->d_inode`（再强调一下这种对应关系在磁盘中是不直接存在的，dentry只存在内存中，它缓存了磁盘文件查找的结果）

![dentry-2-inode](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/inode-2-dentry.png)

此外，查找的起点通常是根目录`/`或者当前目录`CWD`，对`.`表示的同级目录将跳过解析，对`..`上级目录搜索将借助`dentry->d_parent`，`dentry->d_child`，`dentry->d_subdirs`三者的关系如下：

![dentry-parent-child-subdirs](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/dentry-parent-child-subdirs.jpg)

4、[`file`](https://elixir.bootlin.com/linux/v6.5/source/include/linux/fs.h#L961)：文件结构体代表一个打开的文件，系统中的每个打开的文件在内核空间都有一个关联的 `struct file`，它由内核在打开文件时创建，并传递给在文件上进行操作的任何函数。在文件的所有实例都关闭后，内核负责释放此数据结构。**注意文件对象描述的是进程已经打开的文件，因为一个文件可以被多个进程打开，所以一个文件可以存在多个文件对象**，但是由于文件是唯一的，那么 inode 就是唯一的，dentry也是确定的（针对一个指定路径的文件）

需要关注 `struct file` 的这几个重要成员：

-	`f_inode`：直接指向文件对应的inode（注意到此成员与`file.f_path.dentry->d_inode`的指向是相同的）
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

`struct path`结构体存在的另一个意义是**一个非文件系统挂载点的普通路径，必然处于某个文件系统的子路径中**，比如在`/mnt/opt`目录下挂载一个新的文件系统，而后在该目录下创建名为`foo`的目录，`/foo`下再创建名为`bar`的文件，那么对于这个新建的文件路径来说，其`path->dentry`对应的是`/mnt/opt/foo/bar`，而`path->mnt->dentry`对应的则是`/mnt/opt`

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

举个例子，运行`mount /dev/sdb1 /media/dir1`后，挂载点为`/media/dir1`，对于`dir1`这个目录，其产生新的`vfsmount`，独立于根文件系统挂载点`/`所在的`vfsmount`，对于挂载点`/media/dir1`而言，其`vfsmount->mnt_root->f_dentry->d_name.name = '/'`，而`vfsmount->mnt_mountpoint->f_dentry->d_name.name = 'dir1'`，这对于`/media/dir1`下的所有目录和文件而言，都是如此（为了方便举例，这里就借用`mount`结构了），所以，在`/media/dir1`下的所有目录和文件而言，通过dentry树进行向上遍历，只能看到`vfsmount->mnt_root->f_dentry->d_name.name = '/'`这里，如果要获取完整的绝对路径，就需要继续沿着`vfsmount->mnt_mountpoint->f_dentry`向上遍历，这样才能获取绝对路径

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

6、`nameidata`结构用于在`open/mkdir`等系统调用中，逐级向下，一层层地解析目录项，保存中间的查询信息。最核心的成员为`path`，`path`包含了路径dentry本身的信息，还包含了其所在文件系统的挂载点vfsmount的信息。**由于文件系统本身的挂载点也是一个路径，描述挂载点的数据结构vfsmount是将表示路径关系的dentry和表示文件系统实体的super_block结合了**

这里`path`的意义在于，`nameidata->path->dentry`暂存的是路径中当前解析的最后一个dentry，如果某一个目录节点是另一个文件系统的mount point，那么将借助`nameidata->path->mnt`跳转到新的文件系统继续查找


```CPP
struct nameidata {
    struct path path;
    ...
}

struct path {
    struct vfsmount *mnt;
    struct dentry *dentry;
}

struct vfsmount {
    struct dentry *mnt_root;	/* root of the mounted tree */
    struct super_block *mnt_sb;	/* pointer to superblock */
    ...
}
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
	-	`dentry_operations`：目录操作
	-	`file_operations`：文件操作
	-	`inode_operaions`：inode 操作
	-	`address_space_operations`：数据和 page cache 操作

![vfs-relation](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/vfs-core-struct-relation.png)

从上图可以看出

####	设计dentry的意义
dentry是目录项缓存，是一个存放在内存里的缩略版的磁盘文件系统目录树结构，思考几个问题：

1.	由于文件系统内的文件可能非常庞大，目录树结构可能很深，该树状结构中，可能存在几千万、几亿的文件，如何快速的定位到某个路径`/a/b/c`（不可能按照inode一层层定位下去）
2.	Linux提供了page cache页高速缓存，很多文件的内容已经缓存在内存里，如果没有dentry，文件名无法快速地关联到inode，即使文件的内容已经缓存在页高速缓存，但是每一次不得不重复地从磁盘上找出来文件名到VFS inode的关联
3.	需要将文件系统所有文件名到VFS inode的关联都纪录下来，但是这么做并不现实，首先并不是所有磁盘文件的inode都会纪录在内存中，其次磁盘文件数字可能非常庞大，无法简单地建立这种关联，耗尽所有的内存也做不到将文件树结构照搬进内存

所以，dentry存在的意义就是要**建立文件名filename（struct qstr d_name）到inode（struct inode ＊d_inode）的mapping关系，加速文件操作（基于路径）时的搜索速度**，这里再列举下dentry的核心定义：

```CPP
struct dentry {
	// ...
	struct hlist_bl_node d_hash;	/* lookup hash list */
	struct dentry *d_parent;	/* parent directory */
	struct qstr d_name;
	struct inode *d_inode;		/* Where the name belongs to - NULL is
					 * negative */
	unsigned char d_iname[DNAME_INLINE_LEN];	/* small names */

	struct super_block *d_sb;	/* The root of the dentry tree */

	struct list_head d_lru;		/* LRU list */
	struct list_head d_child;	/* child of parent list */
	struct list_head d_subdirs;	/* our children */
	// ...
};

#ifdef __LITTLE_ENDIAN
#define HASH_LEN_DECLARE u32 hash; u32 len;
#else
#define HASH_LEN_DECLARE u32 len; u32 hash;
#endif

struct qstr { //quick string
	union {
		struct {
			HASH_LEN_DECLARE;  //注意此结构体中可以包含hash值
			// 还有一个len变量隐藏在struct HASH_LEN_DECLARE中
		};
		u64 hash_len;
	};
	const unsigned char *name;  //qstr中的name只存放路径的最后一个分量，即basename，/usr/bin/vim 只会存放vim这个名字
};
```

由于每个dentry的父目录是唯一的，所以dentry 有成员变量`d_parent`，也就是说根据dentry很容易找到其父目录。但是dentry也会有子目录对应的dentry，所以提供了`d_subdirs`即链表成员，所有子目录对应的dentry都会挂在该链表上。根据`d_subdirs`已经可以**查找某个目录是否在内存的dentry cache中**，由于用链表查找性能不佳，所以内核提供了hash表即`dentry_hastable`，`d_hash`会放置到hash表某个头节点所在的链表

既然是hash表，那么key的取值就很关键，要尽可能避免冲突（当然不能根据目录的`basename`来hash，重复概率过高），因此计算某个指定dentry的hash value的时候，将**父dentry的地址也放入了hash计算因子，即一个dentry的hash值，取决于两个值：父dentry的地址和该dentry路径的basename**，参考`d_hash`函数：

```cpp
static inline struct hlist_bl_head *d_hash(const struct dentry *parent,
                                         unsigned int hash)
{
         hash += (unsigned long) parent / L1_CACHE_BYTES;
         return dentry_hashtable + hash_32(hash, d_hash_shift);
}
```

小结一下：

-	 内核中有一个哈希表`dentry_hashtable`（作为全局变量[实现](https://elixir.bootlin.com/linux/v4.11.6/source/fs/dcache.c#L105)），是一个`list_head`的指针数组。一旦在内存中建立起一个目录节点的dentry 结构，该dentry结构就通过其`d_hash`域链入哈希表中的某个队列中；内核中还有一个队列`dentry_unused`，凡是已经没有用户（`count`域为`0`）使用的dentry结构就通过其`d_lru`域挂入这个队列
-	`struct qstr` 中字段hash的意义：如果一个目录book，但是每一次都要计算该`basename`的hash值，就会每次查找不得不计算一次book的hash，这样会降低效率，因此采用空间换时间，计算一次后保存，提高查询效率
-	某个目录对应的dentry不在cache中？一开始可能某个目录对应的dentry根本就不在内存中，因此内核提供了`d_lookup`函数，以父dentry和`struct qstr`[类型](https://docs.huihoo.com/doxygen/linux/kernel/3.7/structqstr.html)的`name`为依据，来查找内存中是否已经有了对应的dentry，当然如果没有，就需要分配一个dentry（`d_alloc`函数负责分配dentry结构体，初始化相应的变量，建立与父dentry的关系）

##	0x01	Mnt Namespace 详解
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

2、mount tree：内核引入树形结构来关联 mount 关系（思考下前文，**一个合法的子目录可以成为任意一个块设备的挂载点**），`struct mount` 结构之间也构成了树形结构（问题：mount tree 构造的原则是什么？）

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

如上图，可以看到通过一个 `struct mount` 结构负责引用一颗 **子设备树**，把这颗子设备树挂载到父设备树的其中一个 `dentry` 节点上；如果 `dentry` 成为了挂载点 `mountpoint`，会给其标识成 `DCACHE_MOUNTED`。**在查找路径的时候同样会判断 `dentry` 的 `DCACHE_MOUNTED` 标志，一旦置位就变成了 `mountpoint`，挂载点目录下原有的内容就不能访问了，转而访问子设备树根节点下的内容**

3、`path`：因为 Linux 提供的灵活的挂载规则，所以如果要标识一个路径 `struct path` 的话需要两个元素：`vfsmount` 和 `dentry`

![path](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_path.png)

特别要注意 `struct path`->`struct vfsmount *mnt`->`struct dentry *mnt_mountpoint`，计算绝对路径要用到，指向了挂载点的 `dentry`

从上图，可以看到两个路径 `struct path` 最后引用到了同一 `inode`，但是路径 `path` 是不一样的，因为 `path` 指向的 `vfsmount` 是不一样的（很好的说明的本文开头列举的例子）

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

1、规则一，一个设备可以被挂载多次（本文开头的例子），如下图可以看到同一个子设备树，同时被两个 `struct mount` 结构所引用，被挂载到父设备树的两处不同的 `dentry` 处。特别说明 **虽然子设备树被挂载两次并且通过两处路径都能访问，但子设备的 `dentry` 和 `inode` 只保持一份**

![RULE1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_mmp.png)

2、规则二，一个挂载点可以挂载多个设备，即可以对父设备树的同一个文件夹 `dentry` 进行多次挂载，最后路径查找时生效的是最后一次挂载的子设备树

![RULES2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_mdev.png)


####	多名空间的层次化（mnt_namespace）
为了支持 `mnt_namespace`，内核把 mount 树扩展成了多棵（之前是单棵），每个 `mnt_namespace` 拥有一棵独立的 mount 树，如下图：

![mnt-namespace](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/mnt-ns/mnt_tree_ns.png)

##	0x02	部分核心源码摘要

####	mount 调用路径
`mount()` 系统调用是理解文件系统层次化的核心，它主要包含 `3` 个关键步骤：

1、解析 `mount` 系统调用中的参数挂载点路径 `pathname` ，返回对应的 `struct path` 结构

```BASH
SYSCALL_DEFINE5(mount) -> do_mount() -> user_path() -> user_path_at_empty() -> filename_lookup() -> path_lookupat() -> link_path_walk() -> walk_component() -> follow_managed()
```

2、解析 `mount` 系统调用中的参数文件系统类型 `-t type` 和设备路径 `devname` ，建立起子设备的树形结构（如果之前已经创建过引用即可），建立起新的 `struct mount` 结构对其引用

```BASH
SYSCALL_DEFINE5(mount) -> do_mount() -> do_new_mount() -> vfs_kern_mount() -> mount_fs() -> type->mount()
```

3、将新建立的 `struct mount` 结构挂载到查找到的 `struct path` 结构上

```BASH
SYSCALL_DEFINE5(mount) -> do_mount() -> do_new_mount() -> do_add_mount() -> graft_tree() -> attach_recursive_mnt() -> commit_tree()
```

##	0x03	用户态视角

####	VFS的调用路径
假设有一个文件`/myfile.txt`位于 `ext4` 作为文件系统的磁盘上，那么读取这个文件（确保它没有被缓存）的流程如下：

![ext4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/open/open-0.png)

```bash
# 系统调用链
PID     TID     COMM            FUNC
28653   28653   cat             blk_start_request
        blk_start_request+0x1 
        scsi_request_fn+0xf5 
        __blk_run_queue+0x43 
        queue_unplugged+0x2a 
        blk_flush_plug_list+0x20a 
        blk_finish_plug+0x2c 
        __do_page_cache_readahead+0x1da 
        ondemand_readahead+0x11a 
        page_cache_sync_readahead+0x2e 
        generic_file_read_iter+0x7fb 
        ext4_file_read_iter+0x56 
        new_sync_read+0xe4 
        __vfs_read+0x29 
        vfs_read+0x8e 
        sys_read+0x55 
        do_syscall_64+0x73 
        entry_SYSCALL_64_after_hwframe+0x3d 
```

####	Open Then Write（With PageCache）
以文件写入为例，系统调用先 `open` 再 `write`：

![vfsops](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/vfsops.png)

-	`open`：工作流程大致为，系统调用将创建一个 `file` 对象（首先通过查找 dentry cache 来确定 `file` 存在的位置），并且在 open files tables 中（即 `task_struct` 的 fd table）分配一个索引
-	`write`：由于 block I/O 非常耗时，所以 Linux 内核会使用 page cache 来缓存每次 read file 的内容， 当 write system call 时，系统将这个 page 标记为 dirty，并且将这个 page 移动到 dirty list 上， 系统会定时将这些 page flush 到磁盘上

##	0x04	VFS的应用（项目相关）

####	VFS 关联 task_struct
项目中通常需要基于`task_struct`来获取与VFS相关的事件属性，这里就需要了解`task_struct`与VFS基础数据结构之间的关联关系，如下图所示

-	进程（`task_struct`）所在的当前目录、根目录
-	进程打开的fd是否为socket（SOCKFS），可能需要向上追溯到父进程
-	进程在VFS中的绝对路径

![files_struct](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/vfs/fs.vfs.png)

其他相关知识点可以参考[Linux 内核之旅（一）：进程](https://pandaychen.github.io/2024/10/02/A-LINUX-KERNEL-TRAVEL-1/#0x05-关联文件系统)

####	SOCK_FS文件系统
socket在Linux中对应的文件系统叫sockfs，每创建一个socket，就在sockfs中创建了一个特殊的文件，同时创建了sockfs文件系统中的inode，该inode唯一标识当前socket的通信，那么sockfs是如何注册到VFS中的？本节就简单讨论这个问题

1、核心结构体（参考上文）

-	`struct file_system_type`：每一种文件系统必须要有自己的`file_system_type`结构
-	`struct sock_fs_type`：在Linux内核中`sock_fs_type`结构定义代表了sockfs的网络文件系统
-	`struct vfsmount`与`struct mount`：

```CPP
static struct file_system_type sock_fs_type = {
	.name = "sockfs",
	.mount = sockfs_mount,	//for mount
	.kill_sb = kill_anon_super,
};
```

2、sockfs文件系统的注册过程

内核初始化时，执行`sock_init()`函数注册sockfs，相关实现如下：

```CPP
static int __init sock_init(void)
{
	//......
	err = register_filesystem(&sock_fs_type);//注册SOCK_FS
	//......
	sock_mnt = kern_mount(&sock_fs_type);//安装SOCK_FS
	//......
}

//register_filesystem：注册函数
int register_filesystem(struct file_system_type * fs)
{
	int res = 0;
	struct file_system_type ** p;

	BUG_ON(strchr(fs->name, '.'));
	if (fs->next)
		return -EBUSY;
	write_lock(&file_systems_lock);
	p = find_filesystem(fs->name, strlen(fs->name)); //查找是否存在
	if (*p)
		res = -EBUSY;
	else
		*p = fs; //将filesystem静态变量指向fs
	write_unlock(&file_systems_lock);
	return res;
}

// find_filesystem：for循环一开始的file_systems变量就是上面说的注册文件系统使用到的全局变量指针，
// strncmp去比较file_system_type的第一项name（文件系统名）是否和将要注册的文件系统名字相同
// 如果相同返回的P就是指向同名file_system_type结构的指针
// 如果没找到则指向NULL
static struct file_system_type **find_filesystem(const char *name, unsigned len)
{
	struct file_system_type **p;
	for (p = &file_systems; *p; p = &(*p)->next)
		if (strncmp((*p)->name, name, len) == 0 && 	!(*p)->name[len])
			break;
	return p;
}
```

在返回`register_filesystem`函数后，若检查OK，就把当前要注册的文件系统挂到尾端`file_system_type`的`next`指针上，串联进链表，至此SOCK_FS文件系统模块就注册完成

3、sockfs文件系统的安装

在上面的`sock_init()`函数中的`sock_mnt = kern_mount(&sock_fs_type)`实现了对sockfs的安装过程。`kern_mount`函数主要用于那些没有实体介质的文件系统，该函数主要是获取文件系统的`super_block`对象与根目录的`inode`与`dentry`对象，并将这些对象加入到系统链表

`kern_mount`宏及相关函数实现如下：

```CPP
#define kern_mount(type) kern_mount_data(type, NULL)

struct vfsmount *kern_mount_data(struct file_system_type *type, void *data)
{
	struct vfsmount *mnt;
	//调用vfs_kern_mount
	mnt = vfs_kern_mount(type, SB_KERNMOUNT, type->name, data);
	if (!IS_ERR(mnt)) {
		/*
		* it is a longterm mount, don't release mnt until
		* we unmount before file sys is unregistered
		*/
		real_mount(mnt)->mnt_ns = MNT_NS_INTERNAL;
	}
	return mnt;
}
```

`vfs_kern_mount`函数调用`mount_fs`获取该文件系统的根目录的`dentry`，同时也获取`super_block`，具体实现如下：

```CPP
struct vfsmount *
vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
{
	struct mount *mnt;
	struct dentry *root;

	if (!type)
		return ERR_PTR(-ENODEV);

	mnt = alloc_vfsmnt(name);
	//分配一个mount对象，并对其进行部分初始化
	if (!mnt)
		return ERR_PTR(-ENOMEM);

	if (flags & SB_KERNMOUNT)
		mnt->mnt.mnt_flags = MNT_INTERNAL;

	root = mount_fs(type, flags, name, data);
	//获取该文件系统的根目录的dentry，同时也获取super_block
	if (IS_ERR(root)) {
		mnt_free_id(mnt);
		free_vfsmnt(mnt);
		return ERR_CAST(root);
	}
	//对mnt对象与root进行绑定
	mnt->mnt.mnt_root = root;
	mnt->mnt.mnt_sb = root->d_sb;
	mnt->mnt_mountpoint = mnt->mnt.mnt_root;
	mnt->mnt_parent = mnt;
	lock_mount_hash();
	list_add_tail(&mnt->mnt_instance, &root->d_sb->s_mounts);
	//将mnt添加到root->d_sb->s_mounts链表中 
	unlock_mount_hash();
	return &mnt->mnt;
}
```

接着看下`mount_fs`的实现，其中很重要的一段逻辑`type->mount`，在sockfs中是回调函数`sockfs_mount`

```CPP
struct dentry *
mount_fs(struct file_system_type *type, int flags, const char *name, void *data)
{
	struct dentry *root;
	struct super_block *sb;
	char *secdata = NULL;
	int error = -ENOMEM;

	if (data && !(type->fs_flags & FS_BINARY_MOUNTDATA)) {//在kern_mount调用中data为NULL，所以该if判断为假
		secdata = alloc_secdata();
		if (!secdata)
			goto out;

		error = security_sb_copy_data(data, secdata);
		if (error)
			goto out_free_secdata;
	}

	root = type->mount(type, flags, name, data);//调用file_system_type中的 mount方法
	if (IS_ERR(root)) {
		error = PTR_ERR(root);
		goto out_free_secdata;
	}
	sb = root->d_sb;
	BUG_ON(!sb);
	WARN_ON(!sb->s_bdi);
	sb->s_flags |= SB_BORN;

	error = security_sb_kern_mount(sb, flags, secdata);
	//......
}
```

4、`sockfs_mount`的实现，可以看到`mount_pseudo_xattr`的入参`"socket:"`、`SOCKFS_MAGIC`和`sockfs_ops`，这里关注下`SOCKFS_MAGIC`这个常量，定义在[`magic.h`](https://elixir.bootlin.com/linux/v4.11.6/source/include/uapi/linux/magic.h#L75)


`sockfs_mount`函数进行超级块sb、根root、根dentry相关的创建及初始化操作，其中这段`s->s_d_op=dops`就说指向了`sockfs_ops`结构体，也就是该sockfs文件系统的`struct super_block`的函数操作集指向了`sockfs_ops`（`sockfs_dentry_operations`）

```CPP
static const struct super_operations sockfs_ops = {
.alloc_inode = sock_alloc_inode,
.destroy_inode = sock_destroy_inode,
.statfs = simple_statfs,
};
```

`sockfs_ops`函数表**对sockfs文件系统的节点和目录提供了具体的操作函数，后面涉及到的sockfs文件系统的重要操作均会到该函数表中查找到对应的操作函数**，例如Linux内核在创建socket节点时会查找`sockfs_ops`的`alloc_inode`函数， 从而调用`sock_alloc_inode`函数完成socket以及inode节点的创建

```CPP
static struct dentry *sockfs_mount(struct file_system_type *fs_type,
int flags, const char *dev_name, void *data)
{
	return mount_pseudo_xattr(fs_type, "socket:", &sockfs_ops,
	sockfs_xattr_handlers,
	&sockfs_dentry_operations, SOCKFS_MAGIC);
}

struct dentry *mount_pseudo_xattr(struct file_system_type *fs_type, char *name,
const struct super_operations *ops, const struct xattr_handler **xattr,
const struct dentry_operations *dops, unsigned long magic)
{
	struct super_block *s;
	struct dentry *dentry;
	struct inode *root;
	struct qstr d_name = QSTR_INIT(name, strlen(name));

	s = sget_userns(fs_type, NULL, set_anon_super, SB_KERNMOUNT|SB_NOUSER,
	&init_user_ns, NULL);
	if (IS_ERR(s))
		return ERR_CAST(s);

	s->s_maxbytes = MAX_LFS_FILESIZE;
	s->s_blocksize = PAGE_SIZE;
	s->s_blocksize_bits = PAGE_SHIFT;
	s->s_magic = magic;	//设置super_block的magic
	s->s_op = ops ? ops : &simple_super_operations;
	s->s_xattr = xattr;
	s->s_time_gran = 1;
	root = new_inode(s);
	if (!root)
		goto Enomem;
	/*
	* since this is the first inode, make it number 1. New inodes created
	* after this must take care not to collide with it (by passing
	* max_reserved of 1 to iunique).
	*/
	root->i_ino = 1;
	root->i_mode = S_IFDIR | S_IRUSR | S_IWUSR;
	root->i_atime = root->i_mtime = root->i_ctime = current_time(root);
	dentry = __d_alloc(s, &d_name);
	if (!dentry) {
		iput(root);
		goto Enomem;
	}
	d_instantiate(dentry, root);
	s->s_root = dentry;	//设置根root dentry
	s->s_d_op = dops;	//重要
	s->s_flags |= SB_ACTIVE;
	return dget(s->s_root);

	Enomem:
	deactivate_locked_super(s);
	return ERR_PTR(-ENOMEM);
}
```

5、`sock_alloc_inode`函数

TODO

##  0x05  参考
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
-	[Linux虚拟文件系统(VFS)](https://arkingc.github.io/2017/08/18/2017-08-18-linux-code-vfs/)
-   [LINUX SOCKFS文件系统分析：sockfs文件系统类型的定义及注册](https://blog.csdn.net/lickylin/article/details/102540200)
-	[Linux中的VFS实现 [二]](https://zhuanlan.zhihu.com/p/107247475)