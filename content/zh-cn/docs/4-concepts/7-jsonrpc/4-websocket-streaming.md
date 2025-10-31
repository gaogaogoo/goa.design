---
title: "WebSocket 流式传输"
weight: 4
---

JSON‑RPC 通过 WebSocket 在每个服务上使用单一连接，所有方法共享。JSON‑RPC WebSocket 不支持非流式方法。

批处理不支持在 WebSocket 上；每个帧仅发送一条 JSON‑RPC 消息。

## 模式

- **仅 StreamingPayload：**
  在该模式下，方法只定义 `StreamingPayload`，未指定 `StreamingResult`。这使客户端能够向服务器持续发送消息，通常作为通知使用。由于没有结果流，服务器不会对这些消息发送响应。适用于客户端无需得到回复的场景，例如遥测上传或“发后即忘”的命令。

- **仅 StreamingResult：**
  在该模式下，方法只定义 `StreamingResult`，未指定 `StreamingPayload`。客户端发起流（通常请求为空或最小化），服务器向客户端发送消息流。这些属于服务器到客户端的通知，通常不使用请求 `id`，因为客户端并不期望对某个具体请求的直接响应。适用于服务器推送更新、事件订阅或实时数据流等场景。

- **同时使用 StreamingPayload 与 StreamingResult：**
  当同时定义 `StreamingPayload` 与 `StreamingResult` 时，该方法支持双向流式传输：客户端向服务器发送消息流，服务器也向客户端发送消息流。为实现可靠的消息关联，建议在两种类型中都包含一个可映射的 `ID()` 字段/属性，在双向或复用场景中尤为重要。

## 指南

- **为 WebSocket 连接使用 JSON‑RPC 端点上的 `GET`：**
  建立 JSON‑RPC 的 WebSocket 连接时，始终在指定的 JSON‑RPC 端点（例如 `GET /rpc`）上使用 HTTP `GET` 方法。这符合 WebSocket 协议：它会将 HTTP GET 请求升级为 WebSocket 连接。避免使用 `POST` 或其他 HTTP 方法进行升级。

- **在流式负载与结果类型中都包含 `ID()` 以进行消息关联：**
  为了让客户端与服务器之间的消息能够可靠关联，请在 `StreamingPayload` 与 `StreamingResult` 类型中定义 `ID()` 字段或方法。这样每条双向消息都可携带唯一标识，从而在类型层面匹配请求与其对应的响应或通知。在双向或复用场景中，这一点尤其重要，因为可能同时有多条消息在传输。

- **不要在同一服务中混用 JSON‑RPC WebSocket 与纯 HTTP WebSocket 端点：**
  将实现 JSON‑RPC 的 WebSocket 端点与自定义或非 JSON‑RPC 的 WebSocket 端点分开。混用会导致协议混淆、维护困难与潜在冲突。每个 WebSocket 端点应有明确协议——要么 JSON‑RPC，要么自定义，但不能两者兼具。

## HandleStream

在服务器端，Goa 为 WebSocket 流暴露了类型安全的辅助方法：

- `Send(notification)`：服务器发起的消息（无 `id`）。
- `SendAndClose(result)`：发送最终响应并关闭流（带 `id`）。
- `Recv(payload)`：接收客户端消息（带 `id`）。
- `SendAndWait(result)`：对客户端消息进行响应（使用同一 `id` 进行关联）。

ID 关联规则：
- 对于客户端请求，框架将请求的 `id` 复制到响应结果的 `id`（如果结果未显式设置 `id`）。
- 对于服务器发起的通知，不使用 `id`。

无效消息会产生错误响应；框架会为无法解析或不符合协议的消息返回适当的 JSON‑RPC 错误。

示例服务器处理器草图：

```go
func (s *service) HandleStream(ctx context.Context, stream *calc.Stream) error {
    // 收到的消息会被分发到生成的方法处理器。
    for {
        msg, err := stream.Recv()
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }

        switch msg.Method {
        case "echo":
            // 回显字段
            _ = stream.SendAndWait(&calc.EchoResult{/* 回显字段 */})
        default:
            // 未知方法可返回错误或忽略
        }
    }
    return nil
}
```