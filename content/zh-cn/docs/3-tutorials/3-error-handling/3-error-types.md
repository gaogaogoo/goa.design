---
title: "错误类型"
linkTitle: 错误类型
weight: 3
description: "探索 Goa 的错误类型系统，包括默认的 ErrorResult 类型，以及如何创建自定义错误类型以应对更复杂的错误场景。"
---

Goa 允许你使用默认的 `ErrorResult` 类型或自定义的用户定义类型来定义错误。选择合适的错误类型取决于你需要表示的错误的复杂度与特异性。

## 默认错误类型（`ErrorResult`）

默认情况下，错误使用 `ErrorResult` 类型，该类型包含标准字段，如 `Name`、`ID`、`Message`、`Temporary`、`Timeout` 和 `Fault`。它为你的服务提供了统一的错误结构。

### 结构与字段

- Name：在 DSL 中定义的错误名称。
- ID：该错误实例的唯一标识符，用于关联日志与追踪。
- Message：描述性的错误消息。
- Temporary：指示错误是否为临时的，重试后可能恢复。
- Timeout：指示错误是否由超时导致。
- Fault：指示错误是否由于服务端故障引起。

### 使用示例

使用默认的 `ErrorResult` 定义服务级错误：

```go
var _ = Service("divider", func() {
    Error("DivByZero", ErrorResult, "Division by zero")
    Error("ServiceUnavailable", ErrorResult, "Service is temporarily unavailable.", func() {
        Temporary()
    })
    // ...
})
```

在此示例中，我们定义了两个可被 `divider` 服务中任意方法返回的服务级错误。`DivByZero` 表示除数为零的操作，而 `ServiceUnavailable` 表示服务的临时不可用。两者都使用默认的 `ErrorResult` 类型，但 `ServiceUnavailable` 通过 `Temporary()` 标记为临时错误，提示客户端可以重试。

### 运行时表示

生成的代码会定义函数以实例化
[ServiceError](https://pkg.go.dev/goa.design/goa/v3/pkg#ServiceError) 对象，
用于 `DivByZero` 与 `ServiceUnavailable` 错误。这些函数负责设置错误的合适字段：

```go
// MakeDivByZero 从一个 error 构建 goa.ServiceError。
func MakeDivByZero(err error) *goa.ServiceError {
    return goa.NewServiceError(err, "DivByZero", false, false, false)
}

// MakeServiceUnavailable 从一个 error 构建 goa.ServiceError。
func MakeServiceUnavailable(err error) *goa.ServiceError {
    return goa.NewServiceError(err, "ServiceUnavailable", true, false, false)
}
```

客户端可将服务返回的错误转换为 `ServiceError` 类型，并通过 `Temporary`、`Timeout` 与 `Fault` 字段检查错误细节。

## 自定义错误类型

为了提供更详尽的错误信息，你可以定义自定义错误类型。这允许你包含特定于应用需求的附加字段。

### 创建自定义错误类型

使用 `Type` 函数定义自定义错误类型：

```go
var DivByZero = Type("DivByZero", func() {
    Description("DivByZero is the error returned when using value 0 as divisor.")
    Field(1, "message", String, "Error message for division by zero.")
    Field(2, "divisor", Int, "Divisor that caused the error.")
    Required("message", "divisor")
})
```

### 在服务中使用自定义错误类型

将自定义错误类型集成到服务方法中：

```go
var _ = Service("divider", func() {
    Method("divide", func() {
        Payload(func() {
            Field(1, "dividend", Int)
            Field(2, "divisor", Int)
            Required("dividend", "divisor")
        })
        Result(func() {
            Field(1, "quotient", Int)
            Field(2, "remainder", Int)
            Required("quotient", "remainder")
        })
        Error("DivByZero", DivByZero, "Division by zero")
        // Additional method definitions...
    })
})
```

在此示例中，我们定义了一个名为 `DivByZero` 的方法级错误，它使用了自定义的 `DivByZero` 类型。这使我们能够提供特定于除数为零场景的详细错误信息，包括错误消息和导致错误的实际除数值。

### 使用自定义错误类型的注意事项

- 错误元数据：当在同一方法中使用自定义类型定义多个错误时，必须通过 `struct:error:name` 元数据指定哪个属性包含错误名称。这对 Goa 正确映射错误到其定义至关重要。

```go
var CustomError = Type("CustomError", func() {
    Field(1, "detail", String)
    Field(2, "name", String, func() {
        Meta("struct:error:name")
    })
    Required("detail", "name")
})
```

- 保留属性：自定义错误类型不能包含名为 `error_name` 的属性，因为 Goa 在内部使用它来进行错误标识与序列化。
