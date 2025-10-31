---
title: Unary拦截器
weight: 1
description: >
  学习如何为 Goa 服务实现Unary gRPC 拦截器，并提供常见模式的实际示例。
---

## Unary gRPC 拦截器

Unary拦截器处理 gRPC 服务中的单个请求/响应 RPC。它们非常适合处理协议层面的问题，如元数据处理、日志记录和监控。本指南将向您展示如何为您的 Goa 服务实现有效的Unary拦截器。

## 基本结构

Unary拦截器遵循以下模式：

```go
func UnaryInterceptor(ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {
    
    // 1. 处理程序前操作
    // - 提取元数据
    // - 验证协议要求
    // - 开始计时
    
    // 2. 调用处理程序
    resp, err := handler(ctx, req)
    
    // 3. 处理程序后操作
    // - 记录指标
    // - 转换错误
    // - 添加响应元数据
    
    return resp, err
}
```

这种结构允许您：
- 在请求到达处理程序之前处理它们
- 在处理程序执行后修改或记录响应
- 在协议层面处理错误
- 管理 gRPC 特定的元数据和上下文

## 常见模式

### 1. 元数据处理

此拦截器演示了正确的元数据传播：

```go
func MetadataInterceptor(ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {
    
    // 提取传入的元数据
    md, ok := metadata.FromIncomingContext(ctx)
    if !ok {
        md = metadata.New(nil)
    }
    
    // 添加或修改元数据
    requestID := md.Get("x-request-id")
    if len(requestID) == 0 {
        requestID = []string{uuid.New().String()}
        md = metadata.Join(md, metadata.Pairs("x-request-id", requestID[0]))
    }
    
    // 创建带有元数据的新上下文
    ctx = metadata.NewIncomingContext(ctx, md)
    
    // 调用处理程序
    resp, err := handler(ctx, req)
    
    // 将元数据添加到响应中
    header := metadata.Pairs("x-request-id", requestID[0])
    grpc.SetHeader(ctx, header)
    
    return resp, err
}
```

此示例演示了几个关键的元数据处理功能。它展示了如何从传入请求中提取和验证元数据，确保所需值存在。当请求 ID 等值丢失时，它会生成新值以保持可追溯性。拦截器通过上下文正确传播元数据，使其可用于下游处理程序。最后，它将相关元数据添加到响应中，从而实现请求的端到端跟踪。

### 2. 监控

此拦截器捕获 RPC 指标：

```go
func MonitoringInterceptor(ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {
    
    start := time.Now()
    
    // 提取调用者信息
    peer, _ := peer.FromContext(ctx)
    method := info.FullMethod
    
    // 调用处理程序
    resp, err := handler(ctx, req)
    
    // 记录指标
    duration := time.Since(start)
    status := status.Code(err)
    
    metrics.RecordRPCMetrics(method, peer.Addr.String(), status, duration)
    
    return resp, err
}
```

此模式演示了几个关键的监控功能。它通过在处理程序调用前后捕获时间戳来准确测量 RPC 执行时间。拦截器从上下文中提取重要的调用者信息，使您能够跟踪哪些客户端正在发出请求。它记录可用于监控和警报的标准化指标。最后，它正确处理错误状态码，确保故障在您的指标中得到准确捕获。

### 3. 协议错误处理

处理在服务方法之外发生的协议层面错误：

```go
func ProtocolErrorInterceptor(ctx context.Context,
    req any,
    info *grpc.UnaryServerInfo,
    handler grpc.UnaryHandler) (any, error) {
    
    // 处理上下文错误
    if err := ctx.Err(); err != nil {
        switch err {
        case context.DeadlineExceeded:
            return nil, status.Error(codes.DeadlineExceeded, "请求超时")
        case context.Canceled:
            return nil, status.Error(codes.Canceled, "请求已取消")
        }
    }
    
    // 调用处理程序 (Goa 处理将设计错误映射到 gRPC 状态码)
    resp, err := handler(ctx, req)
    if err != nil {
        return nil, err
    }
    
    // 处理协议特定的验证
    if err := validateProtocolRequirements(resp); err != nil {
        return nil, status.Error(codes.FailedPrecondition, err.Error())
    }
    
    return resp, nil
}
```

此示例演示了 gRPC 拦截器中协议错误处理的几个重要方面。它展示了如何正确处理在 RPC 调用期间可能发生的协议特定错误，如超时和取消。拦截器在为设计错误添加额外的协议层面验证的同时，保留了 Goa 内置的错误映射功能。它还确保在出现协议层面问题时使用适当的 gRPC 状态码，从而与 gRPC 最佳实践保持一致。

## 测试

测试 gRPC 拦截器需要仔细考虑 gRPC 上下文、元数据处理和错误传播。以下是如何使用 Clue 的模拟包有效测试拦截器的方法：

```go
// 用于测试的模拟实现
type mockUnaryHandler struct {
    *mock.Mock
}

func newMockUnaryHandler(t *testing.T) *mockUnaryHandler {
    return &mockUnaryHandler{mock.New()}
}

func (m *mockUnaryHandler) Handle(ctx context.Context, req interface{}) (interface{}, error) {
    if f := m.Next("Handle"); f != nil {
        return f.(func(context.Context, interface{}) (interface{}, error))(ctx, req)
    }
    return nil, nil
}

func TestMetadataInterceptor(t *testing.T) {
    tests := []struct {
        name        string
        setup       func(context.Context, *mockUnaryHandler)
        incomingMD  metadata.MD
        wantReqID   bool
        wantErr     bool
    }{
        {
            name: "添加缺失的请求 ID",
            setup: func(ctx context.Context, h *mockUnaryHandler) {
                h.Set("Handle", func(ctx context.Context, req interface{}) (interface{}, error) {
                    md, ok := metadata.FromIncomingContext(ctx)
                    if !ok {
                        return nil, fmt.Errorf("上下文中没有元数据")
                    }
                    if ids := md.Get("x-request-id"); len(ids) == 0 {
                        return nil, fmt.Errorf("未添加请求 ID")
                    }
                    return "测试响应", nil
                })
            },
            incomingMD: metadata.New(nil),
            wantReqID:  true,
            wantErr:    false,
        },
        {
            name: "保留现有的请求 ID",
            setup: func(ctx context.Context, h *mockUnaryHandler) {
                h.Set("Handle", func(ctx context.Context, req interface{}) (interface{}, error) {
                    md, _ := metadata.FromIncomingContext(ctx)
                    ids := md.Get("x-request-id")
                    if len(ids) != 1 || ids[0] != "test-id" {
                        return nil, fmt.Errorf("请求 ID 未保留")
                    }
                    return "测试响应", nil
                })
            },
            incomingMD: metadata.Pairs("x-request-id", "test-id"),
            wantReqID:  true,
            wantErr:    false,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 创建带有元数据的测试上下文
            ctx := metadata.NewIncomingContext(context.Background(), tt.incomingMD)
            
            // 创建模拟处理程序
            handler := newMockUnaryHandler(t)
            if tt.setup != nil {
                tt.setup(ctx, handler)
            }

            // 调用拦截器
            resp, err := MetadataInterceptor(ctx, "测试请求",
                &grpc.UnaryServerInfo{},
                handler.Handle)

            // 验证错误行为
            if (err != nil) != tt.wantErr {
                t.Errorf("MetadataInterceptor() 错误 = %v, wantErr %v", err, tt.wantErr)
            }

            // 验证所有预期的调用都已执行
            if handler.HasMore() {
                t.Error("并非所有预期的处理程序操作都已执行")
            }
        })
    }
}

// 测试监控拦截器
func TestMonitoringInterceptor(t *testing.T) {
    tests := []struct {
        name       string
        setup      func(*mockUnaryHandler)
        wantMetric string
        wantErr    bool
    }{
        {
            name: "记录成功的调用",
            setup: func(h *mockUnaryHandler) {
                h.Set("Handle", func(ctx context.Context, req interface{}) (interface{}, error) {
                    // 模拟成功处理
                    time.Sleep(10 * time.Millisecond)
                    return "成功", nil
                })
            },
            wantMetric: "成功",
            wantErr:    false,
        },
        {
            name: "记录失败的调用",
            setup: func(h *mockUnaryHandler) {
                h.Set("Handle", func(ctx context.Context, req interface{}) (interface{}, error) {
                    return nil, status.Error(codes.Internal, "测试错误")
                })
            },
            wantMetric: "错误",
            wantErr:    true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            handler := newMockUnaryHandler(t)
            if tt.setup != nil {
                tt.setup(handler)
            }

            resp, err := MonitoringInterceptor(context.Background(), "测试",
                &grpc.UnaryServerInfo{FullMethod: "/test.Service/Method"},
                handler.Handle)

            if (err != nil) != tt.wantErr {
                t.Errorf("MonitoringInterceptor() 错误 = %v, wantErr %v", err, tt.wantErr)
            }

            if handler.HasMore() {
                t.Error("并非所有预期的处理程序操作都已执行")
            }
        })
    }
}

// 测试协议错误处理
func TestProtocolErrorInterceptor(t *testing.T) {
    tests := []struct {
        name      string
        setup     func(context.Context, *mockUnaryHandler)
        ctx       context.Context
        wantCode  codes.Code
    }{
        {
            name: "处理截止日期超限",
            setup: func(ctx context.Context, h *mockUnaryHandler) {
                h.Set("Handle", func(ctx context.Context, req interface{}) (interface{}, error) {
                    return nil, ctx.Err()
                })
            },
            ctx: func() context.Context {
                ctx, cancel := context.WithTimeout(context.Background(), 0)
                defer cancel()
                return ctx
            }(),
            wantCode: codes.DeadlineExceeded,
        },
        {
            name: "处理上下文已取消",
            setup: func(ctx context.Context, h *mockUnaryHandler) {
                h.Set("Handle", func(ctx context.Context, req interface{}) (interface{}, error) {
                    return nil, ctx.Err()
                })
            },
            ctx: func() context.Context {
                ctx, cancel := context.WithCancel(context.Background())
                cancel()
                return ctx
            }(),
            wantCode: codes.Canceled,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            handler := newMockUnaryHandler(t)
            if tt.setup != nil {
                tt.setup(tt.ctx, handler)
            }

            resp, err := ProtocolErrorInterceptor(tt.ctx, "测试",
                &grpc.UnaryServerInfo{},
                handler.Handle)

            if status.Code(err) != tt.wantCode {
                t.Errorf("ProtocolErrorInterceptor() 状态码 = %v, want %v",
                    status.Code(err), tt.wantCode)
            }

            if handler.HasMore() {
                t.Error("并非所有预期的处理程序操作都已执行")
            }
        })
    }
}
```

这些示例演示了 gRPC 拦截器的几种关键测试技术。首先，它们展示了如何有效使用 Clue 的模拟包创建测试替身来验证拦截器行为。`Set` 方法为操作定义默认行为，而 `Next` 允许序列特定的响应。测试涵盖了元数据处理、指标记录和错误状态码的验证。此外，它们还演示了如何测试复杂的上下文场景，例如取消和超时，这些在实际的 gRPC 应用程序中很常见。

## 最佳实践

1.  **保持简单**：每个拦截器应只处理一个问题。
2.  **处理上下文**：尊重上下文取消和截止日期。
3.  **错误处理**：使用适当的 gRPC 状态码。
4.  **性能**：最小化分配和昂贵的操作。
5.  **测试**：测试边缘情况和错误条件。
6.  **日志记录**：使用结构化日志进行调试。
7.  **指标**：记录相关指标以进行监控。

## 下一步

- 学习[流拦截器](@/docs/4-concepts/5-interceptors/3-grpc-interceptors/2-stream.md)
- 回顾[错误处理](@/docs/4-concepts/4-error-handling.md)
- 探索[可观察性](@/docs/5-real-world/2-observability.md)

