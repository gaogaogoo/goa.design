---
title: "运行服务"
linkTitle: "运行"
weight: 3
description: "使用 gRPC CLI、grpcurl 或自定义 Go 客户端运行并测试 Goa gRPC 服务，含常用示例与实践。"
---

完成设计与实现后，下一步是在本地将服务运行起来并验证行为：

1. 启动 gRPC 服务器
2. 使用 gRPC 工具进行调用测试
3. 总结真实使用中的常见后续步骤

## 1. 启动服务器

在项目根目录（如 `grpcgreeter/`）运行 `cmd/greeter/` 下的入口：

```bash
go run grpcgreeter/cmd/greeter
```

正常启动后，服务会监听 `:8090`（由 `main.go` 指定）。您应看到类似日志：

```
gRPC greeter service listening on :8090
```

表示服务已就绪，可接收 gRPC 请求。

## 2. 测试服务

### gRPC CLI

若已安装官方 gRPC CLI（macOS 可 `brew install grpc`），可以直接调用：

```bash
grpc_cli call localhost:8090 SayHello "name: 'Alice'"
```

由于启用了 server reflection，该调用可成功解析服务与方法。

### grpcurl

[gRPCurl](https://github.com/fullstorydev/grpcurl)（macOS 可 `brew install grpcurl`）也是常用测试工具：

```bash
grpcurl -plaintext -d '{"name": "Alice"}' localhost:8090 greeter.Greeter/SayHello
```

### 自定义客户端

也可使用生成的客户端代码编写一个小型 Go 客户端：

```go
package main

import (
    "context"
    "fmt"
    "log"

    gengreeter "grpcgreeter/gen/greeter"
    genclient "grpcgreeter/gen/grpc/greeter/client"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

func main() {
    conn, err := grpc.Dial("localhost:8090", grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
    	log.Fatalf("Failed to connect: %v", err)
    }
    defer conn.Close()

    grpcc := genclient.NewClient(conn)
    c := gengreeter.NewClient(grpcc.SayHello())

    res, err := c.SayHello(context.Background(), &gengreeter.SayHelloPayload{"Alice"})
    if err != nil {
    	log.Fatalf("Error calling SayHello: %v", err)
    }

    fmt.Printf("Server response: %s\n", res.Greeting)
}
```

编译并运行后，将打印服务返回的问候语。

---

至此，您已运行并测试了一个使用 Goa 构建的 **gRPC 服务**。继续探索 DSL 可添加更多能力，如 **流式处理（streaming）**、**认证拦截器**、以及针对多环境的自动代码生成，迈向更健壮的 Go 微服务架构。