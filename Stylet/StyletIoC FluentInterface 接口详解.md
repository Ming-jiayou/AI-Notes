# StyletIoC FluentInterface 接口详解

## 概述

`FluentInterface.cs` 文件定义了 StyletIoC 容器的流畅接口（Fluent Interface）系统。这些接口允许开发者以链式调用的方式配置依赖注入绑定，提供了类型安全和直观的 API 来定义服务与其实现之间的关系。

## 核心接口结构

### IBindTo

```csharp
public interface IBindTo : IToAnyService, IWithKeyOrAndOrToMultipleServices, IWithKeyOrToMulipleServices
{
}
```

**用途**：这是主要的绑定接口，是其他几个接口的组合。通过调用 `StyletIoCBuilder.Bind(..)` 可以获取此接口的实例。

**特点**：
- 作为入口点接口，组合了多个其他接口的功能
- 提供了完整的绑定配置能力

### IToMultipleServices

```csharp
public interface IToMultipleServices
{
    IInScopeOrWithKeyOrAsWeakBinding To(Type implementationType);
    IInScopeOrWithKeyOrAsWeakBinding To<TImplementation>();
    IInScopeOrWithKeyOrAsWeakBinding ToFactory<TImplementation>(Func<IRegistrationContext, TImplementation> factory);
    IWithKeyOrAsWeakBindingOrDisposeWithContainer ToInstance(object instance);
}
```

**用途**：为已创建的绑定提供进一步选项，特别是当绑定到多个服务时。

**主要方法**：
- `To(Type implementationType)`: 将服务绑定到实现该服务的另一个类型
- `To<TImplementation>()`: 泛型版本，将服务绑定到实现该服务的另一个类型
- `ToFactory<TImplementation>(Func<IRegistrationContext, TImplementation> factory)`: 将服务绑定到工厂委托
- `ToInstance(object instance)`: 将服务绑定到给定的未类型化实例

### IToAnyService

```csharp
public interface IToAnyService : IToMultipleServices
{
    IInScopeOrWithKeyOrAsWeakBinding ToSelf();
    IWithKeyOrAsWeakBinding ToAbstractFactory();
    IInScopeOrWithKeyOrAsWeakBinding ToAllImplementations(IEnumerable<Assembly> assemblies, bool allowZeroImplementations = false);
    IInScopeOrWithKeyOrAsWeakBinding ToAllImplementations(bool allowZeroImplementations = false, params Assembly[] assemblies);
}
```

**用途**：为单个服务或多个服务提供绑定选项。

**主要方法**：
- `ToSelf()`: 将服务绑定到自身
- `ToAbstractFactory()`: 如果服务是接口且方法返回其他类型，生成该抽象工厂的实现并绑定到接口
- `ToAllImplementations(...)`: 在指定程序集中发现服务的所有实现并绑定

### IAndTo

```csharp
public interface IAndTo
{
    IWithKeyOrAndOrToMultipleServices And(Type serviceType);
    IWithKeyOrAndOrToMultipleServices And<TService>();
}
```

**用途**：提供 'And' 绑定选项，允许向当前绑定添加另一个服务。

**主要方法**：
- `And(Type serviceType)`: 向此绑定添加另一个服务类型
- `And<TService>()`: 泛型版本，向此绑定添加另一个服务类型

### IWithKeyOrToMulipleServices

```csharp
public interface IWithKeyOrToMulipleServices : IToMultipleServices
{
    IAndOrToMultipleServices WithKey(string key);
}
```

**用途**：为多个服务提供 'WithKey' 或 'ToXXX' 选项。

**主要方法**：
- `WithKey(string key)`: 向多服务绑定的此部分添加键

### IAndOrToMultipleServices

```csharp
public interface IAndOrToMultipleServices : IToMultipleServices, IAndTo
{
}
```

**用途**：为多个服务提供 'And' 或 'ToXXX' 选项，组合了 `IToMultipleServices` 和 `IAndTo` 的功能。

### IWithKeyOrAndOrToMultipleServices

```csharp
public interface IWithKeyOrAndOrToMultipleServices : IWithKeyOrToMulipleServices, IAndTo
{
}
```

**用途**：为多个服务提供 'WithKey' 或 'And' 或 'ToXXX' 选项，组合了 `IWithKeyOrToMulipleServices` 和 `IAndTo` 的功能。

## 配置接口

### IAsWeakBinding

```csharp
public interface IAsWeakBinding
{
    void AsWeakBinding();
}
```

**用途**：允许将绑定标记为弱绑定。

**弱绑定特性**：
- 当容器构建时，检查每个 Type+key 组合的注册集合
- 如果只存在弱绑定，则所有绑定都会构建到容器中
- 如果存在任何正常绑定，则所有弱绑定被忽略，只有正常绑定构建到容器中
- 对于将 StyletIoC 集成到框架中非常有用
- AutoBind 在自绑定具体类型时也会使用弱绑定

### IWithKeyOrAsWeakBinding

```csharp
public interface IWithKeyOrAsWeakBinding : IAsWeakBinding
{
    IAsWeakBinding WithKey(string key);
}
```

**用途**：提供 WithKey 或 AsWeakBinding 功能的流畅接口。

**主要方法**：
- `WithKey(string key)`: 将键与此绑定关联，请求服务时必须指定此键才能检索此绑定的结果

### IWithKeyOrAsWeakBindingOrDisposeWithContainer

```csharp
public interface IWithKeyOrAsWeakBindingOrDisposeWithContainer : IWithKeyOrAsWeakBinding
{
    IWithKeyOrAsWeakBinding DisposeWithContainer(bool disposeWithContainer);
}
```

**用途**：提供 WithKey、AsWeakBinding 或 DisposeWithContainer 功能的流畅接口。

**主要方法**：
- `DisposeWithContainer(bool disposeWithContainer)`: 控制此实例是否在 IoC 容器释放时被释放，默认为 true

### IInScopeOrAsWeakBinding

```csharp
public interface IInScopeOrAsWeakBinding : IAsWeakBinding
{
    IAsWeakBinding WithRegistrationFactory(RegistrationFactory registrationFactory);
    IAsWeakBinding InSingletonScope();
}
```

**用途**：提供修改绑定作用域的方法的流畅接口。

**主要方法**：
- `WithRegistrationFactory(RegistrationFactory registrationFactory)`: 指定用于创建此绑定的 IRegistration 的工厂
- `InSingletonScope()`: 将绑定的作用域修改为单例，为此绑定生成一个实现实例

### IInScopeOrWithKeyOrAsWeakBinding

```csharp
public interface IInScopeOrWithKeyOrAsWeakBinding : IInScopeOrAsWeakBinding
{
    IInScopeOrAsWeakBinding WithKey(string key);
}
```

**用途**：提供 WithKey、AsWeakBinding 或作用域扩展功能的流畅接口。

**主要方法**：
- `WithKey(string key)`: 将键与此绑定关联，请求服务时必须指定此键才能检索此绑定的结果

## 接口关系图

```
IBindTo
├── IToAnyService
│   ├── IToMultipleServices
│   │   ├── IWithKeyOrToMulipleServices
│   │   │   └── IWithKeyOrAndOrToMultipleServices
│   │   └── IAndOrToMultipleServices
│   │       └── IWithKeyOrAndOrToMultipleServices
│   └── IAndTo
│       └── IWithKeyOrAndOrToMultipleServices
├── IWithKeyOrAndOrToMultipleServices
└── IWithKeyOrToMulipleServices

IAsWeakBinding
├── IWithKeyOrAsWeakBinding
│   ├── IWithKeyOrAsWeakBindingOrDisposeWithContainer
│   └── IInScopeOrWithKeyOrAsWeakBinding
└── IInScopeOrAsWeakBinding
```

## 使用示例

### 基本绑定

```csharp
// 将接口绑定到具体实现
builder.Bind<IMyService>().To<MyService>();

// 自绑定
builder.Bind<MyService>().ToSelf();

// 绑定到实例
builder.Bind<IMyService>().ToInstance(new MyService());
```

### 带键的绑定

```csharp
// 使用键区分不同的实现
builder.Bind<IMyService>().To<MyService1>().WithKey("Service1");
builder.Bind<IMyService>().To<MyService2>().WithKey("Service2");
```

### 工厂绑定

```csharp
// 绑定到工厂方法
builder.Bind<IMyService>().ToFactory(c => new MyService(c.Get<IDependency>(), "config"));
```

### 多服务绑定

```csharp
// 将一个实现绑定到多个接口
builder.Bind<IService1>().And<IService2>().To<MyMultiService>();
```

### 作用域控制

```csharp
// 单例作用域
builder.Bind<IMyService>().To<MyService>().InSingletonScope();

// 弱绑定
builder.Bind<IMyService>().To<MyService>().AsWeakBinding();
```

### 抽象工厂

```csharp
// 自动生成抽象工厂实现
builder.Bind<IMyFactory>().ToAbstractFactory();
```

### 自动发现实现

```csharp
// 自动发现并绑定所有实现
builder.Bind<IMyService>().ToAllImplementations();
```

## 设计优势

1. **类型安全**：通过泛型接口提供编译时类型检查
2. **流畅API**：支持链式调用，使配置代码更加清晰易读
3. **灵活性**：支持多种绑定方式和配置选项
4. **弱绑定机制**：允许框架提供默认绑定，同时允许用户覆盖
5. **作用域控制**：支持不同的生命周期管理
6. **多服务绑定**：允许一个实现满足多个接口

## 总结

StyletIoC 的 FluentInterface 系统通过精心设计的接口层次结构，提供了一个强大而灵活的依赖注入配置 API。这些接口不仅支持基本的类型绑定，还支持高级特性如弱绑定、作用域控制、工厂方法和多服务绑定等。通过流畅的接口设计，开发者可以以直观和类型安全的方式配置复杂的依赖关系，同时保持代码的可读性和可维护性。