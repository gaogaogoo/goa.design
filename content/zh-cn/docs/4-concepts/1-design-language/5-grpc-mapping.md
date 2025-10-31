---
title: "传输映射"
linkTitle: "传输映射"
weight: 5
description: >
  定义服务如何在不同传输协议上通信。将服务方法映射到 HTTP 与 gRPC 端点。
---

## 传输映射概览

Goa 同时支持 HTTP 与 gRPC。传输映射 DSL 允许你定义服务方法如何在这些协议上暴露。

## HTTP 传输

[HTTP DSL](https://pkg.go.dev/goa.design/goa/v3/dsl#HTTP) 用于定义服务方法如何映射到 HTTP 端点。你可以在三个层级进行配置：
- API 层：定义全局 HTTP 设置
- 服务层：配置服务范围的 HTTP 属性
- 方法层：指定方法级的 HTTP 行为

### 映射层级

#### API 层
定义作用于所有服务的全局 HTTP 设置：
```go
API("bookstore", func() {
    HTTP(func() {
        Path("/api/v1") // 所有端点的全局前缀
    })
})
```

#### 服务层
为服务内所有方法配置 HTTP 属性：
```go
Service("books", func() {
    HTTP(func() {
        Path("/books")     // 服务范围路径前缀
        Parent("store")    // 父服务，用于路径嵌套
    })
})
```

#### 方法层
为单个方法定义具体的 HTTP 行为：
```go
Method("show", func() {
    HTTP(func() {
        GET("/{id}")       // HTTP 方法与路径
        Response(StatusOK) // 成功响应码
    })
})
```

### HTTP 映射特性

HTTP DSL 提供多种端点配置能力：

1. 路径参数
   - 将载荷字段映射到 URL 路径片段
   - 使用模式匹配与校验
   - 支持可选参数

2. 查询参数
   - 将载荷字段映射到查询字符串参数
   - 定义参数类型与校验
   - 处理可选参数

3. 头
   - 将载荷/结果字段映射到 HTTP 头
   - 设置必填与可选头
   - 定义头的格式与校验

4. 响应码
   - 将结果映射到成功状态码
   - 定义错误响应码
   - 处理不同响应场景

## gRPC 传输

gRPC DSL 用于定义服务方法如何映射到 gRPC 过程。与 HTTP 类似，它也可在多个层级进行配置。

### gRPC 特性

1. 消息映射
   - 定义请求/响应消息结构
   - 将字段映射到 protobuf 类型
   - 配置字段编号与选项

2. 状态码
   - 将服务结果映射到 gRPC 状态码
   - 定义错误码映射
   - 处理标准 gRPC 状态场景

3. 元数据
   - 配置 gRPC 元数据处理
   - 将头映射到元数据
   - 定义元数据校验

### 常见模式

以下是一些常见的传输映射模式：

#### REST 风格资源映射
```go
Service("users", func() {
    HTTP(func() {
        Path("/users")
    })
    
    Method("list", func() {
        HTTP(func() {
            GET("/")              // GET /users
        })
    })
    
    Method("show", func() {
        HTTP(func() {
            GET("/{id}")          // GET /users/{id}
        })
    })
})
```

#### 混合协议支持
服务可以同时支持 HTTP 与 gRPC：
```go
Method("create", func() {
    // HTTP 映射
    HTTP(func() {
        POST("/")
        Response(StatusCreated)
    })
    
    // gRPC 映射
    GRPC(func() {
        Response(CodeOK)
    })
})
```

## 最佳实践

{{< alert title="传输映射指引" color="primary" >}}
HTTP 设计
- 使用一致的 URL 模式
- 遵循 REST 约定
- 选择合适的状态码
- 一致地处理错误

gRPC 设计
- 使用有意义的服务名
- 定义清晰的消息结构
- 遵循 protobuf 最佳实践
- 规划向后兼容

通用建议
- 记录传输层特定行为
- 考量安全影响
- 规划版本化
- 同时测试两种传输层
{{< /alert >}}

## 传输层特定的错误处理

不同传输协议有各自的错误表示方式：

### HTTP 错误
- 映射到合适的状态码
- 在响应体中包含错误详情
- 使用标准头传递额外信息
- 遵循 HTTP 错误约定

### gRPC 错误
- 使用标准 gRPC 状态码
- 包含详细错误消息
- 利用 error details 特性
- 遵循 gRPC 错误模型
