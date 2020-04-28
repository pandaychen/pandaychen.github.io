---
layout:     post
title:      微服务项目阶段小结（2020-05-01）
subtitle:
date:       2020-05-01
author:     pandaychen
header-img: 
catalog: true
category:   false
tags:
    - 微服务
---


##  0x00    开发语言 + 框架
-   Golang
-   RPC：[gRPC-go](https://github.com/grpc/grpc-go)
-   Web：[go-gin](https://github.com/gin-gonic/gin) 或其他通用的 Web 框架
-   微服务框架：[Kratos](https://github.com/go-kratos/kratos)（强烈推荐）、[Micro](https://github.com/micro/go-micro)、[go-chassis](https://github.com/go-chassis/go-chassis)

##  0x01    微服务划分
&emsp;&emsp; 在微服务实践中，如何切分微服务，总结了一些原则如下：

微服务化：
-   逻辑独立、边界清晰的模块作为一个独立的微服务
-   微服务架构中，服务的注册与发现是必不可少的
-   微服务间的同步调用，尽量使用 RPC 方式，采用 Service discovery 方式进行，服务注册中心推荐 etcd 或 consul
-   微服务之间的通信，尽可能采用消息队列实现松耦合，当需要同步调用时再借助于 RPC


存储 && 缓存：
-   每个 table 尽量保证只由一个微服务操作（包括插入、读取、更改、删除等）
-   table 之间不引入外键约束，id 字段全部采用 uuid，id 只做 primary key 使用
-   将需要保持数据一致性的操作放在一个微服务中，避免跨服务带来的数据一致性问题

部署：
-   微服务以独立的容器化部署

##  0x02    微服务模块
下面这张图，基本涵盖了微服务开发上线过程中的知识点：
![image](https://wx1.sbimg.cn/2020/04/28/_20200428191144.png)

##  0x03  参考
-   [Go 微服务实践](https://yushuangqi.com/blog/2017/go-wei-fu-wu-shi-jian.html#%E5%85%B6%E5%AE%83%E7%BB%8F%E9%AA%8C)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权