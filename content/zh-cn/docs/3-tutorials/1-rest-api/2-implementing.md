---
title: 实现
weight: 2
description: "循序渐进实现一个 Goa 的 REST API 服务，涵盖代码生成、服务实现、HTTP 服务器搭建与端点测试。"
---

在使用 Goa 的 DSL 设计完 REST API 后，就该开始实现服务了。本教程将一步一步带你完成实现过程。

1. 使用 Goa CLI 生成代码（`goa gen`）
2. 创建 `main.go` 来实现服务与 HTTP 服务器

## 1. 生成 Goa 工件

在项目根目录（例如 `concerts/`）运行 Goa 代码生成：

```bash
goa gen concerts/design
```

该命令会分析你的设计文件（`design/design.go`），并生成一个 `gen/` 目录，包含：
- 与传输无关的端点（在 `gen/concerts/`）
- HTTP 端（在 `gen/http/concerts/`）的校验与编解码代码，包含服务端与客户端
- OpenAPI 工件（在 `gen/http/`）

注意：如果你修改了设计（例如新增方法或字段），请重新运行 `goa gen` 以保持生成代码同步。

## 2. 浏览生成代码

让我们来了解生成代码的关键组成部分。理解这些文件对于正确实现服务并充分利用 Goa 的能力至关重要。

### gen/concerts

定义与传输协议无关的核心服务组件：
- 用于实现业务逻辑的服务接口（`service.go`）
- 与你的设计对应的 Payload 与 Result 类型
- `NewEndpoints` 函数，用于注入服务实现
- `NewClient` 函数，用于创建服务客户端

### gen/http/concerts/server

包含服务端 HTTP 相关逻辑：
- 包裹服务端点的 HTTP 处理器（handlers）
- 请求与响应的编解码逻辑
- 将请求路由至服务方法
- 面向传输的类型与校验
- 根据设计生成的路径（path）

### gen/http/concerts/client

提供客户端 HTTP 功能：
- 从 HTTP 端点创建客户端
- 请求与响应的编解码
- 路径生成函数
- 面向传输的类型与校验
- 命令行客户端工具的辅助函数

### OpenAPI 规范

`gen/http` 目录包含自动生成的 OpenAPI 规范：
- `openapi3.yaml` 与 `openapi3.json`（OpenAPI 3.0）

这些规范可直接用于 Swagger UI 与其他 API 工具，便于 API 浏览与客户端生成。

## 3. 实现你的服务

`gen/concerts/service.go` 中生成的服务接口定义了你需要实现的方法：

```go
type Service interface {
    // List concerts with optional pagination. Returns an array of concerts sorted by date.
    List(context.Context, *ListPayload) (res []*Concert, err error)
    // Create a new concert entry. All fields are required to ensure complete concert information.
    Create(context.Context, *ConcertPayload) (res *Concert, err error)
    // Get a single concert by its unique ID.
    Show(context.Context, *ShowPayload) (res *Concert, err error)
    // Update an existing concert by ID. Only provided fields will be updated.
    Update(context.Context, *UpdatePayload) (res *Concert, err error)
    // Remove a concert from the system by ID. This operation cannot be undone.
    Delete(context.Context, *DeletePayload) (err error)
}
```

### 实现流程

你的实现需要：

1. 创建一个实现该接口的服务结构体
2. 实现所有必需的方法
3. 与 HTTP 服务器进行装配（wire up）

在 `cmd/concerts/main.go` 创建文件并写入如下实现：

```go
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"

	"github.com/google/uuid"
	goahttp "goa.design/goa/v3/http"

	// 对生成包使用 gen 前缀
	genconcerts "concerts/gen/concerts"
	genhttp "concerts/gen/http/concerts/server"
)

// ConcertsService 实现 genconcerts.Service 接口
type ConcertsService struct {
	concerts []*genconcerts.Concert // 内存存储
}

// 列出将要到来的演唱会，支持分页。
func (m *ConcertsService) List(ctx context.Context, p *genconcerts.ListPayload) ([]*genconcerts.Concert, error) {
	start := (p.Page - 1) * p.Limit
	end := start + p.Limit
	if end > len(m.concerts) {
		end = len(m.concerts)
	}
	return m.concerts[start:end], nil
}

// 创建新的演唱会条目。
func (m *ConcertsService) Create(ctx context.Context, p *genconcerts.ConcertPayload) (*genconcerts.Concert, error) {
	newConcert := &genconcerts.Concert{
		ID:     uuid.New().String(),
		Artist: p.Artist,
		Date:   p.Date,
		Venue:  p.Venue,
		Price:  p.Price,
	}
	m.concerts = append(m.concerts, newConcert)
	return newConcert, nil
}

// 根据 ID 获取单条演唱会。
func (m *ConcertsService) Show(ctx context.Context, p *genconcerts.ShowPayload) (*genconcerts.Concert, error) {
	for _, concert := range m.concerts {
		if concert.ID == p.ConcertID {
			return concert, nil
		}
	}
	// 使用在设计中定义的错误
	return nil, genconcerts.MakeNotFound(fmt.Errorf("concert not found: %s", p.ConcertID))
}

// 根据 ID 更新已有演唱会。
func (m *ConcertsService) Update(ctx context.Context, p *genconcerts.UpdatePayload) (*genconcerts.Concert, error) {
	for i, concert := range m.concerts {
		if concert.ID == p.ConcertID {
			if p.Artist != nil {
				concert.Artist = *p.Artist
			}
			if p.Date != nil {
				concert.Date = *p.Date
			}
			if p.Venue != nil {
				concert.Venue = *p.Venue
			}
			if p.Price != nil {
				concert.Price = *p.Price
			}
			m.concerts[i] = concert
			return concert, nil
		}
	}
	return nil, genconcerts.MakeNotFound(fmt.Errorf("concert not found: %s", p.ConcertID))
}

// 根据 ID 从系统移除演唱会。
func (m *ConcertsService) Delete(ctx context.Context, p *genconcerts.DeletePayload) error {
	for i, concert := range m.concerts {
		if concert.ID == p.ConcertID {
			m.concerts = append(m.concerts[:i], m.concerts[i+1:]...)
			return nil
		}
	}
	return genconcerts.MakeNotFound(fmt.Errorf("concert not found: %s", p.ConcertID))
}

// main 实例化服务并启动 HTTP 服务器。
func main() {
	// 实例化服务
	svc := &ConcertsService{}

	// 用生成的端点进行包装
	endpoints := genconcerts.NewEndpoints(svc)

	// 构建 HTTP 处理器
	mux := goahttp.NewMuxer()
	requestDecoder := goahttp.RequestDecoder
	responseEncoder := goahttp.ResponseEncoder
	handler := genhttp.New(endpoints, mux, requestDecoder, responseEncoder, nil, nil)

	// 将处理器挂载到 mux
	genhttp.Mount(mux, handler)

	// 创建 HTTP 服务器
	port := "8080"
	server := &http.Server{Addr: ":" + port, Handler: mux}

	// 打印已挂载的路由
	for _, mount := range handler.Mounts {
		log.Printf("%q mounted on %s %s", mount.Method, mount.Verb, mount.Pattern)
	}

	// 启动服务器（阻塞执行）
	log.Printf("Starting concerts service on :%s", port)
	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

## 4. 关键实现细节

### 服务结构
- 内存存储：使用简单的切片示例化存储
- UUID 生成：使用 Google 的 UUID 库生成唯一标识
- 错误处理：利用 Goa 的设计化错误类型实现一致的响应

### 方法实现

#### List 方法
- 基于起止索引实现分页
- 处理 end 索引超过现有数量的边界情况
- 返回特定页的演唱会切片

#### Create 方法
- 为每个演唱会生成新的 UUID
- 从 payload 复制字段，构建完整记录
- 将新演唱会追加到内存存储

#### Show 方法
- 遍历演唱会并查找匹配的 ID
- 找到则返回演唱会
- 使用 `genconcerts.MakeNotFound()` 返回一致的错误响应

#### Update 方法
- 通过 ID 查找演唱会
- 仅更新提供的字段（部分更新）
- 通过指针检查判断要更新的字段
- 返回更新后的演唱会或未找到错误

#### Delete 方法
- 通过 ID 定位演唱会
- 使用切片操作移除之
- 若不存在则返回合适错误

### HTTP 服务器设置
- 使用 Goa 内置 HTTP 多路复用器
- 配置请求/响应编解码
- 自动挂载所有端点
- 打印所有挂载路由用于调试

## 5. 依赖

确保将必需的依赖加入 `go.mod`：

```bash
go mod tidy
```

这将自动加入：
- `goa.design/goa/v3` — Goa 核心框架
- `github.com/google/uuid` — UUID 生成

## 6. 运行与测试

1) 生成代码：
```bash
goa gen concerts/design
```

2) 运行服务（在项目根目录）：
```bash
go run cmd/concerts/main.go
```

你应看到类似输出：
```
"List" mounted on GET /concerts
"Create" mounted on POST /concerts
"Show" mounted on GET /concerts/{concertID}
"Update" mounted on PUT /concerts/{concertID}
"Delete" mounted on DELETE /concerts/{concertID}
Starting concerts service on :8080
```

3) 使用 curl 测试端点：
```bash
# 列出演唱会（初始为空）
curl http://localhost:8080/concerts

# 创建新演唱会
curl -X POST "http://localhost:8080/concerts" \
  -H "Content-Type: application/json" \
  -d '{
    "artist": "The White Stripes",
    "date": "2024-12-25",
    "venue": "Madison Square Garden, New York, NY",
    "price": 7500
  }'
```

恭喜！你已成功实现第一个 Goa 服务。该服务现在能够通过规范的校验、错误处理与 HTTP 状态码处理全部 CRUD 操作。继续前往 [运行](./3-running) 学习与服务交互并测试所有端点的不同方式！
