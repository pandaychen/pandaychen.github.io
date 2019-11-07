---
layout:     post
title:      基于gRPC+ETCD实现的服务发现与负载均衡
subtitle:   gRPC实战
date:       2019-07-11
author:     pandaychen
header-img: 
catalog: true
tags:
    - Etcd
    - gRPC
    - Loadbalance
---

## 前言

&emsp;&emsp;关于可用性构建中比较核心的一环，服务发现(Service discovery)和负载均衡(Load balance)一直都是我关注的话题。今天来谈一下我在实际中是如何理解和开发的。

## 负载均衡 && 服务发现

### 基础

&emsp;&emsp;负载均衡，顾名思义，是通过某种手段将流量/请求分配到不通的服务器上去，保证后台的每个服务收到的请求都尽可能保持平衡

&emsp;&emsp;服务发现，就是指客户端按照某种约定的方式主动去（注册中心）寻找服务，然后再连接相应的服务

&emsp;&emsp;关于负载均衡的构建与实现，可以看下这两篇文章：<br>
-   [gRPC服务发现&负载均衡](https://segmentfault.com/a/1190000008672912)
-   [gRPC Load Balancing](https://gRPC.io/blog/loadbalancing/)

### 服务发现概念

&emsp;&emsp;我们说的服务发现，一般理解为客户端如何发现(并连接到)服务，这里一般包含三个组件：

1. 服务消费者，一般指客户端（可以是简单的TCP-Client或者是RPC-Client）
2. 服务提供者，一般指服务提供方，如传统服务，微服务等
3. 服务注册中心，用来存储（Key-Value）服务提供者的服务，一般以DNS/HTTP/RPC等方式对外暴露接口

### 负载均衡概念

&emsp;&emsp;我们把LB看作一个组件，根据组件位置的不同，大致上分为三种：

1. &emsp;&emsp;独立的LB,可以是硬件实现，如F5，或者是nginx这种内置Proxy-pass或者upstream功能的网关，亦或是LVS/HAPROXY，之前也使用[DPDK](http://core.dpdk.org/doc/quick-start/)开发过类似的专用网关。<br>
![image](https://image-static.segmentfault.com/376/097/3760970390-58c6367e9e8e5_articlex)
<br>
2. &emsp;&emsp;进程内LB（集成到客户端），此方案将LB的功能集成到服务消费方进程里，也被称为软负载或者客户端负载方案。服务提供方启动时，首先将服务地址注册到服务注册表，同时定期报心跳到服务注册表以表明服务的存活状态，相当于健康检查，服务消费方要访问某个服务时，它通过内置的LB组件向服务注册表查询，同时缓存并定期刷新目标服务地址列表，然后以某种负载均衡策略选择一个目标服务地址，最后向目标服务发起请求。LB和服务发现能力被分散到每一个服务消费者的进程内部，同时服务消费方和服务提供方之间是直接调用，没有额外开销，性能比较好。<br>
![image](https://image-static.segmentfault.com/816/567/816567186-58c636a93e391_articlex)
<br>
3.  &emsp;&emsp;该方案是针对上一种方案的不足而提出的一种折中方案，原理和第二种方案基本类似。不同之处是将LB和服务发现功能从进程内移出来，变成主机上的一个独立进程。主机上的一个或者多个服务要访问目标服务时，他们都通过同一主机上的独立LB进程做服务发现和负载均衡。该方案也是一种分布式方案没有单点问题，一个LB进程挂了只影响该主机上的服务调用方，服务调用方和LB之间是进程内调用性能好，同时该方案还简化了服务调用方，不需要为不同语言开发客户库，LB的升级不需要服务调用方改代码。 公司的L5是这种方式，每台机器上都安装了L5的agent，供其他服务调用。该方案主要问题：部署较复杂，环节多，出错调试排查问题不方便。<br>
![](https://image-static.segmentfault.com/157/460/1574606891-58c636b7d0619_articlex)
<br>

### gRPC内置的方案
&emsp;&emsp;gRPC的内置方案如下图所示：<br>
![image](https://image-static.segmentfault.com/210/753/2107536928-58c636c2d6702_articlex)
<br>

&emsp;&emsp;gRPC在官网文档中提供了实现LB的思路，并在不同语言的gRPC代码API中已提供了命名解析和负载均衡接口供扩展。默认提供了[DNS-resovler的实现](https://github.com/gRPC/gRPC-go/blob/v1.8.0/resolver/resolver.go)，接口相当规范，实现起来也不复杂，只需要实现服务注册（Registry）和服务监听+解析（Watcher+Resolver）的逻辑就行了，这里简单介绍其基本实现过程：

-   构建注册中心，这里注册中心一般要求具备分布式一致性（满足CAP定理的AP或CP）的高可用的组件集群，如Zookeeper、Consul、Etcd等
-   构建gRPC服务端的注册逻辑，服务启动后定时向注册中心注册自身的关键信息（一般开启新的groutine来完成），至少包含IP和端口，其他可选信息，如自身的负载信息（CPU和memory）、当前实时连接数等，这些辅助信息有助于帮助系统更好的执行LB算法
-   gRPC客户端向注册中心发出服务解析请求（schema），注册中心将请求中关联的所有服务的信息返回给gRPC客户端，客户端与所有在线的服务建立起HTTP2长连接
-   gRPC客户端发起RPC调用，根据LB均衡器中实现的负载均衡策略（gRPC中默认提供的算法是RoundRobin），选择其中一HTTP2长连接进行通信，即LB策略决定那个子通道即gRPC服务器将接收请求
-   RPC请求完成

