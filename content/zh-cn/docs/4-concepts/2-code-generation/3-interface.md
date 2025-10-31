---
title: "生成的服务接口与端点"
linkTitle: "服务接口与端点"
weight: 3
description: "了解 Goa 生成的代码，包括服务接口、端点以及传输层。"
---

## 服务接口

Goa 首先生成的是服务接口层。该基础层同时定义 API 合同与服务实现接口，包含实现 API 端点的所有方法签名，并完整给出载荷（payload）与结果类型定义，以明确服务操作所使用的数据结构。

例如，假设有如下设计：

```go
var _ = Service("calc", func() {
    Description("calc 服务提供加法与乘法运算。")
    Method("add", func() {
        Description("Add 返回 a 与 b 的和")
        Payload(func() {
            Attribute("a", Int)
            Attribute("b", Int)
        })
        Result(Int)
    })
    Method("multiply", func() {
        Description("Multiply 返回 a 与 b 的积")
        Payload(func() {
            Attribute("a", Int)
            Attribute("b", Int)
        })
        Result(Int)
    })
})
```

基于上述设计，Goa 会在 `gen/calc/service.go` 中生成如下服务接口：

```go
// calc 服务提供加法与乘法运算。
type Service interface {
    // Add 返回 a 与 b 的和
    Add(context.Context, *AddPayload) (res int, err error)
    // Multiply 返回 a 与 b 的积
    Multiply(context.Context, *MultiplyPayload) (res int, err error)
}

// AddPayload 是 calc 服务 add 方法的载荷类型。
type AddPayload struct {
    A int32
    B int32
}

// MultiplyPayload 是 calc 服务 multiply 方法的载荷类型。
type MultiplyPayload struct {
    A int32
    B int32
}
```

Goa 还会生成便于配置服务与可观测性栈的常量，例如服务名与方法名：

```go
// APIName 是设计中定义的 API 名称。
const APIName = "calc"

// APIVersion 是设计中定义的 API 版本。
const APIVersion = "0.0.1"

// ServiceName 是设计中定义的服务名称。
// 同样的值也会在端点请求上下文中以 ServiceKey 键进行设置。
const ServiceName = "calc"

// MethodNames 列出了设计中定义的服务方法名。
// 同样的值也会在端点请求上下文中以 MethodKey 键进行设置。
var MethodNames = [1]string{"multiply"}
```

## 端点层

接着，Goa 会在 `gen/calc/endpoints.go` 中生成端点层，此层以与传输无关的方式暴露服务方法，使得可以对服务方法应用中间件及其他横切关注点：

```go
// Endpoints 封装了 "calc" 服务的端点。
type Endpoints struct {
    Add      goa.Endpoint
    Multiply goa.Endpoint
}

// NewEndpoints 使用端点封装 "calc" 服务的方法。
func NewEndpoints(s Service) *Endpoints {
   return &Endpoints{
        Add:      NewAddEndpoint(s),
        Multiply: NewMultiplyEndpoint(s),
    }
}
```

`Endpoints` 结构体可通过服务实现进行初始化，并用于创建特定传输的服务端与客户端实现。

Goa 还会生成将服务方法包装为端点的具体实现：

```go
// NewAddEndpoint 返回调用 "calc" 服务 "add" 方法的端点。
func NewAddEndpoint(s Service) goa.Endpoint {
    return func(ctx context.Context, req any) (any, error) {
        p := req.(*AddPayload)
        return s.Add(ctx, p)
    }
}
```

该模式允许你对特定端点应用中间件，或完全以自定义实现替换端点。

## Goa 端点中间件

最后，Goa 会生成一个 `Use` 函数，用于将中间件应用到所有服务方法：

```go
// Use 将给定的中间件应用到 "calc" 服务的所有端点。
func (e *Endpoints) Use(m func(goa.Endpoint) goa.Endpoint) {
    e.Add = m(e.Add)
	e.Multiply = m(e.Multiply)
}
```

端点中间件是一个以端点为参数并返回新端点的函数。其实现可以修改请求载荷与结果、变更上下文、检查错误，并在端点调用前后执行任何所需操作。Goa 的端点在 `goa` 包中定义如下：

```go
// Endpoint 以与底层传输无关的方式向远程客户端暴露服务方法。
type Endpoint func(ctx context.Context, req any) (res any, err error)
```

例如，下面的 Goa 端点中间件会记录请求与响应：

```go
func LoggingMiddleware(next goa.Endpoint) goa.Endpoint {
    return func(ctx context.Context, req any) (res any, err error) {
        log.Printf("request: %v", req)
        res, err = next(ctx, req)
        log.Printf("response: %v", res)
        return
    }
}
```

你可以这样将该中间件应用到服务端点：

```go
endpoints.Use(LoggingMiddleware)
```
