---
title: "基础 gRPC 服务"
linkTitle: "基础 gRPC 服务"
weight: 2
description: "使用 Goa 的设计优先方法构建完整的 gRPC 服务，涵盖服务设计、实现、protobuf 处理以及演唱会管理系统的部署。"
---

通过本系列教程，学习如何使用 Goa 构建可用于生产的 gRPC 服务。我们将创建一个演唱会管理系统，演示关键的 gRPC 概念，并遵循 Goa 的设计优先方法。

## 教程章节

### 1. [服务设计](./1-designing)
使用 Goa 的 DSL 定义服务：
- 定义服务方法与 RPC
- 创建 Protocol Buffers 消息
- 设置输入校验
- 编写服务行为文档

### 2. [服务实现](./2-implementing)
将设计转化为可运行代码：
- 生成服务脚手架
- 编写业务逻辑
- 增加错误处理
- 搭建 gRPC 服务器

### 3. [运行服务](./3-running)
部署并测试服务：
- 启动 gRPC 服务器
- 发起 RPC 调用
- 验证方法行为
- 使用 gRPC reflection

### 4. [处理 Protobuf](./4-serialization)
处理 Protocol Buffers 消息：
- 消息序列化
- 类型映射
- 自定义字段选项
- 流式数据

## 涵盖核心概念

{{< alert title="你将学到" color="primary" >}}
**gRPC 服务设计**
- 方法定义
- 消息建模
- 流式模式
- 使用状态码进行错误处理

**Goa 开发**
- 设计优先方法论
- Protocol Buffers 生成
- 服务实现
- 传输层配置

**生产实践**
- 输入校验
- 错误处理
- 双向流式处理
- 服务反射
{{< /alert >}}

完成本系列后，您将掌握使用 Goa 创建设计良好且高性能的 gRPC 服务。各章节循序渐进，从初始设计到完整可用的 gRPC 服务。

---

准备开始？先从 [服务设计](./1-designing) 着手，创建你的第一个 Goa gRPC 服务。