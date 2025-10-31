---
title: "实现服务端流式传输"
linkTitle: 服务端
weight: 3
---

一旦你使用 Goa 的 `StreamingResult` DSL 设计好了服务端流式端点，下一步就是同时实现服务端处理结果流的逻辑，以及消费该流的客户端代码。本文将引导你在 Goa 中实现一个流式端点的服务端与客户端两端。

## 服务端实现

当你在 DSL 中定义了服务端流式方法后，Goa 会为服务端生成特定的流接口以供实现。这些接口用于向客户端发送流式数据。

### 服务端流接口

假设有如下设计：
```go
var _ = Service("logger", func() {
    Method("subscribe", func() {
        StreamingResult(LogEntry)
        HTTP(func() {
            GET("/logs/stream")
            Response(StatusOK)
        })
    })
})
```

服务端流接口包含用于发送数据和关闭流的方法：

```go
// 服务端必须满足的接口
type ListServerStream interface {
    // Send 会将 "StoredBottle" 的实例以流式发送
    Send(*LogEntry) error
    // 关闭流
    Close() error
}
```

### 关键方法

- **Send：** 将指定类型（`LogEntry`）的实例发送给客户端。可多次调用以流式传输多个结果。
- **Close：** 关闭流，表示数据传输结束。调用 `Close` 后，任何后续的 `Send` 调用都会返回错误。

### 示例实现

下面是一个服务端流式端点的示例实现：

```go
// Lists 会将日志条目以流式发送给客户端
func (s *loggerSvc) Subscribe(ctx context.Context, stream logger.SubscribeServerStream) error {
    logEntries, err := loadLogEntries()
    if err != nil {
        return fmt.Errorf("failed to load log entries: %w", err)
    }

    for _, logEntry := range logEntries {
        if err := stream.Send(logEntry); err != nil {
            return fmt.Errorf("failed to send log entry: %w", err)
        }
    }

    return stream.Close()
}
```

### 错误处理

恰当的错误处理可确保健壮的流式行为：

- 始终检查 `Send` 的返回值，以处理可能的传输错误
- 当客户端断开连接或上下文被取消时，`Send` 方法会返回错误
- 确保对错误进行适当的上下文包装，便于调试
- 在合适的场景下考虑为瞬时故障实现重试逻辑

## 客户端实现

客户端实现涉及接收并处理流式数据。Goa 会生成便于消费流的客户端接口。

### 客户端流接口

生成的客户端接口包含用于接收数据和管理流的方法：

```go
// 客户端用于接收流的接口
type ListClientStream interface {
    // Recv 返回流中的下一个结果
    Recv() (*LogEntry, error)
    // Close 关闭流
    Close() error
}
```

### 客户端示例实现

以下展示了如何在客户端侧消费一个流：

```go
func processLogEntryStream(client logger.Client) error {
    stream, err := client.List(context.Background())
    if err != nil {
        return fmt.Errorf("failed to start stream: %w", err)
    }
    defer stream.Close()

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
        processLogEntry(logEntry)
    }
}
```

### 客户端关键注意事项

1. **流初始化：**
   - 使用生成的客户端方法创建流
   - 在继续之前检查初始化错误
   - 使用 `defer stream.Close()` 确保清理到位

2. **接收数据：**
   - 使用循环持续接收数据，直到 EOF 或错误
   - 将 `io.EOF` 视为正常的流结束条件
   - 根据应用需求恰当处理其它错误

3. **资源管理：**
   - 使用完毕务必关闭流
   - 根据需要考虑通过上下文设置超时或截止时间
   - 实施恰当的错误处理与日志记录

## 总结

在 Goa 中实现流式传输，既包括服务端的数据流式发送，也包括客户端对流的消费。遵循上述模式与错误处理、资源管理的最佳实践，你可以构建健壮的流式端点，从而提升 API 的响应性与可扩展性。

服务端实现专注于高效发送数据与处理错误，客户端实现则提供了接收和处理流式数据的简洁接口。两者结合起来，为在 Goa 服务中处理实时或海量数据提供了强大的机制。