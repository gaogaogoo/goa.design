---
title: "HTTP 路由"
linkTitle: "路由"
weight: 2
description: "了解 Goa 如何处理 HTTP 路由，包括路径模式、参数、通配符，以及设计干净 URL 的最佳实践。"
menu:
  main:
    parent: "HTTP Advanced Topics"
    weight: 2
---

Goa 提供了强大的路由系统，可将 HTTP 请求映射到服务方法。本文将介绍：

- 路由基础概念与服务定义
- HTTP 方法与 URL 路径模式
- 参数处理（路径、查询和通配符）
- 响应状态码
- API 设计最佳实践
- 服务关系与嵌套资源

## 路由基础

在 Goa 中，路由通过 Service 定义内的 `HTTP` 函数在设计中声明。`HTTP` 函数用于指定服务方法如何通过 HTTP 暴露。

基础示例：

```go
var _ = Service("calculator", func() {
    // 定义服务级别的 HTTP 设置
    HTTP(func() {
        // 为该服务的所有端点设置基础路径
        Path("/calculator")
    })

    Method("add", func() {
        // 定义方法载荷
        Payload(func() {
            // 字段顺序重要 - tag 1 为第一个
            Field(1, "a", Int, "第一个操作数")
            Field(2, "b", Int, "第二个操作数")
        })
        // 定义方法结果
        Result(Int)
        // 定义 HTTP 传输
        HTTP(func() {
            POST("/add")     // 处理 POST /calculator/add
        })
    })
})
```

上述示例：
1. 创建名为 "calculator" 的服务
2. 为所有端点设置基础路径 "/calculator"
3. 定义一个 "add" 方法：
   - 接受两个整数作为输入
   - 返回一个整数
   - 通过 HTTP POST 访问路径 "/calculator/add"

## HTTP 方法与路径

Goa 通过专用 DSL 函数支持所有标准 HTTP 方法：`GET`、`POST`、`PUT`、`DELETE`、`PATCH`、`HEAD`、`OPTIONS` 和 `TRACE`。一个服务方法可以同时处理多个 HTTP 方法或路径：

```go
Method("manage_user", func() {
    Description("创建或更新用户")
    Payload(User)
    Result(User)
    HTTP(func() {
        POST("/users")          // 创建用户
        PUT("/users/{user_id}") // 更新现有用户
        Response(StatusOK)      // 更新返回 200
        Response(StatusCreated) // 创建返回 201
    })
})
```

## 参数处理

### 路径参数

可通过路径参数从 URL 路径中捕获动态值。路径参数使用花括号 `{parameter_name}` 定义，并自动映射到载荷字段。

```go
Method("get_user", func() {
    Description("根据 ID 检索用户")
    Payload(func() {
        // user_id 字段将从 URL 路径中填充
        Field(1, "user_id", String, "来自 URL 路径的用户 ID")
    })
    Result(User)
    HTTP(func() {
        GET("/users/{user_id}")  // 将 {user_id} 映射到 payload.UserID
    })
})
```

### 参数类型与映射

在 Goa 中，参数类型在载荷定义中声明，而不是在 URL 模式中。URL 模式仅用于定义如何将传输名称映射到载荷字段。

1. **原始类型载荷**
   ```go
   Method("get_user", func() {
       // 当载荷为原始类型时，直接映射到路径参数
       Payload(String, "用户 ID")
       Result(User)
       HTTP(func() {
           GET("/users/{user_id}")  // user_id 的值成为整个载荷
       })
   })
   ```

2. **结构化载荷的直接映射**
   ```go
   Method("get_user", func() {
       Payload(func() {
           // 参数类型（Int）在载荷中声明
           Field(1, "user_id", Int, "用户 ID")
       })
       Result(User)
       HTTP(func() {
           GET("/users/{user_id}")  // 直接映射到 payload.UserID
       })
   })
   ```

3. **传输名称映射**
   ```go
   Method("get_user", func() {
       Payload(func() {
           // 内部字段名为 "id"
           Field(1, "id", Int, "用户 ID")
       })
       HTTP(func() {
           // URL 使用 user_id，但映射到 payload.ID
           GET("/users/{user_id:id}")
       })
   })
   ```

路径模式中的 `{name:field}` 语法仅用于名称映射：
- `name` 是 URL 中出现的名称
- `field` 是载荷中的字段名

对于原始类型载荷，路径参数值会成为整个载荷：

```go
Method("download", func() {
    // 整个载荷是表示文件路径的字符串
    Payload(String, "文件路径")
    HTTP(func() {
        GET("/files/{*path}")  // 捕获的路径成为载荷
    })
}

Method("get_version", func() {
    // 载荷是一个简单的整数
    Payload(Int, "API 版本号")
    HTTP(func() {
        GET("/api/{version}")  // 版本号成为载荷
    })
})
```

使用结构化载荷时，可将路径参数与其他载荷字段组合：

```go
Method("update_user_profile", func() {
    Payload(func() {
        // 路径参数
        Field(1, "id", Int, "用户 ID")
        // 请求体字段
        Field(2, "name", String, "用户姓名")
        Field(3, "email", String, "用户邮箱")
    })
    HTTP(func() {
        PUT("/users/{user_id:id}")  // 将 URL 的 user_id 映射到 payload.ID
        Body("name", "email")       // 这些字段来自请求体
    })
})

### 查询参数

查询字符串参数通过 `Param` 函数定义，且必须对应载荷字段。你可以设置默认值和校验规则：

```go
Method("list_users", func() {
    Description("分页列出用户")
    Payload(func() {
        Field(1, "page", Int, "页码", func() {
            Default(1)        // 默认第 1 页
            Minimum(1)        // 页码必须为正
        })
        Field(2, "per_page", Int, "每页数量", func() {
            Default(20)       // 默认每页 20 条
            Minimum(1)
            Maximum(100)      // 限制最大数量
        })
    })
    Result(CollectionOf(User))
    HTTP(func() {
        GET("/users")
        // 将载荷字段映射到查询参数
        Param("page")
        Param("per_page")
    })
})

### 通配符与兜底路由

为了灵活的路径匹配，使用星号语法（`*path`）以捕获剩余的所有路径段。捕获的值会在载荷中可用：

```go
Method("serve_files", func() {
    Description("从目录服务静态文件")
    Payload(func() {
        // path 字段将包含 /files/ 之后的所有段
        Field(1, "path", String, "文件路径")
    })
    HTTP(func() {
        GET("/files/*path")    // 匹配 /files/docs/image.png
    })
})
```

## API 设计最佳实践

### 资源命名

使用名词表示资源，让 HTTP 方法决定动作：

```go
HTTP(func() {
    // 推荐 - 使用 HTTP 方法表达动作
    GET("/articles")        // 列出文章
    POST("/articles")       // 创建文章
    GET("/articles/{id}")   // 获取单篇文章
    PUT("/articles/{id}")   // 更新文章
    DELETE("/articles/{id}") // 删除文章

    // 避免 - 将动作写入 URL
    GET("/list-articles")
    POST("/create-article")
})
```

### 复数一致性

集合端点使用复数名词并保持一致：

```go
HTTP(func() {
    // 推荐 - 始终使用复数
    GET("/users")          // 列出用户
    GET("/users/{id}")     // 获取单个用户
    POST("/users")         // 创建用户
    
    // 避免混用单复数
    GET("/user")          // 不要使用单数
    GET("/users/{id}")    // 不要混用约定
})
```

### 路径前缀层级

Goa 允许在 API 设计的不同层级定义路径前缀：

1. **API 层级** - 作用于所有服务：
```go
var _ = API("myapi", func() {
    HTTP(func() {
        Path("/api")  // 所有服务的全局前缀
    })
})
```

2. **服务层级** - 作用于服务内所有方法：
```go
var _ = Service("users", func() {
    HTTP(func() {
        Path("/v1/users")  // 该服务所有方法的前缀
    })
})
```

最终 URL 路径通过按顺序组合这些前缀构造。例如：

```go
var _ = API("myapi", func() {
    HTTP(func() {
        Path("/api")  // API 级前缀
    })

    Service("users", func() {
        HTTP(func() {
            Path("/v1/users")  // 服务级前缀
        })

        Method("show", func() {
            Payload(func() {
                Field(1, "id", Int)
            })
            HTTP(func() {
                GET("/{id}")  // 方法路径
            })
        })
    })
})
```

这将使 show 方法的路径成为 `/api/v1/users/{id}`。

### API 版本

使用路径前缀为 API 进行版本化：

```go
var _ = Service("users", func() {
    HTTP(func() {
        Path("/v1")  // 所有端点都在 /v1 下
    })
    
    Method("list", func() {
        HTTP(func() {
            GET("/users")  // 最终路径：/v1/users
        })
    })
})
```

## 服务关系

### 父服务

Goa 提供 `Parent` DSL 来建立服务之间的关系。指定父服务后：
1. 父服务的规范路径将作为所有子服务 HTTP 端点的前缀
2. 父方法中映射到路径参数的载荷属性会自动合并到子方法的载荷中

### 规范方法（Canonical Method）

默认情况下，Goa 使用 "show" 方法作为服务的规范方法。规范方法的 HTTP 路径将用作所有子服务端点的前缀。你可以在服务的 HTTP 表达式中使用 `CanonicalMethod` 函数进行覆盖。

示例：

```go
var _ = Service("users", func() {
    HTTP(func() {
        Path("/users/{user_id}")
        // 覆盖默认的 "show" 方法
        CanonicalMethod("get")
    })
    
    Method("get", func() {
        Payload(func() {
            Field(1, "user_id", String)
        })
        HTTP(func() {
            GET("")  // 结果路径为 /users/{user_id}
        })
    })
})

var _ = Service("posts", func() {
    // 指定 users 作为父服务
    Parent("users")
    
    Method("list", func() {
        // user_id 会自动从父服务的规范方法载荷继承
        HTTP(func() {
            GET("/posts")  // 结果路径为 /users/{user_id}/posts
        })
    })
})
```

在此示例中：
1. `users` 服务将规范方法指定为 "get"，而不是默认的 "show"
2. 规范方法的路径（`/users/{user_id}`）成为所有子服务端点的前缀
3. `posts` 服务继承该前缀以及来自父服务规范方法的 `user_id` 参数
4. `list` 方法的最终路径为 `/users/{user_id}/posts`

### 嵌套资源

通过嵌套路径表达资源关系。可使用上面的 Parent DSL，或显式定义嵌套路径：

```go
var _ = Service("social", func() {
    // 定义用户资源的方法
    Method("list_users", func() {
        HTTP(func() {
            GET("/users")  // 列出所有用户
        })
    })

    Method("get_user", func() {
        Payload(func() {
            Field(1, "user_id", String, "用户 ID")
        })
        HTTP(func() {
            GET("/users/{user_id}")  // 获取指定用户
        })
    })

    // 用户下的文章方法
    Method("list_user_posts", func() {
        Payload(func() {
            Field(1, "user_id", String, "用户 ID")
            Field(2, "limit", Int, "返回的最大帖子数量")
        })
        HTTP(func() {
            GET("/users/{user_id}/posts")  // 列出某用户的帖子
            Param("limit")  // 查询参数
        })
    })

    Method("create_user_post", func() {
        Payload(func() {
            Field(1, "user_id", String, "用户 ID")
            Field(2, "title", String, "帖子标题")
            Field(3, "content", String, "帖子内容")
        })
        HTTP(func() {
            POST("/users/{user_id}/posts")  // 为用户创建帖子
            Body("title", "content")  // 这些字段放入请求体
        })
    })

    // 帖子下的评论方法
    Method("list_post_comments", func() {
        Payload(func() {
            Field(1, "user_id", String, "用户 ID")
            Field(2, "post_id", String, "帖子 ID")
        })
        HTTP(func() {
            GET("/users/{user_id}/posts/{post_id}/comments")  // 列出某帖子的评论
        })
    })
})
```

这种方式：
1. 在 URL 中建立清晰的层级（users → posts → comments）
2. 明确资源之间的关系
3. 保持访问嵌套资源的一致性
4. 允许对操作进行正确的作用域限定（例如仅限特定用户下的帖子）

你也可以通过服务级路径前缀将相关端点归组：

```go
var _ = Service("social", func() {
    HTTP(func() {
        Path("/v1/social")  // 该服务所有端点的前缀
    })
    
    Method("list_users", func() {
        HTTP(func() {
            GET("/users")  // 最终路径：/v1/social/users
        })
    })
    
    Method("get_user_posts", func() {
        HTTP(func() {
            GET("/users/{user_id}/posts")  // 最终路径：/v1/social/users/{user_id}/posts
        })
    })
})
```

## 生成代码

Goa 会根据你的设计生成所有必要的路由代码。生成的代码包括：

1. **URL 映射**：将 HTTP 请求路由到相应的服务方法
2. **参数处理**：
   - 提取并校验路径参数
   - 处理查询参数
   - 解析请求体
3. **内容协商**：
   - 处理 Accept 头
   - 管理响应格式
4. **错误处理**：
   - 将错误映射到 HTTP 状态码
   - 生成一致的错误响应

这意味着你可以专注于实现业务逻辑，而由 Goa 负责所有 HTTP 传输细节。