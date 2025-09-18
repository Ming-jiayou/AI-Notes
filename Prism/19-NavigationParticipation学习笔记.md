# 19-NavigationParticipation 项目学习笔记

## 项目概述

NavigationParticipation 项目演示了如何在 Prism 框架中实现导航参与（Navigation Participation）机制。通过实现 [`INavigationAware`](19-NavigationParticipation/ModuleA/ViewModels/ViewAViewModel.cs:6) 接口，视图模型可以参与到导航过程中，控制导航行为并响应导航事件。

## 核心概念

### 导航参与的作用
导航参与允许视图模型：
- **控制导航目标**：决定当前实例是否可以处理特定的导航请求
- **响应导航事件**：在导航发生前后执行自定义逻辑
- **维护状态**：在导航过程中保持或恢复视图状态
- **数据传递**：通过导航上下文接收参数

### 关键特性
- **生命周期管理**：完整的导航生命周期钩子
- **状态保持**：支持视图实例重用和状态恢复
- **条件导航**：根据业务逻辑决定是否接受导航
- **参数传递**：通过导航上下文传递数据

## 项目结构

```
19-NavigationParticipation/
├── NavigationParticipation/      # 主应用程序
│   ├── App.xaml                 # 应用程序入口，配置 Prism
│   ├── App.xaml.cs              # 应用程序逻辑，注册模块
│   ├── Views/
│   │   └── MainWindow.xaml      # 主窗口，包含导航按钮和 TabControl
│   └── ViewModels/
│       └── MainWindowViewModel.cs # 导航命令逻辑
└── ModuleA/                     # 功能模块
    ├── ModuleAModule.cs         # 模块初始化，注册可导航视图
    ├── Views/
    │   ├── ViewA.xaml           # 视图A，显示标题和访问计数
    │   └── ViewB.xaml           # 视图B，显示标题和访问计数
    └── ViewModels/
        ├── ViewAViewModel.cs    # 视图A的ViewModel，实现INavigationAware
        └── ViewBViewModel.cs    # 视图B的ViewModel，实现INavigationAware
```

## 关键实现

### 1. INavigationAware 接口实现 (ViewAViewModel.cs)
```csharp
public class ViewAViewModel : BindableBase, INavigationAware
{
    private int _pageViews;
    public int PageViews
    {
        get { return _pageViews; }
        set { SetProperty(ref _pageViews, value); }
    }

    public void OnNavigatedTo(NavigationContext navigationContext)
    {
        PageViews++;
    }

    public bool IsNavigationTarget(NavigationContext navigationContext)
    {
        return true;
    }

    public void OnNavigatedFrom(NavigationContext navigationContext)
    {
        // 导航离开时的清理工作
    }
}
```

**关键点**：
- [`OnNavigatedTo`](19-NavigationParticipation/ModuleA/ViewModels/ViewAViewModel.cs:27)：导航到当前视图时调用，用于初始化或恢复状态
- [`IsNavigationTarget`](19-NavigationParticipation/ModuleA/ViewModels/ViewAViewModel.cs:32)：决定当前实例是否可以处理导航请求
- [`OnNavigatedFrom`](19-NavigationParticipation/ModuleA/ViewModels/ViewAViewModel.cs:37)：导航离开当前视图时调用，用于清理工作

### 2. 主窗口布局 (MainWindow.xaml)
```xml
<Window.Resources>
    <Style TargetType="TabItem">
        <Setter Property="Header" Value="{Binding DataContext.Title}" />
    </Style>
</Window.Resources>

<DockPanel LastChildFill="True">
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Top" Margin="5" >
        <Button Command="{Binding NavigateCommand}" CommandParameter="ViewA" Margin="5">Navigate to View A</Button>
        <Button Command="{Binding NavigateCommand}" CommandParameter="ViewB" Margin="5">Navigate to View B</Button>
    </StackPanel>
    <TabControl prism:RegionManager.RegionName="ContentRegion" Margin="5"  />
</DockPanel>
```

**关键点**：
- 使用 [`TabControl`](19-NavigationParticipation/NavigationParticipation/Views/MainWindow.xaml:19) 作为导航区域，支持多个视图同时存在
- TabItem 的标题绑定到视图模型的 Title 属性
- 导航按钮触发不同的视图导航

### 3. 视图模型基类实现
```csharp
public class ViewAViewModel : BindableBase, INavigationAware
{
    private string _title = "ViewA";
    public string Title
    {
        get { return _title; }
        set { SetProperty(ref _title, value); }
    }

    public void OnNavigatedTo(NavigationContext navigationContext)
    {
        PageViews++; // 每次导航到时增加访问计数
    }

    public bool IsNavigationTarget(NavigationContext navigationContext)
    {
        return true; // 始终接受导航请求
    }
}
```

### 4. 模块注册 (ModuleAModule.cs)
```csharp
public void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.RegisterForNavigation<ViewA>();
    containerRegistry.RegisterForNavigation<ViewB>();
}
```

## 工作流程

1. **初始化**：应用程序启动时，ModuleA 注册 ViewA 和 ViewB 为可导航视图

2. **首次导航**：用户点击导航按钮，触发 [`RequestNavigate`](19-NavigationParticipation/NavigationParticipation/ViewModels/MainWindowViewModel.cs:30)

3. **视图创建**：Prism 创建目标视图的实例，并调用 [`OnNavigatedTo`](19-NavigationParticipation/ModuleA/ViewModels/ViewAViewModel.cs:27)

4. **状态更新**：在 [`OnNavigatedTo`](19-NavigationParticipation/ModuleA/ViewModels/ViewAViewModel.cs:27) 中更新访问计数器

5. **重复导航**：当再次导航到同一视图时，会重用现有实例并再次调用 [`OnNavigatedTo`](19-NavigationParticipation/ModuleA/ViewModels/ViewAViewModel.cs:27)

## 技术要点

### INavigationAware 接口详解
```csharp
public interface INavigationAware
{
    // 导航到当前视图时调用
    void OnNavigatedTo(NavigationContext navigationContext);
    
    // 决定当前实例是否可以处理导航请求
    bool IsNavigationTarget(NavigationContext navigationContext);
    
    // 导航离开当前视图时调用
    void OnNavigatedFrom(NavigationContext navigationContext);
}
```

### 导航上下文使用
```csharp
public void OnNavigatedTo(NavigationContext navigationContext)
{
    // 获取导航参数
    var parameter = navigationContext.Parameters["key"];
    
    // 获取导航服务
    var navigationService = navigationContext.NavigationService;
    
    // 获取目标 URI
    var uri = navigationContext.Uri;
}
```

### 条件导航实现
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 根据参数决定是否接受导航
    if (navigationContext.Parameters.ContainsKey("id"))
    {
        var id = navigationContext.Parameters["id"].ToString();
        return this.Id == id;
    }
    
    return true; // 默认接受所有导航
}
```

### 状态保持策略
```csharp
public void OnNavigatedFrom(NavigationContext navigationContext)
{
    // 保存当前状态到导航参数
    navigationContext.Parameters.Add("LastState", this.CurrentState);
}

public void OnNavigatedTo(NavigationContext navigationContext)
{
    // 恢复之前保存的状态
    if (navigationContext.Parameters.ContainsKey("LastState"))
    {
        this.CurrentState = navigationContext.Parameters["LastState"].ToString();
    }
}
```

## 扩展应用

### 视图生命周期管理
```csharp
public void OnNavigatedTo(NavigationContext navigationContext)
{
    // 初始化数据
    LoadData();
    
    // 启动定时器
    StartTimer();
    
    // 注册事件
    RegisterEvents();
}

public void OnNavigatedFrom(NavigationContext navigationContext)
{
    // 停止定时器
    StopTimer();
    
    // 注销事件
    UnregisterEvents();
    
    // 清理资源
    Cleanup();
}
```

### 导航权限控制
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 检查用户权限
    if (!UserHasPermission())
    {
        return false;
    }
    
    // 检查业务条件
    if (!MeetsBusinessConditions())
    {
        return false;
    }
    
    return true;
}
```

### 数据刷新策略
```csharp
public void OnNavigatedTo(NavigationContext navigationContext)
{
    // 强制刷新数据
    if (navigationContext.Parameters.ContainsKey("refresh") && 
        (bool)navigationContext.Parameters["refresh"])
    {
        RefreshData();
    }
    
    // 增量更新
    if (navigationContext.Parameters.ContainsKey("lastUpdate"))
    {
        var lastUpdate = (DateTime)navigationContext.Parameters["lastUpdate"];
        UpdateDataSince(lastUpdate);
    }
}
```

## 学习总结

导航参与机制在实际应用中非常重要，它提供了：

1. **细粒度控制**：可以精确控制导航行为
2. **状态管理**：支持复杂的视图状态管理
3. **生命周期管理**：完整的视图生命周期钩子
4. **数据传递**：通过导航上下文传递参数
5. **性能优化**：支持视图实例重用，避免重复创建

这种模式在企业级应用中非常实用，特别是在需要复杂导航逻辑、状态保持和权限控制的应用场景中。通过 [`INavigationAware`](19-NavigationParticipation/ModuleA/ViewModels/ViewAViewModel.cs:6) 接口，开发者可以构建出响应式、状态感知的智能视图。