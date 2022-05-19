---
layout: post
title: 基于 CRON 库扩展的分布式 Crontab 的实现
subtitle: 基于 golang 的分布式定时器任务模型的通用实现
date: 2022-01-16
author: pandaychen
catalog: true
tags:
  - Crontab
  - Golang
---

## 0x00 前言

最新项目中使用到了分布式定时器事件（场景是服务器定期修改密码），本篇文章小结下。

## 0x01 单机 CRON 库介绍
[Golang CRON 库 Crontab 的使用与设计](https://pandaychen.github.io/2021/10/05/A-GOLANG-CRONTAB-V3-BASIC-INTRO/) 前文介绍了单机 Cron 的使用及分析，但是在现实环境中，需要解决单个节点（进程）的高可用问题，保证定时任务正常执行不受影响。

####	应用的需求
-	保证任务不能被多个节点重复执行
-	单个节点宕机时能迅速转移任务至正常节点
-	任务的健壮性保证，执行结果反馈

## 0x02 分布式的实现

为了实现 HA，满足分布式高可用的特性，且实现满足两种定时任务需求的 crontab 系统：

1.  执行周期性任务
2.  只执行一次的任务

试想一下，我们的系统应该是下面这样的，满足如下特性：
![distribute-crontab](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/dcron1.png)

- 存储：定时任务必须落地在满足 AP 或 CP 的分布式系统中，典型的系统如 Redis（AP）、Etcd（CP）
- 工作节点的可用性：系统在定时器触发的那一刻需要至少选出一个工作节点（worknode）来执行任务
- 负载均衡：根据任务数据和节点数据均衡分发任务（避免任务都调度到负载较高的 worknode 上）
- 平滑扩容：如果任务节点负载过大，直接启动新的服务器后部分任务会自动迁移至新服务实现无缝扩容，各个 worknode 最好是无状态的
- 故障转移：单个节点故障，系统会自动将任务自动转移至其他正常节点执行
- 任务唯一：同一个服务内同一个任务只会启动单个运行实例，不会重复执行

## 0x03 源码分析

[dcron](https://github.com/libi/dcron) 正是采用 redis 实现的一个分布式定时任务库。其整体架构如下：
![dcron 架构图](https://raw.githubusercontent.com/libi/dcron/master/dcron.png)

dcron 的背景和我们遇到的场景 [基本类似](https://libisky.com/post/Dcron-%E5%9F%BA%E4%BA%8Eredis%E4%B8%8E%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E5%BA%93)，主要实现多机分布式场景下的定时任务执行问题。

####	核心数据结构
1、`Dcron`：代表一个进程（机器），用以注册（操作）任务，每个 `Dcron` 都关联了唯一的 `NodePool`<br>
```golang
// 一个 cron 代表一个机器或者一个进程
type Dcron struct {
	jobs       map[string]*JobWrapper // 存储任务
	mu         sync.RWMutex
	ServerName string
	nodePool   *NodePool			// 本 dcron 对应的分布式 hashring
	isRun      bool

	logger interface{Printf(string, ...interface{}) }

	nodeUpdateDuration time.Duration
	hashReplicas       int			// 一致性 hash 的虚拟备份数目

	cr        *cron.Cron 			// 单机 CRONTAB，dcron 注册的任务最终会落地到此
	crOptions []cron.Option
}
```

2、`NodePool`：指向本 `Dcron` 的分布式一致性 hash ring，该结构通过一致性 hash 算法包含了所有在线的 Dcron 节点 <br>
```golang
//NodePool is a node pool
type NodePool struct {
	serviceName string
	NodeID      string		//nodeid：唯一

	mu    sync.Mutex
	nodes *consistenthash.Map

	Driver         driver.Driver
	hashReplicas   int
	hashFn         consistenthash.Hash
	updateDuration time.Duration

	dcron *Dcron
}
```

`NodePool` 的重要方法有 2 个，`tickerUpdatePool` 和 `PickNodeByJobName`：

-	`tickerUpdatePool`：定时从 `driver` 中获取 **在线的所有**`Dcron` 的进程（节点）列表，计算本进程的一致性 hash ring（这里每次都是重新生成 hash ring，是否可以优化为增量动态增删？）
-	`PickNodeByJobName`：根据 `jobName`，计算从一致性 hash ring 上选中的虚拟节点（有可能出现选中的不是自己，因为 `updatePool` 是取到所有的在线节点）

所以，dcron 借助分布式一致性 hash 的特性，实现了分布式的特性。

```golang
// 构建一致性 hash
func (np *NodePool) updatePool() error {
	np.mu.Lock()
	defer np.mu.Unlock()
	// 获取所有在线的节点列表
	nodes, err := np.Driver.GetServiceNodeList(np.serviceName)
	if err != nil {
		return err
	}
	// 重新生成 hashring
	np.nodes = consistenthash.New(np.hashReplicas, np.hashFn)
	for _, node := range nodes {
		np.nodes.Add(node)
	}
	return nil
}

//PickNodeByJobName : 使用一致性 hash 算法根据任务名获取一个执行节点
func (np *NodePool) PickNodeByJobName(jobName string) string {
	np.mu.Lock()
	defer np.mu.Unlock()
	if np.nodes.IsEmpty() {
		return ""
	}
	return np.nodes.Get(jobName)
}
```

3、`Dcron` 的核心方法 <br>
-	`addJob`：屏蔽同名（ID）任务，调用 `cronv3` 的接口注册任务
-	`allowThisNodeRun`：任务排他实现

```golang
func (d *Dcron) addJob(jobName, cronStr string, cmd func(), job Job) (err error) {
	d.info("addJob'%s':  %s", jobName, cronStr)
	if _, ok := d.jobs[jobName]; ok {
		return errors.New("jobName already exist")
	}
	innerJob := JobWarpper{
		Name:    jobName,
		CronStr: cronStr,
		Func:    cmd,
		Job:     job,
		Dcron:   d,
	}

	//job 是存储在 cron.Cron 里面
	entryID, err := d.cr.AddJob(cronStr, innerJob)
	if err != nil {
		return err
	}
	innerJob.ID = entryID
	d.jobs[jobName] = &innerJob

	return nil
}

func (d *Dcron) allowThisNodeRun(jobName string) bool {
	allowRunNode := d.nodePool.PickNodeByJobName(jobName)
	d.info("job'%s'running in node %s", jobName, allowRunNode)
	if allowRunNode == "" {
		d.err("node pool is empty")
		return false
	}
	//distributed-cron:server2:ecfb1bb6-71fc-4b84-bdd0-61e92dd4de0d
	return d.nodePool.NodeID == allowRunNode
}
```


4、`JobWarpper`：前文说到过 Cron 库支持的 `JobWarpper` 功能，这里为了实现 **单个同名（ID）任务只在一个节点上运行的功能**，封装了 `JobWarpper`：<br>
```golang
//JobWarpper is a job warpper
type JobWarpper struct {
	ID      cron.EntryID
	Dcron   *Dcron
	Name    string
	CronStr string
	Func    func()
	Job     Job
}

//Run is run job
func (job JobWarpper) Run() {
	// 如果该任务分配给了这个节点 则允许执行
	if job.Dcron.allowThisNodeRun(job.Name) {
		if job.Func != nil {
			job.Func()
		}
		if job.Job != nil {
			job.Job.Run()
		}
	}
}
```


#### 核心数据流

dcron 的核心运行数据流如下：
![dcron-core-data-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/dcron-core-flow.png)

主要关注如下几点细节：
1.	一致性 hash 的使用场景
2.	高可用及节点（进程）迁移是如何实现的？
3.	任务互斥是如何实现的？
4.	cron 任务的封装和存储（dcron 是存储在 redis 中）


####	分布式 dcron 的使用流程
-	`1.1.1.1` 机器，运行 `1` 个进程，注册 serviceA、serviceB
-	`1.1.1.2` 机器，运行 `1` 个进程，注册 serviceB、serviceC
-	`1.1.1.3` 机器，运行 `2` 个进程，其中进程一注册 serviceA、serviceC，另外一个进程注册 serviceA、serviceB、serviceC

![tasks](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/task.png)


#### 分布式一致性 hash 的使用
回顾下，分布式一致性 hash 算法解决了分布式环境中因服务节点的变动而导致的缓存大量的失效的问题，也常应用于负载均衡的场景中，从 dcron 的实现思路中，如定时任务触发时的实现，最终会通过 `PickNodeByJobName` 方法（一致性 hash 的选取 node 实现）获取到一个节点，然后任务在这个选中的节点上运行。

在 dcron 中，一致性 hash 主要用处体现在：
-	对每个注册到中心（redis/etcd 等）的节点（进程），由 dcron 按照一致性 hash 结合虚拟节点的生成算法，初始化 hash ring（只对当前在线的节点或进程计算），定时更新
-	对于每个存活的 dcron 节点（进程），其注册的 job 任务到期触发执行时，先使用一致性 hash 算法从 hash ring 中选择一个节点，由于分布式一致性 hash 的特性，所有 dcron 节点选择出的节点 `NodeId` 必然是相同的（各个节点对一致性 hash 查找算法传入的 key 也需要一致）

此外，这里唯一的保证是，假设有多个进程或者多台机器都同时注册了某个相同的任务 `taskId`，那么当触发时，由于分布式一致性 hash 的特性，根据唯一的 `jobName`，选中的 `NodeId` 也必然是唯一的，这样与自己的 `NodeId` 进行比较后，相同的才运行，不相同的则跳过此次任务（注意看下面 `allowThisNodeRun` 方法中 `d.nodePool.NodeID == allowRunNode` 逻辑）。

```golang
//Run is run job

func (job JobWarpper) Run() {
	// 如果该任务分配给了这个节点 则允许执行
	if job.Dcron.allowThisNodeRun(job.Name) { // 主要看 allowThisNodeRun 的实现
		if job.Func != nil {
			job.Func()
		}
		if job.Job != nil {
			job.Job.Run()
		}
	}
}

//.....
func (d *Dcron) allowThisNodeRun(jobName string) bool {
	allowRunNode := d.nodePool.PickNodeByJobName(jobName)
	d.info("job'%s'running in node %s", jobName, allowRunNode)
	if allowRunNode == "" {
		d.err("node pool is empty")
		return false
	}
	return d.nodePool.NodeID == allowRunNode	//
}

//PickNodeByJobName : 使用一致性 hash 算法根据任务名获取一个执行节点
func (np *NodePool) PickNodeByJobName(jobName string) string {
	np.mu.Lock()
	defer np.mu.Unlock()
	if np.nodes.IsEmpty() {
		return ""
	}
	return np.nodes.Get(jobName)
}
```

#### NodeId 的生成

对于每个启动的进程，其在一致性 hash ring 注册的 NodeId 按照如下方法生成（这里可以加上本机 IP）：

```golang
func (rd *RedisDriver) randNodeID(serviceName string) (nodeID string) {
	return rd.getKeyPre(serviceName) + uuid.New().String()
}
```

#### 一致性 hash ring 的生成

一致性 hash 中，映射的 key 是进程（节点 id，即 `Add` 方法的参数，形如 `distributed-cron:server2:ecfb1bb6-71fc-4b84-bdd0-61e92dd4de0d`）ID，每个节点会分为 `m.replicas` 个虚拟节点：
```golang
// Adds some keys to the hash.
func (m *Map) Add(keys ...string) {
	for _, key := range keys {
		for i := 0; i < m.replicas; i++ {
			hash := int(m.hash([]byte(strconv.Itoa(i) + key)))
			if m.hashMap[hash] == "" {
				m.keys = append(m.keys, hash)
				m.hashMap[hash] = key
			}
		}
	}
	sort.Ints(m.keys)
}
```

####	一致性 hash ring 的查找
查找也易理解，返回节点 ID：
```golang
// Gets the closest item in the hash to the provided key.
func (m *Map) Get(key string) string {
	if m.IsEmpty() {
		return ""
	}

	hash := int(m.hash([]byte(key)))

	// Binary search for appropriate replica.
	idx := sort.Search(len(m.keys), func(i int) bool { return m.keys[i] >= hash })

	// Means we have cycled back to the first replica.
	if idx == len(m.keys) {
		idx = 0
	}

	return m.hashMap[m.keys[idx]]
}
```

这样，就容易理解 dcron 的任务排他性的实现机制了，注册了相同任务的每个不同的 Dcron 都会定时触发同一任务，但是由于分布式一致 hash 的选择，只能选出唯一的节点 `ID`，这样每个 dcron 将此 `ID` 与自己的 `ID` 做比较，相等才能执行，这样就实现了任务的唯一性。

## 0x04 可改进的思路
上述讨论了 dcron 生成 `NodePool` 的模式是基于主动 pull 方式定时（`10s`）获取所有在线节点，若在这 `10s` 内，如果有节点下线或者故障，而且有任务触发执行并调度到下线的节点上，那么任务会执行失败。这里有两种优化思路：
1.	采用 etcd/consul 等中间件的 notify 方式，通知各个节点主动更新 `NodePool`
2.	给 dcron 节点加上主动心跳上报，开发者可以及时感知进程在线情况
3.	dcron 的 REDIS 存储是采用 `Scan` 指令，即扫描通用前缀获取当前注册的所有节点的列表，如果 redis 集群不支持此指令的话，那么可能需要换其他的方式（如采用 `LIST` 结构存储所有的节点 id）

####	Kubernetes 部署
现网中，我们是如此在 kubernetes 部署的，并且实现了如下几点优化：
![dcron-k8s-change](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/dcron-k8s-change.png)
-	增加了 namespace 的支持，比如任务是属于某个 namespace 下的
-	增加了 etcd 支持，采用 etcd 存储每个 pod 节点信息及任务信息（实际上 pod 节点和主机节点的功能一样）
-	对外部提供 API 接口操作任务的增加 / 删除，需要注意，由于 dcron 的任务是存储在单进程的内存中（其他进程不可见），有如下几种可选的改造或应用方式：
	-	以 Service + 单个 pod 启动：同时启动多个 Service，注册的时候向多个 Service 均发送定时任务的注册请求
	-	以 Service + 单个 pod 启动：依据 Service 对 pod 的负载均衡方式（比如轮询），向所有的 pod 发送定时任务的注册请求，此方式不够优雅
	-	以单 Service + 多个 pod 启动：单 pod 下，在 dcron 中增加异步任务同步模块，注册任务接口将任务写入分布式存储（如 Etcd/redis 等），任务同步模块实时将任务取出，存储到自己的 dcron 中，比较推荐此方式实现

这样，以 Service 方式部署于 kubernetes 中，初始化默认生成多个 pod 副本就可以实现任务定时执行的高可用了。


##	0x05	总结
本文分析了分布式任务调度中间件 dcron 的实现。每个启动的 dcron 示例都将其注册入分布式存储，各个 dcron 使用一致性 hash 算法来选举出执行单个任务的节点来保证唯一性，所有节点都按照写入的 cron 预执行，在任务执行入口处根据一致性 hash 算法来判断该任务是否应该由当前节点执行。

## 0x06 参考

- [dcron 库](https://github.com/libi/dcron)
- [Dcron: 基于 redis 与一致性哈希算法的分布式定时任务库](https://libisky.com/post/Dcron-%E5%9F%BA%E4%BA%8Eredis%E4%B8%8E%E4%B8%80%E8%87%B4%E6%80%A7%E5%93%88%E5%B8%8C%E7%AE%97%E6%B3%95%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E5%AE%9A%E6%97%B6%E4%BB%BB%E5%8A%A1%E5%BA%93)
