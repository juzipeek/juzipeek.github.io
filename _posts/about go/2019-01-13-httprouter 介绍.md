---
layout: post
title: httprouter 介绍
category: Go
tags: Go
keywords: Go
description:
---

## 前言

Gin 使用 httprouter 用作请求路由。本文看下其实现以及与 NGINX 的 location 查找做对比。

## httprouter 使用

以下是 `httprouter` 的简单使用示例

```go
package main

import (
	"net/http"
	"os"
	
	"github.com/gogap/logrus"
	"github.com/julienschmidt/httprouter"
)

func init() {
	logrus.SetOutput(os.Stdout)
	logrus.SetLevel(logrus.DebugLevel)
	logrus.SetFormatter(&logrus.TextFormatter{})
}

func main() {
	r := httprouter.New()
	r.GET("/hi/:name/:act", comHandler)
	r.GET("/catch-all/*all", comHandler)
	http.ListenAndServe(":8080", r)
}

func comHandler(resp http.ResponseWriter, req *http.Request, params httprouter.Params) {
	logrus.Debugf("request client ip:%s uri:%s", req.RemoteAddr, req.RequestURI)
	for k, v := range req.Header {
		logrus.Debugf("header [%s]=[%s]", k, v)
	}
	
	for _, p := range params {
		logrus.Debugf("params [%s]=[%s]", p.Key, p.Value)
	}
	
	resp.Write([]byte("hello..."))
}
```

测试命令：

```shell
curl -i "http://localhost:8080/hi/zhoucj/sayhi"

curl -i "http://localhost:8080/catch-all/zhoucj/sayhi"
# 日志输出如下：
# DEBU[0006] params [all]=[/zhoucj/sayhi] 
```

在 `httprouter.router` 源文件中有注释：

```
// The router matches incoming requests by the request method and the path.
// If a handle is registered for this path and method, the router delegates the
// request to that function.
// For the methods GET, POST, PUT, PATCH and DELETE shortcut functions exist to
// register handles, for all other methods router.Handle can be used.
//
// The registered path, against which the router matches incoming requests, can
// contain two types of parameters:
//  Syntax    Type
//  :name     named parameter
//  *name     catch-all parameter
//
// Named parameters are dynamic path segments. They match anything until the
// next '/' or the path end:
//  Path: /blog/:category/:post
//
//  Requests:
//   /blog/go/request-routers            match: category="go", post="request-routers"
//   /blog/go/request-routers/           no match, but the router would redirect
//   /blog/go/                           no match
//   /blog/go/request-routers/comments   no match
//
// Catch-all parameters match anything until the path end, including the
// directory index (the '/' before the catch-all). Since they match anything
// until the end, catch-all parameters must always be the final path element.
//  Path: /files/*filepath
//
//  Requests:
//   /files/                             match: filepath="/"
//   /files/LICENSE                      match: filepath="/LICENSE"
//   /files/templates/article.html       match: filepath="/templates/article.html"
//   /files                              no match, but the router would redirect
//
// The value of parameters is saved as a slice of the Param struct, consisting
// each of a key and a value. The slice is passed to the Handle func as a third
// parameter.
// There are two ways to retrieve the value of a parameter:
//  // by the name of the parameter
//  user := ps.ByName("user") // defined by :user or *user
//
//  // by the index of the parameter. This way you can also get the name (key)
//  thirdKey   := ps[2].Key   // the name of the 3rd parameter
//  thirdValue := ps[2].Value // the value of the 3rd parameter
```

对于参数（动态）路由，`httprouter` 支持两种类型：命名参数路由、所有路由。

- 命名参数
> 将路径上的参数依次作为命名参数。`/hi/:name/:act` 会匹配 `/hi/joy/sayhi`，并且参数“name”、“act”依次为“joy”、“sayhi”。

- catch-all 路由

> 从路径索引开始匹配到路径截止。`/catch-all/*all` 会匹配以 `/catch-all` 开始的路径，`/catch-all` 之后的所有参数会被作为 `all` 参数处理。

## 实现

阅读 `httprouter` 源码，可以发现：
- Router 中每种 HTTP 方法单独一棵树
- Router.trees 存储路径对应的处理函数

## NGINX 静态 location 查找

在 NGINX 内部静态 location 以二叉树存储，查找时需要进行二叉树查找。