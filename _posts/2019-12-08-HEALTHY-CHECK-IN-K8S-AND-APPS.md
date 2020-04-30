---
layout:     post
title:      Kubernetes 应用改造（二）-- 健康检查
subtitle:   健康检查与探针的日常应用和封装
date:       2019-12-08
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
---

##  0x00	前言
&emsp;&emsp; Kubernetes 服务的 ** 主动 ** 健康检查机制是通过健康探针来实现的。Kubernetes 提供了两种探针：

#### 就绪探针 (Readiness Probe)
&emsp;&emsp; 在 Pod 启动时，Kubelet 使用 Readiness probe（就绪探针）来确定容器是否已经就绪可以接受流量。只有当 Pod 中的容器都处于就绪状态时 Kubelet 才会认定该 Pod 处于就绪状态。该信号的作用是控制哪些 Pod 应该作为 service 的后端。如果 Pod 处于非就绪状态，那么它们将会被从 Service 的 Load balancer 中移除

#### 存活探针 (Liveness Probe)
&emsp;&emsp; 在 Pod 运行中，Kubelet 使用 Liveness probe（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，Liveness 探针将捕获到 Deadlock，重启处于该状态下的容器，使应用程序在存在 bug 的情况下依然能够继续运行下去
<br>

####	就绪探针的意义
很常见的一个场景，某个服务刚启动，需要做一系列的初始化工作（比如初始化 Mysql/Redis/Kafka 等），需要一些时间，而这段时间里，不希望 kubernetes 把流量导入到这些还未初始化完成的 Pod 上。所以就绪探针非常适合这种场景，只有 Pod 就绪后 Kubernetes 才会把请求打过来，如果非就绪状态，这些 Pod 会从 Service 的 Load balancer 中短暂移除

就绪探测常用 Http-CGI 接口来实现（当然也可以用命令），如下所示一个 readiness probe（就绪探针）的一般配置：<br>
在服务端，一般以一个单独的 groutine 启动 CGIserver，实现 `/ready` 的逻辑，在逻辑中加入对初始化的检查，如检查 Mysql 是否建立连接成功、Redis 数据查询是否可用等等。
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8001
    scheme: HTTP
  initialDelaySeconds: 30
  timeoutSeconds: 1
  periodSeconds: 30
```

##  0x04  参考文档
- [GRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)
- [A command-line tool to perform health-checks for gRPC applications in Kubernetes etc.](https://github.com/grpc-ecosystem/grpc-health-probe)