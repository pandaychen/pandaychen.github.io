---
layout:     post
title:  Linux 内核之旅（十五）：管道的实现
subtitle:   有名/匿名管道实现原理：内核视角下的数据流转
date:       2025-07-01
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
这篇文章来源于对一个问题的思考：Linux中两个进程通过有名（`mkfifo`）或者匿名（`pipe`）管道进行通信时，在这两个进程的内核VFS视图中，数据是如何流转的？代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

前置基础：
-	[数据结构与算法回顾（四）：环形内存缓冲区 ringbuffer](https://pandaychen.github.io/2022/04/05/A-LINIX-C-BASED-RING-BUFFER-ANALYSIS/)

####    管道
管道应用场景：
-   zero copy
-   进程间通信，又分匿名（`pipe`）与有名（`mkfifo`）

linux的pipe和FIFO都是基于pipe文件系统（pipefs）的，pipe和FIFO都是半双工，即数据流向只能是一个方向。pipe机制（匿名管道）只能在pipe的创建进程及其后代进程（后代进程fork/exec时，通过继承父进程的打开文件描述符表）之间使用，来实现通信；有名pipe FIFO，即可以通过名称查找到pipe，所以无上述匿名管道限制，可以通过名称找到pipe文件，创建相应的pipe，可以实现跨进程间的通信

管道在Linux零拷贝中也有应用，零拷贝是一种优化数据传输的技术，它可以减少数据在内核态和用户态之间的拷贝次数，提高数据传输的效率。在传统的数据传输过程中，数据需要从内核缓冲区拷贝至应用程序的缓冲区，然后再从应用程序缓冲区拷贝到网络设备的缓冲区，最后才能发送出去。而零拷贝技术通过直接在应用程序和网络设备之间传输数据，避免了中间的拷贝过程，从而提高了数据传输的效率

TODO

##  0x01    管道实现分析：数据结构
在内核中，管道本质一个环形缓冲区，通过管道可以将数据从一个文件拷贝另外一个文件

####    pipefs：文件系统
pipe 是一个伪文件系统（pipefs），内核初始化时会注册到 Linux 系统

TODO

####    内核数据结构
1、`pipe_buffer`：管道缓存，用于暂存写入管道的数据；写进程通过管道写入端将数据写入管道缓存中，读进程通过管道读出端将数据从管道缓存中读出，成员定义如下：

-	`page`：页帧，用于存储pipe数据；pipe缓存与页帧是一对一的关系
-	`offset`：页内偏移，用于记录有效数据在页帧的超始地址（只能用偏移，而不能用地址，因为高内存页帧在内核空间中没有虚拟地址与之对应）
-	`len`：有效数据长度
-	`ops`：缓存操作集（`pipe_buf_operations`）
-	`flags`：缓存标识
-	`private`：缓存操作私有数据

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pipe_fs_i.h#L20
struct pipe_buffer {
	struct page *page;
	unsigned int offset, len;
	const struct pipe_buf_operations *ops;
	unsigned int flags;
	unsigned long private;
};
```

2、`pipe_inode_info`：定义为管道描述符，用于表示一个管道，存储管道相应的信息

-	wait：读/写/poll等待队列；由于读/写不可能同时出现在等待的情况，所以可以共用等待队列；poll读与读，poll写与写可以共存出现在等待队列中
-	nrbufs：非空的pipe_buffer数量
-	curbuf：数据的起始pipe_buffer
-	tmp_page：页缓存，可以加速页帧的分配过程；当释放页帧时将页帧记入tmp_page，当分配页帧时，先从-	tmp_page中获取，如果tmp_page为空才从伙伴系统中获取
-	readers：当前管道的读者个数；每次以读方式打开时，readers加1；关闭时readers减1
-	writers：当前管道的写者个数；每次以写方式打开时，writers加1；关闭时writers减1
-	waiting_writers：被阻塞的管道写者个数；写进程被阻塞时，waiting_writers加1；被唤醒时，waiting_writers
-	r_counter：管道读者记数器，每次以读方式打开管道时，r_counter加1；关闭是不变
-	w_counter：管道读者计数器；每次以写方式打开时，w_counter加1；关闭是不变
-	fasync_readers：读端异步描述符
-	fasync_writers：写端异步描述符
-	inode：pipe对应的inode
-	bufs：pipe_buffer回环数据

```CPP
struct pipe_inode_info {
	struct mutex mutex;
	wait_queue_head_t wait;
	unsigned int nrbufs, curbuf, buffers;
	unsigned int readers;
	unsigned int writers;
	unsigned int files;
	unsigned int waiting_writers;
	unsigned int r_counter;
	unsigned int w_counter;
	struct page *tmp_page;
	struct fasync_struct *fasync_readers;
	struct fasync_struct *fasync_writers;
	struct pipe_buffer *bufs;
	struct user_struct *user;
};
```

3、`pipe_buf_operations`：记录pipe缓存的操作集

-	can_merge：合并标识；如果pipe_buffer中有空闲空间，有数据写入时，如果can_merge置位，会先写-	pipe_buffer的空闲空间；否则重新分配一个pipe_buffer来存储写入数据
-	map：由于pipe_buffer的page可能是高内存页帧，由于内核空间页表没有相应的页表项，所以内核不能直接访问page；只有通过map将page映射到内核地址空间后，内核才能访问
-	unmap：map的逆过程；因为内核地址空间有限，所以page访问完后释文地址映射
-	confirm：检验pipe_buffer中的数据
-	release：当pipe_buffer中的数据被读完后，用于释放pipe_buffer
-	get：增加pipe_buffer的引用计数器

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pipe_fs_i.h#L74
struct pipe_buf_operations {
	/*
	 * This is set to 1, if the generic pipe read/write may coalesce
	 * data into an existing buffer. If this is set to 0, a new pipe
	 * page segment is always used for new data.
	 */
	int can_merge;

	/*
	 * ->confirm() verifies that the data in the pipe buffer is there
	 * and that the contents are good. If the pages in the pipe belong
	 * to a file system, we may need to wait for IO completion in this
	 * hook. Returns 0 for good, or a negative error value in case of
	 * error.
	 */
	int (*confirm)(struct pipe_inode_info *, struct pipe_buffer *);

	/*
	 * When the contents of this pipe buffer has been completely
	 * consumed by a reader, ->release() is called.
	 */
	void (*release)(struct pipe_inode_info *, struct pipe_buffer *);

	/*
	 * Attempt to take ownership of the pipe buffer and its contents.
	 * ->steal() returns 0 for success, in which case the contents
	 * of the pipe (the buf->page) is locked and now completely owned
	 * by the caller. The page may then be transferred to a different
	 * mapping, the most often used case is insertion into different
	 * file address space cache.
	 */
	int (*steal)(struct pipe_inode_info *, struct pipe_buffer *);

	/*
	 * Get a reference to the pipe buffer.
	 */
	void (*get)(struct pipe_inode_info *, struct pipe_buffer *);
};
```

![pipe_base_relations]()

4、`anon_pipe_buf_ops` && `packet_pipe_buf_ops`

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/pipe.c#L233
static const struct pipe_buf_operations anon_pipe_buf_ops = {
	.can_merge = 1,
	.confirm = generic_pipe_buf_confirm,
	.release = anon_pipe_buf_release,
	.steal = anon_pipe_buf_steal,
	.get = generic_pipe_buf_get,
};

static const struct pipe_buf_operations packet_pipe_buf_ops = {
	.can_merge = 0,
	.confirm = generic_pipe_buf_confirm,
	.release = anon_pipe_buf_release,
	.steal = anon_pipe_buf_steal,
	.get = generic_pipe_buf_get,
};
```

####	pipefs 在VFS的四大对象及对应的operations
当创建pipe/FIFO时，内核会分配VFS的file/dentry/inode/pipe_inode_info对象，并将file对象的`f_op`指向`fifo_open/pipe_read/pipe_write/pipe_poll`等方法，当后续的`read/write/poll`等系统调用，会通过vfs调用相应的`f_op`中方法

pipe/FIFO VFS相关结构及操作集如下：

1、文件对象（File）：`struct file`，需要区分读写端`fd[0]`为读端，`fd[1]` 为写端，操作集`file_operations`如下：

```CPP
const struct file_operations pipefifo_fops = {
	.open		= fifo_open,
	.llseek		= no_llseek,
	.read_iter	= pipe_read,	//pipe读端文件操作/FIFO只读方式文件操作
	.write_iter	= pipe_write,	//pipe写端文件操作/FIFO只写方式文件操作
	.poll		= pipe_poll,
	.unlocked_ioctl	= pipe_ioctl,
	.release	= pipe_release,
	.fasync		= pipe_fasync,
};
```

在一个进程创建打开了pipe/fifo之后，在进程打开的文件描述符数组中是占据两个fd，其中对于`fd[0]`读端而言，`pipe_write`是无用的；反之对写端而言，`pipe_read`是无用的

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/pipe.c
int do_pipe_flags(int *fd, int flags)
{
	struct file *files[2];
	int error = __do_pipe_flags(fd, files, flags);
	if (!error) {
		fd_install(fd[0], files[0]);
		fd_install(fd[1], files[1]);
	}
	return error;
}
```

2、目录项对象（Dentry）：通过 `mount_pseudo()` 创建根目录项，无实际磁盘路径。操作集为`simple_dentry_operations`

```CPP
static const struct dentry_operations pipefs_dentry_operations = {
	.d_dname	= pipefs_dname,	//仅需处理内存回收，无磁盘同步逻辑，终删除 dentry（内存临时对象）
};
```

3、 索引节点对象（Inode），关联扩展[结构](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/pipe_fs_i.h#L47) `struct pipe_inode_info`，存储管道核心数据（如环形缓冲区、等待队列），对应于`struct inode`的`struct pipe_inode_info	*i_pipe`成员

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/fs.h#L554
struct inode {
	......
	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;

	......
	const struct file_operations	*i_fop;	/* former ->i_op->default_file_ops */
	
	......
	union {
		struct pipe_inode_info	*i_pipe;	// 管道procfs
		struct block_device	*i_bdev;
		struct cdev		*i_cdev;
		char			*i_link;
		unsigned		i_dir_seq;
	};
	......
};

```


4、super_block，超级块对象（Superblock）

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/pipe.c#L1173
static const struct super_operations pipefs_ops = {
	.destroy_inode = free_inode_nonrcu,
	.statfs = simple_statfs,
};
```

##	0x02	管道操作分析
本节仅分析`pipe`匿名管道的实现

####	管道创建
系统调用`pipe/pipe2`，用于创建管道，函数调用链为`pipe2->__do_pipe_flags->create_pipe_files`

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/pipe.c#L839
SYSCALL_DEFINE2(pipe2, int __user *, fildes, int, flags)
{
	// files 数组，0和1
	struct file *files[2];
	int fd[2];
	int error;

	error = __do_pipe_flags(fd, files, flags);
	if (!error) {
		......
		fd_install(fd[0], files[0]);
		fd_install(fd[1], files[1]);
	}
	return error;
}

static int __do_pipe_flags(int *fd, struct file **files, int flags)
{
	int error;
	int fdw, fdr;

	if (flags & ~(O_CLOEXEC | O_NONBLOCK | O_DIRECT))
		return -EINVAL;

	error = create_pipe_files(files, flags);
	if (error)
		return error;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		goto err_read_pipe;
	fdr = error;

	error = get_unused_fd_flags(flags);
	if (error < 0)
		goto err_fdr;
	fdw = error;

	audit_fd_pair(fdr, fdw);
	fd[0] = fdr;
	fd[1] = fdw;
	return 0;
	......	
}
```

[`create_pipe_files`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/pipe.c#L736)是创建pipe的核心实现：

```CPP
int create_pipe_files(struct file **res, int flags)
{
	int err;
	// 创建inode节点
	struct inode *inode = get_pipe_inode();
	struct file *f;
	struct path path;
	static struct qstr name = { .name = "" };

	......

	path.dentry = d_alloc_pseudo(pipe_mnt->mnt_sb, &name);
	if (!path.dentry)
		goto err_inode;
	path.mnt = mntget(pipe_mnt);

	// 关联dentry与inode的关系
	d_instantiate(path.dentry, inode);

	// 构建file结构，设置file_operations为管理文件操作符集pipefifo_fops
	f = alloc_file(&path, FMODE_WRITE, &pipefifo_fops);
	if (IS_ERR(f)) {
		err = PTR_ERR(f);
		goto err_dentry;
	}

	f->f_flags = O_WRONLY | (flags & (O_NONBLOCK | O_DIRECT));
	// 注意：file结构的private_data指向了inode->i_pipe，即管道的pipe_inode_info节点
	f->private_data = inode->i_pipe;

	// 创建读端的file结构
	res[0] = alloc_file(&path, FMODE_READ, &pipefifo_fops);
	if (IS_ERR(res[0])) {
		err = PTR_ERR(res[0]);
		goto err_file;
	}

	path_get(&path);

	// 0：读端，设置了O_RDONLY标志
	res[0]->private_data = inode->i_pipe;
	res[0]->f_flags = O_RDONLY | (flags & O_NONBLOCK);

	// 1：写端，设置了O_WRONLY标志
	res[1] = f;
	return 0;
	.......
}
```

`create_pipe_files`函数中有几个细节：

-	`get_pipe_inode`：创建并初始化pipe的inode核心结构`struct pipe_inode_info`，对应函数[`alloc_pipe_info`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/pipe.c#L620)
-	`d_alloc_pseudo/d_instantiate`：构建dentry及inode关联关系
-	`alloc_file`：两次调用，先初始化写端的file结构、再初始化读端的file结构

```CPP
static struct inode * get_pipe_inode(void)
{
	struct inode *inode = new_inode_pseudo(pipe_mnt->mnt_sb);
	struct pipe_inode_info *pipe;
	......

	inode->i_ino = get_next_ino();

	pipe = alloc_pipe_info();
	if (!pipe)
		goto fail_iput;

	// 初始化pipe_inode_info对象
	inode->i_pipe = pipe;
	pipe->files = 2;
	pipe->readers = pipe->writers = 1;

	//inode的i_fop与file的fop是一样的
	inode->i_fop = &pipefifo_fops;

	inode->i_state = I_DIRTY;
	inode->i_mode = S_IFIFO | S_IRUSR | S_IWUSR;
	inode->i_uid = current_fsuid();
	inode->i_gid = current_fsgid();
	inode->i_atime = inode->i_mtime = inode->i_ctime = current_time(inode);

	return inode;
	......
}
```

####	pipe匿名共享的本质
问题：两个进程共享pipe，在内核的本质是什么？回到上面的`create_pipe_files`函数实现，在关键字节用序号注释标识

1、在`get_pipe_inode`中初始化了inode节点，并且将`inode->i_pipe`（union成员）指向了pipe管理结构`pipe_inode_info`对象

2、在`create_pipe_files`中，`d_instantiate(path.dentry, inode)`函数相当于执行了`path.dentry->d_inode = inode`，将dentry关联到inode结构


3、同时在`create_pipe_files`函数中，完成了读端写端的`struct file`的初始化，即`files[0] = alloc_file(&path, FMODE_READ, &pipefifo_fops);files[1] = alloc_file(&path, FMODE_WRITE, &pipefifo_fops);`，从第二个参数就知道，这两个file限制了各自的功能，也就意味着对应的两个fd也限制了各自的功能

4、在`alloc_file`函数中，`file->f_path = *path`，就相当于`file[x]->f_path.dentry->d_inode = inode`，**也即两个file指向了同一个inode，这个就是所谓的两个file关联**（记得`struct path`中存在dentry成员），这就是pipe共享的本质

5、下面这段代码将把pipe放在file中是方便取值，否则给定一个files，若需要获取pipe，就需要从`file->f_path.dentry->d_inode->i_pipe`来获取，这显然不够优雅

```CPP
files[0]->private_data = inode->i_pipe;
files[1]->private_data = inode->i_pipe;
```

6、至此`pipe2`系统调用执行完毕，它创建了两个个互相关联的file，然后创建了2个fd，fd与file一一对应，其次file都指向了同一个inode，读写均操作统一inode（即pipe结构`pipe_inode_info`）;此外，由于创建父子进程时，`task_struct`关联的打开文件描述符表fdtable也要复制一份，即父子进程的文件描述符表均指向上述的pipe结构（当然需要关闭一方的读、另一方的写），这样父子进程便可以借助pipe实现单向通信了

```CPP
static struct inode * get_pipe_inode(void)
{
	struct inode *inode = new_inode_pseudo(pipe_mnt->mnt_sb);
	struct pipe_inode_info *pipe;
	......

	pipe = alloc_pipe_info();
	......
	// 初始化pipe_inode_info对象
	// 将这个inode
	inode->i_pipe = pipe;
	pipe->files = 2;
	pipe->readers = pipe->writers = 1;

	//inode的i_fop与file的fop是一样的
	//告诉vfs如何读写pipefs（文件系统）等操作
	inode->i_fop = &pipefifo_fops;

	......
	return inode;
	......
}

// alloc_file：创建file结构，并关联到path
struct file *alloc_file(const struct path *path, fmode_t mode,	const struct file_operations *fop)
{
	struct file *file;

	file = get_empty_filp();
	if (IS_ERR(file))
		return file;
	
	// file关联到dentry
	file->f_path = *path;
	file->f_inode = path->dentry->d_inode;
	file->f_mapping = path->dentry->d_inode->i_mapping;
	if ((mode & FMODE_READ) &&
	     likely(fop->read || fop->read_iter))
		mode |= FMODE_CAN_READ;
	if ((mode & FMODE_WRITE) &&
	     likely(fop->write || fop->write_iter))
		mode |= FMODE_CAN_WRITE;
	file->f_mode = mode;
	file->f_op = fop;
	if ((mode & (FMODE_READ | FMODE_WRITE)) == FMODE_READ)
		i_readcount_inc(path->dentry->d_inode);
	return file;
}

int create_pipe_files(struct file **res, int flag s)
{
	int err;
	// 1. 创建inode节点并绑定到pipe的管理结构
	struct inode *inode = get_pipe_inode();
	struct file *f;
	struct path path;
	static struct qstr name = { .name = "" };

	......

	path.dentry = d_alloc_pseudo(pipe_mnt->mnt_sb, &name);
	if (!path.dentry)
		goto err_inode;
	path.mnt = mntget(pipe_mnt);

	// 2. 关联dentry与inode的关系，将dentry指向inode
	d_instantiate(path.dentry, inode);

	// 3. 构建（写端）file结构，设置file_operations为管理文件操作符集pipefifo_fops，同时将file->path指向 *path
	f = alloc_file(&path, FMODE_WRITE, &pipefifo_fops);
	if (IS_ERR(f)) {
		err = PTR_ERR(f);
		goto err_dentry;
	}

	f->f_flags = O_WRONLY | (flags & (O_NONBLOCK | O_DIRECT));
	// 注意：file结构的private_data指向了inode->i_pipe，即管道的pipe_inode_info节点
	f->private_data = inode->i_pipe;

	// 4.	创建读端的file结构
	res[0] = alloc_file(&path, FMODE_READ, &pipefifo_fops);
	if (IS_ERR(res[0])) {
		err = PTR_ERR(res[0]);
		goto err_file;
	}

	path_get(&path);

	// 5. 快捷访问设置：读/写端file直接访问到pipe结构
	// 0：读端，设置了O_RDONLY标志
	res[0]->private_data = inode->i_pipe;
	res[0]->f_flags = O_RDONLY | (flags & O_NONBLOCK);

	// 1：写端，设置了O_WRONLY标志
	res[1] = f;
	return 0;
	.......
}
```


####	pipe读写的实现原理（数据结构）


####	pipe的读过程


####	pipe的写过程


##  0x0 参考
-	[linux pipe文件系统(pipefs)](https://blog.csdn.net/Morphad/article/details/9219843)