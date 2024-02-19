---
layout:     post
title:      Etcd 开发中的细节梳理（总结）
subtitle:   如何更优雅的使用 Etcd 实现工作项目
date:       2020-05-30
author:     pandaychen
catalog:    true
tags:
    - Etcd
---

##  0x00    前言
本文主要梳理下在使用 Etcd（仅限于 V3 版本） 进行开发中遇到的一些细节问题及解决。必读的文档如下：
-   [etcd3 API](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/api.md)


##  0x01    Etcd 的客户端
Etcd 官方提供了丰富的客户端 [实现](https://github.com/etcd-io/etcd/tree/master/client/v3/experimental/recipes)，值得参考。

####    key.go
[key.go](https://github.com/etcd-io/etcd/blob/v3.3.10/contrib/recipes/key.go)


####    客户端的选项
阅读 etcd 分布式锁 mutex 的源码时，遇到 waitDeletes 函数

##  0x02    Etcd 的 Key-Value
Etcd 的 Key-Value 结构的 PB 定义如下：
```golang
message KeyValue {
  bytes key = 1;
  int64 create_revision = 2;
  int64 mod_revision = 3;
  int64 version = 4;
  bytes value = 5;
  int64 lease = 6;
}
```

##  0x02    各种 Version 的含义
主要来自 [issue：what is different about Revision, ModRevision and Version?](https://github.com/etcd-io/etcd/issues/6518)（需要了解什么是 MVCC）：
开发者提了一个好问题：<br>
`Revision` in `ResponseHeader`, and `ModRevision` and `Version` in `KeyValue`, what is different about them. I am confused from the comment.
If I Watch with `WithRev`, which one I should fill into `WithRev`?

```golang
type ResponseHeader struct {
    // cluster_id is the ID of the cluster which sent the response.
    ClusterId uint64 `protobuf:"varint,1,opt,name=cluster_id,json=clusterId,proto3" json:"cluster_id,omitempty"`
    // member_id is the ID of the member which sent the response.
    MemberId uint64 `protobuf:"varint,2,opt,name=member_id,json=memberId,proto3" json:"member_id,omitempty"`
    // revision is the key-value store revision when the request was applied.
    Revision int64 `protobuf:"varint,3,opt,name=revision,proto3" json:"revision,omitempty"`
    // raft_term is the raft term when the request was applied.
    RaftTerm uint64 `protobuf:"varint,4,opt,name=raft_term,json=raftTerm,proto3" json:"raft_term,omitempty"`
}

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

从 `Key-Value` 的结构看，其关联了 `CreateRevision`、`ModRevision` 及 `Version`，而 `Revision` 是在每次客户端请求服务端的 `ResponseHeader` 中返回的。

答案如下：<br>
> the Revision is the current revision of etcd. It is incremented every time the v3 backed is modified (e.g., Put, Delete, Txn). ModRevision is the etcd revision of the last update to a key. Version is the number of times the key has been modified since it was created. Get(..., WithRev(rev)) will perform a Get as if the etcd store is still at revision rev.

验证一下上面的结论：
```bash
[root@VM_0_7_centos etcd_tools]# etcdctl get key1 --write-out=json
{"header":{"cluster_id":14841639068965178418,"member_id":10276657743932975437,"revision":51,"raft_term":2},"kvs":[{"key":"a2V5MQ==","create_revision":47,"mod_revision":51,"version":5,"value":"dmFsdWUx"}],"count":1}

[root@VM_0_7_centos etcd_tools]# etcdctl del key1
1

[root@VM_0_7_centos etcd_tools]# etcdctl get key1 --write-out=json
{"header":{"cluster_id":14841639068965178418,"member_id":10276657743932975437,"revision":52,"raft_term":2}}

[root@VM_0_7_centos etcd_tools]# etcdctl put key1 value11
OK
[root@VM_0_7_centos etcd_tools]# etcdctl get key1 --write-out=json
{"header":{"cluster_id":14841639068965178418,"member_id":10276657743932975437,"revision":53,"raft_term":2},"kvs":[{"key":"a2V5MQ==","create_revision":53,"mod_revision":53,"version":1,"value":"dmFsdWUxMQ=="}],"count":1}
```

此时，我们通过下面的代码查询 `key1` 的值，使用 `clientv3.WithRev(47)` 这个条件，仍然得到了 `create_revision=47` 时 `key1` 的值，同时 `head revision=53` 告诉我们当前 `key1` 的 `Revision` 是 `53`：
```golang
func main() {
        endpoints := []string{"127.0.0.1:2379"}
        cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
        if err != nil {
                log.Fatal(err)
        }
        defer cli.Close()

        getOps := append(clientv3.WithLastCreate(), clientv3.WithRev(47))
        resp, err := cli.Get(context.TODO(), "key1", getOps...)
        if err != nil {
                log.Fatal(err)
        }
        fmt.Println("len(resp.Kvs)=",len(resp.Kvs))
        if len(resp.Kvs) != 0 {
                fmt.Printf("%s -> %s, create revsion=%d, head revision=%d\n",
                        string(resp.Kvs[0].Key),
                        string(resp.Kvs[0].Value),
                        resp.Kvs[0].CreateRevision,
                        resp.Header.Revision)
        }

}
```
结果为：
```javascript
[root@VM_0_7_centos etcd_tools]# go run withrev.go
key1 -> value1, create revsion=47, head revision=53
```

再次删掉 `key1`，查询一个不存在的 `Revision`，会得到 `len(resp.Kvs)==0` 的结果，符合我们的预期。
```bash
[root@VM_0_7_centos etcd_tools]# etcdctl del key1
1
[root@VM_0_7_centos etcd_tools]# etcdctl get key1 --write-out=json
{"header":{"cluster_id":14841639068965178418,"member_id":10276657743932975437,"revision":54,"raft_term":2}}
```

总结下：<br>

| 字段 | 关联范围 | 说明 |
|------|------------|----------|
| Revision   |  全局     | Revision 表示改动序号（ID），每次 KV 的变化，Leader 节点都会修改 Revision 值，因此，这个值在 Cluster（集群）内是全局唯一的，而且是递增的 |
| ModRevison   |   Key      |     记录了某个 key 最近修改时的 Revision，即它是与 key 关联的  |
| Version  | Key |  表示 KV 的版本号，初始值为 1，每次修改 KV 对应的 version 都会加 1，亦是作用在 Key 之内的 |
| CreateRevision | 全局 | 某个 Key 创建时对应的 Revision，通过此值可以从集群获取到该 Revision 对应的 Key 属性，不管此 Key 有无被删除 |

最后，再举一个简单的例子加深下印象：
```javascript
Revision and ModRevision are different in that ModRevision is the last revision of a key.

$ ETCDCTL_API=3 ./bin/etcdctl put foo bar

$ ETCDCTL_API=3 ./bin/etcdctl get foo --write-out=json
revision: 2
mod_revision: 2
version: 1

$ ETCDCTL_API=3 ./bin/etcdctl put foo bar

$ ETCDCTL_API=3 ./bin/etcdctl get foo --write-out=json
revision: 3
mod_revision: 3
version: 2

$ ETCDCTL_API=3 ./bin/etcdctl put hello world

$ ETCDCTL_API=3 ./bin/etcdctl get foo --write-out=json
revision: 4
mod_revision: 3
version: 2

$ ETCDCTL_API=3 ./bin/etcdctl get hello --write-out=json
revision: 4
mod_revision: 4
version: 1

$ ETCDCTL_API=3 ./bin/etcdctl put hello world

$ ETCDCTL_API=3 ./bin/etcdctl get hello --write-out=json
revision: 5
mod_revision: 5
version: 2
```

##  0x03    Etcd 的 Resp 的字段含义

##  0x04    Etcd 的 Txn
Etcd 使用 `Txn` 提供了事务处理的机制。使用 Txn 特性，可以一次性操作多条语句（在一个 `Revision` 中完成）：
```golang
func main() {
        endpoints := []string{"127.0.0.1:2379"}
        cli, err := clientv3.New(clientv3.Config{Endpoints: endpoints})
        if err != nil {
                log.Fatal(err)
        }
        defer cli.Close()

        _, err = cli.Txn(context.TODO()).If().Then(
                clientv3.OpPut("key1", "value1"),
                clientv3.OpPut("key2", "value2"),
                clientv3.OpPut("key3", "value3"),
        ).Commit()
        if err != nil {
                log.Fatal(err)
        }
        resp, err := cli.Txn(context.TODO()).If().Then(
                clientv3.OpGet("key1"),
                clientv3.OpGet("key2"),
                clientv3.OpGet("key3"),
        ).Commit()
        if err != nil {
                log.Fatal(err)
        }
        for _, rp := range resp.Responses {
                for _, ev := range rp.GetResponseRange().Kvs {
                        fmt.Printf("%s->%s,create revision=%d,version=%d,mod revivison=%d\n",
                        string(ev.Key), string(ev.Value), ev.CreateRevision, ev.Version, ev.ModRevision)
                }
        }
}
```
上面输入如下，三个 key 具有相同的 `Create Revision` 值：
```javascript
key1->value1,create revision=47,version=5,mod revivison=51
key2->value2,create revision=47,version=5,mod revivison=51
key3->value3,create revision=47,version=5,mod revivison=51
```

##  参考
-   [what is different about Revision, ModRevision and Version?](https://github.com/etcd-io/etcd/issues/6518)
-   [重要文档：etcd3 API](https://github.com/etcd-io/etcd/blob/master/Documentation/learning/api.md)
-   [etcd:recipes](https://github.com/etcd-io/etcd/tree/v3.3.10/contrib/recipes)
