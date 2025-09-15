# Stylet Bootstrapper机制详解

## 概述

Stylet的Bootstrapper（引导程序）是整个框架的核心启动机制，负责应用程序的初始化、IoC容器的配置、视图管理器的设置以及根视图的显示。它提供了一个统一的应用程序启动流程，支持多种IoC容器，并允许开发者通过简单的继承和配置来自定义应用程序的启动行为。

## 核心架构

### 1. Bootstrapper类层次结构

```
IBootstrapper (接口)
    ↓
BootstrapperBase (抽象基类)
    ↓
StyletIoCBootstrapperBase (StyletIoC特定基类)
    ↓
Bootstrapper<TRootViewModel> (泛型实现)
```

**同时还有其他IoC容器的实现：**
- AutofacBootstrapper<TRootViewModel>
- CastleWindsorBootstrapper<TRootViewModel>
- NinjectBootstrapper<TRootViewModel>
- MicrosoftDependencyInjectionBootstrapper<TRootViewModel>
- NoIoCContainerBootstrapper

## 核心接口和基类

### 1. IBootstrapper接口

```csharp
// 文件：Stylet\IBootstrapper.cs
/// <summary>
/// 指定由ApplicationLoader加载的bootstrapper的接口
/// </summary>
public interface IBootstrapper
{
    /// <summary>
    /// 在应用程序设置期间调用，允许bootstrapper配置自身
    /// 应该钩入Application.Startup事件
    /// </summary>
    /// <param name="application">当前应用程序的引用</param>
    void Setup(Application application);
}
```

### 2. BootstrapperBase抽象基类

```csharp
// 文件：Stylet\BootstrapperBase.cs
/// <summary>
/// 不想使用StyletIoC作为IoC容器的应用程序要扩展的Bootstrapper基类
/// </summary>
public abstract class BootstrapperBase : IBootstrapper, IWindowManagerConfig, IDisposable
{
    /// <summary>
    /// 获取当前应用程序
    /// </summary>
    public Application Application { get; private set; }

    /// <summary>
    /// 获取从命令提示符或桌面传递给应用程序的命令行参数
    /// </summary>
    public string[] Args { get; private set; }

    /// <summary>
    /// 由ApplicationLoader调用，配置bootstrapper与应用程序的关联
    /// </summary>
    public void Setup(Application application)
    {
        this.Application = application ?? throw new ArgumentNullException(nameof(application));

        // 为Execute使用当前应用程序的调度器
        Execute.Dispatcher = new ApplicationDispatcher(this.Application.Dispatcher);

        // 绑定应用程序事件
        this.Application.Startup += (o, e) => this.Start(e.Args);
        this.Application.Exit += (o, e) =>
        {
            this.OnExit(e);
            this.Dispose();
        };

        // 处理未处理的异常
        this.Application.DispatcherUnhandledException += (o, e) =>
        {
            LogManager.GetLogger(typeof(BootstrapperBase)).Error(e.Exception, "Unhandled exception");
            this.OnUnhandledException(e);
        };
    }
}
```

### 3. 应用程序启动流程

```csharp
// 文件：Stylet\BootstrapperBase.cs
/// <summary>
/// 在Application.Startup时调用，执行启动应用程序所需的一切
/// </summary>
public virtual void Start(string[] args)
{
    // 1. 设置命令行参数
    this.Args = args;
    this.OnStart();

    // 2. 配置引导程序（设置IoC容器）
    this.ConfigureBootstrapper();

    // 3. 将ViewManager注册到应用程序资源
    this.Application?.Resources.Add(View.ViewManagerResourceKey, this.GetInstance(typeof(IViewManager)));

    // 4. 执行用户自定义配置
    this.Configure();
    
    // 5. 启动应用程序（显示根视图）
    this.Launch();
    
    // 6. 启动完成后的钩子
    this.OnLaunch();
}
```

## StyletIoC集成

### 1. StyletIoCBootstrapperBase

```csharp
// 文件：Stylet\StyletIoCBootstrapperBase.cs
/// <summary>
/// 想要使用StyletIoC但没有根ViewModel的应用程序要扩展的Bootstrapper
/// </summary>
public abstract class StyletIoCBootstrapperBase : BootstrapperBase
{
    /// <summary>
    /// 获取或设置Bootstrapper的IoC容器
    /// </summary>
    protected IContainer Container { get; set; }
    
    /// <summary>
    /// 重写自BootstrapperBase，设置IoC容器
    /// </summary>
    protected sealed override void ConfigureBootstrapper()
    {
        var builder = new StyletIoCBuilder();
        builder.Assemblies = new List<Assembly> { this.GetType().Assembly };

        // 先调用用户配置，再调用默认配置，这样用户可以自定义builder.Assemblies
        this.ConfigureIoC(builder);
        this.DefaultConfigureIoC(builder);

        this.Container = builder.BuildContainer();
    }

    /// <summary>
    /// 执行StyletIoC的默认配置
    /// </summary>
    protected virtual void DefaultConfigureIoC(StyletIoCBuilder builder)
    {
        // 配置ViewManager
        var viewManagerConfig = new ViewManagerConfig()
        {
            ViewFactory = this.GetInstance,
            ViewAssemblies = new List<Assembly> { this.GetType().Assembly }
        };
        builder.Bind<ViewManagerConfig>().ToInstance(viewManagerConfig).AsWeakBinding();
        builder.Bind<IViewManager>().And<ViewManager>().To<ViewManager>().InSingletonScope().AsWeakBinding();

        // 配置窗口管理器
        builder.Bind<IWindowManagerConfig>().ToInstance(this).DisposeWithContainer(false).AsWeakBinding();
        builder.Bind<IWindowManager>().To<WindowManager>().InSingletonScope().AsWeakBinding();
        
        // 配置事件聚合器
        builder.Bind<IEventAggregator>().To<EventAggregator>().InSingletonScope().AsWeakBinding();
        
        // 配置消息框
        builder.Bind<IMessageBoxViewModel>().To<MessageBoxViewModel>().AsWeakBinding();
        builder.Bind<MessageBoxView>().ToSelf();

        // 自动绑定程序集中的类型
        builder.Autobind();
    }

    /// <summary>
    /// 用户重写此方法来添加自己的类型到IoC容器
    /// </summary>
    protected virtual void ConfigureIoC(IStyletIoCBuilder builder) { }

    /// <summary>
    /// 使用IoC容器获取指定类型的实例
    /// </summary>
    public override object GetInstance(Type type)
    {
        return this.Container.Get(type);
    }
}
```

### 2. 标准Bootstrapper实现

```csharp
// 文件：Stylet\Bootstrapper.cs
/// <summary>
/// 使用StyletIoC的标准应用程序Bootstrapper
/// </summary>
/// <typeparam name="TRootViewModel">根ViewModel的类型</typeparam>
public abstract class Bootstrapper<TRootViewModel> : StyletIoCBootstrapperBase where TRootViewModel : class
{
    private TRootViewModel _rootViewModel;

    /// <summary>
    /// 获取根ViewModel，必要时先创建它
    /// </summary>
    protected virtual TRootViewModel RootViewModel => 
        this._rootViewModel ??= this.Container.Get<TRootViewModel>();

    /// <summary>
    /// 应用程序启动时调用，显示根视图
    /// </summary>
    protected override void Launch()
    {
        this.DisplayRootView(this.RootViewModel);
    }

    /// <summary>
    /// 释放资源
    /// </summary>
    public override void Dispose()
    {
        // 如果根ViewModel还不存在，就不要创建它
        ScreenExtensions.TryDispose(this._rootViewModel);
        base.Dispose();
    }
}
```

## 其他IoC容器集成

### 1. AutofacBootstrapper示例

```csharp
// 文件：Bootstrappers\AutofacBootstrapper.cs
public class AutofacBootstrapper<TRootViewModel> : BootstrapperBase where TRootViewModel : class
{
    private IContainer container;
    private TRootViewModel _rootViewModel;
    
    protected virtual TRootViewModel RootViewModel => 
        this._rootViewModel ??= (TRootViewModel)this.GetInstance(typeof(TRootViewModel));

    protected override void ConfigureBootstrapper()
    {
        var builder = new ContainerBuilder();
        this.DefaultConfigureIoC(builder);
        this.ConfigureIoC(builder);
        this.container = builder.Build();
    }

    /// <summary>
    /// 执行IoC容器的默认配置
    /// </summary>
    protected virtual void DefaultConfigureIoC(ContainerBuilder builder)
    {
        // 配置ViewManager
        var viewManagerConfig = new ViewManagerConfig()
        {
            ViewFactory = this.GetInstance,
            ViewAssemblies = new List<Assembly> { this.GetType().Assembly }
        };
        builder.RegisterInstance<IViewManager>(new ViewManager(viewManagerConfig));
        builder.RegisterType<MessageBoxView>();

        // 配置Stylet核心服务
        builder.RegisterInstance<IWindowManagerConfig>(this).ExternallyOwned();
        builder.RegisterType<WindowManager>().As<IWindowManager>().SingleInstance();
        builder.RegisterType<EventAggregator>().As<IEventAggregator>().SingleInstance();
        builder.RegisterType<MessageBoxViewModel>().As<IMessageBoxViewModel>().ExternallyOwned();

        // 自动注册程序集中的类型
        builder.RegisterAssemblyTypes(this.GetType().Assembly)
               .Where(x => !x.Name.Contains("ProcessedByFody"))
               .ExternallyOwned();
    }

    /// <summary>
    /// 用户重写此方法来配置IoC容器
    /// </summary>
    protected virtual void ConfigureIoC(ContainerBuilder builder) { }

    public override object GetInstance(Type type)
    {
        return this.container.Resolve(type);
    }

    protected override void Launch()
    {
        base.DisplayRootView(this.RootViewModel);
    }
}
```

### 2. NoIoCContainerBootstrapper示例

```csharp
// 文件：Bootstrappers\NoIoCContainerBootstrapper.cs
/// <summary>
/// 不使用IoC容器的简单Bootstrapper实现
/// </summary>
public abstract class NoIoCContainerBootstrapper : BootstrapperBase
{
    protected readonly Dictionary<Type, Func<object>> Container = new();

    protected override void ConfigureBootstrapper()
    {
        this.DefaultConfigureContainer();
        this.ConfigureContainer();
    }

    protected abstract object RootViewModel { get; }

    protected virtual void DefaultConfigureContainer()
    {
        // 手动配置Stylet服务
        var viewManagerConfig = new ViewManagerConfig()
        {
            ViewFactory = this.GetInstance,
            ViewAssemblies = new List<Assembly> { this.GetType().Assembly }
        };
        var viewManager = new ViewManager(viewManagerConfig);
        this.Container.Add(typeof(IViewManager), () => viewManager);

        var windowManager = new WindowManager(viewManager, 
            () => (IMessageBoxViewModel)this.Container[typeof(IMessageBoxViewModel)](), this);
        this.Container.Add(typeof(IWindowManager), () => windowManager);

        var eventAggregator = new EventAggregator();
        this.Container.Add(typeof(IEventAggregator), () => eventAggregator);

        this.Container.Add(typeof(IMessageBoxViewModel), () => new MessageBoxViewModel());
        this.Container.Add(typeof(MessageBoxView), () => new MessageBoxView());
    }

    /// <summary>
    /// 用户重写此方法来添加自己的类型
    /// </summary>
    protected virtual void ConfigureContainer() { }

    public override object GetInstance(Type type)
    {
        if (this.Container.TryGetValue(type, out Func<object> factory))
            return factory();
        else
            return null;
    }

    protected override void Launch()
    {
        base.DisplayRootView(this.RootViewModel);
    }
}
```

## ApplicationLoader机制

### 1. ApplicationLoader类

```csharp
// 文件：Stylet\Xaml\ApplicationLoader.cs
/// <summary>
/// 添加到App.xaml中，负责加载指定的Bootstrapper和Stylet的其他资源
/// </summary>
public class ApplicationLoader : ResourceDictionary
{
    private readonly ResourceDictionary styletResourceDictionary;

    public ApplicationLoader()
    {
        // 加载Stylet的资源字典
        this.styletResourceDictionary = new ResourceDictionary() 
        { 
            Source = new Uri("pack://application:,,,/Stylet;component/Xaml/StyletResourceDictionary.xaml", UriKind.Absolute) 
        };
        this.LoadStyletResources = true;
    }

    private IBootstrapper _bootstrapper;

    /// <summary>
    /// 获取或设置用于启动应用程序的bootstrapper实例
    /// </summary>
    public IBootstrapper Bootstrapper
    {
        get => this._bootstrapper;
        set
        {
            this._bootstrapper = value;
            // 关键：设置Bootstrapper时立即调用Setup
            this._bootstrapper.Setup(Application.Current);
        }
    }

    /// <summary>
    /// 是否加载Stylet自己的资源（如StyletConductorTabControl）
    /// </summary>
    public bool LoadStyletResources
    {
        get => this._loadStyletResources;
        set
        {
            this._loadStyletResources = value;
            if (this._loadStyletResources)
                this.MergedDictionaries.Add(this.styletResourceDictionary);
            else
                this.MergedDictionaries.Remove(this.styletResourceDictionary);
        }
    }
}
```

### 2. App.xaml中的配置

```xml
<!-- 典型的App.xaml配置 -->
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

## 完整启动流程分析

### 步骤1：应用程序启动和XAML解析
```
1. WPF应用程序启动
2. App.xaml被解析
3. ApplicationLoader被创建并添加到Application.Resources
4. HelloBootstrapper实例被创建
5. ApplicationLoader.Bootstrapper属性被设置
```

### 步骤2：Bootstrapper设置
```csharp
// 当设置Bootstrapper属性时
public IBootstrapper Bootstrapper
{
    set
    {
        this._bootstrapper = value;
        // 立即调用Setup，建立与Application的关联
        this._bootstrapper.Setup(Application.Current);
    }
}
```

### 步骤3：Setup方法执行
```csharp
public void Setup(Application application)
{
    this.Application = application;
    
    // 设置调度器
    Execute.Dispatcher = new ApplicationDispatcher(this.Application.Dispatcher);

    // 绑定应用程序事件
    this.Application.Startup += (o, e) => this.Start(e.Args);
    this.Application.Exit += (o, e) => { this.OnExit(e); this.Dispose(); };
    this.Application.DispatcherUnhandledException += (o, e) => this.OnUnhandledException(e);
}
```

### 步骤4：Application.Startup事件触发
```
Application.Startup事件被触发 → Bootstrapper.Start(args)被调用
```

### 步骤5：Start方法执行完整流程
```csharp
public virtual void Start(string[] args)
{
    // 1. 设置命令行参数
    this.Args = args;
    this.OnStart();           // 用户钩子：启动前

    // 2. 配置IoC容器
    this.ConfigureBootstrapper();
    
    // 3. 注册ViewManager到应用程序资源（关键！）
    this.Application?.Resources.Add(View.ViewManagerResourceKey, this.GetInstance(typeof(IViewManager)));

    // 4. 用户配置
    this.Configure();         // 用户钩子：IoC容器配置后

    // 5. 启动应用程序
    this.Launch();            // 显示根视图

    // 6. 启动完成
    this.OnLaunch();          // 用户钩子：启动后
}
```

### 步骤6：IoC容器配置（StyletIoC示例）
```csharp
protected sealed override void ConfigureBootstrapper()
{
    var builder = new StyletIoCBuilder();
    builder.Assemblies = new List<Assembly> { this.GetType().Assembly };

    // 用户配置（可以修改Assemblies等）
    this.ConfigureIoC(builder);
    
    // 默认配置（注册Stylet核心服务）
    this.DefaultConfigureIoC(builder);

    // 构建容器
    this.Container = builder.BuildContainer();
}
```

### 步骤7：默认服务注册
```csharp
protected virtual void DefaultConfigureIoC(StyletIoCBuilder builder)
{
    // ViewManager配置
    var viewManagerConfig = new ViewManagerConfig()
    {
        ViewFactory = this.GetInstance,  // 使用IoC容器创建View
        ViewAssemblies = new List<Assembly> { this.GetType().Assembly }
    };
    builder.Bind<IViewManager>().To<ViewManager>().InSingletonScope();

    // 窗口管理器
    builder.Bind<IWindowManagerConfig>().ToInstance(this);
    builder.Bind<IWindowManager>().To<WindowManager>().InSingletonScope();
    
    // 事件聚合器
    builder.Bind<IEventAggregator>().To<EventAggregator>().InSingletonScope();
    
    // 消息框服务
    builder.Bind<IMessageBoxViewModel>().To<MessageBoxViewModel>();
    builder.Bind<MessageBoxView>().ToSelf();

    // 自动绑定用户类型
    builder.Autobind();
}
```

### 步骤8：ViewManager注册到应用程序资源
```csharp
// 这是关键步骤！使得View.Model绑定能够工作
this.Application?.Resources.Add(View.ViewManagerResourceKey, this.GetInstance(typeof(IViewManager)));

// View.ViewManagerResourceKey = "b9a38199-8cb3-4103-8526-c6cfcd089df7"
// 这样View.cs中的附加属性就能找到ViewManager实例
```

### 步骤9：Launch - 显示根视图
```csharp
protected override void Launch()
{
    // 从IoC容器获取根ViewModel
    this.DisplayRootView(this.RootViewModel);
}

protected virtual void DisplayRootView(object rootViewModel)
{
    // 获取WindowManager并显示根窗口
    var windowManager = (IWindowManager)this.GetInstance(typeof(IWindowManager));
    windowManager.ShowWindow(rootViewModel);
}
```

### 步骤10：WindowManager显示窗口
```csharp
// WindowManager内部流程：
// 1. 使用ViewManager根据ShellViewModel查找ShellView
// 2. 创建ShellView实例
// 3. 绑定ShellView和ShellViewModel
// 4. 显示窗口
windowManager.ShowWindow(shellViewModel);
```

## 自定义Bootstrapper示例

### 1. 基础自定义

```csharp
// 最简单的Bootstrapper
public class HelloBootstrapper : Bootstrapper<ShellViewModel>
{
    // 只需要指定根ViewModel类型，其他都是默认配置
}
```

### 2. 添加自定义服务

```csharp
public class CustomBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(StyletIoC.IStyletIoCBuilder builder)
    {
        base.ConfigureIoC(builder);

        // 添加自定义服务
        builder.Bind<IDialogFactory>().ToAbstractFactory();
        builder.Bind<IDataService>().To<DataService>().InSingletonScope();
        builder.Bind<ISettings>().ToInstance(LoadSettings());
    }

    protected override void OnStart()
    {
        base.OnStart();
        // 启动前的自定义逻辑
        InitializeLogging();
    }

    protected override void Configure()
    {
        base.Configure();
        // IoC容器配置后的自定义逻辑
        ConfigureDatabase();
    }

    protected override void OnLaunch()
    {
        base.OnLaunch();
        // 应用程序启动后的自定义逻辑
        StartBackgroundServices();
    }

    protected override void OnExit(ExitEventArgs e)
    {
        // 应用程序退出时的清理逻辑
        CleanupResources();
        base.OnExit(e);
    }
}
```

### 3. 自定义ViewManager

```csharp
public class CustomBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(StyletIoC.IStyletIoCBuilder builder)
    {
        base.ConfigureIoC(builder);

        // 替换默认的ViewManager
        builder.Bind<IViewManager>().To<CustomViewManager>().InSingletonScope();
    }
}
```

### 4. 多程序集支持

```csharp
public class MultiAssemblyBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(StyletIoC.IStyletIoCBuilder builder)
    {
        // 添加额外的程序集用于自动绑定和View查找
        builder.Assemblies.Add(typeof(SomeOtherAssemblyType).Assembly);
        
        base.ConfigureIoC(builder);
    }
}
```

## 设计优势

### 1. 统一的启动流程
- 所有类型的应用程序都遵循相同的启动模式
- 提供清晰的生命周期钩子方法

### 2. IoC容器抽象
- 支持多种IoC容器（StyletIoC、Autofac、Castle Windsor等）
- 提供统一的容器配置接口

### 3. 约定优于配置
- 默认配置涵盖大多数常见场景
- 开发者只需要关注业务特定的配置

### 4. 可扩展性
- 提供多个扩展点供用户自定义
- 支持完全替换核心服务（如ViewManager）

### 5. 资源管理
- 自动处理应用程序生命周期
- 正确释放IoC容器和相关资源

## 常见配置模式

### 1. 数据库连接配置
```csharp
protected override void ConfigureIoC(IStyletIoCBuilder builder)
{
    base.ConfigureIoC(builder);
    
    builder.Bind<IDbContext>().To<MyDbContext>()
           .WithConstructorParam("connectionString").ToValue(GetConnectionString());
}
```

### 2. 日志配置
```csharp
protected override void OnStart()
{
    base.OnStart();
    
    LogManager.Enabled = true;
    LogManager.LoggerFactory = name => new NLogLogger(name);
}
```

### 3. 异常处理
```csharp
protected override void OnUnhandledException(DispatcherUnhandledExceptionEventArgs e)
{
    // 记录异常
    LogManager.GetLogger(typeof(Bootstrapper)).Error(e.Exception);
    
    // 显示用户友好的错误消息
    MessageBox.Show("应用程序遇到错误，即将关闭", "错误", MessageBoxButton.OK, MessageBoxImage.Error);
    
    // 标记为已处理，防止应用程序崩溃
    e.Handled = true;
}
```

## 总结

Stylet的Bootstrapper机制通过以下关键组件协同工作：

1. **ApplicationLoader**：XAML中的引导加载器，连接App.xaml和Bootstrapper
2. **IBootstrapper接口**：定义引导程序的基本契约
3. **BootstrapperBase**：提供核心启动流程和应用程序生命周期管理
4. **具体实现**：StyletIoC、Autofac等不同IoC容器的Bootstrapper实现
5. **生命周期钩子**：OnStart、Configure、Launch、OnLaunch、OnExit等扩展点

整个机制的核心价值在于：
- 将复杂的应用程序启动逻辑标准化
- 提供IoC容器无关的统一接口
- 自动配置Stylet框架的核心服务
- 为开发者提供清晰的自定义扩展点

这种设计使得开发者只需要继承`Bootstrapper<TRootViewModel>`并指定根ViewModel类型，就能快速启动一个功能完整的Stylet应用程序，同时保留了充分的自定义空间。
