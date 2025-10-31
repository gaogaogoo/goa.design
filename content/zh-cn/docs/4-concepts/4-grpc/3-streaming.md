---
title: "流式通信"
linkTitle: "流式通信"
weight: 3
description: "学习如何在 Goa 中实现 gRPC 流式服务，包括服务端流、客户端流与双向流等模式"
---

Goa 对 gRPC 流式通信提供了全面支持，可用于构建能够实时处理持续数据传输的服务。本文介绍 gRPC 中可用的不同流式模式，以及如何使用 Goa 进行实现。

{{< alert title="另见" color="info" >}}
有关跨传输规则以及每个传输支持的有效流式模式，请参阅
[Transports](../../6-transports)。
{{< /alert >}}

## 流式模式

gRPC 支持三种流式模式：

### 服务端流（Server-Side Streaming）

在服务端流中，客户端发送单个请求并接收一个响应流。该模式适用于：
- 实时数据馈送
- 进度更新
- 系统监控

如下定义一个服务端流方法：

```go
var _ = Service("monitor", func() {
    Method("watch", func() {
        Description("流式传输系统指标")
        
        Payload(func() {
            Field(1, "interval", Int, "采样间隔（秒）")
            Required("interval")
        })
        
        StreamingResult(func() {
            Field(1, "cpu", Float32, "CPU 使用率（百分比）")
            Field(2, "memory", Float32, "内存使用率（百分比）")
            Required("cpu", "memory")
        })
        
        GRPC(func() {
            Response(CodeOK)
        })
    })
})
```

### 客户端流（Client-Side Streaming）

客户端流允许客户端发送一个请求流并接收单个响应。常用于：
- 文件上传
- 批处理
- 数据聚合

示例定义：

```go
var _ = Service("analytics", func() {
    Method("process", func() {
        Description("处理分析事件流")
        
        StreamingPayload(func() {
            Field(1, "event_type", String, "事件类型")
            Field(2, "timestamp", String, "事件时间戳")
            Field(3, "data", Bytes, "事件数据")
            Required("event_type", "timestamp", "data")
        })
        
        Result(func() {
            Field(1, "processed_count", Int64, "已处理事件数量")
            Required("processed_count")
        })
        
        GRPC(func() {
            Response(CodeOK)
        })
    })
})
```

### 双向流（Bidirectional Streaming）

双向流允许客户端与服务端同时发送消息流。该模式非常适合：
- 实时聊天应用
- 游戏
- 交互式数据处理

示例定义：

```go
var _ = Service("chat", func() {
    Method("connect", func() {
        Description("建立双向聊天连接")
        
        StreamingPayload(func() {
            Field(1, "message", String, "聊天消息")
            Field(2, "user_id", String, "用户标识")
            Required("message", "user_id")
        })
        
        StreamingResult(func() {
            Field(1, "message", String, "聊天消息")
            Field(2, "user_id", String, "用户标识")
            Field(3, "timestamp", String, "消息时间戳")
            Required("message", "user_id", "timestamp")
        })
        
        GRPC(func() {
            Response(CodeOK)
        })
    })
})
```

## 实现

在 Goa 中实现 gRPC 流式通信涉及服务端与客户端代码。Goa 会基于你的服务定义生成所需的接口与类型，你需要实现这些接口以处理流式逻辑。

### 服务端实现

服务端需要实现处理流式通信的方法。不同的流式模式对数据流的处理方式不同。下面分别说明：

#### 服务端流示例：系统监控

该示例实现一个服务，按固定间隔向客户端流式发送系统指标（CPU 与内存使用率）。服务端保持连接打开并持续向客户端发送数据。

Goa 提供的 `monitor.WatchServerStream` 接口包含两项主要能力：
1. `Send(*WatchResult) error`：向客户端发送单个结果
2. 通过 `Context() context.Context` 访问上下文

使用方式如下：

```go
// 服务端流示例
func (s *monitorService) Watch(ctx context.Context, p *monitor.WatchPayload, stream monitor.WatchServerStream) error {
    // 根据客户端提供的间隔创建定时器
    ticker := time.NewTicker(time.Duration(p.Interval) * time.Second)
    // 确保在结束时清理定时器
    defer ticker.Stop()

    // 无限循环以持续发送指标
    for {
        select {
        // 使用上下文检查客户端是否取消请求
        case <-ctx.Done():
            return ctx.Err()
        // 等待下一次触发
        case <-ticker.C:
            // 获取当前系统指标（实现省略）
            metrics := getSystemMetrics()
            // 使用流的 Send 方法向客户端发送指标
            // 每次调用 Send 都会在流中传输一条消息
            if err := stream.Send(&monitor.WatchResult{
                CPU:    metrics.CPU,
                Memory: metrics.Memory,
            }); err != nil {
                return err
            }
        }
    }
}
```

#### 客户端流示例：分析处理

该示例演示如何处理来自客户端的事件流。`analytics.ProcessServerStream` 接口提供三个关键方法：
1. `Recv() (*ProcessPayload, error)`：接收来自客户端的下一条消息
2. `SendAndClose(*ProcessResult) error`：发送最终响应并关闭流
3. 通过 `Context() context.Context` 访问上下文

使用方式如下：

```go
// 客户端流示例
func (s *analyticsService) Process(ctx context.Context, stream analytics.ProcessServerStream) error {
    // 记录已处理事件数量
    var count int64
    
    // 持续从流中读取事件，直到其关闭
    for {
        // 使用 Recv() 获取下一条消息
        // Recv 会阻塞直到收到消息或流关闭
        event, err := stream.Recv()
        if err == io.EOF {
            // 客户端已发送完数据
            // 使用 SendAndClose 发送最终结果并关闭流
            // 客户端流仅能发送一个响应
            return stream.SendAndClose(&analytics.ProcessResult{
                ProcessedCount: count,
            })
        }
        if err != nil {
            return err
        }
        
        // 处理收到的事件（实现省略）
        if err := processEvent(event); err != nil {
            return err
        }
        count++
    }
}
```

#### 双向流示例：聊天服务

该示例演示一个聊天服务，双方可在任意时刻发送消息。`chat.ConnectServerStream` 接口同时具备两类流式能力：
1. `Recv() (*ConnectPayload, error)`：接收来自客户端的消息
2. `Send(*ConnectResult) error`：向客户端发送消息
3. 通过 `Context() context.Context` 访问上下文

使用方式如下：

```go
// 双向流示例
func (s *chatService) Connect(ctx context.Context, stream chat.ConnectServerStream) error {
    // 持续处理消息直到客户端断开连接
    for {
        // 使用 Recv() 等待并接收下一条客户端消息
        // 此调用会阻塞直到消息到达或客户端关闭流
        msg, err := stream.Recv()
        if err == io.EOF {
            // 客户端已关闭其发送流
            return nil
        }
        if err != nil {
            return err
        }

        // 构造带当前时间戳的响应
        response := &chat.ConnectResult{
            Message:   msg.Message,
            UserID:    msg.UserID,
            Timestamp: time.Now().Format(time.RFC3339),
        }
        
        // 使用 Send() 向客户端发送消息
        // 在双向流中，可任意顺序发送与接收
        if err := stream.Send(response); err != nil {
            return err
        }
    }
}
```

### 客户端实现

客户端侧的接口与服务端接口相互镜像，但从客户端视角出发。分别如下：

#### 服务端流客户端：监控指标

客户端获得的 `monitor.WatchClient` 接口提供：
1. `Recv() (*WatchResult, error)`：接收下一次指标更新
2. `Close() error`：关闭流

```go
// 服务端流客户端
func watchMetrics(ctx context.Context, client *monitor.Client) error {
    // 使用初始参数建立流式连接
    // 返回用于接收指标的流接口
    stream, err := client.Watch(ctx, &monitor.WatchPayload{
        Interval: 5, // 每 5 秒请求一次指标
    })
    if err != nil {
        return err
    }

    // 持续接收指标直到流结束
    for {
        // 使用 Recv() 获取下一次指标更新
        // 该调用会阻塞直到新指标到达或服务端关闭流
        metrics, err := stream.Recv()
        if err == io.EOF {
            // 服务端已关闭流
            break
        }
        if err != nil {
            return err
        }
        // 处理收到的指标（此处仅记录日志）
        log.Printf("CPU: %.2f%%, Memory: %.2f%%", metrics.CPU, metrics.Memory)
    }
    return nil
}
```

#### 客户端流客户端：上传事件

客户端获得的 `analytics.ProcessClient` 接口提供：
1. `Send(*ProcessPayload) error`：向服务端发送事件
2. `CloseAndRecv() (*ProcessResult, error)`：关闭发送流并等待最终响应

```go
// 客户端流客户端
func uploadEvents(ctx context.Context, client *analytics.Client, events []*analytics.Event) error {
    // 初始化流式连接
    // 返回用于发送事件的流接口
    stream, err := client.Process(ctx)
    if err != nil {
        return err
    }

    // 通过流的 Send 方法逐个发送事件到服务端
    for _, event := range events {
        if err := stream.Send(event); err != nil {
            return err
        }
    }

    // 使用 CloseAndRecv 关闭发送流并获取服务端响应
    // 此调用会阻塞直到服务端处理完成并返回结果
    result, err := stream.CloseAndRecv()
    if err != nil {
        return err
    }
    log.Printf("Processed %d events", result.ProcessedCount)
    return nil
}
```

#### 双向流客户端：聊天客户端

客户端获得的 `chat.ConnectClient` 接口同时具备以下能力：
1. `Send(*ConnectPayload) error`：向服务端发送消息
2. `Recv() (*ConnectResult, error)`：接收来自服务端的消息
3. `CloseSend() error`：关闭发送流

```go
// 双向流客户端
func startChat(ctx context.Context, client *chat.Client, userID string) error {
    // 初始化双向流
    // 返回同时用于发送与接收的流接口
    stream, err := client.Connect(ctx)
    if err != nil {
        return err
    }

    // 启动独立的 goroutine 负责发送消息
    // 展示并发发送与接收
    go func() {
        for {
            // 使用 Send 发送消息到服务端
            if err := stream.Send(&chat.ConnectPayload{
                Message: "Hello",
                UserID:  userID,
            }); err != nil {
                log.Printf("Send error: %v", err)
                return
            }
            time.Sleep(time.Second)
        }
    }()

    // 主 goroutine 负责接收消息
    for {
        // 使用 Recv 接收服务端的下一条消息
        msg, err := stream.Recv()
        if err == io.EOF {
            // 服务端已关闭流
            break
        }
        if err != nil {
            return err
        }
        // 处理收到的消息
        log.Printf("Received: %s from %s at %s",
            msg.Message, msg.UserID, msg.Timestamp)
    }
    return nil
}
```

## 错误处理

在实现流式端点时，正确的错误处理至关重要：

1. 上下文取消（Context Cancellation）：始终检查上下文取消以优雅处理客户端断开。
2. EOF 处理：正确处理 io.EOF 以识别流结束。
3. 资源清理：使用 defer 确保资源得到正确清理。
4. 部分失败：为瞬时错误考虑实现重试逻辑。

## 最佳实践

1. 消息大小：保持消息大小合理以避免内存压力。
2. 流量控制：实现恰当的流控，避免任一端被过载。
3. 超时：为流式操作设置合适的超时。
4. 监控：添加指标以跟踪流式性能与错误。
5. 文档：清晰记录流式行为与错误条件。