---
layout: post
title: Golang 网络编程：UDP 的若干细节
subtitle: 基于 Golang Udp 的高性能编程总结
date: 2022-03-03
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - UDP
  - 系统编程
  - Golang
---

##	0x00	前言
内网 UDP 的好处：
-	内网的 UDP 丢包率极小（低于万分之三）
-	发送端和接收端约定好通信协议，为了避免分片，每个 UDP 包的最大字节数应该是 `1500-20-8=1472`

##	0x01	golang-UDP 的连接性？
当然，这里的连接性指的是 Lib 层面，golang 中 UDP 分为已连接和未连接两种方式，二者在发送、接收消息行为模式上有重大区别

- 已连接状态：通过 `DialUDP` 创建的 UDP 为已连接状态，其会记录远端 remote 的 ip 及 port 信息，相当于在两者之间建立了持续通路，发送、接收函数为 Write、Read，不需要填 remote 信息

- 未连接状态：通过 `ListenUDP` 建立的 UDP 为未连接形式，发送、接收函数为 `WriteTo`、`ReadFrom`，需要填写 remote 信息

在实际应用中，如果需要在不同的连接上完成 UDP 收发，那么使用 `DialUDP` 就会有问题，如下面的场景：

##	0x02	Golang 中的实现细节

####  DialUDP 的实现
```golang
func DialUDP(network string, laddr, raddr *UDPAddr) (*UDPConn, error) {
	switch network {
	case "udp", "udp4", "udp6":
	default:
		return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: raddr.opAddr(), Err: UnknownNetworkError(network)}
	}
	if raddr == nil {
		return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: nil, Err: errMissingAddress}
	}
	sd := &sysDialer{network: network, address: raddr.String()}
	c, err := sd.dialUDP(context.Background(), laddr, raddr)
	if err != nil {
		return nil, &OpError{Op: "dial", Net: network, Source: laddr.opAddr(), Addr: raddr.opAddr(), Err: err}
	}
	return c, nil
}

//sd.dialUDP
func (sd *sysDialer) dialUDP(ctx context.Context, laddr, raddr *UDPAddr) (*UDPConn, error) {
	fd, err := internetSocket(ctx, sd.network, laddr, raddr, syscall.SOCK_DGRAM, 0, "dial", sd.Dialer.Control)
	if err != nil {
		return nil, err
	}
	return newUDPConn(fd), nil
}
```

####  ListenUDP 的实现
```golang
func ListenUDP(network string, laddr *UDPAddr) (*UDPConn, error) {
	switch network {
	case "udp", "udp4", "udp6":
	default:
		return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: laddr.opAddr(), Err: UnknownNetworkError(network)}
	}
	if laddr == nil {
		laddr = &UDPAddr{}
	}
	sl := &sysListener{network: network, address: laddr.String()}
	c, err := sl.listenUDP(context.Background(), laddr)
	if err != nil {
		return nil, &OpError{Op: "listen", Net: network, Source: nil, Addr: laddr.opAddr(), Err: err}
	}
	return c, nil
}

func (sl *sysListener) listenUDP(ctx context.Context, laddr *UDPAddr) (*UDPConn, error) {
	fd, err := internetSocket(ctx, sl.network, laddr, nil, syscall.SOCK_DGRAM, 0, "listen", sl.ListenConfig.Control)
	if err != nil {
		return nil, err
	}
	return newUDPConn(fd), nil
}
```

####  底层调用
`sl.listenUDP`、`sd.dialUDP` 都会调用底层的 `internetSocket`，只是调用参数有去呗，`listenUDP` 调用时 `raddr` 为 `nil`，而 `dialUDP` 会传入此值；`internetSocket` 内部会调用 `socket` 函数，如下：


```golang
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
	s, err := sysSocket(family, sotype, proto)
	if err != nil {
		return nil, err
	}
	if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}
	if fd, err = newFD(s, family, sotype, net); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}

	// This function makes a network file descriptor for the
	// following applications:
	//
	// - An endpoint holder that opens a passive stream
	//   connection, known as a stream listener
	//
	// - An endpoint holder that opens a destination-unspecific
	//   datagram connection, known as a datagram listener
	//
	// - An endpoint holder that opens an active stream or a
	//   destination-specific datagram connection, known as a
	//   dialer
	//
	// - An endpoint holder that opens the other connection, such
	//   as talking to the protocol stack inside the kernel
	//
	// For stream and datagram listeners, they will only require
	// named sockets, so we can assume that it's just a request
	// from stream or datagram listeners when laddr is not nil but
	// raddr is nil. Otherwise we assume it's just for dialers or
	// the other connection holders.

  //laddr 不为 nil，而 raddr 为 nil，说明是监听 socket（listenUDP）
	if laddr != nil && raddr == nil {
		switch sotype {
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
			if err := fd.listenStream(laddr, listenerBacklog(), ctrlFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		case syscall.SOCK_DGRAM:
			if err := fd.listenDatagram(laddr, ctrlFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
	}

  //DialUDP
	if err := fd.dial(ctx, laddr, raddr, ctrlFn); err != nil {
		fd.Close()
		return nil, err
	}
	return fd, nil
}
```

####  底层调用：dial
`dial` 方法解答了 `ListenUDP`、`DialUDP` 的行为差异的原因：
- `laddr!=nil` 时，调用 `bind` 方法，绑定本地 ip 及 port
- `raddr!=nil` 时，调用 `fd.connect` 与 remote 建立连接（`raddr` 的区别）

根据上小节的调用参数看，`ListenUDP` 方法构造的 UDPConn 为未连接状态，而 `DialUDP` 方法构造的 UDPConn 为已连接状态，因而 `DialUDP` 方法构建的 UDPConn 只能从指定 remote 接收数据，而 `ListenUDP` 方法构建的 UDPConn 则可以从任何远端接收数据

```golang
func (fd *netFD) dial(ctx context.Context, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) error {
	if ctrlFn != nil {
		c, err := newRawConn(fd)
		if err != nil {
			return err
		}
		var ctrlAddr string
		if raddr != nil {
			ctrlAddr = raddr.String()
		} else if laddr != nil {
			ctrlAddr = laddr.String()
		}
		if err := ctrlFn(fd.ctrlNetwork(), ctrlAddr, c); err != nil {
			return err
		}
	}
	var err error
	var lsa syscall.Sockaddr
	if laddr != nil {
    //laddr not nil
		if lsa, err = laddr.sockaddr(fd.family); err != nil {
			return err
		} else if lsa != nil {
			if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
				return os.NewSyscallError("bind", err)
			}
		}
	}
	var rsa syscall.Sockaddr  // remote address from the user
	var crsa syscall.Sockaddr // remote address we actually connected to
	if raddr != nil {
    //raddr not nil
		if rsa, err = raddr.sockaddr(fd.family); err != nil {
			return err
		}
		if crsa, err = fd.connect(ctx, lsa, rsa); err != nil {
			return err
		}
		fd.isConnected = true
	} else {
		if err := fd.init(); err != nil {
			return err
		}
	}
	// Record the local and remote addresses from the actual socket.
	// Get the local address by calling Getsockname.
	// For the remote address, use
	// 1) the one returned by the connect method, if any; or
	// 2) the one from Getpeername, if it succeeds; or
	// 3) the one passed to us as the raddr parameter.
	lsa, _ = syscall.Getsockname(fd.pfd.Sysfd)
	if crsa != nil {
		fd.setAddr(fd.addrFunc()(lsa), fd.addrFunc()(crsa))
	} else if rsa, _ = syscall.Getpeername(fd.pfd.Sysfd); rsa != nil {
		fd.setAddr(fd.addrFunc()(lsa), fd.addrFunc()(rsa))
	} else {
		fd.setAddr(fd.addrFunc()(lsa), raddr)
	}
	return nil
}
```

##	0x03	UDP 优化

####  服务端优化

####  客户端优化
- 减少锁竞争：实例化多个 UDP 连接到一个 slice 中，在客户端代码里随机使用 slice 的 UDP 进行 连接

##	0x04	总结


##	0x05	参考
- [如何在 Go 中实现百万级 UDP 通信](https://zhuanlan.zhihu.com/p/357902432)
- [几行代码为老板省百万 - 某高并发服务 Go GC 及 UDP Pool 优化](https://mp.weixin.qq.com/s/YAz5NyiNWJCMlGsJRTAaxw)
- [golang 中 udp 的连接性](https://zhuanlan.zhihu.com/p/94680036)