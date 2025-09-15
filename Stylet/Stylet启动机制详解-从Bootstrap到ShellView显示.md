# Stylet启动机制详解：从Bootstrap到ShellView显示

## 前言

本文档以`Stylet.Samples.Hello`为例，深入剖析Stylet框架的启动机制，解释为什么`ShellView`会作为第一个窗口显示。通过详细的源码分析和流程说明，帮助新手理解Stylet的设计理念和实现细节。

## 项目结构概览

首先，让我们看看`Stylet.Samples.Hello`项目的结构：

```
Stylet.Samples.Hello/
├── App.xaml                 # 应用程序入口点
├── HelloBootstrapper.cs     # 启动配置类
├── ShellView.xaml          # 主窗口视图
├── ShellViewModel.cs       # 主窗口视图模型
└── Stylet.Samples.Hello.csproj
```

## 启动流程全景图

Stylet的启动流程可以分为以下几个关键阶段：

1. **WPF应用程序启动**（Application.Startup）
2. **ApplicationLoader初始化**（XAML解析阶段）
3. **Bootstrapper配置与启动**
4. **IoC容器初始化**
5. **根视图模型创建**
6. **视图定位与创建**
7. **窗口显示**

让我们逐步深入每个阶段。

## 阶段1：WPF应用程序启动

### App.xaml的关键作用

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

**关键点解析：**

- `ApplicationLoader`是Stylet提供的特殊资源字典，继承自`ResourceDictionary`
- 当WPF解析App.xaml时，会创建`ApplicationLoader`实例
- `ApplicationLoader`的构造函数会立即执行，加载Stylet的基础资源

### ApplicationLoader源码分析

```csharp
public class ApplicationLoader : ResourceDictionary
{
    private readonly ResourceDictionary styletResourceDictionary;
    
    public ApplicationLoader()
    {
        // 加载Stylet的基础样式资源
        this.styletResourceDictionary = new ResourceDictionary() { 
            Source = new Uri("pack://application:,,,/Stylet;component/Xaml/StyletResourceDictionary.xaml", UriKind.Absolute) 
        };
        this.LoadStyletResources = true;
    }

    public IBootstrapper Bootstrapper
    {
        get => this._bootstrapper;
        set
        {
            this._bootstrapper = value;
            // 关键：立即调用Setup方法
            this._bootstrapper.Setup(Application.Current);
        }
    }
}
```

当XAML解析器遇到`<local:HelloBootstrapper/>`时，会：
1. 创建`HelloBootstrapper`实例
2. 设置给`ApplicationLoader.Bootstrapper`属性
3. 触发`Bootstrapper.Setup(Application.Current)`调用

## 阶段2：Bootstrapper配置与启动

### HelloBootstrapper的极简设计

```csharp
public class HelloBootstrapper : Bootstrapper<ShellViewModel>
{
    // 空实现！所有功能都在基类中
}
```

虽然`HelloBootstrapper`看起来很简单，但它继承的`Bootstrapper<ShellViewModel>`提供了完整的启动逻辑。

### Bootstrapper继承链

```
HelloBootstrapper
    ↓
Bootstrapper<ShellViewModel>
    ↓
StyletIoCBootstrapperBase
    ↓
BootstrapperBase
```

### BootstrapperBase.Setup方法详解

```csharp
public void Setup(Application application)
{
    this.Application = application;
    
    // 设置UI线程调度器
    Execute.Dispatcher = new ApplicationDispatcher(this.Application.Dispatcher);

    // 关键：注册Startup事件
    this.Application.Startup += (o, e) => this.Start(e.Args);
    
    // 注册其他生命周期事件
    this.Application.Exit += (o, e) =>
    {
        this.OnExit(e);
        this.Dispose();
    };
    
    this.Application.DispatcherUnhandledException += (o, e) =>
    {
        LogManager.GetLogger(typeof(BootstrapperBase)).Error(e.Exception, "Unhandled exception");
        this.OnUnhandledException(e);
    };
}
```

**重要说明：**
- `Setup`方法在XAML解析阶段就被调用
- 但实际的启动逻辑在`Application.Startup`事件触发时才执行
- 这确保了所有配置都在UI线程上完成

## 阶段3：IoC容器初始化

### Start方法的完整流程

```csharp
public virtual void Start(string[] args)
{
    // 1. 保存命令行参数
    this.Args = args;
    this.OnStart();
    
    // 2. 配置Bootstrapper（主要是IoC容器）
    this.ConfigureBootstrapper();
    
    // 3. 注册ViewManager到应用程序资源
    this.Application?.Resources.Add(View.ViewManagerResourceKey, this.GetInstance(typeof(IViewManager)));
    
    // 4. 用户自定义配置
    this.Configure();
    
    // 5. 显示根视图
    this.Launch();
    
    // 6. 启动完成通知
    this.OnLaunch();
}
```

### StyletIoC容器配置

在`StyletIoCBootstrapperBase`中：

```csharp
protected sealed override void ConfigureBootstrapper()
{
    var builder = new StyletIoCBuilder();
    builder.Assemblies = new List<Assembly>(new List<Assembly>() { this.GetType().Assembly });

    // 用户自定义IoC配置
    this.ConfigureIoC(builder);
    
    // 默认配置
    this.DefaultConfigureIoC(builder);
    
    // 构建容器
    this.Container = builder.BuildContainer();
}

protected virtual void DefaultConfigureIoC(StyletIoCBuilder builder)
{
    // 配置ViewManager
    var viewManagerConfig = new ViewManagerConfig()
    {
        ViewFactory = this.GetInstance,
        ViewAssemblies = new List<Assembly>() { this.GetType().Assembly }
    };
    builder.Bind<ViewManagerConfig>().ToInstance(viewManagerConfig).AsWeakBinding();

    // 注册核心服务
    builder.Bind<IViewManager>().And<ViewManager>().To<ViewManager>().InSingletonScope();
    builder.Bind<IWindowManager>().To<WindowManager>().InSingletonScope();
    builder.Bind<IEventAggregator>().To<EventAggregator>().InSingletonScope();
    
    // 自动绑定特性
    builder.Autobind();
}
```

**注册的默认服务：**
- `IViewManager` → `ViewManager`（视图管理器）
- `IWindowManager` → `WindowManager`（窗口管理器）
- `IEventAggregator` → `EventAggregator`（事件聚合器）

## 阶段4：根视图模型创建

### Bootstrapper<TRootViewModel>的核心逻辑

```csharp
public abstract class Bootstrapper<TRootViewModel> : StyletIoCBootstrapperBase 
    where TRootViewModel : class
{
    private TRootViewModel _rootViewModel;

    // 延迟加载根视图模型
    protected virtual TRootViewModel RootViewModel => 
        this._rootViewModel ??= this.Container.Get<TRootViewModel>();

    // 启动时显示根视图
    protected override void Launch()
    {
        this.DisplayRootView(this.RootViewModel);
    }
}
```

**关键点：**
- 使用延迟加载模式，只有在需要时才创建`TRootViewModel`
- 通过IoC容器解析`ShellViewModel`，支持构造函数注入
- 在`HelloBootstrapper`中，`TRootViewModel`就是`ShellViewModel`

## 阶段5：视图定位与创建

### DisplayRootView方法

```csharp
protected virtual void DisplayRootView(object rootViewModel)
{
    var windowManager = (IWindowManager)this.GetInstance(typeof(IWindowManager));
    windowManager.ShowWindow(rootViewModel);
}
```

### WindowManager的窗口创建流程

```csharp
public void ShowWindow(object viewModel)
{
    this.CreateWindow(viewModel, false, null).Show();
}

protected virtual Window CreateWindow(object viewModel, bool isDialog, IViewAware ownerViewModel)
{
    // 1. 通过ViewManager创建视图
    UIElement view = this.viewManager.CreateAndBindViewForModelIfNecessary(viewModel);
    
    // 2. 确保视图是Window类型
    if (view is not Window window)
    {
        throw new StyletInvalidViewTypeException(...);
    }
    
    // 3. 设置窗口属性
    if (viewModel is IHaveDisplayName haveDisplayName)
    {
        window.SetBinding(Window.TitleProperty, new Binding("DisplayName"));
    }
    
    // 4. 设置窗口位置
    if (window.WindowStartupLocation == WindowStartupLocation.Manual)
    {
        window.WindowStartupLocation = WindowStartupLocation.CenterScreen;
    }
    
    return window;
}
```

### ViewManager的视图创建

```csharp
public virtual UIElement CreateViewForModel(object model)
{
    // 1. 定位视图类型
    Type viewType = this.LocateViewForModel(model.GetType());
    
    // 2. 创建视图实例
    var view = (UIElement)this.ViewFactory(viewType);
    
    // 3. 初始化视图
    this.InitializeView(view, viewType);
    
    return view;
}

protected virtual Type LocateViewForModel(Type modelType)
{
    string modelName = modelType.FullName; // "Stylet.Samples.Hello.ShellViewModel"
    string viewName = this.ViewTypeNameForModelTypeName(modelName);
    // 转换后: "Stylet.Samples.Hello.ShellView"
    
    Type viewType = this.ViewTypeForViewName(viewName, new[] { modelType.Assembly });
    return viewType;
}
```

**命名转换规则：**
- `ShellViewModel` → `ShellView`
- `MainViewModel` → `MainView`
- 规则：将"ViewModel"后缀替换为"View"

## 阶段6：ShellView显示全过程

### 完整的启动时序图

```
WPF Application
      ↓
Application Startup
      ↓
ApplicationLoader.Setup()
      ↓
Bootstrapper.Setup()
      ↓
注册Startup事件
      ↓
Application.Startup触发
      ↓
Bootstrapper.Start()
      ↓
ConfigureBootstrapper()
      ↓
IoC容器构建
      ↓
Launch()调用
      ↓
DisplayRootView(ShellViewModel)
      ↓
WindowManager.ShowWindow()
      ↓
ViewManager.CreateViewForModel()
      ↓
定位ShellView类型
      ↓
创建ShellView实例
      ↓
绑定ShellView到ShellViewModel
      ↓
显示ShellView窗口
```

### ShellViewModel的实例化

当`Container.Get<ShellViewModel>()`被调用时：

```csharp
public ShellViewModel(IWindowManager windowManager)
{
    this.DisplayName = "Hello, Stylet";
    this.windowManager = windowManager;
}
```

StyletIoC会自动注入`IWindowManager`依赖。

### ShellView的创建与绑定

1. **定位视图**：根据`ShellViewModel`找到`ShellView`
2. **创建实例**：使用`Activator.CreateInstance`
3. **初始化**：调用`InitializeComponent()`
4. **数据绑定**：设置`DataContext = ShellViewModel`
5. **显示窗口**：调用`Window.Show()`

## 关键设计模式解析

### 1. 模板方法模式（Template Method）

`BootstrapperBase`使用模板方法模式定义启动流程：

```csharp
public virtual void Start(string[] args)
{
    this.OnStart();           // 可重写
    this.ConfigureBootstrapper(); // 可重写
    this.Configure();         // 可重写
    this.Launch();            // 必须重写
    this.OnLaunch();          // 可重写
}
```

### 2. 工厂模式（Factory Pattern）

`ViewManager`作为视图工厂：
- 根据ViewModel类型创建对应的View
- 隐藏了视图创建的具体细节

### 3. 依赖注入（Dependency Injection）

- 通过构造函数注入服务
- 自动解析依赖关系
- 支持生命周期管理

### 4. 约定优于配置（Convention over Configuration）

- 自动的ViewModel到View的映射
- 统一的命名约定
- 减少配置代码

## 新手常见问题解答

### Q1: 为什么我的窗口没有显示？

**可能原因：**
1. 没有正确继承`Bootstrapper<T>`，而是继承了`BootstrapperBase`
2. 没有重写`Launch`方法
3. ViewModel或View的命名不符合约定

### Q2: 如何自定义启动流程？

**解决方案：**
```csharp
public class CustomBootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void OnStart()
    {
        // 启动前配置
    }
    
    protected override void Configure()
    {
        // 自定义IoC配置
    }
    
    protected override void OnLaunch()
    {
        // 启动后处理
    }
}
```

### Q3: 如何显示不同的初始窗口？

**解决方案：**
```csharp
public class CustomBootstrapper : Bootstrapper<MainViewModel> // 改为其他ViewModel
{
    // 或者重写Launch方法
    protected override void Launch()
    {
        // 显示多个窗口
        this.DisplayRootView(new FirstViewModel());
        this.DisplayRootView(new SecondViewModel());
    }
}
```

## 调试技巧

### 1. 启用详细日志

在App.xaml.cs中添加：

```csharp
protected override void OnStartup(StartupEventArgs e)
{
    Stylet.Logging.LogManager.Enabled = true;
    base.OnStartup(e);
}
```

### 2. 断点调试建议

- 在`Bootstrapper.Start()`设置断点
- 在`ViewManager.CreateViewForModel()`设置断点
- 在`ShellViewModel`构造函数设置断点

### 3. 查看IoC容器内容

```csharp
protected override void Configure()
{
    var container = this.Container;
    // 查看已注册的服务
    var registrations = container.GetCurrentRegistrations();
}
```

## 总结

Stylet的启动机制设计精巧，通过以下方式实现了简洁而强大的启动流程：

1. **声明式配置**：通过XAML配置启动器
2. **约定优于配置**：自动的ViewModel-View映射
3. **依赖注入**：自动的依赖解析
4. **生命周期管理**：完整的应用程序生命周期钩子
5. **可扩展性**：通过继承和重写实现自定义

理解这些机制后，你就能：
- 快速创建新的Stylet应用程序
- 调试启动相关的问题
- 根据需求自定义启动流程
- 更好地利用Stylet的高级特性

记住：Stylet的设计哲学是让简单的事情保持简单，让复杂的事情成为可能。通过理解这些基础机制，你可以构建出更加优雅和可维护的WPF应用程序。