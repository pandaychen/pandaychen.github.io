---
layout:     post
title:      理解 Kratos 的数据统计类型 Metrics（二）
subtitle:   分析 Kratos 框架中的 Metrics
date:       2020-05-12
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Metrics
---

##  0x00    前言


##  0x01    RollingCounter
RollingCounter就是基于滑动窗口的计数器。

```golang
type rollingCounter struct {
	policy *RollingPolicy
}


type RollingCounter interface {
	Metric
	Aggregation
	Timespan() int
	// Reduce applies the reduction function to all buckets within the window.
	Reduce(func(Iterator) float64) float64
}
```


```golang
func (r *rollingCounter) Add(val int64) {
    //计数器不能为负数
	if val < 0 {
		panic(fmt.Errorf("stat/metric: cannot decrease in value. val: %d", val))
	}
	r.policy.Add(float64(val))
}
```

`r.policy.Add`，回想上一篇博客分析Window 的 `Add` 方法：向指定的偏移 offset 的 `0` 号位置累加值。图示如下：
```golang
// Add adds the given value to the latest point within bucket.
func (r *RollingPolicy) Add(val float64) {
	r.add(r.window.Add, val)
}
```

![image](https://wx1.sbimg.cn/2020/05/08/rollingcounter.png)



##  参考
-   [rolling_gauge的源码](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_gauge.go)
-   [rolling_counter的源码](https://github.com/go-kratos/kratos/blob/master/pkg/stat/metric/rolling_counter.go)