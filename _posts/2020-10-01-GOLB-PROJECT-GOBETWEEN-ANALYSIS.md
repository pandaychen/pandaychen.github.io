---
layout:     post
title:      基于 Golang 实现的负载均衡网关：gobetween 分析（一）
subtitle:   分析一款 Golang 实现的四层代理 CLB：主要逻辑
date:       2020-10-01
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Gateway
---


##  0x00    前言
前面分析过一款简单的反向代理实现：[一个 Http(s) 网关的实现分析](https://pandaychen.github.io/2020/03/22/A-GOLANG-HTTP-GATEWAY-ANALYSIS/)，这篇文章分析一款商用的 LB 开源项目：gobetween。它是一款 Pure-Golang 实现的四层代理网关，[文档在此](http://gobetween.io/documentation.html)，本文来探索下其实现及核心的源码分析。

> For a long time all of us have been using "traditional" load balancers / proxies like nginx, haproxy, and others.
> But in modern world balancing become more and more flexible because of environment changes are made more often. Nodes behind load balancer are spawning and disappearing according to load and/or other requirements. Auto scaling and containerization became almost a "silver bullet" in modern IT infrastructure architectures.
> In the IP-telephony world DNS SRV records are main mechanism to find out nearest and less loaded call router.
> Same situation is in modern microservices world, but unfortunately, there are almost no lb / proxy that has flexible and complete automatic discovery feature. There are lot's of tricks and workarounds like this.
> gobetween is aiming to fill this gap and provide fast, flexible and full-featured load balancing solution for modern microservice architectures.

gobetween 的架构图如下：
![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/gobetween/arch.png)

## 0x01 特性
官方提供的特性，如下：
* [Fast L4 Load Balancing](https://github.com/yyyar/gobetween/wiki) -- 支持代理的方式
  * **TCP** - with optional [The PROXY Protocol](https://github.com/yyyar/gobetween/wiki/Proxy-Protocol) support
  * **TLS** - [TLS Termination](https://github.com/yyyar/gobetween/wiki/Protocols#tls) + [ACME](https://github.com/yyyar/gobetween/wiki/Protocols#tls) & [TLS Proxy](https://github.com/yyyar/gobetween/wiki/Tls-Proxying)
  * **UDP** - with optional virtual sessions and transparent mode


* [Clear & Flexible Configuration](https://github.com/yyyar/gobetween/wiki/Configuration) with [TOML](config/gobetween.toml) or [JSON](config/gobetween.json) -- 提供本地配置或远程配置
  * **File** - read configuration from the file
  * **URL** - query URL by HTTP and get configuration from the response body
  * **Consul** - query Consul key-value storage API for configuration

* [Management REST API](https://github.com/yyyar/gobetween/wiki/REST-API) -- 管理端的 API 设置及基础监控、后端节点管理等
  * **System Information** - general server info
  * **Configuration** - dump current config
  * **Servers** - list, create & delete
  * **Stats & Metrics** - for servers and backends including rx/tx, status, active connections & etc.

* [Discovery](https://github.com/yyyar/gobetween/wiki/Discovery) -- 后端服务发现的方式
  * **Static** - hardcode backends list in the config file
  * **Docker** - query backends from Docker / Swarm API filtered by label
  * **Exec** - execute an arbitrary program and get backends from its stdout
  * **JSON** - query arbitrary http url and pick backends from response json (of any structure)
  * **Plaintext** - query arbitrary http and parse backends from response text with customized regexp
  * **SRV** - query DNS server and get backends from SRV records
  * **Consul** - query Consul Services API for backends
  * **LXD** - query backends from LXD

* [Healthchecks](https://github.com/yyyar/gobetween/wiki/Healthchecks)  -- 支持健康检查的方式
  * **Ping** - simple TCP ping healthcheck
  * **Exec** - execute arbitrary program passing host & port as options, and read healthcheck status from the stdout
  * **Probe** - send specific bytes to backend (udp, tcp or tls) and expect a correct answer (bytes or regexp)

* [Balancing Strategies](https://github.com/yyyar/gobetween/wiki/Balancing) (with [SNI](https://github.com/yyyar/gobetween/wiki/Server-Name-Indication) support) -- 后端节点负载均衡的策略
  * **Weight** - select backend from pool based relative weights of backends
  * **Roundrobin** - simple elect backend from pool in circular order
  * **Iphash** - route client to the same backend based on client ip hash
  * **Iphash1** - same as iphash but backend removal consistent (clients remain connecting to the same backend, even if some other backends down)
  * **Leastconn** - select backend with least active connections
  * **Leastbandwidth** -  backends with least bandwidth

从上面的特性中，也可以看出，一个代理网关需要具备的基本要素，在先前这篇文章 [一个 Http(s) 网关的实现分析](https://pandaychen.github.io/2020/03/22/A-GOLANG-HTTP-GATEWAY-ANALYSIS/) 也梳理过：
-	管理端 API：提供（代理）后端 backend 实时信息、配置、统计信息（metrics）等管理及查询的 Restful 接口
-	代理网关进行服务发现的机制
-	对后端 backend 节点的健康检查
-	负载均衡的策略
-	代理网关的实现（TCP/UDP/HTTP 等）
-	易用的负载均衡器配置

##  0x02   分析路线
个人比较感兴趣的点有如下几块：
1.  Gateway 实现的模型，各个子模块之间的联动策略及通信方式
2.  和 `Consul` 的结合做服务发现（Service Discovery）
3.  Metrics 指标及采集方法
4.  配置热重启
5.  Gateway 的扩展能力及高可用实现


##  0x03    代码分析 - 总览
此外，在 gobetween 的实现中，大部分异步通信都是通过 channel 来完成的，由 scheduler 中的 `for...select` 结构完成大部分核心的事假调度。

核心逻辑罗列如下：
-	[scheduler](https://github.com/yyyar/gobetween/blob/master/src/server/scheduler/scheduler.go#L93)：负责整个网关各类事件的调度及处理
-	[discovery](https://github.com/yyyar/gobetween/blob/master/src/discovery/discovery.go)：负责后端 backend 节点的服务发现
-	[server-proxy](https://github.com/yyyar/gobetween/blob/master/src/server/tcp/server.go)：代理的实现逻辑


下面，我们按照上面的基本要素来分析下 gobetween 的实现：

####    配置 Config
gobetween 的 [主要配置结构如下](https://github.com/yyyar/gobetween/blob/master/src/config/config.go)：
`Config` 是全局配置，不难了解各参数的意义，注意 `Servers  map[string]Server`，一个 `Server`（Key）名字代表了一个 LB 负载均衡器：
```golang
type Config struct {
	Logging  LoggingConfig     `toml:"logging" json:"logging"`
	Api      ApiConfig         `toml:"api" json:"api"`
	Metrics  MetricsConfig     `toml:"metrics" json:"metrics"`
	Defaults ConnectionOptions `toml:"defaults" json:"defaults"`
	Acme     *AcmeConfig       `toml:"acme" json:"acme"`
	Profiler *ProfilerConfig   `toml:"profiler" json:"profiler"`
	Servers  map[string]Server `toml:"servers" json:"servers"`
}

type Server struct {
	ConnectionOptions

	// hostname:port
	Bind string `toml:"bind" json:"bind"`

	// tcp | udp | tls
	Protocol string `toml:"protocol" json:"protocol"`

	// weight | leastconn | roundrobin
	Balance string `toml:"balance" json:"balance"`

	// Optional configuration for server name indication
	Sni *Sni `toml:"sni" json:"sni"`

	// Optional configuration for protocol = tls
	Tls *Tls `toml:"tls" json:"tls"`

	// Optional configuration for backend_tls_enabled = true
	BackendsTls *BackendsTls `toml:"backends_tls" json:"backends_tls"`

	// Optional configuration for protocol = udp
	Udp *Udp `toml:"udp" json:"udp"`

	// Access configuration
	Access *AccessConfig `toml:"access" json:"access"`

	// ProxyProtocol configuration
	ProxyProtocol *ProxyProtocol `toml:"proxy_protocol" json:"proxy_protocol"`

	// Discovery configuration
	Discovery *DiscoveryConfig `toml:"discovery" json:"discovery"`

	// Healthcheck configuration
	Healthcheck *HealthcheckConfig `toml:"healthcheck" json:"healthcheck"`
}
```

##  0x04	核心数据结构
[`src/core`](https://github.com/yyyar/gobetween/tree/master/src/core) 下面定义了 gobetween 的核心结构的抽象，这里列出来一下：<br>
1、[`Balancer` 结构](https://github.com/yyyar/gobetween/blob/master/src/core/balancer.go)，负载均衡算法的抽象，需要实现 `Elect` 方法：
```golang
/**
 * Balancer interface
 */
type Balancer interface {
	/**
	 * Elect backend based on Balancer implementation
	 */
	Elect(Context, []*Backend) (*Backend, error)
}
```
2、[Server 结构](https://github.com/yyyar/gobetween/blob/master/src/core/server.go)：抽象 LB 负载均衡器的公共接口，对应 [于此](https://github.com/yyyar/gobetween/blob/master/src/server/server.go)
```golang
type Server interface {
	/**
	 * Start server
	 */
	Start() error

	/**
	 * Stop server and wait until it stop
	 */
	Stop()

	/**
	 * Get server configuration
	 */
	Cfg() config.Server
}
```
3、`ReadWriteCount` 结构：
```golang
type ReadWriteCount struct {
	/* Read bytes count */
	CountRead uint

	/* Write bytes count */
	CountWrite uint

	Target Target
}
```
4、[`Context` 及 `TcpContext`](https://github.com/yyyar/gobetween/blob/master/src/core/context.go)：抽象了 TCP 连接的属性
```golang
type Context interface {
	String() string
	Ip() net.IP
	Port() int
	Sni() string
}

/**
 * Proxy tcp context
 */
type TcpContext struct {
	Hostname string
	/**
	 * Current client connection
	 */
	Conn net.Conn
}
```
5、[`Service` 结构](https://github.com/yyyar/gobetween/blob/master/src/core/service.go)
```golang
/**
 * Service is a global facility that could be Enabled or Disabled for a number
 * of core.Server instances, depending on their configration. See services/registry
 * for exact examples.
 */
type Service interface {
	/**
	 * Enable service for Server
	 */
	Enable(Server) error

	/**
	 * Disable service for Server
	 */
	Disable(Server) error
}
```
6、[`Backend`](https://github.com/yyyar/gobetween/blob/master/src/core/backend.go)：定义了后端节点及统计信息的通用结构
```golang
/**
 * Backend means upstream server
 * with all needed associate information
 */
type Backend struct {
	Target
	Priority int          `json:"priority"`
	Weight   int          `json:"weight"`
	Sni      string       `json:"sni,omitempty"`
	Stats    BackendStats `json:"stats"`
}

/**
 * Backend status
 */
type BackendStats struct {
	Live               bool   `json:"live"`
	Discovered         bool   `json:"discovered"`
	TotalConnections   int64  `json:"total_connections"`
	ActiveConnections  uint   `json:"active_connections"`
	RefusedConnections uint64 `json:"refused_connections"`
	RxBytes            uint64 `json:"rx"`
	TxBytes            uint64 `json:"tx"`
	RxSecond           uint   `json:"rx_second"`
	TxSecond           uint   `json:"tx_second"`
}
```

##	0x05	核心模块分析 - 主流程
我们从 [main.go](https://github.com/yyyar/gobetween/blob/master/main.go) 开始，`main` 方法中独立启动了 `3` 个子逻辑，传入参数为 `cfg` 配置：<br>
1.  `manager`：核心逻辑
2.  `metrics`：启动 metrics 服务
3.  `api`：使用 `gin` 框架构建的管理端 CGI 服务

这里通过 goroutine 的方式实现，好处是简化了通信，如果作为独立的服务来实现就需要提供接口给其他模块进行调用了。

```golang
func main(){
    ...
    // Process flags and start
	cmd.Execute(func(cfg *config.Config) {
		// Configure logging
		logging.Configure(cfg.Logging.Output, cfg.Logging.Level, cfg.Logging.Format)
		// Start manager
		manager.Initialize(*cfg)
		/* setup metrics */
		metrics.Start((*cfg).Metrics)
		// Start API
		api.Start((*cfg).Api)
		// block forever
		<-(chan string)(nil)
    })
}
```
下面就这 `3` 个自逻辑展开进行分析。

##	0x06	Manage 核心管理逻辑

####	创建和启动代理 Server
[Manage 模块](https://github.com/yyyar/gobetween/blob/master/src/manager/manager.go#L184) 的核心方法是 `manage.Create()`，在 [`prepareConfig` 方法](https://github.com/yyyar/gobetween/blob/master/src/manager/manager.go#L184) 中，根据配置的类型，如 `TLS` 配置、启动代理的类型、服务发现的类型等等初始化 `config.Server` 结构，然后通过 `server.New()` 方法创建一个代理网关结构，最后通过 `server.Start()` 启动代理：

PS：注意这里是启动一个代理类型，

```golang
func Initialize(cfg config.Config) {
	......
	// Go through config and start servers for each server
	for name, serverCfg := range cfg.Servers {
		err := Create(name, serverCfg)
		if err != nil {
			log.Fatal(err)
		}
	}
	......
}

// 启动单个代理
func Create(name string, cfg config.Server) error {

	servers.Lock()
	defer servers.Unlock()

	if _, ok := servers.m[name]; ok {
		return errors.New("Server with this name already exists:" + name)
	}

	// 根据配置初始化 Server 结构
	c, err := prepareConfig(name, cfg, defaults)
	if err != nil {
		return err
	}

	// 初始化
	server, err := server.New(name, c)

	if err != nil {
		return err
	}

	for _, srv := range services {
		err = srv.Enable(server)
		if err != nil {
			return err
		}
	}

	// 启动代理
	if err = server.Start(); err != nil {
		return err
	}
	servers.m[name] = server
	return nil
}
```
接下来，我们看下 `server.New()` 及 `server.Start()` 做了什么事情。

####	Server 的初始化及启动
本节以 Tcp 代理的初始化及启动 [代码为例](https://github.com/yyyar/gobetween/blob/master/src/server/tcp/server.go)，先看下 `tcp.Server` 的结构体定义，从此结构入手来分析一个代理的实现要素：

这里列举下 `tcp.Server` 的结构中比较重要的成员：<br>
-	`scheduler`：调度器，负责本代理的（后端选择）负载均衡算法、后端节点的服务发现、健康度探测以及代理自身的数据指标统计
-	`clients`：保存了本代理的活跃接入 TCP 连接
-	`connect/disconnect`：`channel` 类型，用于连接接入 / 退出时的异步通信
-	`access`：用于代理访问权限检查

具体结构和初始化方法如下代码所示：
```golang
type Server struct {
	/* Server friendly name */
	name string

	/* Listener */
	listener net.Listener

	/* Configuration */
	cfg config.Server

	/* Scheduler deals with discovery, balancing and healthchecks */
	scheduler scheduler.Scheduler		// 每一个代理结构都包含一个调度器 scheduler

	/* Current clients connection */
	clients map[string]net.Conn			//map 结构保持了 TCP 代理的所有连接（前置）

	/* Stats handler */
	statsHandler *stats.Handler			// 代理的统计回调

	/* ----- channels ----- */

	/* Channel for new connections */
	connect chan (*core.TcpContext)

	/* Channel for dropping connections or connectons to drop */
	disconnect chan (net.Conn)

	/* Stop channel */
	stop chan bool

	/* Tls config used to connect to backends */
	backendsTlsConfg *tls.Config

	/* Tls config used for incoming connections */
	tlsConfig *tls.Config

	/* Get certificate filled by external service */
	GetCertificate func(*tls.ClientHelloInfo) (*tls.Certificate, error)

	/* ----- modules ----- */
	/* Access module checks if client is allowed to connect */
	access *access.Access
}

// 初始化 Server 的方法：
func New(name string, cfg config.Server) (*Server, error) {
	......
	log := logging.For("server")

	var err error = nil
	statsHandler := stats.NewHandler(name)

	// Create server
	server := &Server{
		name:         name,
		cfg:          cfg,
		stop:         make(chan bool),
		disconnect:   make(chan net.Conn),
		connect:      make(chan *core.TcpContext),
		clients:      make(map[string]net.Conn),
		statsHandler: statsHandler,
		scheduler: scheduler.Scheduler{		// 初始化 scheduler
			Balancer:     balance.New(cfg.Sni, cfg.Balance),		// 负载均衡器
			Discovery:    discovery.New(cfg.Discovery.Kind, *cfg.Discovery),	// 服务发现
			Healthcheck:  healthcheck.New(cfg.Healthcheck.Kind, *cfg.Healthcheck),	// 健康检查
			StatsHandler: statsHandler,		// 状态上报 & 监控
		},
	}
	......
}
```

初始化 `Server` 代理结构完成之后，接下来就是启动代理工作的流程 [`server.Start()`](https://github.com/yyyar/gobetween/blob/master/src/server/tcp/server.go#L136)，这里主要逻辑如下：
1.	代理 `server.Start()` 中，启动一个子 routione 完成连接的调度（`server.HandleClientDisconnect` 和 `server.HandleClientConnect` 方法）及退出时资源的回收工作
2.	启动一个子 goroutine，实现代理状态统计逻辑 [`this.statsHandler.Start()`](https://github.com/yyyar/gobetween/blob/master/src/stats/handler.go#L92)
3.	启动一个子 goroutine，运行代理的调度器 [`scheduler.Start()`](https://github.com/yyyar/gobetween/blob/master/src/server/scheduler/scheduler.go#L93)
4.	在主 goroutine 中启动 Tcp 代理的 `Listen` 方法，接收连接事件，若有新连接接入时，通过 `Server.connect` 这个 `chan (*core.TcpContext)` 类型的 channel，将该事件通知到 `1` 中的逻辑，触发对新连接的代理连接逻辑实现

下面是 `server.Start()` 的代码：
```golang
func (this *Server) Start() error {
	var err error
	this.tlsConfig, err = tlsutil.MakeTlsConfig(this.cfg.Tls, this.GetCertificate)
	if err != nil {
		return err
	}

	// 代理工作的核心循环
	go func() {
		for {
			select {
				// 处理连接断开的事件
			case client := <-this.disconnect:
				this.HandleClientDisconnect(client)
				// 处理新连接事件
			case ctx := <-this.connect:
				this.HandleClientConnect(ctx)
				// 处理代理退出事件
			case <-this.stop:
				this.scheduler.Stop()
				this.statsHandler.Stop()
				if this.listener != nil {
					this.listener.Close()
					for _, conn := range this.clients {
						conn.Close()
					}
				}
				this.clients = make(map[string]net.Conn)
				return
			}
		}
	}()

	// Start stats handler
	this.statsHandler.Start()

	// Start scheduler
	this.scheduler.Start()

	// Start listening
	if err := this.Listen(); err != nil {
		this.Stop()
		return err
	}

	return nil
}
```

这里我们先看下 `server.Listen()` 方法：
```golang
func (this *Server) Listen() (err error) {

	log := logging.For("server.Listen")

	// create tcp listener
	this.listener, err = net.Listen("tcp", this.cfg.Bind)

	if err != nil {
		log.Error("Error starting", this.cfg.Protocol+"server:", err)
		return err
	}

	sniEnabled := this.cfg.Sni != nil

	// 在子 goroutine 中接受连接，不优雅（参考 fasthttp）
	go func() {
		for {
			conn, err := this.listener.Accept()

			if err != nil {
				log.Error(err)
				return
			}
			// 处理新连接
			go this.wrap(conn, sniEnabled)
		}
	}()

	return nil
}

// 处理新连接的方法
func (this *Server) wrap(conn net.Conn, sniEnabled bool) {
	log := logging.For("server.Listen.wrap")

	var hostname string
	var err error

	if sniEnabled {
		var sniConn net.Conn
		sniConn, hostname, err = sni.Sniff(conn, utils.ParseDurationOrDefault(this.cfg.Sni.ReadTimeout, time.Second*2))

		if err != nil {
			log.Error("Failed to get / parse ClientHello for sni:", err)
			conn.Close()
			return
		}

		conn = sniConn
	}

	if this.tlsConfig != nil {
		conn = tls.Server(conn, this.tlsConfig)
	}

	// 将新连接封装为 core.TcpContext 结构，放入 channel
	this.connect <- &core.TcpContext{
		hostname,
		conn,
	}
}
```

下面一篇文章，再分析下 gobetween 代理实现的部分代码细节。下一小节我们看下 `server.scheduler` 的结构及实现。

##	0x07	Server 代理模块的 Scheduler 结构及实现
上一节说到 `Server` 中重要的 [成员 `Scheduler`](https://github.com/yyyar/gobetween/blob/master/src/server/scheduler/scheduler.go)，这个结构是实现整个负载均衡调度的核心，它包含了如下组件：
-	负载均衡选择器 `core.Balancer`
-	后端服务发现组件 `discovery.Discovery`
-	健康检查组件 `healthcheck.Healthcheck`
-	Metrics 收集组件 `stats.Handler`

```golang
type Scheduler struct {
	/* Balancer impl */
	Balancer core.Balancer
	/* Discovery impl */
	Discovery *discovery.Discovery
	/* Healthcheck impl */
	Healthcheck *healthcheck.Healthcheck
	/* ----- backends ------*/
	/* Current cached backends map */
	backends map[core.Target]*core.Backend
	/* Stats */
	StatsHandler *stats.Handler
	/* ----- channels ----- */
	/* Backend operation channel */
	ops chan Op
	/* Stop channel */
	stop chan bool
	/* Elect backend channel */
	elect chan ElectRequest
}
```

####	Scheduler 的核心方法
[`Scheduler.Start()`](https://github.com/yyyar/gobetween/blob/master/src/server/scheduler/scheduler.go#L93) 是 `Scheduler` 的核心启动方法，
```golang
func (this *Scheduler) Start() {

	log := logging.For("scheduler")

	log.Info("Starting scheduler", this.StatsHandler.Name)

	this.ops = make(chan Op)
	this.elect = make(chan ElectRequest)
	this.stop = make(chan bool)
	this.backends = make(map[core.Target]*core.Backend)

	this.Discovery.Start()
	this.Healthcheck.Start()

	// backends stats pusher ticker
	backendsPushTicker := time.NewTicker(2 * time.Second)

	/**
	 * Goroutine updates and manages backends
	 */
	go func() {
		for {
			select {

			/* ----- discovery ----- */

			// handle newly discovered backends
			case backends := <-this.Discovery.Discover():
				this.HandleBackendsUpdate(backends)
				this.Healthcheck.In <- this.Targets()
				this.StatsHandler.BackendsCounter.In <- this.Targets()

			/* ------ healthcheck ----- */

			// handle backend healthcheck result
			case checkResult := <-this.Healthcheck.Out:
				this.HandleBackendLiveChange(checkResult.Target, checkResult.Status == healthcheck.Healthy)

			/* ----- stats ----- */

			// push current backends to stats handler
			case <-backendsPushTicker.C:
				this.StatsHandler.Backends <- this.Backends()

			// handle new bandwidth stats of a backend
			case bs := <-this.StatsHandler.BackendsCounter.Out:
				this.HandleBackendStatsChange(bs.Target, &bs)

			/* ----- operations ----- */

			// handle backend operation
			case op := <-this.ops:
				this.HandleOp(op)

			// elect backend
			case electReq := <-this.elect:
				this.HandleBackendElect(electReq)

			/* ----- stop ----- */

			// handle scheduler stop
			case <-this.stop:
				log.Info("Stopping scheduler", this.StatsHandler.Name)
				backendsPushTicker.Stop()
				this.Discovery.Stop()
				this.Healthcheck.Stop()
				metrics.RemoveServer(fmt.Sprintf("%s", this.StatsHandler.Name), this.backends)
				return
			}
		}
	}()
}
```

####	后端服务发现组件 Discovery
[Discovery](https://github.com/yyyar/gobetween/blob/master/src/discovery/discovery.go) 主要用于与服务注册中心通信，拿到服务名字对应的实时在线后端列表：

`Discovery` 组件中，不同的注册中心通过 `map[string]func(config.DiscoveryConfig) interface{}` 来存储，其中 value 对应的是相应服务发现方法的初始化。以 consul 为例：
```golang
func init() {
	registry["consul"] = NewConsulDiscovery
}
```
`Discovery` 结构如下，`fetch FetchFunc` 成员，在初始化时，会被赋值为 consul 的实现 [`consulFetch`](https://github.com/yyyar/gobetween/blob/master/src/discovery/consul.go#L44)，其他成员的含义见注释：
```golang
type Discovery struct {
	/**
	 * Cached backends
	 */
	backends *[]core.Backend	// 缓存从 consul-API 获取到的后端节点
	/**
	 * Function to fetch / discovery backends
	 */
	fetch FetchFunc				//consul-API 拉取后端节点的方法
	/**
	 * Options for fetch
	 */
	opts DiscoveryOpts			// 服务发现的配置选项
	/**
	 * Discovery configuration
	 */
	cfg config.DiscoveryConfig
	/**
	 * Channel where to push newly discovered backends
	 */
	out chan ([]core.Backend)	// 用于和 scheduler 模块通信的 channel，用于传送最新的 backends（后端节点）
	/**
	 * Channel for stopping discovery
	 */
	stop chan bool
}
```

[`discovery.consulFetch` 方法](https://github.com/yyyar/gobetween/blob/master/src/discovery/consul.go#L44)，通过 consul 的 Client，通过调用 `client.Health().Service()` 方法获取后端节点列表，并存在 `backends *[]core.Backend` 中：
```golang
func consulFetch(cfg config.DiscoveryConfig) (*[]core.Backend, error) {
	......
	scheme := "http"
	transport := &http.Transport{
		DisableKeepAlives: true,
	}

	// Enable tls if needed
	if cfg.ConsulTlsEnabled {
		tlsConfig := &consul.TLSConfig{
			Address:  cfg.ConsulHost,
			CertFile: cfg.ConsulTlsCertPath,
			KeyFile:  cfg.ConsulTlsKeyPath,
			CAFile:   cfg.ConsulTlsCacertPath,
		}
		tlsClientConfig, err := consul.SetupTLSConfig(tlsConfig)
		if err != nil {
			return nil, err
		}
		transport.TLSClientConfig = tlsClientConfig
		scheme = "https"
	}

	// Parse http timeout
	timeout := utils.ParseDurationOrDefault(cfg.Timeout, consulTimeout)

	// Create consul client
	client, _ := consul.NewClient(&consul.Config{
		Token:      cfg.ConsulAclToken,
		Scheme:     scheme,
		Address:    cfg.ConsulHost,
		Datacenter: cfg.ConsulDatacenter,
		HttpAuth: &consul.HttpBasicAuth{
			Username: cfg.ConsulAuthUsername,
			Password: cfg.ConsulAuthPassword,
		},
		HttpClient: &http.Client{Timeout: timeout, Transport: transport},
	})

	// Query service
	service, _, err := client.Health().Service(cfg.ConsulServiceName, cfg.ConsulServiceTag, cfg.ConsulServicePassingOnly, nil)
	if err != nil {
		return nil, err
	}

	// Gather backends
	backends := []core.Backend{}
	for _, entry := range service {
		s := entry.Service
		sni := ""

		for _, tag := range s.Tags {
			split := strings.SplitN(tag, "=", 2)

			if len(split) != 2 {
				continue
			}

			if split[0] != "sni" {
				continue
			}
			sni = split[1]
		}

		var host string
		if s.Address != "" {
			host = s.Address
		} else {
			host = entry.Node.Address
		}

		backends = append(backends, core.Backend{
			Target: core.Target{
				Host: host,
				Port: fmt.Sprintf("%v", s.Port),
			},
			Priority: 1,
			Weight:   1,
			Stats: core.BackendStats{
				Live: true,
			},
			Sni: sni,
		})
	}

	return &backends, nil
}
```
####	负载均衡选择组件 Balancer
gobetween 的负载均衡算法也是 `map[string]reflect.Type` 方式存储的，通过 `name` 初始化注册，这里我们简单分析下 [`RoundrobinBalancer` 的实现](https://github.com/yyyar/gobetween/blob/master/src/balance/roundrobin.go)：
```golang
var typeRegistry = make(map[string]reflect.Type)

func init() {
	typeRegistry["roundrobin"] = reflect.TypeOf(RoundrobinBalancer{})
}

func New(sniConf *config.Sni, balance string) core.Balancer {
	balancer := reflect.New(typeRegistry[balance]).Elem().Addr().Interface().(core.Balancer)

	if sniConf == nil {
		return balancer
	}

	return &middleware.SniBalancer{
		SniConf:  sniConf,
		Delegate: balancer,
	}
}
```

实现 `Balancer` 的核心是实现 `Elect` 方法，该方法在 [`Scheduler.HandleBackendElect()` 方法](https://github.com/yyyar/gobetween/blob/master/src/server/scheduler/scheduler.go#L270) 中被调用，达到根据负载均衡算法选择后端的目的：
```golang
/**
 * Roundrobin balancer
 */
type RoundrobinBalancer struct {

	/* Current backend position */
	current int
}

/**
 * Elect backend using roundrobin strategy
 */
func (b *RoundrobinBalancer) Elect(context core.Context, backends []*core.Backend) (*core.Backend, error) {

	if len(backends) == 0 {
		return nil, errors.New("Can't elect backend, Backends empty")
	}

	sort.SliceStable(backends, func(i, j int) bool {
		return backends[i].Target.String() < backends[j].Target.String()
	})

	if b.current >= len(backends) {
		b.current = 0
	}

	backend := backends[b.current]
	b.current += 1

	return backend, nil
}
```

####	健康检查组件 Healthcheck
[`Healthcheck` 结构](https://github.com/yyyar/gobetween/blob/master/src/healthcheck/healthcheck.go) 如下，值得关注的是 `workers []*Worker` 这个成员实现了一个 [工作池 workerpool](https://github.com/yyyar/gobetween/blob/master/src/healthcheck/worker.go)，该组件的核心流程如下所示：

![healthycheck](https://b1.sbimg.org/file/chevereto-jia/2020/10/22/GlQXw.png)

```golang
type Healthcheck struct {
	/* Healthcheck function */
	check CheckFunc
	/* Healthcheck configuration */
	cfg config.HealthcheckConfig
	/* Input channel to accept targets */
	In chan []core.Target		// 与 scheduler 模块通信 channel，接收待探测的后端节点
	/* Output channel to send check results for individual target */
	Out chan CheckResult		// 与 scheduler 模块通信 channel，将拨测结果告知 scheduler
	/* Current check workers */
	workers []*Worker
	/* Channel to handle stop */
	stop chan bool
}
```

如官方文档描述，healthycheck 提供了 3 种健康探测的方式，在 `init()` 中进行了注册，每个 value 都是对应的方法入口，在 `Healthcheck` 的初始化方法 `healthcheck.New()` 中传递给 `Healthcheck` 结构的 `check` 成员：
```golang
func init() {
	registry["ping"] = ping
	registry["probe"] = probe
	registry["exec"] = exec
	registry["none"] = nil
}

func New(strategy string, cfg config.HealthcheckConfig) *Healthcheck {
	// 根据 strategy 的名字获取对应的探测方法的指针
	check := registry[strategy]
	h := Healthcheck{
		check:   check,	// 初始化
		cfg:     cfg,
		In:      make(chan []core.Target),
		Out:     make(chan CheckResult),
		workers: []*Worker{},		// 初始化为空
		stop:    make(chan bool),
	}

	return &h
}
```

healthycheck 的核心循环 `healthy.Start()` 逻辑中，主要完成了 `2` 件事情：
1.	监听 `this.In` 这个 channel，从 `Scheduler` 模块获取最新的后端列表
2.	监听 `this.stop`，触发上层的控制退出指令，清理资源和退出

```golang
func (this *Healthcheck) Start() {

	go func() {
		for {
			select {

			/* got new targets */
			case targets := <-this.In:
				this.UpdateWorkers(targets)

			/* got stop requst */
			case <-this.stop:
				// Stop all workers
				for i := range this.workers {
					this.workers[i].Stop()
				}
				// And free it's memory
				this.workers = []*Worker{}

				return
			}
		}
	}()
}
```

我们再看下 `this.UpdateWorkers()` 这个方法，根据传入的后端节点 `targets`，
```golang
func (this *Healthcheck) UpdateWorkers(targets []core.Target) {
	//result 为本次更新的 workerlist
	result := []*Worker{}
	// Keep or add needed workers
	for _, t := range targets {
		var keep *Worker
		// 遍历旧的 workers 列表
		for i := range this.workers {
			c := this.workers[i]
			if t.EqualTo(c.target) {
				// 还在使用的 worker
				keep = c
				break
			}
		}

		// 如果没找到针对 targets 的 worker，那么就新建一个 worker
		if keep == nil {
			keep = &Worker{
				target: t,
				stop:   make(chan bool),
				out:    this.Out,		//this.Out 为 healthycheck 的 channel
				cfg:    this.cfg,
				check:  this.check,
				LastResult: CheckResult{
					Status: Initial,
				},
			}
			// 启动 worker
			keep.Start()
		}
		// 将目前在使用的 worker 加入列表
		result = append(result, keep)
	}

	// Stop needed workers
	// 移除不再使用的 worker
	for i := range this.workers {
		c := this.workers[i]
		remove := true
		for _, t := range targets {
			if c.target.EqualTo(t) {
				remove = false
				break
			}
		}

		if remove {
			// 向 worker-goroutine 发送退出 channel
			c.Stop()
		}
	}

	// 更新 worker 列表
	this.workers = result
}

func (this *Healthcheck) Stop() {
	this.stop <- true
}
```

最后，简单看下 `Worker` 的结构定义，及核心工作流程 [`Worker.Start()` 方法](https://github.com/yyyar/gobetween/blob/master/src/healthcheck/worker.go#L52)，此方法完成了如下事情：
1.	开启一个定时器 `time.NewTicker(interval)` 定期检测，`interval` 的时间由配置指定
2.	单独开启 goroutine 运行 `this.check` 方法，在此方法中，执行真正的健康检查逻辑，结果通过 `result chan<- CheckResult` 异步通知 worker
3.	在 `this.process` 方法中，将检测的结果通过 `healthy.Out` 这个 channel 传递给 `Scheduler` 模块
4.	等待上层通知本 worker 退出

```golang
type Worker struct {
	/* Target to monitor and check */
	target core.Target
	/* Function that does actual check */
	check CheckFunc
	/* Channel to write changed check results */
	out chan<- CheckResult
	/* Healthcheck configuration */
	cfg config.HealthcheckConfig
	/* Stop channel to worker to stop */
	stop chan bool
	/* Last confirmed check result */
	LastResult CheckResult
	/* Current passes count, if LastResult.Live = true */
	passes int
	/* Current fails count, if LastResult.Live = false */
	fails int
}

/**
 * Start worker
 */
func (this *Worker) Start() {
	// Special case for no healthcheck, don't actually start worker
	if this.cfg.Kind == "none" {
		return
	}

	interval, _ := time.ParseDuration(this.cfg.Interval)

	ticker := time.NewTicker(interval)
	c := make(chan CheckResult, 1)

	go func() {
		/* Check health before any delay*/
		log.Debug("Initial check", this.cfg.Kind, "for", this.target)
		go this.check(this.target, this.cfg, c)
		for {
			select {

			/* new check interval has reached */
			case <-ticker.C:
				log.Debug("Next check", this.cfg.Kind, "for", this.target)
				go this.check(this.target, this.cfg, c)

			/* new check result is ready */
			case checkResult := <-c:
				log.Debug("Got check result", this.cfg.Kind, ":", checkResult)
				this.process(checkResult)

			/* request to stop worker */
			case <-this.stop:
				ticker.Stop()
				//close(c) // TODO: Check!
				return
			}
		}
	}()
}

func (this *Worker) process(checkResult CheckResult) {
	//......
	if checkResult.Status == this.LastResult.Status {
		// check status not changed
		return
	}

	if checkResult.Status == Unhealthy {
		this.passes = 0
		this.fails++
	} else if checkResult.Status == Healthy {
		this.fails = 0
		this.passes++
	}

	if this.passes == 0 && this.fails >= this.cfg.Fails ||
		this.fails == 0 && this.passes >= this.cfg.Passes {
		this.LastResult = checkResult

		log.Info("Sending to scheduler:", this.LastResult)
		// 通过 out 这个 channel（本质是 healthy 的 Out）将结果通知给 Scheduler
		this.out <- checkResult
	}
}
```

####	Metrics 组件 Handler



##	0x08	Metrics 模块
Metrcis 模块的代码 [位于此](https://github.com/yyyar/gobetween/blob/master/src/metrics/metrics.go)，主要完成了如下事情（较为典型的实现）：<br>
1.	暴露了给 Prometheus-Client 的采集的 [接口](https://github.com/yyyar/gobetween/blob/master/src/metrics/metrics.go#L188)
2.	提供给 Manage 模块上报 Prometheus-Vec 的方法：
	-	`ReportHandleStatsChange`
	-	`ReportHandleBackendStatsChange`
	-	`ReportHandleOp`
	-	`ReportHandleStatsChange`

Metrics 的代码实现非常简洁及典型。主要指标如下，个人感觉也加入访问后端的延迟，成功率等等。
```golang
serverCount             *prometheus.GaugeVec
serverActiveConnections *prometheus.GaugeVec
serverRxTotal           *prometheus.GaugeVec
serverTxTotal           *prometheus.GaugeVec
serverRxSecond          *prometheus.GaugeVec
serverTxSecond          *prometheus.GaugeVec

backendActiveConnections  *prometheus.GaugeVec
backendRefusedConnections *prometheus.GaugeVec
backendTotalConnections   *prometheus.GaugeVec
backendRxBytes            *prometheus.GaugeVec
backendTxBytes            *prometheus.GaugeVec
backendRxSecond           *prometheus.GaugeVec
backendTxSecond           *prometheus.GaugeVec
backendLive               *prometheus.GaugeVec
```

##	0x09	API 模块
API 模块的代码 [位于此](https://github.com/yyyar/gobetween/tree/master/src/api)，主要使用 `gin` 构建的 API 管理端。主要提供了如下一些功能：
-	[Dump current config as TOML](https://github.com/yyyar/gobetween/blob/master/src/api/root.go#L42)：导出当前配置
-	[Servers Restful api implementation](https://github.com/yyyar/gobetween/blob/master/src/api/servers.go#L21)：通过调用 `manage` 模块提供的接口来操作后端资源及属性


##  0x0A	参考
-   [golb](https://github.com/onestraw/golb)
-   [vulcand](https://github.com/vulcand/vulcand)
-   [gobetween](https://github.com/yyyar/gobetween)


