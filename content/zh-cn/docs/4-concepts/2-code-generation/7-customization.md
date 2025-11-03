---
title: "自定义"
linkTitle: "自定义"
weight: 7
description: "学习如何使用元数据自定义与扩展 Goa 的代码生成。"
---

## 概览

元数据（Metadata）允许你通过简单标签控制与自定义代码生成。使用 `Meta` 函数为设计元素添加元数据。

### 基础类型生成控制

默认情况下，Goa 仅生成被服务方法使用的类型。如果你在设计中定义了类型但未在任何方法的参数或结果中引用，Goa 将跳过生成。

`"type:generate:force"` 元数据标签可覆盖上述行为。它接收服务名作为参数，用于指定哪些服务的生成代码应包含该类型；若不提供服务名，则在所有服务中生成：

```go
var MyType = Type("MyType", func() {
    // 即使未被使用，也在 service1 与 service2 中强制生成类型
    Meta("type:generate:force", "service1", "service2")
    Attribute("name", String)
})

var OtherType = Type("OtherType", func() {
    // 在所有服务中强制生成类型
    Meta("type:generate:force")
    Attribute("id", String)
})
```

### 包组织

你可以通过包相关元数据控制类型的生成位置。默认情况下，类型会生成在各自的服务包中，但你也可以将其生成到共享包中。当多个服务需要处理相同的 Go 结构体（例如共享业务逻辑或数据访问代码）时，这尤其有用。通过在共享包中生成类型，可以避免在各服务间来回转换重复的类型定义：

```go
var CommonType = Type("CommonType", func() {
    // 在共享的 types 包中生成
    Meta("struct:pkg:path", "types")
    
    Attribute("id", String)
    Attribute("createdAt", String)
})
```

生成结构示例：
```
project/
├── gen/
│   └── types/              # 共享的 types 包
│       └── common_type.go # 由 CommonType 生成
```

{{< alert title="重要说明" color="primary" >}}
- 所有关联类型必须使用相同的包路径
- 相互引用的类型必须位于同一包
- `types` 包通常用于共享类型
- 使用共享包可消除在服务共享代码时复制或转换重复类型定义的需求
{{< /alert >}}

### 字段自定义

默认情况下，Goa 会将属性名转换为驼峰命名来生成字段名。例如，属性名 "user_id" 会在生成结构体中变为 "UserID"。

Goa 同时提供了从设计类型到 Go 类型的默认映射：
- `String` → `string`
- `Int` → `int`
- `Int32` → `int32`
- `Int64` → `int64`
- `Float32` → `float32`
- `Float64` → `float64`
- `Boolean` → `bool`
- `Bytes` → `[]byte`
- `Any` → `any`

你可以使用多个元数据标签自定义个别字段：

- `struct:field:name`：覆盖生成的字段名
- `struct:field:type`：覆盖生成的字段类型
- `struct:tag:*`：添加自定义 struct 标签

综合示例：

```go
var Message = Type("Message", func() {
    Meta("struct:pkg:path", "types")
    
    Attribute("id", String, func() {
        // 覆盖字段名
        Meta("struct:field:name", "ID")
        // 添加自定义的 MessagePack 标签
        Meta("struct:tag:msgpack", "id,omitempty")
        // 使用自定义类型覆盖字段类型
        Meta("struct:field:type", "bison.ObjectId", "github.com/globalsign/mgo/bson", "bison")
    })
})
```

将生成如下 Go 结构体：

```go
type Message struct {
    ID bison.ObjectId `msgpack:"id,omitempty"`
}
```

{{< alert title="重要限制" color="primary" >}}
使用 `struct:field:type` 时：
- 覆盖后的类型必须支持与原类型一致的编解码行为
- Goa 会基于原始类型定义生成编解码代码
- 若编解码行为不兼容将导致运行时错误
{{< /alert >}}

## Protocol Buffer 自定义

使用 Protocol Buffers 时，可以通过若干元数据键自定义生成的 protobuf 代码：

### 消息类型名

`struct:name:proto` 允许覆盖生成的 protobuf 消息名。默认情况下，Goa 使用设计中的类型名：

```go
var MyType = Type("MyType", func() {
    // 将 protobuf 消息名更改为 "CustomProtoType"
    Meta("struct:name:proto", "CustomProtoType")
    
    Field(1, "name", String)
})
```

### 字段类型

`struct:field:proto` 允许覆盖生成的 protobuf 字段类型。这在使用 protobuf well-known 类型或其他 proto 文件中的类型时尤为有用。它最多接收 4 个参数：

1. protobuf 类型名
2.（可选）proto 文件导入路径
3.（可选）Go 类型名
4.（可选）Go 包导入路径

```go
var MyType = Type("MyType", func() {
    // 简单类型覆盖
    Field(1, "status", Int32, func() {
        // 从默认的 sint32 改为 int32
        Meta("struct:field:proto", "int32")
    })

    // 使用 Google 的 well-known timestamp 类型
    Field(2, "created_at", Timestamp, func() {
        Meta("struct:field:proto", 
            "google.protobuf.Timestamp",           // Proto 类型
            "google/protobuf/timestamp.proto",     // Proto 导入
            "Timestamp",                           // Go 类型
            "google.golang.org/protobuf/types/known/timestamppb") // Go 导入
    })
})
```

将生成如下 protobuf 定义：

```protobuf
import "google/protobuf/timestamp.proto";

message MyType {
    int32 status = 1;
    google.protobuf.Timestamp created_at = 2;
}
```

### 导入路径

`protoc:include` 指定在调用 `protoc` 编译器时使用的导入路径。可在 API 或服务级别进行设置：

```go
var _ = API("calc", func() {
    // 为所有服务设置全局导入路径
    Meta("protoc:include", 
        "/usr/include",
        "/usr/local/include")
})

var _ = Service("calculator", func() {
    // 为该服务设置特定导入路径
    Meta("protoc:include", 
        "/usr/local/include/google/protobuf")
    
    // ... 服务方法 ...
})
```

当设置在 API 定义上时，这些导入路径适用于所有服务；当设置在具体服务上时，路径仅适用于该服务。

{{< alert title="重要说明" color="primary" >}}
- 使用外部 proto 类型时，`struct:field:proto` 必须提供必要的导入信息
- `protoc:include` 中的路径应指向包含 .proto 文件的目录
- 服务级导入路径是对 API 级路径的补充，而非替代
{{< /alert >}}

## OpenAPI 规范控制

### 基础 OpenAPI 设置

控制 OpenAPI 规范的生成与格式化：

```go
var _ = API("MyAPI", func() {
    // 控制是否生成 OpenAPI
    Meta("openapi:generate", "false")
    
    // 设置 JSON 输出格式
    Meta("openapi:json:prefix", "  ")
    Meta("openapi:json:indent", "  ")
})
```

这会影响 OpenAPI JSON 的格式：
```json
{
  "openapi": "3.0.3",
  "info": {
    "title": "MyAPI",
    "version": "1.0"
  }
}
```

### 操作与类型自定义

可通过多个元数据键自定义 OpenAPI 规范中的操作与类型展示：

#### 操作 ID（Operation ID）

`openapi:operationId` 允许自定义操作 ID 的生成。它支持以占位符的形式引用实际值：

- `{service}`：替换为服务名
- `{method}`：替换为方法名
- `{#routeIndex}`：替换为路由索引（仅当方法具有多条路由时）

示例：
```go
var _ = Service("UserService", func() {
    Method("ListUsers", func() {
        // 生成 operationId: "users/list"
        Meta("openapi:operationId", "users/list")  // 静态值
    })
    
    Method("CreateUser", func() {
        // 生成 operationId: "UserService.CreateUser"
        Meta("openapi:operationId", "{service}.{method}")
    })
    
    Method("UpdateUser", func() {
        // 多条路由时生成：
        // - "UserService_UpdateUser_1"（第一条路由）
        // - "UserService_UpdateUser_2"（第二条路由）
        Meta("openapi:operationId", "{service}_{method}_{#routeIndex}")
        
        HTTP(func() {
            PUT("/users/{id}")   // 第一条路由
            PATCH("/users/{id}") // 第二条路由
        })
    })
})
```

#### 操作摘要（Summary）

若未通过元数据提供摘要，Goa 的默认行为为：
1. 若定义了方法描述则使用描述
2. 若未定义描述，则使用 HTTP 动词与路径（例如 "GET /users/{id}"）

`openapi:summary` 允许覆盖默认摘要。摘要出现在每个操作的顶部，应简要说明该操作的目的。

可以使用：
- 静态字符串
- 特殊占位符 `{path}`，会替换为操作的 HTTP 路径

```go
var _ = Service("UserService", func() {
    Method("CreateUser", func() {
        // 使用该描述作为默认摘要
        Description("在系统中创建新用户")
        
        HTTP(func() {
            POST("/users")
        })
    })
    
    Method("UpdateUser", func() {
        // 覆盖默认摘要
        Meta("openapi:summary", "处理对 {path} 的 PUT 请求")
        
        HTTP(func() {
            PUT("/users/{id}")
        })
    })
    
    Method("ListUsers", func() {
        // 无描述或摘要元数据
        // 默认摘要将为："GET /users"
        
        HTTP(func() {
            GET("/users")
        })
    })
})
```

将生成如下 OpenAPI 规范：
```json
{
  "paths": {
    "/users": {
      "post": {
        "summary": "在系统中创建新用户",
        "operationId": "UserService.CreateUser"
      },
      "get": {
        "summary": "GET /users",
        "operationId": "UserService.ListUsers"
      }
    },
    "/users/{id}": {
      "put": {
        "summary": "处理对 /users/{id} 的 PUT 请求",
        "operationId": "UserService.UpdateUser"
      }
    }
  }
}
```

{{< alert title="最佳实践" color="primary" >}}
- 保持摘要简洁但具描述性
- 在相关操作间使用一致措辞
- 可在摘要中包含关键参数或约束
- 当 HTTP 路径具有重要上下文时使用 {path}
{{< /alert >}}

#### 类型名

`openapi:typename` 允许仅在 OpenAPI 规范中覆盖类型名而不影响 Go 类型名：

```go
var User = Type("User", func() {
    // 在 OpenAPI 规范中，该类型名将变为 "CustomUser"
    Meta("openapi:typename", "CustomUser")
    
    Attribute("id", Int)
    Attribute("name", String)
})
```

#### 示例生成

Goa 允许在设计中为类型指定示例。若未指定示例，Goa 默认会生成随机示例。`openapi:example` 可用来禁用示例生成：

```go
var User = Type("User", func() {
    // 指定一个示例（会用于 OpenAPI 规范）
    Example(User{
        ID:   123,
        Name: "John Doe",
    })
    
    Attribute("id", Int)
    Attribute("name", String)
})

var Account = Type("Account", func() {
    // 禁用该类型的示例生成
    Meta("openapi:example", "false")
    
    Attribute("id", Int)
    Attribute("balance", Float64)
})

var _ = API("MyAPI", func() {
    // 为所有类型禁用示例生成
    Meta("openapi:example", "false")
})
```

{{< alert title="注意" color="primary" >}}
- 默认情况下，未显式示例的类型会生成随机示例
- 使用 `Example()` DSL 指定自定义示例
- 使用 `Meta("openapi:example", "false")` 禁止示例生成
- 在 API 级别将 `openapi:example` 设为 false 会影响所有类型
{{< /alert >}}

### 标签与扩展

你可以在服务与方法级为 OpenAPI 规范添加标签与自定义扩展。标签可帮助归类相关操作，扩展允许为 API 规范添加自定义元数据。

#### 服务级标签

应用在服务级别的标签会对该服务的所有方法生效：

```go
var _ = Service("UserService", func() {
    // 为整个服务定义标签
    HTTP(func() {
        // 添加简单标签
        Meta("openapi:tag:Users")
        
        // 添加带描述的标签
        Meta("openapi:tag:Backend:desc", "后端 API 操作")
        
        // 为标签添加文档 URL
        Meta("openapi:tag:Backend:url", "http://example.com/docs")
        Meta("openapi:tag:Backend:url:desc", "API 文档")
    })
    
    // 该服务中的所有方法将继承这些标签
    Method("CreateUser", func() {
        HTTP(func() {
            POST("/users")
        })
    })
    
    Method("ListUsers", func() {
        HTTP(func() {
            GET("/users")
        })
    })
})
```

#### 方法级标签

也可以为特定方法添加标签，或在方法级覆盖服务级标签：

```go
var _ = Service("UserService", func() {
    Method("AdminOperation", func() {
        HTTP(func() {
            // 仅为该方法添加额外标签
            Meta("openapi:tag:Admin")
            Meta("openapi:tag:Admin:desc", "管理操作")
            
            POST("/admin/users")
        })
    })
})
```

#### 自定义扩展

可以在多个层级添加扩展，以自定义 OpenAPI 规范的不同部分：

```go
var _ = API("MyAPI", func() {
    // API 级扩展
    Meta("openapi:extension:x-api-version", `"1.0"`)
})

var _ = Service("UserService", func() {
    // 服务级扩展
    HTTP(func() {
        Meta("openapi:extension:x-service-class", `"premium"`)
    })
    
    Method("CreateUser", func() {
        // 方法级扩展
        HTTP(func() {
            Meta("openapi:extension:x-rate-limit", `{"rate": 100, "burst": 200}`)
            POST("/users")
        })
    })
})
```

将生成如下 OpenAPI 规范：
```json
{
  "info": {
    "x-api-version": "1.0"
  },
  "tags": [
    {
      "name": "Users",
      "description": "用户管理操作"
    },
    {
      "name": "Backend",
      "description": "后端 API 操作",
      "externalDocs": {
        "description": "API 文档",
        "url": "http://example.com/docs"
      }
    },
    {
      "name": "Admin",
      "description": "管理操作"
    }
  ],
  "paths": {
    "/users": {
      "post": {
        "tags": ["Users", "Backend"],
        "x-service-class": "premium",
        "x-rate-limit": {
          "rate": 100,
          "burst": 200
        }
      },
      "get": {
        "tags": ["Users", "Backend"]
      }
    },
    "/admin/users": {
      "post": {
        "tags": ["Users", "Backend", "Admin"],
        "x-service-class": "premium"
      }
    }
  }
}
```

{{< alert title="重要说明" color="primary" >}}
- 服务级标签会应用到该服务的所有方法
- 方法级标签会在服务级标签基础上叠加
- 扩展可在 API、服务与方法级添加
- 扩展的值必须为有效的 JSON 字符串
- 标签有助于在 API 文档中组织与分组相关操作
{{< /alert >}}

### 测试自定义类型

当你处理自定义类型与字段覆盖时，测试其行为是否正确很重要。以下演示如何使用 Clue 的 mock 包有效测试自定义类型实现：

```go
// 引入 Clue 的 mock 包
import (
    "github.com/goadesign/clue/mock"
)

// 示例：带覆盖字段类型的自定义类型
type Message struct {
    ID bison.ObjectId `msgpack:"id,omitempty"`
}

// 使用 Clue 的 mock 包实现的存根
type mockMessageStore struct {
    *mock.Mock // 嵌入 Clue 的 Mock 类型
}

// Store 使用 Clue 的 Next 模式实现模拟
func (m *mockMessageStore) Store(ctx context.Context, msg *Message) error {
    if f := m.Next("Store"); f != nil {
        return f.(func(context.Context, *Message) error)(ctx, msg)
    }
    return errors.New("unexpected call to Store")
}

func TestMessageStore(t *testing.T) {
    // 使用 Clue 的 mock 包创建模拟存储
    store := &mockMessageStore{mock.New()}
    
    tests := []struct {
        name    string
        msg     *Message
        setup   func(*mockMessageStore)
        wantErr bool
    }{
        {
            name: "successful store",
            msg: &Message{
                ID: bison.NewObjectId(),
            },
            setup: func(m *mockMessageStore) {
                m.Set("Store", func(ctx context.Context, msg *Message) error {
                    return nil
                })
            },
            wantErr: false,
        },
        {
            name: "store error",
            msg: &Message{
                ID: bison.NewObjectId(),
            },
            setup: func(m *mockMessageStore) {
                m.Set("Store", func(ctx context.Context, msg *Message) error {
                    return fmt.Errorf("storage error")
                })
            },
            wantErr: true,
        },
        {
            name: "invalid message",
            msg:  &Message{}, // 空 ID
            setup: func(m *mockMessageStore) {
                m.Set("Store", func(ctx context.Context, msg *Message) error {
                    if msg.ID.IsZero() {
                        return fmt.Errorf("invalid message ID")
                    }
                    return nil
                })
            },
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 为每个用例创建新的 mock
            mock := &mockMessageStore{mock.New()}
            if tt.setup != nil {
                tt.setup(mock)
            }
            
            // 执行测试
            err := mock.Store(context.Background(), tt.msg)
            
            // 校验错误行为
            if (err != nil) != tt.wantErr {
                t.Errorf("Store() error = %v, wantErr %v", err, tt.wantErr)
            }
            
            // 校验所有期望的调用是否已发生
            if mock.HasMore() {
                t.Error("not all expected operations were performed")
            }
        })
    }
}
```

该示例展示了 Clue 的 mock 包的多项关键特性：

1. **类型安全的 Mock**：通过嵌入 Clue 的 `Mock` 类型，模拟实现（`mockMessageStore`）提供类型安全的接口
2. **自定义类型处理**：使用正确的字段类型测试自定义类型的校验与行为
3. **顺序控制**：需要顺序操作时可使用 `Add`
4. **默认行为**：使用 `Set` 在各测试用例中提供一致的响应
5. **全面验证**：`HasMore` 方法确保所有期望的操作已执行

这些测试用例覆盖了关键场景：验证成功的存储操作按预期完成；当存储失败时正确处理错误；校验自定义类型字段满足约束，确保数据完整性；最后通过校验所有预期的 mock 调用已被执行来确认清理得当。