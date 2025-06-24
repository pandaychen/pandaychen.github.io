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

2、`epoll_ctl`：向 `epoll` 对象中添加要管理的连接

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
eventpoll的定义如下，这里重点看下面几个成员：

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

##  0x03    服务端accept新连接（fd）


##  0x04    epoll_ctl实现：操作fd

####    相关数据结构
`epoll_ctl`操作`eventpoll`对象时会涉及到多个数据结构：

-   `struct epitem`
-   `struct eppoll_entry`

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

####    `epoll_ctl`系统调用

```CPP
SYSCALL_DEFINE4(epoll_ctl, int, epfd, int, op, int, fd,
        struct epoll_event __user *, event)
{
    struct eventpoll *ep;
    struct file *file, *tfile;

    //根据 epfd 找到 eventpoll 内核对象
    file = fget(epfd);
    ep = file->private_data;

    //根据 socket 句柄号， 找到其 file 内核对象
    tfile = fget(fd);

    switch (op) {
    case EPOLL_CTL_ADD:
        if (!epi) {
            epds.events |= POLLERR | POLLHUP;
            error = ep_insert(ep, &epds, tfile, fd);
        } else
            error = -EEXIST;
        clear_tfile_check_list();
        break;

    //.....
}
```

####    EPOLL_CTL_ADD


##  0x05    epoll_wait实现：等待就绪connection

##  0x06    connection到达后的流程


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

##  0x08    总结

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