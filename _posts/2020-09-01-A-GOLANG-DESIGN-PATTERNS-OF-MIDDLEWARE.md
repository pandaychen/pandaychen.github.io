---
layout:     post
title:      Golang Web/RPC 框架中的设计模式
subtitle:   装饰器模式及 Pipeline 模式回顾
date:       2020-09-01
header-img: img/super-mario.jpg
author:     pandaychen
catalog:    true
tags:
    - 设计模式
---

##  0x00    前言
这篇文章想聊聊，在使用 gRPC 和 Gin 框架中，理解到的设计模式。主要有下面几个：
1.  装饰器模式
2.  Pipeline 模式

##  0x01    基础回顾
1、 装饰器模式
装饰器模式，是一种动态地往一个类中添加新的行为的设计模式。就功能而言，修饰模式相比生成子类更为灵活，这样可以给某个对象而不是整个类添加一些功能。<br>
看起来像洋葱一样, 洋葱内部最嫩最核心的时原始对象，然后外面包了一层业务逻辑，然后还可以继续在外面包一层业务逻辑。

装饰器模式的原理是：增加一个修饰类包裹原来的类，包裹的方式一般是通过在将原来的对象作为修饰类的构造函数的参数。装饰类实现新的功能，但是，在不需要用到新功能的地方，它可以直接调用原来的类中的方法。与适配器模式不同，装饰器模式的修饰类必须和原来的类有相同的接口。它是在运行时动态的增加对象的行为，在执行包裹对象的前后执行特定的逻辑，而代理模式主要控制内部对象行为，一般在编译器就确定了。

对于 Go 语言来说，因为函数是第一类的，所以包裹的对象也可以是函数，比如最常见的时 http 的例子：

```golang
func log(h http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Println("Before Serve")
        h.ServeHTTP(w, r)
        log.Println("After Serve")
    })
}
```

上面提供了打印日志的修饰器，在执行实际的包裹对象前后分别打印出一条日志。 因为 http 标准库既可以以函数的方式 (http.HandlerFunc) 提供 router, 也可以以 struct 的方式 (http.Handler) 提供，所以这里提供了两个修饰器 (修饰函数) 的实现。

##  0x02    泛型修饰器


##  0x03    Pipeline 模式

Pipeline 模式，包含一组命令和一系列的处理对象 (handler)，每个处理对象定义了它要处理的命令的业务逻辑，剩余的命令交给其它处理对象处理。所以整个业务你看起来就是 `if ... else if ... else if ....... else ... endif` 这样的逻辑判断，handler 还负责分发剩下待处理的命令给其它 handler。职责链既可以是直线型的，也可以复杂的树状拓扑。

pipeline 是一种架构模式，它由一组处理单元组成的链构成，处理单元可以是进程、线程、纤程、函数等构成，链条中的每一个处理单元处理完毕后就会把结果交给下一个。处理单元也叫做 filter, 所以这种模式也叫做 pipes and filters design pattern。

和职责链模式不同，职责链设计模式不同，职责链设计模式是一个行为设计模式，而 pipeline 是一种架构模式；其次 pipeline 的处理单元范围很广，大到进程小到函数，都可以组成 pipeline 设计模式；第三狭义来讲 pipeline 的处理方式是单向直线型的，不会有分叉树状的拓扑；第四是针对一个处理数据，pipeline 的每个处理单元都会处理参与处理这个数据 (或者上一个处理单元的处理结果)。

pipeline 模式经常用来实现中间件，比如 java web 中的 filter, Go web 框架中的中间件。接下来让我们看看 Go web 框架的中间件实现的各种技术。


##  0x04    中间件对模式的应用
Go web 框架的中间件，这里我们说的 web 中间件是指在接收到用户的消息，先进行一系列的预处理，然后再交给实际的 http.Handler 去处理，处理结果可能还需要一系列的后续处理。

虽说，最终各个框架还是通过修饰器的方式实现 pipeline 的中间件，但是各个框架对于中间件的处理还是各有各的风格，区别主要是 When、Where 和 How。


##  使用 filter 数组实现

在 `gin` 框架中，使用 filter 数组（Slice） 实现了中间件拦截器，如下面的例子：
```golang
// HandlersChain defines a HandlerFunc array.
type HandlersChain []HandlerFunc

// Last returns the last handler in the chain. ie. the last handler is the main own.
func (c HandlersChain) Last() HandlerFunc {
	if length := len(c); length > 0 {
		return c[length-1]
	}
	return nil
}
```


配置：
```golang
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```


因为中间件也是 HandlerFunc, 可以当作一个 handler 来处理。



##  参考
-   [明白了，原来 Go web 框架中的中间件都是这样实现的](https://colobu.com/2019/08/21/decorator-pattern-pipeline-pattern-and-go-web-middlewares/)