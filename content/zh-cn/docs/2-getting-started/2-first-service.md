---
title: 第一个服务
weight: 2
description: "通过这个实践教程创建您的第一个 Goa 服务，涵盖服务设计、代码生成、实现和简单 HTTP 端点的测试。"
---

## 先决条件

准备好构建一些很棒的东西了吗？本指南假设您已安装 `curl`。任何其他 HTTP 客户端也可以使用。

## 1. 创建新模块

让我们通过为您的第一个 Goa 服务设置一个新工作区来开始我们的旅程：

```bash
mkdir hello-goa && cd hello-goa  
go mod init hello
```

> 注意：虽然我们在本指南中使用简单的模块名 `hello`，但在实际项目中
> 您通常会使用域名，如 `github.com/yourusername/hello-goa`。别担心——
> 您将学到的概念工作方式完全相同！

## 2. 设计您的第一个服务

现在到了激动人心的部分——设计您的服务！Goa 强大的 DSL 将帮助您在
仅仅几行代码中创建一个干净、专业的 API。

1. **添加 `design` 文件夹**

```bash
mkdir design
```

2. **创建设计文件** (`design/design.go`)：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
)

var _ = Service("hello", func() {
    Description("一个简单的打招呼服务。")

    Method("sayHello", func() {
        Payload(String, "要问候的名字")
        Result(String, "问候消息")

        HTTP(func() {
            GET("/hello/{name}")
        })
    })
})
```

让我们分解一下这个设计的作用：

- `Service("hello", ...)` 定义一个名为 "hello" 的新服务
- 在服务内部，我们定义一个单一方法 `sayHello`，它：
  - 接受一个字符串 `Payload` - 这将是我们想要问候的名字
  - 返回一个字符串 `Result` - 我们的问候消息
  - 映射到 `/hello/{name}` 的 HTTP GET 端点，其中 `{name}` 将自动绑定到我们的负载

这个简单的设计展示了 Goa 的声明式方法——我们描述_想要_ API 做什么，Goa 处理所有实现细节，如参数绑定、路由和 OpenAPI 文档。

## 3. 生成代码

这就是魔法发生的地方！让我们使用 Goa 的代码生成器将您的设计转换为
一个完全可用的服务结构：

```bash
goa gen hello/design
```

这创建了一个 `gen` 文件夹，包含您所需的一切——端点、传输逻辑，甚至
OpenAPI 规范。很酷，对吧？

现在，让我们使用 `example` 命令搭建一个可工作的服务：

```bash
goa example hello/design
```

> 注意：将 `example` 命令视为您的起点——它为您提供一个可工作的实现
> 您可以在此基础上构建。虽然您会在设计更改时重新运行 `gen`，但来自 `example` 的代码
> 是您可以自定义和增强的。

这是您将在 `hello-goa` 文件夹中找到的内容：

```
hello-goa
├── cmd
│   ├── hello
│   │   ├── http.go
│   │   └── main.go
│   └── hello-cli
│       ├── http.go
│       └── main.go
├── design
│   └── design.go
├── gen
│   ├── ...
│   └── http
└── hello.go
```

## 4. 实现服务

是时候让您的服务活起来了！编辑 `hello.go` 文件并替换
`SayHello` 方法为这个友好的实现：

```go
func (s *hellosrvc) SayHello(ctx context.Context, name string) (string, error) {
	log.Printf(ctx, "hello.sayHello")
    return fmt.Sprintf("Hello, %s!", name), nil
}
```

您快完成了——是不是出人意料地简单？

## 5. 运行和测试

### 启动服务器

首先，让我们整理所有依赖项：

```bash
go mod tidy
```

现在是关键时刻——让您的服务上线：

```bash
go run hello/cmd/hello --http-port=8080
INFO[0000] http-port=8080
INFO[0000] msg=HTTP "SayHello" mounted on GET /hello/{name}
INFO[0000] msg=HTTP server listening on "localhost:8080"
```

### 调用服务

打开一个新终端，让我们看看您的服务运行：

```bash
curl http://localhost:8080/hello/Alice
"Hello, Alice!"
```

🎉 太棒了！您刚刚创建并部署了您的第一个 Goa 服务。这只是
使用 Goa 构建的开始！

### 使用 CLI 客户端

想尝试一些更酷的东西吗？Goa 自动为您生成了一个命令行客户端。
试试看：

```bash
go run hello/cmd/hello-cli --url=http://localhost:8080 hello say-hello -p=Alice
```

好奇 CLI 还能做什么吗？查看所有功能：

```bash
go run hello/cmd/hello-cli --help
```

## 6. 持续开发

### 编辑 DSL → 重新生成

随着服务的增长，您会想要添加新功能。无论何时您使用
新方法、字段或错误更新设计，只需运行：

```bash
goa gen hello/design
```

您的服务代码由您演进——Goa 不会触碰 `gen` 文件夹之外的任何内容，
所以请尽情增强和自定义吧！

## 7. 下一步

准备好将您的 Goa 技能提升到下一个水平了吗？深入了解我们的 [教程](../3-tutorials)，在那里
您将学习构建强大的 REST API、gRPC 服务等等。可能性是无限的！