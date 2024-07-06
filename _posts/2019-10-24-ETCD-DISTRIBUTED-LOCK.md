---
layout:     post
title:      Etcd 应用开发之分布式锁
subtitle:   Etcd 应用开发（续）
date:       2019-10-24
author:     pandaychen
catalog:    true
tags:
    - 分布式锁
    - Etcd
---

##  0x00	分布式锁基础
在分布式系统中，为了实现对互斥资源的安全访问（独占），必须要用到分布式锁。另外在工作中，还有一个应用场景是，在分布式的后台服务中，某些服务只允许单个实例（机器或 Docker）运行，其他的备份机器作为 BackUps 备份节点（比如在笔者的项目中，负责数据同步的逻辑同一时刻只能有一台机器进程来执行），当正在运行的单实例机器（进程）故障后，在线的 BackUps 按照排队顺序，争抢到分布式锁，继续执行，这样也保证了单实例场景的高可用性。

分布式锁应具备以下特点：
-  互斥性：任意时刻，同一个锁，只有一个进程能持有
-  安全性：避免死锁，当进程没有主动释放锁（进程崩溃退出），保证其他进程能够加锁
-  可用性：当提供锁的服务节点故障（宕机）时，热备节点能够接替故障的节点继续提供服务，并保证自身持有的数据与故障节点一致
-  对称性：对同一个锁，加锁和解锁必须是同一个进程，即某进程不能把其他进程持有的锁给释放了

可以 <font color="#dd0000"> 基于数据库 / 缓存 / 中间件等实现分布式锁，当下比较主流的是使用 Redis、Etcd 或 ZooKeeper 等开源项目来构建 </font>。

##  0x01	分布式锁的两种形态
通常，分布式锁有如下两种形态：
-  短期锁：即用即申请，用完即释放，一般用于资源控制粒度比较细的系统，这种场景会频繁的调用 Etcd 服务
-  长期锁：这种锁是先到先得，得到即长期占有，更多是用在主备系统的切换场景，如果占有锁的服务不发生异常，则不会主动与 Etcd 交互，对于笔者的数据同步服务是一种长期锁

这里多提一句，无论是短期还是长期占有的锁，在锁竞争上最好都遵从公平的策略，即先注册（排队）的进程（实例）先获取锁。

##	0x02	分布式锁的实现方案
要实现分布式锁，核心在两点，一是如何达成共识、二是状态同步。目前的实现方式一般会借助于 Redis/ZooKeeper/Etcd 来实现：
1.	采用 Redis 的 `SETNX` 命令 + Lua 原子脚本来实现（单 Redis 实例），存在单点的风险
2.	对 `1` 的改进版是采用 Redis Master 集群的 [Redlock](http://stor.51cto.com/art/201901/590874.htm) 改进算法
3.	[基于 ZooKeeper 实现的分布式锁](https://juejin.im/post/5c01532ef265da61362232ed)
4.	基于 Etcd 实现的分布式锁：[分布式锁的最佳实践之：基于 Etcd 的分布式锁](https://gitbook.cn/books/5bb037728f7d8b7e900ff2d7/index.html)

此外，下面这篇文章值得一读：
[基于 Redis 的分布式锁到底安全吗](http://zhangtielei.com/posts/blog-redlock-reasoning.html)

本文主要介绍如何使用 Etcd 来实现分布式锁，基于 Golang 实现。

##	0x03	Etcd 实现分布式锁
这些年，随着 [Raft 协议](https://raft.github.io/) 的广泛应用，涌现出很多的基于此协议实现的高可用分布式系统，比较知名的有 [Etcd](https://github.com/etcd-io/etcd)、[Consul](https://github.com/hashicorp/consul) 等，由于其内部实现了 [分布式一致性](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86)，所以非常适合做分布式锁。本片文章就以 Etcd 为例，来介绍下构建分布式锁的方法。值得注意的是，Etcd 分布式锁的实现是在客户端做的。

Etcd 支持以下功能，正是依赖这些功能来实现分布式锁的：
-	Lease 机制：即租约机制（TTL，Time To Live），Etcd 可以为存储的 Key-Value 对设置租约，当租约到期，Key-Value 将失效删除；同时也支持续约续期（KeepAlive）
-	Revision 机制：每个 key 带有一个 `Revision` 属性值，Etcd 每进行一次事务对应的全局 `Revision` 值都会加一，因此每个 Key 对应的 `Revision` 属性值都是全局唯一的。通过比较 `Revision` 的大小就可以知道进行写操作的顺序。在实现分布式锁时，多个程序同时抢锁，根据 `Revision` 值大小依次获得锁，可以避免惊群效应，实现公平锁
-	Prefix 机制：即前缀机制（或目录机制）。可以根据前缀（目录）获取该目录下所有的 Key 及对应的属性（包括 Key、Value 以及 `Revision` 等）
-	Watch 机制：即监听机制，Watch 机制支持 Watch 某个固定的 Key，也支持 Watch 一个目录前缀（前缀机制），当被 Watch 的 Key 或目录发生变化，客户端将收到通知

####	Lease 机制
从结构体 `KeyValue` 出发，每一个 `KeyValue` 都包含了一个 `Lease` 成员
```golang
type KeyValue struct {
    // key is the key in bytes. An empty key is not allowed.
    Key []byte `protobuf:"bytes,1,opt,name=key,proto3" json:"key,omitempty"`
    // create_revision is the revision of last creation on this key.
    CreateRevision int64 `protobuf:"varint,2,opt,name=create_revision,json=createRevision,proto3" json:"create_revision,omitempty"`
    // mod_revision is the revision of last modification on this key.
    ModRevision int64 `protobuf:"varint,3,opt,name=mod_revision,json=modRevision,proto3" json:"mod_revision,omitempty"`
    // version is the version of the key. A deletion resets
    // the version to zero and any modification of the key
    // increases its version.
    Version int64 `protobuf:"varint,4,opt,name=version,proto3" json:"version,omitempty"`
    // value is the value held by the key, in bytes.
    Value []byte `protobuf:"bytes,5,opt,name=value,proto3" json:"value,omitempty"`
    // lease is the ID of the lease that attached to key.
    // When the attached lease expires, the key will be deleted.
    // If lease is 0, then no lease is attached to the key.
    Lease int64 `protobuf:"varint,6,opt,name=lease,proto3" json:"lease,omitempty"`
}
```

####	Revision 机制
[Revision](https://github.com/etcd-io/etcd/blob/v3.3.10/mvcc/revision.go) 是全局的版本概念，不针对某一具体的 Key，`Revision` 即对应到 MVCC（Multi-Version Concurrency Control）中的版本，Etcd 中的每一次 Key-Value 的修改操作都会有一个相应的 `Revision`。`revision` 的组成如下：
```golang
// A revision indicates modification of the key-value space.
// The set of changes that share same main revision changes the key-value space atomically.
type revision struct {
    // main is the main revision of a set of changes that happen atomically.
    main int64

    // sub is the the sub revision of a change in a set of changes that happen
    // atomically. Each change has different increasing sub revision in that
    // set.
    sub int64
}
```
这里的 `main` 属性对应事务 ID，<font color="#dd0000"> 全局递增不重复 </font>，在 Etcd 中被当做一个逻辑时钟来使用。而 `sub` 代表 Etcd 一次事务中不同的修改操作（如 `Put` 和 `Delete`）编号，从 `0` 开始依次递增。即在一次事务中，每一个修改操作所绑定的 `Revision` 依次为 `{txID, 0}`, `{txID, 1}`, `{txID, 2}` 依次递增。

####	Prefix && Watch 机制

##  0x04	分析 Concurrency 包
EtcdV3 版本的 [Concurrency 包](https://godoc.org/github.com/coreos/etcd/clientv3/concurrency) 提供了实现 [分布式锁](https://github.com/coreos/etcd/blob/master/clientv3/concurrency/mutex.go) 及 [分布式选主](https://github.com/coreos/etcd/blob/master/clientv3/concurrency/election.go#L31) 的接口，这两个概念间是有共性的，譬如，谁抢到了锁谁就是 Master。

下面从 Etcd 提供的 [Lock 方法](https://github.com/etcd-io/etcd/blob/master/etcdctl/ctlv3/command/lock_command.go) 来入手，看看 `command.lockUntilSignal` 操作是如何实现的：
```golang
func lockUntilSignal(c *clientv3.Client, lockname string, cmdArgs []string) error {
	// 通过 concurrency 的 NewMutex 初始化
    m := concurrency.NewMutex(s, lockname)
    ctx, cancel := context.WithCancel(context.TODO())
    //...
    if err := m.Lock(ctx); err != nil {	// 加锁，Etcd 的分布式锁的主要实现
		return err
    }
    //...
}
```

先贴一下锁 `Mutex` 部分的 Code，[Mutex](https://github.com/coreos/etcd/blob/master/clientv3/concurrency/mutex.go#L26) 的结构如下：
```golang
// Mutex implements the sync Locker interface with etcd
type Mutex struct {
	s *Session
	pfx   string   // 锁的共同前缀 pfx，如 "/service/lock/"
	myKey string   // 当前持有锁的客户端的 leaseid 值（完整 Key 的组成为 pfx+"/"+leaseid）
	myRev int64    //revision，理解为当前持有锁的 Revision 编号
	hdr   *pb.ResponseHeader
}
```

初始化 `Mutex`，可以看到在 `NewMutex` 方法中，并非直接拿传进来的 `pfx` 作为 Prefix 的，而且在后面加了个 `/`，在 Etcd 开发项目中，定义或查找 Prefix 或 Suffix 时都建议加上分隔符 `/`，这是个好习惯，也可以避免出现问题。
```golang
func NewMutex(s *Session, pfx string) *Mutex {
   //Etcd 这里默认将 / path 的 key 结构转换为一个目录结构 /path/
   return &Mutex{s, pfx + "/", "", -1, nil}
}
```

接着看下最基础的加锁方法 `TryAcquire`，该方法是通过 Etcd 的 `TXN`（多键条件事务）原子方式来实现的，EtcdV3 的 `TXN` 的语意是 <font color="#dd0000"> 先做一个比较（Compare）操作，如果比较成立则执行一系列操作，如果比较不成立则执行另外一系列操作 </font>。类似于 C 语言中的条件表达式。如下：

```golang
//Txn 事务，以原子方式判定 cmp 的条件，如果为 True，就设置 getOwner 的值；
// 否则 False，返回 getOwner 的值；
// 最终用 Commit() 提交
resp, err := client.Txn(ctx).If(cmp).Then(put, getOwner).Else(get, getOwner).Commit()
```

然后看下 `TryAcquire` 方法：
```golang
func (m *Mutex) tryAcquire(ctx context.Context) (*v3.TxnResponse, error) {
	s := m.s
	client := m.s.Client()
	// 下面的 m.pfx 就是 prefix，是传进来的前缀，后面的 s.Lease() 会返回一个租约，是一个 int64 的整数，和 session 有关系
	//mykex 先理解为是 / prefix/leaseid 这样的结构
	m.myKey = fmt.Sprintf("%s%x", m.pfx, s.Lease())

	/* 这里定义一个 cmp 方法，比较上面 m.myKey 的 CreateRevision 是否为 0
	等于 0 表示目前不存在该 key，需要执行 Put 操作
	不等于 0 表示对应的 key 已经创建了，需要执行 Get 操作 */
	cmp := v3.Compare(v3.CreateRevision(m.myKey), "=", 0)
	// put self in lock waiters via myKey; oldest waiter holds lock
	put := v3.OpPut(m.myKey, "", v3.WithLease(s.Lease()))
	// reuse key in case this session already holds the lock
	get := v3.OpGet(m.myKey)
	// fetch current holder to complete uncontended path with only one RPC
    // 获取当前锁的真正持有者
	getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)
	//Txn 事务，判断 cmp 的条件是否成立，成立执行 Then，不成立执行 Else，最终执行 Commit()
	resp, err := client.Txn(ctx).If(cmp).Then(put/*resp.Responses[0]*/, getOwner/*resp.Responses[1]*/).Else(get, getOwner).Commit()
	if err != nil {
		return nil, err
	}
	//succ
	m.myRev = resp.Header.Revision
	if !resp.Succeeded {
		// 失败，执行 Else(get,getOwner)
		m.myRev = resp.Responses[0].GetResponseRange().Kvs[0].CreateRevision
	}
	return resp, nil
}
```

####	v3.WithFirstCreate

在 `tryAcquire` 的 TXN 事务中，不管是执行 `Put` 还是 `Get`，最终都有个 `getOwner` 的过程，看一下这个 `getOwner` 的逻辑，这里有个有趣的选项，`getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)` 中的 `options` 模式里有个 `v3.WithFirstCreate()` 函数调用，这是一个和 prefix 相关的函数，看看它的实现：

```golang
// WithFirstCreate gets the key with the oldest creation revision in the request range.
func WithFirstCreate() []OpOption { return withTop(SortByCreateRevision, SortAscend) }

// withTop gets the first key over the get's prefix given a sort order
func withTop(target SortTarget, order SortOrder) []OpOption {
   return []OpOption{WithPrefix(), WithSort(target, order), WithLimit(1)}
}

// WithPrefix enables 'Get', 'Delete', or 'Watch' requests to operate
// on the keys with matching prefix. For example, 'Get(foo, WithPrefix())'
// can return 'foo1', 'foo2', and so on.
func WithPrefix() OpOption {
	// 返回所有满足 prefix 匹配的 key-value，和 etcdctl get key --prefix 功能一样
   return func(op *Op) {
      if len(op.key) == 0 {
         op.key, op.end = []byte{0}, []byte{0}
         return
      }
      op.end = getPrefix(op.key)
   }
}
```

看到上面的代码，`WithPrefix/WithSort`，所以 `getOwner` 的具体执行效果是会把虽有以 `lockkey` 开头的 Key-Value 都拿到，且按照 `CreateRevision` 升序排列，并取第一个值，这个意思就很明白了，就是要拿到当前以 `lockkey` 为 Prefix 的且 `CreatereVision` 最小的那个 Key，就是目前已经拿到锁的 Key;

`TryLock/Lock` 上层调用：
```golang
// Lock locks the mutex with a cancelable context. If the context is canceled
// while trying to acquire the lock, the mutex tries to clean its stale lock entry.
func (m *Mutex) Lock(ctx context.Context) error {
	// 先调用 tryAcquire 尝试获取锁
	resp, err := m.tryAcquire(ctx)
	if err != nil {
		return err
	}
	// if no key on prefix / the minimum rev is key, already hold the lock
	ownerKey := resp.Responses[1].GetResponseRange().Kvs
	// 比较如果当前没有人获得锁（第一次场景）
	// 或者锁的 owner 的 CreateRevision 等于当前的 kv 的 Revision，则表示已成功获得锁，就可以退出了
	if len(ownerKey) == 0 || ownerKey[0].CreateRevision == m.myRev {
		// 拿当前进程的 Revision 和 Owner 的 CreateVision 比较，如果相等，就是拿到了锁
		m.hdr = resp.Header
		return nil
	}
	client := m.s.Client()
	// wait for deletion revisions prior to myKey
	// 走到这里代表没有获得锁，需要等待之前的锁被释放，即 revision 小于当前 revision 的 kv 被删除
	hdr, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
	// 阻塞等待 Owner 释放锁，下面分析 waitDeletes
	// release lock key if wait failed
	if werr != nil {
		m.Unlock(client.Ctx())
	} else {
		// 成功获取锁
		m.hdr = hdr
	}
	return werr
}
```

再看看 `waitDeletes` 函数的行为, `waitDeletes` <font color="#dd0000"> 模拟了一种公平的先来后到的排队逻辑，循环等待所有当前比当前 Key 的 </font> `Revision` <font color="#dd0000"> 值小的 Key 被删除后，锁释放后才返回 </font>，注意上面的关键字：循环等待。此处正是利用了 `Revision` 是单调递增的特性来实现。
```golang
// 注意 waitDeletes 的调用方式：hdr, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
func waitDeletes(ctx context.Context, client *v3.Client, pfx string, maxCreateRev int64) (*pb.ResponseHeader, error) {
   // 注意 WithLastCreate 和 WithMaxCreateRev 这两个特性
   getOpts := append(v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev))
   for {
      // 循环
      resp, err := client.Get(ctx, pfx, getOpts...)
      if err != nil {
         return nil, err
      }
      if len(resp.Kvs) == 0 {
         // 表示在 maxCreateRev 前面已经没有比本进程优先级更高的客户端了，本客户端可以获得锁
         return resp.Header, nil
      }
      lastKey := string(resp.Kvs[0].Key)
      // 阻塞，waitDelete 等待 lastKey+resp.Header.Revision 被成功删除
      //resp.Header.Revision 是当前最小的
      if err = waitDelete(ctx, client, lastKey, resp.Header.Revision); err != nil {
         return nil, err
      }
   }
}

//
func waitDelete(ctx context.Context, client *v3.Client, key string, rev int64) error {
   cctx, cancel := context.WithCancel(ctx)
   defer cancel()

   var wr v3.WatchResponse
   // wch 是个 channel，key 被删除后会往这个 chan 发数据
   wch := client.Watch(cctx, key, v3.WithRev(rev))
   for wr = range wch {
      for _, ev := range wr.Events {
         if ev.Type == mvccpb.DELETE {
            return nil
         }
      }
   }
   if err := wr.Err(); err != nil {
      return err
   }
   if err := ctx.Err(); err != nil {
      return err
   }
   return fmt.Errorf("lost watcher waiting for delete")
}
```

使用 Etcd 实现分布式锁的架构图如下：
![etcd-d-lock](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/etcd/etcd_lock.png)

##	0x05	总结下 Etcd 分布式锁的步骤
分析完 `concurrency` 包的核心逻辑，这里总结下基于 Etcd 构造（公平式长期）分布式锁的一般流程如下：

1.	假设分布式锁的 Name 为 `/root/lockname`，用来控制某个共享资源，`concurrency` 会自动将其转换为目录形式：`/root/lockname/`

2.	客户端 A 连接 Etcd，创建一个租约 Leaseid_A，并设置 TTL（以业务逻辑来定 TTL 的时间）, 以 `/root/lockname` 为前缀创建全局唯一的 Key，该 Key 的组织形式为 `/root/lockname/{leaseid_A}`，客户端 A 将此 Key 绑定租约写入 Etcd，同时调用 TXN 事务查询写入的情况和具有相同前缀 `/root/lockname/` 的 `Revision` 的排序情况

3.	客户端 A 判断自己是否获得锁，以前缀 `/root/lockname/` 读取 keyValue 列表（keyValue 中带有 Key 对应的 `Revision`），判断自己 Key 的 `Revision` 是否为当前列表中最小的，如果是则认为获得锁；否则阻塞监听列表中前一个 `Revision` 比自己小的 Key 的删除事件，一旦监听到删除事件或者因租约失效而删除的事件，则自己获得锁

4.	执行业务逻辑，操作共享资源

5.	释放分布式锁，现网的程序逻辑需要实现在正常和异常条件下的释放锁的策略，如捕获 `SIGTERM` 后执行 `Unlock`，或者异常退出时，有完善的监控和及时删除 Etcd 中的 Key 的异步机制，避免出现死锁现象

6.	当客户端持有锁期间，其它客户端只能等待，为了避免等待期间租约失效，客户端需创建一个定时任务进行续约续期。如果持有锁期间客户端崩溃，心跳停止，Key 将因租约到期而被删除，从而锁释放，避免死锁

##	0x06	应用 concurrency 包的几个细节

####	concurrency.Session 结构
先看下 `concurrency` 封装的 [Session](https://github.com/etcd-io/etcd/blob/master/clientv3/concurrency/session.go#L28) 结构：
```golang
const defaultSessionTTL = 60	//session 的默认 TTL 是 60s

// Session represents a lease kept alive for the lifetime of a client.
// Fault-tolerant applications may use sessions to reason about liveness.
type Session struct {
	client *v3.Client		// 包含一个 clientv3 客户端
	opts   *sessionOptions
	id     v3.LeaseID		//lease 租约
	//s.Lease() 是一个 64 位的整数值，Etcd v3 引入了 lease（租约）的概念
	//concurrency 包基于 lease 封装了 session，每一个客户端都有自己的 lease，也就是说每个客户端都有一个唯一的 64 位整形值
	cancel context.CancelFunc	//context
	donec  <-chan struct{}		//
}
```
在实际项目中需要使用 Etcd 的分布式或者选主功能时，可以按照上面的类似结构来封装自己的业务结构。`defaultSessionTTL` 的意义在于，在当前某个进程获取到锁后，在其业务逻辑运行间意外崩溃退出了，其他期望的客户端在等待 `<= defaultSessionTTL` 的时间后，此 `LeaseID` 关联的 Key 被删除，下一个客户端必然可以获取到锁。

####	WithMaxCreateRev 、 WithRev、WithLastCreate 的区别


##	0x07	总结
本文分析了基于 Etcd 的分布式锁的实现。

##	0x08	参考
-	[Etcdv3.3.10：concurrency](https://github.com/etcd-io/etcd/tree/v3.3.10/clientv3/concurrency)
-	[Go 高级编程：6.2 分布式锁](https://chai2010.cn/advanced-go-programming-book/ch6-cloud/ch6-02-lock.html)
-	[what is different about Revision, ModRevision and Version](https://github.com/etcd-io/etcd/issues/6518)
-	[etcd3 API central design overview](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/api.md)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
