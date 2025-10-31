---
title: "传输与流式组合"
linkTitle: "传输"
weight: 6
description: "每个服务和每个方法允许的传输组合，以及流模式和约束。"
---

本页面解释了 Goa 服务及其方法可以公开哪些传输组合，以及每种传输的有效流模式。目标是为新手和有经验的用户提供一个单一、权威的参考，说明在 Goa 中“混合传输”意味着什么，以及哪些组合是允许或禁止的。

内容包括：
- 可用传输：HTTP（普通）、HTTP 服务器发送事件（SSE）、HTTP WebSocket、JSON-RPC 2.0（通过 HTTP、SSE、WebSocket）和 gRPC。
- 流模式：无流、客户端流、服务器流、双向。
- 单个服务可以同时公开什么。
- 单个方法可以在每个传输的基础上公开什么。


## 术语

- 无流：标准请求/响应（一元）。
- 客户端流：客户端向服务器发送有效负载流。
- 服务器流：服务器向客户端发送结果流。
- 双向：双方都进行流式传输。
- 混合结果：一个方法同时定义了具有不同类型的 `Result` 和 `StreamingResult`。这使得在常规响应和流式响应之间可以进行内容协商（仅受 SSE 支持）。


## 服务级别：一个服务中可以混合哪些传输？

在下表中，“是”表示传输可以在同一个 Goa 服务中共存。注释解释了约束。

| 同一服务中的传输 | 使用 HTTP（普通） | 使用 HTTP (WS) | 使用 HTTP (SSE) | 使用 JSON-RPC (HTTP) | 使用 JSON-RPC (WS) | 使用 JSON-RPC (SSE) | 使用 gRPC |
|---------------------------|-------------------|----------------|-----------------|----------------------|--------------------|---------------------|-----------|
| HTTP（普通）              | —                 | 是             | 是              | 是                   | 否 [S2]            | 是                  | 是        |
| HTTP (WebSocket)          | 是                | —              | 是              | 是                   | 否 [S2]            | 是                  | 是        |
| HTTP (SSE)                | 是                | 是             | —               | 是                   | 否 [S2]            | 是                  | 是        |
| JSON-RPC (HTTP)           | 是                | 是             | 是              | —                    | 否 [S1]            | 是                  | 是        |
| JSON-RPC (WebSocket)      | 否 [S2]           | 否 [S2]        | 否 [S2]         | 否 [S1]              | —                  | 否 [S1]             | 是        |
| JSON-RPC (SSE)            | 是                | 是             | 是              | 是                   | 否 [S1]            | —                   | 是        |
| gRPC                      | 是                | 是             | 是              | 是                   | 是                 | 是                  | —         |

注意：
- [S1] JSON-RPC WebSocket 不能与同一服务中的其他 JSON-RPC 传输混合。JSON-RPC 服务必须是纯 WebSocket，或者是 HTTP/SSE（可以共存）——不能两者兼有。
- [S2] 服务不能将 JSON-RPC WebSocket 端点与“纯 HTTP” WebSocket 端点混合。JSON-RPC 使用所有方法共享的单个 WS 连接，而纯 HTTP 为每个端点创建一个 WS 连接。

附加的服务级别行为：
- JSON-RPC HTTP 和 JSON-RPC SSE 可以共享同一个 POST 端点，并通过 `Accept` 标头进行选择（例如，`text/event-stream` vs. `application/json`）。
- gRPC 是独立的，可以与 HTTP 和 JSON-RPC 传输自由组合。


## 方法级别：每种传输的有效流模式

在下表中，“是”表示该流模式对于该传输上的方法是有效的。“是（混合）”表示当方法使用混合结果（不同的 `Result` 和 `StreamingResult`）并且端点启用 SSE 时，它是有效的。“否”表示禁止。

| 传输           | 无流           | 客户端流 | 服务器流       | 双向 |
|---------------------|---------------------|---------------|---------------------|---------------|
| HTTP（普通）        | 是                 | 否            | 是（混合）[M1]    | 否            |
| HTTP (SSE)          | 是（混合）[M2]    | 否            | 是 [M3]            | 否            |
| HTTP (WebSocket)    | 否                  | 是 [M4]      | 是 [M4]      | 是 [M4]      |
| JSON-RPC (HTTP)     | 是                 | 否            | 是（混合）[M1,M5] | 否            |
| JSON-RPC (SSE)      | 是（混合）[M2,M5] | 否            | 是 [M6]            | 否            |
| JSON-RPC (WebSocket)| 否                  | 是 [M7]      | 是 [M8]      | 是 [M7]      |
| gRPC                | 是                 | 是           | 是                 | 是           |

注意：
- [M1] 混合结果要求端点上有 SSE 并禁止 `StreamingPayload`。用于在常规响应和 SSE 流之间进行内容协商。
- [M2] 仅当方法具有混合结果时，将 SSE 与非流式方法一起使用才有效（SSE 路径提供流式结果；非 SSE 路径提供常规结果）。
- [M3] SSE 严格是服务器到客户端的；它不能用于客户端或双向流。
- [M4] 纯 HTTP WebSocket 端点必须使用 GET 并且不能包含请求正文。请改用标头/参数映射请求数据。（JSON-RPC WS 是一个例外，因为消息在 WS 通道内传输。）
- [M5] JSON-RPC HTTP 和 JSON-RPC SSE 可以与混合结果共存；服务器根据 `Accept` 标头进行选择。
- [M6] JSON-RPC SSE 在共享的 JSON-RPC 端点上使用 POST；SSE `id` 字段映射到结果 ID 属性。
- [M7] JSON-RPC WebSocket 支持客户端流、服务器流和双向流。JSON-RPC WebSocket 不支持非流式方法。
- [M8] 对于具有服务器流的 JSON-RPC WebSocket，在方法 `Payload` 中定义请求数据；不要同时定义 `StreamingPayload`。


## JSON-RPC 细节

- WebSocket:
  - 每个服务一个 WS 连接，由所有 JSON-RPC 方法共享。
  - WS 端点上没有标头/cookie/参数映射。
  - 支持三种方法模式：
    - 仅 `StreamingPayload()`（客户端到服务器通知）。
    - 仅 `StreamingResult()`（服务器到客户端通知；发出时没有请求 `id`）。
    - `StreamingPayload()` 和 `StreamingResult()`（双向）。
  - JSON-RPC WebSocket 不支持非流式方法。

- HTTP and SSE:
  - 两者都使用相同的 JSON-RPC 路由（POST）。服务器在运行时根据 `Accept` 选择响应行为。
  - 混合结果支持在常规 HTTP JSON-RPC 响应和 SSE 事件流之间进行内容协商。

- ID 处理:
  - 非流式：如果用户代码未设置，框架会将请求 `id` 复制到结果 `ID` 字段。
  - 流式 (WS)：服务器回复重用原始请求 `id`；服务器发起的通知没有 `id`。
  - SSE：`SendAndClose` 发送一个 JSON-RPC 响应；如果设置了 `id`，则 `id` 等于结果 ID，否则等于请求 `id`。


## HTTP 细节

- WebSocket 端点必须使用 GET。SSE 端点可以使用 GET 或 POST。JSON-RPC SSE 使用 POST。
- WebSocket 端点（纯 HTTP）不能有请求正文。通过标头和/或参数映射输入。（JSON-RPC WS 是豁免的，因为 JSON-RPC 消息在 WS 通道上传输。）


## gRPC 细节

相对于 Goa 中的 HTTP/JSON-RPC，gRPC 没有施加额外的限制。一元、客户端流、服务器流和双向流都受支持。gRPC 可以与上面列出的任何 HTTP 和 JSON-RPC 传输自由组合。


## 这些规则在哪里强制执行（供参考）

以下组件强制执行本文档中总结的约束：
- `expr/method.go`: 流类型助手；混合结果检测。
- `dsl/payload.go`, `dsl/result.go`: `StreamingPayload`/`StreamingResult` 如何设置方法流类型。
- `expr/http_endpoint.go`: SSE 约束；混合结果要求；纯 HTTP WS 方法/路由验证；JSON-RPC 端点验证。
- `expr/http_service.go`: JSON-RPC 传输混合规则；JSON-RPC WS 与纯 HTTP WS 冲突；JSON-RPC 路由准备和方法强制执行（WS 为 GET，否则为 POST）。
- `dsl/jsonrpc.go` and `jsonrpc/README.md`: JSON-RPC 传输行为（批处理、通知、WS/SSE 语义），包括“WS 需要流式传输”和“HTTP+SSE 内容协商”。


## 快速示例

将概念与 DSL 连接起来的最小示例（省略无关行）：

```go
// JSON-RPC SSE + HTTP (混合结果)
Method("monitor", func() {
    Result(ResultType)
    StreamingResult(EventType)
    JSONRPC(func() { ServerSentEvents() })
})
```

```go
// JSON-RPC WebSocket (双向)
Method("chat", func() {
    StreamingPayload(Message)
    StreamingResult(Message)
    JSONRPC(func() {})
})
```

```go
// 纯 HTTP SSE (服务器流)
Method("watch", func() {
    StreamingResult(Event)
    HTTP(func() { ServerSentEvents() })
})
```

使用这些示例作为模板，并应用上表来确保您的服务和方法选择有效的组合。


