title: "基础设置"
description: "配置 Clue 与 OpenTelemetry"
weight: 1
---

在 Goa 服务中启用可观测性需要配置 Clue 和 OpenTelemetry。本文将带你完成关键的设置步骤。

## 前置条件

首先，在你的 `go.mod` 中添加所需依赖：

```go
require (
	goa.design/clue
	go.opentelemetry.io/otel 
	go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc 
	go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc 
	go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp 
	go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc
)
```

这些包提供：
- `clue`：Goa 的可观测性工具包
- `otel`：OpenTelemetry 核心功能
- `otlpmetricgrpc` 与 `otlptracegrpc`：用于发送遥测数据的 OTLP 导出器
- `otelhttp` 与 `otelgrpc`：HTTP 与 gRPC 的自动埋点

## 1. 日志上下文（Logger Context）

日志上下文是可观测性设置的基础。它在应用中传递配置与关联 ID：

```go
// Configure logger format based on environment
// 根据环境配置日志格式
format := log.FormatJSON
if log.IsTerminal() {
	format = log.FormatTerminal  // 开发环境下更易读的格式
}

// Create base context with formatting and span tracking
// 创建带格式与 span 跟踪的基础上下文
ctx := log.Context(context.Background(),
	log.WithFormat(format),      // 设置输出格式
	log.WithFunc(log.Span))      // 在日志中包含 trace/span ID

// Enable debug logging if needed
// 如有需要，启用调试日志
if *debugf {
	ctx = log.Context(ctx, log.WithDebug())
	log.Debugf(ctx, "debug logs enabled")
}

// Add service information
// 添加服务信息
ctx = log.With(ctx, 
	log.KV{"service", serviceName},
	log.KV{"version", version},
	log.KV{"env", environment})
```

日志上下文提供：
- 服务内一致的结构化日志
- 日志与追踪的自动关联
- 环境感知的格式（生产为 JSON，开发更易读）
- 调试日志级别控制
- 所有日志项的通用字段

## 2. OpenTelemetry 配置

OpenTelemetry 的设置包括创建导出器并配置全局提供者：

```go
// Create OTLP exporters for sending telemetry to a collector
// 创建 OTLP 导出器以将遥测数据发送到采集器
spanExporter, err := otlptracegrpc.New(ctx,
	otlptracegrpc.WithEndpoint(*coladdr),
	otlptracegrpc.WithTLSCredentials(insecure.NewCredentials()))
if err != nil {
	log.Fatalf(ctx, err, "failed to initialize tracing")
}
defer func() {
	ctx := log.Context(context.Background())
	if err := spanExporter.Shutdown(ctx); err != nil {
		log.Errorf(ctx, err, "failed to shutdown tracing")
	}
}()

metricExporter, err := otlpmetricgrpc.New(ctx,
	otlpmetricgrpc.WithEndpoint(*coladdr),
	otlpmetricgrpc.WithTLSCredentials(insecure.NewCredentials()))
if err != nil {
	log.Fatalf(ctx, err, "failed to initialize metrics")
}
defer func() {
	ctx := log.Context(context.Background())
	if err := metricExporter.Shutdown(ctx); err != nil {
		log.Errorf(ctx, err, "failed to shutdown metrics")
	}
}()

// Initialize Clue with the exporters
// 使用导出器初始化 Clue
cfg, err := clue.NewConfig(ctx,
	serviceName,
	version,
	metricExporter,
	spanExporter,
	clue.WithResourceAttributes(map[string]string{
		"environment": environment,
		"region":     region,
	}))
if err != nil {
	log.Fatalf(ctx, err, "failed to initialize observability")
}
clue.ConfigureOpenTelemetry(ctx, cfg)
```

上述配置为你的服务搭建了核心的 OpenTelemetry 基础设施。它创建导出器将遥测数据发送到采集器以进行处理与存储；同时确保正确的关闭流程，避免服务终止时数据丢失。通过添加如环境、区域等资源属性，有助于组织与筛选遥测数据。最后，初始化全局 OpenTelemetry 提供者，使整个应用具备追踪与指标采集能力。

## 3. HTTP 与 gRPC 配置

对于 HTTP 服务，使用可观测性中间件包装你的处理器：

```go
// Create Goa HTTP muxer
// 创建 Goa HTTP 多路复用器
mux := goahttp.NewMuxer()

// Mount debug endpoints
// 挂载调试端点
debug.MountDebugLogEnabler(debug.Adapt(mux))  // 动态日志级别控制
debug.MountPprofHandlers(debug.Adapt(mux))    // Go 性能分析端点

// Add middleware in correct order (inside to out):
// 按正确顺序（由内到外）添加中间件：
mux.Use(otelhttp.NewMiddleware(serviceName)) // 3. OpenTelemetry
mux.Use(debug.HTTP())                        // 2. 调试端点
mux.Use(log.HTTP(ctx))                       // 1. 请求日志

// Create server with the instrumented handler
// 创建带埋点的处理器并启动服务器
server := &http.Server{
	Addr:         *httpAddr,
	Handler:      mux,
	ReadTimeout:  15 * time.Second,
	WriteTimeout: 15 * time.Second,
}
```

对于 gRPC 服务，使用拦截器：

```go
// Create gRPC client connection with observability
// 创建带可观测性的 gRPC 客户端连接
conn, err := grpc.DialContext(ctx, *serverAddr,
	grpc.WithTransportCredentials(insecure.NewCredentials()),
	grpc.WithUnaryInterceptor(log.UnaryClientInterceptor()),
	grpc.WithStatsHandler(otelgrpc.NewClientHandler()))

// Create gRPC server with observability
// 创建带可观测性的 gRPC 服务器
srv := grpc.NewServer(
	grpc.UnaryInterceptor(log.UnaryServerInterceptor()),
	grpc.StatsHandler(otelgrpc.NewServerHandler()))
```

这些中间件/拦截器提供：
- 所有请求的分布式追踪
- 请求/响应日志
- 动态日志级别控制
- 性能分析端点

## 4. 健康检查（Health Checks）

健康检查用于监控服务及其依赖。Clue 提供两类核心接口用以实现健康检查：

### Pinger 接口

`Pinger` 接口定义了如何检查单个依赖的健康状况：

```go
type Pinger interface {
    // Name returns the name of the remote service
    // 返回远程服务名称
    Name() string
    
    // Ping checks if the service is healthy
    // 检查服务是否健康
    Ping(context.Context) error
}
```

Clue 提供了基于 HTTP 的默认实现以探测健康检查端点：

```go
// Create a pinger for a database service
// 为数据库服务创建 pinger
dbPinger := health.NewPinger("database", "db:8080",
    health.WithScheme("https"),           // Use HTTPS (default: http)
    health.WithPath("/health"))           // 自定义路径（默认：/livez）

// Create a pinger for Redis
// 为 Redis 创建 pinger
redisPinger := health.NewPinger("redis", "redis:6379",
    health.WithPath("/ping"))             // Redis 健康端点
```

你也可以为特殊场景实现自定义 pinger：

```go
type CustomPinger struct {
    name string
    db   *sql.DB
}

func (p *CustomPinger) Name() string { return p.name }

func (p *CustomPinger) Ping(ctx context.Context) error {
    return p.db.PingContext(ctx)
}
```

### Checker 接口

`Checker` 接口聚合多个 pinger，并提供整体健康状态：

```go
type Checker interface {
    // Check returns the health status of all dependencies
    // 返回所有依赖的健康状态
    Check(context.Context) (*Health, bool)
}

// Health contains detailed status information
// Health 包含详细的状态信息
type Health struct {
    Uptime  int64             // Service uptime in seconds
    Version string            // Service version
    Status  map[string]string // Status of each dependency
}
```

创建包含多个依赖的 checker：

```go
// Create health checker with multiple pingers
// 创建包含多个 pinger 的健康检查器
checker := health.NewChecker(
    health.NewPinger("database", *dbAddr),
    health.NewPinger("cache", *cacheAddr),
    health.NewPinger("search", *searchAddr),
    &CustomPinger{name: "custom", db: db},
)

// Create HTTP handler from checker
// 从 checker 创建 HTTP 处理器
check := health.Handler(checker)

// Add logging to health checks
// 为健康检查添加日志
check = log.HTTP(ctx)(check).(http.HandlerFunc)

// Mount health endpoints (often on separate port)
// 挂载健康端点（通常使用独立端口）
http.Handle("/healthz", check)  // Kubernetes 存活探针
http.Handle("/livez", check)    // Kubernetes 就绪探针
```

### 健康检查响应

健康检查端点返回包含详细状态的 JSON 响应：

```json
{
    "uptime": 3600,           // 服务启动以来的秒数
    "version": "1.0.0",       // 服务版本
    "status": {               // 每个依赖的状态
        "database": "OK",
        "cache": "OK",
        "search": "NOT OK"    // 失败的依赖项
    }
}
```

响应码：
- 当所有依赖都健康时返回 `200`
- 当任一依赖不健康时返回 `503`

### 最佳实践

1. **独立端口**：在不同端口运行健康检查，以：
   - 防止与应用流量互相影响
   - 防止与其他可观测性组件互相影响
   - 允许不同的安全策略

```go
// Create main application server
// 创建主应用服务器
appServer := &http.Server{
    Addr:    *httpAddr,
    Handler: appHandler,
}

// Create health check server on different port
// 在不同端口创建健康检查服务器
healthServer := &http.Server{
    Addr:    *healthAddr,
    Handler: check,
}
```

2. **超时处理**：配置合适的超时：

```go
// Create pinger with custom client
// 使用自定义客户端创建 pinger
client := &http.Client{Timeout: 5 * time.Second}
pinger := health.NewPinger("service", *addr,
    health.WithClient(client))
```

3. **错误处理**：带上下文记录健康检查失败：

```go
check = log.HTTP(ctx, 
    log.With(ctx, log.KV{"component", "health"}))(check)
```

4. **Kubernetes 集成**：在部署中配置探针：

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: health-port
  initialDelaySeconds: 10
  periodSeconds: 5

readinessProbe:
  httpGet:
    path: /livez
    port: health-port
  initialDelaySeconds: 5
  periodSeconds: 2
```

健康检查是服务可观测性的关键组成部分。它们持续监控服务依赖的状态，确保在依赖出现不健康时能快速告警。健康检查与 Kubernetes 探针无缝集成，使容器编排器能基于探针结果做出合理的 Pod 生命周期管理决策。当出现问题时，健康检查也会适当记录错误，提供有价值的调试信息。

## 5. 优雅关闭（Graceful Shutdown）

实现正确的关闭流程以确保服务正常终止：

```go
// Create shutdown channel
// 创建关闭通道
errc := make(chan error)
go func() {
	c := make(chan os.Signal, 1)
	signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
	errc <- fmt.Errorf("signal: %s", <-c)
}()

// Start servers
// 启动服务器
var wg sync.WaitGroup
wg.Add(1)
go func() {
	defer wg.Done()
	log.Printf(ctx, "HTTP server listening on %s", *httpAddr)
	if err := server.ListenAndServe(); err != http.ErrServerClosed {
		errc <- err
	}
}()

// Wait for shutdown signal
// 等待关闭信号
if err := <-errc; err != nil {
	log.Errorf(ctx, err, "shutdown initiated")
}

// Graceful shutdown
// 优雅关闭
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

if err := server.Shutdown(ctx); err != nil {
	log.Errorf(ctx, err, "shutdown error")
}
wg.Wait()
```

正确的关闭流程确保：
- 在途请求得到完成
- 资源被清理
- 遥测数据被刷新
- 依赖被通知

## 配置选项

### 采样率

控制收集的追踪数据量。Clue 提供两种采样策略：

#### 固定比率采样

使用固定比例的请求：

```go
cfg := clue.NewConfig(ctx,
    serviceName,
    version,
    metricExporter,
    spanExporter,
    clue.WithSamplingRate(0.1))  // 采样 10% 的请求
```

#### 自适应采样

如需更动态的控制，使用自适应采样器：它会自动调整采样率以维持目标每秒请求数（RPS）：

```go
cfg := clue.NewConfig(ctx,
    serviceName,
    version,
    metricExporter,
    spanExporter,
    clue.WithSampler(
        clue.AdaptiveSampler(
            100,    // 目标每秒 100 条追踪
            1000))) // 每 1000 个请求调整一次比率
```

自适应采样器的优势：
- 根据流量动态调整采样率
- 在高负载时防止追踪数据爆炸
- 在低流量时保持一致采样
- 提供可预期的存储与处理成本

例如：
- 在低流量（50 rps）时，可能采样 100% 的请求
- 在常规流量（200 rps）时，可能采样 50% 的请求
- 在高流量（1000 rps）时，可能采样 10% 的请求

这种自适应采样方式确保你不会超过目标采样率，同时保持最佳可见性。在空闲期间，由于可以采样所有请求，你能获得系统行为的完整可见性；在高峰负载时，采样会自动调整以保持成本可控，同时仍提供具有统计意义的数据用于分析。

### 资源属性（Resource Attributes）

为所有遥测数据添加元数据：

```go
cfg := clue.NewConfig(ctx,
	serviceName,
	version,
	metricExporter,
	spanExporter,
	clue.WithResourceAttributes(map[string]string{
		"environment": "production",
		"region":     "us-west",
		"deployment": "blue",
	}))
```

### 替代导出器（Alternative Exporters）

OpenTelemetry 可以将遥测数据发送到不同的后端（用于存储与可视化可观测性数据的系统）。虽然前述示例使用 OTLP（OpenTelemetry 协议）导出器，你也可以使用其他常用系统：

#### 什么是导出器？

导出器是将遥测数据（追踪、指标、日志）发送至后端系统进行存储与分析的组件。可以将其理解为适配器，将 OpenTelemetry 数据转换为特定后端可理解的格式。

#### 常见后端

1. **Jaeger** - 流行的开源分布式追踪系统：
   ```go
   import "go.opentelemetry.io/otel/exporters/jaeger"

   // Send traces directly to Jaeger
   // 将追踪直接发送到 Jaeger
   spanExporter, err := jaeger.New(
       jaeger.WithCollectorEndpoint(
           jaeger.WithEndpoint("http://jaeger:14268/api/traces")))
   ```

2. **Prometheus** - 业界标准的指标采集系统：
   ```go
   import "go.opentelemetry.io/otel/exporters/prometheus"

   // Export metrics in Prometheus format
   // 以 Prometheus 格式导出指标
   metricExporter, err := prometheus.New(
       prometheus.WithNamespace("myapp"))    // Prefix all metrics
   ```

3. **Zipkin** - 另一种分布式追踪系统：
   ```go
   import "go.opentelemetry.io/otel/exporters/zipkin"

   // Send traces to Zipkin
   // 将追踪发送到 Zipkin
   spanExporter, err := zipkin.New(
       "http://zipkin:9411/api/v2/spans")
   ```

#### 同时使用多个导出器

你可以同时将数据发送到多个后端：

```go
// Create exporters
// 创建导出器
jaegerExp, err := jaeger.New(jaegerEndpoint)
prometheusExp, err := prometheus.New()
otlpExp, err := otlp.New(otlpEndpoint)

// Configure with multiple exporters
// 使用多个导出器进行配置
cfg, err := clue.NewConfig(ctx,
    serviceName,
    version,
    prometheusExp,        // 指标发送到 Prometheus
    otlpExp,              // 追踪发送到 OTLP
    clue.WithTraceExporter(jaegerExp))  // 同时将追踪发送到 Jaeger
```

同时使用多个导出器有诸多好处。你可以利用各类专精工具满足不同的可观测性需求——例如将指标交给 Prometheus、将追踪发送到 Jaeger。这也让你能够并行比较不同后端的能力，以评估最适合你的方案。此外，当需要在不同可观测性系统之间迁移时，可以通过在过渡期并行运行两套系统来实现平滑迁移。

## 后续步骤

现在你已经完成基础的可观测性设置，继续探索：
- [追踪](../2-tracing) - 添加分布式追踪
- [指标](../3-metrics) - 实现服务指标
- [日志](../4-logging) - 配置日志策略
- [健康检查](../5-health) - 监控依赖
- [调试](../6-debugging) - 启用调试工具