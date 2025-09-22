# Prism.Core项目结构分析

## 概述

Prism.Core是Prism框架的核心库，提供了跨平台的基础功能，不依赖于任何特定的应用程序框架。它是所有Prism平台特定实现的基础，包含了MVVM、命令、导航、模块化等核心功能的抽象和基础实现。本文档详细分析了Prism.Core的项目结构，帮助开发者更好地理解其组织架构和各组件的功能。

## 目录结构

```
src/Prism.Core/
├── Commands/          # 命令模式实现
├── Common/           # 通用工具类
├── Dialogs/          # 对话框服务接口
├── Extensions/       # 扩展方法
├── Modularity/       # 模块化系统
├── Mvvm/             # MVVM模式支持
├── Navigation/       # 导航系统
├── Properties/       # 项目属性和资源
├── build/            # 构建配置
├── GlobalSuppressions.cs  # 全局抑制警告
├── IActiveAware.cs   # 活动状态感知接口
└── Prism.Core.csproj # 项目文件
```

## 核心组件详解

### 1. Commands

Commands目录提供了命令模式的实现，是MVVM模式中连接视图和视图模型的关键组件。

#### 主要组件：

- [`DelegateCommandBase.cs`](src/Prism.Core/Commands/DelegateCommandBase.cs): 委托命令基类，提供了命令的基本实现
- [`DelegateCommand.cs`](src/Prism.Core/Commands/DelegateCommand.cs): 委托命令实现，支持同步操作
- [`DelegateCommand{T}.cs`](src/Prism.Core/Commands/DelegateCommand{T}.cs): 泛型委托命令实现，支持带参数的同步操作
- [`AsyncDelegateCommand.cs`](src/Prism.Core/Commands/AsyncDelegateCommand.cs): 异步委托命令实现，支持异步操作
- [`AsyncDelegateCommand{T}.cs`](src/Prism.Core/Commands/AsyncDelegateCommand{T}.cs): 泛型异步委托命令实现，支持带参数的异步操作
- [`CompositeCommand.cs`](src/Prism.Core/Commands/CompositeCommand.cs): 复合命令实现，支持组合多个命令
- [`IAsyncCommand.cs`](src/Prism.Core/Commands/IAsyncCommand.cs): 异步命令接口
- [`PropertyObserver.cs`](src/Prism.Core/Commands/PropertyObserver.cs): 属性观察者，用于监听属性变化
- [`PropertyObserverNode.cs`](src/Prism.Core/Commands/PropertyObserverNode.cs): 属性观察者节点

#### 功能特点：

- 支持同步和异步命令
- 支持带参数的命令
- 支持命令的启用/禁用状态管理
- 支持复合命令，可以组合多个子命令
- 支持属性变化监听，自动更新命令状态

### 2. Common

Common目录提供了通用的工具类和基础接口，被框架的其他部分广泛使用。

#### 主要组件：

- [`IParameters.cs`](src/Prism.Core/Common/IParameters.cs): 参数接口，定义了参数的基本操作
- [`IRegistryAware.cs`](src/Prism.Core/Common/IRegistryAware.cs): 注册表感知接口，用于对象注册表管理
- [`ListDictionary.cs`](src/Prism.Core/Common/ListDictionary.cs): 列表字典实现，支持一键多值
- [`MulticastExceptionHandler.cs`](src/Prism.Core/Common/MulticastExceptionHandler.cs): 多播异常处理器，支持多个异常处理程序
- [`ParametersBase.cs`](src/Prism.Core/Common/ParametersBase.cs): 参数基类，提供了参数的基本实现
- [`ParametersExtensions.cs`](src/Prism.Core/Common/ParametersExtensions.cs): 参数扩展方法
- [`UriParsingHelper.cs`](src/Prism.Core/Common/UriParsingHelper.cs): URI解析助手，用于解析和构建URI

#### 功能特点：

- 提供了参数管理的基础设施
- 支持多播异常处理
- 提供了URI解析和构建功能
- 支持注册表感知，便于对象管理

### 3. Dialogs

Dialogs目录提供了对话框服务的接口和基础实现，用于应用程序中的对话框管理。

#### 主要组件：

- [`ButtonResult.cs`](src/Prism.Core/Dialogs/ButtonResult.cs): 按钮结果枚举，定义了对话框按钮的可能结果
- [`DialogCallback.cs`](src/Prism.Core/Dialogs/DialogCallback.cs): 对话框回调，用于处理对话框结果
- [`DialogCloseListener.cs`](src/Prism.Core/Dialogs/DialogCloseListener.cs): 对话框关闭监听器
- [`DialogException.cs`](src/Prism.Core/Dialogs/DialogException.cs): 对话框异常类
- [`DialogParameters.cs`](src/Prism.Core/Dialogs/DialogParameters.cs): 对话框参数类，继承自ParametersBase
- [`DialogResult.cs`](src/Prism.Core/Dialogs/DialogResult.cs): 对话框结果类
- [`DialogUtilities.cs`](src/Prism.Core/Dialogs/DialogUtilities.cs): 对话框工具类
- [`IDialogAware.cs`](src/Prism.Core/Dialogs/IDialogAware.cs): 对话框感知接口，用于实现对话框视图模型
- [`IDialogParameters.cs`](src/Prism.Core/Dialogs/IDialogParameters.cs): 对话框参数接口
- [`IDialogResult.cs`](src/Prism.Core/Dialogs/IDialogResult.cs): 对话框结果接口
- [`IDialogService.cs`](src/Prism.Core/Dialogs/IDialogService.cs): 对话框服务接口
- [`IDialogServiceExtensions.cs`](src/Prism.Core/Dialogs/IDialogServiceExtensions.cs): 对话框服务扩展方法

#### 功能特点：

- 提供了完整的对话框服务接口
- 支持对话框参数传递
- 支持对话框结果处理
- 支持对话框回调
- 提供了对话框感知接口，便于实现对话框视图模型

### 4. Extensions

Extensions目录提供了扩展方法，增强了框架的功能。

#### 主要组件：

- [`TaskExtensions.cs`](src/Prism.Core/Extensions/TaskExtensions.cs): Task扩展方法
- [`TaskExtensions{T}.cs`](src/Prism.Core/Extensions/TaskExtensions{T}.cs): 泛型Task扩展方法

#### 功能特点：

- 提供了Task相关的扩展方法
- 简化了异步编程

### 5. Modularity

Modularity目录提供了模块化系统的实现，支持应用程序的模块化开发。

#### 主要组件：

- [`IModule.cs`](src/Prism.Core/Modularity/IModule.cs): 模块接口，定义了模块的基本操作
- [`IModuleCatalog.cs`](src/Prism.Core/Modularity/IModuleCatalog.cs): 模块目录接口，用于管理模块
- [`IModuleCatalogCoreExtensions.cs`](src/Prism.Core/Modularity/IModuleCatalogCoreExtensions.cs): 模块目录核心扩展
- [`IModuleCatalogItem.cs`](src/Prism.Core/Modularity/IModuleCatalogItem.cs): 模块目录项接口
- [`IModuleInfo.cs`](src/Prism.Core/Modularity/IModuleInfo.cs): 模块信息接口
- [`IModuleInfoGroup.cs`](src/Prism.Core/Modularity/IModuleInfoGroup.cs): 模块信息组接口
- [`IModuleInitializer.cs`](src/Prism.Core/Modularity/IModuleInitializer.cs): 模块初始化器接口
- [`IModuleManager.cs`](src/Prism.Core/Modularity/IModuleManager.cs): 模块管理器接口
- [`IModuleManagerExtensions.cs`](src/Prism.Core/Modularity/IModuleManagerExtensions.cs): 模块管理器扩展
- [`InitializationMode.cs`](src/Prism.Core/Modularity/InitializationMode.cs): 初始化模式枚举
- [`ModuleCatalogBase.cs`](src/Prism.Core/Modularity/ModuleCatalogBase.cs): 模块目录基类
- [`ModuleDependencyAttribute.cs`](src/Prism.Core/Modularity/ModuleDependencyAttribute.cs): 模块依赖属性
- [`ModuleDependencySolver.cs`](src/Prism.Core/Modularity/ModuleDependencySolver.cs): 模块依赖解析器
- [`ModuleState.cs`](src/Prism.Core/Modularity/ModuleState.cs): 模块状态枚举

#### 异常类：

- [`CyclicDependencyFoundException.cs`](src/Prism.Core/Modularity/CyclicDependencyFoundException.cs): 循环依赖异常
- [`CyclicDependencyFoundException.Desktop.cs`](src/Prism.Core/Modularity/CyclicDependencyFoundException.Desktop.cs): 桌面平台循环依赖异常
- [`DuplicateModuleException.cs`](src/Prism.Core/Modularity/DuplicateModuleException.cs): 重复模块异常
- [`DuplicateModuleException.Desktop.cs`](src/Prism.Core/Modularity/DuplicateModuleException.Desktop.cs): 桌面平台重复模块异常
- [`ModularityException.cs`](src/Prism.Core/Modularity/ModularityException.cs): 模块化异常基类
- [`ModularityException.Desktop.cs`](src/Prism.Core/Modularity/ModularityException.Desktop.cs): 桌面平台模块化异常
- [`ModuleInitializeException.cs`](src/Prism.Core/Modularity/ModuleInitializeException.cs): 模块初始化异常
- [`ModuleInitializeException.Desktop.cs`](src/Prism.Core/Modularity/ModuleInitializeException.Desktop.cs): 桌面平台模块初始化异常
- [`ModuleNotFoundException.cs`](src/Prism.Core/Modularity/ModuleNotFoundException.cs): 模块未找到异常
- [`ModuleNotFoundException.Desktop.cs`](src/Prism.Core/Modularity/ModuleNotFoundException.Desktop.cs): 桌面平台模块未找到异常
- [`ModuleTypeLoadingException.cs`](src/Prism.Core/Modularity/ModuleTypeLoadingException.cs): 模块类型加载异常
- [`ModuleTypeLoadingException.Desktop.cs`](src/Prism.Core/Modularity/ModuleTypeLoadingException.Desktop.cs): 桌面平台模块类型加载异常

#### 事件参数类：

- [`LoadModuleCompletedEventArgs.cs`](src/Prism.Core/Modularity/LoadModuleCompletedEventArgs.cs): 模块加载完成事件参数
- [`ModuleDownloadProgressChangedEventArgs.cs`](src/Prism.Core/Modularity/ModuleDownloadProgressChangedEventArgs.cs): 模块下载进度变化事件参数

#### 功能特点：

- 提供了完整的模块化系统
- 支持模块依赖管理
- 支持模块初始化
- 支持模块状态管理
- 提供了丰富的异常处理
- 支持模块下载进度报告

### 6. Mvvm

Mvvm目录提供了MVVM模式的支持，是Prism框架的核心部分。

#### 主要组件：

- [`BindableBase.cs`](src/Prism.Core/Mvvm/BindableBase.cs): 可绑定基类，实现了INotifyPropertyChanged接口
- [`ErrorsContainer.cs`](src/Prism.Core/Mvvm/ErrorsContainer.cs): 错误容器，用于数据验证
- [`IViewRegistry.cs`](src/Prism.Core/Mvvm/IViewRegistry.cs): 视图注册表接口
- [`PropertySupport.cs`](src/Prism.Core/Mvvm/PropertySupport.cs): 属性支持类，用于属性表达式处理
- [`ViewCreationException.cs`](src/Prism.Core/Mvvm/ViewCreationException.cs): 视图创建异常
- [`ViewModelCreationException.cs`](src/Prism.Core/Mvvm/ViewModelCreationException.cs): 视图模型创建异常
- [`ViewModelLocationProvider.cs`](src/Prism.Core/Mvvm/ViewModelLocationProvider.cs): 视图模型定位提供者
- [`ViewRegistration.cs`](src/Prism.Core/Mvvm/ViewRegistration.cs): 视图注册类
- [`ViewRegistryBase{TBaseView}.cs`](src/Prism.Core/Mvvm/ViewRegistryBase{TBaseView}.cs): 视图注册表基类
- [`ViewType.cs`](src/Prism.Core/Mvvm/ViewType.cs): 视图类型枚举

#### 功能特点：

- 提供了MVVM模式的基础实现
- 支持属性变化通知
- 支持数据验证
- 支持视图和视图模型的自动关联
- 提供了视图注册表，便于视图管理

### 7. Navigation

Navigation目录提供了导航系统的实现，支持页面导航和区域导航。

#### 基础导航组件：

- [`IDestructible.cs`](src/Prism.Core/Navigation/IDestructible.cs): 可销毁接口，用于对象销毁时的资源清理
- [`INavigationParameters.cs`](src/Prism.Core/Navigation/INavigationParameters.cs): 导航参数接口
- [`INavigationParametersInternal.cs`](src/Prism.Core/Navigation/INavigationParametersInternal.cs): 内部导航参数接口
- [`INavigationResult.cs`](src/Prism.Core/Navigation/INavigationResult.cs): 导航结果接口
- [`NavigationException.cs`](src/Prism.Core/Navigation/NavigationException.cs): 导航异常类
- [`NavigationParameters.cs`](src/Prism.Core/Navigation/NavigationParameters.cs): 导航参数实现
- [`NavigationResult.cs`](src/Prism.Core/Navigation/NavigationResult.cs): 导航结果实现

#### 区域导航组件：

- [`IConfirmNavigationRequest.cs`](src/Prism.Core/Navigation/Regions/IConfirmNavigationRequest.cs): 确认导航请求接口
- [`IJournalAware.cs`](src/Prism.Core/Navigation/Regions/IJournalAware.cs): 日志感知接口
- [`INavigateAsync.cs`](src/Prism.Core/Navigation/Regions/INavigateAsync.cs): 异步导航接口
- [`IRegion.cs`](src/Prism.Core/Navigation/Regions/IRegion.cs): 区域接口
- [`IRegionAdapter.cs`](src/Prism.Core/Navigation/Regions/IRegionAdapter.cs): 区域适配器接口
- [`IRegionAware.cs`](src/Prism.Core/Navigation/Regions/IRegionAware.cs): 区域感知接口
- [`IRegionBehavior.cs`](src/Prism.Core/Navigation/Regions/IRegionBehavior.cs): 区域行为接口
- [`IRegionBehaviorCollection.cs`](src/Prism.Core/Navigation/Regions/IRegionBehaviorCollection.cs): 区域行为集合接口
- [`IRegionBehaviorFactory.cs`](src/Prism.Core/Navigation/Regions/IRegionBehaviorFactory.cs): 区域行为工厂接口
- [`IRegionBehaviorFactoryExtensions.cs`](src/Prism.Core/Navigation/Regions/IRegionBehaviorFactoryExtensions.cs): 区域行为工厂扩展
- [`IRegionCollection.cs`](src/Prism.Core/Navigation/Regions/IRegionCollection.cs): 区域集合接口
- [`IRegionManager.cs`](src/Prism.Core/Navigation/Regions/IRegionManager.cs): 区域管理器接口
- [`IRegionManagerExtensions.cs`](src/Prism.Core/Navigation/Regions/IRegionManagerExtensions.cs): 区域管理器扩展
- [`IRegionMemberLifetime.cs`](src/Prism.Core/Navigation/Regions/IRegionMemberLifetime.cs): 区域成员生命周期接口
- [`IRegionNavigationContentLoader.cs`](src/Prism.Core/Navigation/Regions/IRegionNavigationContentLoader.cs): 区域导航内容加载器接口
- [`IRegionNavigationJournal.cs`](src/Prism.Core/Navigation/Regions/IRegionNavigationJournal.cs): 区域导航日志接口
- [`IRegionNavigationJournalEntry.cs`](src/Prism.Core/Navigation/Regions/IRegionNavigationJournalEntry.cs): 区域导航日志条目接口
- [`IRegionNavigationRegistry.cs`](src/Prism.Core/Navigation/Regions/IRegionNavigationRegistry.cs): 区域导航注册表接口
- [`IRegionNavigationService.cs`](src/Prism.Core/Navigation/Regions/IRegionNavigationService.cs): 区域导航服务接口
- [`IRegionViewRegistry.cs`](src/Prism.Core/Navigation/Regions/IRegionViewRegistry.cs): 区域视图注册表接口
- [`IViewsCollection.cs`](src/Prism.Core/Navigation/Regions/IViewsCollection.cs): 视图集合接口

#### 区域导航实现类：

- [`NavigationAsyncExtensions.cs`](src/Prism.Core/Navigation/Regions/NavigationAsyncExtensions.cs): 导航异步扩展
- [`NavigationContext.cs`](src/Prism.Core/Navigation/Regions/NavigationContext.cs): 导航上下文
- [`NavigationContextExtensions.cs`](src/Prism.Core/Navigation/Regions/NavigationContextExtensions.cs): 导航上下文扩展
- [`RegionBehavior.cs`](src/Prism.Core/Navigation/Regions/RegionBehavior.cs): 区域行为实现
- [`RegionBehaviorCollection.cs`](src/Prism.Core/Navigation/Regions/RegionBehaviorCollection.cs): 区域行为集合实现
- [`RegionBehaviorFactory.cs`](src/Prism.Core/Navigation/Regions/RegionBehaviorFactory.cs): 区域行为工厂实现

#### 区域导航属性：

- [`RegionMemberLifetimeAttribute.cs`](src/Prism.Core/Navigation/Regions/RegionMemberLifetimeAttribute.cs): 区域成员生命周期属性
- [`SyncActiveStateAttribute.cs`](src/Prism.Core/Navigation/Regions/SyncActiveStateAttribute.cs): 同步活动状态属性
- [`ViewSortHintAttribute.cs`](src/Prism.Core/Navigation/Regions/ViewSortHintAttribute.cs): 视图排序提示属性

#### 区域导航事件参数：

- [`RegionNavigationEventArgs.cs`](src/Prism.Core/Navigation/Regions/RegionNavigationEventArgs.cs): 区域导航事件参数
- [`RegionNavigationFailedEventArgs.cs`](src/Prism.Core/Navigation/Regions/RegionNavigationFailedEventArgs.cs): 区域导航失败事件参数
- [`ViewRegisteredEventArgs.cs`](src/Prism.Core/Navigation/Regions/ViewRegisteredEventArgs.cs): 视图注册事件参数

#### 区域导航实现类：

- [`RegionNavigationJournal.cs`](src/Prism.Core/Navigation/Regions/RegionNavigationJournal.cs): 区域导航日志实现
- [`RegionNavigationJournalEntry.cs`](src/Prism.Core/Navigation/Regions/RegionNavigationJournalEntry.cs): 区域导航日志条目实现

#### 区域导航扩展：

- [`RegionViewRegistryExtensions.cs`](src/Prism.Core/Navigation/Regions/RegionViewRegistryExtensions.cs): 区域视图注册表扩展

#### 区域导航异常：

- [`RegionCreationException.cs`](src/Prism.Core/Navigation/Regions/RegionCreationException.cs): 区域创建异常
- [`RegionException.cs`](src/Prism.Core/Navigation/Regions/RegionException.cs): 区域异常基类
- [`RegionViewException.cs`](src/Prism.Core/Navigation/Regions/RegionViewException.cs): 区域视图异常
- [`UpdateRegionsException.cs`](src/Prism.Core/Navigation/Regions/UpdateRegionsException.cs): 更新区域异常
- [`ViewRegistrationException.cs`](src/Prism.Core/Navigation/Regions/ViewRegistrationException.cs): 视图注册异常

#### 功能特点：

- 提供了完整的导航系统
- 支持页面导航和区域导航
- 支持导航参数传递
- 支持导航日志管理
- 支持区域行为管理
- 提供了丰富的异常处理
- 支持视图注册和管理

### 8. Properties

Properties目录包含了项目的属性和资源文件。

#### 主要组件：

- [`AssemblyInfo.cs`](src/Prism.Core/Properties/AssemblyInfo.cs): 程序集信息
- [`Resources.Designer.cs`](src/Prism.Core/Properties/Resources.Designer.cs): 资源设计器生成的代码
- [`Resources.resx`](src/Prism.Core/Properties/Resources.resx): 资源文件

### 9. build

build目录包含了构建配置文件。

#### 主要组件：

- [`Package.targets`](src/Prism.Core/build/Package.targets): NuGet包构建目标

### 10. 根目录文件

#### 主要组件：

- [`GlobalSuppressions.cs`](src/Prism.Core/GlobalSuppressions.cs): 全局抑制警告，用于代码分析
- [`IActiveAware.cs`](src/Prism.Core/IActiveAware.cs): 活动状态感知接口，用于对象活动状态管理
- [`Prism.Core.csproj`](src/Prism.Core/Prism.Core.csproj): 项目文件，定义了项目的基本信息和依赖

## 项目配置

### 目标框架

Prism.Core支持多个目标框架：
- netstandard2.0
- net462
- net47
- net6.0

这使得Prism.Core可以在多种平台上运行，包括.NET Framework和.NET Core/5+。

### 依赖项

Prism.Core的主要依赖项包括：
- PolySharp: 提供C#语言特性的API
- Prism.Container.Abstractions: 容器抽象层
- Prism.Events: 事件聚合器实现

## 使用示例

### 1. 使用命令

```csharp
// 创建同步命令
public class MainWindowViewModel : BindableBase
{
    private string _message;
    public string Message
    {
        get => _message;
        set => SetProperty(ref _message, value);
    }

    public ICommand ShowMessageCommand { get; }

    public MainWindowViewModel()
    {
        ShowMessageCommand = new DelegateCommand(ShowMessage);
    }

    private void ShowMessage()
    {
        Message = "Hello, Prism!";
    }
}

// 创建异步命令
public class MainWindowViewModel : BindableBase
{
    private bool _isLoading;
    public bool IsLoading
    {
        get => _isLoading;
        set => SetProperty(ref _isLoading, value);
    }

    private string _result;
    public string Result
    {
        get => _result;
        set => SetProperty(ref _result, value);
    }

    public ICommand LoadDataCommand { get; }

    public MainWindowViewModel()
    {
        LoadDataCommand = new AsyncDelegateCommand(LoadData);
    }

    private async Task LoadData()
    {
        IsLoading = true;
        try
        {
            // 模拟异步操作
            await Task.Delay(1000);
            Result = "Data loaded successfully!";
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

### 2. 使用导航参数

```csharp
// 创建导航参数
var parameters = new NavigationParameters
{
    { "id", 123 },
    { "name", "John Doe" }
};

// 传递导航参数
_regionManager.RequestNavigate("ContentRegion", "ViewA", parameters);

// 在视图模型中接收导航参数
public class ViewAViewModel : BindableBase, INavigationAware
{
    private int _id;
    public int Id
    {
        get => _id;
        set => SetProperty(ref _id, value);
    }

    private string _name;
    public string Name
    {
        get => _name;
        set => SetProperty(ref _name, value);
    }

    public void OnNavigatedTo(NavigationParameters parameters)
    {
        if (parameters.TryGetValue<int>("id", out var id))
        {
            Id = id;
        }

        if (parameters.TryGetValue<string>("name", out var name))
        {
            Name = name;
        }
    }

    public bool IsNavigationTarget(NavigationParameters parameters)
    {
        return true;
    }

    public void OnNavigatedFrom(NavigationParameters parameters)
    {
        // 导航离开时执行
    }
}
```

### 3. 实现模块

```csharp
// 实现IModule接口
public class ModuleA : IModule
{
    private readonly IRegionManager _regionManager;
    private readonly IContainerProvider _containerProvider;

    public ModuleA(IRegionManager regionManager, IContainerProvider containerProvider)
    {
        _regionManager = regionManager;
        _containerProvider = containerProvider;
    }

    public void OnInitialized(IContainerProvider containerProvider)
    {
        // 模块初始化时执行
        _regionManager.RequestNavigate("ContentRegion", "ViewA");
    }

    public void RegisterTypes(IContainerRegistry containerRegistry)
    {
        // 注册模块服务
        containerRegistry.Register<IServiceA, ServiceA>();
        containerRegistry.RegisterForNavigation<ViewA>();
    }
}
```

### 4. 实现对话框

```csharp
// 实现IDialogAware接口
public class NotificationDialogViewModel : BindableBase, IDialogAware
{
    private string _message;
    public string Message
    {
        get => _message;
        set => SetProperty(ref _message, value);
    }

    private string _title = "Notification";
    public string Title
    {
        get => _title;
        set => SetProperty(ref _title, value);
    }

    public event Action<IDialogResult> RequestClose;

    public DialogCloseListener RequestCloseListener { get; }

    public NotificationDialogViewModel()
    {
        RequestCloseListener = new DialogCloseListener();
        OKCommand = new DelegateCommand(OK);
    }

    public ICommand OKCommand { get; }

    private void OK()
    {
        RequestClose.Invoke(new DialogResult(ButtonResult.OK));
    }

    public bool CanCloseDialog()
    {
        return true;
    }

    public void OnDialogClosed()
    {
        // 对话框关闭时执行
    }

    public void OnDialogOpened(IDialogParameters parameters)
    {
        Message = parameters.GetValue<string>("message");
    }
}

// 显示对话框
public class MainWindowViewModel : BindableBase
{
    private readonly IDialogService _dialogService;

    public ICommand ShowNotificationCommand { get; }

    public MainWindowViewModel(IDialogService dialogService)
    {
        _dialogService = dialogService;
        ShowNotificationCommand = new DelegateCommand(ShowNotification);
    }

    private void ShowNotification()
    {
        var parameters = new DialogParameters
        {
            { "message", "This is a notification message." }
        };

        _dialogService.ShowDialog("NotificationDialog", parameters, result =>
        {
            if (result.Result == ButtonResult.OK)
            {
                // 用户点击了OK按钮
            }
        });
    }
}
```

## 设计原则

### 1. 单一职责原则

Prism.Core的每个组件都有明确的职责，例如：
- Commands负责命令模式实现
- Navigation负责导航功能
- Modularity负责模块化系统

### 2. 开闭原则

Prism.Core通过接口和抽象类提供了扩展点，使得框架可以在不修改现有代码的情况下进行扩展。

### 3. 依赖倒置原则

Prism.Core大量使用接口和依赖注入，使得高层模块不依赖于低层模块，两者都依赖于抽象。

### 4. 接口隔离原则

Prism.Core的接口设计精细，客户端不应该被迫依赖它们不使用的方法。

### 5. 平台无关性

Prism.Core不依赖于任何特定的应用程序框架，可以在多种平台上运行。

## 总结

Prism.Core是Prism框架的核心库，提供了跨平台的基础功能，包括命令模式、MVVM支持、导航系统、模块化系统等。它的设计遵循了SOLID原则，具有良好的可扩展性和可维护性。作为所有Prism平台特定实现的基础，Prism.Core为构建松耦合、可维护和可测试的XAML应用程序提供了坚实的基础。