---
layout:     post
title:      Kubernetes应用改造（二）
subtitle:   使用Statefulsets钩建Etcd集群
date:       2019-10-20
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
    - Statefulsets
---

##  前言
本篇文章，简单说下，项目中是如何将 `Etcd` 集群部署在`Kubernetes`上的，涉及到两个核心知识点：
1.  `Statefulsets`及应用
2.  `Pod`共享数据存储的最佳方案

##  什么是StatefulSets
`StatefulSets`是为了有状态的应用和分布式系统设计的。关于`StatefulSets`的应用，从以下几点着手：
-   如何创建一个`StatefulSets`
-   `StatefulSets`如何管理它的`Pods`
-   如何删除一个`StatefulSets`
-   如何扩展一个`StatefulSets`
-   如何更新一个`StatefulSets`中的`Pod`的容器镜像
-   `StatefulSets`的典型应用场景

