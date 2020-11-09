---
layout:     post
title:      基于 gRPC 的服务发现与负载均衡（基础篇）
subtitle:   gRPC 负载均衡架构分析
date:       2019-07-11
author:     pandaychen
catalog: true
tags:
    - gRPC
    - 负载均衡
---

##  0x00    前言
&emsp;&emsp; 在后台服务开发中，高可用性是构建中核心且重要的一环。服务发现（Service discovery）和负载均衡（Load Balance）一直都是我关注的话题。今天来谈一下我在实际中是如何理解及落地的。

##  0x01 负载均衡 && 服务发现

#### 基础
&emsp;&emsp; <font color="#dd0000"> 负载均衡 </font>，顾名思义，是通过某种手段将流量 / 请求分配到不通的服务器上去，保证后台的每个服务收到的请求都尽可能保持平衡 <br>
&emsp;&emsp; <font color="#dd0000"> 服务发现 </font>，就是指客户端按照某种约定的方式主动去（注册中心）寻找服务，然后再连接相应的服务 <br>
&emsp;&emsp; 关于负载均衡的构建与实现，可以看下这几篇文章：
-   [gRPC 服务发现 & 负载均衡](https://segmentfault.com/a/1190000008672912)
-   [gRPC Load Balancing](https://gRPC.io/blog/loadbalancing/)
-	[Load Balancing in gRPC](https://github.com/grpc/grpc/blob/master/doc/load-balancing.md)

#### 服务发现概念

&emsp;&emsp; 我们说的服务发现，一般理解为客户端如何发现 (并连接到) 服务，这里一般包含三个组件：
1. 服务消费者：一般指客户端（可以是简单的 TCP-Client 或者是 RPC-Client ）
2. 服务提供者：一般指服务提供方，如传统服务，微服务等
3. 服务注册中心：用来存储（Key-Value）服务提供者的服务，一般以 DNS/HTTP/RPC 等方式对外暴露接口

#### 负载均衡概念
我们把 LB 看作一个组件，根据组件位置的不同，大致上分为三种：
####    集中式 LB（Proxy Model）
&emsp;&emsp; 独立的 LB, 可以是硬件实现，如 F5，或者是 nginx 这种内置 Proxy-pass 或者 upstream 功能的网关，亦或是 LVS/HAPROXY，之前也使用 [DPDK](http://core.dpdk.org/doc/quick-start/) 开发过类似的专用网关。<br>
![image](https://image-static.segmentfault.com/376/097/3760970390-58c6367e9e8e5_articlex)


####    进程内 LB（Balancing-aware Client）
&emsp;&emsp; 进程内 LB（集成到客户端），此方案将 LB 的功能集成到服务消费方进程里，也被称为软负载或者客户端负载方案。服务提供方启动时，首先将服务地址注册到服务注册表，同时定期报心跳到服务注册表以表明服务的存活状态，相当于健康检查，服务消费方要访问某个服务时，它通过内置的 LB 组件向服务注册表查询，同时缓存并定期刷新目标服务地址列表，然后以某种负载均衡策略选择一个目标服务地址，最后向目标服务发起请求。LB 和服务发现能力被分散到每一个服务消费者的进程内部，同时服务消费方和服务提供方之间是直接调用，没有额外开销，性能比较好。<br>
![image](https://image-static.segmentfault.com/816/567/816567186-58c636a93e391_articlex)


####    独立 LB 进程（External Load Balancing Service）
&emsp;&emsp; 该方案是针对上一种方案的不足而提出的一种折中方案，原理和第二种方案基本类似。不同之处是将 LB 和服务发现功能从进程内移出来，变成主机上的一个独立进程。主机上的一个或者多个服务要访问目标服务时，他们都通过同一主机上的独立 LB 进程做服务发现和负载均衡。该方案也是一种分布式方案没有单点问题，一个 LB 进程挂了只影响该主机上的服务调用方，服务调用方和 LB 之间是进程内调用性能好，同时该方案还简化了服务调用方，不需要为不同语言开发客户库，LB 的升级不需要服务调用方改代码。 公司的 L5 是这种方式，每台机器上都安装了 L5 的 agent，供其他服务调用。该方案主要问题：部署较复杂，环节多，出错调试排查问题不方便。
![](https://image-static.segmentfault.com/157/460/1574606891-58c636b7d0619_articlex)


### gRPC 内置的方案
&emsp;&emsp;gRPC 的内置方案如下图所示：
![image](https://image-static.segmentfault.com/210/753/2107536928-58c636c2d6702_articlex)
<br>
&emsp;&emsp;gRPC 在官网文档中提供了实现 LB 的思路，并在不同语言的 gRPC 代码 API 中已提供了命名解析和负载均衡接口供扩展。默认提供了 [DNS-resolver 的实现](https://github.com/gRPC/gRPC-go/blob/v1.8.0/resolver/resolver.go)，接口相当规范，实现起来也不复杂，只需要实现服务注册（Registry）和服务监听 + 解析（Watcher+Resolver）的逻辑就行了，这里简单介绍其基本实现过程：

1.	构建注册中心，这里注册中心一般要求具备分布式一致性（满足 CAP 定理的 AP 或 CP）的高可用的组件集群，如 Zookeeper、Consul、Etcd 等
2.	构建 gRPC 服务端的注册逻辑，服务启动后定时向注册中心注册自身的关键信息（一般开启新的 groutine 来完成），至少包含 IP 和端口，其他可选信息，如自身的负载信息（CPU 和 Memory）、当前实时连接数等，这些辅助信息有助于帮助系统更好的执行 LB 算法
3.	gRPC 客户端向注册中心发出服务解析请求，注册中心将请求中关联的所有服务的信息返回给 gRPC 客户端，客户端与所有在线的服务建立起 HTTP2 长连接
4.	gRPC 客户端发起 RPC 调用，根据 LB 均衡器中实现的负载均衡策略（gRPC 中默认提供的算法是 RoundRobin），选择其中一 HTTP2 长连接进行通信，即 LB 策略决定哪个子通道 - 即哪个 gRPC 服务器将接收请求

##	0x02 gRPC 负载均衡的运行机制
gRPC 提供了负载均衡实现的用户侧接口，我们可以非常方便的定制化业务的负载均衡策略，为了理解 gRPC 的负载均衡的实现机制，后续博客中我会分析下 `gRPC` 实现负载均衡的代码。
![grpc-lb-basic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc-lb-basic1.png)
1.  Resolver
	-	解析器，用于从注册中心实时获取当前服务端的列表，同步发送给 Balancer
2.  Balancer
	-	平衡器，一是接收从 Resolver 发送的服务端列表，建立并维护（长）连接状态；二是每次当 Client 发起 Rpc 调用时，按照一定算法从连接池中选择一个连接进行 Rpc 调用
3.  Register
	-	注册，用于服务端初始化和在线时，将自己信息上报到注册中心，主要信息有 Ip，端口等

##  0x03 负载均衡的算法及实现
在实践中，如何选取负载均衡策略是一个很有趣的话题，例如 Nginx 的 `upstream` 机制中就有很多经典的 LB 策略，如带权重的轮询 [Weight-RoundRobin](https://github.com/nginx/nginx/blob/master/src/http/ngx_http_upstream_round_robin.c)，一般常用的负载均衡方法有如下几种：

1.  RoundRobin（轮询）
2.  Weight-RoundRobin（加权轮询）<br>
    -   不同的后端服务器可能机器的配置和当前系统的负载并不相同，因此它们的抗压能力也不相同。给配置高、负载低的机器配置更高的权重，而配置低、负载高的机器，给其分配较低的权重，降低其系统负载，加权轮询能很好地处理这一问题，并将请求顺序且按照权重分配到后端。
3.  Random（随机）
4.  Weight-Random（加权随机）<br>
	-	通过系统的随机算法，根据后端服务器的列表随机选取其中的一台服务器进行访问
5.  源地址哈希法
	-	源地址哈希的思想是根据获取客户端的 IP 地址，通过哈希函数计算得到的一个数值，用该数值对服务器列表的大小进行取模运算，得到的结果便是客服端要访问服务器的序号。采用源地址哈希法进行负载均衡，同一 IP 地址的客户端，当后端服务器列表不变时，它每次都会映射到同一台后端服务器进行访问
6.  最小连接数法
	-	最小连接数算法比较灵活和智能，由于后端服务器的配置不尽相同，对于请求的处理有快有慢，它是根据后端服务器当前的连接情况，动态地选取其中当前积压连接数最少的一台服务器来处理当前的请求，尽可能地提高后端服务的利用效率，将负责合理地分流到每一台服务器
7.  一致性哈希算法
	-	常见的是 `Ketama` 算法，该算法是用来解决 `cache` 失效导致的缓存穿透的问题的，当然也可以适用于 gRPC 长连接的场景

##	0x04 gRPC 服务治理的优势
&emsp;&emsp; 在现网环境中，后端服务就是采用了 gRPC 与 Etcd 的服务治理方案，总结下有这么几个优点；
-   采用了 gRPC 实现负载均衡策略，模块之间通信采用长连接方式，避免每次 RPC 调用时新建连接的开销，充分发挥 `HTTP2` 的优势
-   扩容和缩容都及其方便，例如扩容，只要部署上服务，运行后，服务成功注册到 Etcd 便大功告成
-   灵活的自定义的 LB 算法，使得后端压力更为均衡
-   客户端加入重试逻辑，使得网络抖动情况下，可以通过重试连接上另外一台服务

## 0x05	总结
总之，利用 gRPC 接口实现服务端的负载均衡及高可用，这套方案现网实战的效果还是很不错的，后面再写一篇如何使用 Etcd 和 gRPC 实现自定义负载均衡算法的文章。

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
