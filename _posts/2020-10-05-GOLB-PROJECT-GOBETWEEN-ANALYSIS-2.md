---
layout: post
title: 基于 Golang 实现的负载均衡网关：gobetween 分析（二）
subtitle: Metrics 采集技巧
date: 2020-10-05
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - Gateway
  - 网关
---

## 0x00 前言

本文分析下 gobetween 的 metrics 采集方式。

## 0x01 ReadWriteCount 采集

以 `ReadWriteCount` 采集为例，在 `proxy.go` 中，封装了类型 `io.Copy`[方法](https://github.com/yyyar/gobetween/blob/master/src/server/tcp/proxy.go#L106)，将入流量 `readN`、出流量 `writeN` 通过 channel 向上层发送（需要考虑下性能问题？）

```golang
func Copy(to io.Writer, from io.Reader, ch chan<- core.ReadWriteCount) error {

	buf := make([]byte, BUFFER_SIZE)
	var err error = nil

	for {
		readN, readErr := from.Read(buf)

		if readN > 0 {

			writeN, writeErr := to.Write(buf[0:readN])

			if writeN > 0 {
                // 入流量，出流量
				ch <- core.ReadWriteCount{CountRead: uint(readN), CountWrite: uint(writeN)}
			}

			if writeErr != nil {
				err = writeErr
				break
			}

			if readN != writeN {
				err = io.ErrShortWrite
				break
			}
		}

		if readErr == io.EOF {
			break
		}

		if readErr != nil {
			err = readErr
			break
		}
	}

	return err
}
```

上述的 channel 接收方在 `proxy`[方法](https://github.com/yyyar/gobetween/blob/master/src/server/tcp/proxy.go#L31) 中，该方法返回`outStats := make(chan core.ReadWriteCount)`给调用方，用于接受出入流量的 metrics 数据：

```golang
func proxy(to net.Conn, from net.Conn, timeout time.Duration) <-chan core.ReadWriteCount {

	log := logging.For("proxy")

	stats := make(chan core.ReadWriteCount)
	outStats := make(chan core.ReadWriteCount)

	rwcBuffer := core.ReadWriteCount{}
	ticker := time.NewTicker(PROXY_STATS_PUSH_INTERVAL)
	flushed := false

	// Stats collecting goroutine
	go func() {

		if timeout > 0 {
			from.SetReadDeadline(time.Now().Add(timeout))
		}

		for {
			select {
			case <-ticker.C:
                // 每隔一段时间，上报一次 rwcBuffer 的数据，并且重置 flushed 的状态
				if !rwcBuffer.IsZero() {
					outStats <- rwcBuffer
				}
				flushed = true
            // 接收方
			case rwc, ok := <-stats:

				if !ok {
                    //channel 被关闭
					ticker.Stop()
					if !flushed && !rwcBuffer.IsZero() {
						outStats <- rwcBuffer
					}
					close(outStats)
					return
				}

				if timeout > 0 && rwc.CountRead > 0 {
                    // 保活计时器更新
					from.SetReadDeadline(time.Now().Add(timeout))
				}

				// Remove non blocking
                // 视 flushed 进行累加或者初始化（实现按照时间间隔上报读数）
				if flushed {
					rwcBuffer = rwc
				} else {
					rwcBuffer.CountWrite += rwc.CountWrite
					rwcBuffer.CountRead += rwc.CountRead
				}

				flushed = false
			}
		}
	}()

	// Run proxy copier
	go func() {
		err := Copy(to, from, stats)
		// hack to determine normal close. TODO: fix when it will be exposed in golang
		e, ok := err.(*net.OpError)
		if err != nil && (!ok || e.Err.Error() != "use of closed network connection") {
			log.Warn(err)
		}

		to.Close()
		from.Close()

		// Stop stats collecting goroutine
		close(stats)
	}()

	return outStats
}
```

调用`proxy`方法位于`server.go`代理实现的[核心逻辑](https://github.com/yyyar/gobetween/blob/master/src/server/tcp/server.go#L353)中：

```golang
func (this *Server) handle(ctx *core.TcpContext) {
    //......
	/* ----- Stat proxying ----- */

	log.Debug("Begin ", clientConn.RemoteAddr(), " -> ", this.listener.Addr(), " -> ", backendConn.RemoteAddr())
	cs := proxy(clientConn, backendConn, utils.ParseDurationOrDefault(*this.cfg.BackendIdleTimeout, 0))
	bs := proxy(backendConn, clientConn, utils.ParseDurationOrDefault(*this.cfg.ClientIdleTimeout, 0))

	isTx, isRx := true, true
	for isTx || isRx {
		select {
		case s, ok := <-cs:
			isRx = ok
			if !ok {
				cs = nil
				continue
			}
			this.scheduler.IncrementRx(*backend, s.CountWrite)
		case s, ok := <-bs:
			isTx = ok
			if !ok {
				bs = nil
				continue
			}
			this.scheduler.IncrementTx(*backend, s.CountWrite)
		}
	}

	log.Debug("End ", clientConn.RemoteAddr(), " -> ", this.listener.Addr(), " -> ", backendConn.RemoteAddr())
}
```

## 0x0 参考
