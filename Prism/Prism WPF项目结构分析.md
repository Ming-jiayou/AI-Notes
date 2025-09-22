# Prism WPF项目结构分析

## 概述

Prism WPF是Prism框架在Windows Presentation Foundation (WPF)平台上的实现。作为Prism最初支持的平台，WPF实现提供了最完整的功能集，是构建复杂企业级WPF应用程序的理想选择。本文档详细分析了Prism WPF的项目结构，帮助开发者更好地理解其组织架构和各组件的功能。

## 目录结构

```
src/Wpf/
├── Prism.Wpf/           # WPF核心实现
├── Prism.DryIoc.Wpf/    # DryIoc容器WPF实现
└── Prism.Unity.Wpf/     # Unity容器WPF实现
```

## 核心组件详解

### 1. Prism.Wpf

Prism.Wpf是WPF平台的核心实现，提供了不依赖特定容器的WPF功能。

#### 主要功能模块：

- **应用程序基础**
  - [`PrismApplicationBase.cs`](src/Wpf/Prism.Wpf/PrismApplicationBase.cs): WPF应用程序基类，提供了Prism应用程序的基本功能
  - [`PrismBootstrapperBase.cs`](src/Wpf/Prism.Wpf/PrismBootstrapperBase.cs): WPF引导程序基类，用于初始化Prism应用程序
  - [`PrismInitializationExtensions.cs`](src/Wpf/Prism.Wpf/PrismInitializationExtensions.cs): Prism初始化扩展方法

- **Common/**: 通用工具类
  - [`MvvmHelpers.cs`](src/Wpf/Prism.Wpf/Common/MvvmHelpers.cs): MVVM辅助类
  - [`ObservableObject.cs`](src/Wpf/Prism.Wpf/Common/ObservableObject.cs): 可观察对象基类，实现了INotifyPropertyChanged接口

- **Dialogs/**: 对话框服务
  - [`DialogService.cs`](src/Wpf/Prism.Wpf/Dialogs/DialogService.cs): WPF对话框服务实现
  - [`DialogWindow.xaml`](src/Wpf/Prism.Wpf/Dialogs/DialogWindow.xaml): 对话框窗口XAML定义
  - [`DialogWindow.xaml.cs`](src/Wpf/Prism.Wpf/Dialogs/DialogWindow.xaml.cs): 对话框窗口代码后置
  - [`IDialogWindow.cs`](src/Wpf/Prism.Wpf/Dialogs/IDialogWindow.cs): 对话框窗口接口
  - [`IDialogServiceCompatExtensions.cs`](src/Wpf/Prism.Wpf/Dialogs/IDialogServiceCompatExtensions.cs): 对话框服务兼容性扩展
  - [`KnownDialogParameters.cs`](src/Wpf/Prism.Wpf/Dialogs/KnownDialogParameters.cs): 已知对话框参数

- **Extensions/**: 扩展方法
  - [`CollectionExtensions.cs`](src/Wpf/Prism.Wpf/Extensions/CollectionExtensions.cs): 集合扩展方法
  - [`DependencyObjectExtensions.cs`](src/Wpf/Prism.Wpf/Extensions/DependencyObjectExtensions.cs): 依赖对象扩展方法

- **Interactivity/**: 交互功能
  - [`CommandBehaviorBase.cs`](src/Wpf/Prism.Wpf/Interactivity/CommandBehaviorBase.cs): 命令行为基类
  - [`InvokeCommandAction.cs`](src/Wpf/Prism.Wpf/Interactivity/InvokeCommandAction.cs): 调用命令操作

- **Ioc/**: 控制反转容器支持
  - [`ContainerProviderExtension.cs`](src/Wpf/Prism.Wpf/Ioc/ContainerProviderExtension.cs): 容器提供者扩展
  - [`IContainerRegistryExtensions.cs`](src/Wpf/Prism.Wpf/Ioc/IContainerRegistryExtensions.cs): 容器注册扩展

- **Modularity/**: 模块化系统
  - [`ModuleCatalog.cs`](src/Wpf/Prism.Wpf/Modularity/ModuleCatalog.cs): 模块目录实现
  - [`ModuleManager.cs`](src/Wpf/Prism.Wpf/Modularity/ModuleManager.cs): 模块管理器实现
  - [`ModuleInitializer.cs`](src/Wpf/Prism.Wpf/Modularity/ModuleInitializer.cs): 模块初始化器
  - [`XamlModuleCatalog.cs`](src/Wpf/Prism.Wpf/Modularity/XamlModuleCatalog.cs): XAML模块目录
  - [`DirectoryModuleCatalog.net45.cs`](src/Wpf/Prism.Wpf/Modularity/DirectoryModuleCatalog.net45.cs): .NET 4.5目录模块目录
  - [`DirectoryModuleCatalog.netcore.cs`](src/Wpf/Prism.Wpf/Modularity/DirectoryModuleCatalog.netcore.cs): .NET Core目录模块目录
  - [`ConfigurationModuleCatalog.Desktop.cs`](src/Wpf/Prism.Wpf/Modularity/ConfigurationModuleCatalog.Desktop.cs): 桌面配置模块目录

- **Mvvm/**: MVVM模式支持
  - [`ViewModelLocator.cs`](src/Wpf/Prism.Wpf/Mvvm/ViewModelLocator.cs): 视图模型定位器

- **Navigation/Regions/**: 区域导航系统
  - [`Region.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Region.cs): 区域实现
  - [`RegionManager.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/RegionManager.cs): 区域管理器实现
  - [`RegionNavigationService.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/RegionNavigationService.cs): 区域导航服务
  - [`RegionViewRegistry.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/RegionViewRegistry.cs): 区域视图注册表
  - [`AllActiveRegion.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/AllActiveRegion.cs): 全活动区域
  - [`SingleActiveRegion.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/SingleActiveRegion.cs): 单活动区域
  - [`ViewsCollection.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/ViewsCollection.cs): 视图集合

- **Navigation/Regions/Behaviors/**: 区域行为
  - [`AutoPopulateRegionBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/AutoPopulateRegionBehavior.cs): 自动填充区域行为
  - [`BindRegionContextToDependencyObjectBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/BindRegionContextToDependencyObjectBehavior.cs): 绑定区域上下文到依赖对象行为
  - [`ClearChildViewsRegionBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/ClearChildViewsRegionBehavior.cs): 清除子视图区域行为
  - [`DelayedRegionCreationBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/DelayedRegionCreationBehavior.cs): 延迟区域创建行为
  - [`DestructibleRegionBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/DestructibleRegionBehavior.cs): 可销毁区域行为
  - [`RegionActiveAwareBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/RegionActiveAwareBehavior.cs): 区域活动感知行为
  - [`RegionManagerRegistrationBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/RegionManagerRegistrationBehavior.cs): 区域管理器注册行为
  - [`RegionMemberLifetimeBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/RegionMemberLifetimeBehavior.cs): 区域成员生命周期行为
  - [`SelectorItemsSourceSyncBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/SelectorItemsSourceSyncBehavior.cs): 选择器项源同步行为
  - [`SyncRegionContextWithHostBehavior.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/Behaviors/SyncRegionContextWithHostBehavior.cs): 同步区域上下文与主机行为

### 2. Prism.DryIoc.Wpf

Prism.DryIoc.Wpf是Prism在WPF平台上使用DryIoc容器的实现。

#### 主要组件：

- [`PrismApplication.cs`](src/Wpf/Prism.DryIoc.Wpf/PrismApplication.cs): DryIoc WPF应用程序类
- [`PrismBootstrapper.cs`](src/Wpf/Prism.DryIoc.Wpf/PrismBootstrapper.cs): DryIoc WPF引导程序类
- [`build/Package.targets`](src/Wpf/Prism.DryIoc.Wpf/build/Package.targets): NuGet包构建目标

#### 特点：

- 轻量级、高性能的依赖注入容器
- 支持AOP（面向切面编程）
- 高级注册API
- 优秀的性能和内存使用

### 3. Prism.Unity.Wpf

Prism.Unity.Wpf是Prism在WPF平台上使用Unity容器的实现。

#### 主要组件：

- [`PrismApplication.cs`](src/Wpf/Prism.Unity.Wpf/PrismApplication.cs): Unity WPF应用程序类
- [`PrismBootstrapper.cs`](src/Wpf/Prism.Unity.Wpf/PrismBootstrapper.cs): Unity WPF引导程序类
- [`app.config`](src/Wpf/Prism.Unity.Wpf/app.config): 应用程序配置文件
- [`build/Package.targets`](src/Wpf/Prism.Unity.Wpf/build/Package.targets): NuGet包构建目标

#### 特点：

- 成熟稳定的依赖注入容器
- 丰富的企业级功能
- 良好的文档和社区支持
- 支持拦截和策略注入

## WPF特有功能详解

### 1. 区域适配器

Prism WPF提供了多种区域适配器，用于将WPF控件转换为区域：

- [`ContentControlRegionAdapter.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/ContentControlRegionAdapter.cs): ContentControl区域适配器
- [`ItemsControlRegionAdapter.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/ItemsControlRegionAdapter.cs): ItemsControl区域适配器
- [`SelectorRegionAdapter.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/SelectorRegionAdapter.cs): Selector区域适配器

这些适配器使得开发者可以将任何WPF控件用作区域，实现视图的动态加载和管理。

### 2. 区域管理

- [`RegionAdapterMappings.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/RegionAdapterMappings.cs): 区域适配器映射
- [`DefaultRegionManagerAccessor.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/DefaultRegionManagerAccessor.cs): 默认区域管理器访问器
- [`IRegionManagerAccessor.cs`](src/Wpf/Prism.Wpf/Navigation/Regions/IRegionManagerAccessor.cs): 区域管理器访问器接口

这些组件提供了区域管理的核心功能，使得开发者可以轻松地创建和管理区域。

### 3. 模块化系统

Prism WPF提供了完整的模块化系统，支持多种模块加载方式：

- **目录模块加载**: 从指定目录加载模块
- **配置模块加载**: 从配置文件加载模块
- **XAML模块加载**: 从XAML文件加载模块
- **按需加载**: 支持模块的按需加载

### 4. 对话框服务

Prism WPF提供了完整的对话框服务，支持：

- 自定义对话框
- 模态和非模态对话框
- 对话框参数传递
- 对话框结果处理

### 5. MVVM支持

Prism WPF提供了完整的MVVM支持，包括：

- 视图模型定位
- 命令实现
- 数据绑定辅助
- 验证支持

## 使用示例

### 1. 创建Prism WPF应用程序

```csharp
// 使用DryIoc容器
public partial class App : PrismApplication
{
    protected override Window CreateShell()
    {
        return Container.Resolve<MainWindow>();
    }

    protected override void RegisterTypes(IContainerRegistry containerRegistry)
    {
        // 注册服务
        containerRegistry.RegisterSingleton<IService, MyService>();
    }
}

// 使用Unity容器
public partial class App : PrismApplication
{
    protected override Window CreateShell()
    {
        return Container.Resolve<MainWindow>();
    }

    protected override void RegisterTypes(IContainerRegistry containerRegistry)
    {
        // 注册服务
        containerRegistry.RegisterSingleton<IService, MyService>();
    }
}
```

### 2. 定义区域

```xaml
<Window x:Class="PrismWpfApp.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:prism="http://prismlibrary.com/"
        Title="Main Window" Height="350" Width="525">
    <Grid>
        <ContentControl prism:RegionManager.RegionName="ContentRegion" />
    </Grid>
</Window>
```

### 3. 导航到视图

```csharp
// 在视图模型中导航
public class MainWindowViewModel : BindableBase
{
    private readonly IRegionManager _regionManager;

    public MainWindowViewModel(IRegionManager regionManager)
    {
        _regionManager = regionManager;
        
        NavigateCommand = new DelegateCommand(Navigate);
    }

    public ICommand NavigateCommand { get; }

    private void Navigate()
    {
        _regionManager.RequestNavigate("ContentRegion", "ViewA");
    }
}
```

## 最佳实践

1. **依赖注入**: 使用依赖注入容器管理服务生命周期
2. **模块化**: 将应用程序划分为独立的模块
3. **MVVM**: 使用MVVM模式分离UI和业务逻辑
4. **区域导航**: 使用区域导航管理视图切换
5. **事件聚合**: 使用事件聚合器实现组件间通信

## 总结

Prism WPF是构建复杂WPF应用程序的理想选择，提供了完整的功能集和最佳实践指导。无论是使用DryIoc还是Unity容器，Prism WPF都能帮助开发者构建松耦合、可维护和可测试的WPF应用程序。通过模块化设计、区域导航、MVVM支持等特性，Prism WPF使得开发企业级WPF应用程序变得更加简单和高效。