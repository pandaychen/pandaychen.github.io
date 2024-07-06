---
layout:     post
title:      Consul 服务治理的那些事（一）
subtitle:   使用 gRPC+Consul 构建高可用的后端服务
date:       2019-10-12
author:     pandaychen
catalog:    true
tags:
    - Consul
    - 负载均衡
    - gRPC
---

##  0x00    前言
业余时间利用 gRPC 与 Consul 实现的服务发现项目 [grpclb2consul](https://github.com/pandaychen/grpclb2consul/)。这篇文章，总结下我在开发和 Consul 使用过程中的一些经验 <br>

##  0x01    Consul 介绍
目前我使用到的 Consul 功能，服务发现与注册（含健康检查）/ 分布式 KeyValue 存储 / 配置中心 / DNS / 分布式锁, 总而言之，只有理解了 Consul 的设计思路，才能熟练的应用它。（Consul 应用思路和 Etcd 真的很不一样）<br>

Consul 和 Etcd 的功能是有重叠的（有很多文章都有对二者进行比较），不过 Etcd 仅仅是一个 CP 的 KV 存储，而 Consul 更像是一个微服务开发的完整套件。Consul 的官网介绍：<br>

-   服务发现 <br>
Consul 的客户端可用提供一个服务，比如 API 或者 Mysql，另外一些客户端可用使用 Consul 去发现一个指定服务的提供者。通过 DNS 或者 HTTP 应用程序可用很容易的找到他所依赖的服务

-   健康检查 <br>
Consul 客户端可用提供任意数量的健康检查, 指定一个服务 (比如: Webserver 是否返回了 `200 OK` 状态码) 或者使用本地节点 (比如: 内存使用是否大于 `90%`). 这个信息可由 operator 用来监视集群的健康. 被服务发现组件用来避免将流量发送到不健康的主机.

-   Key/Value 存储 <br>
应用程序可用根据自己的需要使用 Consul 的层级的 Key/Value 存储. 比如动态配置, 功能标记, 协调, 领袖选举等等, 简单的 HTTP API 让他更易于使用.

-   多数据中心 <br>
Consul 支持开箱即用的多数据中心. 这意味着用户不需要担心需要建立额外的抽象层让业务扩展到多个区域。（ps：Consul 的集群间同步采用的是 [gossip](https://en.wikipedia.org/wiki/Gossip_protocol) 协议，和 Redis-Cluster 同步使用的是一样的协议）


##  0x02    服务治理的意义
笔者的上手项目，后台都是 gRPC 的服务，最早的调用方式都是 RPC 调用满天飞，各种短连接请求；另外负载均衡的方式主要是 DNS。这种方式主要有以下几个缺点：<br>

-   性能不高 <br>
短连接调用，连接的建立有开销，一旦调用路径增加，性能和时延的问题都会凸显

-   可用性较差 <br>
靠 DNS 来做 LoadBalance，是存在风险的，一旦其中一个后端故障又没有机器剔除后端的话，不幸调度到失败节点上，服务就不可用，当然现网也遇到这种情况
-   扩容麻烦 <br>
扩容，缩容，都需要修改 DNS 指向

-   服务上下线切换 <br>
还是可用性的问题，上下线切换需要重启服务，由于 DNS 的特性，存在服务不可用的风险

使用服务治理就可以解决上面的这些痛点，当然了，引入一些技术栈，会增加系统总体的复杂度，但是这个微服务的年代，传统的单体架构不够灵活不能很好的适应变化，需要一些自动化的策略，服务发现应运而生。


##	0x03    Consul 架构
想搞懂这些分布式系统的原理及使用，有几个关键字一定要弄清楚。

-	CAP 定理
-	集群化部署
-	RAFT 协议与分布式选主

下面这张图来源于 Consul 官网，很好的解释了 Consul 的工作原理，先大致看一下。
![image](https://s2.ax1x.com/2019/10/17/KEfbg1.png)


####	集群架构与节点类型
首先 Consul 支持多数据中心，在上图中有两个数据中心（DataCenter1 和 DataCenter2），通过 Internet 互联，同时请注意为了提高通信效率，只有 Server 节点才加入跨数据中心的通信。

在单个数据中心中，Consul 分为 Client 和 Server 两种节点（所有的节点也被称为 Agent），Server 节点保存数据，Client 负责健康检查（healthy check）及转发数据请求到 Server；

在 Consul 集群中，Server 节点是一定需要的，Client 节点可以不需要；

同一集群内的 Server 节点（图中每个 DataCenter 都具由 `3` 个 Server 节点）有一个 Leader 和多个 Follower，Leader 节点会将数据同步到 Follower，Server 的数量推荐是 `3` 个或者 `5` 个，在 Leader 挂掉的时候会启动选举机制产生一个新的 Leader。Server 间的选举遵循 [RAFT 算法](https://github.com/hashicorp/raft)

集群内的 Consul 节点通过 gossip 协议维护成员关系（这里指的是 Client 和 Server 之间的通信），也就是说某个节点了解集群内现在还有哪些节点，这些节点是 Client 还是 Server。单个数据中心的 gossip 协议同时使用 TCP 和 UDP 通信，并且都使用 `8301` 端口。

跨数据中心的 gossip 协议也同时使用 TCP 和 UDP 通信，端口使用 `8302`。

####	数据读写的流程
集群内数据的读写请求既可以直接发到 Server，也可以通过 Client 使用 RPC 转发到 Server，请求最终会到达 Leader 节点，在允许数据轻微陈旧的情况下，读请求也可以在普通的 Server 节点完成，集群内数据的读写和复制都是通过 TCP 的 `8300` 端口完成（和 ETCD 类似的流程）

##	0x04    Consul 服务发现原理
下面这张图基本描述了服务发现的完整流程，一个可以在现网中集群部署的方式（简单）：<br>

![image](https://s2.ax1x.com/2019/10/17/KEqC8O.png)

首先需要有一个正常的 Consul 集群，有 Server，有 Leader。这里在服务器 Server1、Server2、Server3 上分别部署了 Consul Server，假设他们选举了 Server2 上的 Consul Server 节点为 Leader。这些服务器上最好只部署 Consul 程序，以尽量维护 Consul Server 的稳定。

然后在服务器 Server4 和 Server5 上通过 Consul Client 分别注册 Service A、B、C，这里每个 Service 分别部署在了两个服务器上，这样可以避免 Service 的单点问题。服务注册到 Consul 可以通过 HTTP API（`8500` 端口）的方式，也可以通过 Consul 配置文件的方式。Consul Client 可以认为是无状态的，它将注册信息通过 RPC 转发到 Consul Server，服务信息保存在 Server 的各个节点中，并且通过 Raft 实现了强一致性。

最后在服务器 Server6 中 Program D 需要访问 Service B，这时候 Program D 首先访问本机 Consul Client 提供的 HTTP API，本机 Client 会将请求转发到 Consul Server，Consul Server 查询到 Service B 当前的信息返回，最终 Program D 拿到了 Service B 的所有部署的 IP 和端口，然后就可以选择 Service B 的其中一个部署并向其发起请求了。如果服务发现采用的是 DNS 方式，则 Program D 中直接使用 Service B 的服务发现域名，域名解析请求首先到达本机 DNS 代理，然后转发到本机 Consul Client，本机 Client 会将请求转发到 Consul Server，Consul Server 查询到 Service B 当前的信息返回，最终 Program D 拿到了 Service B 的某个部署的 IP 和端口。

## 0x05 Consul-Docker 部署
Consul 的 Docker 镜像基于 `alpine` 构建的，进入容器的时候需要指定 `/bin/sh`

首先拉取镜像：
```bash
docker pull consul      #拉取镜像
```

先启动第一个 Docker：

```bash
#启动第 1 个 Server 节点，集群要求要有 3 个 Server，将容器 8500 端口映射到主机 8900 端口，同时开启管理界面
docker run -d --name=consul_1 -p 8900:8500 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --bootstrap-expect=3 --client=0.0.0.0 -ui
```
使用 docker exec -ti 7efe(容器 ID)  /bin/sh 进入容器查看下容器的内网 IP:172.17.0.2，以此 IP 为 Server 节点将其他 Consul-Node 加入并构建集群：
![image](https://s2.ax1x.com/2019/10/17/KELqB9.png)

```bash
#启动第 2 个 Server 节点，并加入集群
docker run -d --name=consul_2 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.2

#启动第 3 个 Server 节点，并加入集群
docker run -d --name=consul_3 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=true --client=0.0.0.0 --join 172.17.0.2

#启动第 4 个 Client 节点，并加入集群
docker run -d --name=consul_4 -e CONSUL_BIND_INTERFACE=eth0 consul agent --server=false --client=0.0.0.0 --join 172.17.0.2
```

集群搭建完成后，在容器中执行 `consul members`，查看集群的组成信息，当前集群中，启动 `4` 个 Consul Agent，`3` 个 Server（会选举出一个 Leader），`1` 个 Client。
![image](https://s2.ax1x.com/2019/10/17/KEO5VA.png)

这些 Consul 节点在 Docker 的容器内是互通的，他们通过桥接的模式通信。但是如果主机要访问容器内的网络，需要做端口映射。在启动第一个容器时，将 Consul 的 `8500` 端口映射到了主机的 `8900` 端口，这样就可以方便的通过主机的浏览器查看集群信息。


##  0x06    Consul 的 WEBUI
根据上一步的暴露的 UI 端口 `8900`，打开主页，能看到集群中节点信息，共计 `4` 个节点。
![image](https://s2.ax1x.com/2019/10/17/KEjXAs.png)


##  0x07    Consul 服务发现测试

####    服务注册
Consul 通用的注册方式，JSON 配置文件，需要在配置文件中指定 <font color="#dd0000"> 两个重要信息，一是服务的 IP 和端口，二是健康检查的方法，尤其要注意健康检查，这个在 Consul 实现服务注册时特别重要，一旦健康检查服务失败，服务会被标记为下线 </font>。这个地方需要注意，我在另外一篇文章中详细说。<br>

在测试服务端 [server.go](https://github.com/pandaychen/grpclb2consul/blob/master/example/server.go) 中，设置 gRPC 健康检查方式为 TTL，Consul-Agent 地址设置为 `http://172.17.0.2:8500`，运行 gRPC-Server：
![image](https://s2.ax1x.com/2019/10/18/KVCUvq.png)

查看 WEB，健康检查通过，服务启动成功：
![image](https://s2.ax1x.com/2019/10/18/KVCDVU.png)

####    服务发现
在测试客户端 [client.go](https://github.com/pandaychen/grpclb2consul/blob/master/example/client.go) 中，设置 Consul-Agent 地址为 `http://172.17.0.3:8500`，注意这里和 Server 设置的不一样（当然也可以一样），运行 gRPC-Client：
![image](https://s2.ax1x.com/2019/10/18/KVCfr6.png)

##  0x08    后记
至此，一个 gRPC+Consul 的可用框架就搭建完成了。后面我还会写一篇文章，介绍 Consul 服务注册的详细方法、Consul 与 Etcd 的比较。<br>

##  0x09    参考
-   [使用Consul做服务发现的若干姿势](https://blog.didispace.com/consul-service-discovery-exp/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权


