---
layout:     post
title:      Consul服务治理的那些事(一)
subtitle:   使用gRPC+Consul构建高可用的后端服务
date:       2019-10-12
author:     pandaychen
catalog:    true
tags:
    - Consul
    - Loadbalance
    - gRPC
---

业余时间利用gRPC+Consul实现的服务发现一个项目[grpclb2consul](https://github.com/pandaychen/grpclb2consul/)。这篇文章，总结下我在开发和Consul使用过程中的一些经验<br>

##  Consul介绍

目前我使用到的Consul功能，服务发现与注册（含健康检查）/分布式KeyValue存储/配置中心/DNS/分布式锁,总而言之，只有理解了Consul的设计思路，才能熟练的应用它。（Consul应用思路和Etcd真的很不一样）<br>

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
我们的项目，后台都是gRPC的服务，最早的调用方式都是RPC调用满天飞，各种短连接请求；另外负载均衡的方式主要是DNS。这种方式主要有以下几个缺点（当然现网中也遇到过）：<br>

-   性能不高<br>
短连接调用，连接的建立有开销，一旦调用路径增加，性能和时延的问题都会凸显

-   可用性较差<br>
靠DNS来做LoadBalance，是存在风险的，一旦其中一个后端故障又没有机器剔除后端的话，不幸调度到失败节点上，服务就不可用，当然我们现网也遇到这种情况
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

在单个数据中心中，Consul分为Client和Server两种节点（所有的节点也被称为Agent），Server节点保存数据，Client负责健康检查（healthy check）及转发数据请求到Server；

在Consul集群中，Server节点是一定需要的，Client节点可以不需要；

同一集群内的Server节点（图中每个DataCenter都具由3个Server节点）有一个Leader和多个Follower，Leader节点会将数据同步到Follower，Server的数量推荐是3个或者5个，在Leader挂掉的时候会启动选举机制产生一个新的Leader。Server间的选举遵循[RAFT算法](https://github.com/hashicorp/raft)

集群内的Consul节点通过gossip协议维护成员关系（这里指的是Client和Server之间的通信），也就是说某个节点了解集群内现在还有哪些节点，这些节点是Client还是Server。单个数据中心的gossip协议同时使用TCP和UDP通信，并且都使用8301端口。

跨数据中心的gossip协议也同时使用TCP和UDP通信，端口使用8302。

###	数据读写的流程
集群内数据的读写请求既可以直接发到Server，也可以通过Client使用RPC转发到Server，请求最终会到达Leader节点，在允许数据轻微陈旧的情况下，读请求也可以在普通的Server节点完成，集群内数据的读写和复制都是通过TCP的8300端口完成。（和ETCD类似的流程）

##	Consul服务发现原理
下面这张图基本描述了服务发现的完整流程，一个可以在现网中集群部署的方式（简单）：<br>

![image](https://s2.ax1x.com/2019/10/17/KEqC8O.png)



首先需要有一个正常的Consul集群，有Server，有Leader。这里在服务器Server1、Server2、Server3上分别部署了Consul Server，假设他们选举了Server2上的Consul Server节点为Leader。这些服务器上最好只部署Consul程序，以尽量维护Consul Server的稳定。

然后在服务器Server4和Server5上通过Consul Client分别注册Service A、B、C，这里每个Service分别部署在了两个服务器上，这样可以避免Service的单点问题。服务注册到Consul可以通过HTTP API（8500端口）的方式，也可以通过Consul配置文件的方式。Consul Client可以认为是无状态的，它将注册信息通过RPC转发到Consul Server，服务信息保存在Server的各个节点中，并且通过Raft实现了强一致性。

最后在服务器Server6中Program D需要访问Service B，这时候Program D首先访问本机Consul Client提供的HTTP API，本机Client会将请求转发到Consul Server，Consul Server查询到Service B当前的信息返回，最终Program D拿到了Service B的所有部署的IP和端口，然后就可以选择Service B的其中一个部署并向其发起请求了。如果服务发现采用的是DNS方式，则Program D中直接使用Service B的服务发现域名，域名解析请求首先到达本机DNS代理，然后转发到本机Consul Client，本机Client会将请求转发到Consul Server，Consul Server查询到Service B当前的信息返回，最终Program D拿到了Service B的某个部署的IP和端口。

## Consul-Docker部署

Consul的docker镜像基于alpine构建的，进入容器的时候需要指定/bin/sh

首先拉取镜像：
```
docker pull consul      #拉取镜像
```

先启动第一个Docker：

```
#启动第1个Server节点，集群要求要有3个Server，将容器8500端口映射到主机8900端口，同时开启管理界面
docker run -d --name=consul_1 -p 8900:8500 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --bootstrap-expect=3 --client=0.0.0.0 -ui
```
使用docker exec -ti 7efe(容器ID)  /bin/sh进入容器查看下容器的内网IP:172.17.0.2，以此IP为Server节点将其他Consul-Node加入并构建集群：
![image](https://s2.ax1x.com/2019/10/17/KELqB9.png)

```
#启动第2个Server节点，并加入集群
docker run -d --name=consul_2 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.2
 
#启动第3个Server节点，并加入集群
docker run -d --name=consul_3 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.2
 
#启动第4个Client节点，并加入集群
docker run -d --name=consul_4 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=false --client=0.0.0.0 --join 172.17.0.2
```

集群搭建完成后，我们在容器中执行consul members，查看集群的组成信息，我们的集群中，启动4个Consul Agent，3个Server（会选举出一个leader），1个Client。
![image](https://s2.ax1x.com/2019/10/17/KEO5VA.png)

这些Consul节点在Docker的容器内是互通的，他们通过桥接的模式通信。但是如果主机要访问容器内的网络，需要做端口映射。在启动第一个容器时，将Consul的8500端口映射到了主机的8900端口，这样就可以方便的通过主机的浏览器查看集群信息。


##  Consul的WEB-UI
根据上一步的暴露的UI端口8900，打开主页，能看到我们的集群中节点信息，共计4个节点。
![image](https://s2.ax1x.com/2019/10/17/KEjXAs.png)


##  Consul服务发现测试

### 服务注册
Consul通用的注册方式，JSON配置文件，需要在配置文件中指定两个重要信息，一是服务的IP和端口，二是健康检查的方法，尤其要注意健康检查，这个在Consul实现服务注册时特别重要，一旦健康检查服务失败，服务会被标记为下线。这个地方需要注意，我在另外一篇文章中详细说。<br>

在测试服务端[server.go](https://github.com/pandaychen/grpclb2consul/blob/master/example/server.go)中，设置gRPC健康检查方式为TTL，Consul-Agent地址设置为http://172.17.0.2:8500，运行gRPC-Server：
![image](https://s2.ax1x.com/2019/10/18/KVCUvq.png)

查看WEB，健康检查通过，服务启动成功：
![image](https://s2.ax1x.com/2019/10/18/KVCDVU.png)

### 服务发现
在测试客户端[client.go](https://github.com/pandaychen/grpclb2consul/blob/master/example/client.go)中，设置Consul-Agent地址为http://172.17.0.3:8500，注意这里和Server设置的不一样（当然也可以一样），运行gRPC-Client:
![image](https://s2.ax1x.com/2019/10/18/KVCfr6.png)

##  后记
至此，一个gRPC+Consul的可用框架就搭建完成了。后面我还会写一篇文章，介绍Consul服务注册的详细方法、Consul与Etcd的比较。<br>




