---
title: 覆盖错误序列化
weight: 3
description: "学习如何在 Goa 服务中自定义错误序列化，包括处理验证错误并与组织标准保持一致。"
---

本指南解释如何在 Goa 服务中自定义错误的序列化方式。错误序列化是将错误对象转换为可通过 HTTP 或 gRPC 传输给客户端的格式的过程。这对验证错误尤其重要，因为它们由 Goa 自动生成，使用特定的错误类型，无法在创建时自定义——只能控制其序列化方式。

## 概览

当你的 Goa 服务发生错误时，需要将其转换为客户端能够理解的格式。最常见的需要自定义错误格式的场景是验证错误，它们由 Goa 自动生成并始终使用 `ServiceError` 类型。你无法更改这些错误的创建方式，但可以控制它们在响应中的格式化方式。

需要自定义错误格式的常见场景：

- 探索匹配组织的错误格式标准
  - 你的组织可能对错误响应有特定要求
  - 你可能需要与生态系统中的现有 API 保持一致
  - 你可能需要包含与你的用例相关的附加字段

- 一致地格式化验证错误
  - 处理 Goa 内置的验证错误（`ServiceError`）
  - 以用户友好的格式呈现验证错误
  - 包含与字段相关的验证细节

- 为特定错误类型提供自定义错误响应
  - 不同的错误可能需要不同的格式
  - 某些错误可能需要额外的上下文或详细信息
  - 你可能希望将验证错误与业务逻辑错误区分处理

## 默认错误响应

Goa 在内部将 `ServiceError` 类型用于验证和其他内置错误。该类型包含多个重要字段：

```go
// ServiceError 由 Goa 用于验证和其他内置错误
type ServiceError struct {
    Name      string   // 错误名称（例如："missing_field"）
    ID        string   // 唯一错误 ID
    Field     *string  // 当相关时，导致错误的字段名称
    Message   string   // 可读的错误信息
    Timeout   bool     // 是否为超时错误？
    Temporary bool     // 是否为临时错误？
    Fault     bool     // 是否为服务器故障？
}
```

默认情况下，它会序列化为如下的 JSON 响应：
```json
{
    "name": "missing_field",
    "id": "abc123",
    "message": "email is missing from request body",
    "field": "email"
}
```

## 自定义错误格式化器

要覆盖默认的错误序列化，你需要在创建 HTTP 服务器时提供一个自定义的错误格式化器。

该格式化器必须返回一个实现 `Statuser` 接口的类型，该接口要求实现 `StatusCode()` 方法。此方法决定响应中使用的 HTTP 状态码。

下面是如何实现自定义错误格式化的详细步骤：

```go
// 1. 定义自定义错误响应类型
// 此类型决定错误响应的 JSON 结构
type CustomErrorResponse struct {
    // 机器可读的错误码
    Code    string            `json:"code"`
    Message string            `json:"message"`
    Details map[string]string `json:"details,omitempty"`
}

// 2. 实现 Statuser 接口
// 告诉 Goa 使用哪个 HTTP 状态码
func (r *CustomErrorResponse) StatusCode() int {
    // 你可以在此实现复杂逻辑来决定合适的状态码
    switch r.Code {
    case "VALIDATION_ERROR":
        return http.StatusBadRequest
    case "NOT_FOUND":
        return http.StatusNotFound
    default:
        return http.StatusInternalServerError
    }
}

// 3. 创建格式化器函数
// 此函数将任意错误转换为你的自定义格式
func customErrorFormatter(ctx context.Context, err error) goahttp.Statuser {
    // 处理 Goa 内置的 ServiceError 类型（用于验证错误）
    if serr, ok := err.(*goa.ServiceError); ok {
        switch serr.Name {
        // 常见验证错误
        case goa.MissingField:
            return &CustomErrorResponse{
                Code:    "MISSING_FIELD",
                Message: fmt.Sprintf("The field '%s' is required", *serr.Field),
                Details: map[string]string{
                    "field": *serr.Field,
                },
            }

        case goa.InvalidFieldType:
            return &CustomErrorResponse{
                Code:    "INVALID_TYPE",
                Message: serr.Message,
                Details: map[string]string{
                    "field": *serr.Field,
                },
            }

        case goa.InvalidFormat:
            resp.Details = map[string]string{
                "field": *serr.Field,
                "format_error": serr.Message,
            }

        // 处理其他验证错误
        default:
            return &CustomErrorResponse{
                Code:    "VALIDATION_ERROR",
                Message: serr.Message,
                Details: map[string]string{
                    "error_id": serr.ID,
                    "field":    *serr.Field,
                },
            }
        }
    }

    // 处理其他错误类型
    return &CustomErrorResponse{
        Code:    "INTERNAL_ERROR",
        Message: err.Error(),
    }
}

// 4. 在创建服务器时使用该格式化器
var server *calcsvr.Server
{
    // 创建错误处理器（用于非预期错误）
    eh := errorHandler(logger)
    
    // 使用自定义格式化器创建服务器
    server = calcsvr.New(
        endpoints,    // 服务端点
        mux,          // HTTP 路由器
        dec,          // 请求解码器
        enc,          // 响应编码器
        eh,           // 错误处理器
        customErrorFormatter,  // 你的自定义格式化器
    )
}
```

这将生成如下的 JSON 响应：
```json
{
    "code": "MISSING_FIELD",
    "message": "The field 'email' is required",
    "details": {
        "field": "email"
    }
}
```

## 最佳实践

1. 一致的格式
   - 在整个 API 中使用一致的错误格式
   - 为所有错误响应定义标准结构
   - 包含始终存在的通用字段
   - 详细记录你的错误格式
   
   一致格式示例：
   ```json
   {
       "error": {
           "code": "VALIDATION_ERROR",
           "message": "Invalid input provided",
           "details": {
               "field": "email",
               "reason": "invalid_format",
               "help": "Must be a valid email address"
           },
           "trace_id": "abc-123",
           "timestamp": "2024-01-20T10:00:00Z"
       }
   }
   ```

2. 状态码
   - 选择能够准确反映错误的 HTTP 状态码
   - 保持状态码使用的一致性
   - 记录每个状态码的含义
   - 考虑 HTTP 状态码的标准含义
   
   常见状态码用法：
   ```go
   func (r *CustomErrorResponse) StatusCode() int {
       switch r.Code {
       case "NOT_FOUND":
           return http.StatusNotFound        // 404
       case "VALIDATION_ERROR":
           return http.StatusBadRequest      // 400
       case "UNAUTHORIZED":
           return http.StatusUnauthorized    // 401
       case "FORBIDDEN":
           return http.StatusForbidden       // 403
       case "CONFLICT":
           return http.StatusConflict        // 409
       case "INTERNAL_ERROR":
           return http.StatusInternalServerError // 500
       default:
           return http.StatusInternalServerError
       }
   }
   ```

3. 安全性
   - 切勿在错误中暴露内部系统细节
   - 对所有错误消息进行清理
   - 对内外部 API 使用不同的错误格式
   - 在内部记录详细错误，但对外返回安全消息
   
   安全错误处理示例：
   ```go
   func secureErrorFormatter(ctx context.Context, err error) goahttp.Statuser {
       // 始终记录完整错误细节，便于调试
       log.Printf("Error: %+v", err)
       
       if serr, ok := err.(*goa.ServiceError); ok {
           switch serr.Name {
           // 验证错误源于用户输入，通常可安全暴露
           case goa.MissingField, goa.InvalidFieldType, goa.InvalidFormat,
                goa.InvalidPattern, goa.InvalidRange, goa.InvalidLength:
               return &CustomErrorResponse{
                   Code:    "VALIDATION_ERROR",
                   Message: serr.Message,
                   Details: map[string]string{
                       "field": *serr.Field,
                   }
               }
               
           // 对故障错误要谨慎，可能暴露内部细节
           case "internal_error":
               if serr.Fault {
                   // 内部监控报警，但对外返回通用消息
                   alertMonitoring(serr)
                   return &CustomErrorResponse{
                       Code:    "INTERNAL_ERROR",
                       Message: "An internal error occurred",
                   }
               }
               
           // 对临时错误，提示可重试而非具体原因
           case "service_unavailable":
               if serr.Temporary {
                   return &CustomErrorResponse{
                       Code:    "SERVICE_UNAVAILABLE",
                       Message: "Service temporarily unavailable",
                       Details: map[string]string{
                           "retry_after": "30",
                       },
                   }
               }
           }
       }
       
       // 其他任何错误返回通用错误响应
       // 防止泄露内部实现细节
       return &CustomErrorResponse{
           Code:    "UNEXPECTED_ERROR",
           Message: "An unexpected error occurred",
       }
   }
   ```

4. 兼容性
   - 更改格式时保持向后兼容
   - 若有破坏性变更则为错误格式做版本化
   - 记录所有错误格式的变更
   - 为客户端提供迁移指南
   
   版本化错误格式示例：
   ```go
   func versionedErrorFormatter(ctx context.Context, err error) goahttp.Statuser {
       // 从上下文中检查 API 版本
       version := extractAPIVersion(ctx)
       
       switch version {
       case "v1":
           return formatV1Error(err)
       case "v2":
           return formatV2Error(err)
       default:
           return formatLatestError(err)
       }
   }
   ```

## 结论

自定义错误序列化可以：
- 自定义验证错误的序列化方式
- 以一致的格式呈现错误
- 控制错误细节的暴露
- 合理处理不同类型的错误
- 向客户端提供有意义的错误响应