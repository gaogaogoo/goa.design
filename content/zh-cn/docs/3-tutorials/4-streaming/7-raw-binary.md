---
title: "通过 HTTP 传输原始二进制数据"
linkTitle: "原始二进制流"
weight: 7
description: "学习如何使用 Goa 的底层流式能力，通过 HTTP 高效传输文件和多媒体等原始二进制数据。"
---

尽管 Goa 的 `StreamingPayload` 和 `StreamingResult` 对类型化数据流非常适用，但有时你需要直接访问原始二进制数据流。这在处理文件上传、下载或多媒体流时很常见。Goa 通过 `SkipRequestBodyEncodeDecode` 和 `SkipResponseBodyEncodeDecode` 功能提供了这种能力。

## 选择你的流式方案

Goa 提供了两种截然不同的流式方法，适用于不同需求：

当你处理具有已知类型的结构化数据时，`StreamingPayload` 与 `StreamingResult` 是理想选择。它在需要类型安全、校验或 gRPC 兼容时尤其有用。该方法利用 Goa 的类型系统确保数据流保持预期结构。

`SkipRequestBodyEncodeDecode` 方法则让你直接访问原始 HTTP 请求体流。当处理文件等二进制数据或需要完全掌控数据处理时，它是正确选择。对于大文件，它尤其高效，因为避免了不必要的编解码步骤。

## 请求流（Request Streaming）

请求流允许服务在数据到达时立即处理，而无需等待完整载荷。以下展示如何使用原始流式处理实现文件上传：

```go
var _ = Service("upload", func() {
    Method("upload", func() {
        Payload(func() {
            // 注意：使用流式时不能定义 body 中的属性
            Attribute("content_type", String)
            Attribute("dir", String)
        })
        HTTP(func() {
            POST("/upload/{*dir}")
            Header("content_type:Content-Type")
            SkipRequestBodyEncodeDecode()
        })
    })
})
```

服务实现会接收一个用于读取请求体的 `io.ReadCloser`：

```go
func (s *service) Upload(ctx context.Context, p *upload.Payload, body io.ReadCloser) error {
    defer body.Close()
    
    buffer := make([]byte, 32*1024)
    for {
        n, err := body.Read(buffer)
        if err == io.EOF {
            break
        }
        if err != nil {
            return err
        }
        // 处理 buffer[:n]
    }
    return nil
}
```

## 响应流（Response Streaming）

响应流允许服务按增量向客户端发送数据，适用于文件下载或实时数据源。以下展示其实现方式：

```go
var _ = Service("download", func() {
    Method("download", func() {
        Payload(String)
        Result(func() {
            Attribute("length", Int64)
        })
        HTTP(func() {
            GET("/download/{*filename}")
            SkipResponseBodyEncodeDecode()
            Response(func() {
                Header("length:Content-Length")
            })
        })
    })
})
```

服务实现会同时返回结果与一个 `io.ReadCloser`：

```go
func (s *service) Download(ctx context.Context, p string) (*download.Result, io.ReadCloser, error) {
    file, err := os.Open(p)
    if err != nil {
        return nil, nil, err
    }
    
    stat, err := file.Stat()
    if err != nil {
        file.Close()
        return nil, nil, err
    }
    
    return &download.Result{
        Length: stat.Size(),
    }, file, nil
}
```

## 完整示例

以下是一个在同一服务中同时展示文件上传与下载流式处理的完整示例：

```go
package design

import . "goa.design/goa/v3/dsl"

var _ = API("streaming", func() {
    Title("Streaming API Example")
})

var _ = Service("files", func() {
    Method("upload", func() {
        Payload(func() {
            Attribute("content_type", String)
            Attribute("filename", String)
        })
        HTTP(func() {
            POST("/upload/{filename}")
            Header("content_type:Content-Type")
            SkipRequestBodyEncodeDecode()
        })
    })
    
    Method("download", func() {
        Payload(String)
        Result(func() {
            Attribute("length", Int64)
        })
        HTTP(func() {
            GET("/download/{*filepath}")
            SkipResponseBodyEncodeDecode()
            Response(func() {
                Header("length:Content-Length")
            })
        })
    })
})
```

下面的实现展示了一个同时处理上传与下载的完整文件服务：

```go
type filesService struct {
    storageDir string
}

func (s *filesService) Upload(ctx context.Context, p *files.UploadPayload, body io.ReadCloser) error {
    defer body.Close()
    
    fpath := filepath.Join(s.storageDir, p.Filename)
    f, err := os.Create(fpath)
    if err != nil {
        return err
    }
    defer f.Close()
    
    _, err = io.Copy(f, body)
    return err
}

func (s *filesService) Download(ctx context.Context, p string) (*files.DownloadResult, io.ReadCloser, error) {
    fpath := filepath.Join(s.storageDir, p)
    f, err := os.Open(fpath)
    if err != nil {
        return nil, nil, err
    }
    
    stat, err := f.Stat()
    if err != nil {
        f.Close()
        return nil, nil, err
    }
    
    return &files.DownloadResult{
        Length: stat.Size(),
    }, f, nil
}
```

让我们来看看该实现的关键点：

该服务围绕一个简单的存储目录概念构建。每个实例都配置了一个基础目录，用于存储和读取所有文件。将文件操作限定在特定目录内，为文件操作提供了基本的安全边界。

在上传方面，我们采用了尽量减少内存使用的流式方案。与其将整个文件缓存在内存中，不如使用 `io.Copy` 将数据直接从请求体流式写入文件系统。实现通过 `defer` 语句谨慎管理资源，确保无论操作成功或失败都能得到正确清理。

下载实现同样高效。当发起下载请求时，我们首先打开文件并在一次操作中获取其元数据。这让我们能将文件大小提供给 Goa（用于设置 Content-Length），同时获取用于流式传输的文件句柄。注意在成功的路径中我们不主动关闭文件——Goa 会接管该文件句柄，并在将内容流式发送给客户端后关闭它。

在整个过程里，错误处理是重点。代码在错误发生时正确清理资源、将错误清晰地向调用方传播，并安全地处理文件路径以防止目录穿越攻击。对错误处理的重视有助于确保服务在各种故障情形下保持健壮与安全。

该实现通过以下方式体现了高效的流式处理：
- 直接使用文件系统进行数据流式传输
- 通过 defer 语句正确管理资源
- 提供准确的内容长度信息
- 实现恰当的错误处理
- 确保文件路径处理的安全性

关于静态文件与模板的相关内容，请参阅 [静态内容](../5-static-content) 章节。
