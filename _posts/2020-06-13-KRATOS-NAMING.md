---
layout:     post
title:      Kratos 源码分析（）：Warden 中的多消费者订阅 - Watcher 模式
subtitle:   分析 Kratos 的 Naming 解析器（gRPC-Resolver）
date:       2020-06-13
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
本文分析下 Kratos 框架中的服务注册与发现的代码。

##  0x01   Naming 的公共接口
Kratos 的 [Naming 模块](https://github.com/go-kratos/kratos/blob/master/pkg/naming/naming.go)，是装饰器模式（Decorator）的典型实现。它封装了三个通用的方法：
`Resolver`、`Registry` 及 `Builder`:

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

```golang
// Resolver resolve naming service
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

// Registry Register an instance and renew automatically.
/*
	对应于 register 的实现逻辑
	1. 注册
	2. 解除注册
*/
type Registry interface {
	Register(ctx context.Context, ins *Instance) (cancel context.CancelFunc, err error)
	Close() error
}

// Builder resolver builder.
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

剩下的工作就是对上面这几个方法的实例化（Decorator），Kratos 中提供了 discovery、etcd 及 zookeeper 的三种实现。本文简单分析下 etcd 的相关接口实现。

##  0x02    多消费者订阅-Watcher模型
![image](https://wx1.sbimg.cn/2020/06/12/4ccdbe50e68dd4e5269b30016737d876.png)

##  0x03    Etcd 的 Naming 实现
基于Etcd的封装[代码在此](https://github.com/go-kratos/kratos/blob/master/pkg/naming/etcd/etcd.go)

####    核心结构

`EtcdBuilder`

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

```golang
type appInfo struct {
	resolver map[*Resolve]struct{}
	ins      atomic.Value
	e        *EtcdBuilder
	once     sync.Once
}
```

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


####    Registry的实现

```golang
func (e *EtcdBuilder) Register(ctx context.Context, ins *naming.Instance) (cancelFunc context.CancelFunc, err error) {
	e.mutex.Lock()
	if _, ok := e.registry[ins.AppID]; ok {
		err = ErrDuplication
	} else {
		e.registry[ins.AppID] = struct{}{}
	}
	e.mutex.Unlock()
	if err != nil {
		return
	}
	ctx, cancel := context.WithCancel(e.ctx)
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

//注册和续约公用一个操作
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

func (e *EtcdBuilder) keyPrefix(ins *naming.Instance) string {
	return fmt.Sprintf("/%s/%s/%s", etcdPrefix, ins.AppID, ins.Hostname)
}
```


##
-   []()