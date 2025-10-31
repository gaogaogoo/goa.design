---
title: 编写服务客户端
weight: 2
---

在构建微服务时，一个常见挑战是如何组织服务之间的通信。本文介绍为 Goa 服务编写客户端的最佳实践，重点关注如何创建易维护、易测试的客户端实现。

## 客户端设计理念

为 Goa 服务构建客户端的推荐方法遵循以下关键原则：

1. 单一职责：为每个下游服务创建一个独立客户端，而非共享的通用客户端库
2. 窄接口：定义只暴露消费方所需方法的接口
3. 实现独立：在同一接口后同时支持不同的传输协议（gRPC、HTTP）
4. 可测试性：通过清晰的接口设计便于在测试中进行 Mock

该方法有助于避免形成“分布式单体”，即服务通过共享的客户端库变得紧密耦合。

## 客户端结构

一个典型的 Goa 服务客户端包含：

1. 定义服务契约的客户端接口
2. 表示领域模型的相关数据类型
3. 使用 Goa 生成的客户端的具体实现
4. 用于创建客户端实例的工厂函数

以下是一段完整的天气预报服务客户端示例：

```go
package forecaster

import (
    "context"

    "google.golang.org/grpc"

    "goa.design/clue/debug"
    genforecast "goa.design/clue/example/weather/services/forecaster/gen/forecaster"
    gengrpcclient "goa.design/clue/example/weather/services/forecaster/gen/grpc/forecaster/client"
)

type (
    // Client 是 forecast 服务的客户端。
    Client interface {
        // GetForecast 获取给定位置的天气预报。
        GetForecast(ctx context.Context, lat, long float64) (*Forecast, error)
    }

    // Forecast 表示给定位置的天气预报。
    Forecast struct {
        // Location 为预报的位置。
        Location *Location
        // Periods 为该位置的各时段预报。
        Periods []*Period
    }

    // Location 表示一个预报的地理位置。
    Location struct {
        // Lat 为位置的纬度。
        Lat float64
        // Long 为位置的经度。
        Long float64
        // City 为位置所属城市。
        City string
        // State 为位置所属州/省。
        State string
    }

    // Period 表示一个预报时段。
    Period struct {
        // Name 为预报时段名称。
        Name string
        // StartTime 为预报时段起始时间（RFC3339 格式）。
        StartTime string
        // EndTime 为预报时段结束时间（RFC3339 格式）。
        EndTime string
        // Temperature 为该时段预测温度。
        Temperature int
        // TemperatureUnit 为该时段温度单位。
        TemperatureUnit string
        // Summary 为该时段概述。
        Summary string
    }

    // client 为客户端实现。
    client struct {
        genc *genforecast.Client
    }
)

// New 实例化一个新的 forecast 服务客户端。
func New(cc *grpc.ClientConn) Client {
    c := gengrpcclient.NewClient(cc, grpc.WaitForReady(true))
    forecast := debug.LogPayloads(debug.WithClient())(c.Forecast())
    return &client{genc: genforecast.NewClient(forecast)}
}

// GetForecast 返回给定位置的天气预报。
func (c *client) GetForecast(ctx context.Context, lat, long float64) (*Forecast, error) {
    res, err := c.genc.Forecast(ctx, &genforecast.ForecastPayload{Lat: lat, Long: long})
    if err != nil {
        return nil, err
    }
    l := Location(*res.Location)
    ps := make([]*Period, len(res.Periods))
    for i, p := range res.Periods {
        pval := Period(*p)
        ps[i] = &pval
    }
    return &Forecast{&l, ps}, nil
}
```

下面来分解关键组件：

### 客户端接口

接口定义了消费者将使用的契约：

```go
type Client interface {
    GetForecast(ctx context.Context, lat, long float64) (*Forecast, error)
}
```

这个窄接口仅暴露消费者需要的方法，隐藏了实现细节，更易于维护与测试。

### 领域类型

客户端包定义了自身的领域类型（`Forecast`、`Location`、`Period`），而不是直接暴露代码生成的类型。这样可以：

- 与生成代码的变化解耦
- 更简洁、更聚焦的 API
- 更好地控制对外暴露的数据模型

### 实现

具体实现内部使用 Goa 生成的客户端，同时对外呈现简化后的接口：

```go
type client struct {
    genc *genforecast.Client
}
```

### 工厂函数

`New` 函数以合适的传输层配置实例化客户端：

```go
func New(cc *grpc.ClientConn) Client {
    c := gengrpcclient.NewClient(cc, grpc.WaitForReady(true))
    forecast := debug.LogPayloads(debug.WithClient())(c.Forecast())
    return &client{genc: genforecast.NewClient(forecast)}
}
```

## HTTP 客户端

尽管上面的示例展示了一个 gRPC 客户端，HTTP 客户端遵循相同的模式但初始化方式不同。下面详细看看 HTTP 客户端如何工作。

### Goa 生成的 HTTP 客户端

Goa 会为你的服务生成完整的 HTTP 客户端实现。典型的生成客户端如下所示：

```go
// Client 列出服务端点的 HTTP 客户端。
type Client struct {
    // ForecastDoer 是用于向 forecast 端点发起请求的 HTTP 客户端。
    ForecastDoer goahttp.Doer

    // 配置字段
    RestoreResponseBody bool
    scheme             string
    host               string
    encoder            func(*http.Request) goahttp.Encoder
    decoder            func(*http.Response) goahttp.Decoder
}

// NewClient 为所有服务端实例化 HTTP 客户端。
func NewClient(
    scheme string,
    host string,
    doer goahttp.Doer,
    enc func(*http.Request) goahttp.Encoder,
    dec func(*http.Response) goahttp.Decoder,
    restoreBody bool,
) *Client {
    return &Client{
        ForecastDoer:        doer,
        RestoreResponseBody: restoreBody,
        scheme:             scheme,
        host:               host,
        decoder:            dec,
        encoder:            enc,
    }
}

// Forecast 返回一个向服务 forecast 服务器发起 HTTP 请求的端点。
func (c *Client) Forecast() goa.Endpoint {
    var (
        decodeResponse = DecodeForecastResponse(c.decoder, c.RestoreResponseBody)
    )
    return func(ctx context.Context, v any) (any, error) {
        req, err := c.BuildForecastRequest(ctx, v)
        if err != nil {
            return nil, err
        }
        resp, err := c.ForecastDoer.Do(req)
        if err != nil {
            return nil, goahttp.ErrRequestError("front", "forecast", err)
        }
        return decodeResponse(resp)
    }
}
```

生成的客户端提供：
- 为每个端点提供一个 `Doer` 接口，便于定制 HTTP 客户端行为
- 内置请求编码与响应解码
- 端点级的请求构造器与响应解码器
- 通过 `Doer` 接口支持中间件

### 使用生成的 HTTP 客户端创建你的客户端接口

要用生成的 HTTP 客户端创建一个整洁的客户端接口，可以这样写：

```go
func NewHTTP(doer goa.Doer) Client {
    // 创建生成的 HTTP 客户端
    c := genhttpclient.NewClient(
        "http",                    // scheme
        "weather-service:8080",    // host
        doer,                      // HTTP client
        goahttp.RequestEncoder,    // request encoder
        goahttp.ResponseDecoder,   // response decoder
        false,                     // restore response body
    )

    // 如有需要，使用有效负载日志进行包装
    forecast := debug.LogPayloads(debug.WithClient())(c.Forecast())

    // 返回你的客户端实现
    return &client{genc: genforecast.NewClient(forecast)}
}
```

### 自定义 HTTP 行为

`goa.Doer` 接口（由 `*http.Client` 实现）允许你自定义多种 HTTP 行为：

```go
// 示例：创建具有自定义超时的客户端
httpClient := &http.Client{
    Timeout: 30 * time.Second,
    Transport: &http.Transport{
        MaxIdleConns:        100,
        MaxIdleConnsPerHost: 100,
        IdleConnTimeout:     90 * time.Second,
    },
}

// 使用自定义 HTTP 客户端创建你的客户端
client := NewHTTP(httpClient)
```

## 使用 Mock 进行测试

客户端接口使得在测试中创建 Mock 变得简单。Clue 框架提供了一个名为 `cmg`（Clue Mock Generator）的 Mock 生成器，可为你的客户端接口自动生成 Mock。

### 安装 Mock 生成器

使用以下命令安装 Clue Mock 生成器：

```bash
go install goa.design/clue/mock/cmd/cmg
```

### 生成 Mock

你可以使用 Go 包路径语法为一个或多个包生成 Mock：

```bash
cmg gen goa.design/clue/example/weather/services/...
```

该命令会为指定包中的所有接口生成 Mock 实现。若只针对单个包，可使用：

```bash
cmg gen ./example/weather/services/forecaster
```

### 使用生成的 Mock

生成的 Mock 提供两种定义行为的方式：

1. 常驻 Mock：为某个方法设置一个永久使用的 Mock 函数
2. 顺序 Mock：将多个 Mock 函数加入序列，按调用顺序依次消费

下面是同时展示两种方式的示例：

```go
func TestWeatherService(t *testing.T) {
    // 创建一个新的 mock 客户端
    mock := NewMockClient()

    // 为 GetForecast 设置常驻 Mock
    mock.Set("GetForecast", func(ctx context.Context, lat, long float64) (*Forecast, error) {
        return &Forecast{
            Location: &Location{Lat: lat, Long: long},
            Periods: []*Period{
                {
                    Temperature: 72,
                    Summary:    "Sunny",
                },
            },
        }, nil
    })

    // 测试常驻 Mock
    forecast, err := mock.GetForecast(ctx, 37.7749, -122.4194)
    if err != nil {
        t.Fatal(err)
    }

    // 为不同测试用例添加顺序 Mock
    mock.Add("GetForecast", func(ctx context.Context, lat, long float64) (*Forecast, error) {
        return &Forecast{
            Location: &Location{Lat: lat, Long: long},
            Periods: []*Period{{Temperature: 65, Summary: "Cloudy"}},
        }, nil
    })

    mock.Add("GetForecast", func(ctx context.Context, lat, long float64) (*Forecast, error) {
        return nil, errors.New("service unavailable")
    })

    // 第一次调用返回多云的预报
    forecast1, _ := mock.GetForecast(ctx, 37.7749, -122.4194)

    // 第二次调用返回错误
    forecast2, err := mock.GetForecast(ctx, 37.7749, -122.4194)

    // 序列消费完后，回退到常驻 Mock
    forecast3, _ := mock.GetForecast(ctx, 37.7749, -122.4194)

    // 检查是否所有顺序 Mock 都已消费
    if mock.HasMore() {
        t.Error("Not all sequential mocks were consumed")
    }
}
```

该 Mock 实现通过互斥锁提供对 Mock 函数与序列的线程安全访问，可安全用于并发测试。`Next` 方法在内部处理逻辑：要么返回下一个顺序 Mock，要么在序列耗尽时回退到常驻 Mock。

## 最佳实践

1. 保持接口聚焦：只暴露消费者确实需要的方法
2. 合理处理错误：将传输层特定错误转换为领域内适当的错误
3. 使用上下文：传递 `context` 以支持取消与截止期传播
4. 配置超时：针对你的使用场景设置合理的超时
5. 使用中间件：利用中间件处理日志与指标等跨领域关注点
6. 慎重版本管理：考虑变更对消费者的影响

遵循这些模式，你可以创建易维护、易测试的客户端，促进良好的服务边界并避免服务之间不必要的耦合。