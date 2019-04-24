---
title: b站代码阅读笔记  业务接口篇 
date: 2019-04-25T02:49:47+08:00
lastmod: 2019-04-25T02:49:47+08:00
categories: ["开发"]
description:
---

 ## b站代码阅读笔记  业务接口篇

 前几天b站后端代码被强行开源了,让大家有了学习b站代码的机会.写几篇博客记录一下阅读b站代码的经验



### 目录结构

业务接口目录位于`/app/interface`

子目录分别是

```shell
- bbq # 轻视频
- live # 直播
- main # 主站
- openplatform # 开放平台?
- video # 视频
```



每个子目录下都有分开的子目录,分别负责对应接口.b站使用的是它自己开源的kratos项目,但是好像并完全符合kratos的目录规范 

kratos 标准目录格式如

```
├── CHANGELOG.md           # CHANGELOG
├── CONTRIBUTORS.md        # CONTRIBUTORS
├── README.md              # README
├── api                    # api目录为对外保留的proto文件，及生成的pb.go文件
│   ├── api.proto
│   ├── api.pb.go          # 通过go generate生成的pb.go文件
│   └── generate.go
├── cmd                    # cmd目录为main所在
│   └── main.go            # main.go
├── configs                # configs为配置文件目录
│   ├── application.toml   # 应用的自定义配置文件，可能是一些业务开关如：useABtest = true
│   ├── grpc.toml          # grpc相关配置 
│   ├── http.toml          # http相关配置
│   ├── log.toml           # log相关配置
│   ├── memcache.toml      # memcache相关配置
│   ├── mysql.toml         # mysql相关配置
│   └── redis.toml         # redis相关配置
├── go.mod                 # go.mod
└── internal               # internal为项目内部包，包括以下目录：
    ├── dao                # dao层，用于数据库、cache、MQ、依赖某业务grpc|http等资源访问
    │   └── dao.go
    ├── model              # model层，用于声明业务结构体
    │   └── model.go
    ├── server             # server层，用于初始化grpc和http server
    │   └── http           # http层，用于初始化http server和声明handler
    │       └── http.go
    │   └── grpc           # grpc层，用于初始化grpc server和定义method
    │       └── grpc.go
    └── service            # service层，用于业务逻辑处理，且为方便http和grpc共用方法，建议入参和出参保持grpc风格，且使用pb文件生成代码
        └── service.go
```



而实际中部分目录是

```shell
- cmd # 启动服务的命令行文件和配置文件
- conf # 处理配置的目录
- dao  
- http # 定义外部接口和处理一些参数验证
   - http.go # 主文件
- model 
- service 

```

应该是旧代码 未跟随框架更新.



### 接口定义

http.go文件有以下函数

```go
// Init init http sever instance.
func Init(c *conf.Config) {}

// initService init services.
func initService(c *conf.Config) {}

// 定义路由  函数名不固定 好像命名全看心情
func outerRouter(e *bm.Engine) {}
func initRouter(e *bm.Engine) {}
```



kratos基于gin二开,定义路由风格也差不多

```
# 定义主机地址/x/internal 路由组
uploadInternal := e.Group("/x/internal")
# 定义post方法的路由, 参数分别是路径,可能不止一个的中间件,处理函数. 由于是使用了路由组 这里的实际路径是/x/internal/upload 根据
uploadInternal.POST("/upload", verifySvr.Verify, internalUpload)
```



主要的验证中间件含义如下

```
# verifySvr是变量名 类型是verify.Verify
verifySvr.Verify # 需要appkey签名

# 同上 类型是auth.Auth 需要登陆
authSvr.UserMobile # 需要access_key
authSvr.UserWeb # 需要cookies
authSvr.User # 允许UserMobile or UserWeb

# 不需要登陆
authSvr.GuestWeb
authSvr.GuestMobile
authSvr.Guest
```



### 参数获取

主要参数获取方式

```
# 获取参数
c.Request.Form.Get("mobi_app")

# 获取header
buvid = c.Request.Header.Get("Buvid")
# 文件
c.Request.FormFile("file")

```

通过这些就可以在不了解内部逻辑的情况下,快速获取b站外部api接口定义





