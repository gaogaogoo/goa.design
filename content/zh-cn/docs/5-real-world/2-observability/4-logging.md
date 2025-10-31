---
title: "日志策略"
description: "使用 Clue 配置日志记录"
weight: 4
---

虽然 OpenTelemetry 是 Clue 中可观测性的主要来源，但日志在某些场景下仍然扮演着重要的角色。Clue 提供了一个智能日志系统，可以高效地缓冲和格式化日志消息，帮助您在控制成本和性能的同时保持可见性。

## 主要特性

1. **智能缓冲**:
   Clue 的智能缓冲系统有助于优化日志记录成本和性能。它会将非错误日志缓存在内存中，直到发生错误时才刷新缓冲区，以提供围绕错误的完整上下文。对于被追踪的请求，日志会自动刷新，以确保请求生命周期的完全可见性。该系统还提供手动刷新控制，以便您在特定场景下强制输出日志。为了保持灵活性，可以根据上下文配置缓冲行为，从而使日志记录模式适应不同的情况。

2. **结构化日志**:
   Clue 使用结构化日志使日志更有用且更易于维护。所有日志字段都以键值对的形式存储，确保它们可以被日志工具轻松解析和分析。所有日志的一致格式改进了与监控和分析系统的集成。日志可以根据您的环境需求以不同的格式（如 JSON 或纯文本）输出。此外，基于上下文的字段会自动包含进来，以将日志与请求追踪相关联，从而更容易地在分布式系统中调试问题。

3. **性能**:
   Clue 的日志系统在设计时就考虑到了性能。它使用高效的缓冲技术来最小化 I/O 开销，并采用智能内存管理来降低分配开销。该系统可以根据您的环境需求配置为以不同方式输出日志。您还可以通过条件日志记录来控制日志量，确保只生成您需要的日志。

## 基本设置

记录器在使用前需要进行配置。以下是如何使用常用选项设置记录器：

```go
// 使用选项创建记录器上下文
ctx := log.Context(context.Background(),
    // 在日志中包含 span ID 以便与追踪关联
    log.WithFunc(log.Span),

    // 使用 JSON 格式以便机器读取
    log.WithFormat(log.FormatJSON),

    // 将日志发送到标准输出
    log.WithOutput(os.Stdout),

    // 在请求被追踪时禁用缓冲
    log.WithDisableBuffering(log.IsTracing))

// 添加将包含在所有日志中的通用字段
ctx = log.With(ctx,
    log.KV{"service", "myservice"},  // 用于筛选的服务名称
    log.KV{"env", "production"})     // 用于上下文的环境
```

此配置为您的服务建立了强大的日志记录基础。通过在日志中包含 span ID，您可以轻松地将日志条目与分布式追踪相关联，从而全面了解系统中的请求流。JSON 格式确保您的日志可以被日志聚合和分析工具高效处理。

为方便本地开发，日志被定向到标准输出，可以在终端中轻松查看。智能缓冲系统会根据请求是否被追踪自动调整，从而优化性能和可观测性。

最后，该设置包括将添加到每个日志条目中的通用字段，为筛选和分析提供一致的上下文。这些字段（如服务名称和环境）使在调查问题时可以轻松识别每个日志条目的来源和上下文。

## 日志级别

Clue 支持四个严重性级别，每个级别都有特定的用途并遵循不同的缓冲规则：

```go
// 调试级别 - 用于详细的故障排除
// 仅在通过 WithDebug 启用调试模式时发出
log.Debug(ctx, "request details",
    log.KV{"headers", req.Headers},
    log.KV{"body_size", len(req.Body)})

// 信息级别 - 用于正常操作
// 这些日志默认被缓冲并在出错时刷新
log.Info(ctx, "processing request",
    log.KV{"requestID", req.ID},
    log.KV{"method", req.Method})

// 警告级别 - 用于潜在问题
// 这些日志表示不影响操作的问题
log.Warn(ctx, "resource usage high",
    log.KV{"cpu_usage", cpuUsage},
    log.KV{"memory_usage", memUsage})

// 错误级别 - 用于失败情况
// 这些日志会立即写入并刷新缓冲区
log.Error(ctx, err, "request failed",
    log.KV{"requestID", req.ID},
    log.KV{"status", http.StatusInternalServerError})

// 致命级别 - 用于不可恢复的错误
// 这些日志记录后会导致程序退出
log.Fatal(ctx, err, "cannot start server",
    log.KV{"port", config.Port},
    log.KV{"error", err.Error()})
```

每个级别还有一个对应的格式化版本，接受 printf 风格的格式化：

```go
// 带格式化的调试
log.Debugf(ctx, "processing item %d of %d", current, total)

// 带格式化的信息
log.Infof(ctx, "request completed in %dms", duration.Milliseconds())

// 带格式化的警告
log.Warnf(ctx, "high latency detected: %dms", latency.Milliseconds())

// 带格式化和错误对象的错误
log.Errorf(ctx, err, "failed to process request: %s", req.ID)

// 带格式化和错误对象的致命错误
log.Fatalf(ctx, err, "failed to initialize: %s", component)
```

日志级别的最佳实践：

1. **DEBUG** (SeverityDebug):
   - 用于详细的故障排除信息
   - 仅在启用调试模式时发出
   - 非常适合开发和调试会话
   - 可以包含详细的请求/响应数据

2. **INFO** (SeverityInfo):
   - 用于需要审计的正常操作
   - 默认缓冲以优化性能
   - 记录重要但预期的事件
   - 包含与业务相关的信息

3. **WARN** (SeverityWarn):
   - 用于潜在的有害情况
   - 表示不影响操作的问题
   - 突出显示接近资源限制
   - 标记已弃用的功能使用

4. **ERROR** (SeverityError):
   - 用于任何需要注意的错误情况
   - 自动刷新日志缓冲区
   - 包含错误详细信息和上下文
   - 可用时添加堆栈跟踪

5. **FATAL** (SeverityError + Exit):
   - 仅用于不可恢复的错误
   - 导致程序以状态 1 退出
   - 包含所有相关的上下文以供事后分析
   - 谨慎使用 - 大多数错误应该是可恢复的

特殊行为：
- 调试日志仅在启用调试模式时发出
- 信息日志默认被缓冲
- 警告日志表示需要注意的问题
- 错误日志刷新缓冲区并禁用缓冲
- 致命日志类似于错误，但还会退出程序

颜色编码（使用终端格式时）：
- Debug: 灰色 (37m)
- Info: 蓝色 (34m)
- Warn: 黄色 (33m)
- Error/Fatal: 亮红色 (1;31m)

## 结构化日志

结构化日志使解析和分析日志更容易。以下是构建日志的不同方法：

```go
// 使用 log.KV 表示有序的键值对
// 这是首选方法，因为它能保持字段顺序
log.Print(ctx,
    log.KV{"action", "user_login"},      // 发生了什么
    log.KV{"user_id", user.ID},          // 发生在谁身上
    log.KV{"ip", req.RemoteAddr},        // 附加的上下文
    log.KV{"duration_ms", duration.Milliseconds()})  // 性能数据

// 使用 log.Fields 进行映射式日志记录
// 在处理现有数据映射时很有用
log.Print(ctx, log.Fields{
    "action":     "user_login",
    "user_id":    user.ID,
    "ip":         req.RemoteAddr,
    "duration_ms": duration.Milliseconds(),
})

// 添加将包含在所有后续日志中的上下文字段
// 对请求范围的信息很有用
ctx = log.With(ctx,
    log.KV{"tenant", tenant.ID},     // 多租户上下文
    log.KV{"region", "us-west"},     // 地理上下文
    log.KV{"request_id", reqID})     // 请求关联
```

结构化日志的最佳实践：
- 在整个应用程序中使用一致的字段名称
- 包含相关上下文，但不要信息过载
- 逻辑上对相关字段进行分组
- 在选择字段名称时考虑日志解析和分析

## 输出格式

选择与您的环境和工具相匹配的输出格式：

```go
// 纯文本格式 (logfmt)
// 最适合本地开发和人类可读性
ctx := log.Context(context.Background(),
    log.WithFormat(log.FormatText))
// 输出: time=2024-02-24T12:34:56Z level=info msg="hello world"

// 带颜色的终端格式
// 非常适合本地开发和调试
ctx := log.Context(context.Background(),
    log.WithFormat(log.FormatTerminal))
// 输出: INFO[0000] msg="hello world"

// JSON 格式
// 最适合生产和日志聚合系统
ctx := log.Context(context.Background(),
    log.WithFormat(log.FormatJSON))
// 输出: {"time":"2024-02-24T12:34:56Z","level":"info","msg":"hello world"}

// 自定义格式
// 当您需要特殊格式时使用
ctx := log.Context(context.Background(),
    log.WithFormat(func(entry *log.Entry) []byte {
        return []byte(fmt.Sprintf("[%s] %s: %s
",
            entry.Time.Format(time.RFC3339),
            entry.Level,
            entry.Message))
    }))
```

选择日志格式时，请仔细考虑您的环境和要求。开发环境通常受益于带颜色和格式的人类可读格式，而生产部署通常需要像 JSON 这样的机器可解析格式用于日志聚合系统。您选择的格式应在人类可读性与日志处理管道的需求之间取得平衡。此外，还要考虑所选格式的性能影响——虽然 JSON 提供了丰富的结构，但与简单的文本格式相比，它有更多的处理开销。

## HTTP 中间件

将日志记录添加到 HTTP 处理程序以跟踪请求和响应：

```go
// 基本的日志记录中间件
// 自动记录请求开始/结束和持续时间
mux.Use(log.HTTP(ctx))

// 带有详细日志记录的自定义中间件
func loggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        ctx := r.Context()

        // 在开始时记录请求详细信息
        log.Info(ctx, "request started",
            log.KV{"method", r.Method},
            log.KV{"path", r.URL.Path},
            log.KV{"user_agent", r.UserAgent()})

        start := time.Now()
        sw := &statusWriter{ResponseWriter: w}

        next.ServeHTTP(sw, r)

        // 记录响应详细信息和持续时间
        log.Info(ctx, "request completed",
            log.KV{"status", sw.status},
            log.KV{"duration_ms", time.Since(start).Milliseconds()},
            log.KV{"bytes_written", sw.written})
    })
}
```

该中间件为您的 HTTP 服务提供了全面的日志记录功能。它自动将请求与其相应的响应关联起来，使您能够跟踪每个请求的完整生命周期。捕获计时信息有助于识别性能瓶颈和跟踪响应延迟。该中间件还监控状态码，从而可以轻松检测和调查错误或意外响应。通过错误检测，您可以快速识别和调试服务中的问题。此外，中间件收集的性能指标为您提供了有关服务行为的宝贵见解，并帮助您优化其性能。

## gRPC 拦截器

将日志记录添加到 gRPC 服务以实现一致的可观测性：

```go
// 服务器端一元拦截器
// 记录每个 RPC 调用的方法和持续时间
svr := grpc.NewServer(
    grpc.UnaryInterceptor(log.UnaryServerInterceptor(ctx)))

// 服务器端流拦截器
// 记录流生命周期事件
svr := grpc.NewServer(
    grpc.StreamInterceptor(log.StreamServerInterceptor(ctx)))

// 客户端拦截器
// 记录出站 RPC 以进行调试
conn, err := grpc.DialContext(ctx,
    "localhost:8080",
    grpc.WithUnaryInterceptor(log.UnaryClientInterceptor()))
```

这些拦截器为您的 gRPC 服务提供了全面的日志记录功能。每个 RPC 方法名称都会自动记录，使您能够跟踪正在调用的端点。拦截器监控您的服务返回的状态码，从而可以轻松识别成功调用与失败调用。它们测量每个 RPC 调用的持续时间，帮助您了解性能特征并识别慢速请求。发生错误时，它们会自动记录相关上下文以帮助调试。拦截器还捕获与每个调用相关的元数据，提供有关 RPC 调用的附加上下文，例如身份验证令牌或关联 ID。

## 标准记录器兼容性

将 Clue 的记录器与标准库兼容的代码一起使用：

```go
// 创建一个使用 Clue 日志系统的标准记录器
logger := log.AsStdLogger(ctx)

// 使用标准日志包函数
logger.Print("hello world")               // 基本日志记录
logger.Printf("hello %s", "world")        // 格式化字符串
logger.Println("hello world")             // 带换行符

// 致命日志记录函数
logger.Fatal("fatal error")               // 记录并退出
logger.Fatalf("fatal: %v", err)           // 格式化并退出
logger.Fatalln("fatal error")             // 带换行符
```

兼容层为采用 Clue 日志记录系统的团队提供了几个重要的好处。它能够与依赖标准库记录器的现有代码库无缝集成，使团队能够在过渡到 Clue 的同时保持功能。这使得可以逐步将应用程序的不同部分迁移到结构化日志记录，而不会中断操作。

该层确保了新旧代码路径之间的一致日志处理，保持了统一的日志记录体验。所有日志，无论是来自现代组件还是传统组件，都流经相同的管道并接受相同的格式化和处理。此外，通过保持与 Go 标准库日志记录接口的兼容性，团队可以继续使用熟悉的日志记录模式，同时获得 Clue 日志记录系统的高级功能。

## Goa 集成

将 Clue 的记录器与 Goa 服务一起使用：

```go
// 创建一个与 Goa 中间件兼容的记录器
logger := log.AsGoaMiddlewareLogger(ctx)

// 在 Goa 服务中使用记录器
svc := myservice.New(logger)

// 将日志记录中间件添加到所有端点
endpoints := genmyservice.NewEndpoints(svc)
endpoints.Use(log.Endpoint)
```

将 Clue 的日志记录与 Goa 集成可为您的服务带来几个重要的好处。该集成确保了所有服务之间的一致日志记录模式，从而更容易监控和调试整个系统。通过自动上下文传播，日志在流经不同服务组件时保持与请求的关系。

该集成自动处理请求和响应的日志记录，让您了解流经 API 的数据。发生错误时，它们会自动被跟踪并记录适当的上下文和堆栈跟踪。这使得故障排除效率更高。

此外，该集成还包括内置的性能监控功能。它跟踪请求持续时间等重要指标，并帮助您识别服务中的性能瓶颈。这种全面的日志记录和监控方法有助于团队维护可靠且高性能的服务。

## 使用 Promtail、Loki 和 Grafana 进行集中式日志记录

如果您在 Kubernetes 中运行 Goa 微服务，您可能已经意识到，当您需要跨多个服务调试问题时，`kubectl logs` 已经不够用了。本指南将展示如何使用 Promtail/Loki/Grafana 堆栈设置集中式日志记录。

### 您将获得什么

- 所有服务日志集中存放
- 在数百万条日志行中快速搜索
- 能够按 `customer_id` 或 `job_id` 等自定义字段进行筛选
- Grafana 中日志和指标之间的关联

### 技术栈

我们假设您正在使用官方的 helm charts：<https://github.com/grafana/helm-charts/>

- **Promtail**: 扫描您的 pod 日志并将其发送到 Loki
- **Loki**: 存储和索引您的日志（可以看作是日志领域的 Prometheus）
- **Grafana**: 让您搜索和可视化所有内容

### 在您的 Goa 服务中设置结构化日志记录

首先，配置您的 Goa 服务以输出与 Loki 配合良好的干净 JSON 日志：

```go
// 在您的服务初始化中
logger := log.With().
    Str("service", "my-goa-service").
    Str("version", "1.0.0").
    Logger()

// 在您的端点实现中
func (s *Service) MyEndpoint(ctx context.Context, p *myapi.MyPayload) error {
    // 添加您稍后希望用于搜索的请求特定字段
    logger := log.FromContext(ctx).With().
        Str("account_id", p.AccountID).
        Str("project_id", p.ProjectID).
        Str("job_id", generateJobID()).
        Logger()

    logger.Info().Msg("processing request")

    // 您的业务逻辑在这里...

    logger.Info().Msg("request completed")
    return nil
}
```

此示例将与 Promtail、Loki 和 Grafana 开箱即用。在某些情况下，为 `account_id`、`project_id` 和 `job_id` 添加标签也是有意义的。

### 如何添加自定义标签

日志示例：

```json
{"time":"2025-06-14T20:10:56Z","level":"info","account_id":"ae02fa88-b44b-4dfc-a228-c9b5dd7f0a01","project_id":"bb7fe987-d4e7-4ed5-90a6-2107a2f8a940","job_id":"3569f33a-05af-4f93-b1b6-e01d6989fefe" "msg":"processing request"}
```

那么我们如何通过 promtail 配置来索引它们呢？

您面临的问题是，您的 kubernetes 可能不会创建 100% 的 json 日志：

```bash
tail -n1 /var/log/pods/my-app_my-service-chart-aaa-ccc-foo/chart/0.log
2025-06-14T20:10:56.382822108Z stdout F {"time":"2025-06-14T20:10:56Z","level":"info","account_id":"ae02fa88-b44b-4dfc-a228-c9b5dd7f0a01","project_id":"bb7fe987-d4e7-4ed5-90a6-2107a2f8a940","job_id":"3569f33a-05af-4f93-b1b6-e01d6989fefe" "msg":"processing request"}
```

不幸的是，这不能直接通过 promtail json 解析器处理。那么我们如何解决这个问题呢？

假设您通过以下方式安装 promtail：

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm show values grafana/promtail
helm upgrade --install --values values.yaml promtail grafana/promtail -n monitoring
```

您可以将以下部分添加到您的 `values.yml` 中

```yml
config:
  clients:
    # 'monitoring' 在这种情况下是您安装 loki 的命名空间
    # (根据您的需要进行更改)
    - url: http://loki-gateway.monitoring.svc.cluster.local/loki/api/v1/push

  snippets:
    pipelineStages:
      # 步骤 1：从 Kubernetes 日志格式中提取 JSON
      # K8s 在实际日志之前添加时间戳和流信息：
      # 2025-06-14T20:10:56.382822108Z stdout F {"actual":"json","here":true}
      - regex:
          expression: '^[^ ]+ [^ ]+ [^ ]+ (?P<json_log>{.*})$'

      # 步骤 2：解析 JSON 并提取我们的自定义字段
      - json:
          source: json_log
          expressions:
            account_id: account_id
            project_id: project_id
            job_id: job_id

      # 步骤 3：将这些添加为 Loki 标签（索引字段）
      - labels:
          account_id:
          job_id:
          project_id:
```

有关更多选项/已有的配置，请查看此文件：
<https://github.com/grafana/helm-charts/blob/main/charts/promtail/values.yaml#L438>。

关于 promtail 有很多教程，这些设置的 helm chart 集成需要一些关于 helm chart 结构的知识。

### 在 Grafana 中使用您的日志

一切运行后，您可以编写如下的 LogQL 查询：

查找特定客户的所有错误

```txt
{job="my-app/my-service"} |= "error" | account_id="ae02fa88-b44b-4dfc-a228-c9b5dd7f0a01"
```

跨所有服务跟踪一个作业

```txt
{job_id="3569f33a-05af-4f93-b1b6-e01d6989fefe"}
```

查找慢速请求

```txt
{service="api-gateway"} |= "request completed" | json | duration > 1000
```

### 重要提示：标签基数（索引的磁盘存储）

您关于磁盘大小的提示至关重要。每个唯一的标签组合都会在 Loki 中创建一个新的流。因此，如果您有：

- 1000 个客户 (account_id)
- 每个客户 100 个项目 (project_id)
- 每天 1000 个作业 (job_id)

这可能会产生 1 亿个流！请考虑将高基数字段（如 `job_id`）保留为日志行的一部分，而不是作为标签。您仍然可以对其进行过滤，只是速度会慢一些。

## 进一步阅读

*   [Clue 日志记录示例](https://github.com/go-clue/clue/tree/main/log/example)
*   [标准库日志记录接口文档](https://pkg.go.dev/log)