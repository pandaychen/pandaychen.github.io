---
layout:     post
title:      Kratos 源码分析：Warden 中的 gRPC validator
subtitle:   分析 Kratos 的 gRPC 中的字段验证器
date:       2020-06-13
author:     pandaychen
catalog:    true
tags:
    - Kratos
---


##  0x00    前言
`validator` 的意义是什么，简言之，在协议字段定义规则，使得开发者在代码中简化对字段的验证逻辑。
![image](https://wx1.sbimg.cn/2020/06/22/phpisbestlanguage.jpg)

##  0x01    使用 validator
简单的来说，开启 gRPC 中的协议字段校验需要如下两步：
1.  proto 协议中按照指定 validator 包的规则定义字段的校验规则
2.  在 gRPC 的 server 端、client 端中开启 validator 的拦截器，对 pb 协议中的字段进行校验

##  0x02    定义 validator 字段规则

比如，使用 `github.com/gogo/protobuf/gogoproto/gogo.proto` 定义如下的 protobuf 协议，在 `HelloRequest` 中定义字段及校验规则：
-   `string name = 1 [(gogoproto.jsontag) = "name", (gogoproto.moretags) = "validate:\"required\""];`
-   `int32  age  = 2 [(gogoproto.jsontag) = "age", (gogoproto.moretags) = "validate:\"min=0\""];`
具体含义，从字面上即可直观了解。

带字段校验的 proto 协议定义：

```protobuf
syntax = "proto3";

package testproto;

// 引用包
import "github.com/gogo/protobuf/gogoproto/gogo.proto";

option (gogoproto.goproto_enum_prefix_all) = false;
option (gogoproto.goproto_getters_all) = false;
option (gogoproto.unmarshaler_all) = true;
option (gogoproto.marshaler_all) = true;
option (gogoproto.sizer_all) = true;
option (gogoproto.goproto_registration) = true;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}

  // A bidirectional streaming RPC call recvice HelloRequest return HelloReply
  rpc StreamHello(stream HelloRequest) returns (stream HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1 [(gogoproto.jsontag) = "name", (gogoproto.moretags) = "validate:\"required\""];
  int32  age  = 2 [(gogoproto.jsontag) = "age", (gogoproto.moretags) = "validate:\"min=0\""];
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
  bool success = 2;
}
```

##  0x03    Warden 的 validator 拦截器
上一步，在 proto 文件中已经定义了 `validator` 规则后，下一步就是在 gRPC 服务端的实现中开启对 pb 协议字段的校验。Warden 库使用了 `validator.v9` 这个 package 作为字段拦截器的实现。相较于之前的版本，此版本更为规范，推荐使用。

####    validator.v9 的使用
这里先引用一段 `validator.v9` 包的使用例子：
```golang
type User struct {
    Name string `validate:"is-zhou"`
}

func (u *User) userValidator() error {
    validate := validator.New()
    validate.RegisterValidation("is-zhou", ValidateMyVal)
    err := validate.Struct(u)
    return err
}

// ValidateMyVal implements validator.Func
func ValidateMyVal(fl validator.FieldLevel) bool {
    return fl.Field().String() == "zhou"
}
```

另外，由于 `golang` 语言特性，`struct` 基础数据类型没有赋值会默认零值 (`int` 默认 `0`，`string` 默认 `""` 等等)，所以 `require` 不能校验出基础类型是默认零值，还是被赋为了零值。解决这种问题可以使用指针来代替，比如：

```golang
CommType    int64 `json:"comm_type" validate:"exists"`
CommTypePtr    *int64 `json:"comm_type" validate:"exists"`
```

改成 `Ptr` 类型，这样没赋值时就为 `nil`，赋值为 `0` 时就不是 `nil`，这样就能够解决上面的问题。

####    Warden 的拦截器
Warden 的 validator 拦截器封装代码也非常简洁，注意 `req interface{}` 是 gRPC 的请求结构，所以直接调用 `validate.Struct(req)` 即可：
```golang

var validate = validator.New()

// Validate return a client interceptor validate incoming request per RPC call.
func (s *Server) validate() grpc.UnaryServerInterceptor {
	return func(ctx context.Context, req interface{}, args *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (resp interface{}, err error) {
        // 验证 pb 的字段是否合法
		if err = validate.Struct(req); err != nil {
            err = ecode.RequestErr
            // 不合法，返回错误
			return
		}
		resp, err = handler(ctx, req)
		return
	}
}
```
下面这两个方法，暂时未看明意义：
```golang
// RegisterValidation adds a validation Func to a Validate's map of validators denoted by the key
// NOTE: if the key already exists, the previous validation function will be replaced.
// NOTE: this method is not thread-safe it is intended that these all be registered prior to any validation
func (s *Server) RegisterValidation(key string, fn validator.Func) error {
	return validate.RegisterValidation(key, fn)
}

//GetValidate return the default validate
func (s *Server) GetValidate() *validator.Validate {
	return validate
}
```

此外，[go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware/blob/master/validator/validator.go) 中也提供了 validator 的 gRPC 拦截器实现。

##  0x04    其他的 validator 包
除了 `validator.v9`，[go-proto-validators](https://github.com/mwitkow/go-proto-validators)、[protoc-gen-validate](https://github.com/envoyproxy/protoc-gen-validate) 也是个值得借鉴的实现。相比 `gogoproto`，它的定义感觉更简洁：

```proto
syntax = "proto3";
package validator.examples;
import "github.com/mwitkow/go-proto-validators/validator.proto";

message InnerMessage {
  // some_integer can only be in range (0, 100).
  int32 some_integer = 1 [(validator.field) = {int_gt: 0, int_lt: 100}];
  // some_float can only be in range (0;1).
  double some_float = 2 [(validator.field) = {float_gte: 0, float_lte: 1}];
}

message OuterMessage {
  // important_string must be a lowercase alpha-numeric of 5 to 30 characters (RE2 syntax).
  string important_string = 1 [(validator.field) = {regex: "^[a-z0-9]{5,30}$"}];
  // proto3 doesn't have `required`, the `msg_exist` enforces presence of InnerMessage.
  InnerMessage inner = 2 [(validator.field) = {msg_exists : true}];
}
```

##  0x05    validator 的运行原理
把 `struct` 都可以看成是一棵树（嵌套结构作为子树），那么验证的过程就是遍历（深度优先或广度优先均可）此树的过程。假如我们有如下定义的结构体：
```golang
type Nested struct {
    Email string `validate:"email"`
}
type T struct {
    Age    int `validate:"eq=10"`
    Nested Nested
}
```
对应的树为：
![image](https://wx2.sbimg.cn/2020/06/22/ch6-04-validate-struct-tree.png)

下面给出了一个采用深度方式验证的简单代码（未严格处理 `reflect.Int8/16/32/64`，`reflect.Ptr` 等类型）：

```golang
type Nested struct {
    Email string `validate:"email"`
}
type T struct {
    Age    int `validate:"eq=10"`
    Nested Nested
}

func validateEmail(input string) bool {
    if pass, _ := regexp.MatchString(
        `^([\w\.\_]{2,10})@(\w{1,}).([a-z]{2,4})$`, input,
    ); pass {
        return true
    }
    return false
}

func validate(v interface{}) (bool, string) {
    validateResult := true
    errmsg := "success"
    vt := reflect.TypeOf(v)
    vv := reflect.ValueOf(v)
    for i := 0; i <vv.NumField(); i++ {
        fieldVal := vv.Field(i)
        tagContent := vt.Field(i).Tag.Get("validate")
        k := fieldVal.Kind()

        switch k {
        case reflect.Int:
            val := fieldVal.Int()
            tagValStr := strings.Split(tagContent, "=")
            tagVal, _ := strconv.ParseInt(tagValStr[1], 10, 64)
            if val != tagVal {
                errmsg = "validate int failed, tag is:"+ strconv.FormatInt(
                    tagVal, 10,
                )
                validateResult = false
            }
        case reflect.String:
            val := fieldVal.String()
            tagValStr := tagContent
            switch tagValStr {
            case "email":
                nestedResult := validateEmail(val)
                if nestedResult == false {
                    errmsg = "validate mail failed, field val is:"+ val
                    validateResult = false
                }
            }
        case reflect.Struct:
            // 如果有内嵌的 struct，那么深度优先遍历
            // 就是一个递归过程
            valInter := fieldVal.Interface()
            nestedResult, msg := validate(valInter)
            if nestedResult == false {
                validateResult = false
                errmsg = msg
            }
        }
    }
    return validateResult, errmsg
}
```

测试用例如下：
```golang
func main() {
    var a = T{Age: 10, Nested: Nested{Email: "abc@abc.com"}}

    validateResult, errmsg := validate(a)
    fmt.Println(validateResult, errmsg)
}
```

##  0x06    参考
-   [结构字段验证－－validator.v9](https://www.cnblogs.com/zhzhlong/p/10033234.html)
-   [Go 语言高级编程 - 5.4 validator 请求校验](https://chai2010.gitbooks.io/advanced-go-programming-book/content/ch5-web/ch5-04-validator.html)
-   [validator.v9 DOC](https://godoc.org/gopkg.in/go-playground/validator.v9)
-   [Golang validator 详解](https://blog.xizhibei.me/2019/06/16/an-introduction-to-golang-validator/)
-   [protoc-gen-validate：protoc plugin to generate polyglot message validators](https://github.com/envoyproxy/protoc-gen-validate)