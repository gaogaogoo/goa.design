---
title: "探索 Goa"
linkTitle: "探索 Goa"
weight: 20
description: >
  Goa 文档，这是一个优先设计的框架，用于在 Go 中构建微服务和 API。
---

## 改变您的 API 开发

在微服务和 API 的世界中，设计和实现之间的差距一直是一个挑战。
Goa 以创新的方法弥合了这一差距，改变了您在 Go 中构建服务的方式。
通过将设计放在首位，Goa 消除了困扰传统开发的规范、实现和文档之间
繁琐的来回往复。

想象一下，描述您的 API 一次，就能自动生成您所需的一切：服务器代码、
客户端库、文档等等。这不仅仅是梦想——Goa 做到了这一点。通过利用 Go 的
类型系统和现代设计原则，Goa 帮助您在很短的时间内构建健壮的生产就绪服务。

## 什么让 Goa 与众不同？

Goa 通过将您的 API 设计视为活合约而脱颖而出。这种优先设计的方法意味着：

* 您的 API 文档始终与代码同步——因为它们来自同一个来源
* 通过类型安全接口，保证实现与设计匹配
* 您可以在不更改业务逻辑的情况下在 HTTP 和 gRPC 之间切换
* 您专注于重要的事情：构建提供价值的功能

## Goa 如何工作

![Goa 的分层架构](/img/docs/layers.png)

这就是魔法发生的地方。从一个单一的设计文件开始，Goa 释放出
一串生成的代码，这些代码通常需要几周时间手工编写和维护。
您专注于描述您想要的，Goa 处理繁重的工作：

1. 实现代码 - 基础
    * 生产就绪的服务和客户端接口
    * 传输无关的端点，保持代码整洁
    * 开箱即用的 HTTP 和 gRPC 处理器
    * 您宁愿不编写的所有请求/响应编码

2. 自动营销的文档
    * 漂亮的 OpenAPI 规范
    * 准备跨平台使用的协议缓冲区定义
    * 与您的代码一起演进的文档，而不是事后想法

3. 额外的努力
    * 坚如磐石的输入验证
    * 生产级错误处理
    * 用户会感谢您的客户端库

最好的部分？虽然 Goa 生成数千行样板代码、测试
和文档，但您只编写重要的代码——业务逻辑。
您的三行代码可以变成一个完整的生产就绪服务，具有
HTTP 和 gRPC 支持、命令行工具和全面的 API 文档。

## 简单示例
以下是使用 Goa 设计 API 的样子：

```go
var _ = Service("calculator", func() {
    Method("add", func() {
        Payload(func() {
            Field(1, "a", Int, "第一个数字")
            Field(2, "b", Int, "第二个数字")
            Required("a", "b")
        })
        Result(Int)

        HTTP(func() {
            GET("/add/{a}/{b}")
            Response(StatusOK)
        })
    })
})
```

以下是实现它所需编写的所有代码：

```go
func (s *service) Add(ctx context.Context, p *calc.AddPayload) (int, error) {
    return p.A + p.B, nil
}
```

## 核心概念

### 优先设计：您的单一真实来源

停止在多个 API 规范、文档和实现文件之间来回折腾。使用
Goa，您的设计就是您的契约——一个清晰、可执行的规范，保持
每个人都在同一页面上。团队喜欢这种方法，因为它永远消除了
"但那不是规范所说的"对话。

### 可扩展的清洁架构

Goa 生成连资深架构师都梦想拥有的代码。每个组件都在其完美的位置：
* 服务层：您的领域逻辑，纯粹而整洁
* 端点层：传输无关的业务流程
* 传输层：适应您需求的 HTTP/gRPC 处理器

这不仅仅是架构理论——这是工作的代码，使您的服务
更容易测试、修改和随着需求的发展而扩展。

### 为您提供保障的类型安全

忘记运行时意外吧。Goa 利用 Go 的类型系统在编译时捕获问题：

```go
// 生成的接口 - 您的契约
type Service interface {
    Add(context.Context, *AddPayload) (int, error)
}

// 您的实现 - 简洁且专注
func (s *service) Add(ctx context.Context, p *calc.AddPayload) (int, error) {
    return p.A + p.B, nil
}
```

如果您的实现与设计不匹配，您会在代码投入生产之前就知道。

### 合理的项目结构

不再猜测文件应该放在哪里。Goa 项目遵循水晶般清晰的组织：

```
├── design/         # 您的 API 设计 - 真实来源
├── gen/            # 生成的代码 - 永远不要编辑这个
│   ├── calculator/ # 服务接口
│   ├── http/       # HTTP 传输层
│   └── grpc/       # gRPC 传输层
└── calculator.go   # 您的实现 - 魔法发生的地方
```

每个文件都有自己的位置，您团队中的每个开发人员都会确切地知道在哪里查找。

## 下一步

* 遵循 [入门指南](2-getting-started)
* 探索 [核心教程](3-tutorials)
* 加入社区：
    * [Gophers Slack](https://gophers.slack.com/messages/goa)
    * [GitHub 讨论](https://github.com/goadesign/goa/discussions)
    * [Bluesky](https://goadesign.bsky.social)
