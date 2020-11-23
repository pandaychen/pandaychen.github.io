---
layout:     post
title:      Kubernetes 应用改造（五）：使用 NFS + Statefulsets 搭建 Etcd 集群
subtitle:   Statefulsets 应用 && PV/PVC 介绍
date:       2020-10-11
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
    - Statefulset
    - Etcd
    - NFS
---

##  0x00  前言
本篇文章总结下，项目背景是需要在 Kubernetes 集群中使用 Etcd 的分布式锁，所以需要将 Etcd 集群部署在 `Kubernetes` 上。涉及到如下知识点：
1.  `Statefulsets`：有状态服务
2.  `PV`、`PVC`
3.  `NFS`：网络（共享）文件系统
4.  `Pod` 共享数据（持久化）存储

##  0x01 Kubernetes 的 StatefulSets
[`StatefulSets`](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/) 是为了有状态的应用和分布式系统设计的，核心就是稳定 </font>，`StatefulSets` 的特点如下：

1.  `Pod` 一致性：包含次序（启动、停止次序）、网络一致性。此一致性与 `Pod` 相关，与被调度到哪个 `Node` 节点无关
2.  稳定的次序：对于 `N` 个副本的 `StatefulSet`，每个 `Pod` 都在 `[0，N)` 的范围内分配一个数字序号，且是唯一的
3.  稳定的网络：`Pod` 的 `hostname` 模式为 `$(statefulset 名称)-$(序号)`
4.  稳定（持久）的存储：通过 `VolumeClaimTemplate` 为每个 `Pod` 创建一个 `PV`。删除、减少副本，不会删除相关的卷

##  0x02  PV Vs PVC

- `PV`：PersistentVolume，集群级别的资源，由 集群管理员 or External Provisioner 创建。`PV` 的生命周期独立于使用 `PV` 的 `Pod`，`PV` 的 `.Spec` 中保存了存储设备的详细信息
- `PVC`：PersistentVolumeClaim，命名空间（Namespace）级别的资源，由 用户 or `StatefulSet` 控制器（根据 `VolumeClaimTemplate`） 创建。`PVC` 类似于 `Pod`，`Pod` 消耗 `Node` 资源，`PVC` 消耗 `PV` 资源。`Pod` 可以请求特定级别的资源（CPU 和内存），而 `PVC` 可以请求特定存储卷的大小及访问模式（Access Mode）；
- `StorageClass`：StorageClass 是集群级别的资源，由集群管理员创建。：StorageClass 为管理员提供了一种动态提供存储卷的类模板，：StorageClass 中的 `.Spec` 中详细定义了存储卷 `PV` 的不同服务质量级别、备份策略等等


##  0x04  Etcd 部署
通过如下配置文件，在 Tencent Kubernetes 上部署了一个可用的 Etcd 集群，注意几点：
1.  本配置文件部署为一个 `CLUSTER_SIZE=3` 的集群
2.  数据目录的设定 `--data-dir /var/run/etcd/${IP}/default.etcd`，采用独立的 IP 在 `NFS` 上分隔

```yaml
apiVersion: v1
kind: Service
metadata:
  name: "etcd"
  annotations:
    # Create endpoints also if the related pod isn't ready
    service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
spec:
  ports:
  - port: 2379
    name: client
  - port: 2380
    name: peer
  clusterIP: None
  selector:
    component: "etcd"
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: "etcd"
  labels:
    component: "etcd"
spec:
  serviceName: "etcd"
  # changing replicas value will require a manual etcdctl member remove/add
  # command (remove before decreasing and add after increasing)
  replicas: 3
  template:
    metadata:
      name: "etcd"
      labels:
        component: "etcd"
    spec:
      containers:
      - name: "etcd"
        image: "quay.io/coreos/etcd:v3.2.3"
        ports:
        - containerPort: 2379
          name: client
        - containerPort: 2380
          name: peer
        env:
        - name: CLUSTER_SIZE
          value: "3"
        - name: SET_NAME
          value: "etcd"
        volumeMounts:
        - name: data
          mountPath: /var/run/etcd
        command:
          - "/bin/sh"
          - "-ecx"
          - |
            IP=$(hostname -i)
            for i in $(seq 0 $((${CLUSTER_SIZE} - 1))); do
              while true; do
                echo "Waiting for ${SET_NAME}-${i}.${SET_NAME} to come up"
                ping -W 1 -c 1 ${SET_NAME}-${i}.${SET_NAME}.default.svc.cluster.local > /dev/null && break
                sleep 1s
              done
            done
            PEERS=""
            for i in $(seq 0 $((${CLUSTER_SIZE} - 1))); do
                PEERS="${PEERS}${PEERS:+,}${SET_NAME}-${i}=http://${SET_NAME}-${i}.${SET_NAME}.default.svc.cluster.local:2380"
            done
            # start etcd. If cluster is already initialized the `--initial-*` options will be ignored.
            exec etcd --name ${HOSTNAME} \
              --listen-peer-urls http://${IP}:2380 \
              --listen-client-urls http://${IP}:2379,http://127.0.0.1:2379 \
              --advertise-client-urls http://${HOSTNAME}.${SET_NAME}:2379 \
              --initial-advertise-peer-urls http://${HOSTNAME}.${SET_NAME}:2380 \
              --initial-cluster-token etcd-cluster-1 \
              --initial-cluster ${PEERS} \
              --initial-cluster-state new \
              --data-dir /var/run/etcd/${IP}/default.etcd
## We are using dynamic pv provisioning using the "standard" storage class so
## this resource can be directly deployed without changes to minikube (since
## minikube defines this class for its minikube hostpath provisioner). In
## production define your own way to use pv claims.
  volumes:
    - name: mydata
      nfs:
        path: /
        server: 172.16.0.4
```

##  0x05  参考
- [Create statefulset etcd cluster on kubernetes.](https://github.com/wilhelmguo/etcd-statefulset)
- [一文读懂 K8s 持久化存储流程](https://developer.aliyun.com/article/754434)