# Goa 视图与标签 DSL：响应控制系统---

## 视图系统（View System）

Goa 的视图系统让你能够精确控制 API 响应中出现的数据。它在内部数据模型与向 API 使用者暴露的数据之间建立了清晰的分离。

### 视图的关键收益

1. **客户端可控的数据获取**：视图允许客户端只请求所需的数据，减少带宽占用，并为不同场景简化响应。
2. **隐藏实现细节**：可将用于业务逻辑或状态判定的内部字段隐藏，不对 API 使用者暴露。
3. **字段选择**：可为不同场景定义 `ResultType` 的字段子集，构建定制化表示。

### 使用视图选择

#### 设计中的静态视图选择

当你希望某个端点的响应格式保持一致且可预测时，可以在设计中静态指定视图：

```go
Method("getResource", func() {
    Payload(func() {
        Attribute("id", String)
        Required("id")
    })
    // 为该方法显式选择 "some_view" 视图
    Result(Resource, func() {
        View("some_view") 
    })
    HTTP(func() {
        GET("/resources/{id}/minimal")
        Response(StatusOK)
    })
})
```

采用该方式时，无论其他条件如何，端点都会返回指定视图。

#### 实现中的动态视图选择

当设计未指定固定视图时，实现代码必须选择使用的视图：

```go
// GetResource 的实现代码
func (s *serviceImpl) GetResource(ctx context.Context, p *service.GetResourcePayload) (*service.Resource, string, error) {
    // 获取资源数据
    resource, err := s.repository.Find(p.ID)
    if err != nil {
        return nil, "", err
    }
    
    // 映射到响应类型
    res := &service.Resource{
        ID:   resource.ID,
        Name: resource.Name,
        ID2:  resource.SecondaryID,
    }
    
    // 返回资源并选择所需视图
    return res, "default", nil
}
```

关键点在于：使用动态视图选择时，实现需要将视图名称作为第二个返回值显式返回。

### 基于上下文选择视图

实现可以根据业务规则或用户权限选择不同视图：

```go
func (s *serviceImpl) GetResource(ctx context.Context, p *service.GetResourcePayload) (*service.Resource, string, error) {
    // 获取资源数据
    resource, err := s.repository.Find(p.ID)
    if err != nil {
        return nil, "", err
    }
    
    // 填充响应
    res := &service.Resource{
        ID:   resource.ID,
        Name: resource.Name,
        ID2:  resource.SecondaryID,
    }
    
    // 基于用户角色选择视图
    if isAdmin(ctx) {
        return res, "default", nil  // 管理员使用包含更多字段的默认视图
    }
    
    return res, "some_view", nil  // 其他用户使用精简视图
}
```

### 客户端可控的视图选择

也可以通过参数让客户端指定所需视图：

```go
var _ = Service("service", func() {
    Method("getResource", func() {
        Payload(func() {
            Attribute("id", String, "Unique identifier of the resource")
            // 添加视图参数
            Attribute("view", String, "View to render", func() {
                Enum("default", "some_view")
                Default("default")
            })
            Required("id")
        })
        Result(Resource)
        HTTP(func() {
            GET("/resources/{id}")
            Param("view")  // 将查询参数映射到载荷
            Response(StatusOK)
        })
    })
})
```

实现：

```go
func (s *serviceImpl) GetResource(ctx context.Context, p *service.GetResourcePayload) (*service.Resource, string, error) {
    // 获取资源数据
    resource, err := s.repository.Find(p.ID)
    if err != nil {
        return nil, "", err
    }
    
    // 填充响应
    res := &service.Resource{
        ID:   resource.ID,
        Name: resource.Name,
        ID2:  resource.SecondaryID,
    }
    
    // 使用客户端请求中的视图
    return res, p.View, nil
}
```

## 视图与标签的结合（Combining Tags with Views）

视图控制响应中出现的字段；标签（Tags）则可根据结果内容决定使用的 HTTP 状态码。

### 在改造示例中使用标签

让我们增强示例，通过标签选择 HTTP 状态：

```go
var Resource = ResultType("application/vnd.goa.resource", "Resource", func() {
    Attributes(func() {
        Attribute("name", String, "Name of the resource")
        Attribute("id", String, "Unique identifier of the resource")
        Attribute("id2", String, "Unique identifier of the resource")
        
        // 用于状态判定的内部属性
        Attribute("outcome", String, func() {
            Description("Internal field for response status")
            Meta("struct:tag:json", "-")  // 在 JSON 中隐藏
        })
    })
    
    View("default", func() {
        Attribute("name")
        Attribute("id")
        // 有意不包含 "outcome"
    })
    
    View("some_view", func() {
        Description("Some useful view")
        Attribute("id")
        // 有意不包含 "outcome"
    })
})

var _ = Service("service", func() {
    Method("getResource", func() {
        Payload(func() {
            Attribute("id", String, "Unique identifier of the resource")
            Required("id")
        })
        Result(Resource)
        HTTP(func() {
            GET("/resources/{id}")
            
            Response(StatusOK, func() {
                Tag("outcome", "found")
            })
            
            Response(StatusNotFound, func() {
                Tag("outcome", "not_found")
            })
        })
    })
})
```

带标签的实现：

```go
func (s *serviceImpl) GetResource(ctx context.Context, p *service.GetResourcePayload) (*service.Resource, string, error) {
    // 尝试获取资源数据
    resource, err := s.repository.Find(p.ID)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            // 返回带有 outcome 的资源以触发 404
            return &service.Resource{
                ID:      p.ID,
                Outcome: "not_found",
            }, "some_view", nil
        }
        return nil, "", err
    }
    
    // 找到资源——返回带有 outcome 的资源以触发 200
    res := &service.Resource{
        ID:      resource.ID,
        Name:    resource.Name,
        ID2:     resource.SecondaryID,
        Outcome: "found",
    }
    
    return res, "default", nil
}
```

在该示例中：
1. `outcome` 字段决定使用的 HTTP 状态码；
2. 该字段在所有视图中均被排除，因此不会发送给客户端；
3. 实现同时选择合适的视图并设置 `outcome`，以得到正确的 HTTP 状态。

### 进阶用法：基于状态覆盖视图

也可以为不同的状态响应指定不同视图：

```go
HTTP(func() {
    GET("/resources/{id}")
    
    Response(StatusOK, func() {
        Tag("outcome", "found")
        // 找到资源时使用默认视图
        View("default")
    })
    
    Response(StatusNotFound, func() {
        Tag("outcome", "not_found")
        // 未找到时使用精简视图
        View("some_view")
    })
})
```

采用该方式时，如果希望使用设计中定义的视图，则无需在实现的返回值中指定视图：

```go
func (s *serviceImpl) GetResource(ctx context.Context, p *service.GetResourcePayload) (*service.Resource, string, error) {
    // 尝试获取资源数据
    resource, err := s.repository.Find(p.ID)
    if err != nil {
        if errors.Is(err, ErrNotFound) {
            // 返回带有 outcome 的资源以触发 404——视图由设计决定
            return &service.Resource{
                ID:      p.ID,
                Outcome: "not_found",
            }, "", nil
        }
        return nil, "", err
    }
    
    // 找到资源——视图由设计决定
    res := &service.Resource{
        ID:      resource.ID,
        Name:    resource.Name,
        ID2:     resource.SecondaryID,
        Outcome: "found",
    }
    
    return res, "", nil
}
```

## 实用收益（Practical Benefits）

视图与标签的组合带来清晰的关注点分离：

1. **HTTP 语义**：根据资源状态使用恰当的 HTTP 状态码；
2. **干净的领域模型**：客户端只看到相关字段而非实现细节；
3. **上下文数据**：可基于状态或客户端偏好提供不同视图；
4. **实现简洁**：聚焦业务逻辑而非 HTTP 细节。

借助视图与标签，你可以构建既具备恰当 HTTP 语义、又保持响应模型聚焦领域的 API。