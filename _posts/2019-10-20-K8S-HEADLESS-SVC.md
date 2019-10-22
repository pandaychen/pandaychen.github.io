---
layout:     post
title:      kubernetes应用改造(一)
subtitle:   使用GRPC+Headless-Service构建后端服务
date:       2019-10-20
author:     pandaychen
catalog:    true
tags:
    - Kubernetes
    - GRPC
    - HeadlessSvc
---

## 背景

本地后台程序容器化，服务上云，SASS化尝试，现阶段没有比Kubernetes更好的选择了。从七月到现在，三个月时间，通过对项目代码改造、后台服务的迁移和现网运行排障，算是对Kubernetes的服务部署和微服务化有了比较全面的认知。这篇博客就以GRPC服务在Kubernetes上的LoadBalancer改造来开个头。


## Kubernetes从入门到入门

网上有很多关于Kubernetes的高质量文章和阿里公开课，可以帮助读者构建Kubernetes的生态认知。Kubernetes的几个核心概念中，只要理解了分层的概念，穿透各个组件都是围绕此来设计和实现的。下层为上层提供服务，上层不要知道下层的具体实现细节，只需使用下层提供的服务。而层与层之间联系的桥梁就是接口(Interface)。<br>

##	开发需要掌握的Kubernetes
就我个人而言，作为一名合格的云开发者，我认为需要理清楚下面的知识概念：

1.	POD	--> Deployment--> Service ---> Ingress 服务应用如何对照入座，在K8S中部署的应用无法脱离这几种应用形态(去掉POD)
2.	Statefulset和Damonset的应用场景
3.	JOB/crontabJOB的应用场景
4.	PV和PVC的实践
5.	CONFIG-MAP与Secret如何与应用搭配（或者说更安全、更灵活的搭配）
6.	如何实现CONFIGMAP的热更新
7.	POD之间如何共享数据，POD生产的日志如何持久化
8.	容器的健康检查，就绪探针与保活探针的使用
9.	POD之间的互相访问和服务发现方式
10.	Kubernetes上的负载均衡如何做
11. Kubernetes中的认证与授权、RBAC
12. RBAC的应用之，如何使用ServiceAccount+证书来调用APISERVER的相关操作（当然了，必须对SA授权）
13.	如何通过Kubernetes的API来获取POD列表的实时变化（实现GRPC+Kubernetes的自定义负载均衡的算法）

##  进阶话题
- Kubernetes路由的实现原理，IPVS
- Istio，感觉这个开源的作品在未来若干年会成为主流的管理应用
- CRD开发，各种现网的特殊需求，原生提供的Crontroller可能无法满足

未完待续

##  Service In Kubernetes

引用一下GKE的图:<br>
[Service](https://s2.ax1x.com/2019/10/22/KG5E5D.png)
[Service+LB](https://s2.ax1x.com/2019/10/22/KG5aMn.png)

一般应用中都使用Service的负载均衡方式