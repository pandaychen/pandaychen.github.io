---
layout:     post
title:      基于 Golang 实现的定时器分析与实现（未完成）
subtitle:   Timer and Ticker： events in the future
date:       2020-06-01
author:     pandaychen
catalog:    true
header-img: img/panda-md-pic7.jpg
tags:
    - Golang
    - 定时器
    - Timer
---

##  0x00    前言
定时器（Timer）常用于解决 <font color="#dd0000"> 多长时间后触发处理某些事件 </font> 的问题。在项目中，通常遇到下面两类问题都可以使用定时器来解决：
1.  延迟消息、事件处理，如订单默认 `N` 时间跨度后自动评价等
2.  长连接心跳管理，如在 IM 中，当客户端与服务端 `N` 时间跨度内没有 Heartbeat 的话，需要断开此无效连接

####    实现方案

##  0x01    Golang 原生库的使用

##  0x02    原生库的实现

##  0x03    最小堆的实现
最小堆（Minheap）定时器的经典用法是和 [`epoll_wait` 系统调用](https://man7.org/linux/man-pages/man2/epoll_wait.2.html) 的结合，即与参数 `timeout` 结合：
![epoll_wait](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/linux/epoll_wait.png)
`timeout` 表示超时时间（一般的实例都是设置一个固定的值），但是可以通过与最小堆结合，在事件驱动循环中动态计算 `timeout` 的值。具体做法是：通过小根堆保存每个超时事件，将 `timeout` 设置为 top 结点（即堆顶）的时间，就完美的将网络事件 IO 与超时事件结合在一起。<br>
具体的示例代码可以参见：[min_heap.c](https://github.com/pandaychen/tcpframe/blob/main/src/min_heap.c)

![epoll_min_heap](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/linux/epoll_min_heap.jpg)

##  0x04    GOIM 的最小堆定时器
Goim 是一个 Tcp 长连接模型，实现了 [最小堆 Timer](https://github.com/Terry-Mao/goim/tree/master/pkg/time) 的定时器结构，用来解决海量连接高性能心跳处理的问题。对外提供了三个方法：

Timer 对外提供 `Add`、`Del` 及 `Set` 三个方法用于添加，删除、修改 `TimerData`：
1.  [Add 方法](https://github.com/Terry-Mao/goim/blob/master/pkg/time/timer.go#L102)：
2.  [Del 方法](https://github.com/Terry-Mao/goim/blob/master/pkg/time/timer.go#L114)：
3.  [Set 方法](https://github.com/Terry-Mao/goim/blob/master/pkg/time/timer.go#L168)：

####    基础结构
1.  `Timer` 是管理定时器的结构，管理一组定时器。注意其成员 `timers []*TimerData` 是一个 slice，其中每个元素都是一个 `TimerData` 的指针
2.  `TimerData` 存储单个定时器的信息，从功能来看，到期时则执行回调函数 `fn  func()`

```golang
type Timer struct {
	lock   sync.Mutex       //lock
	free   *TimerData       // 用于提升分配性能的 freelist（优化）
	timers []*TimerData     // 真正存储定时器的 slice
	signal *itime.Timer
	num    int              // 定时器个数
}

type TimerData struct {
	Key    string           // 定时器的 key
	expire itime.Time       // 超时时间
	fn     func()
	index  int              // 保存此定时器节点在定时器管理树的位置，便于查找和删除操作，调整 heap 的时候会更新
	next   *TimerData
}
```

####    Heap 的基本操作
`add` 和 `del` 是堆的最基础操作，是往 `t.timers` 这个堆里添加及删除元素：
-   `add` 为插入节点，因为在堆插入时，一般是放在最后一个位置，然后通过 `up` 方法 <font color="#dd0000"> 自下而上 </font> 调整堆的结构
-   `del` 为删除节点，分为 `4` 步：
    -   先把待删除的节点（位置为 `index`）和当前 heap 中最后一个位置（为 `last`）的节点互换
    -   这样调整后，从 `index` 位置到 `last-1` 位置之间就不符合堆的性质了，调用 `down` 方法从 `index` 位置 <font color="#dd0000"> 自上而下 </font> 调整
    -   最后，调用 `up` 方法从 `index` 位置 <font color="#dd0000"> 自下而上 </font> 调整
    -   去掉 `last` 位置的元素

这里关于 `del` 方法有个细节：`index` 和 `last` 不再同一棵子树是否会有影响？答案是不影响，只需要自上而下调整受影响那棵子树即可（反正 `last` 的位置是要被删除的）

```golang
// Push pushes the element x onto the heap. The complexity is
// O(log(n)) where n = h.Len().
func (t *Timer) add(td *TimerData) {
	var d itime.Duration
	td.index = len(t.timers)
	// add to the minheap last node
	t.timers = append(t.timers, td)
	t.up(td.index)
	if td.index == 0 {
		// if first node, signal start goroutine
		d = td.Delay()
		t.signal.Reset(d)
		if Debug {
			log.Debug("timer: add reset delay %d ms", int64(d)/int64(itime.Millisecond))
		}
	}
	if Debug {
		log.Debug("timer: push item key: %s, expire: %s, index: %d", td.Key, td.ExpireString(), td.index)
	}
	return
}

func (t *Timer) del(td *TimerData) {
	var (
		i    = td.index
		last = len(t.timers) - 1
	)
	if i <0 || i> last || t.timers[i] != td {
		// already remove, usually by expire
		if Debug {
			log.Debug("timer del i: %d, last: %d, %p", i, last, td)
		}
		return
	}
	if i != last {
		t.swap(i, last)
		t.down(i, last)
		t.up(i)
	}
	// remove item is the last node
	t.timers[last].index = -1 // for safety
	t.timers = t.timers[:last]
	if Debug {
		log.Debug("timer: remove item key: %s, expire: %s, index: %d", td.Key, td.ExpireString(), td.index)
	}
	return
}
```

通过 `up` 和 `down` 方法来调整插入后的堆，保证堆顶元素的超时值始终为最小：
```golang
//  up 方法从下至上调整堆，一般用于插入节点
func (t *Timer) up(j int) {
	for {
		i := (j - 1) / 2 // 取 parent 的位置
		if i >= j || !t.less(j, i) {
            // 两个条件退出：
            //1. 堆顶
            //2. 当前调整的节点 j 相对于 比较节点 i，后超时（注意是!）
			break
        }
        // 否则交换节点（slice 的内容已经修改）
		t.swap(i, j)
		j = i
	}
}

func (t *Timer) down(i, n int) {
    //  down 方法
	for {
		j1 := 2*i + 1
		if j1 >= n || j1 < 0 { // j1 < 0 after int overflow
			break
		}
		j := j1 // left child
		if j2 := j1 + 1; j2 <n && !t.less(j1, j2) {
			j = j2 // = 2*i + 2  // right child
		}
		if !t.less(j, i) {
			break
		}
		t.swap(i, j)
		i = j
	}
}

func (t *Timer) less(i, j int) bool {
    //i 在 j 之前（i 先超时），为 true
    return t.timers[i].expire.Before(t.timers[j].expire)
}

//swap 方法：交换 slice 中两个下标的值（指针）及对象的 index 成员的值
func (t *Timer) swap(i, j int) {
	t.timers[i], t.timers[j] = t.timers[j], t.timers[i]
	t.timers[i].index = i
	t.timers[j].index = j
}
```

####    Timer 定时器的初始化及核心循环
再看下定时器的启动及核心 Loop，在 `init` 方法中，启动了一个 goroutine 来管理所有的 timer。`start` 方法内部是一个 Dead Loop，`expire()` 负责设置一个最近的定时器，然后阻塞等待此定时器触发，这样做的好处 <font color="#dd0000"> 是避免 CPU 空转，提高定时器管理的效率 </font>：

```golang
func (t *Timer) init(num int) {
	t.signal = itime.NewTimer(infiniteDuration)
	t.timers = make([]*TimerData, 0, num)
	t.num = num
	t.grow()
	go t.start() // 独立 goroutine 开启轮询
}

//start 方法
func (t *Timer) start() {
	for {
		t.expire()
		<-t.signal.C
	}
}

// expire 方法
func (t *Timer) expire() {
	var (
		fn func()
		td *TimerData
		d  itime.Duration
	)
	t.lock.Lock()
	for {
		if len(t.timers) == 0 { // 没有定时器
			d = infiniteDuration
			if Debug {
				log.Debug("timer: no other instance")
			}
			break
		}
		td = t.timers[0] // 取第一个（也是堆顶）元素，如何还没到期就根据剩余时间重置定时器
		if d = td.Delay(); d> 0 {
			break
		}
		fn = td.fn
		// let caller put back, usually by Del()
		t.del(td) // 从堆中删除
		t.lock.Unlock()
		if fn == nil {
			log.Warn("expire timer no fn")
		} else {
			if Debug {
				log.Debug("timer key: %s, expire: %s, index: %d expired, call fn", td.Key, td.ExpireString(), td.index)
			}
			fn() // 执行回调
		}
		t.lock.Lock()
	}
	t.signal.Reset(d)   // 更新 signal 为 d 的时间值
	if Debug {
		log.Debug("timer: expire reset delay %d ms", int64(d)/int64(itime.Millisecond))
	}
	t.lock.Unlock()
	return
}
```

####    细节：局部优化
在添加、删除 `TimerData` 的逻辑中，一个细节是使用了 `get`、`put` 方式做了简单的 `TimerData` 节点管理，是为了避免频繁申请内存做的优化。Timer 在初始化时就会构造好一条 free 链表 `free   *TimerData`，在 `Add` 时，先取出 `free` 指向的节点，加入到 `t.timers` 堆中。在 `Del` 时，先从堆中删除，再放回 `free` 指向的链表中。
```golang
func (t *Timer) Add(expire itime.Duration, fn func()) (td *TimerData) {
    t.lock.Lock()
    // 负责在链表中申请节点
	td = t.get()
	td.expire = itime.Now().Add(expire)
    td.fn = fn
    // 在获取到节点（TimerData）后进行堆的调整
	t.add(td)
	t.lock.Unlock()
	return
}

func (t *Timer) Del(td *TimerData) {
	t.lock.Lock()
	t.del(td)
	t.put(td)
	t.lock.Unlock()
	return
}

func (t *Timer) Set(td *TimerData, expire itime.Duration) {
	t.lock.Lock()
	t.del(td)
	td.expire = itime.Now().Add(expire)
	t.add(td)
	t.lock.Unlock()
	return
}
```
`get` 和 `put` 方法的实现如下，只是根据当前的 `free` 指针获得或者放回一个 `TimerData`:
```golang
// get get a free timer data.
func (t *Timer) get() (td *TimerData) {
	if td = t.free; td == nil {
		t.grow()
		td = t.free
	}
	t.free = td.next
	return
}

// put put back a timer data.
func (t *Timer) put(td *TimerData) {
	td.fn = nil
	td.next = t.free
	t.free = td
}
```

##  0x04    Timewheel 实现

##  0x05    其他

##  0x06    参考
-   [Go-Zero 如何应对海量定时延迟任务](https://my.oschina.net/u/4628563/blog/4667586)
-   [论 golang Timer Reset 方法使用的正确姿势](https://tonybai.com/2016/12/21/how-to-use-timer-reset-in-golang-correctly/)
-   [定时任务高效触发](https://coolshell.me/articles/cron-job-trigger-high-efficiency.html)