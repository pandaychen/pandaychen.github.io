---
layout:     post
title:      Golang 中的 sync.Pool 使用
subtitle:   调优系列：Golang 优化系列之临时对象池
date:       2020-03-11
author:     pandaychen
header-img:
catalog: true
category:   false
tags:
    - Golang
    - Pool
---

##  0x00	前言

当多个 goroutine 都需要创建同⼀个对象的时候，如果 goroutine 数过多，导致对象的创建数⽬剧增，进⽽导致 GC 压⼒增大。形成并发⼤ `-->` 占⽤内存⼤ `-->` GC 缓慢 `-->` 处理并发能⼒降低 `-->` 并发更⼤这样的恶性循环。 在这个时候，需要有⼀个对象池，每个 goroutine 不再⾃⼰单独创建对象，⽽是从对象池中获取出⼀个对象。`sync.Pool` 本质上就是池化的机制，用来减少 GC 和 malloc 的时间。它可以用来 **保存一组可独立访问的临时对象**。注意这里的临时这两个字，它说明 pool 中的对象可能会被 GC 回收。

sync.Pool 是 Golang 内置的对象池技术，可用于缓存临时对象，避免因频繁建立临时对象所带来的消耗以及对 GC 造成的压力。

先引入一个普通的场景，http 解响应包，在高并发下，`data []byte` 每次都大概率从 heap 上分配新内存，而 `data` 在请求完毕后就成为了垃圾数据等待 gc；即在并发量、请求体都很大的情况下，内存就会迅速被占满，这种情况就非常适合使用 `sync.Pool` 进行优化

```go
func handler(writer http.ResponseWriter, req *http.Request) {
   var (
      err  error
      data []byte
   )
   // ...
   data, err = ioutil.ReadAll(req.Body)
   if err != nil {
      return
   }
   // ...
}
```

####	sync.Pool 的要点

`sync.Pool` 的重点如下：
-	pool 本身就是线程安全的，多个 goroutine 可以并发地存取对象
-	pool 不可在使用之后再复制使用，内嵌了 `noCopy` 结构体，`go vet` 时会检查
-	使用建议：不要对 pool 池中的对象做任何假定，即使用前需要清理缓存对象，有两种方案：在调用 `Pool.Put` 前进行对象 Reset 操作；在 `Pool.Get` 操作后对对象进行 Reset 操作，防止读出脏数据

####	Pool API
-	`New`：`sync.Pool` 的 `New` 字段的类型是函数 `func() interface{}`，当调用 pool 的 `Get` 方法从池中获取元素，没有更多的空闲元素可返回时，就会调用 `New` 方法来创建新的元素；如果未设置 `New`，当无更多的空闲元素可返回时，`Get` 方法将返回 `nil`，表明当前没有可用元素
-	`Get` 方法：从 pool 取走一个元素，该元素会从 pool 中移除，返回给调用者；用户需要判断返回值是否为 `nil`
-	`Put` 方法：将一个元素返还给 pool，pool 会把这个元素（非 `nil` 值）保存到池中，并且可以复用；

##	0x01	sync.Pool 的使用

####	普通 struct
```GOLANG
var pool *sync.Pool

type Person struct {
    Name string
}

func initPool() {
	pool = &sync.Pool{
			New: func() interface{} {
					return new(Person)
			},
	}
}

func main() {
	initPool()

	p := pool.Get().(*Person)
	p.Name = "first"

	// 需要在 Put 前，清除 p 的各个成员，这里为了展示，就不清除了
	// 放回对象池中以供其他goroutine调用
	pool.Put(p)
	fmt.Println("Get from pool:", pool.Get().(*Person))
	fmt.Println("Pool is empty", pool.Get().(*Person))
}
```

使用简单易懂：

1.	初始化一个`sync.Pool`对象，并初始化`New`匿名函数，当对象池中没有对象时调用
2.	通过`Get()`从池子里获取对象，进行业务逻辑处理
3.	使用完后利用`Put()`把对象放回池子里，供后续的goroutine调用

####	bytes.Buffer
`Get` 方法会返回 Pool 中已存在的 `*bytes.Buffer`，否则将调用 `New` 方法来初始化新的 `*bytes.Buffer`，但在缓冲区使用后，必须将其重置并放回 Pool 中

```GOLANG
//1. 定义 bufferPool 缓存池，New 函数用于返回一个新的 bytes.Buffer
var bufferPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

//GetBuffer 返回池中一个 Buffer
func GetBuffer() *bytes.Buffer {
    return bufferPool.Get().(*bytes.Buffer)
}

//PutBuffer 将 Buffer 重新放入池中
func PutBuffer(buf *bytes.Buffer) {
    buf.Reset()	// 注意，放回池前，一定要先对 buf 做重置操作
    bufferPool.Put(buf)
}
```

####	[]byte 池
```golang
const BuffSize = 10 * 1024

var buffPool sync.Pool

func GetBuff() *bytes.Buffer {
	var buffer *bytes.Buffer
	item := buffPool.Get()
	if item == nil {
		var byteSlice []byte
		byteSlice = make([]byte, 0, BuffSize)
		buffer = bytes.NewBuffer(byteSlice)

	} else {
		buffer = item.(*bytes.Buffer)
	}
	return buffer
}

func PutBuff(buffer *bytes.Buffer) {
	buffer.Reset()
	buffPool.Put(buffer)
}

func main(){
	var fy_buf bytes.Buffer
	PutBuff(&fy_buf)
	p:=GetBuff()
	p.WriteString("aaaaa")
	fmt.Println(p.String())
}
```

####	[]byte 多级字节池
[多级字节池](https://github.com/pandaychen/goes-wrapper/blob/master/pypool/buffer_pool.go) 的实现，预先生成不同 size 的 `sync.Pool`：

```golang
var (
	bufPool32k     sync.Pool
	bufPool16k     sync.Pool
	bufPool8k      sync.Pool
	bufPool2k      sync.Pool
	bufPool1k      sync.Pool
	bufPoolDefault sync.Pool
)
```

####	JSON 编码优化

在高并发的 HTTP 服务中，JSON 编码/解码是非常频繁的操作，每次 `json.Marshal` 内部都会创建 `bytes.Buffer`、encoder 等临时对象。通过 `sync.Pool` 复用 `json.Encoder` 和底层 buffer，可以显著减少内存分配：

```golang
var jsonBufPool = sync.Pool{
    New: func() interface{} {
        return new(bytes.Buffer)
    },
}

func MarshalJSON(v interface{}) ([]byte, error) {
    buf := jsonBufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        jsonBufPool.Put(buf)
    }()

    enc := json.NewEncoder(buf)
    if err := enc.Encode(v); err != nil {
        return nil, err
    }
    // Encode 会在末尾追加 '\n'，需要去掉
    result := make([]byte, buf.Len()-1)
    copy(result, buf.Bytes())
    return result, nil
}
```

Go 标准库 `encoding/json` 内部也使用了 `sync.Pool` 来复用 `encodeState` 对象（[源码链接](https://github.com/golang/go/blob/go1.16/src/encoding/json/encode.go#L300)）：

```golang
// encoding/json/encode.go
var encodeStatePool sync.Pool

func newEncodeState() *encodeState {
    if v := encodeStatePool.Get(); v != nil {
        e := v.(*encodeState)
        e.Reset()
        if len(e.ptrLevel) > 0 {
            panic("ptrEncoder.encode should have emptied ptrLevel via
            defer")
        }
        e.ptrSeen = nil
        return e
    }
    return &encodeState{ptrLevel: make([]reflect.Value, 0, startDetectingCyclesAfter)}
}

// Marshal 内部调用
func Marshal(v interface{}) ([]byte, error) {
    e := newEncodeState()
    err := e.marshal(v, encOpts{escapeHTML: true})
    if err != nil {
        return nil, err
    }
    buf := append([]byte(nil), e.Bytes()...)
    encodeStatePool.Put(e)
    return buf, err
}
```

####	Protocol Buffers 序列化优化

在 gRPC 等场景中，protobuf 的序列化/反序列化频率极高。可以使用 `sync.Pool` 复用序列化的 buffer：

```golang
var protoBufPool = sync.Pool{
    New: func() interface{} {
        return make([]byte, 0, 1024)
    },
}

func MarshalProto(msg proto.Message) ([]byte, error) {
    size := proto.Size(msg)
    buf := protoBufPool.Get().([]byte)
    if cap(buf) < size {
        buf = make([]byte, 0, size)
    }
    buf = buf[:size]

    _, err := proto.MarshalOptions{}.MarshalAppend(buf[:0], msg)
    if err != nil {
        protoBufPool.Put(buf[:0])
        return nil, err
    }

    result := make([]byte, size)
    copy(result, buf)
    protoBufPool.Put(buf[:0])
    return result, nil
}
```

实际上 gRPC-Go 框架内部也大量使用了 `sync.Pool` 来复用编解码 buffer，参考 [grpc-go 中的 bufferPool](https://github.com/grpc/grpc-go/blob/master/mem/buffers.go)

####	gzip 压缩/解压优化

`gzip.Writer` 和 `gzip.Reader` 的创建开销较大（内部需要初始化压缩状态机），在需要频繁压缩/解压的场景下适合使用 `sync.Pool`：

```golang
var gzipWriterPool = sync.Pool{
    New: func() interface{} {
        w, _ := gzip.NewWriterLevel(nil, gzip.BestSpeed)
        return w
    },
}

func CompressData(data []byte) ([]byte, error) {
    var buf bytes.Buffer
    w := gzipWriterPool.Get().(*gzip.Writer)
    w.Reset(&buf)

    _, err := w.Write(data)
    if err != nil {
        w.Close()
        gzipWriterPool.Put(w)
        return nil, err
    }
    err = w.Close()
    gzipWriterPool.Put(w)
    if err != nil {
        return nil, err
    }
    return buf.Bytes(), nil
}

var gzipReaderPool sync.Pool

func DecompressData(data []byte) ([]byte, error) {
    br := bytes.NewReader(data)
    var r *gzip.Reader
    if v := gzipReaderPool.Get(); v != nil {
        r = v.(*gzip.Reader)
        r.Reset(br)
    } else {
        var err error
        r, err = gzip.NewReader(br)
        if err != nil {
            return nil, err
        }
    }
    defer func() {
        r.Close()
        gzipReaderPool.Put(r)
    }()

    return ioutil.ReadAll(r)
}
```

Go 标准库 `net/http` 内部也使用了类似的 gzip pool 来优化 HTTP 压缩响应的性能

####	zap 日志库中的应用

高性能日志库 [uber-go/zap](https://github.com/uber-go/zap) 内部使用了 `sync.Pool` 来复用 `Buffer` 对象，避免每条日志都分配新的 buffer：

```golang
// zap/buffer/pool.go
type Pool struct {
    p *sync.Pool
}

func NewPool() Pool {
    return Pool{p: &sync.Pool{
        New: func() interface{} {
            return &Buffer{bs: make([]byte, 0, _size)}
        },
    }}
}

func (p Pool) Get() *Buffer {
    buf := p.p.Get().(*Buffer)
    buf.Reset()
    buf.pool = p
    return buf
}

func (p Pool) put(buf *Buffer) {
    p.p.Put(buf)
}
```

zap 之所以能做到比标准库 `log` 快 `4-10x`，`sync.Pool` 的使用是关键优化手段之一

####	标准库 fmt 中的应用
Go 标准库也大量使用了 `sync.Pool`，例如 `fmt` 和 `encoding/json` 等；下面是 `fmt.Printf` 的实现（[源码链接](https://github.com/golang/go/blob/go1.16/src/fmt/print.go)）。从代码分析可知，`fmt.Printf` 的调用是非常频繁的，利用 `sync.Pool` 复用 `pp` 对象能够极大地提升性能，减少内存占用，同时降低 GC 压力。注意下面 `Put` 放回前，对 `buf` 清空的技巧 `p.buf = p.buf[:0]`

```GOLANG
// Use simple []byte instead of bytes.Buffer to avoid large dependency.
type buffer []byte

func (b *buffer) write(p []byte) {
	*b = append(*b, p...)
}

func (b *buffer) writeString(s string) {
	*b = append(*b, s...)
}

func (b *buffer) writeByte(c byte) {
	*b = append(*b, c)
}

func (bp *buffer) writeRune(r rune) {
	if r < utf8.RuneSelf {
		*bp = append(*bp, byte(r))
		return
	}
	b := *bp
	n := len(b)
	for n+utf8.UTFMax > cap(b) {
		b = append(b, 0)
	}
	w := utf8.EncodeRune(b[n:n+utf8.UTFMax], r)
	*bp = b[:n+w]
}

// pp is used to store a printer's state and is reused with sync.Pool to avoid allocations.
type pp struct {
    buf buffer		//buffer定义见上
    //...
}

var ppFree = sync.Pool{
	New: func() interface{} { return new(pp) },
}

// newPrinter allocates a new pp struct or grabs a cached one.
func newPrinter() *pp {
	p := ppFree.Get().(*pp)
	p.panicking = false
	p.erroring = false
	p.wrapErrs = false
	p.fmt.init(&p.buf)
	return p
}

// free saves used pp structs in ppFree; avoids an allocation per invocation.
func (p *pp) free() {
	if cap(p.buf) > 64<<10 {
		return
	}

	p.buf = p.buf[:0]	// 清零的技巧
	p.arg = nil
	p.value = reflect.Value{}
	p.wrappedErr = nil
	ppFree.Put(p)
}

func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}

// Printf formats according to a format specifier and writes to standard output.
// It returns the number of bytes written and any write error encountered.
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```

从上面的代码分析可知，`sync.Pool`被用来存储`pp`结构体，结构体中的`buf buffer`成员，是一个变长的slice，因而`pp`在使用完毕后，被放回Pool中备其他调用方复用，以此来减少内存的申请和释放次数

特别注意上面`free`的实现中的一个细节，即若slice cap大于某一个阈值的buffer，则对此对象不做放回处理（等待OS层面的GC），以此防止出现大内存不会释放的情况：
```golang
//fmt
func (p *pp) free() {
    if cap(p.buf) > 64<<10 {	//64k
        return
    }

    p.buf = p.buf[:0]
    p.arg = nil
    p.value = reflect.Value{}
    ppFree.Put(p)
}
```

####	gin 框架
gin 框架会给每个请求分配一个 `Context` 用以进行追踪，从 pool 中获取 `Context` 对象，用完后又还回去，注意 还回去之前这里也调用了 `reset()` 方法进行字段清空。如下：
```go
func New() *Engine {
	engine := &Engine{
		
	}
	engine.pool.New = func() interface{} {
		return engine.allocateContext()
	}
	return engine
}

func (engine *Engine) allocateContext() *Context {
	return &Context{engine: engine}
}

func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	//清空
	c.reset()

	engine.handleHTTPRequest(c)

	//使用完毕放回
	engine.pool.Put(c)
}
```

##	0x01	sync.Pool 的实现
本小节，分析下 `sync.Pool` 的源码。

####	sync.Pool 的接口
需要用户实现 `New` 方法，需要注意的是，`sync.Pool` 的名字是具有迷惑性的，这是一个具有易失性的池子，放进池子内的对象会在 GC 的时候被回收掉。可以说 `sync.Pool` 是一个垃圾回收站，因为扔进去的对象就不再保证存在，并且也不会指定取出哪一个；因此 Pool 是用于存放对象的，不建议用来缓存一些有状态的对象（长连接等）或者数据，因为 Pool 中的内容是会随着 GC 而被回收的。
```golang
type Pool struct{
    func (p *Pool) Get() interface{}
    func (p *Pool) Put(x interface{})
    New func() interface{}
}
```

####	Pool 的 gc
对于 Pool 而言，并不能无限扩展，否则对象占用内存太多了，会引起内存溢出。`sync.Pool` 初始化时注册了一个 `poolCleanup` 方法，在每次 GC 启动时调用：
```golang
func init() {
    runtime_registerPoolCleanup(poolCleanup)
}
```

####	优化效果评估
`sync.Pool` 到底能在什么程度上提高内存调度的性能？Go 1.13+ 引入 victim 机制后，Pool 中的对象可以存活**两轮 GC**（而非之前的一轮），这大幅提高了对象的复用率。`sync.Pool` 的优化效果主要体现在以下方面：

-  **减少内存分配次数**：对象复用避免了频繁的 `malloc`/`free`
-  **降低 GC 压力**：减少了堆上的临时对象数量，GC 扫描和回收的工作量减少
-  **减少 GC 停顿时间**：对象复用减少了 heap 的增长速度，GC 触发频率降低

适合 `sync.Pool` 优化的场景特征：对象创建/销毁频繁、生命周期短、无状态或可重置。典型场景包括：buffer 复用、编解码器复用、HTTP 请求上下文复用等。详细的源码分析请参考 [Sync.Pool 应用与分析（续）](https://pandaychen.github.io/2023/05/01/A-GOLANG-SYNC-POOL-ANALYSIS/)


##	0x02	常见问题与陷阱

####	问题 1：使用后忘记 Reset，导致数据泄漏

这是最常见的错误，如果 `Put` 回池前不清空对象状态，下一个使用者可能读到上一次的残留数据：

```golang
buf := pool.Get().(*bytes.Buffer)
buf.WriteString("sensitive data")
// 错误：直接 Put 回去，没有 Reset
pool.Put(buf)

// 其他 goroutine 取出后可能看到 "sensitive data"
buf2 := pool.Get().(*bytes.Buffer)
fmt.Println(buf2.String()) // 可能输出 "sensitive data"
```

正确做法是在 `Put` 前或 `Get` 后调用 `Reset`。推荐在 `Put` 前清理，因为这样调用方对自己放回的对象负责

####	问题 2：Pool 中存放引用类型导致的内存泄漏

当 Pool 中缓存的对象内部包含大的 slice 或 map 时，即使对象被 `Reset`，底层数组的内存可能不会释放。标准库 `fmt` 的做法值得借鉴，当 buffer cap 超过阈值时直接丢弃不放回：

```golang
func (p *pp) free() {
    if cap(p.buf) > 64<<10 {  // 超过 64KB 直接丢弃
        return
    }
    p.buf = p.buf[:0]
    ppFree.Put(p)
}
```

####	问题 3：sync.Pool 与 []byte 结合使用时的 grpc 解包异常

一个经典坑是：从 Pool 获取 `[]byte` 后传给 grpc/protobuf 反序列化，反序列化内部可能持有 slice 的引用（零拷贝优化），此时如果把 `[]byte` 放回 Pool 并被其他 goroutine 复用，会导致数据被覆盖。**必须确保 Pool 中的对象在 Put 时没有其他地方持有引用**

```golang
// 错误示例
buf := bytePool.Get().([]byte)
proto.Unmarshal(buf[:n], &msg) // protobuf 可能零拷贝引用 buf 内部的数据
bytePool.Put(buf)               // 危险：msg 内部可能还在引用 buf

// 正确做法：确保 msg 完全拥有自己的数据拷贝后再放回
```

####	问题 4：Pool 不适合做连接池

`sync.Pool` 的对象会被 GC 回收，不适合缓存有状态的长连接（TCP、DB 连接等）。连接池应使用带有最大连接数、空闲超时等策略的专用连接池实现（如 `database/sql` 内置的连接池）

####	问题 5：过度使用 sync.Pool

不是所有场景都适合 `sync.Pool`。如果对象的创建成本很低（如小的 struct），使用 Pool 反而会因为 `pin`/`unpin`、原子操作等增加额外开销。建议通过 benchmark 验证 Pool 是否真的带来了性能提升：

```golang
func BenchmarkWithPool(b *testing.B) {
    pool := sync.Pool{New: func() interface{} { return new(MyStruct) }}
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            obj := pool.Get().(*MyStruct)
            // 使用 obj
            pool.Put(obj)
        }
    })
}

func BenchmarkWithoutPool(b *testing.B) {
    b.RunParallel(func(pb *testing.PB) {
        for pb.Next() {
            obj := new(MyStruct)
            // 使用 obj
            _ = obj
        }
    })
}
```

##	0x03	总结
golang 中 `sync.Pool` 的目的是缓存已分配但未使用的对象，以便后续取出使用，这样可以减少 GC 压力。但是需要注意的是，`sync.Pool` 在 GC 触发时，会对对象池中的数据进行清理（Go 1.13+ 通过 victim 机制延长到两轮 GC）。最后再总结下：

**`sync.Pool` 的特点**：
-	`sync.Pool` 主要是为了对象的复用，避免重复创建、销毁。将暂时不用的对象缓存起来，待下次需要的时候直接使用，不用再次经过内存分配，复用对象的内存，减轻 GC 的压力
-	是协程安全的，Go 1.13+ 中 `Get` 和 `Put` 操作基于 lock-free 的 `poolChain` 实现，无需互斥锁
-	只适合暂时缓存，不适合长期存储数据；因为 GC 时会清除 `sync.Pool` 的数据，存储在 pool 中的对象在不被通知的情况下随时可能被清理掉，所以 `sync.Pool` 并不适用于连接池、DB 连接池之类的有状态存储
-	当对象被 `Put` 放回 `sync.Pool` 时，不应当还有其他异步操作对放回的对象进行操作，防止发生竞争

**使用最佳实践**：
-	始终设置 `New` 函数，避免 `Get` 返回 `nil` 时需要额外处理
-	`Put` 前或 `Get` 后清空对象状态（推荐 `Put` 前清理）
-	对内含大容量 slice/buffer 的对象，考虑在 `Put` 前检查 cap 大小，超过阈值则丢弃不放回
-	通过 benchmark 验证 `sync.Pool` 确实带来了性能提升，避免过度优化
-	确保 `Put` 回 Pool 的对象不再被任何其他地方引用

##  0x04	参考
-   [Go 1.13 中 sync.Pool 是如何优化的?](https://colobu.com/2019/10/08/how-is-sync-Pool-improved-in-Go-1-13/)
-	[深入 Golang 之 sync.Pool 详解](https://www.cnblogs.com/sunsky303/p/9706210.html)
-	[几行代码为老板省百万 - 某高并发服务 GOGC 及 UDP Pool 优化](https://blog.csdn.net/weixin_45583158/article/details/118533587)
-	[深度解密 Go 语言之 sync.Pool](https://www.cnblogs.com/qcrao-2018/p/12736031.html)
-	[Go 每日一库之 bytebufferpool](https://segmentfault.com/a/1190000039969499)
-	[Go sync.Pool](https://geektutu.com/post/hpg-sync-pool.html)
-	[Anti-memory-waste byte buffer pool](https://github.com/valyala/bytebufferpool)
-	[bucketpool 的实现](https://github.com/vitessio/vitess/blob/main/go/bucketpool/bucketpool.go)
-	[go sync.pool []byte 导致 grpc 解包异常](https://xiaorui.cc/archives/5969)