---
title: JWT 认证
description: 学习如何在你的 Goa API 中实现 JWT 认证
weight: 3
---

[JSON Web Tokens（JWT）](https://jwt.io/introduction) 提供了一种在各方之间安全传输声明（claims）的方式。
它们在微服务架构中尤其有用，当你需要在服务之间传递认证与授权信息时。
JWT 是自包含的令牌，能够包含用户信息、权限以及其他声明。

## JWT 认证如何工作

1. 客户端认证并获得一个 JWT
2. 后续请求中包含该 JWT（通常在 `Authorization` 头中）
3. 服务器验证 JWT 的签名和声明
4. 如果有效，请求将在声明所表示的上下文中被处理

有关 JWT 认证流程的详细说明，请参阅
[JWT 认证流程指南](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow)。

## JWT 结构

一个 JWT 由三部分组成（可在 [JWT.io 调试器](https://jwt.io/#debugger-io) 中查看示例）：
1. 头部（算法与令牌类型）
2. 负载（声明）
3. 签名

示例 JWT：
```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

关于 JWT 声明的更多信息，请参阅
[JWT 声明文档](https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-token-claims)。

## 了解作用域（Scopes）

### 什么是作用域？

作用域是定义客户端在 API 上可以执行哪些操作的权限。
可以将作用域视为实现细粒度访问控制的方式。例如：
- 移动应用可能只有 `read` 作用域来查看数据
- 管理后台可能同时拥有 `read` 和 `write` 作用域
- 备份服务可能具有 `backup` 作用域

### 作用域如何工作

1. **定义**：在你的安全方案中定义作用域
2. **分配**：在生成令牌时，包含被授予的作用域
3. **验证**：处理请求时，验证令牌是否包含所需作用域

这里有一个现实类比：
- 酒店的房卡（JWT）可能具备不同的访问级别（作用域）：
  - `room:access` - 仅可进入你的房间
  - `pool:access` - 可进入游泳池
  - `gym:access` - 可进入健身房
  - `all:access` - 对所有设施的完全访问

### 作用域格式

作用域通常遵循类似 `resource:action` 的模式。常见示例：
```
api:read        # 对 API 的只读访问
api:write       # 对 API 的写入访问
users:create    # 创建用户的能力
admin:*         # 完全的管理员访问
```

### 作用域继承

作用域可以是分层的。例如：
- 如果一个方法需要 `api:read`，则具有 `admin:*` 的令牌也可能有效
- 如果一个方法需要多个作用域，则令牌必须包含所有所需作用域

作用域层级示例：
```
admin:*           # 完全的管理员访问（包含所有管理员作用域）
├── admin:read    # 读取管理员资源
├── admin:write   # 修改管理员资源
└── admin:delete  # 删除管理员资源
```

### 在 Goa 中实现作用域

#### 1. 定义可用的作用域

首先，定义你的 API 中存在哪些作用域：

```go
var JWTAuth = JWTSecurity("jwt", func() {
    Description("带作用域的 JWT 认证")
    
    // 定义所有可用的作用域
    Scope("api:read", "读取 API 资源的访问")
    Scope("api:write", "写入 API 资源的访问")
    Scope("api:admin", "完全的管理访问")
    Scope("users:read", "读取用户资料")
    Scope("users:write", "修改用户资料")
})
```

#### 2. 将作用域应用到方法

接着，为每个端点指定所需的作用域：

```go
var _ = Service("users", func() {
    // 列出用户 - 需要读取访问
    Method("list", func() {
        Security(JWTAuth, func() {
            // 仅需要读取访问
            Scope("users:read")
        })
    })
    
    // 更新用户 - 需要写入访问
    Method("update", func() {
        Security(JWTAuth, func() {
            // 同时需要读取和写入访问
            Scope("users:read", "users:write")
        })
    })
    
    // 删除用户 - 需要管理员访问
    Method("delete", func() {
        Security(JWTAuth, func() {
            Scope("api:admin")
        })
    })
})
```

#### 3. 在令牌中包含作用域

生成令牌时，包含被授予的作用域：

```go
func GenerateUserToken(user *User) (string, error) {
    // 根据用户角色确定作用域
    var scopes []string
    switch user.Role {
    case "admin":
        scopes = []string{"api:admin", "users:read", "users:write"}
    case "editor":
        scopes = []string{"users:read", "users:write"}
    default:
        scopes = []string{"users:read"}
    }
    
    claims := Claims{
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: time.Now().Add(time.Hour * 24).Unix(),
            IssuedAt:  time.Now().Unix(),
            Subject:   user.ID,
        },
        Scopes: scopes,  // 在令牌中包含作用域
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(jwtSecret))
}
```

#### 4. 验证作用域

处理请求时，验证令牌是否具有所需的作用域：

```go
func validateScopes(tokenScopes []string, requiredScopes []string) error {
    // 创建令牌作用域的映射以便高效查找
    scopeMap := make(map[string]bool)
    for _, scope := range tokenScopes {
        scopeMap[scope] = true
    }
    
    // 特殊情况：admin 作用域授予所有访问
    if scopeMap["api:admin"] {
        return nil
    }
    
    // 检查每个所需的作用域
    for _, required := range requiredScopes {
        if !scopeMap[required] {
            return fmt.Errorf("missing required scope: %s", required)
        }
    }
    
    return nil
}
```

### 作用域最佳实践

1. **命名约定**
   - 使用一致的模式（`resource:action`）
   - 名称保持小写并使用冒号分隔
   - 描述清晰但简洁

2. **粒度**
   - 使作用域足够具体以实现细粒度控制
   - 但不要过于具体以至于难以管理
   - 考虑将相关操作分组

3. **文档**
   - 记录每个作用域允许的内容
   - 提供使用每个作用域的示例
   - 解释任何作用域的层级结构

4. **安全性**
   - 始终在服务器端验证作用域
   - 不要信任客户端的作用域检查
   - 将作用域与令牌的过期策略结合考虑

5. **管理**
   - 对敏感操作实施作用域轮换
   - 监控作用域使用情况
   - 定期审计作用域分配

## 在 Goa 中实现 JWT 认证

### 1. 定义安全方案

首先，在你的设计包中定义 JWT 安全方案。

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

// JWTAuth 定义安全方案
var JWTAuth = JWTSecurity("jwt", func() {
    Description("JWT 认证")
    
    // 定义用于授权的作用域
    Scope("api:read", "读取 API 的访问")
    Scope("api:write", "写入 API 的访问")
})
```

### 2. 应用安全方案

JWT 认证可以在不同层级应用，并指定特定的作用域要求。

```go
// API 级别 - 应用于所有服务与方法
var _ = API("secure_api", func() {
    Security(JWTAuth, func() {
        Scope("api:read")  // 默认的最低作用域
    })
})

// 服务级别 - 应用于该服务内的所有方法
var _ = Service("secure_service", func() {
    Security(JWTAuth, func() {
        Scope("api:write")  // 需要写入作用域
    })
})

// 方法级别 - 仅应用于此方法
Method("secure_method", func() {
    Security(JWTAuth, func() {
        Scope("api:read", "api:write")  // 同时需要两个作用域
    })
})
```

### 3. 定义负载（Payload）

对于使用 JWT 认证的方法，在负载中包含令牌。

```go
Method("getData", func() {
    Security(JWTAuth, func() {
        Scope("api:read")
    })
    
    Payload(func() {
        Token("token", String, func() {
            Description("用于认证的 JWT")
        })
        Required("token")
        
        // 其他负载字段
        Field(1, "query", String, "搜索查询")
    })
    
    Result(ArrayOf(String))
    
    HTTP(func() {
        GET("/data")
        // 将令牌映射到 Authorization 头
        Header("token:Authorization")
    })
})
```

### 4. 实现安全处理器

当 Goa 生成代码后，你需要实现一个 JWT 安全处理器。下面的示例使用
[golang-jwt/jwt](https://github.com/golang-jwt/jwt) 库，这是 Go 推荐使用的 JWT 库。

```go
// SecurityJWTFunc 实现 JWT 认证的授权逻辑
func (s *service) JWTAuth(ctx context.Context, token string, 
    scheme *security.JWTScheme) (context.Context, error) {
    
    // 解析并验证 JWT
    claims, err := s.parseAndValidateJWT(token)
    if err != nil {
        return ctx, jwt.Unauthorized("invalid token")
    }
    
    // 验证所需的作用域
    if !hasRequiredScopes(claims.Scopes, scheme.RequiredScopes) {
        return ctx, jwt.Unauthorized("insufficient scopes")
    }
    
    // 将声明添加到上下文
    ctx = context.WithValue(ctx, "jwt_claims", claims)
    return ctx, nil
}

func (s *service) parseAndValidateJWT(token string) (*Claims, error) {
    // 使用你偏好的库解析 JWT
    // 以下示例使用 golang-jwt/jwt：
    claims := &Claims{}
    parsedToken, err := jwt.ParseWithClaims(token, claims, 
        func(token *jwt.Token) (interface{}, error) {
            // 验证签名算法
            if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v", 
                    token.Header["alg"])
            }
            return []byte(s.config.JWTSecret), nil
        })
    
    if err != nil || !parsedToken.Valid {
        return nil, err
    }
    return claims, nil
}

// Claims 定义你的自定义 JWT 声明
type Claims struct {
    jwt.StandardClaims
    UserID string   `json:"uid"`
    Scopes []string `json:"scopes"`
}

func hasRequiredScopes(tokenScopes, requiredScopes []string) bool {
    scopeMap := make(map[string]bool)
    for _, scope := range tokenScopes {
        scopeMap[scope] = true
    }
    
    for _, required := range requiredScopes {
        if !scopeMap[required] {
            return false
        }
    }
    return true
}
```

## JWT 认证最佳实践

关于 JWT 安全最佳实践的全面指南，请参阅
[OWASP JWT 安全备忘单](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)。

### 1. 令牌生成

生成包含适当声明与过期时间的 JWT。关于 JWT 签名方法的更多信息，请参阅
[JWT 签名算法概览](https://auth0.com/docs/secure/tokens/json-web-tokens/json-web-token-signing-algorithms)。

```go
func GenerateJWT(userID string, scopes []string) (string, error) {
    claims := Claims{
        StandardClaims: jwt.StandardClaims{
            ExpiresAt: time.Now().Add(time.Hour * 24).Unix(),
            IssuedAt:  time.Now().Unix(),
            Issuer:    "your-api",
        },
        UserID: userID,
        Scopes: scopes,
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(jwtSecret))
}
```

### 2. 令牌验证

按照 [JWT 最佳实践 RFC](https://datatracker.ietf.org/doc/html/rfc8725) 实现全面的令牌验证：

```go
func ValidateToken(tokenString string) (*Claims, error) {
    // 解析令牌
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, 
        func(token *jwt.Token) (interface{}, error) {
            // 验证签名算法
            if _, ok := token.Method.(*jwt.SigningMethodHS256); !ok {
                return nil, fmt.Errorf("unexpected signing method: %v", 
                    token.Header["alg"])
            }
            return []byte(jwtSecret), nil
        })
    
    if err != nil {
        return nil, err
    }
    
    // 类型断言声明
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        // 额外验证
        if err := validateCustomClaims(claims); err != nil {
            return nil, err
        }
        return claims, nil
    }
    
    return nil, fmt.Errorf("invalid token")
}

func validateCustomClaims(claims *Claims) error {
    // 验证发行者
    if claims.Issuer != "your-api" {
        return fmt.Errorf("invalid issuer")
    }
    
    // 验证其他自定义要求
    return nil
}
```

### 3. 令牌刷新

实现令牌刷新以维持用户会话。有关刷新令牌的更多信息，请参阅
[Auth0 刷新令牌指南](https://auth0.com/docs/secure/tokens/refresh-tokens)。

```go
Method("refresh", func() {
    Description("刷新现有的 JWT 令牌")
    
    Security(JWTAuth)
    
    Payload(func() {
        Token("token", String)
        Required("token")
    })
    
    Result(func() {
        Field(1, "token", String, "新的 JWT 令牌")
        Field(2, "expires_at", String, "令牌过期时间")
        Required("token", "expires_at")
    })
    
    HTTP(func() {
        POST("/auth/refresh")
        Response(StatusOK)
        Response(StatusUnauthorized)
    })
})
```

## 生成的代码

Goa 会为 JWT 认证生成多个组件：

1. **安全类型**
   - JWT 令牌类型
   - 作用域验证
   - 错误类型

2. **中间件**
   - 令牌提取
   - 作用域验证
   - 错误处理

3. **OpenAPI 文档**
   - 安全方案
   - 作用域要求
   - 错误响应

## 常见问题与解决方案

### 1. 令牌验证错误

常见的令牌验证问题：
- 令牌过期
- 签名无效
- 算法不正确
- 缺少必需的声明

解决方案：实施全面验证：

```go
func validateToken(token *jwt.Token) error {
    if err := validateSignature(token); err != nil {
        return err
    }
    if err := validateExpiration(token); err != nil {
        return err
    }
    if err := validateClaims(token); err != nil {
        return err
    }
    return nil
}
```

### 2. 作用域验证

确保正确的作用域检查：

```go
func validateScopes(tokenScopes []string, requiredScopes []string) error {
    scopeMap := make(map[string]bool)
    for _, scope := range tokenScopes {
        scopeMap[scope] = true
    }
    
    for _, required := range requiredScopes {
        if !scopeMap[required] {
            return fmt.Errorf("missing required scope: %s", required)
        }
    }
    return nil
}
```

### 3. 令牌刷新策略

实现稳健的刷新策略：

```go
func refreshToken(oldToken string) (string, error) {
    // 验证旧令牌
    claims, err := validateToken(oldToken)
    if err != nil {
        return "", err
    }
    
    // 检查是否允许刷新
    if time.Unix(claims.ExpiresAt, 0).Sub(time.Now()) > 
        time.Hour*24*7 {
        return "", fmt.Errorf("token too old to refresh")
    }
    
    // 生成新令牌
    return GenerateJWT(claims.UserID, claims.Scopes)
}
```

## 下一步

- 学习 [OAuth2 认证](4-oauth2.md)
- 探索 [API Key 认证](2-api-key.md)
- 阅读 [安全最佳实践](5-best-practices.md)