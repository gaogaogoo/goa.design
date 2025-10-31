---
title: 结合中间件与拦截器
weight: 1
description: >
  学习将 HTTP 中间件与 Goa 拦截器结合的强大模式，构建健壮且可维护的服务。
---

HTTP 中间件与 Goa 拦截器可以协同工作，形成强大的分层解决方案。本文探讨如何有效组合它们的模式与策略。

## 核心概念

### 数据流
中间件与拦截器中的典型数据流：

```
HTTP Request → HTTP Middleware → Goa Transport → Goa Interceptors → Service Method
     └────────────────┴───────────────┴─────────────────────┴────────────────┘
                            Response Flow
```

### 共享上下文
`context.Context` 是共享数据的主要机制：

```go
// HTTP 中间件添加数据
func EnrichContext(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 添加 HTTP 相关数据
        ctx := r.Context()
        ctx = context.WithValue(ctx, "http.start_time", time.Now())
        ctx = context.WithValue(ctx, "http.method", r.Method)
        ctx = context.WithValue(ctx, "http.path", r.URL.Path)
        
        // 使用增强的上下文继续处理
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Goa 拦截器使用这些数据
var _ = Service("api", func() {
    Interceptor("RequestLogger", func() {
        Description("结合 HTTP 上下文记录请求细节")
        
        Request(func() {
            // 在实现中访问 HTTP 上下文
            Attribute("method")
            Attribute("path")
            Attribute("duration")
        })
    })
})

// 实现同时使用两者
func (i *Interceptors) RequestLogger(ctx context.Context, info *RequestLoggerInfo, next goa.Endpoint) (any, error) {
    // 从上下文访问 HTTP 数据
    startTime := ctx.Value("http.start_time").(time.Time)
    method := ctx.Value("http.method").(string)
    path := ctx.Value("http.path").(string)
    
    // 调用服务
    res, err := next(ctx, info.RawPayload())
    
    // 结合数据记录日志
    duration := time.Since(startTime)
    log.Printf("HTTP %s %s completed in %v", method, path, duration)
    
    return res, err
}
```

## 常见模式

### 1. 认证链（Authentication Chain）

将 HTTP 认证与业务授权相结合：

```go
// HTTP 中间件进行 JWT 验证
func JWTAuth(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")
        
        // 验证 JWT
        claims, err := validateJWT(token)
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        
        // 将 claims 放入上下文
        ctx := context.WithValue(r.Context(), "jwt.claims", claims)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Goa 拦截器进行授权
var _ = Service("api", func() {
    Interceptor("Authorizer", func() {
        Description("使用 JWT claims 检查权限")
        
        Request(func() {
            // 在实现中访问 claims
            Attribute("claims")
            Attribute("resource")
            Attribute("action")
        })
    })
})

// 实现将两者结合
func (i *Interceptors) Authorizer(ctx context.Context, info *AuthorizerInfo, next goa.Endpoint) (any, error) {
    // 从 HTTP 上下文获取 claims
    claims := ctx.Value("jwt.claims").(JWTClaims)
    
    // 检查权限
    if !hasPermission(claims, info.Resource(), info.Action()) {
        return nil, goa.NewErrorf(goa.ErrForbidden, "insufficient permissions")
    }
    
    return next(ctx, info.RawPayload())
}
```

### 2. 可观测性栈（Observability Stack）

通过组合 HTTP 与业务指标构建全面的可观测性：

```go
// HTTP 中间件采集请求指标
func HTTPMetrics(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        // 创建记录型 ResponseWriter
        rec := newRecordingResponseWriter(w)
        
        // 处理请求
        next.ServeHTTP(rec, r)
        
        // 记录 HTTP 指标
        duration := time.Since(start)
        metrics.RecordHTTPMetrics(
            r.Method,
            r.URL.Path,
            rec.StatusCode(),
            duration,
            rec.BytesWritten(),
        )
    })
}

// Goa 拦截器添加业务上下文
var _ = Service("api", func() {
    Interceptor("BusinessMetrics", func() {
        Description("记录业务层指标")
        
        Request(func() {
            Attribute("operation")
        })
        Response(func() {
            Attribute("status")
            Attribute("result")
        })
    })
})

// 实现组合指标
func (i *Interceptors) BusinessMetrics(ctx context.Context, info *BusinessMetricsInfo, next goa.Endpoint) (any, error) {
    start := time.Now()
    
    // 调用服务
    res, err := next(ctx, info.RawPayload())
    
    // 记录业务指标
    duration := time.Since(start)
    metrics.RecordBusinessMetrics(
        info.Operation(),
        duration,
        err == nil,
    )
    
    return res, err
}
```

### 3. 缓存策略（Caching Strategy）

通过 HTTP 与业务逻辑实现多层缓存：

```go
// HTTP 中间件处理响应缓存
func HTTPCache(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        key := generateCacheKey(r)
        
        // 检查 HTTP 缓存
        if cached := httpCache.Get(key); cached != nil {
            writeFromCache(w, cached)
            return
        }
        
        // 在上下文中添加标记后继续
        ctx := context.WithValue(r.Context(), "cache.key", key)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Goa 拦截器处理业务层缓存
var _ = Service("api", func() {
    Interceptor("BusinessCache", func() {
        Description("实现业务层缓存")
        
        Request(func() {
            Attribute("cacheKey")
            Attribute("cacheTTL")
        })
    })
})

// 实现组合缓存策略
func (i *Interceptors) BusinessCache(ctx context.Context, info *BusinessCacheInfo, next goa.Endpoint) (any, error) {
    // 从 HTTP 上下文获取缓存键
    httpKey := ctx.Value("cache.key").(string)
    
    // 检查业务缓存
    if cached := businessCache.Get(httpKey); cached != nil {
        return cached, nil
    }
    
    // 调用服务
    res, err := next(ctx, info.RawPayload())
    if err != nil {
        return nil, err
    }
    
    // 缓存结果
    businessCache.Set(httpKey, res, info.CacheTTL())
    
    return res, nil
}
```

## 最佳实践

### 1. 上下文管理

- 使用类型化的上下文键
- 文档化上下文依赖
- 优雅处理缺失的上下文值

```go
// 定义类型化上下文键
type contextKey string

const (
    RequestIDKey   contextKey = "request_id"
    UserClaimsKey  contextKey = "user_claims"
    TraceIDKey     contextKey = "trace_id"
)

// 在中间件中使用类型化键
func WithRequestID(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        requestID := uuid.New().String()
        ctx := context.WithValue(r.Context(), RequestIDKey, requestID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// 在拦截器中安全访问上下文
func (i *Interceptors) Logger(ctx context.Context, info *LoggerInfo, next goa.Endpoint) (any, error) {
    requestID, _ := ctx.Value(RequestIDKey).(string)
    if requestID == "" {
        requestID = "unknown"
    }
    
    // 安全使用 requestID
    return next(ctx, info.RawPayload())
}
```

### 2. 错误处理

- 定义清晰的错误边界
- 保持一致的错误格式
- 保留错误上下文

```go
// HTTP 中间件的错误边界
func ErrorBoundary(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        defer func() {
            if err := recover(); err != nil {
                // 记录 panic
                log.Printf("panic: %v", err)
                
                // 返回 500
                http.Error(w, "Internal Server Error", http.StatusInternalServerError)
            }
        }()
        
        next.ServeHTTP(w, r)
    })
}

// Goa 的错误处理
var _ = Service("api", func() {
    Error("not_found", ErrorResult)
    Error("invalid_input", ErrorResult)
    
    Interceptor("ErrorHandler", func() {
        Description("处理业务错误")
        
        Error("not_found")
        Error("invalid_input")
    })
})
```

### 3. 测试

编写验证集成的测试：

```go
func TestMiddlewareInterceptorIntegration(t *testing.T) {
    // 创建测试服务
    svc := NewService()
    
    // 构建中间件链
    handler := JWTAuth(
        HTTPMetrics(
            // ... 其他中间件
        ),
    )
    
    // 构建拦截器链
    endpoints := NewEndpoints(svc)
    endpoints.Use(BusinessMetrics)
    
    // 创建测试服务器
    server := httptest.NewServer(handler)
    defer server.Close()
    
    // 测试用例
    tests := []struct {
        name           string
        token          string
        expectedStatus int
        expectedBody   string
    }{
        // ... 测试用例
    }
    
    // 运行测试
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 发起请求
            req, _ := http.NewRequest("GET", server.URL, nil)
            req.Header.Set("Authorization", tt.token)
            
            // 验证响应
            resp, err := http.DefaultClient.Do(req)
            if err != nil {
                t.Fatal(err)
            }
            
            // 检查状态码
            if resp.StatusCode != tt.expectedStatus {
                t.Errorf("got status %d, want %d", resp.StatusCode, tt.expectedStatus)
            }
            
            // 检查指标
            // ... 验证同时记录了 HTTP 与业务指标
        })
    }
}
```

## 下一步

- 查看[自定义中间件](./custom)的实现细节
- 在[可观测性](@/docs/5-real-world/2-observability)中探索真实案例
- 了解 Goa 的[错误处理](@/docs/4-concepts/4-error-handling)

有效组合 HTTP 中间件与 Goa 拦截器，能让你以干净、有组织的方式同时处理 HTTP 协议关注点与业务逻辑，从而构建健壮且可维护的服务。