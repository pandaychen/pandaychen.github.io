---
layout:     post
title:      分布式一致性 hash 算法的实现分析
subtitle:   ketama 算法分析与应用
date:       2020-08-18
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - 分布式
    - consistent-hash
---

##  0x00    前言

在工作中，遇到过如下几种需要分布式一致的场景：

1.  分布式一致的缓存系统
2.  gRPC的基于一致性算法的loadbalance实现，某些业务场景下，需要尽可能使指定客户端的请求经由consistent-hash算法到达指定的服务器进行处理

####    consistent-hash 的应用场景
consistent-hash 算法主要的应用场景是：当服务是一个有状态服务等时候，需要根据特定的 key 路由到相同的目标服务机器进行处理的场景。

比如，考虑下面这种有状态的缓存服务，对 N（海量）个员工信息进行缓存，在访问时，需要对其做增删查。同时，数据量规模较大，需要存储在多台机器上（将 hash 缓存分发到多个服务器），要求是 key 的存储尽可能均衡，有几种方式：

```text
john@example.com => john
mark@example.com => mark
tesla@example.com => tesla
adam@examle.com => adam
......
```

1. 普通 hash：按服务器数量，对 key 进行 mod。问题是当某台服务器故障下线了（或者扩容新增服务器），都需要 ReHash，**此方法会导致所有系统中缓存的 key 都要重新计算一次**
    -   对后端服务器来说，将大大增加服务器的负载（因为会丢失缓存后需要重新查询建立映射）
    -   对客户端来说，由于映射方式的改变大量 mail 都被映射到新的服务器上，导致映射在这一刻不可用，并且容易导致缓存穿透问题

所以一致性 hash 非常适合解决此类场景，服务器的伸缩不会导致系统中所有缓存的 Key 都重新计算，仅仅一小部分 Key 需要重新计算。hashring的结构实现了在槽位数量变化前后的增量式的重新映射，避免了全量的重新映射。

##  0x01    consistent-hash的原理

####    实现步骤
1、把节点（用户存储kv的服务器节点）通过hash后，映射到一个范围是`[0,2^32]`的环上，比如，采用`ip:port`来命名一个节点，下图有N0~N2共`3`个物理节点：

![ch-1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consistenthash/consistent-hash-1.png)

2、把要缓存的kv数据也通过hash的方式映射到`[0，2^32]`环上，然后按顺时针方向查找第一个hash值大于等于数据的hash值的节点，该节点即为数据所分配到的节点

![ch-2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consistenthash/consistent-hash-2.png	)

3、可优化点：当删除某个服务器时，可能存在该服务器前和后面的直接的key数据没有均衡分布，所以通常引入虚拟节点来优化均衡性问题。因为节点越多，它们在环上的分布就越均匀，使用虚拟节点还可以降低节点之间的负载差异。理论的情况，从`n`个服务器扩容到`n+1`时，只需要重新映射 $\frac{k}{n+1}$ 的key（`k`为hashring的key总数），如下图：

![ch-3](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consistenthash/consistent-hash-4.png)

此时，查找的逻辑变更为如下：
-	先根据key，通过固定的hash算法计算得到虚拟节点的value（映射到hashring上的value）
-	再根据value查找到对应的真实Node节点

![ch-4](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/consistenthash/consistent-hash-5.png)

##  0x02    consistent-hash的一种实现：ketama算法
本小节分析下ketama的实现[代码](https://github.com/pandaychen/grpclb2etcd/blob/master/utils/ketama.go)，其思路如下：

1.	将server（server的虚拟副本）和key同时映射到hashring上（区间为`[0~2^32]`)
2.	为了将key映射（放置）到server，找到**第一个比key的映射值大的server的映射值**，则key就映射到这台server上；如果没找到，则映射至第一台server（即在hashring上的首个server位置）
3.	为了平衡性，可以添加一层虚拟节点到物理节点的映射，将key首先映射到虚拟节点，然后再映射到物理节点
其中的策略就是将哈希空间固定并分段，段分割点就是新的映射值，将映射到段中间的所有key都映射到段分割点。这样段分割点如果失效，那么只影响映射到该段分割点的key，而不影响其他key，添加段分割点是同样的逻辑

####    数据结构
ketama的管理节点定义如下：
-	`replicas`：生成虚拟节点的副本数
-	`ringKeys`：hashring结构（注意，这里的`[]int`必须是有序的，才能表达出hashring的特性）
-	`hashMap`：存储虚拟节点对应真实节点的映射关系

```GO
type KetamaConsistent struct {
	sync.RWMutex
	hash       HashFunc
	replicas   int
	ringKeys   []int          //  Sorted keys（最终RING）
	hashMap    map[int]string //最终ring上节点的映射
	serviceMap map[string][]string
}
```

####	操作方法

1、向consistent-hash中添加真实Node节点<br>
```GO
//向CH中添加ServerNode（物理节点）
func (k *KetamaConsistent) AddSrvNode(srvnodes ...string) {
	k.Lock()
	defer k.Unlock()

	for _, node := range srvnodes {
		//扩容副本
		for i := 0; i < k.replicas; i++ {
			//将副本转变为ring上的key
			key := int(k.hash([]byte(strconv.Itoa(i) + node)))

			if _, ok := k.hashMap[key]; !ok {
				k.ringKeys = append(k.ringKeys, key)
			}
			k.hashMap[key] = node
			k.serviceMap[node] = append(k.serviceMap[node], strconv.Itoa(key))
		}
	}

	//方便二分查找，对ringKeys数组排序
	sort.Ints(k.ringKeys)
}
```

2、从consistent-hash中移除真实Node<br>
```GOLANG
//有现网服务器宕机,需要将该server关联的所有key，从ring上移除
func (k *KetamaConsistent) RemoveSrvNode(nodes ...string) {
	k.Lock()
	defer k.Unlock()

	deletedkey_list := make([]int, 0)
	for _, node := range nodes {
		for i := 0; i < k.replicas; i++ {
			key := int(k.hash([]byte(strconv.Itoa(i) + node)))

			if _, ok := k.hashMap[key]; ok {
				deletedkey_list = append(deletedkey_list, key)
				delete(k.hashMap, key)
			}
		}
		//删除原有Srv节点的所有映射
		delete(k.serviceMap, node)
	}

	if len(deletedkey_list) > 0 {
		k.deleteKeys(deletedkey_list)
	}
}
```

`k.deleteKeys`为从hashring中移除key数组，实现如下：
```GO
//从ring（数组）中移除key，采用二分法较为高效
func (k *KetamaConsistent) deleteKeys(deletedKeysList []int) {

	//按升序排序
	sort.Ints(deletedKeysList)

	index := 0
	count := 0
	for _, key := range deletedKeysList {
		for ; index < len(k.ringKeys); index++ {
			k.ringKeys[index-count] = k.ringKeys[index]
			if key == k.ringKeys[index] {
				count++
				index++
				break
			}
		}
	}

	for ; index < len(k.ringKeys); index++ {
		k.ringKeys[index-count] = k.ringKeys[index]
	}

	k.ringKeys = k.ringKeys[:len(k.ringKeys)-count]
}
```

3、根据指定的key，在consistent-hash上查询指定的Node<br>
```GOLANG
func (k *KetamaConsistent) GetSrvNode(client_key string) (string, bool) {
	if k.IsEmpty() {
		return "", false
	}
	k.RLock()
	defer k.RUnlock() //HERE must use  k.RUnlock() (core if use  k.Unlock() )

	//计算客户端传入的client_key的hash值
	hashval := int(k.hash([]byte(client_key)))

	//这里隐含了一层意思：k.keys[i] >= hash时，有可能计算出来的hash值比ring上的所有key值都大
	//为了找个一个Key应该放入哪个服务器，先哈希你的key，得到一个无符号整数, 沿着圆环找到和它相邻的最大的数，这个数对应的服务器就是被选择的服务器
	//对于靠近 2^32的 key, 因为没有超过它的数字点，按照圆环的原理，选择圆环中的第一个服务器
	//（通过这个函数，将一个可能不存在与ringkey数组的key（但是一定在环上），修正为离它最近的ringKey数组的key的下标）
	index := sort.Search(len(k.ringKeys), func(i int) bool {
		return k.ringKeys[i] >= hashval //if overflow,then returns len(k.ringKeys)
	})

	if index == len(k.ringKeys) {
		//it will core if not deal this case
		index = 0
	}

	conhash_key := k.ringKeys[index] //

	serveraddr, exsits := k.hashMap[conhash_key]
	return serveraddr, exsits
}
```

##  0x03    其他consistent-hash

####     Jump Consistent Hash
可以参考此文：[一致性哈希算法（三）- 跳跃一致性哈希法](https://writings.sh/post/consistent-hashing-algorithms-part-3-jump-consistent-hash)，根据[论文]()中的测试结果， Jump Consistent Hash在执行速度、内存消耗、映射均匀性上比ketama要好。


##  0x04 参考
-   [一致性 hash 算法 - consistent hashing](https://blog.csdn.net/sparkliang/article/details/5279393)
-   [一致性哈希 （Consistent Hashing）的前世今生](https://candicexiao.com/consistenthashing/)
-   [一致性哈希算法（二）- 哈希环法](https://writings.sh/post/consistent-hashing-algorithms-part-2-consistent-hash-ring)