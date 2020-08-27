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
`HTTP2` 是一个全双工的流式协议, 服务端也可以主动 `ping` 客户端, 且服务端还会有一些检测连接可用性和控制客户端 `ping` 包频率的配置。gRPC 就是采用 `HTTP2` 来作为其基础通信模式的，所以默认的 gRPC 客户端都是长连接。<br>

有这么一种场景，需要客户端和服务端保持持久的长连接，即无论服务端、客户端异常断开或重启，长连接都要具备重试保活（当然前提是两方重启都成功）的需求。在 gRPC 中，对于已经建立的长连接，服务端异常重启之后，客户端一般会收到如下错误：<br>

> rpc error: code = Unavailable desc = transport is closing

大部分的 gRPC 客户端封装都没有很好的处理这类 case，参见 [Warden 关于 Server 端服务重启后 Client 连接断开之后的重试问题](https://github.com/go-kratos/kratos/issues/177)，对于这种错误，推荐有两种处理方法：<br>
1.  重试：在客户端调用失败时，选择以指数退避（Exponential Backoff ）来优雅进行重试
2.  增加 keepalive 的保活策略
3.  增加重连（auto reconnect）策略

这篇文章就来分析下如何实现这样的客户端保活（keepalive）逻辑。提到保活机制，我们先看下 gRPC 的 [keepalive 机制](https://github.com/grpc/grpc/blob/master/doc/keepalive.md)。

##	0x01	HTTP2 的 GOAWAY 帧
`HTTP2` 使用 GOAWAY 帧信号来控制连接关闭，GOAWAY 用于启动连接关闭或发出严重错误状态信号。<br>
GOAWAY 语义为允许端点正常停止接受新的流，同时仍然完成对先前建立的流的处理，当 client 收到这个包之后就会主动关闭连接。下次需要发送数据时，就会重新建立连接。GOAWAY 是实现 `grpc.gracefulStop` 机制的重要保证。


##  0x02    gRPC 客户端 keepalive
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

联想到，在项目中 [`ssh` 客户端](https://pandaychen.github.io/2019/10/20/HOW-TO-BUILD-A-SSHD-WITH-GOLANG/#%E5%AE%A2%E6%88%B7%E7%AB%AF-keepalive-%E6%9C%BA%E5%88%B6) 和 `mysql` 客户端中都有着类似的实现，即单独开启协程来实现 keepalive：
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
		// 上层通知 context 结束
		case <-t.ctx.Done():
			if !timer.Stop() {
				// 返回 false，表示 timer 未被销毁
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
gRPC 的服务端主要有两块逻辑：
1.	接收并相应客户端的 ping 包
2.	单独启动 goroutine 探测客户端是否存活

gRPC 服务端提供 keepalive 配置，分为两部分 `keepalive.EnforcementPolicy` 和 `keepalive.ServerParameters`，如下：
```golang
var kaep = keepalive.EnforcementPolicy{
	MinTime:             5 * time.Second, // If a client pings more than once every 5 seconds, terminate the connection
	PermitWithoutStream: true,            // Allow pings even when there are no active streams
}

var kasp = keepalive.ServerParameters{
	MaxConnectionIdle:     15 * time.Second, // If a client is idle for 15 seconds, send a GOAWAY
	MaxConnectionAge:      30 * time.Second, // If any connection is alive for more than 30 seconds, send a GOAWAY
	MaxConnectionAgeGrace: 5 * time.Second,  // Allow 5 seconds for pending RPCs to complete before forcibly closing connections
	Time:                  5 * time.Second,  // Ping the client if it is idle for 5 seconds to ensure the connection is still active
	Timeout:               1 * time.Second,  // Wait 1 second for the ping ack before assuming the connection is dead
}

func main(){
	...
	s := grpc.NewServer(grpc.KeepaliveEnforcementPolicy(kaep), grpc.KeepaliveParams(kasp))
	...
}
```

`keepalive.EnforcementPolicy`：
-	`MinTime`：如果客户端两次 ping 的间隔小于 `5s`，则关闭连接
-	`PermitWithoutStream`： 即使没有 active stream, 也允许 ping

`keepalive.ServerParameters`：
-	`MaxConnectionIdle`：如果一个 client 空闲超过 `15s`, 发送一个 GOAWAY, 为了防止同一时间发送大量 GOAWAY, 会在 `15s` 时间间隔上下浮动 `15*10%`, 即 `15+1.5` 或者 `15-1.5`
-	`MaxConnectionAge`：如果任意连接存活时间超过 `30s`, 发送一个 GOAWAY
-	`MaxConnectionAgeGrace`：在强制关闭连接之间, 允许有 `5s` 的时间完成 pending 的 rpc 请求
-	`Time`： 如果一个 client 空闲超过 `5s`, 则发送一个 ping 请求
-	`Timeout`： 如果 ping 请求 `1s` 内未收到回复, 则认为该连接已断开

####	gRPC 的实现
服务端处理客户端的 `ping` 包的 response 的逻辑在 [`handlePing` 方法](https://github.com/grpc/grpc-go/blob/master/internal/transport/http2_server.go#L693) 中。<br>
`handlePing` 方法会判断是否违反两条 policy, 如果违反则将 `pingStrikes++`, 当违反次数大于 `maxPingStrikes(2)` 时, 打印一条错误日志并且发送一个 goAway 包，断开这个连接，具体实现如下：
```golang
func (t *http2Server) handlePing(f *http2.PingFrame) {
	if f.IsAck() {
		if f.Data == goAwayPing.data && t.drainChan != nil {
			close(t.drainChan)
			return
		}
		// Maybe it's a BDP ping.
		if t.bdpEst != nil {
			t.bdpEst.calculate(f.Data)
		}
		return
	}
	pingAck := &ping{ack: true}
	copy(pingAck.data[:], f.Data[:])
	t.controlBuf.put(pingAck)

	now := time.Now()
	defer func() {
		t.lastPingAt = now
	}()
	// A reset ping strikes means that we don't need to check for policy
	// violation for this ping and the pingStrikes counter should be set
	// to 0.
	if atomic.CompareAndSwapUint32(&t.resetPingStrikes, 1, 0) {
		t.pingStrikes = 0
		return
	}
	t.mu.Lock()
	ns := len(t.activeStreams)
	t.mu.Unlock()
	if ns < 1 && !t.kep.PermitWithoutStream {
		// Keepalive shouldn't be active thus, this new ping should
		// have come after at least defaultPingTimeout.
		if t.lastPingAt.Add(defaultPingTimeout).After(now) {
			t.pingStrikes++
		}
	} else {
		// Check if keepalive policy is respected.
		if t.lastPingAt.Add(t.kep.MinTime).After(now) {
			t.pingStrikes++
		}
	}

	if t.pingStrikes > maxPingStrikes {
		// Send goaway and close the connection.
		if logger.V(logLevel) {
			logger.Errorf("transport: Got too many pings from the client, closing the connection.")
		}
		t.controlBuf.put(&goAway{code: http2.ErrCodeEnhanceYourCalm, debugData: []byte("too_many_pings"), closeConn: true})
	}
}

```

注意，对 `pingStrikes` 累加的逻辑：
-	`t.lastPingAt.Add(defaultPingTimeout).After(now)`：
-	`t.lastPingAt.Add(t.kep.MinTime).After(now)`：

```golang
func (t *http2Server) handlePing(f *http2.PingFrame) {
	...
	if ns < 1 && !t.kep.PermitWithoutStream {
		// Keepalive shouldn't be active thus, this new ping should
		// have come after at least defaultPingTimeout.
		if t.lastPingAt.Add(defaultPingTimeout).After(now) {
			t.pingStrikes++
		}
	} else {
		// Check if keepalive policy is respected.
		if t.lastPingAt.Add(t.kep.MinTime).After(now) {
			t.pingStrikes++
		}
	}
	if t.pingStrikes > maxPingStrikes {
		// Send goaway and close the connection.
		errorf("transport: Got too many pings from the client, closing the connection.")
		t.controlBuf.put(&goAway{code: http2.ErrCodeEnhanceYourCalm, debugData: []byte("too_many_pings"), closeConn: true})
	}
}
```

####	keepalive 相关代码
gRPC 服务端新建一个 HTTP2 server 的时候会启动一个单独的 goroutine 处理 keepalive 逻辑，[`newHTTP2Server` 方法](https://github.com/grpc/grpc-go/blob/master/internal/transport/http2_server.go#L129)：
```golang
func newHTTP2Server(conn net.Conn, config *ServerConfig) (_ ServerTransport, err error) {
	...
	go t.keepalive()
	...
}
```

简单分析下 `keepalive` 的实现，核心逻辑是启动 `3` 个定时器，分别为 `maxIdle`、`maxAge` 和 `keepAlive`，然后在 `for select` 中处理相关定时器触发事件：
-	`maxIdle` 逻辑： 判断 client 空闲时间是否超出配置的时间, 如果超时, 则调用 `t.drain`, 该方法会发送一个 GOAWAY 包
-	`maxAge` 逻辑： 触发之后首先调用 `t.drain` 发送 GOAWAY 包, 接着重置定时器, 时间设置为 `MaxConnectionAgeGrace`, 再次触发后调用 `t.Close()` 直接关闭（有些 graceful 的意味）
-	`keepalive` 逻辑： 首先判断 activity 是否为 `1`, 如果不是则置 `pingSent` 为 `true`, 并且发送 ping 包, 接着重置定时器时间为 `Timeout`, 再次触发后如果 activity 不为 1（即未收到 ping 的回复） 并且 `pingSent` 为 `true`, 则调用 `t.Close()` 关闭连接

```golang
func (t *http2Server) keepalive() {
	p := &ping{}
	var pingSent bool
	maxIdle := time.NewTimer(t.kp.MaxConnectionIdle)
	maxAge := time.NewTimer(t.kp.MaxConnectionAge)
	keepalive := time.NewTimer(t.kp.Time)
	// NOTE: All exit paths of this function should reset their
	// respective timers. A failure to do so will cause the
	// following clean-up to deadlock and eventually leak.
	defer func() {
		// 退出前，完成定时器的回收工作
		if !maxIdle.Stop() {
			<-maxIdle.C
		}
		if !maxAge.Stop() {
			<-maxAge.C
		}
		if !keepalive.Stop() {
			<-keepalive.C
		}
	}()
	for {
		select {
		case <-maxIdle.C:
			t.mu.Lock()
			idle := t.idle
			if idle.IsZero() { // The connection is non-idle.
				t.mu.Unlock()
				maxIdle.Reset(t.kp.MaxConnectionIdle)
				continue
			}
			val := t.kp.MaxConnectionIdle - time.Since(idle)
			t.mu.Unlock()
			if val <= 0 {
				// The connection has been idle for a duration of keepalive.MaxConnectionIdle or more.
				// Gracefully close the connection.
				t.drain(http2.ErrCodeNo, []byte{})
				// Resetting the timer so that the clean-up doesn't deadlock.
				maxIdle.Reset(infinity)
				return
			}
			maxIdle.Reset(val)
		case <-maxAge.C:
			t.drain(http2.ErrCodeNo, []byte{})
			maxAge.Reset(t.kp.MaxConnectionAgeGrace)
			select {
			case <-maxAge.C:
				// Close the connection after grace period.
				t.Close()
				// Resetting the timer so that the clean-up doesn't deadlock.
				maxAge.Reset(infinity)
			case <-t.ctx.Done():
			}
			return
		case <-keepalive.C:
			if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
				pingSent = false
				keepalive.Reset(t.kp.Time)
				continue
			}
			if pingSent {
				t.Close()
				// Resetting the timer so that the clean-up doesn't deadlock.
				keepalive.Reset(infinity)
				return
			}
			pingSent = true
			if channelz.IsOn() {
				atomic.AddInt64(&t.czData.kpCount, 1)
			}
			t.controlBuf.put(p)
			keepalive.Reset(t.kp.Timeout)
		case <-t.ctx.Done():
			return
		}
	}
}
```


##  0x04    实现健壮的长连接客户端
官方提供了 keepalive 的实例：
-   [服务端](https://github.com/grpc/grpc-go/blob/master/examples/features/keepalive/server/main.go)
-   [客户端](https://github.com/grpc/grpc-go/blob/master/examples/features/keepalive/client/main.go)

##  0x05    参考
-   [GRPC 开箱手册](https://juejin.im/post/6844904096474857485)
