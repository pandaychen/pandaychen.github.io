---
layout:     post
title:      Consul 服务治理的那些事（二）
subtitle:   Consul 的细节梳理
date:       2020-10-13
author:     pandaychen
catalog:    true
tags:
    - Consul
    - 负载均衡
    - gRPC
---

##  0x00    前言
上一篇文章 [Consul 服务治理的那些事](https://pandaychen.github.io/2019/10/12/CONSUL-APPLICATION/) 介绍了 Consul 的基础原理和使用，本文对 Consul 构建项目中的一些细节再做下梳理。consul 是完整的解决方案，与 etcd，zooKeeper 等 k/v 存储相比，无需额外的二次开发就能通过 HTTP 快速实现服务发现和注册。基于 consul agent，原生支持包括 script，http，tcp，TTL，docker，gRPC 等的健康检查，灵活的健康检查简化了服务容灾。基于 gossip 协议实现的原生多数据中心支持可以快速的实现跨 DC，VPC 部署，便于实现业务的高可用和扩展。

关于 Consul 的一些高级方法，推荐 [玩转 CONSUL 系列的文章](http://vearne.cc/archives/13983)

##  0x01    Consul 的高可用机制
consul 是通过 Agent 来运行的，在单个数据中心中，Agent 又分为 Server Agent 和 Client Agent（所有的节点也被称为 Agent）。
-   Server Agent 会将服务的消息存储起来，至少要启动一个 Server Agent，为了保证 HA，集群环境中 Server 通常部署多个；Server 节点有一个 Leader 和多个 Follower，Leader 节点会将数据同步到 Follower，Server 的数量推荐是 `3` 个或者 `5` 个，在 Leader 挂掉的时候会启动选举机制产生一个新的 Leader。

-   Client Agent 主要用于注销服务、健康检查及转发 Server Agent 的查询等，它相当于一个代理，所以此必须在集群的每台主机上都要运行

Consul 中使用 Raft 算法解决一致性问题，使用 Gossip 协议来管理集群成员和广播消息，其中 Raft 属于强一致性模型，Gossip 属于弱一致性模型。注意二者的使用场景是不同的

####    实现原理
其核心在于两点：
1.  集群内节点间信息的高效同步机制，其保障了拓扑变动以及控制信号的及时传递（使用 gossip 协议在集群内传播信息，如广播故障、成员关系等）
2.  Server 集群内日志存储的强一致性（使用 raft 协议来保障日志的一致性，选主还有事务相关等）


##  0x02    服务注册与发现
Consul 的服务发现模式如下图所示：

-   服务注册: 服务端将服务的信息注册到 Consul 里
-   服务发现: 客户端从 Consul 里发现服务信息，主要是服务的地址
-   健康检查: Consul 负责检查服务器的健康状态，基于设置检查策略不通过的情况下剔除节点

![img-consul-service](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consul/consul_service.png)

以项目 [grpclb2consul](https://github.com/pandaychen/grpclb2consul) 为例：


1、服务注册 <br>
在 [服务注册](https://github.com/pandaychen/grpclb2consul/blob/master/consul_discovery/register.go) 实现中，有两种方式：
-   [TTL](https://github.com/pandaychen/grpclb2consul/blob/master/consul_discovery/register.go#L79) 方式：通过定时上报 `PassTTL` 的方法来实现
-   [Agent 健康检查](https://github.com/pandaychen/grpclb2consul/blob/master/consul_discovery/register.go#L146)：在 `consulapi.AgentServiceCheck` 中设置 `Interval` 的值，通过 consul agent 进行健康检查

以后一种方式为例：
```golang
func (c *ConsulRegistry) RegisterWithHealthCheckGRPC() error {
	metadata, err := json.Marshal(c.GeneNodeData)
	if err != nil {
		c.Logger.Error("JSON marshal error", zap.String("errmsg", err.Error()))
		return err
	}
	tags := make([]string, 0)
	tags = append(tags, string(metadata))

	registerfunc := func() error {
		//健康检查
		healthcheck := &consulapi.AgentServiceCheck{
			Interval: "3s",
			GRPC:     fmt.Sprintf("%s:%d/%s", c.GeneNodeData.Ip, c.GeneNodeData.Port, "svcname"), // grpc 支持，执行健康检查的地址，service 会传到 Health.Check 函数中
			DeregisterCriticalServiceAfter: "1m", // 注销时间，相当于过期时间
		}

		crs := &consulapi.AgentServiceRegistration{
			ID:      c.GeneNodeData.UniqID, //uniq-id
			Name:    c.GeneNodeData.ServiceName,
			Address: c.GeneNodeData.Ip,   // 服务 IP
			Port:    c.GeneNodeData.Port, // 服务端口
			Tags:    tags,                // tags，可以为空([]string{})
			Check:   healthcheck}
		err := c.ConsulAgent.Agent().ServiceRegister(crs) //单例模式
		if err != nil {
			return fmt.Errorf("Register with consul error: %s\n", err.Error())
		}
		return nil
	}

	err = registerfunc()
	if err != nil {
		c.Logger.Error("Register with consul error", zap.String("errmsg", err.Error()))
		return err
	}

    //...

	return nil
}
```

服务注册的结构定义如下：
```golang
// AgentServiceRegistration is used to register a new service
type AgentServiceRegistration struct {
		Kind              ServiceKind       `json:",omitempty"`
		ID                string            `json:",omitempty"`
		Name              string            `json:",omitempty"`
		Tags              []string          `json:",omitempty"`
		Port              int               `json:",omitempty"`
		Address           string            `json:",omitempty"`
		EnableTagOverride bool              `json:",omitempty"`
		Meta              map[string]string `json:",omitempty"`
		Weights           *AgentWeights     `json:",omitempty"`
		Check             *AgentServiceCheck
		Checks            AgentServiceChecks
		// DEPRECATED (ProxyDestination) - remove this field
		ProxyDestination string                          `json:",omitempty"`
		Proxy            *AgentServiceConnectProxyConfig `json:",omitempty"`
		Connect          *AgentServiceConnect            `json:",omitempty"`
}
```

2、服务发现 <br>
参见 [consul resolver](https://github.com/pandaychen/grpclb2consul/blob/master/consul_discovery/resolver.go) 的实现，主要注意下监控方法 `WatcherHandler` 的实现[细节](https://github.com/pandaychen/grpclb2consul/blob/master/consul_discovery/watcher.go#L69)

consul提供的watcher方式有：


3、健康检查 <br>
值得一提的是，consul 检查服务器的健康状态，consul 用 `google.golang.org/grpc/health/grpc_health_v1.HealthServer` 接口，实现了对 gRPC 健康检查的支持。所以需要实现该接口以便于consul 利用此接口作健康检查，如[代码](https://github.com/pandaychen/grpclb2consul/blob/master/consul_discovery/register.go#L157)

```golang
healthcheck := &consulapi.AgentServiceCheck{
	Interval: "3s",
	GRPC:     fmt.Sprintf("%s:%d/%s", c.GeneNodeData.Ip, c.GeneNodeData.Port, "svcname"), // grpc 支持，执行健康检查的地址，service 会传到 Health.Check 函数中
	DeregisterCriticalServiceAfter: "1m", // 注销时间，相当于过期时间
}

crs := &consulapi.AgentServiceRegistration{
	ID:      c.GeneNodeData.UniqID, //uniq-id
	Name:    c.GeneNodeData.ServiceName,
	Address: c.GeneNodeData.Ip,   // 服务 IP
	Port:    c.GeneNodeData.Port, // 服务端口
	Tags:    tags,                // tags，可以为空([]string{})
	Check:   healthcheck}
err := c.ConsulAgent.Agent().ServiceRegister(crs) //单例模式
```

按照如上设置，consul Agent每隔`3s`会使用gRPC的标准健康检查接口发起一次健康[检查请求](https://github.com/pandaychen/grpclb2consul/blob/master/healthcheck/method.go#L42)，访问的endpoint是`fmt.Sprintf("%s:%d/%s", c.GeneNodeData.Ip, c.GeneNodeData.Port, "svcname")`，如下：
```golang
func (h *HealthyCheck) Check(ctx context.Context, in *pb.HealthCheckRequest) (*pb.HealthCheckResponse, error) {
	//more check method/logic could be add
	return &pb.HealthCheckResponse{Status: pb.HealthCheckResponse_SERVING}, nil
	//return &pb.HealthCheckResponse{Status: pb.HealthCheckResponse_NOT_SERVING }, nil
}
```

####	DeregisterCriticalServiceAfter参数的坑
`DeregisterCriticalServiceAfter`[选项](https://www.consul.io/api-docs/agent/check#deregistercriticalserviceafter)：用来设置当服务健康检查异常时超过多久时间服务，就开始注销本服务，consul会周期性发起健康检查，并且根据结果自动移除不可用的服务。不过，在测试时发现，此值设置为较小的值（如`10s`）并不生效。查询官网文档发现下面这段：
>	DeregisterCriticalServiceAfter (string: "") - Specifies that checks associated with a service should deregister after this time. This is specified as a time duration with suffix like "10m". If a check is in the critical state for more than this configured value, then its associated service (and all of its associated checks) will automatically be deregistered. The minimum timeout is 1 minute, and the process that reaps critical services runs every 30 seconds, so it may take slightly longer than the configured timeout to trigger the deregistration. This should generally be configured with a timeout that's much, much longer than any expected recoverable outage for the given service.

查询了下原项目代码[实现](https://github.com/hashicorp/consul/blob/main/agent/agent.go#L2994)，的确如文档描述，超时的最小限制是`1min`

####	consul agent的gRPC健康检查
如上代码所示，启用了gRPC健康检查之后，consul agent会根据开发传入的配置对服务进行探测，基于gRPC的健康监测`Check`接口，consul的实现[如下](https://github.com/hashicorp/consul/blob/main/agent/checks/grpc.go#L52)：
```golang
// Check if the target of this GrpcHealthProbe is healthy
// If nil is returned, target is healthy, otherwise target is not healthy
func (probe *GrpcHealthProbe) Check(target string) error {
	serverAndService := strings.SplitN(target, "/", 2)
	serverWithScheme := fmt.Sprintf("%s:///%s", resolver.GetDefaultScheme(), serverAndService[0])

	ctx, cancel := context.WithTimeout(context.Background(), probe.timeout)
	defer cancel()

	connection, err := grpc.DialContext(ctx, serverWithScheme, probe.dialOptions...)
	if err != nil {
		return err
	}
	defer connection.Close()

	client := hv1.NewHealthClient(connection)
	response, err := client.Check(ctx, probe.request)
	if err != nil {
		return err
	}
	if response.Status != hv1.HealthCheckResponse_SERVING {
		//非HealthCheckResponse_SERVING错误直接返回失败
		return fmt.Errorf("gRPC %s serving status: %s", target, response.Status)
	}

	return nil
}
```

####	Tags用法
参考[Brief overview of using consul tags](https://echorand.me/posts/consul-tags/)

####   Watch 机制：阻塞查询
同 Etcd 的 Watch 一样，Consul 也提供了 Watch 机制，基于 HTTP Long Polling 机制来实现。

##  0x03    Consul 用作服务发现的优势
参考 [Consul vs. Custom Solutions](https://www.consul.io/docs/intro/vs/custom)

####    consul with kubernetes

consul 与 kubernetes 完成了深度整合，契合了服务 docker 化趋势。主要包括以下功能:
-   k8s 内的 consul 部署工具 consul helm chart
-   k8s 自动加入。支持发现和加入基于 k8s 的 Agent 代理。这将使外部 Consul Agent 加入在 Kubernetes 中运行的 Consul 集群
-   k8s 的自动同步。将 k8s 集群内的服务同步至 consul，使得非 k8s 服务也能发现并连接内部服务
-   consul 服务同步至 k8s。以便应用程序可以使用 Kubernetes 本地服务发现来发现和连接在 Kubernetes 之外运行的服务
-   connect 自动注入。部署在 Kubernetes 中的 Pod 可以配置为自动使用 Connect 通过 TLS 进行安全通信


##  0x04    Consul 提供的 API
参考官方文档：[Service - Agent HTTP API](https://www.consul.io/api-docs/agent/service)


##  0x05    Consul 集群监控指标
-   进程：CPU、内存、goroutine、连接数
-   raft：成员状态变动、提交速率、提交耗时、同步心跳、同步延时
-   RPC：连接数、跨 DC 请求数
-   写负载：注册及解注册速率
-   读负载：`Catalog`/`Health`/`PreparedQuery` 请求量，执行耗时

##	0x06	一些细节问题&&总结
1、consul重复服务实例<br>
该问题是同一个服务实例（IP+Port）在Consul中注册并出现了多次，一般有两种原因：
-	每次注册时，使用了不同的实例ID（ServiceID)，比如随机数不同，旧的服务实例没有下线，又注册了新的服务实例
-	两次注册时，注册到不同的Server节点，即使同一实例名称，也可以注册到不同的Service节点


##  0x07    参考
-   [记一次 Consul 故障分析与优化](https://www.infoq.cn/article/qv02j2ezmjbow8ckcopg)
-   [golang consul-grpc 服务注册与发现](http://www.hatlonely.com/2018/06/23/golang-consul-grpc-%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/index.html)
-   [A command-line tool to perform health-checks for gRPC applications in Kubernetes etc.](https://github.com/grpc-ecosystem/grpc-health-probe)
-   [Announcing HashiCorp Consul + Kubernetes](https://www.hashicorp.com/blog/consul-plus-kubernetes)
-   [consul 原理解析](http://ljchen.net/2019/01/04/consul原理解析/)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权


