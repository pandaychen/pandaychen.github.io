---
layout:     post
title:      Kubernetes Certificate 使用 && 总结
subtitle:   How to manipulate kubernetes's certificates？
date:       2020-04-09
author:     pandaychen
header-img: img/golang-horse-fly.png
catalog: true
category:   false
tags:
    - Kubernetes
    - 证书
---

##  0x00    TLS 认证
在前文 [证书（Certificate）的那些事](https://pandaychen.github.io/2019/07/24/auth/) 中介绍了 X509 体系、证书认证等相关知识，采用 TLS/SSL 体系一般两种方式：
* 服务器单向认证：只需要服务器端提供证书，客户端通过服务器端证书验证服务的身份，但服务器并不验证客户端的身份。这种情况一般适用于对 Internet 开放的服务，例如搜索引擎网站，任何客户端都可以连接到服务器上进行访问，但客户端需要验证服务器的身份，以避免连接到伪造的恶意服务器
* 双向 TLS 认证：除了客户端需要验证服务器的证书，服务器也要通过客户端证书验证客户端的身份。只允许特定身份的客户端访问

X509 的证书签发流程如下，对理清下面 kubernetes 的证书关系非常有帮助：
![x509-certallinone](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/basic/x509-cert-allinone.png)

##    0x01  gRPC 的双向认证
Kubernetes 中的组件之间大都通过 HTTP/gRPC 通信，通信流程基本如下图所示：
![kubernetes-components](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/kubernetes-data-flow-call.png)

这里以 gRPC 为例，简单列举下双向认证需要的证书要素，基于开篇图中的证书签发逻辑，假设得到了如下证书，包含了 CA、客户端及服务端：
```bash
conf
├── ca.key
├── ca.crt
├── ca.srl
├── client
│   ├── client.csr
│   ├── client.key
│   └── client.crt
└── server
    ├── server.csr
    ├── server.key
    └── server.crt
```

####    客户端代码
服务端需要 `client.crt`、`client.key` 及 CA 的证书 `ca.crt` （还要包含 Server 证书的 `Common Name`）三要素，加载证书的方法如下：
```golang
func main() {
    cert, err := tls.LoadX509KeyPair("client.crt", "client.key")
    if err != nil {
        log.Fatalf("tls.LoadX509KeyPair err: %v", err)
    }

    certPool := x509.NewCertPool()
    ca, err := ioutil.ReadFile("ca.crt")
    if err != nil {
        log.Fatalf("ioutil.ReadFile err: %v", err)
    }

    if ok := certPool.AppendCertsFromPEM(ca); !ok {
        log.Fatalf("certPool.AppendCertsFromPEM err")
    }

    c := credentials.NewTLS(&tls.Config{
        Certificates: []tls.Certificate{cert},
        ServerName:   "pandaychentest",
        RootCAs:      certPool,
    })

    conn, err := grpc.Dial(":"+PORT, grpc.WithTransportCredentials(c))
    if err != nil {
        log.Fatalf("grpc.Dial err: %v", err)
    }
    //......
}
```

####  服务端代码
服务端需要 `server.crt`、`server.key` 及 CA 的证书 `ca.crt` 三要素，加载证书的方法如下：
```golang
func main() {
    cert, err := tls.LoadX509KeyPair("server.crt", "server.key")
    if err != nil {
        log.Fatalf("tls.LoadX509KeyPair err: %v", err)
    }

    certPool := x509.NewCertPool()
    ca, err := ioutil.ReadFile("ca.crt")
    if err != nil {
        log.Fatalf("ioutil.ReadFile err: %v", err)
    }

    if ok := certPool.AppendCertsFromPEM(ca); !ok {
        log.Fatalf("certPool.AppendCertsFromPEM err")
    }

    c := credentials.NewTLS(&tls.Config{
        Certificates: []tls.Certificate{cert},
        ClientAuth:   tls.RequireAndVerifyClientCert,
        ClientCAs:    certPool,
    })

    server := grpc.NewServer(grpc.Creds(c))
    //......
}
```

##  0x02  Kubernetes 的 TLS 认证
同样的，Kubernetes 的 TLS 认证体系也分成两类：Server Auth 和 Client Auth。

####  Server Auth
从架构看，集群环境中最重要的服务方为 Etcd 和 APIServer，此外还包括 Kubelet：
- Etcd：作为 Kubernetes 中对象的配置信息库
- APIServer：提供所有组件的 API 访问接口
- Kubelet：（每个 node）用于处理 Master 节点下发到本 node 的任务，管理 Pod 和 Container

在证书体系中，此二者各需要一套 CA 证书（信任）关系链，某些场景下，Kubelet 也存在的一套 CA 证书链（非必要）。默认情况下，APIServer 访问 kubelet 时，不需要对 Kubelet 进行 Server Auth

#### Client Auth
客户端信任（Client Auth）包含如下几点：
- 对于访问 APIServer 的其他 Kubernetes 组件，通过 Server Auth 确认 APIServer 是否可信。
- 通过客户端证书是向 APIServer 证明自己亦可信
- Kubernetes 的 Client Auth 机制有：Client Certificate/TLS bootstrapping 等，存储有 Secret、kubeConfig 及本地证书文件
- 如果集群中部署了 extension APIServer（如 Metrics），访问 extension APIServer 也需要进行 Client Auth
- <font color="#dd0000"> 如果存在多集群管理的平台或者通过 kubectl 访问多个 Kubernetes 集群 </font>，那么该容器平台无论通过 WebAPI 或者 kubectl 客户端访问某个 Kubernetes 集群的 APIServer，<font color="#dd0000"> 都需要将平台的 Root CA 证书附加到 APIServer 的信任 CA 证书中，否则 APIServer 会认为该平台不可信 </font>，平台将无法通过 API 访问该 Kubernetes 集群

##  0x03  Kubernetes 的模块调用
下面引入一个典型的较为完整的 Kubernetes [模块调用示意图](https://zhaohuabing.com/post/2020-05-19-k8s-certificate/)，序号标识了服务访问过程，箭头表明了调用方向，箭头所指方向为服务提供方，另一头为服务调用方：
![kubernetes-certificate-indetail](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/kubernetes-certificate-indetail.png)

为了实现 TLS 双向认证，服务提供方需要使用一个服务端证书（server.crt），服务调用方则需要提供一个客户端证书（client.crt），并且双方都需要使用同一个 CA 证书（ca.crt）来验证对方提供的证书。证书的作用如下：

1.  Etcd 集群中各个节点之间相互通信使用的证书，由于一个 Etcd 节点既为其他节点提供服务，又需要作为客户端访问其他节点（该证书同时用作服务端证书和客户端证书）
2.  Etcd 集群向外提供服务（主要提供 APIServer 访问）使用的证书（服务端证书）
3.  kube-apiserver 作为客户端访问 Etcd 使用的证书（客户端证书）
4.  kube-apiserver 对外提供服务使用的证书（服务端证书）
5.  kube-controller-manager 作为客户端访问 kube-apiserver 使用的证书（客户端证书）
6.  kube-scheduler 作为客户端访问 kube-apiserver 使用的证书（客户端证书）
7.  kube-proxy 作为客户端访问 kube-apiserver 使用的证书（客户端证书）
8.  kubelet 作为客户端访问 kube-apiserver 使用的证书（客户端证书）
9.  管理员 / 用户通过 kubectl 访问 kube-apiserver 使用的证书（客户端证书）
10. kubelet 对外提供服务使用的证书（服务端证书）
11. kube-apiserver 作为客户端访问 kubelet 采用的证书（客户端证书）
12. kube-controller-manager 用于生成和验证 service-account token 的证书。该证书并不会像其他证书一样用于身份认证，而是将证书中的公钥 / 私钥对用于 service-account token 的生成和验证。<font color="#dd0000">kube-controller-manager 会用该证书的私钥来生成 service-account token（如何生成？），然后以 secret 的方式加载到 pod 中，pod 中的应用可以使用该 token 来访问 kube-apiserver，kube-apiserver 会使用该证书中的公钥来验证请求中的 token</font>

##  0x04  kubernetes 中的证书
使用 [`kubeadm` 工具](https://github.com/kubernetes/kubeadm) 搭建集群后，证书存放目录如下：
```bash
[root@VM-centos etc]# tree kubernetes/
kubernetes/
|-- admin.conf
|-- controller-manager.conf
|-- kubelet.conf
|-- scheduler.conf
|-- manifests
|   |-- etcd.yaml
|   |-- kube-apiserver.yaml
|   |-- kube-controller-manager.yaml
|   `-- kube-scheduler.yaml
|-- pki
|   |-- apiserver.crt
|   |-- apiserver-etcd-client.crt
|   |-- apiserver-etcd-client.key
|   |-- apiserver.key
|   |-- apiserver-kubelet-client.crt
|   |-- apiserver-kubelet-client.key
|   |-- ca.crt
|   |-- ca.key
|   |-- etcd
|   |   |-- ca.crt
|   |   |-- ca.key
|   |   |-- healthcheck-client.crt
|   |   |-- healthcheck-client.key
|   |   |-- peer.crt
|   |   |-- peer.key
|   |   |-- server.crt
|   |   `-- server.key
|   |-- front-proxy-ca.crt
|   |-- front-proxy-ca.key
|   |-- front-proxy-client.crt
|   |-- front-proxy-client.key
|   |-- sa.key
|   `-- sa.pub
```


####  主要涉及到组件
* etcd.yaml
* kube-apiserver.yaml
* kube-controller-manager.yaml
* kube-scheduler.yaml

####  根证书（ROOT）
kubeadm 默认生成了 `3` 套根证书（用来管理和签发其他证书）：
- `pki/etcd/ca.key[ca.crt]`：用于 Etcd 组件相关的内部（组 Etcd 集群）及外部访问
- `pki/ca.key[ca.crt]`：用于 Kubernates APIServer 相关的访问
- `pki/front-proxy-ca.key[front-proxy-ca.crt]`：用于配置 Kubernates 聚合层使用

为了厘清 Kubernates 证书的关系，这 `3` 套 CA 全部使用一对证书来管理。<font color="#dd0000">只要在通信组件中正确配置用于验证对方证书的 CA 根证书，就可以使用不同的 CA 来颁发不同用途的证书</font>。

####  Etcd CA 信任链 && 证书 && 秘钥对
Etcd 服务是数据中心。用于持久化存储信息。要求高可用和数据一致性。即多部署几个 master 节点
存储哪些信息：k8s 有本身的节点信息、组件信息、运行的 pod、service 都需要做持久化

Etcd CA 信任链是必须的，Etcd CA 根证书路径为 `/etc/kubernetes/pki/etcd/`
- `ca.key`、`ca.crt`：Etcd 节点之间相互进行认证的 peer 证书、私钥以及验证 peer 的 CA
- `healthcheck-client.crt`、`healthcheck-client.key`：ETCD 验证访问其服务的客户端（etcdctl 工具）的 CA
- `server.key`、`server.crt`：Etcd 对外提供服务的服务端证书及私钥

从 Etcd 的配置文件可以很直观的观察上述证书的功能，注意 Etcd 容器中，还启动了一个 `livenessProbe` 来探测 Etcd 容器的 Live 存活情况：
```yaml
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://10.0.4.3:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://10.0.4.3:2380
    - --initial-cluster=vm-4-3-centos=https://10.0.4.3:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key
    - --listen-client-urls=https://127.0.0.1:2379,https://10.0.4.3:2379
    - --listen-peer-urls=https://10.0.4.3:2380
    - --name=vm-4-3-centos
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    image: k8s.gcr.io/etcd:3.3.10
    imagePullPolicy: IfNotPresent
    livenessProbe:
      exec:
        command:
        - /bin/sh
        - -ec
        - ETCDCTL_API=3 etcdctl --endpoints=https://[127.0.0.1]:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt
          --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key
          get foo
      failureThreshold: 8
```

####  kube-apiserver 证书 && 秘钥对
ApiServer 提供了集群管理的 API 接口，包括认证授权、数据校验、集群状态变更、与其他模块之间的数据交互和通信；Kubernetes的其他模块，也是通过 ApiServer 查询操作数据（只有 ApiServer 才能直接操作 Etcd）

`kube-apiserver` 的配置如下，`apiserver` 的证书和秘钥对在此路径 `/etc/kubernetes/pki`，`apiserver` 和多个 kubernetes 模块都有交互：
```YAML
spec:
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=10.0.4.3
    - --allow-privileged=true
    - --authorization-mode=Node,RBAC
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --enable-admission-plugins=NodeRestriction
    - --enable-bootstrap-token-auth=true
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
    - --insecure-port=0
    - --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt
    - --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt
    - --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key
    - --requestheader-allowed-names=front-proxy-client
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-username-headers=X-Remote-User
    - --secure-port=6443
    - --service-account-key-file=/etc/kubernetes/pki/sa.pub
    - --service-cluster-ip-range=10.96.0.0/12
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
    image: k8s.gcr.io/kube-apiserver:v1.15.2
```

1、`apiserver-etcd-client.key`、`apiserver-etcd-client.crt`：访问 ETCD 的客户端证书及私钥，这个证书 `apiserver-etcd-client.crt` 是由 ETCD 的 CA 证书私钥 `etcd/ca.key` 签发，因此也需要在 apiserver 中配置 etcd 的 CA 证书 `--etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt`<br>
2、`ca.crt`、`ca.key`：用来签发 kubernetes 中其他证书的 <font color="#dd0000">CA 根证书 </font > 及私钥 < br>
3、`apiserver.crt`、`apiserver.key`：apiServer 的对外提供服务的服务端证书及私钥 <br>
4、`apiserver-kubelet-client.crt`，`apiserver-kubelet-client.key`：apiserver 访问 kubelet 所需的客户端证书及私钥，同样需要一个 kubelet 的证书公钥 `--client-ca-file=/etc/kubernetes/pki/ca.crt`

####  Extension apiserver 证书 && 秘钥对
extension apiserver 的 CA 信任链按需使用，但不可与 apiserver CA 相同！
1、`front-proxy-ca.crt`、`front-proxy-ca.key`：配置 Extension apiserver 的 <font color="#dd0000">CA 根证书和私钥 </font> <br>
2、`front-proxy-client.crt`、`front-proxy-client.key`：

配置聚合层（apiserver 扩展）的 CA 和客户端证书及私钥
说明：要使聚合层在您的环境中正常工作以支持代理服务器和扩展 apiserver 之间的相互 TLS 身份验证， 需要满足一些设置要求。Kubernetes 和 kube-apiserver 具有多个 CA， 因此请确保代理是由聚合层 CA 签名的，而不是由主 CA 签名的。扩展 apiserver 为了能够和 apiserver 通讯，所以需要在 apiserver 中配置，假如你不需要这个功能可以不配置该证书。

####  验证 service account token 的公钥
`sa.pub`
到这里，集群生成的所有证书介绍完了，那么像 kube-controller-mananger、kube-scheduler、kube-proxy、kubele 这些组件也是需要访问 apiserver 的，那么他们是怎么通讯的呢？下面我们可以看看这些组件是如何和 apiserver 进行通讯的。


##  0x05  kubernetes 其他组件
组件和 apiserver 通讯
那么像 kube-controller-mananger、kube-scheduler、kube-proxy、kubele 这些组件也是需要访问 apiserver 的，那么他们是怎么通讯的呢？下面我们可以看看这些组件是如何和 apiserver 进行通讯的。
Kubernetes 这里的设计是这样的，kube-controller-mananger、kube-scheduler、kube-proxy、kubelet 等组件，采用一个 kubeconfig 文件中配置的信息来访问 kube-apiserver。该文件中包含了 kube-apiserver 的地址，验证 kube-apiserver 服务器证书的 CA 证书，自己的客户端证书和私钥等访问信息，这样组件只需要配置这个 kubeconfig 就行。

最大区别是：你会发现在 yaml 中配置了 / etc/kubernetes/controller-manager.conf 这个配置文件，而不是配置 controller-manager 的客户端证书之类的。

####  kube-controller-mananger
kube-controller-mananger 的配置文件如下：
```yaml
spec:
  containers:
  - command:
    - kube-controller-manager
    - --allocate-node-cidrs=true
    - --authentication-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --authorization-kubeconfig=/etc/kubernetes/controller-manager.conf
    - --bind-address=127.0.0.1
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --cluster-cidr=10.244.0.0/16
    - --cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
    - --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
    - --controllers=*,bootstrapsigner,tokencleaner
    - --kubeconfig=/etc/kubernetes/controller-manager.conf
    - --leader-elect=true
    - --node-cidr-mask-size=24
    - --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt
    - --root-ca-file=/etc/kubernetes/pki/ca.crt
    - --service-account-private-key-file=/etc/kubernetes/pki/sa.key
    - --use-service-account-credentials=true
    image: k8s.gcr.io/kube-controller-manager:v1.15.2
```

由于创建工作负载的时候，我们有时候会用到 service accout，那么这里需要和 apiserver 认证，所以我们需要在 controller-manager

配置上 sa 私钥，当然需要和 apiserver 通讯，自然需要配置上 kubernates 的 CA 证书

下面我们看看 controller-manager.conf 这个文件配置的证书和秘钥是什么。
```yaml
[root@VM-4-3-centos kubernetes]# cat controller-manager.conf
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EZ3lNREF5TXpBd05Wb1hEVE13TURneE9EQXlNekF3TlZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSndoCmw4ZVd5SlBsSWpwajlTN09VSWRSTWVxV0Mwb2crN3hQemJQZDhzS2NTemZqWjdHc0ttUXlvQjhoQnNlaVVDdUwKai9teVl5Tk02MkxIa0ZKbDI3MXNFWVdmOEtiWS81Y210UmFjRnlMOEpyaTNLQi91eHZnZlEvMXhMK2c3UmRBcQpGQllWRzNtaSs1T1orTExyZlVMUU5qemtoTVllaEhDdHNDRmZJMGF5amJpYk1UUGJLT3lobjV3cHVMZzgvOVdlClNTSnI1TmtnK2R0WHJSZ05YelNpc1JMQVF5MmdEczdOaTN0SklaNjRuRGdIakpyS21HR2dqbEljN1RFdGFUdWcKcnltKy92akVZZ2NxTlhHakY2ekJlT1FXNW5NdUh0K1plYXphZ1QyQTNkUDhGY3lEWVZrSFJVd0RESDBZOVZlcwpOUFAyZnhURzVVZlhWOUV0WVJNQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFEajZLYXVQR2dvVnlGQmdNUzFZYlVFRXFHQmoKN3IwaG5vclNuOVp4dlUxZkM1UkZ0UEd0OEI0YU40T3RMa1REUno5ZmdFc1ZidFdoMXRXWURIWUF6N2FDYkVZawpMRTArRzZQMkpxR043SHlrd05BZFp1QS96emhOdVFKZnhjZG5qVHlIRWZXZyt5OEd1S2JqSU1QdFJVOU45bFpoCkZTeUxsYjNvektYbURDK2RuSHhHMXhNbnpCM05TQStYeGk3ZDVHakExemUzYXFxZXM2bWVONTNYWnFkeDE2N0gKLzNBNld6NjZ4UE9nOHlsUFNVa3R5bU1HNTFkOTFsdTFiZWJYUExtdmc0K3BBeFdhZGJGZ21MR0Z0UE1URXcrWgpIRHZzK3E2NDBIOWJpeitPV2Rld0hjUXE0TW9oQ1dubDhhVzVJYWVSYW1mWS9zZy8xd1NXMkZteGViQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://10.0.4.3:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-controller-manager
  name: system:kube-controller-manager@kubernetes
current-context: system:kube-controller-manager@kubernetes
kind: Config
preferences: {}
users:
- name: system:kube-controller-manager
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUM1ekNDQWMrZ0F3SUJBZ0lJSWN4Ynk4VWEvV1F3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURBNE1qQXdNak13TURWYUZ3MHlNVEE0TWpBd01qTXdNRGRhTUNreApKekFsQmdOVkJBTVRIbk41YzNSbGJUcHJkV0psTFdOdmJuUnliMnhzWlhJdGJXRnVZV2RsY2pDQ0FTSXdEUVlKCktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU5YK0Vqa3c0NDNTNzh5d05LL0dIQTV2eFZCS0Rhbi8KZ21yaUlFTW1hYWhlbDllREJXR0s0dVVtY1VXMXU1TCszeUR0amJlKy83MHZ2M3hvSWY1VkNZQXZqYUorN2twUQpyYW5RUE93cFJnbUlqNTEzV1FsZzMxWDlqREpuNlAybVpYTmZ6YWVOalBwOXdrZGkzZGVqSUZaSm1zYjQ0R3VwCkNrdlpodE5iYUlrVVU1U3dCT3h1dE92Um1uemdHQ3BQa0c4ME9pNWdYcDVzTHJ2dmVYSWxpem5wbHNsa3pxbjQKdWNJMHZMekhQY0JsSWhncEVJOXdCVTFOK3VWLzIxTmRaT3p1UlpFVFRMQ0xmNjhVR0FlM0ZCVXJHblJCUTJJZgpKLzhpNnJVQ2l1T25PQWUvOFNLbzlVM0ExOHN3RDJYandTZVo1NzRRclRGdkFjUjBYQ1BibW4wQ0F3RUFBYU1uCk1DVXdEZ1lEVlIwUEFRSC9CQVFEQWdXZ01CTUdBMVVkSlFRTU1Bb0dDQ3NHQVFVRkJ3TUNNQTBHQ1NxR1NJYjMKRFFFQkN3VUFBNElCQVFBcTY0cVBnVllzRzFGb05QQTRTNlJ0bGwrbUdTVUE2QlVNakQrWkt0eVM1NExCVFZnWQp5K1IrL0Zpd3o2RW1xWUpnZ0EyNWZGdkszSWlGNCt5d3JxeDZETlVZa3BBQkZFWXQ5VjU4a2gxV0pha3BvMEZQCnRZRkFaNmlEMlg4UlBZeUUwSXBMYlFqTGRncS9LYTRiSlhZRFhsS3RTV2UwbmJoY2FUWjRpRm5BcldndmpRQ0sKU05kV0tmSUpGNjJiWGE5a1BGc3ROYWVrWjdoQVZEZzhBbEd1c0tlYVFLdFNLZ2dMREFreElRWjlnNTZSVUprYwp6UUhRVHlibmVTcXJEN3cxT0xIR2RpYmZEYXhzMWdtbi9oL20xNk5ib3NMUlgxNkkxK3VKOWV1d29TWlp3Z29zCmpVRExuWVg1Zm1ZcEdhK2ZDbjdiMTJ4Mzg3SFpmbkE4eTFDTQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFb3dJQkFBS0NBUUVBMWY0U09URGpqZEx2ekxBMHI4WWNEbS9GVUVvTnFmK0NhdUlnUXlacHFGNlgxNE1GCllZcmk1U1p4UmJXN2t2N2ZJTzJOdDc3L3ZTKy9mR2doL2xVSmdDK05vbjd1U2xDdHFkQTg3Q2xHQ1lpUG5YZFoKQ1dEZlZmMk1NbWZvL2FabGMxL05wNDJNK24zQ1IyTGQxNk1nVmttYXh2amdhNmtLUzltRzAxdG9pUlJUbExBRQo3RzYwNjlHYWZPQVlLaytRYnpRNkxtQmVubXd1dSs5NWNpV0xPZW1XeVdUT3FmaTV3alM4dk1jOXdHVWlHQ2tRCmozQUZUVTM2NVgvYlUxMWs3TzVGa1JOTXNJdC9yeFFZQjdjVUZTc2FkRUZEWWg4bi95THF0UUtLNDZjNEI3L3gKSXFqMVRjRFh5ekFQWmVQQko1bm52aEN0TVc4QnhIUmNJOXVhZlFJREFRQUJBb0lCQURCTHVrTXNGSDlpdHZwRQpYbSs1VDRXMmxocXJ5Kyt0R2ZzVGMrS1QzYzdCSXBYaUhTbkpsYkhQL2txVVhIUXRqNkEzM1A4MlhUT09maklPCnNuVmJMZHkvWHNEbzB0RDA2bXpqOFl2L09LNVlJc21RTVFrYjB1dnVZR0RUOE5LbVpra211eHh3cHZ1MXZFNHUKTXhGQzRMNTR1RFRsNElpTHl5WVpQd09lb3JZazlYVi9LSkN4a2g1RnVmZzBublI5MjNXQ1lDZVNyaUVWRm9LbQovbzBKYmlVNE1MU3FxallRWnljRnFSbGM0Vy9sMVJuMldLbU1KZ29EVUE4eEZiOEtJYjk4bGpOR0F0Z2QyNFQwCmcxS1VnbDRNazlPOTEvUzdrbHc3L3dsaHBkY3g0eFJ2dEtBTWZiM0RBa1V4MmpFZDB2ckZvU3NseHM0NXJOc2QKM296ZDhFRUNnWUVBM1p2OGJZTDE0ZlU3c0ZnVXlXekl4ejA3WlJ5czFzZitESmVXRmNCOEZoa2Jpb3Q1T0dqZwp0RHZmQlcvOXliMmtPM3RRNlJxNkFMOFpKcGE1QjcyOVF2YUJ1bDlpRHladVZndC8xUnY1d290Smo1SGZQS25vCnFVNzh6NVdtQUR2VitmQTVXaW9ad0hBVzQ3RHFLUU5OdzYyNWZaZFV3NTFXblZOWHpBZWR2VkVDZ1lFQTl6TjgKU3JrOXlsUlBaZnQ4emgrK05OZndoOFgzRWlZR2JwUHNpWG4zTitxYnQ4eXJORFhNYXRId1NrS2dxWDdxU0twQQpDc3ZGeXRreDhBc2VMaDZhQzBMbXh1aVVtQW8yMnBBU21veDY3VFo1ditqeXdtNGt3TFFXdjh6R0ptMjhyUlRVClZkejMvZC9pTkJHZDlKaHB0dzY3REUvcENPTm9vVWhOOHFwbVQyMENnWUJSYm9vNWE1QVNzZHgzRmthOUpWNDUKNkVRMUNXNXhsaGZDWk1sZndOVllBVzNmWVJUd0o0bTZjTzJvdjloUUU0R1A0ZVovWWJUTHBXMEdnd2dHMGpBRAp0VFZDV041ZGxzK2dpcVUwbUEwVThiM2NKY3dVTEpNejg3UnVTeDB1cE00aUE2WHZmZHpzbThPdGMwcjRPeUNPCk1QNGlLa09aaGUxWDdsSXF4UG12b1FLQmdFdk45UUp4RmJxeTZmb3JDWlduOUVyK0lSdHhvSmRuSTdmTEV0RUIKbnNiOTRheVdUYlhmL1lTUVJuQnZTQmRSL1FRMWVSZ1didHdLaUo3RXVnZUlpTktGUElHb2x0Q2M2VDlTeVBHdAp2SkI3a1JCQm5oZnpjTC9MT2VLdEorSm02bUhsTGt2NlMrNEZOcmVpNDE0N1VzZTQ4N0VOM0RkR2pUSlFHdDhjClUrMXRBb0dCQU1JVzFrcHhGZ1NOUjJORGczdHlJWGNhVDJiQStPTWZrc25nNVdrQUdqb0xveS9laE5waWtJTHAKbHFVVG5oZENaMHBvV3d2MUkxdkZ5VVRJTTREUHd1WVNicnZQNjV2UkJua1M5RGlldVE5Q0FEbXRkT0h1WWR2VgpzSy90cmQ5RTNTdUNVNWNSdXJqVkFacGJoOVNIQzU3bk9rVTRJY2EzT0EvbGZsSmRvbUl0Ci0tLS0tRU5EIFJTQSBQUklWQVRFIEtFWS0tLS0tCg==
```
解析下 `clusters.cluster.certificate-authority-data` 这个证书的内容（需 `base64` 解码），就是 `/etc/kubernetes/pki/ca.crt` 的内容（apiserver 的 crt 证书）。即 `kube-controller-mananger` 的 `kubeconfig` 配置的就是 apiserver 的 CA 证书，`users.user.client-certificate-data` 和 `users.user.client-key-data` 就是 `controller-manager` 用来访问 apiserver 的客户端证书（使用 `/etc/kubernetes/pki/ca.key` 签发的）和秘钥，只不过 kubeconfig 对内容进行了 base64 编码。这个就是整个 controller-manager 和 apiserver 证书认证的方式。

可以使用 `openssl verify` 来验证 `users.user.client-certificate-data` 是否由 `/etc/kubernetes/pki/ca.key` 签发的：
```bash
[root@VM_0_7_centos ~]# openssl verify -CAfile ca.crt client.crt
client.crt: OK
```

####  kube-scheduler
kube-scheduler 负责实现调度策略，分配调度 Pod 到集群内的节点上，它监听 api-server, 查询还未分配的 Node 的 Pod, 然后根据调度策略为这些 Pod 分配到合适的 Node 上：

同理，解析 certificate-authority-data 也是 kubernates 的 CA 证书，client-certificate-data 和 client-key-data 就是 kube-scheduler 用来访问 apiserver 的客户端证书和秘钥
kube-scheduler 也是同样的原理，也是在 yaml 中配置一个 kubeconfig 来进行访问 apiserver

```yaml
[root@VM-4-3-centos kubernetes]# cat scheduler.conf
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EZ3lNREF5TXpBd05Wb1hEVE13TURneE9EQXlNekF3TlZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSndoCmw4ZVd5SlBsSWpwajlTN09VSWRSTWVxV0Mwb2crN3hQemJQZDhzS2NTemZqWjdHc0ttUXlvQjhoQnNlaVVDdUwKai9teVl5Tk02MkxIa0ZKbDI3MXNFWVdmOEtiWS81Y210UmFjRnlMOEpyaTNLQi91eHZnZlEvMXhMK2c3UmRBcQpGQllWRzNtaSs1T1orTExyZlVMUU5qemtoTVllaEhDdHNDRmZJMGF5amJpYk1UUGJLT3lobjV3cHVMZzgvOVdlClNTSnI1TmtnK2R0WHJSZ05YelNpc1JMQVF5MmdEczdOaTN0SklaNjRuRGdIakpyS21HR2dqbEljN1RFdGFUdWcKcnltKy92akVZZ2NxTlhHakY2ekJlT1FXNW5NdUh0K1plYXphZ1QyQTNkUDhGY3lEWVZrSFJVd0RESDBZOVZlcwpOUFAyZnhURzVVZlhWOUV0WVJNQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFEajZLYXVQR2dvVnlGQmdNUzFZYlVFRXFHQmoKN3IwaG5vclNuOVp4dlUxZkM1UkZ0UEd0OEI0YU40T3RMa1REUno5ZmdFc1ZidFdoMXRXWURIWUF6N2FDYkVZawpMRTArRzZQMkpxR043SHlrd05BZFp1QS96emhOdVFKZnhjZG5qVHlIRWZXZyt5OEd1S2JqSU1QdFJVOU45bFpoCkZTeUxsYjNvektYbURDK2RuSHhHMXhNbnpCM05TQStYeGk3ZDVHakExemUzYXFxZXM2bWVONTNYWnFkeDE2N0gKLzNBNld6NjZ4UE9nOHlsUFNVa3R5bU1HNTFkOTFsdTFiZWJYUExtdmc0K3BBeFdhZGJGZ21MR0Z0UE1URXcrWgpIRHZzK3E2NDBIOWJpeitPV2Rld0hjUXE0TW9oQ1dubDhhVzVJYWVSYW1mWS9zZy8xd1NXMkZteGViQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://10.0.4.3:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:kube-scheduler
  name: system:kube-scheduler@kubernetes
current-context: system:kube-scheduler@kubernetes
kind: Config
preferences: {}
users:
- name: system:kube-scheduler
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMzakNDQWNhZ0F3SUJBZ0lJVlUybER1V2Y1OHd3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURBNE1qQXdNak13TURWYUZ3MHlNVEE0TWpBd01qTXdNRGhhTUNBeApIakFjQmdOVkJBTVRGWE41YzNSbGJUcHJkV0psTFhOamFHVmtkV3hsY2pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCCkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU14SzJrNmZnWG50cHVNM2JPZ2ZUS0V4aVhsdzdMQVc2VHpUK2thcndVS2UKK2hKSExWSjF4OUphazlDajZ2VWRZdEdkRzMyd0V1R0VFa3ltN0dFZXlyeHJneGRsU3NyVmRqQkFTYnhwNndpZApvZ3dmL2xVa2kza2FPVUozVXd6bmFnWCt6ZUh1d2hVN0R3NkNuaUpkMy9SZW9hU0FjZitvbDl0TTRiazRldVRrCnRXaUE5SDk0VnlQam42SUpkUDdNb1h4TWpZN1c1UysxNy9aczBwbXJabHhuWFdqZjZESXdyNnplbStSNlF1YnAKeE5adEk1WWdsNDk2a09BaTZMVW5xemhCNHIzaDdDOUd0SjFnVDk1YmxiQ0VZNzRtNmVLREZpNXFwZ3JRZnA0YwoxMlhRYzNtcGQzY2IrZXlGUFNsYUVDUmRwS1BKazRpZXgxNnN4TmwzRmk4Q0F3RUFBYU1uTUNVd0RnWURWUjBQCkFRSC9CQVFEQWdXZ01CTUdBMVVkSlFRTU1Bb0dDQ3NHQVFVRkJ3TUNNQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUIKQVFBWFovVTcxSnRqQXQ3MjJLeVl6Q1RDZlF1bHdMM2EySGN6NGw5NXVaMFNWVG5ncTNhWUJxeVdwQ2puM3VNaApTaGN5OUZ4ZC92am52YXVTWUdXY05abm84dEVNUFhTaitNNzI5bW1vTUNUa0xCUGJSVGZwRGt3aDNnRS9IRWtuCnN0emRoZTZ3dWp4OWduMXl5WTJSOTFTZ3U3cjdwZjlLM1hOeFh2SFo3Z0tDQnJIVisyMVlQTkNCaC8rYlVuZkcKY2pvNlNNZHphT0Y5SlJod2pUS0l5VTlkeXJkbFBLUlR0Q3NGVEttdy9HM1d4Z1gvbGRCZnNsZmNaVXR4TlpsYQpablBVNlYvK3gwelBTVG56RzRmYTQ3UkhlZmc3YzczQkZjL0ZiYW9obmhrZHNPMVBNWGdhSjQ1bGo1NVNPL1phCmlIbUphZUF3bnh5d0hMazFtclE1b0ttVAotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcEFJQkFBS0NBUUVBekVyYVRwK0JlZTJtNHpkczZCOU1vVEdKZVhEc3NCYnBQTlA2UnF2QlFwNzZFa2N0ClVuWEgwbHFUMEtQcTlSMWkwWjBiZmJBUzRZUVNUS2JzWVI3S3ZHdURGMlZLeXRWMk1FQkp2R25yQ0oyaURCLysKVlNTTGVSbzVRbmRURE9kcUJmN040ZTdDRlRzUERvS2VJbDNmOUY2aHBJQngvNmlYMjB6aHVUaDY1T1MxYUlEMApmM2hYSStPZm9nbDAvc3loZkV5Tmp0YmxMN1h2OW16U21hdG1YR2RkYU4vb01qQ3ZyTjZiNUhwQzV1bkUxbTBqCmxpQ1hqM3FRNENMb3RTZXJPRUhpdmVIc0wwYTBuV0JQM2x1VnNJUmp2aWJwNG9NV0xtcW1DdEIrbmh6WFpkQnoKZWFsM2R4djU3SVU5S1ZvUUpGMmtvOG1UaUo3SFhxekUyWGNXTHdJREFRQUJBb0lCQVFDbmJhVlRFSGlkdy82bApjMVJIUFBlaG1DYXlKN0ZqYzdOOWpjRXRVREJZZUZBODBLYTlVUmdPTnZ1ejM5TjlSYk1xVlpjbFFEdUpKYU9WCnZLdzN3SE9wVG5lbW9mWlZHL0w4QW9RcjdhYVpiZzlUM3BpamtRclptbnRaRk5BMDRDZk5lQkdsMi9hbVRidSsKU2FCdVMvOXltR2ZqbVAxVTZRaGp5N09uQ0RuNEFsMmw1SXJpUCtlSTgxS3ZjVjRWUWNhcmtQL0F5SityQm56SgpSZjNRSFBRL2pFaUVRMm9kKzQ4N1FzSWNZVlcwZ0FmQ3pKN05NMUJMeTE0Yy8zUFJRS3JLZTdXT3AwM2Z3QmkxCk5TRlc1dmxSUkpnTDduSHQ4TkdsSUhHRE5VT29KR0UvVnppekJUVFpQRS9nOEkxZ1FLYko5b3ZSdXJQa0J6VGsKRHNGQ281bEJBb0dCQU5qalNTVXIybFo0Z1B6MEZ6bnBMTEFBaVhBSDkwZFlBb1BQSUhoWDR2WmtRUjE4Ykx2NgoraXVSR2dMTkQyckV4dHk3dE1tcTQrZkt2S1VXRWorOVR0KzNqWTVmTFdCeWhNTk1uaXN2eDFsdkxlZnFybkRvCnlkODdPb0p3TnZZdit2YitQR3NGaU51SHdXUTR4Wit3WTFaYitCUnB5UVJNUEs1TnVEbDRFMXZQQW9HQkFQRWkKRitwS1VJSmE0NGZuWDk3L1lHalh6Z1lWTEQ4RkRRMTh4dmY3TG43UEhUNzJoK1VCaXJFV29uK2RmcDFBZS95UwpTMER6Q2ZLUDdiM0R3YkxPbmRKcHdLWnUrUjdBaEs5RGFlVEJ3Y2FkRDFpTzNoME5RSVFoVFJPaVJRclN4RWpmCllXVnRmUXFuSUJhS3pMSnRCakVtakRDcXJ2QjJ3QTRza1Z5d09WZWhBb0dBTWZWb3k5OG1FL1QrQVVaWWMwWjYKdksvaStLTmRHbG56ZWxranFaVFUrdHh0QTFXOTFpOGhvUmR6WG1ITncxSkFYR2dBWk5Pd1c1d2ZpQWRsZkxrbQppZkhGOFoySzNrU0N3Rm5OdFRUMFBtMlZyVzRwY0dpdTEzVFZMV2Fid21tYTdYbnlnTlJ0aWVQamNDcURteDBPClJMNDZqcmt2VElZakZDTmk1Qm44bTVFQ2dZQmNHdUs1cW1Nd041bGJpd1J5d0dkS0JNeDhSRkFmVGtXYkZrTkYKNjVycDh5Qy9zUmxkWHdaaitEcGZ0bi9yZnZzZEVhQlBFY2FGOFhZbEd3WDh6N0UyOHhBVVFxVkRtdFBUd2xOTApmcnNPcTJWMk5UUWdNclNuQTdWV1A1QlJ2d29jcjc2YktJUXZzb0N1TzV4T3R4ZzdZL2IraStQQWxBdHVIcFh6CnFwaHNvUUtCZ1FERkxITzFwTTNPNlRWN3cybThKWVI0WGxBUWtLZkRPMlFGaDB0bGM1bk1rZUdZbHZFUUlZdVMKS2liV3NJNHVwMHFRcFZjdHF2VU9wc2V1Rk5ZdGVRQzF6YncxNWp4a0xEMm9Gb2c1Yk9WRXk3ekZERU1kVmdpRwpEbjhkbHN3SWp0bUF1SDFGOWdBbGR1V1M0cXkyV0I0SlRPZjBlTDVOM1dTWkRzcm91anA5NlE9PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

#### kube-proxy
kube-proxy 保证集群内的服务，可以被集群外访问到。
每台工作节点上都应该运行一个 kube-proxy 服务，它监听 API server 中 service 和 endpoint 的变化情况，并通过 iptables 等来为服务配置负载均衡，是让我们的服务在集群外可以被访问到的重要方式。

在这里我们并未发现 kube-proxy 的 kubeconfig，kube-proxy 也是需要访问 apiserver 的，那么是如何进行认证的。还是从 yaml 文件进行分析一下，如果需要认证，肯定会在 yaml 中配置对应的证书或者包含证书的文件或 token

```yaml
[root@VM-4-3-centos run]# kubectl get  pod kube-proxy-6bf2t -n kube-system -o yaml
  containers:
  - command:
    - /usr/local/bin/kube-proxy
    - --config=/var/lib/kube-proxy/config.conf
    - --hostname-override=$(NODE_NAME)
    env:
    - name: NODE_NAME
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: spec.nodeName
    image: k8s.gcr.io/kube-proxy:v1.15.2
    imagePullPolicy: IfNotPresent
    name: kube-proxy
    resources: {}
    securityContext:
      privileged: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/lib/kube-proxy
      name: kube-proxy
    - mountPath: /run/xtables.lock
      name: xtables-lock
    - mountPath: /lib/modules
      name: lib-modules
      readOnly: true
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-proxy-token-rd92l
      readOnly: true
  dnsPolicy: ClusterFirst
  .....
  volumes:
  - configMap:
      defaultMode: 420
      name: kube-proxy
    name: kube-proxy
  - hostPath:
      path: /run/xtables.lock
      type: FileOrCreate
    name: xtables-lock
  - hostPath:
      path: /lib/modules
      type: ""
    name: lib-modules
  - name: kube-proxy-token-rd92l
    secret:
      defaultMode: 420
      secretName: kube-proxy-token-rd92l
```

`kube-proxy` 挂载一个 `secret`，通过 `kubectl get secret -n kube-system kube-proxy-token-rd92l -o yaml` 这边猜测这个就是用来进行认证的。我们查看下这个 token
```yaml
apiVersion: v1
data:
  ca.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EZ3lNREF5TXpBd05Wb1hEVE13TURneE9EQXlNekF3TlZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSndoCmw4ZVd5SlBsSWpwajlTN09VSWRSTWVxV0Mwb2crN3hQemJQZDhzS2NTemZqWjdHc0ttUXlvQjhoQnNlaVVDdUwKai9teVl5Tk02MkxIa0ZKbDI3MXNFWVdmOEtiWS81Y210UmFjRnlMOEpyaTNLQi91eHZnZlEvMXhMK2c3UmRBcQpGQllWRzNtaSs1T1orTExyZlVMUU5qemtoTVllaEhDdHNDRmZJMGF5amJpYk1UUGJLT3lobjV3cHVMZzgvOVdlClNTSnI1TmtnK2R0WHJSZ05YelNpc1JMQVF5MmdEczdOaTN0SklaNjRuRGdIakpyS21HR2dqbEljN1RFdGFUdWcKcnltKy92akVZZ2NxTlhHakY2ekJlT1FXNW5NdUh0K1plYXphZ1QyQTNkUDhGY3lEWVZrSFJVd0RESDBZOVZlcwpOUFAyZnhURzVVZlhWOUV0WVJNQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFEajZLYXVQR2dvVnlGQmdNUzFZYlVFRXFHQmoKN3IwaG5vclNuOVp4dlUxZkM1UkZ0UEd0OEI0YU40T3RMa1REUno5ZmdFc1ZidFdoMXRXWURIWUF6N2FDYkVZawpMRTArRzZQMkpxR043SHlrd05BZFp1QS96emhOdVFKZnhjZG5qVHlIRWZXZyt5OEd1S2JqSU1QdFJVOU45bFpoCkZTeUxsYjNvektYbURDK2RuSHhHMXhNbnpCM05TQStYeGk3ZDVHakExemUzYXFxZXM2bWVONTNYWnFkeDE2N0gKLzNBNld6NjZ4UE9nOHlsUFNVa3R5bU1HNTFkOTFsdTFiZWJYUExtdmc0K3BBeFdhZGJGZ21MR0Z0UE1URXcrWgpIRHZzK3E2NDBIOWJpeitPV2Rld0hjUXE0TW9oQ1dubDhhVzVJYWVSYW1mWS9zZy8xd1NXMkZteGViQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
  namespace: a3ViZS1zeXN0ZW0=
  token: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklpSjkuZXlKcGMzTWlPaUpyZFdKbGNtNWxkR1Z6TDNObGNuWnBZMlZoWTJOdmRXNTBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5dVlXMWxjM0JoWTJVaU9pSnJkV0psTFhONWMzUmxiU0lzSW10MVltVnlibVYwWlhNdWFXOHZjMlZ5ZG1salpXRmpZMjkxYm5RdmMyVmpjbVYwTG01aGJXVWlPaUpyZFdKbExYQnliM2g1TFhSdmEyVnVMWEprT1RKc0lpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkltdDFZbVV0Y0hKdmVIa2lMQ0pyZFdKbGNtNWxkR1Z6TG1sdkwzTmxjblpwWTJWaFkyTnZkVzUwTDNObGNuWnBZMlV0WVdOamIzVnVkQzUxYVdRaU9pSmhOemRrTjJKaE1TMW1Zek5pTFRRM1lUTXRZV00wWkMweVpXRmtPVEV6WkRVd09ESWlMQ0p6ZFdJaU9pSnplWE4wWlcwNmMyVnlkbWxqWldGalkyOTFiblE2YTNWaVpTMXplWE4wWlcwNmEzVmlaUzF3Y205NGVTSjkuSTRuR0UxOVhJakFPU0lKcWZyb3A2azhHcXBickxBeVFzQ3NoeFhxMEc3RklTZmJudS1TTW9xV1pHUjU0S2hwREdlaGd6WkQwMGVGZG14bEM1ZzBIc2ZzZE40V0tmVFI1ZjY1b3kzTnVvWUxxcUIzUzgySUxLelJHREVBNHpwWmFXeG1lRmtzdU1mdl9UWDRjSGdtYUI3V0ZQZzJ5RWtxV0VPa3kwT0hOWnIxNmd4Mzl3S1owWDRhQ29FOVd0cGlZU1BKYU5SdmtVbENfNTlPZHJTYnBCYnlkd2JOaWVaRjdhcWRBbFdWQ3JXQkRfWmlCaHNnZklVYUpEcVg5TWtRbUpjVS1Yb2pzWUpXNFpNejZ3OEZFTHY4THpCazRLTUc5V185aG5Jc3FfVlFUM2xDek5iSHlNSktWeXZ1VlVrblo5X3AwaTJGQlpDeGVVdlpVazdrd01R
kind: Secret
metadata:
  annotations:
    kubernetes.io/service-account.name: kube-proxy
    kubernetes.io/service-account.uid: a77d7ba1-fc3b-47a3-ac4d-2ead913d5082
  creationTimestamp: "2020-08-20T02:30:48Z"
  name: kube-proxy-token-rd92l
  namespace: kube-system
  resourceVersion: "196"
  selfLink: /api/v1/namespaces/kube-system/secrets/kube-proxy-token-rd92l
  uid: c9ff07a0-4176-4053-a93c-11c7d0aff285
type: kubernetes.io/service-account-token
```
从上面 token 的内容，这个里面包含一个 CA 证书是 kubernates 查到 ca 证书，token 就是用来和 apiserver 通讯的客户端证书和秘钥


####  kubelet（工作节点）
`kubelet` 会在每个工作节点上都运行一个 `kubelet` 服务进程，默认监听 10250 端口。`kubelet` 负责接收并执行 `master` 发来的指令，管理 Pod 以及 Pod 中的容器。每个 `kubelet` 进程会在 APIServer 上注册节点自身信息，定期向 master 节点汇报节点的资源使用情况，并通过 cAdvisor 监控节点和容器的资源。kubelet 也是用 `kubeconfig` 来进行（与 APIServer）认证的，都是用 kubernates APIServer 的 CA 生成。看下 `kubelet` 的配置，也是使用 ``


1、master 节点的 kubelet<br>
```yaml
[root@VM-4-3-centos kubernetes]# cat kubelet.conf
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EZ3lNREF5TXpBd05Wb1hEVE13TURneE9EQXlNekF3TlZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSndoCmw4ZVd5SlBsSWpwajlTN09VSWRSTWVxV0Mwb2crN3hQemJQZDhzS2NTemZqWjdHc0ttUXlvQjhoQnNlaVVDdUwKai9teVl5Tk02MkxIa0ZKbDI3MXNFWVdmOEtiWS81Y210UmFjRnlMOEpyaTNLQi91eHZnZlEvMXhMK2c3UmRBcQpGQllWRzNtaSs1T1orTExyZlVMUU5qemtoTVllaEhDdHNDRmZJMGF5amJpYk1UUGJLT3lobjV3cHVMZzgvOVdlClNTSnI1TmtnK2R0WHJSZ05YelNpc1JMQVF5MmdEczdOaTN0SklaNjRuRGdIakpyS21HR2dqbEljN1RFdGFUdWcKcnltKy92akVZZ2NxTlhHakY2ekJlT1FXNW5NdUh0K1plYXphZ1QyQTNkUDhGY3lEWVZrSFJVd0RESDBZOVZlcwpOUFAyZnhURzVVZlhWOUV0WVJNQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFEajZLYXVQR2dvVnlGQmdNUzFZYlVFRXFHQmoKN3IwaG5vclNuOVp4dlUxZkM1UkZ0UEd0OEI0YU40T3RMa1REUno5ZmdFc1ZidFdoMXRXWURIWUF6N2FDYkVZawpMRTArRzZQMkpxR043SHlrd05BZFp1QS96emhOdVFKZnhjZG5qVHlIRWZXZyt5OEd1S2JqSU1QdFJVOU45bFpoCkZTeUxsYjNvektYbURDK2RuSHhHMXhNbnpCM05TQStYeGk3ZDVHakExemUzYXFxZXM2bWVONTNYWnFkeDE2N0gKLzNBNld6NjZ4UE9nOHlsUFNVa3R5bU1HNTFkOTFsdTFiZWJYUExtdmc0K3BBeFdhZGJGZ21MR0Z0UE1URXcrWgpIRHZzK3E2NDBIOWJpeitPV2Rld0hjUXE0TW9oQ1dubDhhVzVJYWVSYW1mWS9zZy8xd1NXMkZteGViQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://10.0.4.3:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: system:node:vm-4-3-centos
  name: system:node:vm-4-3-centos@kubernetes
current-context: system:node:vm-4-3-centos@kubernetes
kind: Config
preferences: {}
users:
- name: system:node:vm-4-3-centos
  user:
    client-certificate-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMrVENDQWVHZ0F3SUJBZ0lJUW92eFg0d2RyTmd3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TURBNE1qQXdNak13TURWYUZ3MHlNVEE0TWpBd01qTXdNRGRhTURzeApGVEFUQmdOVkJBb1RESE41YzNSbGJUcHViMlJsY3pFaU1DQUdBMVVFQXhNWmMzbHpkR1Z0T201dlpHVTZkbTB0Ck5DMHpMV05sYm5SdmN6Q0NBU0l3RFFZSktvWklodmNOQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU1scld4Y3kKOEhyT0FPeXNGcjJxUXhOdGZ4QllBMWJYZ1BkRy95eW9rWjVLdm1nc0NjbGh2MWkyaWF2bzF6T1d4QkNZSHlJZQpyK0hSdDQ5TFFzcjZIU2k3QlZveEtuWm8rekpjSFBIVHk0VUtESVYrOUxUU29INDA3akhXeGFOWXc0RWw2QVB0Ck92UzcvcXVBb3BBOFNPVVU2YTdWcFc2OHRvNVJtRGNwcGVMZnlmL3dXdXRpZFZ0dTRmTE5RK3gwRnBGRWZXMisKY3ZHNWd2ZlZnaldaUnhzbXoyWlFESEo0NStSWldmVnF6bHdjYVlkMytROTFiK1BmUHhlVU81bmdRMGtpbTVTbAp6K1c4Q1p6Q01UOUF6QVU2USt1L1oyKzlsaU9xbzdxWnEySHZMMms5R05GNjVHcVVnWWpDT3hVRFpoanZ5RkxQCjNvdHR0ZTJQeHZkMXJRa0NBd0VBQWFNbk1DVXdEZ1lEVlIwUEFRSC9CQVFEQWdXZ01CTUdBMVVkSlFRTU1Bb0cKQ0NzR0FRVUZCd01DTUEwR0NTcUdTSWIzRFFFQkN3VUFBNElCQVFBcHpFNkxJU25jZFE1di9yZ2ZJbHRQV2l3Lwp4WjFQdHk2WWZTYmw2UXlNY2g1Qk1RNDhyL3o1ZXV5V21kVXhVUmFHaUlldUVaODAxQ2xteitoRjQ2RFRyZGlaCmZmTUNLcXE5Z3lKbkRBT1VYMkpkY3RZb2IvNWp0T0t0ejVYYmRrMzI5ZG01cUVEQ1gzRm44M3NWcWJPZnRXUnQKM2NvdFdUYjM4cmljQm5sRTRwZjBUdXk5RVZIZFBFazgyOUh0UVdDc3JDVFNQZXZ0S29oM01mdUZ0ek9leXJsMQpDSXFjNXVDM0k1dU1xS0tBeFNvSk1Pd1pVUklrRkpTQ09lRlRRVnd1ZE9vVi92STcrR0NBcVBObS9IUVRlNUROCkdPUXVWNTRMYWFkRzVRSHdDOEJhWDZTc0JQb09KOG5vWlM4cllSSERjMnFBcUYzMkhReHlXVHc1aE53dgotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    client-key-data: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlFcFFJQkFBS0NBUUVBeVd0YkZ6THdlczRBN0t3V3ZhcERFMjEvRUZnRFZ0ZUE5MGIvTEtpUm5rcSthQ3dKCnlXRy9XTGFKcStqWE01YkVFSmdmSWg2djRkRzNqMHRDeXZvZEtMc0ZXakVxZG1qN01sd2M4ZFBMaFFvTWhYNzAKdE5LZ2ZqVHVNZGJGbzFqRGdTWG9BKzA2OUx2K3E0Q2lrRHhJNVJUcHJ0V2xicnkyamxHWU55bWw0dC9KLy9CYQo2MkoxVzI3aDhzMUQ3SFFXa1VSOWJiNXk4Ym1DOTlXQ05abEhHeWJQWmxBTWNuam41RmxaOVdyT1hCeHBoM2Y1CkQzVnY0OTgvRjVRN21lQkRTU0tibEtYUDVid0puTUl4UDBETUJUcEQ2NzluYjcyV0k2cWp1cG1yWWU4dmFUMFkKMFhya2FwU0JpTUk3RlFObUdPL0lVcy9laTIyMTdZL0c5M1d0Q1FJREFRQUJBb0lCQUIwTXFycVIwalVqK09ZcAplNjRuSEQxMUVWcGVGejB6SDVxS1Zzc3VGTEpydlVKdzk0aGYzS1VDenFCSW1LRU1JWUx6TGFwU0dyUEs5MXBuClZGN0o2K0t2OW5tbmxhUTJSK1JmZkowMEdxbzVaTXpzSG9ibHlkZnA4bUNseFNObDdleDJkeHY1M3dMbENqbloKOTVndDJhV1FlcE9JcEs5djhEUmVlRUdjZEJ4Z1FMRm5nTU5SR3NzQ0lOSUt6Wkc5aDlwd2htZTRGUW9sRUlNMwp1ZkxkbzVZM0UyeFk3bkV4TTYvTnEvNUVQL1k2S1NGQlJwaXBxeWF2MERIaEVoaU9HNGtCS1l2ajZwdWdXcDgrCkNvcFZ0NWtNWjlwSWx3UElnT1UwQzB3L3Y5K0M2S0hIbzZUZExlTE9GeDlKMER4bXluMWJRMENvTGFTUXRiUUkKOG12UHFuRUNnWUVBOG92a0l1YjBUSHZlQVlSUG1hT1RMVFg4QkZiWWMwUEdhNDljMlNsaU95QmV1M0dxSEwzQgpWT3JKRXUwbVdQQU5pMDIrR0lpZG9MejFUTC82dlRmTkR6ajZRTWFRNUJsZVVKdUZlOGxuMEliUG5jbjZzN0MzCkp5eGZRMzA1emFONFJjWXQyQWZrZkg1bnZpK3d1YUpXbzAySUQzOXczdU1BMkNFZFdGbVdBMzhDZ1lFQTFKZDMKOUk0dktLZFFhNytKZkFLWG9mYWd2cUtWNnRHSkVPWjE2NXJNS1FTUmJvcDM3UXhmQ2dLRXpKMEI0U1p5eGFZYgpLdWJTZ1hjQXVrbER6TVpqc2ZsL0VEMmhDcFFQSGhnVUNNTGVQUkl3SDJWM2gvZk9nMlU3SEtFdHMvSXNsdlVRCk1JWDJpTnQ5MnUyU3AvUzBsMWowTitndjZ3M0R6S2xHS2dUWWMzY0NnWUVBcTJ3aGxtVmkvbmVCUmRNc3F5ckkKQjJrak1ESHRFekl3bDY2Z2NiOWs5T01BOFR2NWZnekRDbkJTSXJWSHFBNHBsRzRpejVZbXlnY2kyOWJIc1ZveAo3UE5aTTlUamJNTmRQRjFlcjBsK3ZRdTZ5d3VJeTkwMjVWSGdGb1A0Q1pYaW1IWGp5czV4TjJmampMQ0tGL2xiCmdGbDRzM05mNDdmT3pmSkJta0xlMnFNQ2dZRUFsdEtMRU41YXlLM0RHVjQrek5NTi9xTDVNYVlwVS9tcUUycGQKR0hTdkNSNnJpdEFEK3hIK3p4d3dXUFcrNHB3amF1UElmR3hieGV2R2dXTC9EZVZsejFzaGNVVTMza2hpWFVoWgoxa2xoMzlQcWZpdS9YS0JMUzk3aXpCSHhXYXVqUk1uQjNacjg1K1ZJYWF5SWtrM0NYV21IZ2E1aGFKSlFhZjloCnZ1ZkhKRXNDZ1lFQW9wSnQzSGdzS3R6VSs2YkMzdEJwQkEweXFZQ09yTkFQRjV5SWJzZnZXRmM1WENpc25xZE4KUG9hNzBJTVJwWFhCREo1V01lVWhiVVQzbHRtNWppV3dkVGp1WTJSYWorNVhZeHplN1NpWW9Dak9xckdua0NoLwppUG1pb21wNGViUU9DS3A3WTN5cG0rdy9RcENSN2ZBUEI1NkZVWnRZbktEbEs3RHlsWjFIQ0s0PQotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQo=
```

这边我们会给每个节点生成一份客户端的证书和私钥，master 上用的是 kubelet.conf ，节点上的 kubelet.conf 如下，直接指向一个 kubelet-client-current.pem 文件，这里包含了证书和私钥，每一个节点都不一样。因此每个节点都会有一个自己的客户端证书和私钥。

2、worker 节点上的 kubelet
```yaml
[root@VM-4-9-centos ~]# cat /etc/kubernetes/kubelet.conf
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EZ3lNREF5TXpBd05Wb1hEVE13TURneE9EQXlNekF3TlZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSndoCmw4ZVd5SlBsSWpwajlTN09VSWRSTWVxV0Mwb2crN3hQemJQZDhzS2NTemZqWjdHc0ttUXlvQjhoQnNlaVVDdUwKai9teVl5Tk02MkxIa0ZKbDI3MXNFWVdmOEtiWS81Y210UmFjRnlMOEpyaTNLQi91eHZnZlEvMXhMK2c3UmRBcQpGQllWRzNtaSs1T1orTExyZlVMUU5qemtoTVllaEhDdHNDRmZJMGF5amJpYk1UUGJLT3lobjV3cHVMZzgvOVdlClNTSnI1TmtnK2R0WHJSZ05YelNpc1JMQVF5MmdEczdOaTN0SklaNjRuRGdIakpyS21HR2dqbEljN1RFdGFUdWcKcnltKy92akVZZ2NxTlhHakY2ekJlT1FXNW5NdUh0K1plYXphZ1QyQTNkUDhGY3lEWVZrSFJVd0RESDBZOVZlcwpOUFAyZnhURzVVZlhWOUV0WVJNQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFEajZLYXVQR2dvVnlGQmdNUzFZYlVFRXFHQmoKN3IwaG5vclNuOVp4dlUxZkM1UkZ0UEd0OEI0YU40T3RMa1REUno5ZmdFc1ZidFdoMXRXWURIWUF6N2FDYkVZawpMRTArRzZQMkpxR043SHlrd05BZFp1QS96emhOdVFKZnhjZG5qVHlIRWZXZyt5OEd1S2JqSU1QdFJVOU45bFpoCkZTeUxsYjNvektYbURDK2RuSHhHMXhNbnpCM05TQStYeGk3ZDVHakExemUzYXFxZXM2bWVONTNYWnFkeDE2N0gKLzNBNld6NjZ4UE9nOHlsUFNVa3R5bU1HNTFkOTFsdTFiZWJYUExtdmc0K3BBeFdhZGJGZ21MR0Z0UE1URXcrWgpIRHZzK3E2NDBIOWJpeitPV2Rld0hjUXE0TW9oQ1dubDhhVzVJYWVSYW1mWS9zZy8xd1NXMkZteGViQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://10.0.4.3:6443
  name: default-cluster
contexts:
- context:
    cluster: default-cluster
    namespace: default
    user: default-auth
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: default-auth
  user:
    client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem
    client-key: /var/lib/kubelet/pki/kubelet-client-current.pem
```

注意：worker 节点上的 `/var/lib/kubelet/pki/kubelet-client-current.pem` 的秘钥及证书和 master 节点上的是不一样的。


####  关于 kubelet 的疑问
现在有一个问题就是，<font color="#dd0000">kubernetes 中节点可能有上万个，那么是如何快速给节点自动生成客户端证书和秘钥 </font> 然后配置给每个 worker 节点上的 `kubelet` 呢？答案就是 [TLS BOOTSTRAPPING 机制](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/)，该机制大大简化了 kubelet 节点客户端证书的生成步骤（有点类似于 Access Token/Json Web Token）：

```JAVASCRIPT
[root@VM-4-11-centos pki]# ps -ef | grep kubelet
root     14746     1  0 Aug20 ?        17:50:19 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --cgroup-driver=cgroupfs --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1
```

查看 `kubelet` 进程可以发现其加载了一个 `kubeconfig` 是 `bootstrap-kubelet.conf`，我们来看看这个文件配置的是什么内容？

```YAML
[root@VM-4-11-centos pki]# cat /etc/kubernetes/bootstrap-kubelet.conf
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUN5RENDQWJDZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJd01EZ3lNREF5TXpBd05Wb1hEVE13TURneE9EQXlNekF3TlZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBSndoCmw4ZVd5SlBsSWpwajlTN09VSWRSTWVxV0Mwb2crN3hQemJQZDhzS2NTemZqWjdHc0ttUXlvQjhoQnNlaVVDdUwKai9teVl5Tk02MkxIa0ZKbDI3MXNFWVdmOEtiWS81Y210UmFjRnlMOEpyaTNLQi91eHZnZlEvMXhMK2c3UmRBcQpGQllWRzNtaSs1T1orTExyZlVMUU5qemtoTVllaEhDdHNDRmZJMGF5amJpYk1UUGJLT3lobjV3cHVMZzgvOVdlClNTSnI1TmtnK2R0WHJSZ05YelNpc1JMQVF5MmdEczdOaTN0SklaNjRuRGdIakpyS21HR2dqbEljN1RFdGFUdWcKcnltKy92akVZZ2NxTlhHakY2ekJlT1FXNW5NdUh0K1plYXphZ1QyQTNkUDhGY3lEWVZrSFJVd0RESDBZOVZlcwpOUFAyZnhURzVVZlhWOUV0WVJNQ0F3RUFBYU1qTUNFd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0RRWUpLb1pJaHZjTkFRRUxCUUFEZ2dFQkFEajZLYXVQR2dvVnlGQmdNUzFZYlVFRXFHQmoKN3IwaG5vclNuOVp4dlUxZkM1UkZ0UEd0OEI0YU40T3RMa1REUno5ZmdFc1ZidFdoMXRXWURIWUF6N2FDYkVZawpMRTArRzZQMkpxR043SHlrd05BZFp1QS96emhOdVFKZnhjZG5qVHlIRWZXZyt5OEd1S2JqSU1QdFJVOU45bFpoCkZTeUxsYjNvektYbURDK2RuSHhHMXhNbnpCM05TQStYeGk3ZDVHakExemUzYXFxZXM2bWVONTNYWnFkeDE2N0gKLzNBNld6NjZ4UE9nOHlsUFNVa3R5bU1HNTFkOTFsdTFiZWJYUExtdmc0K3BBeFdhZGJGZ21MR0Z0UE1URXcrWgpIRHZzK3E2NDBIOWJpeitPV2Rld0hjUXE0TW9oQ1dubDhhVzVJYWVSYW1mWS9zZy8xd1NXMkZteGViQT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQo=
    server: https://10.0.4.3:6443 #apiserver 的地址
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: tls-bootstrap-token-user
  name: tls-bootstrap-token-user@kubernetes
current-context: tls-bootstrap-token-user@kubernetes
kind: Config
preferences: {}
users:
- name: tls-bootstrap-token-user
  user:
    token: xxxxx.xxxxxx.token
```
简单描述下 TLS bootstrapping 机制以及采用 TLS bootstrapping 生成证书的流程：

> Kubernetes 提供了 TLS bootstrapping 的方式来简化 Kubelet 证书的生成过程，其原理是预先提供一个 bootstrapping token，kubelet 通过该 kubelet 调用 kube-apiserver 的证书签发 API 来生成 自己需要的证书。要启用该功能，需要在 kube-apiserver 中启用 --enable-bootstrap-token-auth ，并创建一个 kubelet 访问 kube-apiserver 使用的 bootstrap token secret。如果使用 kubeadmin 安装，可以使用 kubeadm token create 命令来创建 token。

采用 TLS bootstrapping 机制生成 worker 节点证书的流程如下：

1.  调用 kube-apiserver 生成一个 bootstrap token（也需要防止泄露）
2.  将该 bootstrap token 写入到一个 kubeconfig 文件中，作为 kubelet 调用 kube-apiserver 的客户端验证方式
3.  通过 --bootstrap-kubeconfig 启动参数将 bootstrap token 传递给 kubelet 进程
4.  Kubelet 采用 bootstrap token 调用 kube-apiserver API，生成自己所需的 key 和客户端证书
5.  证书生成后，Kubelet 采用生成的证书和 kube-apiserver 进行通信，并删除本地的 kubeconfig 文件，以避免 bootstrap token 泄漏风险

####  Service Account 认证
如果我们需要在某个 Pod 中（调用）访问 apiserver 的接口，怎么做？答案就是 Service Account。前面有提到在 `kube-apiserver` 和 `kube-controller-manager` 分配配置的 sa 的公钥 `sa.pub` 和私钥 `sa.key`，这里就是用来对 Service Account 来进行认证的，我们一般在 RBAC 中来限制 service account 的访问限制。kubernetes 会为该 service account 生成一个 JWT token，并使用 secret 将该 service account token 加载到 pod 上。pod 中的应用可以使用 service account token 来访问 api server。service account 证书被用于生成和验证 service account token。该证书的用法和前面介绍的其他证书不同，因为实际上使用的是其公钥和私钥，而并不需要对证书进行验证。下面是 service account 的认证方式

![kubernetes-service-accout-auth](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kubernetes/kubernetes-service-account.png)



##  0x06  开源的 cert 管理工具
[](https://github.com/jetstack/cert-manager)
![cert-manager](https://camo.githubusercontent.com/94e6e2096b0bc286c36b61494276534d8f70f5e7e6171587c65832f2c621f688/68747470733a2f2f636572742d6d616e616765722e696f2f696d616765732f686967682d6c6576656c2d6f766572766965772e737667)

##  0x07  参考
-   [一文带你彻底厘清 Kubernetes 中的证书工作机制](https://zhaohuabing.com/post/2020-05-19-k8s-certificate/)
-   [Kubernetes 数字证书体系浅析](https://www.secrss.com/articles/13238)
-   [关于 Kubernetes 证书的那点事](https://cloud.tencent.com/developer/article/1744610)