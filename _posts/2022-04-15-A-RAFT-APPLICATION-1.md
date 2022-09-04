---
layout: post
title: Raft 协议分析与实战：构建自己的分布式缓存
subtitle: hashicorp/raft的应用实战与分析
date: 2022-04-15
author: pandaychen
catalog: true
tags:
  - 分布式理论
  - Raft
---

##  0x00 前言

##  0x01  验证
按照如下组成一个raft集群：

| raft节点 | 端口 | peer端口 | 存储目录 |
| :-----:| :----: | :----: | :----: |
| Node-1 | `6000` | `7000` | `node0` |
| Node-2 | `6001` | `7001` | `node1` |
| Node-3| `6002` | `7002` | `node2` |

1、启动集群，并成为leader<br>
```text
[root@VM_120_245_centos ~/stcache]# ./stcached --http=127.0.0.1:6000 --raft=127.0.0.1:7000 --node=./node0 -bootstrap=true


stCached: 2022/07/29 18:08:53 http server listen:127.0.0.1:6000
2022-07-29T18:08:53.158+0800 [INFO]  raft: initial configuration: index=0 servers=[]
2022-07-29T18:08:53.158+0800 [INFO]  raft: entering follower state: follower="Node at 127.0.0.1:7000 [Follower]" leader-address= leader-id=
2022-07-29T18:08:54.395+0800 [WARN]  raft: heartbeat timeout reached, starting election: last-leader-addr= last-leader-id=
2022-07-29T18:08:54.396+0800 [INFO]  raft: entering candidate state: node="Node at 127.0.0.1:7000 [Candidate]" term=2
2022-07-29T18:08:54.413+0800 [DEBUG] raft: votes: needed=1
2022-07-29T18:08:54.413+0800 [DEBUG] raft: vote granted: from=127.0.0.1:7000 term=2 tally=1
2022-07-29T18:08:54.413+0800 [INFO]  raft: election won: tally=1
2022-07-29T18:08:54.413+0800 [INFO]  raft: entering leader state: leader="Node at 127.0.0.1:7000 [Leader]"
stCached: 2022/07/29 18:08:54 become leader, enable write api #成为leader
```

2、启动第`2`个节点<br>
```bash
[root@VM_120_245_centos ~/blog_backup/stcache/stcache]# ./stcached --http=127.0.0.1:6001 --raft=127.0.0.1:7001 --node=./node1 --join=127.0.0.1:6000
stCached: 2022/07/29 18:10:35 http server listen:127.0.0.1:6001
2022-07-29T18:10:35.580+0800 [INFO]  raft: initial configuration: index=0 servers=[]
2022-07-29T18:10:35.580+0800 [INFO]  raft: entering follower state: follower="Node at 127.0.0.1:7001 [Follower]" leader-address= leader-id=
2022-07-29T18:10:35.592+0800 [WARN]  raft: failed to get previous log: previous-index=3 last-index=0 error="log not found"
2022-07-29T18:11:08.417+0800 [ERROR] raft: failed to take snapshot: error="nothing new to snapshot"
```

3、启动第`3`个节点<br>
```bash
[root@VM_120_245_centos ~/blog_backup/stcache/stcache]# ./stcached --http=127.0.0.1:6002 --raft=127.0.0.1:7002 --node=./node2 --join=127.0.0.1:6000
stCached: 2022/07/29 18:10:55 http server listen:127.0.0.1:6002
2022-07-29T18:10:55.231+0800 [INFO]  raft: initial configuration: index=0 servers=[]
2022-07-29T18:10:55.231+0800 [INFO]  raft: entering follower state: follower="Node at 127.0.0.1:7002 [Follower]" leader-address= leader-id=
2022-07-29T18:10:55.239+0800 [WARN]  raft: failed to get previous log: previous-index=4 last-index=0 error="log not found"
```

节点`2`、节点`3`加入集群后，leader日志：
![raft-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/hashcorp-raft-cache/1-nodes-joined.png)


4、写入/读取数据：<br>
```bash
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6000/set?key=abc&value=efg"
ok
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6000/get?key=abc"
efg
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6001/get?key=abc"
efg
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6002/get?key=abc"
efg
[root@VM_120_245_centos ~]# 
```

5、不可以向非leader写入数据<br>
```bash
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6001/set?key=abc&value=efg"
write method not allowed
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6002/set?key=abc&value=efg"
write method not allowed
```

6、写入数据时leader/follower的状态变化<br>

leader写入数据<br>
![raft-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/hashcorp-raft-cache/2-leader-write-data.png)

节点`2`（follower1）<br>
![raft-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/hashcorp-raft-cache/2-follower-write-data.png)

节点`3`（follower2）<br>
![raft-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/hashcorp-raft-cache/2-follower-write-data-2.png)

7、关闭leader节点<br>
关闭节点`1`后，节点`3`通过选举成为了新的leader

![raft-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/hashcorp-raft-cache/3-new-term-leader.png)

节点`3`可写：
```bash
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6002/set?key=cba&value=gfe"
ok
```

8、查询follower变迁及数据<br>
```bash
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6001/get?key=cba"
gfe
```

9、将节点`1`重启，再加入集群，自动变为follower
![raft-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/hashcorp-raft-cache/3-new-term-follower.png)

leader状态：
![raft-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/raft/hashcorp-raft-cache/3-new-term-leader.png)

再查询节点`1`的数据：
```bash
[root@VM_120_245_centos ~]# curl "http://127.0.0.1:6000/get?key=cba"
gfe
```

##  参考
-   [基于hashicorp/raft的分布式一致性实战教学](https://cloud.tencent.com/developer/article/1183490)
-   [Golang implementation of the Raft consensus protocol](https://github.com/hashicorp/raft)
-   [使用开源技术构建有赞分布式KV存储服务](https://tech.youzan.com/shi-yong-kai-yuan-ji-zhu-gou-jian-you-zan-fen-bu-shi-kvcun-chu-fu-wu/)
-   [深入分布式缓存-Raft & Lease机制](https://juejin.cn/post/7044442593895120903)