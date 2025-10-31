---
linkTitle: 拦截器与中间件
title: 拦截器与中间件
weight: 5
---

构建现代 API 需要在应用的不同层次处理请求。Goa 提供了一套综合方案，将类型安全的拦截器与传统中间件模式结合在一起，兼得二者优势。

## 理解不同的处理方式

在处理 Goa 服务中的请求时，你有三种互补的工具可用。每种工具都承担特定职责，并与其他工具协同，形成完整的请求处理管道。

### 类型安全拦截器的力量

Goa 设计的核心是其独特的拦截器系统。与传统中间件不同，Goa 拦截器在编译期就能安全地访问服务的领域类型。对比传统中间件与 Goa 拦截器的差异便一目了然：

```go
// 传统中间件通常基于原始字节或 interface{} 工作，
// 容易出错且需要进行类型断言
func middleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 希望请求体就是你期望的！
        data := parseBody(r)
        // 需要类型断言和错误处理
    })
}

// Goa 拦截器可对领域类型进行类型安全访问，
// 具备编译期检查与生成的辅助方法
func (i *Interceptor) Process(ctx context.Context, info *ProcessInfo, next goa.Endpoint) (any, error) {
    // 直接访问已类型化的负载字段
    amount := info.Amount()
    if amount > 1000 {
        // 使用生成的错误构造函数
        return nil, goa.MakeInvalidAmount(fmt.Errorf("Amount exceeds maximum"))
    }
    // 在类型安全下继续处理
}
```

### 传输相关的中间件

尽管 Goa 拦截器负责业务逻辑，你仍需处理传输层面的关注点。为此，Goa 与标准的 Go 中间件模式无缝集成：

1. [HTTP 中间件](./2-http-middleware) 使用标准 `http.Handler` 模式，处理压缩、CORS、会话管理等 HTTP 相关任务。

2. [gRPC 拦截器](./3-grpc-interceptors) 使用标准 gRPC 模式，处理如流式操作与元数据管理等 RPC 相关需求。

## 组合这三种方式

下面以支付处理服务的真实案例，展示三种组件如何协同工作。每一层专注其最擅长的部分，实现清晰的关注点分离。

首先，设置 HTTP 中间件以处理协议层面的关注点：

```go
func main() {
    // 创建基础 HTTP 复用器
    mux := goahttp.NewMuxer()
    
    // 从内到外构建中间件链

    // 通过 OpenTelemetry 增加可观测性
    mux.Use(otelhttp.NewMiddleware("payment-svc"))
    
    // 启用调试工具与日志
    mux.Use(debug.HTTP())
    mux.Use(log.HTTP(ctx))
    
    // 挂载运行时控制的调试端点
    debug.MountDebugLogEnabler(debug.Adapt(mux))
    debug.MountPprofHandlers(debug.Adapt(mux))
}
```

接着，使用 Goa 拦截器定义业务逻辑，提供类型安全的校验与处理：

```go
var _ = Service("payment", func() {
    // 定义一个类型安全的支付校验器
    var ValidatePayment = Interceptor("ValidatePayment", func() {
        Description("校验支付详情")
        
        // 指定要访问的负载字段
        ReadPayload(func() {
            Attribute("amount")
            Attribute("currency")
        })
        
        // 定义可能的校验错误
        Error("invalid_amount")
        Error("unsupported_currency")
    })
    
    Method("process", func() {
        // 将校验器应用到该方法
        ServerInterceptor(ValidatePayment)
        
        // 定义方法的负载
        Payload(func() {
            Attribute("amount", Int)
            Attribute("currency", String)
            Required("amount", "currency")
        })
    })
})
```

最后，设置 gRPC 拦截器处理 RPC 特定的关注点：

```go
func setupGRPC() *grpc.Server {
    return grpc.NewServer(
        grpc.UnaryInterceptor(
            grpc_middleware.ChainUnaryServer(
                // 添加 RPC 层特性
                grpc_recovery.UnaryServerInterceptor(),   // Panic 恢复
                grpc_prometheus.UnaryServerInterceptor,   // 指标
                grpc_ctxtags.UnaryServerInterceptor(),   // 上下文打标
            ),
        ),
    )
}
```

## 中间件链

要理解这些组件如何协同工作，需先了解它们的执行顺序。Goa 通过谨慎排序的处理链路来最大化各层的效益。

### 执行顺序

1. 传输相关中间件最先运行，处理诸如请求追踪与日志等协议级别关注点。这确保我们在请求处理开始时就拥有良好的可观测性。

2. 随后运行 Goa 拦截器，提供对领域类型的类型安全访问，用于业务级别的校验与转换。

3. 最后执行服务逻辑，接收已完全校验与转换后的数据。

响应沿着反向路径返回，使每一层都能适当地处理响应。以下是典型支付处理流程中的示意图：

```
请求处理：
─────────────────────────────────────────────────────────────────────────────>
OpenTelemetry → 调试/日志 → 业务校验 → 限流 → 支付

响应处理：
<─────────────────────────────────────────────────────────────────────────────
支付 → 限流 → 业务校验 → 响应日志 → 追踪
```

这种分层方式具备以下优势：

1. 可观测性包裹所有操作，提供对请求处理的完整可见性。

2. 在需要时可启用调试工具，更易诊断问题。

3. 业务校验在类型安全下进行，减少错误并提升可维护性。

4. 每一层专注其特定职责，代码更清晰且更易维护。

## 下一步

现在你已经了解各组件如何协作，接下来可深入学习各自的细节：

先阅读 [Goa 拦截器](./1-goa-interceptors) 以了解类型安全的请求处理，然后在对应章节探索 HTTP 与 gRPC 的中间件模式。

