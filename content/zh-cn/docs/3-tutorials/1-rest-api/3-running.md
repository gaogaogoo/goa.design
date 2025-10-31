---
title: 运行 Concerts 服务
linkTitle: Running
weight: 3
description: "学习如何运行基于 Goa 的 Concerts 服务，使用 HTTP 请求测试 REST 端点，并查看自动生成的 OpenAPI 文档。"
---

你已经完成了 API 设计并实现了服务方法。现在是时候运行 Concerts 服务并测试其端点了。

## 1. 启动服务器

在项目根目录构建并运行应用：

```bash
go run cmd/concerts/main.go
```

服务默认监听 8080 端口（除非在 `main.go` 中修改）。你会看到类似输出：

```
"List" mounted on GET /concerts
"Create" mounted on POST /concerts
"Show" mounted on GET /concerts/{concertID}
"Update" mounted on PUT /concerts/{concertID}
"Delete" mounted on DELETE /concerts/{concertID}
Starting concerts service on :8080
```

## 2. 测试端点

让我们探索你的新 API！你可以使用常见的 HTTP 工具与服务交互：

- `curl`：命令行快速测试
- [HTTPie](https://httpie.org)：更友好的 CLI 体验
- [Postman](https://www.postman.com/)：强大的 GUI，支持请求历史与集合

以下示例使用 `curl`，因为其在多数系统上都可用。你也可以使用自己喜欢的 HTTP 客户端，概念相同。

我们将测试以下内容：
- 创建演唱会（`POST`）
- 通过分页列出所有演唱会（`GET`）
- 获取指定演唱会（`GET`）
- 更新演唱会详情（`PUT`）
- 删除演唱会（`DELETE`）

### 创建演唱会

创建新演唱会。该请求以 JSON 格式发送演唱会详情，服务器会生成唯一 ID 并返回完整对象。

价格以美分存储（例如 8500 = $85.00）：

```bash
curl -X POST http://localhost:8080/concerts \
  -H "Content-Type: application/json" \
  -d '{
    "artist": "The White Stripes",
    "date": "2024-12-25",
    "venue": "Madison Square Garden, New York, NY",
    "price": 8500
  }'
```

再创建一条以演示分页：

```bash
curl -X POST http://localhost:8080/concerts \
  -H "Content-Type: application/json" \
  -d '{
    "artist": "Pink Floyd",
    "date": "2025-07-15", 
    "venue": "The O2 Arena, London, UK",
    "price": 12000
  }'
```

### 列出演唱会

可选的分页参数：
- `page`：页号（默认 1，最小 1）
- `limit`：每页数量（默认 10，范围 1–100）

示例：

```bash
# 使用默认分页
debug http://localhost:8080/concerts

# 每页 1 条
curl "http://localhost:8080/concerts?page=1&limit=1"

# 第 2 页，每页 5 条
curl "http://localhost:8080/concerts?page=2&limit=5"
```

### 查看指定演唱会

将 `<concertID>` 替换为创建时返回的 ID（例如 `550e8400-e29b-41d4-a716-446655440000`）：

```bash
curl http://localhost:8080/concerts/<concertID>
```

示例响应：
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "artist": "The White Stripes",
  "date": "2024-12-25",
  "venue": "Madison Square Garden, New York, NY",
  "price": 8500
}
```

### 更新演唱会

可修改多个字段：

```bash
curl -X PUT http://localhost:8080/concerts/<concertID> \
  -H "Content-Type: application/json" \
  -d '{
    "artist": "The White Stripes",
    "date": "2024-12-26",
    "venue": "Madison Square Garden, New York, NY",
    "price": 9000
  }'
```

仅更新价格：

```bash
curl -X PUT http://localhost:8080/concerts/<concertID> \
  -H "Content-Type: application/json" \
  -d '{
    "price": 9500
  }'
```

### 删除演唱会

```bash
curl -X DELETE http://localhost:8080/concerts/<concertID>
```

## 3. 错误处理

API 会返回一致的错误响应与合适的 HTTP 状态码：

### Not Found（404）
请求不存在的演唱会：

```bash
curl http://localhost:8080/concerts/invalid-id
```

响应：
```json
{
  "message": "Concert with ID invalid-id not found",
  "code": "not_found"
}
```

### Bad Request（400）
创建演唱会时提供无效数据：

```bash
curl -X POST http://localhost:8080/concerts \
  -H "Content-Type: application/json" \
  -d '{
    "artist": "",
    "date": "invalid-date",
    "venue": "",
    "price": -100
  }'
```

API 将返回以下校验错误：
- 艺术家名称为空（必须 1–200 字符）
- 日期格式无效（必须为 YYYY‑MM‑DD）
- 场地名称为空（必须 1–300 字符）
- 票价为负（必须 ≥ 0 且 ≤ 100000 美分）

## 4. 访问 API 文档

Goa 自动为你的 API 生成 OpenAPI 文档。服务运行后，你可以直接访问规范：

### OpenAPI 3.0

- JSON：`http://localhost:8080/openapi3.json`
- YAML：`http://localhost:8080/openapi3.yaml`

### 使用 Swagger UI

{{< alert title="快速设置" color="primary" >}}
1. 前提条件：已安装 Docker
2. 启动 Swagger UI：
   ```bash
   docker run -p 8081:8080 swaggerapi/swagger-ui
   ```
3. 浏览文档：
   - 打开 `http://localhost:8081`
   - 在 Swagger UI 中输入 `http://localhost:8080/openapi3.yaml`
{{< /alert >}}

### 其他文档工具
- Redoc：另一款流行的 OpenAPI 文档查看器
- OpenAPI Generator：生成多语言客户端库
- Speakeasy：生成更佳开发者体验的 SDK

## 下一步

现在你已经探索了基本的 API 操作，前往了解 Goa 如何处理 [HTTP 编解码](../4-encoding) 以理解请求与响应的处理方式。
