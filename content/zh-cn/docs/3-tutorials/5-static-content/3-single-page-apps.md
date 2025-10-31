---
title: "单页应用（SPA）集成"
linkTitle: 单页应用
weight: 3
description: "学习在 Goa 服务中嵌入并提供单页应用（SPA），包含 React 集成、客户端路由支持以及生产部署策略。"
---

对于简单应用，你可以使用 `go:embed` 将 React 应用直接嵌入到 Go 二进制中。这种方式兼具现代前端开发的优势与 Go 的简化部署能力。通过将整个应用（后端 API 与 React 前端）打包为一个自包含的可执行文件，你无需额外管理部署产物或配置静态文件服务器。只需构建二进制、部署并运行即可。该方法极大简化了部署，同时确保前后端版本保持一致。

## 项目结构

包含 React SPA 的项目结构示例如下：

```
myapp/
├── cmd/                  # 主应用
├── design/               # 共享的设计构件
│   ├── design.go         # 导入非 API 服务设计
│   └── shared/           # 共享设计构件
├── gen/                  # 非 API 服务的 Goa 生成代码
└── services/
    ├── api/
    │   ├── design/       # API 设计
    │   ├── gen/          # Goa 生成代码
    │   ├── api.go        # API 实现
    │   └── ui/
    │       ├── build/    # 前端构建产物
    │       ├── src/      # React 源码
    │       └── public/   # 静态资源
    ├── service1/         # 其他服务
    :
```

该设计组织遵循特定模式，以保持清晰的关注点分离：

1. 公共 HTTP API 服务（`services/api`）拥有独立的 `design` 包与 `gen` 目录。该隔离使生成的 OpenAPI 规范仅关注公共 API 端点。

2. 其他服务的设计统一导入到顶层 `design` 包，其代码生成到根部的 `gen` 目录。这样只需两条命令即可完成代码生成：
   - `goa gen myapp/services/api/design`（生成公共 API）
   - `goa gen myapp/design`（生成其他服务）

这种结构清晰标识哪些端点属于公共 API，同时保持代码生成流程高效。

此外，顶层 `design` 包可包含所有服务共享的设计构件。

`ui/` 目录包含会被嵌入到 Go 二进制中的 React 应用与静态资源。

## 设计（Design）

定义服务以同时处理 API 端点并提供 SPA。在开发期间，需要配置 CORS 以允许 React 开发服务器与 Goa 后端通信：

```go
var _ = Service("myapp", func() {
    Description("myapp 服务同时提供前端 UI 与 API。")
    
    // 为开发环境配置 CORS
    cors.Origin("http://localhost:3000", func() {
        cors.Headers("Content-Type")
        cors.Methods("GET", "POST", "PUT", "DELETE")
        cors.Credentials()
    })
    
    // 提供 React 应用
    Files("/ui/{*filepath}", "ui/build")
    
    // 直接提供静态资源
    Files("/robots.txt", "ui/public/robots.txt")
    Files("/favicon.ico", "ui/public/favicon.ico")
    
    // 处理 UI 路径
    Method("home", func() {
        HTTP(func() {
            GET("/")
            GET("/ui")
            Redirect("/ui/", StatusMovedPermanently) // 将根路径重定向到 /ui/
        })
    })
    
    // API 端点
    Method("list_widgets", func() {
        Description("列出部件。")
        Result(ArrayOf(Widget))
        HTTP(func() {
            GET("/api/widgets")
            Response(StatusOK)
        })
    })
    
    // ... 更多 API 端点
})
```

该设计包含以下目的：
1. 为 React 默认端口的开发环境配置 CORS
2. 将 React 应用挂载到 `/ui`
3. 处理静态资源与重定向
4. 在 `/api` 下提供 API 端点
5. 通过通配符路径支持客户端路由

关于 CORS 设置的详细信息请参阅 [CORS 插件](https://github.com/goadesign/plugins/tree/master/cors)。

## 实现（Implementation）

### 服务结构

服务实现会嵌入 React 构建产物并处理 API 请求：

```go
package front

import (
    "embed"
    "context"
)

//go:embed ui/build
var UIBuildFS embed.FS

type Service struct {
    // 依赖等
}

func New() *Service {
    return &Service{}
}

// Home 实现重定向处理器
func (svc *Service) Home(ctx context.Context) error {
    return nil
}

// API 方法实现
func (svc *Service) ListWidgets(ctx context.Context) ([]*Widget, error) {
    // 具体实现
}
```

### main 函数设置

在 `main` 中配置服务：

```go
func main() {
    // 创建服务与端点
    svc := myapp.New()
    endpoints := genmyapp.NewEndpoints(svc)
    
    // 创建传输层
    mux := goahttp.NewMuxer()
    server := genserver.New(
        endpoints,
        mux,
        goahttp.RequestDecoder,
        goahttp.ResponseEncoder,
        nil,
        nil,
        http.FS(myapp.UIBuildFS),  // 提供 UI
    )
    genserver.Mount(mux, server)
    
    // 启动服务器
    if err := http.ListenAndServe(":8000", mux); err != nil {
        log.Fatal(err)
    }
}
```

## 开发工作流

本地开发：

1. 在 package.json 中配置 React 开发服务器：
```json
{
  "proxy": "http://localhost:8000"
}
```

2. 启动 React 开发服务器：
```bash
cd services/api/ui
npm start
```

3. 运行 Go 服务：
```bash
go run myapp/cmd/myapp
```

package.json 中的代理配置会与 CORS 设置配合，支持顺畅的开发体验。React 开发服务器会在本地直接提供 UI，同时将 API 请求转发到 Goa 后端。

## 构建流程

1. 构建 React 应用：
```bash
cd services/api/ui && npm run build
```

2. 构建 Go 二进制：
```bash
go build myapp/cmd/myapp
```

## 最佳实践

1. **API 组织**
   将所有 API 路由置于统一的 `/api` 前缀下，以清晰区分 API 与静态内容。同时，确保错误响应在所有端点上遵循统一格式，为使用者提供一致体验。最后，根据功能或资源类型对相关端点进行分组组织，保持结构清晰、直观。

2. **CORS 配置**
   在生产环境中，明确允许访问 API 的来源域，避免使用通配符；对预检请求进行合理缓存以减少不必要的 OPTIONS 请求并提升性能；仅暴露 API 实际需要的 HTTP 头与方法，遵循最小权限原则；谨慎评估启用凭据模式的安全影响（会允许跨域请求携带 cookie 与认证头），仅在确有必要时启用。

3. **SPA 提供**
   将 SPA 放在专用路径（如 `/ui`）下提供，以清晰地与 API 路由分离；实现对根路径重定向的合理处理，保证简洁友好的 URL；服务器需支持客户端路由，对前端应用定义的所有路由返回主 `index.html`。

4. **开发**
   开发阶段使用 React 开发服务器并配置代理，将 API 请求路由到 Go 服务；同时获得热更新等开发体验。保持 UI 代码靠近提供它的服务，便于维护前后端的关联关系；并正确处理跨域（CORS），保障开发与生产环境中的前后端通信安全。

5. **生产**
   在生产部署中，务必先构建 React 应用再构建 Go 二进制，确保最新前端代码被嵌入服务；为 JS、CSS、图片等静态资源设置合理的缓存头以提升性能并降低负载；建立完善的日志与监控，以便观测运行状况与性能；实现优雅关闭，确保服务停止时在途请求能够成功完成。