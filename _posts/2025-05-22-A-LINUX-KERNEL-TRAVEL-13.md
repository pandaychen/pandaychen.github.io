---
layout:     post
title:  Linux 内核之旅（十三）：epoll
subtitle:   epoll机制在内核的运行原理
date:       2025-05-22
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
Linux的IO 多路复用机制（I/O Multiplexing）是一种通过单个线程或进程同时管理多个 I/O 通道（如网络套接字、文件描述符）的机制，用来解决大量并发文件描述符fd场景下，如何快速发现哪些fd触发了事件（读/写）。为了解决遍历fds导致的性能浪费，内核提供了`select`、`poll`、`epoll`这几类机制，本文就以内核的角度来拆解下`epoll`机制的实现

####    epoll 简单服务端示例
```CPP
void epoll_server_run()
{   
    //....
	char buf[BUF_SIZE];
	struct sockaddr_in srv_addr;
	struct sockaddr_in cli_addr;
	struct epoll_event events[MAX_EVENTS];

	listen_sock = socket(AF_INET, SOCK_STREAM, 0);

	set_sockaddr(&srv_addr);
	bind(listen_sock, (struct sockaddr *)&srv_addr, sizeof(srv_addr));

	//注意需要设置为NON-BLOCK
	setnonblocking(listen_sock);
	listen(listen_sock, MAX_CONN);
    
    //1.   创建epoll fd 并对listener fd加入监听
	epfd = epoll_create(1);
	epoll_ctl_add(epfd, listen_sock, EPOLLIN | EPOLLOUT | EPOLLET);

	socklen = sizeof(cli_addr);
	for (;;) {
        //2.   调用epoll_wait并等待返回就绪事件
		nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);
        //3.   处理就绪事件（读、写、退出等），要区分是否为listener fd或者accept fd
		for (i = 0; i < nfds; i++) {
			if (events[i].data.fd == listen_sock) {
				/* handle new connection */
				conn_sock =
				    accept(listen_sock,
					   (struct sockaddr *)&cli_addr,
					   &socklen);

				inet_ntop(AF_INET, (char *)&(cli_addr.sin_addr),
					  buf, sizeof(cli_addr));
				printf("[+] connected with %s:%d\n", buf,
				       ntohs(cli_addr.sin_port));

				setnonblocking(conn_sock);
				epoll_ctl_add(epfd, conn_sock,
					      EPOLLIN | EPOLLET | EPOLLRDHUP |
					      EPOLLHUP);
			} else if (events[i].events & EPOLLIN) {
				/* handle EPOLLIN event */
				for (;;) {
					bzero(buf, sizeof(buf));
					n = read(events[i].data.fd, buf,
						 sizeof(buf));
					if (n <= 0 /* || errno == EAGAIN */ ) {
						break;
					} else {
						printf("[+] data: %s\n", buf);
						write(events[i].data.fd, buf,
						      strlen(buf));
					}
				}
			} else {
				printf("[+] unexpected\n");
			}
			/* check if the connection is closing */
			if (events[i].events & (EPOLLRDHUP | EPOLLHUP)) {
				printf("[+] connection closed\n");
                // 将fd移除epoll监听
				epoll_ctl(epfd, EPOLL_CTL_DEL,
					  events[i].data.fd, NULL);
				close(events[i].data.fd);
				continue;
			}
		}
	}
}
```

##  0x01    前置知识 && 问题
1、`struct socket/sock/sock_common`等结构与`struct file`关系

2、`struct sock`的等待队列`sock->wq`的作用及何时被唤醒、唤醒条件是什么？这里分为两种socket，其一是listener fd，其二是通过accept拿到的fd

3、`epoll`（通过`epoll_create`创建）的等待队列的作用及机制，调用`epoll_wait`系统调用的进程何时主动让出CPU（睡眠）？何时被唤醒？唤醒之后的流程是什么？

4、同步阻塞IO机制下的收包流程，在软中断核心逻辑中，当数据包送到某个socket（sock）关联的接收队列上时，内核是如何通知应用层有数据到达的？参考此文[深入理解高性能网络开发路上的绊脚石 - 同步阻塞网络 IO](https://mp.weixin.qq.com/s?__biz=MjM5Njg5NDgwNA==&mid=2247484834&idx=1&sn=b8620f402b68ce878d32df2f2bcd4e2e&scene=21#wechat_redirect)

5、内核协议栈的收包过程，[前文](https://pandaychen.github.io/2025/03/02/A-LINUX-KERNEL-TRAVEL-8/)已描述

6、`epoll`机制下，各个内核struct之间的关系以及回调函数（赋值为何种函数）

7、`epoll`机制下，为何要使用红黑树而不是hashtable？

####    struct sock/socket/sock_comm

![sock-relation]()

####    epoll的API
1、`epoll_create`：创建一个 `epoll` 对象，返回fd

2、`epoll_ctl`：向 `epoll` 对象中添加要管理的连接，在代码示例中主要涉及到操作两类fd，第一类是listnerfd（服务端绑定socket），第二类是acceptfd（客户端新连接）

3、`epoll_wait`：等待其管理的连接上的 IO 事件

##  0x02    epoll_create实现
在用户进程调用 `epoll_create` 函数时，内核会创建一个 `struct eventpoll` 的[内核结构](https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L185)对象，注意`epoll_create`成功时会返回一个`fd`，内核也会把它关联到当前进程的已打开文件列表`fdtable`中

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L1793
SYSCALL_DEFINE1(epoll_create1, int, flags)
{
	int error, fd;
	struct eventpoll *ep = NULL;
	struct file *file;
    // ...

    //1. 创建一个 eventpoll 对象
	error = ep_alloc(&ep);
	if (error < 0)
		return error;

	fd = get_unused_fd_flags(O_RDWR | (flags & O_CLOEXEC));
	if (fd < 0) {
		error = fd;
		goto out_free_ep;
	}
	file = anon_inode_getfile("[eventpoll]", &eventpoll_fops, ep,
				 O_RDWR | (flags & O_CLOEXEC));
	if (IS_ERR(file)) {
		error = PTR_ERR(file);
		goto out_free_fd;
	}
	ep->file = file;
    //安装fd
	fd_install(fd, file);
	return fd;
    //....
}
```

####    eventpoll结构
`struct eventpoll`的定义如下，这里重点看下面几个成员：

-   `wait_queue_head_t wq`：`epoll_wait`/`sys_epoll_wait`用到的等待队列（链表），内核软中断运行过程中，当数据就绪的时候会通过此 `wq` 来找到阻塞在 `epoll` 对象上的用户进程
-   `struct list_head rdllist`：接收就绪的描述符都会放到这里，即存放就绪的描述符的链表。当有连接就绪的时候，内核会把就绪的连接放到 `rdllist` 链表里。这样应用进程只需要遍历`rdllist`链表就能找出就绪进程，而不用去遍历整棵树
-   `struct rb_root rbr`：每个epoll对象中都有一颗红黑树，为了支持对海量连接的高效查找、插入和删除，`eventpoll` 内部使用了一棵红黑树并通过这棵树来管理用户进程下添加进来的所有 socket 连接（红黑树的key为`fd`）

```CPP
struct eventpoll {
	/* Protect the access to this structure */
	spinlock_t lock;

	/*
	 * This mutex is used to ensure that files are not removed
	 * while epoll is using them. This is held during the event
	 * collection loop, the file cleanup path, the epoll file exit
	 * code and the ctl operations.
	 */
	struct mutex mtx;

	/* Wait queue used by sys_epoll_wait() */
	wait_queue_head_t wq;

	/* Wait queue used by file->poll() */
	wait_queue_head_t poll_wait;

	/* List of ready file descriptors */
	struct list_head rdllist;

	/* RB tree root used to store monitored fd structs */
	struct rb_root rbr;

	/*
	 * This is a single linked list that chains all the "struct epitem" that
	 * happened while transferring ready events to userspace w/out
	 * holding ->lock.
	 */
	struct epitem *ovflist;

	/* wakeup_source used when ep_scan_ready_list is running */
	struct wakeup_source *ws;

	/* The user that created the eventpoll descriptor */
	struct user_struct *user;

	struct file *file;

	/* used to optimize loop detection check */
	int visited;
	struct list_head visited_list_link;
};
```

####    ep_alloc的实现
[`ep_alloc`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L940)的实现如下，其中包含了上面列出的重要成员的初始化工作

```CPP
static int ep_alloc(struct eventpoll **pep)
{
	int error;
	struct user_struct *user;
	struct eventpoll *ep;

	user = get_current_user();
	error = -ENOMEM;

    //1.  申请 epollevent 内存
	ep = kzalloc(sizeof(*ep), GFP_KERNEL);
	if (unlikely(!ep))
		goto free_uid;

	spin_lock_init(&ep->lock);
	mutex_init(&ep->mtx);

    //2. 初始化等待队列头
	init_waitqueue_head(&ep->wq);
	init_waitqueue_head(&ep->poll_wait);

    //3. 初始化就绪列表
	INIT_LIST_HEAD(&ep->rdllist);

    //4. 初始化红黑树指针
	ep->rbr = RB_ROOT;
	ep->ovflist = EP_UNACTIVE_PTR;
	ep->user = user;

	*pep = ep;

	return 0;

free_uid:
	free_uid(user);
	return error;
}
```

至此，完成了epoll机制最核心的管理结构`struct eventpoll`的创建及初始化工作

##  0x03    服务端accept新连接（fd）
前文[]()已经介绍了服务端accept的内核实现，这里再回顾下accept时`struct socket` 内核结构的创建、以及和内核已存在的`struct sock`关联的主要过程：

1.	初始化 `struct socket` 对象，并使用listnerfd关联的socket信息初始化（如`type`、`ops`等）
2.	为新 `struct socket` 对象申请 `struct file` 结构，关联`sock->file`指针，在 `accept` 方法里会调用 `sock_alloc_file` 函数来申请内存并初始化，该指针的主要目的是通过`task_struct->fdtable(fd)->file(private)->socket->sock....`，即通过进程+fd找到对应的`struct socket/sock`结构
3.	在`sock_alloc_file->alloc_file`函数中，会把`socket_file_ops`赋值给新建的`socket->file->f_op`
4.	调用`sock->ops->accept`（即`inet_accept`）接收新连接，
5.	调用`fd_install`将`accept`返回的新连接`fd`加到当前进程打开文件列表`fdtable`

```CPP
SYSCALL_DEFINE4(accept4, int, fd, struct sockaddr __user *, upeer_sockaddr,
        int __user *, upeer_addrlen, int, flags)
{
    struct socket *sock, *newsock;

    //根据 fd 查找到监听的 socket
    sock = sockfd_lookup_light(fd, &err, &fput_needed);

    //1、申请并初始化新的 socket
    newsock = sock_alloc();

	//2、把 listen 状态的 socket 对象（关联AF_INET）上的协议操作函数集合 ops等赋值给新的 socket
    newsock->type = sock->type;
    newsock->ops = sock->ops;

    //3、 申请新的 file 对象，并设置到新 socket 上
	// sock_alloc_file最终会完成 sock->file = file
	// 
    newfile = sock_alloc_file(newsock, flags, sock->sk->sk_prot_creator->name);
    ......

    //1.3 接收连接
    err = sock->ops->accept(sock, newsock, sock->file->f_flags);

    //1.4 添加新文件到当前进程的打开文件列表
    fd_install(newfd, newfile);
```

####	新建立socket/file结构
![socket-2-file-relation]()

上文说到，在`accept`里创建的新 `struct socket` 成员`file`，即`struct file`的`f_op`成员（类型为`struct file_operations`），被赋值为下面`socket_file_ops`的[方法](https://elixir.bootlin.com/linux/v4.11.6/source/net/socket.c#L140)：

```CPP
static const struct file_operations socket_file_ops = {
	.llseek =	no_llseek,
	.read_iter =	sock_read_iter,
	.write_iter =	sock_write_iter,
	.poll =		sock_poll,		//记住这个sock_poll函数
	.unlocked_ioctl = sock_ioctl,
	.mmap =		sock_mmap,
	.release =	sock_close,
	.fasync =	sock_fasync,
	.sendpage =	sock_sendpage,
	.splice_write = generic_splice_sendpage,
	.splice_read =	sock_splice_read,
};
```

记住上面这个`sock_poll`函数：在`accept`里创建的新 socket 里的 `file->f_op->poll` 函数指向的是 `sock_poll`

####	接收新连接
这里只列举`sock->ops->accept`中几个关键赋值结点：

内核在三次握手完成时会从全连接队列（已完成TCP连接建立）中取出，关联函数为[`inet_csk_accept`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/inet_connection_sock.c#L427)，从这里可以看到，这个`struct sock`对象`sk`是内核在三次握手完成就已经创建完成的（这点与系统调用`socket`的流程不一样），在系统调用`accept`中只是将上层的`struct socket`与这个`sk`做了关联而已（实现函数为[`sock_graft`](https://elixir.bootlin.com/linux/v4.11.6/source/include/net/sock.h#L1705)）

还需要记住的一个细节，内核创建`struct sock`结构时同样是调用[`sock_init_data`](https://elixir.bootlin.com/linux/v4.11.6/source/net/core/sock.c#L2460)完成，注意其中的`sk->sk_data_ready = sock_def_readable;`这段代码，这段代码的意义是告诉内核，当前的`sk`上有数据了，该怎么处理（调用`sock_def_readable`回调函数）

```CPP
struct sock *inet_csk_accept(struct sock *sk, int flags, int *err, bool kern)
{
    struct inet_connection_sock *icsk = inet_csk(sk);
    struct request_sock_queue *queue = &icsk->icsk_accept_queue;
    struct request_sock *req;
    struct sock *newsk;

	// ......

    /* 从已完成连接队列中移除 */
    req = reqsk_queue_remove(queue, sk);

    /* 设置新控制块指针 */
	/* sk为 struct sock *sk 类型*/
    newsk = req->sk;

	// ......

    /* 返回找到的连接控制块 */
    return newsk;
}
```

```CPP
void sock_init_data(struct socket *sock, struct sock *sk)
{
    sk->sk_wq   =   NULL;
    sk->sk_data_ready   =   sock_def_readable;
}
```

##  0x04    epoll_ctl 实现：操作fd
这一步是`epoll`机制的核心，在上面的示例代码中，通过`epoll_ctl`操作网络连接socket fd的内核大致过程描述如下：

1.	分配一个红黑树节点对象 `epitem`
2.	添加等待事件到 socket 的等待队列中，其回调函数是 `ep_poll_callback`
3.  将 `epitem` 插入到 epoll 对象管理结构`eventpoll`的红黑树

####    相关数据结构
`epoll_ctl`操作`eventpoll`对象时会涉及到多个数据结构：

-	`struct epoll_filefd`：
-   `struct epitem`：
-   `struct eppoll_entry`：
-	[`struct poll_table`](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/poll.h#L40)
-	[`ep_pqueue`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L248)


1、`epitem`，从其成员定义易知，这就是红黑树的元素节点

```cpp
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L141
struct epitem {
	union {
		/* RB tree node links this structure to the eventpoll RB tree */
		struct rb_node rbn;
		/* Used to free the struct epitem */
		struct rcu_head rcu;
	};

	/* List header used to link this structure to the eventpoll ready list */
	struct list_head rdllink;

	/*
	 * Works together "struct eventpoll"->ovflist in keeping the
	 * single linked chain of items.
	 */
	struct epitem *next;

	/* The file descriptor information this item refers to */
	struct epoll_filefd ffd;

	/* Number of active wait queue attached to poll operations */
	int nwait;

	/* List containing poll wait queues */
	struct list_head pwqlist;

	/* The "container" of this item */
	struct eventpoll *ep;

	/* List header used to link this item to the "struct file" items list */
	struct list_head fllink;

	/* wakeup_source used when EPOLLWAKEUP is set */
	struct wakeup_source __rcu *ws;

	/* The structure that describe the interested events and the source fd */
	struct epoll_event event;
};
```

2、`eppoll_entry`

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L230
struct eppoll_entry {
	/* List header used to link this structure to the "struct epitem" */
	struct list_head llink;

	/* The "base" pointer is set to the container "struct epitem" */
	struct epitem *base;

	/*
	 * Wait queue item that will be linked to the target file wait
	 * queue head.
	 */
	wait_queue_t wait;

	/* The wait queue head that linked the "wait" wait queue item */
	wait_queue_head_t *whead;
};
```

3、`poll_table`

```CPP
typedef struct poll_table_struct {
	poll_queue_proc _qproc;
	unsigned long _key;
} poll_table;
```

4、`ep_pqueue`

```CPP
/* Wrapper struct used by poll queueing */
struct ep_pqueue {
	poll_table pt;
	struct epitem *epi;
};
```

####    `epoll_ctl`系统调用
`epoll_ctl`的主要过程如下：

1.	根据传入参数 `ebpf`、`fd` 找到 `eventpoll`、`socket`相关的内核对象
2.	调用`ep_*`完成`fd`关联的socket/sock结构的注册（等待队列的事件就绪回调函数）
3.	操作rbtree

对于`EPOLL_CTL_ADD`选项而言，主要调用函数链为`epoll_ctl()->ep_insert()`

```CPP
// epoll_ctl的参数包括两个fd
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
        struct epoll_event __user *, event)
{
    struct eventpoll *ep;
    struct file *file, *tfile;

    //根据 epfd 找到 eventpoll 内核对象
    file = fget(epfd);
    ep = file->private_data;	// 又一次看到了private_data

    //根据 socket 句柄号， 找到其 file 内核对象
    tfile = fget(fd);

    switch (op) {
    case EPOLL_CTL_ADD:
        if (!epi) {
            epds.events |= POLLERR | POLLHUP;
			// for add
            error = ep_insert(ep, &epds, tfile, fd);
        } else
            error = -EEXIST;
        clear_tfile_check_list();
        break;

    //.....
}
```

####    EPOLL_CTL_ADD的过程（重点）
[`ep_insert`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L1293)又是`epoll_ctl`函数的核心实现过程，所有的注册都是在这个函数中完成的。`ep_insert`的核心逻辑：

1.	将需要监听的fd（不区分listenerfd或acceptfd）包装为`epitem`对象，这个结构是作为`eventpoll`管理结构红黑树上的某个树节点
2.	`ep_insert`的最终目的是让告诉内核，这个`fd`上面发生了事件（读写错误等），要怎么处理。所以涉及到事件及事件的回调方法注册，这里还包含了一个隐藏的细节，当`fd`有事件发生时，还需要具备（内核）能够通过这个`fd`找到其归属的`task_struct`（进程），然后唤醒它，让它来干活（处理连接）了
3.	根据上面的描述，所以`ep_insert`就会利用内核的机制（如waitqueue）把上面的工作完成
4.	内核还需要考虑，如何在海量fd的集合中快速的定位到`fd`节点（`epoll_ctl`针对fd的CRUD操作）

```CPP
static int ep_insert(struct eventpoll *ep, 
                struct epoll_event *event,
                struct file *tfile, int fd)
{
    //1、分配并初始化 epitem
    struct epitem *epi;	    //分配一个epi对象
    if (!(epi = kmem_cache_alloc(epi_cache, GFP_KERNEL)))
        return -ENOMEM;

    //对分配的epi进行初始化
    //epi->ffd中存了句柄号和struct file对象地址
    INIT_LIST_HEAD(&epi->pwqlist);

	// 设置epitem的ep成员指向eventpoll管理对象
    epi->ep = ep;

	// 设置epitem的ffd 指向要添加的fd 相关对象
    ep_set_ffd(&epi->ffd, tfile, fd);

    //2、设置 socket 等待队列
    //定义并初始化 ep_pqueue 对象
    struct ep_pqueue epq;
    epq.epi = epi;
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);

    //调用 ep_ptable_queue_proc 注册回调函数 
    //实际注入的函数为 ep_poll_callback
    revents = ep_item_poll(epi, &epq.pt);

    ......
    //3、将epi插入到 eventpoll 对象中的红黑树中
    ep_rbtree_insert(ep, epi);
    ......
}
```

对于上面的代码，这里拆开来分析：

1、初始化`epitem`，建立成员关系

```CPP
// 参数ffd为 新建的epitem成员
// 参数file/fd为 要添加监听的socket成员（epoll_ctl的操作对象）
static inline void ep_set_ffd(struct epoll_filefd *ffd,
                        struct file *file, int fd)
{
    ffd->file = file;
    ffd->fd = fd;
}
```

2、设置`epoll_ctl`监控fd（socket）对象的`struct socket/sock`的等待队列（非常重要），

**在创建 `epitem` 并初始化之后，`ep_insert` 会设置 socket 对象上的等待任务队列，并把函数 `ep_poll_callback` 设置为数据就绪时候的回调函数**，这里需要搞懂几个细节

-	`init_poll_funcptr`的参数是什么？
-	`ep_ptable_queue_proc`是什么？
-	内核设计`struct ep_pqueue epq`这个结构的意义是什么？

```CPP
static int ep_insert(...)
{
    ...
	// 新建一个 ep_pqueue 对象
    struct ep_pqueue epq;
	// 将新建的epitem挂在 epq上
	// 记住ep_pqueue结构包含了两个成员：
	// 1、poll_table pt
	// 2、struct epitem *epi
    epq.epi = epi;
    init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
    ...
	revents = ep_item_poll(epi, &epq.pt);
	// ......
}

static inline void init_poll_funcptr(poll_table *pt, 
    poll_queue_proc qproc)
{
    pt->_qproc = qproc;	// 这里的qproc就是ep_ptable_queue_proc
    pt->_key   = ~0UL; /* all events enabled */
}
```

![ep_pqueue_relation]()

3、`ep_item_poll`在`init_poll_funcptr`之后被调用，注意它的参数是`struct ep_pqueue`的两个成员

-	`ep_item_poll`中，走到`epi->ffd.file->f_op->poll`调用，对应的函数是`sock_poll`，这个`epi`是关联`epoll_ctl`操作的那个fd
-	`sock_poll`中，走到`sock->ops->poll`调用，对应的函数是`tcp_poll`
-	`tcp_poll`中，走到`sock_poll_wait`调用

```CPP
static inline unsigned int ep_item_poll(struct epitem *epi, poll_table *pt)
{
	//......
    pt->_key = epi->event.events;

	// 重要：epi->ffd.file->f_op 找到被操作的fd的f_op成员
	// sock_poll
    return epi->ffd.file->f_op->poll(epi->ffd.file, pt) & epi->event.events;
}

static unsigned int sock_poll(struct file *file, poll_table *wait)
{
    ...
	//tcp_poll
    return sock->ops->poll(file, sock, wait);
}

unsigned int tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
{
    struct sock *sk = sock->sk;
	// ......
    sock_poll_wait(file, sk_sleep(sk), wait);
}
```

在进入`sock_poll_wait`函数之前，先看下`sk_sleep(sk)`获取的是啥结构？从实现来看，返回结果为`wait_queue_head_t *`结构，在此函数它获取了 `struct sock` 对象下的等待队列列表头 `wait_queue_head_t`，待会等待队列项就插入这里，还是需要强调两点：

-	这个`struct sock`是`epoll_ctl`函数操作的目标fd（listenerfd或acceptfd）
-	这个等待队列是 socket/sock 的等待队列，非 epoll 对象，二者的作用完全不同（见附录）

```CPP
static inline wait_queue_head_t *sk_sleep(struct sock *sk)
{
    BUILD_BUG_ON(offsetof(struct socket_wq, wait) != 0);
    return &rcu_dereference_raw(sk->sk_wq)->wait;
}
```

4、分析下`sock_poll_wait->poll_wait`的实现，其参数为：

-	`file *filp`：`p->_qproc`的回调参数之一，需要操作哪个fd的file结构
-	`wait_queue_head_t *wait_address`：等待队列的头部
-	`poll_table *p`：需要使用到`p->_qproc`成员

终于在`poll_wait`中看到调用了前面在`init_poll_funcptr`函数中注册的回调函数`ep_ptable_queue_proc`

```CPP
static inline void sock_poll_wait(struct file *filp,
        wait_queue_head_t *wait_address, poll_table *p)
{	
	//poll_wait
    poll_wait(filp, wait_address, p);
}

static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
    if (p && p->_qproc && wait_address)
		// qproc 为函数指针，
		// 在前面的 init_poll_funcptr 调用时被设置成了 ep_ptable_queue_proc 函数
		// 调用其实是ep_ptable_queue_proc方法
        p->_qproc(filp, wait_address, p);
}
```

所以，`ep_insert`最终的逻辑是在 `ep_ptable_queue_proc` 函数中，新建了一个等待队列项，并注册其回调函数为 `ep_poll_callback` 函数，然后再将这个等待项添加到fd关联的 socket 的等待队列中

5、回调函数`ep_ptable_queue_proc`真正完成了socket等待队列的初始化及添加等工作，注意到参数`whead`是socket等待队列的头结点，等待队列项最终会通过`whead`插入

```CPP
//file: fs/eventpoll.c
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead,
                 poll_table *pt)
{
	//新建一个eppoll_entry对象
    struct eppoll_entry *pwq;
    f (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
		//初始化回调方法
		init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);

		//将ep_poll_callback放入socket的等待队列whead（注意不是epoll的等待队列）
		add_wait_queue(whead, &pwq->wait);
    }
	//....
}
```

`ep_ptable_queue_proc`调用`init_waitqueue_func_entry`初始化一个等待队列项，等待队列项中仅仅只设置了回调函数 `q->func` 为 `ep_poll_callback`，其定义在[`ep_poll_callback`](https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L1004)

在内核软中断ksoftirqd将数据收到 socket 的接收队列后，内核会唤醒这个socket等待队列上的所有项，会通过注册的这个 `ep_poll_callback` 函数来回调（依次调用其注册的回调函数），进而通知到 `epoll` 对象

此外，这里思考下为何在epoll机制下的`init_waitqqueue_func_entry`中`q->private`要设置为`NULL`？首先这个`private`的意义是的内核实现等待队列机制中，当等待条件可能就绪时内核要唤醒的进程结构`task_struct`，但是**在epoll机制中，socket是由epoll统一管理的，不需要在一个 socket 就绪的时候就唤醒进程（这样效率也极低），所以这里的 `q->private` 没意义就设置成了 NULL**

```CPP
static inline void init_waitqqueue_func_entry(
    wait_queue_t *q, wait_queue_func_t func)
{
    q->flags = 0;
	// 这里为什么是NULL？
    q->private = NULL;

    //ep_poll_callback 注册到 wait_queue_t对象上
    //有数据到达的时候调用 q->func
    q->func = func;   
}
```

总结下`epoll_ctl(ADD)`的过程：

1.	建立epoll机制的事件结构，并为其建立关联关系（这样某个`sk`上有数据到达时，内核可以通过此`sk`找到对应的`fd`，从而告知epoll哪些fd上有事件发生了）
2.	为操作参数fd关联的socket的等待队列，注册等待队列项及数据就绪时候的回调函数
3.	操作eventpoll的红黑树，变更信息

![epoll_ctl_flow]()

##  0x05    epoll_wait实现：等待就绪connection

##  0x06    connection到达后的流程

####	如何根据sk（有数据）找到epitem/eventpoll结构？


##  0x07    番外

####    epoll机制中的等待队列
1、`epoll_wait`中的等待队列机制实现：`epoll_wait->ep_poll`，在`ep_poll`函数的实现中发现了经典的等待->唤醒机制的模式，类似于`wait_event`内核调用中的`condition`参数，这里可以看到`condition`为`ep_events_available`函数的实现，即检查`eventpoll`的就绪队列`rdllist`是否不为空`!list_empty(&ep->rdllist)`，具体流程请看注释

```CPP
// 检查就绪队列是否可用（不为空）
static inline int ep_events_available(struct eventpoll *ep)
{
	return !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
}


//https://elixir.bootlin.com/linux/v4.11.6/source/fs/eventpoll.c#L1614
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events,
		   int maxevents, long timeout)
{
	int res = 0, eavail, timed_out = 0;
	unsigned long flags;
	u64 slack = 0;
	wait_queue_t wait;
	ktime_t expires, *to = NULL;

	if (timeout > 0) {
		struct timespec64 end_time = ep_set_mstimeout(timeout);

		slack = select_estimate_accuracy(&end_time);
		to = &expires;
		*to = timespec64_to_ktime(end_time);
	} else if (timeout == 0) {
		/*
		 * Avoid the unnecessary trip to the wait queue loop, if the
		 * caller specified a non blocking operation.
		 */
		timed_out = 1;
		spin_lock_irqsave(&ep->lock, flags);
		goto check_events;
	}

fetch_events:
	spin_lock_irqsave(&ep->lock, flags);

	if (!ep_events_available(ep)) {
		/*
		 * We don't have any available event to return to the caller.
		 * We need to sleep here, and we will be wake up by
		 * ep_poll_callback() when events will become available.
		 */
        // 1. 先初始化等待队列并移入
		init_waitqueue_entry(&wait, current);
		__add_wait_queue_exclusive(&ep->wq, &wait);

		for (;;) {
			/*
			 * We don't want to sleep if the ep_poll_callback() sends us
			 * a wakeup in between. That's why we set the task state
			 * to TASK_INTERRUPTIBLE before doing the checks.
			 */
			set_current_state(TASK_INTERRUPTIBLE);

            // 2. 检查condition条件是否成立
			if (ep_events_available(ep) || timed_out)
                // 3. 若condition成立，则跳出for(;;)循环
				break;
			if (signal_pending(current)) {
				res = -EINTR;
				break;
			}

			spin_unlock_irqrestore(&ep->lock, flags);
            // 4. 走到这里说明condition不成立，那么就先让出CPU
            // 进程第一调度定律
            // 那么内核执行进程切换，上下文保存到堆栈至此
			if (!schedule_hrtimeout_range(to, slack, HRTIMER_MODE_ABS))
				timed_out = 1;

            //5. 执行到这里大概率说明内核通过wakeup机制唤醒了当前的进程，继续for(;;)循环
            // 下一步是检查condition是否满足
			spin_lock_irqsave(&ep->lock, flags);
		}

        // 6. 已经被唤醒了，且condition也满足了，那么就移除等待队列
        // 且设置当前的进程的运行状态的TASK_RUNNING
		__remove_wait_queue(&ep->wq, &wait);
		__set_current_state(TASK_RUNNING);
	}
check_events:
	/* Is it worth to try to dig for events ? */
	eavail = ep_events_available(ep);

	spin_unlock_irqrestore(&ep->lock, flags);

	/*
	 * Try to transfer events to user space. In case we get 0 events and
	 * there's still timeout left over, we go trying again in search of
	 * more luck.
	 */
	if (!res && eavail &&
	    !(res = ep_send_events(ep, events, maxevents)) && !timed_out)
		goto fetch_events;

	return res;
}
```

##	0x08	惊群问题（`epoll_wait`+IO多路复用机制下）

####	惊群问题的本质

####	解决方案一：EPOLLEXCLUSIVE选型

####	解决方案二：SO_REUSEPORT技术

[`reuseport_select_sock`](https://elixir.bootlin.com/linux/v4.11.6/source/net/core/sock_reuseport.c#L196)

##  0x09    总结

####    epoll 事件处理流程的完整步骤
1、构建内核`eventpoll`结构，其中包含了 epoll进程的等待队列`wq`、fd（`epitem`）管理结构rbtree（`rbr`）以及事件就绪链表`rdllist`等核心成员，并初始化这些成员

2、注册监听（epoll_ctl），当调用 `epoll_ctl(EPOLL_CTL_ADD, fd)` 时，内核创建 `epitem` 结构体（封装 `fd` 及其监听事件），插入到 `eventpoll` 的红黑树（`rbr`）中，实现 `O(logN)` 的高效查找。这里有个关键流程是内核会同时创建 `eppoll_entry` 结构体，它作为桥梁，一端指向当前 `epitem`，另一端会通过 `wait_queue_t` 挂载到 socket 关联的 `struct sock` 的等待队列（`socket.sock->sk_wq`）上，这样当该socket（fd）发生可读可写事件时，内核会遍历该socket等待队列，对每个唤醒的结构（`curr->func`），调用其已注册的函数`ep_poll_callback`（在`ep_insert` 调用时会设置该 `func`为 `ep_poll_callback`）

3、事件触发与回调（`ep_poll_callback`）过程，当注册到epoll体系的 socket 发生可读/可写事件时（如收到数据），内核执行以下动作：

-   从 socket 关联的等待队列中获取 `eppoll_entry`，进而找到对应的 `epitem`
-   通过 `epitem` 定位到所属的 `eventpoll` 对象（每个 epoll 实例一个）
-   将 `epitem` 加入 `eventpoll` 的就绪链表（`rdllist`），此处是 `O(1)` 操作
-   唤醒逻辑：检查 `eventpoll->wq`（阻塞在 `epoll_wait` 上的进程等待队列），若有进程等待，则唤醒它处理已经就绪的fds

4、进程处理就绪事件（`epoll_wait`），被唤醒的进程从 `rdllist` 中取出 `epitem`，将其从`rdllist`链表中移除（但仍在红黑树中）,然后将就绪的 fd 和事件拷贝到用户空间（如 `events` 数组），最后返回就绪的 fd 数量，用户进程即可处理这些fds事件


####    Why rbtree？

##  0x0 参考
-	[linux源码解读（十七）：红黑树在内核的应用——epoll](https://www.cnblogs.com/theseventhson/p/15829130.html)
-   [深入揭秘 epoll 是如何实现 IO 多路复用的](https://mp.weixin.qq.com/s/OmRdUgO1guMX76EdZn11UQ)
-	[epoll和惊群](https://plantegg.github.io/2019/10/31/epoll和惊群/)
-	[从内核看SO_REUSEPORT的实现（基于5.9.9）](https://cloud.tencent.com/developer/article/1844163)