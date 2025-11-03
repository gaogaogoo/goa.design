---
title: "分布式追踪"
description: "使用 OpenTelemetry 实现分布式追踪"
weight: 2
---

现代应用是复杂的分布式系统，一个用户请求可能会触达几十个服务、数据库以及外部 API。
当出现问题时，理解到底发生了什么可能非常困难。这正是分布式追踪发挥作用的地方。

## 什么是分布式追踪？

分布式追踪沿着请求在系统中的传播路径进行跟踪，在每一个步骤记录耗时、错误以及上下文。
可以把它想象成请求的 GPS 轨迹：你可以精确看到它去了哪里、每一步花了多久、在哪一步出现了问题。

### 核心概念

1. **Trace（跟踪）**：一个请求在系统中完整的旅程
2. **Span（跨度）**：旅程中的单个操作（例如数据库查询或一次 API 调用）
3. **Context（上下文）**：跟随请求在系统中传播的信息（例如用户 ID 或关联 ID）
4. **Attributes（属性）**：描述发生了什么的键值对（例如订单 ID 或错误详情）

下面是一个可视化示例：

```
Trace: 创建订单
├── Span: 校验用户 (10ms)
│   └── Attribute: user_id=123
├── Span: 检查库存 (50ms)
│   ├── Attribute: product_id=456
│   └── Event: "库存较低"
└── Span: 处理支付 (200ms)
    ├── Attribute: amount=99.99
    └── Error: "资金不足"
```

## 自动仪表化

开始使用追踪最简单的方式是启用自动仪表化。
Clue 提供中间件，能够在零代码改动的情况下自动追踪 HTTP 与 gRPC 请求：

```go
// 对于 HTTP 服务器，用 OpenTelemetry 中间件包裹你的处理器。
// 这会为所有进入的请求自动创建追踪。
handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    // 你的处理逻辑
})

// 添加追踪中间件
mux.Use(otelhttp.NewMiddleware("my-service"))

// 中间件将会：
// - 为每个请求创建一个 span
// - 记录 HTTP 方法、状态码与 URL
// - 追踪请求耗时
// - 将上下文传播到下游服务
```

对于 gRPC 服务，使用所提供的拦截器：

```go
// 创建开启追踪的 gRPC 服务器
server := grpc.NewServer(
    // 添加 OpenTelemetry 处理器以追踪所有 RPC
    grpc.StatsHandler(otelgrpc.NewServerHandler()))

// 这将自动：
// - 追踪所有 gRPC 方法
// - 记录方法名与状态码
// - 追踪延迟
// - 处理上下文传播
```

## 手动仪表化

自动仪表化适合请求边界，但你经常需要为关键的业务操作添加自定义 span。
下面演示如何在代码中添加自定义追踪：

```go
func processOrder(ctx context.Context, order *Order) error {
    // 为该操作启动一个新的 span。
    // span 名称 "process_order" 会出现在你的追踪中。
    ctx, span := otel.Tracer("myservice").Start(ctx, "process_order")
    
    // 在函数返回时务必结束 span
    defer span.End()

    // 将业务上下文作为 span 属性添加
    span.SetAttributes(
        // 这些属性有助于过滤与分析追踪
        attribute.String("order.id", order.ID),
        attribute.Float64("order.amount", order.Amount),
        attribute.String("customer.id", order.CustomerID))

    // 用事件记录关键步骤与时间戳
    span.AddEvent("validating_order")
    if err := validateOrder(ctx, order); err != nil {
        // 记录带上下文的错误
        span.RecordError(err)
        span.SetStatus(codes.Error, "order validation failed")
        return err
    }
    span.AddEvent("order_validated")

    // 为子操作创建嵌套的 span
    ctx, paymentSpan := otel.Tracer("myservice").Start(ctx, "process_payment")
    defer paymentSpan.End()

    if err := processPayment(ctx, order); err != nil {
        paymentSpan.RecordError(err)
        paymentSpan.SetStatus(codes.Error, "payment failed")
        return err
    }

    return nil
}
```

## 追踪外部调用

当你的服务调用其他服务或数据库时，你希望把这些操作也纳入追踪范围。
下面展示如何为不同类型的客户端进行仪表化：

### HTTP Clients

```go
// 创建启用追踪的 HTTP 客户端
client := &http.Client{
    // 使用 OpenTelemetry 包裹默认传输层
    Transport: otelhttp.NewTransport(
        http.DefaultTransport,
        // 启用更详细的 HTTP 追踪（可选）
        otelhttp.WithClientTrace(func(ctx context.Context) *httptrace.ClientTrace {
            return otelhttptrace.NewClientTrace(ctx)
        }),
    ),
}

// 现在所有请求都会被自动追踪
resp, err := client.Get("https://api.example.com/data")
```

### gRPC Clients

```go
// 创建启用追踪的 gRPC 客户端连接
conn, err := grpc.DialContext(ctx,
    "service:8080",
    // 添加 OpenTelemetry 处理器
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    // Other options...
    grpc.WithTransportCredentials(insecure.NewCredentials()))

// 使用该连接进行的所有调用都会被追踪
client := pb.NewServiceClient(conn)
```

### Database Calls

对于数据库操作，创建自定义 span 以追踪查询：

```go
func (r *Repository) GetUser(ctx context.Context, id string) (*User, error) {
    // 为数据库操作创建一个 span
    ctx, span := otel.Tracer("repository").Start(ctx, "get_user")
    defer span.End()

    // 添加查询上下文
    span.SetAttributes(
        attribute.String("db.type", "postgres"),
        attribute.String("db.user_id", id))

    // 执行查询
    var user User
    if err := r.db.GetContext(ctx, &user, "SELECT * FROM users WHERE id = $1", id); err != nil {
        // 记录数据库错误
        span.RecordError(err)
        span.SetStatus(codes.Error, "database query failed")
        return nil, err
    }

    return &user, nil
}
```

## 上下文传播

要使追踪在服务边界之间生效，必须随请求传播追踪上下文。
上述已仪表化的客户端会自动完成这件事，下面展示手动实现的方式：

```go
// 收到请求时，提取追踪上下文
func handleIncoming(w http.ResponseWriter, r *http.Request) {
    // 从请求头中提取追踪上下文
    ctx := otel.GetTextMapPropagator().Extract(r.Context(),
        propagation.HeaderCarrier(r.Header))
    
    // 使用该上下文执行后续所有操作
    processRequest(ctx)
}

// 发起请求时，注入追踪上下文
func makeOutgoing(ctx context.Context) error {
    req, _ := http.NewRequestWithContext(ctx, "GET", "https://api.example.com", nil)
    
    // 将追踪上下文注入请求头
    otel.GetTextMapPropagator().Inject(ctx,
        propagation.HeaderCarrier(req.Header))
    
    resp, err := http.DefaultClient.Do(req)
    return err
}
```

上下文传播使用 W3C Trace Context 标准，以确保追踪能在不同服务与可观测性系统间互操作。
了解更多上下文传播相关内容：

- [OpenTelemetry Context and Propagation](https://opentelemetry.io/docs/concepts/context-propagation/)
- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry Go SDK Propagation](https://pkg.go.dev/go.opentelemetry.io/otel/propagation)

## 控制追踪数据量

在生产系统中，追踪每一个请求会产生大量数据，从而带来高昂的存储成本与性能开销。
采样可以帮助你在控制成本的同时，收集足够的追踪来理解系统行为。

### 为什么需要采样？

1. **成本控制**：存储与处理追踪数据可能非常昂贵
2. **性能**：生成追踪会对请求带来一定开销
3. **分析**：理解系统行为并不总需要每一条追踪
4. **存储**：追踪数据可能迅速占用大量存储空间

### 固定比率采样

最简单的方式是按固定百分比对请求进行采样。这种方式可预测且易于理解：

```go
// 采样 10% 的请求（0.1 = 10%）
cfg := clue.NewConfig(ctx,
    serviceName,
    version,
    metricExporter,
    spanExporter,
    clue.WithSamplingRate(0.1))

// 常见采样率：
// 1.0    = 100%（所有请求，适合开发环境）
// 0.1    = 10% （生产环境常用）
// 0.01   = 1%  （高流量服务）
// 0.001  = 0.1%（超高流量服务）
```

固定比率采样适用于：
- 你的流量比较稳定
- 你希望存储成本可预测
- 你不需要动态调整采样率

然而，固定比率采样也有重要的局限：

1. **低流量盲区**
   在低峰期，固定采样率可能导致可观测性产生明显缺口。
   例如，采样率为 10%，每分钟只有 10 个请求时，平均每分钟仅能捕获 1 条追踪。
   每分钟仅 2 个请求时，可能数分钟都捕获不到任何追踪。
   恰恰在你最需要可见性的时刻（异常的低流量时期，可能意味着问题发生）会出现盲区。

2. **统计不稳定性**
   在低流量情况下，实际采样率可能与配置值严重偏离。
   例如，5 分钟内只有 5 个请求且采样率为 10%：你可能一个追踪都没有（实际 0%），也可能捕获 2 条追踪（实际 40%）。
   二者都无法准确代表设定的 10% 采样率，导致难以从数据中得出可靠结论。

3. **遗漏边缘情况**
   一些关键但不频繁的事件（如错误或罕见边界情况）如果恰好发生在未被采样的请求上，可能完全没有记录。
   在低流量时期尤为糟糕：低流量与固定采样率的叠加会显著增加遗漏重要事件的概率。

对于流量不稳定或较低的服务，考虑如下替代方案：
- 使用自适应采样（下一节将介绍）
- 对重要事件启用条件采样：
  
  条件采样意味着：无论常规采样率如何，始终捕获满足特定条件（对业务重要）的请求或事件。
  可以把它想象成 VIP 名单——这些特殊情况总是被记录，而常规流量遵循正常采样规则。

  条件采样的常见场景包括：
  - 错误场景：当出错时总是捕获（如 HTTP 500 或交易失败）
  - 性能问题：当请求耗时超过阈值时总是捕获
  - 业务重要性：对高价值交易或关键业务操作总是捕获
  - 调试需求：对正在调试的特定用户、功能或端点总是捕获

  这种方法两全其美：你不会错过需要调查的重要事件，同时通过较低比率采样常规流量来控制总体追踪量（与成本）。

- 在已知低流量时段使用更高的采样率

### 自适应采样

为了更动态地进行控制，Clue 提供了自适应采样器，它会根据观测到的请求速率自动调整采样比率，以维持目标采样速率：

```go
// 目标每秒 100 个样本，每 1000 个请求重新计算采样率
cfg := clue.NewConfig(ctx,
    serviceName,
    version,
    metricExporter,
    spanExporter,
    clue.WithSampler(
        clue.AdaptiveSampler(
            100,    // 目标采样速率（样本/秒）
            1000))) // 用于调整采样率的样本窗口大小
```

自适应采样器的工作方式如下：
1. 初始阶段，采样所有请求直到达到第一个样本窗口大小
2. 每经过一个样本窗口大小（例如每 1000 个请求），它会：
   - 计算当前请求速率（请求/秒）
   - 调整采样概率以达到目标采样速率
   - 重置计数进入下一个窗口

例如，目标是每秒 100 个样本：
- 低流量（50 rps）：采样器会采集所有请求，因为 50 < 100

- 正常流量（500 rps）：采样器会采集约 20% 的请求（100/500），以维持每秒 100 的目标

- 高流量（2000 rps）：采样器会采集约 5% 的请求（100/2000），以维持目标速率

自适应采样带来多项关键好处：即使一天中流量波动，它也能维持稳定的样本采集速率。
在低峰期，所有请求都可被采样，从而获得最大可见性；在流量高峰时，采样率会自动降低以控制成本，同时仍提供具有统计意义的数据。
这种调整会随流量变化平滑进行，避免采样率的骤降或骤升对分析造成偏差。

进一步了解采样：
- [OpenTelemetry Sampling](https://opentelemetry.io/docs/concepts/sampling/)
- [Sampling Performance Impact](https://www.jaegertracing.io/docs/1.41/sampling/#performance-overhead)
- [Adaptive Sampling Design](https://github.com/jaegertracing/jaeger/blob/main/docs/adaptive_sampling.md)

## 错误处理最佳实践

在追踪中进行正确的错误处理对于调试和监控应用至关重要。
当出现问题时，你的追踪应提供足够的上下文来理解发生了什么、为何发生以及发生在何处。

### 核心原则

1. **始终记录错误**：每一个错误都应该在追踪中被捕获
2. **添加上下文**：包含导致错误的相关信息
3. **设置状态**：更新 span 状态以反映错误条件
4. **保护隐私**：不要在错误详情中包含敏感数据
5. **保持一致**：在整个服务中使用一致的错误属性命名

### 基础错误记录

在追踪中处理错误时，有两种不同的方式来记录错误信息：

1. **SetStatus**：设置 span 的整体状态。每个 span 只能调用一次，表示操作的最终状态：
   ```go
   // 设置 span 的最终状态（只能调用一次）
   span.SetStatus(codes.Error, "operation failed")
   ```

2. **RecordError**：向 span 的时间线添加错误事件。可多次调用以记录在该 span 生命周期内出现的不同错误：
   ```go
   // 将多个错误记录为事件（可多次调用）
   span.RecordError(err1)  // 第一次错误
   span.RecordError(err2)  // 后续另一次错误
   span.RecordError(err3)  // 又一次错误
   ```

如下是二者配合使用的示例：
```go
func processWithRetries(ctx context.Context) error {
    ctx, span := tracer.Start(ctx, "process_with_retries")
    defer span.End()

    for attempt := 1; attempt <= maxRetries; attempt++ {
        err := process()
        if err != nil {
            // 将每次失败尝试记录为错误事件
            span.RecordError(err,
                trace.WithAttributes(
                    attribute.Int("attempt", attempt)))
            continue
        }
        // 重试后操作成功
        span.SetStatus(codes.Ok, "succeeded after retries")
        return nil
    }

    // 所有重试均失败后设置最终错误状态
    span.SetStatus(codes.Error, "all retries failed")
    return errors.New("max retries exceeded")
}
```

将 `RecordError` 与 `SetStatus` 分离带来多项好处：
首先，它允许你在操作过程中记录每一次发生的错误；其次，通过 `SetStatus` 明确标示操作的最终结果；
第三，由于错误以带时间戳的事件记录，你可以获得清晰的时间顺序；最后，该方式有助于区分可恢复的临时失败与最终的操作结果。

### 详细的错误处理

对于更复杂的场景，添加上下文并对错误进行分类：

```go
func processRequest(ctx context.Context, req *Request) error {
    ctx, span := otel.Tracer("service").Start(ctx, "process_request")
    defer span.End()

    // 添加有助于调试的请求上下文
    span.SetAttributes(
        attribute.String("request.id", req.ID),
        attribute.String("request.type", req.Type),
        attribute.String("user.id", req.UserID))

    if err := validate(req); err != nil {
        // 对于校验错误，包含失败字段信息
        span.RecordError(err,
            trace.WithAttributes(
                attribute.String("error.type", "validation"),
                attribute.String("validation.field", err.Field),
                attribute.String("validation.constraint", err.Constraint)))
        
        // 添加事件以标记错误发生时刻
        span.AddEvent("validation_failed",
            trace.WithAttributes(
                attribute.String("error.details", err.Error())))
        
        span.SetStatus(codes.Error, "request validation failed")
        return err
    }

    if err := process(req); err != nil {
        // 对于系统错误，包含相关系统上下文
        span.RecordError(err,
            trace.WithAttributes(
                attribute.String("error.type", "system"),
                attribute.String("error.component", err.Component),
                attribute.String("error.operation", err.Operation)))
        
        // 对于严重错误，你可能希望记录堆栈
        span.AddEvent("system_error",
            trace.WithAttributes(
                attribute.String("error.stack", string(debug.Stack()))))
        
        span.SetStatus(codes.Error, "request processing failed")
        return err
    }

    // 记录成功完成
    span.SetStatus(codes.Ok, "request processed successfully")
    return nil
}
```

### 错误分类

将错误组织为类别，以便更容易分析：

```go
// 常见错误类别
const (
    ErrorTypeValidation = "validation"  // 输入校验错误
    ErrorTypeSystem     = "system"      // 内部系统错误
    ErrorTypeExternal   = "external"    // 外部服务错误
    ErrorTypeResource   = "resource"    // 资源可用性错误
    ErrorTypeSecurity   = "security"    // 安全相关错误
)

// 按类别记录错误
func recordError(span trace.Span, err error, errType string) {
    span.RecordError(err,
        trace.WithAttributes(
            attribute.String("error.type", errType),
            attribute.String("error.message", err.Error())))
}

// 使用示例
if err := validateInput(req); err != nil {
    recordError(span, err, ErrorTypeValidation)
    span.SetStatus(codes.Error, "validation failed")
    return err
}
```

### 错误属性

对错误信息使用一致的属性命名：

```go
// 标准的错误属性
span.SetAttributes(
    attribute.String("error.type", errType),       // 错误类别
    attribute.String("error.message", err.Error()), // 错误描述
    attribute.String("error.code", errCode),        // 错误码（如果有）
    attribute.String("error.component", component), // 失败的组件
    attribute.String("error.operation", operation), // 失败的操作
    attribute.Bool("error.retryable", isRetryable), // 是否可重试
)
```

### 错误事件

使用事件记录与错误相关的时间戳：

```go
// 记录错误发生
span.AddEvent("error_detected",
    trace.WithAttributes(
        attribute.String("error.message", err.Error())))

// 记录重试尝试
span.AddEvent("retry_attempt",
    trace.WithAttributes(
        attribute.Int("retry.count", retryCount),
        attribute.Int("retry.max", maxRetries)))

// 记录错误解决
span.AddEvent("error_resolved",
    trace.WithAttributes(
        attribute.String("resolution", "retry_succeeded")))
```

### 最佳实践

1. **错误粒度**：
   ```go
   // 过于泛化
   span.RecordError(err)  // 不建议这样做
   
   // 更好——包含上下文
   span.RecordError(err,
       trace.WithAttributes(
           attribute.String("error.type", errType),
           attribute.String("error.context", context)))
   ```

2. **隐私与安全**：
   ```go
   // 不要包含敏感数据
   span.RecordError(err,
       trace.WithAttributes(
           attribute.String("user.password", password),    // 不可！
           attribute.String("credit_card", cardNumber)))   // 不可！
   
   // 可以包含安全的标识信息
   span.RecordError(err,
       trace.WithAttributes(
           attribute.String("user.id", userID),           // 可
           attribute.String("transaction.id", txID)))     // 可
   ```

3. **错误恢复**：
   ```go
   func processWithRetry(ctx context.Context) error {
       ctx, span := tracer.Start(ctx, "process_with_retry")
       defer span.End()
       
       for attempt := 1; attempt <= maxRetries; attempt++ {
           if err := process(); err != nil {
               span.AddEvent("retry_attempt",
                   trace.WithAttributes(
                       attribute.Int("attempt", attempt)))
               continue
           }
           return nil
       }
       
       span.SetStatus(codes.Error, "all retries failed")
       return errors.New("max retries exceeded")
   }
   ```

4. **与日志关联**：
   ```go
   if err != nil {
       // 将 Trace ID 写入日志以便关联
       logger.Error("operation failed",
           "trace_id", span.SpanContext().TraceID(),
           "error", err)
           
       span.RecordError(err)
       span.SetStatus(codes.Error, "operation failed")
   }
   ```

### 延伸阅读

关于在追踪中处理错误的更多信息：
- [OpenTelemetry Error Handling](https://opentelemetry.io/docs/concepts/signals/traces/#errors)
- [Semantic Conventions for Errors](https://opentelemetry.io/docs/concepts/semantic-conventions/exceptions/)
- [Error Status Codes](https://pkg.go.dev/go.opentelemetry.io/otel/codes)

## 实践建议

### 1. Span 命名

使用一致且具描述性的名称，帮助识别操作：

```go
// 推荐的 span 名称
"http.request"              // 类型 + 操作
"db.query.get_user"        // 组件 + 类型 + 操作
"payment.process_charge"    // 领域 + 操作

// 不推荐的 span 名称
"process"                  // 含义模糊
"handleFunc"              // 实现细节
"do_stuff"                // 缺乏描述性
```

### 2. Attributes（属性）

添加有用的上下文，同时避免过度堆积：

```go
// 推荐的属性
span.SetAttributes(
    attribute.String("user.id", userID),        // 身份标识
    attribute.String("order.status", status),   // 状态
    attribute.Int64("items.count", count))      // 指标

// 不推荐的属性
span.SetAttributes(
    attribute.String("raw_json", hugejson),     // 数据量过大
    attribute.String("password", password),      // 敏感数据
    attribute.String("tmp", "xyz"))             // 缺乏意义
```

### 3. 错误处理

记录带上下文的错误：

```go
if err != nil {
    // 按类型并带上下文记录错误
    span.RecordError(err,
        trace.WithAttributes(
            attribute.String("error.type", errorType),
            attribute.String("error.context", context)))
    
    // 设置错误状态并附带描述
    span.SetStatus(codes.Error, err.Error())
    
    // 可选：添加带时间戳的错误事件
    span.AddEvent("error_occurred",
        trace.WithAttributes(
            attribute.String("error.stack", stack)))
}
```

### 4. 性能考量

- 使用适当的采样率
- 不要为极小的操作创建 span
- 避免添加非常大的属性或事件
- 在高吞吐场景中考虑批处理

## 进一步了解

关于分布式追踪的更多信息：

- [OpenTelemetry Tracing Specification](https://opentelemetry.io/docs/concepts/signals/traces/)
- [Trace Semantic Conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/)
- [Sampling Documentation](https://opentelemetry.io/docs/concepts/sampling/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
