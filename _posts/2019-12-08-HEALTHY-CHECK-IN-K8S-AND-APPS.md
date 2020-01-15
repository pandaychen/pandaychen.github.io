---
layout:     post
title:      Kubernetes应用改造（三）
subtitle:   健康检查与探针的日常应用和封装
date:       2019-12-08
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
    - 健康检查
---

##	Consul的健康检查
https://github.com/hashicorp/consul/blob/master/agent/checks/grpc.go

## 0x00 Kubernetes服务的健康检查机制

## 就绪探针(Readiness)

kubelet 使用 readiness probe（就绪探针）来确定容器是否已经就绪可以接受流量。只有当 Pod 中的容器都处于就绪状态时 kubelet 才会认定该 Pod处于就绪状态。该信号的作用是控制哪些 Pod应该作为service的后端。如果 Pod 处于非就绪状态，那么它们将会被从 service 的 load balancer中移除。


## 存活探针(Liveness)


kubelet 使用 liveness probe（存活探针）来确定何时重启容器。例如，当应用程序处于运行状态但无法做进一步操作，liveness 探针将捕获到 deadlock，重启处于该状态下的容器，使应用程序在存在 bug 的情况下依然能够继续运行下去。

 
## YAML配置livenessProbe

```
livenessProbe:
  exec:
    command:
    - /root/client
    - -a
    - 127.0.0.1:19000
  initialDelaySeconds: 2
  periodSeconds: 2

k8s会每2秒执行 /root/client -a 127.0.0.1:19000, 执行成功的话代表存活,不成功的话k8s会重启pod
```


## DockerFile

```
FROM centos:7
ADD server /root/server
ADD client /root/client
RUN chmod +x /root/server \
        && chmod +x /root/client
EXPOSE 11111
CMD /root/server -p 11111
```

## 镜像构建
```
docker build -t rpccheck:v1  .
```
