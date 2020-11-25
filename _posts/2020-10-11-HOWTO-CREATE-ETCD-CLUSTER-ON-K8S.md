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
[`StatefulSets`](https://kubernetes.io/zh/docs/concepts/workloads/controllers/statefulset/) 是为了有状态的应用和分布式系统设计的，<font color="#dd0000"> 核心就是稳定 </font>，它的特点如下：
1.  `Pod` 一致性：包含次序（启动、停止次序）、网络一致性。此一致性与 `Pod` 相关，与被调度到哪个 `Node` 节点无关
2.  稳定的次序：对于 `N` 个副本的 `StatefulSet`，每个 `Pod` 都在 `[0，N)` 的范围内分配一个数字序号，且是唯一的
3.  稳定的网络：`Pod` 的 `hostname` 模式为 `(statefulset 名称)-(序号)`
4.  稳定（持久）的存储：通过 `VolumeClaimTemplate` 为每个 `Pod` 创建一个 `PV`。删除、减少副本，不会删除相关的卷

##  0x02  PV Vs PVC Vs StorageClass
- `PV`：PersistentVolume，集群级别的资源，由 集群管理员 or External Provisioner 创建。`PV` 的生命周期独立于使用 `PV` 的 `Pod`，`PV` 的 `.Spec` 中保存了存储设备的详细信息
- `PVC`：PersistentVolumeClaim，命名空间（Namespace）级别的资源，由 用户 or `StatefulSet` 控制器（根据 `VolumeClaimTemplate`） 创建。`PVC` 类似于 `Pod`，`Pod` 消耗 `Node` 资源，`PVC` 消耗 `PV` 资源。`Pod` 可以请求特定级别的资源（CPU 和内存），而 `PVC` 可以请求特定存储卷的大小及访问模式（Access Mode）
- `StorageClass`：StorageClass 是集群级别的资源，由集群管理员创建。它为管理员提供了一种动态提供存储卷的类模板，其中的 `.Spec` 中详细定义了存储卷 `PV` 的不同服务质量级别、备份策略等等

通俗点说，PersistentVolume 是对存储资源创建和使用的抽象，使得存储作为集群中的资源管理，存储资源提供者和提供存储容量；PersistentVolumeClaim 让用户不需要关心具体的 Volume 实现细节，是存储资源的消费者和消费容量。PersistentVolume 与 PersistentVolumeClaim 是绑定关系。

对应于上面的描述，我们在 TKE 上的三要素配置如下（10G 的 [CFS 存储集群](https://cloud.tencent.com/product/cfs)）：
1、StorageClass 配置 <br>
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  creationTimestamp: "2020-07-14T11:02:51Z"
  name: etcd-cfs
  resourceVersion: "3410126455"
  selfLink: /apis/storage.k8s.io/v1/storageclasses/bifrost-cfs
  uid: 8faee47c-c5c1-11ea-9ede-ae73adf48c93
parameters:
  pgroupid: pgroupbasic
  storagetype: SD
  subnetid: subnet-ent0xze2
  vpcid: vpc-6fynwssx
  zone: na-siliconvalley-1
provisioner: com.tencent.cloud.csi.cfs
reclaimPolicy: Delete
volumeBindingMode: Immediate
```

2、PersistentVolume 配置 <br>
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    pv.kubernetes.io/provisioned-by: com.tencent.cloud.csi.cfs
  creationTimestamp: "2020-07-14T11:10:33Z"
  finalizers:
  - kubernetes.io/pv-protection
  name: pvc-92d1ae5a-c5c2-11ea-9ede-ae73adf48c93
  resourceVersion: "3410213475"
  selfLink: /api/v1/persistentvolumes/pvc-92d1ae5a-c5c2-11ea-9ede-ae73adf48c93
  uid: a3084861-c5c2-11ea-9ede-ae73adf48c93
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: bfsession-pvc1
    namespace: project-test
    resourceVersion: "3410211147"
    uid: 92d1ae5a-c5c2-11ea-9ede-ae73adf48c93
  csi:
    driver: com.tencent.cloud.csi.cfs
    fsType: ext4
    volumeAttributes:
      fsid: 32fefr60
      host: 172.16.0.4
      pgroupid: pgroupbasic
      storage.kubernetes.io/csiProvisionerIdentity: 1594724300399-8081-com.tencent.cloud.csi.cfs
      storagetype: SD
      subnetid: subnet-ent0xze2
      vpcid: vpc-6fynwssx
      zone: na-siliconvalley-1
    volumeHandle: cfs-jmtmbjh1
  persistentVolumeReclaimPolicy: Delete
  storageClassName: etcd-cfs
  volumeMode: Filesystem
status:
  phase: Bound
```

3、PersistentVolume 配置 <br>
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    pv.kubernetes.io/bind-completed: "yes"
    pv.kubernetes.io/bound-by-controller: "yes"
    volume.beta.kubernetes.io/storage-provisioner: com.tencent.cloud.csi.cfs
  creationTimestamp: "2020-07-14T11:10:06Z"
  finalizers:
  - kubernetes.io/pvc-protection
  name: bfsession-pvc1
  namespace: project-test
  resourceVersion: "3410213481"
  selfLink: /api/v1/namespaces/project-test/persistentvolumeclaims/bfsession-pvc1
  uid: 92d1ae5a-c5c2-11ea-9ede-ae73adf48c93
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  storageClassName: etcd-cfs
  volumeMode: Filesystem
  volumeName: pvc-92d1ae5a-c5c2-11ea-9ede-ae73adf48c93
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  phase: Bound
```

##  0x04  Etcd 部署
有两种方式，采用官方的 Etcd-Operator 方式或者 `Statefulset` 部署

####  使用官方的 Etcd-Operator 部署
官方提供了 [etcd-operator](https://github.com/coreos/etcd-operator)，Etcd-Operator 的部署方式对 Etcd 版本和 Kubernetes 版本都有要求, 详见 [官方文档](https://github.com/coreos/etcd-operator/blob/master/README.md)。

####  独立部署
相对于上一种方式，`Statefulset` 可以实现更灵活的配置，比如亲和性、`PV` 使用等等。这里我们通过如下配置文件，在 TKE 上部署一个可用的 Etcd 集群，注意几点：
1.  本配置文件部署为一个 `CLUSTER_SIZE=3` 的集群
2.  数据目录的设定 `--data-dir /var/run/etcd/${IP}/default.etcd`，采用独立的 IP 在 `NFS` 上分隔
3.  `spec.ClusterIP` 必须为 `None`，采用 Headless Service 方式部署，访问采用 DNS 方式（`{SET_NAME}-{i}.{SET_NAME}.default.svc.cluster.local`）

TKE 上的完整的配置文件如下：
```yaml
apiVersion: v1
kind: Service #Service 定义
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
  clusterIP: None #以 headless 方式部署
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
  replicas: 3   #创建 3 个 POD 节点
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
          mountPath: /var/run/etcd  #容器内挂载路径
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

上面的 yaml 配置可以拆分为 `kind: Service` 和 `kind: StatefulSet` 两个部分，配置中 `{SET_NAME}` 的值必须和 `StatefulSet` 的 `.metadata.name` 一致。在集群内部可以通过 `http://{SET_NAME}-{i}.{SET_NAME}.{CLUSTER_NAMESPACE}:2379` 访问 Etcd 集群了。如果需要从外部（VPN 内网或者公网）访问集群，那么需要结合 NodePort 或 Ingress 来暴露对外端口。

此外，[使用 Statefulset 在 Kubernetes 集群中部署 Etcd 集群](https://wilhelmguo.cn/blog/post/william/%E4%BD%BF%E7%94%A8Statefulset%E5%9C%A8Kubernetes%E9%9B%86%E7%BE%A4%E4%B8%AD%E9%83%A8%E7%BD%B2etcd%E9%9B%86%E7%BE%A4) 提供了一个基于 Ceph Block Device 的 Etcd 搭建实现。

####  扩缩容
使用 `kubectl scale` 即可方便的实现集群扩容缩容：
```bash
kubectl scale --replicas=5 statefulset etcd
```

##  0x05  参考
- [Create statefulset etcd cluster on kubernetes.](https://github.com/wilhelmguo/etcd-statefulset)
- [一文读懂 K8s 持久化存储流程](https://developer.aliyun.com/article/754434)
- [Introducing the etcd Operator: Simplify etcd cluster configuration and management](https://coreos.com/blog/introducing-the-etcd-operator.html)
- [使用 Statefulset 在 Kubernetes 集群中部署 Etcd 集群](https://wilhelmguo.cn/blog/post/william/%E4%BD%BF%E7%94%A8Statefulset%E5%9C%A8Kubernetes%E9%9B%86%E7%BE%A4%E4%B8%AD%E9%83%A8%E7%BD%B2etcd%E9%9B%86%E7%BE%A4)