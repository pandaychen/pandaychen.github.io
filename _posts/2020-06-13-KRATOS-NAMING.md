---
layout:     post
title:      Kratos 源码分析：Naming 解析（上）
subtitle:   分析 Naming 的多消费者订阅 - Watcher 模式
date:       2020-06-13
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
分析下 Kratos 框架中的服务注册与发现的代码。准备分为两篇文章，一篇介绍 Naming 的公共接口及基于 Etcd 的 naming 机制实现，另一篇文章介绍 Warden 库是如何使用 Naming 的接口来完成服务注册与发现机制。总体而言，Kratos 的 Naming 逻辑分为如下几块：
-	公共的 `Naming` 逻辑与接口封装
	-	`Naming` 的接口实例化：如 `ETCD`、`Zookeeper` 的实现
-	Warden 库基于 gRPC 接口实现的 `Resovler` 逻辑封装，调用 `Naming` 的提供接口完成，将从 `Naming` 获取的结果通过 gRPC 的接口通知上层
-	调用方（Server 端代码和 Client 端代码）决定使用哪个 `Naming` 的实例化作为本身的服务注册和发现实现


这样，对于不同的框架，模块的内聚性更强。换句话说，对于 gRPC，它无需关注服务发现及注册的细节，以及使用何种服务治理的中介，只需要调用相关接口，获取结果并调用 gRPC 的用户接口更新即可。

##  0x01   Naming 包的公共接口
Kratos 的 [Naming 模块](https://github.com/go-kratos/kratos/blob/master/pkg/naming/naming.go)，是装饰器模式（Decorator）的典型实现。它封装了三个通用的方法：
`naming.Resolver`、`naming.Registry` 及 `naming.Builder`:

-   `Resolver`，解析器的实现，包含三个方法：
    -   `Fetch`：
    -   `Watch`：
    -   `Close`：

-   `Registry`，注册器的实现，包含两个方法：
    -   `Register`：
    -   `Close`：

-   `Builder`：该方法生成（构造）一个 Resolver，包含两个方法：
    -   `Build`：
    -   `Scheme`：

`Naming.Resolver` 的定义：
```golang
/*
对应于 watcher 的实现逻辑
1.Fetch--- 从注册中心获取列表
2.Watch--- 监听改变，通知上层逻辑
3.Close-- 关闭 watcher
*/
type Resolver interface {
	Fetch(context.Context) (*InstancesInfo, bool)
	Watch() <-chan struct{}
	Close() error
}
```

`Naming.Registry` 的定义：
```golang
/*
	对应于 register 的实现逻辑
	1. 注册
	2. 解除注册
*/
type Registry interface {
	Register(ctx context.Context, ins *Instance) (cancel context.CancelFunc, err error)
	Close() error
}
```

`Naming.Builder` 的定义，和 gRPC 的 [`resolver.Builder`](https://godoc.org/google.golang.org/grpc/resolver#Builder) 的定义一样，可以互转：
```golang
/*
对应于 resolver 的实现逻辑（grpc 接口）
1. Build--- 构造一个 resolver
2. Scheme -- 返回 resolver 的名字
*/
type Builder interface {
	Build(id string, options ...BuildOpt) Resolver
	Scheme() string
}
```

那么下面的工作就是对上面这几个方法的实例化（Decorator 设计模式），Kratos 中提供了 [Discovery](https://github.com/go-kratos/kratos/blob/master/pkg/naming/discovery/discovery.go)、[Etcd](https://github.com/go-kratos/kratos/blob/master/pkg/naming/etcd/etcd.go) 及 [Zookeeper](https://github.com/go-kratos/kratos/blob/master/pkg/naming/zookeeper/zookeeper.go) 的三种实现。本文简单分析下 Etcd 的相关接口实现及应用。

##  0x02    多消费者订阅 - Watcher 模型
使用过 Etcd 的同学们应该知道，Etcd 是非常典型的 CP 系统，客户端可以通过 Watcher 机制来实现对 1 个（或多个）Key 前缀节点的动态监听，所有发生的改变客户端都可以获取到。所以，Kratos 基于 Etcd 的 Naming 实现，是非常的经典的多消费者订阅 - Watcher 模型，大概思路如下：
![image](https://wx1.sbimg.cn/2020/06/12/4ccdbe50e68dd4e5269b30016737d876.png)

##  0x03    Etcd 的 Naming 实现
基于 Etcd 的封装 [代码在此](https://github.com/go-kratos/kratos/blob/master/pkg/naming/etcd/etcd.go)。本小节来分析下其封装实现。

####    核心结构
`etcd.EtcdBuilder` 实现了 `Naming.Registry` 及 `Naming.Builder` 的 `interface{}` 定义的方法，而 `etcd.Resolve` 则实现了 `naming.Resolver` 定义的方法：
注意：`EtcdBuilder` 包含了 map 型的成员 `apps     map[string]*appInfo`，意为 `EtcdBuilder` 可以实现对多个 key 的监听，key 对应的结果保存在 `appInfo` 中
```golang
// EtcdBuilder is a etcd clientv3 EtcdBuilder
type EtcdBuilder struct {
	cli        *clientv3.Client
	ctx        context.Context
	cancelFunc context.CancelFunc

	mutex    sync.RWMutex
	apps     map[string]*appInfo
	registry map[string]struct{}
}
```

而 appInfo 的结构如下，注意 `ins	atomic.Value`

```golang
type appInfo struct {
	resolver map[*Resolve]struct{}
	ins      atomic.Value
	e        *EtcdBuilder
	once     sync.Once
}
```
`etcd.Resolve` 结构如下：
```golang
// Resolve etch resolver.
type Resolve struct {
	id    string
	event chan struct{}
	e     *EtcdBuilder
	opt   *naming.BuildOptions
}
```

####    初始化
初始化的部分比较简单，就是构造一个 `EtcdBuilder` 对象，里面包含 etcdv3 的客户端及 `apps` 和 `registry` 两个 `map`：
```golang
// New is new a etcdbuilder
func New(c *clientv3.Config) (e *EtcdBuilder, err error) {
	......
	ctx, cancel := context.WithCancel(context.Background())
	e = &EtcdBuilder{
		cli:        cli,
		ctx:        ctx,			// 创建根 ctx，用于取消子协程
		cancelFunc: cancel,
		apps:       map[string]*appInfo{},	// 初始化
		registry:   map[string]struct{}{},	// 初始化
	}
	......
}
```

####	实现 `naming.Builder` 的方法
`EtcdBuilder.Scheme` 方法：
```golang
func (e *EtcdBuilder) Scheme() string {
	return "etcd"

}
```

`EtcdBuilder.Build` 方法完成如下的事情：
1.	创建以 `appid string` 为 key 的消费者 `r`，`r` 是 `etcd.Resolve` 结构，并返回
2.	构建映射关系，主要是 `EtcdBuilder` <---> `appid` <---> `appInfo` <---> `Resolve`(EtcdBuilder)
3.	使用 `sync.Once` 针对 `appid` 开启唯一的 `watcher` 协程

```golang
func (e *EtcdBuilder) Build(appid string, opts ...naming.BuildOpt) naming.Resolver {
	r := &Resolve{
		id:    appid,
		e:     e,
		event: make(chan struct{}, 1),
		opt:   new(naming.BuildOptions),
	}
	e.mutex.Lock()
	// 查看以 appid 为 key 是否存在，不存在则创建
	app, ok := e.apps[appid]
	if !ok {
		app = &appInfo{
			resolver: make(map[*Resolve]struct{}),
			e:        e,
		}
		e.apps[appid] = app
	}
	// 将 r 保存在 e.apps[string].resolver[*Resolve]
	app.resolver[r] = struct{}{}
	e.mutex.Unlock()

	// 已存在
	if ok {
		select {
		case r.event <- struct{}{}:
		default:
		}
	}

	// 开启针对 appid 的 watcher 机制
	app.once.Do(func() {
		//appinfo 实现的具体方法
		go app.watch(appid)
		log.Info("etcd: AddWatch(%s) already watch(%v)", appid, ok)
	})
	return r
}
```
####	实现 `naming.Registry` 的方法
`EtcdBuilder.Registry` 用来完成将服务的注册到 Etcd 及 TTL 续期，`Register` 方法主要用于提供给服务端进行服务注册，基于先前的经验，此方法必须实现如下的逻辑：
1.  向注册中心标识自己的服务名字（ServiceNameId）
2.  向注册中心上报自己的服务 IP 和端口信息（及其他一些信息，如权重等等）
3.  解除注册的逻辑，如采用定期上报心跳、出现异常时主动解除注册等

参数 `ins` 中保存了需要注册到 Etcd 的信息，此外需要注意的是：通过 ticker 方法来续期，特别注意的是 ticker 的周期要小于 TTL，不然 Etcd 中保存的 Key 会因 TTL 超时被删除。其实这里的场景还是比较复杂的，如重复注册的处理、Key 异常丢失等等。Warden 中默认的 TTL 为 `registerTTL=90` 秒

```golang
func (e *EtcdBuilder) Register(ctx context.Context, ins *naming.Instance) (cancelFunc context.CancelFunc, err error) {
	e.mutex.Lock()
	if _, ok := e.registry[ins.AppID]; ok {
		// 不允许重复注册
		err = ErrDuplication
	} else {
		e.registry[ins.AppID] = struct{}{}
	}
	e.mutex.Unlock()
	if err != nil {
		return
	}
	ctx, cancel := context.WithCancel(e.ctx)
	// 调用 e.register 进行注册，如果注册失败则解除注册
	if err = e.register(ctx, ins); err != nil {
		e.mutex.Lock()
		delete(e.registry, ins.AppID)
		e.mutex.Unlock()
		cancel()
		return
	}
	ch := make(chan struct{}, 1)
	cancelFunc = context.CancelFunc(func() {
		cancel()
		<-ch
	})

	go func() {
		// ticker 的周期设置为 TTL 的 1/3
		ticker := time.NewTicker(time.Duration(registerTTL/3) * time.Second)
		defer ticker.Stop()
		for {
			select {
			case <-ticker.C:
				_ = e.register(ctx, ins)
			case <-ctx.Done():
				_ = e.unregister(ins)
				ch <- struct{}{}
				return
			}
		}
	}()
	return
}
```

简单看看 `register` 和 `unregister` 方法，主要是调用 Etcd 的接口进行 key 操作，Kratos 中的 keyPrefix 组成是 `fmt.Sprintf("/%s/%s/%s", etcdPrefix, ins.AppID, ins.Hostname)`，注意 Etcd 是以 `/` 来标识前缀的，这点前文也已经分析过源码。Kratos 中注册和续约公用 `register` 操作，也可以使用 Etcd 提供的 `KeepAlive` 来完成续期的功能。
`register` 及 `unregister` 方法如下：
```golang
func (e *EtcdBuilder) register(ctx context.Context, ins *naming.Instance) (err error) {
	prefix := e.keyPrefix(ins)
	val, _ := json.Marshal(ins)

	ttlResp, err := e.cli.Grant(context.TODO(), int64(registerTTL))
	if err != nil {
		log.Error("etcd: register client.Grant(%v) error(%v)", registerTTL, err)
		return err
	}
	_, err = e.cli.Put(ctx, prefix, string(val), clientv3.WithLease(ttlResp.ID))
	if err != nil {
		log.Error("etcd: register client.Put(%v) appid(%s) hostname(%s) error(%v)",
			prefix, ins.AppID, ins.Hostname, err)
		return err
	}
	return nil
}

func (e *EtcdBuilder) unregister(ins *naming.Instance) (err error) {
	prefix := e.keyPrefix(ins)

	if _, err = e.cli.Delete(context.TODO(), prefix); err != nil {
		log.Error("etcd: unregister client.Delete(%v) appid(%s) hostname(%s) error(%v)",
			prefix, ins.AppID, ins.Hostname, err)
	}
	log.Info("etcd: unregister client.Delete(%v)  appid(%s) hostname(%s) success",
		prefix, ins.AppID, ins.Hostname)
	return
}
```

实现 `Close` 方法，常用的套路，调用 `e.cancelFunc()` 取消所有的子协程，配合 `case <-ctx.Done()` 使用：
```golang
// Close stop all running process including etcdfetch and register
func (e *EtcdBuilder) Close() error {
	e.cancelFunc()
	return nil
}
```

####	`etcd.appInfo` 封装的方法
`appInfo` 可以看做是一个独立的监听器：
-	watch
-	fetchstore
-	store
-	paserIns

```golang
func (a *appInfo) watch(appID string) {
	_ = a.fetchstore(appID)
	prefix := fmt.Sprintf("/%s/%s/", etcdPrefix, appID)
	rch := a.e.cli.Watch(a.e.ctx, prefix, clientv3.WithPrefix())
	for wresp := range rch {
		for _, ev := range wresp.Events {
			if ev.Type == mvccpb.PUT || ev.Type == mvccpb.DELETE {
				_ = a.fetchstore(appID)
			}
		}
	}
}

func (a *appInfo) fetchstore(appID string) (err error) {
	prefix := fmt.Sprintf("/%s/%s/", etcdPrefix, appID)
	resp, err := a.e.cli.Get(a.e.ctx, prefix, clientv3.WithPrefix())
	if err != nil {
		log.Error("etcd: fetch client.Get(%s) error(%+v)", prefix, err)
		return err
	}

	ins, err := a.paserIns(resp)
	if err != nil {
		return err
	}
	a.store(ins)
	return nil
}
func (a *appInfo) store(ins *naming.InstancesInfo) {

	a.ins.Store(ins)
	a.e.mutex.RLock()
	for rs := range a.resolver {
		select {
		case rs.event <- struct{}{}:
		default:
		}
	}
	a.e.mutex.RUnlock()
}

func (a *appInfo) paserIns(resp *clientv3.GetResponse) (ins *naming.InstancesInfo, err error) {
	ins = &naming.InstancesInfo{
		Instances: make(map[string][]*naming.Instance, 0),
	}
	for _, ev := range resp.Kvs {
		in := new(naming.Instance)

		err := json.Unmarshal(ev.Value, in)
		if err != nil {
			return nil, err
		}
		ins.Instances[in.Zone] = append(ins.Instances[in.Zone], in)
	}
	return ins, nil
}
```

####	实现 `naming.Resolver` 方法
```golang

// Watch watch instance.
func (r *Resolve) Watch() <-chan struct{} {
	return r.event
}

// Fetch fetch resolver instance.
func (r *Resolve) Fetch(ctx context.Context) (ins *naming.InstancesInfo, ok bool) {
	r.e.mutex.RLock()
	app, ok := r.e.apps[r.id]
	r.e.mutex.RUnlock()
	if ok {
		ins, ok = app.ins.Load().(*naming.InstancesInfo)
		return
	}
	return
}

// Close close resolver.
func (r *Resolve) Close() error {
	r.e.mutex.Lock()
	if app, ok := r.e.apps[r.id]; ok && len(app.resolver) != 0 {
		delete(app.resolver, r)
	}
	r.e.mutex.Unlock()
	return nil
}
```


##	参考
- [Warden resovler](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-resolver.md)