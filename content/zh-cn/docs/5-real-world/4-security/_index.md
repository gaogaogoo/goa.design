---
linkTitle: 安全
title: 安全
weight: 4
description: 了解如何使用多种认证方式保护你的 HTTP Goa API
---

Goa 提供了健壮的安全特性，使你能够在多个层次保护 API。无论你需要基本认证、API Key、JWT 令牌，还是 OAuth2，Goa 的安全 DSL 都能让你轻松实现安全的端点。

本节将从基础概念到进阶实现，带你逐步了解 Goa 的安全功能。我们会详细介绍每种认证方式，并提供保护 API 的最佳实践。

## 理解 Goa 中的安全

Goa 中的安全通过“安全方案”（Security Schemes）实现——它们是可复用的定义，用来指定认证和授权应该如何工作。你可以把这些方案理解为模板，用于定义 API 如何验证试图访问端点的客户端身份。

这些安全方案可以应用在三个不同的层次，提供灵活的安全配置：

- API 级别：当在 API 级别应用时，安全方案会成为整个 API 所有端点的默认方案。适用于希望在整个 API 中保持一致安全策略的场景。
- 服务级别：服务可以覆盖 API 级别的安全，或者在 API 级别未定义时自行定义安全。适用于不同端点组需要不同安全要求的场景。
- 方法级别：单个方法（端点）可以覆盖 API 和服务级别的安全。它提供了最细粒度的控制，使特定端点可以使用不同的安全方案，甚至不使用安全。

## 可用的安全方案

Goa 通过专用 DSL 函数支持多种常见的安全机制。每种方案都针对特定的使用场景而设计：

### 基本认证（Basic Authentication）

基本认证是最简单的 API 安全形式之一，客户端需要提供用户名和密码。尽管简单，但应仅在 HTTPS 上使用，以确保凭据在传输过程中被加密。

[了解更多关于 Basic Authentication →](1-basic-auth.md)

```go
var BasicAuth = BasicAuthSecurity("basic", func() {
    Description("使用用户名和密码的基本认证")
})
```

### API Key 认证

API Key 通过单个令牌提供一种简单的客户端认证方式。它们通常通过请求头或查询参数传递。该方法在需要跟踪使用情况或实施限流的公共 API 中很常见。

[了解更多关于 API Key 认证 →](2-api-key.md)

```go
var APIKeyAuth = APIKeySecurity("api_key", func() {
    Description("通过要求 API Key 来保护端点。")
})
```

### JWT（JSON Web Token）认证

JWT 认证非常适合使用签名令牌进行无状态认证，并可携带声明（claims）。在需要在微服务之间传递认证与授权信息的架构中，JWT 是完美选择。

[了解更多关于 JWT 认证 →](3-jwt.md)

```go
var JWTAuth = JWTSecurity("jwt", func() {
    Description("要求有效的 JWT 令牌来保护端点。")
    Scope("api:read", "读取 API 权限")
    Scope("api:write", "写入 API 权限")
})
```

### OAuth2 认证

OAuth2 为委托授权提供了全面的解决方案。当你需要允许第三方应用代表你的用户访问 API 时，它是理想选择。

[了解更多关于 OAuth2 认证 →](4-oauth2.md)

```go
var OAuth2 = OAuth2Security("oauth2", func() {
    Description("OAuth2 认证")
    ImplicitFlow("/authorization")
    Scope("api:write", "写入访问")
    Scope("api:read", "读取访问")
})
```

## 安全层级示例

下面我们来看一个完整示例，展示安全方案如何应用在不同层级。该示例体现了 Goa 安全系统的灵活性，以及如何组合不同的安全需求：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

// 定义安全方案
var BasicAuth = BasicAuthSecurity("basic", func() {
    Description("基本认证")
})

var APIKeyAuth = APIKeySecurity("api_key", func() {
    Description("通过要求 API Key 来保护端点。")
})

var JWTAuth = JWTSecurity("jwt", func() {
    Description("要求有效的 JWT 令牌来保护端点。")
})

// 在 API 级别应用安全
var _ = API("hierarchy", func() {
    Title("安全示例 API")
    Description("该 API 展示了在 API、服务或方法级别使用 Security 的效果")

    // 为所有端点设置 Basic 认证为默认安全方案
    Security(BasicAuth)
})
```

### 默认服务（Basic 认证）

该服务继承了 API 级别的基本认证。注意载荷必须包含用户名和密码字段：

```go
var _ = Service("default_service", func() {
    Method("default", func() {
        Description("default_service 的 default 方法使用基本认证保护")
        // 为基本认证定义预期的载荷
        Payload(func() {
            Username("username")  // 用户名字段的专用 DSL
            Password("password")  // 密码字段的专用 DSL
            Required("username", "password")
        })
        HTTP(func() { GET("/default") })
    })
})
```

### 混合安全的服务

该服务展示了如何在服务级别和方法级别覆盖安全，体现 Goa 安全系统的灵活性：

```go
var _ = Service("api_key_service", func() {
    Description("svc 服务使用 API Key 认证进行保护")
    HTTP(func() { Path("/svc") })

    // 覆盖整个服务的 API 级别安全
    Security(APIKeyAuth)

    Method("default", func() {
        // 此方法使用服务级别的 API Key 安全
        Payload(func() {
            APIKey("api_key", "key", String, func() {
                Description("用于认证的 API Key")
            })
            Required("key")
        })
        HTTP(func() { GET("/default") })
    })

    Method("secure", func() {
        // 为该方法覆盖服务级别的安全为 JWT
        Security(JWTAuth)
        Description("该方法需要有效的 JWT 令牌。")
        Payload(func() {
            Token("token", String, func() {
                Description("用于认证的 JWT 令牌")
            })
            Required("token")
        })
        HTTP(func() { GET("/secure") })
    })

    Method("unsecure", func() {
        Description("该方法不启用安全。")
        // 移除该方法的所有安全要求
        NoSecurity()
    })
})
```

## 使用 NoSecurity() 取消安全

有时你需要让某些端点公开访问，例如健康检查或公开文档端点。`NoSecurity()` 函数会移除方法上的任何安全要求：

```go
Method("health", func() {
    Description("公开健康检查端点")
    NoSecurity()
    HTTP(func() { GET("/health") })
})
```

## 最佳实践

在 Goa 服务中实施安全时，请遵循以下指南以获得最佳结果：

[了解更多安全最佳实践 →](5-best-practices.md)

1. 在 API 级别定义安全方案以获得一致的默认行为
2. 仅在服务需要不同认证时在服务级别覆盖安全
3. 谨慎使用方法级别安全，仅用于特殊情况
4. 当端点公开时务必显式使用 `NoSecurity()`
5. 在安全方案中包含清晰描述，帮助 API 使用者理解
6. 生产环境始终使用 HTTPS
7. 对 API Key 认证实施限流
8. 为 JWT 令牌设置合适的过期时间
9. 定期轮换密钥和机密
10. 记录并监控认证失败

## 生成的代码

Goa 会自动生成用于强制执行安全要求的必要代码。生成的代码带来以下好处：

### 生成的安全特性

当你在 Goa 设计中定义安全要求时，框架会生成完整的安全中间件来处理所有关键的认证任务。该中间件会自动从入站请求中提取凭据、根据你定义的要求进行验证、管理认证错误，并强制执行你为 OAuth2 或 JWT 指定的任何作用域要求。

安全定义也会反映在生成的 OpenAPI/Swagger 文档中，使 API 使用者能轻松理解你的认证要求。文档会清楚地说明支持的认证方法、不同端点所需的作用域、凭据的预期格式，以及客户端需要处理的错误响应。

为确保类型安全与可维护性，Goa 会生成与安全相关的强类型接口与结构体。这包括用于实现安全处理器的类型安全接口、与安全方案匹配的强类型凭据结构体、用于从请求上下文访问安全信息的辅助函数，以及用于处理各种安全相关失败的预定义错误类型。类型安全有助于在编译期而非运行期捕获潜在的安全实现问题。

这意味着你可以专注于以声明式方式定义安全要求，而由 Goa 处理具体实现细节。生成的代码类型安全，并遵循 Go 的最佳实践。