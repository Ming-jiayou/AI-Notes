# StyletIoC IContainer 接口详解

## 概述

`IContainer.cs` 文件定义了 StyletIoC 容器的核心接口 `IContainer`，它继承自 `IDisposable`。这个接口描述了一个 IoC（控制反转）容器的基本功能，提供了依赖注入的核心操作，包括服务解析、绑定编译和对象构建等功能。

## 接口定义

```csharp
public interface IContainer : IDisposable
{
    void Compile(bool throwOnError = true);
    object Get(Type type, string key = null);
    T Get<T>(string key = null);
    IEnumerable<object> GetAll(Type type, string key = null);
    IEnumerable<T> GetAll<T>(string key = null);
    object GetTypeOrAll(Type type, string key = null);
    T GetTypeOrAll<T>(string key = null);
    void BuildUp(object item);
}
```

## 方法详解

### Compile

```csharp
void Compile(bool throwOnError = true);
```

**用途**：编译所有已知的绑定，并检查依赖图的一致性。

**参数**：
- `throwOnError` (bool, 可选, 默认为 true): 如果为 true，则在编译类型失败时抛出异常

**说明**：
- 此方法会预先编译所有已注册的绑定，而不是在需要时才进行编译
- 通过检查依赖图的一致性，可以在早期发现潜在的依赖注入问题
- 在应用程序启动时调用此方法可以提高性能，因为避免了首次解析时的编译开销
- 如果设置为 false，则在编译失败时不会抛出异常，但可能会导致后续的解析操作失败

### Get

```csharp
object Get(Type type, string key = null);
T Get<T>(string key = null);
```

**用途**：获取指定类型的单个实例。

**参数**：
- `type` (Type): 要获取实现的服务类型
- `key` (string, 可选, 默认为 null): 服务实现注册时使用的键
- `T` (泛型): 要获取实现的服务类型

**返回值**：
- 请求的服务实例

**说明**：
- 这是最基本的服务解析方法，用于获取单个服务实例
- 支持通过键来区分同一服务类型的不同实现
- 泛型版本提供了类型安全的解析方式
- 如果找不到匹配的服务实现，通常会抛出异常
- 根据绑定的作用域（如单例、瞬态等），返回的实例生命周期会有所不同

### GetAll

```csharp
IEnumerable<object> GetAll(Type type, string key = null);
IEnumerable<T> GetAll<T>(string key = null);
```

**用途**：获取实现指定服务的所有类型的实例。

**参数**：
- `type` (Type): 要获取实现的服务类型
- `key` (string, 可选, 默认为 null): 服务实现注册时使用的键
- `T` (泛型): 要获取实现的服务类型

**返回值**：
- 请求的服务的所有实现，具有请求的键

**说明**：
- 当一个服务有多个实现时，此方法可以获取所有已注册的实现
- 支持通过键来过滤特定实现
- 泛型版本提供了类型安全的解析方式
- 返回的是一个集合，即使只有一个匹配的实现
- 常用于策略模式、插件系统等需要多个实现的场景

### GetTypeOrAll

```csharp
object GetTypeOrAll(Type type, string key = null);
T GetTypeOrAll<T>(string key = null);
```

**用途**：根据类型决定是获取单个实例还是所有实例。

**参数**：
- `type` (Type): 如果是 IEnumerable{T}，将获取 T 的所有实现，否则获取单个 T
- `key` (string, 可选, 默认为 null): 服务实现注册时使用的键
- `T` (泛型): 如果是 IEnumerable{T}，将获取 T 的所有实现，否则获取单个 T

**返回值**：
- 解析结果

**说明**：
- 这是一个智能解析方法，根据请求的类型自动决定使用 Get 还是 GetAll
- 如果请求的类型是 IEnumerable{T} 或类似集合类型，则等同于调用 GetAll{T}
- 否则，等同于调用 Get{T}
- 提供了更灵活的解析方式，特别适用于需要根据类型动态决定解析策略的场景
- 泛型版本提供了类型安全的解析方式

### BuildUp

```csharp
void BuildUp(object item);
```

**用途**：对带有 [Inject] 特性的属性/字段进行依赖注入。

**参数**：
- `item` (object): 要构建的对象

**说明**：
- 此方法用于对已存在的对象进行依赖注入
- 会扫描对象的所有属性和字段，查找带有 [Inject] 特性的成员
- 为这些成员自动解析并注入相应的服务实例
- 常用于非容器创建的对象需要依赖注入的场景
- 例如，在反序列化后或手动创建的对象上应用依赖注入

## 使用示例

### 基本服务解析

```csharp
// 获取单个服务实例
var myService = container.Get<IMyService>();

// 使用键获取特定实现
var specificService = container.Get<IMyService>("SpecialImplementation");

// 获取所有实现
var allServices = container.GetAll<IMyService>();
```

### 类型安全的泛型解析

```csharp
// 类型安全的单个服务解析
var myService = container.Get<MyService>();

// 类型安全的多个服务解析
var allServices = container.GetAll<IMyService>();
```

### 智能解析

```csharp
// 根据类型自动决定解析策略
var singleService = container.GetTypeOrAll<IMyService>(); // 等同于 Get<IMyService>()
var allServices = container.GetTypeOrAll<IEnumerable<IMyService>>(); // 等同于 GetAll<IMyService>()
```

### 对象构建

```csharp
// 创建对象后进行依赖注入
var myObject = new MyObject();
container.BuildUp(myObject); // 为带有 [Inject] 特性的属性/字段注入依赖
```

### 容器编译

```csharp
// 在应用程序启动时编译所有绑定
try
{
    container.Compile(); // 检查依赖图一致性
}
catch (Exception ex)
{
    // 处理编译错误
    Console.WriteLine($"依赖注入配置错误: {ex.Message}");
}
```

## 设计特点

### 1. 灵活的解析策略

- 支持单个服务解析和多个服务解析
- 提供智能解析方法，根据类型自动选择解析策略
- 支持通过键区分同一服务的不同实现

### 2. 类型安全

- 提供泛型方法版本，确保编译时类型检查
- 减少运行时类型转换错误

### 3. 延迟编译

- 支持显式编译所有绑定，提前发现依赖问题
- 可以选择在编译失败时是否抛出异常

### 4. 对象构建支持

- 不仅支持创建新对象，还支持对现有对象进行依赖注入
- 通过特性标记的方式指定需要注入的成员

### 5. 资源管理

- 继承自 `IDisposable`，支持资源的正确释放
- 确保容器管理的资源能够被正确清理

## 最佳实践

### 1. 应用程序启动时编译

```csharp
// 在应用程序启动时编译所有绑定
container.Compile();
```

这样可以提前发现依赖配置问题，避免在运行时出现错误。

### 2. 使用泛型方法

```csharp
// 推荐：使用泛型方法
var service = container.Get<IMyService>();

// 不推荐：使用非泛型方法
var service = (IMyService)container.Get(typeof(IMyService));
```

泛型方法提供更好的类型安全性和可读性。

### 3. 合理使用键

```csharp
// 为不同实现使用有意义的键
builder.Bind<IMyService>().To<MyService1>().WithKey("Default");
builder.Bind<IMyService>().To<MyService2>().WithKey("Advanced");

// 获取特定实现
var defaultService = container.Get<IMyService>("Default");
var advancedService = container.Get<IMyService>("Advanced");
```

使用有意义的键可以提高代码的可读性和可维护性。

### 4. 适当使用 BuildUp

```csharp
// 对于非容器创建的对象使用 BuildUp
var viewModel = new MyViewModel();
container.BuildUp(viewModel);
```

但应尽量避免过度使用，优先考虑通过构造函数注入。

## 总结

`IContainer` 接口定义了 StyletIoC 容器的核心功能，提供了完整的服务解析和依赖注入能力。通过灵活的解析策略、类型安全的 API 和对象构建支持，开发者可以轻松实现依赖注入模式，提高代码的可测试性和可维护性。合理使用这些方法，可以构建出松耦合、易于扩展的应用程序架构。