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
3.	注册任务按照 **当前刻度 + 延时时长 % 时间轮周期** 计算得出，假设当前指针在 `5s` 的位置，此时添加一个延时周期为 `5s` 的任务，那么该任务需要注册到刻度为 `10s` 的格子对应的任务链表中
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
`timers` 主要用于保存任务（key 为标识）及其在时间轮中的 slot 的 `pos` 位置，方便查找的时候快速定位
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

这里需要着重理解下`timingEntry.diff`字段的意义（见下文分析）

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
		// 异步处理删除任务
		case key := <-tw.removeChannel:
			tw.removeTask(key)
		// 异步处理任务更新操作
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

####	时间轮转动 onTick 及扫描任务链表
每隔`ticker`时间，定时移动时间轮的指针，先保存当前指针的位置，然后从slot中拿出对应的`list.List`，传参list到`scanAndRunTask`方法中执行，如下：
```golang
func (tw *TimingWheel) onTick() {
	tw.tickedPos = (tw.tickedPos + 1) % tw.numSlots
	//获取时间轮（槽）对应的任务链表
	l := tw.slots[tw.tickedPos]
	//扫描当前的任务链表
	tw.scanAndRunTasks(l)
}
```

`scanAndRunTask`方法的步骤如下：
1.	遍历整个list，先清理掉被删掉任务（`task.removed`被置位），再将循环圈数`circle`不为`0`的任务的圈数减去`1`，因为时间流转了刚好`1`个周期
2.	剩下的是`circle`为`0`的有效任务，考虑到有更新操作，若为更新操作（`task.diff`被置位），则将当前任务删除后，根据任务更新的触发时间`task.diff`重新注册到时间轮中
3.	经由前两步过滤，剩下的任务就是scan要执行的任务，把待执行的任务加入到执行待执行队列`tasks`中，通过`tw.runTasks(tasks)`方法并发执行（扫描完list之后并发执行。注意：会控制并发数）

```golang
func (tw *TimingWheel) scanAndRunTasks(l *list.List) {
	var tasks []timingTask

	for e := l.Front(); e != nil; {
		task := e.Value.(*timingEntry)
		if task.removed {
			// 被标记为删除，在scanAndRunTasks中进行清理
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

		// task.circle==0
		// && task.diff == 0 ：说明定时器已经到达触发时间

		tasks = append(tasks, timingTask{
			key:   task.key,
			value: task.value,
		})
		next := e.Next()
		l.Remove(e)
		tw.timers.Del(task.key)
		e = next
	}

	//一轮扫描完之后，并发执行tasks
	tw.runTasks(tasks)
}
```

##	0x04	代码分析（2）：任务操作
任务的添加和删除是必须要实现的，更新可以通过先删除再添加的方式实现。不过go-zero的任务更新实现有些许不一样，此外，在 go-zero 中，任务更新的场景，比如基于 TTL 的缓存的设置 [逻辑](https://github.com/zeromicro/go-zero/blob/master/core/collection/cache.go#L114)，一旦有 key 被设置就刷新 TTL

####	MoveTimer：任务更新
`MoveTimer` 这个方法主要用于动态更新时间轮已存在的 key 及其过期时间，对应的处理方法是 `moveTask`；
```golang
// MoveTimer moves the task with the given key to the given delay.
func (tw *TimingWheel) MoveTimer(key interface{}, delay time.Duration) error {
	if delay <= 0 || key == nil {
		return ErrArgument
	}

	select {
		//异步处理
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


####	更新任务：moveTask
`moveTask`方法是更新timeWheel中已存在的任务的延迟时间。有两种调用场景：
1.	添加任务时，如果任务已经存在，那么只需更新方法（在`setTask`中判断如果有这个key就调用`moveTask`）
2.	通过更新的API接口更新

```golang
func (tw *TimingWheel) moveTask(task baseEntry) {
    // timers: 通过任务的 key 获取 pos 位置及 task
    val, ok := tw.timers.Get(task.key)
    if !ok {
        return
    }

    timer := val.(*positionEntry)
    // {delay < interval} => 延迟时间比一个时间格间隔还小，没有更小的刻度，说明任务应该立即执行
	// 可能是task设置的延迟时间太小了，那就直接执行
    if task.delay < tw.interval {
        threading.GoSafe(func() {
            tw.execute(timer.item.key, timer.item.value)
        })
        return
    }
    // 如果 > interval，则通过 延迟时间 delay 计算其出时间轮中的 new pos, circle
	// 需要重新设置触发时间，即根据新的延迟时间计算出新的定位和circle
    pos, circle := tw.getPositionAndCircle(task.delay)
	//根据pos和circle还有旧数据，修改task的信息，做一些标记，在扫描到这个task的时候再真正修改和重新定位
	//新/旧任务的pos不一样，需要移动，但是由于并发问题，不会在这里移动
    if pos >= timer.pos {
        timer.item.circle = circle
        // 记录前后的移动 offset，为了后面过程重新入队（先提前计算好位置）
        timer.item.diff = pos - timer.pos			//diff的值分两种情况，diff为0,说明不需要移动；非0才需要移动
    } else if circle > 0 {
		 //说明pos（旧） < timer.pos（新）且剩余圈数大于0，即任务触发的时间提前了，但是不会在这一圈触发，需要计算一下diff偏移量和走多少圈
        // 先把该任务转移到下一层（即circle减一），将 circle 转换为 diff 的一部分
        circle--
		//更新circle
        timer.item.circle = circle
        // 因为是一个数组，要加上 numSlots [也就是相当于要走到下一层]
        timer.item.diff = tw.numSlots + pos - timer.pos	//注意这里
    } else {
        // 如果 offset 提前了，此时 task 也还在第一层
        // 标记删除老的 task，并重新入队，等待被执行

		//pos（旧） < timer.pos（新），且circle==0，说明是在本链表中执行任务，这里删除旧的添加新的（其实是一样）       
        timer.item.removed = true	//标记当前节点待删除
        newItem := &timingEntry{
            baseEntry: task,
            value:     timer.item.value,
        }
		//重新加到任务链表的尾部
        tw.slots[pos].PushBack(newItem)
        tw.setTimerPosition(pos, newItem)
    }
}
```

关于`moveTask`的这部分实现，有个问题是为何不在这里直接就移动节点呢？个人观点是基于效率的考虑，注意到`moveTask`方法本质上还是对任务节点的操作，并且任务节点是散落在各个slot的任务链表中的。如果在`moveTask`中去移动，需要遍历找到相应的slot节点，然后遍历链表，找到对应的节点进行操作（因为需要找到该节点对应于链表中的后节点），因此，这里的逻辑仅仅是设置标志（如`item.diff`），对节点的处理放在扫描任务链表的方法`scanAndRunTasks`中实现。

从另一个角度看，`MoveTask`的实现中，着重突出了**延迟操作**，即只在扫描链表的时候才处理更新/删除节点等操作，其好处是如果某些任务key频繁改动，无需频繁进行重新定位操作（在时间轮中重新定位），而重新定位操作需要保证并发安全，引入了复杂度

```golang
if pos >= timer.pos {
    timer.item.circle = circle
	// 记录前后的移动 offset，为了后面过程重新入队（先提前计算好位置）
	timer.item.diff = pos - timer.pos			//diff的值分两种情况，diff为0,说明不需要移动；非0才需要移动
} else if circle > 0 {
		//说明pos（旧） < timer.pos（新）且剩余圈数大于0，即任务触发的时间提前了，但是不会在这一圈触发，需要计算一下diff偏移量和走多少圈
	// 先把该任务转移到下一层（即circle减一），将 circle 转换为 diff 的一部分
	circle--
	//更新circle
	timer.item.circle = circle
	// 因为是一个数组，要加上 numSlots [也就是相当于要走到下一层]
	timer.item.diff = tw.numSlots + pos - timer.pos	//注意这里
} 
```


####	增加任务setTask
任务是通过异步方式增加到时间轮的，主要逻辑如下代码所示：
1.	从`timers`即map中查询此任务是否已经存在，若存在，则调用`moveTask`更新任务（主要是执行触发时间）
2.	若任务不存在，会通过`getPositionAndCircle`方法计算出任务在时间轮中相对于当前的ticked的定位`pos`，以及要转的圈数`circle`，将任务放在时间轮槽对应的任务链表队列上，并且维护`timers`的map索引，方便查询任务定位

```golang
func (tw *TimingWheel) setTask(task *timingEntry) {
    if task.delay < tw.interval {
        task.delay = tw.interval
    }

    if val, ok := tw.timers.Get(task.key); ok {
        entry := val.(*positionEntry)
        entry.item.value = task.value
        tw.moveTask(task.baseEntry)
    } else {
        pos, circle := tw.getPositionAndCircle(task.delay)
        task.circle = circle
        tw.slots[pos].PushBack(task)	//向时间轮的list插入任务，注意使用尾插法
        tw.setTimerPosition(pos, task)	//更新timers map
    }
}
```

####	删除任务removeTask
删除任务是通过索引找到这个task，然后把task标记为删除，即置位`timer.item.removed`，然后再每一轮扫描到list的`scanAndRunTask`方法中再做清理，即直接跳过该任务：
```golang
func (tw *TimingWheel) removeTask(key interface{}) {
    val, ok := tw.timers.Get(key)
    if !ok {
        return
    }

    timer := val.(*positionEntry)
    timer.item.removed = true
    tw.timers.Del(key)
}
```


##	0x05	一些细节问题
1、ticker丢失<br>
注意到核心scheduler的各个`case`，在另外一个版本的[实现](https://github.com/ouqiang/timewheel/blob/master/timewheel.go#L96)如下：
```golang
func (tw *TimeWheel) start() {
	for {
		select {
		case <-tw.ticker.C:
			tw.tickHandler()
		case task := <-tw.addTaskChannel:
			tw.addTask(&task)
		case key := <-tw.removeTaskChannel:
			tw.removeTask(key)
		case <-tw.stopChannel:
			tw.ticker.Stop()
			return
		}
	}
}

func (tw *TimeWheel) tickHandler() {
	l := tw.slots[tw.currentPos]
	tw.scanAndRunTask(l)
	if tw.currentPos == tw.slotNum-1 {
		tw.currentPos = 0
	} else {
		tw.currentPos++
	}
}
```

假设`tickHandler`的执行耗时超过了一个`tw.ticker.C`周期，那么就会导致时间轮精度不准，这里如何优化的思路是，借助一个`tickQueue chan time.Time`，将`<-tw.ticker.C`的结果进行存储，当`tickQueue`满时抛出异常，说明当前时间轮的任务执行有问题（延迟任务），优化代码如下：


```go
//增加一个异步的ticker缓冲器
func (tw *TimeWheel) tickGenerator() {
	if tw.tickQueue == nil {
		return
	}

	for  {
		select {
		case <-tw.ticker.C:
			select {
			case tw.tickQueue <- time.Now():
			default:
				panic("raise long time blocking")
			}
		}
	}
}


func (tw *TimeWheel) start() {
	for {
		select {
		case <-tw.tickQueue:	//替换为从缓冲器触发
			tw.tickHandler()
		case task := <-tw.addTaskChannel:
			tw.addTask(&task)
		case key := <-tw.removeTaskChannel:
			tw.removeTask(key)
		case <-tw.stopChannel:
			tw.ticker.Stop()
			return
		}
	}
}
```

2、时间轮本身的任务执行应该是**异步的（回调任务的执行不应该堵塞）**，可以考虑将回调任务使用协程池的方式进行调度，或者结合一些异步队列中间件，将到期的任务进行异步化处理

3、思考这个问题：**时间轮（内存）的任务如何持久化呢？比如重启之后，时间轮的任务会丢失，如何才能恢复到重启前的状态？**

##  0x06  总结
本文分析了一款典型的简单时间轮的实现，通过给任务节点添加 `circle` 字段来解决一维时间轮无法扩展时间的问题，从而突破长时间的限制。可以借鉴的地方有如下：
1.	任务的删除、更新操作，都仅仅通过标记的方式延迟进行，避免并发的加锁问题，仅在方法`scanAndRunTasks`中实现
2.	外部操作接口，如任务的增删改，也是通过异步的方式实现
3.	在时间轮的scheduler核心方法`TimingWheel.run`中，要注意每个`case`条件下的逻辑运行时间，如果运行时间过长会导致其他的`case`条件不能及时得到运行

![go-zero-timewheel](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/timewheel/go-zero-timewheel-1.jpg)

##  0x07  参考
-   [go-zero 如何应对海量定时 / 延迟任务？](https://segmentfault.com/a/1190000037496480)
-	[层级时间轮的 Golang 实现](http://russellluo.com/2018/10/golang-implementation-of-hierarchical-timing-wheels.html)