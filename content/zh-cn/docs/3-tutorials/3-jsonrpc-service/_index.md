---
title: "基础 JSON‑RPC 服务"
linkTitle: "基础 JSON‑RPC 服务"
weight: 3
description: "在 Goa 中设计并实现一个简单的 JSON‑RPC 2.0 服务。学习 DSL、ID 映射、传输方式（HTTP、SSE、WebSocket）、通知与批处理。"
---

使用 Goa 的设计优先工作流构建一个小而完整的 JSON‑RPC 2.0 服务。您将使用 DSL 定义 API，生成类型安全的代码，实现业务逻辑，并通过 HTTP、Server‑Sent Events（SSE）与 WebSocket 运行服务。

### 你将学到

- 使用 Goa DSL 进行 JSON‑RPC 服务设计
- JSON‑RPC 的 ID 如何映射到 Payload 与 Result
- 在 HTTP、SSE 与 WebSocket 间如何选择及适用场景
- 发送通知（notification）与批处理（batch）
- 使用 curl 与生成的客户端运行并调用服务

### 先决条件

- 安装 Go 1.21+
- 安装 `goa` 代码生成器（`go install goa.design/goa/v3/cmd/goa@latest`）
- 熟悉 Goa 的 REST/gRPC 教程更佳，但不是必需

### 你将构建

一个暴露 `Add` 方法的 `calculator` 服务，通过 JSON‑RPC 提供。可先用 HTTP 进行简单的请求/响应，随后逐步扩展到 SSE/WebSocket。

从第一步开始：[服务设计](./1-designing.md)。
