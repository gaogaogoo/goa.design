---
title: "优雅单体"
linkTitle: "优雅单体"
description: "使用 Goa 构建单体服务的指南"
weight: 4
---

Goa 简化了优雅单体架构的采用，将所有服务合并为单一代码库并在单一进程中运行。本指南将逐步说明在 Goa 中设置优雅单体的过程。

## Goa 端点

在 Goa 中构建单体应用的核心是 `Endpoint` 构造：

```go
type Endpoint func(ctx context.Context, request any) (response any, err error)
```

端点定义服务端与客户端的可远程调用函数。服务端包装业务逻辑，客户端包装传输层。

例如给出以下 Goa 设计：

```go
var _ = Service("calc", func() {
    Method("multiply", func() {
        Payload(func() {
            Attribute("a", Int)
            Attribute("b", Int)
        })
        Result(Int)
        HTTP(func() {
            GET("/multiply/{a}/{b}")
        })
    })
})
```

Goa 在服务端生成列出所有服务端点的 Go 结构体，并提供一个给定业务逻辑实例化该结构体的辅助函数：

```go
// Endpoints 包装 "calc" 服务的端点。
type Endpoints struct {
	Multiply goa.Endpoint
}

// NewEndpoints 将 "calc" 服务的方法包装为端点。
func NewEndpoints(s Service) *Endpoints {
    return &Endpoints{
        Multiply: NewMultiplyEndpoint(s),
    }
}
```

Goa 在客户端生成暴露服务方法的 Go 结构体，可给定端点构造：

```go
// Client 是 "calc" 服务客户端。
type Client struct {
	MultiplyEndpoint goa.Endpoint
}

// NewClient 使用给定的端点初始化 "calc" 服务客户端。
func NewClient(multiply goa.Endpoint) *Client {
	return &Client{
		MultiplyEndpoint: multiply,
	}
}

// Multiply 调用 "calc" 服务的 "multiply" 端点。
func (c *Client) Multiply(ctx context.Context, p *MultiplyPayload) (res int, err error) {
    // 为简洁起见省略
}
```

## 构建单体应用

`Endpoint` 的优势在于可独立于传输层使用。特别是可基于内存实现的端点构造客户端，按前面的例子：

```go
package main

import (
    "context"
    "github.com/<your username>/calc/gen/calc"
)

func main() {
    // 1. 实例化服务
    // NewService() 是您的函数，返回实现生成服务接口的结构体
    service := NewService() 

    // 2. 使用 Goa 生成的 NewEndpoints 实例化服务端端点
    endpoints := calc.NewEndpoints(service)

    // 3. 使用 Goa 生成的 NewClient 实例化客户端
    client := calc.NewClient(endpoints.Multiply)

    // 4. 使用客户端
    res, err := client.Multiply(context.Background(), &calc.MultiplyPayload{A: 1, B: 2})

    // ...
}

// NewService 返回新的服务实例。
func NewService() calc.Service {
    // ...
}
```

该模式在部署与运维上享受单体应用的简单性，同时保持模块化与可扩展架构。

## 将单体架构应用到整个系统

该模式可应用到由多个服务组成的系统，遵循
[多个服务](./2-multiple-services.md)指南中的布局。主要差异在于 `main` 实现实例化所有服务，而非仅一个。

假设系统同时包含 `users` 和 `products` 服务，且 `products` 依赖 `users`：

```go
// main.go
package main

import (
    "context"
    "flag"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "strings"
    "sync"
    "syscall"
    "time"

    "goa.design/clue/log"
    goahttp "goa.design/goa/v3/http"
    
    genusersserver "myapi/services/users/gen/http/users/server"
    genusers "myapi/services/users/gen/users"
    "myapi/services/users"
    
    genproductsserver "myapi/services/products/gen/http/products/server"
    genproducts "myapi/services/products/gen/products"
    "myapi/services/products"
)

func main() {
    // 解析命令行标志
    var (
        httpAddr = flag.String("http-addr", ":8080", "HTTP listen address")
        debug    = flag.Bool("debug", false, "Enable debug mode")
    )
    flag.Parse()

    // 使用日志记录器初始化上下文
    format := log.FormatJSON
    if log.IsTerminal() {
        format = log.FormatTerminal
    }
    ctx := log.Context(context.Background(), log.WithFormat(format))
    if *debug {
        ctx = log.Context(ctx, log.WithDebug())
        log.Debugf(ctx, "debug mode enabled")
    }

    /*------------------------------------------*
     * 本节特定于单体应用 *
     *------------------------------------------*/

    // 创建服务和端点
    usersSvc := users.NewUsers()
    usersEndpoints := genusers.New(usersSvc)
    // 为 users 服务创建内存客户端
    usersClient := genusers.NewClient(usersEndpoints.Create, usersEndpoints.Find, usersEndpoints.Update, usersEndpoints.Delete)
    productsSvc := products.NewProducts(usersClient)
    productsEndpoints := genproducts.New(productsSvc)

    // 创建传输处理器
    mux := goahttp.NewMuxer()
    usersServer := genusersserver.New(usersEndpoints, mux, goahttp.RequestDecoder, goahttp.ResponseEncoder, nil, nil)
    usersServer.Mount(mux)
    productsServer := genproductsserver.New(productsEndpoints, mux, goahttp.RequestDecoder, goahttp.ResponseEncoder, nil, nil)
    productsServer.Mount(mux)

    // 记录已挂载的端点
    for _, m := range usersServer.Mounts {
        log.Printf(ctx, "mounted %s %s", m.Method, m.Pattern)
    }
    for _, m := range productsServer.Mounts {
        log.Printf(ctx, "mounted %s %s", m.Method, m.Pattern)
    }

    /*------------------------------------------*
     * 其余部分与任何 Goa 应用相同 *
     *------------------------------------------*/

    // 创建 HTTP 服务器
    mux.Use(log.HTTP(ctx)) // 将日志记录器添加到请求上下文
    httpServer := &http.Server{
        Addr:    *httpAddr,
        Handler: mux,
    }

    // 优雅处理关闭
    errc := make(chan error)
    go func() {
        c := make(chan os.Signal, 1)
        signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
        errc <- fmt.Errorf("signal: %s", <-c)
    }()

    ctx, cancel := context.WithCancel(ctx)
    var wg sync.WaitGroup
    wg.Add(1)

    go func() {
        defer wg.Done()

        // 启动 HTTP 服务器
        go func() {
            log.Printf(ctx, "HTTP server listening on %s", *httpAddr)
            errc <- httpServer.ListenAndServe()
        }()

        <-ctx.Done()
        log.Print(ctx, "shutting down HTTP server")

        // 优雅关闭，30s 超时
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()

        if err := httpServer.Shutdown(ctx); err != nil {
            log.Errorf(ctx, err, "failed to shutdown HTTP server")
        }
    }()

    // 等待关闭
    if err := <-errc; err != nil && !strings.HasPrefix(err.Error(), "signal:") {
        log.Errorf(ctx, err, "server error")
    }
    cancel()
    wg.Wait()
    log.Print(ctx, "server exited")
}
```

本示例创建单个 HTTP 服务器，为单体中所有服务提供服务。服务间通信由内存客户端处理。






