---
title: "服务器发送事件（Server-Sent Events）"
linkTitle: "SSE"
weight: 8
description: >
  学习如何在 Goa 服务中实现服务器发送事件（SSE）端点。
---

服务器发送事件（SSE）是一种基于 HTTP 的服务器到客户端的流式协议，可实现从服务器到客户端的实时更新。Goa 原生支持实现 SSE 端点，使你可以轻松为服务添加实时流能力。

## 概览

SSE 特别适用于需要从服务器主动推送更新到客户端的场景。可以把它看作单向的“电台广播”——服务器发送消息，客户端接收消息。它非常适合：

- 实时通知，让用户随时了解最新信息
- 自动更新的实时数据源
- 长时间运行操作的进度更新
- 用于监控与日志的事件流

该协议基于标准 HTTP，因而实现简单、与现代浏览器和 HTTP 客户端兼容性好。当连接丢失时，客户端会自动尝试重连，使其在实时应用中具有可靠性。

### 何时在 Goa 中使用 SSE

Goa 提供三种主要的流式选项：

1. **服务器发送事件（SSE）**：基于 HTTP 的单向服务器到客户端流
2. **WebSocket**：支持全双工的双向流通信
3. **gRPC**：高性能的 RPC，支持流式

在以下情况下选择 SSE：
- 只需要服务器到客户端的通信
- 希望利用 HTTP 的简单性与兼容性
- 需要自动重连处理
- 构建需要实时更新的 Web 应用

## 实现

### 设计

我们来创建一个完整的 SSE 服务。首先，在你的 `design` 包中创建新文件（例如 `design/sse.go`）：

```go
package design

import . "goa.design/goa/v3/dsl"

// Event 表示通过 SSE 发送的消息
var Event = Type("Event", func() {
    Description("通过 SSE 发送的通知消息")
    Attribute("message", String, "消息体")
    Attribute("timestamp", Int, "Unix 时间戳")
    Required("message", "timestamp")
})

// SSEService 定义 SSE 服务
var _ = Service("sse", func() {
    Description("演示服务器发送事件的服务")

    Method("stream", func() {
        Description("使用 SSE 进行事件流式传输")
        StreamingResult(Event) // SSE 方法必须使用 StreamingResult
        HTTP(func() {
            GET("/events/stream")
            ServerSentEvents() // 使用 SSE 而非 WebSocket
        })
    })
})
```

### 代码生成

定义设计后，生成服务代码：

```bash
goa gen github.com/yourusername/yourproject/design
```

这将生成：
- 服务接口与实现存根
- HTTP 服务器与客户端代码
- OpenAPI 规范
- 客户端示例代码

### 服务器端实现

为服务实现创建新文件（例如 `sse.go`）：

```go
package sse

import (
    "context"
    "time"

    "github.com/yourusername/yourproject/gen/sse"
)

type Service struct {
    // 在此添加依赖
}

// NewService 创建一个新的 SSE 服务
func NewService() *Service {
    return &Service{}
}

// Stream 实现 SSE 端点
func (s *Service) Stream(ctx context.Context, stream sse.StreamServerStream) error {
    // 每秒发送一条消息
    ticker := time.NewTicker(time.Second)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            // 创建并发送事件
            event := &sse.Event{
                Message:   "Hello from server!",
                Timestamp: time.Now().Unix(),
            }
            if err := stream.Send(event); err != nil {
                return err
            }
        case <-ctx.Done():
            return nil
        }
    }
}
```

### 自定义 SSE 事件

SSE 协议为我们提供了多种自定义事件发送方式。可以把这些视为广播的不同“频道”——我们可以发送不同类型的消息、维护消息顺序、并控制客户端的重连行为。

如下自定义事件：

```go
ServerSentEvents(func() {
    SSEEventData("message") // 实际的消息内容
    SSEEventType("type")    // 消息类型
    SSEEventID("id")        // 消息的唯一标识符
    SSEEventRetry("retry")  // 连接丢失后重连的等待时间
})
```

各字段含义：

- **数据字段**（`SSEEventData`）：消息的主体内容。可以是任何可转换为 JSON 的数据类型。如果不指定，则整个事件对象会作为数据发送。
- **事件类型**（`SSEEventType`）：用于对消息进行分类。例如可区分“notification”（通知）与“alert”（告警）。客户端可以监听特定类型的消息。
- **事件 ID**（`SSEEventID`）：类似消息编号，帮助客户端跟踪已接收的消息，尤其在连接丢失并恢复时有用。
- **重试间隔**（`SSEEventRetry`）：指示客户端在连接丢失后尝试重连前等待的时间。

如下为一个使用全部特性的完整示例：

```go
var Event = Type("Event", func() {
    Description("通过 SSE 发送的通知消息")
    Attribute("message", String, "消息体")
    Attribute("type", String, "事件类型（例如 'notification'、'alert'）")
    Attribute("id", String, "唯一事件标识符")
    Attribute("retry", Int, "以毫秒为单位的重连等待时间")
    Required("message", "type", "id")
})

Method("stream", func() {
    Description("使用服务器发送事件进行流式传输")
    StreamingResult(Event)
    HTTP(func() {
        GET("/events/stream")
        ServerSentEvents(func() {
            SSEEventData("message")
            SSEEventType("type")
            SSEEventID("id")
            SSEEventRetry("retry") // 仅当 retry 字段非空时发送
        })
    })
})
```

该端点发送的事件示例：
```
event: notification
id: 123
data: {"message": "Hello"}

event: alert
id: 124
data: {"message": "Warning"}
```

### 处理 Last-Event-Id

SSE 的一个强大特性是在连接丢失时能从上次中断的位置恢复流式传输。这由 `Last-Event-Id` 头实现。可将其视为“书签”——当客户端重连时，它可以告诉服务器“我上次接收到的消息编号是 X，请从之后的消息继续发送”。

#### 为什么使用 Last-Event-Id？

`Last-Event-Id` 对构建可靠的实时应用至关重要。它确保客户端在连接中断时不会错过任何消息，并帮助维护消息的正确顺序。这对于那些丢失或错序消息会导致问题的应用尤为重要。

#### 实现

让我们在服务中实现对 `Last-Event-Id` 的支持：

1. 首先，在设计中接收最后一个事件 ID：

```go
Method("stream", func() {
    Description("使用服务器发送事件进行流式传输")
    Payload(func() {
        Attribute("startID", String, "最后接收事件的 ID", func() {
            Description("用于从特定事件恢复流式传输")
            Example("123")
        })
    })
    StreamingResult(Event)
    HTTP(func() {
        GET("/events/stream")
        ServerSentEvents(func() {
            SSERequestID("startID") // 将 Last-Event-Id 头映射到 startID
        })
    })
})
```

2. 然后，在服务器逻辑中处理从特定事件恢复：

```go
func (s *svc) Stream(ctx context.Context, p *svc.StreamPayload, stream svc.StreamServerStream) error {
    // 从载荷中获取最后事件 ID
    lastID := p.StartID

    // 如果存在最后 ID，则跳过事件直到找到它
    if lastID != "" {
        // 跳过事件直到找到最后接收的事件
        for ev := range s.events {
            if ev.ID == lastID {
                break
            }
        }
    }

    // 开始流式发送新事件
    for ev := range s.events {
        if err := stream.Send(ev); err != nil {
            return err
        }
    }
    return nil
}
```

3. 最后，实现一个可处理重连的客户端：

```javascript
class EventSourceWithRetry extends EventSource {
    constructor(url) {
        super(url);
        this.lastEventId = null;
        
        // 存储最后事件 ID
        this.addEventListener('message', (event) => {
            if (event.lastEventId) {
                this.lastEventId = event.lastEventId;
            }
        });
    }

    // 覆盖默认的重连行为
    reconnect() {
        if (this.lastEventId) {
            // 使用 Last-Event-Id 头创建新的 EventSource
            const headers = new Headers();
            headers.append('Last-Event-Id', this.lastEventId);
            return new EventSourceWithRetry(this.url, { headers });
        }
        return new EventSourceWithRetry(this.url);
    }
}

// 用法
const eventSource = new EventSourceWithRetry('/events/stream');
```

#### 最佳实践

实现 `Last-Event-Id` 时，请注意：

1. **事件 ID 格式**：选择适合你应用的格式。顺序编号便于维护顺序；UUID 更适合保证唯一性。确保 ID 有足够信息以唯一标识事件。
2. **存储考虑**：决定需要保留历史事件的时长。可以将最后事件 ID 存在浏览器 localStorage 中以在页面刷新后仍可恢复；也可实现旧事件 ID 的清理机制。
3. **错误处理**：为最后事件 ID 不再可用的情况做规划。可能服务器清理了旧事件，或 ID 无效。为这些情况准备回退机制。
4. **性能**：注意事件的存储与查找方式。可采用滑动窗口策略，仅在内存中保留最近的事件。

## 客户端使用

### 浏览器客户端

连接到 SSE 端点非常简单。以下是使用浏览器 `EventSource` API 的基本示例：

```javascript
const eventSource = new EventSource('/events/stream');

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    console.log('Received event:', data);
};

eventSource.onerror = (error) => {
    console.error('EventSource failed:', error);
    eventSource.close();
};
```

### Go 客户端

Goa 会生成可在你的 Go 应用中使用的客户端代码：

```go
package main

import (
    "context"
    "log"

    "github.com/yourusername/yourproject/gen/sse"
    "github.com/yourusername/yourproject/gen/sse/client"
)

func main() {
    // 创建客户端
    c := client.NewClient("http://localhost:8080")

    // 创建上下文
    ctx := context.Background()

    // 开始流式接收
    stream, err := c.Stream(ctx)
    if err != nil {
        log.Fatal(err)
    }

    // 接收事件
    for {
        event, err := stream.Recv()
        if err != nil {
            log.Fatal(err)
        }
        log.Printf("Received: %+v", event)
    }
}
```

## 测试

### 服务器端测试

如下测试你的 SSE 端点：

```go
func TestStream(t *testing.T) {
    // 创建服务
    svc := NewService()

    // 创建测试上下文
    ctx := context.Background()

    // 创建测试流
    stream := &TestStream{
        events: make(chan *sse.Event),
        errors: make(chan error),
    }

    // 在 goroutine 中启动流式
    go func() {
        err := svc.Stream(ctx, stream)
        if err != nil {
            stream.errors <- err
        }
    }()

    // 接收事件
    for i := 0; i < 5; i++ {
        select {
        case event := <-stream.events:
            if event.Message == "" {
                t.Error("Expected message, got empty string")
            }
        case err := <-stream.errors:
            t.Fatal(err)
        case <-time.After(time.Second):
            t.Fatal("Timeout waiting for event")
        }
    }
}

type TestStream struct {
    events chan *sse.Event
    errors chan error
}

func (s *TestStream) Send(event *sse.Event) error {
    s.events <- event
    return nil
}

func (s *TestStream) Close() error {
    close(s.events)
    close(s.errors)
    return nil
}
```

### 客户端测试

在客户端侧，可以使用模拟服务器：

```go
func TestClient(t *testing.T) {
    // 创建测试服务器
    server := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 设置 SSE 响应头
        w.Header().Set("Content-Type", "text/event-stream")
        w.Header().Set("Cache-Control", "no-cache")
        w.Header().Set("Connection", "keep-alive")

        // 发送测试事件
        for i := 0; i < 5; i++ {
            fmt.Fprintf(w, "data: {\"message\":\"test %d\"}\n\n", i)
            w.(http.Flusher).Flush()
            time.Sleep(100 * time.Millisecond)
        }
    }))
    defer server.Close()

    // 创建客户端
    c := client.NewClient(server.URL)

    // 开始流式
    stream, err := c.Stream(context.Background())
    if err != nil {
        t.Fatal(err)
    }

    // 接收事件
    for i := 0; i < 5; i++ {
        event, err := stream.Recv()
        if err != nil {
            t.Fatal(err)
        }
        if event.Message != fmt.Sprintf("test %d", i) {
            t.Errorf("Expected message 'test %d', got '%s'", i, event.Message)
        }
    }
}
```

## 局限性

尽管 SSE 很强大，但仍需注意以下局限：

- 单向通信——服务器可以向客户端发送，客户端无法回发
- 仅限文本数据（可发送 JSON）
- 浏览器会限制并发 SSE 连接数
- 浏览器会自动处理重连，但自定义客户端需要自行实现重连逻辑

## 参阅

- [服务器发送事件规范](https://html.spec.whatwg.org/multipage/server-sent-events.html)
- [示例实现](https://github.com/goadesign/examples/tree/master/sse)
- [Goa 设计文档](/docs/4-concepts/design)
- [Goa 流式教程](/docs/3-tutorials/4-streaming) 