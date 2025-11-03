---
title: "HTTP 与 SSE 语义"
weight: 3
---

HTTP 与 SSE 共享相同的 JSON‑RPC 路由（通常为 `POST /rpc`）。客户端通过发送 `Accept: text/event-stream` 选择 SSE。SSE 仅适用于声明了流式结果（或混合结果）的方法。

## HTTP（非流式）

- 使用标准的 JSON‑RPC 请求/响应主体结构。
- 支持批处理。
- 支持通知。
- 支持 JSON 与 CBOR 编码。
- 支持 gzip。

典型 curl 示例：

```bash
curl -s -X POST localhost:8080/rpc -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":"1","method":"method","params":{}}'
```

## SSE（服务器流式）

- 使用 SSE 线格式（`data:`、`id:`、`event:` 字段）。
- 每个 JSON‑RPC 通知或响应对应一个独立的 SSE 事件。
- 使用 `Send(ctx, event)` 发送通知；使用 `SendAndClose(ctx, result)` 发送最终 JSON‑RPC 响应并结束流。
- 当存在时，SSE 的 `id:` 字段与 JSON‑RPC 响应的 `id` 一致。
- SSE 不支持批处理；一次发送一条消息。
- 仅支持 JSON 编码（SSE 不支持 CBOR）。
- 不支持 gzip。

使用 curl 请求 SSE：

```bash
curl -N -X POST localhost:8080/rpc -H 'Accept: text/event-stream' \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":"1","method":"stream","params":{}}'
```

## 混合结果

- 同一端点同时定义 `Result` 与 `StreamingResult`，以支持在 HTTP 与 SSE 之间进行内容协商。
- 服务器根据 `Accept` 选择使用 HTTP 或 SSE。

设计草图：

```go
var _ = Service("summaries", func() {
  Method("report", func() {
    Payload(func() { /* ... */ })
    Result(func() { /* HTTP 的非流式形状 */ })
    StreamingResult(func() { /* SSE 的流式形状 */ })

    JSONRPC(func() { ServerSentEvents(func() {}) })
})
```

操作提示：

- 在SSE响应上设置`Cache-Control: no-store`，以避免中间缓存。
- 如果客户端或代理会超时空闲连接，请发送周期性的保持连接（例如，空的评论）。
- 限制SSE事件的大小和频率，以避免客户端缓冲区问题。
