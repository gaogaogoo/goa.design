---
title: "错误处理"
linkTitle: "错误处理"
weight: 4
description: "学习如何在 Goa 的 gRPC 服务中进行错误处理，包括状态码、错误定义与错误传播"
---

本文介绍如何在使用 Goa 的 gRPC 服务中处理错误。

## 错误类型

### 状态码（Status Codes）

gRPC 使用状态码来表示错误。Goa 提供了这些状态码的内置映射：

```go
Method("divide", func() {
    // 定义可能出现的错误
    Error("division_by_zero")
    Error("invalid_input")

    GRPC(func() {
        // 将错误映射到 gRPC 状态码
        Response(CodeOK)
        Response("division_by_zero", CodeInvalidArgument)
        Response("invalid_input", CodeInvalidArgument)
    })
})
```

常见状态码映射：

| Goa Error | gRPC Status Code | 使用场景 |
|-----------|------------------|----------|
| `not_found` | `CodeNotFound` | 资源不存在 |
| `invalid_argument` | `CodeInvalidArgument` | 输入无效 |
| `internal_error` | `CodeInternal` | 服务器内部错误 |
| `unauthenticated` | `CodeUnauthenticated` | 凭据缺失/无效 |
| `permission_denied` | `CodePermissionDenied` | 权限不足 |

## 错误定义

### 基础错误定义

可在服务或方法级别定义错误：

```go
var _ = Service("users", func() {
    // 服务级错误
    Error("not_found", func() {
        Description("用户未找到")
    })
    Error("invalid_input")

    Method("getUser", func() {
        // 方法特定错误
        Error("profile_incomplete")

        GRPC(func() {
            // 映射所有可能错误
            Response(CodeOK)
            Response("not_found", CodeNotFound)
            Response("invalid_input", CodeInvalidArgument)
            Response("profile_incomplete", CodeFailedPrecondition)
        })
    })
})
```

## 错误实现

### 返回错误

在实现服务方法时，需要处理多种类型的错误。对于输入校验，最好直接在 Goa DSL 中定义这些约束：

```go
var _ = Service("users", func() {
    Method("createUser", func() {
        Payload(func() {
            Field(1, "name", String, "用户全名")
            Field(2, "age", Int, "用户年龄")
            Field(3, "email", String, "用户邮箱")
            
            // 在 DSL 中定义校验规则
            Required("name", "age", "email")
            Minimum("age", 0)
            Maximum("age", 150)
            Pattern("email", "^[^@]+@[^@]+$")
        })
        
        Error("database_error")
        Error("duplicate_email")
        
        GRPC(func() {
            Response(CodeOK)
            Response("database_error", CodeInternal)
            Response("duplicate_email", CodeAlreadyExists)
        })
    })
})
```

对于无法通过 DSL 校验的运行时错误（如数据库冲突、外部服务失败或业务逻辑违规），使用生成的错误构造函数：

```go
func (s *users) CreateUser(ctx context.Context, p *users.CreateUserPayload) (*users.User, error) {
    // 检查邮箱是否已存在
    exists, err := s.db.EmailExists(ctx, p.Email)
    if err != nil {
        // 包装数据库错误
        return nil, users.MakeDatabaseError(fmt.Errorf("failed to check email: %w", err))
    }
    if exists {
        // 返回业务逻辑错误
        return nil, users.MakeDuplicateEmail(fmt.Sprintf("email %s is already registered", p.Email))
    }
    
    // 在数据库中创建用户
    user, err := s.db.CreateUser(ctx, p)
    if err != nil {
        return nil, users.MakeDatabaseError(fmt.Errorf("failed to create user: %w", err))
    }
    
    return user, nil
}
```

### 流式中的错误处理

在处理 gRPC 流式方法时，错误处理更为复杂，需要同时处理与流相关的错误和业务错误。以下示例展示如何在流式服务方法中处理不同类型的错误：

```go
func (s *service) StreamData(stream service.StreamDataServerStream) error {
    for {
        data, err := getData()
        if err != nil {
            if isRateLimitError(err) {
                return service.MakeRateLimitExceeded(err)
            }
            return service.MakeInternalError(err)
        }

        if err := stream.Send(data); err != nil {
            if isStreamInterrupted(err) {
                return service.MakeStreamInterrupted(err)
            }
            return err
        }
    }
}
```

## 错误处理模式

### 错误包装（Wrapping Errors）

当处理来自外部包或底层组件的错误时，需在保留适当的 gRPC 状态码的同时提供上下文。以下展示如何在保留错误链的同时进行错误包装：

```go
func (s *service) ProcessData(ctx context.Context, p *service.ProcessDataPayload) (*service.Result, error) {
    result, err := s.processor.Process(p.Data)
    if err != nil {
        switch {
        case errors.Is(err, ErrInvalidFormat):
            return nil, service.MakeInvalidInput(fmt.Errorf("invalid data format: %w", err))
        case errors.Is(err, ErrProcessingFailed):
            return nil, service.MakeInternalError(fmt.Errorf("processing failed: %w", err))
        default:
            return nil, service.MakeInternalError(err)
        }
    }
    return result, nil
}
```

### 错误恢复（Error Recovery）

在长时间运行或批处理操作中，通常需要实现错误恢复机制以处理瞬时失败。以下示例展示如何实现带重试逻辑与错误统计的批处理：

```go
func (s *service) ProcessBatch(stream service.ProcessBatchServerStream) error {
    var processed, failed int

    for {
        payload, err := stream.Recv()
        if err == io.EOF {
            // 发送最终状态
            return stream.SendAndClose(&service.BatchResult{
                Processed: processed,
                Failed:    failed,
            })
        }
        if err != nil {
            return service.MakeStreamInterrupted(err)
        }

        // 带错误恢复的处理
        if err := s.processWithRetry(payload); err != nil {
            failed++
            // 记录错误但继续处理
            log.Printf("Failed to process item: %v", err)
            continue
        }
        processed++
    }
}

func (s *service) processWithRetry(payload *service.Payload) error {
    var err error
    for retries := 0; retries < 3; retries++ {
        err = s.process(payload)
        if err == nil {
            return nil
        }
        // 仅对瞬时错误进行重试
        if !isTransientError(err) {
            return err
        }
        time.Sleep(time.Second * time.Duration(retries+1))
    }
    return err
}
```

## 最佳实践

### 错误设计指南

1. 定义 API 级通用错误
   
   在 API 层定义通用错误以确保跨服务的一致性并实现复用，减少重复并统一错误处理：

   ```go
   var _ = API("myapi", func() {
       // API 共享的通用错误
       Error("unauthorized", func() {
           Description("请求需要认证")
       })
       Error("not_found")  // 使用默认 ErrorResult 类型
       Error("validation_error", ValidationError, "校验失败")

       // 定义通用的 HTTP 映射
       HTTP(func() {
           Response("unauthorized", StatusUnauthorized)
           Response("not_found", StatusNotFound)
           Response("validation_error", StatusBadRequest)
       })
   })
   ```

2. 将错误映射到恰当的状态码
   
   基于错误的语义意义选择合适的 gRPC 状态码，而不只是其来源：

   ```go
   var _ = Service("users", func() {
       Method("createUser", func() {
           Error("invalid_input")
           Error("email_taken")
           Error("database_error")

           GRPC(func() {
               // 映射到语义化的 gRPC 状态码
               Response("invalid_input", CodeInvalidArgument)
               Response("email_taken", CodeAlreadyExists)
               Response("database_error", CodeInternal)  // 隐藏实现细节
           })
       })
   })
   ```

3. 文档化错误条件
   
   为每种错误类型提供清晰的描述与示例，帮助 API 使用者正确处理错误：

   ```go
   var _ = Service("payment", func() {
       Method("processPayment", func() {
           Error("insufficient_funds", func() {
               Description("账户余额不足，无法完成交易")
               Example(func() {
                   Value(Val{
                       "message": "Insufficient funds",
                       "balance": 50.00,
                       "required": 100.00,
                   })
               })
           })
           
           Error("card_declined", func() {
               Description("支付卡被提供方拒绝")
               Meta("docs:example", "卡片过期或无效")
           })
       })
   })
   ```

4. 错误层级与继承
   
   以层级化结构组织错误，便于同时进行通用与特定的错误处理：

   ```go
   var _ = Service("orders", func() {
       // 适用于所有方法的服务级错误
       Error("database_error")

       Method("placeOrder", func() {
           // 方法级特定错误
           Error("inventory_unavailable")
           Error("payment_failed")
           
           // 仍可使用服务级错误
           // 无需重新定义 database_error
       })
   })
   ```

遵循以上最佳实践可在整个 API 中保持一致性，同时为客户端提供清晰的错误信息。这有助于实现高效的错误处理并保持文档清晰易用；此外还能在隐藏实现细节的同时向使用者暴露有意义的错误信息。跨方法与服务复用错误也能有效减少 API 设计中的重复。