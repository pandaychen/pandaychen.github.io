---
layout:     post
title:      Kubernetes 应用改造（二）：健康检查
subtitle:   健康检查与探针的日常应用和封装
date:       2019-12-08
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
---

##  0x00	Kubernetes 中的探针
&emsp;&emsp; Kubernetes 服务的 <font color="#dd0000"> 主动健康检查机制 </font> 是通过健康探针来实现的。Kubernetes 提供了两种探针：

#### 就绪探针 (Readiness Probe)
&emsp;&emsp; 在 Pod 启动时，Kubelet 使用 Readiness probe（就绪探针）来确定容器是否已经就绪可以接受流量。只有当 Pod 中的容器都处于就绪状态时 Kubelet 才会认定该 Pod 处于就绪状态。该信号的作用是控制哪些 Pod 应该作为 service 的后端。如果 Pod 处于非就绪状态，那么它们将会被从 Service 的 Load balancer 中移除

#### 存活探针 (Liveness Probe)
&emsp;&emsp; 在 Pod 运行中，Kubelet 使用 Liveness probe（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，Liveness 探针将捕获到 Deadlock，重启处于该状态下的容器，使应用程序在存在 bug 的情况下依然能够继续运行下去
<br>

####	就绪探针的意义
很常见的一个场景，某个服务刚启动，需要做一系列的初始化工作（比如初始化 Mysql/Redis/Kafka 等），需要一些时间，而这段时间里，不希望 Kubernetes 把流量导入到这些还未初始化完成的 Pod 上。所以就绪探针非常适合这种场景，只有 Pod 就绪后 Kubernetes 才会把请求打过来，如果非就绪状态，这些 Pod 会从 Service 的 Load balancer 中短暂移除

日常项目服务中的就绪探测功能通常用 CGI 接口来实现（当然也可以用命令工具），如下所示一个 Readiness Probe（就绪探针）的一般实现：<br>
在服务端，一般以一个单独的 goroutine 启动 CGI Server，实现 `/ready` 的逻辑，在 `/ready` 的处理逻辑中加入对初始化的检查，如检查 Mysql 是否建立连接成功、Redis 数据查询是否可用等等。
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

####  优雅关闭
这里我们先简单介绍下 Kubernetes 如何支持服务的优雅退出，优雅关闭是指在 Pod 准备关闭时，可能还需要做一些处理，比如保存数据等。这期间服务不会接受新的请求。Kubernetes 提供了 `terminationGracePeriodSeconds` 的配置选项：在给 Pod 发出关闭指令时，Kubernetes 将会给应用发送 SIGTERM 信号，程序只需要捕获 SIGTERM 信号并做相应处理即可。配置为 Kubernetes 会等待 `terminationGracePeriodSeconds` 秒后关闭。

```yaml
terminationGracePeriodSeconds: 60
```

##  0x02  Kubernetes 中的优雅退出
本小节，详细介绍下 Kubernetes 中的 Pod 优雅退出的步骤。<br>

Kubernetes 终止一个完全健康的容器有很多原因，常见的有：
1.  滚动更新更新部署时，Kubernetes 会在启动新 Pod 时慢慢终止旧 Pod。
2.  节点驱逐，Kubernetes 将终止该节点上的所有 Pod，如果节点资源不足，Kubernetes 将终止 Pod 以释放这些资源

优雅退出，是 Kubernetes 在完全清理掉一个 Pod 之前，会发送信号通知 Pod 中的进程，并且预留了一个等待期，在等待期之后，Kubernetes 会完全强制删除该 Pod。在这段时间，我们的服务可以实现优雅退出，举例来说：不再接受新连接，有序释放掉已存在的连接等，同时可能需要保存相关的数据，关闭网络连接，完成剩下的收尾工作。

facebook 开源的库 [grace](https://github.com/facebookarchive/grace)，可以实现 httpserver 的优雅退出。

一般而言，Kubernetes 清除一个 Pod 需要走完下面五部曲：

1.  Pod 被设置为 <font color="#dd0000"> 终止 </font> 状态，并从所有服务的端点列表中删除，此时，Pod 停止获得新的流量。在 Pod 中运行的容器不会受到影响

2.  `preStop Hook` 被执行：`preStop Hook` 是一个特殊命令或 http 请求，发送到 pod 中的容器 <br>
如果应用程序在接收 `SIGTERM` 时没有正常关闭，可以使用此 Hook 来触发正常关闭。 接收 `SIGTERM` 时大多数程序都会正常关闭，但是如果使用的是第三方代码或管理系统则无法控制，所以 `preStop Hook` 是在不修改应用程序的情况下触发正常关闭的好方法

3. `SIGTERM` 信号被发送到 Pod
此时，Kubernetes 将向 Pod 中的容器发送 `SIGTERM` 信号，这个信号让容器知道它们很快就会被关闭。这里很重要，在我们的代码应该监听此事件并在此时开始执行收尾工作、清理资源及优雅退出

4. Kubernetes 优雅等待期
此时，Kubernetes 等待指定的时间称为优雅终止等待期。 默认情况为 `30` 秒（可以视程序的逻辑处理适当延长）。 值得注意的是，这与 `preStop Hook` 和 `SIGTERM` 信号并行发生。 Kubernetes 不会等待 `preStop Hook` 完成。如果应用程序完成关闭并在 `terminationGracePeriod` 完成之前退出，Kubernetes 会立即进入下一步

5. `SIGKILL` 信号被发送到 Pod，并删除 Pod
如果容器在优雅等待期结束后仍在运行，则会发送 `SIGKILL` 信号并强制删除。 此时，所有 Kubernetes 对象也会被清除，至此，Kubernetes 清理 Pod 的整个流程结束

##  0x02  Kubernetes 中的 gRPC 服务

####  gRPC 的健康检查协议
&emsp;&emsp;gRPC 中提供了健康检查的协议，[文档在此](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)。所以，RPC 服务需要实现 `Check` 和 `Watch` 这两个方法。官方文档建议的实现方式如下：
> The suggested format of service name is package_names.ServiceName, such as grpc.health.v1.Health.

健康检查的协议如下：
```protobuf
syntax = "proto3";

package grpc.health.v1;

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
  }
  ServingStatus status = 1;
}

service Health {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);

  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

####  gRPC 健康检查 demo


##  0x04  参考文档
- [GRPC Health Checking Protocol](https://github.com/grpc/grpc/blob/master/doc/health-checking.md)
- [A command-line tool to perform health-checks for gRPC applications in Kubernetes etc.](https://github.com/grpc-ecosystem/grpc-health-probe)
- [LIVENESS PROBES ARE DANGEROUS](https://srcco.de/posts/kubernetes-liveness-probes-are-dangerous.html)
- [Health checking gRPC servers on Kubernetes](https://kubernetes.io/blog/2018/10/01/health-checking-grpc-servers-on-kubernetes/)