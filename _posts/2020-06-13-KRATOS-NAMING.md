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

##  多消费者订阅-Watcher模型
![image](https://wx1.sbimg.cn/2020/06/12/4ccdbe50e68dd4e5269b30016737d876.png)

##  Etcd 的 Naming 实现
基于Etcd的封装[代码在此[(https://github.com/go-kratos/kratos/blob/master/pkg/naming/etcd/etcd.go)


##
-   []()