---
layout:     post
title:      Envoy 应用入门（TODO）
subtitle:
date:       2023-10-03
author:     pandaychen
catalog:    true
tags:
    - Envoy
---


##  0x00    前言
Envoy 是一个开源的边缘服务代理，也是 Istio Service Mesh 默认的数据平面，专为云原生应用程序设计。与 HAProxy 以及 Nginx 等传统 Proxy 依赖静态配置文件来定义各种资源以及数据转发规则不同，Envoy 几乎所有配置都可以通过订阅来动态获取。对应的发现服务以及各种各样的 API 统称为 `xDS`。Envoy 与 `xDS` 之间通过 Proto 约定请求和响应的数据模型，不同类型资源，对应的数据模型也不同。Envoy 因其设计理念 ** 一个为大型现代服务化架构设计的七层代理和通信总线 **，成为了云原生时代七层网关的最佳选择。

####    envoy 架构
Envoy 进程中运行着一系列 Inbound/Outbound 监听器（Listener），其中 Inbound 代理入站流量，Outbound 代理出站流量。Listener 的核心就是过滤器链（FilterChain），链中每个过滤器都能够控制流量的处理流程。Envoy 接收到请求后，会先走 FilterChain，通过各种 L3/L4/L7 Filter 对请求进行处理，然后再路由到指定的集群（Cluster），并通过负载均衡获取一个目标地址，最后再转发出去。其中每一个环节可以静态配置，也可以动态服务发现（即 xDS，包含了LDS/RDS/CDS/EDS/SDS等）

![ARCH1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/envoy/envoy-proxy-architecture-diagram.png)

笔者理解，可以通过envoy配置出想要的转发模型

##  envoy 配置
Envoy 使用 `YAML` 配置来控制代理的行为，可以通过静态/动态来配置。大概结构如下：

```TEXT
listen -- 监听器
    1.监听的地址
    2.过滤链
        filter1
            路由: 转发到哪里
                virtual_hosts(route_config)
                    只转发什么
                    转发到哪里 --> 由后面的 cluster 来定义
        filter2
        filter3
        # envoyproxy.io/docs/envoy/v1.28.0/api-v3/config/filter/filter
cluster
    转发规则
    endpoints
        --指定了转到后端地址
```

当然，这只是最基础的配置，实际envoy可以有多种组合，甚至是Listener To Listener都可以

![ARCH2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/envoy/envoy-flow.png)


##  第一个例子：简单代理
本例子的功能：监听 `10000` 端口，将请求转发到 `www.baidu.com:80`，配置文件 [参考](https://github.com/pandaychen/envoy-study/blob/main/basic/envoy-1.yaml)，下面简单分析下这个yaml文件：

1、需要在静态配置下面定义 Envoy 的监听器（Listener），监听器是 Envoy 监听请求的网络配置（如 IP 地址和端口）。这里设置监听 IP 地址为 `0.0.0.0`，并在端口 `10000` 上进行监听

```YAML
static_resources:       #定义正在使用的接口配置，配置静态 API
  listeners:
    - name: listener_0 # 监听器的名称
      address:
        socket_address:
          address: 0.0.0.0 # 监听器的地址
          port_value: 10000 # 监听器的端口
```

2、通过 Envoy 监听传入的流量，下一步是定义如何处理这些请求。每个监听器都有一组过滤器（filters），并且不同的监听器可以具有一组不同的过滤器。过滤器是通过 `filter_chains` 来定义，每个过滤器的目的是找到传入请求的匹配项，以使其与目标地址进行匹配。写法如下：

```TEXT
name: 指定使用哪个过滤器
typed_config:
  "@type": type.googleapis.com/envoy.过滤器的具体值 #@type 的值到文档里找具体的值
  参数1：值1
  参数2：值2
  ......
```

一般需要解决如下问题：
-   需要实现针对哪些流量处理？选择合适的filter（HTTP 相关的，一般选择 `HTTP connection manager`，name 的位置应该写 `envoy.filters.network.http_connection_manager`）
-   需要处理何种参数？设置该filter支持的处理参数，参数选择[在此](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/filter/filter)

参考本例配置（注释）：
```yaml
static_resources:
  listeners:
    - name: listener_0 # 监听器的名称
      address:
        socket_address:
          address: 0.0.0.0 # 监听器的地址
          port_value: 10000 # 监听器的端口

      filter_chains: # 配置过滤器链
        # 在此地址收到的任何请求都会通过这一系列过滤链发送。
        - filters:
            # 指定要使用哪个过滤器，下面是envoy内置的网络过滤器，如果请求是 HTTP 它将通过此 HTTP 过滤器
            # 该过滤器将原始字节转换为HTTP级别的消息和事件（例如接收到的header、接收到的正文数据等）
            # 它还处理所有HTTP连接和请求中常见的功能，例如访问日志记录、请求ID生成和跟踪、请求/响应头操作、路由表管理和统计信息。
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                # 需要配置下面的类型，启用 http_connection_manager
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                access_log: # 连接管理器发出的 HTTP 访问日志的配置
                  - name: envoy.access_loggers.stdout # 输出到stdout
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.access_loggers.stream.v3.StdoutAccessLog
                http_filters: # 定义http过滤器链
                  - name: envoy.filters.http.router # 调用7层的路由过滤器
                    typed_config:
                      "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains: ["*"] # 要匹配的主机名列表，*表示匹配所有主机
                      routes:
                        - match:
                            prefix: "/" # 要匹配的 URL 前缀（这里为根路由）
                          route: # 路由规则，发送请求到 service_baidu 集群
                            host_rewrite_literal: www.baidu.com # 更改 HTTP 请求的入站 Host 头信息
                            cluster: service_baidu # 将要处理请求的集群名称，下面会有相应的实现
```

在上述配置中，过滤器使用了 `envoy.filters.network.http_connection_manager`，这是为 HTTP 连接设计的一个内置过滤器，它的作用将原始字节转换为 HTTP 级别的消息和事件（例如接收到的 header、接收到的正文数据等），它还处理所有 HTTP 连接和请求中常见的功能，例如访问日志记录、请求 ID 生成和跟踪、请求/响应头操作、路由表管理和统计信息，该过滤器有如下常用参数：

-   `stat_prefix`：为连接管理器发出统计信息时使用的一个前缀
-   `route_config`：路由配置，如果虚拟主机匹配上了则检查路由。在配置中，无论请求的主机域名是什么，`route_config` 都匹配所有传入的 HTTP 请求
-   `routes`：如果 URL 前缀匹配，则一组路由规则定义了下一步将发生的状况 
-   `host_rewrite_literal`：更改 HTTP 请求的入站 `Host` 头信息
-   `cluster`: 将要处理请求的集群名称，下面会有相应的实现
-   `http_filters`: 该过滤器允许 Envoy 在处理请求时去适应和修改请求

请求在filter匹配完成后，最后会到达 Clusters，本例中将主机定义为访问 HTTPS 的 `www.baidu.com` 域名，如果定义了多个主机，则 Envoy 将执行轮询（Round Robin）策略

```YAML
clusters:
  - name: service_baidu # 集群的名称，与上面的 router 中的 cluster 对应
    type: LOGICAL_DNS # 用于解析集群（生成集群端点）时使用的服务发现类型，可用值有STATIC、STRICT_DNS 、LOGICAL_DNS、ORIGINAL_DST和EDS等
    connect_timeout: 0.25s
    dns_lookup_family: V4_ONLY
    lb_policy: ROUND_ROBIN # 负载均衡算法，支持ROUND_ROBIN、LEAST_REQUEST、RING_HASH、RANDOM、MAGLEV和CLUSTER_PROVIDED；
    load_assignment: # 以前的 v2 版本的 hosts 字段废弃了，现在使用 load_assignment 来定义集群的成员，指定 STATIC、STRICT_DNS 或 LOGICAL_DNS 集群的成员需要设置此项。
      cluster_name: service_baidu # 集群的名称
      endpoints: # 需要进行负载均衡的端点列表
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: www.baidu.com
                    port_value: 443
    transport_socket: # 用于与上游集群通信的传输层配置
      name: envoy.transport_sockets.tls # tls 传输层
      typed_config:
        "@type": type.googleapis.com/envoy.extensions.transport_sockets.tls.v3.UpstreamTlsContext
        sni: www.baidu.com
```


完整配置文件 [参考](https://github.com/pandaychen/envoy-study/blob/main/basic/envoy-1.yaml)

####    部署此例子

```BASH
docker logs -f  --tail=200 cfb4ba22c969
```

####    配置流程解读
如何容易理解envoy配置？还是结合代理程序的流程来思考，作为一个代理，首先要能获取请求流量，通常是采用监听端口的方式实现；其次拿到请求数据后需要对其做处理，例如添加 `Header` 或校验某个 `Header` 字段的内容等，这里针对来源数据的层次不同，又可以分为 L3/L4/L7，然后将请求转发出去；转发这里又可以衍生出如果后端是一个集群，需要从中挑选一台机器，如何挑选又涉及到负载均衡等；所以，回看上面的yaml配置：

-   `listener`：Envoy 的监听地址，真正干活的。Envoy 会暴露一个或多个 Listener 来监听客户端的请求
-   `filter`: 过滤器：在 Envoy 中指的是一些可插拔和可组合的逻辑处理层，是 Envoy 核心逻辑处理单元
-   `route_config`：路由规则配置，即将请求路由到后端的哪个集群
-   `cluster`：服务提供方集群，Envoy 通过服务发现定位集群成员并获取服务，具体路由到哪个集群成员由负载均衡策略决定

![flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/envoy/envoy-config-flow.jpg)



##  第个例子：使用 SSL/TLS 保护流量


##  Nginx 迁移到 Envoy

##  其他例子
[](https://github.com/envoyproxy/envoy/tree/main/examples)

##  0x 参考
-   [Envoy 架构及其在网易轻舟的落地实践](https://mp.weixin.qq.com/s?src=11&timestamp=1701179881&ver=4924&signature=r4fQN4Dk3Zuce3C1ojExLYSF-G0a1j20ogEqbD7ZACIiyi8LCYh0ZtwImTjC7hWkqTo2BtMQ69aLJbH7Mwkwj297AgS2cmLSN6OzeCn5Kjbz2JtZKRva6wIY3AuDfiKF&new=1)
-   [Envoy 的架构与基本配置解析](https://jimmysong.io/blog/envoy-archiecture-and-terminology/)
-   [](https://academy.tetrate.io/courses/envoy-fundamentals)
-   [Envoy 架构理解 -- 理解 xDS/Listener/Cluster/Router/Filter](https://blog.csdn.net/gengzhikui1992/article/details/117449972)
-   [Envoy 入门教程](https://www.qikqiak.com/envoy-book/)
-   [envoy proxy 调研笔记](https://hxysayhi.com/blog/posts/4a1349e1/)
-   [动态转发代理](https://cloudnative.to/envoy/configuration/http/http_filters/dynamic_forward_proxy_filter.html)
-   [Envoy 简单入门示例](https://www.qikqiak.com/post/envoy-usage-demo/)
-   [为什么 Envoy Gateway 是云原生时代的七层网关？](https://www.zhaohuabing.com/post/2023-04-11-why-eg-is-the-gateway-in-cloud-native-era/)
-   [Envoy Gateway 指南：架构设计与开源贡献](https://mp.weixin.qq.com/s/XPgP47eb40JJD96cN_gyWQ)
-   [Envoy Gateway 首个正式开源版本介绍](https://jimmysong.io/blog/envoy-gateway-release/)
-   [Envoy Gateway 指南：架构设计与开源贡献](https://mp.weixin.qq.com/s/XPgP47eb40JJD96cN_gyWQ)