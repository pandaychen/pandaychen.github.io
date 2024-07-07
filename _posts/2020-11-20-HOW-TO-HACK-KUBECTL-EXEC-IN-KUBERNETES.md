---
layout:     post
title:      Kubernetes 零信任实战：Teleport
subtitle:   如何实现对 kubectl exec 的劫持？
date:       2020-11-20
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Teleport
    - Zero Trust
---

##  0x00    前言
在 kubernetes 中，常用如下指令进入一个 Pod（在 Pod 运行命令），或者说 [获取正在运行容器的 Shell](https://kubernetes.io/zh/docs/tasks/debug-application-cluster/get-shell-running-container/)：
```bash
kubectl exec ${pod_name} -- date
kubectl exec -it ${pod_name} -- /bin/bash
```

既然是 Shell 操作，基于零信任的理念，如何实现对这个过程的审计与管控呢，具体来说需要完成这么几个事情：
1.  权限的托管，简单说就是哪个账号可以访问哪个 Pod（或其他 kubernetes 资源）
2.  登录过程的可信任，链路是可信任的
3.  对整个过程的审计（会话审计）


##  0x01    实现思路
首先了解下 kubernetes 的 exec 的工作原理，参见此文章 [Kubectl exec 的工作原理解读](https://juejin.im/post/6844904168860155911)。先看看 `kubectl exec` 的详细日志，如下：
```bash
#   kubectl -v=7 exec -it nginx -- /bin/bash
I0125 10:51:55.434043   28053 loader.go:359] Config loaded from file:  /home/isim/.kube/kind-config-linkerd
I0125 10:51:55.438595   28053 round_trippers.go:416] GET https://127.0.0.1:38545/api/v1/namespaces/default/pods/nginx
I0125 10:51:55.438607   28053 round_trippers.go:423] Request Headers:
I0125 10:51:55.438611   28053 round_trippers.go:426]     Accept: application/json, */*
I0125 10:51:55.438615   28053 round_trippers.go:426]     User-Agent: kubectl/v1.15.0 (linux/amd64) kubernetes/e8462b5
I0125 10:51:55.445942   28053 round_trippers.go:441] Response Status: 200 OK in 7 milliseconds
I0125 10:51:55.451050   28053 round_trippers.go:416] POST https://127.0.0.1:38545/api/v1/namespaces/default/pods/nginx/exec?command=%2Fbin%2Fbash&container=nginx&stdin=true&stdout=true&tty=true
I0125 10:51:55.451063   28053 round_trippers.go:423] Request Headers:
I0125 10:51:55.451067   28053 round_trippers.go:426]     X-Stream-Protocol-Version: v4.channel.k8s.io
I0125 10:51:55.451090   28053 round_trippers.go:426]     X-Stream-Protocol-Version: v3.channel.k8s.io
I0125 10:51:55.451096   28053 round_trippers.go:426]     X-Stream-Protocol-Version: v2.channel.k8s.io
I0125 10:51:55.451100   28053 round_trippers.go:426]     X-Stream-Protocol-Version: channel.k8s.ioI0125 10:51:55.451121   28053 round_trippers.go:426]     User-Agent: kubectl/v1.15.0 (linux/amd64) kubernetes/e8462b5
I0125 10:51:55.465690   28053 round_trippers.go:441] Response Status: 101 Switching Protocols in 14 milliseconds
root@nginx:/#
```

总结下上面的过程：
1、两个连续 HTTP 请求 <br>
-   GET 请求用来获取 Pod 信息：`https://127.0.0.1:38545/api/v1/namespaces/default/pods/nginx`
-   POST 请求调用 Pod 的子资源 exec 在容器内执行命令（进入容器）：`https://127.0.0.1:38545/api/v1/namespaces/default/pods/nginx/exec?command=%2Fbin%2Fbash&container=nginx&stdin=true&stdout=true&tty=true`

子资源（subresource）隶属于某个 K8S 资源，表示为父资源下方的子路径，例如 /logs、/status、/scale、/exec 等。其中每个子资源支持的操作根据对象的不同而改变

2、API Server 返回了 101 Ugrade 响应，向客户端表示已切换到 SPDY 协议

ps：SPDY 允许在单个 TCP 连接上复用独立的 Stdin/Stdout/Stderr/Spdy-error 流

从上面的过程来看，很容易联想到使用中间人（MITM）的方式来实现对此过程的劫持，如下图所示：
![kubernetes-mitm-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/kubernetes-mitm-1.png)

即需要实现一个 MITM，完成如下的功能：
-   构造一个 fake kubernetes API-Server，实现满足 exec 场景的 API 劫持及转发
-   内置一个与真正集群打通的 kubectl 客户端（或者实现相关的 kubectl 请求发起的接口）
-   实现对 User 的鉴权、认证
-   打通到真实 kubernetes 集群 Pod 访问的整个通道
-   整个流程中 CA 证书的签发及生效（推荐使用证书，这样 Kubernetes 所有的安全访问都是基于 CA 证书的）

##  0x02    业界实现

业界有两款开源的实现：

####    Teleport
Teleport 的整体架构如下：
![teleport](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/teleport-kubernetes-outside.png)

关于 kubernetes 的详细实现架构如下：
![kubenetes-teleport](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/teleport-kubectl-hacker1.png)

This model allows Teleport proxy to work with any standard Kubernetes clusters supporting out of the box CSR API.
-   User uses tsh login with proxy, nothing changes compared to usual SSH flow.
-   Client receives short lived x509 certs and SSH certs issued by Telepoort CA. User's kubernetes configuration is updated with x509 certs and credentials.
-   Kubernetes API request goes to teleport proxy, as set up by kubeconfig
-   Before processing the request, proxy needs a valid certificate recognized by Kubernetes cluster. Proxy delegates this task to the auth server and passes
the list of kubernetes groups derived from user roles.
-   Auth server uses Kubernetes native CSR API to send and approve request. Auth server acts both as a requester and approver.
-   Proxy can now use the certificates issued by Kubernetes CA to terminate the request and proxy it to the auth server, capture the exec traffic

####    Pomerium
[pomerium](https://github.com/pomerium/pomerium) 的实现如下：


##  0x03    Teleport 的方案分析
![teleport-k8s-hacker](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/teleport-kubectl-hacker1.png)


##  参考
-   [Kubectl exec 的工作原理解读](https://zhuanlan.zhihu.com/p/143734114)
-   [How It Works — kubectl exec](https://itnext.io/how-it-works-kubectl-exec-e31325daa910)
-   [图解 kubernetes 命令执行核心实现](https://www.kubernetes.org.cn/7195.html)
-   [Teleport - Kubernetes support](https://github.com/gravitational/teleport/issues/1986)
-   [Teleport Kubernetes Access Guide](https://gravitational.com/teleport/docs/kubernetes-ssh/)
-   [Teleport Kubernetes: how it works](https://gravitational.com/teleport/how-it-works/)
-   [pomerium - Overview](https://www.pomerium.io/docs/#)
-   [pomerium - kubernetes](https://www.pomerium.com/docs/quick-start/kubernetes.html)
-   [Achieving Cloud Native Security and Compliance with Teleport](https://www.infracloud.io/blogs/achieving-cloud-native-security-compliance-teleport/)