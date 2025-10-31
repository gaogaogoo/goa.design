---
title: "跨域资源共享（CORS）"
description: "使用 CORS 插件在 Goa 服务中配置跨域策略"
weight: 4
---

## 跨域资源共享（CORS）

跨域资源共享（CORS）是由浏览器实现的安全特性，用于控制一个域中的网页如何请求并与不同域中的资源交互。在构建需要被不同域上的 Web 应用访问的 API 时，正确配置 CORS 至关重要。你可以在
[MDN Web Docs](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) 了解更多。

Goa 提供了强大的
[CORS 插件](https://github.com/goadesign/plugins/tree/v3/cors)，可轻松为服务定义与实现 CORS 策略。该插件会自动处理所有必要的 HTTP 头与预检（preflight）请求，让你专注于访问策略的定义。

## 设置 CORS 插件

要在 Goa 设计中使用 CORS 功能，需要同时引入 CORS 插件与标准 Goa DSL 包：

```go
import (
    cors "goa.design/plugins/v3/cors/dsl"
    . "goa.design/goa/v3/dsl"
)
```

## 定义 CORS 策略

可在 Goa 设计的两个层级定义 CORS 策略：

1. API 层级：策略全局作用于所有服务的全部端点
2. 服务层级：策略仅作用于特定服务内的端点

CORS 插件提供多种函数用于配置策略的各方面：

- `Origin`：指定允许访问 API 的来源（域名）
- `Methods`：定义允许的 HTTP 方法
- `Headers`：指定允许的 HTTP 头
- `Expose`：列出允许浏览器访问的响应头
- `MaxAge`：设置预检请求结果在浏览器中的缓存时长
- `Credentials`：允许发送 Cookie 与 HTTP 认证信息

### 示例：服务层级的 CORS 配置

以下示例展示在服务层级配置 CORS 的不同方式：

```go
var _ = Service("calc", func() {
    // 仅允许来自 "localhost" 的请求
    cors.Origin("localhost")

    // 允许 domain.com 的任意子域
    cors.Origin("*.domain.com", func() {
        cors.Headers("X-Shared-Secret", "X-Api-Version")
        cors.MaxAge(100)
        cors.Credentials()
    })

    // 允许任何来源
    cors.Origin("*")

    // 允许匹配正则表达式的来源
    cors.Origin("/.*domain.*/", func() {
        cors.Headers("*")
        cors.Methods("GET", "POST")
        cors.Expose("X-Time")
    })
})
```

### 完整设计示例

以下示例展示如何在计算器服务的 API 与服务层级同时实现 CORS：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
    cors "goa.design/plugins/v3/cors/dsl"
)

var _ = API("calc", func() {
    Title("CORS 示例 Calc API")
    Description("该 API 演示了 goa CORS 插件的使用")
    
    // API 层级的 CORS 策略
    cors.Origin("http://127.0.0.1", func() {
        cors.Headers("X-Shared-Secret")
        cors.Methods("GET", "POST")
        cors.Expose("X-Time")
        cors.MaxAge(600)
        cors.Credentials()
    })
})

var _ = Service("calc", func() {
    Description("calc 服务暴露了定义 CORS 策略的公共端点。")
    
    // 服务层级的 CORS 策略
    cors.Origin("/.*localhost.*/", func() {
        cors.Methods("GET", "POST")
        cors.Expose("X-Time", "X-Api-Version")
        cors.MaxAge(100)
    })

    Method("add", func() {
        // ... 按常规实现 ...
    })
})
```

## 工作原理

在设计中启用 CORS 插件后，Goa 会自动：

1. 生成处理来自浏览器的预检（OPTIONS）请求的 CORS 处理器
2. 根据你的策略定义为所有 HTTP 端点响应添加合适的 CORS 头
3. 处理将来源与模式或正则表达式进行匹配的复杂逻辑
4. 按设计管理缓存相关响应头与凭据设置

该插件负责所有底层的 CORS 实现细节，让你能够使用 DSL 在更高层次上专注于安全策略的定义。
