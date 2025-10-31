---
title: "生成的客户端"
linkTitle: "客户端"
weight: 4
description: "了解 Goa 生成的客户端代码，包括客户端端点与客户端结构体。"
---

## Goa 端点

端点抽象表示服务中的单个 RPC 方法。端点既可以在服务端创建以表示服务实现的方法，也可以在客户端创建以表示客户端调用的方法。两种情况下，端点都由 `goa.Endpoint` 函数表示。

## 端点客户端

继续使用 `calc` 示例：Goa 会在 `gen/calc/client.go` 中生成与传输无关的客户端结构体。该结构体包含客户端侧的端点，并提供强类型方法来发起服务请求。生成的客户端代码如下：

```go
// Client 是 "calc" 服务的客户端。
type Client struct {
        MultiplyEndpoint goa.Endpoint
        AddEndpoint      goa.Endpoint
}

// NewClient 根据给定端点初始化一个 "calc" 服务客户端。
func NewClient(multiply, add goa.Endpoint) *Client {
        return &Client{
                AddEndpoint:      add,
                MultiplyEndpoint: multiply,
        }
}

// Add 调用 "calc" 服务的 "add" 端点。
func (c *Client) Add(ctx context.Context, p *AddPayload) (res int, err error) {
        var ires any
        ires, err = c.AddEndpoint(ctx, p)
        if err != nil {
                return
        }
        return ires.(int), nil
}

// Multiply 调用 "calc" 服务的 "multiply" 端点。
func (c *Client) Multiply(ctx context.Context, p *MultiplyPayload) (res int, err error) {
        var ires any
        ires, err = c.MultiplyEndpoint(ctx, p)
        if err != nil {
                return
        }
        return ires.(int), nil
}
```

该客户端结构体包含两个字段：`AddEndpoint` 与 `MultiplyEndpoint`，分别表示 `add` 与 `multiply` 方法的客户端端点。`NewClient` 函数使用提供的端点初始化客户端结构体。

Goa 生成的与传输相关的代码会包含用于创建传输层端点的工厂方法，这些工厂方法用于以合适的端点初始化客户端结构体。

更多关于传输相关的客户端实现，请参见 [HTTP](./5-http.md) 与 [gRPC](./6-grpc.md) 章节。


