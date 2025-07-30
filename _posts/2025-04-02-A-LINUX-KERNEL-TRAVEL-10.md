---
layout:     post
title:  Linux 内核之旅（十）：内核数据包发送
subtitle:   基础知识汇总与可观测
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

##  0x01   报文发送过程
本小节使用以太网的物理网卡，以一个UDP包的发送过程作为示例，了解下具体的发包过程

####    socket层
1、`socket()`：创建一个UDP socket结构体，并初始化相应的UDP操作函数

2、`sendto(sock, ...)`：应用层的程序（Application）调用该函数开始发送数据包，该函数会进而调用`inet_sendmsg`

3、`inet_sendmsg`：该函数主要是检查当前socket有无绑定源端口，如果没有的话，调用`inet_autobind`分配一个，然后调用UDP层的函数

4、`inet_autobind`：该函数会调用socket上绑定的`get_port`函数获取一个可用端口，由于该socket是UDP的socket，所以`get_port`函数会调到UDP内核实现里面的相应函数

```TEXT
               +-------------+
               | Application |
               +-------------+
                     |
                     |
                     ↓
+------------------------------------------+
| socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP) |
+------------------------------------------+
                     |
                     |
                     ↓
           +-------------------+
           | sendto(sock, ...) |
           +-------------------+
                     |
                     |
                     ↓
              +--------------+
              | inet_sendmsg |
              +--------------+
                     |
                     |
                     ↓
             +---------------+
             | inet_autobind |
             +---------------+
                     |
                     |
                     ↓
               +-----------+
               | UDP layer |
               +-----------+

```

####    UDP层

```TEXT
                     |
                     |
                     ↓
              +-------------+
              | udp_sendmsg |
              +-------------+
                     |
                     |
                     ↓
          +----------------------+
          | ip_route_output_flow |
          +----------------------+
                     |
                     |
                     ↓
              +-------------+
              | ip_make_skb |
              +-------------+
                     |
                     |
                     ↓
         +------------------------+
         | udp_send_skb(skb, fl4) |
         +------------------------+
                     |
                     |
                     ↓
                +----------+
                | IP layer |
                +----------+
```

####    IP层

```TEXT
          |
          |
          ↓
   +-------------+
   | ip_send_skb |
   +-------------+
          |
          |
          ↓
  +-------------------+       +-------------------+       +---------------+
  | __ip_local_out_sk |------>| NF_INET_LOCAL_OUT |------>| dst_output_sk |
  +-------------------+       +-------------------+       +---------------+
                                                                  |
                                                                  |
                                                                  ↓
 +------------------+        +----------------------+       +-----------+
 | ip_finish_output |<-------| NF_INET_POST_ROUTING |<------| ip_output |
 +------------------+        +----------------------+       +-----------+
          |
          |
          ↓
  +-------------------+      +------------------+       +----------------------+
  | ip_finish_output2 |----->| dst_neigh_output |------>| neigh_resolve_output |
  +-------------------+      +------------------+       +----------------------+
                                                                   |
                                                                   |
                                                                   ↓
                                                           +----------------+
                                                           | dev_queue_xmit |
                                                           +----------------+
```

####    netdevice子系统

```TEXT
                          |
                          |
                          ↓
                   +----------------+
  +----------------| dev_queue_xmit |
  |                +----------------+
  |                       |
  |                       |
  |                       ↓
  |              +-----------------+
  |              | Traffic Control |
  |              +-----------------+
  | loopback              |
  |   or                  +--------------------------------------------------------------+
  | IP tunnels            ↓                                                              |
  |                       ↓                                                              |
  |            +---------------------+  Failed   +----------------------+         +---------------+
  +----------->| dev_hard_start_xmit |---------->| raise NET_TX_SOFTIRQ |- - - - >| net_tx_action |
               +---------------------+           +----------------------+         +---------------+
                          |
                          +----------------------------------+
                          |                                  |
                          ↓                                  ↓
                  +----------------+              +------------------------+
                  | ndo_start_xmit |              | packet taps(AF_PACKET) |
                  +----------------+              +------------------------+
```

####    Device Driver

##  0x02    内核数据发送：详细分析
![send-data](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/how-to-send-data-1.png)

##     0x0    内核数据发送
本小节基于TCP三次握手完成，通过accept获取到客户端的连接fd，基于这个fd发送数据的场景进行分析

####   1、accept 获取fd完成的布局


####   2、send* 系统调用
不管是`send`、`sendto`、[`sendmsg`](https://elixir.bootlin.com/linux/v4.11.6/source/net/socket.c#L1921)等系统调用，最终都会调用`sock_sendmsg`，主要完成：

1. 通过fd在内核中定位到对应的 socket/sock 结构对象，在这个对象里记录着各种协议栈的函数地址（在生成fd的时候就已经初始化好了）
2. 构造一个 `struct msghdr` 对象，把用户传入的数据，比如 buffer地址、数据长度等等都设置进去
3. 调用`sock_sendmsg`，即协议栈对应的函数 `inet_sendmsg` ，其中 `inet_sendmsg` 函数地址是通过 socket 内核对象里的 `ops` 成员找到的

以`sendto`[系统调用](https://elixir.bootlin.com/linux/v4.11.6/source/net/socket.c#L1664)为例：

```CPP
SYSCALL_DEFINE6(sendto, int, fd, void __user *, buff, size_t, len,
		unsigned int, flags, struct sockaddr __user *, addr,
		int, addr_len)
{
	struct socket *sock;
	struct sockaddr_storage address;
	int err;
	struct msghdr msg;
	struct iovec iov;
	
       ......
       // 根据fd定位socket/sock结构
	sock = sockfd_lookup_light(fd, &err, &fput_needed);
	if (!sock)
		goto out;

       // 设置msghdr对象
	msg.msg_name = NULL;
	msg.msg_control = NULL;
	msg.msg_controllen = 0;
	msg.msg_namelen = 0;
	if (addr) {
		err = move_addr_to_kernel(addr, addr_len, &address);
		if (err < 0)
			goto out_put;
		msg.msg_name = (struct sockaddr *)&address;
		msg.msg_namelen = addr_len;
	}
	if (sock->file->f_flags & O_NONBLOCK)
		flags |= MSG_DONTWAIT;
	msg.msg_flags = flags;
	err = sock_sendmsg(sock, &msg);

out_put:
	fput_light(sock->file, fput_needed);
out:
	return err;
}
```

继续`sock_sendmsg -> sock_sendmsg_nosec`：

```CPP
static inline int sock_sendmsg_nosec(struct socket *sock, struct msghdr *msg)
{
       // sendmsg对应 inet_sendmsg
       // 该函数是 AF_INET 协议族提供的通用发送函数
	int ret = sock->ops->sendmsg(sock, msg, msg_data_left(msg));
	BUG_ON(ret == -EIOCBQUEUED);
	return ret;
}
```

####   3、传输层处理

####   4、网络层发送处理

####   5、邻居子系统

####   6、网络设备子系统

####   7、软中断调度

####   8、igb 网卡驱动发送

####   9、发送完成硬中断


##  0x0  参考
-   [Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data/)
-   [Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)
-   [Linux网络 - 数据包的发送过程](https://segmentfault.com/a/1190000008926093)
-   [图解 Linux 网络包发送过程](https://zhuanlan.zhihu.com/p/373060740)