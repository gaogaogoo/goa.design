---
title: "拦截器实现"
description: "了解如何实现 Goa 拦截器以及常见模式"
weight: 3
---

本文解释了如何在 Goa 中实现拦截器，重点介绍拦截器模式与 `next` 函数所提供的灵活性。

## 实现结构

Goa 会根据你的设计生成类型安全的拦截器接口。每个拦截器方法遵循如下签名：

```go
func (i *Interceptor) MethodName(ctx context.Context, info *InterceptorInfo, next goa.Endpoint) (any, error)
```

含义：
- `ctx`：请求上下文
- `info`：对 Payload 与 Result 属性的类型安全访问
- `next`：被包装的端点（服务方法或下一个拦截器）

## next 函数

`next` 函数是拦截器灵活性的关键。它表示被包装的端点，可在拦截器代码中的任意时刻调用。常见的三种模式如下：

### 1. 预处理模式（Pre-Processing）

在修改上下文或 Payload 后，最后调用 `next`：

```go
func (i *Interceptor) SetDeadline(ctx context.Context, info *SetDeadlineInfo, next goa.Endpoint) (any, error) {
    // 在调用端点前修改上下文
    deadline := time.Now().Add(30 * time.Second)
    ctx, cancel := context.WithDeadline(ctx, deadline)
    defer cancel()
    
    // 使用修改后的上下文调用端点
    return next(ctx, info.RawPayload())
}
```

### 2. 后处理模式（Post-Processing）

先调用 `next`，再处理其结果：

```go
func (i *Interceptor) Cache(ctx context.Context, info *CacheInfo, next goa.Endpoint) (any, error) {
    // 先调用端点
    resp, err := next(ctx, info.RawPayload())
    if err != nil {
        return nil, err
    }
    
    // 处理响应
    if result := info.Result(resp); result != nil {
        // 缓存结果...
    }
    
    return resp, nil
}
```

### 3. 包裹模式（Wrapper）

在调用 `next` 前后均进行处理：

```go
func (i *Interceptor) RequestAudit(ctx context.Context, info *RequestAuditInfo, next goa.Endpoint) (any, error) {
    // 前置处理
    start := time.Now()
    payload := info.RawPayload()
    
    // 调用端点
    resp, err := next(ctx, payload)
    
    // 后置处理
    duration := time.Since(start)
    if err != nil {
        log.Printf("request failed: %v, duration: %v", err, duration)
        return nil, err
    }
    
    log.Printf("request succeeded, duration: %v", duration)
    return resp, nil
}
```

## 使用 info 对象

生成的 `info` 对象提供对 Payload 与 Result 的类型安全访问。访问方法会根据你的设计 DSL 自动生成：

```go
// 设计中
var TraceRequest = Interceptor("TraceRequest", func() {
    Description("为请求添加追踪上下文")
    
    ReadPayload(func() {
        Attribute("trace_id")    // 生成 info.TraceID() 方法
        Attribute("span_id")     // 生成 info.SpanID() 方法
    })
    
    WriteResult(func() {
        Attribute("duration")    // 生成 info.SetDuration() 方法
    })
})

// 生成的实现中
func (i *Interceptor) TraceRequest(ctx context.Context, info *TraceRequestInfo, next goa.Endpoint) (any, error) {
    // 生成的方法与设计中的属性名一致
    traceID := info.TraceID()   // 返回 trace_id 的值
    spanID := info.SpanID()     // 返回 span_id 的值
    
    resp, err := next(ctx, info.RawPayload())
    if err != nil {
        return nil, err
    }
    
    // 使用生成的 setter 写入结果属性
    if result := info.Result(resp); result != nil {
        info.SetDuration(result, time.Since(start))
    }
    
    return resp, nil
}
```

对于设计中的每个属性：
- `ReadPayload`/`ReadResult` 属性会生成 getter 方法
- `WritePayload`/`WriteResult` 属性会生成 setter 方法
- 方法名为属性名的驼峰形式
- 类型与设计定义保持一致

## 流式拦截器

流式拦截器用于处理流式方法，与常规拦截器的关键区别是：它们针对流中的每条消息调用，而不是每个请求调用一次。与常规拦截器一样，拦截器运行于服务端或客户端任一侧（非同时）：

```go
// 服务端流式拦截器
func (i *Interceptor) ServerStreamMonitor(ctx context.Context, info *ServerStreamMonitorInfo, next goa.Endpoint) (any, error) {
    // 该拦截器会在流中的每条消息上被调用
    
    // 对于服务端流式结果：
    // - 每次服务端将要发送消息时调用
    // - info.StreamingResult() 包含即将发送的消息
    resp, err := next(ctx, info.RawPayload())
    if err != nil {
        return nil, err
    }
    
    if result := info.StreamingResult(resp); result != nil {
        // 监控服务端流的出站消息
        log.Printf("server sending message: %v", result)
    }
    
    return resp, nil
}

// 客户端流式拦截器
func (i *Interceptor) ClientStreamMonitor(ctx context.Context, info *ClientStreamMonitorInfo, next goa.Endpoint) (any, error) {
    // 该拦截器会在流中的每条消息上被调用
    
    // 对于客户端流式请求：
    // - 每次客户端发送消息时调用
    // - info.StreamingPayload() 包含即将发送的消息
    if payload := info.StreamingPayload(); payload != nil {
        // 监控客户端流的出站消息
        log.Printf("client sending message: %v", payload)
    }
    
    return next(ctx, info.RawPayload())
}
```

按消息执行可实现：
- 在数据流经系统时逐条处理
- 使用拦截器实例字段在消息间维护状态
- 通过返回错误提前终止流
- 对消息进行转换或过滤

示例：服务端速率限制拦截器：

```go
type StreamRateLimiter struct {
    messageCount int
    lastReset   time.Time
    limit       int
}

func (i *StreamRateLimiter) LimitServerStream(ctx context.Context, info *LimitServerStreamInfo, next goa.Endpoint) (any, error) {
    i.mu.Lock()
    // 每分钟重置计数
    if time.Since(i.lastReset) > time.Minute {
        i.messageCount = 0
        i.lastReset = time.Now()
    }

    // 在处理消息前检查速率
    if i.messageCount >= i.limit {
        i.mu.Unlock()
        return nil, fmt.Errorf("rate limit exceeded") 
    }

    // 处理消息并递增计数
    i.messageCount++
    i.mu.Unlock()

    return next(ctx, info.RawPayload())
}
```

## 错误处理

拦截器可处理被包装端点返回的错误：

```go
func (i *Interceptor) ErrorHandler(ctx context.Context, info *ErrorHandlerInfo, next goa.Endpoint) (any, error) {
    resp, err := next(ctx, info.RawPayload())
    if err != nil {
        // 将错误转换为合适类型
        if gerr, ok := err.(*goa.ServiceError); ok {
            // 处理服务错误...
            return nil, gerr
        }
        // 包装未知错误
        return nil, goa.NewServiceError("internal error")
    }
    return resp, nil
}
```

## 下一步

- 查看拦截器设计的[最佳实践](../4-best-practices)
