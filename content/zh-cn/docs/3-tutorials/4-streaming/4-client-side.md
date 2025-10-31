---
title: "实现客户端流式传输"
linkTitle: 客户端
weight: 4
---

一旦你使用 Goa 的 `StreamingPayload` DSL 设计好了客户端流式端点，下一步就是同时实现客户端处理数据流的逻辑，以及在服务端处理该流的代码。本文将引导你在 Goa 中实现一个流式端点的客户端与服务端两端。

## 客户端实现

当你在 DSL 中定义了客户端流式方法后，Goa 会为客户端生成特定的流接口以供实现。这些接口用于向服务器发送流式数据。

### 客户端流接口

假设有如下设计：

```go
var _ = Service("logger", func() {
    Method("upload", func() {
        StreamingPayload(LogEntry)
        HTTP(func() {
            GET("/logs/upload")
            Response(StatusOK)
        })
        GRPC(func() {})
    })
})
```

客户端流接口包含用于发送数据和关闭流的方法：

```go
// 客户端必须满足的接口
type UploadClientStream interface {
    // Send 会将 "LogEntry" 的实例以流式发送
    Send(*LogEntry) error
    // Close 关闭流
    Close() error
}
```

### 关键方法

- **Send：** 将指定类型（`LogEntry`）的实例发送给服务器。可多次调用以流式传输多个载荷。
- **Close：** 关闭流；调用 `Close` 后再调用 `Send` 将返回错误。

### 示例实现

下面是一个客户端流式端点的示例实现：

```go
func uploadLogEntries(client *logger.Client, logEntries []*LogEntry) error {
    stream, err := client.Upload(context.Background())
    if err != nil {
        return fmt.Errorf("failed to start upload stream: %w", err)
    }

    for _, logEntry := range logEntries {
        if err := stream.Send(logEntry); err != nil {
            return fmt.Errorf("failed to send log entry: %w", err)
        }
    }

    if err := stream.Close(); err != nil {
        return fmt.Errorf("failed to close stream: %w", err)
    }

    return nil
}
```

### 错误处理

恰当的错误处理可确保健壮的流式行为：

- 始终检查 `Send` 的返回值，以处理可能的传输错误
- 当服务器断开连接或上下文被取消时，`Send` 方法会返回错误
- 确保对错误进行适当的上下文包装，便于调试
- 在合适的场景下考虑为瞬时故障实现重试逻辑

## 服务端实现

服务端实现涉及接收并处理流式数据。Goa 会生成便于处理入站流的服务端接口。

### 服务端流接口

生成的服务端接口包含用于接收数据和管理流的方法：

```go
// 服务端用于接收流的接口
type UploadServerStream interface {
    // Recv 返回流中的下一个载荷
    Recv() (*LogEntry, error)
    // Close 关闭流
    Close() error
}
```

### 服务端示例实现

以下展示了如何在服务端侧处理一个流：

```go
func (s *loggerSvc) Upload(ctx context.Context, stream logger.UploadServerStream) error {
    for {
        logEntry, err := stream.Recv()
        if err == io.EOF {
            // 流已结束
            return nil
        }
        if err != nil {
            return fmt.Errorf("error receiving log entry: %w", err)
        }

        // 处理收到的日志条目
        if err := s.processLogEntry(logEntry); err != nil {
            return fmt.Errorf("error processing log entry: %w", err)
        }
    }
}
```

### 服务端关键注意事项

1. **流处理：**
   - 使用循环持续接收数据，直到 EOF 或错误
   - 将 `io.EOF` 视为正常的流结束条件
   - 按到达的顺序处理入站数据

2. **资源管理：**
   - 考虑为入站数据实现速率限制
   - 在处理大流时监控内存使用
   - 实施恰当的错误处理与日志记录

3. **错误处理：**
   - 对校验失败返回合适的错误
   - 恰当处理上下文取消
   - 视情况考虑实现部分成功的响应

## 总结

在 Goa 中实现客户端流式传输，既包括客户端的数据流式发送，也包括服务端对流的处理。遵循上述模式与错误处理、资源管理的最佳实践，你可以构建健壮的流式端点，从而提升 API 的效率。

客户端实现专注于高效发送数据与处理错误，服务端实现则提供了接收和处理流式数据的简洁接口。两者结合起来，为在 Goa 服务中处理上传或实时数据摄取提供了强大的机制。