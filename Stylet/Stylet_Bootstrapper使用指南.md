# Stylet Bootstrapper 使用指南

本文档基于 `Stylet.Samples.Hello` 项目，详细讲解 Stylet 框架中 Bootstrapper 的使用方法和原理。

## 1. Bootstrapper 概述

Bootstrapper 是 Stylet 应用程序的启动器，负责：
- 初始化 IoC 容器
- 配置应用程序
- 启动主窗口
- 处理应用程序生命周期事件

## 2. 基本结构

### 2.1 HelloBootstrapper 类

```csharp
public class HelloBootstrapper : Bootstrapper<ShellViewModel>
{
}
```

这是一个最简化的 Bootstrapper 实现，继承自 `Bootstrapper<TRootViewModel>` 泛型类，其中：
- `TRootViewModel` 是主窗口对应的 ViewModel 类型
- 本例中为 `ShellViewModel`

### 2.2 继承关系

```
Bootstrapper<TRootViewModel>
    └── StyletIoCBootstrapperBase
        └── BootstrapperBase
            └── IBootstrapper
```

## 3. 配置启动流程

### 3.1 App.xaml 配置

```xml
<Application x:Class="Stylet.Samples.Hello.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:s="https://github.com/canton7/Stylet"
             xmlns:local="clr-namespace:Stylet.Samples.Hello">
    <Application.Resources>
        <s:ApplicationLoader>
            <s:ApplicationLoader.Bootstrapper>
                <local:HelloBootstrapper/>
            </s:ApplicationLoader.Bootstrapper>
        </s:ApplicationLoader>
    </Application.Resources>
</Application>
```

关键配置说明：
- `ApplicationLoader` 是 Stylet 提供的启动加载器
- `Bootstrapper` 属性指定使用的 Bootstrapper 实例
- 使用 `local` 命名空间引用项目中的 `HelloBootstrapper`

### 3.2 启动流程

1. **应用程序启动**：WPF 应用程序启动
2. **ApplicationLoader 初始化**：解析 App.xaml 中的配置
3. **Bootstrapper 实例化**：创建 HelloBootstrapper 实例
4. **Setup 阶段**：调用 BootstrapperBase.Setup() 方法
5. **Start 阶段**：调用 BootstrapperBase.Start() 方法
6. **配置 IoC 容器**：设置依赖注入
7. **显示主窗口**：创建并显示 ShellView

## 4. Bootstrapper 核心功能

### 4.1 BootstrapperBase 核心方法

#### 生命周期方法
- `OnStart()`：应用程序启动时调用（最早）
- `ConfigureBootstrapper()`：配置 IoC 容器
- `Configure()`：IoC 容器配置完成后调用
- `Launch()`：启动主窗口
- `OnLaunch()`：主窗口显示后调用
- `OnExit()`：应用程序退出时调用
- `OnUnhandledException()`：处理未捕获异常

#### 关键属性
- `Application`：当前 WPF 应用程序实例
- `Args`：命令行参数数组

### 4.2 StyletIoCBootstrapperBase 功能

#### IoC 容器配置
```csharp
protected sealed override void ConfigureBootstrapper()
{
    var builder = new StyletIoCBuilder();
    builder.Assemblies = new List<Assembly>(new List<Assembly>() { this.GetType().Assembly });

    this.ConfigureIoC(builder);        // 用户自定义配置
    this.DefaultConfigureIoC(builder); // 默认配置

    this.Container = builder.BuildContainer();
}
```

#### 默认服务注册
- `IViewManager`：视图管理器
- `IWindowManager`：窗口管理器
- `IEventAggregator`：事件聚合器
- `IMessageBoxViewModel`：消息框 ViewModel

### 4.3 Bootstrapper<TRootViewModel> 功能

#### 主窗口管理
```csharp
protected virtual TRootViewModel RootViewModel => 
    this._rootViewModel ??= this.Container.Get<TRootViewModel>();

protected override void Launch()
{
    this.DisplayRootView(this.RootViewModel);
}
```

## 5. 自定义 Bootstrapper

### 5.1 扩展配置

```csharp
public class HelloBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(IStyletIoCBuilder builder)
    {
        // 注册自定义服务
        builder.Bind<IMyService>().To<MyService>().InSingletonScope();
        
        // 注册配置
        builder.Bind<IConfiguration>().ToInstance(new AppConfiguration());
    }
    
    protected override void OnStart()
    {
        // 应用程序启动时的初始化
        LogManager.Enabled = true;
    }
    
    protected override void OnLaunch()
    {
        // 主窗口显示后的操作
        Logger.Info("应用程序已启动");
    }
}
```

### 5.2 重写启动行为

```csharp
public class CustomBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void Launch()
    {
        // 自定义启动逻辑
        var windowManager = this.Container.Get<IWindowManager>();
        var shell = this.RootViewModel;
        
        // 设置窗口属性
        dynamic settings = new ExpandoObject();
        settings.Title = "自定义标题";
        settings.SizeToContent = SizeToContent.WidthAndHeight;
        
        windowManager.ShowWindow(shell, settings);
    }
}
```

## 6. 依赖注入使用

### 6.1 在 ViewModel 中使用

```csharp
public class ShellViewModel : Screen
{
    private readonly IWindowManager windowManager;
    
    public ShellViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
        this.DisplayName = "Hello, Stylet";
    }
    
    public void SayHello()
    {
        this.windowManager.ShowMessageBox($"Hello, {this.Name}");
    }
}
```

### 6.2 服务注册方式

#### 单例模式
```csharp
builder.Bind<IMyService>().To<MyService>().InSingletonScope();
```

#### 瞬态模式
```csharp
builder.Bind<IMyService>().To<MyService>();
```

#### 实例注册
```csharp
builder.Bind<IConfiguration>().ToInstance(new AppConfiguration());
```

## 7. 高级配置

### 7.1 多窗口管理

```csharp
public class MultiWindowBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(IStyletIoCBuilder builder)
    {
        base.ConfigureIoC(builder);
        
        // 注册多个窗口
        builder.Bind<ChildViewModel>().ToSelf();
        builder.Bind<SettingsViewModel>().ToSelf();
    }
}
```

### 7.2 自定义 ViewManager

```csharp
public class CustomBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(IStyletIoCBuilder builder)
    {
        base.ConfigureIoC(builder);
        
        // 替换默认的 ViewManager
        builder.Bind<IViewManager>().To<CustomViewManager>().InSingletonScope();
    }
}
```

### 7.3 异常处理

```csharp
public class RobustBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void OnUnhandledException(DispatcherUnhandledExceptionEventArgs e)
    {
        // 记录异常
        Logger.Error(e.Exception, "未处理的异常");
        
        // 显示错误对话框
        var windowManager = this.Container.Get<IWindowManager>();
        windowManager.ShowMessageBox(
            $"发生错误：{e.Exception.Message}", 
            "错误", 
            MessageBoxButton.OK, 
            MessageBoxImage.Error);
        
        e.Handled = true;
    }
}
```

## 8. 最佳实践

### 8.1 项目结构建议

```
MyApp/
├── App.xaml
├── Bootstrapper.cs
├── ViewModels/
│   ├── ShellViewModel.cs
│   └── ...
├── Views/
│   ├── ShellView.xaml
│   └── ...
├── Services/
│   ├── IMyService.cs
│   └── MyService.cs
└── Models/
    └── ...
```

### 8.2 配置分离

```csharp
public class AppBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(IStyletIoCBuilder builder)
    {
        // 服务注册
        this.RegisterServices(builder);
        
        // 配置注册
        this.RegisterConfiguration(builder);
        
        // 视图模型注册
        this.RegisterViewModels(builder);
    }
    
    private void RegisterServices(IStyletIoCBuilder builder)
    {
        builder.Bind<IDataService>().To<DataService>().InSingletonScope();
        builder.Bind<IApiClient>().To<ApiClient>().InSingletonScope();
    }
    
    private void RegisterConfiguration(IStyletIoCBuilder builder)
    {
        var config = ConfigurationManager.LoadConfiguration();
        builder.Bind<IAppConfiguration>().ToInstance(config);
    }
}
```

## 9. 调试技巧

### 9.1 启用日志

```csharp
protected override void OnStart()
{
    LogManager.Enabled = true;
    LogManager.LoggerFactory = type => new TraceLogger(type);
}
```

### 9.2 调试 IoC 注册

```csharp
protected override void ConfigureIoC(IStyletIoCBuilder builder)
{
    // 添加调试输出
    var originalAssemblies = builder.Assemblies;
    Console.WriteLine($"注册程序集: {string.Join(", ", originalAssemblies.Select(a => a.GetName().Name))}");
    
    base.ConfigureIoC(builder);
}
```

## 10. 常见问题

### 10.1 无法解析依赖
确保所有需要的服务都在 `ConfigureIoC` 中注册。

### 10.2 窗口不显示
检查 `TRootViewModel` 是否正确指定，以及对应的 View 是否存在。

### 10.3 设计时支持
使用 `DesignMode` 检测设计时状态：

```csharp
public ShellViewModel()
{
    if (Execute.InDesignMode)
    {
        // 设计时数据
        this.Name = "设计时名称";
    }
}
```

通过以上指南，您应该能够熟练使用 Stylet 的 Bootstrapper 来构建 WPF 应用程序。Bootstrapper 提供了简洁而强大的应用程序启动和配置机制，是 Stylet 框架的核心组件之一。