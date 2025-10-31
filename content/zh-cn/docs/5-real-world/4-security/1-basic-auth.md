---
title: 基本认证
description: 了解如何在您的 Goa API 中实现基本认证
weight: 1
---

基本认证是 HTTP 协议内置的一种简单认证方案。
虽然它是最简单的认证形式之一，但它仍然被广泛使用，
尤其适用于内部 API 或开发环境。

## 基本认证如何工作

使用基本认证时：

1. 客户端将用户名和密码用冒号连接起来 (username:password)
2. 然后该字符串被 base64 编码
3. 编码后的字符串在 Authorization 头中发送：
   `Authorization: Basic base64(username:password)`

## 在 Goa 中实现基本认证

### 1. 定义安全方案

首先，在您的设计包中定义您的基本认证安全方案：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

// BasicAuth 定义了我们的安全方案
var BasicAuth = BasicAuthSecurity("basic", func() {
    Description("使用您的用户名和密码访问 API")
})
```

### 2. 应用安全方案

您可以在不同级别应用基本认证：

```go
// API 级别 - 应用于所有服务和方法
var _ = API("secure_api", func() {
    Security(BasicAuth)
})

// 服务级别 - 应用于服务中的所有方法
var _ = Service("secure_service", func() {
    Security(BasicAuth)
})

// 方法级别 - 仅应用于此方法
Method("secure_method", func() {
    Security(BasicAuth)
})
```

### 3. 定义有效负载

对于使用基本认证的方法，您需要定义有效负载以包含用户名
和密码字段：

```go
Method("login", func() {
    Security(BasicAuth)
    Payload(func() {
        // Goa 会识别这些特殊的 DSL 函数
        Username("username", String, "用于认证的用户名")
        Password("password", String, "用于认证的密码")
        Required("username", "password")
    })
    Result(String)
    HTTP(func() {
        POST("/login")
        // Response 定义了成功认证后发生的情况
        Response(StatusOK)
    })
})
```

### 4. 实现安全处理器

当 Goa 生成代码时，您需要实现一个安全处理器。这是一个
示例：

```go
// SecurityBasicAuthFunc 实现基本认证的授权逻辑
func (s *service) BasicAuth(ctx context.Context, user, pass string) (context.Context, error) {
    // 在此处实现您的认证逻辑
    if user == "admin" && pass == "secret" {
        // 认证成功
        return ctx, nil
    }
    // 认证失败
    return ctx, basic.Unauthorized("无效的凭据")
}
```

## 基本认证的最佳实践

1. **始终使用 HTTPS**
   基本认证以 base64 编码（未加密）发送凭据。始终使用 HTTPS 来
   保护传输中的凭据。

2. **安全密码存储**
   - 切勿以纯文本形式存储密码
   - 使用强哈希算法（如 bcrypt）
   - 为密码哈希添加盐
   - 考虑使用安全的密码管理库

3. **速率限制**
   实施速率限制以防止暴力攻击：

   ```go
   var _ = Service("secure_service", func() {
       Security(BasicAuth)

       // 添加速率限制注解
       Meta("ratelimit:limit", "60")
       Meta("ratelimit:window", "1m")
   })
   ```

4. **错误消息**
   不要透露是用户名还是密码不正确。使用通用消息：

   ```go
   return ctx, basic.Unauthorized("无效的凭据")
   ```

5. **日志记录**
   记录认证尝试，但切勿记录密码：

   ```go
   func (s *service) BasicAuth(ctx context.Context, user, pass string) (context.Context, error) {
       // 好的：只记录用户名和结果
       log.Printf("用户 %s 的认证尝试", user)

       // 坏的：永远不要这样做
       // log.Printf("密码尝试：%s", pass)
   }
   ```

## 示例实现

这是一个完整的示例，展示了如何在 Goa 服务中实现基本认证：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

var BasicAuth = BasicAuthSecurity("basic", func() {
    Description("用于 API 访问的基本认证")
})

var _ = API("secure_api", func() {
    Title("安全 API 示例")
    Description("演示基本认证的 API")

    // 默认对所有端点应用基本认证
    Security(BasicAuth)
})

var _ = Service("secure_service", func() {
    Description("需要认证的安全服务")

    Method("getData", func() {
        Description("获取受保护的数据")

        // 定义安全要求
        Security(BasicAuth)

        // 定义有效负载（凭据将自动添加）
        Payload(func() {
            // 在此处添加任何其他有效负载字段
            Field(1, "query", String, "搜索查询")
        })

        // 定义结果
        Result(ArrayOf(String))

        // 定义 HTTP 传输
        HTTP(func() {
            GET("/data")
            Response(StatusOK)
            Response(StatusUnauthorized, func() {
                Description("无效的凭据")
            })
        })
    })

    // 公共端点的示例
    Method("health", func() {
        Description("健康检查端点")
        NoSecurity()
        Result(String)
        HTTP(func() {
            GET("/health")
        })
    })
})
```

## 生成的代码

Goa 为基本认证生成了几个组件：

1. **安全类型**
   - 凭据类型
   - 认证失败的错误类型

2. **中间件**
   - 从请求中提取凭据
   - 调用您的安全处理器
   - 处理认证错误

3. **OpenAPI 文档**
   - 记录安全要求
   - 显示必填字段
   - 记录错误响应

## 常见问题和解决方案

### 1. 未发送凭据

如果未发送凭据，请检查：
- `Authorization` 头的格式
- Base64 编码
- 特殊字符的 URL 编码

### 2. 总是收到未授权

常见原因：
- 设计中缺少 `Security()`
- 安全处理器的实现不正确
- 中间件顺序问题

### 3. CORS 问题

对于基于浏览器的客户端，请确保正确的 CORS 配置：

```go
var _ = Service("secure_service", func() {
    HTTP(func() {
        // 在 CORS 中允许凭据
        Meta("cors:expose_headers", "Authorization")
        Meta("cors:allow_credentials", "true")
    })
})
```

## 后续步骤

- 了解 [API 密钥认证](2-api-key.md)
- 探索 [JWT 认证](3-jwt.md)
- 理解 [OAuth2 认证](4-oauth2.md)
- 阅读有关[安全最佳实践](5-best-practices.md)