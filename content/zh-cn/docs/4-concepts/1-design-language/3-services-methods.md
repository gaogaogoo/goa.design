---
title: "服务与方法"
linkTitle: "服务与方法"
weight: 3
description: >
  使用 Goa 的服务定义 DSL 来描述 API 的服务与方法。以强类型的载荷与结果创建清晰、文档完善的端点。
---

## 服务（Services）

在 Goa 中，服务代表一组相关方法，它们协同工作以提供特定功能。服务有助于将你的 API 组织成合理的逻辑分组。

### Service DSL

Service DSL 提供多种选项来配置与记录服务：

```go
var _ = Service("users", func() {
    // 基础文档
    Description("User management service")
    
    // 详细文档
    Docs(func() {
        Description("Detailed documentation for the user service")
        URL("https://example.com/docs/users")
    })

    // 服务级错误定义
    Error("unauthorized", String, "Authentication failed")
    Error("not_found", NotFound, "Resource not found")
    
    // 服务范围元信息
    Meta("swagger:tag", "Users")
    Meta("rpc:package", "usersvc")
    
    // 安全要求
    Security(OAuth2, func() {
        Scope("read:users")
        Scope("write:users")
    })
    
    // 服务级变量
    Variable("version", String, func() {
        Description("API version")
        Default("v1")
        Enum("v1", "v2")
    })
    
    // 方法
    Method("create", func() {
        // ... 方法定义
    })
    
    Method("list", func() {
        // ... 方法定义
    })
    
    // 服务提供的静态文件
    Files("/docs", "./swagger", func() {
        Description("API documentation")
    })
})
```

### 服务级错误（Service-Level Errors）

定义可被服务中所有方法返回的错误：

```go
var _ = Service("orders", func() {
    // 服务中所有方法都可能返回该错误
    Error("unauthorized")
})
```

> 注意：`Error` DSL 用于定义可被服务内所有方法返回的错误。它不适合只针对少数方法的特定错误。后者请在方法或 API 定义中使用 `Error` DSL 来完成。

### 服务文档（Service Documentation）

使用 Docs DSL 提供详细文档：

```go
var _ = Service("payments", func() {
    Description("Payment processing service")
    
    Docs(func() {
        Description(`The payment service handles all payment-related operations.
            
It provides methods for:
- Processing payments
- Refunding transactions
- Querying payment status
- Managing payment methods`)
        
        URL("https://example.com/docs/payments")
        
        // 额外文档元信息
        Meta("doc:section", "Financial Services")
        Meta("doc:category", "Core APIs")
    })
})
```

### 多服务（Multiple Services）

复杂 API 可以拆分为多个服务：

```go
var _ = Service("users", func() {
    Description("User management service")
    // ... 用户相关方法
})

var _ = Service("billing", func() {
    Description("Billing and payment service")
    // ... 计费相关方法
})
```

## 方法（Methods）

方法定义了在服务中可执行的操作。每个方法都要明确其输入（载荷）、输出（结果）以及错误条件。

### 基本方法结构

```go
Method("add", func() {
    Description("Add two numbers together")
    
    // 输入参数
    Payload(func() {
        Field(1, "a", Int32, "First operand")
        Field(2, "b", Int32, "Second operand")
        Required("a", "b")
    })
    
    // 成功响应
    Result(Int32)
    
    // 错误响应
    Error("overflow")
})
```

### 载荷类型（Payload Types）

方法可以接受不同类型的载荷：

```go
// 使用已有类型作为简单载荷
Method("getUser", func() {
    Payload(String, "User ID")
    Result(User)
})

// 内联定义结构化载荷
Method("createUser", func() {
    Payload(func() {
        Field(1, "name", String, "User's full name")
        Field(2, "email", String, "Email address", func() {
            Format(FormatEmail)
        })
        Field(3, "role", String, "User role", func() {
            Enum("admin", "user", "guest")
        })
        Required("name", "email", "role")
    })
    Result(User)
})

// 引用预定义载荷类型
Method("updateUser", func() {
    Payload(UpdateUserPayload)
    Result(User)
})
```

### 结果类型（Result Types）

方法可以返回不同类型的结果：

```go
// 简单原始类型结果
Method("count", func() {
    Result(Int64)
})

// 内联定义结构化结果
Method("search", func() {
    Result(func() {
        Field(1, "items", ArrayOf(User), "Matching users")
        Field(2, "total", Int64, "Total count")
        Required("items", "total")
    })
})
```

### 错误处理（Error Handling）

为方法定义期望的错误条件：

```go
Method("divide", func() {
    Payload(func() {
        Field(1, "a", Float64, "Dividend")
        Field(2, "b", Float64, "Divisor")
        Required("a", "b")
    })
    Result(Float64)
    
    // 方法级错误
    Error("division_by_zero", func() {
        Description("Attempted to divide by zero")
    })
})
```

### 流式方法（Streaming Methods）

Goa 同时支持载荷与结果的流式处理：

```go
Method("streamNumbers", func() {
    Description("Stream a sequence of numbers")
    
    // 输入为整数流
    StreamingPayload(Int32)
    
    // 输出为整数流
    StreamingResult(Int32)
})

Method("processEvents", func() {
    Description("Process a stream of events")
    
    // 流式结构化数据
    StreamingPayload(func() {
        Field(1, "event_type", String)
        Field(2, "data", Any)
        Required("event_type", "data")
    })
    
    // 返回汇总结果
    Result(func() {
        Field(1, "processed", Int64, "Number of events processed")
        Field(2, "errors", Int64, "Number of errors encountered")
        Required("processed", "errors")
    })
})
```

流式处理详情参见 [流式教程](../../3-tutorials/4-streaming)。

## 最佳实践

{{< alert title="服务设计指引" color="primary" >}}
服务组织
- 将相关功能归组到服务
- 保持服务范围聚焦与内聚
- 使用清晰、描述性的服务名
- 记录服务的目的与使用方式

方法设计
- 使用清晰、动作导向的方法名
- 提供详细描述
- 定义合适的错误响应
- 考量校验需求
- 记录期望行为

类型使用
- 使用强类型的载荷与结果
- 为常见结构定义可复用类型
- 使用合适的校验规则
- 提供有意义的示例
{{< /alert >}}
