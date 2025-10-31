---
title: "JSON‑RPC 概览"
weight: 1
---

Goa 在多种传输上支持 JSON‑RPC 2.0：

- HTTP（非流式的一元请求与响应）
- HTTP 服务器发送事件（SSE），用于服务器发起的流式传输
- WebSocket，用于双向流式传输

关键特性：

- 在适用的场景下支持批量请求与通知。
- 同一个 JSON‑RPC HTTP 路由可通过 `Accept` 头进行内容协商来提供 SSE（`application/json` 与 `text/event-stream`）。
- JSON‑RPC WebSocket 每个服务使用单一连接，由所有方法共享。

相关概念：参见[传输](../6-transports)，了解单个服务及每个方法的允许组合矩阵。


### 本节主题

- [ID 与封装（Envelope）映射](./2-ids-and-envelope)
- [HTTP 与 SSE 语义](./3-http-and-sse)
- [WebSocket 流式传输](./4-websocket-streaming)
- [批处理与通知](./5-batching-and-notifications)
- [错误映射](./6-error-mapping)

如果你偏好按步骤构建，请从教程开始：[基础 JSON‑RPC 服务](../../3-tutorials/3-jsonrpc-service/)。

### 方法名

在线路层，JSON‑RPC 的 `method` 值就是 DSL 的方法名（例如：`add`）。每个服务只有一个 JSON‑RPC 端点，因此方法名作用域为该服务，不需要 `service.method` 前缀。

### 传输摘要

| 传输      | 连接                   | 模式                              | 批处理               | 通知          |
|-----------|-----------------------|----------------------------------|---------------------|---------------|
| HTTP      | 请求/响应              | 非流式（一元）                    | 支持（数组请求体）    | 客户端 → 支持 |
| SSE       | 长连接 HTTP 响应       | 服务器流式、混合结果              | 不适用               | 服务器 → 支持 |
| WebSocket | 持久（全双工）         | 客户端流式、服务器流式、双向      | 非典型（消息导向）    | 双向          |

注意：

- SSE 与 HTTP 共享相同路由。客户端通过 `Accept: text/event-stream` 选择 SSE，且方法必须声明流式结果（或混合结果）。
- 在 JSON‑RPC 端点上使用 `GET` 即启用 WebSocket。JSON‑RPC WebSocket 不支持非流式方法。

