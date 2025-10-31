---
title: "实现"
linkTitle: "实现"
weight: 5
description: "学习如何使用 Goa 生成的代码实现 gRPC 服务，包括服务端与客户端实现"
---

本文介绍如何使用 Goa 生成的代码来实现 gRPC 服务。若需创建完整 gRPC 服务的分步演示，请参阅
[gRPC 服务教程](../../3-tutorials/2-grpc-service/2-implementing)。

## 传输无关性

Goa 的一个关键特性是业务逻辑实现与传输协议保持独立。生成于 `gen/service` 的服务接口是协议无关的，这使你可以：

1. 专注实现核心业务逻辑，而无需关心传输细节
2. 使用同一套服务实现同时支持多种传输（gRPC、HTTP 等）
3. 在不受传输影响的环境中独立测试业务逻辑

与传输相关的代码（此处为 gRPC）是单独生成的，它将你的服务实现适配到特定协议的要求。

## 代码生成概览

运行 `goa gen` 时，Goa 会生成以下组件：

```
gen/
├── grpc/
│   ├── pb/                   # Protocol Buffer 生成的代码
│   │   └── service.pb.go
│   ├── client/              # gRPC 客户端代码
│   │   └── client.go
│   └── server/              # gRPC 服务端代码
│       └── server.go
└── service/                 # 服务接口与类型
    └── service.go
```

关于生成与理解这些组件的详细说明，请参阅
[实现服务](../../3-tutorials/2-grpc-service/2-implementing/#1-generate-the-grpc-artifacts)
教程章节。

## gRPC 特性

### 流式通信

gRPC 支持服务端流、客户端流与双向流。有关在 Goa 中实现流式通信的详细信息（含示例与最佳实践），请参阅专门的[流式通信](../3-streaming)指南。

## 最佳实践

### 错误处理

使用 gRPC 专属错误码并包含有意义的元数据：

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *calculator) Divide(ctx context.Context, p *calc.DividePayload) (*calc.DivideResult, error) {
    if p.Divisor == 0 {
        return nil, status.Error(codes.InvalidArgument, "division by zero")
    }
    
    quotient := float64(p.Dividend) / float64(p.Divisor)
    return &calc.DivideResult{Quotient: quotient}, nil
}
```

### 上下文使用

正确处理 gRPC 上下文的取消与超时：

```go
func (s *calculator) LongOperation(ctx context.Context, p *calc.LongOperationPayload) (*calc.LongOperationResult, error) {
    select {
    case <-ctx.Done():
        return nil, status.Error(codes.Canceled, "operation canceled")
    case result := <-s.processAsync(p):
        return result, nil
    }
}
```

### 资源管理

正确管理 gRPC 连接与资源：

```go
type grpcServer struct {
    server *grpc.Server
    lis    net.Listener
}

func NewGRPCServer(svc calc.Service) (*grpcServer, error) {
    srv := grpc.NewServer()
    calcsvr.Register(srv, svc)
    
    lis, err := net.Listen("tcp", ":8080")
    if err != nil {
        return nil, err
    }
    
    return &grpcServer{
        server: srv,
        lis:    lis,
    }, nil
}

func (s *grpcServer) Start() error {
    return s.server.Serve(s.lis)
}

func (s *grpcServer) Stop() {
    s.server.GracefulStop()
    s.lis.Close()
}
```

关于设置具备完善错误处理、日志与优雅关闭的 gRPC 服务端的完整示例，请参阅教程中的
[服务器设置](../../3-tutorials/2-grpc-service/2-implementing/#4-setting-up-the-grpc-server)
章节。