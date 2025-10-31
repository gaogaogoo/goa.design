---
title: "代码化的架构图"
linkTitle: "架构图"
weight: 3
description: >
  学习如何使用 Model（一个用于 C4 建模的开源项目）创建和维护架构图，以及通过 model 插件与 Goa 集成。
---

现代软件架构，尤其是围绕微服务构建的架构，需要
清晰且可维护的文档。虽然服务契约和 API 文档
至关重要，但理解服务如何交互以及如何融入更大的系统
往往很有挑战性。这就是架构图变得不可或缺的地方。

## 理解 Model

[Model](https://github.com/goadesign/model) 是一个开源项目，将
代码的力量带到架构文档中。它实现了 C4 模型方法，
通过不同的抽象级别提供描述和沟通软件架构的
分层方式。

通过在代码中定义架构图，Model 使您能够：
- 对架构文档进行版本控制
- 与实现一起维护图表
- 自动化图表更新
- 确保文档的一致性

要开始使用 Model，请安装所需的命令行工具：

```bash
# 安装图表编辑器和查看器
go install goa.design/model/cmd/mdl@latest
```

### 创建您的第一个图表

创建一个简单的系统图，展示用户与使用数据库的服务交互。以下示例演示 Model 设计语言的关键概念：

```go
package design

import . "goa.design/model/dsl"

var _ = Design("Getting Started", "This is a model of my software system.", func() {
    // 定义主要软件系统 - 这代表您的整个应用程序
    var System = SoftwareSystem("Software System", "My software system.", func() {
        // 在系统内定义一个数据库容器
        Database = Container("Database", "Stores information", "MySQL", func() {
            Tag("Database")  // 标签有助于样式和筛选
        })
        
        // 定义一个使用数据库的服务容器
        Container("Service", "My service", "Go and Goa", func() {
            Uses(Database, "Reads and writes data")
        })
    })

    // 定义系统的一个外部用户
    Person("User", "A user of my software system.", func() {
        Uses(System, "Uses")  // 创建与系统的关系
        Tag("person")         // 用于样式的标签
    })

    // 创建一个视图以可视化架构
    Views(func() {
        SystemContextView(System, "System", "System Context diagram.", func() {
            AddAll()                    // 包含所有元素
            AutoLayout(RankLeftRight)   // 自动排列元素
        })
    })
})
```

此示例介绍几个关键概念：
1. `Design` 函数定义架构范围
2. `SoftwareSystem` 代表应用程序
3. `Container` 定义主要组件（服务、数据库等）
4. `Person` 代表用户或角色
5. `Uses` 创建元素之间的关系
6. `Views` 定义不同的架构可视化方式

### 理解 C4 视图

Model 实现 C4 模型描述软件架构的分层方法。如 [C4 模型](https://c4model.com/) 所定义，每个视图类型都有特定用途和受众：

```go
Views(func() {
    // 系统景观：显示企业景观中的所有系统和人员
    SystemLandscapeView("landscape", "Overview", func() {
        AddAll()
        AutoLayout(RankTopBottom)
    })

    // 系统上下文：专注于一个系统及其直接关系
    SystemContextView(System, "context", func() {
        AddAll()
        AutoLayout(RankLeftRight)
    })

    // 容器：显示高级技术构建块
    ContainerView(System, "containers", func() {
        AddAll()
        AutoLayout(RankTopBottom)
    })

    // 组件：详述特定容器的内部
    ComponentView("System/Container", "components", func() {
        AddAll()
        AutoLayout(RankLeftRight)
    })
})
```

让我们详细查看每种视图类型：

#### 系统景观视图
此视图展示软件系统景观的全貌。它帮助利益相关者理解系统如何融入整体企业 IT 环境。

#### 系统上下文视图
此图在环境中展示软件系统，重点关注使用它的人员以及与之交互的其他系统。这是面向技术和非技术受众记录和沟通上下文的极佳起点。

#### 容器视图
如 [C4 模型容器图指南](https://c4model.com/diagrams/container) 所述，容器视图聚焦软件系统以显示高级技术构建块。此处的“容器”代表可独立运行/部署的单元，用于执行代码或存储数据，例如：

- 服务端 Web 应用程序
- 单页应用程序
- 桌面应用程序
- 移动应用
- 数据库模式
- 文件系统

此视图帮助开发者和运维人员理解：
- 软件架构的高级结构
- 职责如何分配
- 主要技术选择
- 容器之间的通信方式

注意此图有意省略部署细节（集群、负载均衡器和复制），因这些通常在环境中各异。

#### 组件视图
此视图聚焦单个容器，显示其组件及其交互。对代码库中的开发者理解容器内部结构很有用。

## 使用图表

Model 通过 `mdl` 命令提供交互式编辑器，便于细化和导出图表。启动编辑器：

```bash
# 使用默认输出目录启动 (./gen)
mdl serve goa.design/examples/model/design

# 或指定自定义输出目录
mdl serve goa.design/examples/model/design -dir diagrams
```

将在 http://localhost:8080 启动 Web 界面，可：
- 拖动元素排列
- 调整关系路径
- 实时预览更改
- 将图表导出为 SVG

{{< figure src="/img/docs/model-editor.png" alt="Model editor interface showing interactive diagram editing" class="full-width-image" >}}

### 交互式编辑

编辑器提供多种操纵图表的方式：

1. 元素定位：
   - 拖动元素定位
   - 使用方向键微调
   - 按住 SHIFT + 方向键可更大移动

2. 关系管理：
   - ALT + 点击添加关系顶点
   - 使用 BACKSPACE 或 DELETE 选择并删除顶点
   - 拖动顶点调整关系路径

3. 选择工具：
   - 点击选择单个元素
   - SHIFT + 点击或拖动选择多个元素
   - CTRL + A 全选
   - ESC 清除选择

### 键盘快捷键参考

以下快捷键有助于高效使用编辑器：

| 类别 | 快捷键 | 效果 |
|----------|----------|--------|
| 帮助 | ?, SHIFT + F1 | 显示键盘快捷键 |
| 文件 | CTRL + S | 保存 SVG |
| 历史 | CTRL + Z | 撤销 |
| 历史 | CTRL + SHIFT + Z, CTRL + Y | 重做 |
| 缩放 | CTRL + =, CTRL + 滚轮 | 放大 |
| 缩放 | CTRL + -, CTRL + 滚轮 | 缩小 |
| 缩放 | CTRL + 9 | 适应窗口 |
| 缩放 | CTRL + 0 | 100% |
| 选择 | CTRL + A | 全选 |
| 选择 | ESC | 取消选择 |
| 移动 | 方向键 | 移动选择 |
| 移动 | SHIFT + 方向键 | 更快移动选择 |

### 图表样式

Model 允许通过样式自定义图表外观：

```go
Views(func() {
    // 定义视图
    SystemContextView(System, "context", func() {
        AddAll()
        AutoLayout(RankTopBottom)
    })
    
    // 应用自定义样式
    Styles(func() {
        // 标记为 "system" 的元素的样式
        ElementStyle("system", func() {
            Background("#1168bd")  // 蓝色背景
            Color("#ffffff")       // 白色文本
        })
        
        // 标记为 "person" 的元素的样式
        ElementStyle("person", func() {
            Shape(ShapePerson)     // 使用人员图标
            Background("#08427b")  // 深蓝色背景
            Color("#ffffff")       // 白色文本
        })
    })
})
```

## 与 Structurizr 集成

虽然 `mdl` 非常适合本地开发和 SVG 导出，Model 还与 [Structurizr](https://structurizr.com/) 服务集成以提供高级可视化与共享。`stz` 管理该集成：

```bash
# 安装 stz
go install goa.design/model/cmd/stz@latest

# 生成 Structurizr workspace 文件
stz gen goa.design/examples/model/design

# 上传到 Structurizr workspace
stz put workspace.json -id ID -key KEY -secret SECRET

# 从 Structurizr 下载
stz get -id ID -key KEY -secret SECRET -out workspace.json
```

在以下情况下考虑使用 Structurizr：
- 与更广泛受众共享图表
- 协作编辑
- 替代可视化选项
- 与现有 Structurizr 工作流集成

## Goa Model 插件

Model 为 Goa 提供插件，确保架构图与服务定义同步。在 Goa 设计中导入以启用：

```go
import (
    . "goa.design/goa/v3/dsl"
    . "goa.design/plugins/v3/model/dsl"
)

var _ = API("calc", func() {
    // 链接到 Model 设计包
    Model("goa.design/model/examples/basic/model", "calc")
    
    // 配置容器命名（可选）
    ModelContainerFormat("%s Service")
    
    // 排除特定容器不验证
    ModelExcludedTags("database", "external", "thirdparty")
    
    // 启用完整验证（可选）
    ModelComplete()
})
```

插件提供：
1. 验证每个 Goa 服务有对应容器
2. 确保服务与容器的命名一致
3. 支持排除特定容器（如数据库）不验证
4. 可选验证所有容器有匹配服务

也可在服务级别配置验证：

```go
var _ = Service("calculator", func() {
    // 覆盖容器名
    ModelContainer("Calculator Service")
    
    // 或禁用此服务验证
    // ModelNone()
})
```

## 最佳实践

使用 Model 和架构图时：

1. 版本控制
   将架构图纳入源码控制，并在代码审查中审查。这有助于保持准确性并保留架构决策历史。

2. 图表组织
   将复杂架构拆分为聚焦视图。使用一致的命名并为所有元素编写清晰描述。

3. Goa 集成
   使用 Goa 插件时，保持服务定义与架构图一致。使用相同术语以避免混淆。

4. 文档
   在描述中包含上下文与理由。记录重要架构决策，保持图表聚焦其目的。

## 延伸阅读

- [Model 项目文档](https://github.com/goadesign/model)
- [C4 模型](https://c4model.com/)
- [Structurizr](https://structurizr.com/)
