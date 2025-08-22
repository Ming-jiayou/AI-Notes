# Stylet IWindowManager 详细用法指南

## 概述

`IWindowManager` 是 Stylet 框架中用于管理窗口和对话框的核心接口。它提供了显示窗口、对话框和消息框的统一方式，隐藏了 WPF 中复杂的窗口管理细节。通过 `IWindowManager`，开发者可以专注于业务逻辑，而不需要关心视图和视图模型之间的复杂交互。

## 接口定义

`IWindowManager` 接口定义在 [`Stylet/WindowManager.cs`](Stylet/WindowManager.cs:13-64) 中，包含以下核心方法：

```csharp
public interface IWindowManager
{
    // 显示普通窗口
    void ShowWindow(object viewModel);
    void ShowWindow(object viewModel, IViewAware ownerViewModel);
    
    // 显示模态对话框
    bool? ShowDialog(object viewModel);
    bool? ShowDialog(object viewModel, IViewAware ownerViewModel);
    
    // 显示消息框
    MessageBoxResult ShowMessageBox(string messageBoxText, string caption = "",
        MessageBoxButton buttons = MessageBoxButton.OK,
        MessageBoxImage icon = MessageBoxImage.None,
        MessageBoxResult defaultResult = MessageBoxResult.None,
        MessageBoxResult cancelResult = MessageBoxResult.None,
        IDictionary<MessageBoxResult, string> buttonLabels = null,
        FlowDirection? flowDirection = null,
        TextAlignment? textAlignment = null);
}
```

## 核心功能详解

### 1. 显示普通窗口 (ShowWindow)

#### 基本用法
```csharp
// 最简单的用法
windowManager.ShowWindow(new MyViewModel());

// 指定所有者窗口
windowManager.ShowWindow(new MyViewModel(), ownerViewModel);
```

#### 实现原理
在 [`WindowManager.CreateWindow`](Stylet/WindowManager.cs:176-241) 方法中：
1. 通过 `IViewManager` 创建并绑定视图
2. 确保视图是 `Window` 类型
3. 自动设置窗口标题（如果视图模型实现了 `IHaveDisplayName`）
4. 设置窗口所有者
5. 配置窗口启动位置
6. 创建 `WindowConductor` 管理窗口生命周期

### 2. 显示模态对话框 (ShowDialog)

#### 基本用法
```csharp
// 显示对话框并获取结果
var result = windowManager.ShowDialog(new MyDialogViewModel());
if (result == true)
{
    // 用户点击了确定
}

// 指定所有者窗口
var result = windowManager.ShowDialog(new MyDialogViewModel(), ownerViewModel);
```

#### 特点
- 模态对话框会阻塞用户与其他窗口的交互
- 返回 `bool?` 类型，表示用户的操作结果
- 可以通过设置 `DialogResult` 来控制返回值

### 3. 显示消息框 (ShowMessageBox)

#### 基本用法
```csharp
// 简单消息
windowManager.ShowMessageBox("操作成功完成");

// 确认对话框
var result = windowManager.ShowMessageBox("确定要删除吗？", "确认删除", 
    MessageBoxButton.YesNo, MessageBoxImage.Question);
if (result == MessageBoxResult.Yes)
{
    // 执行删除操作
}
```

#### 高级配置
```csharp
// 自定义按钮文本
var buttonLabels = new Dictionary<MessageBoxResult, string>
{
    { MessageBoxResult.Yes, "确认删除" },
    { MessageBoxResult.No, "取消操作" }
};

var result = windowManager.ShowMessageBox(
    "此操作不可撤销，确定继续吗？",
    "警告",
    MessageBoxButton.YesNo,
    MessageBoxImage.Warning,
    MessageBoxResult.No,  // 默认按钮
    MessageBoxResult.No,  // 取消按钮
    buttonLabels);
```

## Stylet.Samples.Hello 示例分析

在 [`Samples/Stylet.Samples.Hello/ShellViewModel.cs`](Samples/Stylet.Samples.Hello/ShellViewModel.cs:20-31) 中展示了 `IWindowManager` 的典型用法：

```csharp
public class ShellViewModel : Screen
{
    private readonly IWindowManager windowManager;

    public ShellViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
    }

    public void SayHello()
    {
        this.windowManager.ShowMessageBox($"Hello, {this.Name}");
    }
}
```

### 关键点分析

1. **依赖注入**：`IWindowManager` 通过构造函数注入，这是 Stylet 推荐的方式
2. **简洁调用**：一行代码即可显示消息框
3. **自动绑定**：框架自动处理视图和视图模型的绑定

## 高级用法

### 1. 自定义对话框视图模型

创建自定义的对话框视图模型：

```csharp
public class UserInputViewModel : Screen
{
    public string UserInput { get; set; }
    
    public void OK()
    {
        this.DialogResult = true;
        this.RequestClose(true);
    }
    
    public void Cancel()
    {
        this.DialogResult = false;
        this.RequestClose(false);
    }
}
```

使用对话框：

```csharp
var dialog = new UserInputViewModel();
var result = windowManager.ShowDialog(dialog);
if (result == true)
{
    var input = dialog.UserInput;
}
```

### 2. 窗口生命周期管理

`WindowManager` 通过内部的 `WindowConductor` 类管理窗口生命周期：

- **激活/停用**：窗口最大化/恢复时激活视图模型，最小化时停用
- **关闭处理**：支持 `IGuardClose` 接口的异步关闭确认
- **事件清理**：自动清理事件处理器，防止内存泄漏

### 3. 自定义消息框

通过继承 `MessageBoxViewModel` 可以创建自定义消息框：

```csharp
public class CustomMessageBoxViewModel : MessageBoxViewModel
{
    public override string Text => "自定义消息内容";
    
    // 可以重写其他属性来自定义外观和行为
}
```

## 配置和扩展

### 1. 本地化支持

`MessageBoxViewModel` 提供了静态属性用于本地化：

```csharp
// 在应用程序启动时配置
MessageBoxViewModel.ButtonLabels[MessageBoxResult.OK] = "确定";
MessageBoxViewModel.ButtonLabels[MessageBoxResult.Cancel] = "取消";
MessageBoxViewModel.ButtonLabels[MessageBoxResult.Yes] = "是";
MessageBoxViewModel.ButtonLabels[MessageBoxResult.No] = "否";
```

### 2. 自定义图标和声音

```csharp
// 自定义图标映射
MessageBoxViewModel.IconMapping[MessageBoxImage.Information] = SystemIcons.Information;

// 自定义声音映射
MessageBoxViewModel.SoundMapping[MessageBoxImage.Error] = SystemSounds.Hand;
```

### 3. 默认样式设置

```csharp
// 设置默认文本对齐方式
MessageBoxViewModel.DefaultTextAlignment = TextAlignment.Center;

// 设置默认文本方向
MessageBoxViewModel.DefaultFlowDirection = FlowDirection.LeftToRight;
```

## 最佳实践

### 1. 依赖注入

始终通过构造函数注入 `IWindowManager`：

```csharp
public class MyViewModel : Screen
{
    private readonly IWindowManager windowManager;
    
    public MyViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager ?? throw new ArgumentNullException(nameof(windowManager));
    }
}
```

### 2. 错误处理

在使用对话框时考虑用户取消操作的情况：

```csharp
var result = windowManager.ShowDialog(new EditViewModel());
if (result == true)
{
    // 保存更改
}
else
{
    // 处理取消操作
}
```

### 3. 视图模型设计

确保用于 `ShowWindow` 和 `ShowDialog` 的视图模型：
- 继承自 `Screen` 或实现必要的接口
- 实现 `IHaveDisplayName` 以自动设置窗口标题
- 实现 `IGuardClose` 以支持关闭确认

### 4. 资源管理

避免在视图模型中直接引用 `Window` 对象，让 `WindowManager` 处理窗口生命周期。

## 常见问题

### 1. 视图类型错误

如果尝试显示非 `Window` 类型的视图，会抛出 `StyletInvalidViewTypeException`。确保所有通过 `WindowManager` 显示的视图都继承自 `System.Windows.Window`。

### 2. 所有者窗口设置

当应用程序关闭时，可能会出现所有者窗口设置异常。`WindowManager` 已经处理了这种情况，会记录错误但不会中断应用程序关闭。

### 3. 异步关闭确认

实现 `IGuardClose` 接口时，确保 `CanCloseAsync` 方法能够快速完成，以避免用户体验问题。

## 总结

`IWindowManager` 是 Stylet 框架中窗口管理的核心组件，它提供了：
- 简洁的 API 用于显示窗口和对话框
- 自动的视图/视图模型绑定
- 完整的生命周期管理
- 可扩展的消息框系统
- 良好的设计模式支持

通过使用 `IWindowManager`，开发者可以专注于业务逻辑，而不需要处理 WPF 窗口管理的复杂性。结合依赖注入和 MVVM 模式，可以创建出结构清晰、易于维护的 WPF 应用程序。