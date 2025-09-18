# 18-NavigationCallback 项目学习笔记

## 项目概述

NavigationCallback 项目演示了如何在 Prism 框架中使用导航回调（Navigation Callback）机制来处理导航完成后的操作。当区域导航完成后，可以通过回调函数获取导航结果并执行相应的后续处理。

## 核心概念

### 导航回调的作用
导航回调允许开发者在导航操作完成后执行自定义逻辑，例如：
- 确认导航是否成功
- 获取导航的详细信息
- 执行导航完成后的清理工作
- 显示导航结果提示

### 关键特性
- **异步处理**：导航完成后异步执行回调
- **结果信息**：提供详细的导航结果信息
- **错误处理**：可以处理导航失败的情况
- **用户反馈**：可以向用户显示导航状态

## 项目结构

```
18-NavigationCallback/
├── BasicRegionNavigation/        # 主应用程序
│   ├── App.xaml                 # 应用程序入口，配置 Prism
│   ├── App.xaml.cs              # 应用程序逻辑，注册模块
│   ├── Views/
│   │   └── MainWindow.xaml      # 主窗口，包含导航按钮和 ContentRegion
│   └── ViewModels/
│       └── MainWindowViewModel.cs # 包含导航逻辑和回调处理
└── ModuleA/                     # 功能模块
    ├── ModuleAModule.cs         # 模块初始化，注册可导航视图
    ├── Views/
    │   ├── ViewA.xaml           # 视图A
    │   └── ViewB.xaml           # 视图B
    └── ViewModels/
        ├── ViewAViewModel.cs    # 视图A的ViewModel
        └── ViewBViewModel.cs    # 视图B的ViewModel
```

## 关键实现

### 1. 导航命令实现 (MainWindowViewModel.cs)
```csharp
public DelegateCommand<string> NavigateCommand { get; private set; }

public MainWindowViewModel(IRegionManager regionManager)
{
    _regionManager = regionManager;
    NavigateCommand = new DelegateCommand<string>(Navigate);
}

private void Navigate(string navigatePath)
{
    if (navigatePath != null)
        _regionManager.RequestNavigate("ContentRegion", navigatePath, NavigationComplete);
}

private void NavigationComplete(NavigationResult result)
{
    System.Windows.MessageBox.Show(String.Format("Navigation to {0} complete. ", result.Context.Uri));
}
```

**关键点**：
- `RequestNavigate` 方法接受第三个参数作为回调函数
- 回调函数接收 `NavigationResult` 对象，包含导航的详细信息
- 可以根据 `result.Result` 判断导航是否成功

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
- 两个按钮分别触发导航到 ViewA 和 ViewB
- 使用 `CommandParameter` 传递目标视图的名称
- ContentControl 作为导航的目标区域

### 3. 视图注册 (ModuleAModule.cs)
```csharp
public void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.RegisterForNavigation<ViewA>();
    containerRegistry.RegisterForNavigation<ViewB>();
}
```

**关键点**：
- 使用 `RegisterForNavigation` 方法注册可导航的视图
- 注册后的视图可以通过名称进行导航

### 4. NavigationResult 对象
`NavigationResult` 包含以下重要属性：
- `Result`：导航结果（成功/失败）
- `Context`：导航上下文信息
- `Context.Uri`：目标视图的 URI
- `Error`：导航失败时的异常信息

## 工作流程

1. **初始化**：应用程序启动时，ModuleA 注册 ViewA 和 ViewB 为可导航视图

2. **用户交互**：用户点击导航按钮，触发 `NavigateCommand`

3. **导航请求**：ViewModel 调用 `_regionManager.RequestNavigate`，传入目标视图名称和回调函数

4. **导航执行**：Prism 框架执行导航操作，将目标视图加载到 ContentRegion

5. **回调执行**：导航完成后，自动调用 `NavigationComplete` 回调函数

6. **结果处理**：在回调函数中处理导航结果，显示提示信息

## 技术要点

### 导航回调的使用方式
```csharp
// 基本导航回调
_regionManager.RequestNavigate("RegionName", "ViewName", NavigationComplete);

// 回调函数定义
private void NavigationComplete(NavigationResult result)
{
    if (result.Result == true)
    {
        // 导航成功
        Console.WriteLine($"Navigation to {result.Context.Uri} completed successfully");
    }
    else
    {
        // 导航失败
        Console.WriteLine($"Navigation failed: {result.Error?.Message}");
    }
}
```

### 错误处理
```csharp
private void NavigationComplete(NavigationResult result)
{
    if (result.Error != null)
    {
        // 处理导航错误
        MessageBox.Show($"Navigation failed: {result.Error.Message}");
        return;
    }
    
    // 导航成功，执行后续操作
    // ...
}
```

### 导航上下文信息
```csharp
private void NavigationComplete(NavigationResult result)
{
    var navigationContext = result.Context;
    
    // 获取目标 URI
    var targetUri = navigationContext.Uri;
    
    // 获取导航参数
    var navigationParameters = navigationContext.Parameters;
    
    // 获取导航服务
    var navigationService = navigationContext.NavigationService;
}
```

## 扩展应用

### 条件导航
```csharp
private void NavigationComplete(NavigationResult result)
{
    if (result.Result == true)
    {
        // 导航成功后，可以执行其他操作
        // 如：记录日志、更新状态、加载数据等
        LogNavigationSuccess(result.Context.Uri);
        UpdateApplicationState();
    }
}
```

### 导航链
```csharp
private void NavigationComplete(NavigationResult result)
{
    if (result.Result == true && result.Context.Uri == "ViewA")
    {
        // 导航到 ViewA 成功后，可以继续导航到其他视图
        _regionManager.RequestNavigate("ContentRegion", "ViewC", NextNavigationComplete);
    }
}
```

## 学习总结

导航回调机制在实际应用中非常重要，它提供了：

1. **导航状态监控**：可以实时监控导航操作的状态
2. **错误处理机制**：能够捕获和处理导航失败的情况
3. **后续操作执行**：导航完成后可以执行必要的后续操作
4. **用户体验优化**：可以向用户提供导航反馈
5. **调试和日志**：便于调试导航问题和记录导航历史

这种模式在企业级应用中非常实用，特别是在需要复杂导航逻辑和状态管理的场景中。