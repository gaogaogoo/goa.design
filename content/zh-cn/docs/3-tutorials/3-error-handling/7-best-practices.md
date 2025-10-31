---
title: 最佳实践
linkTitle: 最佳实践
weight: 7
description: "在 Goa 服务中实现健壮错误处理的关键指南与推荐实践，包括命名约定与测试策略。"
---

要构建可靠、可维护的 API，健壮的错误处理至关重要。以下是在基于 Goa 的服务中定义与管理错误时应遵循的最佳实践：

## 1. 保持一致的错误命名

描述性命名：为错误使用清晰且具描述性的名称，准确反映问题，从而让开发者更容易理解并正确处理错误。

优秀示例：

```go
Error("DivByZero", func() { Description("DivByZero is returned when the divisor is zero.") })
```

不佳示例：

```go
Error("Error1", func() { Description("An unspecified error occurred.") })
```

## 2. 优先使用 ErrorResult 而非自定义类型

简洁性：对于大多数错误，使用默认的 ErrorResult 类型以保持服务内的一致性与简洁性。

何时使用自定义类型：当你需要包含超出 ErrorResult 所提供范围的额外上下文信息时，再使用自定义错误类型。

使用 ErrorResult：

```go
var _ = Service("calculator", func() {
    Error("InvalidInput", func() { Description("Invalid input provided.") })
})
```

或：

```go
var _ = Service("calculator", func() {
    Error("InvalidInput", ErrorResult, "Invalid input provided.")
})
```

使用自定义类型：

```go
var _ = Service("calculator", func() {
    Error("InvalidOperation", InvalidOperation, "Unsupported operation.")
})
```

## 3. 充分利用 DSL 功能

错误标识：使用 `Temporary()`、`Timeout()` 与 `Fault()` 等 DSL 功能为错误提供额外元数据，丰富错误信息，便于客户端更好处理。

示例：

```go
Error("ServiceUnavailable", func() { 
    Description("ServiceUnavailable is returned when the service is temporarily unavailable.")
    Temporary()
})
```

描述：始终为错误提供有意义的描述，有助于文档与客户端理解。

## 4. 详尽记录错误

清晰描述：确保每个错误都有清晰、简洁的描述，帮助客户端理解错误的上下文与原因。

生成文档：利用 Goa 能从 DSL 定义生成文档的能力。良好的错误文档能提升 API 使用者的开发体验。

示例：

```go
Error("AuthenticationFailed", ErrorResult, Description("AuthenticationFailed is returned when user credentials are invalid."))
```

## 5. 正确实现错误映射

传输一致性：确保错误被一致地映射到合适的传输层状态码（HTTP、gRPC），为客户端提供有意义的响应。

自动化映射：使用 Goa 的 DSL 定义这些映射，降低不一致与样板代码的风险。

示例：

```go
var _ = Service("auth", func() {
    Error("InvalidToken", func() {
        Description("InvalidToken is returned when the provided token is invalid.")
    })

    HTTP(func() {
        Response("InvalidToken", StatusUnauthorized)
    })

    GRPC(func() {
        Response("InvalidToken", CodeUnauthenticated)
    })
})
```

## 6. 测试错误处理

自动化测试：编写自动化测试以确保错误被正确定义、映射与处理，有助于在开发早期发现问题。

客户端模拟：模拟客户端交互以验证错误在不同传输层上的沟通是否符合预期。

示例测试用例：

```go
func TestDivideByZero(t *testing.T) {
    svc := internal.NewDividerService()
    _, err := svc.Divide(context.Background(), &divider.DividePayload{A: 10, B: 0})
    if err == nil {
        t.Fatalf("expected error, got nil")
    }
    if serr, ok := err.(*goa.ServiceError); !ok || serr.Name != "DivByZero" {
        t.Fatalf("expected DivByZero error, got %v", err)
    }
}
```

## 结论

遵循以上最佳实践，可确保你的 Goa 服务拥有健壮且一致的错误处理机制。通过利用 Goa 的 DSL 功能、保持清晰且具描述性的错误定义，以及实施完善的测试，你可以构建既对开发者友好又对终端用户可靠的 API。