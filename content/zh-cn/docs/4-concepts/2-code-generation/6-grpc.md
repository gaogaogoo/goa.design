---
title: "生成的 gRPC 服务端与客户端代码"
linkTitle: "gRPC 代码"
weight: 6
description: "了解 Goa 生成的代码，包括服务接口、端点与传输层。"
---

gRPC 代码生成会产出完整的客户端与服务端实现，处理所有传输层相关事务。Goa 会为每个服务生成 Protobuf 定义，并自动调用 `protoc` 生成服务端与客户端的底层代码。同时，Goa 还会生成基于这些 Protobuf 代码的高层 gRPC 服务端与客户端实现。

## Protobuf 定义

protobuf 服务定义会生成在
`gen/grpc/<服务名>/pb/goagen_<API 名>_<服务名>.proto`：

```protobuf
syntax = "proto3";

package calc;

service Calc {
    rpc Add (AddRequest) returns (AddResponse);
    rpc Multiply (MultiplyRequest) returns (MultiplyResponse);
}

message AddRequest {
    int64 a = 1;
    int64 b = 2;
}

message AddResponse {
    int64 result = 1;
}
// ...more messages...
```

该 protobuf 定义会通过 `protoc` 编译器生成底层 gRPC 代码，Goa 会在代码生成过程中自动调用该编译器。

## gRPC 服务端实现

Goa 会在 `gen/grpc/<服务名>/server/server.go` 中生成完整的 gRPC 服务端实现，以 gRPC 暴露服务方法。服务端可通过生成的 `New` 函数实例化，该函数接收服务端点与可选的一元（Unary）处理器。如果未提供一元处理器，服务端将使用 Goa 提供的默认处理器。

```go
// New 使用 calc 服务端点实例化服务端结构体。
func New(e *calc.Endpoints, uh goagrpc.UnaryHandler) *Server {
	return &Server{
		AddH: NewAddHandler(e.Add, uh),
		MultiplyH: NewMultiplyHandler(e.Multiply, uh),
	}
}
```

上述代码中的 `goagrpc` 指的是 Goa 的 gRPC 包，位于 `goa.design/goa/v3/grpc`。`UnaryHandler` 类型是一个函数，接收上下文与请求并返回响应与错误。若服务暴露了流式方法，`New` 也会接收流式处理器。

`Server` 结构体会暴露可用于修改各个处理器或为特定端点应用中间件的字段：

```go
// Server 列出了 calc 服务端点的 gRPC 处理器。
type Server struct {
    AddH      goagrpc.UnaryHandler
    MultiplyH goagrpc.UnaryHandler
    // ... 私有字段 ...
}
```

服务端文件的其余部分会为每个服务方法实现 gRPC 处理器。这些处理器负责解码请求、调用服务方法，并编码响应与错误。

### 综合示例

服务接口层与端点层是 Goa 生成服务的核心构建块。你的 `main` 包会使用这些层来创建特定传输的服务端与客户端实现，从而通过 gRPC 运行并与服务交互：

```go
package main

import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"

    "github.com/<your username>/calc"
    gencalc "github.com/<your username>/calc/gen/calc"
    genpb "github.com/<your username>/calc/gen/grpc/calc/pb"
    gengrpc "github.com/<your username>/calc/gen/grpc/calc/server"
)

func main() {
    svc := calc.New()                        // 你的服务实现
    endpoints := gencalc.NewEndpoints(svc)   // 创建服务端点
    svr := grpc.NewServer(nil)               // 创建 gRPC 服务端
    gensvr := gengrpc.New(endpoints, nil)    // 创建服务端实现
    genpb.RegisterCalcServer(svr, genserver) // 在 gRPC 上注册服务端
    lis, _ := net.Listen("tcp", ":8080")     // 启动 gRPC 服务端监听
    svr.Serve(lis)                           // 启动 gRPC 服务端
}
```

## gRPC 客户端实现

Goa 会在 `gen/grpc/<服务名>/client/client.go` 中生成完整的 gRPC 客户端实现。与 HTTP 客户端类似，它提供为各服务方法创建 Goa 端点的方法，随后可将其包装为与传输无关的客户端实现。

### 客户端创建

生成的 `NewClient` 函数会创建一个对象，用于创建与传输无关的客户端端点：

```go
// NewClient 为所有 calc 服务端点实例化 gRPC 客户端。
func NewClient(cc *grpc.ClientConn, opts ...grpc.CallOption) *Client {
    return &Client{
        grpccli: calcpb.NewCalcClient(cc),
        opts:    opts,
    }
}
```

该函数需要一个 gRPC 连接（`*grpc.ClientConn`），用于创建底层的 protobuf 客户端。它还接收可选的 gRPC 调用选项（`...grpc.CallOption`），用于配置 gRPC 客户端。

### gRPC 客户端

实例化后的 `Client` 结构体会暴露用于构建与传输无关端点的方法：

```go
// Add 返回一个对 calc 服务 add 服务端进行 gRPC 请求的端点。
func (c *Client) Add() goa.Endpoint

// Multiply 返回一个对 calc 服务 multiply 服务端进行 gRPC 请求的端点。
func (c *Client) Multiply() goa.Endpoint
```

### 综合示例

下面是为 `calc` 服务创建并使用 gRPC 客户端的示例：

```go
package main

import (
    "context"
    "log"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    
    gencalc "github.com/<your username>/calc/gen/calc"
    genclient "github.com/<your username>/calc/gen/grpc/calc/client"
)

func main() {
    // 创建 gRPC 连接
    conn, err := grpc.Dial("localhost:8080",
        grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        log.Fatal(err)
    }
    defer conn.Close()

    // 创建 gRPC 客户端
    grpcClient := genclient.NewClient(conn)

    // 创建端点客户端
    client := gencalc.NewClient(
        grpcClient.Add(),          // Add 端点
        grpcClient.Multiply(),     // Multiply 端点
    )

    // 调用服务方法
    result, err := client.Add(context.Background(), &gencalc.AddPayload{A: 1, B: 2})
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("1 + 2 = %d", result)
}
```

该示例展示了：
1. 创建 gRPC 连接
2. 基于该连接创建 gRPC 客户端
3. 包装为端点客户端
4. 使用客户端发起服务调用

gRPC 客户端会处理所有传输层相关事务，而端点客户端为服务调用提供了整洁的接口。

### 结语

生成的 gRPC 客户端为通过 gRPC 与服务交互提供了便捷途径。客户端处理 gRPC 通信的全部复杂性，同时提供与其他传输实现一致的接口。