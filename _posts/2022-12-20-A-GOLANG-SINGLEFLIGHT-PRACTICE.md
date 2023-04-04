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

-       `Do` 方法，接受 Key 和待调用的函数，会返回调用函数的结果和错误；使用 `Do` 方法的时候，它会根据提供的 Key 判断是否去真正调用 `fn` 函数。同一个 key，在同一时间只有第一次调用 `Do` 方法时才会去执行 `fn` 函数，其他并发的请求会等待调用的执行结果
-       `DoChan` 方法：异步调用。它会返回一个 channel，等 `fn` 函数执行完，产生了结果以后，就能从这个 `chan` 中接收这个结果
-       `Forget` 方法：在 SingleFlight 中删除一个 Key，之后这个 Key 的 `Do` 方法（仅限于唯一一个）调用会执行 `fn` 函数，而不是等待前一个未完成的 `fn` 函数的结果

singleflight 机制的使用场景有哪些呢？就个人经验而言，满足如下两个场景都可以使用：
-	多 goroutine 并发（大量并发，并发量过低效果不明显）
-	执行相同的逻辑获取结果，区别仅仅是调用参数不同

####  应用一：DNS 库的应用
介绍下 [net/lookup.go](https://github.com/golang/go/blob/master/src/net/lookup.go#L323) 中的机制，`lookupGroup` 作用是将对相同域名的 DNS 记录查询合并成一个查询。下面是 net 库提供的 DNS 记录查询方法 `lookupGroup` 的实现，使用的是异步查询方法 `DoChan`
```GOLANG
type Resolver struct {
	// .......
	// Dial optionally specifies an alternate dialer for use by
	// Go's built-in DNS resolver to make TCP and UDP connections
	// to DNS services. The host in the address parameter will
	// always be a literal IP address and not a host name, and the
	// port in the address parameter will be a literal port number
	// and not a service name.
	// If the Conn returned is also a PacketConn, sent and received DNS
	// messages must adhere to RFC 1035 section 4.2.1, "UDP usage".
	// Otherwise, DNS messages transmitted over Conn must adhere
	// to RFC 7766 section 5, "Transport Protocol Selection".
	// If nil, the default dialer is used.
	Dial func(ctx context.Context, network, address string) (Conn, error)

	// lookupGroup merges LookupIPAddr calls together for lookups for the same
	// host. The lookupGroup key is the LookupIPAddr.host argument.
	// The return values are ([]IPAddr, error).
	lookupGroup singleflight.Group

	// TODO(bradfitz): optional interface impl override hook
	// TODO(bradfitz): Timeout time.Duration?
}
```

`lookupIPAddr` 的实现如下，执行的并发逻辑是 `lookupIPAddr`：
```golang
// lookupIPAddr looks up host using the local resolver and particular network.
// It returns a slice of that host's IPv4 and IPv6 addresses.
func (r *Resolver) lookupIPAddr(ctx context.Context, network, host string) ([]IPAddr, error) {
	// Make sure that no matter what we do later, host=="" is rejected.
	if host == "" {
		return nil, &DNSError{Err: errNoSuchHost.Error(), Name: host, IsNotFound: true}
	}
	if ip, err := netip.ParseAddr(host); err == nil {
		return []IPAddr{{IP: IP(ip.AsSlice()).To16(), Zone: ip.Zone()}}, nil
	}
	trace, _ := ctx.Value(nettrace.TraceKey{}).(*nettrace.Trace)
	if trace != nil && trace.DNSStart != nil {
		trace.DNSStart(host)
	}
	// The underlying resolver func is lookupIP by default but it
	// can be overridden by tests. This is needed by net/http, so it
	// uses a context key instead of unexported variables.
	resolverFunc := r.lookupIP
	if alt, _ := ctx.Value(nettrace.LookupIPAltResolverKey{}).(func(context.Context, string, string) ([]IPAddr, error)); alt != nil {
		resolverFunc = alt
	}

	// We don't want a cancellation of ctx to affect the
	// lookupGroup operation. Otherwise if our context gets
	// canceled it might cause an error to be returned to a lookup
	// using a completely different context. However we need to preserve
	// only the values in context. See Issue 28600.
	lookupGroupCtx, lookupGroupCancel := context.WithCancel(withUnexpiredValuesPreserved(ctx))

        // 用于 singleflight 的 key
	lookupKey := network + "\000" + host
	dnsWaitGroup.Add(1)
        // 使用 SingleFlight 的 DoChan 合并多个查询请求（非阻塞）
	ch := r.getLookupGroup().DoChan(lookupKey, func() (any, error) {
		return testHookLookupIP(lookupGroupCtx, resolverFunc, network, host)
	})

	dnsWaitGroupDone := func(ch <-chan singleflight.Result, cancelFn context.CancelFunc) {
		<-ch
		dnsWaitGroup.Done()
		cancelFn()
	}
	select {
	case <-ctx.Done():
		// Our context was canceled. If we are the only
		// goroutine looking up this key, then drop the key
		// from the lookupGroup and cancel the lookup.
		// If there are other goroutines looking up this key,
		// let the lookup continue uncanceled, and let later
		// lookups with the same key share the result.
		// See issues 8602, 20703, 22724.
		if r.getLookupGroup().ForgetUnshared(lookupKey) {
			lookupGroupCancel()
			go dnsWaitGroupDone(ch, func() {})
		} else {
			go dnsWaitGroupDone(ch, lookupGroupCancel)
		}
		ctxErr := ctx.Err()
		err := &DNSError{
			Err:       mapErr(ctxErr).Error(),
			Name:      host,
			IsTimeout: ctxErr == context.DeadlineExceeded,
		}
		if trace != nil && trace.DNSDone != nil {
			trace.DNSDone(nil, false, err)
		}
		return nil, err
	case r := <-ch:
		dnsWaitGroup.Done()
		lookupGroupCancel()
		err := r.Err
		if err != nil {
			if _, ok := err.(*DNSError); !ok {
				isTimeout := false
				if err == context.DeadlineExceeded {
					isTimeout = true
				} else if terr, ok := err.(timeout); ok {
					isTimeout = terr.Timeout()
				}
				err = &DNSError{
					Err:       err.Error(),
					Name:      host,
					IsTimeout: isTimeout,
				}
			}
		}
		if trace != nil && trace.DNSDone != nil {
			addrs, _ := r.Val.([]IPAddr)
			trace.DNSDone(ipAddrsEface(addrs), r.Shared, err)
		}

                // 正常返回数据
		return lookupIPReturn(r.Val, err, r.Shared)
	}
}

// lookupIPReturn turns the return values from singleflight.Do into
// the return values from LookupIP.
func lookupIPReturn(addrsi any, err error, shared bool) ([]IPAddr, error) {
	if err != nil {
		return nil, err
	}
	addrs := addrsi.([]IPAddr)
	if shared {
		clone := make([]IPAddr, len(addrs))
		copy(clone, addrs)
		addrs = clone
	}
	return addrs, nil
}
```

##  0x02  负面影响

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

####	解决方案（1）：有效超时控制，避免一个阻塞，全员等待
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

####	解决方案（2）：频率控制，解决一个出错，全部出错
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
本文介绍了 singleflight 机制的应用及避坑场景，另外注意几点
1.      **项目中 singleflight 的变量尽量设置成全局**，如果定义为局部变量（如 `http.HandleFunc` 方法），导致没有发挥 singleflight 的作用
2.      `DoChan` 方法配置 `context.WithTimeout` 使用效果更好（参考上面 `net.Resolver` 实现）


## 0x04 参考
-	[一篇带给你 Go 并发编程 Singleflight](https://www.51cto.com/article/652064.html)
-	[并发编程 -- 用 SingleFlight 合并重复请求](https://zhuanlan.zhihu.com/p/340191872)
-	[SingleFlight 和 CyclicBarrier：请求合并和循环栅栏该怎么用？](https://time.geekbang.org/column/article/309098?noteid=7260356)
-       [缓存三坑与 singleflight](https://www.xcssuper.com/post/%E7%BC%93%E5%AD%98%E4%B8%89%E5%9D%91%E4%B8%8Esingleflight/)
-	[Introduction to the singleflight package of Go](https://nsega.medium.com/introduction-to-the-singleflight-package-of-go-4ee6349e3e78)
-       [golang-dns](https://github.com/golang/go/blob/master/src/net/lookup.go)