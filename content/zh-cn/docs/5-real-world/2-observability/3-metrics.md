---
title: "服务度量"
description: "使用 OpenTelemetry 实现服务度量"
weight: 3
---

现代应用需要定量数据来理解其行为和性能。
我们处理了多少请求？它们耗时多久？我们的资源是否即将耗尽？
度量通过为服务运行提供数值化的观测，帮助回答这些问题。

## 认识度量

OpenTelemetry 提供多种度量仪表，每种都为特定的测量需求设计。每个仪表由以下要素定义：
- **名称**：你在测量什么（例如 `http.requests.total`）
- **类型**：数值如何变化（例如只递增、可上下变化）
- **单位**：可选的度量单位（例如 `ms`、`bytes`）
- **描述**：可选的说明，解释该度量代表什么

下面分别介绍各类仪表：

### 同步仪表
当事件发生时，这类仪表会在你的代码中被直接调用：

1. **计数器（Counter）**
   只会递增的数值，就像汽车的里程表：
   ```go
   requestCounter, _ := meter.Int64Counter("http.requests.total",
       metric.WithDescription("Total number of HTTP requests"),
       metric.WithUnit("{requests}"))
   
   // Usage: Increment when request received
   requestCounter.Add(ctx, 1)
   ```
   适用于：
   - 请求计数
   - 处理的字节数
   - 完成的任务数

2. **升降计数器（UpDownCounter）**
   可增可减的数值，类似队列中的项目数量：
   ```go
   queueSize, _ := meter.Int64UpDownCounter("queue.items",
       metric.WithDescription("Current items in queue"),
       metric.WithUnit("{items}"))
   
   // Usage: Add when enqueueing, subtract when dequeueing
   queueSize.Add(ctx, 1)  // Item added
   queueSize.Add(ctx, -1) // Item removed
   ```
   适用于：
   - 队列长度
   - 活跃连接数
   - 线程池大小

3. **直方图（Histogram）**
   跟踪数值分布，例如请求持续时间：
   ```go
   latency, _ := meter.Float64Histogram("http.request.duration",
       metric.WithDescription("HTTP request duration"),
       metric.WithUnit("ms"))
   
   // Usage: Record value when request completes
   latency.Record(ctx, time.Since(start).Milliseconds())
   ```
   适用于：
   - 请求延迟
   - 响应大小
   - 队列等待时间

### 异步仪表
这类仪表通过你注册的回调函数定期采集：

1. **异步计数器（Asynchronous Counter）**
   用于只会递增但你只能获取总量的数值：
   ```go
   bytesReceived, _ := meter.Int64ObservableCounter("network.bytes.received",
       metric.WithDescription("Total bytes received"),
       metric.WithUnit("By"))
   
   // Usage: Register callback to collect current value
   meter.RegisterCallback([]instrument.Asynchronous{bytesReceived},
       func(ctx context.Context) {
           bytesReceived.Observe(ctx, getNetworkStats().TotalBytesReceived)
       })
   ```
   适用于：
   - 传输的总字节数
   - 系统运行时长
   - 外部系统的累计事件

2. **异步升降计数器（Asynchronous UpDownCounter）**
   用于可增可减但你只查看当前状态的数值：
   ```go
   goroutines, _ := meter.Int64ObservableUpDownCounter("system.goroutines",
       metric.WithDescription("Current number of goroutines"),
       metric.WithUnit("{goroutines}"))
   
   // Usage: Register callback to collect current value
   meter.RegisterCallback([]instrument.Asynchronous{goroutines},
       func(ctx context.Context) {
           goroutines.Observe(ctx, int64(runtime.NumGoroutine()))
       })
   ```
   适用于：
   - 当前连接数
   - 资源池大小
   - 线程数

3. **异步仪表（Gauge）**
   用于定期采样的当前值测量：
   ```go
   cpuUsage, _ := meter.Float64ObservableGauge("system.cpu.usage",
       metric.WithDescription("CPU usage percentage"),
       metric.WithUnit("1"))
   
   // Usage: Register callback to collect current value
   meter.RegisterCallback([]instrument.Asynchronous{cpuUsage},
       func(ctx context.Context) {
           cpuUsage.Observe(ctx, getCPUUsage())
       })
   ```
   适用于：
   - CPU 使用率
   - 内存使用量
   - 温度读数
   - 磁盘空间

### 选择合适的仪表

1. 先自问以下问题：
   - 我需要在事件发生时记录数值（同步），还是定期检查状态（异步）？
   - 这个数值只会上升（计数器）还是会上下变化（升降计数器）？
   - 我是否需要分析数值的分布（直方图）？
   - 我是在测量某个当前状态（仪表）吗？

2. 常见用法：
   - 事件计数 → 计数器
   - 持续时间测量 → 直方图
   - 资源使用 → 异步仪表
   - 队列大小 → 升降计数器
   - 系统统计 → 异步仪表

## 自动度量

Clue 会为你的服务自动采集多个关键度量。这些度量无需编写代码即可提供即时可见性：

### HTTP 服务端度量
当你使用 OpenTelemetry 中间件包裹 HTTP 处理器：
```go
mux.Use(otelhttp.NewMiddleware("service"))
```

你会自动获得：
- **请求计数**：按路径、方法和状态码统计的总请求数
- **时长直方图**：请求处理耗时的分布
- **进行中请求**：当前活跃请求数
- **响应大小**：响应负载大小的分布

### gRPC 服务端度量
当你创建带有 OpenTelemetry 采集的 gRPC 服务器：
```go
server := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()))
```

你会自动获得：
- **RPC 计数**：按方法和状态码统计的 RPC 总数
- **时长直方图**：RPC 完成所需时间的分布
- **进行中 RPC**：当前活跃 RPC 数量
- **消息大小**：请求/响应大小的分布

## 自定义度量

尽管自动度量很有帮助，你通常还需要跟踪与业务相关的特定测量。下面介绍如何高效创建和使用自定义度量：

### 创建度量

首先，为你的服务获取一个 meter：
```go
meter := otel.Meter("myservice")
```

然后创建所需的度量：

1. **计数器示例**：跟踪业务事件
   ```go
   orderCounter, _ := meter.Int64Counter("orders.total",
       metric.WithDescription("Total number of orders processed"),
       metric.WithUnit("{orders}"))
   ```

2. **直方图示例**：测量处理时间
   ```go
   processingTime, _ := meter.Float64Histogram("order.processing_time",
       metric.WithDescription("Time taken to process orders"),
       metric.WithUnit("ms"))
   ```

3. **升降计数器示例**：监控队列深度
   ```go
   queueDepth, _ := meter.Int64UpDownCounter("orders.queue_depth",
       metric.WithDescription("Current number of orders in queue"),
       metric.WithUnit("{orders}"))
   ```

### 使用度量

下面是一个完整示例，它展示了如何在真实场景中使用不同类型的度量。该示例演示如何监控订单处理系统：

```go
func processOrder(ctx context.Context, order *Order) error {
    // Track total orders (counter)
    // 每处理一个订单将计数器加 1，并添加属性用于分析
    orderCounter.Add(ctx, 1,
        attribute.String("type", order.Type),
        attribute.String("customer", order.CustomerID))

    // Measure processing time (histogram)
    // 使用 defer 确保始终记录耗时，即使函数提前返回
    start := time.Now()
    defer func() {
        processingTime.Record(ctx,
            time.Since(start).Milliseconds(),
            attribute.String("type", order.Type))
    }()

    // Monitor queue depth (gauge)
    // 通过入队加一、处理完成减一，跟踪队列大小
    queueDepth.Add(ctx, 1)  // 入队时加一
    defer queueDepth.Add(ctx, -1)  // 完成时减一

    return processOrderInternal(ctx, order)
}
```

该示例体现了多项最佳实践：
- 使用计数器记录离散事件（已处理订单数）
- 使用直方图记录时长（处理时间）
- 使用升降计数器记录当前状态（队列深度）
- 添加相关属性以便分析
- 使用 defer 进行正确的清理

## 服务等级指标（SLI）

服务等级指标是帮助你理解服务健康状况和性能的关键度量。四大黄金信号（延迟、流量、错误和饱和度）能全面展现服务行为。下面分别实现：

### 1. 延迟（Latency）
延迟衡量处理请求所需的时间。以下示例展示如何在 HTTP 中间件中跟踪请求持续时间：

```go
// Create a histogram to track request durations
requestDuration, _ := meter.Float64Histogram("http.request.duration",
    metric.WithDescription("HTTP request duration"),
    metric.WithUnit("ms"))

// Middleware to measure request duration
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        next.ServeHTTP(w, r)
        // Record the duration with the request path as an attribute
        requestDuration.Record(r.Context(),
            time.Since(start).Milliseconds(),
            attribute.String("path", r.URL.Path))
    })
}
```

### 2. 流量（Traffic）
流量衡量系统的需求。以下示例统计 HTTP 请求数量：

```go
// Create a counter for incoming requests
requestCount, _ := meter.Int64Counter("http.request.count",
    metric.WithDescription("Total HTTP requests"),
    metric.WithUnit("{requests}"))

// Middleware to count requests
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Increment the counter with method and path attributes
        requestCount.Add(r.Context(), 1,
            attribute.String("method", r.Method),
            attribute.String("path", r.URL.Path))
        next.ServeHTTP(w, r)
    })
}
```

### 3. 错误（Errors）
错误跟踪有助于识别服务中的问题。以下示例统计 HTTP 5xx 错误：

```go
// Create a counter for server errors
errorCount, _ := meter.Int64Counter("http.error.count",
    metric.WithDescription("Total HTTP errors"),
    metric.WithUnit("{errors}"))

// Middleware to track errors
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // Use a custom ResponseWriter to capture the status code
        sw := &statusWriter{ResponseWriter: w}
        next.ServeHTTP(sw, r)
        
        // Count 5xx errors
        if sw.status >= 500 {
            errorCount.Add(r.Context(), 1,
                attribute.Int("status_code", sw.status),
                attribute.String("path", r.URL.Path))
        }
    })
}
```

### 4. 饱和度（Saturation）
饱和度衡量服务“有多满”。以下示例监控系统资源：

```go
// Create gauges for CPU and memory usage
cpuUsage, _ := meter.Float64ObservableGauge("system.cpu.usage",
    metric.WithDescription("CPU usage percentage"),
    metric.WithUnit("1"))

memoryUsage, _ := meter.Int64ObservableGauge("system.memory.usage",
    metric.WithDescription("Memory usage bytes"),
    metric.WithUnit("By"))

// Start a goroutine to periodically collect system metrics
go func() {
    ticker := time.NewTicker(time.Second)
    for range ticker.C {
        ctx := context.Background()
        
        // Update CPU usage
        var cpu float64
        cpuUsage.Observe(ctx, getCPUUsage())
        
        // Update memory usage using runtime statistics
        var mem runtime.MemStats
        runtime.ReadMemStats(&mem)
        memoryUsage.Observe(ctx, int64(mem.Alloc))
    }
}()
```

## 度量导出器

在完成代码的度量采集后，你需要将度量导出到监控系统。以下是常见导出器的示例：

### Prometheus
Prometheus 是常用的度量采集方案。如下配置方式：

```go
// Create a Prometheus exporter with custom histogram boundaries
exporter, err := prometheus.New(prometheus.Config{
    DefaultHistogramBoundaries: []float64{
        1, 2, 5, 10, 20, 50, 100, 200, 500, 1000, // in milliseconds
    },
})
```

直方图边界对精确测量延迟至关重要。选择能够覆盖预期延迟范围的边界。

### OpenTelemetry 协议（OTLP）
OTLP 是 OpenTelemetry 的原生协议。使用它将度量发送到采集器：

```go
// Create an OTLP exporter connecting to a collector
exporter, err := otlpmetricgrpc.New(ctx,
    otlpmetricgrpc.WithEndpoint("collector:4317"),
    otlpmetricgrpc.WithTLSCredentials(insecure.NewCredentials()))
```

在生产环境中请正确配置 TLS。

## 最佳实践

### 1. 命名约定
遵循一致的模式使度量更易发现和理解：
```
<namespace>.<type>.<name>
```

例如：
- `http.request.duration` - HTTP 请求延迟
- `database.connection.count` - 数据库连接数
- `order.processing.time` - 订单处理时长

该模式有助于用户在不查阅文档的情况下找到并理解度量。

### 2. 单位
始终在度量描述中标注单位以避免歧义：
- 时间：`ms`（毫秒）、`s`（秒）
- 字节：`By`（字节）
- 计数：`{requests}`、`{errors}`
- 比率：`1`（无量纲）

使用一致的单位能让度量可比较并避免换算错误。

### 3. 性能
为保持良好性能，请考虑以下因素：

- **采集间隔**：根据度量波动性选择合适的采集频率
  - 高频变化：1-5 秒
  - 稳定度量：15-60 秒
  - 资源开销较大：5 分钟以上

- **批量更新**：尽可能将度量更新归并
  ```go
  // Instead of this:
  counter.Add(ctx, 1)
  counter.Add(ctx, 1)
  
  // Do this:
  counter.Add(ctx, 2)
  ```

- **基数增长**：监控唯一时间序列的数量
  - 限制属性组合数量
  - 定期审查并清理未使用的度量
  - 对高基数度量使用记录规则

- **聚合**：对高流量度量进行预聚合
  ```go
  // Instead of recording every request:
  histogram.Record(ctx, duration)
  
  // Batch and record summaries:
  type window struct {
      count int64
      sum   float64
  }
  ```

### 4. 文档
为每个度量编写详尽文档，帮助用户理解并有效使用：

文档应包含：
- **清晰描述**：该度量测量什么以及其重要性
- **度量单位**：使用的具体单位（如毫秒、字节）
- **属性有效值**：每个属性的预期取值范围
- **更新频率**：该度量的更新频率
- **保留周期**：度量数据的保留时长

示例文档：
```go
// http.request.duration measures the time taken to process HTTP requests.
// Unit: milliseconds
// Attributes:
//   - method: HTTP method (GET, POST, etc.)
//   - path: Request path
//   - status_code: HTTP status code
// Update frequency: Per request
// Retention: 30 days
requestDuration, _ := meter.Float64Histogram(
    "http.request.duration",
    metric.WithDescription("Time taken to process HTTP requests"),
    metric.WithUnit("ms"))
```

## 进一步了解

关于度量的更多详细信息：

- [OpenTelemetry Metrics](https://opentelemetry.io/docs/concepts/signals/metrics/)
  OpenTelemetry 度量的官方概念与实现指南。

- [Metric Semantic Conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/)
  常用度量的标准命名与属性。

- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)
  关于度量命名和标签的优秀实践指南。

- [Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/)
  Google 关于关键服务度量的指南。

这些资源可帮助你深入了解度量的实现与最佳实践。

### 选择属性

属性为度量提供上下文，使其更有助于分析。但选择合适的属性需要谨慎，以避免性能问题并保持数据质量。

推荐包含的属性：
- **高基数**：`customer_type`、`order_status`、`error_code`
  这些属性的可能取值有限，能够提供有意义的分组。
- **业务相关**：`subscription_tier`、`payment_method`
  有助于将度量与业务结果相关联。
- **技术分组**：`region`、`datacenter`、`instance_type`
  便于运维分析和故障排查。

应避免的属性：
- **唯一 ID**：不要使用 `user_id`、`order_id`（应在追踪中使用）
  这些属性会产生过多的唯一时间序列，可能使度量存储不堪重负。
- **时间戳**：度量本身已经带有时间戳
  额外添加时间戳属性是冗余的，并浪费存储。
- **敏感数据**：切勿包含 PII 或密钥
  度量在组织内部通常被广泛访问。
