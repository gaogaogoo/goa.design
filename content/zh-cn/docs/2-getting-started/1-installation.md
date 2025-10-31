---
title: 安装
weight: 1
description: "安装 Goa 和设置开发环境的逐步指南，包括先决条件和验证步骤。"
---

## 先决条件

Goa 需要使用 **Go 模块**，因此请确保在您的 Go 环境中启用了它们。

- 使用 **Go 1.18+**（推荐）。
- 启用 **Go 模块**：确认在您的环境中已启用（例如，`GO111MODULE=on` 或使用 Go 1.16+ 默认设置）。

## 安装 Goa

```bash
# 拉取 Goa 包
go get goa.design/goa/v3/...

# 安装 Goa CLI
go install goa.design/goa/v3/cmd/goa@latest

# 验证安装
goa version
```

您应该看到当前的 Goa 版本（例如 `v3.x.x`）。

---

继续查看 [第一个服务](./2-first-service/) 了解如何创建您的第一个服务。
