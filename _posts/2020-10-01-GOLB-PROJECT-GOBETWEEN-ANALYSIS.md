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
![image](https://camo.githubusercontent.com/252e2764756a51aea3ae4783fccef0a760eecb86/687474703a2f2f692e70696363792e696e666f2f69392f38623932313534343335626533326632316561613366663762336463366431632f313436363234343333322f37343435372f313034333438372f676f672e706e67)

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
6、[`Backend`(https://github.com/yyyar/gobetween/blob/master/src/core/backend.go)：定义了后端节点及统计信息的通用结构
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
```golang
type Server struct {
	/* Server friendly name */
	name string

	/* Listener */
	listener net.Listener

	/* Configuration */
	cfg config.Server

	/* Scheduler deals with discovery, balancing and healthchecks */
	scheduler scheduler.Scheduler

	/* Current clients connection */
	clients map[string]net.Conn

	/* Stats handler */
	statsHandler *stats.Handler

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
```




##	0x07	Metrics 模块
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

##	0x08	API 模块
API 模块的代码 [位于此](https://github.com/yyyar/gobetween/tree/master/src/api)，主要使用 `gin` 构建的 API 管理端。主要提供了如下一些功能：
-	[Dump current config as TOML](https://github.com/yyyar/gobetween/blob/master/src/api/root.go#L42)：导出当前配置
-	[Servers Restful api implementation](https://github.com/yyyar/gobetween/blob/master/src/api/servers.go#L21)：通过调用 `manage` 模块提供的接口来操作后端资源及属性


##  0x09	参考
-   [golb](https://github.com/onestraw/golb)
-   [vulcand](https://github.com/vulcand/vulcand)
-   [gobetween](https://github.com/yyyar/gobetween)


