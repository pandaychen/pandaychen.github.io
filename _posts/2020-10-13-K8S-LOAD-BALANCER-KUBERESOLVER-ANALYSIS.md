---
layout:     post
title:      Kubernetes 应用改造（六）：使用原生 API 实现 gRPC 负载均衡
subtitle:   基于 kubernetes-API 实现 gRPC-LB：kuberesolver
date:       2020-10-13
author:     pandaychen
header-img:	img/super-mario.jpg
catalog:    true
tags:
    - Kubernetes
---

##  0x00  前言
上一篇文章聊到了 Kubernetes 中的负载均衡实现：[Kubernetes 应用改造（三）：负载均衡](https://pandaychen.github.io/2020/06/01/K8S-LOADBALANCE-WITH-KUBERESOLVER/)，其中提到了一款开源项目：[kuberesolver](https://github.com/sercand/kuberesolver)，这篇文章就来分析下它的实现。

##	0x01	Kubernetes 相关概念

了解 Kubernetes 的朋友应该知道，可以通过 Kubernetes APIserver 提供的 [REST API](https://k8smeetup.github.io/docs/tasks/administer-cluster/access-cluster-api/) 直接操作 Kubernetes 的内置资源（如 `Pod`、`Service`、`Deployment`、`ReplicaSet`、`StatefulSet`、`Job`、`CronJob` 等），当然前提是操作方（如某个业务 `Pod` 容器）需要有相关的接口操作权限。参见 [一文带你彻底厘清 Kubernetes 中的证书工作机制](https://zhuanlan.zhihu.com/p/142990931)


####	APIServer
在 Kubernetes 架构中，Etcd 存储位于 APIServer 之后，集群内的各种组件均需要需要统一经过 APIServer 做身份认证和鉴权等安全控制，即认证 + 授权，准入后才可以访问对应的资源数据（存储在 Etcd 中）。

-	`GET`（SELECT）：从服务器读取资源，`GET` 请求对应 Kubernetes API 的获取信息功能。需要获取信息时需要使用 GET 请求。
-	`POST`（CREATE）：在服务器新建一个资源。`POST` 请求对应 Kubernetes API 的创建功能。需要创建 Pods、ReplicaSet 或者 Service 的时候请使用这种方式发起请求。
-	`PUT`（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源），对应更新 Nodes 或 Pods 的状态、ReplicaSet 的自动备份数量等。
-	`PATCH`（UPDATE）：在服务器更新资源（客户端提供改变的属性）
-	`DELETE`（DELETE）：删除资源，如删除 Pods 等

##	0x02	Kubernetes 的 List-Watch 机制
Kubernetes 提供的 Watch 功能是建立在对 Etcd 的 Watch 之上的，当 Etcd 的 Key-Value（Kubernetes 中资源的持久化信息） 出现变化时，会通知 APIServer。

对于 Kubernetes 集群内的各种资源，Kubernetes 的控制管理器和调度器需要感知到各种资源的状态变化（比如创建、更新、删除），然后根据变化事件履行自己的管理职责。Kubernetes 的实现是使用 APIServer 来充当简单的消息总线（Message Bus）的角色，提供两类机制（Push && Pull）来实现消息传递：
-	`List` 机制：提供查询当前最新状态的接口，以 `HTTP` 短连接方式提供
-	`Watch`（监听）机制：所有的组件通过 `Watch` 机制建立 `HTTP` 长链接，随时获悉自己感兴趣的资源的变化事件，实现对应的功能后还是调用 APIServer 来写入组件的 `Spec`，比如客户端 （Kubelet/Scheduler/Controller-manager） 通过 List-Watch 机制监听 APIServer 中资源 (Pods/ReplicaSet/Service 等等) 的 Create//Update/Delete 事件，并针对事件类型调用相应的事件处理函数。

![img](https://wx1.sbimg.cn/2020/09/19/G3yRo.jpg)

以 Pods 为例，List API 一般为 `GET /api/v1/pods`， Watch API 一般为 `GET /api/v1/watch/pods`，并且会带上 `watch=true`，表示采用 HTTP 长连接持续监听 Pods 相关事件，每当有事件来临，返回一个 WatchEvent。

##	Watch 的实现机制
通常我们使用使用 HTTP 大都是短连接方式，那么 Watch 是如何通过 HTTP 长连接获取 APIServer 发来的资源变更事件呢？答案就是 [Chunked transfer encoding](https://zh.wikipedia.org/zh-hans/%E5%88%86%E5%9D%97%E4%BC%A0%E8%BE%93%E7%BC%96%E7%A0%81)。如下，注意看 HTTP Response 中的 `Transfer-Encoding: chunked`：

```javascript
$ curl -i http://{kube-api-server-ip}:8080/api/v1/watch/pods?watch=yes
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
Date: Thu, 02 Jan 2020 20:22:59 GMT
Transfer-Encoding: chunked

{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"ADDED", "object":{"kind":"Pod","apiVersion":"v1",...}}
{"type":"MODIFIED", "object":{"kind":"Pod","apiVersion":"v1",...}}
```

下面给出两个实际的查询例子：

####	List 查询
```javascript
$ curl -X GET -i http://127.0.0.1:8080/api/v1/pods
HTTP/1.1 200 OK
Content-Type: application/json
Date: Tue, 15 Mar 2016 08:18:25 GMT
Transfer-Encoding: chunked

{
  "kind": "PodList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/pods",
    "resourceVersion": "228211"
  },
  "items": [
    {
      "metadata": {
        "name": "nginx-app-o0kvl",
        "generateName": "nginx-app-",
        "namespace": "default",
        "selfLink": "/api/v1/namespaces/default/pods/nginx-app-o0kvl",
        "uid": "090cc0c8-ea84-11e5-8c79-42010af00003",
        "resourceVersion": "228094",
        "creationTimestamp": "2016-03-15T08:00:51Z",
        "labels": {
          "run": "nginx-app"
        },
        "annotations": {
          "kubernetes.io/created-by": "{\"kind\":\"SerializedReference\",\"apiVersion\":\"v1\",\"reference\":{\"kind\":\"ReplicationController\",\"namespace\":\"default\",\"name\":\"nginx-app\",\"uid\":\"090bfb57-ea84-11e5-8c79-42010af00003\",\"apiVersion\":\"v1\",\"resourceVersion\":\"228081\"}}\n"
        }
      },
      "spec": {
        "volumes": [
          {
            "name": "default-token-wpfjn",
            "secret": {
              "secretName": "default-token-wpfjn"
            }
          }
        ],
        "containers": [
          {
            "name": "nginx-app",
            "image": "nginx",
            "ports": [
              {
                "containerPort": 80,
                "protocol": "TCP"
              }
            ],
            "resources": {},
            "volumeMounts": [
              {
                "name": "default-token-wpfjn",
                "readOnly": true,
                "mountPath": "/var/run/secrets/kubernetes.io/serviceaccount"
              }
            ],
            "terminationMessagePath": "/dev/termination-log",
            "imagePullPolicy": "Always"
          }
        ],
        "restartPolicy": "Always",
        "terminationGracePeriodSeconds": 30,
        "dnsPolicy": "ClusterFirst",
        "serviceAccountName": "default",
        "serviceAccount": "default",
        "nodeName": "10.240.0.4",
        "securityContext": {}
      },
      "status": {
        "phase": "Running",
        "conditions": [
          {
            "type": "Ready",
            "status": "True",
            "lastProbeTime": null,
            "lastTransitionTime": "2016-03-15T08:00:52Z"
          }
        ],
        "hostIP": "10.240.0.4",
        "podIP": "172.16.49.2",
        "startTime": "2016-03-15T08:00:51Z",
        "containerStatuses": [
          {
            "name": "nginx-app",
            "state": {
              "running": {
                "startedAt": "2016-03-15T08:00:52Z"
              }
            },
            "lastState": {},
            "ready": true,
            "restartCount": 0,
            "image": "nginx",
            "imageID": "docker://sha256:af4b3d7d5401624ed3a747dc20f88e2b5e92e0ee9954aab8f1b5724d7edeca5e",
            "containerID": "docker://b97168314ad58404dbce7cb94291db7a976d2cb824b39e5864bf4bdaf27af255"
          }
        ]
      }
    }
  ]
}
```

####	Watcher 查询
```javascript
$ curl -X GET -i http://127.0.0.1:8080/api/v1/pods?watch=true
HTTP/1.1 200 OK
Transfer-Encoding: chunked
Date: Tue, 15 Mar 2016 08:00:09 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked

{"type":"ADDED","object":{"kind":"Pod","apiVersion":"v1","metadata":{"name":"nginx-app-o0kvl","generateName":"nginx-app-","namespace":"default","selfLink":"/api/v1/namespaces/default/pods/nginx-app-o0kvl","uid":"090cc0c8-ea84-11e5-8c79-42010af00003","resourceVersion":"228082","creationTimestamp":"2016-03-15T08:00:51Z","labels":{"run":"nginx-app"},"annotations":{"kubernetes.io/created-by":"{\"kind\":\"SerializedReference\",\"apiVersion\":\"v1\",\"reference\":{\"kind\":\"ReplicationController\",\"namespace\":\"default\",\"name\":\"nginx-app\",\"uid\":\"090bfb57-ea84-11e5-8c79-42010af00003\",\"apiVersion\":\"v1\",\"resourceVersion\":\"228081\"}}\n"}},"spec":{"volumes":[{"name":"default-token-wpfjn","secret":{"secretName":"default-token-wpfjn"}}],"containers":[{"name":"nginx-app","image":"nginx","ports":[{"containerPort":80,"protocol":"TCP"}],"resources":{},"volumeMounts":[{"name":"default-token-wpfjn","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","securityContext":{}},"status":{"phase":"Pending"}}}
{"type":"MODIFIED","object":{"kind":"Pod","apiVersion":"v1","metadata":{"name":"nginx-app-o0kvl","generateName":"nginx-app-","namespace":"default","selfLink":"/api/v1/namespaces/default/pods/nginx-app-o0kvl","uid":"090cc0c8-ea84-11e5-8c79-42010af00003","resourceVersion":"228084","creationTimestamp":"2016-03-15T08:00:51Z","labels":{"run":"nginx-app"},"annotations":{"kubernetes.io/created-by":"{\"kind\":\"SerializedReference\",\"apiVersion\":\"v1\",\"reference\":{\"kind\":\"ReplicationController\",\"namespace\":\"default\",\"name\":\"nginx-app\",\"uid\":\"090bfb57-ea84-11e5-8c79-42010af00003\",\"apiVersion\":\"v1\",\"resourceVersion\":\"228081\"}}\n"}},"spec":{"volumes":[{"name":"default-token-wpfjn","secret":{"secretName":"default-token-wpfjn"}}],"containers":[{"name":"nginx-app","image":"nginx","ports":[{"containerPort":80,"protocol":"TCP"}],"resources":{},"volumeMounts":[{"name":"default-token-wpfjn","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"10.240.0.4","securityContext":{}},"status":{"phase":"Pending"}}}
{"type":"MODIFIED","object":{"kind":"Pod","apiVersion":"v1","metadata":{"name":"nginx-app-o0kvl","generateName":"nginx-app-","namespace":"default","selfLink":"/api/v1/namespaces/default/pods/nginx-app-o0kvl","uid":"090cc0c8-ea84-11e5-8c79-42010af00003","resourceVersion":"228088","creationTimestamp":"2016-03-15T08:00:51Z","labels":{"run":"nginx-app"},"annotations":{"kubernetes.io/created-by":"{\"kind\":\"SerializedReference\",\"apiVersion\":\"v1\",\"reference\":{\"kind\":\"ReplicationController\",\"namespace\":\"default\",\"name\":\"nginx-app\",\"uid\":\"090bfb57-ea84-11e5-8c79-42010af00003\",\"apiVersion\":\"v1\",\"resourceVersion\":\"228081\"}}\n"}},"spec":{"volumes":[{"name":"default-token-wpfjn","secret":{"secretName":"default-token-wpfjn"}}],"containers":[{"name":"nginx-app","image":"nginx","ports":[{"containerPort":80,"protocol":"TCP"}],"resources":{},"volumeMounts":[{"name":"default-token-wpfjn","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"10.240.0.4","securityContext":{}},"status":{"phase":"Pending","conditions":[{"type":"Ready","status":"False","lastProbeTime":null,"lastTransitionTime":"2016-03-15T08:00:51Z","reason":"ContainersNotReady","message":"containers with unready status: [nginx-app]"}],"hostIP":"10.240.0.4","startTime":"2016-03-15T08:00:51Z","containerStatuses":[{"name":"nginx-app","state":{"waiting":{"reason":"ContainerCreating","message":"Image: nginx is ready, container is creating"}},"lastState":{},"ready":false,"restartCount":0,"image":"nginx","imageID":""}]}}}
{"type":"MODIFIED","object":{"kind":"Pod","apiVersion":"v1","metadata":{"name":"nginx-app-o0kvl","generateName":"nginx-app-","namespace":"default","selfLink":"/api/v1/namespaces/default/pods/nginx-app-o0kvl","uid":"090cc0c8-ea84-11e5-8c79-42010af00003","resourceVersion":"228094","creationTimestamp":"2016-03-15T08:00:51Z","labels":{"run":"nginx-app"},"annotations":{"kubernetes.io/created-by":"{\"kind\":\"SerializedReference\",\"apiVersion\":\"v1\",\"reference\":{\"kind\":\"ReplicationController\",\"namespace\":\"default\",\"name\":\"nginx-app\",\"uid\":\"090bfb57-ea84-11e5-8c79-42010af00003\",\"apiVersion\":\"v1\",\"resourceVersion\":\"228081\"}}\n"}},"spec":{"volumes":[{"name":"default-token-wpfjn","secret":{"secretName":"default-token-wpfjn"}}],"containers":[{"name":"nginx-app","image":"nginx","ports":[{"containerPort":80,"protocol":"TCP"}],"resources":{},"volumeMounts":[{"name":"default-token-wpfjn","readOnly":true,"mountPath":"/var/run/secrets/kubernetes.io/serviceaccount"}],"terminationMessagePath":"/dev/termination-log","imagePullPolicy":"Always"}],"restartPolicy":"Always","terminationGracePeriodSeconds":30,"dnsPolicy":"ClusterFirst","serviceAccountName":"default","serviceAccount":"default","nodeName":"10.240.0.4","securityContext":{}},"status":{"phase":"Running","conditions":[{"type":"Ready","status":"True","lastProbeTime":null,"lastTransitionTime":"2016-03-15T08:00:52Z"}],"hostIP":"10.240.0.4","podIP":"172.16.49.2","startTime":"2016-03-15T08:00:51Z","containerStatuses":[{"name":"nginx-app","state":{"running":{"startedAt":"2016-03-15T08:00:52Z"}},"lastState":{},"ready":true,"restartCount":0,"image":"nginx","imageID":"docker://sha256:af4b3d7d5401624ed3a747dc20f88e2b5e92e0ee9954aab8f1b5724d7edeca5e","containerID":"docker://b97168314ad58404dbce7cb94291db7a976d2cb824b39e5864bf4bdaf27af255"}]}}}
```

通过上面的例子对 kuberesolver 中 HTTP 接口返回的接口解析部分的代码有所帮助。

##  0x03	看看 kuberesolver
先简单介绍下 kuberesolver 的使用方法，在 gRPC 客户端做如下形式的调用：
```golang
import "github.com/sercand/kuberesolver/v3"

// Register kuberesolver to grpc before calling grpc.Dial
kuberesolver.RegisterInCluster()

// it is same as
resolver.Register(kuberesolver.NewBuilder(nil /*custom kubernetes client*/ , "kubernetes"))

// if schema is 'kubernetes' then grpc will use kuberesolver to resolve addresses
cc, err := grpc.Dial("kubernetes:///service.namespace:portname", opts...)
// 或者
grpc.DialContext(ctx,  "kubernetes:///service:grpc", grpc.WithBalancerName("round_robin"), grpc.WithInsecure())
```

kuberesolver 支持如下形式的寻址方式，在 [`parseResolverTarget` 方法](https://github.com/sercand/kuberesolver/blob/master/builder.go#L76) 中实现寻址的解析：
```golang
kubernetes:///service-name:8080
kubernetes:///service-name:portname
kubernetes:///service-name.namespace:8080

kubernetes://namespace/service-name:8080
kubernetes://service-name:8080/
kubernetes://service-name.namespace:8080/
```

####	kuberesolver 整体架构
根据先前已有对 gRPC `+` Etcd/Consul 负载均衡器的实现经验，思路大致都是一样的：
1.	实现 `Resolver`：创建全局 `Resolver` 的标识符及实现 `grpc.Resolver` 的接口
2.	实现 `Watcher`（一般作为 `Resolver` 的子协程单独创建）：接收 Kubernetes 的 `API` 的改变通知并调用 gRPC 的接口通知 gRPC 内部

kuberesolver 的整体项目架构如下所示：
![img]()

下面按照架构图的模块对源码进一步分析。

##	0x04	结构定义 && 功能
[`models.go`](https://github.com/sercand/kuberesolver/blob/master/models.go#L5) 中定义了 kubernetes 的 API 返回结果的 `JSON` 结构体，对于 HTTP-Watcher 接口返回的结果，即监听的事件分为三种：
```golang
const (
	Added    EventType = "ADDED"
	Modified EventType = "MODIFIED"
	Deleted  EventType = "DELETED"
	Error    EventType = "ERROR"
)
```

##	0x05	Resolver 实现
先看看 [`Resolver` 的实现](https://github.com/sercand/kuberesolver/blob/master/builder.go)，`Resolver` 的 name 为 `kubernetes`，此值会用在 `grpc.Dial()` 的模式中。
```golang
const (
	kubernetesSchema = "kubernetes"
	defaultFreq      = time.Minute * 30
)
```

####	kubeBuilder 及 kubeBuilder.Build 方法
`kubeBuilder` 结构及 `NewBuilder` 初始化，注意其中的 `k8sClient`，本质就是一个 [`http.Client` 封装](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L25)：
```golang
// NewBuilder creates a kubeBuilder which is used by grpc resolver.
func NewBuilder(client K8sClient, schema string) resolver.Builder {
	return &kubeBuilder{
		k8sClient: client,
		schema:    schema,
	}
}

type kubeBuilder struct {
	k8sClient K8sClient
	schema    string
}
```

下面是核心函数 [`Build`](https://github.com/sercand/kuberesolver/blob/master/builder.go#L122)，构造生成 `resolver.Builder` 的方法：<br>
```golang
// Build creates a new resolver for the given target.
//
// gRPC dial calls Build synchronously, and fails if the returned error is
// not nil.
func (b *kubeBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOptions) (resolver.Resolver, error) {
	if b.k8sClient == nil {
		if cl, err := NewInClusterK8sClient(); err == nil {
			b.k8sClient = cl
		} else {
			return nil, err
		}
	}

	// 解析用户传入的 resolver 地址
	ti, err := parseResolverTarget(target)
	if err != nil {
		return nil, err
	}
	if ti.serviceNamespace == "" {
		ti.serviceNamespace = getCurrentNamespaceOrDefault()
	}

	// 初始化 kResolver 结构体
	ctx, cancel := context.WithCancel(context.Background())
	r := &kResolver{
		target:    ti,
		ctx:       ctx,
		cancel:    cancel,
		cc:        cc,
		rn:        make(chan struct{}, 1),
		k8sClient: b.k8sClient,
		t:         time.NewTimer(defaultFreq),
		freq:      defaultFreq,

		endpoints: endpointsForTarget.WithLabelValues(ti.String()),
		addresses: addressesForTarget.WithLabelValues(ti.String()),
	}

	// 开启单独的 watcher 模式
	go until(func() {
		r.wg.Add(1)
		err := r.watch()
		if err != nil && err != io.EOF {
			grpclog.Errorf("kuberesolver: watching ended with error='%v', will reconnect again", err)
		}
	}, time.Second, ctx.Done())
	return r, nil
}
```

此方法的核心步骤（通用范式）是：
1、首先，解析 `parseResolverTarget` 传入的地址，本项目中支持如下格式
```javascript
kubernetes:///service-name:8080
kubernetes:///service-name:portname
kubernetes:///service-name.namespace:8080

kubernetes://namespace/service-name:8080
kubernetes://service-name:8080/
kubernetes://service-name.namespace:8080/
```

上面的地址格式被解析为如下结构 `targetInfo`，在 Kubernetes 的定义中，APIServer 的调用服务地址必须包含如下关键信息：
-	`serviceName`：服务 name
-	`serviceNamespace`：命名空间
-	`port`：APIServer 的端口

```golang
type targetInfo struct {
	serviceName       string
	serviceNamespace  string
	port              string
	resolveByPortName bool
	useFirstPort      bool
}

func (ti targetInfo) String() string {
	return fmt.Sprintf("kubernetes://%s/%s:%s", ti.serviceNamespace, ti.serviceName, ti.port)
}
```

2、第二步，根据解析的结果，初始化 `kResolver` 结构体，此结构体包含了独立 goroutine 运行所需要的所有成员：
```golang
type kResolver struct {
	target targetInfo			//APISERVER 地址
	ctx    context.Context		// 用于 goroutine 退出
	cancel context.CancelFunc
	cc     resolver.ClientConn	// 用于调用 grpc 的方法通知
	// rn channel is used by ResolveNow() to force an immediate resolution of the target.
	rn        chan struct{}		// 用于触发 ResolveNow() 逻辑的信号 signal
	k8sClient K8sClient			// 用于请求 APISERVER 的 HTTP 客户端
	// wg is used to enforce Close() to return after the watcher() goroutine has finished.
	wg   sync.WaitGroup
	t    *time.Timer			// 用于 LIST 方式（定时拉取一次全量，用于初始化时）
	freq time.Duration

	endpoints prometheus.Gauge
	addresses prometheus.Gauge
}
```

3、最后，开启独立的子 goroutine，实现 `Watcher` 逻辑
```golang
...
	// 开启单独的 watcher 模式
	go until(func() {
		r.wg.Add(1)
		err := r.watch()
		if err != nil && err != io.EOF {
			grpclog.Errorf("kuberesolver: watching ended with error='%v', will reconnect again", err)
		}
	}, time.Second, ctx.Done())
...
```


####	kResolver 的实现
作为 `resovler.Builder` 的实例化实现（在 `return` 中返回 `resovler.Builder`），`kResolver` 必须实现 `resovler.Builder` 定义的方法：
-	`ResolveNow`：立即执行一次 resolve，本方法非必要实现
-	`Close`：关闭 Watcher，回收资源

```golang
// ResolveNow will be called by gRPC to try to resolve the target name again.
// It's just a hint, resolver can ignore this if it's not necessary.
func (k *kResolver) ResolveNow(resolver.ResolveNowOptions) {
	select {
	case k.rn <- struct{}{}:
	default:
	}
}

// Close closes the resolver.
func (k *kResolver) Close() {
	k.cancel()
	k.wg.Wait()
}
```

####	kResolver.resolve
`kResolver.resolve` 是 Watcher 的核心方法，这是个经典的 `for-select-loop`，主要接受如下几类触发的 channel 事件：
1.	`k.ctx.Done()`：上层通知退出
2.	`k.t.C`：定时器触发
3.	`k.rn`：`ResolveNow` 触发
4.	`up, hasMore := <-sw.ResultChan()`：`streamWatcher` 有新消息（事件）通知时，调用 `k.handle(up.Object)` 处理事件，即监听的资源发生了改变（扩容 / 缩容 / 下线等）

```golang
func (k *kResolver) handle(e Endpoints) {
	// 通过 makeAddresses 得到全量 IP 列表
	result, _ := k.makeAddresses(e)
	//	k.cc.NewServiceConfig(sc)
	if len(result) > 0 {
		k.cc.NewAddress(result)
	}

	k.endpoints.Set(float64(len(e.Subsets)))
	k.addresses.Set(float64(len(result)))
}

func (k *kResolver) resolve() {
	e, err := getEndpoints(k.k8sClient, k.target.serviceNamespace, k.target.serviceName)
	if err == nil {
		k.handle(e)
	} else {
		grpclog.Errorf("kuberesolver: lookup endpoints failed: %v", err)
	}
	// Next lookup should happen after an interval defined by k.freq.
	k.t.Reset(k.freq)
}

func (k *kResolver) watch() error {
	defer k.wg.Done()
	// watch endpoints lists existing endpoints at start
	sw, err := watchEndpoints(k.k8sClient, k.target.serviceNamespace, k.target.serviceName)
	if err != nil {
		return err
	}
	for {
		select {
		case <-k.ctx.Done():
			return nil
		case <-k.t.C:
			k.resolve()
		case <-k.rn:
			k.resolve()
		case up, hasMore := <-sw.ResultChan():
			if hasMore {
				k.handle(up.Object)
			} else {
				return nil
			}
		}
	}
}
```

##	0x06	 K8sClient 分析
在分析 Watcher 的逻辑之前，我们先看下 [`k8sClient` 封装](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go)，正如文章开始我们说的那样，提供了 `List` 和 `Watch` 两种调用方式：
-	[`watchEndpoints` 方法](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L120)：提供 `Watch` 方式的增量接口
-	[`getEndpoints` 方法](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#97)：提供 `List` 方式的全量接口

####	k8sClient 抽象接口 && 定义 && 初始化

```golang
// K8sClient is minimal kubernetes client interface
// 大写为 interface{}
type K8sClient interface {
	Do(req *http.Request) (*http.Response, error)
	GetRequest(url string) (*http.Request, error)
	Host() string
}

// 小写为具体实现
type k8sClient struct {
	host       string
	token      string		//http-token 认证方式
	httpClient *http.Client	// 真正的 http.Client
}
```

注意看初始化里面：
1.	APIServer 的证书加载路径
2.	`net.http` 默认是长连接方式，不过这里并未涉及超时等相关参数，默认为阻塞 client

```golang
const(
	serviceAccountToken     = "/var/run/secrets/kubernetes.io/serviceaccount/token"
	serviceAccountCACert    = "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt"
	kubernetesNamespaceFile = "/var/run/secrets/kubernetes.io/serviceaccount/namespace"
)

// NewInClusterK8sClient creates K8sClient if it is inside Kubernetes
func NewInClusterK8sClient() (K8sClient, error) {
	host, port := os.Getenv("KUBERNETES_SERVICE_HOST"), os.Getenv("KUBERNETES_SERVICE_PORT")
	if len(host) == 0 || len(port) == 0 {
		return nil, fmt.Errorf("unable to load in-cluster configuration, KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT must be defined")
	}
	token, err := ioutil.ReadFile(serviceAccountToken)
	if err != nil {
		return nil, err
	}
	ca, err := ioutil.ReadFile(serviceAccountCACert)
	if err != nil {
		return nil, err
	}
	certPool := x509.NewCertPool()
	certPool.AppendCertsFromPEM(ca)
	transport := &http.Transport{TLSClientConfig: &tls.Config{
		MinVersion: tls.VersionTLS10,
		RootCAs:    certPool,
	}}
	httpClient := &http.Client{Transport: transport, Timeout: time.Nanosecond * 0}

	return &k8sClient{
		host:       "https://" + net.JoinHostPort(host, port),
		token:      string(token),
		httpClient: httpClient,
	}, nil
}
```

####	getEndpoints && watchEndpoints
[`getEndpoints` 方法](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L97) 就是普通的 GET 查询，采用 HTTP 短连接方式
```golang
func getEndpoints(client K8sClient, namespace, targetName string) (Endpoints, error) {
	u, err := url.Parse(fmt.Sprintf("%s/api/v1/namespaces/%s/endpoints/%s",
		client.Host(), namespace, targetName))
	if err != nil {
		return Endpoints{}, err
	}
	req, err := client.GetRequest(u.String())
	if err != nil {
		return Endpoints{}, err
	}
	resp, err := client.Do(req)
	if err != nil {
		return Endpoints{}, err
	}
	defer resp.Body.Close()
	if resp.StatusCode != http.StatusOK {
		return Endpoints{}, fmt.Errorf("invalid response code %d", resp.StatusCode)
	}
	result := Endpoints{}
	err = json.NewDecoder(resp.Body).Decode(&result)
	return result, err
}
```

而 [`watchEndpoints` 方法](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L120) 则实现了 Http 的长连接方式（由服务端控制），通过调用 `newStreamWatcher` 返回一个 `streamWatcher`，传入参数为 `resp.Body`。<br>

特别注意这里的 HTTP 客户端的初始化方式，是适用于 `Transfer-Encoding: chunked` 模式的：
```golang
func watchEndpoints(client K8sClient, namespace, targetName string) (watchInterface, error) {
	u, err := url.Parse(fmt.Sprintf("%s/api/v1/watch/namespaces/%s/endpoints/%s",
		client.Host(), namespace, targetName))
	if err != nil {
		return nil, err
	}
	req, err := client.GetRequest(u.String())
	if err != nil {
		return nil, err
	}
	resp, err := client.Do(req)
	if err != nil {
		return nil, err
	}
	if resp.StatusCode != http.StatusOK {
		defer resp.Body.Close()
		return nil, fmt.Errorf("invalid response code %d", resp.StatusCode)
	}
	return newStreamWatcher(resp.Body), nil
}
```

##	0x07	Watcher 实现
接上一小节，Watcher 逻辑代码主要 [在此](https://github.com/sercand/kuberesolver/blob/master/stream.go)，其本质就是 [`streamWatcher`](https://github.com/sercand/kuberesolver/blob/master/stream.go#L26)，`streamWatcher` 扮演了这么一种角色，先回想下一般的 HTTP 请求模型：
```golang
	//...
	resp, err := client.Do(req)
	if err != nil {
		//...
	}
	defer resp.Body.Close()

	json.NewDecoder(resp.Body).Decode(&result)
	//...
```
上面这个是非常普通的 HTTP 请求代码，其中的 `resp.Body` 的类型是 `io.ReadCloser`，在 HTTP 的 `Transfer-Encoding: chunked` 模式下，如果不关闭 `resp.Body`，那么我们就可以实现在这个 stream 长连接上不断的读取数据。<br>
切记不要调用 `defer resp.Body.Close()`。


####	streamWatcher 的结构
`streamWatcher` 的结构如下，其中的 `result  chan Event` 用于向外部传递解析的结果（HTTP-API 接口返回）
```golang
// Interface can be implemented by anything that knows how to watch and report changes.
type watchInterface interface {
	// Stops watching. Will close the channel returned by ResultChan(). Releases
	// any resources used by the watch.
	Stop()

	// Returns a chan which will receive all the events. If an error occurs
	// or Stop() is called, this channel will be closed, in which case the
	// watch should be completely cleaned up.
	ResultChan() <-chan Event
}

// StreamWatcher turns any stream for which you can write a Decoder interface
// into a watch.Interface.
type streamWatcher struct {
	result  chan Event		// 用来向外部传递解析 respose 的结果
	r       io.ReadCloser
	decoder *json.Decoder
	sync.Mutex
	stopped bool
}
```

注意上面的 `watchInterface` 暴露的对外部的接口：
1.	`ResultChan` 方法：返回 `streamWatcher` 结构中的 `result  chan Event` 成员给外部
2.	`Stop` 方法：关闭整个 `streamWatcher` 的监听逻辑，清理资源并退出

`streamWatcher` 的实现是个非常经典的观察者模式。

####	初始化 NewStreamWatcher && Watcher
在 `newStreamWatcher` 中，通过单独调用 `go sw.receive()` 来开启独立的 Watcher 逻辑：
```golang
// NewStreamWatcher creates a StreamWatcher from the given io.ReadClosers.
func newStreamWatcher(r io.ReadCloser) watchInterface {
	sw := &streamWatcher{
		r:       r,
		decoder: json.NewDecoder(r),
		result:  make(chan Event),
	}
	go sw.receive()
	return sw
}

// receive reads result from the decoder in a loop and sends down the result channel.
func (sw *streamWatcher) receive() {
	defer close(sw.result)
	defer sw.Stop()		// 在此方法中关闭 resp.Body
	//watcher  LOOP
	for {
		// 通过 sw.Decode() 不停的触发 Read
		obj, err := sw.Decode()
		if err != nil {
			// 错误处理
			// Ignore expected error.
			if sw.stopping() {
				return
			}
			switch err {
			case io.EOF:
				// watch closed normally
			case io.ErrUnexpectedEOF:
				grpclog.Infof("kuberesolver: Unexpected EOF during watch stream event decoding: %v", err)
			default:
				grpclog.Infof("kuberesolver: Unable to decode an event from the watch stream: %v", err)
			}
			return
		}
		sw.result <- obj
	}
}
```
最后看下 `Decode` 方法，解析 HTTP-API 的响应，将 JSON 序列化的结果返回：
```golang
// Decode blocks until it can return the next object in the writer. Returns an error
// if the writer is closed or an object can't be decoded.
func (sw *streamWatcher) Decode() (Event, error) {
	var got Event
	if err := sw.decoder.Decode(&got); err != nil {
		return Event{}, err
	}
	switch got.Type {
	case Added, Modified, Deleted, Error:
		return got, nil
	default:
		return Event{}, fmt.Errorf("got invalid watch event type: %v", got.Type)
	}
}
```

##	0x08	总结
本文分析了使用 Kubernetes API 的方式来实现 gRPC 负载均衡的一种思路，项目中我们也是这么使用来实现 Kubernetes Service 的负载均衡的。

##	0x09	参考
-   [理解 K8S 的设计精髓之 list-watch](http://wsfdl.com/kubernetes/2019/01/10/list_watch_in_k8s.html)
-   [kubernetes 代码阅读 - apiserver 之 list-watch 篇](https://developer.aliyun.com/article/680204)
-	[Kubernetes API Watcher Design](https://docs.openstack.org/kuryr/0.2.0/devref/k8s_api_watcher_design.html)
-	[What could happen if I don't close response.Body?](https://stackoverflow.com/questions/33238518/what-could-happen-if-i-dont-close-response-body)
-	[Go Http 包解析：为什么需要 response.Body.Close()](https://segmentfault.com/a/1190000020086816)