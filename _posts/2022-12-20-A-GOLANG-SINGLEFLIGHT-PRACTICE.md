---
layout: post
title:  Singleflight：原理与应用（续）
subtitle: 如何安全且正确的使用 singleflight
date: 2023-03-25
author: pandaychen
catalog: true
tags:
  - Golang
  - singleflight
---

## 0x00 前言
前文 [微服务中的缓存（一）：Cache 使用与优化](https://pandaychen.github.io/2020/02/22/A-CACHE-STUDY/#singleflight - 机制) 介绍过 singleflight 的应用场景之解决缓存失效时的并发穿透场景。本文就笔者项目中对 Singleflight 机制的实际使用再做一次总结。

![cache-bugs]()


##  0x01  使用场景
singleflight 提供了以下方法，重点考虑 `Do` 和 `DoChan` 方法：

```GOLANG
// Do(): 相同的 key，fn 同时只会执行一次，返回执行的结果给 fn 执行期间，所有使用该 key 的调用
// v: fn 返回的数据
// err: fn 返回的 err
// shared: 表示返回数据是调用 fn 得到的还是其他相同 key 调用返回的
func (g *Group) Do(key string, fn func() (interface{}, error)) (v interface{}, err error, shared bool) {
// DoChan(): 类似 Do，以 chan 返回结果
func (g *Group) DoChan(key string, fn func() (interface{}, error)) <-chan Result {
// Forget(): 失效 key，后续对此 key 的调用将执行 fn，而不是等待前面的调用完成
func (g *Group) Forget(key string)
```

singleflight机制的使用场景有哪些呢？就个人经验而言，满足如下两个场景都可以使用：
-	多goroutine并发（大量并发，并发量过低效果不明显）
-	执行相同的逻辑获取结果，区别仅仅是调用参数不同

####  应用一：DNS 库的应用


##  0x02  负面
SingleFlight 有坑吗？先看一个最普通的例子：
```GOLANG
type Result string

func find(ctx context.Context, query string) (Result, error) {
    //time.Sleep(60*time.Second)
	return Result(fmt.Sprintf("result for %q", query)), nil
}

func main() {
	var g singleflight.Group
	const n = 5
	waited := int32(n)
	done := make(chan struct{})
	key := "some_related_key"
	for i := 0; i < n; i++ {
		go func(j int) {
			v, _, shared := g.Do(key, func() (interface{}, error) {
				ret, err := find(context.Background(), key)
				return ret, err
			})
			if atomic.AddInt32(&waited, -1) == 0 {
				close(done)
			}
			fmt.Printf("index: %d, val: %v, shared: %v\n", j, v, shared)
		}(i)
	}

	select {
	case <-done:
	case <-time.After(time.Second):
		fmt.Println("Do hangs")
	}
}
```


## 0x02 参考
-	[一篇带给你 Go 并发编程 Singleflight](https://www.51cto.com/article/652064.html)
-	[并发编程 -- 用 SingleFlight 合并重复请求](https://zhuanlan.zhihu.com/p/340191872)
- [缓存三坑与 singleflight](https://www.xcssuper.com/post/%E7%BC%93%E5%AD%98%E4%B8%89%E5%9D%91%E4%B8%8Esingleflight/)