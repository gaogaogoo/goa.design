---
title: 自定义 HTTP 中间件
weight: 2
description: >
  学习如何编写与 Goa 服务高效协作的 HTTP 中间件，包含实践示例与集成模式。
---

Goa 服务使用标准的 Go HTTP 处理器，因此你可以使用任何遵循 Go 标准中间件模式的 HTTP 中间件。本文展示如何编写与 Goa 服务良好协作的高效 HTTP 中间件，并结合真实场景示例。

HTTP 中间件应聚焦 HTTP 协议关注点，如请求头、Cookie 以及请求/响应处理。涉及业务逻辑与对服务 Payload 与 Result 的类型安全访问，请使用 Goa 拦截器。拦截器可直接访问服务的领域类型，更适合业务层关注点。

## 常见模式

下面是构建 Goa 服务时特别有用的一些中间件模式。这些模式使用标准的 Go HTTP 中间件技术，并可与 Goa 生成的 HTTP 处理器组合使用。

### 1. 包装 ResponseWriter

标准接口 `http.ResponseWriter` 在写出后无法访问响应元数据。该模式展示如何捕获这类信息：

```go
type responseWriter struct {
    http.ResponseWriter
    status int
    size   int64
}

func (rw *responseWriter) WriteHeader(status int) {
    rw.status = status
    rw.ResponseWriter.WriteHeader(status)
}

func (rw *responseWriter) Write(b []byte) (int, error) {
    size, err := rw.ResponseWriter.Write(b)
    rw.size += int64(size)
    return size, err
}

func MetricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 创建包装器
        rw := &responseWriter{
            ResponseWriter: w,
            status:        http.StatusOK,
        }
        
        start := time.Now()
        next.ServeHTTP(rw, r)
        duration := time.Since(start)
        
        // 记录指标
        metrics.RecordHTTPMetrics(r.Method, r.URL.Path, rw.status, rw.size, duration)
    })
}
```

该模式在 HTTP 请求处理的多个关键领域发挥作用：通过捕获响应状态码与大小，精确采集 HTTP 层指标；便于全面记录响应日志，使你清楚了解服务返回内容；同时也为实现响应转换提供基础，可在响应到达客户端前进行修改或丰富。

注意：若需要访问或修改实际的负载数据（而非仅 HTTP 元数据），请考虑使用 Goa 拦截器。拦截器可类型安全地访问服务的领域类型，无需解析原始 HTTP Body。

### 2. 基于路径的过滤

在使用 Goa 服务时，常需要对不同端点进行差异化处理。该模式展示如何选择性应用中间件：

```go
func PathFilterMiddleware(next http.Handler) http.Handler {
    // 为效率预编译正则
    noLogRegexp := regexp.MustCompile(`^/(healthz|livez|metrics)$`)
    
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        // 对健康检查与指标端点跳过处理
        if noLogRegexp.MatchString(r.URL.Path) {
            next.ServeHTTP(w, r)
            return
        }
        
        // 处理其他请求
        // ... 在此编写你的中间件逻辑 ...
        next.ServeHTTP(w, r)
    })
}
```

基于路径的过滤在需要对不同端点采取不同策略时非常有用。例如，可将健康检查端点排除在日志管道之外以减少噪声；对 API 路由与静态文件路由采用不同处理；并通过在某些路径上跳过不必要的处理来优化中间件性能。这种选择性应用有助于保持服务的高效与良好组织。

### 3. 限流

当保护 API 免受过度使用时，限流是属于中间件的常见 HTTP 层关注点：

```go
type RateLimiter struct {
    requests map[string]*tokenBucket
    mu       sync.RWMutex
    rate     float64
    capacity int64
}

func NewRateLimiter(rate float64, capacity int64) *RateLimiter {
    return &RateLimiter{
        requests: make(map[string]*tokenBucket),
        rate:     rate,
        capacity: capacity,
    }
}

func RateLimitMiddleware(limiter *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 获取客户端标识（例如 IP 地址）
            clientID := r.RemoteAddr
            
            // 检查速率限制
            if !limiter.Allow(clientID) {
                w.Header().Set("Retry-After", "60")
                http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            
            // 添加限流相关响应头
            limit := strconv.FormatInt(limiter.capacity, 10)
            w.Header().Set("X-RateLimit-Limit", limit)
            w.Header().Set("X-RateLimit-Remaining", 
                strconv.FormatInt(limiter.Remaining(clientID), 10))
            
            next.ServeHTTP(w, r)
        })
    }
}
```

该中间件展示了如何处理纯 HTTP 协议关注点：
- 通过令牌桶算法管理请求速率
- 设置合适的限流响应头
- 当超出限制时返回标准的 HTTP 429 状态码
- 纯粹在 HTTP 协议层工作而不涉及业务逻辑

与由 Goa 插件系统处理的 CORS 不同，限流属于协议特定关注点，非常适合放在自定义 HTTP 中间件中。

## 集成示例

以下示例展示如何将 HTTP 中间件与 Goa 生成的处理器集成以添加常见功能。请记住，这些中间件聚焦 HTTP 层关注点——业务逻辑请使用 Goa 拦截器。

### 1. 组织上下文

在多租户服务中，常需要校验并注入组织信息。该中间件负责组织校验的 HTTP 方面：

```go
func OrganizationMiddleware(orgService OrganizationService) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // 从路径或请求头提取组织名
            orgName := extractOrgName(r)
            
            // 将组织名转换为 ID
            orgID, err := orgService.GetOrgID(r.Context(), orgName)
            if err != nil {
                http.Error(w, "Invalid organization", http.StatusBadRequest)
                return
            }
            
            // 将组织 ID 写入上下文
            ctx := context.WithValue(r.Context(), "org.id", orgID)
            next.ServeHTTP(w, r.WithContext(ctx))
        })
    }
}
```

注意：若需基于组织进行业务校验或访问类型化的 Payload，请在 Goa 拦截器中实现，在那里你可以直接访问服务领域类型。

### 2. 请求超时

实现请求级超时以维持服务稳定性：

```go
func TimeoutMiddleware(timeout time.Duration) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ctx, cancel := context.WithTimeout(r.Context(), timeout)
            defer cancel()
            
            done := make(chan struct{})
            go func() {
                next.ServeHTTP(w, r.WithContext(ctx))
                close(done)
            }()
            
            select {
            case <-done:
                return
            case <-ctx.Done():
                w.WriteHeader(http.StatusGatewayTimeout)
                return
            }
        })
    }
}
```

### 3. Authorization Cookie

通过将基于 Header 的认证转换为基于 Cookie 的认证来处理 WebSocket 认证：

```go
func AuthorizationCookieMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        if websocket.IsWebSocketUpgrade(r) {
            // 从 Authorization 头提取令牌
            token := r.Header.Get("Authorization")
            if token != "" {
                // 为 WebSocket 认证设置临时 Cookie
                http.SetCookie(w, &http.Cookie{
                    Name:     "Authorization",
                    Value:    token,
                    Path:     "/",
                    HttpOnly: true,
                    Secure:   true,
                    SameSite: http.SameSiteStrictMode,
                })
            }
        }
        
        next.ServeHTTP(w, r)
    })
}
```

## 完整示例

以下展示如何将这些中间件模式与 Goa HTTP 服务器结合：

```go
func main() {
    // 创建 Goa HTTP 处理器
    mux := goahttp.NewMuxer()
    server := genhttp.New(endpoints, mux, decoder, encoder, eh, eh)
    genhttp.Mount(mux, server)
    
    // 从最外层到最内层构建中间件链
    mux.Use(AuthorizationCookieMiddleware)
    mux.Use(OrganizationMiddleware(orgService))
    mux.Use(TimeoutMiddleware(30 * time.Second))
    mux.Use(PathFilterMiddleware)
    mux.Use(MetricsMiddleware)
    
    // 创建带超时的服务器
    httpServer := &http.Server{
        Addr:              ":8080",
        Handler:           mux,
        ReadHeaderTimeout: 10 * time.Second,
        WriteTimeout:      30 * time.Second,
        IdleTimeout:       120 * time.Second,
    }
}
```

## 测试自定义中间件

使用 Clue 的 [mock 包](https://github.com/goadesign/clue/tree/main/mock) 测试你的中间件：

```go
// 引入 Clue 的 mock 包
import (
    "github.com/goadesign/clue/mock"
)

func TestOrganizationMiddleware(t *testing.T) {
    // 使用 Clue 的 mock 包创建模拟组织服务
    mockOrgService := &mockOrgService{mock.New()}
    
    tests := []struct {
        name     string
        orgName  string
        setup    func(*mockOrgService)
        wantErr  bool
        wantCode int
    }{
        {
            name:    "valid organization",
            orgName: "test-org",
            setup: func(m *mockOrgService) {
                m.Set("GetOrgID", func(ctx context.Context, name string) (string, error) {
                    if name == "test-org" {
                        return "org-123", nil
                    }
                    return "", fmt.Errorf("unknown org")
                })
            },
            wantErr:  false,
            wantCode: http.StatusOK,
        },
        {
            name:    "invalid organization",
            orgName: "invalid-org",
            setup: func(m *mockOrgService) {
                m.Set("GetOrgID", func(ctx context.Context, name string) (string, error) {
                    return "", fmt.Errorf("unknown org")
                })
            },
            wantErr:  true,
            wantCode: http.StatusBadRequest,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 为每个测试创建新的 mock
            mock := &mockOrgService{mock.New()}
            if tt.setup != nil {
                tt.setup(mock)
            }
            
            // 创建中间件
            mw := OrganizationMiddleware(mock)
            
            // 创建测试请求
            req := httptest.NewRequest("GET", "/", nil)
            req.Header.Set("X-Organization", tt.orgName)
            
            // 创建响应记录器
            rec := httptest.NewRecorder()
            
            // 创建测试处理器
            handler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                // 验证上下文中的组织 ID
                orgID := r.Context().Value("org.id")
                if orgID != "org-123" && !tt.wantErr {
                    t.Errorf("expected org ID org-123, got %v", orgID)
                }
                w.WriteHeader(http.StatusOK)
            })
            
            // 执行中间件
            mw(handler).ServeHTTP(rec, req)
            
            // 检查响应
            if rec.Code != tt.wantCode {
                t.Errorf("expected status code %d, got %d", tt.wantCode, rec.Code)
            }
            
            // 验证所有期望的调用已执行
            if mock.HasMore() {
                t.Error("not all expected operations were performed")
            }
        })
    }
}

// 使用 Clue 的 mock 包的模拟实现
// 展示如何使用 Clue 正确组织一个 mock
type mockOrgService struct {
    *mock.Mock // 嵌入 Clue 的 Mock 类型
}

// 使用 Clue 的 Next 模式实现 GetOrgID
func (m *mockOrgService) GetOrgID(ctx context.Context, name string) (string, error) {
    if f := m.Next("GetOrgID"); f != nil {
        return f.(func(context.Context, string) (string, error))(ctx, name)
    }
    return "", errors.New("unexpected call to GetOrgID")
}
```

该示例展示了 Clue 的 mock 包的关键特性：

1. 类型安全的 Mock 实现
2. 使用 `Add` 控制调用顺序
3. 使用 `Set` 定义默认行为
4. 使用 `HasMore` 验证所有期望均已满足

## 最佳实践

1. 保持中间件聚焦：每个中间件应处理单一的 HTTP 关注点。业务逻辑使用 Goa 拦截器。
2. 使用中间件选项：通过函数式选项使中间件可配置。
3. 优雅处理错误：返回合适的 HTTP 状态码与错误信息。
4. 优化性能：预编译正则表达式并使用对象池。
5. 测试边界场景：测试错误条件、超时与并发请求。
6. 文档化行为：记录中间件使用的任何头或上下文值。
7. 关注点分离：HTTP 协议关注点用中间件，业务逻辑使用 Goa 拦截器。

## 下一步

- 查看 [HTTP 传输](@/docs/4-concepts/3-http) 了解 Goa 的 HTTP 处理细节
- 学习 [拦截器](@/docs/4-concepts/5-interceptors) 以处理业务逻辑
- 探索 [可观测性](@/docs/5-real-world/2-observability) 的监控模式
- 查看 [安全](@/docs/5-real-world/3-security) 的最佳实践