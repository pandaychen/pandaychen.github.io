---
layout:     post
title:      Kratos 源码分析：ecode 错误代码
subtitle:   分析 Kratos 的 Error-code
date:       2020-07-10
header-img: img/golang-tools-fun.png
author:     pandaychen
catalog:    true
tags:
    - Kratos
---

## 0x00 背景
本篇文章来分析下 Kratos 对错误码的封装（HTTP && RPC）。一般而言，错误码封装的方式：
1.  整形值的错误码
2.  错误码对应的出错信息
3.	HTTP 或 RPC 的方便定义

错误码，一般被用来进行异常传递，且需要具有携带 `message` 文案信息的能力。

##	0x01	Kratos 的错误码使用
我们先从用例入手，然后再简单分析下 ecode 内部实现及其与 RPC 协议的封装。

```golang
import (
	"fmt"
	"github.com/go-kratos/kratos/pkg/ecode"
)

var _ ecode.Codes

// 用户错误码定义
var (
	UserNotLogin = ecode.New(123)
	UserLoginAuthError = ecode.New(-304)
)

var cms = map[int]string{
	0:    "SUCC",
	-304: "NOT MODIFIED",
	-404: "NOT FOUND",
	123:  "USER DEFINE MESSAGE",
}

// 调用 ecode 包的 Register 方法注册错误码 Map
func init() {
    ecode.Register(cms)
}

func main() {
    fmt.Println(UserNotLogin.Error(), UserNotLogin.Message())	// 输出为 `123 USER DEFINE MESSAGE
}
```

同时，Kratos 也提供了 [工具生成的方式](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/ecode.md)，使用 proto 协议定义错误码，格式如下：

```proto
// user.proto
syntax = "proto3";

package ecode;

enum UserErrCode {
  UserUndefined = 0; // 因 protobuf 协议限制必须存在！！！无意义的 0，工具生成代码时会忽略该参数
  UserNotLogin = 123; // 正式错误码
}
```

需要注意以下几点：

1. 必须是 enum 类型，且名字规范必须以 "ErrCode" 结尾，如：`UserErrCode`
2. 因为 protobuf 协议限制，第一个 enum 值必须为无意义的 `0`

使用 `kratos tool protoc --ecode user.proto` 进行生成，生成如下代码：

```go
package ecode

import (
    "github.com/go-kratos/kratos/pkg/ecode"
)

var _ ecode.Codes

// UserErrCode
var (
    UserNotLogin = ecode.New(123);
)
```

##	0x02	错误码的设计 && 实现
在 Kratos 中，使用包中定义的全局变量 [`_codes`](https://github.com/go-kratos/kratos/blob/master/pkg/ecode/ecode.go#L13) 来存储错误码，使用 `_messages` 来存储错误码的 map，使用 `ecode.Register` 方法向 `_messages` 存储一个 map：
```golang
var (
	_messages atomic.Value         // NOTE: stored map[int]string
	_codes    = map[int]struct{}{} // register codes.
)

// Register register ecode message map.
func Register(cm map[int]string) {
	_messages.Store(cm)
}
```

####	Code 封装
在 `kratos` 里，错误码被设计成 `Codes` 接口（Interface{}），[声明如下](https://github.com/go-kratos/kratos/blob/master/pkg/ecode/ecode.go)：

```golang
// Codes ecode error interface which has a code & message.
type Codes interface {
	// sometimes Error return Code in string form
	// NOTE: don't use Error in monitor report even it also work for now
	Error() string
	// Code get error code.
	Code() int
	// Message get code message.
	Message() string
	//Detail get error detail,it may be nil.
	Details() []interface{}
}

// A Code is an int error code spec.
type Code int
```

可以看到该 `Codes` 接口一共有四个方法，且 `type Code int` 结构体实现了该接口。
-	`Error()`：返回错误码字符串（类似于 `err.Error()`）
-	`Code()`：返回错误码整形值

封装的方法代码如下，着重看下 `Message()` 方法，它的过程是从全局 `_messages` 中取出 `map`，再从 map 中取出 `e Code` 对应的错误码字符串：

```golang
func (e Code) Error() string {
	return strconv.FormatInt(int64(e), 10)
}

// Code return error code
func (e Code) Code() int { return int(e) }

// Message return error message
func (e Code) Message() string {
	if cm, ok := _messages.Load().(map[int]string); ok {
		if msg, ok := cm[e.Code()]; ok {
			return msg
		}
	}
	return e.Error()
}

// Details return details.
func (e Code) Details() []interface{} { return nil }
```

#### 注册 Message
一个 `Code` 错误码可以对应一个 `message`，默认实现会从全局变量 `_messages` 中获取，业务可以将自定义 `Code` 对应的 `message` 通过调用 `Register` 方法的方式传递进去，如前面的例子，使用的错误码映射关系为 `int`==>`string`，当然这也不是绝对的，比如有业务要支持多语言的场景就可以扩展为类似 `map[int]LangStruct` 的结构，因为全局变量 `_messages` 是 `atomic.Value` 类型，只需要修改对应的 `Message` 方法实现即可。

```golang
	UserNotLogin = ecode.New(123)
	var cms = map[int]string{
		0:    "SUCC",
		-304: "NOT MODIFIED",
		-404: "NOT FOUND",
		123:  "USER DEFINE MESSAGE",
	}
	ecode.Register(cms)
	fmt.Println(ecode.UserNotLogin.Message())
```

####	`String` 和 `Int` 方法
`ecode` 包提供了 `Int` 和 `String` 两个方法，用来将 `int` 型和 `string` 转换为 `Code` 类型：

```golang
// Int parse code int to error.
func Int(i int) Code { return Code(i) }

// String parse code string to error.
func String(e string) Code {
	if e == "" {
		return OK
	}
	// try error string
	i, err := strconv.Atoi(e)
	if err != nil {
		return ServerErr
	}
	return Code(i)
}
```

#### `Details` 接口的作用
`Details` 接口为 `gRPC` 预留，`gRPC` 传递异常会将服务端的错误码 pb 序列化之后赋值给 `Details`，客户端拿到之后反序列化得到，这里在下面的章节中详细分析下。

#### 如何转换为 ecode？
在我们开发中，可以按照如下方式将 `errors` 或错误码转换为 ecode 类型，通常而言，错误码转换有以下两种情况：
1. 因为框架传递错误是靠 `ecode` 错误码，比如 bm 框架返回的 `code` 字段默认就是数字，那么客户端接收到如 `{"code":-404}` 的话，可以使用 `ec := ecode.Int(-404)` 或 `ec := ecode.String("-404")` 来进行转换
2. 在项目中 `dao` 层返回一个错误码，往往返回参数类型建议为 `error` 而不是 `ecode.Codes`，因为 `error` 更通用，那么上层 `service` 就可以使用 `ec := ecode.Cause(err)` 进行转换（为 `ecode`）

`ecode.Cause()` 方法实现如下，将标准的 `error` 转为 `ecode`，注意，这里调用了 `errors.Cause` 来获取底层的错误：
```golang
// Cause cause from error to ecode.
func Cause(e error) Codes {
	if e == nil {
		return OK
	}
	ec, ok := errors.Cause(e).(Codes)
	if ok {
		return ec
	}
	//e.Error() 返回是字符串的错误码，通过 String() 方法转为 Codes（int）类型
	return String(e.Error())
}
```

#### 错误码之比较
`ecode` 包提供了 `Equal()` 和 `EqualError` 方法，来判断（两个）错误码是否相等：
-	`ecode` 与 `ecode` 判断：使用 `ecode.Equal(ec1, ec2)` 方法
-	`ecode` 与标注库的 `error` 判断：使用 `ecode.EqualError(ec, err)` 方法

```golang
// Equal equal a and b by code int.
func Equal(a, b Codes) bool {
	if a == nil {
		a = OK
	}
	if b == nil {
		b = OK
	}
	return a.Code() == b.Code()
}

// EqualError equal error
func EqualError(code Codes, err error) bool {
	return Cause(err).Code() == code.Code()
}
```

##	0x03	ecode 与 RPC 返回码的封装
在 RPC 中，可以使用 `ecode` 包来简化 RPC 返回码。如下面这个 `SayHello` 的实现中，传递异常会将服务端的错误码 pb 序列化之后赋值给 `Details`，客户端拿到之后反序列化得到错误码及错误信息：
```golang
func (s *helloServer) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	if in.Name == "err_detail_test" {
			err, _ := ecode.Error(ecode.AccessDenied, "AccessDenied").WithDetails(&pb.HelloReply{Success: true, Message: "this is test detail"})
			return nil, err
	}
	count++
	if count%3==0{
			return nil,errors.New("rpc error")
	}
	return &pb.HelloReply{Message: fmt.Sprintf("hello %s from %s", in.Name, s.addr)}, nil
}
```
1. `ecode` 包内的 `Status` 结构体实现了 `Codes` 接口 [代码位置](https://github.com/go-kratos/kratos/blob/master/pkg/ecode/status.go)
2. `warden/internal/status` 包内包装了 `ecode.Status` 和 `grpc.Status` 进行互相转换的方法 [代码位置](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/internal/status/status.go)
3. `warden` 的 `client` 和 `server` 则使用转换方法将 `gRPC` 底层返回的 `error` 最终转换为 `ecode.Status` [代码位置](https://github.com/go-kratos/kratos/blob/master/pkg/net/rpc/warden/client.go#L162)



##	0x04	参考
-	[ecode 使用文档](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/ecode.md)