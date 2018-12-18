---
layout: post
title: goa-call_me service
category: Go
tags: Go
keywords: Go,Goa
description:GOA Design Language
---

## 概述

goa 框架 demo。创建了一个名为 `call_me` 的服务，支持使用 `GET` 请求，`url` 中有 `uid` 参数。请求和应答包体均为 `json` 格式。

## 代码生成

```bash
cd ~/Documents/00Code/call_me/src
goagen bootstrap -d call_me/design

go build -o run
```

## 日志封装

在 `goa` 框架内部使用 `logrus` 日志。其实在框架内封装了 `logrus` 的适配器，使用还是比较方便，但是 `goa` 的官方文档并没有做详细介绍。

`logrus` 设置方法：
- 创建 `logrus` 的 `logger`
- 使用 `gologrus.New` 函数创建 `goa.LogAdapter` 对象
- 使用 `service.WithLogger(goalogrus.New(logger))` 将 `goa.LogAdapter` 设置到 `service` 对象

`logrus` 使用：
- 使用 `goalogrus.Entry(ctx)` 将 `ctx` 中 `logrus.Entry` 取出
- 使用 `logru.Entry` 对象进行日志打印

## middleware

在 `goa` 中可以使用 `middleware` 添加（HTTP）请求处理阶段。使用 `comm_middleware.NewEntryMiddleware` 创建新的 `middleware`，将其注册进 `goa
.Service` 内。`middleware` 创建方法：

```go
func NewEntryMiddleware() goa.Middleware {
	
	return func(h goa.Handler) goa.Handler {
		return func(ctx context.Context, rw http.ResponseWriter, req *http.Request) error {
			goa.LogInfo(ctx, "NewEntryMiddleware entry")
			
			// real process
			err := h(ctx, rw, req)
			resp := goa.ContextResponse(ctx)
			code := resp.ErrorCode
			goa.LogInfo(ctx, "NewEntryMiddleware end", "status", resp.Status, "error", code,
				"bytes", resp.Length, "ctrl", goa.ContextController(ctx), "action", goa.ContextAction(ctx))
			
			return err
		}
	}
}
```
其中必须有 `err := h(ctx, rw, req)`，保证请求正常进行。

使用示例：
```go
package main

....

func main()  {
    service := goa.New("call_me")
    service.Use(comm_middleware.NewEntryMiddleware())
    ...
}
```

## 测试命令

接口测试命令：
```bash
curl -i -X GET \
    -H "X-CallMeHeader:Test" \
    -H 'Content-Type: application/json' \
    -d '{"name":"Beauty","age":18,"gender":"male","phone_no":"132xxxx2638"}' \
    "http://localhost:8080/meet/1"

# 压力测试
ab -n 1000 -c 10 \
    -p call_me.json -m GET \
    -T application/json \
    -H "X-CallMeHeader:Test" \
    -H "Content-Type: application/json" \
    "http://localhost:8080/meet/1"
```
`call_me.json` 中上报内容：
```json
{"name":"zhoucj","age":18,"gender":"male","phone_no":"13288882638"}
```

## 细节

在使用 `goa` 进行设计时发现在一个微服务项目中只能有一个 `API`、`Resource` 函数。还有另外一个问题，如果已经存在了 `Controller` 文件之后不会再生成 `Controller` 文件。取巧的解决方法，将生成的 `Controller` 文件保存到另外一个文件中，删除生成的 `Controller`；当更新了设计接口时可以直接重新运行 `goagen` 命令。

## 项目链接

- [demo]（https://github.com/juzipeek/call_me）

