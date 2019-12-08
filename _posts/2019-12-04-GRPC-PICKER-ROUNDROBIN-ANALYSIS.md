---
layout:     post
title:      gRPC源码分析之官方Picker实现（Picker With RoundRobin）
subtitle:   gRPC客户端选择器分析
date:       2019-12-06
author:     pandaychen
header-img: 
catalog: true
tags:
    - gRPC
    - Loadbalance
---

##	0x00	前言
gRPC官方提供了基于[RoundRobin轮询算法](https://en.wikipedia.org/wiki/Round-robin_scheduling)的[Picker](https://github.com/grpc/grpc-go/blob/dc49de8acd511e4d47ad0cbf58bd77f4775c165f/balancer/roundrobin/roundrobin.go)实现。这篇文章简单分析下其源码，理解此过程，可以很轻易的实现自定义的负载均衡逻辑。前面文章已经介绍了Balancer和Picker的内部实现机制，本篇就在此基础上进行分析。官方给出的Picker接口实例化，整体逻辑比较直观，先贴下源码：
```go
package roundrobin

import (
	"sync"

	"google.golang.org/grpc/balancer"
	"google.golang.org/grpc/balancer/base"
	"google.golang.org/grpc/grpclog"
	"google.golang.org/grpc/internal/grpcrand"
)

// Name is the name of round_robin balancer.
const Name = "round_robin"

// newBuilder creates a new roundrobin balancer builder.
func newBuilder() balancer.Builder {
	return base.NewBalancerBuilderV2(Name, &rrPickerBuilder{}, base.Config{HealthCheck: true})
}

func init() {
	balancer.Register(newBuilder())
}

type rrPickerBuilder struct{}

func (*rrPickerBuilder) Build(info base.PickerBuildInfo) balancer.V2Picker {
	grpclog.Infof("roundrobinPicker: newPicker called with info: %v", info)
	if len(info.ReadySCs) == 0 {
		return base.NewErrPickerV2(balancer.ErrNoSubConnAvailable)
	}
	var scs []balancer.SubConn
	for sc := range info.ReadySCs {
		scs = append(scs, sc)
	}
	return &rrPicker{
		subConns: scs,
		// Start at a random index, as the same RR balancer rebuilds a new
		// picker when SubConn states change, and we don't want to apply excess
		// load to the first server in the list.
		next: grpcrand.Intn(len(scs)),
	}
}

type rrPicker struct {
	// subConns is the snapshot of the roundrobin balancer when this picker was
	// created. The slice is immutable. Each Get() will do a round robin
	// selection from it and return the selected SubConn.
	subConns []balancer.SubConn

	mu   sync.Mutex
	next int
}

func (p *rrPicker) Pick(balancer.PickInfo) (balancer.PickResult, error) {
	p.mu.Lock()
	sc := p.subConns[p.next]
	p.next = (p.next + 1) % len(p.subConns)
	p.mu.Unlock()
	return balancer.PickResult{SubConn: sc}, nil
}

```

## 0x01	初始化
首先，定义Pcker的名字和结构，`rrPickerBuilder`需要实现如何根据当前活跃的连接`info.ReadySCs`，生成初始化的ConnectionPool（可以看出gRPC提供了非常灵活的LB实现接口），`rrPicker`结构用来从ConnectionPool中，按照一定的策略来选择单个连接，给上层
```go
const Name = "round_robin"
type rrPickerBuilder struct{}

type rrPicker struct {
	// subConns is the snapshot of the roundrobin balancer when this picker was
	// created. The slice is immutable. Each Get() will do a round robin
	// selection from it and return the selected SubConn.
	subConns []balancer.SubConn

	mu   sync.Mutex
	next int
}
```
在`rrPicker`的`Pick`方法中，返回`balancer.PickResult`：
```go
return balancer.PickResult{SubConn: sc}
```

## 0x02	注册  Picker
使用`NewBalancerBuilderV2`来实现将我们实现的Picker逻辑嵌入（注册）到Balancer中，同时提供一个Picker的名字（关联对应的`balancer.Builder`），将其注册到Balancer的全局map中。

包初始化方法`init`：
```go
func init() {
	balancer.Register(newBuilder())
}

```
生成`balancer.Builder`的方法：
```go
func newBuilder() balancer.Builder {
	return base.NewBalancerBuilderV2(Name, &rrPickerBuilder{}, base.Config{HealthCheck: true})
}
```
`NewBalancerBuilderV2`方法，返回`balancer.Builder`：
```go
// NewBalancerBuilderV2 returns a base balancer builder configured by the provided config.
func NewBalancerBuilderV2(name string, pb V2PickerBuilder, config Config) balancer.Builder {
	return &baseBuilder{
		name:            name,
		v2PickerBuilder: pb,
		config:          config,
	}
}
```

看下[`baseBuilder`](https://github.com/grpc/grpc-go/blob/dc49de8acd511e4d47ad0cbf58bd77f4775c165f/balancer/base/balancer.go)的结构：
```go
type baseBuilder struct {
	name            string
	pickerBuilder   PickerBuilder
	v2PickerBuilder V2PickerBuilder
	config          Config
}
```

当然了，按照`balancer.Builder`的要求，需要实现`Build`和`Name`方法：
```go
func (bb *baseBuilder) Build(cc balancer.ClientConn, opt balancer.BuildOptions) balancer.Balancer {
	bal := &baseBalancer{
		cc:              cc,
		pickerBuilder:   bb.pickerBuilder,
		v2PickerBuilder: bb.v2PickerBuilder,

		subConns: make(map[resolver.Address]balancer.SubConn),
		scStates: make(map[balancer.SubConn]connectivity.State),
		csEvltr:  &balancer.ConnectivityStateEvaluator{},
		config:   bb.config,
	}
	// Initialize picker to a picker that always returns
	// ErrNoSubConnAvailable, because when state of a SubConn changes, we
	// may call UpdateState with this picker.
	if bb.pickerBuilder != nil {
		bal.picker = NewErrPicker(balancer.ErrNoSubConnAvailable)
	} else {
		bal.v2Picker = NewErrPickerV2(balancer.ErrNoSubConnAvailable)
	}
	return bal
}

func (bb *baseBuilder) Name() string {
	return bb.name
}
```


## 0x03 生成balancer.V2Picker



##	0x04 构建负载均衡选择器rrPicker
这里选择官方的`V2Picker`来作为Picker，只需要实现`Pick`方法就ok：
```go
type V2Picker interface {
	// Pick returns the connection to use for this RPC and related information.
	//
	// Pick should not block.  If the balancer needs to do I/O or any blocking
	// or time-consuming work to service this call, it should return
	// ErrNoSubConnAvailable, and the Pick call will be repeated by gRPC when
	// the Picker is updated (using ClientConn.UpdateState).
	//
	// If an error is returned:
	//
	// - If the error is ErrNoSubConnAvailable, gRPC will block until a new
	//   Picker is provided by the balancer (using ClientConn.UpdateState).
	//
	// - If the error implements IsTransientFailure() bool, returning true,
	//   wait for ready RPCs will wait, but non-wait for ready RPCs will be
	//   terminated with this error's Error() string and status code
	//   Unavailable.
	//
	// - Any other errors terminate all RPCs with the code and message
	//   provided.  If the error is not a status error, it will be converted by
	//   gRPC to a status error with code Unknown.
	Pick(info PickInfo) (PickResult, error)
}
```

此步骤为最后一步，就是构建负载均衡算法的实现，最终只需要返回`balancer.PickResult`给调用方，就大功告成了。看下RR算法的实现代码：
```go
type rrPicker struct {
	// subConns is the snapshot of the roundrobin balancer when this picker was
	// created. The slice is immutable. Each Get() will do a round robin
	// selection from it and return the selected SubConn.
	subConns []balancer.SubConn		//保存了Conntion Pool（可以有重复长连接）

	mu   sync.Mutex					//一般需要加锁访问
	next int						//rr算法需要一个全局的index
}
```
实现`V2Picker`的`Pick`方法：
```go
func (p *rrPicker) Pick(balancer.PickInfo) (balancer.PickResult, error) {
	p.mu.Lock()
	sc := p.subConns[p.next]		//选择一个活跃的连接
	p.next = (p.next + 1) % len(p.subConns)		//更新全局index
	p.mu.Unlock()
	return balancer.PickResult{SubConn: sc}, nil		//返回结果
}
```

其中，`PickResult`的结构如下：
```go
// PickResult contains information related to a connection chosen for an RPC.
type PickResult struct {
	// SubConn is the connection to use for this pick, if its state is Ready.
	// If the state is not Ready, gRPC will block the RPC until a new Picker is
	// provided by the balancer (using ClientConn.UpdateState).  The SubConn
	// must be one returned by ClientConn.NewSubConn.
	SubConn SubConn

	// Done is called when the RPC is completed.  If the SubConn is not ready,
	// this will be called with a nil parameter.  If the SubConn is not a valid
	// type, Done may not be called.  May be nil if the balancer does not wish
	// to be notified when the RPC completes.
	Done func(DoneInfo)
}
```

##	0x05	总结
可以看出，gRPC的`Picker`结构实现，还是非常友好的。只要理解了代码的流程，很容易的可以写出自己的负载均衡实现逻辑，下一篇文章，再聊聊目前比较流行的负载均衡算法，如待权重的rr、P2C、随机、一致性hash、会话保持等实现。