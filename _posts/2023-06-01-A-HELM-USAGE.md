---
layout:     post
title:      helm 入门与应用：Kubernetes 包管理器
subtitle:   使用 helm 部署 kubernetes 应用（helm V3）
date:       2023-06-01
author:     pandaychen
catalog:    true
tags:
    - Helm
    - kubernetes
---

##  0x00    前言
Helm 是 Kubernetes 的包管理工具，类似于 Linux 下的包管理工具如 yum 等。通过 helm 可以将打包好的 yaml 文件部署到 Kunernetes 集群
####    Helm 的应用场景
将所有的 yaml 文件（deployment、Service、Ingress 等等）进行整体的管理，实现 yaml 文件的高效复用。这里的高效复用是指 yaml 文件的格式基本相同，一般只是属性值有所变化。使用 helm 后，针对格式和结构基本相同的 yaml 文件直接复用即可；除此之外，Helm 还可以进行应用级别的版本管理，包括版本更新、回退等


Helm 中有三个主要概念：

-   helm：一个命令行工具，主要用于 k8s 应用 Chart 的创建、打包、发布和管理
-   Chart：应用描述，它是一系列用于描述 k8s 资源相关文件的集合（可理解为 yaml 的集合）
-   Release：基于 Chart 的部署实体，一个 Chart 被 Helm 运行后将会生成一个对应的 release，然后将在 k8s 中创建出真正运行的资源对象，它是一个应用级别的版本管理


![arch](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/helm-2.jpg)

####    工作原理
helm 是作为 Helm Repository 的客户端工具，默认工作时，会从本地 home 目录中去获取 chart，只有本地没有 chart 时，它才会到远端的 Helm Repository 上去获取 Chart，当然你也可以自己在本地做一个 Chart，当需要应用 chart 到 K8s 上时，就需要 helm 去联系 K8s Cluster 上部署的 Tiller Server，当 helm 将应用 Chart 的请求给 Tiller Server 时，Tiller Server 接受完 helm 发来的 charts(可以是多个 chart) 和 chart 对应的 Config 后，它会自动联系 API Server，去请求应用 chart 中的配置清单文件，最终这些清单文件会被实例化为 Pod 或其它定义的资源，而这些通过 chart 创建的资源，统称为 release，一个 chart 可被实例化多次，其中的某些参数是会根据 Config 规则自动更改，例如 Pod 的名字等。

##  0x01    基本用法

注意：一般 helm 需要配合 kubernetes 集群一起使用，在笔者的项目中，主要拿来做 helm 本地配置检查、打包及推送（到托管集群），所以并不涉及到直接操作 kubernetes 集群

1、配置 Helm 仓库 <br>

```bash
helm repo add 仓库名称 仓库地址

#查看仓库
[root@VM_6_254_centos /data/build/helm/bk-sam-install/helm-charts/bk-sam]# helm repo list
NAME    URL
bitnami https://charts.bitnami.com/bitnami
xxxxx https://helm.xxx.xxx.com/xxx/xxx/     #私有仓库
```

2、使用 Helm 快速部署应用 <br>


3、使用 helm 配合自定义 Chart 部署应用 <br>

-   创建：`helm create mychart`，目录如下
-   发布：`helm install myweb1 mychart/`
-   升级：`helm upgrade myweb1 mychart/`

```text
.
|-- charts
|-- Chart.yaml  #当前 chart 属性的配置信息
|-- .helmignore
|-- templates   #自己定义的 yaml 文件（按需修改为自己的应用配置）
|   |-- deployment.yaml
|   |-- _helpers.tpl        #公共模板，作为子模板，可被其他模板文件引用，Helm不会交给K8s处理
|   |-- hpa.yaml
|   |-- ingress.yaml
|   |-- NOTES.txt
|   |-- serviceaccount.yaml
|   |-- service.yaml
|   `-- tests
|       `-- test-connection.yaml
`-- values.yaml     #定义 yaml 文件的全局配置（对应于_helpers.tpl 中的模板实例化位置的值）
```


4、使用 Helm 实现 yaml 文件高效复用 <br>

主要实现原理就是通过动态传递参数、动态渲染模板、动态传入参数生成 yaml 文件内容


5、使用 helm 打包 Chart 配置文件 <br>

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm plugin install https://github.com/chartmuseum/helm-push        #安装 push 插件
helm package $repo_name ${repo_path}                                #打包配置（默认生成 ${repo_path}/Chart.yaml 中的 Version 值）
helm template ${repo_name} ./{repo_path} -f ./helm-values.yaml      #检查配置还原是否正确
helm cm-push ./${repo_name}-0.2.14.tgz bifrost -f                   #推送到 bifrost 仓库
```


6、使用 helm 回滚配置 <br>

[Helm 回滚](https://helm.sh/zh/docs/helm/helm_rollback/)

##  0x02    go template

helm chart 的语法是 go template 的语法，所以建议可以先系统的学习一下 go template

1、管道的用法<br>

在编写Helm template时，建议优先使用管道来替代函数调用的方式，如下面的例子：

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
data:
  myvalue: "Hello World"
  drink: {{ .Values.favorite.drink | quote }}   
  food: {{ .Values.favorite.food | upper | quote }}   # 大写，然后双引号包裹
```

2、内置对象<br>

参考[内置对象](https://helm.sh/docs/chart_template_guide/#built-in-objects)


##  0x03    模板的一些坑

####    生成证书格式的模板
比如需要引用的`values.yaml`中包含了如下x509证书格式的内容:

```YAML
xxxxxxx-svc:
  service:
    type: LoadBalancer
    annotations:
      service.kubernetes.io/loadbalance-id: lb-xxxxxx
      service.kubernetes.io/tke-existed-lbid: lb-xxxxxx
  jwtkeys:
    RsaPriKey: |-
      -----BEGIN PRIVATE KEY-----
      MIIEvwIBADANBgkqhkiG9w0BAQEFAASCBKkwggSlAgEAAoIBAQC76fWIS89iXwKY
      xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
      Avlug15mg2MkciSsy7YlzDdAkA==
      -----END PRIVATE KEY-----
    RsaPubKey: |-
      -----BEGIN PUBLIC KEY-----
      MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAu+n1iEvPYl8CmNijKzY1
      xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
      jQIDAQAB
      -----END PUBLIC KEY-----
```

那么，chart模板文件中的格式要写成下面这样（假设服务的配置使用`configmap.yaml`）：

```text
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "common.names.fullname" . }}-xxxxxxxx
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": pre-upgrade,pre-install
    "helm.sh/hook-weight": "-2"
data:
  xxxxxxx-svc.yaml: |-
    {{ include "xxxx.svcCommonConfig" . | nindent 4 }}

    debug: {{ .Values.xxxxxxxx.debug }}
    server:
      host: 0.0.0.0
      port: 80
      readTimeout: 60
      writeTimeout: 60
      idleTimeout: 180


    jwtKeys:
      RsaPriKey: |- {{ .Values.xxxxxxxx.jwtkeys.RsaPriKey | nindent 8 }}    #改成.Values.xxxxxxxx.jwtkeys.RsaPriKey会报错
      RsaPubKey: |- {{ .Values.xxxxxxxx.jwtkeys.RsaPubKey | nindent 8 }}
```

####    生成 yaml 数组的模板


##  0x04  helm模板参考
直接学习现有项目的helm配置（画瓢）更快，比如：

- [pomerium-helm](https://github.com/pomerium/pomerium-helm/tree/main/charts/pomerium/templates)
- [Helm chart repository for example charts](https://github.com/helm/examples)

##  0x05    参考
-   [Kubernetes 包管理器](https://helm.sh/zh/)
-   [一文掌握 kubernetes 包管理工具 Helm](https://blog.csdn.net/weixin_53072519/article/details/126693667)
-   [Helm Chart & template 函数](https://jicki.cn/helm-chart/)
-   [通过 Helm 部署 CMDB](https://github.com/TencentBlueKing/bk-cmdb/blob/master/helm/README.md)
-   [Helm template快速入门](https://juejin.cn/post/6844904199818313735)
-   [玩K8S不得不会的HELM](https://juejin.cn/post/6844904081366974472)