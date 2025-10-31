---
title: "生成流程"
linkTitle: "生成流程"
weight: 2
description: "了解 Goa 如何将你的设计转换为代码，包括生成流水线、表达式求值与输出结构。"
---

## 生成流水线（Generation Pipeline）

运行 `goa gen` 时，Goa 会按系统化流程将你的设计转换为可运行的代码：

### 1. 引导阶段（Bootstrap Phase）

Goa 首先创建并运行一个临时程序：
在该阶段，Goa 会创建一个临时的 `main.go` 程序，负责：

1. 引入用于代码生成与求值的 Goa 包
2. 引入你的设计包
3. 运行 DSL 以评估你的设计
4. 触发代码生成流程

该临时程序是将设计转化为代码的入口。在生成过程中会自动创建与删除，你无需手动管理。

### 2. 设计评估（Design Evaluation）

此阶段 Goa 会加载并评估你的设计包：

1. 执行 DSL 函数以创建表示 API 设计的表达式对象
2. 将这些表达式组合为完整的 API 结构与行为模型
3. 系统分析并建立各表达式之间的关系
4. 严格校验所有设计规则与约束以确保正确性

评估阶段至关重要，它将你的声明式设计转换为可用于代码生成的结构化模型。

### 3. 代码生成（Code Generation）

表达式通过校验后会传递给 Goa 的代码生成器。生成器以表达式为输入渲染各类代码模板，生成面向 HTTP 与 gRPC 的传输层代码、创建所有必要的支撑文件，并将完整输出写入 `gen/` 目录。该步骤会生成运行服务所需的全部代码，同时确保代码库的一致性。

## 生成结构（Generated Structure）

典型的生成项目结构如下：

```
myservice/
├── cmd/             # 生成的示例命令
│   └── calc/
│       ├── grpc.go
│       └── http.go
├── design/          # 你的设计文件
│   └── design.go
├── gen/            # 生成代码
│   ├── calc/       # 服务相关代码
│   │   ├── client/
│   │   ├── endpoints/
│   │   └── service/
│   └── http/       # 传输层
│       ├── client/
│       └── server/
└── myservice.go    # 生成的服务实现桩
```

更多关于生成代码的细节，参见
[生成的代码](/4-concepts/2-code-generation/3-generated-code) 部分。