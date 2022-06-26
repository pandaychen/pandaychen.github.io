---
layout:     post
title:      GoIM 源码分析（五）：一些细节
subtitle:   GOIM
date:       2020-08-01
author:     pandaychen
catalog:    true
tags:
    - GOIM
---


##  0x00    前言

##  0x01    数据结构：Ring

####    Ring的使用场景
[Ring](https://github.com/Terry-Mao/goim/blob/master/internal/comet/ring.go#L11)本质上是一个环形缓冲区，其中保存的是空闲的`protocol.Proto`对象，主要用于长连接下TCP数据的拆包封包（每个连接都会初始化自己的`Ring`结构）

为什么要设计如此的结构呢？这里有个[issue](https://github.com/Terry-Mao/goim/issues/109)，思考下，长连接的场景，连接上不断有报文收发，使用一个可复用协议包体的数据结构是一个不错的优化手段（避免频繁内存申请释放，此外还可以控制总大小）。

如下面的结构，一个`Channel`代表一个TCP连接：
```golang
// Channel used by message pusher send msg to write goroutine.
type Channel struct {
        Room     *Room
        CliProto Ring
        signal   chan *protocol.Proto   //
        Writer   bufio.Writer
        Reader   bufio.Reader
        Next     *Channel
        Prev     *Channel
        mutex    sync.RWMutex
        //...
}

####    Ring的结构

// Ring ring proto buffer.
type Ring struct {
	// read
	rp   uint64 // read位置
	num  uint64 // 总长度
	mask uint64 // 掩码：用于%num的效果
	// TODO split cacheline, many cpu cache line size is 64
	// pad [40]byte
	// write
	wp   uint64  // write位置
	data []protocol.Proto   // 对象数组
}
```

```golang
//初始化对象数组长度为2的n次方
func (r *Ring) init(num uint64) {
	// 2^N
	if num&(num-1) != 0 {
		for num&(num-1) != 0 {
			num &= num - 1
		}
		num <<= 1
	}
	r.data = make([]protocol.Proto, num)
	r.num = num
	r.mask = r.num - 1
}

// 读写入的对象内容
func (r *Ring) Get() (proto *protocol.Proto, err error) {
	if r.rp == r.wp {
		return nil, errors.ErrRingEmpty
	}
	proto = &r.data[r.rp&r.mask]
	return
}

// 读游标+1
func (r *Ring) GetAdv() {
	r.rp++
	if conf.Conf.Debug {
		log.Infof("ring rp: %d, idx: %d", r.rp, r.rp&r.mask)
	}
}

//  获取待写入对象
func (r *Ring) Set() (proto *protocol.Proto, err error) {
	if r.wp-r.rp >= r.num {
		return nil, errors.ErrRingFull
	}
	proto = &r.data[r.wp&r.mask]
	return
}

//写入游标+1
func (r *Ring) SetAdv() {
	r.wp++
	if conf.Conf.Debug {
		log.Infof("ring wp: %d, idx: %d", r.wp, r.wp&r.mask)
	}
}

// 重置
func (r *Ring) Reset() {
	r.rp = 0
	r.wp = 0
	// prevent pad compiler optimization
	// r.pad = [40]byte{}
}
```

####    Ring在项目中的应用

##  0x03    参考
-   [goim源码剖析](https://laohanlinux.github.io/2016/12/22/goim%E6%BA%90%E7%A0%81%E5%89%96%E6%9E%90/)