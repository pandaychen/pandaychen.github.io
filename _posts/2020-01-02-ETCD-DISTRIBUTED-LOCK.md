---
layout:     post
title:      Etcd 应用开发之分布式锁
subtitle:   Etcd 应用开发（续）
date:       2020-01-02
author:     pandaychen
catalog:    true
tags:
    - Etcd Lock
    - 分布式锁
---

##  分布式锁基础
在分布式系统中，为了实现对互斥资源的安全访问（独占），必须要用到分布式锁。另外在工作中，还有一个应用场景是，在分布式的后台服务中，某些服务只允许单个实例（机器或docker）运行，其他的备份机器作为BackUps，当正在运行的单实例机器故障后，在线的BackUps按照排队顺序，争抢到锁，继续执行，这样也保证了单实例场景的高可用性。

分布式锁应具备以下特点：
-  互斥性：任意时刻，同一个锁，只有一个进程能持有
-  安全性：避免死锁，当进程没有主动释放锁（进程崩溃退出），保证其他进程能够加锁
-  可用性：当提供锁的服务节点故障（宕机）时，"热备"节点能够接替故障的节点继续提供服务，并保证自身持有的数据与故障节点一致
-  对称性：对同一个锁，加锁和解锁必须是同一个进程，即不能把其他进程持有的锁给释放了

可以基于数据库/缓存/中间件等实现分布式锁，比较主流的是使用Redis、Etcd或ZooKeeper等开源项目来构建。

##  分布式锁的两种形态
通常，分布式锁有如下两种形态：
-  短期锁：即用即申请，用完即释放，一般用于资源控制粒度比较细的系统，这种场景会频繁的调用 Etcd 服务
-  长期锁：这种锁是先到先得，得到即长期占有，更多是用在主备系统的切换场景，如果占有锁的服务不发生异常，则不会主动与 Etcd 交互
这里多提一句，无论是短期还是长期占有的锁，在锁竞争上最好都遵从公平的策略，即先注册（排队）的进程（实例）先获取锁。

##	分布式锁的实现方案
要实现分布式锁，核心在两点，一是如何达成共识、二是状态同步。目前的实现方式一般会借助于Redis/ZooKeeper/Etcd来实现：
1.	采用REDIS的SETNX命令+LUA原子脚本来实现（单REDIS实例），存在单点的风险
2.	对1的改进版是采用集群Redis的[Redlock](http://stor.51cto.com/art/201901/590874.htm)
3. [基于ZooKeeper实现的分布式锁](https://juejin.im/post/5c01532ef265da61362232ed)<br>
此外，下面这篇文章值得一读：
[基于Redis的分布式锁到底安全吗](http://zhangtielei.com/posts/blog-redlock-reasoning.html)

## Etcd实现分布式锁
这些年，随着[Raft协议](https://raft.github.io/)的广泛应用，涌现出很多的基于此协议实现的高可用分布式系统，比较知名的有[Etcd](https://github.com/etcd-io/etcd)、[Consul](https://github.com/hashicorp/consul)等，由于其内部实现了[分布式一致性](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86)，所以非常适合做分布式锁。本片文章就以Etcd为例，来介绍下构建分布式锁的方法。值得注意的是，Etcd分布式锁的实现是在客户端做的。

Etcd 支持以下功能，正是依赖这些功能来实现分布式锁的：
-	Lease 机制：即租约机制（TTL，Time To Live），Etcd 可以为存储的 KV 对设置租约，当租约到期，KV 将失效删除；同时也支持续约，即 KeepAlive
-	Revision 机制：每个 key 带有一个 Revision 属性值，Etcd 每进行一次事务对应的全局 Revision 值都会加一，因此每个 key 对应的 Revision 属性值都是全局唯一的。通过比较 Revision 的大小就可以知道进行写操作的顺序。 在实现分布式锁时，多个程序同时抢锁，根据 Revision 值大小依次获得锁，可以避免"惊群效应"，实现公平锁
-	Prefix 机制：即前缀机制，也称目录机制。可以根据前缀（目录）获取该目录下所有的 key 及对应的属性（包括 key, value 以及 revision 等）
-	Watch 机制：即监听机制，Watch 机制支持 Watch 某个固定的 key，也支持 Watch 一个目录（前缀机制），当被 Watch 的 key 或目录发生变化，客户端将收到通知

###  分析Concurrency包

EtcdV3版本的[Concurrency包](https://godoc.org/github.com/coreos/etcd/clientv3/concurrency)提供了一些实现[分布式锁](https://github.com/coreos/etcd/blob/master/clientv3/concurrency/mutex.go)及[分布式选主](https://github.com/coreos/etcd/blob/master/clientv3/concurrency/election.go#L31)的接口，这两个概念间是有共性的，譬如，谁抢到了锁谁就是Master.
我们从Etcd提供的[Lock方法](https://github.com/etcd-io/etcd/blob/master/etcdctl/ctlv3/command/lock_command.go)来入手，看看Lock操作是如何实现的：

command.lockUntilSignal
``` golang
func lockUntilSignal(c *clientv3.Client, lockname string, cmdArgs []string) error {
   m := concurrency.NewMutex(s, lockname)		//通过concurrency的NewMutex初始化
    ctx, cancel := context.WithCancel(context.TODO())
    ...
    if err := m.Lock(ctx); err != nil {	//加锁，Etcd的分布式锁的主要实现
    	return err
    }
    ...
}
```
先贴一下锁Mutex部分的Code，[Mutex](https://github.com/coreos/etcd/blob/master/clientv3/concurrency/mutex.go#L26)：
在Code中注释介绍了具体的实现，代码也不长，简单分析一下：
```
// Mutex implements the sync Locker interface with etcd
type Mutex struct {
	s *Session
	pfx   string   //锁的共同前缀pfx，如"/service/lock/"
	myKey string   //当前持有锁的客户端的leaseid值（完整Key的组成为 pfx+"/"+leaseid）
	myRev int64    //revision，理解为当前持有锁的Revision编号
	hdr   *pb.ResponseHeader
}
```
初始化Mutex，可以看到在NewMutex方法中，并非直接拿传进来的pfx作为prefix的，而且在后面加了个"/"，在Etcd开发项目中，定义或查找prefix或suffix时都建议加上分隔符"/"，这是个好习惯，也可以避免出现问题。
```
func NewMutex(s *Session, pfx string) *Mutex {
   //Etcd这里默认将/path的key结构转换为一个目录结构 /path/
   return &Mutex{s, pfx + "/", "", -1, nil}     
}
```
TryAcquire：最基础的加锁方法，不难看出，该方法是通过Etcd的TXN（多键条件事务）原子方式来实现的，EtcdV3的TXN的语意是先做一个比较（compare）操作，如果比较成立则执行一系列操作，如果比较不成立则执行另外一系列操作。有类似于C语言中的条件表达式。即下面这句：
Txn事务，以原子方式判定cmp的条件，如果为True，就设置getOwner的值；否则False，返回getOwner的值；最终用Commit()提交
```
resp, err := client.Txn(ctx).If(cmp).Then(put, getOwner).Else(get, getOwner).Commit()
```
下面看下TryAcquire方法：
```
func (m *Mutex) tryAcquire(ctx context.Context) (*v3.TxnResponse, error) {
	s := m.s
	client := m.s.Client()
	//下面的m.pfx就是prefix，是传进来的前缀，后面的s.Lease()会返回一个租约，是一个int64的整数，和session有关系
	//mykex先理解为是/prefix/leaseid 这样的结构
	m.myKey = fmt.Sprintf("%s%x", m.pfx, s.Lease())

	/*这里定义一个cmp方法，比较上面m.myKey的CreateRevision是否为0
	等于0表示目前不存在该key，需要执行Put操作
	不等于0表示对应的key已经创建了，需要执行Get操作*/
	cmp := v3.Compare(v3.CreateRevision(m.myKey), "=", 0)
	// put self in lock waiters via myKey; oldest waiter holds lock
	put := v3.OpPut(m.myKey, "", v3.WithLease(s.Lease()))
	// reuse key in case this session already holds the lock
	get := v3.OpGet(m.myKey)
	// fetch current holder to complete uncontended path with only one RPC
    // 获取当前锁的真正持有者
	getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)
	//Txn事务，判断cmp的条件是否成立，成立执行Then，不成立执行Else，最终执行Commit()
	resp, err := client.Txn(ctx).If(cmp).Then(put/*resp.Responses[0]*/, getOwner/*resp.Responses[1]*/).Else(get, getOwner).Commit()
	if err != nil {
		return nil, err
	}
	//succ
	m.myRev = resp.Header.Revision
	if !resp.Succeeded {
		//失败，执行Else(get,getOwner)
		m.myRev = resp.Responses[0].GetResponseRange().Kvs[0].CreateRevision
	}
	return resp, nil
}
```
在tryAcquire的TXN事务中，不管是执行Put还是Get，最终都有个getOwner的过程，看一下这个getOwner，这里有个有趣的选项，getOwner := v3.OpGet(m.pfx, v3.WithFirstCreate()...)中的options模式里有个v3.WithFirstCreate函数调用，这是一个和prefix相关的函数，看看它的实现：
```
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
	//返回所有满足prefix匹配的key-value，和etcdctl get key --prefix功能一样
   return func(op *Op) {
      if len(op.key) == 0 {
         op.key, op.end = []byte{0}, []byte{0}
         return
      }
      op.end = getPrefix(op.key)
   }
}
```
看到上面的代码，WithPrefix/WithSort，所以getOwner的具体执行效果是会把虽有以lockkey开头的key-value都拿到，且按照CreateRevision升序排列，取第一个值，这个意思就很明白了，就是要拿到当前以lockkey为prefix的且CreatereVision最小的那个key，就是目前已经拿到锁的key;

TryLock/Lock上层调用
```
// Lock locks the mutex with a cancelable context. If the context is canceled
// while trying to acquire the lock, the mutex tries to clean its stale lock entry.
func (m *Mutex) Lock(ctx context.Context) error {
	//先调用tryAcquire尝试获取锁
	resp, err := m.tryAcquire(ctx)
	if err != nil {
		return err
	}
	// if no key on prefix / the minimum rev is key, already hold the lock
	ownerKey := resp.Responses[1].GetResponseRange().Kvs
	//比较如果当前没有人获得锁（第一次场景）
	//或者锁的owner的CreateRevision等于当前的kv的Revision，则表示已成功获得锁，就可以退出了
	if len(ownerKey) == 0 || ownerKey[0].CreateRevision == m.myRev {	
		//拿当前进程的Revision和Owner的CreateVision比较，如果相等，就是拿到了锁
		m.hdr = resp.Header
		return nil
	}
	client := m.s.Client()
	// wait for deletion revisions prior to myKey
	// 走到这里代表没有获得锁，需要等待之前的锁被释放，即revision小于当前revision的kv被删除
	hdr, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
	// 阻塞等待Owner释放锁，下面分析waitDeletes
	// release lock key if wait failed
	if werr != nil {
		m.Unlock(client.Ctx())
	} else {
		//成功获取锁
		m.hdr = hdr
	}
	return werr
}
```
再看看waitDeletes函数的行为, waitDeletes 模拟了一种公平的先来后到的排队逻辑，等待所有当前比当前key的revision小的key被删除后，锁释放后才返回
```
func waitDeletes(ctx context.Context, client *v3.Client, pfx string, maxCreateRev int64) (*pb.ResponseHeader, error) {
   // 注意WithLastCreate和WithMaxCreateRev这两个特性
   getOpts := append(v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev))
   for {
      resp, err := client.Get(ctx, pfx, getOpts...)
      if err != nil {
         return nil, err
      }
      if len(resp.Kvs) == 0 {
         return resp.Header, nil
      }
      lastKey := string(resp.Kvs[0].Key)
      if err = waitDelete(ctx, client, lastKey, resp.Header.Revision); err != nil {
         return nil, err
      }
   }
}

func waitDelete(ctx context.Context, client *v3.Client, key string, rev int64) error {
   cctx, cancel := context.WithCancel(ctx)
   defer cancel()
 
   var wr v3.WatchResponse
   // wch是个channel，key被删除后会往这个chan发数据
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
##	总结下Etcd实现分布式锁的步骤
分析完concurrency的主要代码，不难总结出用Etcd构造（公平式长期）分布式锁的一般流程如下：

1.	假设锁的name为 /root/lockname，用来控制某个共享资源，concurrency会自动将其转换为目录形式：/root/lockname/

2.	客户端A连接 Etcd，创建一个租约Leaseid_A，并设置 TTL（以业务逻辑来定）,以 /root/lockname 为前缀创建全局唯一的 key，该key的组织形式为/root/lockname/{leaseid_A}，客户端A将此 Key 绑定租约写入 Etcd，同时调用TXN事务查询写入的情况和具有相同前缀/root/lockname/的Revision的排序情况

3.	客户端A判断自己是否获得锁，以前缀 /root/lockname/ 读取 keyValue 列表（keyValue 中带有 key 对应的 Revision），判断自己 key 的 Revision 是否为当前列表中最小的，如果是则认为获得锁；否则阻塞监听列表中前一个 Revision 比自己小的 key 的删除事件，一旦监听到删除事件或者因租约失效而删除的事件，则自己获得锁。

4.	执行业务逻辑，操作共享资源

5.	释放分布式锁，现网的程序逻辑需要实现在正常和异常条件下的释放锁的策略，如捕获SIGTERM后执行Unlock，或者异常退出时，有完善的监控和及时删除Etcd中的Key的异步机制，避免出现"死锁"现象

6.	当客户端持有锁期间，其它客户端只能等待，为了避免等待期间租约失效，客户端需创建一个定时任务进行续约续期。如果持有锁期间客户端崩溃，心跳停止，Key 将因租约到期而被删除，从而锁释放，避免死锁

##	应用concurrency包的几个细节
让我们来看下concurrency封装的[Session](https://github.com/etcd-io/etcd/blob/master/clientv3/concurrency/session.go#L28)结构
```
const defaultSessionTTL = 60	//session的默认TTL是60s

// Session represents a lease kept alive for the lifetime of a client.
// Fault-tolerant applications may use sessions to reason about liveness.
type Session struct {
	client *v3.Client
	opts   *sessionOptions
	id     v3.LeaseID	
	//s.Lease()是一个64位的整数值，Etcd v3引入了lease（租约）的概念
	//concurrency包基于lease封装了session，每一个客户端都有自己的lease，也就是说每个客户端都有一个唯一的64位整形值
	cancel context.CancelFunc
	donec  <-chan struct{}
}

```

##	总结

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权