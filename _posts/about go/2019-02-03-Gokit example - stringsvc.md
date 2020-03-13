---
layout: post
title: Gokit example - stringsvc
category: Go
tags: Go
keywords: Go,Gokit
description:Gokit 示例
---

## 一 极简 stringsvc 微服务

使用单个 main.go 文件构建一个 stringsvc 微服务，提供以下两个功能：

-   Uppercase

    > 将字符串转换为大写

-   Count

    > 计算字符串长度

### 1 服务接口定义

首先将微服务功能抽象为接口：

```go
type StringService interface {
    Uppercase(string) (string, error)
    Count(string) int
}
```

接口实现（服务实现）：

```go
import (
    "context"
    "errors"
    "strings"
)
type stringService struct{}

func (stringService) Uppercase(s string) (string, error) {
    if s == "" {
        return "",ErrEmpty
    }
    return strings.ToUpper(s), nil
}

func (stringService) Count(s string) int {
    return len(s)
}

var ErrEmpty = errors.New("Empty string")
```

### 2 请求与应答

在 Gokit 中主要的消息传递模式是 RPC，因此我们接口中的每个方法都将被建模为远程过程调用。对于每个方法，我们定义请求和响应结构，分别捕获所有输入和输出参数。

```go
type uppercaseRequest struct {
    S string `json:"s"`
}

type uppercaseResponse struct {
    V string `json:"v"`
    Err string `json:"err,omitempty"` // errors do not JSON-marshal, so we use a string
}

type countRequest struct {
    S string `json:"s"`
}

type countResponse struct {
    V int `json:"v"`
}
```

### 3 端点

Gokit 的绝大多数功能是通过在抽象的端点层提供的。端点定义如下：

```go
type EndPoint func(ctx context.Context, request interface{})(response interface{}, err error)
```

一个端点代表一个 RPC，也就是说我们的服务接口中有个一方法。我们编写简单的适配器，将每个服务的方法转换为端点。每个适配器都接受一个 StringService，并返回一个与其中一个方法对应的端点。

```go
import (
	"context"
	"github.com/go-kit/kit/endpoint"
)

func makeUppercaseEndpoint(svc StringService) endpoint.Endpoint {
    return func(_ context.Context, request interface{}) (interface {}, error) {
        req := request.(uppercaseRequest)
        v, err := svc.Uppercase(req.S)
        if err != nil {
            return uppercaseResponse{v, err.Error()}, nil
        }
        return uppercaseResponse{v, ""}, nil
    }
}

func makeCountEndPoint(svc StringService) endpoint.Endpoint {
    return func(_ context.Context, request interface{}) (interface{}, error) {
        req := request.(countRequest)
        v := svc.Count(req.S)
        return countResponse{v}, nil
    }
}
```

### 4 传输层设置

现在需要将服务暴露，以便调用。以下代码使用 json over HTTP 方式暴露服务：
```go
import (
	"context"
	"encoding/json"
	"log"
	"net/http"
	
	httptransport "github.com/go-kit/kit/transport/http"
)

func main(){
    svc := stringService{}
    
    uppercaseHandler := httptransport.NewServer(
    	makeUppercaseEndpoint(svc),
    	decodeUppercaseRequest,
    	encodeResponse,
    )
    
    countHandler := httptransport.NewServer(
    	makeCountEndpoint(svc),
    	decodeCountRequest,
    	encodeResponse,
    )
    
    http.Handle("/uppercase", uppercaseHandler)
    http.Handle("/count", countHandler)
    log.Fatal(http.ListenAndServe(":8080", nil))
}

func decodeUppercaseRequest(_ context.Context, r *http.Request) (interface{}, error) {
    var request uppercaseRequest
    if err := json.NewDecoder(r.Body).Decode(&request); err !=nil {
        return nil, err
    }
    return request, nil
}

func decodeCountRequest(_ context.Context, r *http.Request)(interface{}, error) {
    var request countRequest
    if err := json.NewDecoder(r.Body).Decode(&request); err != nil {
        return nil, err
    }
    return request, nil
}

func encodeResponse(_ context.Context, w http.ResponseWriter, response interface{}) error {
    return json.NewEncoder(w).Encode(response)
}
```

可以下载[完整示例](https://github.com/go-kit/kit/blob/master/examples/stringsvc1/main.go).执行命令运行服务：

```bash
go run stringsvc1.go

# 测试
curl -X POST -d'{"s":"hello, world"}' localhost:8080/uppercase
curl -X POST -d'{"s":"hello, world"}' localhost:8080/count
```

## 二 添加中间件

### 1. 关注点分割

将调用流程的每一层分割成单独的文件，可以在增加服务端点时更轻松的阅读 go-kit 项目。在上面的示例中我们将所有的层都放在同一个 main.go 文件中，在我们增加更多复杂代码之前将代码分割成以下文件，并将所有代码保留在 main.go 中。

将服务层相关的类型和功能放到 service.go 文件中：

```go
type StringService
type stringService
var ErrEmpty
```

将传输层相关的类型和功能放到 transport.go 文件中：

```go
func makeUppercaseEndpoint
func makeCountEndpoint
func decodeUppercaseRequest
func decodeCountRequest
func encodeResponse
type uppercaseRequest
type uppercaseResponse
type countRequest
type countResponse
```

### 2. 传输层日志

任何需要进行日志记录的组件需要将 logger 当做一项依赖，与数据库连接相同。因此我们在 main 函数中创建 logger 并将其传递给其他组件，而不是使用全局 logger。

在 stringService 中，我们可以直接将 logger 传递给 stringService 对象，也可以使用中间件（或者叫做装饰器）模式。所谓的中间件就是入参是 Endpoint 返回值同样是 Endpoint：

```go
type Middleware func(Endpoint) Endpoint
```
> go-kit 已经定义 Middleware，不再需要自己定义。

以下是一个基础的日志中间件实现：

```go
func loggingMiddleware(logger log.Logger) Middleware {
	return func(next endpoint.Endpoint) endpoint.Endpoint {
		return func(_ context.Context, request interface{}) (interface{}, error) {
			logger.Log("msg", "calling endpoint")
			defer logger.Log("msg", "called endpoint")
			return next(request)
		}
	}
}
```

将中间件添加到处理函数中：

```go
logger := log.NewLogfmtLogger(os.Stderr)

svc := stringService{}

var uppercase endpoint.Endpoint
uppercase = makeUppercaseEndpoint(svc)
uppercase = loggingMiddleware(log.With(logger, "method", "uppercase"))(uppercase)

var count endpoint.Endpoint
count = makeCountEndpoint(svc)
count = loggingMiddleware(log.With(logger, "method", "count"))(count)

uppercaseHandler := httptransport.NewServer(
	// ...
	uppercase,
	// ...
)

countHandler := httptransport.NewServer(
	// ...
	count,
	// ...
)
```

### 3. 服务层日志

我们同样可以在服务层定义中间件，既然 StringService 定义为一个接口，我们只需要定义一个新类型对原有 StringService 进行包裹，并负责日志职责。代码如下：

```go
type loggingMiddleware struct {
	logger log.Logger
	next   StringService
}

func (mw loggingMiddleware) Uppercase(s string) (output string, err error) {
	defer func(begin time.Time) {
		mw.logger.Log(
			"method", "uppercase",
			"input", s,
			"output", output,
			"err", err,
			"took", time.Since(begin),
		)
	}(time.Now())

	output, err = mw.next.Uppercase(s)
	return
}

func (mw loggingMiddleware) Count(s string) (n int) {
	defer func(begin time.Time) {
		mw.logger.Log(
			"method", "count",
			"input", s,
			"n", n,
			"took", time.Since(begin),
		)
	}(time.Now())

	n = mw.next.Count(s)
	return
}
```

并将其添加到 main 函数中：

```go
import (
	"os"

	"github.com/go-kit/kit/log"
	httptransport "github.com/go-kit/kit/transport/http"
)

func main() {
	logger := log.NewLogfmtLogger(os.Stderr)

	var svc StringService
	svc = stringService{}
	svc = loggingMiddleware{logger, svc}

	// ...

	uppercaseHandler := httptransport.NewServer(
		// ...
		makeUppercaseEndpoint(svc),
		// ...
	)

	countHandler := httptransport.NewServer(
		// ...
		makeCountEndpoint(svc),
		// ...
	)
}
```

在端点层使用中间件可以关注于传输域中的问题，例如断路或速率限制。在服务层使用中间件可以关注于业务域中的问题，例如日志记录、仪表盘。仪表盘中间件：

```go
type instrumentingMiddleware struct {
	requestCount   metrics.Counter
	requestLatency metrics.Histogram
	countResult    metrics.Histogram
	next           StringService
}

func (mw instrumentingMiddleware) Uppercase(s string) (output string, err error) {
	defer func(begin time.Time) {
		lvs := []string{"method", "uppercase", "error", fmt.Sprint(err != nil)}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
	}(time.Now())

	output, err = mw.next.Uppercase(s)
	return
}

func (mw instrumentingMiddleware) Count(s string) (n int) {
	defer func(begin time.Time) {
		lvs := []string{"method", "count", "error", "false"}
		mw.requestCount.With(lvs...).Add(1)
		mw.requestLatency.With(lvs...).Observe(time.Since(begin).Seconds())
		mw.countResult.Observe(float64(n))
	}(time.Now())

	n = mw.next.Count(s)
	return
}
```

将其集成到服务中：

```go
import (
	stdprometheus "github.com/prometheus/client_golang/prometheus"
	kitprometheus "github.com/go-kit/kit/metrics/prometheus"
	"github.com/go-kit/kit/metrics"
)

func main() {
	logger := log.NewLogfmtLogger(os.Stderr)

	fieldKeys := []string{"method", "error"}
	requestCount := kitprometheus.NewCounterFrom(stdprometheus.CounterOpts{
		Namespace: "my_group",
		Subsystem: "string_service",
		Name:      "request_count",
		Help:      "Number of requests received.",
	}, fieldKeys)
	requestLatency := kitprometheus.NewSummaryFrom(stdprometheus.SummaryOpts{
		Namespace: "my_group",
		Subsystem: "string_service",
		Name:      "request_latency_microseconds",
		Help:      "Total duration of requests in microseconds.",
	}, fieldKeys)
	countResult := kitprometheus.NewSummaryFrom(stdprometheus.SummaryOpts{
		Namespace: "my_group",
		Subsystem: "string_service",
		Name:      "count_result",
		Help:      "The result of each count method.",
	}, []string{}) // no fields here

	var svc StringService
	svc = stringService{}
	svc = loggingMiddleware{logger, svc}
	svc = instrumentingMiddleware{requestCount, requestLatency, countResult, svc}

	uppercaseHandler := httptransport.NewServer(
		makeUppercaseEndpoint(svc),
		decodeUppercaseRequest,
		encodeResponse,
	)

	countHandler := httptransport.NewServer(
		makeCountEndpoint(svc),
		decodeCountRequest,
		encodeResponse,
	)

	http.Handle("/uppercase", uppercaseHandler)
	http.Handle("/count", countHandler)
	http.Handle("/metrics", promhttp.Handler())
	logger.Log("msg", "HTTP", "addr", ":8080")
	logger.Log("err", http.ListenAndServe(":8080", nil))
}
```

整个服务代码示例[在此](https://github.com/go-kit/kit/blob/master/examples/stringsvc2)。

## 三 调用其他服务

假设我们的字符串服务需要使用其他字符串服务来实现 Uppercase 方法，我们使用代理将请求转向另外一个服务，采用与日志中间件相同方式，我们将代理中间件实现为服务中间件。

```go
// proxmw 实现 StringService，同时将 Uppercase 请求转发到 uppercase 端点，并使用 StringService 类型的 next 成员处理其他请求 
// proxymw implements StringService, forwarding Uppercase requests to the
// provided endpoint, and serving all other (i.e. Count) requests via the
// next StringService.
type proxymw struct {
	next      StringService     // Serve most requests via this service...
	uppercase endpoint.Endpoint // ...except Uppercase, which gets served by this endpoint
}
```

要调用客户端点只需要进行一些简单转换：

```go
func (mw proxymw) Uppercase(s string) (string, error) {
	response, err := mw.uppercase(uppercaseRequest{S: s})
	if err != nil {
		return "", err
	}
	resp := response.(uppercaseResponse)
	if resp.Err != "" {
		return resp.V, errors.New(resp.Err)
	}
	return resp.V, nil
}
```

现在，为了构建这些代理中间件之一，我们将代理 URL 字符串转换为端点。如果我们通过 HTTP 传输 JSON，我们可以使用 transport / http 包中的帮助函数。

```go
import (
	httptransport "github.com/go-kit/kit/transport/http"
)

func proxyingMiddleware(proxyURL string) ServiceMiddleware {
	return func(next StringService) StringService {
		return proxymw{next, makeUppercaseProxy(proxyURL)}
	}
}

func makeUppercaseProxy(proxyURL string) endpoint.Endpoint {
	return httptransport.NewClient(
		"GET",
		mustParseURL(proxyURL),
		encodeUppercaseRequest,
		decodeUppercaseResponse,
	).Endpoint()
}
```

# TODO 将上面的添加到服务中，并测试。

## 四 服务发现和负载均衡

Gokit 为不同的服务发现系统提供适配器，以获取最新的实例集，作为单独的端点公开，这些适配器被称为 `subscribers`。

```go
type Subscriber interface {
    Endpoints()([]endpoint.Endpoint, error)
}
```

在内部，`subscriber` 使用工厂函数将发现的每个实例字符串转换为可用的端点。

```go
type Factory func(instance string) (endpoint.Endpoint, error)
```

到目前为止工厂函数 `makeUppercaseProxy` 只是直接调用 URL，但是将一些安全中间件（断路器限流器）放入工厂函数也是否重要。

```go
var e endpoint.Endpoint
e = makeUppercaseProxy(instance)
e = circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{}))(e)
e = kitratelimit.NewTokenBucketLimiter(jujuratelimit.NewBucketWithRate(float64(maxQPS), int64(maxQPS)))(e)
```

现在我们已经有了一组端点，可以使用负载均衡器选择其中一个端点。Gokit 提供了一系列负载均衡器组件，当然也可以编写自己的负载均衡器。负载均衡器定义：

```go
type Balancer interface {
	Endpoint() (endpoint.Endpoint, error)
}
```

可以使用重试策略包装负载均衡器，返回一个可用端点，使用重试策略将重试失败的请求，直到达到最大尝试次数或超时。

```go
func Retry(max int, timeout time.Duration, lb Balancer) endpoint.Endpoint
```

最终实现：

```go
func proxyingMiddleware(instances string, logger log.Logger) ServiceMiddleware {
	// If instances is empty, don't proxy.
	if instances == "" {
		logger.Log("proxy_to", "none")
		return func(next StringService) StringService { return next }
	}

	// Set some parameters for our client.
	var (
		qps         = 100                    // beyond which we will return an error
		maxAttempts = 3                      // per request, before giving up
		maxTime     = 250 * time.Millisecond // wallclock time, before giving up
	)

	// Otherwise, construct an endpoint for each instance in the list, and add
	// it to a fixed set of endpoints. In a real service, rather than doing this
	// by hand, you'd probably use package sd's support for your service
	// discovery system.
	var (
		instanceList = split(instances)
		subscriber   sd.FixedSubscriber
	)
	logger.Log("proxy_to", fmt.Sprint(instanceList))
	for _, instance := range instanceList {
		var e endpoint.Endpoint
		e = makeUppercaseProxy(instance)
		e = circuitbreaker.Gobreaker(gobreaker.NewCircuitBreaker(gobreaker.Settings{}))(e)
		e = kitratelimit.NewTokenBucketLimiter(jujuratelimit.NewBucketWithRate(float64(qps), int64(qps)))(e)
		subscriber = append(subscriber, e)
	}

	// Now, build a single, retrying, load-balancing endpoint out of all of
	// those individual endpoints.
	balancer := lb.NewRoundRobin(subscriber)
	retry := lb.Retry(maxAttempts, maxTime, balancer)

	// And finally, return the ServiceMiddleware, implemented by proxymw.
	return func(next StringService) StringService {
		return proxymw{next, retry}
	}
}
```

[完整示例](https://github.com/go-kit/kit/blob/master/examples/stringsvc3)

可以使用 Gokit 为服务创建客户端软件包，以便从其他 Go 程序消费服务。实际上，您的客户端软件包将提供服务接口的实现，该接口使用特定传输调用远程服务实例。

-   [示例代码1](https://github.com/go-kit/kit/blob/master/examples/addsvc/cmd/addcli/addcli.go)
-   [示例代码2](https://github.com/go-kit/kit/blob/master/examples/profilesvc/client/client.go)

