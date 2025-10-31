---
title: 将错误映射到传输状态码
linkTitle: 传输映射
weight: 4
description: "学习如何将 Goa 的错误映射到合适的 HTTP 与 gRPC 状态码，确保在不同传输协议下返回一致的错误响应。"
---

在 Goa 的 DSL 中定义好错误后，下一步是将这些错误映射到相应的传输层状态码。这样可以确保客户端根据错误性质获得有意义、规范化的响应。Goa 允许你使用 DSL 中的 `Response` 函数为不同传输协议（如 HTTP 与 gRPC）定义这些映射。

## HTTP 传输映射

对于 HTTP 传输，你可以在服务或方法定义中使用 `HTTP` 函数将错误映射到具体的 HTTP 状态码。该映射确保当发生错误时，客户端能够收到带有正确状态码与错误信息的 HTTP 响应。

示例

```go
var _ = Service("divider", func() {
    Error("DivByZero", func() {
        Description("DivByZero is the error returned when the divisor is zero.")
    })

    HTTP(func() {
        // 将 "DivByZero" 错误映射到 HTTP 400 Bad Request
        Response("DivByZero", StatusBadRequest)
    })

    Method("integral_divide", func() {
        Error("HasRemainder", func() {
            Description("HasRemainder is returned when an integer division has a remainder.")
        })

        HTTP(func() {
            // 将 "HasRemainder" 错误映射到 HTTP 417 Expectation Failed
            Response("HasRemainder", StatusExpectationFailed)
        })

        // 其他方法定义…
    })

    Method("divide", func() {
        // 方法特定的定义…
    })
})
```

在此示例中：

- `DivByZero`：映射到 HTTP 状态码 400 Bad Request。
- `HasRemainder`：映射到 HTTP 状态码 417 Expectation Failed。

## 定义响应

在 `HTTP` 函数中，使用 `Response` 将每个错误与一个 HTTP 状态码关联。语法如下：

```go
Response("<ErrorName>", <HTTPStatusCode>, func() {
    Description("<Optional description>")
})
```

- `<ErrorName>`：在 DSL 中定义的错误名称。
- `<HTTPStatusCode>`：要映射的 HTTP 状态码。
- `Description`：（可选）为文档目的提供响应描述。

## 完整的 HTTP 映射示例

```go
var _ = Service("divider", func() {
    // 服务级错误
    Error("DivByZero", func() {
        Description("DivByZero is the error returned when the divisor is zero.")
    })

    HTTP(func() {
        // 服务范围的错误映射
        Response("DivByZero", StatusBadRequest)           // 400
    })

    Method("integral_divide", func() {
        Description("Performs integer division and checks for remainders")
        
        Payload(func() {
            Field(1, "dividend", Int, "Number to be divided")
            Field(2, "divisor", Int, "Number to divide by")
            Required("dividend", "divisor")
        })
        Result(Int)

        Error("HasRemainder", func() {
            Description("HasRemainder is returned when an integer division has a remainder.")
        })

        HTTP(func() {
            POST("/divide/integral")
            
            // 方法特定的错误映射
            Response("HasRemainder", StatusExpectationFailed, func() { // 417
                Description("Returned when the division results in a remainder")
            })
        })
    })

    Method("divide", func() {
        Description("Performs floating-point division")
        
        Payload(func() {
            Field(1, "dividend", Float64, "Number to be divided")
            Field(2, "divisor", Float64, "Number to divide by")
            Required("dividend", "divisor")
        })
        Result(Float64)

        Error("Overflow", func() {
            Description("Overflow is returned when the result exceeds maximum value.")
        })

        HTTP(func() {
            POST("/divide")
            
            // 方法特定的错误映射
            Response("Overflow", StatusUnprocessableEntity, func() { // 422
                Description("Returned when the division result exceeds maximum value")
            })
        })
    })
})
```

该示例展示了：

1. 服务级错误：适用于所有方法的通用错误：
   - `DivByZero`：尝试以零为除数时

2. 方法特定错误：每个方法定义其特定错误：
   - `integral_divide`：处理有余数的情况
   - `divide`：处理浮点溢出

3. HTTP 状态码映射：
   - 400 Bad Request：除数为零
   - 417 Expectation Failed：整数除法有余数
   - 422 Unprocessable Entity：浮点溢出

4. 不同端点：展示两个不同除法操作的错误映射：
   - `/divide/integral` 用于整数除法
   - `/divide` 用于浮点除法

这些映射确保每种错误情况都返回一个准确反映错误性质的 HTTP 状态码。

## gRPC 传输映射

对于 gRPC 传输，你可以在服务或方法定义中使用 `GRPC` 函数将错误映射到具体的 gRPC 状态码。该映射确保当发生错误时，客户端能够收到带有正确状态码与错误信息的 gRPC 响应。

示例

```go
var _ = Service("divider", func() {
    Error("DivByZero", func() {
        Description("DivByZero is the error returned when the divisor is zero.")
    })

    GRPC(func() {
        // 将 "DivByZero" 错误映射到 gRPC 状态码 InvalidArgument（3）
        Response("DivByZero", CodeInvalidArgument)
    })

    Method("integral_divide", func() {
        Error("HasRemainder", func() {
            Description("HasRemainder is returned when an integer division has a remainder.")
        })

        GRPC(func() {
            // 将 "HasRemainder" 错误映射到 gRPC 状态码 Unknown（2）
            Response("HasRemainder", CodeUnknown)
        })

        // 其他方法定义…
    })

    Method("divide", func() {
        // 方法特定的定义…
    })
})
```

在此示例中：

- `DivByZero`：映射到 gRPC 状态码 InvalidArgument（代码 3）。
- `HasRemainder`：映射到 gRPC 状态码 Unknown（代码 2）。

## 定义响应

在 `GRPC` 函数中，使用 `Response` 将每个错误与一个 gRPC 状态码关联。语法如下：

```go
Response("<ErrorName>", Code<StatusCode>, func() {
    Description("<Optional description>")
})
```

- `<ErrorName>`：在 DSL 中定义的错误名称。
- `Code<StatusCode>`：要映射的 gRPC 状态码，前缀为 Code。
- `Description`：（可选）为文档目的提供响应描述。

## 同时定义 HTTP 与 gRPC 映射

Goa 允许你在同一服务或方法内同时定义 HTTP 与 gRPC 的映射。当服务支持多种传输协议时，这样可确保无论客户端使用哪种协议，错误都能被正确映射。

示例

```go
var _ = Service("divider", func() {
    Error("DivByZero", func() {
        Description("DivByZero is returned when attempting to divide by zero.")
    })

    Method("divide", func() {
        Payload(func() {
            Field(1, "dividend", Float64, "Number to be divided")
            Field(2, "divisor", Float64, "Number to divide by")
            Required("dividend", "divisor")
        })
        Result(Float64)

        HTTP(func() {
            POST("/divide")
            // 将除数为零映射到 HTTP 422 Unprocessable Entity
            Response("DivByZero", StatusUnprocessableEntity)
        })

        GRPC(func() {
            // 将除数为零映射到 INVALID_ARGUMENT
            Response("DivByZero", CodeInvalidArgument)
        })
    })
})
```

在此示例中，`DivByZero` 错误同时被映射到：

- HTTP 状态码 422 Unprocessable Entity。
- gRPC 状态码 InvalidArgument（代码 3）。

## 总结

在 Goa 中将错误映射到特定传输层状态码，能确保客户端依据错误性质获得清晰且合适的响应。通过在 DSL 中定义这些映射，Goa 自动生成所需代码与文档，保持一致性并减少样板代码。无论你使用 HTTP、gRPC 或两者兼有，Goa 灵活的错误映射能力都能帮助你构建健壮且友好的 API。