# 20-NavigateToExistingViews 项目学习笔记

## 项目概述

NavigateToExistingViews 项目演示了如何在 Prism 框架中控制导航到现有视图（Navigate To Existing Views）的行为。通过实现 [`INavigationAware`](20-NavigateToExistingViews/ModuleA/ViewModels/ViewAViewModel.cs:6) 接口的 [`IsNavigationTarget`](20-NavigateToExistingViews/ModuleA/ViewModels/ViewAViewModel.cs:32) 方法，可以精确控制何时重用现有视图实例，何时创建新实例。

## 核心概念

### 导航到现有视图的作用
导航到现有视图机制允许开发者：
- **控制视图重用**：决定何时重用现有视图实例，何时创建新实例
- **优化性能**：避免不必要的视图创建和销毁
- **保持状态**：在适当的时候保持视图状态
- **条件导航**：根据业务逻辑决定是否接受导航请求

### 关键特性
- **智能重用**：根据条件智能决定是否重用视图
- **状态控制**：精确控制视图状态和生命周期
- **性能优化**：减少内存消耗和提高响应速度
- **灵活配置**：支持各种重用策略

## 项目结构

```
20-NavigateToExistingViews/
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
        ├── ViewAViewModel.cs    # 视图A的ViewModel，控制导航目标
        └── ViewBViewModel.cs    # 视图B的ViewModel，控制导航目标
```

## 关键实现

### 1. 智能导航目标控制 (ViewAViewModel.cs)
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
        return PageViews / 3 != 1; // 当访问次数为3时返回false，强制创建新实例
    }
}
```

**关键点**：
- [`IsNavigationTarget`](20-NavigateToExistingViews/ModuleA/ViewModels/ViewAViewModel.cs:32) 方法控制是否重用当前视图实例
- 当返回 `true` 时，重用现有实例并调用 [`OnNavigatedTo`](20-NavigateToExistingViews/ModuleA/ViewModels/ViewAViewModel.cs:27)
- 当返回 `false` 时，创建新实例并导航到新实例

### 2. 不同的重用策略 (ViewBViewModel.cs)
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    return PageViews / 6 != 1; // 当访问次数为6时返回false，强制创建新实例
}
```

**关键点**：
- ViewB 使用不同的条件（6次访问）来触发新实例创建
- 展示了不同视图可以有不同的重用策略

### 3. 主窗口布局 (MainWindow.xaml)
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
- 使用 [`TabControl`](20-NavigateToExistingViews/NavigationParticipation/Views/MainWindow.xaml:19) 作为导航区域
- TabItem 的标题绑定到视图模型的 Title 属性
- 可以直观地看到多个视图实例的存在

### 4. 模块注册 (ModuleAModule.cs)
```csharp
public void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.RegisterForNavigation<ViewA>();
    containerRegistry.RegisterForNavigation<ViewB>();
}
```

## 工作流程

1. **首次导航**：用户点击导航按钮，触发 [`RequestNavigate`](20-NavigateToExistingViews/NavigationParticipation/ViewModels/MainWindowViewModel.cs:30)

2. **实例检查**：Prism 检查是否存在现有实例，并调用 [`IsNavigationTarget`](20-NavigateToExistingViews/ModuleA/ViewModels/ViewAViewModel.cs:32)

3. **条件判断**：
   - 如果 [`IsNavigationTarget`](20-NavigateToExistingViews/ModuleA/ViewModels/ViewAViewModel.cs:32) 返回 `true` → 重用现有实例
   - 如果 [`IsNavigationTarget`](20-NavigateToExistingViews/ModuleA/ViewModels/ViewAViewModel.cs:32) 返回 `false` → 创建新实例

4. **导航执行**：根据判断结果，导航到现有实例或新实例

5. **状态更新**：调用 [`OnNavigatedTo`](20-NavigateToExistingViews/ModuleA/ViewModels/ViewAViewModel.cs:27) 更新访问计数

## 技术要点

### IsNavigationTarget 的返回值含义
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 返回 true: 重用当前实例，导航到现有视图
    // 返回 false: 创建新实例，导航到新视图
    
    return SomeCondition(); // 根据业务逻辑决定
}
```

### 常见的重用策略
```csharp
// 1. 始终重用
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    return true;
}

// 2. 从不重用
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    return false;
}

// 3. 基于参数的重用
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    if (navigationContext.Parameters.ContainsKey("id"))
    {
        var id = navigationContext.Parameters["id"].ToString();
        return this.Id == id; // 只处理相同ID的请求
    }
    return false;
}

// 4. 基于状态的重用
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    return !this.IsDirty; // 只有在没有未保存更改时才重用
}

// 5. 基于时间的重用
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    return DateTime.Now - this.CreatedTime < TimeSpan.FromMinutes(5);
}
```

### 导航参数的使用
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 根据导航参数决定是否重用
    if (navigationContext.Parameters.ContainsKey("forceNew"))
    {
        return !(bool)navigationContext.Parameters["forceNew"];
    }
    
    // 默认重用策略
    return this.SomeCondition;
}
```

### 状态保持与恢复
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 保存当前状态到导航参数
    navigationContext.Parameters["SavedState"] = this.CurrentState;
    
    // 决定是否重用
    return this.CanReuse;
}

public void OnNavigatedTo(NavigationContext navigationContext)
{
    // 恢复之前保存的状态
    if (navigationContext.Parameters.ContainsKey("SavedState"))
    {
        this.CurrentState = navigationContext.Parameters["SavedState"];
    }
}
```

## 扩展应用

### 主从视图模式
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 主视图：根据选中项ID决定是否重用
    if (navigationContext.Parameters.ContainsKey("selectedId"))
    {
        var selectedId = navigationContext.Parameters["selectedId"].ToString();
        return this.CurrentItemId == selectedId;
    }
    return true;
}
```

### 工作流状态管理
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 根据工作流状态决定是否重用
    switch (this.WorkflowState)
    {
        case WorkflowState.Editing:
            return false; // 编辑状态不重用，保持数据隔离
        case WorkflowState.Viewing:
            return true;  // 查看状态可以重用
        default:
            return true;
    }
}
```

### 内存优化策略
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 内存压力大时避免创建新实例
    if (GC.GetTotalMemory(false) > MemoryThreshold)
    {
        return true; // 强制重用现有实例
    }
    
    // 正常情况下根据业务逻辑决定
    return this.NormalReuseLogic();
}
```

## 性能考虑

### 重用 vs 创建新实例
- **重用现有实例**：
  - ✅ 优点：内存效率高，状态保持，响应速度快
  - ❌ 缺点：可能持有过时数据，状态管理复杂

- **创建新实例**：
  - ✅ 优点：数据新鲜，状态隔离，安全性高
  - ❌ 缺点：内存消耗大，创建开销，GC压力

### 最佳实践
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    // 1. 检查是否需要强制刷新
    if (RequiresFreshData(navigationContext))
        return false;
    
    // 2. 检查内存使用情况
    if (MemoryPressureHigh())
        return true;
    
    // 3. 检查业务逻辑
    if (!MeetsReuseConditions(navigationContext))
        return false;
    
    // 4. 默认重用
    return true;
}
```

## 学习总结

导航到现有视图机制在实际应用中非常重要，它提供了：

1. **性能优化**：通过智能重用减少资源消耗
2. **状态管理**：精确控制视图状态和生命周期
3. **用户体验**：保持用户操作状态，避免重复输入
4. **内存管理**：根据系统状况调整重用策略
5. **业务逻辑**：支持复杂的业务重用规则

通过本项目中的访问计数逻辑，可以直观地看到视图实例何时被重用，何时被重新创建，这对于理解 Prism 的导航机制和优化应用性能具有重要意义。