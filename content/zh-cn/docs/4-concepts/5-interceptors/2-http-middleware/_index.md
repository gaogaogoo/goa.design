---
linkTitle: HTTP 中间件
title: HTTP 中间件
weight: 2
description: >
  学习如何在 Goa 服务中使用 HTTP 中间件处理协议层关注点，如日志、指标、追踪与请求上下文。
---

Goa 服务中的 HTTP 中间件用于处理协议层关注点，如日志、指标、追踪与请求上下文管理。本文介绍如何在 Goa 服务中高效使用中间件。

## 核心概念

HTTP 中间件通过包裹 HTTP 处理器形成处理链。每个中间件可在请求处理前后执行操作：

```go
func ExampleMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 预处理
        // 例如：日志、指标、追踪

        next.ServeHTTP(w, r)

        // 后处理
        // 例如：响应日志、清理
    })
}
```

## 常见中间件栈

典型的 Goa 服务使用如下中间件栈：

```go
mux.Use(debug.HTTP())                               // 调试日志控制
mux.Use(otelhttp.NewMiddleware("service"))          // OpenTelemetry 探针
mux.Use(log.HTTP(ctx))                              // 请求日志
mux.Use(goahttpmiddleware.RequestID())              // 生成请求 ID
mux.Use(goahttpmiddleware.PopulateRequestContext()) // 填充 Goa 上下文
```

## 关键中间件类型

### 1. 可观测性中间件

处理日志、指标与追踪：

```go
// 带路径过滤的日志中间件
mux.Use(log.HTTP(ctx, 
    log.WithPathFilter(regexp.MustCompile(`^/(healthz|metrics)$`))))

// OpenTelemetry 追踪中间件
mux.Use(otelhttp.NewMiddleware("service-name",
    otelhttp.WithMessageEvents(otelhttp.ReadEvents, otelhttp.WriteEvents)))
```

### 2. 上下文管理

向请求上下文中丰富信息：

```go
func ContextEnrichmentMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 添加请求范围的值
        ctx := r.Context()
        ctx = context.WithValue(ctx, "request.start", time.Now())
        ctx = context.WithValue(ctx, "request.id", r.Header.Get("X-Request-ID"))
        
        // 使用丰富后的上下文继续
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}
```

### 3. 安全中间件

处理认证与请求校验：

```go
func SecurityHeadersMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 设置安全响应头
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-XSS-Protection", "1; mode=block")
        
        next.ServeHTTP(w, r)
    })
}
```

## 最佳实践

### 1. 中间件顺序

谨慎安排中间件顺序，通常从最外层到最内层：

1. Panic 恢复
2. 请求 ID 生成
3. 日志/追踪
4. 安全响应头
5. 认证
6. 上下文填充
7. 业务逻辑

### 2. 性能优化

优化中间件性能：

```go
func OptimizedMiddleware(next http.Handler) http.Handler {
    // 预编译高开销对象
    pathRegex := regexp.MustCompile(`^/api/v\d+/`)
    
    // 频繁分配对象使用 sync.Pool
    bufPool := sync.Pool{
        New: func() interface{} {
            return new(bytes.Buffer)
        },
    }
    
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 非匹配路径跳过中间件
        if !pathRegex.MatchString(r.URL.Path) {
            next.ServeHTTP(w, r)
            return
        }
        
        // 使用对象池资源
        buf := bufPool.Get().(*bytes.Buffer)
        buf.Reset()
        defer bufPool.Put(buf)
        
        next.ServeHTTP(w, r)
    })
}
```

### 3. 错误处理

一致地处理错误：

```go
func ErrorHandlingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                // 使用请求上下文记录错误
                log.Printf("panic recovered: %v", err)
                
                // 返回 500
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}
```

## 与 Goa 服务集成

在 Goa 服务中集成中间件：

```go
func main() {
    // 1. 创建基础 muxer
    mux := goahttp.NewMuxer()
    
    // 2. 创建并挂载 Goa 服务器
    server := genhttp.New(endpoints, mux, decoder, encoder, eh, eh)
    genhttp.Mount(mux, server)
    
    // 3. 添加中间件栈
    mux.Use(debug.HTTP())                  // 调试日志
    mux.Use(otelhttp.NewMiddleware("svc")) // 追踪
    mux.Use(log.HTTP(ctx))                 // 请求日志
    mux.Use(goahttpmiddleware.RequestID()) // 请求 ID
    
    // 4. 创建带超时的 HTTP 服务
    httpServer := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 10 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
    }
}
```

## 测试

在隔离与链路中测试中间件：

```go
func TestMiddleware(t *testing.T) {
    // 创建测试处理器
    handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
    })
    
    // 添加中间件
    handler = YourMiddleware(handler)
    
    // 创建测试请求
    req := httptest.NewRequest("GET", "/test", nil)
    rec := httptest.NewRecorder()
    
    // 执行测试
    handler.ServeHTTP(rec, req)
    
    // 断言结果
    if rec.Code != http.StatusOK {
        t.Errorf("got status %d, want %d", rec.Code, http.StatusOK)
    }
}
```

## 下一步

- 了解 Goa 的[HTTP 传输](@/docs/4-concepts/3-http)
- 探索[可观测性](@/docs/5-real-world/2-observability)模式
- 查看[安全](@/docs/5-real-world/3-security)最佳实践

HTTP 中间件是处理协议特定关注点的强大工具。遵循这些模式与最佳实践，你可以在 Goa 服务中构建干净、可维护且高效的 HTTP 处理管线。

