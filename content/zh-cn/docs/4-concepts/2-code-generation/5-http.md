---
title: "生成的 HTTP 服务端与客户端代码"
linkTitle: "HTTP 代码"
weight: 5
description: "了解 Goa 生成的 HTTP 代码，包括服务端与客户端实现、路由与错误处理。"
---

HTTP 代码生成会产出完整的客户端与服务端实现，处理所有传输层相关的事务，包括请求路由、数据序列化、错误处理等。本文介绍生成的 HTTP 代码的关键组件。

## HTTP 服务端实现

Goa 会在 `gen/http/<服务名>/server/server.go` 中生成完整的 HTTP 服务端实现，根据设计中描述的 HTTP 路径与动词暴露服务方法。该服务端实现会处理所有网络细节，包括请求路由、响应编码与错误处理：

```go
// New 使用提供的编码器与解码器为所有 calc 服务端点实例化 HTTP 处理器。
// 这些处理器会根据设计中定义的 HTTP 动词与路径挂载到给定的 mux 上。
// 当响应编码失败时会调用 errhandler；formatter 用于在编码前格式化
// 服务方法返回的错误。二者均为可选参数，可以为 nil。
func New(
    e *calc.Endpoints,
    mux goahttp.Muxer,
    decoder func(*http.Request) goahttp.Decoder,
    encoder func(context.Context, http.ResponseWriter) goahttp.Encoder,
    errhandler func(context.Context, http.ResponseWriter, error),
    formatter func(ctx context.Context, err error) goahttp.Statuser,
) *Server
```

实例化 HTTP 服务端需要提供服务端点、用于将请求路由到正确处理器的 muxer，以及用于编解码数据的编码器与解码器函数。服务端还支持自定义错误处理与格式化（两者皆为可选）。更多关于 HTTP 编码、错误处理与格式化的信息请参见 [HTTP 服务](../3-http-services)。

Goa 还会生成一个辅助函数用于将 HTTP 服务端挂载到 muxer：

```go
// Mount 配置 mux 以服务 calc 端点。
func Mount(mux goahttp.Muxer, h *Server) {
    MountAddHandler(mux, h.Add)
    MountMultiplyHandler(mux, h.Multiply)
}
```

该函数会根据设计中定义的 HTTP 动词与路径自动配置 muxer，将请求路由到正确的处理器。

`Server` 结构体还暴露了可用于修改单个处理器或将中间件应用到特定端点的字段：

```go
// Server 列出了 calc 服务端点的 HTTP 处理器。
type Server struct {
    Mounts   []*MountPoint // 方法名及其关联的 HTTP 动词与路径
    Add      http.Handler  // "add" 服务方法的 HTTP 处理器
    Multiply http.Handler  // "multiply" 服务方法的 HTTP 处理器
}
```

`Mounts` 字段包含所有已挂载处理器的元数据，便于自省；`Add` 与 `Multiply` 字段分别暴露每个服务方法的单独处理器。

`MountPoint` 结构体包含每个处理器的“方法名、HTTP 动词与路径”。这些元数据对于自省与调试非常有用。

最后，`Server` 结构体还提供了 `Use` 方法，可将 HTTP 中间件应用到所有服务方法：

```go
// Use 将给定中间件应用到 "calc" 服务的所有 HTTP 处理器。
func (s *Server) Use(m func(http.Handler) http.Handler) {
    s.Add = m(s.Add)
    s.Multiply = m(s.Multiply)
}
```

例如，可以将一个日志中间件应用到所有处理器：

```go
// LoggingMiddleware 是一个示例中间件，用于记录每个请求。
func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("Received request: %s %s", r.Method, r.URL.Path)
        next.ServeHTTP(w, r)
    })
}

// 将中间件应用到服务端
server.Use(LoggingMiddleware)
```

#### HTTP 路径

Goa 会为每个服务方法生成用于返回其 HTTP 路径的函数（根据所需参数）。例如，`Add` 方法会生成如下函数：

```go
// AddCalcPath 返回指向 calc 服务 add HTTP 端点的 URL 路径。
func AddCalcPath(a int, b int) string {
	return fmt.Sprintf("/add/%v/%v", a, b)
}
```

此类函数可用于为服务方法生成 URL。

### 综合示例

服务接口层与端点层是 Goa 生成服务的核心构建块。你的 `main` 包会使用这些层来创建特定传输的服务端与客户端实现，从而通过 HTTP 运行并与服务交互：

```go
package main

import (
    "net/http"

    goahttp "goa.design/goa/v3/http"

    "github.com/<your username>/calc"
    gencalc "github.com/<your username>/calc/gen/calc"
    genhttp "github.com/<your username>/calc/gen/http/calc/server"
)

func main() {
    svc := calc.New()                      // 你的服务实现
    endpoints := gencalc.NewEndpoints(svc) // 创建服务端点
    mux := goahttp.NewMuxer()              // 创建 HTTP 请求复用器
    server := genhttp.New(                 // 创建 HTTP 服务端
        endpoints,
        mux,
        goahttp.RequestDecoder,
        goahttp.ResponseEncoder,
        nil, nil)              
    genhttp.Mount(mux, server)             // 挂载服务端
    http.ListenAndServe(":8080", mux)      // 启动 HTTP 服务
}
```

## HTTP 客户端实现

Goa 也会生成完整的 HTTP 客户端实现用于与服务交互。客户端代码位于 `client` 包：`gen/http/<服务名>/client/client.go`，提供为各服务方法创建 Goa 端点的方法。这些端点随后可被包装为与传输无关的客户端实现，并与 `gen/<服务名>/client.go` 中生成的端点客户端代码配合使用：

HTTP `client` 包中生成的 `NewClient` 函数用于创建一个可生成与传输无关客户端端点的对象：

```go  
// NewClient 为所有 calc 服务的服务端实例化 HTTP 客户端。
func NewClient(
	scheme string,
	host string,
	doer goahttp.Doer,
	enc func(*http.Request) goahttp.Encoder,
	dec func(*http.Response) goahttp.Decoder,
	restoreBody bool,
) *Client
```

该函数需要服务的 scheme（`http` 或 `https`）、host（例如 `example.com`）以及一个可发起请求的 HTTP 客户端。标准库的 Go HTTP 客户端满足 `goahttp.Doer` 接口。`NewClient` 还需要编码器与解码器函数用于数据编解码；`restoreBody` 标志指示在解码后是否恢复底层 Go `io.Reader` 对象中的响应体。

### HTTP 客户端

实例化后的 `Client` 结构体会为每个服务端点暴露字段，使得可以为特定端点覆盖 HTTP 客户端：

```go
// Client 列出了 calc 服务端点的 HTTP 客户端。
type Client struct {
	// AddDoer 是用于向 add 端点发起请求的 HTTP 客户端。
	AddDoer goahttp.Doer

	// MultiplyDoer 是用于向 multiply 端点发起请求的 HTTP 客户端。
	MultiplyDoer goahttp.Doer

	// RestoreResponseBody 控制是否在解码后重置响应体，以便再次读取。
	RestoreResponseBody bool

    // 私有字段...
}
```

该结构体同时暴露了用于构建与传输无关端点的方法：

```go
// Multiply 返回一个对 calc 服务 multiply 服务端进行 HTTP 请求的端点。
func (c *Client) Multiply() goa.Endpoint
```

### 综合示例

如 [./4-client.md](Client) 一节所述，Goa 会为每个服务生成与传输无关的客户端。这些客户端通过合适的端点进行初始化，并可用于向服务发起请求。

下面是为 `calc` 服务创建并使用 HTTP 客户端的示例：

```go
package main

import (
    "context"
    "log"
    "net/http"

    goahttp "goa.design/goa/v3/http"
    
    gencalc "github.com/<your username>/calc/gen/calc"
    genclient "github.com/<your username>/calc/gen/http/calc/client"
)

func main() {
    // 创建 HTTP 客户端
    httpClient := genclient.NewClient(
        "http",                    // Scheme
        "localhost:8080",          // Host
        http.DefaultClient,        // HTTP 客户端
        goahttp.RequestEncoder,    // 请求编码器
        goahttp.ResponseDecoder,   // 响应解码器
        false,                     // 不恢复响应体
    )

    // 创建端点客户端
    client := gencalc.NewClient(
        httpClient.Add(),          // Add 端点
        httpClient.Multiply(),     // Multiply 端点
    )

    // 调用服务方法
    result, err := client.Add(context.Background(), &gencalc.AddPayload{A: 1, B: 2})
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("1 + 2 = %d", result)
}
```

该示例展示了：如何创建 HTTP 客户端、将其包装为端点客户端，并向服务发起请求。HTTP 客户端会处理所有传输层相关事务，而端点客户端为服务调用提供了整洁的接口。

### 结语

生成的 HTTP 客户端为通过 HTTP 与服务交互提供了便捷途径。通过创建客户端实例并调用服务方法，你可以轻松将服务集成到应用中。