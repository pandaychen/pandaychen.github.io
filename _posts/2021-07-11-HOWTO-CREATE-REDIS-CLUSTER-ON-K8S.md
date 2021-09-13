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

本篇文章总结下，项目背景是需要在 Kubernetes 集群中部署 Redis 单机 / 集群，需要利用 `Statefulsets` 与 `NFS` 来生成。

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
apiVersion: apps/v1beta2
kind: StatefulSet
metadata:
  creationTimestamp: "2021-06-27T10:03:08Z"
  generation: 1
  name: redis
  namespace: xxxx
spec:
  podManagementPolicy: OrderedReady
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      name: redis
  serviceName: redis
  template:
    metadata:
      creationTimestamp: null
      labels:
        name: redis
    spec:
      containers:
        - command:
            - sh
            - -c
            - exec redis-server /data/middleware-data/redis/conf/redis.conf
          image: redis:5.0.6
          imagePullPolicy: IfNotPresent
          name: redis
          ports:
            - containerPort: 6379
              name: redis
              protocol: TCP
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /data/middleware-data/redis/conf/
              name: redis-config
            - mountPath: /data/middleware-data/redis/
              name: data
      dnsPolicy: ClusterFirst
      initContainers:
        - command:
            - sh
            - -c
            - mkdir -p /data/middleware-data/redis/log/;mkdir -p /data/middleware-data/redis/conf/;mkdir -p /data/middleware-data/redis/data/
          image: busybox
          imagePullPolicy: Always
          name: init-redis
          resources: {}
          terminationMessagePath: /dev/termination-log
          terminationMessagePolicy: File
          volumeMounts:
            - mountPath: /data/middleware-data/redis/
              name: data
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
      volumes:
        - configMap:
            defaultMode: 420
            name: redis-conf
          name: redis-config
        - hostPath:
            path: /data/middleware-data/redis/
            type: ""
          name: data
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate
```

#### 创建 Service

```yaml
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: "2021-06-27T10:03:44Z"
  labels:
    name: redis
  name: redis
  namespace: xxxx
spec:
  clusterIP: 172.16.255.66
  externalTrafficPolicy: Cluster
  ports:
    - name: redis
      nodePort: 30020
      port: 6379
      protocol: TCP
      targetPort: 6379
  selector:
    name: redis
  sessionAffinity: None
  type: NodePort
```

## 0x02 Redis 集群搭建

####  第一步：创建 NFS 存储
####  创建 PV
####  创建 PVC

使用 NFS 配置 StatefulSet 的动态持久化存储

####  创建 Configmap
####  创建 Headless Service
####  创建 Redis StatefulSet

## 0x03 参考
