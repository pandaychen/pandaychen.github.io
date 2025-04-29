---
layout:     post
title:      netlink 应用
subtitle:   如何基于 netlink 机制实现进程监控
date:       2024-10-09
author:     pandaychen
header-img:
catalog: true
tags:
    - netlink
---

##  0x00    前言
本文介绍下基于 netlink 机制构建进程创建审计监控，Netlink 是一个套接字家族（socket family），被用于内核与用户态进程以及用户态进程之间的 IPC 通信

[Netlink Connector](https://github.com/torvalds/linux/tree/master/drivers/connector) 是一种特殊的基于 Netlink 协议的通信机制（协议号是 `NETLINK_CONNECTOR`），它构建在 Linux 内核中，用于内核与用户空间应用之间的通信。Netlink 本身是一种灵活的 IPC 机制，主要用于网络配置和管理，但其使用范围已经扩展到了各种系统事件的通知。Netlink Connector 则专门用于传递事件和消息，包括进程事件，如进程创建和终止等；此外 Netlink Connector 使用标准的 Netlink 套接字和通信机制，但它专注于事件的传递，它允许内核组件注册事件源，并将这些事件广播给订阅这些事件的用户空间应用程序

比如常用的 `ip`（如 `ip link add xxxx`）/`ss` 等命令，都是使用 netlink 去跟内核通信实现（`ss`是通过 Netlink 与内核通信获取的信息）

![arch](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/netlink/netlink-process.jpg?raw=true)

linux 内核从 `2.6.15` 版本开始支持 connector，可以检查内核模块配置文件是否配置 `CONFIG_CONNECTOR`，`CONFIG_PROC_EVENTS`

```BASH
[root@VM-x-x-centos edriver]# cat /boot/config-$(uname -r)| egrep 'CONFIG_CONNECTOR|CONFIG_PROC_EVENTS'
CONFIG_CONNECTOR=y
CONFIG_PROC_EVENTS=y
```

相关代码：[`Connector`](https://github.com/torvalds/linux/tree/master/drivers/connector)，其中 `connectors.c` 和 `cnqueue.c` 是 Netlink Connector 的实现，而 `cnproc.c` 是一个应用实例，名为进程事件连接器，可以通过该连接器来实现对进程创建的监控

####	通信机制的比较
用户空间和内核空间的通信方式通常有三种：`/proc`、ioctl以及netlink，前两种都是单向的，而netlink可以实现双工通信。netlink机制的优点如下：

1.	netlink是一种异步通信机制，在内核与用户态应用之间传递的消息保存在socket缓存队列中，发送消息只是把消息保存在接收者的socket的接 收队列，而不需要等待接收者收到消息，但系统调用与 ioctl 则是同步通信机制，如果传递的数据太长，将影响调度粒度
2.	netlink 支持多播，内核模块或应用可以把消息多播给一个netlink组，属于该neilink 组的任何内核模块或应用都能接收到该消息，内核事件向用户态的通知机制就使用了这一特性，任何对内核事件感兴趣的应用都能收到该子系统发送的内核事件
3.	内核可以使用 netlink 首先发起会话，但系统调用和 ioctl 只能由用户应用发起调用
4.	netlink 使用标准的 socket API
5.	netlink协议基于BSD socket和AF_NETLINK地址簇，使用`32`位的端口号寻址，每个Netlink协议通常与一个或一组内核服务/组件相关联

####    关键组件（内核态 + 用户态）

1、连接器模块（Connector Module），内核模块，负责管理事件的注册和广播，处理来自用户空间的订阅请求，并在相应的事件发生时向订阅者广播通知

2、用户空间应用程序：通过创建 Netlink 套接字并绑定到特定的 Netlink 协议（如 `NETLINK_CONNECTOR`）来与内核模块通信。应用程序发送订阅请求到内核，并监听来自内核的事件通知

##  0x01    Netlink Connector 开发
netlink connector 的开发流程较为简洁，如下：

-   内核层配置：确保内核配置中启用了 Netlink Connector 支持（见上面）
-   用户空间监听程序
    -   创建一个 Netlink socket
    -   绑定到 Netlink Connector
    -   发送订阅消息给内核，表明它希望接收特定类型的事件（如 `PROC_EVENT_FORK`、`PROC_EVENT_EXECVE` 等）
    -   监听套接字并处理接收到的事件通知

![flow](https://github.com/pandaychen/pandaychen.github.io/blob/master/blog_img/netlink/process-event-flow.png?raw=true)

####    基于 golang 的库：vishvananda/netlink
[vishvananda/netlink：Simple netlink library for go](https://github.com/vishvananda/netlink)，该项目给出了一个linux下的事件接收框架[实现](https://github.com/vishvananda/netlink/blob/main/proc_event_linux.go)

例如，此库封装了 `NetlinkSocket`[结构](https://github.com/vishvananda/netlink/blob/main/nl/nl_linux.go#L689)，

```GO
type NetlinkSocket struct {
	fd             int32
	file           *os.File
	lsa            unix.SockaddrNetlink
	sendTimeout    int64 // Access using atomic.Load/StoreInt64
	receiveTimeout int64 // Access using atomic.Load/StoreInt64
	sync.Mutex
}
```

通过 [`SubscribeAt`](https://github.com/vishvananda/netlink/blob/main/nl/nl_linux.go#L821) 方法生成一个 `NetlinkSocket`，`SubscribeAt`最终会调用`Subscribe`[方法](https://github.com/vishvananda/netlink/blob/main/nl/nl_linux.go#L791C6-L791C15)：

```GO
// SubscribeAt works like Subscribe plus let's the caller choose the network
// namespace in which the socket would be opened (newNs). Then control goes back
// to curNs if open, otherwise to the netns at the time this function was called.
func SubscribeAt(newNs, curNs netns.NsHandle, protocol int, groups ...uint) (*NetlinkSocket, error) {
	c, err := executeInNetns(newNs, curNs)
	if err != nil {
		return nil, err
	}
	defer c()
	return Subscribe(protocol, groups...)
}
```

####	vishvananda/netlink的两种典型用法
1、 通过 netlink 查询接口信息（socket info），这种工作模式是用户态进程通过 netlink socket 主动发送一个请求消息给内核，内核处理该请求后返回一个响应消息。整个过程是同步的，用户态进程会阻塞等待响应。通常应用于需要一次性获取数据或执行某个操作，例如查询网络接口信息、路由表、修改某个配置项等，代码片段如下：

```GO
//通过 Linux Netlink 的 INET_DIAG 子系统​实现了查询网络套接字状态信息的功能，尤其是TCP/UDP套接字的详细诊断信息。其作用类似于命令行工具 ss 或 netstat，但直接通过内核接口获取数据
// subscribe the netlink

	//订阅 INET_DIAG 协议，NETLINK_INET_DIAG​是 Netlink 协议族，专用于查询IPv4/IPv6 套接字诊断信息​（如 TCP/UDP 状态、统计信息）
	if s, err = nl.Subscribe(unix.NETLINK_INET_DIAG); err != nil {
		return
	}
	defer s.Close()
	// send the netlink request
	//构造 Netlink 请求
	//SOCK_DIAG_BY_FAMILY表示按协议族（如 AF_INET 或 AF_INET6）查询套接字
	//NLM_F_DUMP 为标志位，表示请求内核返回所有匹配的套接字信息（类似 GET ALL）
	req = nl.NewNetlinkRequest(nl.SOCK_DIAG_BY_FAMILY, unix.NLM_F_DUMP)
	
	//添加过滤条件
	req.AddData(&socketRequest{
		Family:   family,	// 协议族：AF_INET（IPv4）或 AF_INET6（IPv6）
		Protocol: protocol,	// 协议类型：IPPROTO_TCP 或 IPPROTO_UDP
		Ext:      (1 << (INET_DIAG_VEGASINFO - 1)) | (1 << (INET_DIAG_INFO - 1)),	// 请求扩展信息
		States:   uint32(1 << state),	// 状态过滤：如 TCP_ESTABLISHED
	})
	if err = s.Send(req); err != nil {
		return
	}
```

2、类似于订阅模式（异步多播组），工作机制是用户态进程订阅一个或多个多播组，如此内核在特定事件发生时（如网络接口状态变化），主动向这些多播组推送通知消息，用户态进程无需主动轮询，异步接收事件通知即可。适用场景为需要实时监控内核事件，例如网络接口 UP/DOWN、路由表更新、进程审计事件等。代码片段如下：


```GO
//通过 Netlink 的 Connector 子系统订阅进程事件多播通知（例如进程创建、退出等）
//核心功能：订阅进程事件
//代码通过 NETLINK_CONNECTOR 协议订阅进程事件多播组（CN_IDX_PROC），实现实时监控进程生命周期，当系统中发生进程创建（fork）、退出（exit）等事件时，内核主动推送通知

	//构造 Netlink 消息
	nlmsg := nlmsg nl.NetlinkRequest{}	
	nlmsg.Pid = uint32(os.Getpid())	 // 设置发送者 PID（当前进程）
	nlmsg.Type = unix.NLMSG_DONE		 // 消息类型
	nlmsg.Len = uint32(unix.SizeofNlMsghdr)	// 消息头长度（需校验是否包含数据部分）
	nlmsg.AddData(
		// 添加 Connector 控制消息，其中CN_IDX_PROC 和 CN_VAL_PROC 标识 Connector 子系统中的进程事件通道
		// PROC_CN_MCAST_LISTEN表示订阅多播组的控制命令
		nl.NewCnMsg(CN_IDX_PROC,	// Connector 子系统索引（进程事件）
		 			CN_VAL_PROC,	 // Connector 子系统值（进程事件）
		  			PROC_CN_MCAST_LISTEN),	 // 控制命令：订阅多播组
	)

	//创建并订阅 netlink socket
	if n.sock, err = nl.SubscribeAt(
		netns.None(),		// PID 命名空间（None 表示当前命名空间）
		netns.None(),		 // 网络命名空间（None 表示当前命名空间）
		unix.NETLINK_CONNECTOR,	// 使用 Connector 协议
		CN_IDX_PROC); err != nil { 	// 订阅的多播组（进程事件）
		return err
	}

	//发送订阅请求
	if err = n.sock.Send(&nlmsg); err != nil {
		return err
	}
```

####    支持的 events 事件
事件列表 [如下](https://github.com/vishvananda/netlink/blob/main/proc_event_linux.go#L17)

```GO
const (
	PROC_EVENT_NONE     = 0x00000000
	PROC_EVENT_FORK     = 0x00000001
	PROC_EVENT_EXEC     = 0x00000002
	PROC_EVENT_UID      = 0x00000004
	PROC_EVENT_GID      = 0x00000040
	PROC_EVENT_SID      = 0x00000080
	PROC_EVENT_PTRACE   = 0x00000100
	PROC_EVENT_COMM     = 0x00000200
	PROC_EVENT_COREDUMP = 0x40000000
	PROC_EVENT_EXIT     = 0x80000000
)
```

| 事件类型 | 作用	 | 触发条件 |
| :-----:| :----: | :----: |
| `PROC_EVENT_FORK` | 进程 fork 事件，返回进程 id，内核进程 id，进程父 id，内核进程父 id | 系统调用 `fork`、`vfork` |
| `PROC_EVENT_EXEC` | 进程 exec 事件，返回进程 id，内核进程 id | 系统调用 `execl`、 `execlp`、 `execle`、 `execv`、 `execvp`、 `execvpe` |
| `PROC_EVENT_UID`，`PROC_EVENT_GID` | 进程 id 事件，返回进程 id，内核进程 id，uid 和 gid 或 euid、egid | 系统调用 `setuid`、`seteuid`、`setreuid`、`setgid`、`setegid`、`setregid` |
| `PROC_EVENT_SID` | 进程 sid 事件，返回进程 id，内核进程 id | 系统调用 `setsid` |
| `PROC_EVENT_PTRACE` | 进程被调试事件，进程 id，内核进程 id，调试器进程 id，调试器内核进程 id | 系统调用 `ptrace` |
| `PROC_EVENT_COMM` | 对进程属性操作的事件，返回进程 id，内核进程 id，进程名称 | 系统调用 `prctl` |
| `PROC_EVENT_COREDUMP` | 进程 coredump 的事件，返回进程 id，内核进程 id | 各种异常信号 |
| `PROC_EVENT_EXIT` | 进程退出事件 ，返回进程 id，内核进程 id，退出码，退出信号 | 异常信号，被杀死，异常退出或正常退出 |

几个要点：
-	通过 `PROC_EVENT_FORK`、 `PROC_EVENT_EXEC`、 `PROC_EVENT_EXIT` 可以获取进程启动和退出事件
-	`PROC_EVENT_COMM` 事件可以获取进程名称，但需要调用 `prctl`，在内核里会对进程结构加锁，不建议在高并发场景使用。降级方案是通过读取 `proc` 文件系统获取获取进程name，缺点是对短时进程无法获取


##	0x02	进程监控方案对比&&项目实践

| 方案 | Docker兼容性	 | 数据准确性 | 系统侵入性|
| :-----:| :----: | :----: | :----: | 
| cn_proc | 不支持Docker | 存在内核拿到的PID，在`/proc/`下丢失的情况 |  无 |
| Audit | 不支持Docker | 同上 |  依赖Auditd |
| Hook | 定制 | 精确 |  强 |


##  0x03 参考
-	[深入理解Linux Netlink机制：进程间通信的关键](https://zhuanlan.zhihu.com/p/693908092)
-   [连接器（Netlink_Connector）及其应用](https://imagine4077.github.io/Hogwarts/c/2016/05/02/%E8%BF%9E%E6%8E%A5%E5%99%A8-Netlink_Connector-%E5%8F%8A%E5%85%B6%E5%BA%94%E7%94%A8.html)
-   [netlink socket 编程 --why & how](https://e-mailky.github.io/2017-02-14-netlink)
-	[保障IDC安全：分布式HIDS集群架构设计](https://tech.meituan.com/2019/01/17/distributed-hids-cluster-architecture-design.html)
-	[Copy: Linux process monitoring (exec, fork, exit, set*uid, set*gid)](https://github.com/ggrandes-clones/pmon/tree/master)