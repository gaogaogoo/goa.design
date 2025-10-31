---
title: "拦截器入门"
description: "学习如何创建与使用 Goa 拦截器"
weight: 1
---

本文将带你创建并使用第一个 Goa 拦截器。我们将创建一个简单的日志拦截器，用于记录方法调用的耗时。

## 定义拦截器

使用设计中的 `Interceptor` 函数定义拦截器。以下为一个简单的日志拦截器：

```go
var RequestLogger = Interceptor("RequestLogger", func() {
    Description("记录入站请求及其耗时")
    
    // 我们希望从结果中读取方法状态码
    ReadResult(func() {
        Attribute("status", Int, "返回的状态码") // 业务状态码，非 HTTP
    })
    
    // 我们将向结果中添加计时信息
    WriteResult(func() {
        Attribute("processedAt", String, "请求被处理的时间")
        Attribute("duration", Int, "处理耗时（毫秒）")
    })
})
```

`Interceptor` DSL 定义了名为 `RequestLogger` 的拦截器。结合 `ReadResult` 与 `WriteResult` 指定需要访问的结果字段——此处为读取结果状态码并写入计时信息。Goa 也提供用于读取与写入 Payload 的等效 DSL。

## 应用拦截器

拦截器可在服务级与方法级进行应用：

```go
var _ = Service("calculator", func() {
    // 应用于服务中的所有方法
    ServerInterceptor(RequestLogger)
    
    Method("add", func() {
        // 方法特定拦截器
        ServerInterceptor(ValidateNumbers)
        
        Payload(func() {
            Attribute("a", Int)
            Attribute("b", Int)
        })
        Result(Int)
    })
})
```

该示例展示了如何在服务设计中使用 `ServerInterceptor` 应用拦截器。既可在服务级（影响全部方法）应用，也可在方法级（仅影响该方法）应用。

这样即可在保持 Goa 生成代码类型安全的前提下，构建跨服务的统一计时日志系统。

## 实现拦截器

生成的代码会提供类型安全的接口用于实现你的拦截器。以下是日志拦截器的实现方式：

```go
func (i *ServerInterceptors) RequestLogger(ctx context.Context, info *RequestLoggerInfo, next goa.Endpoint) (any, error) {
    start := time.Now()
    
    // 调用下一个拦截器或最终端点
    res, err := next(ctx, info.RawPayload())
    if err != nil {
        return nil, err
    }
    
    // 通过类型安全接口访问结果
    r := info.Result(res)
    
    // 添加计时信息
    r.SetProcessedAt(time.Now().Format(time.RFC3339))
    r.SetDuration(int(time.Since(start).Milliseconds()))
    
    return res, nil
}
```

拦截器工作机制解析：

1. 函数签名遵循 Goa 的拦截器模式：
   - 接收上下文、类型安全的 info 对象与下一个端点
   - 返回结果与错误

2. 记录计时：

   ```go
   start := time.Now()
   ```

   记录请求开始时间

3. 调用下一个处理器：

   ```go
   res, err := next(ctx, info.RawPayload())
   ```

   - 执行下一个拦截器或最终端点
   - 透传原始 Payload
   - 出错则提前返回

4. 访问结果：
   ```go
   r := info.Result(res)
   ```
   使用生成的类型安全接口访问结果

5. 添加计时信息：
   ```go
   r.SetProcessedAt(time.Now().Format(time.RFC3339))
   r.SetDuration(int(time.Since(start).Milliseconds()))
   ```
   - 记录处理完成时间
   - 计算并写入总耗时
   - 使用生成的 Setter 以确保类型安全

6. 返回修改后的结果：
   ```go
   return res, nil
   ```
   将丰富后的响应传递回链路


## 使用拦截器

定义拦截器后，Goa 会生成必要代码将其接入你的服务。生成代码结构如下：

1. Goa 首先生成定义所有服务端拦截器的 `ServerInterceptors` 接口：

```go
// ServerInterceptors 定义所有服务端拦截器的接口
type ServerInterceptors interface {
    RequestLogger(ctx context.Context, info *RequestLoggerInfo, next goa.Endpoint) (any, error)
    // ... 其他拦截器 ...
}
```

2. 对每个拦截器，Goa 生成类型安全的 info 结构与接口：

```go
// Info 结构提供拦截相关的元信息
type RequestLoggerInfo struct {
    service    string
    method     string
    callType   goa.InterceptorCallType
    rawPayload any
}

// 类型安全的结果访问接口
type RequestLoggerResult interface {
    Status() int
    SetProcessedAt(string)
    SetDuration(int)
}
```

3. 在你的服务中实现 `ServerInterceptors` 接口：

```go
type interceptors struct {
    logger *log.Logger
}

func NewInterceptors(logger *log.Logger) *interceptors {
    return &interceptors{logger: logger}
}

func (i *interceptors) RequestLogger(ctx context.Context, info *RequestLoggerInfo, next goa.Endpoint) (any, error) {
    // 复用前文示例实现
    start := time.Now()
    res, err := next(ctx, info.RawPayload())
    if err != nil {
        return nil, err
    }
    
    r := info.Result(res)
    r.SetProcessedAt(time.Now().Format(time.RFC3339))
    r.SetDuration(int(time.Since(start).Milliseconds()))
    
    return res, nil
}
```

4. Goa 生成包装函数以将拦截器应用到端点：

```go
func main() {
    // 创建服务实现
    svc := NewService()
    
    // 创建拦截器
    interceptors := NewInterceptors(log.Default())
    
    // 创建带拦截器的端点
    endpoints := NewEndpoints(svc, interceptors)
    
    // ... 像往常一样继续 ...
}
```

生成的代码带来以下好处：

- 通过类型安全接口访问 Payload 与 Result
- 自动以正确顺序包装端点
- 清晰分离拦截器的定义与实现
- 正确处理不同调用类型（一元、服务端流、客户端流、双向流）

这些接口与包装器确保你的拦截器在保持类型安全的同时，正确接入请求处理管道。

## 拦截器的执行顺序

当应用多个拦截器时，其执行顺序如下：

1. 服务级拦截器（按声明顺序）
2. 方法级拦截器（按声明顺序）
3. 实际端点
4. 方法级拦截器（逆序）
5. 服务级拦截器（逆序）

这意味着拦截器同时包裹了请求与响应流程。

## 下一步

现在你已理解基础内容：

- 了解不同的[拦截器类型](../2-interceptor-types)
- 学习[拦截器实现](3-interceptor-implementation)的细节与模式
- 查看用于生产的[最佳实践](../4-best-practices)