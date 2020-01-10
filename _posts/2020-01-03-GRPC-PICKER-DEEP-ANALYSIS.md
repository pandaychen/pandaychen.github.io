---
layout:     post
title:      gRPC源码分析之Picker篇
subtitle:   gRPC客户端选择器实现分析
date:       2020-01-03
author:     pandaychen
catalog:    true
tags:
    - gRPC负载均衡
    - Picker
---

##	0x00	再看RR-Picker实现
&emsp;&emsp;前文中分析了官方提供的轮询 `Picker` 代码，我们可以使用 gRPC 提供的 `balancer` 包中的接口实现自定义的选择器 `Picker`，也就是自定义的负载均衡逻辑，只需要三步即可。这篇文章，讨论下，我们自己实现的`Picker`逻辑是如何gRPC中生效的。

###	Picker实现
一个简单的实现如下所示：
1.	第一步：设定全局`balancer`的名字和创建全局变量（package级别）

```go
var _ base.PickerBuilder = &roundRobinPickerBuilder{}		//创建全局变量(package级别)
var _ balancer.Picker = &roundRobinPicker{}					//创建全局变量(package级别)

const RoundRobin = "round_robin"		//注册到resolver中的lb全局名字

// newRoundRobinBuilder creates a new roundrobin balancer builder.
func newRoundRobinBuilder() balancer.Builder {
	return base.NewBalancerBuilderWithConfig(RoundRobin, &roundRobinPickerBuilder{}, base.Config{HealthCheck: true})
}

func init() {
	balancer.Register(newRoundRobinBuilder())
}
```

2.	第二步：定义`PickerBuilder`的实例化结构，并实现接口中定义的`Build`方法，该方法返回一个`balancer.Picker`

```go
type roundRobinPickerBuilder struct{}

func (*roundRobinPickerBuilder) Build(readySCs map[resolver.Address]balancer.SubConn) balancer.Picker {
	grpclog.Infof("roundrobinPicker: newPicker called with readySCs: %v", readySCs)

	if len(readySCs) == 0 {
		return base.NewErrPicker(balancer.ErrNoSubConnAvailable)
	}
	var scs []balancer.SubConn
	for addr, sc := range readySCs {
		weight := 1
		if addr.Metadata != nil {
			if m, ok := addr.Metadata.(*map[string]string); ok {
				w, ok := (*m)["weight"]
				if ok {
					n, err := strconv.Atoi(w)
					if err == nil && n > 0 {
						weight = n
					}
				}
			}
		}
		for i := 0; i < weight; i++ {
			scs = append(scs, sc)
		}
	}

	return &roundRobinPicker{
		subConns: scs,
		next:     rand.Intn(len(scs)),
	}
}
```

3.	第三步： 构建`roundRobinPicker`，该结构需要实现`balancer.Picker`定义的`Pick`方法
```go
type roundRobinPicker struct {
	subConns []balancer.SubConn
	mu       sync.Mutex
	next     int
}

func (p *roundRobinPicker) Pick(ctx context.Context, opts balancer.PickOptions) (balancer.SubConn, func(balancer.DoneInfo), error) {
	p.mu.Lock()
	sc := p.subConns[p.next]
	p.next = (p.next + 1) % len(p.subConns)
	p.mu.Unlock()
	return sc, nil, nil
}
```

##	0x01	balancer.Picker 与 base.PickerBuilder
上面两个结构，`roundRobinPickerBuilder`对应了`base.PickerBuilder`，`roundRobinPicker`对应了`balancer.Picker`，下面就分析这两个结构。

###	balancer.Picker
&emsp;&emsp;[`balancer.Picker`](https://github.com/grpc/grpc-go/blob/master/balancer/balancer.go)定义如下，从描述看其即将会被 `V2Picker` 取代，这里我们先只看 `Picker`。`Picker` 也是一个`interface{}`，封装它必须要实现 `Pick` 方法，该方法是从给定的连接池中，选取一个可用连接并返回。
```go
// Picker is used by gRPC to pick a SubConn to send an RPC.
// Balancer is expected to generate a new picker from its snapshot every time its
// internal state has changed.
//
// The pickers used by gRPC can be updated by ClientConn.UpdateBalancerState().
//
// Deprecated: use V2Picker instead		
type Picker interface {
	// Pick returns the SubConn to be used to send the RPC.
	// The returned SubConn must be one returned by NewSubConn().
	//
	// This functions is expected to return:
	// - a SubConn that is known to be READY;
	// - ErrNoSubConnAvailable if no SubConn is available, but progress is being
	//   made (for example, some SubConn is in CONNECTING mode);
	// - other errors if no active connecting is happening (for example, all SubConn
	//   are in TRANSIENT_FAILURE mode).
	//
	// If a SubConn is returned:
	// - If it is READY, gRPC will send the RPC on it;
	// - If it is not ready, or becomes not ready after it's returned, gRPC will
	//   block until UpdateBalancerState() is called and will call pick on the
	//   new picker. The done function returned from Pick(), if not nil, will be
	//   called with nil error, no bytes sent and no bytes received.
	//
	// If the returned error is not nil:
	// - If the error is ErrNoSubConnAvailable, gRPC will block until UpdateBalancerState()
	// - If the error is ErrTransientFailure or implements IsTransientFailure()
	//   bool, returning true:
	//   - If the RPC is wait-for-ready, gRPC will block until UpdateBalancerState()
	//     is called to pick again;
	//   - Otherwise, RPC will fail with unavailable error.
	// - Else (error is other non-nil error):
	//   - The RPC will fail with the error's status code, or Unknown if it is
	//     not a status error.
	//
	// The returned done() function will be called once the rpc has finished,
	// with the final status of that RPC.  If the SubConn returned is not a
	// valid SubConn type, done may not be called.  done may be nil if balancer
	// doesn't care about the RPC status.
	Pick(ctx context.Context, info PickInfo) (conn SubConn, done func(DoneInfo), err error)
}
```

###	base.PickerBuilder
[`base.PickerBuilder`](https://github.com/grpc/grpc-go/blob/master/balancer/base/base.go)定义如下，其定义了一个`Build`方法，该方法需要返回`balancer.Picker`。
```go
// PickerBuilder creates balancer.Picker.
type PickerBuilder interface {
	// Build takes a slice of ready SubConns, and returns a picker that will be
	// used by gRPC to pick a SubConn.
	Build(readySCs map[resolver.Address]balancer.SubConn) balancer.Picker
}

// V2PickerBuilder creates balancer.V2Picker.
type V2PickerBuilder interface {
	// Build returns a picker that will be used by gRPC to pick a SubConn.
	Build(info PickerBuildInfo) balancer.V2Picker
}
```

当我们创建了自定义的`PickerBuilder`后，通过`base.NewBalancerBuilderWithConfig`方法，注册我们实现的`Picker`逻辑
```go
// NewBalancerBuilderWithConfig returns a base balancer builder configured by the provided config.
func NewBalancerBuilderWithConfig(name string, pb PickerBuilder, config Config) balancer.Builder {
	return &baseBuilder{
		name:          name,
		pickerBuilder: pb,		//这里保存了我们自定义了roundRobinPickerBuilder
		config:        config,
	}
}
```

在包的`init`方法中，使用[`balancer.Register`方法](https://github.com/grpc/grpc-go/blob/master/balancer/balancer.go)注册我们的`Picker`
```go
var (
	// m is a map from name to balancer builder.
	m = make(map[string]Builder)		//全局balancer-Map
)

// Register registers the balancer builder to the balancer map. b.Name
// (lowercased) will be used as the name registered with this builder.  If the
// Builder implements ConfigParser, ParseConfig will be called when new service
// configs are received by the resolver, and the result will be provided to the
// Balancer in UpdateClientConnState.
//
// NOTE: this function must only be called during initialization time (i.e. in
// an init() function), and is not thread-safe. If multiple Balancers are
// registered with the same name, the one registered last will take effect.
func Register(b Builder) {
	m[strings.ToLower(b.Name())] = b
}
```

##	0x02	应用自定义Picker
这节，我们从`NewBalancerBuilderWithConfig`方法开始，看下自定义的Picker是如何生效的
```go
// NewBalancerBuilderWithConfig returns a base balancer builder configured by the provided config.
func NewBalancerBuilderWithConfig(name string, pb PickerBuilder, config Config) balancer.Builder {
	return &baseBuilder{			//返回baseBuilder
		name:          name,
		pickerBuilder: pb,			//这里保存了我们自定义了roundRobinPickerBuilder
		config:        config,
	}
}
```

在`baseBuilder`的[`Build`方法](https://github.com/grpc/grpc-go/blob/master/balancer/base/balancer.go#L38)中，初始化了重要的`balancer.Balancer`。
```go
func (bb *baseBuilder) Build(cc balancer.ClientConn, opt balancer.BuildOptions) balancer.Balancer {
	bal := &baseBalancer{
		cc:              cc,
		pickerBuilder:   bb.pickerBuilder,		//这里是我们自定义的roundRobinPickerBuilder
		v2PickerBuilder: bb.v2PickerBuilder,

		subConns: make(map[resolver.Address]balancer.SubConn),
		scStates: make(map[balancer.SubConn]connectivity.State),
		csEvltr:  &balancer.ConnectivityStateEvaluator{},
		config:   bb.config,
	}

	if bb.pickerBuilder != nil {
		bal.picker = NewErrPicker(balancer.ErrNoSubConnAvailable)
	} else {
		bal.v2Picker = NewErrPickerV2(balancer.ErrNoSubConnAvailable)
	}
	return bal
}
```

在`balancer.Balancer`这个interface{}中，`UpdateSubConnState` 和`HandleSubConnStateChange`功能是一样的，我们继续分析下`UpdateSubConnState`和``这两个方法在哪里被调用的
```go

// Balancer takes input from gRPC, manages SubConns, and collects and aggregates
// the connectivity states.
//
// It also generates and updates the Picker used by gRPC to pick SubConns for RPCs.
//
// HandleSubConnectionStateChange, HandleResolvedAddrs and Close are guaranteed
// to be called synchronously from the same goroutine.
// There's no guarantee on picker.Pick, it may be called anytime.
type Balancer interface {
	// HandleSubConnStateChange is called by gRPC when the connectivity state
	// of sc has changed.
	// Balancer is expected to aggregate all the state of SubConn and report
	// that back to gRPC.
	// Balancer should also generate and update Pickers when its internal state has
	// been changed by the new state.
	//
	// Deprecated: if V2Balancer is implemented by the Balancer,
	// UpdateSubConnState will be called instead.
	HandleSubConnStateChange(sc SubConn, state connectivity.State)
	// HandleResolvedAddrs is called by gRPC to send updated resolved addresses to
	// balancers.
	// Balancer can create new SubConn or remove SubConn with the addresses.
	// An empty address slice and a non-nil error will be passed if the resolver returns
	// non-nil error to gRPC.
	//
	// Deprecated: if V2Balancer is implemented by the Balancer,
	// UpdateClientConnState will be called instead.
	HandleResolvedAddrs([]resolver.Address, error)
	// Close closes the balancer. The balancer is not required to call
	// ClientConn.RemoveSubConn for its existing SubConns.
	Close()
}
```
###	UpdateSubConnState的实现
(https://github.com/grpc/grpc-go/blob/master/balancer/base/balancer.go#L178)，其中有个很重要的方法`regeneratePicker`，在发生下面的情况时，需要重建Picker，这个很好理解，注意看下`func (*roundRobinPickerBuilder) Build(readySCs map[resolver.Address]balancer.SubConn) balancer.Picker` 的`readySCs`参数，这个参数表示当前可用的连接，如果连接发生问题，当然需要重建连接池
```go
   // 当下面情况发生时，需要重新创建Picker：
   //    - 连接由其他状态转变为Ready状态
   //    - 连接由Ready状态转变为其他状态
   //    - balancer转变为TransientFailure状态
   //    - balancer由TransientFailure转变为Non-TransientFailure状态
```

```go
func (b *baseBalancer) UpdateSubConnState(sc balancer.SubConn, state balancer.SubConnState) {
	s := state.ConnectivityState
	if grpclog.V(2) {
		grpclog.Infof("base.baseBalancer: handle SubConn state change: %p, %v", sc, s)
	}
	oldS, ok := b.scStates[sc]
	if !ok {
		if grpclog.V(2) {
			grpclog.Infof("base.baseBalancer: got state changes for an unknown SubConn: %p, %v", sc, s)
		}
		return
	}
	b.scStates[sc] = s
	switch s {
	case connectivity.Idle:
		sc.Connect()
	case connectivity.Shutdown:
		// When an address was removed by resolver, b called RemoveSubConn but
		// kept the sc's state in scStates. Remove state for this sc here.
		delete(b.scStates, sc)
	}

	oldAggrState := b.state
	b.state = b.csEvltr.RecordTransition(oldS, s)

	// Regenerate picker when one of the following happens:
	//  - this sc became ready from not-ready
	//  - this sc became not-ready from ready
	//  - the aggregated state of balancer became TransientFailure from non-TransientFailure
	//  - the aggregated state of balancer became non-TransientFailure from TransientFailure
	if (s == connectivity.Ready) != (oldS == connectivity.Ready) ||
		(b.state == connectivity.TransientFailure) != (oldAggrState == connectivity.TransientFailure) {
		//重新生成Picker
		b.regeneratePicker(state.ConnectionError)
	}

	if b.picker != nil {
		//将自定义的picker传入UpdateBalancerState，b.picker是我们自己实现的lb逻辑
		b.cc.UpdateBalancerState(b.state, b.picker)
	} else {
		b.cc.UpdateState(balancer.State{ConnectivityState: b.state, Picker: b.v2Picker})
	}
}
```

`regeneratePicker`的实现如下，注意看`if b.pickerBuilder != nil {...b.picker = b.pickerBuilder.Build(readySCs)...}`这里，就解答了文章的第一个问题，在这里调用了我们自定义的`PickerBuilder`方法。注意：这里的`readySCs`可以理解为当前健康的连接池。
```go
// regeneratePicker takes a snapshot of the balancer, and generates a picker
// from it. The picker is
//  - errPicker with ErrTransientFailure if the balancer is in TransientFailure,
//  - built by the pickerBuilder with all READY SubConns otherwise.
func (b *baseBalancer) regeneratePicker(err error) {
	if b.state == connectivity.TransientFailure {
		if b.pickerBuilder != nil {
			b.picker = NewErrPicker(balancer.ErrTransientFailure)
		} else {
			if err != nil {
				b.v2Picker = NewErrPickerV2(balancer.TransientFailureError(err))
			} else {
				// This means the last subchannel transition was not to
				// TransientFailure (otherwise err must be set), but the
				// aggregate state of the balancer is TransientFailure, meaning
				// there are no other addresses.
				b.v2Picker = NewErrPickerV2(balancer.TransientFailureError(errors.New("resolver returned no addresses")))
			}
		}
		return
	}
	if b.pickerBuilder != nil {
		readySCs := make(map[resolver.Address]balancer.SubConn)

		// Filter out all ready SCs from full subConn map.
		for addr, sc := range b.subConns {
			if st, ok := b.scStates[sc]; ok && st == connectivity.Ready {
				//readySCs中保存了connectivity.Ready（连接就绪）的连接集合
				readySCs[addr] = sc
			}
		}
		b.picker = b.pickerBuilder.Build(readySCs)	//重要：执行自定义的PickerBuilder，文中的roundRobinPickerBuilder方法
	} else {
		readySCs := make(map[balancer.SubConn]SubConnInfo)

		// Filter out all ready SCs from full subConn map.
		for addr, sc := range b.subConns {
			if st, ok := b.scStates[sc]; ok && st == connectivity.Ready {
				readySCs[sc] = SubConnInfo{Address: addr}
			}
		}
		b.v2Picker = b.v2PickerBuilder.Build(PickerBuildInfo{ReadySCs: readySCs})//重要：执行自定义的PickerBuilder，文中的roundRobinPickerBuilder方法
	}
}
```

##	0x03	自定义Picker的调用（1）
最后一个问题，在哪里返回自定义Picker的结果呢？https://github.com/grpc/grpc-go/blob/master/picker_wrapper.go，注册Pciker
// updatePicker is called by UpdateBalancerState. It unblocks all blocked pick.
func (pw *pickerWrapper) updatePicker(p balancer.Picker) {
	pw.updatePickerV2(&v2PickerWrapper{picker: p, connErr: pw.connErr})
}

```go
// v2PickerWrapper wraps a balancer.Picker while providing the
// balancer.V2Picker API.  It requires a pickerWrapper to generate errors
// including the latest connectionError.  To be deleted when balancer.Picker is
// updated to the balancer.V2Picker API.
type v2PickerWrapper struct {
	picker  balancer.Picker
	connErr *connErr
}
```

在v2PickerWrapper的实现中，解答了这个问题。
```go
func (v *v2PickerWrapper) Pick(info balancer.PickInfo) (balancer.PickResult, error) {
	sc, done, err := v.picker.Pick(info.Ctx, info)		//调用我们自定义的Picker，roundRobinPicker，返回一个可用的连接sc给调用方
	if err != nil {
		if err == balancer.ErrTransientFailure {
			return balancer.PickResult{}, balancer.TransientFailureError(fmt.Errorf("%v, latest connection error: %v", err, v.connErr.connectionError()))
		}
		return balancer.PickResult{}, err
	}
	return balancer.PickResult{SubConn: sc, Done: done}, nil
}
```

```go
// pick returns the transport that will be used for the RPC.
// It may block in the following cases:
// - there's no picker
// - the current picker returns ErrNoSubConnAvailable
// - the current picker returns other errors and failfast is false.
// - the subConn returned by the current picker is not READY
// When one of these situations happens, pick blocks until the picker gets updated.
func (pw *pickerWrapper) pick(ctx context.Context, failfast bool, info balancer.PickInfo) (transport.ClientTransport, func(balancer.DoneInfo), error) {
	var ch chan struct{}

	var lastPickErr error
	for {
		pw.mu.Lock()
		if pw.done {
			pw.mu.Unlock()
			return nil, nil, ErrClientConnClosing
		}

		if pw.picker == nil {
			ch = pw.blockingCh
		}
		if ch == pw.blockingCh {
			// This could happen when either:
			// - pw.picker is nil (the previous if condition), or
			// - has called pick on the current picker.
			pw.mu.Unlock()
			select {
			case <-ctx.Done():
				var errStr string
				if lastPickErr != nil {
					errStr = "latest balancer error: " + lastPickErr.Error()
				} else if connectionErr := pw.connectionError(); connectionErr != nil {
					errStr = "latest connection error: " + connectionErr.Error()
				} else {
					errStr = ctx.Err().Error()
				}
				switch ctx.Err() {
				case context.DeadlineExceeded:
					return nil, nil, status.Error(codes.DeadlineExceeded, errStr)
				case context.Canceled:
					return nil, nil, status.Error(codes.Canceled, errStr)
				}
			case <-ch:
			}
			continue
		}

		ch = pw.blockingCh
		p := pw.picker
		pw.mu.Unlock()

		pickResult, err := p.Pick(info)

		if err != nil {
			if err == balancer.ErrNoSubConnAvailable {
				continue
			}
			if tfe, ok := err.(interface{ IsTransientFailure() bool }); ok && tfe.IsTransientFailure() {
				if !failfast {
					lastPickErr = err
					continue
				}
				return nil, nil, status.Error(codes.Unavailable, err.Error())
			}
			if _, ok := status.FromError(err); ok {
				return nil, nil, err
			}
			// err is some other error.
			return nil, nil, status.Error(codes.Unknown, err.Error())
		}

		acw, ok := pickResult.SubConn.(*acBalancerWrapper)
		if !ok {
			grpclog.Error("subconn returned from pick is not *acBalancerWrapper")
			continue
		}
		if t, ok := acw.getAddrConn().getReadyTransport(); ok {
			if channelz.IsOn() {
				return t, doneChannelzWrapper(acw, pickResult.Done), nil
			}
			return t, pickResult.Done, nil
		}
		if pickResult.Done != nil {
			// Calling done with nil error, no bytes sent and no bytes received.
			// DoneInfo with default value works.
			pickResult.Done(balancer.DoneInfo{})
		}
		grpclog.Infof("blockingPicker: the picked transport is not ready, loop back to repick")
		// If ok == false, ac.state is not READY.
		// A valid picker always returns READY subConn. This means the state of ac
		// just changed, and picker will be updated shortly.
		// continue back to the beginning of the for loop to repick.
	}
}
```

##	0x04	自定义Picker的调用（2）
在哪里返回Picker的结果给上层gRPC客户端呢？https://github.com/grpc/grpc-go/blob/master/clientconn.go#L881，看到下面的picker没
```GO
func (cc *ClientConn) getTransport(ctx context.Context, failfast bool, method string) (transport.ClientTransport, func(balancer.DoneInfo), error) {
	t, done, err := cc.blockingpicker.pick(ctx, failfast, balancer.PickInfo{	//注意这里调用的是pick方法，自定义的Picker在picker中
		Ctx:            ctx,
		FullMethodName: method,
	})
	if err != nil {
		return nil, nil, toRPCErr(err)
	}
	return t, done, nil
}
```

在 newClientStream 方法中，我们通过 getTransport 方法获取了 Transport 层中抽象出来的 ClientTransport 和 ServerTransport，实际上就是获取一个连接给后续 RPC 调用传输使用。到此，gRPC的客户端就获取了由自定义LoadBalancer算法得到的最终的`TCP`连接

##	0x06	总结
&emsp;&emsp;本文分析了gRPC是如何将自定义的Picker实现应用在最终的流程。

##	0X04	参考
-	[grpc-client端分析](https://mcll.top/2019/07/29/grpc-client%E7%AB%AF%E5%88%86%E6%9E%901/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
