---
title: "设计 gRPC 服务"
linkTitle: "设计"
weight: 1
description: "学习在 Goa 中设计 gRPC 服务，包括服务定义、方法字段标注、protobuf 生成，以及正确的 gRPC 状态码映射。"
---

本教程将带您在 Goa 中设计一个简单的 gRPC 服务。虽然 Goa 经常用于 REST 端点，但它同样对 **gRPC** 传输提供一等支持。您将学到：

- 在 Goa DSL 中定义服务与方法；
- 为 gRPC 进行标注，确保生成代码包含 `.proto` 文件；
- 校验 Payload，并将错误映射到 gRPC 状态码。

## 我们将构建什么

我们将创建一个 **`greeter`** 服务，包含一个名为 `SayHello` 的方法。该方法在请求体中接收姓名并返回问候语。同时演示如何使用标准 gRPC 状态码来限定（qualify）响应。

| Method   | gRPC RPC      | Description                      |
|----------|---------------|----------------------------------|
| SayHello | rpc SayHello  | 根据用户提供的姓名返回问候语 |

## 1. 创建 Module 与目录

为项目 `grpcgreeter` 创建全新 Go module：

```bash
mkdir grpcgreeter
cd grpcgreeter
go mod init grpcgreeter
```

在该目录下创建 `design/` 存放 DSL 文件：

```bash
mkdir design
```

## 2. 编写服务设计

新建 `design/greeter.go` 并写入：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

// gRPC 问候服务
var _ = Service("greeter", func() {
    Description("一个简单的 gRPC 问候服务。")

    Method("SayHello", func() {
        Description("向用户发送问候。")

        // 请求载荷（客户端发送）
        Payload(func() {
            Field(1, "name", String, "要问候的用户名", func() {
                Example("Alice")
                MinLength(1)
            })
            Required("name")
        })

        // 返回结果（服务端返回）
        Result(func() {
            Field(1, "greeting", String, "友好的问候消息")
            Required("greeting")
        })

        // 暴露为 gRPC 方法
        GRPC(func() {
            // 成功响应默认是 CodeOK (0)
            // 如需自定义映射可使用：
            // Response(CodeOK)
        })
    })
})
```

### 关键点

- 使用 `Method("SayHello", ...)` 定义远程过程调用；
- **Payload** 指定输入字段，对应 gRPC 的请求消息；
- **Result** 指定输出字段，对应 gRPC 的响应消息；
- 添加 **`GRPC(func() {...})`** 以生成 `.proto` 定义与桩代码；
- 通过 `Field(1, "name", ...)` 定义请求与响应消息的字段编号（tag），该编号用于生成的 `.proto` 文件。支持 HTTP 与 gRPC 的方法建议统一使用 `Field` 来定义字段（HTTP 传输会忽略该编号）。

## 3. 下一步

完成 **gRPC 服务设计** 后，继续：

- [实现服务](./2-implementing.md)：生成代码、编写业务逻辑，并学习如何在 Goa 中运行 gRPC 服务器；
- [运行服务](./3-running.md)：使用官方 gRPC CLI 或其它工具调用端点并验证行为。

至此，您已使用 Goa 设计了最简 gRPC 服务。DSL 的单一真源让请求/响应类型、校验与 gRPC 状态映射集中管理，使服务更易于演进与维护。