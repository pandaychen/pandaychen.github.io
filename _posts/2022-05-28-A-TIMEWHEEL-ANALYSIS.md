---
layout: post
title: 数据结构与算法回顾（三）：时间轮
subtitle: 一种高效的定时器算法实现（简单时间轮）
date: 2022-05-28
header-img: img/super-mario.jpg
author: pandaychen
catalog: true
tags:
  - 数据结构
  - timewheel
---

##  0x00    前言
时间轮是用来解决海量百万级定时器（或延时）任务的最佳方案，linux 的内核定时器就是采用该数据结构实现。本文介绍 go-zero 框架中时间轮的实现及使用场景。

####	应用场景
1.	自动删除缓存中过期的 Key：缓存中设置了 TTL 的 kv，通过把该 key 对应的 TTL 以及回调方法注册到 timewheel，到期直接删除
2.	延时任务，将任务注册到 timewheel，过期自动触发执行
3.  在 TcpServer 中，用来管理海量 Tcp 连接的超时定时器，如 zinx 的定时器 [实现](https://github.com/aceld/zinx/blob/master/ztimer/timewheel.go)

##  0x01    时间轮基础

####	简单时间轮

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/timewheel/3-timewheel.png)

如上图，一个普通的时间轮，类似于时钟表盘，指针（pointer）每隔一段时间前进一格（interval，tick 一次），一圈代表一个周期（circle），定时任务以链表（双向）方式置放在表盘的刻度处，当指针前进到当前位置时，遍历任务链表，执行相应的任务。

从开发角度而言，实现一个时间轮：
1.	时间轮是一个由固定长度 `length` 的数组（本例子中就是 `[1,12]`）构造而成的环形队列
2.	时间轮的长度决定了延时任务的刻度，假设上面的刻度为 `1s`（即时间轮 `1s` 前进一格），那么该时间轮只能表达延时任务在 `1s` 至 `12s` 内的任务；时间轮的长度也即时间轮的周期（`12s`）
3.	注册任务按照 ** 当前刻度 + 延时时长 % 时间轮周期 ** 计算得出，假设当前指针在 `5s` 的位置，此时添加一个延时周期为 `5s` 的任务，那么该任务需要注册到刻度为 `10s` 的格子对应的任务链表中
4.	数组中的每个元素都指向一个双向链表，用于存储对应的延时任务
5.	时间轮的插入复杂度是 `O(1)`，删除指定节点的复杂度是 `O(n)`，因为需要遍历双向链表以查找到要删除的节点
6.	当时间轮指针转动到对应的单元格时，顺序执行双向链表中存储的任务

基础时间轮的缺点是无法注册延时超过时间轮周期的任务，如何解决呢？

####	解决方法 1：加 circle 计数器
此方法相当于给双向链表中存储的任务加多一个 "圈数" 的维度，如某任务需要 `30s` 后执行，当前指针刻度在 `1s`，那么该任务的圈数就是 `2`，放在第 `6` 格中，即时间轮转 `2` 圈加 `6` 个格子后，触发此任务；

注意，此策略中，时间轮指针每前进一格，需要把此格对应的任务链表中，所有的任务的 circle 计数器都减 `1`，如果 `circle==0`，那么说明，任务时间已到期，执行该任务

![circle](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/timewheel/simple-timewheel-with-circle-counter.jpg)

####	解决方法 2：层级时间轮
这是一种典型的 "空间换时间" 的思路，按照时间轮周期的倍数进行合理分层，有两个优点：
1.  避免任务堆积在某个 slot 上
2.  支持任意长时间的延时任务注册

![hierarchical](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/timewheel/hierarchical-timewheel-2.jpg)

##	0x02	go-zero 的时间轮
时间轮的实现大同小异，这里选取 go-zero 的实现做简单分析。先抽象中时间轮的核心数据结构及方法：

```golang
type TimingWheel struct {
	//...
	interval      time.Duration	// 时间轮 ticker 时间
	slots         []*list.List	// 模拟时间轮环形结构，加任务存储
	timers        *SafeMap		// 用于存储
	tickedPos     int			// 记录当前指针所在的位置
	numSlots      int		// 时间轮的槽位数量
	//...
}
```

![wheel](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/datastructure/ds-timewheel-1.png)


####	时间轮
1、ticker<br>
用于时间轮的转动，同时更新 `tw.tickedPos` 的值
```golang
func (tw *TimingWheel) onTick() {
	tw.tickedPos = (tw.tickedPos + 1) % tw.numSlots
	l := tw.slots[tw.tickedPos]
	tw.scanAndRunTasks(l)
}
```

2、getPositionAndCircle<br>
用于根据传入参数 `d time.Duration`，计算出，这个 `d` 对应的任务该放在时间轮的哪个 slot 里面，即 `pos` 值；同时，假设 `d` 已经超过一个时间轮的范围了，计算其对应的转动圈数 `circle` 值

```golang
func (tw *TimingWheel) getPositionAndCircle(d time.Duration) (pos, circle int) {
	steps := int(d / tw.interval)
	pos = (tw.tickedPos + steps) % tw.numSlots
	circle = (steps - 1) / tw.numSlots

	return
}
```
3、`timers`：Map 的作用 <br>
`timers` 主要用于保存任务（key 为标识）及其在时间轮中的 slot 的 pos 位置，方便查找的时候快速定位
```golang
func (tw *TimingWheel) setTimerPosition(pos int, task *timingEntry) {
	if val, ok := tw.timers.Get(task.key); ok {
		timer := val.(*positionEntry)
		timer.item = task
		timer.pos = pos
	} else {
		// 保存位置 pos 和任务
		tw.timers.Set(task.key, &positionEntry{
			pos:  pos,
			item: task,
		})
	}
}
```

####	任务结构
```golang
type timingEntry struct {
	baseEntry
	value   interface{}
	circle  int		// 记住这个字段：用以解决分层的问题
	diff    int
	removed bool
}
```

##	0x03	代码分析（1）
本小节分析下 [timingwheel](https://github.com/zeromicro/go-zero/blob/master/core/collection/timingwheel.go) 的实现。

####	结构体
时间轮的 [定义](https://github.com/zeromicro/go-zero/blob/master/core/collection/timingwheel.go#L26) 如下，不难看出，`TimingWheel` 中的 `channel` 把时间轮的添加 / 删除操作做成异步的，避免加锁带来的复杂度：

```golang
// A TimingWheel is a timing wheel object to schedule tasks.
type TimingWheel struct {
	interval      time.Duration  // 单个时间格时间间隔
	ticker        timex.Ticker	// 定时器，做时间推动，以 interval 为单位推进
	slots         []*list.List	// 时间轮
	timers        *SafeMap	// 存储 task{key, value} 的 map [执行 execute 所需要的参数]
	tickedPos     int	 // at previous virtual circle
	numSlots      int	// 初始化 slots num
	execute       Execute	 // 执行函数
	setChannel    chan timingEntry
	moveChannel   chan baseEntry
	removeChannel chan interface{}
	drainChannel  chan func(key, value interface{})
	stopChannel   chan lang.PlaceholderType
}
```

####	初始化时间轮
```golang
// 真正做初始化
func newTimingWheelWithClock(interval time.Duration, numSlots int, execute Execute, ticker timex.Ticker) (
    *TimingWheel, error) {
	//...
    // 初始化 slots 中用来存储任务的所有 slots
    tw.initSlots()
    // start ticker
    go tw.run()

    return tw, nil
}
```

异步开启的 `run` 方法，本质上是一个基于 `for...select` 模式的 scheduler：
```golang
func (tw *TimingWheel) run() {
	for {
		select {
		// 定时器 ticker ，时间推动
		case <-tw.ticker.Chan():
			tw.onTick()
		// 异步处理增加任务
		case task := <-tw.setChannel:
			tw.setTask(&task)
		case key := <-tw.removeChannel:
			tw.removeTask(key)
		case task := <-tw.moveChannel:
			tw.moveTask(task)
		case fn := <-tw.drainChannel:
			tw.drainAll(fn)
		case <-tw.stopChannel:
			tw.ticker.Stop()
			return
		}
	}
}
```

####	时间轮转动 onTick

```golang
func (tw *TimingWheel) onTick() {
	tw.tickedPos = (tw.tickedPos + 1) % tw.numSlots
	l := tw.slots[tw.tickedPos]
	tw.scanAndRunTasks(l)
}

func (tw *TimingWheel) scanAndRunTasks(l *list.List) {
	var tasks []timingTask

	for e := l.Front(); e != nil; {
		task := e.Value.(*timingEntry)
		if task.removed {
			next := e.Next()
			l.Remove(e)
			e = next
			continue
		} else if task.circle > 0 {
			task.circle--
			e = e.Next()
			continue
		} else if task.diff > 0 {
			next := e.Next()
			l.Remove(e)
			// (tw.tickedPos+task.diff)%tw.numSlots
			// cannot be the same value of tw.tickedPos
			pos := (tw.tickedPos + task.diff) % tw.numSlots
			tw.slots[pos].PushBack(task)
			tw.setTimerPosition(pos, task)
			task.diff = 0
			e = next
			continue
		}

		tasks = append(tasks, timingTask{
			key:   task.key,
			value: task.value,
		})
		next := e.Next()
		l.Remove(e)
		tw.timers.Del(task.key)
		e = next
	}

	tw.runTasks(tasks)
}
```

##	0x04	代码分析（2）：任务操作
任务的添加和删除是必须要实现的，更新可以通过先删除再添加的方式实现。

####	MoveTimer：任务更新
`MoveTimer` 这个方法主要用于动态更新时间轮已存在的 key 及其过期时间，对应的处理方法是 `moveTask`；
```golang
// MoveTimer moves the task with the given key to the given delay.
func (tw *TimingWheel) MoveTimer(key interface{}, delay time.Duration) error {
	if delay <= 0 || key == nil {
		return ErrArgument
	}

	select {
	case tw.moveChannel <- baseEntry{
		delay: delay,
		key:   key,
	}:
		return nil
	case <-tw.stopChannel:
		return ErrClosed
	}
}
```

在 go-zero 中，任务更新的场景，比如基于 TTL 的缓存的设置 [逻辑](https://github.com/zeromicro/go-zero/blob/master/core/collection/cache.go#L114)，一旦有 key 被设置就刷新 TTL

####	moveTask
```golang
func (tw *TimingWheel) moveTask(task baseEntry) {
    // timers: 通过任务的 key 获取 pos 位置及 task
    val, ok := tw.timers.Get(task.key)
    if !ok {
        return
    }

    timer := val.(*positionEntry)
      // {delay < interval} => 延迟时间比一个时间格间隔还小，没有更小的刻度，说明任务应该立即执行
    if task.delay < tw.interval {
        threading.GoSafe(func() {
            tw.execute(timer.item.key, timer.item.value)
        })
        return
    }
    // 如果 > interval，则通过 延迟时间 delay 计算其出时间轮中的 new pos, circle
    pos, circle := tw.getPositionAndCircle(task.delay)
    if pos >= timer.pos {
        timer.item.circle = circle
        // 记录前后的移动 offset。为了后面过程重新入队
        timer.item.diff = pos - timer.pos
    } else if circle > 0 {
        // 转移到下一层，将 circle 转换为 diff 一部分
        circle--
        timer.item.circle = circle
        // 因为是一个数组，要加上 numSlots [也就是相当于要走到下一层]
        timer.item.diff = tw.numSlots + pos - timer.pos
    } else {
        // 如果 offset 提前了，此时 task 也还在第一层
        // 标记删除老的 task，并重新入队，等待被执行
        timer.item.removed = true
        newItem := &timingEntry{
            baseEntry: task,
            value:     timer.item.value,
        }
        tw.slots[pos].PushBack(newItem)
        tw.setTimerPosition(pos, newItem)
    }
}
```



##  0x05  总结
本文分析了一款典型的简单时间轮的实现，通过给任务节点添加 `circle` 字段来解决一维时间轮无法扩展时间的问题，从而突破长时间的限制。

##  0x06  参考
-   [go-zero 如何应对海量定时 / 延迟任务？](https://segmentfault.com/a/1190000037496480)
-	[层级时间轮的 Golang 实现](http://russellluo.com/2018/10/golang-implementation-of-hierarchical-timing-wheels.html)