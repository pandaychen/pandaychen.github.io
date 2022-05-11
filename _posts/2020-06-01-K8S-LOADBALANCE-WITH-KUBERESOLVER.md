---
layout:     post
title:      Kubernetes 应用改造（三）：负载均衡
subtitle:   Kubernetes 中的负载均衡总结
date:       2020-06-01
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
    - 负载均衡
---


##  0x00    前言
&emsp;&emsp; 在前文中，曾经讨论过 "Kubernetes 负载均衡失效" 的问题。即 Kubernetes 的默认负载平衡通常不能与 gRPC 一起使用，在不使用 LoadBalance service 的情况下，因为 HTTP/2 链接复用特性，导致客户端的所有请求都发往同一个 Pod，导致负载不均衡。具体原因可见此文：[gRPC Load Balancing on Kubernetes without Tears](https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/)，原文中对 gRPC 失效的原因的摘抄如下：

>   However, gRPC also breaks the standard connection-level load balancing, including what’s provided by Kubernetes.
>   This is because gRPC is built on HTTP/2, and HTTP/2 is designed to have a single long-lived TCP connection, across which all requests are multiplexed—meaning multiple requests can be active on the same connection at any point in time.
>   Normally, this is great, as it reduces the overhead of connection management.
>   However, it also means that (as you might imagine) connection-level balancing isn’t very useful.
>   Once the connection is established, there’s no more balancing to be done. All requests will get pinned to a single destination pod ...

问题在于 gRPC 是基于 HTTP/2 长连接的实现，gRPC 客户端与服务端建立长连接后会保持并通过这个连接持续的发送请求。而 Kubernetes Service 属于基于 TCP 连接级别的负载均衡（Connection-level Balancing），虽然支持 gRPC 通信，但其负载均衡的作用却会失效。

####  解决办法

gRPC 官方博客的文章 [gRPC Load Balancing](https://grpc.io/blog/grpc-load-balancing/) 给出了 gRPC 负载均衡的思路：

1.  客户端型负载均衡（ZooKeeper/Etcd/Consul/DNS 等），适用于流量较高的场景
2.  代理式的负载均衡（Haproxy/Nginx/Traefik/Envoy/Linkerd 等），适用于服务器需要对外提供统一的入口场景，通常我们说的 ServiceMesh 就是这种方式

此外，这里汇总了目前主流的 [Ingress 网关](https://docs.google.com/spreadsheets/d/16bxRgpO1H_Bn-5xVZ1WrR_I-0A-GOI6egmhvqqLMOmg/edit#gid=1351816353)，可以用作参考。

##  0x01    DnsResolver + Headless-Service
&emsp;&emsp; 这是我们项目最开始使用的 LB 解析方案，也是 TKE 集群上默认自带的服务发现方式。Headless-Service 可以将 Service 的 Endpoint 直接更新到 kubernetes 的 coreDNS 中，所以客户端只要使用 coreDNS 作为名字解析，就能获取下游所有的 Endpoint(s)，然后客户端通过传入的 LB 算法来实现负载均衡策略。

![coredns.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/k8s-lb-headless-service.png)

##  0x02  DNS 解析的缺陷
&emsp;&emsp; 在使用 gRPC+DnsResolver+HeadlessService 实现 Kubernetes 负载均衡方案时，有个明显的缺陷就是，DnsResolver 是定时触发主动去解析获取当前 ServiceName 对应的 Pod 列表的，如果在两次解析的间隔期间，某个 Pod 发生了重启或者漂移（导致 IP 改变），那么这种情况对我们的解析器 DNSResolver 是无法立即感知到的。

设想一下，有没有一种方法，可以使得我们像使用 Etcd 的 Watch 方法那样，可以监听在某个事件（在 Kubernetes 环境为 Pod）的增删上面呢？翻阅下 Kubernetes 的手册，发现提供查询的 API`/api/v1/watch/namespaces/{namespace}/endpoints/${service}`，这样使得我们可以主动监听某个 Service 下面的 Podlist 变化。

这种方案就是 Watch Endpoint 方式，该方案主要在客户端实现负载均衡，通过 kubernetes API 获取 Service 的 Endpont。**客户端和每个 POD 保持一个长连接**，然后使用 gRPC Client 的负载均衡策略解决问题。开源项目 [kuberesolver](https://github.com/sercand/kuberesolver) 已经实现了这种方式。在下面的篇幅中会简单分析下该项目的代码。

![kuberesolver.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/k8s-lb-watcher-endpoint.png)

##  0x03  代理方案之 Proxy1 - Linkerd
&emsp;&emsp; Linkerd 是一种基于 CNCF 的 Kubernetes 服务网格，它通过 Sidecar 的方式来实现负载均衡。Linkerd 作为 Service Sidecar 部署在 Pod 中，自动检测 HTTP/2 和 HTTP/1.x 并执行 L7 负载平衡。为服务添加 Linkerd，等价于为每个 Pod 添加了一个很小并且很高效的代理，由这些代理来监控 Kubernetes 的 API，并自动实现 gRPC 的负载均衡。<br>

Linkerd 通过 webhook，在有 Pod 新增的时候，会向 Pod 中注入额外的 Container，以此获取所有的 Pod。当然作为 service mesh，拥有更多的 feature 保证服务之间的调用，如监控、请求状态、Controller 等等。

![linkerd.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/k8s-lb-linkerd.png)

##  0x04  代理方案之 Proxy2 - Nginx
Nginx 1.13.10 版本支持了 HTTP/2 的负载均衡，[Introducing gRPC Support with NGINX 1.13.10](https://www.nginx.com/blog/nginx-1-13-10-grpc/)

它的原理是：gRPC 客户端与 Nginx 建立 HTTP/2 连接后，Nginx 再与后端 server 建立多个 Http/2 连接。

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/gRPC-nginx-routing.png)

Nginx gRPC 负载均衡的一般配置如下：gRPC 代理要用 `grpc_pass` 指令，协议要用 `grpc://`，同时 `upstream` 机制对 gRPC 也是同样适用的。

```bash
worker_processes  4;     # cpu core

error_log  logs/error.log;

pid        /run/nginx.pid;

events {
    worker_connections  1o24; # max connection =  worker_processes * worker_connections
#    multi_accept on;
#    use epoll;
}


http {
    log_format  main  '$remote_addr - $remote_user [$time_local]"$request" '
                      '$status $body_bytes_sent"$http_referer" '
                      '"$http_user_agent"';

    upstream grpc_servers {
        server 1.1.1.1:50051;
        server 2.2.2.2:50051;
    }

    server {
        listen 30051 http2;

        access_log logs/access.log main;

        location / {
            grpc_pass grpc://grpc_servers;
        }
    }
}
```

##  0x05  代理方案之 Proxy3 - Istio（Envoy）
&emsp;&emsp; 同 Nginx 一样，Envoy 也是非常优秀的 Ingress，都是支持 HTTP/2 的 Load Balancer。Envoy 支持多种 [不同的服务发现机制](https://www.servicemesher.com/envoy/intro/arch_overview/service_discovery.html)。

在基于 Istio+Envoy 实现的服务网格中，Istio 的角色是控制平面，它是实现了 Envoy 的发现协议集 xDS 的管理服务器端。Envoy 本身则作为网格的数据平面，和 Istio 通信，获得各种资源的配置并更新自身的代理规则。这里以 Istio 配置 gRPC 负载均衡为例：

![envoy.png](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/loadbalance/k8s-lb-envoy.png)

1、配置一个常规的 Kubernetes Service，作为 Backend：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: grpc-helloworld
  namespace: grpc-example
spec:
  externalIPs:
  - 172.30.10.1
  - 172.30.10.2
  ports:
  - name: grpc
    port: 50051
    protocol: TCP
    targetPort: 50051
  selector:
    app: grpc-helloworld
  sessionAffinity: None
  type: ClusterIP
```

2、配置一个 Istio Gateway，关联 `ingressgateway` 的 80 端口：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grpc-helloworld-gateway
  namespace: grpc-example
  uid: e5613eb1-fecb-11e8-854e-0050568156a5
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: grpc
      number: 80
      protocol: HTTP
```

3、配置一个 Istio VirtualService，将 Istio Gateway 与 Kubernetes service 关联：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grpc-helloworld
  namespace: grpc-example
spec:
  gateways:
  - grpc-helloworld-gateway
  hosts:
  - '*'
  http:
  - match:
    - uri:
        regex: .*
    route:
    - destination:
        host: grpc-helloworld
        port:
          number: 50051
```

##  0x06  总结
本文分析了 gRPC+Kubernetes 环境的负载均衡的问题，以及对应的解决方案。从当前云原生的发展来看，个人比较看好 Istio。

##  0x07  参考
-   [kuberesolver 的实现](https://github.com/sercand/kuberesolver)
-   [gRPC Load Balancing](https://grpc.io/blog/grpc-load-balancing/)
-   [Ingress](https://kubedex.com/ingress/)
-   [A Kubernetes Service Mesh (Part 9): gRPC for Fun and Profit](https://dzone.com/articles/a-service-mesh-for-kubernetes-part-ix-grpc-for-fun)
-   [必读！Istio Service Mesh 中的流量管理概念解析](https://juejin.im/post/5b7fc952f265da4358368734)
