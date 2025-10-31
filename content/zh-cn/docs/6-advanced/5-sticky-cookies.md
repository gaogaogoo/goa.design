---
title: "粘性 Cookie"
linkTitle: "粘性 Cookie"
description: "使用流式/websockets 服务的部署策略"
weight: 4
---

使用流式服务/websockets 时，部署需要额外步骤。

## 为什么需要粘性 Cookie？

您可能希望在 Kubernetes 中部署 Goa 服务并进行水平扩展。

通常场景是运行同一个 Docker 容器（副本数）。

负载均衡器（ingress、traefik、haproxy 等）会对发往公共端点的请求进行负载均衡以命中 pod。目标 pod 由负载均衡器选择，是随机/负载均衡的。

常规 REST (http) 或 gRPC 调用正是如此。

对于流式/websockets，特定 pod 的客户端/服务端需保持“它们的”连接。

为此，负载均衡器使用“粘性 Cookie”技术。

流式/websocket 客户端首次命中负载均衡器时生成 cookie，并通过 pod 响应返回客户端。

使用 websocket 时，客户端与服务端的连接持续开放。

- 客户端在下次调用服务端时使用并设置该 cookie。
- 负载均衡器使用 cookie 将请求路由到指定 pod。

**提示**：除非有明确理由，不要在 REST/gRPC 调用中使用粘性 Cookie。流式/websockets 在使用水平扩展时必须使用。

## 开发者测试框架

### Goa 端点

设计

```go
var _ = Service("dummy", func() {
    Description("Private functions")

    Method("hostname", func() {
        Result(func() {
            Field(1, "ip", String, "IP of the host")
            Field(2, "hostname", String, "Name of the host")
        })

        HTTP(func() {
            GET("/hostname")
        })
    })
})
```

实现

```go
func (s *dummysrvc) Hostname(ctx context.Context) (res *dummy.HostnameResult, err error) {
    res = &dummy.HostnameResult{}
    log.Printf(ctx, "dummy.hostname")

    hostname, _ := os.Hostname()
    addrs, _ := net.LookupIP(hostname)
    for _, addr := range addrs {
        if ipv4 := addr.To4(); ipv4 != nil {
            var str = ipv4.String()
            res.IP = &str
            break
        }
    }
    res.Hostname = &hostname

    return
}
```

假设您有一个名为 `my-service` 的 Goa 服务 Docker 容器，容器暴露 http 端口 `8000`。（待办：简单教程说明如何执行）。

```yml
---
services:
  server:
    image: my-service
    deploy:
      replicas: 3
    labels:
      - "traefik.http.routers.webapp.rule=Host(`localhost`)"
      - "traefik.http.services.webapp.loadbalancer.server.port=8000"
      - "traefik.http.services.webapp.loadbalancer.sticky.cookie.name=ws-session"
  traefik-lb:
    image: traefik:latest
    command: --api.insecure=true --providers.docker
    ports:
      - 8000:8000
      # traefik web interface
      - 8080:8080
    labels:
    - "traefik.http.routers.api.rule=Host(`admin.localhost`)"
    - "traefik.http.routers.api.insecure=true"
    - "traefik.http.routers.api.service=api@internal"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

**提示**：查看[此仓库](https://github.com/07prajwal2000/Auto-Scaling-Websockets-using-Traefik/tree/master)了解更多思路。

测试

```bash
# 这将给您来自 server-1, server-2, server-3 的响应
# （或者在未使用 traefik 负载均衡器运行时来自您的开发机器）
curl -vs "http://localhost:8000/hostname"
```

```bash
# 多次运行此命令 - 您将粘附到一个服务器

# 重要：为简化测试，我们使用 REST 示例！
# 不要为您的 REST 端点使用粘性 cookie（除非您知道原因）

curl -vs -c "cookies.txt" -b "cookies.txt" "http://localhost:8000/hostname"
```

```javascript
// 这是一个 javascript 客户端
// 大多数 WebSocket 库开箱即用地支持 cookie
import WebSocket from "ws";

// 查看 goa "Bidirectional Streaming Example"

const api_key = "secret";
const ws = new WebSocket("ws://localhost:8000/streaming", {
  // 您甚至可以使用 headers 进行认证
  // 查看 goa "Security" 示例
  headers: {
    Authorization: api_key,
  },
});

ws.on("error", console.error);

ws.on("open", function open() {
  console.log("connected");
  const json = {
    topic: "my topic",
  };
  const payload = JSON.stringify(json);
  ws.send(payload);
});

ws.on("close", function close() {
  console.log("disconnected");
});

ws.on("message", function message(data) {
  console.log("result", data.toString());

  setTimeout(function timeout() {
    const json = {
      topic: "" + Date.now(),
    };
    const payload = JSON.stringify(json);

    ws.send(payload);
  }, 500);
});
```

**提示**：使用 `docker compose logs -f` 验证始终使用同一服务器。
