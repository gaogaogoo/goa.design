---
title: "API 定义"
linkTitle: "API"
weight: 2
description: >
  使用 Goa 的 API DSL 定义服务的全局属性，配置元数据、文档、服务器以及全局设置。
---

## API 定义

`API` 函数是顶层 DSL，用于定义服务的全局属性。它是设计的根，奠定其他所有组件的基础。每个设计包只能包含一个 API 声明，作为服务定义的入口。

### 目的与用法

API 定义的主要作用包括：
- 为 API 文档提供元数据
- 配置服务器端点与变量
- 为所有服务建立全局设置
- 定义文档与许可信息
- 设置联系与支持信息

### 基本结构

下面是一个最小化的 API 定义：

```go
var _ = API("calculator", func() {
    Title("Calculator API")
    Description("A simple calculator service")
    Version("1.0.0")
})
```

这将创建一个名为 "calculator" 的 API，并带有基础文档。API 名称应为有效的 Go 标识符，因为它会用于生成的代码中。

### 完整示例

下面是一个全面的示例，展示了所有可用的 API 选项以及详细说明：

```go
var _ = API("bookstore", func() {
    // 基础 API 信息 —— 用于 OpenAPI 文档
    Title("Bookstore API")
    Description(`A modern bookstore management API.
    
This API provides endpoints for:
- Managing books and inventory
- Processing orders
- Customer management
- Analytics and reporting`)
    Version("2.0.0")
    
    // 服务条款 —— 法律要求与使用条款
    TermsOfService("https://example.com/terms")
    
    // 联系方式 —— 支持渠道
    Contact(func() {
        Name("API Support")
        Email("support@example.com")
        URL("https://example.com/support")
    })
    
    // 许可信息 —— API 的使用许可
    License(func() {
        Name("Apache 2.0")
        URL("https://www.apache.org/licenses/LICENSE-2.0.html")
    })
    
    // 文档 —— 详细指南与参考
    Docs(func() {
        Description(`Comprehensive API documentation including:
- Getting started guides
- Authentication details
- API reference
- Best practices
- Example code`)
        URL("https://example.com/docs")
    })
    
    // 服务器定义 —— API 的访问位置
    Server("production", func() {
        Description("Production server")
        
        // 多主机与变量
        Host("production", func() {
            Description("Production host")
            // URI 中的变量在运行时替换
            URI("https://{version}.api.example.com")
            URI("grpcs://{version}.grpc.example.com")
            
            // 定义版本变量
            Variable("version", String, "API version", func() {
                Default("v2")
                Enum("v1", "v2")
            })
        })
    })
    
    // 开发服务器 —— 用于测试
    Server("development", func() {
        Description("Development server")
        
        Host("localhost", func() {
            // 本地开发端点
            URI("http://localhost:8000")
            URI("grpc://localhost:8080")
        })
    })
})
```

### API 属性详解

#### 基础元数据
这些属性是 API 文档与发现的核心：

- `Title`：API 的简短描述性名称
- `Description`：API 的详细说明
- `Version`：API 版本，通常遵循语义化版本
- `TermsOfService`：服务条款链接

支持 Markdown 的示例：
```go
Title("Order Management API")
Description(`
# Order Management API

This API allows you to:
- Create and manage orders
- Track shipments
- Process returns
- Generate invoices

## Rate Limits
- 1000 requests per hour for authenticated users
- 100 requests per hour for anonymous users
`)
```

#### 联系信息
联系方式帮助 API 使用者在需要支持时进行沟通：

```go
Contact(func() {
    Name("API Team")
    Email("api@example.com")
    URL("https://example.com/contact")
})
```

此信息会显示在 API 文档中，帮助用户在需要时获得协助。

#### 许可信息
指定你的 API 如何被使用：

```go
License(func() {
    Name("MIT")
    URL("https://opensource.org/licenses/MIT")
})
```

许可信息对理解使用权与限制至关重要。

#### 文档链接
提供额外的文档资源：

```go
Docs(func() {
    Description(`Detailed documentation including:
- API Reference
- Integration Guides
- Best Practices
- Example Code`)
    URL("https://example.com/docs")
})
```

### 服务器配置

服务器定义了可以访问 API 的端点。你可以为不同环境定义多个服务器：

```go
Server("main", func() {
    Description("Main API server")
    
    // 生产主机
    Host("production", func() {
        Description("Production endpoints")
        // 同时支持 HTTP 与 gRPC
        URI("https://api.example.com")
        URI("grpcs://grpc.example.com")
    })
    
    // 带变量的区域主机
    Host("regional", func() {
        Description("Regional endpoints")
        URI("https://{region}.api.example.com")
        
        // 定义 region 变量
        Variable("region", String, "Geographic region", func() {
            Description("AWS region for the API endpoint")
            Default("us-east")
            Enum("us-east", "us-west", "eu-central")
        })
    })
})
```

URI 中的变量使配置更灵活，常见用途包括：
- 支持多区域
- 多版本 API
- 环境差异化配置
- 多租户管理

### 最佳实践

{{< alert title="API 设计指引" color="primary" >}}
文档
- 提供清晰、简洁的标题与描述
- 使用 Markdown 进行富文档编写
- 补充完善的联系信息
- 链接到更详细的外部文档
- 明确许可与服务条款

版本化
- 采用语义化版本（MAJOR.MINOR.PATCH）
- 在服务器 URL 中包含版本信息
- 提前规划版本迁移与向后兼容
- 记录版本间的不兼容变更

服务器配置
- 定义所有生产与开发服务器
- 使用变量提升配置灵活性
- 需要时同时提供 HTTP 与 gRPC 端点
- 说明各环境与其用途
- 为变量提供合理默认值
- 考量区域与扩展性需求

通用建议
- 描述聚焦且相关
- 命名一致
- 预留扩展空间
- 注重安全影响
- 记录频控与配额
{{< /alert >}}

### API 级错误

API DSL 允许在 API 层定义可在所有服务与方法间复用的错误。这能促进错误处理的一致性并减少设计中的重复。

#### 目的与收益

在 API 层定义错误可以建立一致的术语与结构，在各处复用。集中式的方式确保统一的错误处理，同时简化文档与传输映射。与其在多个地方重复定义相似错误，不如按需引用共享定义，从而在整个 API 中提升一致性与可维护性。

#### 错误定义如何工作

当你在 API 层定义错误时，需要确定：
1. 唯一的错误标识符
2. 错误的数据结构（类型）
3. 文档与描述
4. 可选的传输层行为

示例（基于以下 API 定义）：
```go
var _ = API("bookstore", func() {
    Error("unauthorized", ErrorResult, "Authentication failed")
    HTTP(func() {
        Response("unauthorized", StatusUnauthorized)
    })
    GRPC(func() {
        Response("unauthorized", CodeUnauthenticated)
    })
})
```

服务与方法可以通过名称引用 API 级错误：

```go
var _ = Service("billing", func() {
    Error("unauthorized") // 无需再次指定类型、描述或传输映射
})
```

#### 错误的继承

服务与方法按名称引用 API 级错误时：
- 继承该错误的所有属性
- 可以添加传输层映射
- 不能修改错误结构
- 可以补充上下文相关的文档

这种继承模型在确保一致性的同时，允许在使用方式上保持灵活。

#### 默认错误类型

如果未显式指定错误类型，Goa 会使用内置的 `ErrorResult` 类型。该类型包含：
- 错误消息
- 错误 ID
- 可选的临时/超时标记
- 可选的错误原因栈

对于简单错误你可以直接使用该默认类型；而在更复杂的场景中可定义自定义类型。

#### 传输映射

API 级错误可以定义在不同传输协议中的表现方式。例如：
- HTTP 状态码与响应头
- gRPC 状态码
- 自定义错误序列化

当错误在服务与方法中被使用时，这些映射会被继承。

#### 错误设计最佳实践

{{< alert title="错误设计指引" color="primary" >}}
错误组织
- 在 API 层定义通用且可复用的错误
- 使用清晰、可表达含义的错误名

错误用法
- 优先引用 API 层错误而不是重复定义
- 复用错误时补充上下文描述
{{< /alert >}}

#### 常见模式

常见的 API 级错误模式包括：
1. 认证/鉴权错误
2. 资源未找到
3. 校验错误（在设计中已有的校验规则之上）
4. 频率限制错误
5. 服务端错误

通过在 API 层定义这些通用模式，可确保你的服务在错误处理上的一致性。
