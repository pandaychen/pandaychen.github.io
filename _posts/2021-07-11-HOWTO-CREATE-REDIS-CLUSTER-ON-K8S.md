---
layout: post
title: Kubernetes 应用改造（八）：如何在 TKE 上搭建 Redis 环境
subtitle: Statefulsets 应用场景
date: 2021-07-11
author: pandaychen
catalog: true
tags:
  - Kubernetes
  - Statefulset
  - Redis
---

## 0x00 前言

本篇文章总结下，项目背景是需要在 Kubernetes 集群中部署 Redis 单机 / 集群，也是需要利用 `Statefulsets` 来生成。

## 0x01 单机 Redis 搭建

#### 创建 ConfigMap

首先，把 `Redis` 的配置文件 `redis.conf` 存储在 `configmap` 中，注意其中定义的 `Redis` 运行目录：`/data/middleware-data/redis/`

```yaml
apiVersion: v1
data:
  redis.conf: |
    bind 0.0.0.0
    port 6379
    requirepass xxxxxx
    pidfile .pid
    appendonly yes
    cluster-config-file nodes-6379.conf
    pidfile /data/middleware-data/redis/log/redis-6379.pid
    cluster-config-file /data/middleware-data/redis/conf/redis.conf
    dir /data/middleware-data/redis/data/
    logfile "/data/middleware-data/redis/log/redis-6379.log"
    cluster-node-timeout 5000
    protected-mode no
kind: ConfigMap
metadata:
  creationTimestamp: "2021-06-27T10:01:29Z"
  name: redis-conf
  namespace: xxxx
```

#### 创建 StatefulSet

创建 `StatefulSet`，并把数据挂载到宿主机上。注意，我们使用 `initContainers` 来完成前置工作，如目录创建等；

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  replicas: 1
  serviceName: redis
  selector:
    matchLabels:
      name: redis
  template:
    metadata:
      labels:
        name: redis
    spec:
      initContainers:
        - name: init-redis
          image: busybox
          command:
            [
              "sh",
              "-c",
              "mkdir -p /data/middleware-data/redis/log/;mkdir -p /data/middleware-data/redis/conf/;mkdir -p /data/middleware-data/redis/data/",
            ]
          volumeMounts:
            - name: data
              mountPath: /data/middleware-data/redis/
      containers:
        - name: redis
          image: redis:5.0.6
          imagePullPolicy: IfNotPresent
          command:
            - sh
            - -c
            - "exec redis-server /data/middleware-data/redis/conf/redis.conf"
          ports:
            - containerPort: 6379
              name: redis
              protocol: TCP
          volumeMounts:
            - name: redis-config
              mountPath: /data/middleware-data/redis/conf/
            - name: data
              mountPath: /data/middleware-data/redis/
      volumes:
        - name: redis-config
          configMap:
            name: redis-conf
        - name: data
          hostPath:
            path: /data/middleware-data/redis/
```

#### 创建 Service

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    name: redis
  name: redis
spec:
  type: NodePort
  ports:
    - name: redis
      port: 6379
      targetPort: 6379
      nodePort: 30020
  selector:
    name: redis
```

## 0x02 Redis 集群搭建

## 0x03 参考
