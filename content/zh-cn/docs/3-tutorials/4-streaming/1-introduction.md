---
title: "介绍"
linkTitle: Introduction
weight: 1
---

流式传输是一项强大的特性，使 API 能够高效处理海量数据与实时更新。在 Goa 中，流式支持允许你定义可发送或接收一系列结果的端点，从而提升服务的响应性与可扩展性。

## 为什么流式传输很重要

- 效率：流式传输通过在单个连接上持续传输数据，减少多次 HTTP 请求的开销。
- 实时数据：支持实时数据更新，这对直播、通知与数据监控类应用至关重要。
- 可扩展性：以分块方式处理大型数据集，而非一次性将全部数据加载到内存，更加优雅。
- 更佳用户体验：以增量方式传递数据，为用户提供更平滑、更灵敏的体验。

## Goa 的流式能力

处理大型文件或实时数据流时，先将完整负载读入内存再处理并不总是可行或理想的。Goa 提供多种处理流式数据的方式：

当你需要以下能力时使用 `StreamingPayload` 与 `StreamingResult`：

- 以已知类型传输结构化数据
- 利用 Goa 的类型系统与校验
- 使用 gRPC 流式传输

当你需要以下能力时使用 `SkipRequestBodyEncodeDecode` 与 `SkipResponseBodyEncodeDecode`：

- 传输原始二进制数据或未知内容类型
- 实现自定义流式协议
- 处理多媒体流

当你需要以下能力时使用 `ServerSentEvents`：

- 在 HTTP 上实现服务端到客户端的流式传输
- 支持原生 EventSource API 的浏览器客户端
- 处理实时通知与更新

Goa 在多种传输协议上支持单向与双向流式传输，包括 HTTP（通过 WebSocket 与 SSE）与 gRPC。借助 Goa 的领域特定语言（DSL），你可以定义与传输无关的流式端点，便于在服务架构中实现无缝集成与灵活性。

### 关键特性

- 单向流式：允许服务器或客户端单方发送数据流。
- 双向流式：允许服务器与客户端同时发送数据流。
- 传输无关：定义可跨多种传输协议工作的流式逻辑，无需修改。
- 生成的流接口：基于你的流式定义自动生成服务端与客户端的流接口。
- 自定义视图：支持流式结果的多视图，灵活呈现给客户端。
- 多种协议：支持 WebSocket、Server-Sent Events（SSE）与 gRPC 的流式传输。

## 示例概览

下面以管理日志条目的 `logger` 服务为例，演示多种真实场景下的流式用法。另见章节
[通过 HTTP 传输原始二进制数据](./7-raw-binary)，了解如何通过 HTTP 传输原始二进制数据。

### 服务端流式示例

```go
var _ = Service("logger", func() {
    // 服务器为特定主题推送日志条目
    Method("subscribe", func() {
        Description("Stream log entries for a specific topic")
        
        Payload(func() {
            Field(1, "topic", String, "Topic of the log to subscribe to")
            Required("topic")
        })
        
        // 服务器端流式日志条目
        StreamingResult(func() {
            Field(1, "timestamp", String, "Time of reading")
            Field(2, "message", String, "Log message")
            Required("timestamp", "message")
        })

        HTTP(func() {
            GET("/logs/{topic}/stream")
            Response(StatusOK)
        })
    })
})
```

### 双向流式示例

```go
var _ = Service("logger", func() {
    // 使用双向流式进行实时日志与主题管理
    Method("subscribe", func() {
        Description("Bidirectional stream for real-time log updates and topic management")
        
        // 客户端流式发送订阅更新
        StreamingPayload(func() {
            Field(1, "topic", String, "Topic of the log to subscribe to")
            Required("topic")
        })
        
        // 服务器流式发送状态与告警
        StreamingResult(func() {
            Field(1, "timestamp", String, "Time of reading")
            Field(2, "message", String, "Log message")
            Required("timestamp", "message")
        })

        HTTP(func() {
            GET("/logs/{topic}/stream")
            Response(StatusOK)
        })
    })
})
```

这些示例展示了：

#### 1. 服务端流式

这是一种单向流式模式，服务器针对一次客户端请求返回多次响应。

示例：日志监控
    客户端请求（一次）：“监控‘数据库错误’日志”
    
    服务器响应（持续）：
    10:00:01 - 数据库连接超时
    10:00:05 - 查询执行失败
    10:00:08 - 连接池耗尽
    （随着新日志出现持续发送）

关键特征：初始请求之后，数据仅从服务器流向客户端。

#### 2. 双向流式
一种允许双方在一段时间内发送多条消息的模式。

示例：交互式日志管理
    客户端：“开始监控 ‘database’ 日志”
    服务器：发送与 database 相关的日志
    客户端：“更新筛选器，包含 ‘network’”
    服务器：同时发送 database 与 network 日志
    客户端：“移除 ‘database’ 筛选器”
    服务器：仅发送 network 日志
    （双方持续通信）

关键特征：支持持续的往返通信，在连接生命周期内，客户端与服务器都可发送多条消息。

## 后续步骤

通过本章对 Goa 流式传输的介绍，你现在可以进一步深入到流式端点的设计、实现与管理。后续章节将引导你了解：

- [设计流式端点](./2-designing)
- [服务端流式](./3-server-side)
- [客户端流式](./4-client-side)
- [双向流式](./5-bidirectional)
- [处理多视图](./6-views)
- [通过 HTTP 传输原始二进制数据](./7-raw-binary)
- [Server-Sent Events](./8-sse)
