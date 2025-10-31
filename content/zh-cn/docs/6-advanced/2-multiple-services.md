---
Title: 在 Goa 中使用多个服务
linkTitle: 多个服务
weight: 2
description: >
  使用 Goa 设计和实现可扩展的微服务架构
---

在实际应用程序中，通常有多个服务协同工作以形成一个完整的系统。Goa 使在单个项目中设计和实现多个服务变得容易。本指南将引导您完成有效创建和管理多个服务的过程。

另请参阅如何使用 Goa 构建[优雅的单体](./4-elegant-monolith.md)。

## 理解多个服务

Goa 中的服务表示提供特定功能的相关端点的逻辑分组。虽然简单的应用程序可能只需要一个服务，但较大的应用程序通常受益于将功能拆分到多个服务中。这种方法可以更好地组织 API 端点、更清晰地分离关注点、更容易地进行维护和测试、独立的部署能力以及精细的安全控制。

## 服务架构模式

在设计多服务系统时，服务通常分为两类：暴露给外部世界的前端服务和供前端服务使用的后端服务。理解这些模式有助于设计可扩展和可维护的架构。

## 服务组织

Goa 在组织服务的设计和生成的代码方面提供了灵活性。让我们探讨两种主要方法：独立设计和统一设计。

### 独立设计方法

独立方法将每个服务视为一个独立的单元，在这种情况下，会为每个服务分别调用 `goa gen`，并且它们各自包含自己的 `gen/` 目录。这使得将服务轻松移动到单独的存储库并独立进行版本控制成为可能。

### 统一设计方法

统一方法定义了一个顶级设计文件，该文件导入所有服务的设计包并定义一个通用的 API：

```go
// design/design.go - 顶级设计文件
package design

import (
    _ "myapi/services/users/design"    // 每个服务都有自己的设计，并将其导入顶级设计中
    _ "myapi/services/products/design"
    . "goa.design/goa/v3/dsl"
)

var _ = API("myapi", func() {
    Title("My API")
    Description("多服务 API 示例")
})
```

这种方法集中了代码生成和类型共享：
- 生成的代码位于根 `gen/` 目录中
- 单个 `goa gen` 命令生成所有服务代码
- 使用 `Meta` `struct:pkg:path` 键定义的共享类型在服务之间自动可用
- 系统维护统一的 OpenAPI 规范

统一方法非常适合对相关服务进行分组和共享通用类型。它确保所有相关服务都一起进行版本控制，并使管理依赖项和更新变得更加容易。

## 传输注意事项

您选择的传输协议会显著影响服务之间的交互方式。让我们研究一下每种方法的优点：

### HTTP 服务

HTTP 是面向外部服务的绝佳选择。它提供通用的客户端兼容性、丰富的工具和中间件生态系统以及熟悉的 REST 模式。HTTP 也易于调试和测试，非常适合 Web 应用程序。

然而，HTTP 的灵活性也带来了成本：服务器可能需要处理不同类型的编码，还需要为其 API 选择特定的样式。可以说，与 protobuf 等二进制协议相比，其主要缺点通常是编码性能较差。

### gRPC 服务

[gRPC](https://grpc.io) 特别适用于内部服务通信，因为它具有高性能、低延迟和内置的流支持。启用反射后，gRPC 还提供内置的服务发现功能。此外，gRPC 支持在单个连接上对请求和响应进行高效的多路复用，这可以在服务之间通信时带来显著的性能提升。

gRPC 的主要缺点是它不太适合面向外部的服务，因为它需要客户端库来编码和解码二进制消息。此外，gRPC 的采用范围不如 HTTP 广泛，因此它可能不适合期望有各种客户端的服务。

### 总结

在构建由多个服务组成的系统时，一个好的方法是对需要暴露给外部世界的服务使用 HTTP，而对只需要与其他内部服务通信的服务使用 gRPC。

## 存储库结构

一个组织良好的存储库有助于团队有效地导航和维护代码库。统一的结构也使开发人员更容易在系统和服务之间切换。以下是推荐的结构：

```
myapi/
├── README.md          # 系统概述和设置指南
├── design/            # 共享设计元素
│   ├── design.go      # 统一方法的顶级设计
│   └── types/         # 使用 Meta("struct:pkg:path") 定义的共享类型定义
├── gen/               # 生成的代码（统一设计方法）
│   ├── http/          # HTTP 传输层代码
│   ├── grpc/          # gRPC 传输层代码
│   └── types/         # 生成的共享类型
├── scripts/           # 开发和部署脚本
└── services/          # 服务实现
    ├── users/         # 示例：用户服务
    │   ├── cmd/       # 服务可执行文件
    │   ├── design/    # 特定于服务的设计
    │   ├── gen/       # 生成的代码（独立设计方法）
    │   ├── users.go   # 业务逻辑
    │   └── README.md  # 服务文档
    └── products/      # 示例：产品服务
        └── ...
```

目录结构遵循清晰的关注点分离和模块化组织：

- `README.md`：包含整个系统的文档、设置说明和架构概述。

- `design/`：存放跨服务的共享设计元素
  - `design.go`：使用 Goa 的 DSL 为统一方法定义顶级 API 设计
  - `types/`：包含跨多个服务使用的共享类型定义

- `gen/`：包含由 Goa 从统一设计方法生成的代码

- `services/`：各个服务的实现
  - 每个服务（例如 `orders/`、`products/`）都遵循一致的结构：
    - `cmd/`：服务入口点和可执行文件
    - `design/`：特定于服务的设计
    - `gen/`：特定于服务的生成代码
    - `users.go`：业务逻辑实现
    - `README.md`：特定于服务的文档

这种结构支持单体和微服务部署，从而实现：
- 服务之间的清晰分离
- 共享类型和设计元素
- 独立的服务演进
- 轻松的服务发现和导航

## 服务通信模式

在设计服务交互时，请考虑以下常见模式：

### 前端和后端服务

服务通常分为两类：

1. **前端服务**：面向公众的服务，其特点是：
   - 使用 HTTP 作为传输方式，以实现广泛的客户端兼容性
   - 专注于将请求协调到后端服务
   - 处理外部请求的身份验证和授权
   - 启动可观察性上下文（跟踪、指标）
   - 定义具有浅层实现的广泛 API

2. **后端服务**：内部服务，其特点是：
   - 通常使用 gRPC 以获得性能优势
   - 实现核心业务逻辑
   - 可能使用私有身份机制（例如 spiffe）
   - 为现有的可观察性上下文做出贡献
   - 定义具有深层实现的重点 API

一种常见的架构模式是拥有几个（有时只有一个）前端服务，将您平台的功能暴露给外部客户端，并由多个后端服务处理实际的业务逻辑。

## 脚本和自动化

`scripts/` 目录为常见的开发和部署任务提供自动化。这些脚本适用于统一和独立两种方法，使您可以轻松地管理服务，而无需考虑所选择的架构。

### 开发脚本

核心开发脚本处理代码生成、构建和测试：

```bash
# scripts/gen.sh - 代码生成脚本
#!/bin/bash
if [ "$1" == "" ]; then
    # 统一方法：生成所有服务
    goa gen myapi/design
else
    # 独立方法：生成特定服务
    cd services/$1 && goa gen myapi/services/$1/design
fi

# scripts/build.sh - 构建脚本
#!/bin/bash
if [ "$1" == "" ]; then
    # 构建所有服务
    for service in services/*/; do
        service=${service%*/}
        echo "正在构建 ${service##*/}..."
        go build -o bin/${service##*/} ./$service/cmd/${service##*/}
    done
else
    # 构建特定服务
    go build -o bin/$1 ./services/$1/cmd/$1
fi

# scripts/test.sh - 测试运行程序
#!/bin/bash
if [ "$1" == "" ]; then
    # 测试所有服务和共享代码
    go test ./... -v
else
    # 测试特定服务
    go test ./services/$1/... -v
fi
```

### 部署脚本

部署脚本处理服务执行和容器部署：

```bash
# scripts/run.sh - 本地服务运行程序
#!/bin/bash
if [ "$1" != "" ]; then
    # 运行特定服务
    ./bin/$1
else
    # 列出可用服务
    echo "可用服务："
    ls bin/
fi

# scripts/deploy.sh - Kubernetes 部署
#!/bin/bash
if [ "$1" != "" ]; then
    deploy_service() {
        echo "正在部署 $1..."
        docker build -t myapi/$1 ./services/$1
        docker push myapi/$1
        kubectl apply -f ./services/$1/k8s/
    }
    deploy_service $1
else
    # 部署所有服务
    for service in services/*/; do
        service=${service%*/}
        deploy_service ${service##*/}
    done
fi
```

这些脚本支持两种开发工作流程：

**统一设计方法**：
- 单个命令生成所有服务代码
- 跨服务的集中式测试
- 协调的构建和部署
- 共享的依赖项管理

**独立设计方法**：
- 每个服务的代码生成
- 隔离的测试环境
- 独立的构建过程
- 特定于服务的部署

## 服务实现

每个服务都作为单独的可执行文件运行，从而促进了隔离和独立扩展。以下是服务实现的示例：

```go
// services/users/cmd/users/main.go - 服务入口点
package main

import (
    "context"
    "flag"
    "fmt"
    "net/http"
    "os"
    "os/signal"
    "strings"
    "sync"
    "syscall"
    "time"

    "goa.design/clue/log"
    goahttp "goa.design/goa/v3/http"
    
    genusersserver "myapi/services/users/gen/http/users/server"
    genusers "myapi/services/users/gen/users"
    "myapi/services/users"
)

func main() {
    // 解析命令行标志
    var (
        httpAddr = flag.String("http-addr", ":8080", "HTTP 侦听地址")
        debug    = flag.Bool("debug", false, "启用调试模式")
    )
    flag.Parse()

    // 使用记录器初始化上下文
    format := log.FormatJSON
    if log.IsTerminal() {
        format = log.FormatTerminal
    }
    ctx := log.Context(context.Background(), log.WithFormat(format))
    if *debug {
        ctx = log.Context(ctx, log.WithDebug())
        log.Debugf(ctx, "调试模式已启用")
    }

    // 创建服务和端点
    svc := users.NewUsers()
    endpoints := genusers.NewEndpoints(svc)

    // 创建传输处理程序
    mux := goahttp.NewMuxer()
    server := genusersserver.New(endpoints, mux, goahttp.RequestDecoder, goahttp.ResponseEncoder, nil, nil)
    server.Mount(mux)

    // 记录已挂载的端点
    for _, m := range server.Mounts {
        log.Printf(ctx, "已挂载 %s %s", m.Method, m.Pattern)
    }

    // 创建 HTTP 服务器
    mux.Use(log.HTTP(ctx)) // 将记录器添加到请求上下文中
    httpServer := &http.Server{
        Addr:    *httpAddr,
        Handler: mux,
    }

    // 优雅地处理关闭
    errc := make(chan error)
    go func() {
        c := make(chan os.Signal, 1)
        signal.Notify(c, syscall.SIGINT, syscall.SIGTERM)
        errc <- fmt.Errorf("信号：%s", <-c)
    }()

    ctx, cancel := context.WithCancel(ctx)
    var wg sync.WaitGroup
    wg.Add(1)

    go func() {
        defer wg.Done()

        // 启动 HTTP 服务器
        go func() {
            log.Printf(ctx, "HTTP 服务器正在侦听 %s", *httpAddr)
            errc <- httpServer.ListenAndServe()
        }()

        <-ctx.Done()
        log.Print(ctx, "正在关闭 HTTP 服务器")

        // 优雅地关闭，超时时间为 30 秒
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        defer cancel()

        if err := httpServer.Shutdown(ctx); err != nil {
            log.Errorf(ctx, err, "无法关闭 HTTP 服务器")
        }
    }()

    // 等待关闭
    if err := <-errc; err != nil && !strings.HasPrefix(err.Error(), "signal:") {
        log.Errorf(ctx, err, "服务器错误")
    }
    cancel()
    wg.Wait()
    log.Print(ctx, "服务器已退出")
}
```

服务实现包含业务逻辑：

```go
// services/users/users.go - 服务实现
package users

import (
    "context"

    "goa.design/clue/log"
    "myapi/services/users/gen/users"
)

// Users 实现用户服务接口
type Users struct {}

// NewUsers 创建一个新的用户服务实例
func NewUsers() *Users {
    return &Users{}
}

// List 检索所有用户
func (s *Users) List(ctx context.Context, p *users.ListPayload) (*users.UserCollection, error) {
    log.Printf(ctx, "正在使用筛选器列出用户：%v", p.Filter)
    // 实现细节...
    return nil, nil
}
```

## 最佳实践

在使用 Goa 构建多服务系统时，请遵循以下准则：

1. 选择合适的传输方式
   对内部服务使用 gRPC，对外部 API 使用 HTTP。

2. 为演进做好规划
   对您的服务进行版本控制，并为向后兼容性做好规划。

3. 实现稳健的错误处理
   定义清晰的错误类型，并优雅地处理跨服务故障。

4. 记录服务交互
   维护清晰的服务 API 和依赖项文档。

## 后续步骤

要加深您对多服务系统的理解：

- [编写服务客户端](/docs/5-real-world/3-common-patterns/1-clients) - 了解如何实现稳健的服务通信
- [HTTP 服务](/docs/4-concepts/3-http) - 了解 HTTP 服务实现
- [gRPC 服务](/docs/4-concepts/4-grpc) - 了解 gRPC 服务开发
- [拦截器](/docs/4-concepts/5-interceptors) - 添加横切关注点