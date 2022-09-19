---
layout: post
title: DiDi 开源 API 网关 gatekeeper （v1.0）项目分析
subtitle: 一个微服务网关的设计与实现
date: 2021-06-10
author: pandaychen
header-img:
catalog: true
category: false
tags:
  - Gateway
  - 微服务网关
---

## 0x00 前言

[Didi-gatekeeper](https://github.com/didi/gatekeeper) 是一个 `Golang` 的不依赖分布式数据库的 `API` 网关（基于 `gin`），使用它可以高效进行服务代理，支持在线化热更新服务配置以及纯文件方式服务配置，支持主动探测方式自动剔除故障节点以及手动方式关闭下游节点流量，还可以通过自定义中间件方式灵活拓展其他功能。

本文主要分析下其实现中可以借鉴的思路及细节：

- 插件式中间件引入
- 部署架构

其 [官网文档](https://github.com/didi/gatekeeper/blob/master/README.md) 有比较详细的项目介绍，特性为：

- 支持 `http`、`websocket`、`tcp` 服务代理
- 自动剔除故障节点
- 手动关闭下游节点流量
- 加权负载轮询
- `URL` 地址重写
- 服务限流：支持独立 `IP` 限流
- 高拓展性：如支持自定义请求前验证 `request`、请求后更改 `response`、tcp 中间件、http 中间件等方法
- 最少依赖：无需任何额外组件即可运行，`mysql`、`redis` 只做在线管理和统计使用，可随时关闭

#### 部署架构图

![img](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/didi-gatekeeper/architecture1.png)

`GateKeeper` 的架构图是比较典型的网关层次架构，前端使用通用的 Nginx 接入，`GateKeeper` 本身可以做无状态的平行扩展（gatekeeper的代理层暴露在nginx之后，不同于有些网关直接暴露给用户侧）

- 接入层（`Nginx`，`Haproxy` 等）
- 网关层（包含配置中心、管理平面、健康度监控等）
- 业务层

流程如下：

1. 用户通过接入层连接到 `GateKeeper` 实例，接入层一般选用 `nginx`、`Haproxy`、`LVS` 等高可用 HA 集群（当然也可以直接让 `GateKeeper` 实例对外，不推荐）
2. 每个 `GateKeeper` 实例，针对每个服务模块，单独进行服务探测
3. 在线服务管理时，配置数据先保存到 `GateKeeper` 配置 `DB` 中，然后再通过调用配置更新接口（ `/reload` ），更新所有实例机器配置

## 0x01 gatekeeper配置&&使用

配置如下：
应用module配置：
![app](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/gatekeeper/gk-module-1.png)

bifrost_app详细配置：
![app1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/gatekeeper/gk-module-settings-1.png)

test_app详细配置：
![app2](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/gatekeeper/gk-module-settings-2.png)

应用后端代码，[参考](https://github.com/pandaychen/golang_in_action/blob/master/reverse_proxy/gatekeeper/backend.go)

浏览器访问如下 URL：
```text
http://127.0.0.1:8081/gatekeeper/bifrost_app/a?app_id=test_app&sign=62fda0f2212eaffd90dbf04136768c5f
```

返回

```json
{"message":"b"}
```

观察访问视图：
![flow1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/gatekeeper/gk-module-flow-1.png)

## 0x02 核心流程分析--数据结构


####	网关配置

1、[APP](https://github.com/pandaychen/gatekeeper/blob/master/dao/app.go)，类似租户的概念<br>
```golang
//GatewayAPP app配置
type GatewayAPP struct {
	ID              int64  `json:"id" toml:"-" orm:"column(id);auto" description:"自增主键"`
	AppID           string `json:"app_id" toml:"app_id" validate:"required"  orm:"column(app_id);" description:"租户id"`
	Name            string `json:"name" toml:"name" validate:"required"  orm:"column(name);" description:"租户名称"`
	Secret          string `json:"secret" toml:"secret" validate:"required"  orm:"column(secret);" description:"密钥"`
	Method          string `json:"method" toml:"method" validate:""  orm:"column(method);" descriId           int6ption:"请求方法"`
	Timeout         int64  `json:"timeout" toml:"timeout" orm:"column(timeout);" description:"超时时间"`
	OpenAPI         string `json:"open_api" toml:"open_api" orm:"column(open_api);" description:"接口列表，支持前缀匹配"`
	WhiteIps        string `json:"white_ips" toml:"white_ips" orm:"column(white_ips);" description:"ip白名单，支持前缀匹配"`
	CityIDs         string `json:"city_ids" toml:"city_ids" orm:"column(city_ids);" description:"city_id数据权限"`
	TotalQueryDaily int64  `json:"total_query_daily" toml:"total_query_daily" orm:"column(total_query_daily);" description:"日请求量"`
	QPS             int64  `json:"qps" toml:"qps" orm:"column(qps);" description:"qps"`
	GroupID         int64  `json:"group_id" toml:"group_id" orm:"column(group_id);" description:"数据关联id"`
}
```

2、[module](https://github.com/pandaychen/gatekeeper/blob/master/dao/module.go#L24)，应用，每个module代表一个代理转发完整配置<br>

####  GatewayModule
一个`GatewayModule`代表一个完整的转发配置，包含如下结构：
- `GatewayModuleBase`：存储了module的基本信息，下面的属性和`GatewayModuleBase`的信息关联
- `GatewayMatchRule`：存储了url的映射关系（网关前的url如何映射到网关后端的请求url规则）
- `LoadBalance`：存储了lb后端节点的一些配置，如后端ip列表、超时时间等
- `AccessControl`：该module关联的的一些控制属性


```golang
//GatewayModule module配置
type GatewayModule struct {
	Base          *GatewayModuleBase    `json:"base" validate:"required" toml:"base"`
	MatchRule     []*GatewayMatchRule   `json:"match_rule" validate:"required"  toml:"match_rule"`
	LoadBalance   *GatewayLoadBalance   `json:"load_balance" validate:"required" toml:"load_balance"`
	AccessControl *GatewayAccessControl `json:"access_control" toml:"access_control"`
}

//GatewayModuleBase base数据表结构体
type GatewayModuleBase struct {
	ID           int64  `json:"id" toml:"-" orm:"column(id);auto" description:"自增主键"`
	LoadType     string `json:"load_type" toml:"load_type" validate:"" orm:"column(load_type);size(255)" description:"负载类型 http/tcp"`
	Name         string `json:"name" toml:"name" validate:"required" orm:"column(name);size(255)" description:"模块名"`
	ServiceName  string `json:"service_name" toml:"service_name" validate:"" orm:"column(service_name);size(255)" description:"服务名称"`
	PassAuthType int8   `json:"pass_auth_type" toml:"pass_auth_type" validate:"" orm:"column(pass_auth_type)" description:"认证传参类型"`
	FrontendAddr string `json:"frontend_addr" toml:"frontend_addr" validate:"" orm:"column(frontend_addr);size(255)" description:"前端绑定ip地址"`
}

//GatewayMatchRule match_rule数据表结构体
type GatewayMatchRule struct {
	ID         int64  `json:"id" toml:"-" orm:"column(id);auto" description:"自增主键"`
	ModuleID   int64  `json:"module_id" toml:"-" orm:"column(module_id)" description:"模块id"`
	Type       string `json:"type" toml:"type" validate:"required" orm:"column(type)" description:"匹配类型"`
	Rule       string `json:"rule" toml:"rule" validate:"required" orm:"column(rule);size(1000)" description:"规则"`
	RuleExt    string `json:"rule_ext" validate:"required" toml:"rule_ext" orm:"column(rule_ext);size(1000)" description:"拓展规则"`
	URLRewrite string `json:"url_rewrite" validate:"required" toml:"url_rewrite" orm:"column(rule_ext);size(1000)" description:"url重写"`
}


//GatewayLoadBalance load_balance数据表结构体
type GatewayLoadBalance struct {
	ID            int64  `json:"id" toml:"-" orm:"column(id);auto" description:"自增主键"`
	ModuleID      int64  `json:"module_id" toml:"-" orm:"column(module_id)"`
	CheckMethod   string `json:"check_method" validate:"required" toml:"check_method" orm:"column(check_method);size(200)" description:"检查方法"`
	CheckURL      string `json:"check_url" validate:"" toml:"check_url" orm:"column(check_url);size(500)" description:"检测url"`
	CheckTimeout  int    `json:"check_timeout" validate:"required,min=100" toml:"check_timeout" orm:"column(check_timeout);size(500)" description:"检测超时时间"`
	CheckInterval int    `json:"check_interval" validate:"required,min=100" toml:"check_interval" orm:"column(check_interval);size(500)" description:"检测url"`

	Type                string `json:"type" validate:"required" toml:"type" orm:"column(type);size(100)" description:"轮询方式"`
	IPList              string `json:"ip_list" validate:"required" toml:"ip_list" orm:"column(ip_list);size(500)" description:"ip列表"`
	WeightList          string `json:"weight_list" validate:"" toml:"weight_list" orm:"column(weight_list);size(500)" description:"ip列表"`
	ForbidList          string `json:"forbid_list" validate:"" toml:"forbid_list" orm:"column(forbid_list);size(1000)" description:"禁用 ip列表"`
	ProxyConnectTimeout int    `json:"proxy_connect_timeout" validate:"required,min=1" toml:"proxy_connect_timeout" orm:"column(proxy_connect_timeout)" description:"单位ms，连接后端超时时间"`
	ProxyHeaderTimeout  int    `json:"proxy_header_timeout" validate:"" toml:"proxy_header_timeout" orm:"column(proxy_header_timeout)" description:"单位ms，后端服务器数据回传时间"`
	ProxyBodyTimeout    int    `json:"proxy_body_timeout" validate:"" toml:"proxy_body_timeout" orm:"column(proxy_body_timeout)" description:"单位ms，后端服务器响应时间"`
	MaxIdleConn         int    `json:"max_idle_conn" validate:"" toml:"max_idle_conn" orm:"column(max_idle_conn)"`
	IdleConnTimeout     int    `json:"idle_conn_timeout" validate:"" toml:"idle_conn_timeout" orm:"column(idle_conn_timeout)" description:"keep-alived超时时间，新增"`
}


//GatewayAccessControl access_control 数据表结构体
type GatewayAccessControl struct {
	ID              int64  `json:"id" toml:"-" orm:"column(id);auto" description:"自增主键"`
	ModuleID        int64  `json:"module_id" toml:"-" orm:"column(module_id)" description:"模块id"`
	BlackList       string `json:"black_list" toml:"black_list" orm:"column(black_list);size(1000)" description:"黑名单ip"`
	WhiteList       string `json:"white_list" toml:"white_list" orm:"column(white_list);size(1000)" description:"白名单ip"`
	WhiteHostName   string `json:"white_host_name" toml:"white_host_name" orm:"column(white_host_name);size(1000)" description:"白名单主机"`
	AuthType        string `json:"auth_type" toml:"auth_type" orm:"column(auth_type);size(100)" description:"认证方法"`
	ClientFlowLimit int64  `json:"client_flow_limit" toml:"client_flow_limit" orm:"column(client_flow_limit);size(100)" description:"客户端ip限流"`
	Open            int64  `json:"open" toml:"open" orm:"column(open);size(100)" description:"是否开启权限功能"`
}
```

####	GateWayService
每个请求到达时，都会使用`GateWayService`对其进行封装，加入网关连接的所有配置属性（请求依赖），`GateWayService`提供的方法[在此](https://github.com/pandaychen/gatekeeper/blob/master/service/gateway_service.go)
```GOLANG
//GateWayService 网关核心服务
type GateWayService struct {
	currentModule *dao.GatewayModule
	w             http.ResponseWriter
	req           *http.Request
}

//NewGateWayService 构建一个服务
func NewGateWayService(w http.ResponseWriter, req *http.Request) *GateWayService {
	return &GateWayService{
		w:   w,
		req: req,
	}
}
```


####	Context结构
```GO
//Context 对response和request方法的封装
type Context struct {
	Res        http.ResponseWriter
	Req        *http.Request
	StatusCode int
	urlValue   url.Values
	formValue  url.Values
	done       bool
}
```

####	全局中间件
[BeforeRequestAuthRegisterFuncs](https://github.com/pandaychen/gatekeeper/blob/master/service/register_func.go#L16)、[ModifyResponseRegisterFuncs](https://github.com/pandaychen/gatekeeper/blob/master/service/register_func.go#L18)提供了在请求前后对数据处理的中间件数组，对应的注册方法为`RegisterBeforeRequestAuthFunc`、`RegisterModifyResponseFunc`

```golang
//BeforeRequestAuthRegisterFuncs 验证方法列表
var BeforeRequestAuthRegisterFuncs []func(m *dao.GatewayModule, req *http.Request, res http.ResponseWriter) (bool, error)

//ModifyResponseRegisterFuncs 过滤方法列表
var ModifyResponseRegisterFuncs []func(m *dao.GatewayModule, req *http.Request, res *http.Response) error
```


##  0x03  gatekeeper数据流程

gatekeeper 对请求的主要处理如下图所示：
![flow1](https://raw.githubusercontent.com/pandaychen/pandaychen.github.io/master/blog_img/gateway/gatekeeper/gatekeeper-flow1.png)


##  0x04  核心流程

#### 初始化 & 启动



####	路由规则匹配与转发
核心[流程](https://github.com/pandaychen/gatekeeper/blob/master/middleware/match_rule.go)：
```golang
//MatchRule 匹配模块中间件
func MatchRule() gin.HandlerFunc {
	return func(c *gin.Context) {
		//封装resp和request
		gws := service.NewGateWayService(c.Writer, c.Request)
		if err := gws.MatchRule(); err != nil {
			public.ResponseError(c, http.StatusBadRequest, err)
			return
		}
		c.Set(MiddlewareServiceKey, gws)
	}
}
```

在`MatchRule`方法中，主要完成对前置路由到后置路由的匹配及替换工作，主要逻辑在`gws.MatchRule`中：
```GOLANG

//MatchRule 匹配规则
func (o *GateWayService) MatchRule() error {
	var currentModule *dao.GatewayModule
	modules := SysConfMgr.GetModuleConfig()
Loop:
	for _, module := range modules.Module {
		if module.Base.LoadType != "http" {
			continue
		}
		for _, matchRule := range module.MatchRule {
			urlStr := o.req.URL.Path
			if matchRule.Type == "url_prefix" && strings.HasPrefix(urlStr, matchRule.Rule+"/") {
				currentModule = module
				//提前检测，减少资源消耗
				if matchRule.URLRewrite == "" {
					break Loop
				}
				for _, uw := range strings.Split(matchRule.URLRewrite, ",") {
					uws := strings.Split(uw, " ")
					if len(uws) == 2 {
						re, regerr := regexp.Compile(uws[0])
						if regerr != nil {
							return regerr
						}
						rep := re.ReplaceAllString(urlStr, uws[1])
						o.req.URL.Path = rep
						public.ContextNotice(o.req.Context(), DLTagMatchRuleSuccess, map[string]interface{}{
							"url":       o.req.RequestURI,
							"write_url": rep,
						})
						if o.req.URL.Path != urlStr {
							break
						}
					}
				}
				break Loop
			}
		}
	}
	if currentModule == nil {
		public.ContextWarning(o.req.Context(), DLTagMatchRuleFailure, map[string]interface{}{
			"msg": "module_not_found",
			"url": o.req.RequestURI,
		})
		return errors.New("module not found")
	}
	public.ContextNotice(o.req.Context(), DLTagMatchRuleSuccess, map[string]interface{}{
		"url": o.req.RequestURI,
	})
	o.SetCurrentModule(currentModule)
	return nil
}
```

####	中间件：Picker后端节点

```golang
//LoadBalance 负载均衡中间件
func LoadBalance() gin.HandlerFunc {
	return func(c *gin.Context) {
		gws, ok := c.MustGet(MiddlewareServiceKey).(*service.GateWayService)
		if !ok {
			public.ResponseError(c, http.StatusBadRequest, errors.New("gateway_service not valid"))
			return
		}

		//根据LB策略，选择一个后端
		proxy, err := gws.LoadBalance()
		if err != nil {
			public.ResponseError(c, http.StatusProxyAuthRequired, err)
			return
		}
		requestBody, ok := c.MustGet(MiddlewareRequestBodyKey).([]byte)
		if !ok {
			public.ResponseError(c, http.StatusBadRequest, errors.New("request_body not valid"))
			return
		}
		c.Request.Body = ioutil.NopCloser(bytes.NewBuffer(requestBody))
		proxy.ServeHTTP(c.Writer, c.Request)

		//到这里就停止了
		c.Abort()
	}
}
```


```golang
//LoadBalance 请求负载
func (o *GateWayService) LoadBalance() (*httputil.ReverseProxy, error) {
	ipList, err := SysConfMgr.GetModuleIPList(o.currentModule.Base.Name)
	if err != nil {
		public.ContextWarning(o.req.Context(), DLTagLoadBalanceFailure, map[string]interface{}{
			"msg":             err,
			"modulename":      o.currentModule.Base.Name,
			"availableIpList": SysConfMgr.GetAvaliableIPList(o.currentModule.Base.Name),
		})
		return nil, errors.New("get_iplist_error")
	}
	if len(ipList) == 0 {
		public.ContextWarning(o.req.Context(), DLTagLoadBalanceFailure, map[string]interface{}{
			"msg":             "empty_iplist_error",
			"modulename":      o.currentModule.Base.Name,
			"availableIpList": SysConfMgr.GetAvaliableIPList(o.currentModule.Base.Name),
		})
		return nil, errors.New("empty_iplist_error")
	}
	proxy, err := o.GetModuleHTTPProxy()
	if err != nil {
		public.ContextWarning(o.req.Context(), DLTagLoadBalanceFailure, map[string]interface{}{
			"msg":    err,
			"module": o.currentModule.Base.Name,
		})
		return nil, err
	}
	return proxy, nil
}
```

最终调用`SysConfigManage.GetModuleHTTPProxy`方法，选择后端节点：
```go
//GetModuleHTTPProxy 获取http代理方法
func (s *SysConfigManage) GetModuleHTTPProxy(moduleName string) (*httputil.ReverseProxy, error) {
	rr, err := s.GetModuleRR(moduleName)
	if err != nil {
		return nil, err
	}
	s.moduleProxyFuncMapLocker.RLock()
	defer s.moduleProxyFuncMapLocker.RUnlock()
	proxyFunc, ok := s.moduleProxyFuncMap[moduleName]
	if ok {
		return proxyFunc(rr), nil
	}
	return nil, errors.New("module proxy empty")
}
```


## 0x05 参考

- [gatekeeper repo](https://github.com/didi/gatekeeper)

转载请注明出处，本文采用 [CC4.0](http://creativecommons.org/licenses/by-nc-nd/4.0/) 协议授权
