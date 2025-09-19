# Stylet WindowManager 设计与实现详解

## 概述

WindowManager 是 Stylet 框架中负责窗口和对话框管理的核心组件，它提供了以 ViewModel 为中心的窗口显示机制。通过 WindowManager，开发者可以轻松地显示窗口、对话框，并处理窗口的生命周期事件，而无需直接操作 WPF 的 Window 类。这个类实现了 ViewModel-First 架构中窗口管理的重要部分，使得窗口操作完全符合 MVVM 模式。

## 设计目标

1. **ViewModel-First 窗口管理**：以 ViewModel 为中心创建和管理窗口
2. **统一接口**：提供一致的窗口和对话框显示接口
3. **生命周期集成**：与 Stylet 的屏幕生命周期管理集成
4. **异步关闭支持**：支持异步的窗口关闭确认机制
5. **所有者窗口管理**：自动处理窗口的所有者关系
6. **消息框集成**：提供统一的消息框显示接口

## 核心接口设计

### IWindowManager 接口

```csharp
public interface IWindowManager
{
    void ShowWindow(object viewModel);
    void ShowWindow(object viewModel, IViewAware ownerViewModel);
    bool? ShowDialog(object viewModel);
    bool? ShowDialog(object viewModel, IViewAware ownerViewModel);
    MessageBoxResult ShowMessageBox(string messageBoxText, string caption = "", ...);
}
```

**接口特点：**
- **简洁性**：提供最小化的必要操作
- **一致性**：窗口和对话框的显示接口保持一致
- **灵活性**：支持所有者窗口的显式指定
- **完整性**：包含消息框显示功能

### IWindowManagerConfig 配置接口

```csharp
public interface IWindowManagerConfig
{
    Window GetActiveWindow();
}
```

**配置抽象：**
- 提供获取当前活动窗口的能力
- 允许不同的配置实现
- 支持依赖注入配置

## 核心功能详解

### 1. 窗口创建和配置

#### CreateWindow 方法核心逻辑

```csharp
protected virtual Window CreateWindow(object viewModel, bool isDialog, IViewAware ownerViewModel)
{
    UIElement view = this.viewManager.CreateAndBindViewForModelIfNecessary(viewModel);
    if (view is not Window window)
    {
        var e = new StyletInvalidViewTypeException(string.Format("WindowManager.ShowWindow or .ShowDialog tried to show a View of type '{0}', but that View doesn't derive from the Window class. " +
            "Make sure any Views you display using WindowManager.ShowWindow or .ShowDialog derive from Window (not UserControl, etc)",
            view == null ? "(null)" : view.GetType().Name));
        logger.Error(e);
        throw e;
    }

    // 标题绑定：只有在未设置标题且没有绑定的情况下
    if (viewModel is IHaveDisplayName haveDisplayName && (string.IsNullOrEmpty(window.Title) || window.Title == view.GetType().Name) && 
        BindingOperations.GetBindingBase(window, Window.TitleProperty) == null)
    {
        var binding = new Binding("DisplayName") { Mode = BindingMode.TwoWay };
        window.SetBinding(Window.TitleProperty, binding);
    }

    // 所有者窗口设置
    if (ownerViewModel?.View is Window explicitOwner)
    {
        window.Owner = explicitOwner;
    }
    else if (isDialog)
    {
        Window owner = this.InferOwnerOf(window);
        if (owner != null)
        {
            try
            {
                window.Owner = owner;
            }
            catch (InvalidOperationException e)
            {
                logger.Error(e, "This can occur when the application is closing down");
            }
        }
    }

    // 窗口位置设置
    if (window.WindowStartupLocation == WindowStartupLocation.Manual && double.IsNaN(window.Top) && double.IsNaN(window.Left) &&
        BindingOperations.GetBinding(window, Window.TopProperty) == null && BindingOperations.GetBinding(window, Window.LeftProperty) == null)
    {
        window.WindowStartupLocation = window.Owner == null ? WindowStartupLocation.CenterScreen : WindowStartupLocation.CenterOwner;
    }

    // 创建窗口指挥器管理生命周期
    new WindowConductor(window, viewModel);

    return window;
}
```

**设计亮点：**

1. **类型安全检查**：确保视图是 Window 类型
2. **智能标题绑定**：自动绑定 DisplayName 到窗口标题
3. **所有者推断**：智能推断对话框的所有者窗口
4. **位置管理**：自动设置窗口启动位置
5. **生命周期管理**：创建 WindowConductor 管理窗口生命周期

#### 所有者窗口推断

```csharp
private Window InferOwnerOf(Window window)
{
    Window active = this.getActiveWindow();
    return ReferenceEquals(active, window) ? null : active;
}
```

**推断逻辑：**
- 获取当前活动窗口
- 避免窗口拥有自身
- 支持对话框的自动所有者设置

### 2. 窗口生命周期管理

#### WindowConductor 类设计

WindowManager 使用内部的 WindowConductor 类来管理窗口的生命周期，这是该类的核心设计特色：

```csharp
private class WindowConductor : IChildDelegate
{
    private readonly Window window;
    private readonly object viewModel;

    public WindowConductor(Window window, object viewModel)
    {
        this.window = window;
        this.viewModel = viewModel;

        // 设置父子关系
        if (this.viewModel is IChild viewModelAsChild)
            viewModelAsChild.Parent = this;

        // 激活视图模型
        ScreenExtensions.TryActivate(this.viewModel);

        // 注册窗口事件
        if (this.viewModel is IScreenState viewModelAsScreenState)
        {
            window.StateChanged += this.WindowStateChanged;
            window.Closed += this.WindowClosed;
        }

        // 注册关闭检查
        if (this.viewModel is IGuardClose)
            window.Closing += this.WindowClosing;
    }
}
```

**生命周期集成：**
- **父子关系**：建立窗口和视图模型的父子关系
- **激活管理**：自动激活视图模型
- **事件注册**：注册窗口状态变更和关闭事件
- **关闭保护**：支持异步关闭确认

#### 窗口状态管理

```csharp
private void WindowStateChanged(object sender, EventArgs e)
{
    switch (this.window.WindowState)
    {
        case WindowState.Maximized:
        case WindowState.Normal:
            logger.Info("Window {0} maximized/restored: activating", this.window);
            ScreenExtensions.TryActivate(this.viewModel);
            break;

        case WindowState.Minimized:
            logger.Info("Window {0} minimized: deactivating", this.window);
            ScreenExtensions.TryDeactivate(this.viewModel);
            break;
    }
}
```

**状态同步：**
- 最大化/正常状态：激活视图模型
- 最小化状态：停用视图模型
- 自动同步窗口和视图模型状态

#### 窗口关闭处理

```csharp
private void WindowClosed(object sender, EventArgs e)
{
    this.window.StateChanged -= this.WindowStateChanged;
    this.window.Closed -= this.WindowClosed;
    this.window.Closing -= this.WindowClosing;

    ScreenExtensions.TryClose(this.viewModel);
}
```

**清理工作：**
- 取消事件注册
- 关闭视图模型
- 确保资源正确清理

### 3. 异步关闭支持

#### 关闭确认机制

```csharp
private async void WindowClosing(object sender, CancelEventArgs e)
{
    if (e.Cancel)
        return;

    logger.Info("ViewModel {0} close requested because its View was closed", this.viewModel);

    // 检查同步完成情况
    System.Threading.Tasks.Task<bool> task = ((IGuardClose)this.viewModel).CanCloseAsync();
    if (task.IsCompleted)
    {
        // 同步完成，直接处理结果
        if (!task.Result)
            logger.Info("Close of ViewModel {0} cancelled because CanCloseAsync returned false", this.viewModel);
        e.Cancel = !task.Result;
    }
    else
    {
        // 异步完成，需要延迟关闭
        e.Cancel = true;
        logger.Info("Delaying closing of ViewModel {0} because CanCloseAsync is completing asynchronously", this.viewModel);
        if (await task)
        {
            this.window.Closing -= this.WindowClosing;
            this.window.Close();
        }
        else
        {
            logger.Info("Close of ViewModel {0} cancelled because CanCloseAsync returned false", this.viewModel);
        }
    }
}
```

**异步处理策略：**
- **同步优化**：如果任务已完成，直接处理结果
- **异步延迟**：对于异步任务，先取消关闭，等待结果
- **错误处理**：记录关闭取消的原因
- **事件管理**：正确处理事件处理器的移除

### 4. 消息框功能

#### 统一消息框接口

```csharp
public MessageBoxResult ShowMessageBox(string messageBoxText, string caption = "",
    MessageBoxButton buttons = MessageBoxButton.OK,
    MessageBoxImage icon = MessageBoxImage.None,
    MessageBoxResult defaultResult = MessageBoxResult.None,
    MessageBoxResult cancelResult = MessageBoxResult.None,
    IDictionary<MessageBoxResult, string> buttonLabels = null,
    FlowDirection? flowDirection = null,
    TextAlignment? textAlignment = null)
{
    IMessageBoxViewModel vm = this.messageBoxViewModelFactory();
    vm.Setup(messageBoxText, caption, buttons, icon, defaultResult, cancelResult, buttonLabels, flowDirection, textAlignment);
    this.ShowDialog(vm);
    return vm.ClickedButton;
}
```

**功能特点：**
- **完整参数支持**：支持所有标准 MessageBox 参数
- **自定义标签**：支持自定义按钮标签
- **本地化支持**：支持流方向和文本对齐设置
- **统一接口**：与窗口显示接口保持一致

### 5. 父子窗口关系管理

#### IChildDelegate 实现

```csharp
async void IChildDelegate.CloseItem(object item, bool? dialogResult)
{
    if (item != this.viewModel)
    {
        logger.Warn("IChildDelegate.CloseItem called with item {0} which is _not_ our ViewModel {1}", item, this.viewModel);
        return;
    }

    if (this.viewModel is IGuardClose guardClose && !await guardClose.CanCloseAsync())
    {
        logger.Info("Close of ViewModel {0} cancelled because CanCloseAsync returned false", this.viewModel);
        return;
    }

    logger.Info("ViewModel {0} close requested with DialogResult {1} because it called RequestClose", this.viewModel, dialogResult);

    // 清理事件处理器
    this.window.StateChanged -= this.WindowStateChanged;
    this.window.Closed -= this.WindowClosed;
    this.window.Closing -= this.WindowClosing;

    // 设置对话框结果
    if (dialogResult != null)
        this.window.DialogResult = dialogResult;

    ScreenExtensions.TryClose(this.viewModel);
    this.window.Close();
}
```

**关系管理：**
- **身份验证**：确保关闭请求针对正确的视图模型
- **关闭检查**：执行异步关闭确认
- **结果设置**：支持对话框结果设置
- **生命周期管理**：正确关闭视图模型和窗口

## 使用模式和最佳实践

### 基本窗口显示

```csharp
public class MyViewModel
{
    private readonly IWindowManager windowManager;
    
    public MyViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
    }
    
    public void ShowDetails()
    {
        var detailsViewModel = new DetailsViewModel();
        this.windowManager.ShowWindow(detailsViewModel);
    }
}
```

### 对话框显示和结果处理

```csharp
public async Task<bool> EditItem()
{
    var editViewModel = new EditViewModel();
    var result = this.windowManager.ShowDialog(editViewModel);
    return result ?? false;
}
```

### 带所有者的窗口显示

```csharp
public void ShowChildWindow()
{
    var childViewModel = new ChildViewModel();
    this.windowManager.ShowWindow(childViewModel, this);
}
```

### 异步关闭确认

```csharp
public class ConfirmCloseViewModel : Screen, IGuardClose
{
    public async Task<bool> CanCloseAsync()
    {
        var result = await this.dialogManager.AskUserAsync("确定要关闭吗？");
        return result == UserResponse.Yes;
    }
}
```

### 消息框使用

```csharp
public void ShowMessage()
{
    var result = this.windowManager.ShowMessageBox(
        "操作成功完成",
        "提示",
        MessageBoxButton.OK,
        MessageBoxImage.Information);
}
```

## 设计优势

### 1. ViewModel-First 架构
- **完全解耦**：视图模型不依赖具体的窗口实现
- **可测试性**：易于单元测试和模拟
- **灵活性**：支持不同的视图实现

### 2. 生命周期集成
- **自动管理**：与 Stylet 屏幕生命周期自动集成
- **状态同步**：窗口状态与视图模型状态同步
- **事件处理**：完整的窗口事件处理

### 3. 异步支持
- **非阻塞**：异步关闭确认不阻塞 UI
- **性能优化**：智能处理同步和异步场景
- **用户体验**：提供流畅的用户体验

### 4. 错误处理
- **类型安全**：运行时类型检查和验证
- **详细日志**：完整的操作日志记录
- **异常处理**：友好的异常信息和处理

### 5. 灵活性
- **自定义消息框**：支持自定义消息框视图模型
- **所有者管理**：灵活的所有者窗口设置
- **配置选项**：支持各种窗口配置选项

## 总结

WindowManager 是 Stylet 框架中窗口管理的核心组件，它通过精心设计的 API 和生命周期管理，为 WPF MVVM 应用程序提供了完整的窗口管理解决方案。其 ViewModel-First 的设计理念、异步关闭支持、与 Stylet 生命周期系统的深度集成，以及灵活的配置选项，使得它成为构建专业级 WPF 应用程序的重要基础设施。通过 WindowManager，开发者可以专注于业务逻辑的实现，而无需关心窗口管理的细节，真正实现了关注点分离和 MVVM 模式的最佳实践。