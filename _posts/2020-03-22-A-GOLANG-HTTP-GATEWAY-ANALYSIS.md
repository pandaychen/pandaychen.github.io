---
layout:     post
title:      一个 Http(s) 反向代理（网关）的实现分析
subtitle:   如何使用几百行代码实现一个高可用的反向代理服务
date:       2020-03-22
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 反向代理
    - ReverseProxy
    - Golang
---

##  0x00    前言
本篇文章主要是对先前阅读过一篇关于 http-gateway 实现的文章的总结：[Let's Create a Simple Load Balancer With Go](https://kasvith.me/posts/lets-create-a-simple-lb-go/) 的小结。

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/reverseproxy/http-lb1.png)

##  0x01    目标拆解
我们从要实现的功能及目标出发，来拆解一个 **高可用** 的（反向代理）网关，需要支持哪些功能？

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2020/reverseproxy/http-lb-all-part.png)

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
2.  使用闭包和递归方式完成 `ReverseProxy` 的错误重试逻辑
3.  实现了主动和被动的健康度检查机制


####    使用
创建 `4` 个后端节点，使用 `./simplelb --backends=http://localhost:3031,http://localhost:3032,http://localhost:3033,http://localhost:3034` 启动代理网关。
```bash
http://localhost:3031
http://localhost:3032
http://localhost:3033
http://localhost:3034
```

####    ReverseProxy 的用法
负载均衡器的作用是将流量负载分发到后端的服务器上，并将结果返回给客户端。Golang 标准库中的 ReverseProxy 就是这么一种 HTTP 处理器，它接收入向请求，将请求发送给另一个服务器，然后将响应发送回客户端。根据 Golang 文档的 [描述](https://golang.org/src/net/http/httputil/reverseproxy.go)：

```golang
u, _ := url.Parse("http://localhost:8080")
rp := httputil.NewSingleHostReverseProxy(u)
// 初始化服务器，并添加处理器
http.HandlerFunc(rp.ServeHTTP)
```

如上面的例子，使用 `httputil.NewSingleHostReverseProxy(url)` 初始化一个反向代理，这个反向代理可以将请求中继到指定的 url。在上面的例子中，所有的请求都会被中转到 `localhost:8080`，结果被发送给初始客户端。如果看一下 `ServeHTTP` 方法的签名，我们会发现它返回的是一个 HTTP 处理器，所以我们可以将它传给 http 的 `HandlerFunc`。在上面的例子中，可以使用 Backend 后端节点中的 url 来初始化 `ReverseProxy`，这样反向代理就会把请求路由给指定的 url

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
1.  遍历初始化传入的后端节点 `serverList`，初始化 `proxy` 及错误处理函数 `proxy.ErrorHandler`
2.  初始化代理服务
3.  开启健康探测子 goroutine
4.  启动代理服务网关

```golang
func main() {
	// 遍历传入的后端节点
	for _, tok := range tokens {
		serverUrl, err := url.Parse(tok)
		if err != nil {
			log.Fatal(err)
		}

		// 初始化服务器，并添加处理器
        proxy := httputil.NewSingleHostReverseProxy(serverUrl)
        // 在发生错误时，ReverseProxy 会触发 ErrorHandler 回调函数，我们可以利用它来检查故障及实现重试逻辑
		proxy.ErrorHandler = func(writer http.ResponseWriter, request *http.Request, e error) {
			log.Printf("[%s] %s\n", serverUrl.Host, e.Error())
            retries := GetRetryFromContext(request)
            // 检查是否达到了最大的重试上限
            // 检查重试次数，如果小于 3，就把同一个请求发送给同一个后端服务器
			if retries < 3 {
                // 从请求里拿到重试次数，如果已经达到最大上限，就终结这个请求
				select {
                case <-time.After(10 * time.Millisecond):
                    // 进行重试的逻辑
                    // 因为服务器可能会发生临时错误，在经过短暂的延迟（比如服务器没有足够的 socket 来接收请求）之后，服务器又可以继续处理请求
                    // 这里使用了 case 计时器触发的逻辑，把重试时间间隔设定在 10 ms 左右，10ms 后 case 被触发执行
					ctx := context.WithValue(request.Context(), Retry, retries+1)
					proxy.ServeHTTP(writer, request.WithContext(ctx))
				}
				return
			}

			// 在三次重试之后，把这个后端标记为宕机
			serverPool.MarkBackendStatus(serverUrl, false)

			// if the same request routing for few attempts with different backends, increase the count
			//// 同一个请求在尝试了几个不同的后端之后，增加计数
			attempts := GetAttemptsFromContext(request)
            log.Printf("%s(%s) Attempting retry %d\n", request.RemoteAddr, request.URL.Path, attempts)、
            // 使用 context 来维护重试次数，在增加重试次数后，我们把它传回 lb，选择一个新的后端来处理请求
			ctx := context.WithValue(request.Context(), Attempts, attempts+1)

			// 注意这个 lb 方法，采用了递归方式来处理重试逻辑！
			lb(writer, request.WithContext(ctx))
		}

		serverPool.AddBackend(&Backend{
			URL:          serverUrl,
			Alive:        true,
			ReverseProxy: proxy,
		})
		log.Printf("Configured server: %s\n", serverUrl)
	}

	// create http server
	server := http.Server{
		Addr: fmt.Sprintf(":%d", port),
		// 这个方法可以作为 HandlerFunc 传给 http 服务器
		Handler: http.HandlerFunc(lb),
	}

	// start health checking
	// 使用单独的 goroutine 来执行
	go healthCheck()

	log.Printf("Load Balancer started at :%d\n", port)
	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```


####   出错重试

在发生错误时，ReverseProxy 会触发 ErrorHandler 回调函数，本文利用它来检查故障。这里我们详细分析下 `ReverseProxy` 的 `proxy.ErrorHandler` 的定义，在这里完成对出错重试的核心逻辑。代码如下：
重试的逻辑是：
1.  获取一个后端节点 BackendNode1
2.  使用重试 `GetRetryFromContext` 加上 `proxy.ServeHTTP(writer, request.WithContext(ctx))` 的方法尝试代理请求
3.  在第 `2` 步出错时，主动标识此后端节点 BackendNode1 为 `Failed`
4.  利用递归的方式，使用 RoundRobin 方法拿到下一个后端节点 BackendNode2，然后再使用 `lb` 方法继续尝试代理请求，直到尝试完所有的节点或者成功时，退出
（第 `4` 步是在 `lb` 方法中，获取到其他的后端节点 BackendNode2，然后执行和 `proxy.ErrorHandler` 中相同的逻辑）

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
-   对 `context` 的使用，使用 `context` 保存了递归调用的（变量）信息，如下：
更新数据：`ctx := context.WithValue(request.Context(), Retry, retries+1)`
读取数据：

```golang
const (
	Attempts int = iota
	Retry
)
// 取数据
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

####  健康度检查
本项目中实现了两种健康检查机制：
-   主动（Active）方式：在处理当前请求时，如果发现当前的后端节点没有响应，就把它标记为已宕机
-   被动（Passive）方式：定时内对后端节点执行 ping 操作，以此来检查（并更新）服务器的状态

主动方式是上面 `proxy.ErrorHandler` 中定义的 `MarkBackendStatus` 方式，这里看下被动方式的实现：

在主函数中开启了单独的 goroutine 来实现健康度检查，`healthCheck`：
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
            // 调用 serverpool 的 HealthCheck 实现
			serverPool.HealthCheck()
			log.Println("Health check completed")
		}
	}
}
```

被动方式的实现 `isBackendAlive`，定时对后端执行 `Dial` 操作，以此来检查后端节点的状态（当然也可以使用 `wget`，在主函数中实现健康检查的 CGI 逻辑即可）
1.  提供了简单的端口拨测来验证 BackendNode 的连通性
2.  执行完操作后要关闭连接，避免消耗服务端的资源
3.  注意控制健康检查的频率（不加控制的话有可能会影响服务端的正常服务）

```golang
// isAlive 通过建立 TCP 连接检查后端是否还活着
func isBackendAlive(u *url.URL) bool {
	timeout := 2 * time.Second
	// 只是简单的 dial，可以加上更多逻辑，如自定义 url，cpu、memory 使用率等
	conn, err := net.DialTimeout("tcp", u.Host, timeout)
	if err != nil {
		log.Println("Site unreachable, error:", err)
		return false
	}
	_ = conn.Close() // 不需要维护连接，把它关闭
	return true
}
```

####    LB 负载均衡的实现
在主入口中定义了负载均衡的逻辑 `lb`，`lb` 对入向请求进行负载均衡，这个方法可以作为 HandlerFunc 传给 http 服务器。
`lb` 的核心逻辑是从 `serverPool` 中拿到一个可用的后端节点 `peer`，然后代理真正的请求 `peer.ReverseProxy.ServeHTTP(w, r)`，通过 `GetAttemptsFromContext` 来实现错误的重试逻辑
```golang
server := http.Server{
    Addr:    fmt.Sprintf(":%d", port),
    Handler: http.HandlerFunc(lb),		//lb 的参数，为输入和输出
}


func lb(w http.ResponseWriter, r *http.Request) {
	attempts := GetAttemptsFromContext(r)
	if attempts > 3 {
		log.Printf("%s(%s) Max attempts reached, terminating\n", r.RemoteAddr, r.URL.Path)
		http.Error(w, "Service not available", http.StatusServiceUnavailable)
		return
	}

    // GetNextPeer 返回下一个可用的服务器
	peer := serverPool.GetNextPeer()
	if peer != nil {
		// 注意，这里使用了递归（实现使用了递归）
		peer.ReverseProxy.ServeHTTP(w, r)
		return
	}
	http.Error(w, "Service not available", http.StatusServiceUnavailable)
}
```

serverPool 提供的 `GetNextPeer` 方法使用 RoundRobin 方式选择下一个节点，使用 `atomic` 操作来保证操作 `s.current` 的原子性：`atomic.AddUint64(&s.current, uint64(1))`，使用 `IsAlive` 方法来确定当前的节点是否 ok，选中节点后再次调用 `atomic` 更新 `s.current` 的值：
```golang
func (s *ServerPool) NextIndex() int {
	return int(atomic.AddUint64(&s.current, uint64(1)) % uint64(len(s.backends)))
}

func (s *ServerPool) GetNextPeer() *Backend {
	// loop entire backends to find out an Alive backend
	// 遍历后端列表，找到可用的服务器
	next := s.NextIndex()
	l := len(s.backends) + next
	// 从 next 开始遍历
	for i := next; i < l; i++ {
		idx := i % len(s.backends)
		// 通过取模计算获得索引
		// 如果找到一个可用的服务器，将它作为当前服务器。如果不是初始的那个，就把它保存下来
		if s.backends[idx].IsAlive() {
			// 标记当前可用服务器
			if i != next {
				// 在找到可用的服务器后，我们将它标记为当前可用服务器。
				atomic.StoreUint64(&s.current, uint64(idx)) // 标记当前可用服务器
			}
			return s.backends[idx]
		}
	}
	return nil
}
```

##  0x04    总结
本文介绍了实现代理均衡网关的一般思路及模块拆解，分析了一款开源的反向代理网关的实现。不过本文介绍的网关实现不太适合在现网使用，现网的 LB 网关使用的话推荐一下几个项目（基于 golang）：
-   [manba：Manba is a restful API gateway based on HTTP, which can be used as a unified API access layer.](https://github.com/fagongzi/manba)
-   [Vulcand Gateway](https://github.com/vulcand/vulcand)

##  0x05    参考
-   [Let's Create a Simple Load Balancer With Go](https://kasvith.me/posts/lets-create-a-simple-lb-go/)
-   [Gobetween - Introduction](http://gobetween.io/documentation.html)
-   [Yet another load balancer](https://github.com/onestraw/golb)
-   [Vulcand Gateway Doc](https://vulcand.github.io/quickstart.html)
-   [Vulcand - Programmatic load balancer backed by Etcd](https://github.com/vulcand/vulcand)