---
title: "运行服务"
linkTitle: "运行"
weight: 3
description: "通过 HTTP 与 SSE 运行并调用 JSON‑RPC 服务。"
---

从模块根目录启动服务器：

```bash
go run ./cmd/server
```

### 通过 HTTP 调用

使用 curl 发送 JSON‑RPC 请求：

```bash
curl -s -X POST localhost:8080/rpc -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":"1","method":"add","params":{"a":2,"b":40}}'
```

### 通过 SSE（服务端流）调用

当方法使用混合结果（非流式 + 流式）时，请求 SSE：

```bash
curl -N -X POST localhost:8080/rpc -H 'Accept: text/event-stream' \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","id":"1","method":"add","params":{"a":2,"b":40}}'
```

### WebSocket（双向流式）

对于双向流式，使用 `GET` 暴露 JSON‑RPC 端点以允许 WebSocket 升级，并在设计中使用 `StreamingPayload`/`StreamingResult`。客户端为每个服务保持单一连接，并在其上交换 JSON‑RPC 消息。

### 批处理与通知

- 要发送通知，请在请求体中省略 `id`；服务器不会返回响应。
- 要进行批处理，请发送请求对象数组；响应为同序的结果数组（通知不产生条目）。

详情参见 JSON‑RPC 概念章节：

- [WebSocket 流式传输](../../4-concepts/7-jsonrpc/4-websocket-streaming)
- [批处理与通知](../../4-concepts/7-jsonrpc/5-batching-and-notifications)
- [错误映射](../../4-concepts/7-jsonrpc/6-error-mapping)


