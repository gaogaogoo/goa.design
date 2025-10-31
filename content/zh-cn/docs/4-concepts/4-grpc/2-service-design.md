---
title: "服务设计"
linkTitle: "服务设计"
weight: 2
description: "学习在 Goa 中设计 gRPC 服务，包括服务定义、消息类型与类型系统"
---

本文解释如何使用 Goa 的 DSL 设计 gRPC 服务，重点介绍服务定义与消息类型。有关 Goa 类型系统与数据建模能力的完整概览，请参阅 [数据建模](/docs/concepts/design-language/data-modeling)。

## 服务定义

使用启用 `GRPC` 传输的 `Service` DSL 来定义 Goa gRPC 服务：

```go
var _ = Service("calculator", func() {
    Description("The Calculator service performs arithmetic operations")

    // 启用 gRPC 传输
    GRPC(func() {
        // 服务级 gRPC 设置
        Metadata("package", "calculator.v1")
        Metadata("go.package", "calculatorpb")
    })

    // 服务方法...
})
```

### 方法定义

方法定义服务提供的操作。可在方法级使用 `GRPC` DSL 自定义请求/响应处理：

```go
Method("add", func() {
    Description("Add two numbers")

    // 输入消息
    Payload(func() {
        Field(1, "a", Int, "第一个操作数")
        Field(2, "b", Int, "第二个操作数")
        Required("a", "b")
    })

    // 输出消息
    Result(func() {
        Field(1, "sum", Int, "加法结果")
        Required("sum")
    })

    // 方法级 gRPC 设置
    GRPC(func() {
        // 定义成功响应
        Response(CodeOK)
        
        // 定义错误响应
        Response("not_found", CodeNotFound)
        Response("invalid_argument", CodeInvalidArgument)
    })
})
```

### 请求-响应自定义

`GRPC` DSL 提供多种函数用于自定义数据的传输方式：

#### 消息定制

使用 `Message` 自定义哪些载荷字段进入 gRPC 请求消息：

```go
var CreatePayload = Type("CreatePayload", func() {
    Field(1, "name", String, "账户名称")
    TokenField(2, "token", String, "JWT 令牌")
    Field(3, "metadata", String, "附加信息")
})

Method("create", func() {
    Payload(CreatePayload)
    
    GRPC(func() {
        // 请求消息仅包含指定字段
        Message(func() {
            Attribute("name")
            Attribute("metadata")
        })
        Response(CodeOK)
    })
})
```

#### 元数据处理

使用 `Metadata` 指定哪些载荷字段应作为 gRPC 元数据（metadata）发送而不是放入消息体：

```go
Method("create", func() {
    Payload(CreatePayload)
    
    GRPC(func() {
        // 将 token 放入元数据
        Metadata(func() {
            Attribute("token")
        })
        Response(CodeOK)
    })
})
```

> 注意：与安全相关的属性（通过 `TokenField` 定义或使用 `Security` 方案）会自动包含在请求元数据中，除非在 `Message` 中显式包含到消息体。

#### 响应头与尾部（Trailers）

使用 `Headers` 与 `Trailers` 控制响应元数据：

```go
var CreateResult = ResultType("application/vnd.create", func() {
    Field(1, "name", String, "资源名称")
    Field(2, "id", String, "资源 ID")
    Field(3, "status", String, "处理状态")
})

Method("create", func() {
    Result(CreateResult)
    
    GRPC(func() {
        Response(func() {
            Code(CodeOK)
            // 在响应头中发送 ID
            Headers(func() {
                Attribute("id")
            })
            // 在 trailers 中发送状态
            Trailers(func() {
                Attribute("status")
            })
        })
    })
})
```

## gRPC 中的消息类型

在设计 gRPC 服务时，你将使用 Goa 的类型系统来定义消息类型。与常规类型定义的主要差异是使用 `Field` DSL（而非 `Attribute`）来指定 Protocol Buffer 字段编号。

### 字段编号

遵循 Protocol Buffer 的字段编号最佳实践：

1. 高频字段使用 1-15（1 字节编码）
2. 低频字段使用 16-2047（2 字节编码）
3. 预留编号以保持向后兼容

```go
Method("createUser", func() {
    Payload(func() {
        // 高频字段（1 字节编码）
        Field(1, "id", String)
        Field(2, "name", String)
        Field(3, "email", String)

        // 低频字段（2 字节编码）
        Field(16, "preferences", func() {
            Field(1, "theme", String)
            Field(2, "language", String)
        })
    })
})
```

### 使用复杂类型

可在 gRPC 服务中使用 [数据建模](/docs/concepts/design-language/data-modeling) 指南中描述的全部类型系统特性。以下展示常见模式的用法：

#### 结构体与嵌套类型

```go
var Address = Type("Address", func() {
    Field(1, "street", String)
    Field(2, "city", String)
    Field(3, "country", String)
    Required("street", "city", "country")
})

var User = Type("User", func() {
    Field(1, "id", String)
    Field(2, "name", String)
    Field(3, "address", Address)  // 嵌套类型
    Required("id", "name")
})
```

#### OneOf 类型

使用 `OneOf` 定义互斥字段：

```go
var ContactInfo = Type("ContactInfo", func() {
    OneOf("contact", func() {
        Field(1, "email", String)
        Field(2, "phone", String)
        Field(3, "address", Address)
    })
})
```

## 最佳实践

### 前向兼容

为未来的可扩展性设计消息：

1. 新增字段使用可选
2. 预留字段编号与名称
3. 将相关字段分组到嵌套消息中

```go
var UserProfile = Type("UserProfile", func() {
    // 当前版本字段
    Field(1, "basic_info", func() {
        Field(1, "name", String)
        Field(2, "email", String)
    })

    // 为未来使用预留
    Reserved(2, 3, 4)
    ReservedNames("location", "department")

    // 扩展点
    Field(5, "extensions", MapOf(String, Any))
})
```

### 文档化

添加全面的文档：

```go
var _ = Service("users", func() {
    Description("The users service manages user accounts and profiles")

    Method("create", func() {
        Description("创建新用户账户")

        Payload(func() {
            Field(1, "username", String, "账户唯一用户名")
            Field(2, "email", String, "主邮箱地址")
            Field(3, "full_name", String, "用户全名")
            Example("username", "johndoe")
            Example("email", "john@example.com")
        })

        Result(func() {
            Field(1, "id", String, "新建用户的唯一标识")
            Field(2, "created_at", String, func() {
                Format(FormatDateTime)
                Description("账户创建时间戳")
            })
        })
    })
})
```

有关校验规则、内置格式与示例的更多信息，请参阅 [数据建模](/docs/concepts/design-language/data-modeling) 指南。