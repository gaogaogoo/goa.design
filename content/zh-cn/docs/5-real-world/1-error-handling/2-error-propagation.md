---
title: 错误传播
description: "了解错误如何在 Goa 服务的各层中传播"
weight: 2
---

本指南解释了错误如何在 Goa 服务的不同层中传播，从业务逻辑到客户端。

## 概览

Goa 中的错误传播遵循清晰的路径：
1. 业务逻辑产生错误
2. 错误与其设计定义进行匹配
3. 传输层对错误进行转换
4. 客户端接收并解释错误

## 错误流程

### 1. 业务逻辑层

错误通常源自你的服务实现：

```go
func (s *paymentService) Process(ctx context.Context, p *payment.ProcessPayload) (*payment.ProcessResult, error) {
    // 业务逻辑可能通过多种方式返回错误：
    
    // 1. 使用生成的辅助函数（用于 ErrorResult）
    if !hasEnoughFunds(p.Amount) {
        return nil, payment.MakeInsufficientFunds(
            fmt.Errorf("账户余额 %d 低于所需金额 %d", balance, p.Amount))
    }
    
    // 2. 返回自定义错误类型
    if err := validateCard(p.Card); err != nil {
        return nil, &payment.PaymentError{
            Name:    "card_expired",
            Message: err.Error(),
        }
    }
    
    // 3. 传播下游服务的错误
    result, err := s.processor.ProcessPayment(ctx, p)
    if err != nil {
        // 用你的领域错误包装外部错误
        return nil, payment.MakeProcessingFailed(fmt.Errorf("支付处理器错误：%w", err))
    }
    
    return result, nil
}
```

### 2. 错误匹配

Goa 运行时会将返回的错误与其设计定义进行匹配：

```go
var _ = Service("payment", func() {
    // 此处定义的错误按名称匹配
    Error("insufficient_funds")
    Error("card_expired")
    Error("processing_failed", func() {
        // 属性会影响错误处理
        Temporary()
        Fault()
    })
})
```

匹配过程：
1. 针对 `ErrorResult`：使用生成的 `MakeXXX` 函数中的错误名称
2. 针对自定义类型：使用带有 `struct:error:name` 标记的字段
3. 未知错误：视为内部服务器错误

### 3. 传输层

匹配后，错误会根据传输特定规则进行转换：

```go
var _ = Service("payment", func() {
    HTTP(func() {
        // HTTP 映射规则
        Response("insufficient_funds", StatusPaymentRequired)
        Response("card_expired", StatusUnprocessableEntity)
        Response("processing_failed", StatusServiceUnavailable)
    })
    
    GRPC(func() {
        // gRPC 映射规则
        Response("insufficient_funds", CodeFailedPrecondition)
        Response("card_expired", CodeInvalidArgument)
        Response("processing_failed", CodeUnavailable)
    })
})
```

传输层将：
1. 应用适当的状态码
2. 格式化错误消息和详细信息
3. 序列化响应

### 4. 客户端接收

客户端接收与设计匹配的强类型错误：

```go
client := payment.NewClient(endpoint)
result, err := client.Process(ctx, payload)
if err != nil {
    switch e := err.(type) {
    case *payment.InsufficientFundsError:
        // 处理余额不足（包含错误属性）
        if e.Temporary {
            return retry(ctx, payload)
        }
        return promptForTopUp(e.Message)
        
    case *payment.CardExpiredError:
        // 处理卡过期
        return promptForNewCard(e.Message)
        
    case *payment.ProcessingFailedError:
        // 处理处理失败
        if e.Temporary {
            return retryWithBackoff(ctx, payload)
        }
        return reportSystemError(e)
        
    default:
        // 处理意外错误
        return handleUnknownError(err)
    }
}
```

## 最佳实践

1. **错误包装**
   - 用领域错误包装外部错误
   - 使用 `fmt.Errorf("...%w", err)` 保留根因
   - 添加与你的领域相关的上下文

2. **一致传播**
   - 尽可能使用生成的辅助函数
   - 在链路中保持错误属性
   - 不要不必要地混合错误类型

3. **传输考量**
   - 为每种传输定义合适的状态码
   - 包含相关的标头/元数据
   - 考虑客户端需求

4. **客户端体验**
   - 提供强类型错误
   - 包含足够的处理上下文
   - 文档化重试策略

## 错误转换示例

以下是错误在系统中如何转换的完整示例：

```go
// 1. Business Logic (Service Implementation)
if !hasEnoughFunds(amount) {
    return nil, payment.MakeInsufficientFunds(
        fmt.Errorf("余额 %d 低于所需 %d", balance, amount))
}

// 2. Error Definition (Design)
var _ = Service("payment", func() {
    Error("insufficient_funds", func() {
        Description("账户余额不足")
        Temporary()  // 充值后可重试
    })
})

// 3. Transport Mapping (Design)
HTTP(func() {
    Response("insufficient_funds", StatusPaymentRequired)
})

// 4. Client Reception
result, err := client.Process(ctx, payload)
if err != nil {
    if e, ok := err.(*payment.InsufficientFundsError); ok {
        if e.Temporary {
            // 等待 Retry-After 标头时长
            time.Sleep(retryAfter)
            return retry(ctx, payload)
        }
    }
}
```

## 结论

Goa 的错误传播系统确保：
- 错误在各层保持其语义含义
- 传输特定的细节由系统自动处理
- 客户端接收强类型、可操作的错误
- 错误处理在整个 API 中保持一致

