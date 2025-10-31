---
title: API 密钥认证
description: 了解如何在您的 Goa API 中实现 API 密钥认证
weight: 2
---

API 密钥认证是保护 API 的一种简单而流行的方法。它涉及
向客户端分发唯一的密钥，然后客户端在其请求中包含这些密钥。
此方法对于您希望跟踪使用情况、
实施速率限制或为不同客户端提供不同访问级别的公共 API 特别有用。

## API 密钥认证如何工作

API 密钥可以通过多种方式传输：
1. 作为标头（最常见）
2. 作为查询参数
3. 在请求正文中

最安全的方法是使用标头，通常名称类似于 `X-API-Key`
或 `Authorization`。

## 在 Goa 中实现 API 密钥认证

### 1. 定义安全方案

首先，在您的设计包中定义您的 API 密钥安全方案：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

// APIKeyAuth 定义了我们的安全方案
var APIKeyAuth = APIKeySecurity("api_key", func() {
    Description("API 密钥安全")
    Header("X-API-Key")  // 指定标头名称
})
```

您也可以使用查询参数代替标头：

```go
var APIKeyAuth = APIKeySecurity("api_key", func() {
    Description("API 密钥安全")
    Query("api_key")  // 指定查询参数名称
})
```

### 2. 应用安全方案

与其他安全方案一样，API 密钥认证可以应用于不同的级别：

```go
// API 级别 - 应用于所有服务和方法
var _ = API("secure_api", func() {
    Security(APIKeyAuth)
})

// 服务级别 - 应用于服务中的所有方法
var _ = Service("secure_service", func() {
    Security(APIKeyAuth)
})

// 方法级别 - 仅应用于此方法
Method("secure_method", func() {
    Security(APIKeyAuth)
})
```

### 3. 定义有效负载

对于使用 API 密钥认证的方法，请在有效负载中包含密钥：

```go
Method("getData", func() {
    Security(APIKeyAuth)
    Payload(func() {
        APIKey("api_key", "key", String, func() {
            Description("用于认证的 API 密钥")
            Example("abcdef123456")
        })
        Required("key")

        // 其他有效负载字段
        Field(1, "query", String, "搜索查询")
    })
    Result(ArrayOf(String))
    Error("unauthorized")
    HTTP(func() {
        GET("/data")
        // 将密钥映射到标头
        Header("key:X-API-Key")
        Response("unauthorized", StatusUnauthorized)
    })
})
```

### 4. 实现安全处理器

当 Goa 生成代码时，您需要实现一个安全处理器：

```go
// SecurityAPIKeyFunc 实现 API 密钥认证的授权逻辑
func (s *service) APIKeyAuth(ctx context.Context, key string) (context.Context, error) {
    // 在此处实现您的密钥验证逻辑
    valid, err := s.validateAPIKey(key)
    if err != nil {
        return ctx, err
    }
    if !valid {
        return ctx, genservice.MakeUnauthorized(fmt.Errorf("无效的 API 密钥"))
    }

    // 您可以将特定于密钥的数据添加到上下文中
    ctx = context.WithValue(ctx, "api_key_id", key)
    return ctx, nil
}

func (s *service) validateAPIKey(key string) (bool, error) {
    // 密钥验证的实现
    // 这可以根据数据库、缓存等进行检查。
    return key == "valid-key", nil
}
```

## API 密钥认证的最佳实践

### 1. 密钥生成

生成强大的随机 API 密钥：

```go
func GenerateAPIKey() string {
    // 生成 32 个随机字节
    bytes := make([]byte, 32)
    if _, err := rand.Read(bytes); err != nil {
        panic(err)
    }
    // 编码为 base64
    return base64.URLEncoding.EncodeToString(bytes)
}
```

### 2. 密钥存储

安全地存储 API 密钥：
- 在存储之前对密钥进行哈希处理
- 使用安全的键值存储或数据库
- 实施密钥轮换机制

示例密钥存储模式：

```sql
CREATE TABLE api_keys (
    id UUID PRIMARY KEY,
    key_hash VARCHAR(64) NOT NULL,
    client_id UUID NOT NULL,
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP,
    last_used_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true
);
```

### 4. 密钥元数据

将元数据与 API 密钥关联以实现更好的控制：

```go
type APIKeyMetadata struct {
    ClientID    string
    Plan        string    // 例如，"free"、"premium"
    Permissions []string  // 例如，["read", "write"]
    ExpiresAt   time.Time
}

func (s *service) APIKeyAuth(ctx context.Context, key string) (context.Context, error) {
    metadata, err := s.getAPIKeyMetadata(key)
    if err != nil {
        return ctx, err
    }

    // 将元数据添加到上下文
    ctx = context.WithValue(ctx, "api_key_metadata", metadata)
    return ctx, nil
}
```

## 示例实现

这是一个完整的示例，展示了如何在 Goa 服务中实现 API 密钥认证：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

var APIKeyAuth = APIKeySecurity("api_key", func() {
    Description("使用 API 密钥进行认证")
    Header("X-API-Key")
})

var _ = API("weather_api", func() {
    Title("天气 API")
    Description("带有 API 密钥认证的天气预报 API")

    // 默认应用 API 密钥认证
    Security(APIKeyAuth)
})

var _ = Service("weather", func() {
    Description("天气预报服务")

    Method("forecast", func() {
        Description("获取天气预报")

        Payload(func() {
            // API 密钥将自动包含
            Field(1, "location", String, "获取预报的位置")
            Field(2, "days", Int, "预报的天数")
            Required("location")
        })

        Result(func() {
            Field(1, "location", String, "位置")
            Field(2, "forecast", ArrayOf(WeatherDay))
        })

        HTTP(func() {
            GET("/forecast/{location}")
            Param("days")
            Response(StatusOK)
            Response(StatusUnauthorized, func() {
                Description("无效或丢失的 API 密钥")
            })
            Response(StatusTooManyRequests, func() {
                Description("超出速率限制")
            })
        })
    })

    // 公共端点示例
    Method("health", func() {
        Description("健康检查端点")
        NoSecurity()
        Result(String)
        HTTP(func() {
            GET("/health")
        })
    })
})

// WeatherDay 定义了单日天气预报
var WeatherDay = Type("WeatherDay", func() {
    Field(1, "date", String, "预报日期")
    Field(2, "temperature", Float64, "摄氏温度")
    Field(3, "conditions", String, "天气状况")
    Required("date", "temperature", "conditions")
})
```

## 生成的代码

Goa 为 API 密钥认证生成了几个组件：

1. **安全类型**
   - API 密钥的类型
   - 认证失败的错误类型

2. **中间件**
   - 从请求中提取 API 密钥
   - 调用您的安全处理器
   - 处理认证错误

3. **OpenAPI 文档**
   - 记录安全要求
   - 显示 API 密钥位置（标头/查询）
   - 记录错误响应

## 常见问题和解决方案

### 1. 未发送密钥

如果未正确发送 API 密钥，请检查：
- 标头名称是否完全匹配
- 密钥格式是否正确
- 客户端是否实际发送了密钥

### 2. 性能注意事项

对于高流量 API：
- 缓存 API 密钥验证结果
- 使用快速键值存储
- 实施密钥前缀以实现快速失效

示例缓存实现：

```go
func (s *service) APIKeyAuth(ctx context.Context, key string) (context.Context, error) {
    // 首先检查缓存
    if metadata, found := s.cache.Get(key); found {
        return context.WithValue(ctx, "api_key_metadata", metadata), nil
    }

    // 验证密钥并获取元数据
    metadata, err := s.validateAPIKey(key)
    if err != nil {
        return ctx, err
    }

    // 缓存结果
    s.cache.Set(key, metadata, time.Minute*5)
    return context.WithValue(ctx, "api_key_metadata", metadata), nil
}
```

## 后续步骤

- 了解 [JWT 认证](3-jwt.md)
- 探索 [OAuth2 认证](4-oauth2.md)
- 阅读有关[安全最佳实践](5-best-practices.md)