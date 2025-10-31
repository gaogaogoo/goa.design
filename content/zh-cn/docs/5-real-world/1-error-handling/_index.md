---
linkTitle: 错误处理
title: Goa 中的错误处理
weight: 1
description: "了解如何在 Goa 服务中有效处理错误，包括错误定义、传输映射和最佳实践。"
---

Goa 提供了一个强大的错误处理系统，使您能够有效地定义、管理和在您的服务之间传达错误。本指南涵盖了您需要了解的有关 Goa 错误处理的所有信息。

## 概述

Goa 对错误处理采取“开箱即用”的方法，只需最少的信息（仅一个名称）即可定义错误，同时在需要时也支持完全自定义的错误类型。该框架会根据您的错误定义生成代码和文档，确保整个 API 的一致性。

Goa 错误处理的主要特点：

- 服务级别和方法级别的错误定义
- 默认和自定义错误类型
- 特定于传输的状态码映射 (HTTP/gRPC)
- 用于创建错误的生成辅助函数
- 自动生成文档

## 定义错误

### API 级别的错误

可以在 API 级别定义错误，以创建可重用的错误定义。
与服务级别的错误不同，API 级别的错误不会自动应用于所有方法。
相反，它们提供了一种方法来定义错误属性，包括传输映射，一次定义，跨服务和方法重用：

```go
var _ = API("calc", func() {
    // 定义可重用的错误及传输映射
    Error("invalid_argument")  // 使用默认的 ErrorResult 类型
    HTTP(func() {
        Response("invalid_argument", StatusBadRequest)
    })
})

var _ = Service("divider", func() {
    // 引用 API 级别的错误
    Error("invalid_argument")  // 重用上面定义的错误
                              // 无需再次定义 HTTP 映射

    Method("divide", func() {
        Payload(DivideRequest)
        // 具有自定义类型的方法特定错误
        Error("div_by_zero", DivByZero, "除以零")
    })
})
```

这种方法：
- 在您的 API 中实现一致的错误定义
- 减少传输映射的重复
- 允许集中的错误处理策略
- 使维护一致的错误响应更容易

### 服务级别的错误

服务级别的错误可用于服务中的所有方法。与提供可重用定义的 API 级别错误不同，服务级别错误会自动应用于服务中的每个方法：

```go
var _ = Service("calc", func() {
    // 此服务中的任何方法都可以返回此错误
    Error("invalid_arguments", ErrorResult, "提供的参数无效") 
    
    Method("divide", func() {
        // 此方法可以返回 invalid_arguments 而无需显式声明
        Payload(func() {
            Field(1, "dividend", Int)
            Field(2, "divisor", Int)
            Required("dividend", "divisor")
        })
        // ... 其他方法定义
    })

    Method("multiply", func() {
        // 此方法也可以返回 invalid_arguments
        // ... 方法定义
    })
})
```

在服务级别定义错误时：
- 该错误可用于服务中的所有方法
- 每个方法都可以返回该错误而无需显式声明
- 为该错误定义的传输映射适用于所有方法

### 方法级别的错误

方法特定的错误作用域限定于特定方法：

```go
var _ = Service("calc", func() {
    Method("divide", func() {
        Payload(func() {
            Field(1, "dividend", Int)
            Field(2, "divisor", Int)
            Required("dividend", "divisor")
        })
        Result(func() {
            Field(1, "quotient", Int)
            Field(2, "reminder", Int)
            Required("quotient", "reminder")
        })
        Error("div_by_zero") // 方法特定的错误
    })
})
```

### 自定义错误类型

对于更复杂的错误场景，您可以定义自定义错误类型。自定义错误类型允许您包含特定于您的错误情况的附加上下文信息。

#### 基本自定义错误类型

这是一个简单的自定义错误类型：

```go
var DivByZero = Type("DivByZero", func() {
    Description("DivByZero 是使用值 0 作为除数时返回的错误。")
    Field(1, "message", String, "除以零会导致无穷大。")
    Required("message")
})
```

#### 错误名称字段要求

在同一方法中为多个错误使用自定义错误类型时，Goa 需要知道哪个字段包含错误名称。这对于以下方面至关重要：
- 将错误与其设计定义相匹配
- 确定正确的 HTTP/gRPC 状态码
- 生成正确的文档
- 在客户端中实现正确的错误处理

要指定错误名称字段，请使用 `struct:error:name` 元数据：

```go
var DivByZero = Type("DivByZero", func() {
    Description("DivByZero 是使用值 0 作为除数时返回的错误。")
    Field(1, "message", String, "除以零会导致无穷大。")
    Field(2, "name", String, "错误名称", func() {
        Meta("struct:error:name")  // 告诉 Goa 此字段包含错误名称
    })
    Required("message", "name")
})
```

标有 `Meta("struct:error:name")` 的字段：
- 必须是字符串类型
- 必须是必填字段
- 必须设置为设计中定义的错误名称
- 不能命名为 `"error_name"`（由 Goa 保留）

#### 使用多种错误类型

当一个方法可以返回多种不同的自定义错误类型时，名称字段变得尤为重要。原因如下：

1. **错误类型解析**：当可能出现多种错误类型时，Goa 使用名称字段来确定设计中的哪个错误定义与实际返回的错误相匹配。这使得 Goa 能够：
   - 应用正确的传输映射（HTTP/gRPC 状态码）
   - 生成准确的 API 文档
   - 实现正确的客户端错误处理

2. **传输层处理**：如果没有名称字段，当定义了具有不同状态码的多种错误类型时，传输层将不知道使用哪个状态码：
   ```go
   HTTP(func() {
       Response("div_by_zero", StatusBadRequest)        // 400
       Response("overflow", StatusUnprocessableEntity)  // 422
   })
   ```

3. **客户端类型断言**：名称字段使 Goa 能够为您的设计中定义的每个错误生成特定的错误类型。这些生成的类型使错误处理类型安全，并提供对所有错误字段的访问：

以下示例显示了设计中的错误名称必须与实现相匹配：

```go
var _ = Service("calc", func() {
    Method("divide", func() {
        // 这些名称（"div_by_zero" 和 "overflow"）必须在错误类型的名称字段中完全一致地使用
        Error("div_by_zero", DivByZero)
        Error("overflow", NumericOverflow)
        // ... 其他方法定义
    })
})

// 处理这些错误的示例客户端代码
res, err := client.Divide(ctx, payload)
if err != nil {
    switch err := err.(type) {
    case *calc.DivideDivByZeroError:
        // 此错误对应于设计中的 Error("div_by_zero", ...)
        fmt.Printf("除以零错误: %s\n", err.Message)
        fmt.Printf("试图将 %d 除以零\n", err.Dividend)
    case *calc.DivideOverflowError:
        // 此错误对应于设计中的 Error("overflow", ...)
        fmt.Printf("溢出错误: %s\n", err.Message)
        fmt.Printf("结果值 %d 超出最大值\n", err.Value)
    case *goa.ServiceError:
        // 处理常规服务错误（验证等）
        fmt.Printf("服务错误: %s\n", err.Message)
    default:
        // 处理未知错误
        fmt.Printf("未知错误: %s\n", err.Error())
    }
}
```

对于您设计中定义的每个错误，Goa 会生成：
- 一个特定的错误类型（例如，`"div_by_zero"` 的 `DivideDivByZeroError`）
- 用于创建和处理这些错误的辅助函数
- 在传输层进行正确的错误类型转换

设计和实现之间的联系通过错误名称来维持：
1. 设计中 `Error("name", ...)` 使用的名称
2. 您的错误类型中的名称字段必须完全匹配
3. 生成的错误类型将以此命名（例如，`MethodNameError`）

### 错误属性

错误属性是通知客户端错误性质并使其能够实施适当处理策略的关键标志。这些属性**仅在使用默认 `ErrorResult` 类型时可用** - 在使用自定义错误类型时它们不起作用。

这些属性使用 DSL 函数定义：

- `Temporary()`: 表示错误是暂时的，如果重试，相同的请求可能会成功
- `Timeout()`: 表示错误是由于超出截止日期而发生的
- `Fault()`: 表示服务器端错误（错误、配置问题等）

使用默认 `ErrorResult` 类型时，这些属性会自动映射到生成的 `ServiceError` 结构中的字段，从而实现复杂的客户端错误处理：

```go
var _ = Service("calc", func() {
    // 临时错误建议客户端重试
    Error("service_unavailable", ErrorResult, func() {  // 注意：使用 ErrorResult 类型
        Description("服务暂时不可用")
        Temporary()  // 在 ServiceError 中设置 Temporary 字段
    })

    // 超时错误帮助客户端调整其超时
    Error("request_timeout", ErrorResult, func() {      // 注意：使用 ErrorResult 类型
        Description("请求超时")
        Timeout()    // 在 ServiceError 中设置 Timeout 字段
    })

    // 故障错误表示需要管理员注意的服务器问题
    Error("internal_error", ErrorResult, func() {       // 注意：使用 ErrorResult 类型
        Description("内部服务器错误")
        Fault()      // 在 ServiceError 中设置 Fault 字段
    })
})
```

然后，客户端可以使用这些属性来实现复杂的错误处理：

```go
res, err := client.Divide(ctx, payload)
if err != nil {
    switch e := err.(type) {
    case *goa.ServiceError:  // 只有 ServiceError 具有这些属性
        if e.Temporary {
            // 实现带退避的重试
            return retry(ctx, func() error {
                res, err = client.Divide(ctx, payload)
                return err
            })
        }
        if e.Timeout {
            // 可能为下一个请求增加超时
            ctx = context.WithTimeout(ctx, 2*time.Second)
            return client.Divide(ctx, payload)
        }
        if e.Fault {
            // 记录错误并提醒管理员
            log.Error("检测到服务器故障", "error", e)
            alertAdmins(e)
        }
    default:
        // 自定义错误类型将不具有这些属性
        log.Error("发生错误", "error", err)
    }
}
```

这些属性使客户端能够：
- 为临时错误实施智能重试策略
- 为超时错误调整超时或有效负载大小
- 正确升级服务器端故障
- 就​​是否重试操作做出明智的决定

注意：如果您需要将这些属性与自定义错误类型一起使用，则需要在您的自定义类型中实现类似的字段，并在您的代码中显式处理它们。

## 传输映射

Goa 允许您将错误映射到适当的特定于传输的状态码。此映射对于在不同协议之间提供一致且有意义的错误响应至关重要。

### HTTP 状态码

对于 HTTP 传输，将错误映射到最能代表错误条件的标准 HTTP 状态码。映射在 `HTTP` DSL 中定义：

```go
var _ = Service("calc", func() {
    Method("divide", func() {
        // 定义可能的错误及其描述
        Error("div_by_zero", ErrorResult, "除以零错误")
        Error("overflow", ErrorResult, "数字溢出错误")
        Error("unauthorized", ErrorResult, "需要身份验证")

        HTTP(func() {
            POST("/")
            // 将每个错误映射到适当的 HTTP 状态码
            Response("div_by_zero", StatusBadRequest)
            Response("overflow", StatusUnprocessableEntity)
            Response(
authorized", StatusUnauthorized)
        })
    })
})
```

发生错误时，Goa 将：
1. 将错误与其设计定义相匹配
2. 在 HTTP 响应中使用映射的状态码
3. 根据响应定义序列化错误
4. 包括任何指定的标头或元数据

### gRPC 状态码

对于 gRPC 传输，将错误映射到标准 gRPC 状态码。映射遵循类似的原则，但使用 gRPC 特定的代码：

```go
var _ = Service("calc", func() {
    Method("divide", func() {
        // 定义可能的错误及其描述
        Error("div_by_zero", ErrorResult, "除以零错误")
        Error("overflow", ErrorResult, "数字溢出错误")
        Error("unauthorized", ErrorResult, "需要身份验证")

        GRPC(func() {
            // 将每个错误映射到适当的 gRPC 状态码
            Response("div_by_zero", CodeInvalidArgument)
            Response("overflow", CodeOutOfRange)
            Response("unauthorized", CodeUnauthenticated)
        })
    })
})
```

常见的 gRPC 状态码映射：
- `CodeInvalidArgument`: 用于验证错误（例如，div_by_zero）
- `CodeNotFound`: 用于资源未找到错误
- `CodeUnauthenticated`: 用于身份验证错误
- `CodePermissionDenied`: 用于授权错误
- `CodeDeadlineExceeded`: 用于超时错误
- `CodeInternal`: 用于服务器端故障

### 默认映射

如果未提供显式映射：
- HTTP: 对验证错误使用状态码 400 (Bad Request)，对其他错误使用 500 (Internal Server Error)
- gRPC: 对未映射的错误使用 `CodeUnknown`

## 下一步

现在您已经了解了 Goa 中错误处理的基础知识，请探索以下主题以加深您的知识：

- [领域与传输错误](1-domain-vs-transport.md) - 了解如何将业务逻辑错误与其传输表示分离
- [错误传播](2-error-propagation.md) - 了解错误如何在您的服务的不同层之间流动
- [错误序列化](3-error-serialization.md) - 了解如何自定义错误序列化和格式化

这些指南将帮助您在 Goa 服务中实施全面的错误处理策略。

