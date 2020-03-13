---
layout: post
title: Working with Data Types
category: Go
tags: Go
keywords: Go,Goa
description:GOA Design Language
---

## 1. 概述

goa DSL 的一个重要方面在于如何定义和使用类型。[概述](https://goa.design/design/overview/)介绍了使用类型和媒体类型的基础知识，本文档退后一步，解释了 DSL 的基本原理。

设计中使用 Attribute 函数或其别名之一（Member，Header 或 Param）描述数据结构。这种描述客观存在 - 即它与给定语言（例如Go）或技术无关。这使得可以生成许多输出，范围从 Go 代码到 JSON 格式和 Swagger 形式的文档或绑定到其他语言（例如 JavaScript 客户端）。DSL 包括概述中列出的许多基本类型，但也可以通过 `Type` 函数递归地描述任意数据结构。

## 2. 请求包体

以这种方式定义的类型的主要应用之一是描述给定动作的请求包体，请求包体描述请求主体的类型。`goagen` 使用该描述生成操作方法在其请求上下文中接收的相应 `Go` 结构。有效负载可以内联定义如下：

```go
Action("create", func() {
	Routing(POST(""))
	Payload(func() {
		Member("name")
	})
	Response(Created, "/accounts/[0-9]+")
})
```

或者可以使用预先定义的类型：

```go
var CreatePayload = Type("CreatePayload", func() {
	Attribute("name")
})

Action("create", func() {
	Routing(POST(""))
	Payload(CreatePayload)
	Response(Created, "/accounts/[0-9]+")
})
```

前一种表示法最终会创建一个由生成器内部使用的匿名类型。请注意，DSL 是递归的，上面的示例没有指定 `name` 属性的类型，因此默认为 `String`。但它可以使用任何其他类型，包括内联定义的数据结构或通过预定义类型：

```go
Action("create", func() {
	Routing(POST(""))
	Payload(func() {
		Member("address", func() {
			Attribute("street")
			Attribute("number", Integer)
		})
	})
	Response(Created, "/accounts/[0-9]+")
})
```

或者：

```go
Action("create", func() {
	Routing(POST(""))
	Payload(func() {
		Member("name", Address) // where Address is a predefined type
	})
	Response(Created, "/accounts/[0-9]+")
})
```

无论在何处使用属性，都存在相同的灵活性。

可以使用如上所示的变量或使用其名称来引用预定义类型，即 `Payload(CreatePayload)` 也可以写为 `Payload("CreatePayload")`，因为 “CreatePayload” 是 CreatePayload 类型定义中给出的名称。他可以定义相互依赖的类型，而不会让 Go 编译器出现依赖循环。

## 3. 媒体类型

类型的另一个常见用途是描述响应媒体类型。响应媒体类型定义响应包体的类型。媒体类型与类型的不同之处在于它们还定义了视图和链接，有关详细信息请参阅[概述](https://goa.design/design/overview/)。基本媒体类型定义可能如下所示：

```go
var MT = MediaType("application/vnd.app.mt", func() {
	Attributes(func() {
		Attribute("name")
	})
	View("default", func() {
		Attribute("name")
	})
})
```

`MediaType` 的第一个参数是 [RFC 6838](https://tools.ietf.org/html/rfc6838) 定义的媒体类型标识符。DSL 列出的属性类似于如何在类型中定义属性，视图以及可选的链接到其他媒体类型，这里有 `default` 默认视图。然后可以使用这种媒体类型来定义动作的响应，如下所述：

```go
Action("show", func() {
	Routing(GET("/:accountID"))
	Params(func() {
		Param("accountID", Integer, "Account ID")
	})
	Response(OK, func() {
		Media(MT)
	})
	Response(NotFound)
})
```

这相当于：

```go
Action("show", func() {
	Routing(GET("/:accountID"))
	Params(func() {
		Param("accountID", Integer, "Account ID")
	})
	Response(OK, MT)
	Response(NotFound)
})
```

或者这样：

```go
Action("show", func() {
	Routing(GET("/:accountID"))
	Params(func() {
		Param("accountID", Integer, "Account ID")
	})
	Response(OK, "application/vnd.app.mt")
	Response(NotFound)
})
```

可以使用变量或媒体类型标识符来引用媒体类型，类似于如何使用变量或其名称引用类型。

通常情况是使用相同的属性来定义动作请求包体和响应媒体类型。REST API 尤其如此，其中在响应中返回它的表示形式与请求的形式相同。goa 设计语言通过提供可以在 `Type` 和 `MediaType` 函数调用中使用的 `Reference` 函数来帮助解决这种常见情况。此函数接受一个参数，该参数可以是包含类型或媒体类型的变量，也可以是类型或媒体类型的名称。使用 `Reference` 函数可以重用所引用类型的属性，而无需重新定义其所有属性（类型，描述，示例，验证等）。给出以下类型定义：

```go
var CreatePayload = Type("CreatePayload", func() {
	Attribute("name", String, "Name of thingy", func() {
		MinLength(5)
		MaxLength(256)
		Pattern("^[a-zA-Z]([a-zA-Z ]+)")
	})
})
```

媒体类型可以利用 `name` 属性定义，如下所示：

```go
var MT = MediaType("application/vnd.app.mt", func() {
	Reference(CreatePayload)
	Attributes(func() {
		Attribute("name")
	})
	View("default", func() {
		Attribute("name")
	})
})
```

`name` 属性自动继承相应 `CreatePayload` 属性中定义的类型，描述和验证。请注意，媒体类型定义仍然需要列出所引用的属性的名称，这使得可以选择要“继承”的属性。如果需要，媒体类型也可以覆盖 `name` 属性的属性（例如，改变类型，描述，验证等）。

媒体类型也可以使用媒体类型标识符来引用自己。这使得定义递归媒体类型成为可能，并且不让 Go 编译器产生循环依赖：

```go
var MT = MediaType("application/vnd.app.mt", func() {
	Reference(CreatePayload)
	Attributes(func() {
		Attribute("name")
		Attribute("children", CollectionOf("application/vnd.app.mt"))
	})
	View("default", func() {
		Attribute("name")
	})
})
```

## 4. 混合 `Type` 和 `MediaType`

我们已经看到了 `Reference` 如何在类型和媒体类型之间重用属性定义，媒体类型是一种特殊的类型。这意味着它们可以用于代替可以使用 `Type` 的类型（在定义属性的任何地方）。最佳实践包括仅使用媒体类型来定义响应主体，如上所示，并使用其他所有类型。这是因为媒体类型定义了其他属性，例如仅适用于该用例的标识符，视图和链接。因此，在 `Type` 和 `MediaType` 之间共享相同的属性时，定义 `Type` 并利用 `Reference` 关键字。