---
linkTitle: Goa 拦截器
title: Goa 拦截器
description: "了解 Goa 面向横切关注点的类型安全拦截器系统"
weight: 1
---

Goa 拦截器提供了一种强大且类型安全的机制，用于在服务方法中注入横切关注点。它们允许你在客户端与服务端两侧拦截并修改请求与响应，同时保持完整的类型安全与良好的 IDE 支持。

## 什么是拦截器？

拦截器是一种组件，允许你在服务方法之前、之后或包裹其周围执行附加行为。它们可以：

- 读取并修改入站请求
- 读取并修改出站响应
- 处理错误
- 添加上下文信息
- 实现日志、监控或数据转换等横切关注点

## 关键特性

- **类型安全**：所有拦截器交互均由生成的辅助类型进行完整类型约束
- **灵活应用位置**：可在服务级与方法级应用
- **双向支持**：同时支持客户端与服务端拦截
- **流式支持**：完整支持一元与流式操作
- **简洁 DSL**：用于定义拦截器的清晰声明式语法
- **显式访问控制**：明确指定可访问的 Payload 与 Result 字段

## 基本示例

以下示例定义一个客户端重试拦截器：

```go
var RetryPolicy = Interceptor("RetryPolicy", func() {
    Description("为失败的操作实现指数退避重试")
    
    // 在结果中记录重试次数
    WriteResult(func() {
        Attribute("attempts")
    })
})

var _ = Service("payment", func() {
    // 在整个服务上应用重试策略
    ClientInterceptor(RetryPolicy)
    
    Method("process", func() {
        Payload(func() {
            Attribute("amount", Int, "支付金额")
            Attribute("currency", String, "支付币种")
        })
        Result(func() {
            Attribute("id", String, "交易 ID")
            Attribute("status", String, "交易状态")
            Attribute("attempts", Int, "已进行的重试次数")
        })
        // 方法级的其他配置...
    })
})
```
在该示例中，我们定义了实现指数退避的 `RetryPolicy` 拦截器。要点如下：

1. **拦截器定义**：

   ```go
   var RetryPolicy = Interceptor("RetryPolicy", func() {
   ```

   创建名为 "RetryPolicy" 的拦截器。

2. **结果修改**：

   ```go
   WriteResult(func() {
       Attribute("attempts")
   })
   ```

   声明该拦截器会向响应中的 "attempts" 字段写入重试次数。

3. **服务应用**：

   ```go
   ClientInterceptor(RetryPolicy)
   ```

   在服务级应用该拦截器，意味着它影响服务中的所有方法。

4. **方法定义**：

   `process` 方法展示了拦截器与服务的集成方式：
   - Payload 定义支付细节（amount 与 currency）
   - Result 包含标准交易字段（id、status）
   - Result 中的 `attempts` 字段用于重试拦截器记录其行为

拦截器实现后，会对失败的操作自动进行退避重试，提升对瞬时错误的抵抗能力。

## 何时使用拦截器

拦截器适用于服务架构中的诸多常见场景：

- **客户端韧性**：实现重试策略与熔断器
- **资源管理**：限流与节流
- **监控**：跟踪操作指标与性能
- **数据转换**：在数据流动过程中修改或丰富数据
- **关联追踪**：添加请求关联与跟踪 ID
- **缓存**：实现客户端或服务端缓存
- **校验**：补充 Goa 内置校验之外的自定义校验规则
- **错误处理**：标准化错误响应与恢复策略

注意：涉及安全（认证与授权）时，应使用 Goa 的内置安全 DSL，而非拦截器，因为其提供更稳健且专用的安全特性。

## 下一步

- 了解如何[入门](1-getting-started)使用拦截器
- 探索不同[拦截器类型](2-interceptor-types)以适配具体场景
- 学习[拦截器实现](3-interceptor-implementation)的细节与模式
- 查看[最佳实践](4-best-practices)，高效使用拦截器

