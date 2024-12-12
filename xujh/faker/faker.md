# OpenTofu Provider Faker 设计文档

## 1. 目标

设计一个 Provider Faker 系统，用于:
- 拦截真实 Provider 的执行逻辑
- 注入模拟数据
- 管理模拟数据的生命周期

## 2. 核心组件设计

### 2.1 Faker Provider 包装器
```go
type FakerProviderWrapper struct {
realProvider providers.Interface
fakeData FakeDataStore
}
```

主要职责:
- 包装真实 Provider
- 拦截所有 Provider 接口调用
- 根据配置决定是返回假数据还是调用真实 Provider

### 2.2 假数据存储

```go
type FakeDataStore struct {
    store    map[string]interface{}
    lifetime time.Duration
    mu       sync.RWMutex
}
```

主要职责:
- 存储模拟数据
- 管理数据生命周期
- 提供线程安全的数据访问

### 2.3 Faker API

提供外部接口用于:
- 设置模拟数据
- 配置数据生命周期
- 清理过期数据

## 3. 实现步骤

1. Provider 初始化拦截
   - 在 Context 初始化 Provider 时注入 Faker
   - 保持与原有接口的兼容性

2. 请求拦截处理
   - 拦截所有 Provider 接口调用
   - 检查是否存在对应的模拟数据
   - 决定是返回模拟数据还是调用真实 Provider

3. 数据生命周期管理
   - 实现数据过期机制
   - 定期清理过期数据
   - 支持手动清理数据

## 4. 使用示例

```go
// 初始化 Faker
faker := NewProviderFaker()

// 设置模拟数据
faker.SetFakeData("aws_instance", mockResponse, 30*time.Minute)

// 清理数据
faker.ClearFakeData("aws_instance")
```

## 5. 注意事项

1. 性能考虑
   - 最小化拦截层开销
   - 高效的数据存储和检索

2. 线程安全
   - 并发访问保护
   - 原子操作保证

3. 测试策略
   - 单元测试覆盖
   - 集成测试验证

4. 错误处理
   - 优雅降级机制
   - 完善的错误报告

## 6. 后续优化

1. 支持更复杂的模拟场景
   - 条件匹配
   - 动态响应生成
   - 状态转换模拟

2. 提供数据模板机制
   - 预定义模板
   - 模板参数化
   - 模板继承和组合

3. 添加监控和统计功能
   - 拦截统计
   - 性能监控
   - 使用分析

4. 支持动态配置更新
   - 热重载配置
   - 动态调整生命周期
   - 实时控制开关

## 7. 代码实现示例

### 7.1 Provider 包装器

```go
type FakerProviderWrapper struct {
    realProvider providers.Interface
    fakeData     *FakeDataStore
}

func (f *FakerProviderWrapper) ReadResource(req ReadResourceRequest) ReadResourceResponse {
    if fakeResp := f.fakeData.GetFakeResponse(req); fakeResp != nil {
        return *fakeResp
    }
    return f.realProvider.ReadResource(req)
}

func (f *FakerProviderWrapper) PlanResourceChange(req PlanResourceChangeRequest) PlanResourceChangeResponse {
    if fakeResp := f.fakeData.GetFakeResponse(req); fakeResp != nil {
        return *fakeResp
    }
    return f.realProvider.PlanResourceChange(req)
}

func (f *FakerProviderWrapper) ApplyResourceChange(req ApplyResourceChangeRequest) ApplyResourceChangeResponse {
    if fakeResp := f.fakeData.GetFakeResponse(req); fakeResp != nil {
        return *fakeResp
    }
    return f.realProvider.ApplyResourceChange(req)
}
```

### 7.2 数据存储实现

```go
type FakeData struct {
    Data      interface{}
    ExpiresAt time.Time
}

type FakeDataStore struct {
    store map[string]*FakeData
    mu    sync.RWMutex
}

func (s *FakeDataStore) SetFakeData(key string, data interface{}, lifetime time.Duration) {
    s.mu.Lock()
    defer s.mu.Unlock()
    
    s.store[key] = &FakeData{
        Data:      data,
        ExpiresAt: time.Now().Add(lifetime),
    }
}

func (s *FakeDataStore) GetFakeResponse(req interface{}) interface{} {
    s.mu.RLock()
    defer s.mu.RUnlock()
    
    key := s.generateKey(req)
    if data, exists := s.store[key]; exists {
        if time.Now().Before(data.ExpiresAt) {
            return data.Data
        }
        // 数据过期，异步清理
        go s.Remove(key)
    }
    return nil
}

func (s *FakeDataStore) Remove(key string) {
    s.mu.Lock()
    defer s.mu.Unlock()
    delete(s.store, key)
}
```

### 7.3 初始化注入

```go
func initProviderWithFaker(ctx context.Context, factory providers.Factory) (providers.Interface, error) {
    realProvider, err := factory()
    if err != nil {
        return nil, err
    }
    
    return &FakerProviderWrapper{
        realProvider: realProvider,
        fakeData:     NewFakeDataStore(),
    }, nil
}
```

## 8. 配置示例

```hcl
faker "aws" {
  enabled = true
  
  mock "aws_instance" {
    lifetime = "30m"
    response = {
      id = "i-fake123"
      instance_type = "t2.micro"
      state = "running"
    }
  }
  
  mock "aws_s3_bucket" {
    lifetime = "1h"
    response = {
      id = "fake-bucket"
      arn = "arn:aws:s3:::fake-bucket"
    }
  }
}
```

## 9. 总结

Provider Faker 系统通过提供一个灵活的模拟层，使得开发和测试过程中可以更好地控制 Provider 的行为。该设计既保持了与原有系统的兼容性，又提供了强大的模拟能力，同时考虑了性能、安全性和可维护性等关键因素。

后续可以根据实际使用情况和需求，进一步优化和扩展功能，使其成为更加完善的测试和开发工具。