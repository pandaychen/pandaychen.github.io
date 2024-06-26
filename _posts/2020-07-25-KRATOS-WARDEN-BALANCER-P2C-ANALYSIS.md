---
layout:     post
title:      Kratos 源码分析：Warden 负载均衡算法之 P2C
subtitle:   分析 Warden 实现的负载均衡算法 P2C（Power of Two Choices）
date:       2020-07-25
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

##  0x00    前言
Warden 框架默认实现了两种负载均衡算法：
1.  WRR：Weight Round Robin（Dynamic）
2.  P2C：Power of Two Choice

之前的文章[Kratos源码分析：分析 Kratos 中的 Dynamic-Wrr 负载均衡算法的实现](https://pandaychen.github.io/2020/03/11/KRATOS-LB-ALGORITHM-ANALYSIS/)，简单分析了WRR算法的实现思路及解决的场景。那么WRR算法存在哪些问题呢？

从B站的实践经验来看，动态感知的 WRR 算法，利用每次 RPC 请求返回的 Response 夹带 CPU 使用率，尽可能感知到服务负载，并且每隔一段时间整体调整一次节点的权重分数，虽然解决了原生WRR的权重固定的问题，但是存在羊群效应问题：["发现其实是信息滞后和分布式带来的羊群效应"](https://mp.weixin.qq.com/s/OAjcQdQxQzTRRv6q52JPNg)，即

> 就是WRR版本的负载均衡算法虽然会自动刷新权重值，但是在刷新时无法做到完全的实时，再快也不可能超过一个 RTT，都会存在一些信息延迟差。当后台资源比较稀缺时，遇到网络抖动时，就可能会把该节点炸掉，但是在监控上面是感觉不到的，因为 CPU 已经被平均掉了。

从上面的描述看，问题的本质在于gRPC的负载均衡是在客户端实现的，客户端每次请求完获取到服务端CPU数据，可能会存在延迟，在延迟的空档期，大量请求涌入，导致某个后端Node接收大量请求，从而不可用的问题。

####	WRR (Weighted Round Robin)
该算法在加权轮询法基础上增加了动态调节权重值，用户可以在为每一个节点先配置一个初始的权重分，之后算法会根据节点 cpu、延迟、服务端错误率、客户端错误率动态打分，在将打分乘用户自定义的初始权重分得到最后的权重值。


##  0x01    P2C 算法介绍
Power of Two Choices (P2C，两次随机选择) 负载均衡算法，主要用于为每个 RPC 请求返回一个 Server 节点以供调用，该算法策略出自论文 [《The Power of Two Random Choices: A Survey of Techniques and Results》](http://www.eecs.harvard.edu/~michaelm/NEWWORK/postscripts/twosurvey.pdf) ，主要思路如下：

1.  从可用后端节点列表中做 `2` 次选择操作（随机算法 or 依据一定策略来选择），得到节点 `nodeA`、`nodeB`
2.  比较 `nodeA`、`nodeB` 两个节点，选出负载最低（一般是正在处理的连接数 / 请求数最少）的节点作为被选中的节点

伪代码如下：
```golang
nodeA = random_choice(nodes)
nodeB = random_choice(nodes)
best = least_connection_choice([nodeA, nodeB])
```

此外，有几篇详细的算法实践的文章推荐阅读下：
-   来自 HAPROXY 的 [Test Driving Power of Two Random Choices Load Balancing](https://www.haproxy.com/blog/power-of-two-load-balancing/)
-   [The power of two random choices](https://brooker.co.za/blog/2012/01/17/two-random.html)

##	0x02	思考
既然 gRPC 的负载均衡的算法是在客户端实现的，那么如何设计一款好的负载均衡算法？

1.	默认情况下（HTTP2）每个 gRPC-Client 都维护这与 N 个后端 gRPC-Server 的长连接
2.	`N` 个后端服务节点，初始化有一个权重值，每次调用 RPC 方法前，先根据算法打分，权重高的后端被选择用于本次 RPC 过程，那么如何调整权重值？有几个维度需要关心的
	-	gRPC-Client 实时根据当前的后端服务端节点的负载动态调整权重：考虑服务端的 CPU（区间采样，如何消除毛刺？）、内存信息
	-	gRPC-Client 本身记录当前并发的（积压）RPC 请求
	-	gRPC-Client 每次完成 RPC 请求，记录到这个后端服务节点的 latency（延迟）
	-	gRPC-Client 累计的请求失败率、服务端失败率（不同视角）
3.	考虑 RPC 失败后的 retry 重试逻辑
4.	考虑客户端 gRPC-Client 在收到服务端 gRPC-Server 的指标信息的滞后（客户端接收服务端指标数据可能有延迟的场景）

这些点和疑问，在阅读 Warden 负载均衡的算法时，都得到了解答。

##	0x03 P2C 实现
[官方的文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-balancer.md) 上给出了实现的一些关键细节，先列举下，然后再结合代码做下分析：

#### P2C (Pick of two choices) 算法描述
本算法通过随机选择两个 `node` 选择优胜者来避免羊群效应，并通过 EWMA 尽量获取服务端的实时状态。

针对 **服务端** 的指标：
服务端获取最近 `500ms` 内的 CPU 使用率（针对容器场景：需要将 `cgroup` 设置的限制考虑进去，并除于 CPU 核心数），并将 CPU 使用率乘与 1000 后塞入每次 gRPC 请求中的的 `Trailer` 中夹带返回


针对 **客户端** 的指标，主要参数如下：
* server_cpu：通过每次请求中服务端塞在 trailer 中的 cpu_usage 拿到服务端最近 500ms 内的 cpu 使用率
* inflight：当前客户端正在发送并等待 response 的请求数（pending request）
* latency: 加权移动平均算法计算出的接口延迟
* client_success: 加权移动平均算法计算出的请求成功率（只记录 gRPC 内部错误，比如 context deadline）

####	权重计算
计算权重的公式如下：

$$\frac{success*metaWeight}{cpu*\sqrt{lag}*(inflight+1)}$$

这个公式的含义很直观，对权重有积极影响的因子，如成功率，初始权重等，位于分子；对权重有消极影响的因子，如cpu负载（过高）、lag延迟（过大）、以及积压的请求数inflight，放在分母的位置。（为了防止除法溢出，inflight做了加`1`处理）

####	EWMA 算法简介
P2C 代码实现中大量使用了 EWMA 算法（指数加权移动平均算法），此算法，是对观察值分别给予不同的权数，按不同权数求得移动平均值，并以最后的移动平均值为基础，确定预测值的方法。采用加权移动平均法，是因为观察期的近期观察值对预测值有较大影响，它更能反映近期变化的趋势。
-	指数移动加权平均法，是指各数值的加权系数随时间呈指数式递减，越靠近当前时刻的数值加权系数就越大
-	指数移动加权平均较传统的平均法来说，一是不需要保存过去所有的数值；二是计算量显著减小

感兴趣的可以阅读这篇文章：[指数加权移动平均法（EWMA）](https://www.cnblogs.com/jiangxinyang/p/9705198.html)

EWMA 算法的修正（预测）公式是：

$$Z_i=w*X_{cur} + (1-w)*Z_{i-1}$$

其中，在 P2C 算法中，w 系数按照下面的规则获取：
`w := math.Exp(float64(-td) / float64(tau))`

预测的方法是，每隔一段时间进行一次采样，每次采样完成之后，就对预测值进行一次修正，这种方法的特点是近期的采样值对预测值的影响大，远期的影响较小。从算法应用场景可知，和 P2C 算法计算场景比较类似，这里针对 `p2c.subConn` 结构使用 EWMA 算法，只需要保存上一次计算拿到的结果即可。

##  0x04	代码分析

全局变量：
```golang
const (
	// The mean lifetime of `cost`, it reaches its half-life after Tau*ln(2).
	tau = int64(time.Millisecond * 600)
	// if statistic not collected,we add a big penalty to endpoint
	penalty = uint64(1000 * time.Millisecond * 250)

	forceGap = int64(time.Second * 3)
)
```

`p2c.subConn`，封装了 `balancer.SubConn`，代表了 Client 到 Server 的一条长连接，封装了核心属性（计算权重需要）：
其中重要的字段说明如下（牢记一个 subConn 代表了客户端到某个服务端 Node 的唯一属性）：
-	`meta`：在服务发现（Etcd）中设置的 Node 的初始值
-	`lag`：请求延迟（用于与下次实现加权计算）
-	`success`：使用加权算法拿到的客户端 RPC 调用成功率
-	`inflight`：当前正在处理的请求数
-	`svrCPU`：保存了服务端返回的最近一段时间的 CPU 使用率
-	`stamp`：保存上次计算权重的时间戳（Nano）

gRPC客户端的连接池如下图所示：
![p2c-subconn](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kratos/loadbalance/GRPC-LB-P2C.png)

```golang
type subConn struct {
	// metadata
	conn balancer.SubConn
	addr resolver.Address
	meta wmd.MD

	//client statistic data
	lag      uint64
	success  uint64
	inflight int64
	// server statistic data
	svrCPU uint64

	//last collected timestamp
	stamp int64
	//last pick timestamp
	pick int64
	// request number in a period time
	reqs int64
}
```


`p2c.subConn` 实现的方法：
-	`valid`：
-	`health`：获取当前长连接中存储的成功率 `sc.success` 的值
-	`load`：
-	`cost`：

```golang
func (sc *subConn) valid() bool {
	return sc.health()> 500 && atomic.LoadUint64(&sc.svrCPU) < 900
}

func (sc *subConn) health() uint64 {
	return atomic.LoadUint64(&sc.success)
}

func (sc *subConn) load() uint64 {
	lag := uint64(math.Sqrt(float64(atomic.LoadUint64(&sc.lag))) + 1)
	load := atomic.LoadUint64(&sc.svrCPU) * lag * uint64(atomic.LoadInt64(&sc.inflight))
	if load == 0 {
		// penalty 是初始化没有数据时的惩罚值，默认为 1e9 * 250
		load = penalty
	}
	return load
}

func (sc *subConn) cost() uint64 {
	load := atomic.LoadUint64(&sc.svrCPU) * atomic.LoadUint64(&sc.lag) * uint64(atomic.LoadInt64(&sc.inflight))
	if load == 0 {
		// penalty 是初始化没有数据时的惩罚值，默认为 1e9 * 250
		load = penalty
	}
	return load
}
```

####	指标的计算详解-inflight
![inflight](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/kratos/loadbalance/GRPC-LB-P2C-1.png)




####	LB 核心逻辑 -- Builder
`p2cPickerBuilder.Build` 在每次后端节点有增减的情况下调用，初始化时会调用一次，`readySCs` 保存了后端服务器的基础信息，从代码实现上看，也是做了对节点的初始化工作：
当后端有删减时，也会强制把已存在的后端节点进行初始化（感觉这一点，gRPC 不是很友好）：

这里后端的初始化数据为
-	`svrCPU`:   500
-	`lag`：      0
-	`success`：1000
-	`inflight`： 1

```golang
type p2cPickerBuilder struct{}

func (*p2cPickerBuilder) Build(readySCs map[resolver.Address]balancer.SubConn) balancer.Picker {
	p := &p2cPicker{
		colors: make(map[string]*p2cPicker),
		// 初始化 rand 种子
		r:      rand.New(rand.NewSource(time.Now().UnixNano())),
	}

	// 初始化后端的权重数据
	for addr, sc := range readySCs {
		fmt.Println("[p2cPickerBuilder.Builder]init:",addr,sc)
		meta, ok := addr.Metadata.(wmd.MD)
		if !ok {
			meta = wmd.MD{
				Weight: 10,
			}
		}
		subc := &subConn{
			conn: sc,
			addr: addr,
			meta: meta,

			svrCPU:   500,
			lag:      0,
			success:  1000,
			inflight: 1,
		}
		if meta.Color == "" {
			p.subConns = append(p.subConns, subc)
			continue
		}
		// if color not empty, use color picker
		// 如果服务端定义了 color 筛选，则启动 color 的选择逻辑（测试时没用）
		cp, ok := p.colors[meta.Color]
		if !ok {
			cp = &p2cPicker{r: rand.New(rand.NewSource(time.Now().UnixNano()))}
			p.colors[meta.Color] = cp
		}
		cp.subConns = append(cp.subConns, subc)
	}
	return p
}
```

####	LB 核心逻辑 - Picker
`p2c.Picker` 实现了负载均衡的选择逻辑：
1.	`Pick`：主方法入口
2.	先调用 `prePick`，选择两个随机的 node
3.	最后从上一步的 node 列表中选取一个合适的 node，返回其对应的`subConn`，执行 RPC 请求
4.	根据RPC请求结果及 node 回带的服务端 CPU 字段信息，更新本 `subConn` 的核心因子信息
	-	`stamp`
	-	`success`
	-	`lag`

`p2cPicker` 结构：
```golang
type p2cPicker struct {
	// subConns is the snapshot of the weighted-roundrobin balancer when this picker was
	// created. The slice is immutable. Each Get() will do a round robin
	// selection from it and return the selected SubConn.
	subConns []*subConn		// 保存所有的后端
	colors   map[string]*p2cPicker
	logTs    int64		//
	r        *rand.Rand
	lk       sync.Mutex
}
```

接下来着重分析下 `pick` 和 `prePick` 方法：

`prePick` 方法，实现了随机选择的逻辑，总循环 `3` 次，随机从 `subConns []*subConn` 中选择两个节点 `nodeB` 和 `nodeA`，如果满足 `node.valid()` 的要求，直接返回，不满足的话，返回最后一次的选择的两个节点：
注意返回值中的 `nodeA` 和 `nodeB` 都是 `subConn` 结构。
```golang
// choose two distinct nodes
func (p *p2cPicker) prePick() (nodeA *subConn, nodeB *subConn) {
	for i := 0; i < 3; i++ {
		p.lk.Lock()
		a := p.r.Intn(len(p.subConns))
		b := p.r.Intn(len(p.subConns) - 1)
		p.lk.Unlock()
		if b >= a {
			b = b + 1
		}
		nodeA, nodeB = p.subConns[a], p.subConns[b]
		if nodeA.valid() || nodeB.valid() {
			break
		}
	}
	return
}
```
`picker` 的实现如下，重要部分已加了注释，需要关注的有如下几点信息：
1.	`DoneInfo()` 是在 RPC 方法执行完成后的回调，主要用于在 gRPC 的 `Trailer` 返回的 `pc` 对应的服务端 CPU 信息，根据此 CPU 信息，更新 `pc` 这个 `subConn` 的相关信息
2.	计算权重分数的方法，每次请求来时我们都会更新延迟，并且把之前获得的时间延迟进行权重的衰减，新获得的时间提高权重，这样就实现了滚动更新

```golang
func (p *p2cPicker) Pick(ctx context.Context, opts balancer.PickInfo) (balancer.SubConn, func(balancer.DoneInfo), error) {
	// FIXME refactor to unify the color logic
	color := nmd.String(ctx, nmd.Color)
	if color == ""&& env.Color !="" {
		color = env.Color
	}
	if color != "" {
		if cp, ok := p.colors[color]; ok {
			return cp.pick(ctx, opts)
		}
	}
	return p.pick(ctx, opts)
}


func (p *p2cPicker) pick(ctx context.Context, opts balancer.PickInfo) (balancer.SubConn, func(balancer.DoneInfo), error) {
	var pc, upc *subConn
	start := time.Now().UnixNano()

	if len(p.subConns) <= 0 {
		return nil, nil, balancer.ErrNoSubConnAvailable
	} else if len(p.subConns) == 1 {
		// 只有 1 个节点，直接返回
		pc = p.subConns[0]
	} else {
		nodeA, nodeB := p.prePick()
		// meta.Weight 为服务发布者在 disocvery 中设置的权重
		if nodeA.load()*nodeB.health()*nodeB.meta.Weight > nodeB.load()*nodeA.health()*nodeA.meta.Weight {
			//pc 为本次算法选择的节点
			pc, upc = nodeB, nodeA
		} else {
			pc, upc = nodeA, nodeB
		}
		// 如果选中的节点，在 forceGap 期间内没有被选中一次，那么强制一次
		// 利用强制的机会，来触发成功率、延迟的衰减
		// 原子锁 conn.pick 保证并发安全，放行一次
		pick := atomic.LoadInt64(&upc.pick)
		if start-pick > forceGap && atomic.CompareAndSwapInt64(&upc.pick, pick, start) {
			pc = upc
		}
	}

	// 节点未发生切换才更新 pick 时间
	if pc != upc {
		atomic.StoreInt64(&pc.pick, start)
	}

	atomic.AddInt64(&pc.inflight, 1)
	atomic.AddInt64(&pc.reqs, 1)

	// RPC 方法执行完成后，更新状态
	return pc.conn, func(di balancer.DoneInfo) {
		atomic.AddInt64(&pc.inflight, -1)
		now := time.Now().UnixNano()
		// get moving average ratio w
		stamp := atomic.SwapInt64(&pc.stamp, now)
		td := now - stamp
		if td < 0 {
			td = 0
		}
		w := math.Exp(float64(-td) / float64(tau))

		lag := now - start
		if lag < 0 {
			lag = 0
		}
		oldLag := atomic.LoadUint64(&pc.lag)
		if oldLag == 0 {
			w = 0.0
		}
		lag = int64(float64(oldLag)*w + float64(lag)*(1.0-w))
		atomic.StoreUint64(&pc.lag, uint64(lag))

		success := uint64(1000) // error value ,if error set 1
		if di.Err != nil {
			if st, ok := status.FromError(di.Err); ok {
				// only counter the local grpc error, ignore any business error
				if st.Code() != codes.Unknown && st.Code() != codes.OK {
					success = 0
				}
			}
		}
		oldSuc := atomic.LoadUint64(&pc.success)
		success = uint64(float64(oldSuc)*w + float64(success)*(1.0-w))
		atomic.StoreUint64(&pc.success, success)

		// 从服务端的 Trailer 中拿到 CPU 的值
		trailer := di.Trailer
		if strs, ok := trailer[wmd.CPUUsage]; ok {
			if cpu, err2 := strconv.ParseUint(strs[0], 10, 64); err2 == nil && cpu > 0 {
				atomic.StoreUint64(&pc.svrCPU, cpu)
			}
		}

		logTs := atomic.LoadInt64(&p.logTs)
		if now-logTs > int64(time.Second*3) {
			if atomic.CompareAndSwapInt64(&p.logTs, logTs, now) {
				p.printStats()
			}
		}
	}, nil
}
```

这里我们把 RPC 调用成功时的回调逻辑简单分析下：
```golang
start := time.Now().UnixNano()
...
return pc.conn, func(di balancer.DoneInfo) {
	// 当前正在处理的请求数减 1，好理解
	atomic.AddInt64(&pc.inflight, -1)

	// 取当前的时间戳（Nano）
	now := time.Now().UnixNano()
	// get moving average ratio w
	// 获取 && 设置上次测算的时间点，将 pc.stamp 的值更新为 now
	stamp := atomic.SwapInt64(&pc.stamp, now)

	// 获取时间间隔
	td := now - stamp
	if td < 0 {
		td = 0
	}

	// 获取时间衰减系数
	w := math.Exp(float64(-td) / float64(tau))

	// 获得本次延迟数据 1（注意 start 是在 pick 开始计时的）
	lag := now - start
	if lag < 0 {
		lag = 0
	}

	// 获取上次保存的延迟数据 2
	oldLag := atomic.LoadUint64(&pc.lag)
	if oldLag == 0 {
		w = 0.0
	}

	// 延迟数据 1 与延迟数据 2，计算出平均延迟（EWMA）
	lag = int64(float64(oldLag)*w + float64(lag)*(1.0-w))

	// 保存本地计算出的延迟数据
	atomic.StoreUint64(&pc.lag, uint64(lag))

	success := uint64(1000) // error value ,if error set 1
	if di.Err != nil {
		if st, ok := status.FromError(di.Err); ok {
			// only counter the local grpc error, ignore any business error
			if st.Code() != codes.Unknown && st.Code() != codes.OK {
				success = 0
			}
		}
	}
	oldSuc := atomic.LoadUint64(&pc.success)
	success = uint64(float64(oldSuc)*w + float64(success)*(1.0-w))
	atomic.StoreUint64(&pc.success, success)

	// 从服务端的 Trailer 中拿到 CPU 的值
	trailer := di.Trailer
	if strs, ok := trailer[wmd.CPUUsage]; ok {
		if cpu, err2 := strconv.ParseUint(strs[0], 10, 64); err2 == nil && cpu > 0 {
			atomic.StoreUint64(&pc.svrCPU, cpu)
		}
	}

	logTs := atomic.LoadInt64(&p.logTs)
	if now-logTs > int64(time.Second*3) {
		// 超过一个 3s 的周期，尝试打印当前状态
		if atomic.CompareAndSwapInt64(&p.logTs, logTs, now) {
			p.printStats()
		}
	}
}, nil
```

###	统计节点

```text
//INFO 07/24-07:50:56.452 /root/delete/kratos-note/pkg/net/rpc/warden/balancer/p2c/p2c.go:292 p2c  : [{addr:127.0.0.1:8081 score:783813.9288438039 cs:1000 lantency:2612717 cpu:789 inflight:1 reqs:1}]
```

```golang
// statistics is info for log
type statistic struct {
	addr     string
	score    float64
	cs       uint64
	lantency uint64
	cpu      uint64
	inflight int64
	reqs     int64
}


func (p *p2cPicker) printStats() {
	if len(p.subConns) <= 0 {
		return
	}
	stats := make([]statistic, 0, len(p.subConns))
	for _, conn := range p.subConns {
		var stat statistic
		log.Info("p2c detail [OLD] info %s : %+v", p.subConns[index].addr.ServerName, conn)
		stat.addr = conn.addr.Addr
		stat.cpu = atomic.LoadUint64(&conn.svrCPU)
		stat.cs = atomic.LoadUint64(&conn.success)
		stat.inflight = atomic.LoadInt64(&conn.inflight)
		stat.lantency = atomic.LoadUint64(&conn.lag)
		stat.reqs = atomic.SwapInt64(&conn.reqs, 0)
		load := conn.load()
		if load != 0 {
			stat.score = float64(stat.cs*conn.meta.Weight*1e8) / float64(load)
		}
		stats = append(stats, stat)
	}
	log.Info("p2c %s : %+v", p.subConns[0].addr.ServerName, stats)
    for index,conn:=range p.subConns{
        log.Info("p2c detail [NEW] info %s : %+v", p.subConns[index].addr.ServerName, conn)
    }
}

```


##	0x05	测试
我们给关键路径加上日志，看下这里具体参数和值的变化过程，这样对算法能够有更为直观的理解，我们以 etcd 为注册服务，启动三个后端（单个单个启动），观察下数据的变化。



##  0x06	参考
-	[指数加权移动平均法（EWMA）](https://www.cnblogs.com/jiangxinyang/p/9705198.html)
-	[Warden Balancer DOC](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-balancer.md)
-	[一场跨年晚会挣了50亿，B站在微服务治理中如何探索与实践？](https://mp.weixin.qq.com/s/OAjcQdQxQzTRRv6q52JPNg)