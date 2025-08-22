# Stylet Conductor<IScreen> 详细用法指南

## 概述

`Conductor<IScreen>`是Stylet框架中用于实现**单页面导航模式**的核心组件。它继承自`ConductorBaseWithActiveItem<IScreen>`，专门用于管理单个活动屏幕（页面），是WPF MVVM应用程序中实现导航功能的基石。

## 核心概念

### 什么是Conductor？

Conductor（指挥者）是Stylet框架中的一个设计模式，用于管理子屏幕（Screens）的生命周期。它负责：
- 激活/停用子屏幕
- 管理子屏幕的生命周期
- 处理屏幕之间的导航
- 维护屏幕状态

### IScreen接口

`IScreen`是Stylet中所有可管理对象的基接口，定义了基本的生命周期方法：
- `Activate()` - 激活屏幕
- `Deactivate()` - 停用屏幕
- `CanCloseAsync()` - 检查是否可以关闭

## 继承层次结构

```
Screen (基础屏幕类)
  └── ConductorBase<T> (抽象导体基类)
      └── ConductorBaseWithActiveItem<T> (带活动项的导体基类)
          └── Conductor<T> (单活动项导体)
              └── Conductor<IScreen> (用于屏幕导航)
```

## 在NavigationController示例中的应用

### 1. ShellViewModel作为Conductor

在Stylet.Samples.NavigationController示例中，`ShellViewModel`充当了导航的主导体：

```csharp
public class ShellViewModel : Conductor<IScreen>, INavigationControllerDelegate
{
    public HeaderViewModel HeaderViewModel { get; }

    public ShellViewModel(HeaderViewModel headerViewModel)
    {
        this.HeaderViewModel = headerViewModel ?? throw new ArgumentNullException(nameof(headerViewModel));
    }

    public void NavigateTo(IScreen screen)
    {
        this.ActivateItem(screen);
    }
}
```

### 2. 导航流程分析

整个导航流程如下：

1. **用户触发导航** → HeaderViewModel/页面ViewModel
2. **调用NavigationController** → 创建目标页面ViewModel
3. **NavigationController回调** → ShellViewModel.NavigateTo()
4. **Conductor激活新页面** → ActivateItem(newScreen)

## Conductor<IScreen>的核心功能

### 1. 页面激活管理

```csharp
// 激活新页面，自动停用当前页面
public void NavigateTo(IScreen screen)
{
    this.ActivateItem(screen);
}

// 内部实现流程
protected override void ChangeActiveItem(T newItem, bool closePrevious)
{
    // 1. 停用当前活动项
    ScreenExtensions.TryDeactivate(this.ActiveItem);
    
    // 2. 如果需要关闭前一个页面
    if (closePrevious)
        this.CloseAndCleanUp(this.ActiveItem, this.DisposeChildren);

    // 3. 设置新的活动项
    this._activeItem = newItem;

    // 4. 激活新页面
    if (newItem != null)
    {
        this.EnsureItem(newItem);
        if (this.IsActive)
            ScreenExtensions.TryActivate(newItem);
    }

    // 5. 通知属性变更
    this.NotifyOfPropertyChange(nameof(this.ActiveItem));
}
```

### 2. 生命周期管理

Conductor自动管理页面的生命周期：

- **激活时**：自动调用新页面的`OnActivate()`
- **停用时**：自动调用旧页面的`OnDeactivate()`
- **关闭时**：自动调用页面的`OnClose()`

### 3. 内存管理

通过`DisposeChildren`属性控制页面关闭时是否释放资源：

```csharp
// 默认值为true，页面关闭时自动调用Dispose()
public virtual bool DisposeChildren { get; set; } = true;
```

## 实际使用示例

### 1. 创建主窗口ViewModel

```csharp
public class MainViewModel : Conductor<IScreen>
{
    private readonly Func<HomeViewModel> _homeFactory;
    private readonly Func<SettingsViewModel> _settingsFactory;

    public MainViewModel(
        Func<HomeViewModel> homeFactory,
        Func<SettingsViewModel> settingsFactory)
    {
        _homeFactory = homeFactory;
        _settingsFactory = settingsFactory;
    }

    public void ShowHome()
    {
        this.ActivateItem(_homeFactory());
    }

    public void ShowSettings()
    {
        this.ActivateItem(_settingsFactory());
    }
}
```

### 2. 创建页面ViewModel

```csharp
public class HomeViewModel : Screen
{
    private readonly IConductor<IScreen> _conductor;

    public HomeViewModel(IConductor<IScreen> conductor)
    {
        _conductor = conductor;
    }

    public void NavigateToSettings()
    {
        // 通过导航服务或直接使用conductor
        _conductor.ActivateItem(new SettingsViewModel());
    }

    protected override void OnActivate()
    {
        // 页面激活时的逻辑
        base.OnActivate();
    }

    protected override void OnDeactivate()
    {
        // 页面停用时清理资源
        base.OnDeactivate();
    }
}
```

### 3. XAML绑定

```xml
<!-- MainView.xaml -->
<Window x:Class="YourApp.Views.MainView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:s="https://github.com/canton7/Stylet">
    <DockPanel>
        <!-- 导航菜单 -->
        <Menu DockPanel.Dock="Top">
            <MenuItem Header="Home" Command="{s:Action ShowHome}"/>
            <MenuItem Header="Settings" Command="{s:Action ShowSettings}"/>
        </Menu>
        
        <!-- 页面内容区域 -->
        <ContentControl s:View.Model="{Binding ActiveItem}"/>
    </DockPanel>
</Window>
```

## 高级用法

### 1. 带参数的页面导航

```csharp
public class MainViewModel : Conductor<IScreen>
{
    public void ShowUserDetail(int userId)
    {
        var userDetail = _container.Get<UserDetailViewModel>();
        userDetail.Initialize(userId);
        this.ActivateItem(userDetail);
    }
}
```

### 2. 导航确认

```csharp
public class EditViewModel : Screen, IGuardClose
{
    public bool HasChanges { get; private set; }

    public async Task<bool> CanCloseAsync()
    {
        if (HasChanges)
        {
            var result = await _windowManager.ShowMessageBox(
                "有未保存的更改，确定要离开吗？",
                "确认",
                MessageBoxButton.YesNo);
            
            return result == MessageBoxResult.Yes;
        }
        return true;
    }
}
```

### 3. 页面状态保持

```csharp
public class MainViewModel : Conductor<IScreen>
{
    private readonly Dictionary<string, IScreen> _cachedScreens = new();
    
    public void ShowPage<T>(string key) where T : IScreen
    {
        if (!_cachedScreens.TryGetValue(key, out var screen))
        {
            screen = _container.Get<T>();
            _cachedScreens[key] = screen;
        }
        
        this.ActivateItem(screen);
    }
}
```

## 与NavigationController模式的集成

在NavigationController示例中，Conductor与NavigationController模式的结合使用：

### 1. 定义导航接口

```csharp
public interface INavigationController
{
    void NavigateToPage1();
    void NavigateToPage2(string initiator);
}

public interface INavigationControllerDelegate
{
    void NavigateTo(IScreen screen);
}
```

### 2. 实现导航控制器

```csharp
public class NavigationController : INavigationController
{
    private readonly Func<Page1ViewModel> _page1Factory;
    private readonly Func<Page2ViewModel> _page2Factory;
    
    public INavigationControllerDelegate Delegate { get; set; }

    public void NavigateToPage1()
    {
        Delegate?.NavigateTo(_page1Factory());
    }

    public void NavigateToPage2(string initiator)
    {
        var vm = _page2Factory();
        vm.Initiator = initiator;
        Delegate?.NavigateTo(vm);
    }
}
```

### 3. ShellViewModel实现双重角色

```csharp
public class ShellViewModel : Conductor<IScreen>, INavigationControllerDelegate
{
    // 作为Conductor管理页面
    // 作为INavigationControllerDelegate接收导航请求
    
    public void NavigateTo(IScreen screen)
    {
        this.ActivateItem(screen);
    }
}
```

## 最佳实践

### 1. 使用工厂模式创建页面

```csharp
// 在Bootstrapper中配置
protected override void ConfigureIoC(IStyletIoCBuilder builder)
{
    builder.Bind<Func<HomeViewModel>>()
        .ToFactory<Func<HomeViewModel>>(c => () => c.Get<HomeViewModel>());
}
```

### 2. 避免内存泄漏

```csharp
public class MainViewModel : Conductor<IScreen>
{
    protected override void OnClose()
    {
        // 确保所有子页面都被正确清理
        if (this.ActiveItem != null)
        {
            this.CloseItem(this.ActiveItem);
        }
        base.OnClose();
    }
}
```

### 3. 处理导航异常

```csharp
public class SafeConductor<T> : Conductor<T> where T : class
{
    public override async void ActivateItem(T item)
    {
        try
        {
            await Task.Run(() => base.ActivateItem(item));
        }
        catch (Exception ex)
        {
            // 处理导航异常
            _logger.Error($"导航失败: {ex.Message}");
        }
    }
}
```

## 性能优化

### 1. 延迟加载页面

```csharp
public class LazyConductor<T> : Conductor<T> where T : class
{
    private readonly Dictionary<Type, T> _cache = new();
    
    public override void ActivateItem(T item)
    {
        if (item != null && !_cache.ContainsKey(item.GetType()))
        {
            _cache[item.GetType()] = item;
        }
        base.ActivateItem(item);
    }
}
```

### 2. 异步页面加载

```csharp
public class AsyncConductor<T> : Conductor<T> where T : class
{
    public async Task ActivateItemAsync(T item)
    {
        if (item is IAsyncInitializable asyncItem)
        {
            await asyncItem.InitializeAsync();
        }
        this.ActivateItem(item);
    }
}

public interface IAsyncInitializable
{
    Task InitializeAsync();
}
```

## 总结

`Conductor<IScreen>`是Stylet框架中实现单页面导航的核心组件，它提供了：

1. **简洁的API**：通过`ActivateItem()`方法实现页面切换
2. **自动生命周期管理**：自动处理页面的激活、停用和清理
3. **内存管理**：自动释放不再使用的页面资源
4. **扩展性**：支持自定义导航行为和生命周期管理

通过结合NavigationController模式，可以构建出结构清晰、易于维护的WPF导航应用程序。这种模式特别适合需要简单页面切换的场景，如设置向导、标签页界面等。