---
layout: post
title: Confd 使用 && 源码分析
subtitle: Confd 源码分析：强大的动态配置更新（基于 etcdv3/redis 存储）
date: 2021-01-01
author: pandaychen
header-img: img/super-mario.jpg
catalog: true
category: false
tags:
  - Confd
  - Golang
---

## 0x00 前言

现网有类似下面的 Nginx 配置，如何对 `upstream` 指向的后端服务节点做动态的上下线切换（即时剔除无效节点）？一个可行的解决方案就是使用 Confd+Etcd。

```text
upstream backend_cluster {
    server 172.19.161.1:9081;
    server 172.19.161.2:9081;
    server 172.19.161.3:9081;
}

server {
        listen       8080;
        #..
        location ^~ /cgi/ {
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Cookie $http_cookie;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_pass http://backend_cluster/;
        proxy_pass_request_headers      on;
    }
}
```

最近准备给 Nginx 代理的配置文件提供动态更新 reload 的功能，采用 Etcd+Confd 实现，本文简单分析下 [Confd](https://github.com/kelseyhightower/confd) 的实现。<br>
Confd 是一个轻量级的配置管理工具，可以通过查询后端存储系统来实现第三方应用的（动态）配置管理，如 Nginx、HAproxy、Docker 配置等。Confd 能够查询和监听后端系统的数据变更，结合配置模版引擎动态更新本地配置文件，保持和后端系统的数据一致，并且能够执行命令或者脚本实现系统的 reload 或者 restart 等操作。

## 0x01 COnfd 的运行原理

一般配置中心的运行模式如下，这种方式需要在服务端代码中加入定期 PULL or Watch 配置文件发生改变，改变需要重新加载本进程配置，侵入性较强：
![config-center]()

而使用了 Confd 的方式，我们只要在服务端代码中加入配置热重启的逻辑就行（也可以直接如 Nginx 的 `reload` 指令），配置的远程同步交给 Confd 来完成。

![confd]()

## 0x02 Confd 的使用

1、配置文件 `confd.toml`，主要记录了使用的存储后端、`confdir` 等，参数 `watch` 表示实时监听 etcdV3 的变化，如有变化则更新 Confd 管理的配置（推荐使用）<br>

```toml
backend = "etcdv3"
confdir = "/etc/confd"
log-level = "debug"
interval = 5
nodes = [
  "http://127.0.0.1:2379",
]
scheme = "http"
watch = true
```

#### confdir 的目录配置

包含配置与模板两个目录：

- `./conf.d/`：Confd 的配置文件，包含配置的生成逻辑
- `./templates`：配置模板 Template，即基于不同应用组件的配置，遵循 Golang text templates 的模板文件，详见 [文档](https://github.com/kelseyhightower/confd/blob/master/docs/templates.md)

例如：

#### 同步生成 Nginx 配置

在实际线上应用中，可以考虑使用 [supervisord](http://supervisord.org/)+ Confd 的部署方式。

## 0x03 Confd 逻辑分析

Confd 的整体应用架构如下：
![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/etcd/confd-architecture.png)

从部署来看，Confd 需要完成如下几件事情：

1. 一个客户端动态 PULL 或者 Watch 远端配置中心的改变，及时将数据拉取到本地
2. 本地文件和上一步获取的文件内容比较
3. 更新本地文件及后续的自定义操作
4. 对模板语法的处理

#### 配置模板结构

[TemplateResource](https://github.com/kelseyhightower/confd/blob/master/resource/template/resource.go) 结构如下，此结构代表了解析后的每个模板文件：

```golang
type Config struct {
	ConfDir       string `toml:"confdir"`
	ConfigDir     string
	KeepStageFile bool
	Noop          bool   `toml:"noop"`
	Prefix        string `toml:"prefix"`
	StoreClient   backends.StoreClient
	SyncOnly      bool `toml:"sync-only"`
	TemplateDir   string
	PGPPrivateKey []byte
}

// TemplateResource is the representation of a parsed template resource.
type TemplateResource struct {
	CheckCmd      string `toml:"check_cmd"`
	Dest          string
	FileMode      os.FileMode
	Gid           int
	Keys          []string
	Mode          string
	Prefix        string
	ReloadCmd     string `toml:"reload_cmd"`
	Src           string
	StageFile     *os.File
	Uid           int
	funcMap       map[string]interface{}
	lastIndex     uint64
	keepStageFile bool
	noop          bool
	store         memkv.Store
	storeClient   backends.StoreClient
	syncOnly      bool
	PGPPrivateKey []byte
}
```

#### main 逻辑

启动部分的 [代码](https://github.com/kelseyhightower/confd/blob/master/confd.go#L16) 完成了几件事情：

1.  初始化配置文件的存储后端的 Client
2.  初始化配置模板

```golang
func main(){
    //...
    storeClient, err := backends.New(config.BackendsConfig)
	if err != nil {
		log.Fatal(err.Error())
	}

	config.TemplateConfig.StoreClient = storeClient
	if config.OneTime {
		if err := template.Process(config.TemplateConfig); err != nil {
			log.Fatal(err.Error())
		}
		os.Exit(0)
	}
}
```

接着就是标准的启动停止（处理信号），针对远程配置的监控，分为两种方式：

1. `template.WatchProcessor`：以 watch 方式监控
2. `template.IntervalProcessor`：定时轮询

```golang
func main(){
    //...

    stopChan := make(chan bool)
	doneChan := make(chan bool)
	errChan := make(chan error, 10)

	var processor template.Processor
	switch {
    case config.Watch:
        // 初始化 Watch 模式的结构体
		processor = template.WatchProcessor(config.TemplateConfig, stopChan, doneChan, errChan)
	default:
		processor = template.IntervalProcessor(config.TemplateConfig, stopChan, doneChan, errChan, config.Interval)
	}

	// 单独启动 process
	go processor.Process()

	signalChan := make(chan os.Signal, 1)
	signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM)
	for {
		select {
		case err := <-errChan:
			log.Error(err.Error())
		case s := <-signalChan:
			log.Info(fmt.Sprintf("Captured %v. Exiting...", s))
			close(doneChan)
		case <-doneChan:
			os.Exit(0)
		}
    }
}
```

#### WatchProcessor

我们先分析下 `WatchProcessor` 的实现，其中 [getTemplateResources 方法](https://github.com/kelseyhightower/confd/blob/master/resource/template/processor.go#L111) 是获取 Confd 模板目录下的所有模板文件，`watchProcessor.Process` 方法针对每个模板文件都建立一个单独的 Watcher：

```golang
type watchProcessor struct {
	config   Config
	stopChan chan bool		// 用于控制
	doneChan chan bool		// 用于控制
	errChan  chan error		// 错误上报
	wg       sync.WaitGroup	// 用于并发控制
}

func WatchProcessor(config Config, stopChan, doneChan chan bool, errChan chan error) Processor {
	var wg sync.WaitGroup
	return &watchProcessor{config, stopChan, doneChan, errChan, wg}
}

func (p *watchProcessor) Process() {
	defer close(p.doneChan)
	// 从模板配置中获取模板文件（ts 为 TemplateResource 结构数组）
	ts, err := getTemplateResources(p.config)
	if err != nil {
		log.Fatal(err.Error())
		return
	}
	for _, t := range ts {
		t := t
		p.wg.Add(1)
		// 对每个模板都开启独立的 goroutine
		go p.monitorPrefix(t)
	}
	p.wg.Wait()
}
```

`monitorPrefix` 方法用于监听每个模板文件的改变（当监听到远程配置中心发生改变时，从远程拉取一次配置并更新本地），注意下面这个 `t.lastIndex` 的意义。先看下 EtcdV3 的 [实现](https://github.com/kelseyhightower/confd/blob/master/backends/etcdv3/client.go#L225)，该方法是一个 `forever-loop`：

1. 使用 `t.storeClient.WatchPrefix`<font color="#dd0000"> 阻塞监听 Etcd 的指定 Key 值的改变 </font>，当发生改变时，返回 Etcd 中最新（发生修改的）数据的 Last seen revision
2. 保存本次的 `index` 值 `t.lastIndex`
3. 调用 `t.process()`[方法](https://github.com/kelseyhightower/confd/blob/master/resource/template/resource.go#L349) 更新本地配置，`t.process()` 方法的步骤为：
   - `setFileMode`[方法](https://github.com/kelseyhightower/confd/blob/master/resource/template/resource.go#L366)：设定文件掩码
   - `setVars`[方法](https://github.com/kelseyhightower/confd/blob/master/resource/template/resource.go#L175)：先从 Etcd 拉取对应 Key 的数据，然后按照格式以 key-value 方式保存在本地 [内存中](https://github.com/kelseyhightower/confd/blob/master/vendor/github.com/kelseyhightower/memkv/store.go)
   - `createStageFile`[方法](https://github.com/kelseyhightower/confd/blob/master/resource/template/resource.go#L198)：将上一步拉取的配置按照 template 模板保存为临时文件
   - `sync`[方法](https://github.com/kelseyhightower/confd/blob/master/resource/template/resource.go#L238)：应用新的配置（当改变时），视是否配置 `ReloadCmd` 调用相应命令重启服务
4. 继续下一次循环

实现如下：

```golang
// 监听每个模板文件
func (p *watchProcessor) monitorPrefix(t *TemplateResource) {
	defer p.wg.Done()
	keys := util.AppendPrefix(t.Prefix, t.Keys)
	for {
		// 调用不同类型的客户端 WatchPrefix 方法
		// 返回
		index, err := t.storeClient.WatchPrefix(t.Prefix, keys, t.lastIndex, p.stopChan)
		if err != nil {
			p.errChan <- err
			// Prevent backend errors from consuming all resources.
			time.Sleep(time.Second * 2)
			continue
		}
		t.lastIndex = index
		if err := t.process(); err != nil {
			p.errChan <- err
		}
	}
}
```

#### Confd 的客户端封装

#### Confd 的客户端实现：EtcdV3 的客户端

在 Confd 的 EtcdV3 [客户端的实现](https://github.com/kelseyhightower/confd/blob/master/backends/etcdv3/client.go) 中，重要的结构体是 `Client` 和 `Watch`，注意 `Client` 结构的 `watches` 成员，其存储了所有需要监听改变的 `key`，对应于配置文件中的 `key` 数组（ps：线上项目中建议开启 Etcd 客户端的 TLS + 认证机制）

```golang
// Client is a wrapper around the etcd client
type Client struct {
	client *clientv3.Client
	watches map[string]*Watch	// 每一个监控的 key 结构对映射一个独立的 Watcher
	// Protect watch
	wm sync.Mutex
}
```

而 `Watch` 结构则记录，当前关联的 `key` 在 Etcd 中存储最新的 `revision` 值，由于 Etcd 本身是个 MVCC 存储，所以我们保存了 `revision`（这里的 `revision` 是最新的 revision）：

```golang
// A watch only tells the latest revision
type Watch struct {
	// Last seen revision
	revision int64
	// A channel to wait, will be closed after revision changes
	cond chan struct{}
	// Use RWMutex to protect cond variable
	rwl sync.RWMutex
}
```

#### WatchPrefix 方法

分析下在上一小节 `watchProcessor.monitorPrefix` 中调用的 `WatchPrefix` 方法：

```golang
func (c *Client) WatchPrefix(prefix string, keys []string, waitIndex uint64, stopChan chan bool) (uint64, error) {
	var err error

	// Create watch for each key
	watches := make(map[string]*Watch)
	c.wm.Lock()
	// 针对 keys 中的每个 key，设置或新建监听的 client-watch 对象
	for _, k := range keys {
		watch, ok := c.watches[k]
		if !ok {
			// 对每个配置文件的 key，都创建单独的 goroutine 来监听其改变
			watch, err = createWatch(c.client, k)
			if err != nil {
				c.wm.Unlock()
				return 0, err
			}
			c.watches[k] = watch
		}
		// 转移到新的 map
		watches[k] = watch
	}
	c.wm.Unlock()

	ctx, cancel := context.WithCancel(context.Background())
	cancelRoutine := make(chan struct{})
	defer cancel()
	defer close(cancelRoutine)
	go func() {
		select {
		case <-stopChan:
			cancel()
		case <-cancelRoutine:
			return
		}
	}()

	notify := make(chan int64)
	// Wait for all watches
	for _, v := range watches {
		// 异步，调用 WaitNext 方法监控 w.revision 是否发生改变
		go v.WaitNext(ctx, int64(waitIndex), notify)
	}
	select{
		// 当有修改时，正常情况下 notify 会接收到新的 revision
	case nextRevision := <- notify:
		return uint64(nextRevision), err
	case <-ctx.Done():
		return 0, ctx.Err()
	}
	return 0, err
}
```

`WatchPrefix` 调用 `createWatch` 来监听器 `Watch`，此方法也是 Etcd-Watcher 的核心实现逻辑：

```golang
func createWatch(client *clientv3.Client, prefix string) (*Watch, error) {
	w := &Watch{0, make(chan struct{}), sync.RWMutex{}}
	go func() {
		rch := client.Watch(context.Background(), prefix, clientv3.WithPrefix(),
		clientv3.WithCreatedNotify())
		log.Debug("Watch created on %s", prefix)
		for {
			// 默认情况下，会阻塞在 rch 这个 channel 上面
			for wresp := range rch {
				if wresp.CompactRevision > w.revision {
					// respect CompactRevision
					w.update(wresp.CompactRevision)
					log.Debug("Watch to'%s'updated to %d by CompactRevision", prefix, wresp.CompactRevision)
				} else if wresp.Header.GetRevision()> w.revision {
					// Watch created or updated
					w.update(wresp.Header.GetRevision())
					log.Debug("Watch to'%s'updated to %d by header revision", prefix, wresp.Header.GetRevision())
				}
				if err := wresp.Err(); err != nil {
					log.Error("Watch error: %s", err.Error())
				}
			}
			// 当 rch 被外部调用关闭时
			log.Warning("Watch to'%s'stopped at revision %d", prefix, w.revision)
			// Disconnected or cancelled
			// Wait for a moment to avoid reconnecting
			// too quickly
			time.Sleep(time.Duration(1) * time.Second)
			// Start from next revision so we are not missing anything

			// 根据 w.revision 的值重新开始新一轮 watch
			if w.revision > 0 {
				// 从 revision+1 开始，其他 key 的改变也会导致 revision 增加，但是这里加了 WithPrefix 保证我们始终拿到 prefix 关联的 key
				rch = client.Watch(context.Background(), prefix, clientv3.WithPrefix(),
					clientv3.WithRev(w.revision + 1))
			} else {
				// Start from the latest revision
				//
				rch = client.Watch(context.Background(), prefix, clientv3.WithPrefix(),
				clientv3.WithCreatedNotify())
			}
		}
	}()
	return w, nil
}
```

`Client.WatchPrefix` 方法中调用了 `WaitNext` 方法，该方法是循环检测 `w.revision` 的值，当 `w.revision > lastRevision` 时，将 `w.revision` 的值通过 channel `notify` 通知给上层。否则，阻塞 `select...case` 上，等待 `update`[方法](https://github.com/kelseyhightower/confd/blob/master/backends/etcdv3/client.go#L51) 中唤醒。这里个人感觉需要阻塞的原因是因为避免多次对读写锁 `w.rwl` 加解锁？

```golang
// Update revision
func (w *Watch) update(newRevision int64){
	w.rwl.Lock()
	defer w.rwl.Unlock()
	w.revision = newRevision
	// 向 w.cond 发送信号
	close(w.cond)
	// 重新创建 w.cond
	w.cond = make(chan struct{})
}

// Wait until revision is greater than lastRevision
func (w *Watch) WaitNext(ctx context.Context, lastRevision int64, notify chan<-int64) {
	for {
		w.rwl.RLock()
		if w.revision > lastRevision {
			w.rwl.RUnlock()
			break
		}
		cond := w.cond
		w.rwl.RUnlock()
		select{
			// 一轮循环之后阻塞在此，等待 update 的逻辑中 close(w.cond) 唤醒
		case <-cond:
		case <-ctx.Done():
			return
		}
	}
	// We accept larger revision, so do not need to use RLock
	select{
	case notify<-w.revision:
	case <-ctx.Done():
	}
}
```

#### GetValues 方法

```golang
// GetValues queries etcd for keys prefixed by prefix.
func (c *Client) GetValues(keys []string) (map[string]string, error) {
	// Use all operations on the same revision
	var first_rev int64 = 0
	vars := make(map[string]string)
	// Default ETCDv3 TXN limitation. Since it is configurable from v3.3,
	// maybe an option should be added (also set max-txn=0 can disable Txn?)
	maxTxnOps := 128
	getOps := make([]string, 0, maxTxnOps)
	doTxn := func (ops []string) error {
		ctx, cancel := context.WithTimeout(context.Background(), time.Duration(3) * time.Second)
		defer cancel()

		txnOps := make([]clientv3.Op, 0, maxTxnOps)

		for _, k := range ops {
			txnOps = append(txnOps, clientv3.OpGet(k,
			clientv3.WithPrefix(),
			clientv3.WithSort(clientv3.SortByKey, clientv3.SortDescend),
			clientv3.WithRev(first_rev)))
		}

		result, err := c.client.Txn(ctx).Then(txnOps...).Commit()
		if err != nil {
			return err
		}
		for i, r := range result.Responses {
			originKey := ops[i]
			// append a '/' if not already exists
			originKeyFixed := originKey
			if !strings.HasSuffix(originKeyFixed, "/") {
				originKeyFixed = originKey + "/"
			}
			for _, ev := range r.GetResponseRange().Kvs {
				k := string(ev.Key)
				if k == originKey || strings.HasPrefix(k, originKeyFixed) {
					vars[string(ev.Key)] = string(ev.Value)
				}
			}
		}
		if first_rev == 0 {
			// Save the revison of the first request
			first_rev = result.Header.GetRevision()
		}
		return nil
	}
	for _, key := range keys {
		getOps = append(getOps, key)
		if len(getOps) >= maxTxnOps {
			if err := doTxn(getOps); err != nil {
				return vars, err
			}
			getOps = getOps[:0]
		}
	}
	if len(getOps) > 0 {
		if err := doTxn(getOps); err != nil {
			return vars, err
		}
	}
	return vars, nil
}
```

## 0x04 基于 Redis 的实现

## 0x05 总结

## 0x06 参考

- [Quick Start Guide](https://github.com/kelseyhightower/confd/blob/master/docs/quick-start-guide.md)
