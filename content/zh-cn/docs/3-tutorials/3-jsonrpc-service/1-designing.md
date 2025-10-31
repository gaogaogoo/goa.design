---
title: "设计 JSON‑RPC 服务"
linkTitle: "设计"
weight: 1
description: "学习如何使用 Goa 设计 JSON‑RPC 2.0 服务，包括共享端点、方法暴露、ID 映射与传输方式选择。"
---

在本教程中，你将使用 Goa 设计一个简单的 JSON‑RPC 服务。Goa 在 HTTP、SSE 与 WebSocket 上对 JSON‑RPC 2.0 提供一等支持。

你将学到：

- 在 Goa 的 DSL 中定义服务与方法
- 为服务与方法启用 JSON‑RPC
- 将 JSON‑RPC 的 ID 映射到字段
- 选择传输方式（HTTP、SSE、WebSocket）

## 我们将构建什么

我们将创建一个 `calculator` 服务，其中包含一个 `add` 方法用于对两个整数求和。

| 方法 | 描述                     |
|------|--------------------------|
| add  | 返回 `a + b` 的和。      |

## 1. 创建新模块与文件夹

创建新模块 `jsonrpccalc`：

```bash
mkdir jsonrpccalc
cd jsonrpccalc
go mod init jsonrpccalc
```

创建 `design/` 目录：

```bash
mkdir design
```

## 2. 编写服务设计

创建 `design/calculator.go`：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

// JSON‑RPC 计算器服务。
var _ = Service("calculator", func() {
    Description("A simple calculator exposed via JSON‑RPC")

    // 通过 JSON‑RPC 暴露该服务，路由为 POST /rpc
    JSONRPC(func() { POST("/rpc") })

    Method("add", func() {
        Description("Add two integers")

        Payload(func() {
            Attribute("a", Int, "Left operand")
            Attribute("b", Int, "Right operand")
            Required("a", "b")
        })

        Result(func() {
            Attribute("sum", Int, "Sum of a and b")
            Required("sum")
        })

        // 为该方法启用 JSON‑RPC
        JSONRPC(func() {})
    })
})
```

关键点：

- `JSONRPC(func(){ POST("/rpc") })` 为所有服务方法定义一个共享的 HTTP 路由。
- 通过 JSON‑RPC 暴露的每个方法都需声明 `JSONRPC(func(){})`。
- 在非流式场景下，ID 处理是自动的：响应包中的 `id` 等于请求的 `id`。你无需在设计中包含 ID 字段。如果需要在处理器中访问该 ID，可在负载中添加 `ID("request_id", String)`。
- SSE 与 HTTP 共享同一路由；当请求头包含 `Accept: text/event-stream` 时，内容协商选择 SSE。

## 3. 后续步骤

继续阅读[实现服务](./2-implementing.md)，生成代码并添加业务逻辑。

另见：[JSON‑RPC 概览](../../4-concepts/7-jsonrpc/1-overview)。


