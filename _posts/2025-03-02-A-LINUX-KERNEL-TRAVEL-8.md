---
layout:     post
title:  Linux 内核之旅（八）：内核数据包接收
subtitle:   基础知识汇总
date:       2025-03-02
author:     pandaychen
header-img:
catalog: true
tags:
    - Linux
    - Kernel
---

##  0x00    前言
笔者最近在研究基于ebpf的网络协议栈可观测及tracing，本文对协议栈的数据处理基础做了若干总结

本文代码基于 [v4.11.6](https://elixir.bootlin.com/linux/v4.11.6/source/include) 版本

##  0x01   网卡的报文接收过程
一些背景知识：

-   网卡驱动是加载到内核中的模块，负责衔接网卡和内核的网络模块，驱动在加载的时候将自己注册进网络模块，当相应的网卡收到数据包时，网络模块会调用相应的驱动程序处理数据
-   常见的intel网卡：igb（网卡，其中的 i 是 intel，gb 表示每秒 1Gb）、ixgbe（xgb 表示 10Gb，e 表示以太网）、i40e（intel 40Gbps 以太网）

![recv-arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/kernel_packet.jpg)

本小节使用以太网的物理网卡结合一个UDP packet的接收过程为例子描述下内核收包过程，如下：

一、阶段1：数据包从网卡到内存

下图展示了数据包（packet）如何进入内存，并被内核的网络模块开始处理

```TEXT
                   +-----+
                   |     |                            Memroy
+--------+   1     |     |  2  DMA     +--------+--------+--------+--------+
| Packet |-------->| NIC |------------>| Packet | Packet | Packet | ...... |
+--------+         |     |             +--------+--------+--------+--------+
                   |     |<--------+
                   +-----+         |
                      |            +---------------+
                      |                            |
                    3 | Raise IRQ                  | Disable IRQ
                      |                          5 |
                      |                            |
                      ↓                            |
                   +-----+                   +------------+
                   |     |  Run IRQ handler  |            |
                   | CPU |------------------>| NIC Driver |
                   |     |       4           |            |
                   +-----+                   +------------+
                                                   | 
                                                6  | Raise soft IRQ
                                                   |
                                                   ↓
```

0、系统初始化时，网卡驱动程序会向内核申请一块内存（RingBuffer），用于存储未来到达的网络数据包；网卡驱动程序将申请的RingBuffer地址告诉网卡

1、数据包由外部网络进入物理网卡，如果目的地址非该网卡，且该网卡未开启混杂（promiscuous）模式，该包会被网卡丢弃

2、网卡将数据包通过DMA（Direct Memory Access）的方式写入到指定的内存地址，该地址由网卡驱动分配并初始化（网卡会通过DMA将数据拷贝到RingBuffer中，此过程不需要cpu参与）

3、网卡通过硬件中断（IRQ）通知CPU，告诉CPU有数据来了，CPU必须最高优先级处理，否则数据待会存不下了

4、CPU根据中断表，调用已经注册的中断函数，这个中断函数会调到驱动程序（NIC Driver）中相应的函数（调用对应的网卡驱动硬中断处理程序）

5、网卡驱动被调用后，**网卡驱动先禁用网卡的中断，表示驱动程序已经知道内存中有数据了，告诉网卡下次再收到数据包直接写内存就可以了，不要再通知CPU了，这样可以提高效率，避免CPU不停的被中断**；然后启动对应的软中断函数

6、启动软中断，这步结束后，硬件中断处理函数就结束返回了。由于硬中断处理程序执行的过程中不能被中断，所以如果它执行时间过长，会导致CPU没法响应其它硬件的中断，于是内核引入软中断，这样可以将硬中断处理函数中耗时的部分移到软中断处理函数里面来慢慢处理

（软中断函数开始从RingBuffer中进行循环取包，并且封装为`sk_buff`，然后投递给网络协议栈进行处理；协议栈处理完成后数据就进入用户态的对应进程，进程就可以操作数据了）

二、阶段2：内核的网络模块

软中断会触发内核网络模块中的软中断处理函数，继续上面的流程

```TEXT
                                                     +-----+
                                             17      |     |
                                        +----------->| NIC |
                                        |            |     |
                                        |Enable IRQ  +-----+
                                        |
                                        |
                                  +------------+                                      Memroy
                                  |            |        Read           +--------+--------+--------+--------+
                 +--------------->| NIC Driver |<--------------------- | Packet | Packet | Packet | ...... |
                 |                |            |          9            +--------+--------+--------+--------+
                 |                +------------+
                 |                      |    |        skb
            Poll | 8      Raise softIRQ | 6  +-----------------+
                 |                      |             10       |
                 |                      ↓                      ↓
         +---------------+  Call  +-----------+        +------------------+        +--------------------+  12  +---------------------+
         | net_rx_action |<-------| ksoftirqd |        | napi_gro_receive |------->| enqueue_to_backlog |----->| CPU input_pkt_queue |
         +---------------+   7    +-----------+        +------------------+   11   +--------------------+      +---------------------+
                                                               |                                                      | 13
                                                            14 |        + - - - - - - - - - - - - - - - - - - - - - - +
                                                               ↓        ↓
                                                    +--------------------------+    15      +------------------------+
                                                    | __netif_receive_skb_core |----------->| packet taps(AF_PACKET) |
                                                    +--------------------------+            +------------------------+
                                                               |
                                                               | 16
                                                               ↓
                                                      +-----------------+
                                                      | protocol layers |
                                                      +-----------------+

```


7、内核中的`ksoftirqd`进程专门负责软中断的处理，当它收到软中断后，就会调用相应软中断所对应的处理函数，对于上面第`6`步中是网卡驱动模块抛出的软中断，`ksoftirqd`会调用网络模块的`net_rx_action`函数（`ksoftirqd`线程开始调用驱动的`poll`函数收包）

8、`net_rx_action`会调用网卡驱动里的`poll`函数（对于igb网卡驱动来说，此[函数](https://elixir.bootlin.com/linux/v4.11.6/source/drivers/net/ethernet/intel/igb/igb_main.c#L6600)就是`igb_poll`）来一个个的处理数据包（`poll`函数将收到的包送到协议栈注册的`ip_rcv`函数中）

9、在`poll`函数中，驱动会一个接一个的读取网卡写到内存中的数据包，内存中数据包的格式只有驱动知道

10、网卡驱动程序将内存中的数据包转换成内核网络模块能识别的`skb`格式，然后调用`napi_gro_receive`函数

11、`napi_gro_receive`会处理`GRO`（Generic Receive Offloading）相关的内容，也就是将可以合并的数据包进行合并，这样就只需要调用一次协议栈。然后判断是否开启了`RPS`（Receive Packet Steering），如果开启了，将会调用`enqueue_to_backlog`

12、在`enqueue_to_backlog`函数中，会将数据包放入CPU的`softnet_data`结构体的`input_pkt_queue`中，然后返回，如果`input_pkt_queue`满了的话，该数据包将会被丢弃，queue的大小可以通过`net.core.netdev_max_backlog`来配置（备注：`enqueue_to_backlog`函数也会被`netif_rx`函数调用，而`netif_rx`正是`lo`设备发送数据包时调用的函数）

13、CPU会接着在自己的软中断上下文中处理自己`input_pkt_queue`里的网络数据（调用`__netif_receive_skb_core`）

14、如果没开启`RPS`，`napi_gro_receive`会直接调用`__netif_receive_skb_core`

15、看是不是有`AF_PACKET`类型的socket（raw socket），如果有的话，拷贝一份数据给它（`tcpdump`即抓取来自这里的packet）

16、调用协议栈相应的函数，将数据包交给协议栈处理

17、待内存中的所有数据包被处理完成后（即`poll`函数执行完成），**会再次启用网卡的硬中断，这样下次网卡再收到数据的时候就会通知CPU**


三、阶段3：协议栈网络层（IP）

接着看下数据包来到协议栈的处理过程，重要的内核函数如下：

```TEXT
          |
          |
          ↓         promiscuous mode &&
      +--------+    PACKET_OTHERHOST (set by driver)   +-----------------+
      | ip_rcv |-------------------------------------->| drop this packet|
      +--------+                                       +-----------------+
          |
          |
          ↓
+---------------------+
| NF_INET_PRE_ROUTING |
+---------------------+
          |
          |
          ↓
      +---------+
      |         | enabled ip forword  +------------+        +----------------+
      | routing |-------------------->| ip_forward |------->| NF_INET_FORWARD |
      |         |                     +------------+        +----------------+
      +---------+                                                   |
          |                                                         |
          | destination IP is local                                 ↓
          ↓                                                 +---------------+
 +------------------+                                       | dst_output_sk |
 | ip_local_deliver |                                       +---------------+
 +------------------+
          |
          |
          ↓
 +------------------+
 | NF_INET_LOCAL_IN |
 +------------------+
          |
          |
          ↓
    +-----------+
    | UDP layer |
    +-----------+
```

-   `ip_rcv`：此函数是IP模块的入口函数，该函数第一件事就是将垃圾数据包（目的mac地址不是当前网卡，但由于网卡设置了混杂模式而被接收进来）直接丢掉，然后调用注册在`NF_INET_PRE_ROUTING`上的函数
-   `NF_INET_PRE_ROUTING`： netfilter注册在协议栈中的钩子，可以通过`iptables`来注入一些数据包处理函数，用来修改或者丢弃数据包，如果数据包没被丢弃，将继续往下走
-   `routing`： 进行路由，如果是目的IP不是本地IP，且没有开启ip forward功能，那么数据包将被丢弃；如果开启了ip forward功能，那将进入`ip_forward`函数（转发模式常用）
-   `ip_forward`： `ip_forward`会先调用netfilter注册的`NF_INET_FORWARD`相关函数，如果数据包没有被丢弃，那么将继续往后调用`dst_output_sk`函数
-   `dst_output_sk`： 该函数会调用IP层的相应函数将该数据包发送出去（参考协议栈数据包发送流程的后半部分）
-   `ip_local_deliver`：如果上面`routing`的时候发现目的IP是本地IP，那么将会调用该函数。该函数会先调用`NF_INET_LOCAL_IN`相关的钩子程序，如果通过，数据包将会向下发送到UDP层

四、阶段4：协议栈传输层（UDP）

```TEXT
          |
          |
          ↓
      +---------+            +-----------------------+
      | udp_rcv |----------->| __udp4_lib_lookup_skb |
      +---------+            +-----------------------+
          |
          |
          ↓
 +--------------------+      +-----------+
 | sock_queue_rcv_skb |----->| sk_filter |
 +--------------------+      +-----------+
          |
          |
          ↓
 +------------------+
 | __skb_queue_tail |
 +------------------+
          |
          |
          ↓
  +---------------+
  | sk_data_ready |
  +---------------+

```

-   `udp_rcv`： 此[函数](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/udp.c#L2105)是UDP模块的入口函数，它里面会调用其它的函数，主要是做一些必要的检查，其中一个重要的调用是`__udp4_lib_lookup_skb`，该函数会根据目的IP和端口找对应的socket，如果没有找到相应的socket，那么该数据包将会被丢弃，否则继续（关联hook点`kprobe:udp_rcv`）
-   `sock_queue_rcv_skb`： 该函数会检查这个socket的receive buffer是不是满了，如果满了的话，丢弃该数据包，然后就是调用`sk_filter`看这个包是否是满足条件的包，如果当前socket上设置了filter，且该包不满足条件的话，这个数据包也将被丢弃（在Linux里面，每个socket上都可以像tcpdump里面一样定义filter，不满足条件的数据包将会被丢弃）
-   `__skb_queue_tail`： 将数据包放入socket接收队列的末尾
-   `sk_data_ready`： 通知socket数据包已经准备好；调用完`sk_data_ready`之后，一个数据包处理完成，等待应用层程序来读取，上面所有函数的执行过程都在软中断的上下文中

五、阶段5：socket应用程序

应用层一般有两种方式接收数据：
-   `recvfrom`函数阻塞等待数据到来，这种情况下当socket收到通知后，`recvfrom`就会被唤醒，然后读取接收队列的数据
-   通过`epoll`/`select`监听相应的socket，当收到通知后，再调用`recvfrom`函数去读取接收队列的数据

至此，一个UDP包就经由网卡成功送到了应用层程序

####    网卡收包的若干细节


##  0x02  核心数据结构
-   `struct socket`：传输层使用的[数据结构](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/net.h#L111)，用于声明、定义套接字
-   `struct sock`：网络层会调用`struct sock`[结构体](https://elixir.bootlin.com/linux/v4.11.6/source/include/net/sock.h#L311)，该结构体包含`struct sock_common`[结构体](https://elixir.bootlin.com/linux/v4.11.6/source/include/net/sock.h#L120)
-   `struct sk_buff`：内核中使用的套接字缓冲区[结构体](https://elixir.bootlin.com/linux/v4.11.6/source/include/linux/skbuff.h#L566)，套接字结构体用于表示一个网络连接对应的本地接口的网络信息，而`sk_buff`结构则是该网络连接对应的数据包的存储

####    struct sk_buff结构： 套接字缓冲区
对于ebpf应用开发者而言，最关注的结构莫过于`sk_buff`了。`sk_buff`用来管理和控制接收OR发送数据包的信息，各层协议都依赖于`sk_buff`而存在。内核中`sk_buff`结构体在各层协议之间传输不是用拷贝`sk_buff`结构体，而是通过增加协议头和移动指针来操作的。如果是从L4传输到L2，则是通过往`sk_buff`结构体中增加该层协议头来操作；如果是从L4到L2，则是通过移动`sk_buff`结构体中的data指针来实现，不会删除各层协议头


```C
struct sk_buff {
	union {
		struct {
			/* These two members must be first. 构成sk_buff链表*/
			struct sk_buff		*next;
			struct sk_buff		*prev;
			union {
				struct net_device	*dev;	//网络设备对应的结构体，很重要但是不是本文重点，所以不做展开
				/* Some protocols might use this space to store information,
				 * while device pointer would be NULL.
				 * UDP receive path is one user.
				 */
				unsigned long		dev_scratch;   // 对于某些不适用net_device的协议需要采用该字段存储信息，如UDP的接收路径
			};
		};
		struct rb_node		rbnode; /* used in netem, ip4 defrag, and tcp stack 将sk_buff以红黑树组织，在TCP中有用到*/
		struct list_head	list;   // sk_buff链表头指针（5.10内核后）
	};
	union {
		struct sock		*sk;       // 指向网络层套接字结构体
		int			ip_defrag_offset;   //用来处理IPv4报文分片
	};
	union {
		ktime_t		tstamp;    // 时间戳
		u64		skb_mstamp_ns; /* earliest departure time */
	};
	/* 存储私有信息
	 * This is the control buffer. It is free to use for every
	 * layer. Please put your private variables there. If you
	 * want to keep them across layers you have to do a skb_clone()
	 * first. This is owned by whoever has the skb queued ATM.
	 */
	char			cb[48] __aligned(8);
	union {
		struct {
			unsigned long	_skb_refdst;				   // 目标entry
			void		(*destructor)(struct sk_buff *skb);	// 析构函数
		};
		struct list_head	tcp_tsorted_anchor;			    // TCP发送队列(tp->tsorted_sent_queue)
	};
....
	unsigned int		len,	// 实际长度
				data_len;	    // 数据长度
	__u16			mac_len,    // mac层长度
				hdr_len;        // 可写头部长度
	/* Following fields are _not_ copied in __copy_skb_header()
	 * Note that queue_mapping is here mostly to fill a hole.
	 */
	__u16			queue_mapping;   // 多队列设备的队列映射
......
	/* fields enclosed in headers_start/headers_end are copied
	 * using a single memcpy() in __copy_skb_header()
	 */
	/* private: */
	__u32			headers_start[0];	
	/* public: */
......
	__u8			__pkt_type_offset[0];
	__u8			pkt_type:3;
	__u8			ignore_df:1;
	__u8			nf_trace:1;
	__u8			ip_summed:2;
	__u8			ooo_okay:1;
	__u8			l4_hash:1;
	__u8			sw_hash:1;
	__u8			wifi_acked_valid:1;
	__u8			wifi_acked:1;
	__u8			no_fcs:1;
	/* Indicates the inner headers are valid in the skbuff. */
	__u8			encapsulation:1;
	__u8			encap_hdr_csum:1;
	__u8			csum_valid:1;
......
	__u8			__pkt_vlan_present_offset[0];
	__u8			vlan_present:1;
	__u8			csum_complete_sw:1;
	__u8			csum_level:2;
	__u8			csum_not_inet:1;
	__u8			dst_pending_confirm:1;
......
	__u8			ipvs_property:1;
	__u8			inner_protocol_type:1;
	__u8			remcsum_offload:1;
......
	union {
		__wsum		csum;
		struct {
			__u16	csum_start;
			__u16	csum_offset;
		};
	};
	__u32			priority;
	int			skb_iif;		// 接收到该数据包的网络接口的编号
	__u32			hash;
	__be16			vlan_proto;
	__u16			vlan_tci;
......
	union {
		__u32		mark;
		__u32		reserved_tailroom;
	};
	union {
		__be16		inner_protocol;
		__u8		inner_ipproto;
	};
	__u16			inner_transport_header;
	__u16			inner_network_header;
	__u16			inner_mac_header;
	__be16			protocol;
	__u16			transport_header;	// 传输层头部
	__u16			network_header;		// 网络层头部
	__u16			mac_header;			// mac层头部
	/* private: */
	__u32			headers_end[0];
	/* public: */
	/* These elements must be at the end, see alloc_skb() for details.  */
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head, *data;
	unsigned int		truesize;
	refcount_t		users;
......
};
```

####    `struct sk_buff`成员&&管理
`sk_buff`的成员主要关注如下三类：

-   组织布局（Layout）
-   通用数据成员（General）
-   管理`sk_buff`结构体的函数（Management functions）

1、`sk_buff`的管理组织方式

通常`sk_buff`使用双链表`sk_buff_head`结构进行管理（并且`sk_buff`需要能在`O(1)`时间内获得双链表的头节点），布局如下图所示

```CPP
//有些情况下sk_buff不是用双链表而是用红黑树组织的，那么有效的域是rbnode
//5.10内核中，list域是一个list_head结构体，而非sk_buff_head
struct sk_buff_head {
	/* These two members must be first. */
	struct sk_buff	*next;  //sk_buff中的next、prev指向相邻的sk_buff
	struct sk_buff	*prev;

	__u32		qlen;   //链表中元素的数量
	spinlock_t	lock;   //并发访问时保护
};
```

![layout1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/sk_buff/sk_buff_layout1.png)

2、`sk_buff`的线性空间管理 && 创建

```CPP
/* These elements must be at the end, see alloc_skb() for details.  */
	sk_buff_data_t		tail;
	sk_buff_data_t		end;
	unsigned char		*head,*data;
```

从下图中可以发现，`head`与`end`指向的位置始终不变，数据的变化、协议头的添加都是依靠`tail`与`data`的移动来实现的；此外，初始化时，`head`、`data`、`tail`都指向开始位置

![layout2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/sk_buff/sk_buff_layout2.png)

`sk_buff`线性数据区的创建过程如下，`sk_buff`结构数据区初始化成功后，此时 `head` 指针、`data` 指针、`tail` 指针都是指向同一个地方（**`head` 指针和 `end` 指针指向的位置一直都不变，而对于数据的变化和协议信息的添加都是通过 `data` 指针和 `tail` 指针的改变来表现的**）

3、重要成员

-   `struct sock *sk`：指向拥有该`sk_buff`的套接字（即skb 关联的socket），当这个包是socket 发出或接收时，这里指向对应的socket，而如果是转发包，这里是`NULL`；在可观测场景下，当socket已经建立时，可以用来解析获取传输层的关键信息
-   `unsigned int truesize`：表示skb使用的大小，包括skb结构体以及它所指向的数据
-   `unsigned int		len`：所有数据的长度之和，包括分片的数据以及协议头
-   `unsigned int      data_len`：分片的数据长度
-   `__u16			mac_len`：链路层帧头长度
-   `__u16 hdr_len`：被copy的skb中可写的头部长度

4、General成员，即skb的通用成员，与协议类型或内核特性无关，这里也列举几个

`struct net_device	*dev`成员，用来表示从哪个设备收到报文，或将把报文发到哪个设备

```CPP
//include/linux/skbuff.h
			union {
				struct net_device	*dev;
				/* Some protocols might use this space to store information,
				 * while device pointer would be NULL.
				 * UDP receive path is one user.
				 */
				unsigned long		dev_scratch;
			};
```

`char cb[48]`成员，是skb能被各层共用的精髓，`48`即为TCP的控制块`tcp_sbk_cb`[数据结构](https://elixir.bootlin.com/linux/v4.11.6/source/include/net/tcp.h#L736)的size

```CPP
//include/linux/skbuff.h
	/*
	 * This is the control buffer. It is free to use for every
	 * layer. Please put your private variables there. If you
	 * want to keep them across layers you have to do a skb_clone()
	 * first. This is owned by whoever has the skb queued ATM.
	 */
	char			cb[48] __aligned(8);
```

`tcp_sbk_cb`结构如下，可以通过`TCP_SKB_CB`这个宏获取`cb`字段（被转换为`struct tcp_skb_cb *`类型）
```CPP
//include/net/tcp.h
#define TCP_SKB_CB(__skb)	((struct tcp_skb_cb *)&((__skb)->cb[0]))
//include/net/tcp.h
struct tcp_skb_cb {
	__u32		seq;		/* Starting sequence number	*/
	__u32		end_seq;	/* SEQ + FIN + SYN + datalen	*/

	/* ... */
	__u8		tcp_flags;	/* TCP header flags. (tcp[13])	*/

	/* ... */
};
```

`pkt_type`这个字段用于表示数据包类型，此类型是由目标MAC地址决定的

```CPP
//include/linux/skbuff.h
	__u8			pkt_type:3;


#define PACKET_HOST		0		/* To us		*/
#define PACKET_BROADCAST	1		/* To all		*/
#define PACKET_MULTICAST	2		/* To group		*/
#define PACKET_OTHERHOST	3		/* To someone else 	*/
#define PACKET_OUTGOING		4		/* Outgoing of any type */
#define PACKET_LOOPBACK		5		/* MC/BRD frame looped back */
#define PACKET_USER		6		/* To user space	*/
#define PACKET_KERNEL		7		/* To kernel space	*/
```

`__be16 protocol`这个字段标识了L2上层的协议类型，典型的协议类型如下：

```CPP
//include/uapi/linux/if_ether.h
/*
 *	These are the defined Ethernet Protocol ID's.
 */


#define ETH_P_IP	0x0800		/* Internet Protocol packet	*/
#define ETH_P_X25	0x0805		/* CCITT X.25			*/
#define ETH_P_ARP	0x0806		/* Address Resolution packet	*/

#define ETH_P_IPV6	0x86DD		/* IPv6 over bluebook		*/
```

`transport_header`、`network_header`、`mac_header`这三个字段标识了各层头部相对于`head`的偏移量，在ebpf对`sk_buff`结构的解析中也非常常用

```cpp
//include/linux/skbuff.h
	__u16			transport_header;
	__u16			network_header;
	__u16			mac_header;
```

5、管理函数（Management functions）

5.1、分配和释放内存，通过`alloc_skb`获取一个`struct sk_buff`加长度为`size`（经过对齐）的数据缓冲区，其中`skb->end`指向的是一个`skb_shared_info`结构体，`head`、`data`、`tail`以及`end`初始化指向如图所示，`end`留出了padding（tailroom）保证使读取为主的`skb_shared_info`结构与前面的数据不在一个缓存行

![alloc_skb](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/sk_buff/alloc_skb.png)

```CPP
//net/core/skbuff.c
static inline struct sk_buff *alloc_skb(unsigned int size, gfp_t priority){
    //....
	skb = kmem_cache_alloc_node(cache, gfp_mask & ~__GFP_DMA, node);
	/* ... */
	size = SKB_DATA_ALIGN(size);
	size += SKB_DATA_ALIGN(sizeof(struct skb_shared_info));
	data = kmalloc_reserve(size, gfp_mask, node, &pfmemalloc);
	/* kmalloc(size) might give us more room than requested.
	 * Put skb_shared_info exactly at the end of allocated zone,
	 * to allow max possible filling before reallocation.
	 */
	size = SKB_WITH_OVERHEAD(ksize(data));
	/* ... */
	skb->head = data;
	skb->data = data;
	skb_reset_tail_pointer(skb);
	skb->end = skb->tail + size;
    //...
}
```

5.2、`sk_buff`数据缓冲区指针操作

![sk_buff_oper](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/sk_buff/sk_buff_oper_function.png)

-   `skb_put`
-   `skb_push`
-   `skb_pull`：常用于协议栈接收报文时，从外到内剥离协议头（以太网头-IP 头-TCP 头）的操作
-   `skb_reserve`

5.3、接收数据包的处理过程，伪代码描述如下：

```CPP
// 驱动构造 skb（DMA 数据已写入）
struct sk_buff *skb = build_skb(dma_buffer, buffer_size);
skb_put(skb, packet_len);  // 设置数据长度

// 协议栈处理（以太网层）
__be16 proto = eth_type_trans(skb, dev);
skb_pull(skb, ETH_HLEN);    // 剥离以太网头

// 协议栈处理（IP 层）
struct iphdr *iph = ip_hdr(skb);
skb_pull(skb, iph->ihl * 4); // 剥离 IP 头

// 协议栈处理（TCP 层）
struct tcphdr *th = tcp_hdr(skb);
skb_pull(skb, th->doff * 4); // 剥离 TCP 头

// 应用层获取负载数据
char *payload = skb->data;
int payload_len = skb->len;
```

5.4、发送数据包的处理过程，下图展示了发送数据包时skb缓冲区被填满的过程，伪代码描述如下：

```CPP
unsigned int total_header_len = ETH_HLEN + IP4_HLEN + TCP_HLEN;
unsigned int payload_len = 1000;
// 分配skb，总空间为 headers + payload，初始 data 和 tail 指向缓冲区起始位置
struct sk_buff *skb = alloc_skb(total_header_len + payload_len, GFP_KERNEL);
// 预留所有协议头的空间，预留头部空间，data 和 tail 后移
skb_reserve(skb, total_header_len);
// 添加负载数据，填充负载数据，tail 后移，扩展数据区
skb_put(skb, payload_len);
memcpy(skb->data, payload, payload_len);
// 添加TCP头，添加协议头（从内到外），data 前移，覆盖预留空间
tcp_header = skb_push(skb, TCP_HLEN);
// 添加IP头
ip_header = skb_push(skb, IP4_HLEN);
// 添加以太网头
eth_header = skb_push(skb, ETH_HLEN);
// 发送
dev_queue_xmit(skb);
```

![sk_buff_send_packet](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/sk_buff/sk_buff_eg2_from_tcplayer_to_linklayer.png)


####    struct socket结构
每个`struct socket`结构都有一个`struct sock`结构成员，`sock`是对`socket`的扩充，`socket->sk`指向对应的`sock`结构，`sock->socket` 指向对应的`socket`结构

```CPP
struct socket {
	socket_state		state;  //链接状态
	short			type;       //套接字类型，如SOCK_STREAM等
	unsigned long		flags;
	struct socket_wq	*wq;
	struct file		*file;      //套接字对应的文件指针，毕竟Linux一切皆文件
	struct sock		*sk;        //网络层的套接字
	const struct proto_ops	*ops;
};
```

####    struct sock结构

```CPP
struct sock {
	struct sock_common	__sk_common;	   // 网络层套接字通用结构体
......
	socket_lock_t		sk_lock;	       // 套接字同步锁
	atomic_t		sk_drops;	           // IP/UDP包丢包统计
	int			sk_rcvlowat;        	   // SO_RCVLOWAT标记位
......
	struct sk_buff_head	sk_receive_queue;	// 收到的数据包队列
......
	int			sk_rcvbuf;				  // 接收缓存大小
......
	union {
		struct socket_wq __rcu	*sk_wq;	    // 等待队列
		struct socket_wq	*sk_wq_raw;
	};
......
	int			sk_sndbuf;			       // 发送缓存大小
	/* ===== cache line for TX ===== */
	int			sk_wmem_queued;			   // 传输队列大小
	refcount_t		sk_wmem_alloc;		    // 已确认的传输字节数
	unsigned long		sk_tsq_flags;	    // TCP Small Queue标记位
	union {
		struct sk_buff	*sk_send_head;		// 发送队列对首
		struct rb_root	tcp_rtx_queue;		 
	};
	struct sk_buff_head	sk_write_queue;		 // 发送队列
......
	u32			sk_pacing_status; /* see enum sk_pacing 发包速率控制状态*/ 
	long			sk_sndtimeo;		    // SO_SNDTIMEO 标记位
	struct timer_list	sk_timer;			// 套接字清空计时器
	__u32			sk_priority;		    // SO_PRIORITY 标记位
......
	unsigned long		sk_pacing_rate; /* bytes per second 发包速率*/
	unsigned long		sk_max_pacing_rate;  // 最大发包速率
	struct page_frag	sk_frag;		    // 缓存页帧
......
	struct proto		*sk_prot_creator;
	rwlock_t		sk_callback_lock;
	int			sk_err,					  // 上次错误
				sk_err_soft;			  // “软”错误：不会导致失败的错误
	u32			sk_ack_backlog;			   // ack队列长度
	u32			sk_max_ack_backlog;		   // 最大ack队列长度
	kuid_t			sk_uid;				  // user id
	struct pid		*sk_peer_pid;		   // 套接字对应的peer的id
......
	long			sk_rcvtimeo;		  // 接收超时
	ktime_t			sk_stamp;			  // 时间戳
......
	struct socket		*sk_socket;		   // Identd协议报告IO信号
	void			*sk_user_data;		  // RPC层私有信息
......
	struct sock_cgroup_data	sk_cgrp_data;   // cgroup数据
	struct mem_cgroup	*sk_memcg;		   // 内存cgroup关联
	void			(*sk_state_change)(struct sock *sk);	// 状态变化回调函数
	void			(*sk_data_ready)(struct sock *sk);		// 数据处理回调函数
	void			(*sk_write_space)(struct sock *sk);		// 写空间可用回调函数
	void			(*sk_error_report)(struct sock *sk);    // 错误报告回调函数
	int			(*sk_backlog_rcv)(struct sock *sk, struct sk_buff *skb);	// 处理存储区回调函数
......
	void                    (*sk_destruct)(struct sock *sk);	// 析构回调函数
	struct sock_reuseport __rcu	*sk_reuseport_cb;			   // group容器重用回调函数
......
};
```

####    struct sock_common结构
`struct sock_common`是套接口在网络层的最小表示，即最基本的网络层套接字信息
```CPP
struct sock_common {
	/* skc_daddr and skc_rcv_saddr must be grouped on a 8 bytes aligned
	 * address on 64bit arches : cf INET_MATCH()
	 */
	union {
		__addrpair	skc_addrpair;
		struct {
			__be32	skc_daddr;		// 外部/目的IPV4地址
			__be32	skc_rcv_saddr;	// 本地绑定IPV4地址
		};
	};
	union  {
		unsigned int	skc_hash;	// 根据协议查找表获取的哈希值
		__u16		skc_u16hashes[2]; // 2个16位哈希值，UDP专用
	};
	/* skc_dport && skc_num must be grouped as well */
	union {
		__portpair	skc_portpair;	// 
		struct {
			__be16	skc_dport;	    // inet_dport占位符
			__u16	skc_num;	    // inet_num占位符
		};
	};
	unsigned short		skc_family;	      // 网络地址family
	volatile unsigned char	skc_state;    // 连接状态
	unsigned char		skc_reuse:4;      // SO_REUSEADDR 标记位
	unsigned char		skc_reuseport:1;  // SO_REUSEPORT 标记位
	unsigned char		skc_ipv6only:1;   // IPV6标记位
	unsigned char		skc_net_refcnt:1; // 该套接字网络名字空间内引用数
	int			skc_bound_dev_if;		 // 绑定设备索引
	union {
		struct hlist_node	skc_bind_node;     // 不同协议查找表组成的绑定哈希表
		struct hlist_node	skc_portaddr_node; // UDP/UDP-Lite protocol二级哈希表
	};
	struct proto		*skc_prot;			  // 协议回调函数，根据协议不同而不同
......
	union {									
		struct hlist_node	skc_node;		    // 不同协议查找表组成的主哈希表
		struct hlist_nulls_node skc_nulls_node;  // UDP/UDP-Lite protocol主哈希表
	};
	unsigned short		skc_tx_queue_mapping;    // 该连接的传输队列
	unsigned short		skc_rx_queue_mapping;    // 该连接的接受队列
......
	union {
		int		skc_incoming_cpu; // 多核下处理该套接字数据包的CPU编号
		u32		skc_rcv_wnd;	  // 接收窗口大小
		u32		skc_tw_rcv_nxt; /* struct tcp_timewait_sock  */
	};
	refcount_t		skc_refcnt;   // 套接字引用计数
......
};
```

####    小结
网络通信中通过网卡获取到的数据包至少包括了物理层，链路层和网络层的内容，因此套接字结构体仅仅从网络层开始，即通常只定义了传输层的套接字`socket`和网络层的套接字`sock`。`socket` 是用于负责对上给用户提供接口，并且和文件系统关联。而 `sock`负责向下对接内核网络协议栈

从传输层到链路层，它是存放数据的通用结构，为了保持高效率，数据在传递过程中尽量不发生额外的拷贝。因此，从高层到低层的时候，会不断地在数据前加头，因此每一层的协议都会调用`skb_reserve`，为自己的报头预留空间。至于从低层到高层，去掉低层报头的方式就是移动一下指针，指向高层头，非常简洁

##	0x03	可观测：内核收包的主要过程

####	准备工作

Linux驱动，内核协议栈等等模块在具备接收网卡数据包之前，需要完整如下的初始化工作，这部分内容可以参考[图解Linux网络包接收过程](https://mp.weixin.qq.com/s/GoYDsfy9m0wRoXi_NCfCmg)：

1、Linux系统启动，创建ksoftirqd内核线程，用来处理软中断

创建ksoftirqd内核线程关联结构[`softirq_threads`](https://elixir.bootlin.com/linux/v4.11.6/source/kernel/softirq.c#L748)，当ksoftirqd被创建出来以后，它就会进入自己的线程循环函数`ksoftirqd_should_run`和`run_ksoftirqd`，不停地判断有没有软中断需要被处理

```CPP
//file: kernel/softirq.c
static struct smp_hotplug_thread softirq_threads = {
    .store          = &ksoftirqd,
    .thread_should_run  = ksoftirqd_should_run,
    .thread_fn      = run_ksoftirqd,
    .thread_comm        = "ksoftirqd/%u",
};

static __init int spawn_ksoftirqd(void)
{
    register_cpu_notifier(&cpu_nfb);

    BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

    return 0;
}
early_initcall(spawn_ksoftirqd);
```

2、网络子系统初始化

3、协议栈注册，针对协议栈支持的各类协议如arp/icmp/ip/udp/tcp等，每一个协议都会将自己的处理函数注册

4、网卡驱动初始化，初始化DMA以及向内核注册收包函数地址（NAPI的`poll`函数）

5、启动网卡，分配RX/TX队列，注册中断对应的处理函数

```CPP
//file: drivers/net/ethernet/intel/igb/igb_main.c
static int __igb_open(struct net_device *netdev, bool resuming)
{
    /* allocate transmit descriptors */
    err = igb_setup_all_tx_resources(adapter);

    /* allocate receive descriptors */
    err = igb_setup_all_rx_resources(adapter);

    /* 注册中断处理函数 */
    err = igb_request_irq(adapter);
    if (err)
        goto err_req_irq;

    /* 启用NAPI */
    for (i = 0; i < adapter->num_q_vectors; i++)
        napi_enable(&(adapter->q_vector[i]->napi));

    //......
}
```

6、当上面工作都完成之后，就可以打开硬中断，等待数据包的到来

其中协议栈注册主要完成了各层协议处理函数的注册，如内核实现网络层的ip协议，传输层的tcp/udp协议等，这些协议对应的实现函数分别是`ip_rcv()`/`tcp_v4_rcv()`/`udp_rcv()`，内核调用`inet_init`后开始网络协议栈注册。通过`inet_init`将上述函数注册到了`inet_protos`和`ptype_base`数据结构中

![inet_init]()

```CPP
//file: net/ipv4/af_inet.c
//udp_protocol结构体中的handler是udp_rcv，tcp_protocol结构体中的handler是tcp_v4_rcv
//通过inet_add_protocol被初始化到数据结构中
static struct packet_type ip_packet_type __read_mostly = {

    .type = cpu_to_be16(ETH_P_IP),
    .func = ip_rcv,};static const struct net_protocol udp_protocol = {
    .handler =  udp_rcv,
    .err_handler =  udp_err,
    .no_policy =    1,
    .netns_ok = 1,};static const struct net_protocol tcp_protocol = {
    .early_demux    =   tcp_v4_early_demux,
    .handler    =   tcp_v4_rcv,
    .err_handler    =   tcp_v4_err,
    .no_policy  =   1,
    .netns_ok   =   1,

};

static int __init inet_init(void){
    ......

	//inet_add_protocol：注册icmp/tcp/udp等协议钩子
    if (inet_add_protocol(&icmp_protocol, IPPROTO_ICMP) < 0)
        pr_crit("%s: Cannot add ICMP protocol\n", __func__);
    if (inet_add_protocol(&udp_protocol, IPPROTO_UDP) < 0)
        pr_crit("%s: Cannot add UDP protocol\n", __func__);
    if (inet_add_protocol(&tcp_protocol, IPPROTO_TCP) < 0)
        pr_crit("%s: Cannot add TCP protocol\n", __func__);
    ......

	//注册ip_packet_type
	//dev_add_pack(&ip_packet_type)
	//ip_packet_type结构体中的type是协议名，func是ip_rcv函数
	//在dev_add_pack中会被注册到ptype_base哈希表中
    dev_add_pack(&ip_packet_type);
}

/*
inet_add_protocol函数将tcp和udp对应的处理函数都注册到了inet_protos数组
*/
int inet_add_protocol(const struct net_protocol *prot, unsigned char protocol){
    if (!prot->netns_ok) {
        pr_err("Protocol %u is not namespace aware, cannot register.\n",
            protocol);
        return -EINVAL;
    }

    return !cmpxchg((const struct net_protocol **)&inet_protos[protocol],
            NULL, prot) ? 0 : -1;
}

//file: net/core/dev.c
void dev_add_pack(struct packet_type *pt){
    struct list_head *head = ptype_head(pt);
    ......
}

static inline struct list_head *ptype_head(const struct packet_type *pt){
    if (pt->type == htons(ETH_P_ALL))
        return &ptype_all;
    else
        return &ptype_base[ntohs(pt->type) & PTYPE_HASH_MASK];
}
```

小结下，上述逻辑中`inet_protos`记录了udp，tcp处理函数的地址，`ptype_base`存储了`ip_rcv()`函数的处理地址，软中断中会通过`ptype_base`找到`ip_rcv`函数地址，进而将ip包正确地送到`ip_rcv()`函数中执行，进而在`ip_rcv`中将会通过`inet_protos`结构定位到tcp或者udp的处理函数，再而把包转发给`udp_rcv()`或`tcp_v4_rcv()`函数。另外，在`ip_rcv`、`tcp_v4_rcv`、`udp_rcv`等函数中可以了解更详细的处理细节，比如`ip_rcv`中会处理netfilter和iptables过滤规则， netfilter 或 iptables 规则，这些规则都是在软中断的上下文中执行的，会加大网络延迟（规则复杂且数目较多）

####	接收数据的主要流程（核心）
1、硬中断处理

首先数据帧从网线到达网卡的接收队列上，网卡在分配给自己的RingBuffer中寻找可用的内存位置，找到后DMA引擎会把数据DMA到这块Ringbuffer中（该过程对CPU无感）。当DMA操作完成以后，网卡会向CPU发起一个硬中断，通知CPU有数据帧到达。igb网卡的硬中断注册的处理函数是[`igb_msix_ring`](https://elixir.bootlin.com/linux/v4.11.6/source/drivers/net/ethernet/intel/igb/igb_main.c#L5782)，上文也提到过，硬中断里只完成简单必要的工作，剩下的大部分逻辑都是转交给软中断处理。`igb_msix_ring`的逻辑仅仅记录了一个寄存器，修改了一下下CPU的`poll_list`，然后发出软中断即完成

![hardirq]()

-   `igb_write_itr`：记录一下硬件中断频率
-   `napi_schedule->__napi_schedule->____napi_schedule->__raise_softirq_irqoff`：触发软中断

```CPP
//file: drivers/net/ethernet/intel/igb/igb_main.c
static irqreturn_t igb_msix_ring(int irq, void *data)
{
    struct igb_q_vector *q_vector = data;

    /* Write the ITR value calculated from the previous interrupt. */
    igb_write_itr(q_vector);

    napi_schedule(&q_vector->napi);

    return IRQ_HANDLED;
}

/* Called with irq disabled */
static inline void ____napi_schedule(struct softnet_data *sd,
                     struct napi_struct *napi)
{
    //list_add_tail修改了CPU变量softnet_data里的poll_list，将驱动napi_struct传过来的poll_list添加了进来
    //其中softnet_data中的poll_list是一个双向列表，其中的设备都带有输入帧等着被处理
    list_add_tail(&napi->poll_list, &sd->poll_list);

    //触发了一个软中断NET_RX_SOFTIRQ
    //触发过程只是对一个变量进行了一次或运算而已
    __raise_softirq_irqoff(NET_RX_SOFTIRQ);
}

void __raise_softirq_irqoff(unsigned int nr)
{
    trace_softirq_raise(nr);
    or_softirq_pending(1UL << nr);
}
//file: include/linux/irq_cpustat.h

//local_softirq_pending() 修改
#define or_softirq_pending(x)  (local_softirq_pending() |= (x))
```

2、 `ksoftirqd`内核线程处理软中断

ksoftirqd中两个线程函数`ksoftirqd_should_run`和`run_ksoftirqd`的主要逻辑如下：

![]()

-   `ksoftirqd_should_run`：读取`local_softirq_pending`函数的结果（硬中断也调用此函数，硬中断位置会修改写入标记），如果硬中断中设置了`NET_RX_SOFTIRQ`，接下来会真正进入线程函数中`run_ksoftirqd`处理
-   `run_ksoftirqd`：

```CPP
static int ksoftirqd_should_run(unsigned int cpu)
{
    return local_softirq_pending();
}

#define local_softirq_pending() \
    __IRQ_STAT(smp_processor_id(), __softirq_pending)

static void run_ksoftirqd(unsigned int cpu)
{
    //屏蔽当前CPU上的所有中断
    local_irq_disable();
    if (local_softirq_pending()) {
        __do_softirq();
        rcu_note_context_switch(cpu);
        local_irq_enable();
        cond_resched();
        return;
    }

    //用于将CPSR寄存器中的中断使能位设为1，从而使得CPU能够响应中断
    local_irq_enable();
}
```

这里对`run_ksoftirqd`的逻辑进行下说明：

-   `__do_softirq`：判断根据当前CPU的软中断类型，调用其注册的`action`方法（前文描述过为`NET_RX_SOFTIRQ`注册的处理函数[`net_rx_action`](https://elixir.bootlin.com/linux/v4.11.6/source/net/core/dev.c#L5313)）

```CPP
asmlinkage void __do_softirq(void)
{
    do {
        if (pending & 1) {
            unsigned int vec_nr = h - softirq_vec;
            int prev_count = preempt_count();

            //...
            trace_softirq_entry(vec_nr);
            //CALL net_rx_action
            h->action(h);
            trace_softirq_exit(vec_nr);
            //...
        }
        h++;
        pending >>= 1;
    } while (pending);
}
```

`h->action(h)`即调用`net_rx_action`函数，它的工作过程如下：

1.  函数开头的`time_limit`和`budget`是用来控制`net_rx_action`函数主动退出的，目的是保证网络包的接收不霸占CPU不放，等下次网卡再有硬中断过来的时候再处理剩下的接收数据包
2.  `net_rx_action`最核心逻辑是获取到当前CPU变量`softnet_data`，对其`poll_list`进行遍历, 然后执行到网卡驱动注册到的`poll`函数（对于igb网卡来说即igb驱动的`igb_poll`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/drivers/net/ethernet/intel/igb/igb_main.c#L6600)）
3.	`igb_poll`的初始化流程在[`igb_alloc_q_vector`](https://elixir.bootlin.com/linux/v4.11.6/source/drivers/net/ethernet/intel/igb/igb_main.c#L1225)函数中：`netif_napi_add(adapter->netdev, &q_vector->napi,igb_poll, 64)`

```cpp
static void net_rx_action(struct softirq_action *h)
{
    struct softnet_data *sd = &__get_cpu_var(softnet_data);
    unsigned long time_limit = jiffies + 2;
    int budget = netdev_budget;
    void *have;
    //WHY?
    local_irq_disable();

	//https://elixir.bootlin.com/linux/v4.11.6/source/net/core/dev.c#L5269
	//循环收包，直到poll_list为空 or 主动退出
    while (!list_empty(&sd->poll_list)) {
        //......
        n = list_first_entry(&sd->poll_list, struct napi_struct, poll_list);

        work = 0;
        if (test_bit(NAPI_STATE_SCHED, &n->state)) {
            //igb_poll for igb driver
			// mlx5 for mlx5 driver
            work = n->poll(n, weight);
            trace_napi_poll(n);
        }

        budget -= work;
    }
}
```

继续走读下收包核心函数`igb_poll`，其重点工作是对[`igb_clean_rx_irq`](https://elixir.bootlin.com/linux/v4.11.8/source/drivers/net/ethernet/intel/igb/igb_main.c#L7157)的调用，`igb_clean_rx_irq`的主要工作：

-	`igb_fetch_rx_buffer->napi_alloc_skb->__napi_alloc_skb`：
-	`igb_is_non_eop`：`igb_fetch_rx_buffer`和`igb_is_non_eop`的作用就是把数据帧从RingBuffer上取下来，特别注意这里是通过**循环获取的**，因为有可能帧要占多个RingBuffer（当网卡通过 DMA 将数据包写入内存时，可能因硬件特性或数据包长度超过缓冲区容量，将一个完整数据帧分散到多个接收描述符对应的缓冲区中），所以是在一个循环中获取的，直到帧尾部。`igb_is_non_eop`函数的主要功能是用于判断当前接收的数据缓冲区（Rx buffer）是否属于一个完整数据帧的结尾部分。**获取下来的一个数据帧用一个`sk_buff`来表示**，这个结构会贯穿协议栈处理直到应用层
-	`napi_alloc_skb`：调用[`__napi_alloc_skb`](https://elixir.bootlin.com/linux/v4.11.8/source/net/core/skbuff.c#L471)，该方法最终最调用`__build_skb`进行创建`sk_buff`的操作；这里有个细节稍微提一下，generic XDP的处理逻辑就是在`__build_skb`之前，不过此内核igb驱动不支持，可以参考mlx5驱动的实现[`skb_from_cqe`](https://elixir.bootlin.com/linux/v4.11.8/source/drivers/net/ethernet/mellanox/mlx5/core/en_rx.c#L927)
-	`igb_process_skb_fields`：收取完数据以后，对其进行一些校验，然后开始设置`skb`变量的timestamp, VLAN id, protocol等字段
-	`napi_gro_receive`：`napi_skb_finish->netif_receive_skb`

```cpp
/**
 *  igb_poll - NAPI Rx polling callback
 *  @napi: napi polling structure
 *  @budget: count of how many packets we should handle
 **/
static int igb_poll(struct napi_struct *napi, int budget)
{
    //...

	//igb_poll的核心逻辑是对igb_clean_rx_irq的调用
    if (q_vector->tx.ring)
        clean_complete = igb_clean_tx_irq(q_vector);

    if (q_vector->rx.ring)
        clean_complete &= igb_clean_rx_irq(q_vector, budget);
    //...
}

static bool igb_clean_rx_irq(struct igb_q_vector *q_vector, const int budget)
{
    //...
    do {
        /* retrieve a buffer from the ring */
        skb = igb_fetch_rx_buffer(rx_ring, rx_desc, skb);

        /* fetch next buffer in frame if non-eop */
        if (igb_is_non_eop(rx_ring, rx_desc))
            continue;
        }

        /* verify the packet layout is correct */
        if (igb_cleanup_headers(rx_ring, rx_desc, skb)) {
            skb = NULL;
            continue;
        }

        /* populate checksum, timestamp, VLAN, and protocol */
        igb_process_skb_fields(rx_ring, rx_desc, skb);

        napi_gro_receive(&q_vector->napi, skb);
}

static struct sk_buff *igb_fetch_rx_buffer(struct igb_ring *rx_ring,
					   union e1000_adv_rx_desc *rx_desc,
					   struct sk_buff *skb)
{
	unsigned int size = le16_to_cpu(rx_desc->wb.upper.length);
	struct igb_rx_buffer *rx_buffer;
	struct page *page;

	//...

	if (likely(!skb)) {

		/* allocate a skb to store the frags */
		skb = napi_alloc_skb(&rx_ring->q_vector->napi, IGB_RX_HDR_LEN);
		if (unlikely(!skb)) {
			rx_ring->rx_stats.alloc_failed++;
			return NULL;
		}

		/* we will be copying header into skb->data in
		 * pskb_may_pull so it is in our interest to prefetch
		 * it now to avoid a possible cache miss
		 */
		prefetchw(skb->data);
	}
	//...
}
```

这里额外贴下`mlx5`驱动的`skb_from_cqe`[实现](https://elixir.bootlin.com/linux/v4.11.8/source/drivers/net/ethernet/mellanox/mlx5/core/en_rx.c#L762)，可以看到XDP在内核的实现位置：

```cpp
static inline
struct sk_buff *skb_from_cqe(struct mlx5e_rq *rq, struct mlx5_cqe64 *cqe,
			     u16 wqe_counter, u32 cqe_bcnt)
{
	//...
	rcu_read_lock();

	// 优先进行xdp的处理
	consumed = mlx5e_xdp_handle(rq, di, va, &rx_headroom, &cqe_bcnt);
	rcu_read_unlock();
	// 如果xdp以及处理了，就不走后面的流程
	if (consumed)
		return NULL; /* page/packet was consumed by XDP */

	// 走正常协议栈流程
	skb = build_skb(va, RQ_PAGE_SIZE(rq));
	if (unlikely(!skb)) {
		rq->stats.buff_alloc_err++;
		mlx5e_page_release(rq, di, true);
		return NULL;
	}

	/* queue up for recycling ..*/
	page_ref_inc(di->page);
	mlx5e_page_release(rq, di, true);

	skb_reserve(skb, rx_headroom);
	skb_put(skb, cqe_bcnt);

	return skb;
}
```

`igb_clean_rx_irq`函数的最后流程是调用`napi_gro_receive->napi_skb_finish`函数，继续跟踪就看到了熟悉的`netif_receive_skb`函数，在`netif_receive_skb`中，数据包将被送到协议栈中继续处理

```cpp
//file: net/core/dev.c
gro_result_t napi_gro_receive(struct napi_struct *napi, struct sk_buff *skb)
{
    skb_gro_reset_offset(skb);
	/*
	dev_gro_receive这个函数代表的是网卡GRO特性
	可以简单理解成把相关的小包合并成一个大包就行，目的是减少传送给网络栈的包数，这有助于减少 CPU 的使用量
	*/
    return napi_skb_finish(dev_gro_receive(napi, skb), skb);
}

//file: net/core/dev.c
static gro_result_t napi_skb_finish(gro_result_t ret, struct sk_buff *skb)
{
    switch (ret) {
    case GRO_NORMAL:
		// 熟悉的函数
        if (netif_receive_skb(skb))
            ret = GRO_DROP;
        break;
    //......
}
```

3、网络协议栈处理

`netif_receive_skb->__netif_receive_skb_core`函数会根据packet的协议调用注册的协议处理函数处理，在`__netif_receive_skb_core`函数，`__netif_receive_skb_core`取出protocol，它会从数据包中取出协议信息，然后遍历注册在这个协议上的回调函数列表

-	tcpdump的抓包点逻辑
-	`list_for_each_entry_rcu`用于遍历由RCU保护的链表，目的是将网络数据包（sk_buff）分发给注册的协议处理程序（如抓包工具tcpdump或协议栈）,在`__netif_receive_skb_core`中，list_for_each_entry_rcu被用于两个场景（代码片段如下）
	-	遍历`ptype_all`链表，将数据包发送给所有注册的全局协议处理程序，由于可能有多个处理程序注册到`ptype_all`（例如多个抓包实例），所以需循环逐个检查设备是否匹配（`ptype->dev == skb->dev`）并分发数据包
	-	遍历`ptype_base`哈希链表：根据数据包的类型（如IPv4、ARP）分发到对应的协议栈处理程序，由于同一协议类型可能有多个处理程序（如内核协议栈和用户态工具），需遍历所有可能的匹配项

![]()

```cpp
static int __netif_receive_skb_core(struct sk_buff *skb, bool pfmemalloc)
{
    //......

    //pcap逻辑，这里会将数据送入抓包点。tcpdump就是从这个入口获取包的
    list_for_each_entry_rcu(ptype, &ptype_all, list) {
        if (!ptype->dev || ptype->dev == skb->dev) {
			 // 数据包传递给匹配的协议处理程序
            if (pt_prev)
                ret = deliver_skb(skb, pt_prev, orig_dev);
            pt_prev = ptype;
        }
    }

    //......

	//ptype_base 是一个 hashtable
	//ip_rcv 函数地址就存储在这个 hashtable
    list_for_each_entry_rcu(ptype,
            &ptype_base[ntohs(type) & PTYPE_HASH_MASK], list) {
        if (ptype->type == type &&
            (ptype->dev == null_or_dev || ptype->dev == skb->dev ||
             ptype->dev == orig_dev)) {
			// 数据包传递给特定协议类型的处理程序
            if (pt_prev)
                ret = deliver_skb(skb, pt_prev, orig_dev);	//for ip_rcv
            pt_prev = ptype;
        }
    }
}

//file: net/core/dev.c
static inline int deliver_skb(struct sk_buff *skb,
                  struct packet_type *pt_prev,
                  struct net_device *orig_dev)
{
    //......

	//这里调用到了协议层注册的处理函数
	//对于ip包，就会进入到ip_rcv
	//对于arp包，会进入到arp_rcv
    return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
}
```

4、IP协议层处理流程

```cpp
//file: net/ipv4/ip_input.c
int ip_rcv(struct sk_buff *skb, struct net_device *dev, struct packet_type *pt, struct net_device *orig_dev)
{
    //......

	//NF_HOOK是一个钩子函数，当执行完注册的钩子后就会执行到最后一个参数指向的函数ip_rcv_finish
    return NF_HOOK(NFPROTO_IPV4, NF_INET_PRE_ROUTING, skb, dev, NULL,
               ip_rcv_finish);
}

static int ip_rcv_finish(struct sk_buff *skb)
{
    //......

    if (!skb_dst(skb)) {
		//ip_route_input_noref中调用了ip_route_input_mc
        int err = ip_route_input_noref(skb, iph->daddr, iph->saddr,
                           iph->tos, skb->dev);
        //...
    }

	//......
    return dst_input(skb);
}

//file: net/ipv4/route.c
static int ip_route_input_mc(struct sk_buff *skb, __be32 daddr, __be32 saddr,
                u8 tos, struct net_device *dev, int our)
{
    if (our) {
        rth->dst.input= ip_local_deliver;	//函数`ip_local_deliver`被赋值给了dst.input
        rth->rt_flags |= RTCF_LOCAL;
    }
}
```

上面`ip_rcv_finish`中的`return dst_input(skb)`就是调用了`ip_local_deliver`，如下：

```cpp
/* Input packet from network to transport.  */
static inline int dst_input(struct sk_buff *skb)
{
	//skb_dst(skb)->input调用的input方法就是路由子系统赋的ip_local_deliver
    return skb_dst(skb)->input(skb);
}
```

在`ip_local_deliver`函数最后的逻辑`ip_local_deliver_finish`中，会看到根据协议`ip_hdr(skb)->protocol`类型来选择对应的handler进行调用`ipprot->handler(skb)`，skb buffer将会进一步被派送到更上层的协议中，比如udp/tcp

```cpp
//file: net/ipv4/ip_input.c
int ip_local_deliver(struct sk_buff *skb)
{
    /*
     *  Reassemble IP fragments.
     */	
	//如果ip分片需要重组
    if (ip_is_fragment(ip_hdr(skb))) {
        if (ip_defrag(skb, IP_DEFRAG_LOCAL_DELIVER))
            return 0;
    }

    return NF_HOOK(NFPROTO_IPV4, NF_INET_LOCAL_IN, skb, skb->dev, NULL,
               ip_local_deliver_finish);
}

static int ip_local_deliver_finish(struct sk_buff *skb)
{
    //......

    int protocol = ip_hdr(skb)->protocol;
    const struct net_protocol *ipprot;

	//inet_protos中保存着tcp_rcv()和udp_rcv()的函数地址
    ipprot = rcu_dereference(inet_protos[protocol]);
    if (ipprot != NULL) {
        ret = ipprot->handler(skb);
    }
}
```

5、UDP协议层处理过程

UDP协议的处理[函数](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/udp.c#L1873)是`udp_rcv->__udp4_lib_rcv`，关键流程如下：

-	`__udp4_lib_lookup_skb`：根据`skb`来寻找对应的socket结构，即`struct sock *sk`
-	`udp_queue_rcv_skb`：将`skb`成功挂到`sk`对应的接收队列`sk_receive_queue`的尾部，等待上层接收

```cpp
//file: net/ipv4/udp.c
int udp_rcv(struct sk_buff *skb)
{
    return __udp4_lib_rcv(skb, &udp_table, IPPROTO_UDP);
}

/*
 *	All we need to do is get the socket, and then do a checksum.
 */
int __udp4_lib_rcv(struct sk_buff *skb, struct udp_table *udptable,
           int proto)
{
	// ......
	//__udp4_lib_lookup_skb是根据skb来寻找对应的socket，当找到以后将数据包放到socket的缓存队列里
	//如果没有找到，则发送一个目标不可达的icmp包
    sk = __udp4_lib_lookup_skb(skb, uh->source, uh->dest, udptable);

    if (sk != NULL) {
        int ret = udp_queue_rcv_skb(sk, skb);
		//......
    }

    icmp_send(skb, ICMP_DEST_UNREACH, ICMP_PORT_UNREACH, 0);
}
```

`udp_queue_rcv_skb`函数的主要功能是

```cpp
//file: net/ipv4/udp.c
int udp_queue_rcv_skb(struct sock *sk, struct sk_buff *skb)
{   
    //......

	//sk_rcvqueues_full：接收队列如果满了的话，将直接把包丢弃
	//接收队列大小受内核参数net.core.rmem_max和net.core.rmem_default影响
    if (sk_rcvqueues_full(sk, skb, sk->sk_rcvbuf))
        goto drop;

        rc = 0;

    ipv4_pktinfo_prepare(skb);
    bh_lock_sock(sk);
	/*
	sock_owned_by_user：这里判断用户是不是正在这个socket上进行系统调用（socket被占用），如果没有，那就可以直接放到socket的接收队列中
	如果有，那就通过sk_add_backlog把数据包添加到backlog队列
	当用户释放的socket的时候，内核会检查backlog队列，如果有数据再移动到接收队列中
	*/
    if (!sock_owned_by_user(sk))
        rc = __udp_queue_rcv_skb(sk, skb);	//核心处理
    else if (sk_add_backlog(sk, skb, sk->sk_rcvbuf)) {
        bh_unlock_sock(sk);
        goto drop;
    }
    bh_unlock_sock(sk);

    return rc;
}
```

`sock_owned_by_user`函数的场景是TODO

继续看下`__udp_queue_rcv_skb->__udp_enqueue_schedule_skb`的实现：

```cpp
int __udp_enqueue_schedule_skb(struct sock *sk, struct sk_buff *skb)
{
	struct sk_buff_head *list = &sk->sk_receive_queue;
	int rmem, delta, amt, err = -ENOMEM;
	spinlock_t *busy = NULL;
	int size;
	//......

	//把skb挂到list的尾部
	__skb_queue_tail(list, skb);

	//......	
}
EXPORT_SYMBOL_GPL(__udp_enqueue_schedule_skb);
```


##	0x04	应用层处理
上一章节描述了整个Linux内核对数据包的接收和处理过程，最后把数据包放到socket的接收队列中，那么应用层如何接受数据呢？以UDP应用常用的`recvfrom`函数（glibc库函数）为例进行分析

####	系统调用：recvfrom
对于系统库函数`recvfrom`，该函数在执行后会将用户进行陷入到内核态，进入到Linux实现的系统调用`sys_recvfrom`，回想下前文介绍的socket相关结构

![recvfrom_socket](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/recvfrom_socket.png)

如上图，`struct socket`结构中的`const struct proto_ops`成员对应的是协议的方法集合（每个协议都会实现不同的方法集）

```CPP
//file: net/ipv4/af_inet.c
//IPv4 Internet协议族来说,每种协议都有对应的处理方法
const struct proto_ops inet_stream_ops = {
    ......
    .recvmsg       = inet_recvmsg,	//
    .mmap          = sock_no_mmap,    
}

//udp是通过inet_dgram_ops来定义的，其中注册了inet_recvmsg方法
const struct proto_ops inet_dgram_ops = {
    ......
    .sendmsg       = inet_sendmsg,
    .recvmsg       = inet_recvmsg,  //UDP的处理方法
}
```

`struct sock *sk`结构的`sk_prot`成员定义了二级处理函数。对于UDP协议来说，会被设置成UDP协议实现的方法集`udp_prot`

```CPP
//file: net/ipv4/udp.c
struct proto udp_prot = {
    .name          = "UDP",
    .owner         = THIS_MODULE,
    .close         = udp_lib_close,
    .connect       = ip4_datagram_connect,
    ......
    .sendmsg       = udp_sendmsg,
    .recvmsg       = udp_recvmsg,
    .sendpage      = udp_sendpage,    
}
```

[inet_recvmsg](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/af_inet.c#L784)的实现最后会调用`sk->sk_prot->recvmsg`进行后续处理，对于UDP协议的socket来说（`sk_prot`就是`struct proto udp_prot`），该实现就是`udp_recvmsg`

```CPP
//file: net/ipv4/af_inet.c
int inet_recvmsg(struct socket *sock, struct msghdr *msg, size_t size,
		 int flags)
{
	struct sock *sk = sock->sk;
	int addr_len = 0;
	int err;

	sock_rps_record_flow(sk);

	err = sk->sk_prot->recvmsg(sk, msg, size, flags & MSG_DONTWAIT,
				   flags & ~MSG_DONTWAIT, &addr_len);
	if (err >= 0)
		msg->msg_namelen = addr_len;
	return err;
}
EXPORT_SYMBOL(inet_recvmsg);
```

####	sys_recvfrom的调用链
![sys_recvfrom](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/sys_recvfrom.png)

####	udp_recvmsg的核心逻辑
`udp_recvmsg`函数包含下面的主要工作：
1. 取数据包：`__skb_recv_datagram()`函数，从`sk_receive_queue`上取一个`skb`
2. 拷贝数据（内核空间copy到用户空间）：`skb_copy_datagram_iovec()`或`skb_copy_and_csum_datagram_iovec()`
3. 必要时计算校验和：`skb_copy_and_csum_datagram_iovec()`

从内核[udp_recvmsg](https://elixir.bootlin.com/linux/v4.11.6/source/net/ipv4/udp.c#L1408)实现，寻找收包的调用链，最终会调用`__skb_recv_datagram`[函数](https://elixir.bootlin.com/linux/v4.11.6/source/net/core/datagram.c#L270)

```TEXT
udp_recvmsg
	--->__skb_recv_udp
		---> __skb_recv_datagram
			--->__skb_try_recv_datagram
```

`__skb_recv_datagram`的实现如下，从`__skb_try_recv_datagram`函数代码可知[收包过程](https://elixir.bootlin.com/linux/v4.11.6/source/net/core/datagram.c#L206)就是访问`sk->sk_receive_queue`；如果没有数据，且用户允许等待，则将调用`__skb_wait_for_more_packets()`执行等待操作（会让用户进程进入睡眠状态）

```CPP
struct sk_buff *__skb_recv_datagram(struct sock *sk, unsigned int flags,
				    void (*destructor)(struct sock *sk,
						       struct sk_buff *skb),
				    int *peeked, int *off, int *err)
{
	struct sk_buff *skb, *last;
	long timeo;

	timeo = sock_rcvtimeo(sk, flags & MSG_DONTWAIT);

	do {
		//
		skb = __skb_try_recv_datagram(sk, flags, destructor, peeked,
					      off, err, &last);
		if (skb)
			return skb;

		if (*err != -EAGAIN)
			break;
	} while (timeo &&
		!__skb_wait_for_more_packets(sk, err, &timeo, last));

	return NULL;
}
EXPORT_SYMBOL(__skb_recv_datagram);
```

####	一图以蔽之
![recv](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/refs/heads/master/blog_img/kernel/stack/latelee_excellent_send_and_recv_arch_kernel3.17.1_note.png)

##	0x05	主要hook点
从上图中标注的关键节点，列举下内核接收包主要的hooks，如下：

1、[`tracepoint:net:netif_receive_skb`](https://elixir.bootlin.com/linux/v4.11.6/source/net/core/dev.c#L4296)

```CPP
int netif_receive_skb(struct sk_buff *skb)
{
	trace_netif_receive_skb_entry(skb);

	return netif_receive_skb_internal(skb);
}
EXPORT_SYMBOL(netif_receive_skb);
```

##  0x06  参考
-   [Monitoring and Tuning the Linux Networking Stack: Sending Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-sending-data/)
-   [Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/monitoring-tuning-linux-networking-stack-receiving-data/)
-   [Linux网络 - 数据包的发送过程](https://segmentfault.com/a/1190000008926093)
-   [Linux 内核数据流概览初探](http://klworldy.com/posts/kerneldataflow/)
-   [Linux是怎么从网络上接收数据包的](https://mp.weixin.qq.com/s/gVYdRWCoQwpFtsKXMyIHRg)
-   [linux网络子系统研究：数据收发简略流程图](https://www.latelee.cn/net-study/linux-network-data-recv-send.html)
-   [eBPF Docs Program context __sk_buff](https://docs.ebpf.io/linux/program-context/__sk_buff/)
-   [25 张图，一万字，拆解 Linux 网络包发送过程](https://mp.weixin.qq.com/s?__biz=Mzg3ODUxNzM0MA==&mid=2247484532&idx=2&sn=2a934f8d6ae87b53b62671972987388d&scene=21#wechat_redirect)
-   [Linux网络 - 数据包的接收过程](https://segmentfault.com/a/1190000008836467)
-   [Linux 网卡数据收发过程分析](https://mp.weixin.qq.com/s?__biz=MzA3NzYzODg1OA==&mid=2648464515&idx=1&sn=117fa172f80cda8f446a1eb5e191464f&scene=21#wechat_redirect)
-   [图解Linux网络包接收过程](https://cloud.tencent.com/developer/article/1966873)
-   [Illustrated Guide to Monitoring and Tuning the Linux Networking Stack: Receiving Data](https://blog.packagecloud.io/illustrated-guide-monitoring-tuning-linux-networking-stack-receiving-data/)
-   [内核网络中的GRO、RFS、RPS技术介绍和调优](https://www.kerneltravel.net/blog/2020/network_ljr9/)
-   <<深入理解Linux网络技术内幕>>
-   [sk_buff 简介](https://www.llcblog.cn/2020/10/26/how-sk-buff-work/)
-   [【Linux】网络专题（二）——核心数据结构sk_buff](https://void-star.icu/archives/939)
-	[Linux 网络栈接收数据（RX）：配置调优（2022）](https://arthurchiao.art/blog/linux-net-stack-tuning-rx-zh/)
-   [Linux 网络栈接收数据（RX）：原理及内核实现（2022）](https://arthurchiao.art/blog/linux-net-stack-implementation-rx-zh/)