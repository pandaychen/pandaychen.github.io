---
layout:     post
title:      Kubernetes 应用改造（六）：使用原生 API 实现 gRPC 负载均衡
subtitle:   分析一款基于 kubernetes-API 实现的 LB 开源项目：kuberesolver
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

-	`GET`（SELECT）：从服务器取出资源。`GET` 请求对应 k8s api 的获取信息功能。因此，如果是获取信息的命令都要使用 GET 方式发起 HTTP 请求。
-	`POST`（CREATE）：在服务器新建一个资源。`POST` 请求对应 k8s api 的创建功能。因此，需要创建 Pods、ReplicaSet 或者 service 的时候请使用这种方式发起请求。
-	`PUT`（UPDATE）：在服务器更新资源（客户端提供改变后的完整资源）。对应更新 nodes 或 Pods 的状态、ReplicaSet 的自动备份数量等等。
-	`PATCH`（UPDATE）：在服务器更新资源（客户端提供改变的属性）
-	`DELETE`（DELETE）：删除资源

####	ApiServer
Kubernetes 的一个设计原则是 Etcd 存储位于 APIServer 之后，集群内的各种组件均需要需要统一经过 APIServer 做身份认证和鉴权等安全控制，即认证 + 授权，准入后才可以访问对应的资源数据。

对于 Kubernetes 集群内的各种资源，k8s 的控制管理器和调度器需要感知到各种资源的状态变化（比如创建），然后根据变化事件履行自己的管理职责。

Kubernetes 的实现方式是仍然使用 APIServer 来充当简单的消息总线（Message Bus）的角色，提供两类机制来实现消息传递：
1.	List 机制：提供查询当前最新状态的接口，以 `HTTP` 短连接方式提供
2.	Watch（监听）机制：所有的组件通过 `Watch` 机制建立 `HTTP` 长链接，随时获悉自己感兴趣的资源的变化事件，完成自己的功能后还是调用 APIServer 来写入组件的 `Spec`，这份 Spec 会被其它管理程序感知到并且进行处理

![img](https://wx1.sbimg.cn/2020/09/19/G3yRo.jpg)


##	0x02	Kubernetes 的 List-Watch 机制

##  0x03	看看 kuberesolver
根据先前已有对 gRPC `+` Etcd/Consul 负载均衡器的实现经验，思路大致都是一样的：
1.	实现 `Resolver`：创建全局 `Resolver` 的标识符及实现 `grpc.Resolver` 的接口
2.	实现 `Watcher`（一般作为 `Resolver` 的子协程单独创建）：接收 Kubernetes 的 `API` 的改变通知并调用 gRPC 的接口通知 gRPC 内部


##	0x04	Resolver 实现
先看看 [`Resolver` 的实现](https://github.com/sercand/kuberesolver/blob/master/builder.go)，`Resolver` 的 `name` 为 `kubernetes`，
```golang
const (
	kubernetesSchema = "kubernetes"
	defaultFreq      = time.Minute * 30
)
```

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
	ti, err := parseResolverTarget(target)
	if err != nil {
		return nil, err
	}
	if ti.serviceNamespace == "" {
		ti.serviceNamespace = getCurrentNamespaceOrDefault()
	}
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

2、第二步，根据解析的结果，初始化 `kResolver` 结构体：
```golang
type kResolver struct {
	target targetInfo
	ctx    context.Context
	cancel context.CancelFunc
	cc     resolver.ClientConn
	// rn channel is used by ResolveNow() to force an immediate resolution of the target.
	rn        chan struct{}
	k8sClient K8sClient
	// wg is used to enforce Close() to return after the watcher() goroutine has finished.
	wg   sync.WaitGroup
	t    *time.Timer
	freq time.Duration

	endpoints prometheus.Gauge
	addresses prometheus.Gauge
}
```

3、最后，开启独立的子 goroutine，实现 `Watcher` 逻辑

##	0x05	 K8sClient 分析
在分析 Watcher 的逻辑之前，我们先看下 [`k8sClient` 封装](https://github.com/sercand/kuberesolver/blob/master/kubernetes.go)，正如文章开始我们说的那样，提供了 `List` 和 `Watch` 两种调用方式：
-	`watchEndpoints`(https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#L120)：提供 `Watch` 方式的增量接口
-	`getEndpoints`(https://github.com/sercand/kuberesolver/blob/master/kubernetes.go#97)：提供 `List` 方式的全量接口

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
	token      string
	httpClient *http.Client
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
`getEndpoints` 就是普通的 http 查询，Http 短连接方式
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
而 `watchEndpoints` 则实现了 Http 的长连接方式（由服务端控制），通过调用 `newStreamWatcher` 返回一个 `streamWatcher`，传入参数为 `resp.Body`：
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

##	0x06	Watcher 实现
接上一小节，Watcher 逻辑代码主要 [在此](https://github.com/sercand/kuberesolver/blob/master/stream.go)，其本质就是 `streamWatcher`：

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
	result  chan Event
	r       io.ReadCloser
	decoder *json.Decoder
	sync.Mutex
	stopped bool
}

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

// ResultChan implements Interface.
func (sw *streamWatcher) ResultChan() <-chan Event {
	return sw.result
}

// Stop implements Interface.
func (sw *streamWatcher) Stop() {
	sw.Lock()
	defer sw.Unlock()
	if !sw.stopped {
		sw.stopped = true
		sw.r.Close()
	}
}

// stopping returns true if Stop() was called previously.
func (sw *streamWatcher) stopping() bool {
	sw.Lock()
	defer sw.Unlock()
	return sw.stopped
}

// receive reads result from the decoder in a loop and sends down the result channel.
func (sw *streamWatcher) receive() {
	defer close(sw.result)
	defer sw.Stop()
	for {
		obj, err := sw.Decode()
		if err != nil {
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


##	0x07	参考
-   [](http://wsfdl.com/kubernetes/2019/01/10/list_watch_in_k8s.html)
-   [](https://developer.aliyun.com/article/680204)