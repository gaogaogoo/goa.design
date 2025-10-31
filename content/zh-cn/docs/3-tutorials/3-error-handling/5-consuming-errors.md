---
title: 生成与消费错误
linkTitle: 生成错误
weight: 5
description: "在 Goa 服务中生成与处理错误的指南，包括使用生成的助手函数以及在客户端侧处理错误。"
---

在基于 Goa 的服务中，生成与消费错误是错误处理的关键环节。本节详细介绍如何在服务实现中生成错误，以及如何在客户端侧有效处理这些错误。

## 生成错误

### 使用生成的助手函数

Goa 会为已定义的错误生成助手函数，从而简化创建标准化错误响应的过程。这些助手函数确保错误根据服务设计保持一致并正确格式化。助手函数命名为 `Make<ErrorName>`，其中 `<ErrorName>` 为在 DSL 中定义的错误名称。它们会根据服务设计初始化错误字段（例如错误是否为超时、临时等）。

给定如下服务设计：

```go
var _ = Service("divider", func() {
    Method("IntegralDivide", func() {
        Payload(IntOperands)
        Result(Int)
        Error("DivByZero", ErrorResult, "Divisor cannot be zero")
        Error("HasRemainder", ErrorResult, "Remainder is not zero")
        HTTP(func() {
            POST("/divide")
            Response(StatusOK)
            Response("DivByZero", StatusBadRequest)
            Response("HasRemainder", StatusUnprocessableEntity)
        })
    })
})

 var IntOperands = Type("IntOperands", func() {
    Attribute("dividend", Int, "Dividend")
    Attribute("divisor", Int, "Divisor")
    Required("dividend", "divisor")
 })
```

示例实现：

```go
//...
func (s *dividerSvc) IntegralDivide(ctx context.Context, p *divider.IntOperands) (int, error) {
    if p.Divisor == 0 {
        return 0, gendivider.MakeDivByZero(fmt.Errorf("divisor cannot be zero"))
    }
    if p.Dividend%p.Divisor != 0 {
        return 0, gendivider.MakeHasRemainder(fmt.Errorf("remainder is %d", p.Dividend%p.Divisor))
    }
    return p.Dividend / p.Divisor, nil
}
```

在此示例中：

- `gendivider` 包由 Goa 生成（位于 `gen/divider`）。
- `MakeDivByZero` 函数创建标准化的 `DivByZero` 错误。
- `MakeHasRemainder` 函数创建标准化的 `HasRemainder` 错误。

这些助手函数会根据服务设计初始化错误字段，确保错误被正确序列化并映射到传输特定的状态码（在本例中，`DivByZero` 为 400，`HasRemainder` 为 422）。

### 使用自定义错误类型

对于更复杂的错误场景，可能需要定义自定义错误类型。不同于默认的 `ErrorResult`，自定义错误类型允许你包含与错误相关的额外上下文信息。

给定如下服务设计：

```go
var _ = Service("divider", func() {
    Method("IntegralDivide", func() {
        Payload(IntOperands)
        Result(Int)
        Error("DivByZero", DivByZero, "Divisor cannot be zero")
        HTTP(func() {
            POST("/divide")
            Response(StatusOK)
            Response("DivByZero", StatusBadRequest)
        })
    })
})

var DivByZero = Type("DivByZero", func() {
    Description("DivByZero is the error returned when using value 0 as divisor.")
    Field(1, "name", String, "Error name", func() {
        Meta("struct:error:name")
    })
    Field(2, "message", String, "Error message for division by zero.")
    Field(3, "dividend", Int, "Dividend that was used in the operation.")
    Required("name", "message", "dividend")
})
```

示例实现：

```go
func (s *dividerSvc) IntegralDivide(ctx context.Context, p *divider.IntOperands) (int, error) {
    if p.Divisor == 0 {
        return 0, &gendivider.DivByZero{Name: "DivByZero", Message: "divisor cannot be zero", Dividend: p.Dividend}
    }
    // 其他逻辑…
}
```

在此示例中：

- `DivByZero` 结构体是服务设计中定义的自定义错误类型。
- 通过返回 `DivByZero` 的实例，你可以提供自定义且详尽的错误信息。
- 注意：使用自定义错误类型时，需确保错误结构体包含 `Meta("struct:error:name")` 属性，以便 Goa 正确映射错误。该属性必须设置为在服务设计中定义的错误名称。

## 消费错误

在客户端侧处理错误与在服务端侧生成错误同样重要。正确的错误处理可确保客户端对不同错误场景做出适当响应。

### 处理默认错误

当使用默认的 `ErrorResult` 类型时，客户端侧错误是 `goa.ServiceError` 的实例。你可以检查错误类型，并依据错误名称进行处理。

示例：

```go
res, err := client.Divide(ctx, payload)
if err != nil {
    if serr, ok := err.(*goa.ServiceError); ok {
        switch serr.Name {
        case "HasRemainder":
            // 处理存在余数的错误
        case "DivByZero":
            // 处理除数为零的错误
        default:
            // 处理未知错误
        }
    }
}
```

### 处理自定义错误

当使用自定义错误类型时，客户端侧错误是相应生成的 Go 结构体实例。你可以对错误进行类型断言，并据此处理。

示例：

```go
res, err := client.Divide(ctx, payload)
if err != nil {
    if dbz, ok := err.(*gendivider.DivByZero); ok {
        // 处理除数为零的错误
    }
}
```

## 总结

通过高效地生成与消费错误，你可以确保基于 Goa 的服务对失败进行清晰且一致的沟通。对标准错误使用生成的助手函数，对更复杂场景使用自定义错误类型，可实现灵活且健壮的错误处理。完善的客户端侧错误处理进一步提升 API 的可靠性与可用性，为用户提供有意义的反馈并促使采取适当的纠正措施。

### 测试错误处理

测试错误处理需要对错误条件与类型进行仔细校验。以下展示如何使用 Clue 的
[mock 包](https://github.com/goadesign/clue/tree/main/mock) 有效测试错误处理：

```go
// 引入 Clue 的 mock 包
import (
    "github.com/goadesign/clue/mock"
)

// 使用 Clue 的 mock 包进行模拟实现
// 展示如何基于 Clue 正确构造一个 mock
type mockDividerService struct {
    *mock.Mock // 嵌入 Clue 的 Mock 类型
}

// 使用 Clue 的 Next 模式实现 IntegralDivide 的模拟
func (m *mockDividerService) IntegralDivide(ctx context.Context, p *divider.IntOperands) (int, error) {
    if f := m.Next("IntegralDivide"); f != nil {
        return f.(func(context.Context, *divider.IntOperands) (int, error))(ctx, p)
    }
    return 0, errors.New("unexpected call to IntegralDivide")
}

func TestIntegralDivide(t *testing.T) {
    // 使用 Clue 的 mock 包创建模拟服务
    // 展示 Clue 在测试中的强大模拟能力
    svc := &mockDividerService{mock.New()}
    
    tests := []struct {
        name     string
        setup    func(*mockDividerService)
        dividend int
        divisor  int
        wantErr  string
    }{
        {
            name: "division by zero",
            setup: func(m *mockDividerService) {
                m.Set("IntegralDivide", func(ctx context.Context, p *divider.IntOperands) (int, error) {
                    if p.Divisor == 0 {
                        return 0, gendivider.MakeDivByZero(fmt.Errorf("divisor cannot be zero"))
                    }
                    return p.Dividend / p.Divisor, nil
                })
            },
            dividend: 10,
            divisor:  0,
            wantErr:  "divisor cannot be zero",
        },
        {
            name: "has remainder",
            setup: func(m *mockDividerService) {
                m.Set("IntegralDivide", func(ctx context.Context, p *divider.IntOperands) (int, error) {
                    if p.Dividend%p.Divisor != 0 {
                        return 0, gendivider.MakeHasRemainder(fmt.Errorf("remainder is %d", p.Dividend%p.Divisor))
                    }
                    return p.Dividend / p.Divisor, nil
                })
            },
            dividend: 10,
            divisor:  3,
            wantErr:  "remainder is 1",
        },
        {
            name: "successful division",
            setup: func(m *mockDividerService) {
                m.Set("IntegralDivide", func(ctx context.Context, p *divider.IntOperands) (int, error) {
                    return p.Dividend / p.Divisor, nil
                })
            },
            dividend: 10,
            divisor:  2,
            wantErr:  "",
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 为每个测试创建一个全新的 mock
            mock := &mockDividerService{mock.New()}
            if tt.setup != nil {
                tt.setup(mock)
            }
            
            // 构造入参
            p := &divider.IntOperands{
                Dividend: tt.dividend,
                Divisor:  tt.divisor,
            }
            
            // 调用服务
            result, err := mock.IntegralDivide(context.Background(), p)
            
            // 校验错误行为
            if tt.wantErr != "" {
                if err == nil {
                    t.Errorf("expected error containing %q, got nil", tt.wantErr)
                } else if !strings.Contains(err.Error(), tt.wantErr) {
                    t.Errorf("expected error containing %q, got %q", tt.wantErr, err.Error())
                }
            } else if err != nil {
                t.Errorf("unexpected error: %v", err)
            }
            
            // 成功用例校验结果
            if tt.wantErr == "" && result != tt.dividend/tt.divisor {
                t.Errorf("got result %d, want %d", result, tt.dividend/tt.divisor)
            }
            
            // 校验所有期望的调用是否已执行
            if mock.HasMore() {
                t.Error("not all expected operations were performed")
            }
        })
    }
}
