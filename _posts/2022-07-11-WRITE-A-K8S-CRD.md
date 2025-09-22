---
layout: post
title: Kubernetes 应用改造（九）：CRD && Controller && Operator 入门
subtitle:  
date: 2023-07-11
author: pandaychen
catalog: true
tags:
  - Kubernetes
---

## 0x00 前言
先列举几个概念：

- CRD（Custom Resource Definition）
- Controller
- Operator：Operator 是一种封装、部署和管理 kubernetes 应用的方法

####  CR && CRD
本小节摘自[CRD is just a table in Kubernetes](https://itnext.io/crd-is-just-a-table-in-kubernetes-13e15367bbe4)，形象的介绍了CR(D)


####  Operator的应用场景
Operator 是由 kubernetes 自定义资源（CRD, Custom Resource Definition）和控制器（Controller）构成的云原生扩展服务。Operator 把部署应用的过程或业务运行过程中需要频繁操作的命令封装起来，运维人员只需要关注应用的配置和应用的期望运行状态即可，无需花费大量的精力在部署和容易出错的命令上面。如 Redis Operator，只需要关心 `size` 和 `config` 即可，而不用关心其部署过程：

```YAML
apiVersion: Redis.io/v1beta1
kind: RedisCluster
metadata:
  name: my-release
spec:
  size: 3
  imagePullPolicy: IfNotPresent
  resources:
    limits:
      cpu: 1000m
      memory: 1Gi
    requests:
      cpu: 1000m
      memory: 1Gi
  config:
    maxclients: "10000"
```

或者一些自己开发的微服务应用：

```YAML
apiVersion: custom.ops/v1
kind: MicroServiceDeploy
metadata:
  name: ms-sample-v1s0
spec:
  msName: "ms-sample"                     # 微服务名称
  fullName: "ms-sample-v1s0"              # 微服务实例名称
  version: "1.0"                          # 微服务实例版本
  path: "v1"                              # 微服务实例的大版本，该字符串将出现在微服务实例的域名中
  image: "just a image url"               # 微服务实例的镜像地址
  replicas: 3                             # 微服务实例的 replica 数量
  autoscaling: true                       # 该微服务是否开启自动扩缩容功能
  needAuth: true                          # 访问该微服务实例时，是否需要租户 base 认证
  config: "password=12345678"             # 该微服务实例的运行时配置项
  creationTimestamp: "1535546718115"      # 该微服务实例的创建时间戳
  resourceRequirements:                   # 该微服务实例要求的机器资源
    limits:                               # 该微服务实例会使用到的最大资源配置
      cpu: "2"
      memory: 4Gi
    requests:                             # 该微服务实例至少要用到的资源配置
      cpu: "2"
      memory: 4Gi
  idle: false                             # 是否进入空载状态
```

####  CRD的使用场景
1. 提供、管理外部数据存储，因为Customer Resource就是一种存在Etcd中的数据结构，可以将外部数据存储到Etcd中而不使用单独的数据库，用Kubernetes的声明式API去管理其生命周期

2. 对Kubernetes的基础资源进行更高层次的抽象，例如：一些自定义控制器，既可以管理自定义的资源状态，也可以改变Kubernetes原有资源，如Ingress-controller


##  0x01  CRD介绍

##  0x02  Controller介绍

####  Controller工作原理

Controller可以有一个或多个informer来跟踪某一个resource。Informer跟API server保持通讯获取资源的最新状态并更新到本地的cache中，一旦跟踪的资源有变化，informer就会调用callback。把关心的变更的Object放到workqueue里面。然后woker执行真正的业务逻辑，计算和比较workerqueue里items的当前状态和期望状态的差别，然后通过client-go向API server发送请求，直到驱动这个集群向用户要求的状态演化

##  0x03 参考
- [Kubernetes 自定义控制器 Demo 例子](https://github.com/domac/crddemo)
- [Kubernetes CRD 开发实践](https://lihaoquan.me/posts/k8s-crd-develop/)
- [如何从零开始编写一个 Kubernetes CRD](https://cloudnative.to/blog/kubernetes-crd-quick-start/)
- [CRD 就像 Kubernetes 中的一张表](https://zhuanlan.zhihu.com/p/260797410)
- [KUBERNETES CRD&CONTROLLER入门实践](https://davidlovezoe.club/wordpress/archives/690)
- [Repository for sample controller. Complements sample-apiserver](https://github.com/kubernetes/sample-controller)
- [KubeFlow -> CRD开发实践](https://zhuanlan.zhihu.com/p/339946957)
- [Kubernetes Operator 开发教程](https://developer.aliyun.com/article/798703)
- [Operator基础-开始Operator开发](https://isekiro.com/operator%E5%9F%BA%E7%A1%80-%E5%BC%80%E5%A7%8Boperator%E5%BC%80%E5%8F%91/)
- [CRD is just a table in Kubernetes](https://itnext.io/crd-is-just-a-table-in-kubernetes-13e15367bbe4)
- [K8s 的核心是 API 而非容器（一）：从理论到 CRD 实践（2022）](http://arthurchiao.art/blog/k8s-is-about-apis-zh/)
- [Kubernetes Operator 快速入门教程](https://www.qikqiak.com/post/k8s-operator-101/)
- [开发Redis CRD](https://wghdr.top/?p=4150)
- [从零开始写一个 kubebuilder](https://typonotes.com/books/kubebuilder-zero-to-one/chapter01/01-kubebuilder-init-project/)