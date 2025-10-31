---
title: "实现服务"
linkTitle: "实现"
weight: 2
description: "在 Goa 中生成代码并实现一个基础的 JSON‑RPC 服务。"
---

在本步骤中，你将生成代码并实现 `calculator` 服务。

## 1. 生成代码

```bash
go install goa.design/goa/v3/cmd/goa@latest
goa gen jsonrpccalc/design
```

上述命令会生成基于 HTTP 的 JSON‑RPC 的服务端与客户端代码（若方法使用流式结果也会生成 SSE 相关代码）。关于 WebSocket 流式传输，请参阅 JSON‑RPC 概念章节。

## 2. 实现服务逻辑

创建 `cmd/server/main.go`，并在生成的 `service` 包中实现 `Add` 方法。实现方式与 REST/gRPC 教程一致。

最小化的服务器接线示例：

```go
// cmd/server/main.go
package main

// 导入为你的服务生成的包以及生成的 JSON‑RPC HTTP 服务器包。
// 具体导入路径取决于你的模块名与服务名。

type calcSvc struct{}

// 实现生成的接口方法。
// func (calcSvc) Add(ctx context.Context, p *calc.AddPayload) (*calc.AddResult, error) {
//     return &calc.AddResult{Sum: p.A + p.B}, nil
// }

func main() {
    // 1) 创建服务实现
    // 2) 使用生成的助手构建端点
    // 3) 将生成的 JSON‑RPC HTTP 服务器挂载到你的 mux 上
    // 4) 在 :8080 启动 HTTP 服务器
}
```

关于 ID 与流式传输的说明（权威）：

- 非流式：即使未声明 `ID()` 字段，框架仍会使用请求包的 `id` 进行响应关联。仅当你的处理器需要读取该 `id` 值时，才在负载中添加 `ID("request_id", String)`。
- SSE：若设置了结果 ID，`SendAndClose` 会将响应包的 `id` 设为结果 ID；否则使用原始请求的 `id`。
- WebSocket：双向流式传输每个服务使用一个连接；当需要类型化关联时，使用 `StreamingPayload`/`StreamingResult`，并在两者中都包含 `ID()`。

## 3. 运行服务器

继续阅读[运行](./3-running.md)，以启动服务器并通过 HTTP 与 SSE 进行调用。



