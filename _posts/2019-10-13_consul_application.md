---
layout:     post
title:      Consul的那些事
subtitle:   使用Consul构建高可用的后端服务
date:       2019-10-12
author:     pandaychen
header-img: 
catalog: true
tags:
    - Consul
    - Loadbalance
    - GRPC
---

##  Consul的那些事
业余时间利用GRPC+CONSUL实现的服务发现一个项目(grpclb2consul)[https://github.com/pandaychen/grpclb2consul/]。这篇文章，总结下我在开发和Consul使用过程中的一些经验<br>

##  Consul介绍

目前我使用到的Consul功能，服务发现与注册（含健康检查）/分布式KeyValue存储/配置中心/DNS/分布式锁<br>

Consul和Etcd的功能是有重叠的（有很多文章都有对二者进行比较），不过Etcd仅仅是一个CP的KV存储，而Consul更像是一个微服务开发的完整套件。Consul的官网介绍：<br>

-   服务发现<br>
Consul的客户端可用提供一个服务，比如 api 或者mysql，另外一些客户端可用使用Consul去发现一个指定服务的提供者。通过DNS或者HTTP应用程序可用很容易的找到他所依赖的服务

-   健康检查<br>
Consul客户端可用提供任意数量的健康检查,指定一个服务(比如:webserver是否返回了200 OK 状态码)或者使用本地节点(比如:内存使用是否大于90%). 这个信息可由operator用来监视集群的健康.被服务发现组件用来避免将流量发送到不健康的主机.

-   Key/Value存储<br>
应用程序可用根据自己的需要使用Consul的层级的Key/Value存储.比如动态配置,功能标记,协调,领袖选举等等,简单的HTTP API让他更易于使用.

-   多数据中心<br>
Consul支持开箱即用的多数据中心.这意味着用户不需要担心需要建立额外的抽象层让业务扩展到多个区域。（ps：Consul的集群间同步采用的是[gossip](https://en.wikipedia.org/wiki/Gossip_protocol)协议，和Redis-Cluster同步使用的是一样的协议）


##  为什么要使用服务治理
我们的项目，后台都是GRPC的服务，最早的调用方式都是RPC调用满天飞，各种短连接请求；另外负载均衡的方式主要是DNS。这种方式主要有以下几个缺点（当然现网中也遇到过）：<br>

-   性能不高<br>
短连接调用，连接的建立有开销，一旦调用路径增加，性能和时延的问题都会凸显

-   可用性较差<br>
靠DNS来做LoadBalance，是存在风险的，一旦其中一个后端故障又没有机器剔除后端的话，不幸调度到失败节点上，服务就可用，当然我们现网也遇到这种情况
-   扩容麻烦<br>
扩容，缩容，都需要修改DNS指向

-   服务上下线切换<br>
还是可用性的问题，上下线切换需要重启服务，由于DNS的特性，存在服务不可用的风险

使用服务治理就可以解决我们上面的这些痛点，当然了，引入一些技术栈，会增加系统总体的复杂度，但是这个微服务的年代，传统的单体架构不够灵活不能很好的适应变化，我们需要一些自动化的策略，服务发现应运而生。


##	Consul内部原理
想搞懂这些分布式系统的原理及使用，有几个关键字一定要弄清楚。

-	CAP定理
-	集群化部署
-	RAFT协议与分布式选主
	
下面这张图来源于Consul官网，很好的解释了Consul的工作原理，先大致看一下。
![image](https://s2.ax1x.com/2019/10/17/KEfbg1.png)


###	集群架构与节点类型
首先Consul支持多数据中心，在上图中有两个数据中心（DataCenter1个DataCenter2），他们通过Internet互联，同时请注意为了提高通信效率，只有Server节点才加入跨数据中心的通信。

在单个数据中心中，Consul分为Client和Server两种节点（<font color=red>所有的节点也被称为Agent</font>），Server节点保存数据，Client负责健康检查（healthy check）及转发数据请求到Server；

在Consul集群中，Server节点是一定需要的，Client节点可以不需要；

同一集群内的Server节点（图中每个DataCenter都具由3个Server节点）有一个Leader和多个Follower，Leader节点会将数据同步到Follower，Server的数量推荐是3个或者5个，在Leader挂掉的时候会启动选举机制产生一个新的Leader。Server间的选举遵循(RAFT算法)[https://github.com/hashicorp/raft]

集群内的Consul节点通过gossip协议维护成员关系（这里指的是Client和Server之间的通信），也就是说某个节点了解集群内现在还有哪些节点，这些节点是Client还是Server。单个数据中心的gossip协议同时使用TCP和UDP通信，并且都使用8301端口。

跨数据中心的gossip协议也同时使用TCP和UDP通信，端口使用8302。

###	数据读写的流程（和ETCD类似）
集群内数据的读写请求既可以直接发到Server，也可以通过Client使用RPC转发到Server，请求最终会到达Leader节点，在允许数据轻微陈旧的情况下，读请求也可以在普通的Server节点完成，集群内数据的读写和复制都是通过TCP的8300端口完成。

##	Consul服务发现原理
下面这张图基本描述了服务发现的完整流程
![image](consul2.png)
