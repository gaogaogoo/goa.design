---
title: "代码生成"
linkTitle: "代码生成"
weight: 2
description: "了解 Goa 如何从你的设计生成代码，包括命令行用法、生成流程与自定义选项。"
menu:
  main:
    parent: "概念"
    weight: 2
---

Goa 的代码生成系统会将你的设计转化为可直接用于生产的代码。不同于仅生成脚手架，Goa 会生成完整且可运行的服务实现，遵循最佳实践并在整个 API 中保持一致性。

## 代码生成的益处

- **一致性**：生成代码遵循一致的模式与最佳实践
- **类型安全**：在实现中保持强类型
- **验证**：基于设计规则自动验证请求
- **文档**：生成 OpenAPI 规范与文档
- **传输支持**：单一设计支持多种传输协议
- **可维护性**：设计更改会自动反映到实现中

## 生成概览

Goa 的代码生成会从你的设计文件出发，产出完整、可运行的服务实现。

## 命令行工具

### 安装

使用以下命令安装 Goa 的命令行工具：

```bash
go install goa.design/goa/v3/cmd/goa@latest
```

### 关键命令

Goa 提供两个命令帮助你生成与脚手架服务。所有命令都需要 Go 包导入路径，而非文件系统路径：

```bash
# ✅ 正确：使用 Go 包导入路径
goa gen goa.design/examples/calc/design

# ❌ 错误：使用文件系统路径
goa gen ./design
```

#### 生成代码（`goa gen`）

```bash
goa gen <design-package-import-path> [-o <output-dir>]
```

主要的代码生成命令，功能包括：
- 处理你的设计包并生成实现代码
- 每次运行都会从零重建 `gen/` 目录
- 每次设计变更后都应运行
- 通过 `-o` 指定输出位置（默认 `./gen`）

#### 创建示例（`goa example`）

```bash
goa example <design-package-import-path> [-o <output-dir>]
```

脚手架命令，作用包括：
- 仅一次性创建服务的示例实现
- 生成带有示例逻辑的处理器桩代码
- 仅在新项目开始时运行一次
- 设计变更后不应重复运行
- 即便重复运行也不会覆盖已存在的自定义实现

#### 显示版本（`goa version`）

```bash
goa version
```

显示已安装的 Goa 版本。

## 生成流程

当你运行 Goa 的代码生成命令时，Goa 会遵循系统化流程将设计转化为可工作的代码：

### 设计加载

生成流程包含以下阶段：

1. **引导（Bootstrap）**：
   先创建一个临时的 `main.go` 文件，导入你的设计包与 Goa 包。该临时文件随后被编译并作为单独进程执行，以引导代码生成。

2. **DSL 执行**：
   设计包的初始化函数先执行，随后执行 DSL 函数以在内存中构建表达式对象。这些表达式共同组成一个完整模型，表示整个 API 的设计。

3. **验证**：
   在验证阶段，Goa 会对表达式树进行全面检查，确保其完整且结构良好。会验证表达式之间的必要关系是否定义完备，并确保设计遵循所有规则与约束。此步骤有助于在生成代码前及早发现潜在问题。

4. **代码生成**：
   验证完成后，Goa 将有效的表达式传递给代码生成器。生成器会使用表达式数据渲染模板，产出实际代码文件。生成的文件会写入项目中的 `gen/` 目录，并按服务与传输层进行组织。

## 自定义生成

### 使用元数据（Metadata）

`Meta` 函数允许你自定义代码生成行为。以下是影响生成的关键元数据标签：

#### 类型生成控制

`"type:generate:force"` 标签可强制生成某个类型，即使它未被任何方法直接引用。取值为需要生成该类型的服务名。

```go
var MyType = Type("MyType", func() {
    // 即使未使用也强制生成类型
    Meta("type:generate:force", "service1", "service2")
    
    Attribute("name", String)
})
```

#### 包与结构体自定义

`"struct:pkg:path"` 标签可指定类型的包与路径。取值为相对于 `gen` 包的包路径。

```go
var MyType = Type("MyType", func() {
    // 在自定义包中生成类型
    Meta("struct:pkg:path", "types")
    
    Attribute("ssn", String, func() {
        // 覆盖字段名
        Meta("struct:field:name", "SSN")
        // 自定义 struct 标签
        Meta("struct:tag:json", "ssn,omitempty")
    })
})
```

#### Protocol Buffer 自定义

`"struct:name:proto"` 标签可指定类型的协议缓冲消息名。取值包含包路径、消息名以及协议缓冲类型的导入路径。

```go
var Timestamp = Type("Timestamp", func() {
    // 覆盖 protobuf 的消息名
    Meta("struct:name:proto", "MyProtoType")
    
    Field(1, "created_at", String, func() {
        // 使用 Google 的时间戳类型
        Meta("struct:field:proto", 
            "google.protobuf.Timestamp",
            "google/protobuf/timestamp.proto",
            "Timestamp",
            "google.golang.org/protobuf/types/known/timestamppb")
    })
})
```

#### OpenAPI 生成

`"openapi:generate"` 标签可用于禁用某服务的 OpenAPI 生成。取值为需要生成类型的服务名。

`"openapi:operationId"` 标签可为方法指定操作 ID。取值为服务名与方法名。

`"openapi:tag"` 标签可为服务指定 OpenAPI 标签。取值为服务名与标签名。

```go
var _ = Service("MyService", func() {
    // 禁用该服务的 OpenAPI 生成
    Meta("openapi:generate", "false")
    
    Method("MyMethod", func() {
        // 自定义操作 ID
        Meta("openapi:operationId", "{service}.{method}")
        // 添加 OpenAPI 标签
        Meta("openapi:tag:Backend", "后端 API")
    })
})
```

常见元数据用途：
- 控制哪些类型需要生成
- 自定义生成的结构体字段与标签
- 覆盖包位置
- 配置 Protocol Buffer 的生成
- 自定义 API 文档

{{< alert title="元数据提示" color="primary" >}}
- 当类型仅被间接引用时使用 `type:generate:force`
- 在相关类型间保持包路径（`struct:pkg:path`）的一致性
- 自定义 OpenAPI 生成时需考虑文档影响
- 谨慎进行字段级自定义以保持一致性
{{< /alert >}}

### 插件系统

Goa 的插件系统允许你扩展与自定义代码生成流程。插件可以在管线的特定阶段进行拦截，帮助你添加功能、修改生成代码或产出全新的输出。

#### 插件能力

插件可通过三种主要方式与 Goa 交互：

1. **新增 DSL**  
   插件可提供与 Goa 核心 DSL 并行工作的设计语言构造。例如，
   [CORS 插件](https://github.com/goadesign/plugins/tree/master/cors) 添加了用于定义跨域策略的 DSL：

```go
var _ = Service("calc", func() {
    Description("计算器服务")
    
    // CORS 插件新增的 DSL
    cors.Origin("/.*localhost.*/", func() {
        cors.Headers("X-Shared-Secret")
        cors.Methods("GET", "POST")
    })
})
```

2. **修改生成代码**  
   插件可以检查并修改 Goa 生成的文件，或新增文件：插件的 `Generate` 函数会在设计评估完成后的代码生成阶段被 Goa 调用。它接收：
   
   - `genpkg`：生成代码将被放置的 Go 包路径
   - `roots`：已评估的设计根节点，包含全部设计数据
   - `files`：到目前为止 Goa 已生成的文件数组
   
   该函数允许插件检查与修改任何生成文件、向输出新增文件、从生成中移除文件，以及基于设计转换代码。其灵活性使插件可对最终生成的代码库进行全面控制。
   
#### 常见用例

插件通常用于：
- 为特定协议或传输方式添加支持（如 CORS）
- 生成额外的文档格式
- 实现自定义校验规则
- 添加跨领域关注点（日志、指标等）
- 生成配套的配置文件

#### 插件快速上手

使用现有插件：
1. 导入插件包
2. 在设计中使用其 DSL
3. 像往常一样运行 `goa gen`——插件会自动集成

```go
import (
    . "goa.design/goa/v3/dsl"
    cors "goa.design/plugins/v3/cors/dsl"
)
```

{{< alert title="了解更多插件信息" color="primary" >}}
以上仅是 Goa 插件系统的概览。关于以下内容的详细信息：
- 如何创建自定义插件
- 可用的插件钩子
- 插件最佳实践
- 示例实现

请参阅专门的 [插件](../../6-advanced/1-plugins) 章节。
{{< /alert >}}
