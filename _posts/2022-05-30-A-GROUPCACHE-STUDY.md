---
layout: post
title: Golang 的分布式缓存库：GroupCache 分析
subtitle: 分析一个 GroupCache 的应用
date: 2022-05-30
author: pandaychen
catalog: true
tags:
    - GroupCache
    - 缓存
---

##  0x00    前言
[groupcache](https://github.com/golang/groupcache) 是一个分布式缓存库，支持多节点互备热数据，有良好的稳定性和较高的并发性。测试用例，可以参考此文章：[Playing with groupcache](https://capotej.com/blog/2013/07/28/playing-with-groupcache/)；此外，还可以参考作者的幻灯片：[dl.google.com: Powered by Go](https://go.dev/talks/2013/oscon-dl.slide#63)

先前讨论缓存更新时，分布式场景下需要解决 ** 缓存更新导致的缓存一致性问题 **，而 GroupCache 仅支持外部 get，不支持外部 set/update/delete（当然有内部 Set 操作），自然就没有更新导致的缓存一致性问题。由于 GroupCache 只能 get，不能 update 和 delete，也无法设置过期时间，只能通过 lru 算法淘汰最近最少访问的数据；因而 Groupcache 比较适合那些数据长时间不变更的缓存。典型的应用场景是为下载服务提供静态资源缓存


##  0x01    groupcache-db-experiment 的应用
goupcache 仅仅是缓存库，业务逻辑需要自己构建。[文章](https://capotej.com/blog/2013/07/28/playing-with-groupcache/) 给出的部署架构如下：

![bushu](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/groupcache/app-1.png)

如图，客户端使用 `set/get` 指令时，直接操作存取的是数据源（数据库 DB Server）；使用 `cget` 指令，则是从 groupcache 中查找数据。该 demo 封装了 groupcache，有如下组件：
-   Db-server：模拟本地存储
-   front-Server（内置了 groupCache Server）：监听了两个 tcp 端口，一个用于和客户端通信，另一个用于和 groupcache peer 进行通信
-   Client 客户端：模拟客户端请求

其中，客户端和 groupcache 通过 rpc（`net/rpc`）进行通信，而 groupcache 的各个 peer 之间通过 http 协议进行通信。set/get 是直接针对数据存储服务器 DB Server 的存取，cget 是通过一个前端 frontend 的 rpc 接口间接访问 groupcache-server 来获取数据，** 假如 groupcache server 中没有数据，那么就进一步访问数据存储服务器来获取数据，将数据存储在缓存中，并返还给客户端 **。上述的逻辑如下 [代码](https://github.com/capotej/groupcache-db-experiment/blob/master/frontend/frontend.go#L22)：

```GO
func (s *Frontend) Get(args *api.Load, reply *api.ValueResult) error {
	var data []byte
	fmt.Printf("cli asked for %s from groupcache\n", args.Key)
    // 从 groupcache 中获取缓存数据
	err := s.cacheGroup.Get(nil, args.Key,
		groupcache.AllocatingByteSliceSink(&data))

	reply.Value = string(data)
	return err
}
```


####    模拟运行
```bash
./cli -set -key foo -value bar      ## 模拟写入本地存储
./cli -get -key foo should see bar in 300 ms        ## 模拟从本地存储读数据
./cli -cget -key foo should see bar in 300ms (cache is loaded)      ##
./cli -cget -key foo should see bar instantly       ## 从 groupcache 读取并返回
```


####    组件分析
```TEXT
[root@VM_120_245_centos ~/tw/groupcache-db-experiment]# tree -a
.
├── api
│   └── api.go      #存储数据的基本格式
├── build.sh
├── cli
│   ├── cli
│   └── cli.go      #客户端实现的多种方式存取数据，指令实现：cget,get,set
├── client
│   └── client.go   #直接从数据存储服务器（dbserver）中的存取数据实现
├── dbserver
│   ├── dbserver
│   ├── dbserver.go #RPC 服务器，实现了针对数据存取服务器的操作，提供 rpc 接口调用
│   └── .DS_Store
├── frontend
│   ├── frontend
│   └── frontend.go #RPC 服务器，客户端发起 CGET 查询请求，首先到 frontend 端，frontend 查询占据 8001 端口的 groupcache 服务器获取查询值，如果没有，访问数据存取服务器 dbserver 来填充 groupcache，然后再返回给客户端
├── go.mod
├── go.sum
└── slowdb
    └── slowdb.go   #服务器端的具体存取数据，模拟慢 db
```


####    cget 流程
CGET 的调用路径如下：

![CGET]()

```GOLANG
func main(){
    //...
    if *cget {
            // 调用 Frontend.Get 方法获取缓存
            client, err := rpc.DialHTTP("tcp", "localhost:"+*port)
            if err != nil {
                fmt.Printf("error %s", err)
            }
            args := &api.Load{*key}
            var reply api.ValueResult
            err = client.Call("Frontend.Get", args, &reply)
            if err != nil {
                fmt.Printf("error %s", err)
            }
            fmt.Println(string(reply.Value))
            return
    }
}

func (s *Frontend) Get(args *api.Load, reply *api.ValueResult) error {
	var data []byte
	fmt.Printf("cli asked for %s from groupcache\n", args.Key)

    // 调用 groupcache 的 Get 接口获取
	err := s.cacheGroup.Get(nil, args.Key,
		groupcache.AllocatingByteSliceSink(&data))

	reply.Value = string(data)
	return err
}
```


####    调用 groupcache 的代码分析
整个封装都在 [`frontend.go`](https://github.com/capotej/groupcache-db-experiment/blob/master/frontend/frontend.go) 中，初始化的代码如下：

```GOLANG
type Frontend struct {
	cacheGroup *groupcache.Group
}

func main() {

	var port = flag.String("port", "8001", "groupcache port")
	flag.Parse()

    // 构建 NewHTTPPool
	peers := groupcache.NewHTTPPool("http://localhost:" + *port)

	client := new(client.Client)

	var stringcache = groupcache.NewGroup("SlowDBCache", 64<<20, groupcache.GetterFunc(
		func(ctx groupcache.Context, key string, dest groupcache.Sink) error {
			result := client.Get(key)
			fmt.Printf("asking for %s from dbserver\n", key)
			dest.SetBytes([]byte(result))
			return nil
		}))

	peers.Set("http://localhost:8001", "http://localhost:8002", "http://localhost:8003")

	frontendServer := NewServer(stringcache)

	i, err := strconv.Atoi(*port)
	if err != nil {
		// handle error
		fmt.Println(err)
		os.Exit(2)
	}
	var frontEndport = ":" + strconv.Itoa(i+1000)
	go frontendServer.Start(frontEndport)

	fmt.Println(stringcache)
	fmt.Println("cachegroup slave starting on" + *port)
	fmt.Println("frontend starting on" + frontEndport)
	http.ListenAndServe("127.0.0.1:"+*port, http.HandlerFunc(peers.ServeHTTP))

}

func NewServer(cacheGroup *groupcache.Group) *Frontend {
	server := new(Frontend)
	server.cacheGroup = cacheGroup
	return server
}
```

```GOLANG
func (s *Frontend) Get(args *api.Load, reply *api.ValueResult) error {
	var data []byte
	fmt.Printf("cli asked for %s from groupcache\n", args.Key)
	err := s.cacheGroup.Get(nil, args.Key,
		groupcache.AllocatingByteSliceSink(&data))

	reply.Value = string(data)
	return err
}
```

##  0x02    GroupCache 库：介绍
前面已经介绍了 groupcache 是个 kv 存储库，最大的特点是没有删除接口；kv 一旦设置，那么用户端是无法更新和删除（没有接口）已经缓存的内容。此库的优点如下：
-   client & server
-   缓存一致性：consistent
-   防击穿：singleflight
-   group 的机制
-   协商填充
-   预热机制
-   热点数据多节点备份
-   LRU


####   基础数据结构及实现

1、byteView<br>
[`byteview`](https://github.com/golang/groupcache/blob/master/byteview.go) 对字符串数组和 `string` 进行封装，存储何种形式的字符串对用户透明
```GOLANG
type struct {
     b []byte
     s string
}

func at(i){
     if b!=nil
         return b[i]
     return s[i]
}
```

2、[consistenthash](https://github.com/golang/groupcache/blob/master/consistenthash/consistenthash.go)：一致性 hash<br>
groupcache 中，consistenthash 将 key 按照一致性 hash 分布在各个实例中，主要作用是实例（groupcache 节点）加入离开时不会导致映射关系发生重大变化
```GOLANG
type Hash func(data []byte) uint32

type Map struct {
	hash     Hash
	replicas int
	keys     []int // Sorted
	hashMap  map[int]string
}
```

3、[LRU 算法](https://github.com/golang/groupcache/blob/master/lru/lru.go)<br>
LRU 的作用是，groupcache 没有更新和删除接口，提供了避免空间只增不减的缓存淘汰机制

4、singleflight 机制 <br>
singleflight 保证同时多个并发请求下，多个同参数的 get 请求只执行一次操作功能

5、group 分组 <br>
group 是 key-value 的容器，每个 group 都有一个名字，相当于一批 key-value 的家族标识。当用户访问某个 key 时，group 现在本地内存中维护的 hashmap 中查找，如果没有则从远端的 peerNode 或者本地的文件系统（DBserver）处进行加载。group 类似于命名空间的设计，不同的命名空间是相互隔离的。

groupcache支持多个group：`groups = make(map[string]*Group)`

```GOLANG
// A Group is a cache namespace and associated data loaded spread over
// a group of 1 or more machines.
type Group struct {
	name       string
	getter     Getter
	peersOnce  sync.Once
	peers      PeerPicker
	cacheBytes int64 // limit for sum of mainCache and hotCache size

	// mainCache is a cache of the keys for which this process
	// (amongst its peers) is authoritative. That is, this cache
	// contains keys which consistent hash on to this process's
	// peer number.
	mainCache cache

	// hotCache contains keys/values for which this peer is not
	// authoritative (otherwise they would be in mainCache), but
	// are popular enough to warrant mirroring in this process to
	// avoid going over the network to fetch from a peer.  Having
	// a hotCache avoids network hotspotting, where a peer's
	// network card could become the bottleneck on a popular key.
	// This cache is used sparingly to maximize the total number
	// of key/value pairs that can be stored globally.
	hotCache cache

	// loadGroup ensures that each key is only fetched once
	// (either locally or remotely), regardless of the number of
	// concurrent callers.
	loadGroup flightGroup

	_ int32 // force Stats to be 8-byte aligned on 32-bit platforms

	// Stats are statistics on the group.
	Stats Stats
}
```

6、[httppool](https://github.com/golang/groupcache/blob/master/http.go)<br>
httppool 是 groupcache 中各个 peerNode 通讯的封装，开启通讯 http，group 的 `getFromPeer` 便调用相对应 peer 的 httpool 提供的服务。httppool 同时保存了所有的远端 peerNode 实例的请求地址，实现 `pickPeer` 安装一致性 hash 取得某 key 对应的远端 peerNode 实例的地址。

7、[PeerPicker](https://github.com/golang/groupcache/blob/master/peers.go)<br>

####	groupcache：架构总览

1、API接口<br>
groupcache 的对外接口，即[`Group`](https://github.com/golang/groupcache/blob/master/groupcache.go#L142)的`3`个方法：
-	[`Get`](https://github.com/golang/groupcache/blob/master/groupcache.go#L208)：用户侧，传入一个 key，获取 value
-	`Name`方法：返回group的名字（允许多个group）
-	`CacheStats`方法：统计数据，比如命中率

2、client && server<br>
Groupcache 库既是服务器，也是客户端，当在本地 groupcache 缓存中没有查找的数据时，通过一致性哈希，查找到该 key 所对应的 peer 服务器，再通过 http 协议，从该 peer 服务器上获取所需要的数据；还有一点就是当多个客户端同时访问 groupcache 中不存在的键时，groupcache 只会有一个客户端从 mysql 获取数据，其他客户端阻塞，直到第一个客户端获取到数据之后，再返回给多个客户端

groupcache 实现的是**一个分布式的无中心化的缓存集群**。每个节点，既作为 server ，对外提供 API 访问，返回 key 的数据；节点（peerNode）之间，也是可以C/S 模式互相访问


####	groupCache的缓存一致性


##	0x03	groupCache源码走读


####	Group及其成员
`Group`会缓存自己节点的数据和访问比较频繁的 peer节点 的数据，用LRU算法控制cache容量，当 cache 未命中时，首先看看这个请求归不归该节点管（后面讨论），若是就是调用 `getter`：
```GOLANG
type Group struct {
    name       string
    getter     Getter // cache 没有命中，从数据库获取的通用回调方法，需要开发者自行实现！
    peersOnce  sync.Once 
    peers      PeerPicker // peer 节点调度器
    cacheBytes int64 // 最大cache字节数
    mainCache cache // 此节点缓存
    hotCache cache // 其他节点缓存
    loadGroup flightGroup // 请求并发控制器
    Stats Stats // 统计数据
}
```

`Getter`定义了一个Cache如何从数据库中获取到相应的数据，比如是通过网络还是本地IO等方法，通常由开发者传入这个回调实现，比如前文例子中的[`stringcache`](https://github.com/capotej/groupcache-db-experiment/blob/master/frontend/frontend.go#L60)；`ctx Context` 是操作的附带信息，`key` 请求的 id，`Sink`抽象了数据载体的行为（类型）。groupcache 提供了一些常用的 Sink 如 `StringSink`，`BytesSliceSink` 和 `ProtoSink`等
```GO
type Getter interface {
    Get(ctx Context, key string, dest Sink) error
}

type Sink interface {
    // SetString 写入 string
    SetString(s string) error

    // SetBytes 写入字节数组，调用者会保留 v 引用
    SetBytes(v []byte) error

    // SetProto 写入proto.Message，调用者会保留 m 应用
    SetProto(m proto.Message) error
}
```

成员`PeerPicker`的定义如下，该结构是对节点调度器抽象。调度器主要负责根据管理 key 和节点的一致性映射。groupcache 实现了一个基于HTTP 的 `PeerPicker`，即`HTTPPool`结构，至此，groupcache 通过 `Getter`、`PeerPicker`和`ProtoGetter`这`3`个通用结构定义了cache，节点和调度器之间的连接方式，可以有效地控制耦合度，提供了比较大的灵活性。

```go
// PeerPicker is the interface that must be implemented to locate
// the peer that owns a specific key.
type PeerPicker interface {
	// PickPeer returns the peer that owns the specific key
	// and true to indicate that a remote peer was nominated.
	// It returns nil, false if the key owner is the current peer.
	// PickPeer 根据 key 返回应该处理这个 key 的节点
    // ok 为 true 代表找到了节点
    // nil, false 代表当前节点就是 key 的处理器
	PickPeer(key string) (peer ProtoGetter, ok bool)
}
```
最后，看下`ProtoGetter`结构。groupcache 规定内部 peerNode 节点间数据通信格式使用 [google/protobuf](github.com/golang/protobuf/proto)，为了抽象 peer 节点，定义了 `ProtoGetter`。groupcache提供了`httpGetter`的[实现](https://github.com/golang/groupcache/blob/master/http.go)
```GO
//pb.GetRequest 和 pb.GetResponse 定义了请求和响应 struct，这个抽象可以分离底层传输方式
type ProtoGetter interface {
    Get(context Context, in *pb.GetRequest, out *pb.GetResponse) error
}
```

`httpGetter`的实例化实现，`Get`方法：
```golang
var bufferPool = sync.Pool{
	New: func() interface{} { return new(bytes.Buffer) },
}

func (h *httpGetter) Get(ctx context.Context, in *pb.GetRequest, out *pb.GetResponse) error {
	u := fmt.Sprintf(
		"%v%v/%v",
		h.baseURL,
		url.QueryEscape(in.GetGroup()),
		url.QueryEscape(in.GetKey()),
	)
	req, err := http.NewRequest("GET", u, nil)
	if err != nil {
		return err
	}
	req = req.WithContext(ctx)
	tr := http.DefaultTransport
	if h.transport != nil {
		tr = h.transport(ctx)
	}
	res, err := tr.RoundTrip(req)
	if err != nil {
		return err
	}
	defer res.Body.Close()
	if res.StatusCode != http.StatusOK {
		return fmt.Errorf("server returned: %v", res.Status)
	}

	//optimistic
	b := bufferPool.Get().(*bytes.Buffer)
	b.Reset()
	defer bufferPool.Put(b)
	_, err = io.Copy(b, res.Body)
	if err != nil {
		return fmt.Errorf("reading response body: %v", err)
	}
	err = proto.Unmarshal(b.Bytes(), out)
	if err != nil {
		return fmt.Errorf("decoding response body: %v", err)
	}
	return nil
}
```


##  0x04    groupcache VS memcache
Groupcache 库既是服务器，也是客户端，当在本地 groupcache 缓存中没有查找的数据时，通过一致性哈希，查找到该 key 所对应的 peer 服务器，再通过 http 协议，从该 peer 服务器上获取所需要的数据；还有一点就是当多个客户端同时访问 memcache 中不存在的键时，会导致多个客户端从 mysql 获取数据并同时插入 memcache 中，而在相同情况下，groupcache 只会有一个客户端从 mysql 获取数据，其他客户端阻塞，直到第一个客户端获取到数据之后，再返回给多个客户端


##  0x05  参考
-   [dl.google.com: Powered by Go](https://go.dev/talks/2013/oscon-dl.slide#1)
-   [groupcache 架构设计](https://www.jianshu.com/p/f69f3a3a9a78)
-   [Playing with groupcache](https://capotej.com/blog/2013/07/28/playing-with-groupcache/)
-   [groupcache 一个没有删除的缓存](https://liqingqiya.github.io/groupcache/golang/%E7%BC%93%E5%AD%98/2020/05/10/groupcache.html)