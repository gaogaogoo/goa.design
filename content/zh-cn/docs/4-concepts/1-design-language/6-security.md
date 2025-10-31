---
title: "安全"
linkTitle: "安全"
weight: 6
description: "使用 Goa 的安全 DSL 为服务定义认证与鉴权方案，涵盖 JWT、API Key、Basic Auth 与 OAuth2。"
---

## 安全概览

在保护 API 时，需要区分两个概念：

- 认证（AuthN）：验证客户端身份（“你是谁？”）
- 鉴权（AuthZ）：确定已认证客户端可执行的操作（“你被允许做什么？”）

Goa 提供 DSL 构造来同时定义服务的认证与鉴权要求。

## 安全方案（Security Schemes）

### JWT（JSON Web Token）

JWT 是一个开放标准（[RFC 7519](https://tools.ietf.org/html/rfc7519)），定义了使用 JSON 对象在各方之间安全传输信息的紧凑方式。JWT 通常同时用于认证与鉴权：

1. 认证：JWT 由受信任的签发方签名（使用密钥），本身即可证明持有者已通过认证
2. 鉴权：JWT 可携带声明（如用户角色或权限），服务据此进行授权决策

```go
var JWTAuth = JWTSecurity("jwt", func() {
    Description("JWT-based authentication and authorization")
    // 作用域（Scopes）定义可与 JWT 声明校验的权限
    Scope("api:read", "Read-only access")
    Scope("api:write", "Read and write access")
})
```

#### 关于作用域（Scopes）

作用域是表示客户端被允许执行的操作的命名权限。使用 JWT 时：

1. 认证服务器在签发 JWT 时包含授予的作用域
2. 服务在每个端点校验请求携带的作用域是否满足要求
3. 若 JWT 未包含所需作用域，请求将被拒绝

### API Key

API Key 是客户端随请求附带的简单字符串令牌。尽管常被称为“API Key 认证”，它更准确地说是授权机制：

- 不能真正证明身份（易被共享或窃取）
- 主要用于识别请求来源与实施限流
- 比 JWT 更简单，但安全性与灵活性较低

```go
var APIKeyAuth = APIKeySecurity("api_key", func() {
    Description("API key-based request authorization")
})
```

常见用途：
- 按客户端限流
- 使用情况追踪
- 简单的项目/团队识别
- 公开 API 的基础访问控制

### 基本认证（Basic Authentication）

Basic Auth 是内建于 HTTP 协议的简单认证方案：

- 客户端在每次请求中发送凭据（用户名/密码）
- 凭据以 Base64 编码但不加密（必须使用 HTTPS）
- 提供真实的认证，但不包含内建的鉴权机制

```go
var BasicAuth = BasicAuthSecurity("basic", func() {
    Description("Username/password authentication")
    // 作用域用于在认证成功后授予权限
    Scope("api:read", "Read-only access")
})
```

### OAuth2

OAuth2 是一个完整的授权框架，支持针对不同应用类型的多种流程。其将以下职责分离：

1. 认证（由授权服务器处理）
2. 授权（通过访问令牌授予特定权限）
3. 资源访问（使用访问令牌）

```go
var OAuth2Auth = OAuth2Security("oauth2", func() {
    // 定义 OAuth2 授权码流程的端点
    AuthorizationCodeFlow(
        "http://auth.example.com/authorize",  // 请求授权的地址
        "http://auth.example.com/token",      // 交换 code 为 token 的地址
        "http://auth.example.com/refresh",    // 刷新过期 token 的地址
    )
    // 定义可用权限
    Scope("api:read", "Read-only access")
    Scope("api:write", "Read and write access")
})
```

## 应用安全方案

安全方案可在不同层级应用：

### 方法级安全（Method Level Security）

为单个方法启用一个或多个安全方案：

```go
Method("secure_endpoint", func() {
    Security(JWTAuth, func() {
        Scope("api:read")
    })
    
    Payload(func() {
        TokenField(1, "token", String)
        Required("token")
    })
    
    HTTP(func() {
        GET("/secure")
        Response(StatusOK)
    })
})
```

### 多方案组合（Multiple Schemes）

组合多个安全方案以增强安全性：

```go
Method("doubly_secure", func() {
    Security(JWTAuth, APIKeyAuth, func() {
        Scope("api:write")
    })
    
    Payload(func() {
        TokenField(1, "token", String)
        APIKeyField(2, "api_key", "key", String)
        Required("token", "key")
    })
    
    HTTP(func() {
        POST("/secure")
        Param("key:k")  // 在查询参数中传递 API key
        Response(StatusOK)
    })
})
```

## 传输层特定配置

### HTTP 安全配置

配置凭据如何通过 HTTP 传输：

```go
Method("secure_endpoint", func() {
    Security(JWTAuth)
    Payload(func() {
        TokenField(1, "token", String)
        Required("token")
    })
    HTTP(func() {
        GET("/secure")
        Header("token:Authorization") // 在 Authorization 头中传递 JWT
        Response(StatusOK)
        Response("unauthorized", StatusUnauthorized)
    })
})
```

### gRPC 安全配置

为 gRPC 传输配置安全：

```go
Method("secure_endpoint", func() {
    Security(JWTAuth, APIKeyAuth)
    Payload(func() {
        TokenField(1, "token", String)
        APIKeyField(2, "api_key", "key", String)
        Required("token", "key")
    })
    GRPC(func() {
        Metadata(func() {
            Attribute("token:authorization")  // 在元数据中传递 JWT
            Attribute("api_key:x-api-key")   // 在元数据中传递 API key
        })
        Response(CodeOK)
        Response("unauthorized", CodeUnauthenticated)
    })
})
```

## 错误处理

一致地定义与安全相关的错误：

```go
Service("secure_service", func() {
    Error("unauthorized", String, "Invalid credentials")
    Error("forbidden", String, "Invalid scopes")
    
    HTTP(func() {
        Response("unauthorized", StatusUnauthorized)
        Response("forbidden", StatusForbidden)
    })
    
    GRPC(func() {
        Response("unauthorized", CodeUnauthenticated)
        Response("forbidden", CodePermissionDenied)
    })
})
```

## 最佳实践

{{< alert title="安全实现指引" color="primary" >}}
认证设计
- 针对场景选择合适的安全方案
- 实施正确的令牌校验
- 安全地存储凭据
- 生产环境必须使用 HTTPS

鉴权设计
- 定义清晰的作用域层级
- 使用细粒度权限
- 实施基于角色的访问控制（RBAC）
- 校验所有安全要求

通用建议
- 明确记录安全要求
- 实施完善的错误处理
- 使用安全的默认值
- 定期进行安全审计
{{< /alert >}}

## 安全的具体实现

当你在设计中定义安全方案时，Goa 会根据你的设计生成一个特定的 `Auther` 接口，服务需实现该接口。该接口为你指定的每种安全方案定义方法：

```go
// Auther defines the security requirements for the service.
type Auther interface {
    // BasicAuth implements the authorization logic for basic auth.
    BasicAuth(context.Context, string, string, *security.BasicScheme) (context.Context, error)
    
    // JWTAuth implements the authorization logic for JWT tokens.
    JWTAuth(context.Context, string, *security.JWTScheme) (context.Context, error)
    
    // APIKeyAuth implements the authorization logic for API keys.
    APIKeyAuth(context.Context, string, *security.APIKeyScheme) (context.Context, error)
    
    // OAuth2Auth implements the authorization logic for OAuth2.
    OAuth2Auth(context.Context, string, *security.OAuth2Scheme) (context.Context, error)
}
```

服务必须实现这些方法来处理认证/鉴权逻辑。以下示例展示具体实现：

### Basic Auth 实现

```go
// BasicAuth implements the authorization logic  for the "basic" security scheme.
func (s *svc) BasicAuth(ctx context.Context, user, pass string, scheme *security.BasicScheme) (context.Context, error) {
    if user != "goa" || pass != "rocks" {
        return ctx, ErrUnauthorized
    }
    // 将认证信息存入上下文，供后续使用
    ctx = contextWithAuthInfo(ctx, authInfo{
        user: user,
    })
    return ctx, nil
}
```

### JWT 实现

```go
// JWTAuth implements the authorization logic for the "jwt" security scheme.
func (s *svc) JWTAuth(ctx context.Context, token string, scheme *security.JWTScheme) (context.Context, error) {
    claims := make(jwt.MapClaims)
    
    // 解析并校验 JWT token
    _, err := jwt.ParseWithClaims(token, claims, func(_ *jwt.Token) (interface{}, error) { 
        return Key, nil 
    })
    if err != nil {
        return ctx, ErrInvalidToken
    }

    // 校验所需作用域
    if claims["scopes"] == nil {
        return ctx, ErrInvalidTokenScopes
    }
    scopes, ok := claims["scopes"].([]any)
    if !ok {
        return ctx, ErrInvalidTokenScopes
    }
    scopesInToken := make([]string, len(scopes))
    for _, scp := range scopes {
        scopesInToken = append(scopesInToken, scp.(string))
    }
    if err := scheme.Validate(scopesInToken); err != nil {
        return ctx, securedservice.InvalidScopes(err.Error())
    }

    // 将声明存入上下文
    ctx = contextWithAuthInfo(ctx, authInfo{
        claims: claims,
    })
    return ctx, nil
}
```

### API Key 实现

```go
// APIKeyAuth implements the authorization logic for service "secured_service"
// for the "api_key" security scheme.
func (s *securedServicesrvc) APIKeyAuth(ctx context.Context, key string, scheme *security.APIKeyScheme) (context.Context, error) {
    if key != "my_awesome_api_key" {
        return ctx, ErrUnauthorized
    }
    ctx = contextWithAuthInfo(ctx, authInfo{
        key: key,
    })
    return ctx, nil
}
```

### 创建 JWT 令牌

在实现签发令牌的登录端点时：

```go
// Signin creates a valid JWT token for authentication
func (s *svc) Signin(ctx context.Context, p *gensvc.SigninPayload) (*gensvc.Creds, error) {
    // 创建包含声明的 JWT 令牌
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, jwt.MapClaims{
        "nbf":    time.Date(2015, 10, 10, 12, 0, 0, 0, time.UTC).Unix(),
        "iat":    time.Now().Unix(),
        "scopes": []string{"api:read", "api:write"},
    })

    // 签名令牌
    t, err := token.SignedString(Key)
    if err != nil {
        return nil, err
    }
    
    return &gensvc.Creds{
        JWT:        t,
        OauthToken: t,
        APIKey:     "my_awesome_api_key",
    }, nil
}
```

### 工作机制
当你在 Goa 服务中实现安全方案后，认证与鉴权流程如下：

1. Goa 生成的端点包装器负责处理安全方案校验
2. 每个端点包装器调用你实现的对应认证函数
3. 你的认证函数校验凭据并返回增强的上下文
4. 若认证通过，端点处理器将以增强的上下文被调用
5. 若认证失败，则向客户端返回错误

例如，在多方案组合下：

```go
// Generated endpoint wrapper
func NewDoublySecureEndpoint(s Service, authJWTFn security.AuthJWTFunc, authAPIKeyFn security.AuthAPIKeyFunc) goa.Endpoint {
    return func(ctx context.Context, req any) (any, error) {
        p := req.(*DoublySecurePayload)
        
        // 先校验 JWT
        ctx, err = authJWTFn(ctx, p.Token, &sc)
        if err == nil {
            // 再校验 API key
            ctx, err = authAPIKeyFn(ctx, p.Key, &sc)
        }
        if err != nil {
            return nil, err
        }
        
        // 两次校验均通过后调用服务方法
        return s.DoublySecure(ctx, p)
    }
}
```
