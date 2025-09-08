---
layout:     post
title:  Linux 内核之旅（十六）：内核视角下的IO读写（一）
subtitle:   基础知识 && 数据结构 && IO基础概念
date:       2025-07-03
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言

####    IO过程的性能开销

1、网络包接收流程中的性能损失

-   应用程序通过系统调用（如`recv/read`等）从用户态转为内核态的开销以及系统调用返回时从内核态转为用户态的开销
-   网络数据从内核空间通过CPU拷贝到用户空间的开销
-   内核线程ksoftirqd响应软中断的开销
-   CPU响应硬中断的开销
-   DMA拷贝网络数据包到内存中的开销

2、网络包发送流程中的性能损失

-   应用程序系统调用`write/send`的时候会从用户态转为内核态以及发送完数据后，系统调用返回时从内核态转为用户态的开销
-   用户线程内核态CPU quota用尽时触发NET_TX_SOFTIRQ类型软中断，内核响应软中断的开销
-   网卡发送完数据，向CPU发送硬中断，CPU响应硬中断的开销；在硬中断中发送NET_RX_SOFTIRQ软中断执行具体的内存清理动作以及内核响应软中断的开销
-   内存copy的开销，具体为
    -   在内核协议栈的传输层中，TCP协议对应的发送函数`tcp_sendmsg`会申请`sk_buffer`，将用户要发送的数据拷贝到sk_buffer中
    -   在发送流程从传输层到网络层的时候，会copy一个`sk_buffer`副本出来，将这个sk_buffer副本向下传递。原始`sk_buffer`会保留在socket发送队列中，等待网络对端ACK，对端ACK后删除socket发送队列中的`sk_buffer`。对端没有发送ACK，则重新从socket发送队列中发送，实现TCP协议的可靠传输
    -   在网络层，如果发现要发送的数据大于MTU，则会进行分片操作，申请额外的`sk_buffer`，并将原来的`sk_buffer`拷贝到多个小的`sk_buffer`中

####	零拷贝 VS 异步IO
零拷贝（Zero-copy）和异步I/O 是两种不同的技术，但它们都旨在提高数据传输的效率，特别是在处理大量数据时。零拷贝主要减少数据在内核态和用户空间之间的不必要复制，而异步I/O 允许应用程序在等待I/O操作完成的同时执行其他任务，从而提高并发性，二者可以相互配合使用

##	0x01	IO基础概念
这里以数据接收过程为例，前文介绍内核数据接收流程总结为两个阶段，即数据准备阶段与数据拷贝阶段，这里参考[聊聊Netty那些事儿之从内核角度看IO模型](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483737&idx=1&sn=7ef3afbb54289c6e839eed724bb8a9d6&chksm=ce77c71ef9004e08e3d164561e3a2708fc210c05408fa41f7fe338d8e85f39c1ad57519b614e&scene=178&cur_album_id=2559805446807928833&search_click_id=#rd)一文对阻塞/非阻塞、同步/异步的定义：

![io-flow-basic.png]()

-	**数据准备阶段**：网络数据包到达网卡，通过DMA的方式将数据包拷贝到内存中，然后经过硬中断，软中断，接着通过内核线程ksoftirqd经过内核协议栈的处理，最终将数据拷贝到内核sock/socket的接收缓冲区（队列）中
-	**数据拷贝阶段**：当数据到达内核socket的接收缓冲区中时，此时数据存在于内核空间中，需要将数据拷贝到用户空间中，才能够被应用程序读取

前文[]()描述了同步阻塞IO、以及epoll IO多路复用针对这两个阶段的不同处理

####	阻塞 VS 非阻塞
阻塞与非阻塞的区别主要发生在数据准备阶段，当应用程序发起系统调用`read`时，进（线）程从用户态转为内核态，读取内核socket的接收缓冲区中的网络数据

1、阻塞模式

![block_io_recv]()

如上图，如果这时内核socket的接收缓冲区没有数据，那么线程就会一直等待，直到socket接收缓冲区有数据为止。随后将数据从内核空间拷贝到用户空间，系统调用`read`返回。阻塞的特点是在第一阶段和第二阶段都会（阻塞）等待

2、非阻塞模式（阻塞和非阻塞主要的区分是在第一阶段，即数据准备阶段）

![nonblock_io_recv]()

非阻塞模式在数据接收的流程如下：

-   第一阶段，当socket接收缓冲区中没有数据的时候，阻塞模式下应用线程会一直等待；而非阻塞模式下应用线程不会等待，系统调用直接返回错误`EWOULDBLOCK`
-   当socket接收缓冲区中有数据的时候，阻塞和非阻塞模式的表现是一样的，都会进入第二阶段等待数据从内核空间拷贝到用户空间，然后系统调用返回

小结下，非阻塞的特点是第一阶段不会等待，但是在第二阶段还是会等待

####    同步与异步
同步与异步的主要区别发生在第二阶段，即数据拷贝阶段。数据拷贝阶段主要是将数据从内核空间拷贝到用户空间。然后应用程序才可以读取数据，当内核socket接收缓冲区有数据到达时，进入第二阶段

1、同步模式

同步模式在数据准备好后，是由用户线程的内核态来执行第二阶段。所以应用程序会在第二阶段发生阻塞，直到数据从内核空间拷贝到用户空间，系统调用才会返回。如
Linux下的 epoll 机制就属于同步 IO

![sync]()

2、异步模式

异步模式下是由内核来执行第二阶段的数据拷贝操作，当内核执行完第二阶段，会通知用户线程IO操作已经完成，并将数据回调给用户线程。所以在异步模式下数据准备阶段和数据拷贝阶段均是由内核来完成，不会对应用程序造成任何阻塞。异步模式需要内核底层的支持（如Linux内核 5.1版本引入的异步IO库io_uring）

![async]()

##  0x01    IO模型
基于同步/异步、阻塞/非阻塞可构建如下IO模型（自上而下，性能更优）

-   阻塞IO
-   非阻塞IO
-   IO多路复用
-   信号驱动IO
-   异步IO

####    阻塞IO（BIO）
阻塞IO模型下，网络数据的读写过程如下。由于阻塞IO的读写特点，所以导致在阻塞IO模型下，每个请求都需要被一个独立的线程处理。一个线程在同一时刻只能与一个连接绑定。来一个请求，服务端就需要创建一个线程用来处理请求

1、阻塞读，当用户线程发起`read`系统调用，用户线程从用户态切换到内核态，在内核中去查看socket接收缓冲区是否有数据到来。

-   socket接收缓冲区中有数据，则用户线程在内核态将内核空间中的数据拷贝到用户空间，系统IO调用返回
-   socket接收缓冲区中无数据，则用户线程让出CPU，进入阻塞状态。当数据到达socket接收缓冲区后，内核唤醒阻塞状态中的用户线程进入就绪状态，随后经过CPU的调度获取到CPU quota进入运行状态，将内核空间的数据拷贝到用户空间，随后系统调用返回

2、阻塞写，当用户线程发起`send`系统调用时，用户线程从用户态切换到内核态，将发送数据从用户空间拷贝到内核空间中的Socket发送缓冲区中

-   当socket发送缓冲区能够容纳下发送数据时，用户线程会将全部的发送数据写入socket缓冲区，然后执行在内核->协议栈的发送数据流程完成之后返回
-   当socket发送缓冲区空间不够，无法容纳下全部发送数据时，用户线程让出CPU并进入阻塞状态，直到socket发送缓冲区能够容纳下全部发送数据时，内核唤醒用户线程，执行后续发送流程

![BIO]()

阻塞IO模型的缺点：

1.  一个线程只能处理一个连接，如果这个连接上没有数据的话，那么这个线程就只能阻塞在系统IO调用上，不能干其他的事情，浪费CPU资源
2.  单连接单线程模式下，大量的线程创建导致上下文切换，也是巨大的系统开销

####    非阻塞IO（NIO）
网络读写操作在非阻塞IO下的特点是：

1、非阻塞读，当用户线程发起非阻塞`read`系统调用时，用户线程从用户态转为内核态，在内核中去查看socket接收缓冲区是否有数据到来

-   若socket接收缓冲区中无数据，系统调用立马返回，并带有一个 `EWOULDBLOCK` 或 `EAGAIN` 错误，这个阶段用户线程不会阻塞，也不会让出CPU，而是会继续轮训直到socket接收缓冲区中有数据为止
-   若socket接收缓冲区中有数据，用户线程在内核态会将内核空间中的数据拷贝到用户空间，注意这个数据拷贝阶段，应用程序是阻塞的，当数据拷贝完成，系统调用返回

2、非阻塞写，当发送缓冲区中没有足够的空间容纳全部发送数据时，非阻塞写的特点是尽力写完剩下的缓冲区，如写不下了就立即返回，并将写入到发送缓冲区的字节数返回给应用程序，方便用户线程不断的轮训尝试将剩下的数据写入发送缓冲区中

![nio]()

非阻塞模型的缺点：

1.  用户线程不断地系统调用检查socket接收缓冲区，从用户态到内核态的切换导致的性能开销

####    IO多路复用（）
IO多路复用模型的出发点是如何用尽可能少的线程去处理更多的连接

-   多路：核心需求是要用尽可能少的线程来处理尽可能多的连接，多路指的就是需要处理的众多连接
-   复用：核心需求是使用尽可能少的线程，尽可能少的系统开销去处理尽可能多的连接（多路），那么这里的复用指的就是用有限的资源，比如用一个线程或者固定数量的线程去处理众多连接上的读写事件。换句话说，在阻塞IO模型中一个连接就需要分配一个独立的线程去专门处理这个连接上的读写，到了IO多路复用模型中，多个连接可以复用这一个独立的线程去处理这多个连接上的读写

既然非阻塞IO模型中使用的轮询机制可能存在较多的无效切换，那么Linux内核提供了`select/poll/epoll`等事件通知机制来解决，参考前文[]()

1、`select`机制

![select]()

2、`epoll`机制

##  0x01    Direct IO VS Buffered IO

####    Direct IO

####    Buffered IO

##  0x01    page cache：页高速缓存

##  0x02    典型的数据结构

##  0x0     内核IO相关的数据结构（读）
几个核心的数据结构：
-   `struct iovec`
-   `struct kiocb`
-   `struct iov_iter iter`：`read`用于读取数据到一个用户态缓冲区，`readv`读取数据到多个用户态缓冲区，为了兼容这两种syscall，引入了数据结构`iovec`，而`iov_iter`又是对`iovec`的迭代器，**使用`iov_iter`结构体的本质是用于协助处理用户态缓冲区数据和页缓存之间的映射关系**

以ext4为例，其文件系统中管理的文件对应的 `file_operations` 指向 `ext4_file_operations`，专门用于操作 ext4 文件系统中的文件

```CPP
const struct file_operations ext4_file_operations = {
    // 并未定义 .read方法，只实现了 .read_iter方法
    .read_iter  = ext4_file_read_iter,
    .write_iter  = ext4_file_write_iter,
}
```

![ext4_file_operations]()

那么对ext4文件的VFS读取调用链路中，`__vfs_read` 调用的是 `new_sync_read` 方法

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/read_write.c#L446
ssize_t __vfs_read (struct file *file, char __user *buf, size_t count,  loff_t *pos) {  
        ......
        if (file->f_op->read)    
                return file->f_op->read(file, buf, count, pos); 
        else if (file->f_op->read_iter)    
                return new_sync_read(file, buf, count, pos);  
        else    
                return -EINVAL;
}
```

在`new_sync_read`方法中会对系统调用传进来的参数进行重新封装，把下述四个参数重新封装到 `struct iovec` 和 `struct kiocb`结构体中

-   `struct file *filp`：要读取文件的 `struct file` 结构
-   `char __user *buf`：用户空间（特意加了`__user`标识）的 Buffer，这里由用户态传入的最终内核要copy数据的目的地
-   `size_t count`：进行读取的字节数，即传入的用户态缓冲区剩余可容纳的容量大小
-   `loff_t *pos`：文件当前读取位置偏移 `offset`

```CPP
static inline void init_sync_kiocb(struct kiocb *kiocb, struct file *filp)
{
	*kiocb = (struct kiocb) {
		.ki_filp = filp,
		.ki_flags = iocb_flags(filp),
	};
}

void iov_iter_init(struct iov_iter *i, int direction,
			const struct iovec *iov, unsigned long nr_segs,
			size_t count)
{
	/* It will get better.  Eventually... */
	if (segment_eq(get_fs(), KERNEL_DS)) {
		direction |= ITER_KVEC;
		i->type = direction;
		i->kvec = (struct kvec *)iov;
	} else {
		i->type = direction;
		i->iov = iov;
	}
	i->nr_segs = nr_segs;
	i->iov_offset = 0;
	i->count = count;
}

//https://elixir.bootlin.com/linux/v4.11.6/source/fs/read_write.c#L429
static ssize_t new_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)
{
    // 1、将用户态缓存空间以及要读取的字节数封装进 iovec 结构体中
    // len：希望读取文件字节数
	struct iovec iov = { .iov_base = buf, .iov_len = len };
	struct kiocb kiocb;
	struct iov_iter iter;
	ssize_t ret;

    // 2、利用文件 struct file 初始化 kiocb 结构体
	init_sync_kiocb(&kiocb, filp);

    // 设置文件读取偏移
	kiocb.ki_pos = *ppos;

    // 3、初始化 iov_iter 结构
	iov_iter_init(&iter, READ, &iov, 1, len);

    // 4、在ext4文件系统，这里最终是调用ext4_file_read_iter函数
	ret = call_read_iter(filp, &kiocb, &iter);
	BUG_ON(ret == -EIOCBQUEUED);
	*ppos = kiocb.ki_pos;
	return ret;
}
```

`new_sync_read`函数做了两件重要的事情：

1.  封装用户请求，将用户传入的缓冲区信息包装成内核通用的 `iovec`和 `iov_iter`结构
2.  初始化上下文，设置同步I/O控制块 `kiocb`，指明操作类型、文件和起始位置

继续跟踪：

```CPP
static inline ssize_t call_read_iter(struct file *file, struct kiocb *kio,
				     struct iov_iter *iter)
{   
    // CALL ext4_file_read_iter
	return file->f_op->read_iter(kio, iter);
}

static ssize_t ext4_file_read_iter(struct kiocb *iocb, struct iov_iter *to)
{
    ......
	return generic_file_read_iter(iocb, to);
}
```

```CPP
ssize_t generic_file_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
    ......

    // generic_file_read_iter 会根据 struct kiocb 中的 ki_flags 属性
    // 判断文件 IO 操作是 Direct IO 还是 Buffered IO
    if (iocb->ki_flags & IOCB_DIRECT) {

        // Direct IO
        // 获取 page cache
        struct address_space *mapping = file->f_mapping;

        ......
        // 绕过 page cache 直接从磁盘中读取数据
        retval = mapping->a_ops->direct_IO(iocb, iter);
    }

    // Buffered IO
    // 从 page cache 中读取数据
    retval = generic_file_buffered_read(iocb, iter, retval);
}
```

####    `struct iovec`结构
`struct iovec` 结构主要用来封装用来接收文件数据用的用户缓存区相关的信息：

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/include/uapi/linux/uio.h#L16
struct iovec
{
	void __user *iov_base;	 // 用户空间缓存区地址
	__kernel_size_t iov_len; // 缓冲区长度
}
```

作为一个整体，`struct iovec`描述了一个缓冲区

####    `struct iov_iter`结构
内核中一般会使用 `struct iov_iter` 结构对 `struct iovec` 进行包装，`iov_iter` 中可以包含多个 `iovec`（`iov_iter`中的关键字 `iter`就说明了这点）。内核使用 `struct iov_iter` 结构体来包装 `struct iovec` 的目的是为了兼容 `readv()` 系列的系统调用，它允许用户使用多个用户缓存区去读取文件中的数据

![iov_iter]()

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/uio.h#L30
struct iov_iter {
	int type;                //标识读 OR 写，以及其他属性
	size_t iov_offset;       //第一个iovec中，数据起始偏移
	size_t count;            //数据大小
    // 注意这是个union类型
	union {
		const struct iovec *iov;     //结构与kvec一致，描述用户态的一段空间
		const struct kvec *kvec;     //描述内核态的一段空间
		const struct bio_vec *bvec;  //描述一个内存页中的一段空间
		struct pipe_inode_info *pipe;   //管道（读写）用
	};
	union {
		unsigned long nr_segs;      //iovec数量
		struct {
			int idx;
			int start_idx;
		};
	};
};
```

在内核中，迭代器`*_iter` 结构是常见设计，通常用来描述一个对象的处理进度。`iov_iter`最初主要用于描述一次IO流程中用户空间的处理进度，其中以`*iov`成员保存用户空间的内存地址，`iov_offset`和`count`记录当前处理进度，这两个参数会随IO的进行（读写）会不断变化

####    `struct kiocb`结构体
`struct kiocb` 结构体则是用来封装文件 IO 相关操作的状态和进度信息

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/fs.h#L274
struct kiocb {
	struct file		*ki_filp;  // 要读取的文件 struct file 结构（指向open文件创建的file结构）
	loff_t			ki_pos; // 文件读取位置偏移，表示文件处理进度
	void (*ki_complete)(struct kiocb *iocb, long ret); // IO完成回调	
	int			ki_flags; // IO类型，比如是 Direct IO 还是 Buffered IO
    void			*private;
};
```

当 `struct iovec` 和 `struct kiocb` 在 `new_sync_read` 方法中被初始化好之后，最终通过 `file_operations` 中定义的函数指针 `.read_iter` 调用到 `ext4_file_read_iter` 方法中，从而进入 ext4 文件系统执行具体的读取操作


`kiocb` 中主要保存了一个`file`结构，以及记录读写偏移，相当于描述了一次IO中文件侧的处理进度，**`iov_iter` 和 `kiocb` 实际上分别描述了一次IO的两端，`iov_iter`描述内存侧，`kiocb`描述文件侧，文件系统提供两个接口基于这两个数据结构封装读写操作**

####    `struct msghdr`

![msghdr-to-ioviter]()

##  0x  IO操作函数：用户态

####    writev 系统调用
`writev` 是一种向文件描述符写入数据的系统调用，它允许从多个缓冲区一次性写入数据，原型如下：

```CPP
/*
- fd：文件描述符，表示要写入的目标
- iov：指向 iovec 结构体数组的指针
- iovcnt：iovec 结构体数组中的元素数量
*/
ssize_t writev(int fd, const struct iovec *iov, int iovcnt);
```

`writev` 的实现原理如下：

1.  参数校验：检查 fd、iov 和 iovcnt 是否有效
2.  锁定内存：锁定用户态的 iovec 数组
3.  分配内核缓冲区：分配内存用于存储数据
4.  数据拷贝：将数据从用户态拷贝到内核缓冲区
5.  实际写操作：调用文件系统或设备驱动的写操作
6.  清理和返回：释放内存并返回写入的字节数

##  0x0    协议栈视角：同步阻塞网络IO

##  0x0 参考
-   [深入理解高性能网络开发路上的绊脚石 - 同步阻塞网络 IO](https://mp.weixin.qq.com/s/cIcw0S-Q8pBl1-WYN0UwnA)
-	[聊聊Netty那些事儿之从内核角度看IO模型](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247483737&idx=1&sn=7ef3afbb54289c6e839eed724bb8a9d6&chksm=ce77c71ef9004e08e3d164561e3a2708fc210c05408fa41f7fe338d8e85f39c1ad57519b614e&scene=178&cur_album_id=2559805446807928833&search_click_id=#rd)
-	[从 Linux 内核角度探秘 JDK NIO 文件读写本质](https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486623&idx=1&sn=0cafed9e89b60d678d8c88dc7689abda&chksm=ce77cad8f90043ceaaca732aaaa7cb692c1d23eeb6c07de84f0ad690ab92d758945807239cee&scene=178&cur_album_id=2559805446807928833&search_click_id=#rd)
-	[什么是零拷贝](https://www.xiaolincoding.com/os/8_network_system/zero_copy.html#%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E6%9C%89-dma-%E6%8A%80%E6%9C%AF)
-   [深入理解 Linux 物理内存分配全链路实现](https://www.cnblogs.com/binlovetech/p/17019710.html)
-   [从 Linux 内核角度探秘 JDK NIO 文件读写本质](https://www.cnblogs.com/binlovetech/p/16661323.html)
-   [The iov_iter interface](https://lwn.net/Articles/625077/)