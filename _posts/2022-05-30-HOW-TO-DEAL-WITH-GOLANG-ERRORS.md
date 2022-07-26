---
layout:     post
title:      Golang 中的错误处理
subtitle:   如何优雅的处理 Golang 错误以及 gRPC 的错误
date:       2022-05-30
author:     pandaychen
catalog:    true
tags:
    - Golang
    - gRPC
---


##  0x00    前言
本文总结下如何优雅的处理 golang 的错误：
1.	本地 error
2.	gPRC 中的错误

再次明确下，golang 中的 error 只是简单的接口，任何实现了 `Error()` 方法的 struct 都可以用来处理错误信息。
```golang
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
type error interface {
	Error() string
}
```

分为三种细化场景：
-	函数内部的错误处理: 这是一个函数在执行过程中遇到各种错误时的错误处理
-	函数 / 模块的错误信息返回: 一个函数在操作错误之后，要怎么将这个错误信息优雅地返回，方便调用方处理
-	服务 / 系统的错误信息返回: 微服务 / 系统在处理失败时，如何返回一个友好的错误信息，依然是需要让调用方优雅地理解和处理。这是一个服务级的问题

####	错误断言
个人觉得最优雅的方式：

```golang
if err := DoSomething(); err != nil {
    // ...
}
```

在服务端后台逻辑中，对于可能引发 coredump 的核心代码，建议还是加上 `panic`、`recover`：
```golang
func DoSomethingImportant() (err error){
    defer func() {
        if e := recover(); e != nil {
            err = e.(error)
        }
    }()

	//do your business
	return
}
```

##	0x01	函数的错误返回与处理
这里的分歧是最大的，简单梳理下：
####	Go 1.13 版本前
1、使用 `=` 来判断 <br>
将各种错误信息直接定义为一个类枚举值的模式，直接比较即可；当遇到相应的错误信息时，直接返回对应的 `error` 类枚举值就行了。对于调用方也非常方便，可以采用 `switch - case` 来判断错误类型。
```golang
var (
    Err1   = errors.New("record not exist")
    Err2 = errors.New("connection closed")
)

switch err {
case nil:
	// ...
case Err1:
	// ...
case Err2:
	// ...
default:
	// ...
}
```

2、使用类型断言 <br>
由于 `error` 本质是一个 `interface{}`，重新自定义一个 `error` 类型。一方面是用不同的类型来表示不同的错误分类，另一方面则能够实现对于同一错误类型，能够给调用方提供更佳详尽的信息。如下：

```golang
type ErrRecordNotExist errImpl
type ErrPermissionDenined errImpl

type errImpl struct {
    msg string
}

func (e *errImpl) Error() string {
    return e.msg
}
```

对于调用方，则通过以下代码来判断不同的错误：
```golang
if err == nil {
	// OK
} else if _, ok := err.(*ErrRecordNotExist); ok {
	// 处理记录不存在的错误
} else if _, ok := err.(*ErrPermissionDenined); ok {
	// 处理权限错误
} else {
	// 处理其他类型的错误
}
```

3、使用 `fmt.Errorf`<br>
这种不怎么靠谱，但是很简单，主要是自定义的额外信息，引入了不可靠性，只能通过 `strings.Contains()` 来判断错误

####	Go 1.13 版本后
两个改动：
1.	针对 `fmt.Errorf` 增加了 `wraping` 功能
2.	在 `errors` 包中添加了 `Is()` 和 `As()` 方法
	-	`Is()` 方法：
	-	`As()` 方法：

在实际应用中，函数 / 模块透传错误时，应该采用 Go 的 error wrapping 模式，也就是 `fmt.Errorf()` 配合 `%w` 使用，业务方可以放心地添加自己的错误信息，只要调用方统一采用 `errors.Is()` 和 `errors.As()` 即可


####	第三方库：pkg/errors
除了官方库，用的最多的就是 `github.com/pkg/errors` 了，提供了 `3` 个有用的方法：
-	`Wrap` 方法用来包装底层错误，增加上下文文本信息并附加调用栈。 一般用于包装对第三方代码（标准库或第三方库）的调用
-	`WithMessage` 方法仅增加上下文文本信息，不附加调用栈。 如果确定错误已被 Wrap 过或不关心调用栈，可以使用此方法。 注意：不要反复 Wrap ，会导致调用栈重复
-	`Cause` 方法用来判断底层错误
-	在调用 `fmt` 输出时，使用 `%v` 作为格式化参数，那么错误信息会保持一行，其中依次包含调用栈的上下文文本。 使用 `%+v`，则会输出完整的调用栈详情
-	无论是 `Wrap`， `WithMessage` 或 `WithStack` ，当传入的 err 参数为 `nil` 时， 都会返回 `nil`， 这意味着我们在调用此方法之前无需作 `nil` 判断，保持了代码简洁


先看下面的例子：
```golang
import (
   "database/sql"
   "fmt"
)

func foo() error {
   return sql.ErrNoRows
}

func bar() error {
   return fmt.Errorf("foo err, %v", sql.ErrNoRows)
}

func zoo() error {
	// 如果不需要增加额外上下文信息，仅附加调用栈后返回，使用 WithStack 方法
   return errors.WithStack(sql.ErrNoRows)
}

func main() {
   err := bar()
   if err == sql.ErrNoRows {
      fmt.Printf("data not found, %+v\n", err)
      return
   }
}
```

上面的代码的错误处理，有两个缺点：
1.	由于在 `bar()` 方法增加了一些上下文信息，`err == sql.ErrNoRows` 不会成立，判断错误变得麻烦
2.	错误信息不包含调用栈

所以，使用 `pkg/errors`，可以左上下文包装，不丢失原始错误信息， 还能尽量保留完整的调用栈，参考上面提供的 `3` 个方法，代码改造如下：

```golang
import (
   "github.com/pkg/errors"
)

func foo() error {
   return errors.Wrap(sql.ErrNoRows, "foo failed")
}

func bar() error {
   return errors.WithMessage(foo(), "bar failed")
}

func main() {
   err := bar()
   if errors.Cause(err) == sql.ErrNoRows {
	  // 为 true
      fmt.Printf("data not found, %v\n", err)
      fmt.Printf("%+v\n", err)
      return
   }
}
```

输出为：
```text
data not found, bar failed: foo failed: sql: no rows in result set
sql: no rows in result set
foo failed
main.foo
    /usr/three/main.go:11
main.bar
    /usr/three/main.go:15
main.main
    /usr/three/main.go:19
runtime.main
```

####	小结：哪个更好用？
个人还是推荐 `github.com/pkg/errors`，不过，由于使用 `errors.Cause(err, sql.ErrNoRows)`，就意味着 `sql.ErrNoRows` 作为实现细节被暴露给外界了，所以得做好数据保护措施（避免泄漏敏感错误信息）

##	0x02	跨进程场景：服务 / 系统的错误信息返回
这里主要指比如 APIserver、gRPC 服务，客户端与服务端直接的错误处理

####	一般方案
最最常用的就是 `code-message` 模式：
-	`code` 是数字或者预定义的字符串，可以视为整型或者是字符串类型的枚举值；如果是数字的话，大部分情况下是使用 `0` 表示成功，非 `0` 表示失败
-	`message`：使用 "success"、"OK" 或者空字符串等表示成功；反之则为错误信息的具体描述

该模式的特点是：`code` 是给代码使用的（比较相对简单），即代码判断这是一个什么类型的错误，进入相应的分支处理；而 `message` 是给人看的，程序可以以某种形式抛出或者记录这个错误信息，供用户查看。

不过，该模式也有缺点，具体为：
1.	需要开发者提供全品类的错误，以及错误提示信息
2.	底层的错误是否需要返回，让用户看到？万一底层的错误里面包含了敏感信息呢？
3.	用户不一定能看懂错误（文案）

####	优化方案
公司内有同事给出了一种有趣的方案，基于定制化错误代码的，待补充。

下一节，探讨下如何优雅的实现 gRPC 的错误。

##	0x03	gRPC 原生提供的错误处理方法

####	最基本的方法
1、[服务端](https://github.com/avinassh/grpc-errors/blob/master/go/server.go#L24)<br>
```golang
func (s *HelloServer) SayHelloStrict(ctx context.Context, req *api.HelloReq) (*api.HelloResp, error) {
	if len(req.GetName()) >= 10 {
		return nil, status.Errorf(codes.InvalidArgument,
			"Length of `Name` cannot be more than 10 characters")
	}

	return &api.HelloResp{Result: fmt.Sprintf("Hey, %s!", req.GetName())}, nil
}
```

2、客户端<br>
```golang
resp, err = c.SayHelloStrict(
	context.Background(),
	&api.HelloReq{Name: "Leonhard Euler"},
)

if err != nil {
	// ouch!
	// lets print the gRPC error message
	// which is "Length of `Name` cannot be more than 10 characters"
	errStatus, _ := status.FromError(err)
	fmt.Println(errStatus.Message())
	// lets print the error code which is `INVALID_ARGUMENT`
	fmt.Println(errStatus.Code())
	// Want its int version for some reason?
	// you shouldn't actullay do this, but if you need for debugging,
	// you can do `int(status_code)` which will give you `3`
	//
	// Want to take specific action based on specific error?
	if codes.InvalidArgument == errStatus.Code() {
		// do your stuff here
		log.Fatal()
	}
}
```

####	我们项目原先的做法
之前gRPC项目的做法是在response包增加一个公共ReplyHeader，每个pb协议的回包中携带RetCode和RetMsg，但是这样会导致返回码，错误信息和业务字段杂糅在一起，并且每个服务接口都需要定义，对业务代码是侵入性的，比较繁琐。

还可以自己定义，用 `status.New` 或 `status.Newf`即可

不过，这种方式不一定能够满足业务的需求。即默认错误处理方式非常简单直白， 但是有个很大的问题就是 表达能力非常有限。 因为使用类似于 HTTP 状态码的有限抽象 code 没法表达出多样的业务层的错误，且 message 这种字符串也是不应该被请求方当做业务错误标识符来使用。所以我们需要一个额外能够传递业务错误码甚至更多额外错误信息字段的功能。

####	gRPC的包结构
gRPC 提供的 `code-message` 模式，主要结构如下图所示：
![grpc-error](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/grpc-error.png)

标准的错误码定义[在此](https://grpc.github.io/grpc/core/md_doc_statuscodes.html)：
```golang
// A Code is an unsigned 32-bit error code as defined in the gRPC spec.
type Code uint32

const (
	// OK is returned on success.
	OK Code = 0

	// Canceled indicates the operation was canceled (typically by the caller).
	//
	// The gRPC framework will generate this error code when cancellation
	// is requested.
	Canceled Code = 1

	// Unknown error. An example of where this error may be returned is
	// if a Status value received from another address space belongs to
	// an error-space that is not known in this address space. Also
	// errors raised by APIs that do not return enough error information
	// may be converted to this error.
	//
	// The gRPC framework will generate this error code in the above two
	// mentioned cases.
	Unknown Code = 2

	// InvalidArgument indicates client specified an invalid argument.
	// Note that this differs from FailedPrecondition. It indicates arguments
	// that are problematic regardless of the state of the system
	// (e.g., a malformed file name).
	//
	// This error code will not be generated by the gRPC framework.
	InvalidArgument Code = 3

	// DeadlineExceeded means operation expired before completion.
	// For operations that change the state of the system, this error may be
	// returned even if the operation has completed successfully. For
	// example, a successful response from a server could have been delayed
	// long enough for the deadline to expire.
	//
	// The gRPC framework will generate this error code when the deadline is
	// exceeded.
	DeadlineExceeded Code = 4

	// NotFound means some requested entity (e.g., file or directory) was
	// not found.
	//
	// This error code will not be generated by the gRPC framework.
	NotFound Code = 5

	// AlreadyExists means an attempt to create an entity failed because one
	// already exists.
	//
	// This error code will not be generated by the gRPC framework.
	AlreadyExists Code = 6

	// PermissionDenied indicates the caller does not have permission to
	// execute the specified operation. It must not be used for rejections
	// caused by exhausting some resource (use ResourceExhausted
	// instead for those errors). It must not be
	// used if the caller cannot be identified (use Unauthenticated
	// instead for those errors).
	//
	// This error code will not be generated by the gRPC core framework,
	// but expect authentication middleware to use it.
	PermissionDenied Code = 7

	// ResourceExhausted indicates some resource has been exhausted, perhaps
	// a per-user quota, or perhaps the entire file system is out of space.
	//
	// This error code will be generated by the gRPC framework in
	// out-of-memory and server overload situations, or when a message is
	// larger than the configured maximum size.
	ResourceExhausted Code = 8

	// FailedPrecondition indicates operation was rejected because the
	// system is not in a state required for the operation's execution.
	// For example, directory to be deleted may be non-empty, an rmdir
	// operation is applied to a non-directory, etc.
	//
	// A litmus test that may help a service implementor in deciding
	// between FailedPrecondition, Aborted, and Unavailable:
	//  (a) Use Unavailable if the client can retry just the failing call.
	//  (b) Use Aborted if the client should retry at a higher-level
	//      (e.g., restarting a read-modify-write sequence).
	//  (c) Use FailedPrecondition if the client should not retry until
	//      the system state has been explicitly fixed. E.g., if an "rmdir"
	//      fails because the directory is non-empty, FailedPrecondition
	//      should be returned since the client should not retry unless
	//      they have first fixed up the directory by deleting files from it.
	//  (d) Use FailedPrecondition if the client performs conditional
	//      REST Get/Update/Delete on a resource and the resource on the
	//      server does not match the condition. E.g., conflicting
	//      read-modify-write on the same resource.
	//
	// This error code will not be generated by the gRPC framework.
	FailedPrecondition Code = 9

	// Aborted indicates the operation was aborted, typically due to a
	// concurrency issue like sequencer check failures, transaction aborts,
	// etc.
	//
	// See litmus test above for deciding between FailedPrecondition,
	// Aborted, and Unavailable.
	//
	// This error code will not be generated by the gRPC framework.
	Aborted Code = 10

	// OutOfRange means operation was attempted past the valid range.
	// E.g., seeking or reading past end of file.
	//
	// Unlike InvalidArgument, this error indicates a problem that may
	// be fixed if the system state changes. For example, a 32-bit file
	// system will generate InvalidArgument if asked to read at an
	// offset that is not in the range [0,2^32-1], but it will generate
	// OutOfRange if asked to read from an offset past the current
	// file size.
	//
	// There is a fair bit of overlap between FailedPrecondition and
	// OutOfRange. We recommend using OutOfRange (the more specific
	// error) when it applies so that callers who are iterating through
	// a space can easily look for an OutOfRange error to detect when
	// they are done.
	//
	// This error code will not be generated by the gRPC framework.
	OutOfRange Code = 11

	// Unimplemented indicates operation is not implemented or not
	// supported/enabled in this service.
	//
	// This error code will be generated by the gRPC framework. Most
	// commonly, you will see this error code when a method implementation
	// is missing on the server. It can also be generated for unknown
	// compression algorithms or a disagreement as to whether an RPC should
	// be streaming.
	Unimplemented Code = 12

	// Internal errors. Means some invariants expected by underlying
	// system has been broken. If you see one of these errors,
	// something is very broken.
	//
	// This error code will be generated by the gRPC framework in several
	// internal error conditions.
	Internal Code = 13

	// Unavailable indicates the service is currently unavailable.
	// This is a most likely a transient condition and may be corrected
	// by retrying with a backoff. Note that it is not always safe to retry
	// non-idempotent operations.
	//
	// See litmus test above for deciding between FailedPrecondition,
	// Aborted, and Unavailable.
	//
	// This error code will be generated by the gRPC framework during
	// abrupt shutdown of a server process or network connection.
	Unavailable Code = 14

	// DataLoss indicates unrecoverable data loss or corruption.
	//
	// This error code will not be generated by the gRPC framework.
	DataLoss Code = 15

	// Unauthenticated indicates the request does not have valid
	// authentication credentials for the operation.
	//
	// The gRPC framework will generate this error code when the
	// authentication metadata is invalid or a Credentials callback fails,
	// but also expect authentication middleware to generate it.
	Unauthenticated Code = 16

	_maxCode = 17
)
```

####	gRPC 网络传输的 Error 细节
gRPC 的客户端服务端通信时，一般是按照下图所示的流程处理的：
![data-flow](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/grpc/grpc-error-data-flow.jpg)

在使用 gRPC 的时候，远程调用过程中，客户端获取的服务端返回的 error，在 tcp 传递的时候实际上是一串文本。客户端拿到这个文本，是要将其反序列化转换为 error，在这个反序列化的过程中，本质上是 `new` 了一个新的 error 地址，这样就无法根据系统库提供的`error.Is`方法判断 error 地址是否相等。

再总结下 gRPC 网络传输的 error 的处理流程：
-	客户端通过 `invoker` 方法将请求发送到服务端
-	服务端通过 `processUnaryRPC` 方法，获取到用户代码的 error 信息
-	服务端通过 `status.FromError` 方法，将 error 转化为 `status.Status`
-	服务端通过 `WriteStatus` 方法将 `status.Status` 里的数据，写入到 `grpc-status`、`grpc-message`、`grpc-status-details-bin` 的 `header` 里
-	客户端通过网络获取到这些 header 头，使用 `strconv.ParseInt` 解析 `grpc-status` 信息、`decodeGrpcMessage` 解析 `grpc-message` 信息、`decodeGRPCStatusDetails` 解析为 `grpc-status-details-bin` 信息
-	客户端通过 `a.Status().Err()` 获取到用户代码的错误


所以就引出了问题：

问题 1：如何优雅的判断 gRPC 的错误，基于 `code` 还是 `message`？<br>
问题 2：如何使用 `github.com/pkg/errors` 提供的方法，判断错误类型？<br>
问题 3：Http 和 gRPC 的错误如何进行统一？

先看下 gRPC 原生提供的方法：

####	grpc/codes 包
[codes](https://github.com/grpc/grpc-go/blob/v1.48.0/codes/codes.go#L31) 包，提供了 gRPC 的 `codes` 定义，为 `uint32` 型，相应的 proto 定义 [在此](https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto)



####	grpc/status 包
[status 包](https://github.com/grpc/grpc-go/blob/v1.48.0/status/status.go#L108) 提供了 gRPC 对外的错误生成、转换方法，典型的如：
-	`status.Errorf`
-	`status.Error`

此外，提供的转换方法如下：
```golang
// FromError returns a Status representation of err.
//
// - If err was produced by this package or implements the method `GRPCStatus()
//   *Status`, the appropriate Status is returned.
//
// - If err is nil, a Status is returned with codes.OK and no message.
//
// - Otherwise, err is an error not compatible with this package.  In this
//   case, a Status is returned with codes.Unknown and err's Error() message,
//   and ok is false.
func FromError(err error) (s *Status, ok bool) {
	if err == nil {
		return nil, true
	}
	if se, ok := err.(interface {
		GRPCStatus() *Status
	}); ok {
		return se.GRPCStatus(), true
	}
	return New(codes.Unknown, err.Error()), false
}

// Convert is a convenience function which removes the need to handle the
// boolean return value from FromError.
func Convert(err error) *Status {
	s, _ := FromError(err)
	return s
}

// Code returns the Code of the error if it is a Status error, codes.OK if err
// is nil, or codes.Unknown otherwise.
func Code(err error) codes.Code {
	// Don't use FromError to avoid allocation of OK status.
	if err == nil {
		return codes.OK
	}
	if se, ok := err.(interface {
		GRPCStatus() *Status
	}); ok {
		return se.GRPCStatus().Code()
	}
	return codes.Unknown
}

// FromContextError converts a context error or wrapped context error into a
// Status.  It returns a Status with codes.OK if err is nil, or a Status with
// codes.Unknown if err is non-nil and not a context error.
func FromContextError(err error) *Status {
	if err == nil {
		return nil
	}
	if errors.Is(err, context.DeadlineExceeded) {
		return New(codes.DeadlineExceeded, err.Error())
	}
	if errors.Is(err, context.Canceled) {
		return New(codes.Canceled, err.Error())
	}
	return New(codes.Unknown, err.Error())
}
```

####	grpc/internal/status 包、genproto/googleapis/rpc/status 包
status 封装了底层的这两个包，[internal/status](https://github.com/grpc/grpc-go/blob/v1.48.0/internal/status/status.go)包，如上图的结构，注意看最终的 [结构](https://pkg.go.dev/google.golang.org/genproto/googleapis/rpc/status#Status) 定义如下：
```golang
type Status struct {
	// The status code, which should be an enum value of [google.rpc.Code][google.rpc.Code].
	Code int32 `protobuf:"varint,1,opt,name=code,proto3" json:"code,omitempty"`
	// A developer-facing error message, which should be in English. Any
	// user-facing error message should be localized and sent in the
	// [google.rpc.Status.details][google.rpc.Status.details] field, or localized by the client.
	Message string `protobuf:"bytes,2,opt,name=message,proto3" json:"message,omitempty"`
	// A list of messages that carry the error details.  There is a common set of
	// message types for APIs to use.
	Details []*anypb.Any `protobuf:"bytes,3,rep,name=details,proto3" json:"details,omitempty"`
	// contains filtered or unexported fields
}
```
上面的 `Details` 字段，给了我们实现自定义服务错误的途径。

####	gRPC 与 HTTP 转换的标准错误定义
Google API Design 中，规范了 http 和 grpc 状态码的对应关系。给出的 [https://cloud.google.com/apis/design/errors#handling_errors]，HTTP 与 gRPC 的标准错误的转换，如下所示：

| HTTP | gRPC | 说明 | 
| :-----:| :----: | :----: | 
|200|	OK	|无错误|
|400|	INVALID_ARGUMENT|	客户端指定了无效参数。如需了解详情，请查看错误消息和错误详细信息|
|400|	FAILED_PRECONDITION|	请求无法在当前系统状态下执行，例如删除非空目录|
|400|	OUT_OF_RANGE|	客户端指定了无效范围|
|401|	UNAUTHENTICATED	|由于 OAuth 令牌丢失、无效或过期，请求未通过身份验证|
|403|	PERMISSION_DENIED|	客户端权限不足。这可能是因为 OAuth 令牌没有正确的范围、客户端没有权限或者 API 尚未启用|
|404|	NOT_FOUND|	未找到指定的资源|
|409|	ABORTED	|并发冲突，例如读取 / 修改 / 写入冲突|
|409|	ALREADY_EXISTS|	客户端尝试创建的资源已存在|
|429|	RESOURCE_EXHAUSTED|	资源配额不足或达到速率限制。如需了解详情，客户端应该查找 google.rpc.QuotaFailure 错误详细信息。
|499|	CANCELLED|	请求被客户端取消|
|500|	DATA_LOSS|	出现不可恢复的数据丢失或数据损坏。客户端应该向用户报告错误|
|500|	UNKNOWN	|出现未知的服务器错误。通常是服务器错误|
|500|	INTERNAL|	出现内部服务器错误。通常是服务器错误|
|501|	NOT_IMPLEMENTED	API| 方法未通过服务器实现|
|502|	不适用|	到达服务器前发生网络错误。通常是网络中断或配置错误|
|503|	UNAVAILABLE|	服务不可用。通常是服务器已关闭|
|504|	DEADLINE_EXCEEDED|	超出请求时限。仅当调用者设置的时限比方法的默认时限短（即请求的时限不足以让服务器处理请求）并且请求未在时限范围内完成时，才会发生这种情况|


####	grpc-gateway：gRPC的错误转换为HTTP的错误码
[grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway/blob/master/runtime/errors.go#L15) 中提供了 `net.http` 标准状态码到 `grpc/codes` 的转换方法，如下：

```golang
import (
	"context"
	"errors"
	"io"
	"net/http"

	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/grpclog"
	"google.golang.org/grpc/status"
)

// HTTPStatusFromCode converts a gRPC error code into the corresponding HTTP response status.
// See: https://github.com/googleapis/googleapis/blob/master/google/rpc/code.proto
func HTTPStatusFromCode(code codes.Code) int {
	switch code {
	case codes.OK:
		return http.StatusOK
	case codes.Canceled:
		return http.StatusRequestTimeout
	case codes.Unknown:
		return http.StatusInternalServerError
	case codes.InvalidArgument:
		return http.StatusBadRequest
	case codes.DeadlineExceeded:
		return http.StatusGatewayTimeout
	case codes.NotFound:
		return http.StatusNotFound
	case codes.AlreadyExists:
		return http.StatusConflict
	case codes.PermissionDenied:
		return http.StatusForbidden
	case codes.Unauthenticated:
		return http.StatusUnauthorized
	case codes.ResourceExhausted:
		return http.StatusTooManyRequests
	case codes.FailedPrecondition:
		// Note, this deliberately doesn't translate to the similarly named'412 Precondition Failed' HTTP response status.
		return http.StatusBadRequest
	case codes.Aborted:
		return http.StatusConflict
	case codes.OutOfRange:
		return http.StatusBadRequest
	case codes.Unimplemented:
		return http.StatusNotImplemented
	case codes.Internal:
		return http.StatusInternalServerError
	case codes.Unavailable:
		return http.StatusServiceUnavailable
	case codes.DataLoss:
		return http.StatusInternalServerError
	}

	grpclog.Infof("Unknown gRPC error code: %v", code)
	return http.StatusInternalServerError
}


// DefaultRoutingErrorHandler is our default handler for routing errors.
// By default http error codes mapped on the following error codes:
//   NotFound -> grpc.NotFound
//   StatusBadRequest -> grpc.InvalidArgument
//   MethodNotAllowed -> grpc.Unimplemented
//   Other -> grpc.Internal, method is not expecting to be called for anything else
func DefaultRoutingErrorHandler(ctx context.Context, mux *ServeMux, marshaler Marshaler, w http.ResponseWriter, r *http.Request, httpStatus int) {
	sterr := status.Error(codes.Internal, "Unexpected routing error")
	switch httpStatus {
	case http.StatusBadRequest:
		sterr = status.Error(codes.InvalidArgument, http.StatusText(httpStatus))
	case http.StatusMethodNotAllowed:
		sterr = status.Error(codes.Unimplemented, http.StatusText(httpStatus))
	case http.StatusNotFound:
		sterr = status.Error(codes.NotFound, http.StatusText(httpStatus))
	}
	mux.errorHandler(ctx, mux, marshaler, w, r, sterr)
}
```

##	0x04	grpc 的错误码在业务中的优化

####	优化方法1：使用`WithDetails`嵌入业务细节
第一种优化方法：
-	通过 `status` 包创建所需的错误码和相应gRPC框架错误详情
-	再使用 Google API 的相应包可以设置更丰富的业务错误详情

本例引用自[GRPC错误处理](https://overstarry.vip/posts/grpc%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86/)。上一节说到，gRPC 提供的错误模型非常有限，可以使用如下方法进行优化，比如需要在RPC接口中处理非法 id 请求。 如果传了不合法的 id 如 `-1`，需要返回错误给客户端。示例如下：
```golang
func (s *server) RpcMethod(ctx context.Context, orderReq *pb.Order) (*wrappers.StringValue, error) {
	if orderReq.Id == "-1" {
		log.Printf("Order ID is vaild : %s", orderReq.Id)

		errorStatus := status.New(codes.InvalidArgument, "Order ID is not valid")
		//附加业务错误
		ds, err := errorStatus.WithDetails(
			&epb.BadRequest_FieldViolation{
				Field:       "ID",
				Description: fmt.Sprintf("Order ID received is not valid %s : %s", orderReq.Id, orderReq.Description),
			},
		)
		if err != nil {
			return nil, errorStatus.Err()
		}

		//返回details错误
		return nil, ds.Err()
	}
	orderMap[orderReq.Id] = *orderReq
	log.Println("Order : ", orderReq.Id, " -> Added")
	return &wrapper.StringValue{Value: "Order Added: " + orderReq.Id}, nil
}
```

上述代码中,`BadRequest_FieldViolation`是由开发者定义的protobuf结构。那么客户端如何处理错误呢？示例代码如下：
```golang
func main(){
	//...
	order1 := pb.Order{Id: "-1",
					   Items:[]string{"iPhone XS", "Mac Book Pro"},
						Destination:"San Jose, CA",
						 Price:2300.00}
	res, addOrderError := client.AddOrder(ctx, &order1)
	if addOrderError != nil {
		errorCode := status.Code(addOrderError)
		if errorCode == codes.InvalidArgument {
			log.Printf("Invalid Argument Error : %s", errorCode)
			errorStatus := status.Convert(addOrderError)
			for _, d := range errorStatus.Details() {
				switch info := d.(type) {
				case *epb.BadRequest_FieldViolation:
					log.Printf("Request Field Invalid: %s", info)
				default:
					log.Printf("Unexpected error type: %s", info)
				}
			}
		} else {
			log.Printf("Unhandled error : %s ", errorCode)
		}
	} else {
		log.Print("AddOrder Response -> ", res.Value)
	}
}
```

`BadRequestFieldViolation`结构定义如下：
```protobuf
//业务错误定义
message BadRequestFieldViolation {
    string field = 1; 
    string description = 2; 
}
```

####	优化方法2：
```golang
func (a A) Dosomethink(c context.Context, sessionRequest *model.MyModel) (*model.Model, error) {
	return &model.Model{}, status.New(400,"Default error message for 400")
}

func (a A) Dosomethink(c context.Context, sessionRequest *model.MyModel) (*model.Model, error) {
	err := status.Errorf(codes.Code(resp.ErrNum),  errcode.GetCodeDesc(int(resp.ErrNum)))
	return &model.Model{}, err
}    
```

####	优化方法3：集成到框架
依然采用gRPC Status Code的方式，因为gRPC预定了一些错误码，业务错误归类到其中一种，通过`Details`字段来实现扩展。这里假设后台Status统一使用[错误码](https://github.com/grpc/grpc-go/blob/master/codes/codes.go#L121)`FailedPrecondition`，而业务信息是自定义的一个`BizErr`结构，如下：
```golang
message BizErr {
    int32 ret = 1;  //错误码
    string msg = 2; //错误信息
    string ext = 3; //扩展错误信息（按照错误码和客户端约定信息格式）
}
```
将业务的错误信息放在`detail`数组里。对象经序列化后传输；客户端在接收到应答的时候，按status对象解出`code`，`message`，`details`，然后遍历`details`数组解出业务具体的错误信息。获取`BizErr.Ret`即业务错误类型。


##	0x05	总结
[grpc-erorrs](https://github.com/avinassh/grpc-errors/blob/master/go/server.go) 项目提供了各个语言的错误例子。


##	0x06	参考
-	[gRPC 的错误处理实践](https://zhuanlan.zhihu.com/p/435011704)
-	[gRPC 中错误的传递实践](https://chunlife.top/2021/05/31/gRPC%E4%B8%AD%E9%94%99%E8%AF%AF%E7%9A%84%E4%BC%A0%E9%80%92%E5%AE%9E%E8%B7%B5/)
-	[gRPC 服务的响应设计](https://tonybai.com/2021/09/26/the-design-of-the-response-for-grpc-server/)
-	[Golang 错误处理最佳实践](https://medium.com/@dche423/golang-error-handling-best-practice-cn-42982bd72672)
-	[gRPC 服务的响应设计](https://tonybai.com/2021/09/26/the-design-of-the-response-for-grpc-server/)
-	[Don’t just check errors, handle them gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)
-	[GRPC错误处理](https://overstarry.vip/posts/grpc%E9%94%99%E8%AF%AF%E5%A4%84%E7%90%86/)