---
layout:     post
title:      一个 Http(s) 网关的实现分析
subtitle:   如何使用几百行代码实现一个高可用的反向代理服务
date:       2020-03-22
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 反向代理
---

##  0x00    前言
本篇文章主要是对先前阅读过一篇关于 http-gateway 实现的文章的总结：[Let's Create a Simple Load Balancer With Go](https://kasvith.me/posts/lets-create-a-simple-lb-go/) 的小结。

![image](https://wx2.sbimg.cn/2020/05/22/http-lb1.png)

##  0x01    目标拆解
我们从要实现的功能及目标出发，来拆解一个 ** 高可用 ** 的（反向代理）网关，需要支持哪些功能？

![image](https://wx2.sbimg.cn/2020/05/22/http-lb-all-part.png)

从架构图来看，我们将网关划分为控制平面（control plane）和数据平面（data plane）：

控制平面包括：
1.  `Backend-Node-Manage`：负责维护后端（Backend）的增删查改
2.  `Backend-Node-Discovery`：后端的服务发现模块
3.  `Backend-Node-HealthyCheck`：对后端的健康检查，如果后端有故障，要及时剔除
4.  `Backend-Node-LB-picker`：选择何种负载均衡算法将客户端请求代理到后端
5.  `Client-API-interface`：客户端调用的 API
6.  `Backend-Node-Register`：后端服务节点启动时，要将自己的信息（定时）上报到注册中心；节点宕机时，要将自己注册信息及时移除

数据平面只负责转发数据流，即类似反向代理的功能，不过也可以实现中间人（MITM）的功能
1.  具备熔断、限速的功能，保证后端的稳定性
2.  数据采集，统计，Metrics
3.  高可用 -- 核心能力
4.  扩展性 -- 性能

##  0x02    Http-Gateway 及使用
[Let's Create a Simple Load Balancer With Go](https://kasvith.me/posts/lets-create-a-simple-lb-go/) 实现了一个短小精悍的 HTTP 反向代理网关，它所采用的技术有如下几点：
1.  使用 Go 语言标准库的 `ReverseProxy` 作为代理逻辑（可以自行实现）
2.  使用闭包和递归方式完成`ReverseProxy`的错误重试逻辑
3.  实现了主动和被动的健康度检查机制


####    使用
创建`4`个后端节点，使用`./simplelb --backends=http://localhost:3031,http://localhost:3032,http://localhost:3033,http://localhost:3034`启动代理网关。
```bash
http://localhost:3031
http://localhost:3032
http://localhost:3033
http://localhost:3034
```

####    ReverseProxy的使用
负载均衡器的作用是将流量负载分发到后端的服务器上，并将结果返回给客户端。Golang 标准库中的 ReverseProxy 就是这么一种 HTTP 处理器，它接收入向请求，将请求发送给另一个服务器，然后将响应发送回客户端。
这刚好是我们想要的，所以我们没有必要重复发明轮子。我们可以直接使用 ReverseProxy 来中继初始请求。

##  0x03    Http-Gateway 代码分析
我们按照如下的模块来分析：
-   后端节点及节点池
-   网关服务（主）入口
-   出错重试
-   负载均衡
-   健康度检查

####    Backend 后端节点
使用 `Backend` 结构来保存一个后端节点的状态，使用 `[]*Backend` 来表示这一组后端节点的状态集合，封装了两个方法：
-   `SetAlive`：更新指定节点的 Alive 状态
-   `IsAlive`：返回指定节点的 Alive 状态

```golang
// Backend holds the data about a server
type Backend struct {
    Serveraddr   string     // 服务 IP:PORT
	URL          *url.URL      //url
	Alive        bool
	mux          sync.RWMutex   // 为 Alive 提供加锁访问（使用读写锁）
	ReverseProxy *httputil.ReverseProxy
}

func (b *Backend) SetAlive(alive bool) {
	b.mux.Lock()
	b.Alive = alive
	b.mux.Unlock()
}

// 如果后端还活着，IsAlive 返回 true
func (b *Backend) IsAlive() (alive bool) {
	// 避免竞态条件
	b.mux.RLock()
	alive = b.Alive
	b.mux.RUnlock()
	return
}
```

####  serverPool 后端节点集合
`ServerPool` 封装了后端节点（池）及相关的操作方法：
1.  `AddBackend`：向 `ServerPool` 中添加节点
2.  `NextIndex`：由于使用了 RR 算法做轮询，这里需要返回下一个轮询的下标 Index
3.  `MarkBackendStatus`：更新某个后端节点的健康度状态
4.  `HealthCheck`：遍历后端的节点列表 `backends`，对每个节点进行健康度检查

`ServerPool` 的结构定义如下：
```golang
var serverPool ServerPool
// 定义：一种方式来跟踪所有后端，以及一个计算器变量
type ServerPool struct {
	backends []*Backend // 保存所有后端的集合
	current  uint64
}

// AddBackend to the server pool
func (s *ServerPool) AddBackend(backend *Backend) {
	s.backends = append(s.backends, backend)
}

// MarkBackendStatus changes a status of a backend
func (s *ServerPool) MarkBackendStatus(backendUrl *url.URL, alive bool) {
	for _, b := range s.backends {
		if b.URL.String() == backendUrl.String() {
			b.SetAlive(alive)
			break
		}
	}
}

// HealthCheck 对后端执行 ping 操作，并更新状态。
func (s *ServerPool) HealthCheck() {
	for _, b := range s.backends {
		// 现在我们可以遍历服务器，并标记它们的状态
		status := "up"
		alive := isBackendAlive(b.URL)
		b.SetAlive(alive)
		if !alive {
			status = "down"
		}
		log.Printf("%s [%s]\n", b.URL, status)
	}
}
```

####  主入口
主函数的逻辑如下：
1.  遍历初始化传入的后端节点`serverList`，初始化`proxy`及错误处理函数`proxy.ErrorHandler`
2.  初始化代理服务
3.  开启健康探测子goroutine
4.  启动代理服务网关


####   出错重试 

在发生错误时，ReverseProxy 会触发 ErrorHandler 回调函数，本文利用它来检查故障。这里我们详细分析下`ReverseProxy`的`proxy.ErrorHandler`的定义，在这里完成对出错重试的核心逻辑。代码如下：
重试的逻辑是：
1.  获取一个后端节点BackendNode1
2.  使用重试`GetRetryFromContext`加上`proxy.ServeHTTP(writer, request.WithContext(ctx))`的方法尝试代理请求
3.  在第`2`步出错时，主动标识此后端节点BackendNode1为`Failed`
4.  利用递归的方式，使用RoundRobin方法拿到下一个后端节点BackendNode2，然后再使用`lb`方法继续尝试代理请求，直到尝试完所有的节点或者成功时，退出
（第`4`步是在`lb`方法中，获取到其他的后端节点BackendNode2，然后执行和`proxy.ErrorHandler`中相同的逻辑）

```golang
serverUrl, err := url.Parse("代理的后端节点")
proxy := httputil.NewSingleHostReverseProxy(serverUrl)
proxy.ErrorHandler = func(writer http.ResponseWriter, request *http.Request, e error) {
    log.Printf("[%s] %s\n", serverUrl.Host, e.Error())
    retries := GetRetryFromContext(request)
    if retries < 3 {
        select {
        case <-time.After(10 * time.Millisecond):
            ctx := context.WithValue(request.Context(), Retry, retries+1)
            proxy.ServeHTTP(writer, request.WithContext(ctx))
        }
        return
    }

    // after 3 retries, mark this backend as down
    // 在三次重试之后，把这个后端标记为宕机
    serverPool.MarkBackendStatus(serverUrl, false)

    // if the same request routing for few attempts with different backends, increase the count
    // 同一个请求在尝试了几个不同的后端之后，增加计数
    attempts := GetAttemptsFromContext(request)
    log.Printf("%s(%s) Attempting retry %d\n", request.RemoteAddr, request.URL.Path, attempts)
    ctx := context.WithValue(request.Context(), Attempts, attempts+1)
    lb(writer, request.WithContext(ctx))
}
```

上面这段代码，有几点是可以借鉴的：
-   对`context`的使用，使用`context`保存了递归调用的（变量）信息，如下：
更新数据：`ctx := context.WithValue(request.Context(), Retry, retries+1)`
读取数据：

```golang
const (
	Attempts int = iota
	Retry
)
//取数据
func GetRetryFromContext(r *http.Request) int {
  if retry, ok := r.Context().Value(Retry).(int); ok {
    return retry
  }
  return 0
}

func GetAttemptsFromContext(r *http.Request) int {
	if attempts, ok := r.Context().Value(Attempts).(int); ok {
		return attempts
	}
	return 1
}
```

##  健康度检查
在主函数中开启了单独的 routine 来实现健康度检查，`healthCheck`：
```golang
// healthCheck runs a routine for check status of the backends every 2 mins
// healthCheck 返回一个 routine，每 2 分钟检查一次后端的状态
// 在上面的例子中，<-t.C 每 20 秒返回一个值，select 会检测到这个事件。在没有 default case 的情况下，select 会一直等待，直到有满足条件的 case 被执行。
func healthCheck() {
	t := time.NewTicker(time.Minute * 2)
	for {
		select {
		case <-t.C:
            log.Println("Starting health check...")
            //调用serverpool的HealthCheck实现
			serverPool.HealthCheck()
			log.Println("Health check completed")
		}
	}
}
```

此外，本文提供了健康度检查的被动方式`isBackendAlive`，被动模式就是定时对后端执行 ping 操作，以此来检查后端节点的状态：
1.  提供了简单的端口拨测来验证BackendNode的连通性
2.  
```golang
我们通过建立 TCP 连接来执行 ping 操作。如果后端及时响应，我们就认为它还活着。
当然，如果你喜欢，也可以改成直接调用某个端点，比如 /status。
切记，在执行完操作后要关闭连接，避免给服务器造成额外的负担，否则服务器会一直维护连接，最后把资源耗尽。
// isAlive 通过建立 TCP 连接检查后端是否还活着
func isBackendAlive(u *url.URL) bool {
	timeout := 2 * time.Second
	//只是简单的dial，可以加上更多逻辑，如自定义url，cpu、memory使用率等
	conn, err := net.DialTimeout("tcp", u.Host, timeout)
	if err != nil {
		log.Println("Site unreachable, error: ", err)
		return false
	}
	_ = conn.Close() // 不需要维护连接，把它关闭
	return true
}
```

####    LB负载均衡的实现
在主入口中定义了负载均衡的逻辑`lb`，`lb` 对入向请求进行负载均衡，这个方法可以作为 HandlerFunc 传给 http 服务器。
`lb`的核心逻辑是从`serverPool`中拿到一个可用的后端节点`peer`，然后代理真正的请求`peer.ReverseProxy.ServeHTTP(w, r)`，通过`GetAttemptsFromContext`来实现错误的重试逻辑
```golang
server := http.Server{
    Addr:    fmt.Sprintf(":%d", port),
    Handler: http.HandlerFunc(lb),		//lb的参数，为输入和输出
}


func lb(w http.ResponseWriter, r *http.Request) {
	attempts := GetAttemptsFromContext(r)
	if attempts > 3 {
		log.Printf("%s(%s) Max attempts reached, terminating\n", r.RemoteAddr, r.URL.Path)
		http.Error(w, "Service not available", http.StatusServiceUnavailable)
		return
	}

	peer := serverPool.GetNextPeer()
	if peer != nil {
		//注意，这里使用了递归（实现使用了递归）
		peer.ReverseProxy.ServeHTTP(w, r)
		return
	}
	http.Error(w, "Service not available", http.StatusServiceUnavailable)
}
```

serverPool提供的`GetNextPeer`方法如下：
```golang

// NextIndex atomically increase the counter and return an index
//遍历一遍 slice，如上图所示，我们将从 next 位置开始遍历整个列表，但在选择索引时，需要保证它处在 slice 的长度之内，这个可以通过取模运算来保证。

func (s *ServerPool) NextIndex() int {
	/*
			在选择下一个服务器时，我们需要跳过已经死掉的服务器，但不管怎样，我们都需要一个计数器。
		因为有很多客户端连接到负载均衡器，所以发生竟态条件是不可避免的。为了防止这种情况，我们需要使用 mutex 给 ServerPool 加锁。但这样做对性能会有影响，更何况我们并不是真想要给 ServerPool 加锁，我们只是想要更新计数器。
		最理想的解决方案是使用原子操作，Go 语言的 atomic 包为此提供了很好的支持。
		我们通过原子操作递增 current 的值，并通过对 slice 的长度取模来获得当前索引值。所以，返回值总是介于 0 和 slice 的长度之间，毕竟我们想要的是索引值，而不是总的计数值。
	*/
	return int(atomic.AddUint64(&s.current, uint64(1)) % uint64(len(s.backends)))
}
// GetNextPeer returns next active peer to take a connection
/*
	我们需要循环将请求路由到后端的每一台服务器上，但要跳过已经死掉的服务。
GetNext() 方法总是返回一个介于 0 和 slice 长度之间的值，如果这个值对应的服务器不可用，我们需要遍历一遍 slice。

遍历一遍 slice
如上图所示，我们将从 next 位置开始遍历整个列表，但在选择索引时，需要保证它处在 slice 的长度之内，这个可以通过取模运算来保证。
在找到可用的服务器后，我们将它标记为当前可用服务器。
*/
// GetNextPeer 返回下一个可用的服务器
//选择可用的后端，我们需要循环将请求路由到后端的每一台服务器上，但要跳过已经死掉的服务。GetNext() 方法总是返回一个介于 0 和 slice 长度之间的值，如果这个值对应的服务器不可用，我们需要遍历一遍 slice。

func (s *ServerPool) GetNextPeer() *Backend {
	// loop entire backends to find out an Alive backend
	// 遍历后端列表，找到可用的服务器
	next := s.NextIndex()
	l := len(s.backends) + next // start from next and move a full cycle
	// 从 next 开始遍历
	for i := next; i < l; i++ {
		idx := i % len(s.backends) // take an index by modding
		// 通过取模计算获得索引
		// 如果找到一个可用的服务器，将它作为当前服务器。如果不是初始的那个，就把它保存下来
		if s.backends[idx].IsAlive() { // if we have an alive backend, use it and store if its not the original one
			// 标记当前可用服务器
			if i != next {
				//在找到可用的服务器后，我们将它标记为当前可用服务器。
				atomic.StoreUint64(&s.current, uint64(idx)) //标记当前可用服务器
			}
			return s.backends[idx]
		}
	}
	return nil
}

```

##  总结
本文介绍了实现代理均衡网关的一般思路及模块拆解，分析了一款开源的反向代理网关的实现。不过本文介绍的网关实现不太适合在现网使用，现网的LB网关使用的话推荐一下几个项目（基于golang）：
-   [manba：Manba is a restful API gateway based on HTTP, which can be used as a unified API access layer.](https://github.com/fagongzi/manba)
-   [Vulcand Gateway](https://github.com/vulcand/vulcand)

##  参考
-   [Let's Create a Simple Load Balancer With Go](https://kasvith.me/posts/lets-create-a-simple-lb-go/)
-   [Gobetween - Introduction](http://gobetween.io/documentation.html)
-   [Yet another load balancer](https://github.com/onestraw/golb)
-   [Vulcand Gateway Doc](https://vulcand.github.io/quickstart.html)
-   [Vulcand - Programmatic load balancer backed by Etcd](https://github.com/vulcand/vulcand)