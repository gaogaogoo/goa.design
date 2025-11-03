---
title: "可观测性"
description: "理解并在 Goa 服务中实现可观测性"
weight: 2
---

现代分布式系统十分复杂。当出现问题时，单靠传统日志不足以理解发生了什么。你需要看到请求如何在系统中流动、衡量性能并监控系统健康状况。这就是可观测性的意义所在。

{{< alert title="注意" color="primary" >}}
Goa 服务是标准的 HTTP 或 gRPC 服务，因此你可以使用任何你偏好的可观测性技术栈。尽管本指南聚焦于 [Clue](https://github.com/goadesign/clue)（Goa 在生成的示例中使用并提供 Goa 特定功能），其原则适用于任何可观测性方案。
{{< /alert >}}

## 什么是可观测性？

可观测性是指你通过观察系统的输出，理解系统内部正在发生什么的能力。在 Goa 中，我们通过三大支柱来实现这一点：

1. **分布式追踪**：随着请求穿越你的服务进行跟踪
2. **指标**：衡量系统行为和性能
3. **日志**：记录特定事件和错误

## Clue 包

Clue 是 Goa 推荐的可观测性包。它基于业界标准的 [OpenTelemetry](https://opentelemetry.io) 构建，并与 Goa 生成的代码紧密集成。

下面是一个实践中可观测性的简单示例：

```go
import (
    "go.opentelemetry.io/otel"                // 标准 OpenTelemetry
    "go.opentelemetry.io/otel/attribute"      // 标准 OpenTelemetry
    "goa.design/clue/log"                     // Clue 的日志包
)

func (s *Service) CreateOrder(ctx context.Context, order *Order) error {
    // 使用标准 OpenTelemetry API
    ctx, span := otel.Tracer("service").Start(ctx, "create_order")
    defer span.End()

    // 标准 OpenTelemetry 属性
    span.SetAttributes(
        attribute.String("order.id", order.ID),
        attribute.Float64("order.amount", order.Amount))

    // 标准 OpenTelemetry 指标
    s.orderCounter.Add(ctx, 1,
        attribute.String("type", order.Type))

    // Clue 的结构化日志（可选）
    log.Info(ctx, "processing order",
        log.KV{"order_id", order.ID})

    if err := s.processOrder(ctx, order); err != nil {
        // 标准 OpenTelemetry 错误记录
        span.RecordError(err)
        return err
    }

    return nil
}
```

请注意，大部分代码使用标准的 OpenTelemetry 包（`go.opentelemetry.io/otel/*`）。只有日志使用 Clue 特定的代码，而且也可以替换为你偏好的日志方案。这意味着你可以：
- 使用任意与 OpenTelemetry 兼容的可观测性后端
- 如有需要，切换为不同的日志库
- 保持你的可观测性代码的可移植性

## 为什么优先使用 OpenTelemetry？

Clue 采用 OpenTelemetry 优先的方式。这意味着：

1. **追踪**是你的主要调试工具。它们展示：
   - 每个请求的精确路径
   - 时间消耗的位置
   - 参与的服务
   - 发生的错误

2. **指标**帮助你监控系统健康：
   - 请求速率与延迟
   - 错误率
   - 资源使用
   - 业务指标

3. **日志**谨慎使用，主要用于：
   - 致命错误
   - 系统启动/关闭
   - 调试特定问题

这种方式比传统日志记录更易扩展，因为：
- 追踪自动提供上下文
- 指标比日志解析更高效
- 日志可以聚焦于关键事项

## 入门

要为你的 Goa 服务添加可观测性，你需要：

1. **设置 Clue**：配置带合适导出器的 OpenTelemetry
2. **添加埋点**：包装你的处理器和客户端
3. **定义指标**：跟踪重要的系统行为
4. **配置健康检查**：监控服务依赖
5. **启用调试**：添加故障排查工具

以下指南会逐步引导你完成每一步：

1. [基础设置](1-setup) - 配置 Clue 与 OpenTelemetry
2. [追踪](2-tracing) - 实现分布式追踪
3. [指标](3-metrics) - 添加服务指标
4. [日志](4-logging) - 配置日志
5. [健康检查](5-health) - 添加健康监控
6. [调试](6-debugging) - 启用调试工具

## 示例服务

下面是一个在实践中具备完整可观测性的 Goa 服务示例：

```go
func main() {
    // 1. 创建具备适当格式的 logger
    format := log.FormatJSON
    if log.IsTerminal() {
        format = log.FormatTerminal
    }
    ctx := log.Context(context.Background(),
        log.WithFormat(format),
        log.WithFunc(log.Span))

    // 2. 使用 OTLP 导出器配置 OpenTelemetry
    spanExporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint(*coladdr),
        otlptracegrpc.WithTLSCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf(ctx, err, "failed to initialize tracing")
    }
    metricExporter, err := otlpmetricgrpc.New(ctx,
        otlpmetricgrpc.WithEndpoint(*coladdr),
        otlpmetricgrpc.WithTLSCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatalf(ctx, err, "failed to initialize metrics")
    }

    // 3. 使用 OpenTelemetry 初始化 Clue
    cfg, err := clue.NewConfig(ctx,
        genservice.ServiceName,
        genservice.APIVersion,
        metricExporter,
        spanExporter)
    clue.ConfigureOpenTelemetry(ctx, cfg)

    // 4. 使用中间件创建服务
    svc := front.New(fc, lc)
    endpoints := genservice.NewEndpoints(svc)
    endpoints.Use(debug.LogPayloads())  // 调试日志
    endpoints.Use(log.Endpoint)         // 请求日志
    endpoints.Use(middleware.ErrorReporter())

    // 5. 配置带可观测性的 HTTP 处理器
    mux := goahttp.NewMuxer()
    debug.MountDebugLogEnabler(debug.Adapt(mux))  // 动态日志级别控制
    debug.MountPprofHandlers(debug.Adapt(mux))    // Go 性能分析端点
    
    // 按正确顺序添加中间件：
    mux.Use(otelhttp.NewMiddleware(serviceName)) // 3. OpenTelemetry
    mux.Use(debug.HTTP())                        // 2. 调试端点
    mux.Use(log.HTTP(ctx))                       // 1. 请求日志

    // 6. 在独立端口挂载健康检查
    check := health.Handler(health.NewChecker(
        health.NewPinger("locator", *locatorHealthAddr),
        health.NewPinger("forecaster", *forecasterHealthAddr)))
    http.Handle("/healthz", log.HTTP(ctx)(check))

    // 7. 启动服务器并支持优雅关闭
    var wg sync.WaitGroup
    wg.Add(1)
    go func() {
        defer wg.Done()
        log.Printf(ctx, "HTTP server listening on %s", *httpAddr)
        if err := server.ListenAndServe(); err != http.ErrServerClosed {
            log.Errorf(ctx, err, "server error")
        }
    }()

    // 处理关闭
    <-ctx.Done()
    if err := server.Shutdown(context.Background()); err != nil {
        log.Errorf(ctx, err, "shutdown error")
    }
    wg.Wait()
}
```

该服务展示了多项重要的可观测性能力，帮助在生产环境中监控和调试应用。它实现了可携带上下文的结构化日志，使请求能够在组件间被追踪。服务集成了 OpenTelemetry 用于分布式追踪与指标采集，提供有关性能与行为的洞察。健康检查端点监控定位器与预报器等依赖的状态；调试端点则支持对运行中的服务进行性能剖析以识别瓶颈。服务还支持动态日志级别控制，可在运行时无需重启调整详细程度。最后，它实现了优雅关闭，在停止服务时妥善清理资源并完成正在处理的请求。

## 了解更多

- [OpenTelemetry 文档](https://opentelemetry.io/docs/)
- [Clue GitHub 仓库](https://github.com/goadesign/clue)
- [Clue Weather 示例](https://github.com/goadesign/clue/tree/main/example/weather)

