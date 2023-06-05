---
layout:     post
title:      helm 入门与应用：Kubernetes包管理器
subtitle:   使用helm部署kubernetes应用
date:       2023-06-01
author:     pandaychen
catalog:    true
tags:
    - helm
---



##  0x00    前言


##  0x01    基本用法与原理


```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm plugin install https://github.com/chartmuseum/helm-push
helm package $repo_name ${repo_path}
helm template ${repo_name} ./{repo_path} -f ./helm-values.yaml 
helm cm-push ./${repo_name}-0.2.14.tgz bifrost -f
```

####    工作原理
helm是作为Helm Repository的客户端工具，默认工作时，会从本地home目录中去获取chart，只有本地没有chart时，它才会到远端的Helm Repository上去获取Chart，当然你也可以自己在本地做一个Chart，当需要应用chart到K8s上时，就需要helm去联系K8s Cluster上部署的Tiller Server，当helm将应用Chart的请求给Tiller Server时，Tiller Server接受完helm发来的charts(可以是多个chart) 和 chart对应的Config 后，它会自动联系API Server，去请求应用chart中的配置清单文件，最终这些清单文件会被实例化为Pod或其它定义的资源，而这些通过chart创建的资源，统称为release，一个chart可被实例化多次，其中的某些参数是会根据Config规则自动更改，例如Pod的名字等。

##  0x



##  参考
-   [Kubernetes包管理器](https://helm.sh/zh/)