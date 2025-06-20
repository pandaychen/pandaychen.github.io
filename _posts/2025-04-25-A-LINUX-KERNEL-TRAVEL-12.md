---
layout:     post
title:  Linux 内核之旅（十二）：内核视角下的三次握手
subtitle:   
date:       2025-04-25
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言

![client-to-server]()

服务端代码：

```CPP
int main(int argc, char const *argv[])
{
	int fd = socket(AF_INET, SOCK_STREAM, 0);
	bind(fd, ...);
	listen(fd, 128);
	accept(fd, ...);
	//handler fd
}
```

##	0x01	server：socket实现
当调用`socket`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/net/socket.c#L1258)创建`struct socket`结构时，在用户层视角只看到返回了一个文件描述符 fd，内核做了哪些事情？

####	socket调用的细节

![socket-api-flow]()

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/af_inet.c#L1014
static const struct net_proto_family inet_family_ops = {
	.family = PF_INET,
	.create = inet_create,
	.owner	= THIS_MODULE,
};

SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
	int retval;
	struct socket *sock;
	int flags;

	//...

	// 对AF_INET，这里的sock_create对应的是inet_create
	retval = sock_create(family, type, protocol, &sock);
	if (retval < 0)
		goto out;

	retval = sock_map_fd(sock, flags & (O_CLOEXEC | O_NONBLOCK));
	if (retval < 0)
		goto out_release;

	//...
}
```
`socket`主要完成：

-	调用`sock_create->__sock_create`，新建一个`struct socket`及相关内容
-	调用`sock_map_fd`，新建一个`struct file` 并将`file`的`private_data`初始化为上一步创建的`struct socket`，这样对文件的操作可以调用`socket`结构体定义的方法，并关联`fd`和`file`

`__socket_create`函数主要工作如下：

-	调用`sock_alloc` 分配一个`struct socket`结构体和`inode`，并且标明`inode`是`socket`类型，这样对`inode`的操作最终可以调用`socket`的相关操作
-	根据输入参数，查找`net_families`数组（该数组通过`inet_init`创建），获得域特定的`socket`创建函数
-	调用实际`create`函数新建，如`inet_create`

```CPP
//sock_alloc
struct socket *sock_alloc(void)
{
	struct inode *inode;
	struct socket *sock;

    /*创建inode和socket*/
	inode = new_inode_pseudo(sock_mnt->mnt_sb);
	if (!inode)
		return NULL;

    /*返回创建的socket指针*/
	sock = SOCKET_I(inode);

    /*inode相关初始化*/
	inode->i_ino = get_next_ino();
	inode->i_mode = S_IFSOCK | S_IRWXUGO;
	inode->i_uid = current_fsuid();
	inode->i_gid = current_fsgid();
	inode->i_op = &sockfs_inode_ops;

	return sock;
}
EXPORT_SYMBOL(sock_alloc);

int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
	int err;
	struct socket *sock;
	const struct net_proto_family *pf;

	//...
	sock = sock_alloc();	/*创建struct socket结构体*/
	//...
	sock->type = type;	/*设置套接字类型*/

	rcu_read_lock();
	pf = rcu_dereference(net_families[family]);	/*获取对应协议族的协议实例对象*/
	err = -EAFNOSUPPORT;
	if (!pf)
		goto out_release;

	//...
	err = pf->create(net, sock, protocol, kern);
	if (err < 0)
		goto out_module_put;
	//...
}
EXPORT_SYMBOL(__sock_create);
```

对于`__sock_create`中的`pf->create`函数，其中`pf`由`net_families[]`数组获得，`net_families[]`数组里存放了各个协议族的信息，以`family`字段作为下标。`net_families[]`数组定义及初始化代码如下：

```CPP
static DEFINE_SPINLOCK(net_family_lock);
static const struct net_proto_family __rcu *net_families[NPROTO] __read_mostly;

static const struct net_proto_family inet_family_ops = {
    .family = PF_INET,
    .create = inet_create,
    .owner  = THIS_MODULE,
};

//net_families[]数组的初始化在inet_init函数
static int __init inet_init(void)
{
...
    (void)sock_register(&inet_family_ops);
...
}

//注册
int sock_register(const struct net_proto_family *ops)
{
...
    rcu_assign_pointer(net_families[ops->family], ops);
...
}
```

TCP协议对应的`family`字段是`AF_INET`，`pf->create`对应的函数即为`inet_create`，核心逻辑如下：

```CPP
static int inet_create(struct net *net, struct socket *sock, int protocol,
		       int kern)
{
	struct sock *sk;

	//socket 状态设置
	sock->state = SS_UNCONNECTED;

	/* Look for the requested type/protocol pair. */
	//查找全局数组inetsw（在inet_init函数中初始化）中对应的协议操作集合，最重要的是struct proto和struct proto_ops，分别用于处理四层和socket相关的内容
lookup_protocol:
	err = -ESOCKTNOSUPPORT;
	rcu_read_lock();
	list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {

		err = 0;
		/* Check the non-wild match. */
		if (protocol == answer->protocol) {
			if (protocol != IPPROTO_IP)
				break;
		} else {
			/* Check for the two wild cases. */
			if (IPPROTO_IP == protocol) {
				protocol = answer->protocol;
				break;
			}
			if (IPPROTO_IP == answer->protocol)
				break;
		}
		err = -EPROTONOSUPPORT;
	}

	//调用sk_alloc()，分配一个struct sock，并将proto类型的指针指向第二步获得的内容
	sk = sk_alloc(net, PF_INET, GFP_KERNEL, answer_prot, kern);
	if (!sk)
		goto out;

	err = 0;
	if (INET_PROTOSW_REUSE & answer_flags)
		sk->sk_reuse = SK_CAN_REUSE;
	
	//初始化inet_sock，调用sock_init_data，形成socket和sock一一对应的关系，相互有指针指向对方
	inet = inet_sk(sk);
	sock_init_data(sock, sk);

	sk->sk_destruct	   = inet_sock_destruct;
	sk->sk_protocol	   = protocol;
	sk->sk_backlog_rcv = sk->sk_prot->backlog_rcv;

	inet->uc_ttl	= -1;
	inet->mc_loop	= 1;
	inet->mc_ttl	= 1;
	inet->mc_all	= 1;
	inet->mc_index	= 0;
	inet->mc_list	= NULL;
	inet->rcv_tos	= 0;

	//...

	//最后调用proto中注册的init函数，err = sk->sk_prot->init(sk)，如果对应于TCP，其函数指针指向tcp_v4_init_sock
	if (sk->sk_prot->init) {
		err = sk->sk_prot->init(sk);
		if (err) {
			sk_common_release(sk);
			goto out;
		}
	}
	//...
}
```

`socket`函数最后的逻辑是调用`sock_map_fd`函数负责分配文件，并与`struct socket`进行绑定，主要做两件事：

```CPP
static int sock_map_fd(struct socket *sock, int flags)
{
	struct file *newfile;
    //分配文件描述符
	int fd = get_unused_fd_flags(flags);
	if (unlikely(fd < 0)) {
		sock_release(sock);
		return fd;
	}

	//调用sock_alloc_file，分配一个struct file，并将私有数据指针指向socket结构
	newfile = sock_alloc_file(sock, flags, NULL);
	if (likely(!IS_ERR(newfile))) {
		//关联文件描述符fd和file
		fd_install(fd, newfile);
		return fd;
	}

	put_unused_fd(fd);
	return PTR_ERR(newfile);
}
```

由于socket也是文件，所以基于VFS的这套框架，各个成员有如下关系：

![socket-relation]()


这里多说一句，内核在`accept`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/net/socket.c#L1489)中也会创建`struct socket`结构，这两个具体的执行流程是不同的

最后，小结下创建socket结构时，内核会：
-	创建接收队列`sk_receive_queue`，用于接收软中断softirq时存储对应的数据包
-	等待队列`sk_wq`，当连接完成后，如果当前没有数据到来，那么当前进程会阻塞，并且状态从运行态切换至阻塞（主动让出CPU），并且当前进程关联的socket存储在该队列中，等到有数据到来的时候，内核再通过该队列中获取对应的进程将其唤醒
-	软中断处理函数`sk_data_ready`，会直接将软中断的回调函数注册好，当数据到来的时候，调用该方法来处理
-	协议族函数`proto_ops`，内核会将一系列内核协议栈相关的处理函数提前注册好，比如针对`AF_INET`注册的是`inet_create`
-	初始化`struct sock`结构内部的相关队列信息

##	0x02	server：listen实现
`listen`[系统调用](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/af_inet.c#L194)的主要作用就是申请和初始化接收队列，包括全连接队列（链表）和半连接队列（hash表），如图

![listen]()

```CPP
SYSCALL_DEFINE2(listen, int, fd, int, backlog)
{
	//...
	//根据 fd 查找 socket 内核对象
	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (sock) {
	//获取内核参数 net.core.somaxconn
	somaxconn = sock_net(sock->sk)->core.sysctl_somaxconn;
	if ((unsigned int)backlog > somaxconn)
	backlog = somaxconn;
	
	//调用协议栈注册的 listen 函数：inet_listen
	err = sock->ops->listen(sock, backlog);
	//...
}
```

`sock->ops->listen` 调用的是 `inet_listen`函数：

```CPP
int inet_listen(struct socket *sock, int backlog)
{
 //还不是 listen 状态（尚未 listen 过）
 if (old_state != TCP_LISTEN) {
  //开始监听
  err = inet_csk_listen_start(sk, backlog);
 }

 //设置全连接队列长度
 sk->sk_max_ack_backlog = backlog;
}
```

[`inet_csk_listen_start`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/inet_connection_sock.c#L864)，其中`icsk->icsk_accept_queue` 定义在 `inet_connection_sock`（类型为`request_sock_queue`），是内核用来接收客户端请求的主要数据结构，其中包含了重要的全连接队列`request_sock`结构成员`rskq_accept_head`和`rskq_accept_tail`，这里**注意对于全连接队列来说，在它上面不需要进行复杂的查找工作，accept 的时候只是先进先出处理就好了，因此全连接队列通过 `rskq_accept_head` 和 `rskq_accept_tail` 以链表的形式来管理**，而半连接队列由于需要快速的查找，所以使用hash表来实现

```CPP
//https://elixir.bootlin.com/linux/v4.11.6/source/include/net/request_sock.h#L161
struct request_sock_queue {
	spinlock_t		rskq_lock;
	u8			rskq_defer_accept;

	atomic_t		qlen;
	atomic_t		young;
 	//全连接队列
	struct request_sock	*rskq_accept_head;
	struct request_sock	*rskq_accept_tail;
	//...
};

int inet_csk_listen_start(struct sock *sk, int backlog)
{
	//将 struct sock 对象强制转换成了 inet_connection_sock
	struct inet_connection_sock *icsk = inet_csk(sk);
	struct inet_sock *inet = inet_sk(sk);
	int err = -EADDRINUSE;

	reqsk_queue_alloc(&icsk->icsk_accept_queue);

	sk->sk_max_ack_backlog = backlog;
	sk->sk_ack_backlog = 0;
	inet_csk_delack_init(sk);

	/* There is race window here: we announce ourselves listening,
	 * but this transition is still not validated by get_port().
	 * It is OK, because this socket enters to hash table only
	 * after validation is complete.
	 */
	sk_state_store(sk, TCP_LISTEN);
	if (!sk->sk_prot->get_port(sk, inet->inet_num)) {
		inet->inet_sport = htons(inet->inet_num);

		sk_dst_reset(sk);
		err = sk->sk_prot->hash(sk);

		if (likely(!err))
			return 0;
	}

	sk->sk_state = TCP_CLOSE;
	return err;
}
EXPORT_SYMBOL_GPL(inet_csk_listen_start);
```

在4.11.6版本的`reqsk_queue_alloc`并未发现半连接hash表初始化的代码，事实上该版本的实现已经不同于2.6了，主要区别是：
-	全局整合：移除独立哈希表，半连接请求（`struct request_sock`）直接插入全局连接哈希表 `ehash`，与其他状态的 socket 共用同一hash表
-	无预分配：`reqsk_queue_alloc` 仅初始化锁和全连接队列头，半连接队列无独立内存预分配

####	ehash的初始化
全局 ehash（Established Hash）是 Linux 内核中用于管理所有非 LISTEN 状态的 TCP 连接的核心哈希表（包括 `SYN_RECV`、`ESTABLISHED`、`TIME_WAIT` 等），其初始化发生在内核启动阶段，位于[`tcp_init`](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/tcp.c#L3378)

```cpp
void __init tcp_init(void)
{
	//...
	tcp_hashinfo.ehash =
		alloc_large_system_hash("TCP established",
					sizeof(struct inet_ehash_bucket),
					thash_entries,
					17, /* one slot per 128 KB of memory */
					0,
					NULL,
					&tcp_hashinfo.ehash_mask,
					0,
					thash_entries ? 0 : 512 * 1024);
	for (i = 0; i <= tcp_hashinfo.ehash_mask; i++)
		INIT_HLIST_NULLS_HEAD(&tcp_hashinfo.ehash[i].chain, i);

	if (inet_ehash_locks_alloc(&tcp_hashinfo))
		panic("TCP: failed to alloc ehash_locks");

	//...
}
```


##	0x	总结

#### socket VS	accept
🔧 ​一、socket() 函数中 sock_init_data() 的意义​
当用户调用 socket() 创建套接字时（如监听套接字），内核通过以下流程初始化 struct sock：

​队列初始化​：
sock_init_data() 初始化核心队列
：
​接收队列​（sk_receive_queue）：用于存储接收到的数据包（sk_buff），但监听套接字本身不使用此队列传输数据。
​发送队列​（sk_write_queue）：预留缓存待发送数据，监听套接字通常不主动发送数据。
​等待队列​（sk_sleep）：管理因 I/O 事件（如 accept() 阻塞）而休眠的进程
。
同时设置回调函数（如 sk_data_ready = sock_def_readable），用于数据到达时唤醒进程
。
​监听套接字的特殊性​：
虽然队列被初始化，但监听套接字的核心功能是管理连接，而非数据传输。
实际用于新连接管理的队列（如全连接队列 icsk_accept_queue）在 listen() 时创建
。
​设计意义​：
​统一接口​：所有套接字（无论类型）均初始化相同基础结构，简化内核逻辑。
​预留能力​：为可能的协议扩展或特殊操作（如监听套接字发送错误信息）提供支持。
🔄 ​二、accept() 中 struct sock 的队列作用​
通过 accept() 创建的新套接字关联的 struct sock 是三次握手期间已创建的​（非 accept 新建），其队列作用完全不同：

​队列来源​：
新连接的 struct sock 在握手完成时创建，并加入监听套接字的 icsk_accept_queue 队列
。
accept() 仅将其取出，并与新 socket 结构绑定
。
​队列的核心作用​：
​接收队列​（sk_receive_queue）：存储客户端发送的数据包，用户调用 recv() 时从此队列读取数据
。
​发送队列​（sk_write_queue）：缓存待发送给客户端的数据，由协议栈逐步发送
。
​等待队列​（sk_sleep）：管理因 recv() 或 send() 阻塞的进程（如缓冲区空/满时）
。
​数据传输的载体​：
这些队列是实际数据收发的核心通道，与监听套接字的预留队列有本质区别。
📊 ​三、关键对比：监听套接字 vs. 连接套接字​
​特性​	​监听套接字（socket() 创建）​​	​连接套接字（accept() 返回）​​
​队列初始化时机​	sock_init_data() 在 socket() 中调用	struct sock 在三次握手时创建并初始化队列
​接收/发送队列作用​	预留但不使用	实际存储收发数据
​等待队列触发场景​	accept() 阻塞时休眠	recv()/send() 阻塞时休眠
​核心功能队列​	全连接队列（icsk_accept_queue）	sk_receive_queue/sk_write_queue
mermaid
复制
graph TD
  A[socket() 创建监听套接字] --> B[sock_init_data() 初始化基础队列]
  B --> C[队列预留但不用于数据传输]
  C --> D[listen() 创建全连接队列 icsk_accept_queue]
  D --> E[三次握手完成时创建新连接的 struct sock]
  E --> F[新 sock 的队列用于实际数据传输]
  F --> G[accept() 取出 sock 并关联新 socket]
💎 ​四、总结：设计哲学与一致性​
​**sock_init_data() 的意义**​：
在 socket() 中为所有套接字提供统一的基础设施，确保协议无关层的完整性。
对监听套接字而言，队列是“占位符”，实际功能由协议专属队列（如 icsk_accept_queue）实现
。
​与 accept() 队列的一致性​：
​结构相同​：二者均包含同名队列（如 sk_receive_queue），但作用完全独立。
​分工明确​：
监听套接字：​连接管理​（通过全连接队列）。
连接套接字：​数据传输​（通过收/发队列）。
​性能优化​：
复用握手阶段创建的 struct sock 减少 accept() 开销
。
分离队列职责避免资源竞争，提升并发能力。
💎 ​核心结论​：sock_init_data() 在 socket() 中的初始化是基础性、通用性的，而 accept() 关联的队列是功能性、专用性的。二者名称相同但角色迥异，体现了 Linux 网络栈“统一接口，分层实现”的设计智慧。

##  0x  参考
-   [深入理解Linux TCP的三次握手](https://mp.weixin.qq.com/s/vlrzGc5bFrPIr9a7HIr2eA)
-   [为什么服务端程序都需要先 listen 一下](https://mp.weixin.qq.com/s/hv2tmtVpxhVxr6X-RNWBsQ)
-	[Linux内核网络（三）：Linux内核中socket函数的实现](https://www.kerneltravel.net/blog/2020/network_ljr_no3/)