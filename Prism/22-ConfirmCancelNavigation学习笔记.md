# 22-ConfirmCancelNavigation 项目学习笔记

## 项目概述

ConfirmCancelNavigation 项目演示了如何在 Prism 框架中实现导航确认和取消（Confirm/Cancel Navigation）机制。通过实现 [`IConfirmNavigationRequest`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:8) 接口，视图可以在导航发生前进行确认，允许用户决定是否继续导航操作。

## 核心概念

### 导航确认的作用
导航确认机制允许视图：
- **用户确认**：在离开当前视图前获取用户确认
- **数据验证**：检查是否有未保存的更改
- **业务逻辑验证**：执行必要的业务逻辑检查
- **条件导航**：根据特定条件决定是否允许导航

### 关键特性
- **异步确认**：支持异步确认操作
- **用户交互**：可以显示对话框获取用户输入
- **条件控制**：根据业务逻辑决定是否允许导航
- **回调机制**：使用回调函数处理确认结果

## 项目结构

```
22-ConfirmCancelNavigation/
├── ConfirmCancelNavigation/      # 主应用程序
│   ├── App.xaml                 # 应用程序入口，配置 Prism
│   ├── App.xaml.cs              # 应用程序逻辑，注册模块
│   ├── Views/
│   │   └── MainWindow.xaml      # 主窗口，包含导航按钮和 ContentRegion
│   └── ViewModels/
│       └── MainWindowViewModel.cs # 导航命令逻辑
└── ModuleA/                     # 功能模块
    ├── ModuleAModule.cs         # 模块初始化，注册可导航视图
    ├── Views/
    │   ├── ViewA.xaml           # 视图A，实现导航确认
│   │   └── ViewB.xaml           # 视图B，普通视图
│   └── ViewModels/
│       ├── ViewAViewModel.cs    # 视图A的ViewModel，实现IConfirmNavigationRequest
│       └── ViewBViewModel.cs    # 视图B的ViewModel，普通ViewModel
```

## 关键实现

### 1. 导航确认接口实现 (ViewAViewModel.cs)
```csharp
public class ViewAViewModel : BindableBase, IConfirmNavigationRequest
{
    public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
    {
        bool result = true;

        if (MessageBox.Show("Do you want to navigate?", "Navigate?", MessageBoxButton.YesNo) == MessageBoxResult.No)
            result = false;

        continuationCallback(result);
    }

    public bool IsNavigationTarget(NavigationContext navigationContext)
    {
        return true;
    }

    public void OnNavigatedFrom(NavigationContext navigationContext)
    {
        // 导航离开时的清理工作
    }

    public void OnNavigatedTo(NavigationContext navigationContext)
    {
        // 导航到当前视图时的初始化工作
    }
}
```

**关键点**：
- 实现 [`IConfirmNavigationRequest`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:8) 接口
- [`ConfirmNavigationRequest`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:15) 方法在导航前被调用
- 使用 [`continuationCallback`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:15) 回调函数返回确认结果
- 返回 `true` 允许导航，返回 `false` 取消导航

### 2. 主窗口布局 (MainWindow.xaml)
```xml
<DockPanel LastChildFill="True">
    <StackPanel Orientation="Horizontal" DockPanel.Dock="Top" Margin="5" >
        <Button Command="{Binding NavigateCommand}" CommandParameter="ViewA" Margin="5">Navigate to View A</Button>
        <Button Command="{Binding NavigateCommand}" CommandParameter="ViewB" Margin="5">Navigate to View B</Button>
    </StackPanel>
    <ContentControl prism:RegionManager.RegionName="ContentRegion" Margin="5"  />
</DockPanel>
```

**关键点**：
- 两个导航按钮分别对应 ViewA（需要确认）和 ViewB（直接导航）
- ContentControl 作为导航的目标区域

### 3. 模块注册 (ModuleAModule.cs)
```csharp
public void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.RegisterForNavigation<ViewA>();
    containerRegistry.RegisterForNavigation<ViewB>();
}
```

### 4. 普通视图模型 (ViewBViewModel.cs)
```csharp
public class ViewBViewModel : BindableBase
{
    public ViewBViewModel()
    {
        // 普通 ViewModel，不需要导航确认
    }
}
```

**对比说明**：
- ViewB 使用普通的 [`BindableBase`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewBViewModel.cs:5)，导航时直接切换
- ViewA 实现 [`IConfirmNavigationRequest`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:8)，导航时会触发确认对话框

## 工作流程

1. **用户点击导航**：用户点击 "Navigate to View A" 按钮

2. **导航请求**：触发 [`RequestNavigate`](22-ConfirmCancelNavigation/ConfirmCancelNavigation/ViewModels/MainWindowViewModel.cs:30) 请求

3. **确认检查**：Prism 检测到目标视图实现了 [`IConfirmNavigationRequest`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:8)，调用 [`ConfirmNavigationRequest`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:15)

4. **用户交互**：显示确认对话框，询问用户是否继续导航

5. **结果处理**：
   - 用户选择 "Yes" → 调用 [`continuationCallback(true)`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:22) → 继续导航
   - 用户选择 "No" → 调用 [`continuationCallback(false)`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:22) → 取消导航

6. **导航执行**：根据确认结果执行或取消导航操作

## 技术要点

### IConfirmNavigationRequest 接口详解
```csharp
public interface IConfirmNavigationRequest : INavigationAware
{
    // 确认导航请求
    void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback);
}
```

### 基本确认逻辑
```csharp
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 简单的 Yes/No 确认
    bool result = MessageBox.Show("Continue navigation?", "Confirm", MessageBoxButton.YesNo) == MessageBoxResult.Yes;
    continuationCallback(result);
}
```

### 异步确认操作
```csharp
public async void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 异步确认（例如从服务器获取数据）
    bool hasUnsavedChanges = await CheckForUnsavedChangesAsync();
    
    if (hasUnsavedChanges)
    {
        var result = MessageBox.Show("You have unsaved changes. Continue?", "Unsaved Changes", MessageBoxButton.YesNo);
        continuationCallback(result == MessageBoxResult.Yes);
    }
    else
    {
        continuationCallback(true);
    }
}
```

### 条件确认逻辑
```csharp
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 根据导航目标决定是否需要确认
    if (navigationContext.Uri.OriginalString == "ViewB")
    {
        // 导航到 ViewB 不需要确认
        continuationCallback(true);
        return;
    }
    
    // 其他情况需要确认
    bool result = ShowConfirmationDialog();
    continuationCallback(result);
}
```

### 数据验证确认
```csharp
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 检查是否有未保存的更改
    if (HasUnsavedChanges)
    {
        var result = MessageBox.Show("You have unsaved changes. Save before navigating?", 
                                   "Unsaved Changes", 
                                   MessageBoxButton.YesNoCancel);
        
        switch (result)
        {
            case MessageBoxResult.Yes:
                SaveChanges();
                continuationCallback(true);
                break;
            case MessageBoxResult.No:
                continuationCallback(true);
                break;
            case MessageBoxResult.Cancel:
                continuationCallback(false);
                break;
        }
    }
    else
    {
        continuationCallback(true);
    }
}
```

## 扩展应用

### 权限验证
```csharp
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 检查用户权限
    if (!CurrentUser.HasPermission(navigationContext.Uri.OriginalString))
    {
        MessageBox.Show("You don't have permission to access this view.", "Access Denied", MessageBoxButton.OK);
        continuationCallback(false);
        return;
    }
    
    continuationCallback(true);
}
```

### 业务逻辑验证
```csharp
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 验证业务规则
    var validationResult = ValidateBusinessRules();
    
    if (!validationResult.IsValid)
    {
        MessageBox.Show($"Cannot navigate: {validationResult.ErrorMessage}", "Validation Error", MessageBoxButton.OK);
        continuationCallback(false);
        return;
    }
    
    continuationCallback(true);
}
```

### 自定义确认对话框
```csharp
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 使用自定义确认对话框
    var dialog = new CustomConfirmDialog
    {
        Title = "Confirm Navigation",
        Message = "Do you want to leave this view?",
        ConfirmButtonText = "Leave",
        CancelButtonText = "Stay"
    };
    
    var result = dialog.ShowDialog() == true;
    continuationCallback(result);
}
```

### 异步业务操作
```csharp
public async void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    try
    {
        // 执行异步业务操作
        var canNavigate = await CanNavigateAsync(navigationContext);
        
        if (!canNavigate)
        {
            MessageBox.Show("Navigation is not allowed at this time.", "Navigation Blocked", MessageBoxButton.OK);
        }
        
        continuationCallback(canNavigate);
    }
    catch (Exception ex)
    {
        MessageBox.Show($"Error during navigation confirmation: {ex.Message}", "Error", MessageBoxButton.OK);
        continuationCallback(false);
    }
}
```

### 状态保存提示
```csharp
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    if (HasUnsavedChanges)
    {
        var result = MessageBox.Show(
            "You have unsaved changes. What would you like to do?",
            "Unsaved Changes",
            MessageBoxButton.YesNoCancel,
            MessageBoxImage.Question,
            MessageBoxResult.Yes,
            MessageBoxOptions.None);
        
        switch (result)
        {
            case MessageBoxResult.Yes:
                // 保存并继续
                if (SaveChanges())
                {
                    continuationCallback(true);
                }
                else
                {
                    MessageBox.Show("Failed to save changes. Navigation cancelled.", "Save Error", MessageBoxButton.OK);
                    continuationCallback(false);
                }
                break;
            case MessageBoxResult.No:
                // 不保存但继续
                continuationCallback(true);
                break;
            case MessageBoxResult.Cancel:
                // 取消导航
                continuationCallback(false);
                break;
        }
    }
    else
    {
        continuationCallback(true);
    }
}
```

## 导航确认链

### 多个视图的确认处理
当导航涉及多个视图时（如嵌套区域），每个实现 [`IConfirmNavigationRequest`](22-ConfirmCancelNavigation/ModuleA/ViewModels/ViewAViewModel.cs:8) 的视图都会被询问：

```csharp
// 场景：从 ViewA 导航到 ViewB
// ViewA 实现了 IConfirmNavigationRequest
// 嵌套区域中的 ViewNested 也实现了 IConfirmNavigationRequest

// 确认顺序：
// 1. ViewA.ConfirmNavigationRequest() 被调用
// 2. ViewNested.ConfirmNavigationRequest() 被调用
// 3. 只有所有视图都返回 true，导航才会继续
```

### 父子视图确认协调
```csharp
public class ParentViewModel : BindableBase, IConfirmNavigationRequest
{
    private readonly IRegionManager _regionManager;
    
    public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
    {
        // 检查子视图是否可以导航
        var childViews = _regionManager.Regions["ChildRegion"].ActiveViews;
        bool canNavigate = true;
        
        foreach (var view in childViews)
        {
            if (view is IConfirmNavigationRequest confirmView)
            {
                // 同步确认（实际应用中应该异步处理）
                bool childResult = false;
                confirmView.ConfirmNavigationRequest(navigationContext, result => childResult = result);
                
                if (!childResult)
                {
                    canNavigate = false;
                    break;
                }
            }
        }
        
        if (canNavigate)
        {
            // 父视图自己的确认逻辑
            canNavigate = ShowParentConfirmationDialog();
        }
        
        continuationCallback(canNavigate);
    }
}
```

## 性能考虑

### 异步确认的重要性
```csharp
// 推荐：异步确认，不阻塞UI线程
public async void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 异步操作不会阻塞UI
    bool result = await SomeAsyncOperation();
    continuationCallback(result);
}

// 避免：同步长时间操作
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 长时间同步操作会阻塞UI线程
    Thread.Sleep(5000); // 不好的做法
    continuationCallback(true);
}
```

### 确认对话框优化
```csharp
public void ConfirmNavigationRequest(NavigationContext navigationContext, Action<bool> continuationCallback)
{
    // 缓存确认结果，避免重复询问
    if (_cachedConfirmationResult.HasValue)
    {
        continuationCallback(_cachedConfirmationResult.Value);
        return;
    }
    
    bool result = ShowConfirmationDialog();
    _cachedConfirmationResult = result;
    continuationCallback(result);
}
```

## 学习总结

导航确认机制在实际应用中非常重要，它提供了：

1. **用户控制**：让用户决定是否继续导航操作
2. **数据保护**：防止意外丢失未保存的数据
3. **业务逻辑验证**：确保导航操作符合业务规则
4. **权限控制**：根据用户权限控制导航行为
5. **状态管理**：在导航前后维护应用状态

通过本项目的学习，可以掌握 Prism 框架中导航确认的核心概念和实现方式，为构建用户友好的企业级应用奠定基础。导航确认是提升用户体验、保护数据安全的关键机制。