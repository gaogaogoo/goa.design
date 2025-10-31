---
title: "数据建模"
linkTitle: "数据建模"
weight: 1
description: >
  使用 Goa 全面的类型系统定义服务的数据结构。通过校验规则和约束在契合领域模型的同时确保数据完整性。
---

Goa 提供了强大的类型系统，帮助你以精确、清晰的方式建模你的领域。从简单的原始类型到复杂的嵌套结构，DSL 都能以自然的方式表达数据关系、约束和校验规则。

## 基础类型

Goa 类型系统的基础由原始类型和基础类型定义构成。这些积木块可以帮助你创建简洁而富有表现力的数据结构。

### 原始类型
Goa 提供了丰富的内置原始类型，作为所有数据建模的基石：

```go
Boolean  // JSON 布尔
Int      // 有符号整数
Int32    // 有符号 32 位整数 
Int64    // 有符号 64 位整数
UInt     // 无符号整数
UInt32   // 无符号 32 位整数
UInt64   // 无符号 64 位整数
Float32  // 32 位浮点数
Float64  // 64 位浮点数
String   // JSON 字符串
Bytes    // 二进制数据
Any      // 任意 JSON 值
```

### 类型定义
Type DSL 函数是定义结构化数据类型的主要方式。它支持属性、校验与文档说明：

```go
var Person = Type("Person", func() {
    Description("A person")
    
    // 基础属性
    Attribute("name", String)
    
    // 带校验的属性
    Attribute("age", Int32, func() {
        Minimum(0)
        Maximum(120)
    })
    
    // 必填字段
    Required("name", "age")
})
```

## 复杂类型

在建模真实世界的领域时，你经常需要更复杂的数据结构。Goa 对集合与嵌套类型提供了完善的支持。

### 数组（Array）
数组允许你定义任意类型的有序集合，并可选地添加校验规则：

```go
var Names = ArrayOf(String, func() {
    // 校验数组元素
    MinLength(1)
    MaxLength(10)
})

var Team = Type("Team", func() {
    Attribute("members", ArrayOf(Person))
})
```

### 映射（Map）
映射提供了键值关联，并同时对键和值提供类型安全与校验：

```go
var Config = MapOf(String, Int32, func() {
    // 键的校验
    Key(func() {
        Pattern("^[a-z]+$")
    })
    // 值的校验 
    Elem(func() {
        Minimum(0)
    })
})
```

## 类型组合

Goa 支持多种类型组合模式，以实现代码复用与关注点分离。

### Reference（引用）
使用 Reference 可从另一个类型为当前类型的属性设置默认属性。当当前类型中的属性与被引用类型的属性同名时，会继承其属性定义。可以指定多个引用，按照出现顺序查找属性：

```go
var Employee = Type("Employee", func() {
    // 复用 Person 的属性定义
    Reference(Person)
    Attribute("name") // 无需再次定义 name 属性
    Attribute("age")  // 无需再次定义 age 属性

    // 新增属性
    Attribute("employeeID", String, func() {
        Format(FormatUUID)
    })
})
```

### Extend（扩展）

`Extend` 基于现有类型创建新类型，适合建模层级关系。与 `Reference` 不同，`Extend` 会自动继承基础类型的所有属性。

```go
var Manager = Type("Manager", func() {
    // 扩展基础类型
    Extend(Employee)
    
    // 经理特有字段
    Attribute("reports", ArrayOf(Employee))
})
```

## 校验规则

Goa 提供了完善的校验能力以确保数据完整性并执行业务规则：
下面是 Goa 中可用的关键校验规则：

### 字符串校验
- `Pattern(regex)` - 正则匹配
- `MinLength(n)` - 最小字符串长度
- `MaxLength(n)` - 最大字符串长度
- `Format(format)` - 预定义格式校验（email、URI 等）

### 数值校验  
- `Minimum(n)` - 最小值（含）
- `Maximum(n)` - 最大值（含）
- `ExclusiveMinimum(n)` - 最小值（不含） 
- `ExclusiveMaximum(n)` - 最大值（不含）

### 数组与映射校验
- `MinLength(n)` - 最小元素个数
- `MaxLength(n)` - 最大元素个数

### 对象校验
- `Required("field1", "field2")` - 必填字段

### 通用校验
- `Enum(value1, value2)` - 枚举约束

此外，数组与映射的元素也可使用与属性相同的规则进行校验。

这些校验规则可以组合，构建完善的校验逻辑：

```go
var UserProfile = Type("UserProfile", func() {
    Attribute("username", String, func() {
        Pattern("^[a-z0-9]+$") // 正则
        MinLength(3)           // 最小长度
        MaxLength(50)          // 最大长度
    })
    
    Attribute("email", String, func() {
        Format(FormatEmail)    // 内置格式
    })
    
    Attribute("age", Int32, func() {
        Minimum(18)            // 最小值
        ExclusiveMaximum(150)  // 最大值（不含）
    })
    
    Attribute("tags", ArrayOf(String, func() { Enum("tag1", "tag2", "tag3") }), func() {
                              // 数组元素的枚举值
        MinLength(1)          // 最小数组长度
        MaxLength(10)         // 最大数组长度
    })
    
    Attribute("settings", MapOf(String, String), func() {
        MaxLength(20)         // 最大 map 长度
    })

    Required("username", "email", "age") // 必填字段
})
```

## 自定义类型

创建可复用的自定义类型，以封装特定领域的格式与校验规则：

```go
// 自定义格式
var UUID = Type("UUID", String, func() {
    Format(FormatUUID)
    Description("RFC 4122 UUID")
})

// 使用自定义类型
var Resource = Type("Resource", func() {
    Attribute("id", UUID)
    Attribute("name", String)
})
```

参见 [Type DSL](https://pkg.go.dev/goa.design/goa/v3/dsl#Type) 获取更多细节。

## 内置格式

Goa 提供了覆盖常见数据模式的预定义格式，这些格式自带校验并拥有明确的语义：

- `FormatDate` - RFC3339 日期
- `FormatDateTime` - RFC3339 日期时间
- `FormatUUID` - RFC4122 UUID
- `FormatEmail` - RFC5322 邮箱地址
- `FormatHostname` - RFC1035 主机名
- `FormatIPv4` - RFC2373 IPv4 地址
- `FormatIPv6` - RFC2373 IPv6 地址
- `FormatIP` - RFC2373 IPv4 或 IPv6 地址
- `FormatURI` - RFC3986 URI
- `FormatMAC` - IEEE 802 MAC-48、EUI-48 或 EUI-64 MAC 地址
- `FormatCIDR` - RFC4632 与 RFC4291 CIDR 表示的 IP 地址
- `FormatRegexp` - RE2 接受的正则表达式语法
- `FormatJSON` - JSON 文本
- `FormatRFC1123` - RFC1123 日期时间

## Attribute 与 Field DSL 的区别

Goa 提供了两种等价的方式来定义类型属性：`Attribute` 与 `Field`。主要差异在于 `Field` 多了一个用于 gRPC 消息字段编号的标签参数。

### Attribute DSL
适用于不需要 gRPC 支持或不会用于 gRPC 消息的类型：

```go
var Person = Type("Person", func() {
    Attribute("name", String)
    Attribute("age", Int32)
})
```

### Field DSL
适用于会在 gRPC 消息中使用的类型。第一个参数为字段编号标签：

```go
var Person = Type("Person", func() {
    Field(1, "name", String)
    Field(2, "age", Int32)
})
```

两种 DSL 在校验、文档和示例方面能力一致。根据是否需要 gRPC 支持进行选择即可。

## 示例（Examples）

Example DSL 允许你为类型和属性提供示例值。这些示例会出现在生成的文档中，帮助 API 使用者理解期望的数据格式。

### 为属性添加示例

```go
var User = Type("User", func() {
    Attribute("name", String, func() {
        Example("John Doe")
    })
    
    Attribute("age", Int32, func() {
        Example(25)
        Minimum(0)
        Maximum(120)
    })
    
    // 多个示例
    Attribute("email", String, func() {
        Example("work", "john@work.com")
        Example("personal", "john@gmail.com")
        Format(FormatEmail)
    })
})
```

### 复杂类型示例

对于复杂类型，你可以提供完整示例，展示多个属性如何协同：

```go
var Address = Type("Address", func() {
    Description("Mailing address")
    
    Attribute("street", String)
    Attribute("city", String)
    Attribute("state", String)
    Attribute("postal_code", String)
    
    Required("street", "city", "state", "postal_code")
    
    Example("Home Address", func() {
        Description("Example of a residential address")
        Value(Val{
            "street": "123 Main St",
            "city": "Boston",
            "state": "MA",
            "postal_code": "02101",
        })
    })
    
    Example("Business Address", func() {
        Description("Example of a business address")
        Value(Val{
            "street": "1 Enterprise Ave",
            "city": "San Francisco",
            "state": "CA",
            "postal_code": "94105",
        })
    })
})
```

### 包含数组与映射的示例

```go
var Order = Type("Order", func() {
    Attribute("id", Int64)
    Attribute("items", ArrayOf(String))
    Attribute("metadata", MapOf(String, String))
    
    Example("Simple Order", func() {
        Description("Basic order with a few items")
        Value(Val{
            "id": 1001,
            "items": []string{"SKU123", "SKU456"},
            "metadata": map[string]string{
                "priority": "high",
                "shipping": "express",
            },
        })
    })
})
```

### 在文档中使用示例

示例会自动包含到生成的 OpenAPI 文档中，便于 API 使用者理解预期的数据格式。它们也可用于测试，以验证 API 能否正确处理典型用例。

最佳实践：
- 提供真实、有意义的示例
- 对复杂类型给出多个示例
- 添加描述以说明上下文
- 覆盖边界情况与不同变体
- 使用示例展示校验规则

## 最佳实践

在设计数据模型时，遵循以下指引有助于创建可维护、健壮的服务：

{{< alert title="设计指引" color="primary" >}}
类型组织
- 将相关类型归组管理
- 使用有意义的字段名与描述
- 遵循一致的命名规范
- 保持类型内聚与专注

校验策略
- 为每个字段添加合适的约束
- 明确指定必填字段
- 对标准格式使用格式校验器
- 根据领域添加特定校验规则

类型组合
- 将复杂类型拆分为更小的组件
- 使用扩展实现特化
- 创建可复用的基础类型
- 维护清晰的类型层级
{{< /alert >}}
