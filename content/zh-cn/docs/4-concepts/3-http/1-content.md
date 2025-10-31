---
title: "内容协商"
linkTitle: "内容协商"
weight: 1
description: "学习如何在 Goa 的 HTTP 服务中处理多种内容类型、解析 Accept 头，并实现自定义编码器/解码器。"
menu:
  main:
    parent: "HTTP Advanced Topics"
    weight: 1
---

内容协商使你的 HTTP 服务能够支持多种内容类型与格式。Goa 提供了灵活的编码与解码策略，可将任意编码器和解码器与 HTTP 响应和请求的内容类型关联起来。

## 服务器构造

生成的 HTTP 服务器构造函数接收编码器与解码器工厂函数作为参数，方便提供自定义实现：

```go
// New 实例化所有服务端点的 HTTP 处理器
func New(
    e *divider.Endpoints,
    mux goahttp.Muxer,
    decoder func(*http.Request) goahttp.Decoder,
    encoder func(context.Context, http.ResponseWriter) goahttp.Encoder,
    errhandler func(context.Context, http.ResponseWriter, error),
    formatter func(context.Context, err error) goahttp.Statuser,
) *Server
```

Goa 的默认编码器与解码器由 `http` 包提供，可如下使用：

```go
import (
    // ...
    "goa.design/goa/v3/http"
)

// ...

server := calcsvr.New(endpoints, mux, http.RequestDecoder, http.ResponseEncoder, nil, nil)
```

## 内容类型支持

在 Goa 中，内容类型支持决定了数据如何在网络边界进行序列化与反序列化。编码与解码角色在客户端与服务端之间按执行顺序切换：

- 客户端编码：准备请求体进行发送
- 服务端解码：处理传入的请求体
- 服务端编码：处理发送给客户端的响应体
- 客户端解码：处理接收到的响应体

### 内置编码器/解码器

默认的 Goa 编解码器支持多种常见内容类型，包括：
- JSON 及其变体（`application/json`、`*+json`）
- XML 及其变体（`application/xml`、`*+xml`）
- Gob 及其变体（`application/gob`、`*+gob`）
- HTML（`text/html`）
- 纯文本（`text/plain`）

后缀匹配模式允许内容类型变体，例如 `application/ld+json`、`application/hal+json` 与 `application/vnd.api+json`。

### 响应内容类型

默认响应编码器实现了内容协商策略，按顺序考虑以下因素：

- 首先检查传入请求的 `Accept` 头，确定客户端的内容类型偏好。
- 若未提供 Accept 头，则查看请求的 `Content-Type` 头。
- 若二者都未提供有效信息，则回退到响应的默认内容类型。

在服务端，编码器会解析客户端的 `Accept` 头以确定偏好，然后在支持的类型中选择最合适的编码器。当在可接受类型中未找到合适匹配时，编码器会默认使用 JSON。

在客户端，解码器会根据响应头中指定的内容类型处理收到的响应。当遇到未知内容类型时，会安全地回退到 JSON 解码以保持兼容性。

### 请求内容类型

请求的内容类型处理比响应更为简单，主要依赖请求的 `Content-Type` 头，在必要时回退到默认内容类型。

在服务端，解码器首先检查请求的 `Content-Type` 头，并据此选择相应的解码器实现（JSON、XML 或 gob）。如果缺失或不支持，则回退到 JSON 以确保请求可继续处理。

在客户端，编码器会基于请求配置设置内容类型，并相应地编码请求体。当未提供具体内容类型时，默认采用 JSON 编码以保证行为一致。

在所有情况下，如果编码或解码失败，Goa 会调用在创建 HTTP 服务器时注册的错误处理器，以便优雅地处理错误并向客户端返回适当反馈。

### 设置默认内容类型

使用 `ContentType` DSL 指定默认响应内容类型：

```go
var _ = Service("media", func() {
    Method("create", func() {
        HTTP(func() {
            POST("/media")
            Response(StatusCreated, func() {
                // 覆盖响应内容类型
                ContentType("application/json")
            })
        })
    })
})
```

设置后会覆盖请求头中指定的内容类型，但不会覆盖 `Accept` 头的取值。

## 自定义编码器/解码器

当 Goa 的内置编码器无法满足需求时，你可以实现自定义编码器与解码器。例如为 MessagePack 或 BSON 等默认未包含的特殊格式提供支持；或为特定场景优化编码性能、在响应中添加压缩或加密层，或与遗留/专有格式保持兼容。

### 创建自定义编码器

编码器需实现 Goa `http` 包中定义的 `Encoder` 接口，并提供构造函数：

```go
// 响应编码器接口
type Encoder interface {
    Encode(v any) error
}

// 构造函数
func NewMessagePackEncoder(ctx context.Context, w http.ResponseWriter) goahttp.Encoder {
    return &MessagePackEncoder{w: w}
}
```

构造函数签名必须返回 `Encoder` 与错误：

```go
// 构造函数签名
func(ctx context.Context, w http.ResponseWriter) (goahttp.Encoder, error)

// MessagePack 编码器示例
type MessagePackEncoder struct {
    w http.ResponseWriter
}

func (enc *MessagePackEncoder) Encode(v interface{}) error {
    // 设置内容类型响应头
    enc.w.Header().Set("Content-Type", "application/msgpack")
    
    // 使用 MessagePack 编码
    return msgpack.NewEncoder(enc.w).Encode(v)
}

// 构造函数
func NewMessagePackEncoder(ctx context.Context, w http.ResponseWriter) goahttp.Encoder {
    return &MessagePackEncoder{w: w}
}
```

上下文包含 `ContentTypeKey` 与 `AcceptTypeKey` 值，可用于内容类型协商。

### 创建自定义解码器

解码器需实现 `goahttp.Decoder` 接口并提供构造函数：

```go
// 构造函数签名
func(r *http.Request) (goahttp.Decoder, error)

// MessagePack 解码器示例
type MessagePackDecoder struct {
    r *http.Request
}

func (dec *MessagePackDecoder) Decode(v interface{}) error {
    return msgpack.NewDecoder(dec.r.Body).Decode(v)
}

// 构造函数
func NewMessagePackDecoder(r *http.Request) goahttp.Decoder {
    return &MessagePackDecoder{r: r}
}
```

构造函数可访问请求对象，并可检查其状态以确定合适的解码器。

### 注册自定义编码器/解码器

在创建 HTTP 服务器时使用自定义的编码器/解码器：

```go
func main() {
    // 创建端点
    endpoints := myapi.NewEndpoints(svc)
    
    // 创建解码器工厂
    decoder := func(r *http.Request) goahttp.Decoder {
        switch r.Header.Get("Content-Type") {
        case "application/msgpack":
            return NewMessagePackDecoder(r)
        default:
            return goahttp.RequestDecoder(r) // 默认 Goa 解码器
        }
    }
    
    // 创建编码器工厂
    encoder := func(ctx context.Context, w http.ResponseWriter) goahttp.Encoder {
        if accept := ctx.Value(goahttp.AcceptTypeKey).(string); accept == "application/msgpack" {
            return NewMessagePackEncoder(ctx, w)
        }
        return goahttp.ResponseEncoder(ctx, w) // 默认 Goa 编码器
    }
    
    // 使用自定义编码器/解码器创建 HTTP 服务器
    server := myapi.NewServer(endpoints, mux, decoder, encoder, nil, nil)
}
```

### 自定义编解码器的最佳实践

1. **错误处理**
   - 在编码/解码失败时返回有意义的错误
   - 为特定失败考虑实现自定义错误类型
   - 正确处理 nil 值与边界情况

2. **性能**
   - 为大载荷考虑缓冲
   - 为高频对象实现池化
   - 对热点路径进行性能剖析与优化

3. **HTTP 头**
   - 设置合适的 Content-Type 头
   - 必要时处理 Content-Length
   - 根据需要添加元数据的自定义头

4. **内容协商**
   - 在选择编码器时尊重 Accept 头
   - 对不支持的格式提供清晰错误信息
   - 考虑支持质量值（q-values）

## 内容类型协商

### 处理 Accept 头

Goa 会自动处理 Accept 头：

1. 客户端发送带偏好的 Accept 头
2. Goa 与支持的内容类型进行匹配
3. 选择最佳匹配的内容类型
4. 按所选类型对响应进行编码

Accept 头示例：

```go
Accept: application/json;q=0.9, application/xml;q=0.8
```

Goa 将：
1. 解析质量值
2. 与支持类型进行检查
3. 选择最高质量的受支持匹配
4. 若无匹配使用默认（通常为 application/json）

### 通过内容类型进行版本化

使用内容类型进行 API 版本化：

```go
var _ = Service("versioned", func() {
    HTTP(func() {
        Response(func() {
            ContentType(
                "application/vnd.api.v1+json",
                "application/vnd.api.v2+json"
            )
        })
    })
})
```

## 最佳实践

{{< alert title="内容类型指南" color="primary" >}}
1. **默认内容类型**：始终提供默认内容类型
2. **内容类型顺序**：按偏好优先级排列
3. **版本化**：考虑使用内容类型进行版本管理
4. **错误响应**：采用一致的错误响应格式
5. **文档**：清晰记录支持的内容类型
{{< /alert >}}

## 测试内容协商

测试自定义编码器与解码器需要对内容类型处理与协商进行严格验证。以下示例展示如何使用 Clue 的 [mock 包](https://github.com/goadesign/clue/tree/main/mock)有效测试内容协商：

```go
// 引入 Clue 的 mock 包
import (
    "github.com/goadesign/clue/mock"
)

// 使用 Clue 的 mock 包实现模拟编码器
// 展示如何使用 Clue 正确组织模拟编码器
type mockEncoder struct {
    *mock.Mock // 嵌入 Clue 的 Mock 类型
}

// Encode 使用 Clue 的 Next 模式实现模拟行为
func (m *mockEncoder) Encode(v interface{}) error {
    if f := m.Next("Encode"); f != nil {
        return f.(func(interface{}) error)(v)
    }
    return errors.New("unexpected call to Encode")
}

// 使用 Clue 的 mock 包实现模拟解码器
// 展示如何使用 Clue 正确组织模拟解码器
type mockDecoder struct {
    *mock.Mock // 嵌入 Clue 的 Mock 类型
}

// Decode 使用 Clue 的 Next 模式实现模拟行为
func (m *mockDecoder) Decode(v interface{}) error {
    if f := m.Next("Decode"); f != nil {
        return f.(func(interface{}) error)(v)
    }
    return errors.New("unexpected call to Decode")
}

func TestContentNegotiation(t *testing.T) {
    // 使用 Clue 的 mock 包创建模拟编码器与解码器
    encoder := &mockEncoder{mock.New()}
    decoder := &mockDecoder{mock.New()}
    
    tests := []struct {
        name         string
        contentType  string
        accept       string
        setupEncoder func(*mockEncoder)
        setupDecoder func(*mockDecoder)
        input        interface{}
        wantErr      bool
    }{
        {
            name:        "JSON 内容类型",
            contentType: "application/json",
            accept:      "application/json",
            setupEncoder: func(e *mockEncoder) {
                // 使用 Clue 的 Set 方法设置永久行为
                e.Set("Encode", func(v interface{}) error {
                    // 验证 JSON 编码
                    _, ok := v.(json.Marshaler)
                    if !ok {
                        return fmt.Errorf("value does not implement json.Marshaler")
                    }
                    return nil
                })
            },
            setupDecoder: func(d *mockDecoder) {
                // 使用 Clue 的 Set 方法设置永久行为
                d.Set("Decode", func(v interface{}) error {
                    // 验证 JSON 解码
                    _, ok := v.(json.Unmarshaler)
                    if !ok {
                        return fmt.Errorf("value does not implement json.Unmarshaler")
                    }
                    return nil
                })
            },
            input:   &TestStruct{ID: "test"},
            wantErr: false,
        },
        {
            name:        "MessagePack 内容类型",
            contentType: "application/msgpack",
            accept:      "application/msgpack",
            setupEncoder: func(e *mockEncoder) {
                // 使用 Clue 的 Add 方法设置按顺序的行为
                e.Add("Encode", func(v interface{}) error {
                    // 验证 MessagePack 编码
                    _, ok := v.(msgpack.Marshaler)
                    if !ok {
                        return fmt.Errorf("value does not implement msgpack.Marshaler")
                    }
                    return nil
                })
            },
            setupDecoder: func(d *mockDecoder) {
                // 使用 Clue 的 Add 方法设置按顺序的行为
                d.Add("Decode", func(v interface{}) error {
                    // 验证 MessagePack 解码
                    _, ok := v.(msgpack.Unmarshaler)
                    if !ok {
                        return fmt.Errorf("value does not implement msgpack.Unmarshaler")
                    }
                    return nil
                })
            },
            input:   &TestStruct{ID: "test"},
            wantErr: false,
        },
        {
            name:        "不支持的内容类型",
            contentType: "application/unknown",
            accept:      "application/unknown",
            setupEncoder: func(e *mockEncoder) {
                // 使用 Clue 模拟错误处理
                e.Set("Encode", func(v interface{}) error {
                    return fmt.Errorf("unsupported content type")
                })
            },
            setupDecoder: func(d *mockDecoder) {
                // 使用 Clue 模拟错误处理
                d.Set("Decode", func(v interface{}) error {
                    return fmt.Errorf("unsupported content type")
                })
            },
            input:   &TestStruct{ID: "test"},
            wantErr: true,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 为每个用例重新创建 Clue 模拟对象
            encoder := &mockEncoder{mock.New()}
            decoder := &mockDecoder{mock.New()}
            
            if tt.setupEncoder != nil {
                tt.setupEncoder(encoder)
            }
            if tt.setupDecoder != nil {
                tt.setupDecoder(decoder)
            }
            
            // 创建带内容类型的请求
            req := httptest.NewRequest("POST", "/", nil)
            req.Header.Set("Content-Type", tt.contentType)
            req.Header.Set("Accept", tt.accept)
            
            // 创建响应记录器
            rec := httptest.NewRecorder()
            
            // 创建编码器工厂函数
            encoderFn := func(ctx context.Context, w http.ResponseWriter) goahttp.Encoder {
                return encoder
            }
            
            // 创建解码器工厂函数
            decoderFn := func(r *http.Request) goahttp.Decoder {
                return decoder
            }
            
            // 使用自定义编解码器创建处理器
            handler := goahttp.RequestDecoder(decoderFn)
            
            // 测试请求解码
            err := handler(req, tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("RequestDecoder() error = %v, wantErr %v", err, tt.wantErr)
            }
            
            // 使用 Clue 的 HasMore 方法验证所有预期调用都已发生
            if encoder.HasMore() {
                t.Error("not all expected encoder operations were performed")
            }
            if decoder.HasMore() {
                t.Error("not all expected decoder operations were performed")
            }
        })
    }
}
```

该示例展示了 Clue 的 [mock 包](https://github.com/goadesign/clue/tree/main/mock) 的多个关键特性：

1. **类型安全的模拟**：通过嵌入 Clue 的 `Mock` 类型，`mockEncoder` 与 `mockDecoder` 提供类型安全的接口。
2. **灵活的行为定义**：
   - 使用 `Set` 定义永久行为（见 JSON 内容类型用例）
   - 使用 `Add` 定义按顺序的行为（见 MessagePack 内容类型用例）
3. **全面的验证**：`HasMore` 方法确保所有预期操作都已执行
4. **错误处理**：演示正确的错误场景与校验（见不支持的内容类型用例）

这些测试用例涵盖了内容协商的核心场景。它们通过永久模拟行为验证了常见的 JSON 内容协商处理，也通过顺序模拟行为验证了二进制格式 MessagePack 的协商支持。错误条件通过验证不支持的内容类型的正确处理来覆盖。测试套件还通过验证所有模拟期望均被满足来确保妥善的清理。

测试验证了：
1. 通过请求/响应头进行内容类型处理
2. 基于内容类型进行编码器/解码器选择
3. 针对不支持格式的错误处理
4. 正确实现编解码接口
5. 对预期操作的完整验证

## 下一步

- 查看 [错误处理](../../3-tutorials/3-error-handling) 获取错误响应格式
- 探索 [流式传输](../../3-tutorials/4-streaming) 支持流式内容类型
- 参考 [静态内容](../../3-tutorials/5-static-content) 了解文件上传/下载
- 学习 [拦截器](../../4-concepts/5-interceptors) 以处理业务逻辑
