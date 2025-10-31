---
title: 设计一个 REST API
linkTitle: 设计
weight: 1
description: "学习使用 Goa 为演唱会管理设计完整的 REST API，包括 CRUD 操作、分页、规范的 HTTP 映射与错误处理。"
---

本教程将带您使用 Goa 设计一个用于管理音乐演唱会的 REST API。您将学会创建完整的 API 设计，涵盖常见操作、规范的 HTTP 映射与错误处理。

## 我们将构建什么

我们将创建一个 `concerts` 服务，提供以下标准 REST 操作：

| Operation | HTTP Method | Path | Description |
|-----------|------------|------|-------------|
| List | GET | /concerts | 通过分页获取所有演唱会 |
| Create | POST | /concerts | 新增一场演唱会 |
| Show | GET | /concerts/{id} | 获取单条演唱会信息 |
| Update | PUT | /concerts/{id} | 修改演唱会信息 |
| Delete | DELETE | /concerts/{id} | 删除演唱会 |

## 设计文件

首先创建一个新的 Go module 来承载服务：

```bash
mkdir concerts
cd concerts
go mod init concerts
```

在 `design/design.go` 新建文件并写入以下内容：

```go
package design

import (
	. "goa.design/goa/v3/dsl"
)

// API 定义
var _ = API("concerts", func() {
	Title("Concert Management API")
	Description("A simple API for managing music concert information")
	Version("1.0")

	Server("concerts", func() {
		Description("Concert management server")
		Host("localhost", func() {
			URI("http://localhost:8080")
		})
	})
})

// 服务定义
var _ = Service("concerts", func() {
	Description("The concerts service manages music concert data. It provides CRUD operations for concert information including artist details, venues, dates, and pricing.")

	Method("list", func() {
		Description("List concerts with optional pagination. Returns an array of concerts sorted by date.")
		Meta("openapi:summary", "List all concerts")

		Payload(func() {
			Attribute("page", Int, "分页页码", func() {
				Minimum(1)
				Default(1)
				Example(1)
				Description("必须不小于 1")
			})
			Attribute("limit", Int, "每页条数", func() {
				Minimum(1)
				Maximum(100)
				Default(10)
				Example(10)
				Description("取值范围 1 到 100")
			})
		})

		Result(ArrayOf(Concert), func() {
			Description("演唱会列表")
		})

		HTTP(func() {
			GET("/concerts")

			Param("page", Int, "页码", func() {
				Minimum(1)
				Example(1)
			})
			Param("limit", Int, "每页条数", func() {
				Minimum(1)
				Maximum(100)
				Example(10)
			})

			Response(StatusOK)
		})
	})

	Method("create", func() {
		Description("创建新的演唱会条目。为确保信息完整，所有字段均为必填。")
		Meta("openapi:summary", "Create a new concert")

		Payload(ConcertPayload, "用于创建的演唱会信息")

		Result(Concert, "新创建的演唱会")

		Error("bad_request", ErrorResult, "输入数据无效")

		HTTP(func() {
			POST("/concerts")

			Response(StatusCreated)
			Response("bad_request", StatusBadRequest)
		})
	})

	Method("show", func() {
		Description("根据唯一 ID 获取单条演唱会信息。")
		Meta("openapi:summary", "Get concert by ID")

		Payload(func() {
			Attribute("concertID", String, "演唱会唯一标识", func() {
				Format(FormatUUID)
				Example("550e8400-e29b-41d4-a716-446655440000")
			})
			Required("concertID")
		})

		Result(Concert, "请求到的演唱会")

		Error("not_found", ErrorResult, "指定 ID 的演唱会不存在")

		HTTP(func() {
			GET("/concerts/{concertID}")

			Response(StatusOK)
			Response("not_found", StatusNotFound)
		})
	})

	Method("update", func() {
		Description("根据 ID 更新现有演唱会，仅更新提供的字段。")
		Meta("openapi:summary", "Update concert")

		Payload(func() {
			Extend(ConcertData)
			Attribute("concertID", String, "待更新的演唱会 ID", func() {
				Format(FormatUUID)
				Example("550e8400-e29b-41d4-a716-446655440000")
			})
			Required("concertID")
		})

		Result(Concert, "更新后的完整演唱会信息")

		Error("not_found", ErrorResult, "指定 ID 的演唱会不存在")
		Error("bad_request", ErrorResult, "更新数据无效")

		HTTP(func() {
			PUT("/concerts/{concertID}")

			Response(StatusOK)
			Response("not_found", StatusNotFound)
			Response("bad_request", StatusBadRequest)
		})
	})

	Method("delete", func() {
		Description("根据 ID 从系统移除演唱会。此操作不可撤销。")
		Meta("openapi:summary", "Delete concert")

		Payload(func() {
			Attribute("concertID", String, "待删除的演唱会 ID", func() {
				Format(FormatUUID)
				Example("550e8400-e29b-41d4-a716-446655440000")
			})
			Required("concertID")
		})

		Error("not_found", ErrorResult, "指定 ID 的演唱会不存在")

		HTTP(func() {
			DELETE("/concerts/{concertID}")

			Response(StatusNoContent)
			Response("not_found", StatusNotFound)
		})
	})
})

// 数据类型
var ConcertData = Type("ConcertData", func() {
	Description("演唱会信息字段。")

	Attribute("artist", String, "表演者或乐队名称", func() {
		MinLength(1)
		MaxLength(200)
		Example("The White Stripes")
		Description("该演唱会的主要表演者")
	})

	Attribute("date", String, "演唱会日期（ISO 8601 格式 YYYY-MM-DD）", func() {
		Format(FormatDate)
		Example("2024-12-25")
		Description("演唱会举行的日期")
	})

	Attribute("venue", String, "演唱会地点名称与位置", func() {
		MinLength(1)
		MaxLength(300)
		Example("Madison Square Garden, New York, NY")
		Description("演唱会举办的场地")
	})

	Attribute("price", Int, "票价（美元，单位为美分）", func() {
		Minimum(0)
		Maximum(100000) // $1000 最大值
		Example(7500)   // $75.00
		Description("基础票价，单位美分（例如 7500 = $75.00）")
	})
})

var ConcertPayload = Type("ConcertPayload", func() {
	Description("创建演唱会条目所需的信息。")

	Extend(ConcertData)

	// 创建时所有字段必填
	Required("artist", "date", "venue", "price")
})

var Concert = Type("Concert", func() {
	Description("包含系统生成 ID 的完整演唱会信息。")

	Attribute("id", String, "演唱会唯一标识", func() {
		Format(FormatUUID)
		Example("550e8400-e29b-41d4-a716-446655440000")
		Description("系统生成的唯一标识符")
	})

	Extend(ConcertData)

	Required("id", "artist", "date", "venue", "price")
})
```

## 理解设计

要查看本教程中所用到的所有 DSL 函数的完整参考，请访问
[Goa DSL 文档](https://pkg.go.dev/goa.design/goa/v3/dsl)。每个函数都配有详细说明与实用示例。

### 1. API 定义
设计从 API 定义开始，设置整体服务：
- 标题与描述：明确标识 API 的用途
- 版本：用于客户端兼容性
- 服务器配置：默认主机与端口设置

### 2. 服务结构
服务由五个遵循 REST 约定的方法组成：
- list：GET `/concerts`，支持分页
- create：POST `/concerts`，新增演唱会
- show：GET `/concerts/{id}`，获取指定演唱会
- update：PUT `/concerts/{id}`，修改演唱会
- delete：DELETE `/concerts/{id}`，删除演唱会

### 3. 关键特性

#### HTTP 映射
API 遵循 REST 约定并提供直观的 HTTP 映射：
- 使用 `GET` 获取数据（列表与单条）
- 使用 `POST` 创建新演唱会
- 使用 `PUT` 更新现有演唱会
- 使用 `DELETE` 删除演唱会
- 通过查询参数（`page` 和 `limit`）处理分页
- 通过路径参数捕获资源 ID（例如 `/concerts/{concertID}`）

#### 数据校验
Goa 提供全面的内置校验：
- 演唱会 ID：必须为合法 UUID，并提供示例
- 表演者名称：长度 1–200，提供有意义示例
- 演唱会日期：校验 ISO 8601 格式（YYYY‑MM‑DD）
- 场地：长度 1–300，支持完整场地描述
- 票价：非负整数，最大 $1000（以美分存储）
- 分页：Page ≥ 1，Limit 1–100，并提供合理默认值

#### 错误处理
API 以优雅方式处理错误：
- 命名错误类型：`not_found` 与 `bad_request`，附清晰描述
- 适当的 HTTP 状态码：如 404（未找到）、400（错误请求）等
- 使用 `ErrorResult` 保持一致的错误响应格式
- 详尽的错误消息便于调试

#### 类型架构
设计采用分层类型以获得最大灵活性：
- ConcertData：基础类型，包含所有演唱会字段，不强制必填
- ConcertPayload：用于创建，继承基础类型并要求所有字段必填
- Concert：用于响应，完整演唱会（含 ID 与所有详情）

该结构使创建时要求完整数据，而更新时可部分更新，提供简洁直观的 API 体验。

#### OpenAPI 集成
设计包含面向 OpenAPI 的元数据：
- 为更佳文档提供摘要注解（summary）
- 为每个端点与字段提供详细描述
- 示例（examples）出现在生成文档中
- 为 API 使用者提供清晰的参数描述

## 下一步

现在您已有完整的 API 设计，请继续阅读[实现教程](./2-implementing)，了解如何使用 Goa 的代码生成将设计落地。
