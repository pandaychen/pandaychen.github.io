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

##	0x01	socket创建
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


##  0x  参考
-   [深入理解Linux TCP的三次握手](https://mp.weixin.qq.com/s/vlrzGc5bFrPIr9a7HIr2eA)
-   [为什么服务端程序都需要先 listen 一下](https://mp.weixin.qq.com/s/hv2tmtVpxhVxr6X-RNWBsQ)
-	[Linux内核网络（三）：Linux内核中socket函数的实现](https://www.kerneltravel.net/blog/2020/network_ljr_no3/)