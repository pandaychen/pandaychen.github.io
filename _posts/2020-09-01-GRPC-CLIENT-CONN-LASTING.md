---
layout:     post
title:      gRPC 客户端长连接机制实现及 keepalive 分析
subtitle:   如何实现针对 gRPC 客户端的自动重连机制
date:       2020-09-01
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - gRPC
---


##  0x00    前言
`HTTP2` 是一个全双工的流式协议, 服务端也可以主动 `ping` 客户端, 且服务端还会有一些检测连接可用性和控制客户端 `ping` 包频率的配置。gRPC 就是采用 `HTTP2` 来作为其基础通信模式的，所以默认的 gRPC 都是长连接。<br>

有这么一种场景，需要客户端和服务端保持持久的长连接，即无论服务端、客户端异常断开或重启，长连接都要具备重试保活（当然前提是两方重启都成功）的需求。在 gRPC 中，对于已经建立的长连接，服务端异常重启之后，客户端一般会收到如下错误：<br>

> rpc error: code = Unavailable desc = transport is closing

大部分的 gRPC 客户端封装都没有很好的处理这类 case，参见 [Warden 关于 Server 端服务重启后 Client 连接断开之后的重试问题](https://github.com/go-kratos/kratos/issues/177)，对于这种错误，推荐有两种处理方法：<br>
1.  重试：在客户端调用失败时，选择以指数退避（Exponential Backoff ）来优雅进行重试
2.  增加 keepalive 的保活策略
3.  增加重连（auto reconnect）策略

这篇文章就来分析下如何实现这样的客户端保活（keepalive）逻辑。提到保活机制，我们先看下 gRPC 的 [keepalive 机制](https://github.com/grpc/grpc/blob/master/doc/keepalive.md)。

##  0x01    gRPC 客户端 keepalive
gRPC 客户端提供 keepalive 配置如下：
```golang
var kacp = keepalive.ClientParameters{
	Time:                10 * time.Second, // send pings every 10 seconds if there is no activity
	Timeout:             time.Second,      // wait 1 second for ping ack before considering the connection dead
	PermitWithoutStream: true,             // send pings even without active streams
}
//Dial 中传入 keepalive 配置
conn, err := grpc.Dial(*addr, grpc.WithInsecure(), grpc.WithKeepaliveParams(kacp))
```

`keepalive.ClientParameters` 参数的含义如下:
-   `Time`：如果没有 activity， 则每隔 `10s` 发送一个 ping 包
-   `Timeout`： 如果 ping ack 1s 之内未返回则认为连接已断开
-   `PermitWithoutStream`：如果没有 active 的 stream， 是否允许发送 ping

联想到，在项目中 [`ssh 客户端 `](https://pandaychen.github.io/2019/10/20/HOW-TO-BUILD-A-SSHD-WITH-GOLANG/#%E5%AE%A2%E6%88%B7%E7%AB%AF-keepalive-%E6%9C%BA%E5%88%B6) 和 `mysql 客户端 ` 中都有着类似的实现，即单独开启协程来实现 keepalive：
如下面的代码（以 `ssh` 为例）：
```golang
go func() {
    t := time.NewTicker(2 * time.Second)
    defer t.Stop()
    for range t.C {
        _, _, err := client.Conn.SendRequest("keepalive@golang.org", true, nil)
        if err != nil {
            return
        }
    }
}()
```

####    gPRC 的实现
在 grpc-go 的 [newHTTP2Client](https://github.com/grpc/grpc-go/blob/master/internal/transport/http2_client.go#L166) 方法中，有下面的逻辑：<br>
即在新建一个 `HTTP2Client` 的时候会启动一个 `goroutine` 来处理 keepalive
```golang
// newHTTP2Client constructs a connected ClientTransport to addr based on HTTP2
// and starts to receive messages on it. Non-nil error returns if construction
// fails.
func newHTTP2Client(connectCtx, ctx context.Context, addr resolver.Address, opts ConnectOptions, onPrefaceReceipt func(), onGoAway func(GoAwayReason), onClose func()) (_ *http2Client, err error) {
    ...
	if t.keepaliveEnabled {
		t.kpDormancyCond = sync.NewCond(&t.mu)
		go t.keepalive()
    }
    ...
}
```

接下来，看下 [`keepalive` 方法](https://github.com/grpc/grpc-go/blob/master/internal/transport/http2_client.go#L1350) 的实现：
```golang
func (t *http2Client) keepalive() {
	p := &ping{data: [8]byte{}} //ping 的内容
	timer := time.NewTimer(t.kp.Time) // 启动一个定时器, 触发时间为配置的 Time 值
	//for loop
	for {
		select {
		// 定时器触发
		case <-timer.C:
			if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
				timer.Reset(t.kp.Time)
				continue
			}
			// Check if keepalive should go dormant.
			t.mu.Lock()
			if len(t.activeStreams) < 1 && !t.kp.PermitWithoutStream {
				// Make awakenKeepalive writable.
				<-t.awakenKeepalive
				t.mu.Unlock()
				select {
				case <-t.awakenKeepalive:
					// If the control gets here a ping has been sent
					// need to reset the timer with keepalive.Timeout.
				case <-t.ctx.Done():
					return
				}
			} else {
				t.mu.Unlock()
				if channelz.IsOn() {
					atomic.AddInt64(&t.czData.kpCount, 1)
				}
				// Send ping.
				t.controlBuf.put(p)
			}

			// By the time control gets here a ping has been sent one way or the other.
			timer.Reset(t.kp.Timeout)
			select {
			case <-timer.C:
				if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
					timer.Reset(t.kp.Time)
					continue
				}
				t.Close()
				return
			case <-t.ctx.Done():
				if !timer.Stop() {
					<-timer.C
				}
				return
			}
		//context 结束
		case <-t.ctx.Done():
			if !timer.Stop() {
				<-timer.C
			}
			return
		}
	}
}
```

从客户端的 `keepalive` 实现中梳理下执行逻辑：
1.  填充 `ping` 包内容, 为 `[8]byte{}`，创建定时器, 触发时间为用户配置中的 `Time`
2.  循环处理，select 的两大分支，一为定时器触发后执行的逻辑，另一分支为 `t.ctx.Done()`，即 `keepalive` 的上层应用调用了 `cancel` 结束 context 子树
3.  核心逻辑在定时器触发的过程中

##  0x03    gRPC 服务端的 keepalive


##  0x04    实现健壮的长连接客户端

##  0x05    参考
-   [GRPC 开箱手册](https://juejin.im/post/6844904096474857485)
