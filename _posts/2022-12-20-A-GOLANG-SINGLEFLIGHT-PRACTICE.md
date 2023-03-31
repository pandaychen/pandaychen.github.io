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
前文 [微服务中的缓存（一）：Cache 使用与优化](https://pandaychen.github.io/2020/02/22/A-CACHE-STUDY/#singleflight - 机制) 介绍过 singleflight 的应用场景之解决缓存失效时的并发穿透场景。本文就笔者项目中对 Singleflight 机制的实际使用再做一次总结。先回顾下 singleflight 的定义：

SingleFlight 是 Go 开发组提供的一个扩展并发原语。它的作用是在处理多个 goroutine 同时调用同一个函数的时候，只让一个 goroutine 去调用这个函数，等到这个 goroutine 返回结果的时候，再把结果返回给这几个同时调用的 goroutine，这样可以减少并发调用的数量。SingleFlight 主要用在合并并发请求的场景中，尤其是缓存场景。


![cache-bugs](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/cache/singleflight-3.png)



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

singleflight 机制的使用场景有哪些呢？就个人经验而言，满足如下两个场景都可以使用：
-	多 goroutine 并发（大量并发，并发量过低效果不明显）
-	执行相同的逻辑获取结果，区别仅仅是调用参数不同

####  应用一：DNS 库的应用
介绍下 [net/lookup.go](https://github.com/golang/go/blob/master/src/net/lookup.go#L323) 中的机制：



##  0x02  负面

####	问题
SingleFlight 有坑吗？先看下面的例子，模拟了 singleflight 中定义的方法阻塞的场景（实际中肯定会遇到这种情况：如果函数执行遇到问题被阻塞了），由于 singleflight `Do` 方法是以阻塞的方式读取下游请求的并发量，在第一个下游请求没返回之前，所有的请求都会被阻塞。假设并发量较大的情况下，如果某一个下游并发请求出现了阻塞，那么可能会出现下述问题：
1.	goroutines 暴增
2.	内存暴涨，主要是 `1` 引发的
3.	后续同类请求都会阻塞在 singleflight 的阻塞等待完成上，所以后续请求耗时必然会增加

```GOLANG
type Result string

func find(ctx context.Context, query string) (Result, error) {
	time.Sleep(30*time.Second)
	return "", errors.New("bad result")
}

func main() {
	var g singleflight.Group
	const n = 10
	waited := int32(n)
	done := make(chan struct{})
	key := "some_related_key"
	for i := 0; i < n; i++ {
		go func(j int) {
			v, err , shared := g.Do(key, func() (interface{}, error) {
				ret, err := find(context.Background(), key)
				return ret, err
			})
			if atomic.AddInt32(&waited, -1) == 0 {
				close(done)
			}
			fmt.Printf("index: %d, val: %v, shared: %v error:%v\n", j, v, shared,err)
		}(i)
	}

	select {
	case <-done:
	case <-time.After(120*time.Second):
		fmt.Println("Do hangs")
	}
}
```

####	解决方案（1）：一个阻塞，全员等待
上述问题的根本原因是阻塞读导致缺少超时控制，难以快速失败；解决方式，就是在对阻塞过久的 singleflight 请求进行超时控制，刚好它提供了 `DoChan` 方法，`DoChan` 通过 channel 返回结果，可使用 `select` 实现超时控制，如下：

```GOLANG
func find(ctx context.Context, query string) (Result, error) {
        time.Sleep(50 * time.Second)
        return "", errors.New("bad result")
}

func main() {
        var (
                g       singleflight.Group
                timeout = time.After(5000 * time.Millisecond)
        )
        const n = 10
        key := "some_related_key"
        for i := 0; i < n; i++ {
                go func(j int) {
                        ch := g.DoChan(key, func() (interface{}, error) {
                                ret, err := find(context.Background(), key)
                                return ret, err
                        })
                        var ret singleflight.Result
                        select {
                        case <-timeout: // Timeout elapsed
                                fmt.Println("Timeout")
                                return
                        case ret = <-ch: // Received result from channel
                                fmt.Printf("index: %d, val: %v, shared: %v\n", j, ret.Val, ret.Shared)
                        }
                }(i)
        }
        fmt.Println("done")
        select {
        case <-time.After(100 * time.Second):
                fmt.Println("Do hangs")
        }
}
```

注意，当超时时，输出如下：
```text
done
Timeout
```

一种更为优雅的封装是走 `context` 方式，如下。调用的时候传入一个含超时的 `context` 即可，执行时就会返回超时错误：
```golang
func singleflightChan(ctx context.Context, sg *singleflight.Group, key string) (string, error) {
        result := sg.DoChan(key, func() (interface{}, error) {
                // 模拟出现问题
                time.Sleep(30 * time.Second)
                return "", errors.New("bad result")
        })

        select {
        case r := <-result:
                return r.Val.(string), r.Err
        case <-ctx.Done():
                return "", ctx.Err()
        }
}
```

####	解决方案（2）：一个出错，全部出错
降低请求数量 在一些对可用性要求极高的场景下，往往需要一定的请求饱和度来保证业务的最终成功率。一次请求还是多次请求，对于下游服务而言并没有太大区别，此时使用 singleflight 只是为了降低请求的数量级，那么使用 `Forget()` 提高下游请求的并发，如下代码：
```GOLANG
func find(ctx context.Context, query string) (Result, error) {
        time.Sleep(1 * time.Second)
        return "succ", nil
}

func main() {
        var g singleflight.Group
        const n = 10
        key := "some_related_key"
        for i := 0; i < n; i++ {
                go func(j int) {
                        v, err, shared := g.Do(key, func() (interface{}, error) {
                                go func() {
                                        time.Sleep(10 * time.Millisecond)
                                        fmt.Printf("Deleting key:%d, %v\n",j, key)
                                        g.Forget(key)
                                }()
                                ret, err := find(context.Background(), key)
                                return ret, err
                        })
                        fmt.Printf("index: %d, val: %v, shared: %v error:%v\n", j, v, shared,err)
                }(i)
        }

        select {
        case <-time.After(100 * time.Second):
                fmt.Println("Do hangs")
        }
}
```
当有一个并发请求超过 `10ms`，那么将会有第二个请求发起，此时只有 `10ms` 内的请求最多发起一次请求，即最大并发：`100 QPS`，单次请求失败的影响大大降低


##	0x03	总结
本文介绍了 singleflight 机制的应用及避坑场景，另外注意一点，**项目中 singleflight 的变量尽量设置成全局**，如果定义为局部变量（如 `http.HandleFunc` 方法），导致没有发挥 singleflight 的作用


## 0x04 参考
-	[一篇带给你 Go 并发编程 Singleflight](https://www.51cto.com/article/652064.html)
-	[并发编程 -- 用 SingleFlight 合并重复请求](https://zhuanlan.zhihu.com/p/340191872)
-	[SingleFlight 和 CyclicBarrier：请求合并和循环栅栏该怎么用？](https://time.geekbang.org/column/article/309098?noteid=7260356)
-   [缓存三坑与 singleflight](https://www.xcssuper.com/post/%E7%BC%93%E5%AD%98%E4%B8%89%E5%9D%91%E4%B8%8Esingleflight/)
-	[Introduction to the singleflight package of Go](https://nsega.medium.com/introduction-to-the-singleflight-package-of-go-4ee6349e3e78)