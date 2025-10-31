---
title: "模板集成"
linkTitle: 模板集成
weight: 2
description: "将 Go 的模板引擎与 Goa 集成以渲染动态 HTML 内容，涵盖模板组合、数据传递以及正确的错误处理。"
---

Goa 服务可使用标准库 `html/template` 渲染动态 HTML 内容。本文将演示如何在 Goa 服务中集成模板渲染。

## 设计（Design）

首先，定义用于渲染 HTML 模板的服务端点：

```go
package design

import . "goa.design/goa/v3/dsl"

var _ = Service("front", func() {
    Description("带模板渲染的前端 Web 服务")

    Method("home", func() {
        Description("渲染首页")
        
        Payload(func() {
            Field(1, "name", String, "在首页上展示的名称")
            Required("name")
        })
        
        Result(Bytes)
        
        HTTP(func() {
            GET("/")
            Response(StatusOK, func() {
                ContentType("text/html")
            })
        })
    })
})
```

## 实现（Implementation）

### 服务结构

创建一个负责模板渲染的服务：

```go
package front

import (
    "context"
    "embed"
    "html/template"
    "bytes"
    "fmt"

    genfront "myapp/gen/front" // 请替换为你的生成包名
)

//go:embed templates/*.html
var templateFS embed.FS

type Service struct {
    tmpl *template.Template
}

func New() (*Service, error) {
    tmpl, err := template.ParseFS(templateFS, "templates/*.html")
    if err != nil {
        return nil, fmt.Errorf("解析模板失败: %w", err)
    }
    
    return &Service{tmpl: tmpl}, nil
}
```

### 模板渲染

实现服务方法以渲染模板：

```go
func (svc *Service) Home(ctx context.Context, p *genfront.HomePayload) ([]byte, error) {
    // 准备模板数据
    data := map[string]interface{}{
        "Title":   "Welcome",
        "Content": "Welcome to " + p.Name + "!",
    }
    
    // 使用缓冲区存储渲染结果
    var buf bytes.Buffer
    
    // 渲染模板到缓冲区
    if err := svc.tmpl.ExecuteTemplate(&buf, "home.html", data); err != nil {
        return nil, fmt.Errorf("渲染模板失败: %w", err)
    }
    
    return buf.Bytes(), nil
}
```

### 模板结构

在 `templates` 目录中创建你的 HTML 模板：

```html
<!-- templates/base.html -->
<!DOCTYPE html>
<html>
<head>
    <title>{{.Title}}</title>
</head>
<body>
    {{block "content" .}}{{end}}
</body>
</html>

<!-- templates/home.html -->
{{template "base.html" .}}
{{define "content"}}
    <h1>{{.Title}}</h1>
    <p>{{.Content}}</p>
{{end}}
```

### 服务器设置

将模板服务集成到服务器：

```go
func main() {
    // 创建服务
    svc := front.New()
    
    // 初始化 HTTP 服务器
    endpoints := genfront.NewEndpoints(svc)
    mux := goahttp.NewMuxer()
    server := genserver.New(endpoints, mux, goahttp.RequestDecoder, goahttp.ResponseEncoder, nil, nil)
    genserver.Mount(server, mux)
    
    // 启动服务器
    if err := http.ListenAndServe(":8080", mux); err != nil {
        log.Fatalf("启动服务器失败: %v", err)
    }
}
```

## 可选：同时提供静态与动态内容

如果既需要提供静态文件也要渲染动态模板，可以在服务设计中进行组合：

```go
var _ = Service("front", func() {
    // 动态模板端点
    Method("home", func() {
        HTTP(func() {
            GET("/")
            Response(StatusOK, func() {
                ContentType("text/html")
            })
        })
    })
    
    // 静态文件服务
    Files("/static/{*filepath}", "public/static")
})
```

该配置实现：
- 根路径（/）下的模板动态内容
- /static 路径下的静态文件（CSS、JS、图片）
- `public/static` 下的所有文件将可通过 /static/ 访问

## 最佳实践

1. **模板组织**
   - 使用基础布局进行模板组合
   - 将模板集中放置在独立目录
   - 使用 embed.FS 将模板与二进制一起打包

2. **错误处理**
   - 在服务初始化时解析模板
   - 模板渲染失败时返回有意义的错误
   - 设置合理的 HTTP 响应码与头

3. **性能**
   - 启动时仅解析一次模板
   - 生产环境使用模板缓存
   - 开发环境可考虑实现模板热重载