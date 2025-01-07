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

通过 [`SubscribeAt`](https://github.com/vishvananda/netlink/blob/main/nl/nl_linux.go#L821) 方法生成一个 `NetlinkSocket`：

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
-   [连接器（Netlink_Connector）及其应用](https://imagine4077.github.io/Hogwarts/c/2016/05/02/%E8%BF%9E%E6%8E%A5%E5%99%A8-Netlink_Connector-%E5%8F%8A%E5%85%B6%E5%BA%94%E7%94%A8.html)
-   [netlink socket 编程 --why & how](https://e-mailky.github.io/2017-02-14-netlink)
-	[保障IDC安全：分布式HIDS集群架构设计](https://tech.meituan.com/2019/01/17/distributed-hids-cluster-architecture-design.html)
-	[Copy: Linux process monitoring (exec, fork, exit, set*uid, set*gid)](https://github.com/ggrandes-clones/pmon/tree/master)