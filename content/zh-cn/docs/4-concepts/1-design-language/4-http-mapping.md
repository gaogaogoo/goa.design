---
title: "HTTP 传输映射"
linkTitle: "HTTP 映射"
weight: 4
description: >
  定义服务方法如何映射到 HTTP 端点。学习如何将载荷映射到 HTTP 请求与响应。
---

本节说明服务方法的载荷如何通过 [HTTP](https://pkg.go.dev/goa.design/goa/v3/dsl#HTTP) 传输 DSL 映射到 HTTP 端点。
载荷类型定义了传入服务方法的参数数据结构，而 HTTP 表达式则指定如何从到达的 HTTP 请求各部分构建这些数据。

## HTTP 请求组成

一个 HTTP 请求由四部分组成：

1. URL 路径参数  
   例如在路由 `/bottle/{id}` 中，`{id}` 即为路径参数。

2. URL 查询字符串参数

3. HTTP 头（Headers）

4. HTTP 请求体（Body）

[HTTP 表达式](https://goa.design/reference/dsl/http/) 指导生成的代码如何将请求解码为期望的载荷：

- Param 表达式：从路径或查询字符串参数加载值。
- Header 表达式：从 HTTP 头加载值。
- Body 表达式：从请求体加载值。

下文将更详细地介绍这些表达式。

---

## 非对象类型载荷的映射

当载荷类型为原始类型（如 `String`、整数类型、`Float`、`Boolean` 或 `Bytes`）、数组或映射时，值将按以下顺序从第一个已定义的元素中加载：

1. 第一个 URL 路径参数（若已定义）
2. 否则，第一个查询字符串参数（若已定义）
3. 否则，第一个头（若已定义）
4. 否则，请求体

### 限制

- 路径参数与头：必须使用原始类型或原始类型数组定义。
- 查询字符串参数：可以是原始类型、数组或映射（其元素为原始类型）。
- 路径与头中的数组：以逗号分隔的值表示。

### 示例

1. 按标识符获取（整数 ID）：

```go
Method("show", func() {
    Payload(Int)
    HTTP(func() {
        GET("/{id}")
    })
})
```

| 生成的方法 | 示例请求 | 对应调用 |
| ---------- | -------- | -------- |
| `Show(int)`| `GET /1` | `Show(1)`|

2. 批量删除（字符串 ID）：

```go
Method("delete", func() {
    Payload(ArrayOf(String))
    HTTP(func() {
        DELETE("/{ids}")
    })
})
```

| 生成的方法         | 示例请求     | 对应调用                         |
| ------------------ | ------------ | -------------------------------- |
| `Delete([]string)` | `DELETE /a,b`| `Delete([]string{"a", "b"})`     |

> 注意：路径参数的具体名称并不重要。

3. 查询字符串中的数组：

```go
Method("list", func() {
    Payload(ArrayOf(String))
    HTTP(func() {
        GET("")
        Param("filter")
    })
})
```

| 生成的方法       | 示例请求                     | 对应调用                      |
| ---------------- | ---------------------------- | ----------------------------- |
| `List([]string)` | `GET /?filter=a&filter=b`    | `List([]string{"a", "b"})`   |

4. 头中的浮点数：

```go
Method("list", func() {
    Payload(Float32)
    HTTP(func() {
        GET("")
        Header("version")
    })
})
```

| 生成的方法       | 示例请求                     | 对应调用 |
| ---------------- | ---------------------------- | -------- |
| `List(float32)`  | `GET /` 且头 `version=1.0`   | `List(1.0)` |

5. 请求体中的映射：

```go
Method("create", func() {
    Payload(MapOf(String, Int))
    HTTP(func() {
        POST("")
    })
})
```

| 生成的方法                 | 示例请求                 | 对应调用                                      |
| -------------------------- | ------------------------ | --------------------------------------------- |
| `Create(map[string]int)`   | `POST / {"a": 1, "b": 2}` | `Create(map[string]int{"a": 1, "b": 2})`   |

---

## 对象类型载荷的映射

当载荷被定义为对象（包含多个属性）时，HTTP 表达式允许你指定每个属性的来源。有些属性来自 URL 路径，有些来自查询参数、头，或请求体。类型限制同上：

- 路径与头属性：必须是原始类型或原始类型数组。
- 查询字符串属性：可以是原始类型、数组或映射（其元素为原始类型）。

### 使用 `Body` 表达式

`Body` 表达式指定哪个载荷属性对应 HTTP 请求体。若省略 `Body` 表达式，则任何未映射到路径、查询或头的属性都默认来自请求体。

#### 示例：路径与请求体混合

给定载荷：

```go
Method("create", func() {
    Payload(func() {
        Attribute("id", Int)
        Attribute("name", String)
        Attribute("age", Int)
    })
})
```

以下 HTTP 表达式将 `id` 属性映射到路径参数，并将剩余属性映射到请求体：

```go
Method("create", func() {
    Payload(func() {
        Attribute("id", Int)
        Attribute("name", String)
        Attribute("age", Int)
    })
    HTTP(func() {
        POST("/{id}")
    })
})
```

| 生成的方法               | 示例请求                      | 对应调用                                      |
| ------------------------ | ----------------------------- | --------------------------------------------- |
| `Create(*CreatePayload)` | `POST /1 {"name": "a", "age": 2}` | `Create(&CreatePayload{ID: 1, Name: "a", Age: 2})` |

### 非对象类型也可使用 `Body`

`Body` 表达式也适用于请求体非对象（例如数组或映射）的场景。

#### 示例：请求体中的映射

载荷：

```go
Method("rate", func() {
    Payload(func() {
        Attribute("id", Int)
        Attribute("rates", MapOf(String, Float64))
    })
})
```

HTTP 表达式：

```go
Method("rate", func() {
    Payload(func() {
        Attribute("id", Int)
        Attribute("rates", MapOf(String, Float64))
    })
    HTTP(func() {
        PUT("/{id}")
        Body("rates")
    })
})
```

| 生成的方法           | 示例请求                    | 对应调用                                                             |
| -------------------- | --------------------------- | -------------------------------------------------------------------- |
| `Rate(*RatePayload)` | `PUT /1 {"a": 0.5, "b": 1.0}` | `Rate(&RatePayload{ID: 1, Rates: map[string]float64{"a": 0.5, "b": 1.0}})` |

若不使用 `Body` 表达式，请求体会被解释为一个仅包含 `rates` 字段的对象。

---

## 将 HTTP 元素名映射到属性名

`Param`、`Header` 与 `Body` 表达式允许你将 HTTP 元素名称（如查询字符串键、头名称或请求体字段名）映射到载荷属性名。语法如下：

```go
"attribute name:element name"
```

例如：

```go
Header("version:X-Api-Version")
```

此时 `version` 属性将从 HTTP 头 `X-Api-Version` 中加载。

### 在请求体中映射字段

`Body` 表达式还支持一种替代语法，可显式列出请求体属性及其对应的 HTTP 字段名。该语法适用于需要指定传入字段名与载荷属性名之间的映射关系：

#### 示例

```go
Method("create", func() {
    Payload(func() {
        Attribute("name", String)
        Attribute("age", Int)
    })
    HTTP(func() {
        POST("")
        Body(func() {
            Attribute("name:n")
            Attribute("age:a")
        })
    })
})
```

| 生成的方法               | 示例请求                | 对应调用                               |
| ------------------------ | ----------------------- | -------------------------------------- |
| `Create(*CreatePayload)` | `POST / {"n": "a", "a": 2}` | `Create(&CreatePayload{Name: "a", Age: 2})` |

---

本指南可帮助你将服务方法有效映射到 HTTP 端点，并清晰说明如何提取与映射 HTTP 请求的不同部分到服务载荷。