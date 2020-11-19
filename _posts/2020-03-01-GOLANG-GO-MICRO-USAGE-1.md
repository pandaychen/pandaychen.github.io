---
layout:     post
title:      Go-Micro 微服务框架使用入门
subtitle:   Go-Mirco 框架之基础
date:       2020-03-01
author:     pandaychen
header-img: img/golang-tools-fun.png
catalog: true
category:   false
tags:
    - GoMicro
    - 微服务框架
---


##	0x00    介绍
[Go-Micro](https://github.com/micro/micro)：是一个 Pure Golang 的微服务开发框架（旧版本 [nitro](https://github.com/asim/nitro)）：官方介绍如下
> Micro is built as a microservices architecture and abstracts away the complexity of the underlying infrastructure. We compose this as a single logical server to the user but decompose that into the various building block primitives that can be plugged into any underlying system.

下面是目前 3.0 版本的架构图：
![micro-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gomicro/micro-3.0.png)

服务端支持如下特性，基本已经涵盖了微服务组件的各个方面：

- **API** - HTTP Gateway which dynamically maps http/json requests to RPC using path based resolution
- **Auth** - Authentication and authorization out of the box using jwt tokens and rule based access control.
- **Broker** - Ephemeral pubsub messaging for asynchronous communication and distributing notifications
- **Config** - Dynamic configuration and secrets management for service level config without the need to restart
- **Events** - Event streaming with ordered messaging, replay from offsets and persistent storage
- **Network** - Inter-service networking, isolation and routing plane for all internal request traffic
- **Proxy** - gRPC identity aware proxy used for remote access and any external grpc request traffic
- **Runtime** - Service lifecyle and process management with support for source to running auto build
- **Registry** - Centralised service discovery and API endpoint explorer with feature rich metadata
- **Store** - Key-Value storage with TTL expiry and persistent crud to keep microservices stateless

Server 的核心模块，服务注册 / 负载均衡 / 传输方法 / 消息队列 / 消息编码解码 / 服务端封装 / 客户端封装：
1、Registry<br>
提供一套服务注册、发现、注销、监测机制，服务注册中心支持 consul、etcd2/3、zookeeper、gossip、k8s、eureka 等

2、Selector<br>
选择器（主要针对客户端）提供了负载均衡，可以通过过滤方法对微服务进行过滤，并通过不同路由算法选择微服务，以及缓存等

3、Transport<br>
微服务间同步请求 / 响应通信方式，相对 Go 标准 net 包做了更高的抽象，支持更多的传输方式，如 http、gRPC、tcp、udp、Rabbitmq 等

4、Broker<br>
微服务间异步发布 / 订阅通信方式，更好的处理分布式系统解耦问题，默认使用 http 方式，生产环境通常会使用消息中间件，如 Kafka、RabbitMQ、NSQ 等

5、Codec<br>
服务间消息的编解码，支持 json、protobuf、bson、msgpack 等，与普通编码格式不同都是支持 RPC 格式

6、Server<br>
用于启动服务，为服务命名、注册 Handler、添加中间件等

7、Client<br>
提供微服务客户端，通过 Registry、Selector、Transport、Broker 实现以服务名来查找服务、负载均衡、同步通信、异步消息等

####    使用场景
1、普通方式 <br>
![micro-normal-1]()

2、微服务之间通信，两个微服务之间的通信是基于 C/S 模型，即服务发请求方充当 Client，服务接收方充当 Server。其通信过程大致如下图：
![micro-normal-2]()

##  0x02    GoMicro 的项目架构
![gomicro-architecture-application]()

##  0x03    项目使用

##	0x04    参考
-	[Write a Microservice via Go-Micro](https://www.slll.info/archives/2834.html)
-   [Writing microservices with Go Micro](https://medium.com/microhq/writing-microservices-with-go-micro-b6e0880b8829)
