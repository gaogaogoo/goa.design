---
title: 覆盖错误序列化
linkTitle: 覆盖序列化
weight: 6
description: "通过实现自定义错误格式化器，定制 Goa 的错误序列化方式，并以定制响应处理特定错误类型。"
---

在 Goa 中，框架会自动处理错误，并以一致的方式传达诸如校验错误、内部服务器错误或自定义业务错误等问题。但在某些情况下，你可能希望定制这些错误的序列化方式或呈现给客户端的形式。本节解释如何在 Goa 中覆盖默认的错误序列化，适用于任何类型的错误，包括内置的校验错误。

## 自定义错误序列化

Goa 允许通过实现自定义错误格式化函数来定制错误的序列化方式。该格式化器接收错误对象，并可检查其属性以决定在响应中如何进行序列化。该格式化器可以处理任意类型的错误，包括：

- Goa 内置的校验错误
- 在 DSL 中定义的自定义服务错误
- 标准 Go 错误
- 第三方错误类型

格式化器返回一个响应对象，用于决定错误负载的结构以及应使用的 HTTP 状态码。这使你可以完全掌控错误在 API 客户端中的呈现方式。

### 示例：为特定错误类型定制序列化

考虑一种场景：你希望定制特定错误类型（例如缺失字段错误或自定义业务错误）的序列化。你可以通过定义自定义错误类型并提供对应的错误格式化器来实现。

#### 步骤 1：定义自定义错误类型

首先，定义用于序列化特定错误的自定义错误类型。这些类型应实现 `Statuser` 接口以返回合适的 HTTP 状态码。

```go
// missingFieldError 用于序列化缺失必填字段的错误。
type missingFieldError string

// StatusCode 返回 400（BadRequest）。
func (missingFieldError) StatusCode() int {
    return http.StatusBadRequest
}

// customBusinessError 用于序列化自定义业务逻辑错误。
type customBusinessError string

// StatusCode 返回 422（Unprocessable Entity）。
func (customBusinessError) StatusCode() int {
    return http.StatusUnprocessableEntity
}
```

#### 步骤 2：实现自定义错误格式化器

接着，实现一个自定义错误格式化器，根据错误属性将错误转换为合适的自定义错误类型。

```go
// customErrorResponse 根据错误属性将 err 转换为自定义错误类型。
func customErrorResponse(ctx context.Context, err error) Statuser {
    if serr, ok := err.(*goa.ServiceError); ok {
        switch serr.Name {
        case "missing_field":
            return missingFieldError(serr.Message)
        case "business_error":
            return customBusinessError(serr.Message)
        default:
            // 其他错误使用 Goa 默认处理
            return goahttp.NewErrorResponse(err)
        }
    }
    // 其他错误类型使用 Goa 默认处理
    return goahttp.NewErrorResponse(err)
}
```

#### 步骤 3：使用自定义错误格式化器

最后，在实例化 HTTP 服务器或处理器时使用该自定义错误格式化器。这样即可确保服务返回的所有错误都应用你的自定义序列化逻辑。

```go
var (
    appServer *appsvr.Server
)
{
    eh := errorHandler(logger)
    appServer = appsvr.New(appEndpoints, mux, dec, enc, eh, customErrorResponse)
    // ...
}
```

### 自定义错误序列化的优势

- 一致性：自定义序列化可将包含校验错误在内的错误呈现保持一致。
- 清晰性：可提供更具描述性的错误消息或附加上下文，帮助客户端理解并解决特定用例中的问题。
- 灵活性：可按需定制错误响应，以适配现有的客户端错误处理逻辑或支持自定义业务规则。

## 总结

在 Goa 中进行自定义错误序列化为你提供了强大的手段，在保持 API 一致性的同时，满足特定需求。通过实现自定义错误格式化器并将其与 Goa 的错误处理机制集成，你可以为客户端提供更有意义、可操作的错误响应。

现在你已了解如何定制错误序列化，请继续阅读下一节[最佳实践](../7-best-practices)，学习在 Goa 服务中实现健壮错误处理的推荐模式与策略。