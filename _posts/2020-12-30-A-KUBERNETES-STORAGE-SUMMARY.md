---
layout:     post
title:      Kubernetes 应用改造（七）：存储那些事
subtitle:   Kubernetes 中的存储方案
date:       2020-12-30
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
    - 存储
---

##  0x00  前言
Pod 是 Kubernetes 中创建和部署的最小单位，Pod 中封装着应用的容器，存储、独立的网络 IP，以及管理容器如何运行的策略选项。由于容器本身是非持久化的，因此需要解决在容器中运行应用程序遇到的一些问题。首先，当容器崩溃时，Kubelet 将重新启动容器，但是写入容器的文件将会丢失，容器将会以镜像的初始状态重新开始；第二，在通过多个 Pod 中一起运行的容器，通常需要共享容器之间一些文件。Kubernetes 通过 PersistentVolume 解决上述的两个问题。无状态的应用部署与容器内，有状态的部分使用 PersistentVolume 来保存成为基于 Kubernetes 构建应用的 basic 原则之一。

在开发及实践中经常遇到的问题是：
1.  Kubernetes 的存储类型
2.  容器中应用的日志如何持久化保存（的方案）
3.  不同 Pod 之间如何共享存储及使用共享存储
4.  PV、PVC 及 SC 的设计思路

##  0x01  再看 PV & PVC & SC

####  宏观看 Kubernetes 的存储
下图直观的展现了 SC-->PV-->PVC 之间的关系：
![kubernetes-sc-all](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/storage/kubernetes_sc1.png)

1.  PV（PersistentVolume） 是对底层的共享存储的一种抽象，PV 由管理员进行创建和配置，它和具体的底层的共享存储技术的实现方式有关，比如 Ceph、GlusterFS、NFS 等，都是通过插件机制完成与共享存储的对接
2.  持久化卷声明 PVC（PersistenVolumeClaim）是用户存储的一种声明，和 Pod 相比，Pod 是消耗节点，PVC 消耗的是 PV 资源，Pod 可以请求 CPU 和内存，而 PVC 可以请求特定的存储空间和访问模式。对于真正存储的用户不需要关心底层的存储实现细节，只需要直接使用 PVC 即可
3.  存储类 SC（StorageClass）为管理员提供了一种描述他们提供的存储的类的方法。不同的类可能映射到服务质量级别，或备份策略，或者由群集管理员确定的任意策略，支持根据 SC 动态创建 PV。

Kubernetes 的设计让分层模型展现的淋漓尽致。存储工程师把分布式存储系统上的总空间划分成一个个小的存储块，K8S 的集群管理员将存储块和 PV 进行一一对应，用户通过 PVC 对对存储进行申请，比如可以指定具体容量的大小、访问模式或存储类型，<font color="#dd0000"> 这样的好处是用户不需要关心底层的存储实现细节，只需要直接申请使用 PVC 即可。若申请的 PVC 所对应的 PV 不能满足用户的要求，不会生效，直到有合适的 PV 生成，PVC 会自动与 PV 完成绑定，存储工程师、K8S 管理员，用户之间业务解耦，灵活性更强 </font>。

####  资源供应
资源供应主要分静态和动态两种创建方式，在静态供应的情况下，由集群管理员预先创建 PV，开发者创建 PVC 和 Pod，Pod 通过 PVC 使用 PV 提供的存储。基于静态供应方式的过程如下图所示：
![kubernetes-sc-static](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/storage/kubernetes_sc3.png)

反之，动态分配方式更加灵活，管理员预先创建 StorageClass，用户只需要创建好 PVC 即可使用 PV，PV 由系统根据 PVC 和 SC 自动创建出来。如下图：
![kubernetes-sc-dynamic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/storage/kubernetes_sc4.png)

对于规模不大的容器平台而言，静态 PV 方式就可以满足需求；但是当容器规模很大时，你不知道用户什么时候创建 PV，因此动态 PV 方式就非常有用了。

####  PV 的操作周期
下图直观的表现了 PV 的整个操作周期：<br>
资源供应 -> 资源绑定 -> 资源使用 -> 资源释放 -> 资源回收 <br>
![kubernetes-pv-life-cycle](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/storage/Kubernetes_PV_and_PVC_Life_Cycle.png)

##  0x02  共享 && 持久化方案
一般常用的共享持久化存储方案有如下几种：
1.  NFS
2.  CephFS
3.  COS

##  0x03 容器日志收集方案
通常解决容器内应用日志收集有如下几种方案：
1.  使用 Sidecar 方式
2.  使用共享存储挂载 Pod 的方式，将日志落地到共享存储
3.  通过 Kafka 将日志异步落地到云硬盘中

（少 - DOCKER 容器收集方案介绍）

##  0x04  共享存储之 NFS


##  0x05  共享存储之 CephFS
本小节，介绍下 CephFS 容器持久化卷解决方案。CephFS 主要使用场景是对象存储。

####  静态方式
先介绍以静态方式为例，使用 cephFS 作为容器持久化卷的步骤：

1、创建 PV，即新建 Cephfs PV 对象文件 `cephfs-pv.yaml`<br>
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cephfs-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  cephfs:
    monitors:
      - 192.168.0.3:6789
    user: kube
    secretRef:
      name: secret-for-cephfs
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle
```
这里使用的 `accessModes` 模型为 `ReadWriteMany`（即支持多个 Pod 同时以读写方式挂载），接下来创建 PV。

2、创建 PV<br>
```bash
$ kubectl create -f cephfs-pv.yaml
persistentvolume "cephfs-pv" created
$ kubectl get pv
NAME        CAPACITY   ACCESSMODES  RECLAIMPOLICY   STATUS
cspfs-pv    10Gi       RWX          Recycle         Available
```

3、创建 PVC<br>
PV 资源已经创建好了，接下来创建对资源的请求 PVC，新建 PVC 配置文件 cephfs-pv-claim.yaml 如下：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pv-claim
  namespace: cephfs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

创建 && 查看 PVC：
```bash
$ kubectl create -f cephfs-pv-claim.yaml
persistentvolumeclaim "cephfs-pv-claim" created
$ kubectl get pvc
NAME              STATUS    VOLUME     CAPACITY   ACCESSMODES   STORAGECLASS   AGE
cephfs-pv-claim   Bound     cephfs-pv  10Gi       RWX                          10s
```

4、创建挂载 CephFS 的 Deployment<br>
最后，创建挂载该 cephFS 的 Deployment 进行测试。将上边创建的 cephfs-pv-claim 请求的资源挂载到容器的 `/data/cephfs` 目录，随着 Pod 的调度和创建，Pod 里面的容器即可正常访问由 Cephfs 提供的持久化卷存储空间。
```yaml
metadata:
  name: cephfs-pvc
  namespace: cephfs
spec:
  replicas: 2   #创建 2 个 pod
  template:
    metadata:
      labels:
        app: cephfs-pvc
    spec:
      containers:
      - name: nginx
        image: busybox:latest
        volumeMounts:
        - name: cephfs-pv
          mountPath: /data/cephfs
          readOnly: false
      volumes:
      - name: cephfs-pv
        persistentVolumeClaim:
          claimName: cephfs-pv-claim
```
如此配置完成。在同一个 Deployment 中的 `2` 个 Pod 容器内都可以读写同一个 cephFS volume 了

####  动态方式
本小节看下如何基于 `external-storage/cephfs` 来实现动态 PV。
> cephfs-provisioner 是 kubernetes 官方社区提供的 Cephfs 的 StorageClass 支持。用于动态的调用后端存储 Cephfs 创建 PV。

![kubernetes-cephfs-dynamic](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/storage/kubernetes_cepfs_dynamic1.png)

- `cephfs-provisoner.go`：cephfs-provisoner（cephfs 的 StorageClass）的核心，主要是 watch kubernetes 中 PVC 资源的 CURD 事件，然后以命令行方式调用 cephfs_provisoner.py 脚本创建 PV
- `cephfs_provisoner.py`：python 脚本实现的与 cephfs 交互的命令行工具。cephfs-provisoner 对 cephfs 端 volume 的创建都是通过该脚本实现。封装了 volume 的增删改查等功能
- `Cephfs.go`：cephfs volume 的核心，主要提供 cephfs 的 mount，unmount 等操作，负责把 cephfs 挂载到 Pod 调度的节点

步骤：
（少）

基于动态 PV 方式，无需再手动创建 PV，只需创建 PVC，`cephfs-provisoner` 会自动的为该 PVC 在后端 cephFS 存储上创建对应的 PV：
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-1
  annotations:
    volume.beta.kubernetes.io/storage-class: "cephfs"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

##  0x06  参考
- [极客时间：Kubernetes 中的存储](https://time.geekbang.org/column/intro/116)
- [Kubernetes 持久化存储 Cephfs](https://www.infoq.cn/article/jqhjzvvl11escvfydruc)