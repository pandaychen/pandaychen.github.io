---
layout: post
title: Golang CRON 库 Crontab 的使用与设计
subtitle: 基于 golang 的定时任务模型分析
date: 2021-10-05
author: pandaychen
catalog: true
header-img: img/about-bg.jpg
tags:
  - Crontab
  - Golang
---

## 0x00 前言

## 0x01 使用

#### crontab

crontab 基本格式：

```bash
# 文件格式說明
# ┌──分钟（0 - 59）
# │  ┌──小时（0 - 23）
# │  │  ┌──日（1 - 31）
# │  │  │  ┌─月（1 - 12）
# │  │  │  │  ┌─星期（0 - 6，表示从周日到周六）
# │  │  │  │  │
# *  *  *  *  * 被执行的命令
```

#### 基础例子

用法极丰富，V3 版本也支持标准的 `crontab` 格式，可以参考 [此文](https://segmentfault.com/a/1190000023029219)：<br>

```golang
func main() {
    job := cron.New(
        cron.WithSeconds(), // 添加秒级别支持，默认支持最小粒度为分钟（如需秒级精度则必须设置）
    )
    // 每秒钟执行一次
    job.AddFunc("* * * * * *", func() {
        fmt.Printf("task run: %v\n", time.Now())
    })
    job.Run()   // 启动
}
```

其他典型的用法还有如下：

```golang
type cronJobDemo int

func (c cronJobDemo) Run() {
        fmt.Println("5s func trigger")
        return
}

func main() {
    c := cron.New(
            cron.WithSeconds(),
    )
    c.AddFunc("0 * * * *", func() { fmt.Println("Every hour on the half hour") })
    c.AddFunc("30 3-6,20-23 * * *", func() { fmt.Println(".. in the range 3-6am, 8-11pm") })
    c.AddFunc("CRON_TZ=Asia/Tokyo 30 04 * * *", func() { fmt.Println("Runs at 04:30 Tokyo time every day") })
    c.AddFunc("@every 5m", func() { fmt.Println("every 5m, start 5m fron now") }) // 容易理解的格式
    // 通过 AddJob 注册
    var cJob cronJobDemo
    c.AddJob("@every 5s", cJob)
    c.Start()
    // c.Stop()

    select {}
}
```

## 0x02 代码分析

#### 核心数据结构

对于 Cron 的整体逻辑，最关键的两个数据结构就是 `Entry` 和 `Cron`：<br>

1、Job：抽象一个定时任务，cron 调度一个 Job，就去执行 Job 的 `Run()` 方法 < br>

```golang
type Job interface {
    Run()
}
```

1、Schedule：描述一个 job 如何循环执行的抽象

```golang
// Schedule describes a job's duty cycle.
type Schedule interface {
	// Next returns the next activation time, later than the given time.
	// Next is invoked initially, and then each time the job is run.
	Next(time.Time) time.Time
}
```

scheduler 的实例化结构有：

- `ConstantDelaySchedule`:https://github.com/robfig/cron/blob/v3/constantdelay.go

2、Entry 结构：抽象了一个 job<br>
每当使用 `AddJob` 注册一个定时调用策略，就会为该策略生成一个唯一的 `Entry`，Entry 里会存储被执行的时间、需要被调度执行的实体 Job

```golang
type Entry struct {
    ID EntryID          // job id，可以通过该 id 来删除 job
    Schedule Schedule   // 用于计算 job 下次的执行时间
    Next time.Time      // job 下次执行时间
    Prev time.Time      // job 上次执行时间，没执行过为 0
    WrappedJob Job      // 修饰器加工过的 job
    Job Job             // 未经修饰的 job，可以理解为 AddFunc 的第二个参数
}
```

3、`Cron`[结构](https://github.com/robfig/cron/blob/v3/cron.go#L13)：<br>
关于 `Cron` 结构，有一些细节，`entries` 为何设计为一个指针 `slice`？

```golang
// Cron keeps track of any number of entries, invoking the associated func as
// specified by the schedule. It may be started, stopped, and the entries may
// be inspected while running.
type Cron struct {
    entries   []*Entry          // 所有 Job 集合
    chain     Chain             // 装饰器链
    stop      chan struct{}     // 停止信号
    add       chan *Entry       // 用于异步增加 Entry
    remove    chan EntryID      //  用于异步删除 Entry
    snapshot  chan chan []Entry
    running   bool              // 是否正在运行
    logger    Logger
    runningMu sync.Mutex        // 运行时锁
    location  *time.Location    // 时区相关
    parser    Parser            // Cron 解析器
    nextID    EntryID
    jobWaiter sync.WaitGroup    // 并发控制，正在运行的 Job
}
```

#### entries 成员

刚才说到 `entries` 为何设计为指针 `slice`，原因在于 cron 核心逻辑中，每次循环开始时都会对 `Cron.entries` 进行排序，排序字段依赖于每个 `Entry` 结构的 `Next` 成员，排序依赖于下面的原则：
1.  按照触发时间正向排序，越先触发的越靠前
2.  `IsZero` 的任务向后面排
3.  由于可能存在相同周期的任务 Job，所以排序是不稳定的

```golang
// byTime is a wrapper for sorting the entry array by time
// (with zero time at the end).
type byTime []*Entry

func (s byTime) Len() int      { return len(s) }
func (s byTime) Swap(i, j int) { s[i], s[j] = s[j], s[i] }
func (s byTime) Less(i, j int) bool {
	// Two zero times should return false.
	// Otherwise, zero is "greater" than any other time.
	// (To sort it at the end of the list.)
	if s[i].Next.IsZero() {
		return false
	}
	if s[j].Next.IsZero() {
		return true
	}
    // 排序的原则，s[i] 比 s[j] 先触发
	return s[i].Next.Before(s[j].Next)
}
```

4、`Chain` 结构 <br>

## 核心方法

#### AddJob 方法

```golang
// AddJob adds a Job to the Cron to be run on the given schedule.
// The spec is parsed using the time zone of this Cron instance as the default.
// An opaque ID is returned that can be used to later remove it.
func (c *Cron) AddJob(spec string, cmd Job) (EntryID, error) {
	schedule, err := c.parser.Parse(spec)
	if err != nil {
		return 0, err
	}
	return c.Schedule(schedule, cmd), nil
}

// Schedule adds a Job to the Cron to be run on the given schedule.
// The job is wrapped with the configured Chain.
func (c *Cron) Schedule(schedule Schedule, cmd Job) EntryID {
	c.runningMu.Lock()
	defer c.runningMu.Unlock()
	c.nextID++
	entry := &Entry{
		ID:         c.nextID,
		Schedule:   schedule,
		WrappedJob: c.chain.Then(cmd),
		Job:        cmd,
	}
	if !c.running {
        //
		c.entries = append(c.entries, entry)
	} else {
        //
		c.add <- entry
	}
	return entry.ID
}
```

#### run 方法

cron 的核心 `run` 方法的实现如下，这个是很经典的 `for-select` 异步处理模型，避免的对 `entries` 加锁：


```golang
func (c *Cron) run() {
    c.logger.Info("start")

    // 初始化，计算每个 Job 下次的执行时间
    now := c.now()
    for _, entry := range c.entries {
        entry.Next = entry.Schedule.Next(now)
        c.logger.Info("schedule", "now", now, "entry", entry.ID, "next", entry.Next)
    }

    // 在 dead loop，进行任务调度
    for {
        // 根据下一次的执行时间，对所有 Job 排序
        sort.Sort(byTime(c.entries))

        // 计时器，用于没有任务可调度时的阻塞操作
        var timer *time.Timer
        if len(c.entries) == 0 || c.entries[0].Next.IsZero() {
            // 无任务可调度，设置计时器到一个很大的值，把下面的 for 阻塞住
            timer = time.NewTimer(100000 * time.Hour)
        } else {
            // 有任务可调度了，计时器根据第一个可调度任务的下次执行时间设置
            // 排过序，所以第一个肯定是最先被执行的
            timer = time.NewTimer(c.entries[0].Next.Sub(now))
        }

        for {
            select {
            // 有 Job 到了执行时间
            case now = <-timer.C:
                now = now.In(c.location)
                c.logger.Info("wake", "now", now)
                // 检查所有 Job，执行到时的任务
                for _, e := range c.entries {
                    // 可能存在相同时间出发的任务
                    if e.Next.After(now) || e.Next.IsZero() {
                        break
                    }
                    // 执行 Job 的 func()
                    c.startJob(e.WrappedJob)

                    // 保存上次执行时间
                    e.Prev = e.Next
                    // 设置 Job 下次的执行时间
                    e.Next = e.Schedule.Next(now)
                    c.logger.Info("run", "now", now, "entry", e.ID, "next", e.Next)
                }

            // 添加新 Job
            case newEntry := <-c.add:
                timer.Stop()        // 必须注意，这里停止定时器，避免内存泄漏！
                now = c.now()
                newEntry.Next = newEntry.Schedule.Next(now)
                c.entries = append(c.entries, newEntry)
                c.logger.Info("added", "now", now, "entry", newEntry.ID, "next", newEntry.Next)

            // 获取所有 Job 的快照
            case replyChan := <-c.snapshot:
                replyChan <- c.entrySnapshot()
                continue

            // 停止调度
            case <-c.stop:
                timer.Stop()
                c.logger.Info("stop")
                return

            // 根据 entryId 删除一个 Job
            case id := <-c.remove:
                timer.Stop()
                now = c.now()
                c.removeEntry(id)
                c.logger.Info("removed", "entry", id)
            }

            break
        }
    }
}
```

![image](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/2022/dcrontab/crontab-core-event-loop.png)

## 参考

- [golang cron v3 定时任务](https://blog.cugxuan.cn/2020/06/04/Go/golang-cron-v3/)
- [v3-repo](https://github.com/robfig/cron/tree/v3)
- [Go 每日一库之 cron](https://segmentfault.com/a/1190000023029219)
- [GO 编程模式：修饰器](https://coolshell.cn/articles/17929.html)
