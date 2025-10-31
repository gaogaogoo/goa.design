---
title: "实现双向流式传输"
linkTitle: 双向
weight: 5
---

一旦你使用 Goa 的 `StreamingPayload` 和 `StreamingResult` DSL 设计好了双向流式端点，下一步就是实现连接两端的逻辑。本文将引导你在 Goa 中实现一个双向流式端点的客户端与服务端组件。

## 设计

假设有如下设计：

```go
var _ = Service("logger", func() {
    Method("monitor", func() {
        StreamingPayload(LogFilter)
        StreamingResult(LogEntry)
        HTTP(func() {
            GET("/logs/monitor")
            Response(StatusOK)
        })
        GRPC(func() {})
    })
})
```

## 客户端实现

当你定义了一个双向流式方法后，Goa 会为客户端生成特定的流接口以供实现。这些接口同时支持发送与接收流式数据。

### 客户端流接口

客户端流接口包含用于同时发送和接收数据的方法：

```go
// 客户端必须满足的接口
type MonitorClientStream interface {
    // Send 会将 "LogFilter" 的实例以流式发送
    Send(*LogFilter) error
    // Recv 返回流中的下一个结果
    Recv() (*LogEntry, error)
    // Close 关闭流
    Close() error
}
```

### 关键方法

- **Send：** 将过滤器更新发送到服务器。可多次调用以更新过滤条件。
- **Recv：** 接收与当前过滤器匹配的服务器日志条目。
- **Close：** 关闭双向流。调用 Close 后，Send 与 Recv 均会返回错误。

### 示例实现

下面是一个实现客户端双向流式端点的示例：

```go
func monitorLogs(client logger.Client, initialFilter *LogFilter) error {
    stream, err := client.Monitor(context.Background())
    if err != nil {
        return fmt.Errorf("failed to start monitor stream: %w", err)
    }
    defer stream.Close()

    // 启动一个 goroutine 处理日志接收
    go func() {
        for {
            logEntry, err := stream.Recv()
            if err == io.EOF {
                return
            }
            if err != nil {
                log.Printf("error receiving log entry: %v", err)
                return
            }
            processLogEntry(logEntry)
        }
    }()

    // 发送初始过滤器
    if err := stream.Send(initialFilter); err != nil {
        return fmt.Errorf("failed to send initial filter: %w", err)
    }

    // 根据条件动态更新过滤器
    for {
        newFilter := waitForFilterUpdate()
        if err := stream.Send(newFilter); err != nil {
            return fmt.Errorf("failed to update filter: %w", err)
        }
    }
}
```

## 服务端实现

服务端实现需要同时处理入站的过滤器更新，并向客户端流式发送匹配的日志条目。

### 服务端流接口

```go
// 服务端必须满足的接口
type MonitorServerStream interface {
    // Send 会将 "LogEntry" 的实例以流式发送
    Send(*LogEntry) error
    // Recv 返回流中的下一个过滤器
    Recv() (*LogFilter, error)
    // Close 关闭流
    Close() error
}
```

### 服务端示例实现

以下展示了如何在服务端侧实现双向流式传输：

```go
func (s *loggerSvc) Monitor(ctx context.Context, stream logger.MonitorServerStream) error {
    // 启动一个 goroutine 处理过滤器更新
    filterCh := make(chan *LogFilter, 1)
    go func() {
        defer close(filterCh)
        for {
            filter, err := stream.Recv()
            if err == io.EOF {
                return
            }
            if err != nil {
                log.Printf("error receiving filter update: %v", err)
                return
            }
            filterCh <- filter
        }
    }()

    // 主循环：处理日志并应用过滤
    var currentFilter *LogFilter
    for {
        select {
        case filter, ok := <-filterCh:
            if !ok {
                // 通道关闭，停止处理
                return nil
            }
            currentFilter = filter
        case <-ctx.Done():
            // 上下文取消，停止处理
            return ctx.Err()
        default:
            if currentFilter != nil {
                logEntry := s.getNextMatchingLog(currentFilter)
                if err := stream.Send(logEntry); err != nil {
                    return fmt.Errorf("error sending log entry: %w", err)
                }
            }
        }
    }
}
```

### 关键注意事项

1. **并发操作：**
   - 使用 goroutine 独立处理发送与接收
   - 为共享状态实现恰当的同步
   - 双向均要处理优雅关闭

2. **资源管理：**
   - 监控入站与出站流的内存使用
   - 双向实现速率限制
   - 在任一侧关闭流时清理资源

3. **错误处理：**
   - 同时处理来自 Send 与 Recv 的错误
   - 将错误恰当地传播到双方
   - 对瞬时故障考虑实现重连逻辑

4. **上下文管理：**
   - 双向都要尊重上下文取消
   - 实现适当的超时
   - 上下文取消时清理资源

## 总结

在 Goa 中实现双向流式传输，需要在客户端与服务端两侧协调好发送与接收的操作。遵循并发操作、错误处理与资源管理的最佳实践，你可以构建健壮的双向流式端点，实现客户端与服务器之间实时、交互式的通信。

该实现允许客户端通过发送过滤器动态更新流式行为，同时保持服务器端的持续响应，从而为 Goa 服务中的实时数据交换提供灵活而强大的机制。