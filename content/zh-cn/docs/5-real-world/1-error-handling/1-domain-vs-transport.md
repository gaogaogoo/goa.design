---
title: 领域与传输
weight: 1
description: "了解 Goa 中领域错误和传输错误之间的区别，以及如何有效地在它们之间进行映射。"
---

在 Goa 中设计错误处理时，了解领域错误与其传输表示之间的区别非常重要。这种分离使您能够保持清晰的领域逻辑，同时确保跨不同协议的正确错误通信。

## 领域错误

领域错误表示应用程序中的业务逻辑失败。它们与协议无关，专注于从业务逻辑角度看出了什么问题。Goa 的默认 `ErrorResult` 类型通常足以表达领域错误——自定义错误类型是可选的，仅在特殊情况下需要。

### 使用默认错误类型

默认的 `ErrorResult` 类型与有意义的名称、描述和错误属性相结合，可以有效地表达大多数领域错误：

```go
var _ = Service("payment", func() {
    // 使用默认 ErrorResult 类型定义领域错误
    Error("insufficient_funds", ErrorResult, func() {
        Description("账户资金不足以进行交易")
        // 错误属性有助于定义错误特征
        Temporary()  // 如果用户充值，错误可能会解决
    })

    Error("card_expired", ErrorResult, func() {
        Description("支付卡已过期")
        // 这是一个永久性错误，直到卡更新为止
    })

    Error("processing_failed", ErrorResult, func() {
        Description("支付处理系统暂时不可用")
        Temporary()  // 稍后可以重试
        Fault()      // 服务器端问题
    })
    
    Method("process", func() {
        // ... 方法定义
    })
})
```

领域错误应该：
- 具有清晰、描述性的名称，以反映业务场景
- 包括有意义的描述以用于文档和调试
- 使用错误属性来指示错误特征
- 独立于它们的传输方式

### 自定义错误类型（可选）

在需要额外结构化错误数据的情况下，您可以定义自定义错误类型。有关自定义错误类型的详细信息，包括对 `name` 字段和 `struct:error:name` 元数据的重要要求，请参阅[主要错误处理文档](_index.md#custom-error-types)。

```go
// 需要额外错误上下文时的自定义类型
var PaymentError = Type("PaymentError", func() {
    Description("PaymentError 表示支付处理失败")
    Field(1, "message", String, "人类可读的错误消息")
    Field(2, "code", String, "内部错误代码")
    Field(3, "transaction_id", String, "失败的交易 ID")
    Field(4, "name", String, "用于传输映射的错误名称", func() {
        Meta("struct:error:name")
    })
    Required("message", "code", "name")
})
```

## 传输映射

传输映射定义了领域错误在特定协议中的表示方式。这包括状态码、标头和响应格式。

### HTTP 传输

```go
var _ = Service("payment", func() {
    // 定义领域错误
    Error("insufficient_funds", PaymentError)
    Error("card_expired", PaymentError)
    Error("processing_failed", PaymentError)
    
    HTTP(func() {
        // 将领域错误映射到 HTTP 状态码
        Response("insufficient_funds", StatusPaymentRequired, func() {
            // 添加特定于支付的标头
            Header("Retry-After")
            // 自定义错误响应格式
            Body(func() {
                Attribute("error_code")
                Attribute("message")
            })
        })
        Response("card_expired", StatusUnprocessableEntity)
        Response("processing_failed", StatusServiceUnavailable)
    })
})
```

### gRPC 传输

```go
var _ = Service("payment", func() {
    // 相同的领域错误
    Error("insufficient_funds", PaymentError)
    Error("card_expired", PaymentError)
    Error("processing_failed", PaymentError)
    
    GRPC(func() {
        // 映射到 gRPC 状态码
        Response("insufficient_funds", CodeFailedPrecondition)
        Response("card_expired", CodeInvalidArgument)
        Response("processing_failed", CodeUnavailable)
    })
})
```

## 分离的好处

这种关注点分离提供了几个优势：

1. **协议独立性**
   - 领域错误仍然专注于业务逻辑
   - 同一个错误可以针对不同协议进行不同映射
   - 易于添加新的传输协议

2. **一致的错误处理**
   - 集中式错误定义
   - 跨服务统一的错误处理
   - 领域错误和传输错误之间的清晰映射

3. **更好的文档**
   - 领域错误记录业务规则
   - 传输映射记录 API 行为
   - 清晰的分离有助于 API 使用者

## 实现示例

以下是这种分离在实践中的工作方式：

### 使用默认 ErrorResult

```go
func (s *paymentService) Process(ctx context.Context, p *payment.ProcessPayload) (*payment.ProcessResult, error) {
    // 领域逻辑
    if !hasEnoughFunds(p.Amount) {
        // 使用生成的辅助函数返回错误
        return nil, payment.MakeInsufficientFunds(
            fmt.Errorf("账户余额 %d 低于所需金额 %d", balance, p.Amount))
    }
    
    if isSystemOverloaded() {
        // 返回临时系统问题的错误
        return nil, payment.MakeProcessingFailed(
            fmt.Errorf("支付系统暂时不可用"))
    }
    
    // 更多处理...
}
```

### 使用自定义错误类型（需要额外上下文时）

```go
func (s *paymentService) Process(ctx context.Context, p *payment.ProcessPayload) (*payment.ProcessResult, error) {
    // 领域逻辑
    if !hasEnoughFunds(p.Amount) {
        // 返回带有附加上下文的领域错误
        return nil, &payment.PaymentError{
            Name:          "insufficient_funds",
            Message:       "账户余额过低，无法进行交易",
            Code:         "FUNDS_001",
            TransactionID: txID,
        }
    }
    
    // 更多处理...
}
```

传输层会自动：
1. 将领域错误映射到适当的状态码
2. 根据协议格式化错误响应
3. 包括任何特定于协议的标头或元数据

## 最佳实践

1. **领域优先**
   - 根据业务需求设计错误
   - 在错误消息中使用领域术语
   - 包括用于调试的相关上下文

2. **一致的映射**
   - 为每个协议使用适当的状态码
   - 在服务之间保持一致的映射
   - 记录映射的基本原理

3. **错误属性**
   - 使用错误属性（`Temporary()`、`Timeout()`、`Fault()`）来指示错误特征
   - 考虑在自定义错误类型中实现类似的属性
   - 记录属性如何影响客户端行为

4. **文档**
   - 同时记录领域含义和传输行为
   - 包括错误响应示例
   - 解释重试策略和客户端处理

## 结论

通过将领域错误与其传输表示分离，Goa 使您能够：
- 保持清晰的领域逻辑
- 提供适合协议的错误响应
- 一致地支持多种协议
- 随着 API 的增长扩展错误处理

