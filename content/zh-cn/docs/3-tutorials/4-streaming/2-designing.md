---
title: "设计流式端点"
linkTitle: 设计
weight: 2
---

在 Goa 中设计流式端点，涉及定义能够处理一系列结果传输的方法。无论你要实现服务端流、客户端流还是双向流，Goa 的 DSL 都提供了清晰简洁的方式来指定这些行为。

## 使用 `StreamingResult` DSL

`StreamingResult` DSL 在方法定义中使用，用来表明该方法会向客户端流式发送一系列结果。它与 `Result` DSL 互斥；同一个方法中只能使用其中之一。

### 示例

```go
var _ = Service("logger", func() {
    Method("subscribe", func() {
        // LogEntry 实例会被流式发送到客户端。
        StreamingResult(LogEntry)
    })
})
```

在此示例中：

- **subscribe 方法：** 定义一个流式端点，发送 `LogEntry` 实例。
- **LogEntry：** 将要流式发送给客户端的结果类型。

定义流式方法时，需要指定将被流式传输的数据类型。通过将结果类型传递给 `StreamingResult` 函数来完成。

### 约束与注意事项

- **互斥性：** 方法要么使用 `Result`，要么使用 `StreamingResult`，不能同时使用。
- **单一结果类型：** 所有被流式发送的结果必须是同一类型的实例。
- **传输无关性：** 设计与传输协议无关，Goa 会生成合适的、面向具体传输的代码。

## 使用 `StreamingPayload` DSL

`StreamingPayload` DSL 在方法定义中使用，用来表明该方法会从客户端接收一系列消息。配合 HTTP 传输时，它与常规的 `Payload` DSL 一起工作，以同时处理初始连接参数和随后的流式数据。

### 示例

```go
var _ = Service("logger", func() {
    Method("subscribe", func() {
        // 客户端流式发送 LogEntry 实例
        StreamingPayload(LogEntry)

        // 处理完所有更新后返回单个结果
        Result(Summary)
    })
})
```

在此示例中：

- **subscribe 方法：** 定义一个端点，接收由客户端发送的 `LogEntry` 流。
- **LogEntry：** 将从客户端流式发送的载荷类型。
- **Summary：** 在处理完所有更新后返回的单个结果。

定义客户端流式方法时，需要指定将在流中接收的数据类型。通过将载荷类型传递给 `StreamingPayload` 函数来完成。

### 约束与注意事项

- **单一载荷类型：** 所有被流式发送的载荷必须是同一类型的实例。
- **传输无关性：** 设计与传输协议无关，Goa 会生成合适的、面向具体传输的代码。
- **可选结果：** 客户端流式方法可以返回一个单一结果，也可以不返回结果。
- **传输行为：** 在 HTTP 中，流在初始请求处理完成并将连接升级为 WebSocket 之后建立。

### 与常规 Payload 结合

对于 HTTP 端点，你通常希望在 WebSocket 升级之前，在初始请求中包含初始化参数。可以通过将 `StreamingPayload` 与常规的 `Payload` 结合来实现：

```go
var _ = Service("logger", func() {
    Method("subscribe", func() {
        // 初始连接参数
        Payload(func() {
            Field(1, "topic", String, "要订阅的日志主题")
            Field(2, "api_key", String, "用于认证的 API 密钥")
            Required("topic", "api_key")
        })

        // 客户端发送的更新流
        StreamingPayload(LogEntry)

        // 处理所有更新后的最终结果
        Result(Summary)

        HTTP(func() {
            GET("/logs/{topic}/stream")
            Param("api_key")
        })
    })
})
```

这种模式特别适用于：
- 在建立流之前进行认证与授权
- 为流式会话提供上下文或作用域
- 设置在整个流期间保持不变的初始参数
- 在升级为 WebSocket 连接前验证请求

在 HTTP 传输中：
1. 初始 GET 请求包含 `topic` 和 `api_key` 参数
2. 验证通过后，连接升级为 WebSocket
3. 随后客户端开始流式发送 `StreamingPayload` 消息
4. 服务器处理该流，并可能返回最终结果

## 总结

使用 `StreamingResult` 和 `StreamingPayload` DSL，Goa 中的流式端点设计十分直观。通过在服务方法中定义要流式传输的数据类型，你可以利用 Goa 强大的代码生成能力来处理底层、特定传输的流式逻辑，从而确保你的流式端点健壮、高效且易于维护。
