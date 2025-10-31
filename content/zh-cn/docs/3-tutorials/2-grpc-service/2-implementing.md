---
title: "实现服务"
linkTitle: "实现"
weight: 2
description: "在 Goa 中实现 gRPC 服务的指南，涵盖代码生成、服务实现、服务器搭建以及对生成的 gRPC 构件的理解。"
---

使用 Goa 的 DSL 设计好你的 gRPC 服务后，是时候让它真正运行起来了！本指南将一步步带你实现服务。你将学到：

1. 生成 gRPC 脚手架
2. 理解生成代码的结构
3. 实现你的服务逻辑
4. 搭建 gRPC 服务器

## 1. 生成 gRPC 构件

首先，让我们生成所有必要的 gRPC 代码。在项目根目录（例如 `grpcgreeter/`）执行：

```bash
goa gen grpcgreeter/design
go mod tidy
```

该命令会分析你的 gRPC 设计（`greeter.go`），并在 `gen/` 目录生成所需代码。将会创建如下结构：

```
gen/
├── grpc/
│   └── greeter/
│       ├── pb/           # Protocol Buffers 定义
│       ├── server/       # 服务端 gRPC 代码
│       └── client/       # 客户端 gRPC 代码
└── greeter/             # 服务接口与类型
```

{{< alert title="重要" >}}
每当你修改设计文件时，请重新运行 `goa gen`，以确保生成代码与服务定义保持同步。
{{< /alert >}}

## 2. 理解生成的代码

来看看 Goa 为我们生成了什么：

### Protocol Buffer 定义（gen/grpc/greeter/pb/）

- **`greeter.proto`**：Protocol Buffers 服务定义
  ```protobuf
  service Greeter {
    rpc SayHello (SayHelloRequest) returns (SayHelloResponse);
  }
  ```
- **`greeter.pb.go`**：由 `.proto` 文件编译生成的 Go 代码

### 服务端代码（gen/grpc/greeter/server/）

- **`server.go`**：将服务方法映射到 gRPC 处理器
- **`encode_decode.go`**：在服务类型与 gRPC 消息之间进行转换
- **`types.go`**：包含服务端特定的类型定义

### 客户端代码（gen/grpc/greeter/client/）

- **`client.go`**：gRPC 客户端实现
- **`encode_decode.go`**：客户端序列化逻辑
- **`types.go`**：客户端特定的类型定义

## 3. 实现你的服务

接下来是最有趣的部分——实现服务逻辑！在你的服务包中新建 `greeter.go` 文件：

```go
package greeter

import (
    "context"
    "fmt"

    // 对生成包使用描述性别名
    gengreeter "grpcgreeter/gen/greeter"
)

// GreeterService 实现 Service 接口
type GreeterService struct{}

// NewGreeterService 创建一个新的服务实例
func NewGreeterService() *GreeterService {
    return &GreeterService{}
}

// SayHello 实现问候逻辑
func (s *GreeterService) SayHello(ctx context.Context, p *gengreeter.SayHelloPayload) (*gengreeter.SayHelloResult, error) {
    // 如有需要可添加入参校验
    if p.Name == "" {
        return nil, fmt.Errorf("name cannot be empty")
    }

    // 构建问候语
    greeting := fmt.Sprintf("Hello, %s!", p.Name)
    
    // 返回结果
    return &gengreeter.SayHelloResult{
        Greeting: greeting,
    }, nil
}
```

### 实现的最佳实践

1. 错误处理：使用合适的 gRPC 状态码
2. 校验：尽早验证输入
3. 上下文使用：尊重上下文取消
4. 日志：添加有意义的日志以便调试
5. 测试：为你的服务逻辑编写单元测试

## 4. 搭建 gRPC 服务器

在 `cmd/greeter/main.go` 中创建服务入口：

```go
package main

import (
    "context"
    "log"
    "net"
    "os"
    "os/signal"
    "syscall"
    
    "grpcgreeter"
    gengreeter "grpcgreeter/gen/greeter"
    genpb "grpcgreeter/gen/grpc/greeter/pb"
    genserver "grpcgreeter/gen/grpc/greeter/server"
    
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
)

func main() {
    // 创建 TCP 监听器
    lis, err := net.Listen("tcp", ":8090")
    if err != nil {
        log.Fatalf("failed to listen: %v", err)
    }

    // 创建带选项的 gRPC 服务器
    srv := grpc.NewServer(
        grpc.UnaryInterceptor(loggingInterceptor),
    )

    // 初始化你的服务
    svc := greeter.NewGreeterService()
    
    // 创建端点
    endpoints := gengreeter.NewEndpoints(svc)
    
    // 在 gRPC 服务器上注册服务
    genpb.RegisterGreeterServer(srv, genserver.New(endpoints, nil))
    
    // 启用服务器反射，便于调试工具使用
    reflection.Register(srv)

    // 处理优雅关闭
    go func() {
        sigCh := make(chan os.Signal, 1)
        signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
        <-sigCh
        log.Println("Shutting down gRPC server...")
        srv.GracefulStop()
    }()

    // 启动服务
    log.Printf("gRPC server listening on :8090")
    if err := srv.Serve(lis); err != nil {
        log.Fatalf("failed to serve: %v", err)
    }
}

// 示例：日志拦截器
func loggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    log.Printf("Handling %s", info.FullMethod)
    return handler(ctx, req)
}
```

### 解析服务器代码

下面分解 gRPC 服务器的关键组件：

1. TCP 监听器设置：
   ```go
   lis, err := net.Listen("tcp", ":8090")
   ```
   打开 8090 端口以接受传入的 gRPC 连接。服务将在此端口监听客户端请求。

2. 服务器创建：
   ```go
   srv := grpc.NewServer(
       grpc.UnaryInterceptor(loggingInterceptor),
   )
   ```
   创建一个支持中间件（拦截器）的 gRPC 服务器。日志拦截器会记录所有传入请求。

3. 服务注册：
   ```go
   svc := greeter.NewGreeterService()
   endpoints := gengreeter.NewEndpoints(svc)
   genpb.RegisterGreeterServer(srv, genserver.New(endpoints, nil))
   ```
   - 创建你的服务实现
   - 使用 Goa 的与传输无关的端点进行封装
   - 将其注册到 gRPC 服务器以处理传入请求

4. 服务器反射：
   ```go
   reflection.Register(srv)
   ```
   启用 gRPC 反射，便于 `grpcurl` 等工具动态发现服务方法。

5. 优雅关闭：
   ```go
   go func() {
       sigCh := make(chan os.Signal, 1)
       signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
       <-sigCh
       srv.GracefulStop()
   }()
   ```
   - 监听中断信号（Ctrl+C）或终止请求
   - 确保在关闭前已完成正在处理的请求
   - 防止连接中断与数据丢失

6. 请求日志：
   ```go
   func loggingInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
       log.Printf("Handling %s", info.FullMethod)
       return handler(ctx, req)
   }
   ```
   - 在请求到达服务之前进行拦截
   - 记录被调用的方法
   - 便于调试与监控
   - 可扩展用于指标、认证或其他横切关注点

### 服务器特性

- 优雅关闭：正确处理终止信号
- 日志：包含基础的请求日志拦截器
- 反射：支持 `grpcurl` 等工具发现服务
- 错误处理：将错误正确传播给客户端
- 可扩展性：易于添加认证、指标等拦截器

## 5. 构建与运行

1. 构建服务：
   ```bash
   go build -o greeter cmd/greeter/main.go
   ```

2. 运行服务器：
   ```bash
   ./greeter
   ```

现在你的 gRPC 服务已经运行，并在 8090 端口准备好接受连接！

## 后续步骤

在实现并运行服务之后，你可以：

- 继续阅读[运行教程](../3-running)以测试你的服务
- 添加指标与监控
- 实现更多服务方法
- 添加认证与授权
- 搭建 CI/CD 流水线

别忘了查看 [gRPC 概念](../../4-concepts/4-grpc) 一章，了解流式传输、中间件与错误处理等高级主题。
