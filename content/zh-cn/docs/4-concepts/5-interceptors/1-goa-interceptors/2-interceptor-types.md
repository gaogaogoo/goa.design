---
title: "拦截器类型"
description: "了解不同类型的 Goa 拦截器及其使用场景"
weight: 2
---

Goa 支持多种拦截器以应对不同场景。本文解释各类拦截器及其适用时机。

## 核心概念

在设计拦截器时，需要考虑三个关键维度：

1. 服务端 vs 客户端：
   - 服务端拦截器运行在服务实现侧
   - 客户端拦截器运行在生成的客户端中

2. Payload vs Result 访问：
   - Payload：访问/修改入站请求
   - Result：访问/修改出站响应

3. 读 vs 写 访问：
   - 读：在不修改的前提下检查数据
   - 写：修改或丰富数据

拦截器仅需按名称引用其要访问的属性——不需要重新定义完整的属性类型或描述。方法设计必须在其 Payload 与 Result 类型中包含这些属性。

## 基本模式

### 只读访问（Read-Only Access）

当需要检查但不修改数据时使用。适用于监控、日志与校验：

```go
var Monitor = Interceptor("Monitor", func() {
    Description("在不修改数据的前提下采集指标")
    
    // 从 Payload 读取请求大小
    ReadPayload(func() {
        Attribute("size")        // 类型与描述源自 Payload 类型
    })
    
    // 从 Result 读取响应状态
    ReadResult(func() {
        Attribute("status")      // 类型与描述源自 Result 类型
    })
})
```

`ReadPayload` 与 `ReadResult` DSL 声明对 Payload 与 Result 属性的只读访问：
- 拦截器仅需列出希望访问的属性名
- 属性类型与描述从方法的 Payload 与 Result 类型继承
- 同一个 `ReadPayload` 或 `ReadResult` 块可列出多个属性
- 拦截器实现以只读字段的形式接收这些属性

### 写访问（Write Access）

当拦截器需要修改或添加数据时使用：

```go
var Enricher = Interceptor("Enricher", func() {
    Description("为请求与响应添加上下文信息")
    
    // 向 Payload 添加请求 ID
    WritePayload(func() {
        Attribute("requestID")   // 必须在 Payload 类型中定义
    })
    
    // 向 Result 添加计时信息
    WriteResult(func() {
        Attribute("processedAt") // 必须在 Result 类型中定义
    })
})
```

`WritePayload` 与 `WriteResult` DSL 声明写访问：
- 列出的属性可由拦截器实现进行修改
- 方法的 Payload 与 Result 类型必须包含这些属性
- 如有需要可定义多个写块
- 写访问隐含对相同属性的读访问

### 组合访问（Combined Access）

当拦截器同时需要读与写访问时，可组合使用：

```go
var DataProcessor = Interceptor("DataProcessor", func() {
    Description("同时处理请求与响应")
    
    // 转换请求数据
    ReadPayload(func() {
        Attribute("rawData")     // 来自 Payload 的输入数据
        Attribute("format")      // 当前格式
    })
    WritePayload(func() {
        Attribute("processed")   // 转换后的数据
        Attribute("newFormat")   // 新格式
    })
    
    // 转换响应数据
    ReadResult(func() {
        Attribute("status")      // 响应状态
        Attribute("data")        // 响应数据
    })
    WriteResult(func() {
        Attribute("enriched")    // 丰富后的响应
        Attribute("metadata")    // 添加的元数据
    })
})
```

组合访问的要点：
- 读与写块可在 Payload 与 Result 上自由混合
- 每个块可列出多个属性
- 同一属性可同时出现在读与写块中
- 块的顺序不影响实现

## 服务端拦截器

服务端拦截器在服务实现侧执行：于请求解码之后、调用服务方法之前。适合用于日志、指标采集、请求丰富与响应转换等横切关注点。

以下为一个针对 GET 请求进行响应缓存的服务端拦截器示例：

```go
var Cache = Interceptor("Cache", func() {
    Description("为 GET 请求实现响应缓存")
    
    // 读取记录 ID 作为缓存键
    ReadPayload(func() {
        Attribute("recordID")    // 来自 Payload 类型的 UUID
    })
    
    // 向响应添加缓存元数据
    WriteResult(func() {
        Attribute("cachedAt")    // 来自 Result 类型的字符串
        Attribute("ttl")         // 来自 Result 类型的整型
    })
})
```

该服务端拦截器展示了：
- 如何将对 Payload 的只读访问与对 Result 的写访问进行组合
- 拦截器可应用在服务级
- DSL 中的属性声明与实现逻辑的分离
- 属性类型由方法定义，而非拦截器定义

服务设计必须包含这些属性：

```go
var _ = Service("catalog", func() {
    // 将缓存应用于服务的所有方法
    ServerInterceptor(Cache)
    
    Method("get", func() {
        Payload(func() {
            // 定义 Cache 拦截器所需属性
            Attribute("recordID", UUID, "用作缓存键的记录标识")
        })
        Result(func() {
            // 定义 Cache 拦截器所需属性
            Attribute("cachedAt", String, "响应被缓存的时间")
            Attribute("ttl", Int, "存活时间（秒）")
            // 其他结果字段...
        })
        HTTP(func() {
            GET("/{recordID}")
            Response(StatusOK)
        })
    })
})
```

## 客户端拦截器

客户端拦截器在请求发往服务端之前于客户端侧执行。它们用于实现请求丰富、响应处理与客户端缓存等行为。

以下示例为一个添加客户端上下文并跟踪速率限制的客户端拦截器：

```go
var ClientContext = Interceptor("ClientContext", func() {
    Description("为请求添加客户端上下文并跟踪速率限制")
    
    // 向出站请求添加客户端上下文
    WritePayload(func() {
        Attribute("clientVersion")  // 来自 Payload 类型的字符串
        Attribute("clientID")       // 来自 Payload 类型的 UUID
        Attribute("region")         // 来自 Payload 类型的字符串
    })
    
    // 从响应中跟踪速率限制信息
    ReadResult(func() {
        Attribute("rateLimit")           // 来自 Result 类型
        Attribute("rateLimitRemaining")  // 来自 Result 类型
        Attribute("rateLimitReset")      // 来自 Result 类型
    })
})
```

该客户端拦截器说明：
- 客户端拦截器如何使用 `WritePayload` 修改出站请求
- 如何使用 `ReadResult` 读取响应数据
- 相同的 DSL 模式同时适用于客户端与服务端拦截器
- 在方法设计中声明所有需要的属性非常重要

服务必须定义这些属性：

```go
var _ = Service("inventory", func() {
    // 确保所有客户端调用都包含上下文信息
    ClientInterceptor(ClientContext)
    
    Method("list", func() {
        Payload(func() {
            // 业务属性
            Attribute("page", Int, "页码")
            Attribute("perPage", Int, "每页条数")
            
            // ClientContext 拦截器所需
            Attribute("clientVersion", String, "客户端库版本")
            Attribute("clientID", UUID, "该客户端实例的唯一标识")
            Attribute("region", String, "客户端所在地域")
        })
        Result(func() {
            // 业务属性
            Attribute("items", ArrayOf(Item))
            
            // ClientContext 拦截器所需
            Attribute("rateLimit", Int, "当前速率限制")
            Attribute("rateLimitRemaining", Int, "当前窗口剩余请求数")
            Attribute("rateLimitReset", Int, "速率限制窗口重置时间")
        })
    })
})
```

## 流式拦截器

流式拦截器处理 Payload、Result 或两者为消息流的流式方法。它们使用专门的流式访问模式：

- `ReadStreamingPayload`/`WriteStreamingPayload`：用于客户端流
- `ReadStreamingResult`/`WriteStreamingResult`：用于服务端流

以下示例展示不同的流式拦截器模式：

```go
// 服务端拦截器：写入流式结果
var ServerProgressTracker = Interceptor("ServerProgressTracker", func() {
    Description("为服务端流响应添加进度信息")
    
    WriteStreamingResult(func() {
        Attribute("percentComplete")  // 来自流式结果类型的 Float32
        Attribute("itemsProcessed")   // 来自流式结果类型的 Int
    })
})

// 客户端拦截器：写入流式请求
var ClientMetadataEnricher = Interceptor("ClientMetadataEnricher", func() {
    Description("为客户端流消息添加元数据")
    
    WriteStreamingPayload(func() {
        Attribute("clientTimestamp")  // 来自流式请求类型
        Attribute("clientRegion")     // 来自流式请求类型
    })
})
```

流式拦截器 DSL 引入了专用模式：
- 客户端流使用 `ReadStreamingPayload`/`WriteStreamingPayload`
- 服务端流使用 `ReadStreamingResult`/`WriteStreamingResult`
- 行为与非流式版本一致
- 区别在于它们作用于流中的每条消息
- 属性声明规则相同：仅列出名称，类型来源于方法

使用流式拦截器的服务示例：

```go
var _ = Service("fileProcessor", func() {
    // 服务端流示例
    Method("processFile", func() {
        Description("处理文件并返回进度更新")
        Payload(FileRequest)              // 单请求
        StreamingResult(func() {          // 多响应
            // 业务字段
            Attribute("data", Bytes)
            
            // ServerProgressTracker 所需
            Attribute("percentComplete", Float32)
            Attribute("itemsProcessed", Int)
        })
        ServerInterceptor(ServerProgressTracker)
    })
    
    // 客户端流示例
    Method("uploadFile", func() {
        Description("分块上传文件")
        StreamingPayload(func() {         // 多请求
            // 业务字段
            Attribute("chunk", Bytes)
            
            // ClientMetadataEnricher 所需
            Attribute("clientTimestamp", Int)
            Attribute("clientRegion", String)
        })
        Result(UploadResult)              // 单响应
        ClientInterceptor(ClientMetadataEnricher)
    })
})
```

## 下一步

- 学习[拦截器实现](3-interceptor-implementation)的细节与模式
- 了解实现拦截器的[最佳实践](../4-best-practices)