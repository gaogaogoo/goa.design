---
title: "理解插件"
linkTitle: "插件"
description: "一份全面的指南，帮助你理解、使用并创建 Goa 插件，用于扩展与定制代码生成"
weight: 1
---

Goa 插件用于扩展和定制你的 API 功能。无论你需要添加限流、集成监控工具，还是生成不同语言的代码，插件都提供了一种灵活方式来增强 Goa 的能力。本指南将从基础开始，逐步讲解如何理解与创建插件，最终走向进阶用法。

## 插件基础

在深入技术细节之前，先了解插件能做什么以及它们如何与 Goa 协同工作。一个插件通常提供三项主要能力：

首先，插件会向 Goa 的 DSL 添加新的设计函数。这些函数让用户能够在 API 设计中配置附加功能。例如，一个限流插件可能会添加 `RateLimit()` 和 `Burst()` 等函数，供用户配置请求限制：

```go
var _ = Service("calculator", func() {
    // 使用插件的 DSL 函数配置限流
    RateLimit(100, func() {      // 允许 100 次请求...
        Period("1m")             // ...每分钟
        Burst(20)                // ...最大突发 20 次
    })
    
    Method("add", func() {
        // 常规 Goa DSL 继续
        Payload(func() {
            Field(1, "a", Int)
            Field(2, "b", Int)
        })
        Result(Int)
    })
})
```

其次，插件会创建自定义的表达式来存储和验证这些配置。这些表达式与 Goa 的 API 设计内部表示集成，确保所有设置有效且一致。

第三，插件会基于其配置生成附加代码。这可能包括中间件、辅助函数或配置文件。例如，我们的限流插件会生成强制执行已配置限制的中间件代码：

```go
// 生成的限流中间件
type calculatorRateMiddleware struct {
    limiter *rate.Limiter
    next    Service
}

func NewRateMiddleware() Middleware {
    // 创建限流器：每分钟 100 次请求，突发 20
    limiter := rate.NewLimiter(rate.Every(time.Minute), 100)
    limiter.SetBurst(20)
    
    return func(next Service) Service {
        return &calculatorRateMiddleware{
            limiter: limiter,
            next:    next,
        }
    }
}
```

生成的代码与 Goa 的标准输出无缝集成，用户只需极少的设置即可使用。

## 基础：Goa 设计语言

要创建高效的插件，你需要理解 Goa 的设计语言是如何工作的。虽然它看起来像常规的 Go 代码，但 Goa 的 DSL（领域特定语言）提供了一种结构化方式来定义服务、方法以及它们的属性。

下面是一个简单的 Goa 设计语言示例：

```go
var _ = Service("calculator", func() {
    Description("一个基础的计算器服务")
    
    Method("add", func() {
        // 定义输入参数
        Payload(func() {
            Field(1, "a", Int, "第一个加数")
            Field(2, "b", Int, "第二个加数")
        })
        // 定义输出
        Result(Int, "a 与 b 的和")
    })
})
```

这段代码定义了一个带加法方法的计算器服务。像 `Service()`、`Method()` 和 `Field()` 这样的函数属于 Goa 的 DSL。当 Goa 处理该设计时，它会创建一个称为“表达式树”的内部表示：

```
Service("calculator")
    └── Method("add")
        ├── Payload
        │   ├── Field("a")
        │   └── Field("b")
        └── Result(Int)
```

## 创建新的 DSL 函数

在构建插件时，你需要创建用户能够在其 API 设计中调用的 DSL 函数。这些函数通常需要通过自定义表达式来存储并验证配置。我们一步步来理解这个过程。

### 理解表达式

表达式在 Goa 中表示你的 API 设计的一部分。当用户编写 DSL 函数时，这些函数会创建并配置表达式。工作原理如下：

```go
var _ = Service("calculator", func() {    // 创建 ServiceExpr
    Method("add", func() {                // 创建 MethodExpr
        Payload(func() {                  // 创建 PayloadExpr
            Field(1, "x", Int)            // 配置载荷
        })
    })
})
```

对于你的插件，你将定义自定义表达式来存储配置。例如，一个限流插件可以定义：

```go
// RateExpr 存储服务的限流配置
type RateExpr struct {
    Service  *expr.ServiceExpr // 应用的服务
    Requests int               // 每个时间段的请求数
    Period   string            // 时间段（例如 "1m"）
    Burst    int               // 最大突发大小
}
```

### 表达式接口

你的表达式必须实现特定接口以配合 Goa 的设计处理。最基本的要求是 `Expression` 接口，用于提供标识：

```go
// 所有表达式都需要实现
type Expression interface {
    // EvalName 返回用于错误消息的描述性名称
    EvalName() string    // 例如："service calculator"
}
```

根据需要，你可以实现其他接口：

```go
// 可选 - 如果你的表达式有子 DSL 函数则实现
type Source interface {
    DSL() func()        // 返回要执行的 DSL 函数
}

// 可选 - 如果需要在验证前准备数据则实现
type Preparer interface {
    Prepare()           // 在准备阶段调用
}

// 可选 - 如果你的表达式需要验证则实现
type Validator interface {
    Validate() error    // 在验证阶段调用
}

// 可选 - 如果需要在验证后进行后处理则实现
type Finalizer interface {
    Finalize()          // 在最终化阶段调用
}
```

下面的完整示例展示了这些接口如何在我们的限流插件中协同工作：

```go
// RateExpr 表示限流配置
type RateExpr struct {
    Service  *expr.ServiceExpr
    Requests int
    Period   string
    Burst    int
    
    // 内部状态
    prepared bool
    dsl      func()
}

// 必需：实现 Expression 接口
func (r *RateExpr) EvalName() string {
    return fmt.Sprintf("服务 %q 的限流", r.Service.Name)
}

// 可选：如果表达式有子 DSL 则实现 Source
func (r *RateExpr) DSL() func() {
    return r.dsl  // 返回用于配置该表达式的 DSL 函数
}

// 可选：实现 Preparer 做初始化
func (r *RateExpr) Prepare() {
    if !r.prepared {
        // 设置合理的默认值
        if r.Period == "" {
            r.Period = "1m"
        }
        if r.Burst == 0 {
            r.Burst = r.Requests
        }
        r.prepared = true
    }
}

// 可选：实现 Validator 做验证
func (r *RateExpr) Validate() error {
    errors := new(eval.ValidationErrors)

    if r.Requests <= 0 {
        errors.Add(r, "requests 必须为正数，得到 %d", r.Requests)
    }

    if _, err := time.ParseDuration(r.Period); err != nil {
        errors.Add(r, "无效的 period %q: %s", r.Period, err)
    }

    if len(errors.Errors) > 0 {
        return errors
    }
    return nil
}

// 可选：实现 Finalizer 做后处理
func (r *RateExpr) Finalize() {
    // 在验证后执行最终处理
}
```

### 创建 DSL 函数

定义好表达式后，你可以创建用户在设计中调用的 DSL 函数。这些函数会创建并配置你的表达式：

```go
// RateLimit 是创建并配置 RateExpr 的 DSL 函数
func RateLimit(requests int, fn func()) {
    if current := eval.Current(); current != nil {
        if svc, ok := current.(*expr.ServiceExpr); ok {
            // 创建我们的表达式
            rate := &RateExpr{
                Service:  svc,
                Requests: requests,
                dsl:     fn,
            }
            // 执行 DSL 函数以进行配置
            if eval.Execute(fn, rate) {
                // 存储到服务的元数据中
                svc.Meta = append(svc.Meta, rate)
            }
        }
    }
}
```

此模式带来多个好处：
1. 类型安全的配置存储
2. 在设计处理过程中进行验证
3. 当出现问题时提供清晰的错误消息
4. 与 Goa 的表达式树集成

## Eval 包：Goa 的插件引擎

理解了表达式和 DSL 函数后，来看 Goa 如何处理它们。`eval` 包是驱动 Goa 插件系统的引擎，它以四个阶段处理设计：

1. 初始执行：首先运行你编写的所有 DSL 函数，构建表示 API 设计的表达式树。
2. 准备阶段：随后准备表达式，处理类型继承解析和嵌套结构扁平化等任务。
3. 验证阶段：然后验证所有表达式以确保它们遵循 DSL 规则并具有逻辑合理性。
4. 最终化阶段：最后执行必要的清理，例如设置默认值或解析设计不同部分之间的引用。

来看一个限流插件的实际过程：

```go
// 在你的设计文件中：
var _ = Service("api", func() {
    RateLimit(100, func() {     // 创建 RateExpr
        Period("1m")            // 配置 period
        Burst(20)               // 设置突发大小
    })
})

// 幕后发生的事情：

// 1. 初始执行
// - 创建一个 requests=100 的 RateExpr
// - 执行 DSL 函数，设置 period="1m" 和 burst=20
// - 将 RateExpr 连接到 ServiceExpr

// 2. 准备阶段
func (r *RateExpr) Prepare() {
    if !r.prepared {
        // 若未指定则设置默认 period
        if r.Period == "" {
            r.Period = "1m"
        }
        // 若未指定则设置默认 burst
        if r.Burst == 0 {
            r.Burst = r.Requests
        }
        r.prepared = true
    }
}

// 3. 验证阶段
func (r *RateExpr) Validate() error {
    errors := new(eval.ValidationErrors)
    
    // 验证 requests
    if r.Requests <= 0 {
        errors.Add(r, "requests 必须为正数，得到 %d", r.Requests)
    }
    
    // 验证 period 格式
    if _, err := time.ParseDuration(r.Period); err != nil {
        errors.Add(r, "无效的 period %q: %s", r.Period, err)
    }
    
    // 验证突发大小
    if r.Burst < 0 {
        errors.Add(r, "burst 必须为非负数，得到 %d", r.Burst)
    }
    
    if len(errors.Errors) > 0 {
        return errors
    }
    return nil
}

// 4. 最终化阶段
func (r *RateExpr) Finalize() {
    // 将 period 规范化为 duration
    if duration, err := time.ParseDuration(r.Period); err == nil {
        r.normalizedPeriod = duration
    }
    
    // 确保 burst 不超过 requests
    if r.Burst > r.Requests {
        r.Burst = r.Requests
    }
}
```

该示例展示了 eval 包如何编排设计处理：
1. 在初始执行阶段按顺序处理 DSL 函数：
   - 首先 `RateLimit(100)` 创建基础表达式
   - 然后 `Period("1m")` 和 `Burst(20)` 对其进行配置
   - 将表达式附加到父服务

2. 在准备阶段设置默认值：
   - 若未指定则默认 period 为 "1m"
   - 若未指定则默认 burst 与 requests 相同
   - 标记表达式已准备，避免重复工作

3. 在验证阶段检查所有规则：
   - 确保 requests 为正
   - 验证 period 格式
   - 检查 burst 非负
   - 在报告前收集所有错误

4. 最终化阶段：
   - 将 period 规范化为 duration
   - 调整 burst 不超过 requests
   - 解析任何交叉引用

此流程确保在代码生成开始前：
- 所有表达式已完全配置
- 所有数值已验证
- 所有交叉引用已解析
- 所有默认值均正确设置

### Eval 包的关键函数

为高效使用该系统，你将会用到 `eval` 包中的若干关键函数。下面逐一说明：

#### 1. Current() 表达式

`Current()` 返回在 DSL 执行中当前被处理的表达式。这对于具备上下文感知的 DSL 函数尤为关键：

```go
// 获取当前正在处理的表达式
func Current() Expression

// 在 DSL 函数中的示例用法：
func RateLimit(requests int) {
    // 获取当前表达式（应为 Service）
    if current := eval.Current(); current != nil {
        // 检查是否处于正确的上下文
        if svc, ok := current.(*expr.ServiceExpr); ok {
            // 我们在 Service 定义中
            // ... 为该服务配置限流
        } else {
            // 上下文错误 - RateLimit 必须在 Service 内使用
            eval.ReportError("RateLimit 必须在 Service 内使用")
        }
    }
}
```

该函数在如下场景特别有用：
- 验证你的 DSL 函数的使用上下文
- 访问包含你的配置的父表达式
- 变更父表达式（例如附加子表达式）

#### 2. Execute(fn func(), def Expression) bool

`Execute` 在特定表达式的上下文中运行一个 DSL 函数。它负责设置与清理执行上下文：

```go
// 在表达式上下文中执行一个 DSL 函数
// 返回执行是否成功
func Execute(fn func(), def Expression) bool

// 示例用法：
func RateLimit(requests int, fn func()) {
    if current := eval.Current(); current != nil {
        if svc, ok := current.(*expr.ServiceExpr); ok {
            // 创建我们的配置表达式
            rate := &RateExpr{
                Service:  svc,
                Requests: requests,
            }
            
            // 在我们的表达式上下文中执行 DSL 函数
            if eval.Execute(fn, rate) {
                // DSL 执行成功，存储配置
                svc.Meta = append(svc.Meta, rate)
            }
            // 注意：若 Execute 返回 false，错误已被报告
        }
    }
}

// 用法如下：
var _ = Service("api", func() {
    RateLimit(100, func() {
        Period("1m")
        Burst(20)
    })
})
```

关于 `Execute` 的要点：
- 它会临时将提供的表达式设为当前表达式
- 在该上下文中运行 DSL 函数
- 完成后恢复之前的上下文
- 若执行过程中发生错误则返回 false

#### 3. 错误报告函数

`eval` 包提供了若干函数用于在 DSL 执行期间报告错误：

##### ReportError(fm string, vals ...any)

`ReportError` 用于报告 DSL 执行期间的错误。它会使用提供的格式字符串与值构建错误消息，并自动附加当前表达式上下文：

```go
// 在 DSL 执行期间报告错误
func ReportError(fm string, vals ...any)
```

示例用法：

```go
func Period(duration string) {
    if rate, ok := eval.Current().(*RateExpr); ok {
        if _, err := time.ParseDuration(duration); err != nil {
            eval.ReportError(
                "无效的 duration %q：必须为合法时长（如 '1m', '1h')",
                duration)
        }
        rate.Period = duration
    }
}
```

当在如下设计中使用：

```go
var _ = Service("orders", func() {
    RateLimit(100, func() {
        Period("2x")  // 非法时长
    })
})

// 错误输出如下：
// /path/to/design/design.go:42: 服务 "orders" 的限流：无效的 duration "2x"：必须为合法时长（如 '1m', '1h')
//
// 错误消息包括：
// - 错误发生的文件与行号
// - 表达式上下文（"服务 'orders' 的限流"）
// - 具体的错误信息
// - 有助于修复问题的提示
```

##### IncompatibleDSL()

`IncompatibleDSL` 用于报告某个 DSL 函数被用于错误的上下文。这是一个针对常见错误场景的便捷函数：

```go
// 当 DSL 函数在错误上下文中被调用（例如在 Service 中调用 Params）时，应调用 IncompatibleDSL。
func IncompatibleDSL() {
    ReportError("无效的 %s 用法", caller())
}
```

在你的 DSL 函数中使用方式如下：

```go
func Burst(n int) {
    if rate, ok := eval.Current().(*RateExpr); ok {
        rate.Burst = n
    } else {
        // Burst() 在 RateLimit 块之外被调用
        eval.IncompatibleDSL()
    }
}
```

当在非法上下文中使用，如：

```go
var _ = Service("orders", func() {
    Burst(20)  // 错误：在 RateLimit 外调用
})
```

将产生如下错误消息：

```
/path/to/design/design.go:42: 无效的 Burst 用法
```

错误包含：
- 错误使用 DSL 函数的文件与行号
- 被错误使用的函数名称

该函数特别适用于：
- 某个 DSL 函数必须在特定父级内使用（例如 `Burst` 必须在 `RateLimit` 内）
- 当前表达式不是期望类型
- 函数需要的上下文不存在

#### 4. Register(r Root) error

`Register` 会向 DSL 添加一个新的根表达式。根表达式是 DSL 的入口并控制执行顺序：

```go
// 注册一个新的根表达式
func Register(r Root) error

// 根表达式示例：
type RateLimitRoot struct {
    *expr.RootExpr
    // 插件特有的附加字段
}

// 实现 Root 接口
func (r *RateLimitRoot) WalkSets(w eval.SetWalker) {
    // 定义表达式求值顺序
    w.Walk(r.Services)
}

func (r *RateLimitRoot) DependsOn() []eval.Root {
    // 指定对其他插件的依赖
    return []eval.Root{
        &security.Root{},
    }
}

func (r *RateLimitRoot) Packages() []string {
    // 返回生成代码所需的导入路径
    return []string{
        "golang.org/x/time/rate",
    }
}

// 在插件的 init 函数中注册根表达式
func init() {
    root := &RateLimitRoot{
        RootExpr: &expr.RootExpr{},
    }
    if err := eval.Register(root); err != nil {
        panic(err) // 或合理处理错误
    }
}
```

根表达式的要点：
- 通过 `WalkSets` 控制 DSL 执行顺序
- 声明对其他插件的依赖
- 指定生成代码所需的包
- 通常在包初始化期间注册

这些函数共同提供了一个健壮的 DSL 执行框架：
1. `Register` 设置你的插件根表达式
2. `Current` 与 `Execute` 管理执行上下文
3. `ReportError` 与 `IncompatibleDSL` 处理错误场景
4. 根表达式控制整体执行流程

## 创建你的第一个插件

将所学付诸实践，一步一步地创建一个限流插件，并解释每个组件及其在插件系统中的作用。

### 搭建项目

首先，为你的插件创建一个新目录，结构如下：

```
ratelimit/
├── dsl/
│   ├── dsl.go      # 你的 DSL 函数（RateLimit, Period 等）
│   └── types.go    # 用于存储配置的表达式类型
├── generate.go     # 代码生成逻辑
├── plugin.go       # 插件注册
└── templates/      # 用于生成的代码模板
    └── middleware.go.tmpl
```

该结构实现关注点分离：
- `dsl` 包包含用户在设计中调用的函数
- 表达式类型用于存储与验证配置
- 代码生成逻辑产生实际的中间件
- 模板定义生成代码的外观

### 第一步：创建 DSL

从 `dsl/dsl.go` 中的 DSL 函数开始。这些函数是用户在 API 设计中调用的：

```go
package dsl

import (
    "goa.design/goa/v3/eval"
    "goa.design/goa/v3/expr"
)

// RateLimit 为服务定义限流配置。
// 示例：
//
//    var _ = Service("calculator", func() {
//        RateLimit(100, func() {  // 100 次请求...
//            Period("1m")         // ...每分钟
//            Burst(20)            // ...最大突发 20
//        })
//    })
func RateLimit(requests int, fn func()) {
    // 获取当前正在处理的表达式
    if current := eval.Current(); current != nil {
        // 检查是否处于 Service 上下文
        if svc, ok := current.(*expr.ServiceExpr); ok {
            // 创建限流配置
            rate := &RateExpr{
                Service:  svc,
                Requests: requests,
            }
            // 执行 DSL 函数以配置限流
            if eval.Execute(fn, rate) {
                // 将配置存储到服务的元数据中
                svc.Meta = append(svc.Meta, rate)
            }
        } else {
            eval.ReportError("RateLimit 必须在 Service 内使用")
        }
    }
}

// Period 设置限流的时间窗口。
// 合法的时间单位包括 "s"、"m"、"h"（秒、分、小时）。
func Period(duration string) {
    // 获取当前表达式（应该是我们的 RateExpr）
    if rate, ok := eval.Current().(*RateExpr); ok {
        rate.Period = duration
    } else {
        eval.IncompatibleDSL()
    }
}

// Burst 设置允许超过限流的最大请求数。
func Burst(n int) {
    if rate, ok := eval.Current().(*RateExpr); ok {
        rate.Burst = n
    } else {
        eval.IncompatibleDSL()
    }
}
```

### 第二步：定义表达式类型

接着在 `dsl/types.go` 中定义用于存储配置的类型：

```go
package dsl

import (
    "time"
    "goa.design/goa/v3/eval"
    "goa.design/goa/v3/expr"
)

// RateExpr 存储服务的限流配置。
type RateExpr struct {
    // 限流所应用的服务
    Service *expr.ServiceExpr
    // 允许的请求数
    Requests int
    // 时间段（例如 "1m"、"1h"）
    Period string
    // 最大突发大小
    Burst int
}

// EvalName 返回用于错误消息的描述性名称
func (r *RateExpr) EvalName() string {
    return "服务 " + r.Service.Name + " 的限流"
}

// Validate 确保配置合法
func (r *RateExpr) Validate() error {
    errors := new(eval.ValidationErrors)
    
    // Requests 必须为正数
    if r.Requests <= 0 {
        errors.Add(r, "requests 必须为正数，得到 %d", r.Requests)
    }
    
    // Period 必须是合法时长
    if _, err := time.ParseDuration(r.Period); err != nil {
        errors.Add(r, "无效的 period 格式 %q，请使用 's'、'm' 或 'h'", r.Period)
    }
    
    // Burst 必须为非负
    if r.Burst < 0 {
        errors.Add(r, "burst 必须为非负数，得到 %d", r.Burst)
    }
    
    if len(errors.Errors) > 0 {
        return errors
    }
    return nil
}
```

### 第三步：实现代码生成

`generate.go` 中的代码生成函数会创建实际的中间件。当你运行 `goa gen` 时，Goa 会调用每个插件的 `Generate` 函数来生成所需的代码文件。原理如下：

```go
// Generate 在代码生成期间由 Goa 调用。它接收：
// - genpkg：生成代码将放置到的包路径
// - roots：包含完整 API 设计的根表达式数组
// 它必须返回生成后应该存在的所有文件，包括未修改的文件。
// 未返回的文件将被删除，允许插件移除之前生成的文件。
func Generate(genpkg string, roots []eval.Root) ([]*codegen.File, error) {
    var files []*codegen.File
    
    for _, root := range roots {
        if r, ok := root.(*expr.RootExpr); ok {
            // 为每个使用限流的服务生成中间件
            for _, svc := range r.Services {
                if rate := findRateLimit(svc); rate != nil {
                    f := generateMiddleware(genpkg, svc, rate)
                    files = append(files, f)
                }
            }
        }
    }
    
    return files, nil
}

// generateMiddleware 创建限流中间件文件
func generateMiddleware(genpkg string, svc *expr.ServiceExpr, rate *RateExpr) *codegen.File {
    // 定义生成文件的路径
    path := filepath.Join(codegen.Gendir, "ratelimit", 
        codegen.SnakeCase(svc.Name)+".go")
    
    // 为模板准备数据
    data := map[string]interface{}{
        "Service": svc,
        "Rate":    rate,
        "Package": genpkg,
    }
    
    // 从模板创建一个 section
    section := &codegen.SectionTemplate{
        Name:    "ratelimit",
        Source: middlewareT,
        Data:    data,
        FuncMap: template.FuncMap{
            "goifyName": codegen.Goify,
        },
    }
    
    return &codegen.File{
        Path:             path,
        SectionTemplates: []*codegen.SectionTemplate{section},
    }
}
```

代码生成过程遵循以下步骤：

1. 当你运行 `goa gen` 时，Goa 按顺序处理所有已注册的插件
2. 对每个插件，Goa 以如下参数调用其 `Generate` 函数：
   - 目标包路径（`genpkg`）
   - 完整的 API 设计（`roots`）
3. 你的插件的 `Generate` 函数会：
   - 检查设计以找到使用你插件的服务
   - 使用模板创建合适的代码文件
   - 必须返回应该存在的所有文件，即使未修改
   - 可以通过不返回来删除文件
4. Goa 管理这些文件：
   - 创建或更新 `Generate` 返回的文件
   - 移除任何之前生成但未返回的文件
   - 将所有文件置于项目的 `gen` 目录：
   ```
   gen/
   ├── calculator/   # 主服务代码
   ├── http/        # HTTP 传输
   ├── cors/        # CORS 插件代码
   └── ratelimit/   # 限流代码
   ```

该过程确保你的插件的代码生成与 Goa 的标准输出无缝集成，并且对文件生命周期具有完全控制，包括在不再需要时删除文件的能力。

### 第四步：创建模板

`templates/middleware.go.tmpl` 中的模板定义了生成代码的外观：

```go
{{ define "ratelimit" }}
// 由 goa v3 ratelimit 插件生成的代码；请勿编辑。
package {{ .Package }}

import (
    "context"
    "time"
    "golang.org/x/time/rate"
)

// {{ goifyName .Service.Name "middleware" }} 为 {{ .Service.Name }} 服务实现限流。
type {{ goifyName .Service.Name "middleware" }} struct {
    limiter *rate.Limiter
    next    Service
}

// New{{ goifyName .Service.Name "middleware" }} 创建一个新的限流中间件。
func New{{ goifyName .Service.Name "middleware" }}() Middleware {
    limiter := rate.NewLimiter(
        rate.Every({{ .Rate.Period }}),
        {{ .Rate.Requests }},
    )
    limiter.SetBurst({{ .Rate.Burst }})
    
    return func(next Service) Service {
        return &{{ goifyName .Service.Name "middleware" }}{
            limiter: limiter,
            next:    next,
        }
    }
}

// Handle 实现中间件接口。
func (m *{{ goifyName .Service.Name "middleware" }}) Handle(ctx context.Context, next func(context.Context) error) error {
    if err := m.limiter.Wait(ctx); err != nil {
        return err
    }
    return next(ctx)
}
{{ end }}
```

### 第五步：注册插件

最后，将插件注册到 Goa。有三种注册函数可用，每种都会影响你的插件相对于其他插件的运行时机：

```go
package ratelimit

import "goa.design/goa/v3/codegen"

// 选项 1：标准注册（中间执行）
func init() {
    // 注册插件在中间阶段执行，按名称字母顺序排序
    codegen.RegisterPlugin("ratelimit", "gen", nil, Generate)
}

// 选项 2：最先执行注册
func init() {
    // 注册插件在其他非 first 插件之前执行
    codegen.RegisterPluginFirst("ratelimit", "gen", nil, Generate)
}

// 选项 3：最后执行注册（多数插件推荐）
func init() {
    // 注册插件在其他非 last 插件之后执行
    codegen.RegisterPluginLast("ratelimit", "gen", nil, Generate)
}
```

注册函数的参数说明：
- `name`：插件的唯一标识符
- `cmd`：该插件适配的 Goa 命令（通常为 "gen"，也可为 "example"）
- `pre`：可选的预处理函数（可为 nil）
- `p`：主要的生成函数

#### 插件执行顺序

Goa 维护三个有序的插件列表：
1. First 插件：在标准插件之前运行（通过 `RegisterPluginFirst` 注册）
2. 标准插件：在中间阶段运行（通过 `RegisterPlugin` 注册）
3. Last 插件：在标准插件之后运行（通过 `RegisterPluginLast` 注册）

在每个列表中，插件按名称字母顺序排序。例如：

```go
// 这些插件将按如下顺序运行：
codegen.RegisterPluginFirst("auth", "gen", nil, Generate)     // 1. auth（first）
codegen.RegisterPluginFirst("cache", "gen", nil, Generate)    // 2. cache（first）
codegen.RegisterPlugin("metrics", "gen", nil, Generate)       // 3. metrics（standard）
codegen.RegisterPlugin("tracing", "gen", nil, Generate)       // 4. tracing（standard）
codegen.RegisterPluginLast("cors", "gen", nil, Generate)      // 5. cors（last）
codegen.RegisterPluginLast("ratelimit", "gen", nil, Generate) // 6. ratelimit（last）
```

#### 选择合适的注册函数

根据你的插件的依赖与影响选择注册方式：

1. 当你的插件：
   - 需要在其他插件看到设计之前修改设计
   - 提供其他插件依赖的功能
   - 必须在特定内置生成器之前运行
   时使用 `RegisterPluginFirst`

2. 当你的插件：
   - 独立于其他插件
   - 没有特定的顺序要求
   - 与 Goa 默认生成代码兼容
   时使用 `RegisterPlugin`（标准）

3. 当你的插件：
   - 需要在其他插件之后看到最终状态
   - 修改或包装其他插件生成的代码
   - 添加诸如中间件等横切关注点
   时使用 `RegisterPluginLast`

对于我们的限流插件，应使用 `RegisterPluginLast`，因为：
- 它生成包裹服务端点的中间件
- 应在主服务代码生成之后运行
- 不影响其他插件的代码生成方式

```go
package ratelimit

import "goa.design/goa/v3/codegen"

func init() {
    // 作为 last 插件注册，因为我们生成的是中间件
    codegen.RegisterPluginLast("ratelimit", "gen", nil, Generate)
}
```

这可以确保我们的限流中间件能够正确包裹其他插件生成的任何中间件或处理器。

### 使用你的插件

现在用户可以在他们的设计中使用你的插件：

```go
package design

import (
    . "goa.design/goa/v3/dsl"
    . "path/to/ratelimit/dsl"
)

var _ = Service("calculator", func() {
    RateLimit(100, func() {
        Period("1m")
        Burst(20)
    })
    
    Method("add", func() {
        Payload(func() {
            Field(1, "a", Int)
            Field(2, "b", Int)
        })
        Result(Int)
    })
})
```

当他们运行 `goa gen` 时，你的插件将：
1. 处理限流配置
2. 验证设置
3. 生成中间件代码
4. 将其放到项目中的正确位置

## 插件进阶主题

在掌握创建插件的基础知识后，继续探索一些进阶技巧，帮助你构建更复杂的插件。

### 与表达式树协作

开发插件时，你经常需要导航并分析由设计构建的表达式树。如下是如何高效地处理表达式：

```go
// 查找服务中需要验证的方法
func findMethodsToValidate(svc *expr.ServiceExpr) []*expr.MethodExpr {
    var methods []*expr.MethodExpr
    
    for _, method := range svc.Methods {
        // 检查该方法是否具有需要验证的载荷
        if method.Payload != nil && needsValidation(method.Payload) {
            methods = append(methods, method)
        }
        
        // 检查结果是否需要验证
        if method.Result != nil && needsValidation(method.Result) {
            methods = append(methods, method)
        }
    }
    
    return methods
}

// 检查某个属性是否需要验证
func needsValidation(attr *expr.AttributeExpr) bool {
    // 在元数据中检查验证规则
    if meta := attr.Meta; meta != nil {
        if _, ok := meta["validate"]; ok {
            return true
        }
    }
    
    // 对于对象，检查每个字段
    if obj, ok := attr.Type.(*expr.Object); ok {
        for _, field := range *obj {
            if needsValidation(field.Attribute) {
                return true
            }
        }
    }
    
    return false
}
```

### 上下文感知的 DSL 函数

你的 DSL 函数应具备上下文感知能力并做出恰当行为。如下是如何创建上下文敏感的函数：

```go
// MaxItems 可用于不同上下文
func MaxItems(n int) {
    switch current := eval.Current().(type) {
    case *ArrayExpr:
        // 直接用于数组类型
        current.MaxItems = n
        
    case *ValidationExpr:
        // 用于验证块内
        if arr, ok := current.Target.Type.(*expr.Array); ok {
            current.MaxItems = &n
        } else {
            eval.ReportError("MaxItems 只能用于数组类型")
        }
        
    default:
        eval.IncompatibleDSL()
    }
}

// 示例用法：
var _ = Service("storage", func() {
    Method("list", func() {
        // 直接用于数组
        Payload(ArrayOf(String, func() {
            MaxItems(100)  // 限制数组大小
        }))
        
        // 用于验证
        Validate(func() {
            MaxItems(50)   // 不同上下文，同一函数
        })
    })
})
```

### 插件依赖

有时你的插件可能依赖其他插件。如下是处理依赖的方式：

```go
// 验证插件的根表达式
type ValidationRoot struct {
    *RootExpr
}

// DependsOn 表示该插件需要 security 插件
func (r *ValidationRoot) DependsOn() []eval.Root {
    return []eval.Root{
        // 此插件依赖 security 插件
        &security.Root{},
    }
}

// Packages 返回该插件所需的导入路径
func (r *ValidationRoot) Packages() []string {
    return []string{
        "goa.design/plugins/v3/security",
        "goa.design/plugins/v3/validation",
    }
}
```

### 进阶错误处理

插件中的错误处理应信息充分且有帮助。如下是创建详细错误消息的方法：

```go
func (v *ValidationExpr) Validate() error {
    errors := new(eval.ValidationErrors)
    
    // 归组相关验证
    if err := v.validateBasicRules(); err != nil {
        if verr, ok := err.(*eval.ValidationErrors); ok {
            errors.Merge(verr)
        }
    }
    
    // 为错误增加上下文
    if v.Maximum != nil && v.Minimum != nil {
        if *v.Maximum < *v.Minimum {
            errors.Add(v, 
                "maximum (%d) 不能小于 minimum (%d)",
                *v.Maximum, *v.Minimum)
        }
    }
    
    // 验证嵌套表达式
    for _, rule := range v.Rules {
        if err := rule.Validate(); err != nil {
            if verr, ok := err.(*eval.ValidationErrors); ok {
                // 合并时保留错误上下文
                errors.Merge(verr)
            } else {
                errors.Add(v, "无效的规则: %s", err)
            }
        }
    }
    
    if len(errors.Errors) > 0 {
        return errors
    }
    return nil
}

// 用于归组相关验证的辅助函数
func (v *ValidationExpr) validateBasicRules() error {
    errors := new(eval.ValidationErrors)
    
    // 检查必需字段
    if v.Pattern != "" {
        if _, err := regexp.Compile(v.Pattern); err != nil {
            errors.Add(v, "无效的正则模式 %q: %s", v.Pattern, err)
        }
    }
    
    return errors
}
```

### 进阶代码生成

对于复杂插件，你可能需要生成多个文件或处理不同的传输层：

```go
func Generate(genpkg string, roots []eval.Root) ([]*codegen.File, error) {
    var files []*codegen.File
    
    for _, root := range roots {
        if r, ok := root.(*expr.RootExpr); ok {
            // 生成特定服务的文件
            for _, svc := range r.Services {
                // 生成主服务文件
                if f := generateService(genpkg, svc); f != nil {
                    files = append(files, f)
                }
                
                // 生成传输层特定代码
                if f := generateHTTP(genpkg, svc); f != nil {
                    files = append(files, f)
                }
                if f := generateGRPC(genpkg, svc); f != nil {
                    files = append(files, f)
                }
                
                // 生成文档
                if f := generateDocs(genpkg, svc); f != nil {
                    files = append(files, f)
                }
            }
        }
    }
    
    return files, nil
}

// 生成传输层特定代码
func generateHTTP(genpkg string, svc *expr.ServiceExpr) *codegen.File {
    path := filepath.Join(codegen.Gendir, "http",
        codegen.SnakeCase(svc.Name)+".go")
    
    data := map[string]interface{}{
        "Service": svc,
        "Package": path.Base(genpkg),
    }
    
    sections := []*codegen.SectionTemplate{
        {
            Name:    "http-handler",
            Source: httpHandlerT,
            Data:    data,
            FuncMap: template.FuncMap{
                "routeName": func(m *expr.MethodExpr) string {
                    return codegen.Goify(m.Name, true) + "Handler"
                },
            },
        },
        {
            Name:    "http-client",
            Source:  httpClientT,
            Data:    data,
        },
    }
    
    return &codegen.File{
        Path:             path,
        SectionTemplates: sections,
    }
}
```

这些进阶技巧将帮助你创建更复杂的插件，能够：
- 导航并分析复杂设计
- 提供上下文感知的 DSL 函数
- 处理插件之间的依赖
- 生成信息充分的错误消息
- 为不同目的生成多个输出文件

## 插件开发最佳实践

结合实际经验，下面是帮助你创建高质量、易维护插件的最佳实践。这些指南基于真实的 Goa 插件开发经验。

### 设计原则

在设计插件接口时，请遵循以下原则：

1. 保持简单
   - 专注于把一个问题解决好
   - 让最常见的使用场景最易实现
   - 为可选设置提供合理默认值

2. 与 Goa 保持一致
   - 遵循 Goa 的 DSL 风格和命名约定
   - 使用与 Goa 内置函数相似的模式
   - 在错误消息和文档中保持一致性

一个设计良好的 DSL 例子：

```go
var _ = Service("orders", func() {
    // 简单常见的用法
    RateLimit(100)
    
    // 带选项的更复杂用法
    RateLimit(100, func() {
        Period("1m")
        Burst(20)
    })
})
```

### 代码组织

以清晰、可维护的方式组织插件代码：

```
plugin-name/
├── dsl/
│   ├── dsl.go       # 公共 DSL 函数
│   ├── types.go     # 表达式类型
│   └── internal.go  # 内部辅助函数
├── generate/
│   ├── generate.go  # 主生成逻辑
│   └── helpers.go   # 生成辅助函数
├── templates/       # 代码模板
│   ├── client.go.tmpl
│   └── server.go.tmpl
├── example/         # 使用示例
│   └── design/
│       └── design.go
└── README.md       # 清晰的文档
```

### 错误处理

实现完善的错误处理以帮助用户快速修复问题。Goa 提供了专门的 `ValidationErrors` 类型用于收集与管理验证错误：

```go
// ValidationErrors 收集多个验证错误及其上下文
type ValidationErrors struct {
    Errors      []error      // 实际错误
    Expressions []Expression // 错误发生的表达式
}

// 在验证函数中使用 ValidationErrors 的示例
func (r *RateExpr) Validate() error {
    errors := new(eval.ValidationErrors)
    
    // 以上下文添加单个错误
    if r.Requests <= 0 {
        errors.Add(r, "requests 必须为正数，得到 %d", r.Requests)
    }
    
    // 验证嵌套配置
    if err := r.validatePeriod(); err != nil {
        if verr, ok := err.(*eval.ValidationErrors); ok {
            // 合并嵌套验证错误
            errors.Merge(verr)
        } else {
            // 以上下文添加单个错误
            errors.AddError(r, err)
        }
    }
    
    if len(errors.Errors) > 0 {
        return errors
    }
    return nil
}

// 展示嵌套验证的辅助函数
func (r *RateExpr) validatePeriod() error {
    errors := new(eval.ValidationErrors)
    
    if r.Period != "" {
        if _, err := time.ParseDuration(r.Period); err != nil {
            // 以适当上下文添加格式错误
            errors.Add(r, 
                "无效的 period %q：必须为合法时长（如 '1m', '1h')",
                r.Period)
        }
    }
    
    return errors
}
```

`ValidationErrors` 类型提供以下关键特性：
1. 错误收集：在验证期间累积多个错误：
   ```go
   errors := new(eval.ValidationErrors)
   errors.Add(expr, "第一个错误: %v", val1)
   errors.Add(expr, "第二个错误: %v", val2)
   ```

2. 上下文保留：每个错误都与其表达式关联：
   ```go
   // 错误消息包含表达式名称：
   // "服务 'api' 的限流：requests 必须为正数，得到 -1"
   errors.Add(rateExpr, "requests 必须为正数，得到 %d", requests)
   ```

3. 错误合并：合并来自嵌套验证的错误：
   ```go
   func (v *ValidationExpr) Validate() error {
       errors := new(eval.ValidationErrors)
       
       // 验证基础配置
       if err := v.validateBasic(); err != nil {
           if verr, ok := err.(*eval.ValidationErrors); ok {
               errors.Merge(verr)  // 合并嵌套验证错误
           }
       }
       
       // 验证每条规则
       for _, rule := range v.Rules {
           if err := rule.Validate(); err != nil {
               if verr, ok := err.(*eval.ValidationErrors); ok {
                   errors.Merge(verr)  // 合并每条规则的错误
               } else {
                   errors.AddError(v, err)  // 添加单个错误
               }
           }
       }
       
       return errors
   }
   ```

4. 扁平化错误消息：`Error()` 方法产生清晰、结构化输出：
   ```go
   // 输出格式：
   // 服务 'api' 的限流：requests 必须为正数，得到 -1
   // 服务 'api' 的限流：无效的 period "2x"，请使用 "s"、"m" 或 "h"
   ```

使用 `ValidationErrors` 的最佳实践：

1. 尽早创建：在验证开始时创建错误容器
   ```go
   func (e *Expr) Validate() error {
       errors := new(eval.ValidationErrors)
       // ... 验证逻辑 ...
   }
   ```

2. 添加上下文：添加错误时始终提供表达式
   ```go
   errors.Add(e, "值 %v 非法", value)  // 推荐
   errors.AddError(e, fmt.Errorf("invalid"))    // 也可
   ```

3. 处理嵌套验证：正确合并子验证的错误
   ```go
   if err := subExpr.Validate(); err != nil {
       if verr, ok := err.(*eval.ValidationErrors); ok {
           errors.Merge(verr)
       } else {
           errors.AddError(e, err)
       }
   }
   ```

4. 及早返回：如果未发生错误则返回 nil
   ```go
   if len(errors.Errors) > 0 {
       return errors
   }
   return nil
   ```

这种结构化的错误处理方法可帮助用户理解并修复其 API 设计中的问题：
- 收集所有验证错误而非在第一个错误处停止
- 提供每个错误发生位置的清晰上下文
- 保持错误与其表达式之间的关联
- 生成格式良好的错误消息

### 代码生成

生成代码时遵循以下实践：

1. 有效使用模板
   ```go
   // 将复杂模板分解为更小、更聚焦的部分
   sections := []*codegen.SectionTemplate{
       {
           Name:   "types",
           Source: typesT,
           Data:   data,
       },
       {
           Name:   "encoders",
           Source: encodersT,
           Data:   data,
       },
   }
   ```

2. 生成整洁代码
   ```go
   // 在模板中添加清晰注释
   {{ define "types" }}
   // {{ .TypeName }} 实现限流配置。
   // 并发安全。
   type {{ .TypeName }} struct {
       limiter *rate.Limiter
       config  *Config
   }
   
   // Config 存储限流参数。
   type Config struct {
       Requests int           // 每周期最大请求数
       Period   time.Duration // 限制的时间周期
       Burst    int           // 最大突发
   }
   {{ end }}
   ```

3. 包含文档
   ```go
   // 在模板中生成包文档
   {{ define "header" }}
   // 包 {{ .Package }} 提供限流功能。
   //
   // 它实现令牌桶算法以控制请求速率。
   // 用法：
   //     limiter := New(100, time.Minute)  // 每分钟 100 次请求
   //     if err := limiter.Wait(ctx); err != nil {
   //         return err
   //     }
   package {{ .Package }}
   {{ end }}
   ```

### 测试

为你的插件实现完善的测试：

```go
func TestRateLimitDSL(t *testing.T) {
    cases := []struct {
        name     string
        design   func()
        wantErr  bool
        errMsg   string
    }{
        {
            name: "basic rate limit",
            design: func() {
                Service("test", func() {
                    RateLimit(100)
                })
            },
        },
        {
            name: "invalid rate limit",
            design: func() {
                Service("test", func() {
                    RateLimit(-1)
                })
            },
            wantErr: true,
            errMsg:  "requests 必须为正数",
        },
    }
    
    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            // 重置设计
            eval.Reset()
            
            // 运行测试
            err := eval.RunDSL(tc.design)
            
            // 检查结果
            if tc.wantErr {
                if err == nil {
                    t.Error("期望错误，实际为 nil")
                } else if !strings.Contains(err.Error(), tc.errMsg) {
                    t.Errorf("期望错误包含 %q，实际为 %q",
                        tc.errMsg, err.Error())
                }
            } else if err != nil {
                t.Errorf("不期望出现错误：%v", err)
            }
        })
    }
}
```

### 文档

提供清晰、全面的文档：

1. README.md
   - 清晰描述插件的目的
   - 安装说明
   - 基本使用示例
   - 配置选项
   - 常见用例

2. 代码注释
   ```go
   // RateLimit 为服务或方法应用限流。
   // 它允许指定每个时间周期允许的最大请求数。
   //
   // 示例：
   //
   //    var _ = Service("api", func() {
   //        // 简单用法：每分钟 100 次请求
   //        RateLimit(100)
   //
   //        // 进阶用法：自定义 period 与 burst
   //        RateLimit(100, func() {
   //            Period("1m")
   //            Burst(20)
   //        })
   func RateLimit(requests int, fn ...func()) { ... }
   ```

3. 示例
   - 在 `example` 目录中提供可运行的示例
   - 包含常见与进阶场景
   - 添加解释关键概念的注释


## 结论

插件是扩展 Goa 能力的强大方式。通过理解插件架构并遵循最佳实践，你可以创建健壮的插件，以满足你在代码生成方面的特定需求。

如需真实世界的示例与灵感，请查看
[官方插件仓库](https://github.com/goadesign/plugins)。