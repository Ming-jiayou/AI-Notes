# StyletIoCBootstrapperBase 设计与实现详解

## 概述

StyletIoCBootstrapperBase 是 Stylet 框架中专门为使用 StyletIoC 容器而设计的引导程序基类。它继承自 BootstrapperBase，提供了 IoC 容器的集成、默认服务配置和依赖注入基础设施。这个类为应用程序提供了完整的依赖注入支持，是构建可测试、可维护的 WPF MVVM 应用程序的重要基础。

## 设计目标

1. **IoC 容器集成**：无缝集成 StyletIoC 依赖注入容器
2. **默认服务配置**：提供框架核心服务的默认绑定
3. **可扩展性**：允许自定义服务注册和配置
4. **生命周期管理**：正确管理容器和服务的生命周期
5. **程序集扫描**：支持自动程序集发现和绑定
6. **弱绑定支持**：允许用户覆盖默认服务实现

## 类结构分析

### 类层次结构

```csharp
public abstract class StyletIoCBootstrapperBase : BootstrapperBase
```

继承自 BootstrapperBase，提供了 IoC 特定的功能扩展。

### 核心属性

```csharp
protected IContainer Container { get; set; }
```

**设计特点：**
- 受保护的属性，允许子类访问容器
- 延迟初始化，在 ConfigureBootstrapper 后创建
- 提供完整的依赖注入功能

## 核心方法详解

### 1. ConfigureBootstrapper - 引导程序配置核心

```csharp
protected sealed override void ConfigureBootstrapper()
{
    var builder = new StyletIoCBuilder();
    builder.Assemblies = new List<Assembly>(new List<Assembly>() { this.GetType().Assembly });

    // Call DefaultConfigureIoC *after* ConfigureIoIC, so that they can customize builder.Assemblies
    this.ConfigureIoC(builder);
    this.DefaultConfigureIoC(builder);

    this.Container = builder.BuildContainer();
}
```

**设计亮点：**

1. **密封方法**：防止子类意外覆盖核心配置逻辑
2. **程序集配置**：默认包含引导程序所在程序集
3. **配置顺序**：先执行用户配置，再执行默认配置
4. **容器构建**：最终构建 IoC 容器实例

**配置流程：**
1. 创建 StyletIoCBuilder
2. 设置默认程序集
3. 调用用户自定义配置 (`ConfigureIoC`)
4. 调用默认配置 (`DefaultConfigureIoC`)
5. 构建容器实例

### 2. DefaultConfigureIoC - 默认服务配置

```csharp
protected virtual void DefaultConfigureIoC(StyletIoCBuilder builder)
{
    // Mark these as weak-bindings, so the user can replace them if they want
    var viewManagerConfig = new ViewManagerConfig()
    {
        ViewFactory = this.GetInstance,
        ViewAssemblies = new List<Assembly>() { this.GetType().Assembly }
    };
    builder.Bind<ViewManagerConfig>().ToInstance(viewManagerConfig).AsWeakBinding();

    // Bind it to both IViewManager and to itself, so that people can get it with Container.Get<ViewManager>()
    builder.Bind<IViewManager>().And<ViewManager>().To<ViewManager>().InSingletonScope().AsWeakBinding();

    builder.Bind<IWindowManagerConfig>().ToInstance(this).DisposeWithContainer(false).AsWeakBinding();
    builder.Bind<IWindowManager>().To<WindowManager>().InSingletonScope().AsWeakBinding();
    builder.Bind<IEventAggregator>().To<EventAggregator>().InSingletonScope().AsWeakBinding();
    builder.Bind<IMessageBoxViewModel>().To<MessageBoxViewModel>().AsWeakBinding();
    // Stylet's assembly isn't added to the container, so add this explicitly
    builder.Bind<MessageBoxView>().ToSelf();

    builder.Autobind();
}
```

**默认服务配置详解：**

#### ViewManagerConfig 配置
```csharp
var viewManagerConfig = new ViewManagerConfig()
{
    ViewFactory = this.GetInstance,
    ViewAssemblies = new List<Assembly>() { this.GetType().Assembly }
};
builder.Bind<ViewManagerConfig>().ToInstance(viewManagerConfig).AsWeakBinding();
```

**设计考虑：**
- 配置视图管理器使用 IoC 容器创建实例
- 默认扫描引导程序所在程序集
- 使用弱绑定允许用户覆盖

#### ViewManager 绑定
```csharp
builder.Bind<IViewManager>().And<ViewManager>().To<ViewManager>().InSingletonScope().AsWeakBinding();
```

**双重绑定策略：**
- 同时绑定到接口和具体类型
- 单例生命周期管理
- 支持通过接口或具体类型获取实例

#### WindowManager 配置
```csharp
builder.Bind<IWindowManagerConfig>().ToInstance(this).DisposeWithContainer(false).AsWeakBinding();
builder.Bind<IWindowManager>().To<WindowManager>().InSingletonScope().AsWeakBinding();
```

**配置分离：**
- 将引导程序自身作为窗口管理器配置
- 不显式处置配置实例（DisposeWithContainer(false)）
- 窗口管理器单例绑定

#### EventAggregator 绑定
```csharp
builder.Bind<IEventAggregator>().To<EventAggregator>().InSingletonScope().AsWeakBinding();
```

**事件聚合：**
- 单例模式确保全局事件总线
- 弱绑定允许自定义实现

#### MessageBox 支持
```csharp
builder.Bind<IMessageBoxViewModel>().To<MessageBoxViewModel>().AsWeakBinding();
builder.Bind<MessageBoxView>().ToSelf();
```

**消息框集成：**
- 提供默认消息框视图模型
- 显式绑定消息框视图
- 支持 IoC 容器创建消息框组件

#### 自动绑定
```csharp
builder.Autobind();
```

**智能绑定：**
- 自动发现和绑定程序集中的类型
- 基于约定的绑定策略
- 减少手动配置工作

### 3. ConfigureIoC - 用户自定义配置

```csharp
protected virtual void ConfigureIoC(IStyletIoCBuilder builder) { }
```

**设计特点：**
- 虚方法允许子类覆盖
- 默认实现为空，不强制任何配置
- 在默认配置之前调用，允许完全自定义

### 4. GetInstance - 实例获取

```csharp
public override object GetInstance(Type type)
{
    return this.Container.Get(type);
}
```

**功能：**
- 重写基类方法，使用 IoC 容器创建实例
- 为 ViewManager 等提供实例创建功能
- 支持依赖注入的实例创建

### 5. Dispose - 资源清理

```csharp
public override void Dispose()
{
    base.Dispose();
    
    // Dispose the container last
    if (this.Container != null)
        this.Container.Dispose();
}
```

**生命周期管理：**
- 先调用基类清理逻辑
- 最后处置 IoC 容器
- 确保所有依赖项正确清理

## 设计优势

### 1. 配置灵活性
- **分层配置**：用户配置和默认配置分离
- **弱绑定支持**：允许覆盖默认服务实现
- **程序集扫描**：自动发现和绑定类型

### 2. 生命周期管理
- **单例服务**：核心服务使用单例生命周期
- **正确处置**：确保容器和服务的正确清理
- **依赖跟踪**：容器管理所有依赖关系

### 3. 可扩展性
- **虚方法**：允许自定义配置逻辑
- **接口编程**：通过接口解耦具体实现
- **约定优于配置**：自动绑定减少配置工作

### 3. 集成性
- **框架服务集成**：提供所有核心 Stylet 服务
- **ViewManager 集成**：支持视图创建和管理的依赖注入
- **事件系统**：集成事件聚合器支持

## 使用模式

### 基本使用

```csharp
public class Bootstrapper : StyletIoCBootstrapperBase
{
    protected override void ConfigureIoC(IStyletIoCBuilder builder)
    {
        // 注册自定义服务
        builder.Bind<IMyService>().To<MyService>().InSingletonScope();
        builder.Bind<IDataRepository>().To<DataRepository>().InTransientScope();
    }
}
```

### 覆盖默认配置

```csharp
public class CustomBootstrapper : StyletIoCBootstrapperBase
{
    protected override void DefaultConfigureIoC(StyletIoCBuilder builder)
    {
        // 先调用基类配置
        base.DefaultConfigureIoC(builder);
        
        // 覆盖默认服务
        builder.Bind<IViewManager>().To<CustomViewManager>().InSingletonScope();
    }
}
```

### 自定义 ViewManagerConfig

```csharp
public class AppBootstrapper : StyletIoCBootstrapperBase
{
    protected override void DefaultConfigureIoC(StyletIoCBuilder builder)
    {
        // 自定义视图管理器配置
        var config = new ViewManagerConfig
        {
            ViewFactory = this.GetInstance,
            ViewAssemblies = new List<Assembly> 
            { 
                this.GetType().Assembly,
                typeof(SomeOtherType).Assembly 
            }
        };
        builder.Bind<ViewManagerConfig>().ToInstance(config);
        
        base.DefaultConfigureIoC(builder);
    }
}
```

## 最佳实践

### 1. 服务注册
- **接口优先**：优先注册接口而不是具体类型
- **生命周期选择**：根据服务性质选择合适的生命周期
- **模块化组织**：按功能模块组织服务注册

### 2. 配置顺序
- **自定义覆盖**：在 ConfigureIoC 中覆盖默认服务
- **程序集管理**：明确指定需要扫描的程序集
- **弱绑定使用**：理解弱绑定的含义和使用场景

### 3. 生命周期管理
- **单例服务**：确保单例服务的线程安全性
- **瞬态服务**：理解瞬态服务的创建成本
- **资源清理**：在 Dispose 中清理非托管资源

### 4. 测试支持
- **可测试性**：通过接口和依赖注入提高可测试性
- **模拟支持**：为关键服务提供模拟实现
- **配置隔离**：为测试提供独立的配置

## 与其他引导程序的比较

### vs BootstrapperBase
- **IoC 集成**：提供内置的 IoC 容器支持
- **默认服务**：提供完整的默认服务配置
- **易用性**：减少手动配置工作

### vs Bootstrapper<TRootViewModel>
- **无根视图模型**：不强制要求指定根视图模型
- **灵活性**：适用于不需要主窗口的应用场景
- **基础功能**：提供核心的 IoC 和基础设施支持

## 总结

StyletIoCBootstrapperBase 通过精心设计的配置流程、默认服务绑定和扩展机制，为 Stylet 应用程序提供了完整的依赖注入基础设施。它的分层配置策略、弱绑定支持和自动绑定功能，使得应用程序既能快速启动，又能灵活定制。作为 Stylet 框架的重要组成部分，它体现了依赖注入最佳实践，是构建可维护、可测试 WPF 应用程序的理想选择。