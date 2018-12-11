---
layout: post
title: GOA example-1
category: Go
tags: Go
keywords: Go
description:
---

## 概述

对 `goa` 框架示例 [Getting Started wit goa](https://goa.design/learn/guide/) 的翻译，看中文版本更方便。创建一个名叫 cellar （地窖）的服务：使用 GET 请求获取瓶子（bottle）模版是否存在。

## 依赖

安装 `goa` 和 `goagen`

```shell
go get -u github.com/goadesign/goa/...

```

## 设计

第一步，使用 `goa` 的 DSL 语言来描述 API 服务接口。在 `$GOPATH/src` 下创建一个项目目录，可以使用目录：`$GOPATH/src/cellar`。在 `cellar` 目录下创建一个子目录，用来保存接口设计文件，目录结构可以是 `$GOPATH/src/cellar/design/design.go`。 `design.go` 内容如下：

```go
package design                                     // 当前目录为 design 所以包名也该是 design

import (
        . "github.com/goadesign/goa/design"        // 使用 . 导入保证 DSL 功能启用；以下并没有使用这两个包
        . "github.com/goadesign/goa/design/apidsl"
)

var _ = API("cellar", func() {                     // API 函数定义了微服务的端点以及其他全局属性。在设计中必须只存在一个 API 定义。
        Title("The virtual wine cellar")           
        Description("A simple goa service")        
        Scheme("http")                             
        Host("localhost:8080")
})

var _ = Resource("bottle", func() {                // 使用 Resource 函数定义一个名为 bottle 的资源。
        BasePath("/bottles")                       
        DefaultMedia(BottleMedia)                  

        Action("show", func() {                    // 使用 Action 函数定义一个资源行为，资源行为定义在 Resource 内部。
                Description("Get bottle by id")    
                Routing(GET("/:bottleID"))         
                Params(func() {                    
                        Param("bottleID", Integer, "Bottle ID")
                })
                Response(OK)                       
                Response(NotFound)                 
        })
})

// BottleMedia defines the media type used to render bottles.
var BottleMedia = MediaType("application/vnd.goa.example.bottle+json", func() {
        Description("A bottle of wine")
        Attributes(func() {                         // Attributes define the media type shape.
                Attribute("id", Integer, "Unique bottle ID")
                Attribute("href", String, "API href for making requests on the bottle")
                Attribute("name", String, "Name of wine")
                Required("id", "href", "name")
        })
        View("default", func() {                    // View defines a rendering of the media type.
                Attribute("id")                     // Media types may have multiple views and must
                Attribute("href")                   // have a "default" view.
                Attribute("name")
        })
})

```

- API：API 函数定义了微服务的端点以及其他全局属性。在设计中必须只存在一个 API 定义。
- Resource：使用 Resource 函数定义一个名为 bottle 的资源。
- Action：使用 `Action` 函数定义行为端点、参数、包体和应答。goa 框架已经为标准 HTTP 状态码定义了默认的应答模版，应答模版描述了应答的 HTTP 状态码、内容类型和 HTTP 头。使用 `ResponseTemplate` 可以自定义应答模版。
- MediaType：使用 `MediaType` 函数定义资源类型。在之前的 `Resource` 中引用到全局变量 `BottleMedia`。

## 实现

使用 `goagen` 工具根据 `design.go` 的设计来生成微服务代码。使用以下命令，将微服务代码生成在目录 `$GOPATH/src/cellar` 中：

```shell
goagen bootstrap -d cellar/design

```

生成的微服务目录结构应该如下：

```
app
app/contexts.go
app/controllers.go
app/hrefs.go
app/media_types.go
app/user_types.go
app/test
app/test/bottle.go
main.go
bottle.go
tool/cellar-cli
tool/cli
client
client
tool/cellar-cli/main.go
tool/cli/commands.go
client/client.go
client/bottle.go
client/datatypes.go
swagger
swagger/swagger.json
swagger/swagger.yaml

```

`goagen` 创建的 `main.go` 作为应用， `bottle.go` 作为逻辑控制。可以从这两个文件进行新的开发，当重新运行 `goagen` 进行代码生成时这两个文件不会被覆盖，只有 `app`、`client`、`tool` 和 `swagger` 目录中的文件会重新生成。

目录功能说明：

- app：用于将 HTTP 请求路由到应用代码。
- client：客户端实现。
- tool：当前服务器（cellar）的命令行接口。
- swagger：JSON 和 YAML 格式定义的接口。

app 目录中文件介绍：

- controllers.go：controller 接口定义。设计中定义的每个 Resource 都有一个这样的接口，文件内还包含将这些控制器接口的实现“安装”到服务上的代码。
- contexts.go：包含 context 数据结构定义。Context 的作用类似 Martini 框架中的 `martini.Context`，goji 框架中的 `web.C` 或者 echo 框架的 `echo.Context`。使用一个不太准确的比喻，它们作为所有控制器操作的第一个参数给出，并提供帮助方法来访问请求状态并写入响应。
- hrefs.go：提供构建资源连接的全局函数。资源连接可以使响应链接到相关资源。goa知道如何通过查看资源“规范”操作的请求路径来构建这些 href。
- media_types.go：包含用于构建响应的媒体类型数据结构。
- user_types.go：包含通过 Type 设计语言函数定义的数据结构。这些类型可用于定义请求有包体和响应包体。
- test/bottle.go：包含测试助手，通过控制器输入调用动作实现并验证生成的媒体类型，可以方便地测试控制器代码。

现在 `goagen` 已经完成了设计工作，需要我们自己实现 `bottle` 控制器。生成的控制器如下：

```go
type BottleController interface {
        goa.Muxer
        Show(*ShowBottleContext) error
}

```

看下 `app/contexts.go` 中的 `ShowBottleContext`:

```go
// ShowBottleContext provides the bottle show action context.
type ShowBottleContext struct {
        context.Context
        *goa.ResponseData
        *goa.RequestData
        BottleID int
}

```

`ShowBottleContext` 上下文数据结构包含声明为 `int` 类型的 `BottleID`，这是在之前的 DSL 中定义的。它还包含匿名字段，用于访问原始底层请求和响应状态（包括访问 `http.Request` 和 `http.ResponseWriter` 对象）。`goa` 上下文数据结构还实现了 golang `context.Context` 接口，该接口使得可以例如在涉及处理请求的不同 goroutine 中携带截止时间和取消信号。

`app/contexts.go` 文件还定义了上下文数据结构的两个方法：

```go
// OK sends a HTTP response with status code 200.
func (ctx *ShowBottleContext) OK(r *GoaExampleBottle) error {
        ctx.ResponseData.Header().Set("Content-Type", "application/vnd.goa.example.bottle")
        return ctx.Service.Send(ctx.Context, 200, r)
}

// NotFound sends a HTTP response with status code 404.
func (ctx *ShowBottleContext) NotFound() error {
        ctx.ResponseData.WriteHeader(404)
        return nil
}

```

`goagen` 还在 `bottle.go` 中生成了一个空的控制器实现，所以我们剩下要做的就是提供一个实际的实现。打开文件 `bottle.go` 并用现有的 `Show` 方法替换：

```go
// Show implements the "show" action of the "bottles" controller.
func (c *BottleController) Show(ctx *app.ShowBottleContext) error {
        if ctx.BottleID == 0 {
                // Emulate a missing record with ID 0
                return ctx.NotFound()
        }
        // Build the resource using the generated data structure
        bottle := app.GoaExampleBottle{
                ID:   ctx.BottleID,
                Name: fmt.Sprintf("Bottle #%d", ctx.BottleID),
                Href: app.BottleHref(ctx.BottleID),
        }

        // Let the generated code produce the HTTP response using the
        // media type described in the design (BottleMedia).
        return ctx.OK(&bottle)
}

```

在我们构建和运行应用程序之前，让我们先来看看 `main.go`：该文件包含 `main` 的默认实现，它实例化一个新的 `goa` 服务，初始化默认中间件，安装 `bottle` 控制器并运行 HTTP 服务器。

```go
func main() {
        // Create service
        service := goa.New("cellar")

        // Mount middleware
        service.Use(middleware.RequestID())
        service.Use(middleware.LogRequest(true))
        service.Use(middleware.ErrorHandler(service, true))
        service.Use(middleware.Recover())

        // Mount "bottle" controller
        c := NewBottleController(service)
        app.MountBottleController(service, c)

        // Start service
        if err := service.ListenAndServe(":8080"); err != nil {
                service.LogError("startup", "err", err)
        }
}

```

将控制器安装到服务上会将所有控制器操作端点注册到路由器。该代码还确保不同操作的路由之间不存在冲突。

现在编译并运行服务：

```go
go build -o cellar
./cellar

```

应该会有如下输出：

```shell
2016/04/10 16:20:37 [INFO] mount ctrl=Bottle action=Show route=GET /bottles/:bottleID
2016/04/10 16:20:37 [INFO] listen transport=http addr=:8080

```

使用 `curl` 进行测试：

```shell
curl -i localhost:8080/bottles/1

HTTP/1.1 200 OK
Content-Type: application/vnd.goa.example.bottle
Date: Sun, 10 Apr 2016 23:21:19 GMT
Content-Length: 48

{"href":"/bottles/1","id":1,"name":"Bottle #1"}

```

您可以通过传递无效（非整数）id 来执行由 goagen 生成的验证代码：

```shell
curl -i localhost:8080/bottles/n

HTTP/1.1 400 Bad Request
Content-Type: application/vnd.api.error+json
Date: Sun, 10 Apr 2016 23:22:46 GMT
Content-Length: 117

{"code":"invalid_request","status":400,"detail":"invalid value \"n\" for parameter \"bottleID\", must be a integer"}

```

我们不使用 `curl`，而是使用生成的 `CLI` 工具向服务发出请求。首先让我们编译 `CLI` 客户端：

```shell
cd tool/cellar-cli
go build -o cellar-cli
./cellar-cli

```

下面的命令打印 cli 用法， `-help` 标志还提供上下文帮助：

```shell
./cellar-cli show bottle --help

```

下面的命令显示了如何调用 `show bottle` 动作：

```shell
./cellar-cli show bottle /bottles/1
2016/04/10 16:26:34 [INFO] started id=Vglid/lF GET=http://localhost:8080/bottles/1
2016/04/10 16:26:34 [INFO] completed id=Vglid/lF status=200 time=773.145µs
{"href":"/bottles/1","id":1,"name":"Bottle #1"}

```

项目示例结束。