---
layout: post
title: grpc gateway 介绍
category: Go
tags: Go,grpc-gateway
keywords: Go,grpc-gateway
description:
---

## 一 概述

本文主要介绍 `grpc-gateway` 使用以及实现原理。

## 二 grpc-gateway 基础

### 1. 是什么

`grpc-gateway` 是 [`protobuf`](https://developers.google.com/protocol-buffers/) 生成器 [`protoc`](https://github.com/protocolbuffers/protobuf) 的一个插件。

### 2. 能做什么

在 `protobuf` 的定义文件中添加 `google.api.http` 注解，`grpc-gateway` 可以读取 `protobuf` 定义文件并且生成反向代理服务，**用来将 `RESTful JSON` 接口转换为`gRPC` 格式**。

### 3. 安装

`grpc-gateway` 依赖 `ProtocolBuffer`，需要先安装 `ProtoBuffer`：

```bash
mkdir tmp
cd tmp
# or wget https://github.com/protocolbuffers/protobuf/archive/v3.7.1.tar.gz
git clone https://github.com/google/protobuf
cd protobuf
./autogen.sh
./configure
make
make check
sudo make install
```

安装 `grpc-gateway`：

```bash
# 下载 genproto
cd $GOPATH/src/google.golang.org
git clone https://github.com/google/go-genproto.git
mv go-genproto/ genproto/

# gopm get -v github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
go install github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway

# gopm get -v github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger
go install github.com/grpc-ecosystem/grpc-gateway/protoc-gen-swagger

# gopm get -v github.com/golang/protobuf/protoc-gen-go
go get -u github.com/golang/protobuf/protoc-gen-go
go install github.com/golang/protobuf/protoc-gen-go
```

## 三 完整示例

创建一个 SayHi 服务，接收 `name` 参数并返回 `echo` 消息。

### 1.  定义 `gRPC` 服务

创建 `say_hi.proto` 文件，文件内容如下：

```go
syntax = "proto3";

package endpoint;

import "google/api/annotations.proto"; // 导入注解包

message HiReq {
	string name = 1;
}

message HiResp {
	string echo = 2;
}

service SayHi {
    rpc SayHi (HiReq) returns (HiResp) {
        // SayHi 接口的 RESTful 格式选项
        option (google.api.http) = {
        	post: "/api/sayhi"
        	body: "*"
        };
    }
}
```

### 2. 生成 `gRPC` 桩代码

```bash
protoc -I/usr/local/include -I. \
	-I/home/zhoucj/Project/go_play/src \
	-I/home/zhoucj/Project/go_play/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
	--go_out=plugins=grpc:. proto/say_hi.proto
```

执行完命令后会生成 `proto/say_hi.pb.go` 桩代码文件。

### 3. 生成反向代理服务

```bash
protoc -I/usr/local/include -I. \
	-I/home/zhoucj/Project/go_play/src \
	-I/home/zhoucj/Project/go_play/src/github.com/grpc-ecosystem/grpc-gateway/third_party/googleapis \
	--grpc-gateway_out=logtostderr=true:. \
	proto/say_hi.proto
```

执行完命令后将生成 `proto/sai_hi.pb.gw.go` 反向代理文件。

### 4.  实现服务端接口

```go
package srv

import (
	"context"
	"fmt"
	
	"endpoint/proto"
)

func NewSayHiSrv() *SayHiSrv {
	return &SayHiSrv{}
}

type SayHiSrv struct {
}

// 服务实现接口
func (s *SayHiSrv) SayHi(c context.Context, req *endpoint.HiReq) (*endpoint.HiResp, error) {
	fmt.Println("request name:", req.Name)
	msg := fmt.Sprintf("%s nice to meet u.", req.Name)
	resp := &endpoint.HiResp{Echo: msg}
	fmt.Println(msg)
	return resp, nil
}
```
以上只需按一般方式实现 `grpc server`。

### 5. 将反向代理服务与 `grpc server` 关联

```go
package main

import (
	"context"
	"fmt"
	"net"
	"net/http"
	
	gw "endpoint/proto"
	"srv"
	
	"github.com/grpc-ecosystem/grpc-gateway/runtime"
	"google.golang.org/grpc"
)

const (
	grpcAddr = ":8090" // grpc 服务端口
	httpAddr = ":8080" // http 协议端口
)

func main() {
	
	{
		// grpc server
		rpcServer := grpc.NewServer()
		// 注册 grpc 处理函数
		gw.RegisterSayHiServer(rpcServer, srv.NewSayHiSrv())
		
		lis, _ := net.Listen("tcp", grpcAddr)
		go rpcServer.Serve(lis)
	}
	
	{
		// grpc gateway server
		mux := runtime.NewServeMux()
		ctx, _ := context.WithCancel(context.Background())
		opts := []grpc.DialOption{grpc.WithInsecure()}
		
		// 注册反向代理服务
		err := gw.RegisterSayHiHandlerFromEndpoint(ctx, mux, grpcAddr, opts)
		if err != nil {
			fmt.Println("RegisterSayHiHandlerFromEndpoint error:", err.Error())
			return
		}
		// 启动反向代理服务
		http.ListenAndServe(httpAddr, mux)
	}
}

```

### 6. 编译测试

```bash
go build main.go
curl -i -X POST -d '{"name":"jay"}' "http://localhost:8080/api/sayhi"

# 可以看到有 8090、8080 两个端口被监听
netstat -ntl
```

## 四 深入`grpc-gateway` 内部实现

在*完整示例*部分展示了如何将 `HTTP` 服务与 `grpc` 服务关联起来，其中的关键代码在下面：

```go
err := gw.RegisterSayHiHandlerFromEndpoint(ctx, mux, grpcAddr, opts)
```

其调用过程为：
```flow
st=>start: main
e=>end: End
op1=>operation: RegisterSayHiHandlerFromEndpoint
op2=>operation: RegisterSayHiHandler
op3=>operation: RegisterSayHiHandlerClient

st->op1->op2->op3->e
```

### 1. `RegisterSayHiHandlerFromEndpoint`

```go
// RegisterSayHiHandlerFromEndpoint is same as RegisterSayHiHandler but
// automatically dials to "endpoint" and closes the connection when "ctx" gets done.
func RegisterSayHiHandlerFromEndpoint(ctx context.Context, mux *runtime.ServeMux, endpoint string, opts []grpc.DialOption) (err error) {
	conn, err := grpc.Dial(endpoint, opts...)
	if err != nil {
		return err
	}
	defer func() {
		if err != nil {
			if cerr := conn.Close(); cerr != nil {
				grpclog.Infof("Failed to close conn to %s: %v", endpoint, cerr)
			}
			return
		}
		go func() {
			<-ctx.Done()
			if cerr := conn.Close(); cerr != nil {
				grpclog.Infof("Failed to close conn to %s: %v", endpoint, cerr)
			}
		}()
	}()

	return RegisterSayHiHandler(ctx, mux, conn)
}
```

这个函数的主要功能是创建一个 `grpc` 连接 `conn`，并将其作为参数传递给 `RegisterSayHiHandler` 函数，`ctx` 在整个调用过链中是为了保证在主调用协程关闭时（调用了 `cancel` 方法）进行资源清理。

### 2. `RegisterSayHiHandlerClient`

```go
// RegisterSayHiHandler registers the http handlers for service SayHi to "mux".
// The handlers forward requests to the grpc endpoint over "conn".
func RegisterSayHiHandler(ctx context.Context, mux *runtime.ServeMux, conn *grpc.ClientConn) error {
	return RegisterSayHiHandlerClient(ctx, mux, NewSayHiClient(conn))
}
```

### 3. `RegisterSayHiHandlerClient`

```go
// RegisterSayHiHandlerClient registers the http handlers for service SayHi
// to "mux". The handlers forward requests to the grpc endpoint over the given implementation of "SayHiClient".
// Note: the gRPC framework executes interceptors within the gRPC handler. If the passed in "SayHiClient"
// doesn't go through the normal gRPC flow (creating a gRPC client etc.) then it will be up to the passed in
// "SayHiClient" to call the correct interceptors.
func RegisterSayHiHandlerClient(ctx context.Context, 
                                mux *runtime.ServeMux, client SayHiClient) error {

	mux.Handle("POST", 
               pattern_SayHi_SayHi_0, 
               func(w http.ResponseWriter, 
                    req *http.Request, 
                    pathParams map[string]string) {

		ctx, cancel := context.WithCancel(req.Context())
		defer cancel()

		// 将 RESTful 请求转换为 protobuf 格式
		inboundMarshaler, outboundMarshaler := runtime.MarshalerForRequest(mux, req)
		rctx, err := runtime.AnnotateContext(ctx, mux, req)
		if err != nil {
			runtime.HTTPError(ctx, mux, outboundMarshaler, w, req, err)
			return
		}

		// 调用 grpc 接口进行处理
		resp, md, err := request_SayHi_SayHi_0(rctx, inboundMarshaler, client, req, pathParams)
		ctx = runtime.NewServerMetadataContext(ctx, md)
		if err != nil {
			runtime.HTTPError(ctx, mux, outboundMarshaler, w, req, err)
			return
		}

		// 将 protobuf 应答转换 RESTful 格式应答并发送给客户端
		forward_SayHi_SayHi_0(ctx, mux, 
                              outboundMarshaler, 
                              w, req, resp, 
                              mux.GetForwardResponseOptions()...)
	})

	return nil
}

// Handle associates "h" to the pair of HTTP method and path pattern.
func (s *ServeMux) Handle(meth string, pat Pattern, h HandlerFunc) {
	s.handlers[meth] = append(s.handlers[meth], handler{pat: pat, h: h})
}
```

本函数是功能的核心，用来将服务接口路径（`pattern_SayHi_SayHi_0`）与处理函数相关联。处理函数注意有分为三个部分：将 RESTful 请求转换为 protobuf 格式、调用 grpc 接口进行处理、将 protobuf 应答转换 RESTful 格式应答并发送给客户端。

### 4. 处理 `HTTP` 请求

以上 1 - 3 都在将请求之前的配置过程，在 `HTTP` 请求到来时怎么做呢？其实核心还是在 `ServeMux` 中，`ServeMux` 实现了 `net/http.Handler` 接口，主程序中已经通过代码 `http.ListenAndServe(httpAddr, mux)` 将 `mux` 设置为 `HTTP` 处理接口。

```go
// ServeHTTP dispatches the request to the first handler whose pattern matches to r.Method and r.Path.
func (s *ServeMux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

    // 1. 请求格式校验
	path := r.URL.Path
	if !strings.HasPrefix(path, "/") {
		if s.protoErrorHandler != nil {
			_, outboundMarshaler := MarshalerForRequest(s, r)
			sterr := status.Error(codes.InvalidArgument, http.StatusText(http.StatusBadRequest))
			s.protoErrorHandler(ctx, s, outboundMarshaler, w, r, sterr)
		} else {
			OtherErrorHandler(w, r, http.StatusText(http.StatusBadRequest), http.StatusBadRequest)
		}
		return
	}

    // 路径解析
	components := strings.Split(path[1:], "/")
	l := len(components)
	var verb string
	if idx := strings.LastIndex(components[l-1], ":"); idx == 0 {
		if s.protoErrorHandler != nil {
			_, outboundMarshaler := MarshalerForRequest(s, r)
			sterr := status.Error(codes.Unimplemented, http.StatusText(http.StatusNotImplemented))
			s.protoErrorHandler(ctx, s, outboundMarshaler, w, r, sterr)
		} else {
			OtherErrorHandler(w, r, http.StatusText(http.StatusNotFound), http.StatusNotFound)
		}
		return
	} else if idx > 0 {
		c := components[l-1]
		components[l-1], verb = c[:idx], c[idx+1:]
	}

	if override := r.Header.Get("X-HTTP-Method-Override"); override != "" && s.isPathLengthFallback(r) {
		r.Method = strings.ToUpper(override)
		if err := r.ParseForm(); err != nil {
			if s.protoErrorHandler != nil {
				_, outboundMarshaler := MarshalerForRequest(s, r)
				sterr := status.Error(codes.InvalidArgument, err.Error())
				s.protoErrorHandler(ctx, s, outboundMarshaler, w, r, sterr)
			} else {
				OtherErrorHandler(w, r, err.Error(), http.StatusBadRequest)
			}
			return
		}
	}
    
    // 根据路径匹配处理函数
	for _, h := range s.handlers[r.Method] {
		pathParams, err := h.pat.Match(components, verb)
		if err != nil {
			continue
		}
		h.h(w, r, pathParams)
		return
	}

    
	// lookup other methods to handle fallback from GET to POST and
	// to determine if it is MethodNotAllowed or NotFound.
	for m, handlers := range s.handlers {
		if m == r.Method {
			continue
		}
		for _, h := range handlers {
			pathParams, err := h.pat.Match(components, verb)
			if err != nil {
				continue
			}
			// X-HTTP-Method-Override is optional. Always allow fallback to POST.
			if s.isPathLengthFallback(r) {
				if err := r.ParseForm(); err != nil {
					if s.protoErrorHandler != nil {
						_, outboundMarshaler := MarshalerForRequest(s, r)
						sterr := status.Error(codes.InvalidArgument, err.Error())
						s.protoErrorHandler(ctx, s, outboundMarshaler, w, r, sterr)
					} else {
						OtherErrorHandler(w, r, err.Error(), http.StatusBadRequest)
					}
					return
				}
				h.h(w, r, pathParams)
				return
			}
            
			if s.protoErrorHandler != nil {
				_, outboundMarshaler := MarshalerForRequest(s, r)
				sterr := status.Error(codes.Unimplemented, http.StatusText(http.StatusMethodNotAllowed))
				s.protoErrorHandler(ctx, s, outboundMarshaler, w, r, sterr)
			} else {
				OtherErrorHandler(w, r, http.StatusText(http.StatusMethodNotAllowed), http.StatusMethodNotAllowed)
			}
			return
		}
	}

	if s.protoErrorHandler != nil {
		_, outboundMarshaler := MarshalerForRequest(s, r)
		sterr := status.Error(codes.Unimplemented, http.StatusText(http.StatusNotImplemented))
		s.protoErrorHandler(ctx, s, outboundMarshaler, w, r, sterr)
	} else {
		OtherErrorHandler(w, r, http.StatusText(http.StatusNotFound), http.StatusNotFound)
	}
}
```

## 五 总结

`grpc-gateway` 的本质是启动另外一个 `HTTP` 服务，用来将 `HTTP` 协议请求转换为 `grpc` 格式请求。

## 六 其他说明

### `ServeMux`

ServeMux(`grpc-ecosystem/grpc-gateway/runtime.ServeMux`)是 `grpc-gateway` 的请求多路复用器，用来匹配 `HTTP` 请求并调用相应的处理函数。
