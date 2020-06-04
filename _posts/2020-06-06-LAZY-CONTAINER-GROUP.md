---
layout:     post
title:      Kratos 源码分析（1）：Lazy Load Container
subtitle:   分析 Kratos 的懒对象容器及应用场景
date:       2020-06-06
author:     pandaychen
catalog:    true
tags:
    - Kratos
---


##  0x00 前言
&emsp;&emsp; 所谓懒加载（Lazy）就是延时加载，延迟加载。至于为什么要用懒加载呢，就是当我们要访问的数据量过大时，明显用缓存不太合适，因为内存容量有限，为了减少并发量及系统资源的消耗，我们让数据在需要的时候才进行加载。这就是懒加载。

在数据、策略等需要 "分组（组中以 Name 区分）" 的情况下，非常适合使用该方式实现存储（取），这里存储的值以 interface{} 表示，可以是一个方法，或是一个结构体等；

##  0x01    核心代码
Group 定义如下，注意 Group 的成员里 `new` 是一个方法成员，该方法的返回值为 `interface{}`：
```golang
// Package group provides a sample lazy load container.
// The group only creating a new object not until the object is needed by user.
// And it will cache all the objects to reduce the creation of object.

// Group is a lazy load container.
type Group struct {
	new  func() interface{} // 存储的对象
	objs sync.Map           //syncMAP
	sync.RWMutex
}
```

Group 只提供了如下对外的方法：
1.  `NewGroup` 方法：初始化一个 Group，传入方法参数 `new func() interface{}`
2.  `Get` 方法：以 `key` 在 `sync.Map` 中搜索，如果不存在则创建并存储，创建调用的方法就是 `NewGroup` 中传入的参数 `new`
3.  `Reset` 方法：传入方法参数 `new func() interface{}`，更新 `new` 成员
4.  `Clear` 方法，遍历 `objs sync.Map`，删除所有的 key

```golang
// NewGroup news a group container.
func NewGroup(new func() interface{}) *Group {
	if new == nil {
		panic("container.group: can't assign a nil to the new function")
	}
	return &Group{
		new: new, //
	}
}

// Get gets the object by the given key.
// 以 key 在 sync.Map 中搜索，如果不存在则创建
func (g *Group) Get(key string) interface{} {
	g.RLock()
	new := g.new
	g.RUnlock()
	obj, ok := g.objs.Load(key) //查询key
	if !ok {
		obj = new()
		g.objs.Store(key, obj)
	}
	return obj
}

// Reset resets the new function and deletes all existing objects.
func (g *Group) Reset(new func() interface{}) {
	if new == nil {
		panic("container.group: can't assign a nil to the new function")
	}
	g.Lock()
	g.new = new
	g.Unlock()
	g.Clear()
}

// Clear deletes all objects.
func (g *Group) Clear() {
	g.objs.Range(func(key, value interface{}) bool {
		g.objs.Delete(key)
		return true
	})
}
```

##  0x02    Group 应用

##  0x03    参考
-   [Group 的实现代码](https://github.com/go-kratos/kratos/blob/master/pkg/container/group/group.go)