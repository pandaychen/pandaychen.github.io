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
  - 热更新
---

## 0x00 前言

现网有类似下面的 Nginx 配置，如何对 `upstream` 指向的后端服务节点做修改，手动的方式：
1.	修改`upstream`配置，增加/删除某些后端
2.	使用`nginx -s reload`指令，重启服务

那么，如何对nginx配置做动态的上下线切换（即时剔除无效节点）呢？一个可行的解决方案就是使用 Confd+Etcd。

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

笔者最近准备给 Nginx 代理的配置文件提供动态更新 `reload` 的功能，采用 Etcd+Confd 实现，本文简单分析下 [Confd](https://github.com/kelseyhightower/confd) 的原理实现。<br>

####	Confd介绍

Confd 是一个轻量级的配置管理工具，可以通过查询后端存储系统来实现第三方应用的（动态）配置管理，如 Nginx、HAproxy、Docker 配置等。Confd 能够查询和监听后端系统的数据变更，结合配置模版引擎动态更新本地配置文件，保持和后端系统的数据一致，并且能够执行命令或者脚本实现系统的 `reload` 或者 `restart` 等操作。

本文分析的版本是`v0.16.0`，参见官方[手册](https://github.com/kelseyhightower/confd/blob/master/docs/quick-start-guide.md)

confd支持多种的配置存储后端，本文分析etcd、redis、consul以及vault的客户端实现
-	etcd
-	consul
-	dynamodb
-	redis
-	vault
-	zookeeper
-	aws ssm parameter store
-	environment variables
-	file

## 0x01 Confd 的运行原理

一般配置中心的运行模式如下，这种方式需要在服务端代码中加入定期 PULL or Watch 配置文件发生改变，改变需要重新加载本进程配置，侵入性较强：
![config-center](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/confd/model-1.png)

而使用了 Confd 的方式，我们只要在服务端代码中加入配置热重启的逻辑就行（也可以直接如 Nginx 的 `reload` 指令），配置的远程同步交给 Confd 来完成。

![confd](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/confd/model-2.png)

## 0x02 Confd 的使用
1、安装confd，confd启动的配置文件 `confd.toml`，主要记录了使用的存储后端、`confdir` 等，参数 `watch` 表示实时监听 etcdV3 的变化，如有变化则更新 Confd 管理的配置（推荐使用）<br>
```toml
backend = "etcdv3"	#etcd为V2版本
confdir = "/etc/confd"	#confdir，配置文件模板的存储路径
log-level = "debug"
interval = 5
nodes = [
  "http://127.0.0.1:2379",
]
scheme = "http"
watch = true	#watch参数表示实时监听后端存储的变化，如有变化则更新confd管理的配置
```

重点的配置：
-	`backend`：存储后端的类型
-	`nodes`：后端节点的地址
-	`prefix`：存储的前缀，在获取变量时需要perfix+key才能找到存储在后端的变量
-	`confdir`：告诉需要被更新的template的存放位置
-	`watch`：触发类型（轮询or通知）

redis的后端配置，参考[Confd的使用](https://www.huweihuang.com/linux-notes/tools/confd-usage.html)

2、配置confdir 的目录以及模板文件<br>
创建confd的配置路径，包含配置与模板两个目录：
```bash
mkdir -p /etc/confd/{conf.d,templates}
```

- `./conf.d/`：Confd 的配置文件，包含配置的生成逻辑
- `./templates`：配置模板 Template，即基于不同应用组件的配置，遵循 Golang text templates 的模板文件，详见 [文档](https://github.com/kelseyhightower/confd/blob/master/docs/templates.md)

2.1、配置Template Resources文件<br>
模板源配置文件是`TOML`格式，包含配置的生成逻辑，例如模板源，后端存储对应的keys，命令执行等。默认目录在`/etc/confd/conf.d`；详细字段说明[见此](https://github.com/kelseyhightower/confd/blob/master/docs/template-resources.md)，例如配置了nginx模板文件`/etc/confd/conf.d/myapp-nginx.toml`

重点配置：
-	`prefix`：后端存储变量的前缀
-	`src`：template文件，即配置文件的模板文件（用占位符表达配置文件的内容，最后用拉取的kv替换占位符，生成应用程序使用的配置文件）
-	`dest`：替换完成后的配置文件存放的路径，即应用程序读取配置的路径
-	`check_cmd`：除了文件更新，confd支持一些shell命令的执行
-	`reload_cmd`：可以借助系统的reload命令重启

```toml
[template]
prefix = "/myapp"
src = "nginx.tmpl"
dest = "/tmp/myapp.conf"
owner = "nginx"
mode = "0644"
keys = [
  "/subdomain",
  "/upstream",
]
check_cmd = "/usr/sbin/nginx -t -c {{.src}}"
reload_cmd = "/usr/sbin/service nginx reload"
```

初始化template文件`/etc/confd/conf.d/yourapp-nginx.toml`，如下：

```text
[template]
prefix = "/yourapp"
src = "nginx.tmpl"
dest = "/tmp/yourapp.conf"
owner = "nginx"
mode = "0644"
keys = [
  "/subdomain",
  "/upstream",
]
check_cmd = "/usr/sbin/nginx -t -c {{.src}}"
reload_cmd = "/usr/sbin/service nginx reload"
```

2.2、配置Template文件<br>
Template定义了单一应用配置的模板，默认存储在`/etc/confd/templates`目录，模板文件符合Golang的`text/template`格式；模板文件常用函数有`base`，`get`，`gets`，`lsdir`，`json`等。如`/etc/confd/templates/nginx.tmpl`；注意，**实际中nginx配置是根据Template模板生成的**

```json
events {
    worker_connections 1024;
}

http{
	upstream {{getv "/subdomain"}} {
	{{range getvs "/upstream/*"}}
		server {{.}};
	{{end}}
	}

	server {
		server_name  {{getv "/subdomain"}}.example.com;
		location / {
			proxy_pass        http://{{getv "/subdomain"}};
			proxy_redirect    off;
			proxy_set_header  Host             $host;
			proxy_set_header  X-Real-IP        $remote_addr;
			proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
	}
	}
}
```

3、向存储中间件写入初始化数据<br>
如果初始化不写入数据，直接运行confd会报错，这里以etcdv2为例：
```bash
etcdctl set /myapp/subdomain myapp
etcdctl set /myapp/upstream/app2 "1.1.1.1:80"
etcdctl set /myapp/upstream/app1 "2.2.2.2:80"
etcdctl set /yourapp/subdomain yourapp
etcdctl set /yourapp/upstream/app2 "1.1.1.1:8080"
etcdctl set /yourapp/upstream/app1 "2.2.2.2:8080"
```

4、启动confd<br>
```BASH
/usr/local/bin/confd -config-file ./config.toml &
```

5、查看生成的配置文件<br>
```text
[root@VM_120_245_centos ~/confd]# cat /tmp/myapp.conf 
events {
    worker_connections 1024;
}

http{
upstream myapp {
    server 1.1.1.1:80;
    server 2.2.2.2:80;
}

server {
    server_name  myapp.example.com;
    location / {
        proxy_pass        http://myapp;
        proxy_redirect    off;
        proxy_set_header  Host             $host;
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
   }
}
}

[root@VM_120_245_centos ~/confd]# cat /tmp/yourapp.conf 
events {
    worker_connections 1024;
}

http{
upstream yourapp {
    server 1.1.1.1:8080;
    server 2.2.2.2:8080;
}

server {
    server_name  yourapp.example.com;
    location / {
        proxy_pass        http://yourapp;
        proxy_redirect    off;
        proxy_set_header  Host             $host;
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
   }
}
}
```

5、再添加多一条upstream信息<br>
```text
etcdctl set /myapp/upstream/app3 "3.3.3.3:80"
```

6、查看myapp对应的配置文件，成功的达到目标<br>
```text
[root@VM_120_245_centos ~/confd]# cat /tmp/myapp.conf 
events {
    worker_connections 1024;
}

http{
upstream myapp {
    server 1.1.1.1:80;
    server 2.2.2.2:80;
    server 3.3.3.3:80;
}

server {
    server_name  myapp.example.com;
    location / {
        proxy_pass        http://myapp;
        proxy_redirect    off;
        proxy_set_header  Host             $host;
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
   }
}
}
```

7、删除一个后端<br>
```text
etcdctl rm /myapp/upstream/app2
```

8、查看最终的文件<br>
```text
[root@VM_120_245_centos ~/confd]# cat /tmp/myapp.conf 
events {
    worker_connections 1024;
}

http{
upstream myapp {
    server 2.2.2.2:80;
    server 3.3.3.3:80;
}

server {
    server_name  myapp.example.com;
    location / {
        proxy_pass        http://myapp;
        proxy_redirect    off;
        proxy_set_header  Host             $host;
        proxy_set_header  X-Real-IP        $remote_addr;
        proxy_set_header  X-Forwarded-For  $proxy_add_x_forwarded_for;
   }
}
}
```

9、`/etc/confd`文件目录如下<br>
```text
[root@VM_120_245_centos /etc/confd]# tree -a
.
├── conf.d
│   ├── myapp-nginx.toml
│   └── yourapp-nginx.toml
└── templates
    └── nginx.tmpl
```


#### 同步生成 Nginx 配置

在实际线上应用中，可以考虑使用 [supervisord](http://supervisord.org/)与 Confd 的部署方式。

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

##	0x04	核心代码分析：主入口

#### main 入口逻辑

启动部分的 [代码](https://github.com/kelseyhightower/confd/blob/master/confd.go#L16) 完成了几件事情：

1.	[初始化配置文件](https://github.com/kelseyhightower/confd/blob/master/config.go)：默认启用etcd（V2），同时对各个后端进行检查
2.  初始化配置文件对应的存储后端的 Client
3.  初始化配置模板

```golang
func main(){
	//...
	if err := initConfig(); err != nil {
		log.Fatal(err.Error())
	}
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

第二步，根据配置文件中的存储后端类型构造一个存储后端的client（`backends.New`），初始化代码[见此](https://github.com/kelseyhightower/confd/blob/master/backends/client.go#L29)，客户端的封装模式非常标准的策略模式，值得借鉴：

每个`StoreClient`需要实现下面`2`个方法：
-	`GetValues`：获取指定的value
-	`WatchPrefix`：监听指定的prefix

`StoreClient`是对后端存储类型的抽象，不同的后端存储类型`GetValues`和`WatchPrefix`方法具体实现是不同的

```golang

// The StoreClient interface is implemented by objects that can retrieve
// key/value pairs from a backend store.
type StoreClient interface {
	GetValues(keys []string) (map[string]string, error)
	WatchPrefix(prefix string, keys []string, waitIndex uint64, stopChan chan bool) (uint64, error)
}

// New is used to create a storage client based on our configuration.
func New(config Config) (StoreClient, error) {

	if config.Backend == "" {
		config.Backend = "etcd"
	}
	backendNodes := config.BackendNodes

	if config.Backend == "file" {
		log.Info("Backend source(s) set to " + strings.Join(config.YAMLFile, ", "))
	} else {
		log.Info("Backend source(s) set to " + strings.Join(backendNodes, ", "))
	}

	switch config.Backend {
	case "consul":
		return consul.New(config.BackendNodes, config.Scheme,
			config.ClientCert, config.ClientKey,
			config.ClientCaKeys,
			config.BasicAuth,
			config.Username,
			config.Password,
		)
	case "etcd":
		// Create the etcd client upfront and use it for the life of the process.
		// The etcdClient is an http.Client and designed to be reused.
		return etcd.NewEtcdClient(backendNodes, config.ClientCert, config.ClientKey, config.ClientCaKeys, config.ClientInsecure, config.BasicAuth, config.Username, config.Password)
	case "etcdv3":
		return etcdv3.NewEtcdClient(backendNodes, config.ClientCert, config.ClientKey, config.ClientCaKeys, config.BasicAuth, config.Username, config.Password)
	case "zookeeper":
		return zookeeper.NewZookeeperClient(backendNodes)
	case "rancher":
		return rancher.NewRancherClient(backendNodes)
	case "redis":
		return redis.NewRedisClient(backendNodes, config.ClientKey, config.Separator)
	case "env":
		return env.NewEnvClient()
	case "file":
		return file.NewFileClient(config.YAMLFile, config.Filter)
	case "vault":
		vaultConfig := map[string]string{
			"app-id":    config.AppID,
			"user-id":   config.UserID,
			"role-id":   config.RoleID,
			"secret-id": config.SecretID,
			"username":  config.Username,
			"password":  config.Password,
			"token":     config.AuthToken,
			"cert":      config.ClientCert,
			"key":       config.ClientKey,
			"caCert":    config.ClientCaKeys,
			"path":      config.Path,
		}
		return vault.New(backendNodes[0], config.AuthType, vaultConfig)
	case "dynamodb":
		table := config.Table
		log.Info("DynamoDB table set to " + table)
		return dynamodb.NewDynamoDBClient(table)
	case "ssm":
		return ssm.New()
	}
	return nil, errors.New("Invalid backend")
}
```

接着就是标准的启动停止（处理信号），针对远程配置的监控，分为两种方式：

1. `template.WatchProcessor`：以 watch 方式动态监控；会定时检查本地配置和远端配置的差异，如果有更新就同步
2. `template.IntervalProcessor`：定时轮询；从etcd来看，会阻塞在watcher上，等待被通知远端监控的数据发生改变了才触发本地配置与远端最新配置的检测，有更新才同步（注：为了通用性考虑，这里并没有采用增量式，watcher仅仅起到通知的功能，最终还是要去fetch一次远端配置来触发本地更新）

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

####	核心结构：Process
`Process`的定义如下，实体化的结构有：
-	`intervalProcessor`：默认的实现体，即没有添加watch参数
-	`watchProcessor`：添加watch参数的实现体

```golang
type Processor interface {
    Process()
}
```

1、WatchProcessor：通过watch机制触发核心逻辑<br>


1.1、`WatchProcessor`定义及`Process`方法<br>
我们先分析下 `WatchProcessor` 的实现，其中 [getTemplateResources 方法](https://github.com/kelseyhightower/confd/blob/master/resource/template/processor.go#L111) 是获取 Confd 模板目录下的所有模板文件，`watchProcessor.Process` 方法针对每个模板文件都建立一个单独的 Watcher：

`watchProcessor`及核心方法`watchProcessor.Process`定义如下，`Process`的功能是对于confd配置的每个Template Resources文件，都开启一个独立的goroutine处理：

```golang
type watchProcessor struct {
	config   Config
	stopChan chan bool		// 用于控制
	doneChan chan bool		// 用于控制
	errChan  chan error		// 错误上报
	wg       sync.WaitGroup	// 用于并发控制
}

//定义
func WatchProcessor(config Config, stopChan, doneChan chan bool, errChan chan error) Processor {
	var wg sync.WaitGroup
	return &watchProcessor{config, stopChan, doneChan, errChan, wg}
}

func (p *watchProcessor) Process() {
	defer close(p.doneChan)
	// 从模板配置中获取模板文件（ts 为 TemplateResource 结构数组）；可能有多个模板文件
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

再看下`monitorPrefix`方法。特别注意：**confd在初始化时，各个客户端都实现了强制拉取一次远端的数据，然后再阻塞在各个watcher上**，生成初始化配置，这点在分析客户端实现的时候再说明

1.2、`monitorPrefix`方法<br>

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
		// 返回（阻塞，有变更才返回）
		index, err := t.storeClient.WatchPrefix(t.Prefix, keys, t.lastIndex, p.stopChan)
		if err != nil {
			p.errChan <- err
			// Prevent backend errors from consuming all resources.
			time.Sleep(time.Second * 2)
			continue
		}
		//保存index
		t.lastIndex = index
		if err := t.process(); err != nil {
			p.errChan <- err
		}
	}
}
```

2、`intervalProcessor`方法：轮询检查<br>

2.1、`intervalProcessor`定义及`Process`方法<br>

```GOLANG
type intervalProcessor struct {
    config   Config
    stopChan chan bool
    doneChan chan bool
    errChan  chan error
    interval int
}

func IntervalProcessor(config Config, stopChan, doneChan chan bool, errChan chan error, interval int) Processor {
    return &intervalProcessor{config, stopChan, doneChan, errChan, interval}
}
```

`intervalProcessor.Process`方法的实现如下，通过定时器触发对Template Resources文件的处理（调用`process`方法）：
```GO
func (p *intervalProcessor) Process() {
    defer close(p.doneChan)
    for {
        ts, err := getTemplateResources(p.config)
        if err != nil {
            log.Fatal(err.Error())
            break
        }
        process(ts)
        select {
        case <-p.stopChan:
            break
        case <-time.After(time.Duration(p.interval) * time.Second):
            continue
        }
    }
}

func process(ts []*TemplateResource) error {
    var lastErr error
    for _, t := range ts {
		//调用每个TemplateResource的process方法
        if err := t.process(); err != nil {
            log.Error(err.Error())
            lastErr = err
        }
    }
    return lastErr
}
```

无论是否加watch参数，即`intervalProcessor`和`watchProcessor`最终都会调用到`TemplateResource.process`方法，如下：

3、`TemplateResource.process`方法<br>
`TemplateResourc.eprocess`方法的核心逻辑如下：

-	调用`setFileMode`方法，设置文件的权限，如果未指定mode参数则默认为`0644`，否则根据配置设置的mode来设置文件权限
-	调用`setVars`方法，将后端存储中最新的值拿出来暂存到内存中供后续进程使用（依赖于各个backend实现的`GetValues`方法，获取template文件`keys`对应的values值）
-	调用`createStageFile`方法，通过src的template文件和最新内存中的变量数据生成`StageFile`，该文件在`sync`方法中和目标文件进行比较，看是否有修改
-	调用`t.sync()`方法，该方法是执行了confd核心功能，即**将配置文件通过模板的方式自动生成，并执行检查check命令和reload命令**

```GOLANG
// process is a convenience function that wraps calls to the three main tasks
// required to keep local configuration files in sync. First we gather vars
// from the store, then we stage a candidate configuration file, and finally sync
// things up.
// It returns an error if any.
func (t *TemplateResource) process() error {
    if err := t.setFileMode(); err != nil {
        return err
    }
    if err := t.setVars(); err != nil {
        return err
    }
    if err := t.createStageFile(); err != nil {
        return err
    }
    if err := t.sync(); err != nil {
        return err
    }
    return nil
}
```


4、`sync`方法<br>
本[方法](https://github.com/kelseyhightower/confd/blob/master/resource/template/resource.go#L238)是confd最核心功能：
-	通过比较源文件和目标文件的差别，如果不同则重新生成新的配置（Overwriting）
-	当设置了`check_cmd`和`reload_cmd`的时候，会执行`check_cmd`的检查命令
-	如果没有问题则执行`reload_cmd`的命令

```GOLANG
func (t *TemplateResource) sync() error {
	//获取上一步生成的临时文件
	staged := t.StageFile.Name()
	if t.keepStageFile {
		log.Info("Keeping staged file: " + staged)
	} else {
		defer os.Remove(staged)
	}

	log.Debug("Comparing candidate config to " + t.Dest)

	//比较文件是否相等
	//https://github.com/kelseyhightower/confd/blob/master/util/util.go#L53
	ok, err := util.IsConfigChanged(staged, t.Dest)
	if err != nil {
		log.Error(err.Error())
	}
	if t.noop {
		log.Warning("Noop mode enabled. " + t.Dest + " will not be modified")
		return nil
	}
	if ok {
		//文件发生改变，触发更新
		log.Info("Target config " + t.Dest + " out of sync")
		//若配置了checkCmd指令，则执行check
		if !t.syncOnly && t.CheckCmd != "" {
			if err := t.check(); err != nil {
				return errors.New("Config check failed: " + err.Error())
			}
		}
		log.Debug("Overwriting target config " + t.Dest)
		//直接改文件名？暴力
		err := os.Rename(staged, t.Dest)
		if err != nil {
			if strings.Contains(err.Error(), "device or resource busy") {
				//如果出现了"device or resource busy"错误，则尝试重新写入文件
				log.Debug("Rename failed - target is likely a mount. Trying to write instead")
				// try to open the file and write to it
				var contents []byte
				var rerr error
				contents, rerr = ioutil.ReadFile(staged)
				if rerr != nil {
					return rerr
				}
				err := ioutil.WriteFile(t.Dest, contents, t.FileMode)
				// make sure owner and group match the temp file, in case the file was created with WriteFile
				os.Chown(t.Dest, t.Uid, t.Gid)
				if err != nil {
					return err
				}
			} else {
				return err
			}
		}
		//如果配置了reloadCmd，则运行reloadCMD
		if !t.syncOnly && t.ReloadCmd != "" {
			if err := t.reload(); err != nil {
				return err
			}
		}
		log.Info("Target config " + t.Dest + " has been updated")
	} else {
		log.Debug("Target config " + t.Dest + " in sync")
	}
	return nil
}
```

下面列举下此流程中的几个被调用的方法实现

4.1、IsConfigChanged 方法<br>
[`IsConfigChanged`方法](https://github.com/kelseyhightower/confd/blob/master/util/util.go#L53)，该方法用来比较源文件和目标文件是否相等，其中比较内容包括：Uid/Gid/Mode/Md5，其中任意值不同则认为两个文件不同。

```GOLANG
// IsConfigChanged reports whether src and dest config files are equal.
// Two config files are equal when they have the same file contents and
// Unix permissions. The owner, group, and mode must match.
// It return false in other cases.
func IsConfigChanged(src, dest string) (bool, error) {
	if !IsFileExist(dest) {
		return true, nil
	}
	d, err := FileStat(dest)
	if err != nil {
		return true, err
	}
	s, err := FileStat(src)
	if err != nil {
		return true, err
	}
	if d.Uid != s.Uid {
		log.Info(fmt.Sprintf("%s has UID %d should be %d", dest, d.Uid, s.Uid))
	}
	if d.Gid != s.Gid {
		log.Info(fmt.Sprintf("%s has GID %d should be %d", dest, d.Gid, s.Gid))
	}
	if d.Mode != s.Mode {
		log.Info(fmt.Sprintf("%s has mode %s should be %s", dest, os.FileMode(d.Mode), os.FileMode(s.Mode)))
	}
	if d.Md5 != s.Md5 {
		log.Info(fmt.Sprintf("%s has md5sum %s should be %s", dest, d.Md5, s.Md5))
	}
	if d.Uid != s.Uid || d.Gid != s.Gid || d.Mode != s.Mode || d.Md5 != s.Md5 {
		return true, nil
	}
	return false, nil
}
```

4.2、check方法<br>
`check`方法检查暂存的配置文件（`stageFile`），此文件由最新的后端存储中的数据生成，直接调用配置的指令运行检查，根据是否执行成功来返回报错。
此外，当`t.check()`命令返回错误时，则直接`return`错误，不再执行重新生成配置文件和`reload等后续操作

`check`指令会通过模板解析方式解析出`checkcmd`中的`{{.src}}`部分，并用`stageFile`来替代。即`check`的命令是拉取最新后端存储的数据形成临时配置文件（`stageFile`），并通过指定的`checkcmd`来检查最新的临时配置文件是否合法，如果合法则替换会新的配置文件，否则返回错误

```GOLANG
// check executes the check command to validate the staged config file. The
// command is modified so that any references to src template are substituted
// with a string representing the full path of the staged file. This allows the
// check to be run on the staged file before overwriting the destination config
// file.
// It returns nil if the check command returns 0 and there are no other errors.
func (t *TemplateResource) check() error {
	var cmdBuffer bytes.Buffer
	data := make(map[string]string)
	data["src"] = t.StageFile.Name()
	tmpl, err := template.New("checkcmd").Parse(t.CheckCmd)
	if err != nil {
		return err
	}
	if err := tmpl.Execute(&cmdBuffer, data); err != nil {
		return err
	}
	return runCommand(cmdBuffer.String())
}
```

4.3、reload方法<br>
```GOLANG
// reload executes the reload command.
// It returns nil if the reload command returns 0.
func (t *TemplateResource) reload() error {
	return runCommand(t.ReloadCmd)
}
```

4.4、runCommand方法<br>
```GOLANG
func runCommand(cmd string) error {
    log.Debug("Running " + cmd)
    var c *exec.Cmd
    if runtime.GOOS == "windows" {
        c = exec.Command("cmd", "/C", cmd)
    } else {
        c = exec.Command("/bin/sh", "-c", cmd)
    }

    output, err := c.CombinedOutput()
    if err != nil {
        log.Error(fmt.Sprintf("%q", string(output)))
        return err
    }
    log.Debug(fmt.Sprintf("%q", string(output)))
    return nil
}
```

4.5、`setVars`方法<br>
本方法是使用`storeClient`提供的`GetValues`方法，从远端获取完整的配置，以etcd和开头的配置为例，`GetValues`的参数是下面两个：
```text
[/yourapp/subdomain /yourapp/upstream]	#prefix为：/yourapp  keys为：[/subdomain /upstream]
[/myapp/subdomain /myapp/upstream]	#prefix为：/myapp keys为：[/subdomain /upstream]
```

```golang
// setVars sets the Vars for template resource.
func (t *TemplateResource) setVars() error {
	var err error
	log.Debug("Retrieving keys from store")
	log.Debug("Key prefix set to " + t.Prefix)

	result, err := t.storeClient.GetValues(util.AppendPrefix(t.Prefix, t.Keys))
	if err != nil {
		return err
	}
	log.Debug("Got the following map from store: %v", result)

	t.store.Purge()

	for k, v := range result {
		t.store.Set(path.Join("/", strings.TrimPrefix(k, t.Prefix)), v)
	}
	return nil
}
```

## 0x05：核心代码分析：Confd 的客户端封装

#### Confd 的客户端实现：EtcdV3 的客户端

confd的etcdv3的客户端封装，稍微有点绕，核心的逻辑如下：
![etcdv3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/confd/confd-goroutine-arch.png)

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

而 `Watch` 结构则记录，当前关联的 `key` 在 Etcd 中存储最新的 `revision` 值，由于 Etcd 本身是个 MVCC 存储，所以我们保存了 `revision`（这里的 `revision` 是最新的 revision），程序启动时，初始化为`0`（这个`0`值确保了confd初始化的时候，强制触发一次etcdv3的拉取操作，后述）：

```golang
// A watch only tells the latest revision
type Watch struct {
	// Last seen revision
	revision int64		//初始化为0
	// A channel to wait, will be closed after revision changes
	cond chan struct{}
	// Use RWMutex to protect cond variable
	rwl sync.RWMutex
}
```

#### etcdV3的WatchPrefix 方法
分析下在上一小节 `watchProcessor.monitorPrefix` 中调用的 `storeClient.WatchPrefix` 方法：

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
	//初始化时，lastRevision的值为0
	for {
		w.rwl.RLock()
		if w.revision > lastRevision {
			//初始化时，这里保证了，需要强制触发一次拉取etcdv3后端的流程
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

####	基于etcdv2实现
`WatchPrefix`[方法](https://github.com/kelseyhightower/confd/blob/master/backends/etcd/client.go#L124)：
```golang
func (c *Client) WatchPrefix(prefix string, keys []string, waitIndex uint64, stopChan chan bool) (uint64, error) {
	// return something > 0 to trigger a key retrieval from the store
	if waitIndex == 0 {
		//注意，初始化confd需要强制拉取etcd一次，这里直接返回1（初始化时waitIndex必然为0）
		return 1, nil
	}

	// Setting AfterIndex to 0 (default) means that the Watcher
	// should start watching for events starting at the current
	// index, whatever that may be.
	watcher := c.client.Watcher(prefix, &client.WatcherOptions{AfterIndex: uint64(0), Recursive: true})
	ctx, cancel := context.WithCancel(context.Background())
	cancelRoutine := make(chan bool)
	defer close(cancelRoutine)

	go func() {
		select {
		case <-stopChan:
			cancel()
		case <-cancelRoutine:
			return
		}
	}()

	for {
		resp, err := watcher.Next(ctx)
		if err != nil {
			switch e := err.(type) {
			case *client.Error:
				if e.Code == 401 {
					return 0, nil
				}
			}
			return waitIndex, err
		}

		// Only return if we have a key prefix we care about.
		// This is not an exact match on the key so there is a chance
		// we will still pickup on false positives. The net win here
		// is reducing the scope of keys that can trigger updates.
		for _, k := range keys {
			if strings.HasPrefix(resp.Node.Key, k) {
				return resp.Node.ModifiedIndex, err
			}
		}
	}
}
```

#### 基于 Redis 的实现

1、`redisClient.WatchPrefix`的实现<br>
`redisClient.WatchPrefix`是当用户设置了watch参数的时候，并且存储后端为redis，则会调用到redis的watch机制，代码如下：

redis的watch主要通过pub-sub（消费者订阅者）的机制，即`WatchPrefix`会根据传入的`prefix`启动一个`sub`的监听机制

```GOLANG
func (c *Client) WatchPrefix(prefix string, keys []string, waitIndex uint64, stopChan chan bool) (uint64, error) {

    if waitIndex == 0 {
        return 1, nil
    }

    if len(c.pscChan) > 0 {
        var respChan watchResponse
        for len(c.pscChan) > 0 {
            respChan = <-c.pscChan
        }
        return respChan.waitIndex, respChan.err
    }

    go func() {
        if c.psc.Conn == nil {
            rClient, db, err := tryConnect(c.machines, c.password, false);

            if err != nil {
                c.psc = redis.PubSubConn{Conn: nil}
                c.pscChan <- watchResponse{0, err}
                return
            }

            c.psc = redis.PubSubConn{Conn: rClient}        

            go func() {
                defer func() {
                    c.psc.Close()
                    c.psc = redis.PubSubConn{Conn: nil}
                }()
                for {
                    switch n := c.psc.Receive().(type) {
                        case redis.PMessage:
                            log.Debug(fmt.Sprintf("Redis Message: %s %s\n", n.Channel, n.Data))
                            data := string(n.Data)
                            commands := [12]string{"del", "append", "rename_from", "rename_to", "expire", "set", "incrby", "incrbyfloat", "hset", "hincrby", "hincrbyfloat", "hdel"}
                            for _, command := range commands {
                                if command == data {
                                    c.pscChan <- watchResponse{1, nil}
                                    break
                                }
                            }
                        case redis.Subscription:
                            log.Debug(fmt.Sprintf("Redis Subscription: %s %s %d\n", n.Kind, n.Channel, n.Count))
                            if n.Count == 0 {
                                c.pscChan <- watchResponse{0, nil}
                                return
                            }
                        case error:
                            log.Debug(fmt.Sprintf("Redis error: %v\n", n))
                            c.pscChan <- watchResponse{0, n}
                            return
                    }
                }
            }()

            c.psc.PSubscribe("__keyspace@" + strconv.Itoa(db) + "__:" + c.transform(prefix) + "*")
        }
    }()

    select {
    case <-stopChan:
        c.psc.PUnsubscribe()
        return waitIndex, nil
    case r := <- c.pscChan:
        return r.waitIndex, r.err
    }
}
```

####	基于consul实现


####	基于vault实现

## 0x06 总结
本文介绍了confd的用法、分析了其实现源码；再回顾一下，confd的作用是通过将配置存放到存储后端，来自动触发更新配置的功能，支持多种后端存储。confd的实现思路非常清晰，代码风格也很值得借鉴

不同的存储后端的watch机制实现大相径庭，如Etcd只需要更新key即可触发watch，而redis除了更新key还需要执行publish的操作。

confd提供了checkcmd和reloadcmd来实现对配置准确性的检查，降低出错的概率；通过配置check_cmd来校验配置文件是否正确，如果配置文件非法则不会执行自动更新配置和reload的操作

缺点是，需要有额外的手段来确保控制存入存储后端的数据始终是合法的。

####	confd的可以优化的点

一般在项目应用中，将confd作为配置中心的组件，实现有两种方式：
1.	基于变量为粒度来维护配置，在代码中用`xxxx_application.yml.tmpl`之类的配置文件描述应用的配置文件，占位符表述的值会被存储到像consul、etcd等kv存储中，并提供可CRUD维护配置变量的管理端。每次发布会拉取最新的key值进行替换占位符得到应用程序使用的配置文件`xxxx_application.yml.tmpl`

2.	以整个配置文件为粒度来维护配置，开发需要将配置文件的整个内容发布到配置中心，kv存储存储的是配置文件整个内容作为一个value，更新配置文件可以用value完成更新


####	confd应用场景
1.	nginx动态生成upstream实现服务发现
2.	prometheus动态生成prometheus.yml实现自动报警


####	confd的可借鉴之处
1.	各类客户端的典型封装，算是比较全面了，项目中可以借鉴

## 0x07 参考
- 	[Quick Start Guide](https://github.com/kelseyhightower/confd/blob/master/docs/quick-start-guide.md)
-	[confd的安装与使用](https://www.huweihuang.com/linux-notes/tools/confd-usage.html)
-	[confd模板语法](https://blog.kelu.org/tech/2021/10/20/confd-templates.html)
