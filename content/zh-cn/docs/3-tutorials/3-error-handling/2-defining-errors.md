---
title: 定义错误
linkTitle: 定义错误
weight: 2
description: "掌握在 Goa 中使用 DSL 定义服务级与方法级错误的技巧，包括自定义错误类型与可复用的错误定义。"
---

Goa 为在服务设计中定义错误提供了灵活而强大的方式。借助 Goa 的领域特定语言（DSL），你可以同时指定服务级与方法级错误，自定义错误类型，并确保 API 在不同传输（如 HTTP 和 gRPC）中对失败进行清晰一致的沟通。

## 服务级错误

服务级错误在服务作用域内定义，可由服务中的任意方法返回。它适用于跨多个方法的通用错误场景。

### 示例

```go
var _ = Service("divider", func() {
    // “DivByZero” 错误定义在服务级，
    // 因此可被 "divide" 与 "integral_divide" 两个方法返回。
    Error("DivByZero", func() {
        Description("DivByZero is the error returned by the service methods when the right operand is 0.")
    })

    Method("integral_divide", func() {
        // 方法特定的定义…
    })

    Method("divide", func() {
        // 方法特定的定义…
    })
})
```

在此示例中，我们定义了一个名为 `DivByZero` 的服务级错误，它可被 `divider` 服务中的任何方法使用。这对可能在多个方法中发生的通用错误（如本例中的除数为零）尤其有用。

## 方法级错误

方法级错误在具体方法的作用域内定义，仅适用于该方法。它允许针对特定操作进行更细粒度的错误处理。

### 示例

```go
var _ = Service("divider", func() {
    Method("integral_divide", func() {
        // “HasRemainder” 错误定义在方法级，
        // 因此仅适用于 "integral_divide"。
        Error("HasRemainder", func() {
            Description("HasRemainder is the error returned when an integer division has a remainder.")
        })
        // 其他方法定义…
    })

    Method("divide", func() {
        // 方法特定的定义…
    })
})
```

在此示例中，我们定义了一个名为 `HasRemainder` 的方法级错误，它特定于 `integral_divide` 方法。当除法运算产生余数时（对整数除法尤其相关），该错误将被使用。

## 可复用的错误定义

Goa 允许你在多个服务与方法间复用错误定义。这对于在 API 多个部分使用的通用错误尤其有用。此类定义必须出现在 `API` DSL 中：

### 示例

```go
var _ = API("example", func() {
    Error("NotFound", func() {
        Description("Resource was not found in the system.")
    })
    HTTP(func() {
        Response("NotFound", StatusNotFound)
    })
    GRPC(func() {
        Response("NotFound", CodeNotFound)
    })
})

var _ = Service("example", func() {
    Method("get", func() {
        Payload(func() {
            Field(1, "id", String, "The ID of the concert to get.")
        })
        Result(Concert)
        Error("NotFound")
        HTTP(func() {
            GET("/concerts/{id}")
        })
        GRPC(func() {})
    })
})
```

在此示例中，我们定义了一个名为 `NotFound` 的可复用错误，可被 `example` 服务内的任意方法使用。该错误定义在 `API` DSL 中，因此对 API 内所有服务与方法均可用。`NotFound` 错误被映射为 HTTP 状态码 `404` 与 gRPC 状态码 `NotFound`，该映射在 `API` DSL 中完成，无需在 `Service` 或 `Method` DSL 中重复。

## 自定义错误类型与描述

Goa 的 Error DSL 提供了多种定制错误定义与文档化的方式。你可以指定描述、临时/超时/故障标识，甚至定义自定义响应结构。

### 基本错误定义

最简单的错误定义形式包含名称与描述：

```go
Error("NotFound", func() {
    Description("Resource was not found in the system.")
})
```

上述定义等价于：

```go
Error("NotFound", ErrorResult, "Resource was not found in the system.")
```

错误的默认类型为 `ErrorResult`，其会在生成代码中映射到
[ServiceError](https://pkg.go.dev/goa.design/goa/v3/pkg#ServiceError) 类型。

### 临时、超时与故障

你可以通过 `Temporary`、`Timeout` 与 `Fault` 函数标识错误是否为临时、超时或故障（或其任意组合）：

```go
Error("ServiceUnavailable", func() {
    Description("Service is temporarily unavailable.")
    Temporary()
})

Error("RequestTimeout", func() {
    Description("Request timed out.")
    Timeout()
})

Error("InternalServerError", func() {
    Description("Internal server error.")
    Fault()
})
```

客户端随后可从
[ServiceError](https://pkg.go.dev/goa.design/goa/v3/pkg#ServiceError) 对象中查找相应字段，
以判断错误是否为临时、超时或故障。

> 注意：此能力仅支持运行时类型为
> [ServiceError](https://pkg.go.dev/goa.design/goa/v3/pkg#ServiceError)
> 的 `ErrorResult` 错误。

### 自定义错误类型

Goa 也使得设计自定义错误类型变得容易，例如：

```go
Error("ValidationError", DivByZero, "DivByZero is the error returned when using value 0 as divisor.")
```

此示例假设 `DivByZero` 是在文件其它位置定义的自定义错误类型，例如：

```go
var DivByZero = Type("DivByZero", func() {
    Field(1, "name", String, "The name of the error.", func() {
        Meta("struct:error:name")
    })
    Field(2, "message", String, "The error message.")
    Required("name", "message")
})
```

这些错误定义既可用于服务级也可用于方法级，为你构建 API 的错误处理提供了灵活性。Error DSL 与 Goa 的代码生成集成，可在不同传输协议之间生成一致的错误响应。

详见[错误类型](../3-error-types)以获取关于自定义错误类型的更多信息。

## 总结

在 Goa 中定义错误是一个与服务设计无缝集成的直观过程。通过使用服务级与方法级错误定义、充分利用默认的 ErrorResult 类型或创建自定义错误类型，你可以确保 API 优雅地处理失败并有效地与客户端沟通。正确的错误定义不仅增强服务的健壮性，也通过提供清晰一致的错误处理机制提升开发者体验。
