---
title: 自定义请求/响应编码
linkTitle: Encoding
weight: 4
description: "掌握 Goa 的编码系统，学习如何自定义请求/响应编码，支持 JSON 与 MessagePack 等多种内容类型，并实现自定义序列化逻辑。"
---

在实现了 Concerts 服务之后，你可能希望通过自定义编解码方式来提升 API：例如使用二进制格式提高性能、进行特殊数据处理、或支持不同内容类型。本指南将帮助你实现这些目标。

## 默认行为

开箱即用的 Goa 提供标准的编码器与解码器，覆盖常见格式：

- JSON（`application/json`）：适用于浏览器与多数 API 客户端
- XML（`application/xml`）：用于遗留系统与企业集成
- Gob（`application/gob`）：适合 Go‑to‑Go 通信

这些默认通常够用，但我们来看看如何自定义以满足特定需求。

## 修改服务器设置

先看看当前的 `main.go` 服务器设置——这里决定了处理不同内容类型的方式：

```go
func main() {
    // ... service initialization ...

    // Default encoders and decoders
    mux := goahttp.NewMuxer()
    handler := genhttp.New(
        endpoints,
        mux,
        goahttp.RequestDecoder,  // 默认请求解码器
        goahttp.ResponseEncoder, // 默认响应编码器
        nil,
        nil,
    )
}
```

### 添加自定义内容类型

为 Concerts 服务添加 MessagePack 支持！MessagePack 是一种更快更紧凑的二进制格式，非常适合高性能 API。实现如下：

```go
package main

import (
    "context"
    "net/http"
    
    "github.com/vmihailenco/msgpack/v5"
    goahttp "goa.design/goa/v3/http"
    "strings"
)

type (
    // MessagePack 编码器实现
    msgpackEnc struct {
        w http.ResponseWriter
    }

    // MessagePack 解码器实现
    msgpackDec struct {
        r *http.Request
    }
)

// 自定义编码器构造函数——创建 MessagePack 编码器
func msgpackEncoder(ctx context.Context, w http.ResponseWriter) goahttp.Encoder {
    return &msgpackEnc{w: w}
}

func (e *msgpackEnc) Encode(v any) error {
    w.Header().Set("Content-Type", "application/msgpack")
    return msgpack.NewEncoder(e.w).Encode(v)
}

// 自定义解码器构造函数——处理 MessagePack 请求体
func msgpackDecoder(r *http.Request) goahttp.Decoder {
    return &msgpackDec{r: r}
}

func (d *msgpackDec) Decode(v any) error {
    return msgpack.NewDecoder(d.r.Body).Decode(v)
}

func main() {
    // ... service initialization ...

    // 根据 Accept 选择编码器
    encodeFunc := func(ctx context.Context, w http.ResponseWriter) goahttp.Encoder {
        accept := ctx.Value(goahttp.AcceptTypeKey).(string)
        
        // 解析可能包含多个类型与 q 值的 Accept
        // 例如："application/json;q=0.9,application/msgpack"
        types := strings.Split(accept, ",")
        for _, t := range types {
            mt := strings.TrimSpace(strings.Split(t, ";")[0])
            switch mt {
            case "application/msgpack":
                return msgpackEncoder(ctx, w)
            case "application/json", "*/*":
                return goahttp.ResponseEncoder(ctx, w)
            }
        }
        
        // 默认返回 JSON
        return goahttp.ResponseEncoder(ctx, w)
    }

    // 根据 Content-Type 选择解码器
    decodeFunc := func(r *http.Request) goahttp.Decoder {
        if r.Header.Get("Content-Type") == "application/msgpack" {
            return msgpackDecoder(r)
        }
        return goahttp.RequestDecoder(r)
    }

    // 装配自定义编码/解码器
    handler := genhttp.New(
        endpoints,
        mux,
        decodeFunc,
        encodeFunc,
        nil,
        nil,
    )
}
```

## 使用不同内容类型

添加 MessagePack 支持后，使用示例：

```bash
# 使用 JSON 创建演唱会
curl -X POST http://localhost:8080/concerts \
    -H "Content-Type: application/json" \
    -d '{"artist":"The Beatles","venue":"O2 Arena"}'

# 使用 MessagePack 获取演唱会（高性能客户端）
curl http://localhost:8080/concerts/123 \
    -H "Accept: application/msgpack" \
    --output concert.msgpack

# 使用 MessagePack 创建演唱会
curl -X POST http://localhost:8080/concerts \
    -H "Content-Type: application/msgpack" \
    --data-binary @concert.msgpack
```

## 最佳实践

### 内容协商（content negotiation）

- 尊重 Accept 头以确定客户端期望的响应格式
- 未指定偏好时以 JSON 作为合理默认
- 对不支持的请求格式返回 `406 Not Acceptable`
- 在 API 文档中清晰列出支持的内容类型

### 性能考量

根据使用场景选择合适的编码格式：
- JSON：适合 Web 与调试，可读性好
- MessagePack/Protocol Buffers：适合服务间通信，对性能要求高
- 二进制格式：适合大负载，降低带宽提升传输速度
- 对高频访问资源使用响应缓存降低编码开销

### 错误处理

- 在处理请求体前校验 Content‑Type 头
- 提供清晰可操作的错误消息，便于客户端诊断
- 保持一致的错误响应结构
- 在文档中列举常见错误场景与响应

### 测试

- 针对每种支持的内容类型测试有效与无效的 payload
- 验证不支持内容类型与畸形数据的错误响应
- 确认 Accept 与 Content‑Type 头的处理
- 覆盖边界情况（空 body、charset 变化）
- 配置自动化测试防止编码回归

更多关于内容协商的细节，参见 [Content Negotiation](../../4-concepts/3-http/1-content)。


## 总结

恭喜！🎉 你已经学会了：
- 支持高效的二进制格式（如 MessagePack）
- 像专家一样处理自定义内容类型
- 实现特殊的编码逻辑
- 掌握内容协商

你的 Concerts API 现在已经可以处理多种格式的数据交换，使其更加通用并具备更好的性能。无论客户端偏好简单的 JSON 还是追求速度的 MessagePack，你都能很好地支持！

至此，我们的 REST API 教程系列告一段落。你现在已经拥有一个功能完备的 Concerts API，具备自定义编码支持，已准备好应对真实世界的场景！