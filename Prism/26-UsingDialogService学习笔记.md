# 26-UsingDialogService 学习笔记

## 🎯 核心概念：对话框服务

DialogService是Prism框架中用于**统一管理对话框**的核心服务，它提供了一种**解耦的、可测试的**方式来显示对话框，避免了直接依赖于具体UI实现。

## 🔑 关键技术点

### 1. 对话框服务接口
```csharp
IDialogService _dialogService;
```

### 2. 核心方法
- `ShowDialog()` - 显示模态对话框
- `ShowNotification()` - 显示通知对话框（在某些实现中）

### 3. 对话框感知接口
```csharp
public class NotificationDialogViewModel : BindableBase, IDialogAware
```

## 🏗️ 项目架构分析

### 1. 主窗口结构
```xml
<Window x:Class="UsingDialogService.Views.MainWindow"
        xmlns:prism="http://prismlibrary.com/"
        prism:ViewModelLocator.AutoWireViewModel="True">
    <!-- 显示对话框按钮 -->
    <Button Command="{Binding Path=ShowDialogCommand}" Content="Show Dialog" />
</Window>
```

### 2. 对话框视图
```xml
<UserControl x:Class="UsingDialogService.Views.NotificationDialog"
             xmlns:prism="http://prismlibrary.com/"             
             prism:ViewModelLocator.AutoWireViewModel="True"
             Width="300" Height="150">
    <Grid>
        <!-- 消息显示 -->
        <TextBlock Text="{Binding Message}" TextWrapping="Wrap" />
        
        <!-- 操作按钮 -->
        <StackPanel Orientation="Horizontal">
            <Button Command="{Binding CloseDialogCommand}" CommandParameter="true" Content="OK" />
            <Button Command="{Binding CloseDialogCommand}" CommandParameter="false" Content="Cancel" />
        </StackPanel>
    </Grid>
</UserControl>
```

## 🔄 对话框机制详解

### 1. 应用程序配置
```csharp
public partial class App
{
    protected override void RegisterTypes(IContainerRegistry containerRegistry)
    {
        // 注册对话框视图和ViewModel
        containerRegistry.RegisterDialog<NotificationDialog, NotificationDialogViewModel>();
    }
}
```

### 2. 主窗口ViewModel
```csharp
public class MainWindowViewModel : BindableBase
{
    private IDialogService _dialogService;

    public MainWindowViewModel(IDialogService dialogService)
    {
        _dialogService = dialogService;
    }

    private void ShowDialog()
    {
        var message = "This is a message that should be shown in the dialog.";
        
        // 使用对话框服务显示对话框
        _dialogService.ShowDialog("NotificationDialog", new DialogParameters($"message={message}"), r =>
        {
            // 处理对话框关闭后的结果
            if (r.Result == ButtonResult.None)
                Title = "Result is None";
            else if (r.Result == ButtonResult.OK)
                Title = "Result is OK";
            else if (r.Result == ButtonResult.Cancel)
                Title = "Result is Cancel";
            else
                Title = "I Don't know what you did!?";
        });
    }
}
```

### 3. 对话框ViewModel
```csharp
public class NotificationDialogViewModel : BindableBase, IDialogAware
{
    private string _message;
    public string Message
    {
        get { return _message; }
        set { SetProperty(ref _message, value); }
    }

    // 对话框标题
    private string _title = "Notification";
    public string Title
    {
        get { return _title; }
        set { SetProperty(ref _title, value); }
    }

    // 对话框关闭事件
    public DialogCloseListener RequestClose { get; }

    // 关闭对话框命令
    private DelegateCommand<string> _closeDialogCommand;
    public DelegateCommand<string> CloseDialogCommand =>
        _closeDialogCommand ?? (_closeDialogCommand = new DelegateCommand<string>(CloseDialog));

    protected virtual void CloseDialog(string parameter)
    {
        ButtonResult result = ButtonResult.None;

        if (parameter?.ToLower() == "true")
            result = ButtonResult.OK;
        else if (parameter?.ToLower() == "false")
            result = ButtonResult.Cancel;

        RaiseRequestClose(new DialogResult(result));
    }

    public virtual void RaiseRequestClose(IDialogResult dialogResult)
    {
        RequestClose.Invoke(dialogResult);
    }

    // 对话框打开时的处理
    public virtual void OnDialogOpened(IDialogParameters parameters)
    {
        Message = parameters.GetValue<string>("message");
    }

    // 对话框关闭时的处理
    public virtual void OnDialogClosed()
    {
        // 可以在这里执行清理操作
    }

    // 是否可以关闭对话框
    public virtual bool CanCloseDialog()
    {
        return true;
    }
}
```

## 🎯 对话框工作流程

### 1. 对话框注册
```
App.xaml.cs → RegisterDialog<NotificationDialog, NotificationDialogViewModel>()
    ↓
Prism容器注册对话框视图和ViewModel的映射关系
```

### 2. 对话框显示
```
MainWindowViewModel → ShowDialog() → _dialogService.ShowDialog()
    ↓
传递对话框名称、参数和回调函数
```

### 3. 参数传递
```
DialogParameters($"message={message}")
    ↓
OnDialogOpened() → Message = parameters.GetValue<string>("message")
```

### 4. 结果处理
```
CloseDialog() → RaiseRequestClose() → 回调函数处理结果
```

## 🎯 企业级应用场景

### 1. 用户确认对话框
- **删除确认**：删除操作前的确认对话框
- **保存确认**：关闭文档前的保存确认
- **退出确认**：应用程序退出前的确认

### 2. 信息通知对话框
- **错误提示**：系统错误信息的显示
- **操作结果**：操作成功或失败的通知
- **系统消息**：重要系统消息的通知

### 3. 数据输入对话框
- **用户信息**：用户信息的输入和编辑
- **配置设置**：系统配置参数的设置
- **表单填写**：复杂表单的分步填写

### 4. 业务流程对话框
- **审批流程**：审批意见的输入
- **订单处理**：订单信息的确认
- **任务分配**：任务分配的确认

## 💡 最佳实践

### 1. 对话框注册
```csharp
protected override void RegisterTypes(IContainerRegistry containerRegistry)
{
    // 注册对话框视图和ViewModel
    containerRegistry.RegisterDialog<NotificationDialog, NotificationDialogViewModel>();
    
    // 可以注册多个对话框
    containerRegistry.RegisterDialog<ConfirmationDialog, ConfirmationDialogViewModel>();
    containerRegistry.RegisterDialog<InputDialog, InputDialogViewModel>();
}
```

### 2. 参数传递
```csharp
// 传递单个参数
var parameters = new DialogParameters($"message={message}");

// 传递多个参数
var parameters = new DialogParameters();
parameters.Add("title", "确认操作");
parameters.Add("message", "您确定要执行此操作吗？");
parameters.Add("confirmText", "确定");
parameters.Add("cancelText", "取消");

_dialogService.ShowDialog("ConfirmationDialog", parameters, r => {
    // 处理结果
});
```

### 3. 结果处理
```csharp
_dialogService.ShowDialog("NotificationDialog", parameters, r => {
    switch (r.Result)
    {
        case ButtonResult.OK:
            // 处理确认操作
            ProcessConfirmation();
            break;
        case ButtonResult.Cancel:
            // 处理取消操作
            ProcessCancellation();
            break;
        default:
            // 处理其他情况
            break;
    }
});
```

### 4. 对话框验证
```csharp
public virtual bool CanCloseDialog()
{
    // 验证输入数据
    if (string.IsNullOrEmpty(UserInput))
    {
        // 显示错误消息
        MessageBox.Show("请输入必要信息");
        return false;
    }
    
    return true;
}
```

## 🚀 技术优势

1. **解耦设计**：ViewModel不直接依赖UI实现
2. **可测试性**：可以轻松模拟对话框服务进行单元测试
3. **统一管理**：所有对话框通过统一服务管理
4. **参数传递**：支持复杂参数的传递和接收
5. **结果回调**：支持异步结果处理
6. **MVVM友好**：完美融入MVVM架构模式

## 📊 对比其他对话框方式

| 特性 | DialogService | 直接MessageBox | 自定义对话框管理 |
|------|---------------|----------------|------------------|
| **解耦性** | ✅ 高度解耦 | ❌ 紧耦合 | ⚠️ 中等耦合 |
| **可测试性** | ✅ 易于测试 | ❌ 难以测试 | ⚠️ 中等测试难度 |
| **统一管理** | ✅ 统一服务 | ❌ 分散调用 | ⚠️ 需要手动管理 |
| **参数传递** | ✅ 支持复杂参数 | ❌ 仅简单文本 | ✅ 支持复杂参数 |
| **结果处理** | ✅ 异步回调 | ❌ 阻塞调用 | ✅ 异步回调 |
| **使用复杂度** | ⚠️ 中等 | ✅ 简单 | ❌ 复杂 |

## 🎯 总结

**DialogService**通过`IDialogService`接口和`IDialogAware`接口，实现了**统一的对话框管理机制**，是构建**企业级WPF应用**中**对话框处理**的**核心技术**。它提供了**解耦的设计**、**可测试的架构**、**统一的管理方式**和**灵活的参数传递机制**，为**用户交互**提供了**完整的解决方案**。

**核心价值**：
1. **架构解耦**：ViewModel与具体UI实现解耦
2. **易于测试**：可以轻松模拟对话框服务进行测试
3. **统一管理**：所有对话框通过统一服务管理
4. **灵活扩展**：支持自定义对话框和复杂参数传递