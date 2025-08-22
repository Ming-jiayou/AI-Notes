# Stylet MessageBox 详细使用指南

本文档基于Stylet源码深度分析，全面讲解MessageBox的使用方法、设计原理和高级特性。通过阅读本文档，你将掌握Stylet中MessageBox的所有使用技巧和最佳实践。

## 目录

1. [MessageBox概述](#messagebox概述)
2. [基本使用方法](#基本使用方法)
3. [MessageBox参数详解](#messagebox参数详解)
4. [返回值处理](#返回值处理)
5. [自定义MessageBox](#自定义messagebox)
6. [本地化支持](#本地化支持)
7. [高级用法示例](#高级用法示例)
8. [设计原理分析](#设计原理分析)
9. [最佳实践](#最佳实践)
10. [常见问题解答](#常见问题解答)

## MessageBox概述

Stylet的MessageBox是对WPF标准MessageBox的MVVM友好封装，提供了以下优势：

- **MVVM模式支持**：通过IWindowManager接口调用，符合MVVM设计原则
- **可测试性**：易于单元测试
- **可定制性**：支持自定义视图和视图模型
- **本地化**：支持按钮文本的本地化
- **样式统一**：与应用程序其他对话框保持一致的样式

### 核心组件

1. **IMessageBoxViewModel接口**：定义MessageBox视图模型的契约
2. **MessageBoxViewModel类**：默认的视图模型实现
3. **MessageBoxView.xaml**：默认的视图定义
4. **IWindowManager.ShowMessageBox()**：显示MessageBox的方法

## 基本使用方法

### 最简单的使用

```csharp
public class ShellViewModel : Screen
{
    private readonly IWindowManager windowManager;

    public ShellViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
    }

    public void ShowSimpleMessage()
    {
        // 最简单的消息框
        this.windowManager.ShowMessageBox("操作成功完成！");
    }
}
```

### 带标题的消息框

```csharp
public void ShowMessageWithTitle()
{
    this.windowManager.ShowMessageBox("文件已成功保存", "保存成功");
}
```

## MessageBox参数详解

### 完整参数列表

```csharp
MessageBoxResult ShowMessageBox(
    string messageBoxText,          // 消息文本（必需）
    string caption = "",            // 标题
    MessageBoxButton buttons = MessageBoxButton.OK,  // 按钮类型
    MessageBoxImage icon = MessageBoxImage.None,     // 图标类型
    MessageBoxResult defaultResult = MessageBoxResult.None,  // 默认按钮
    MessageBoxResult cancelResult = MessageBoxResult.None,   // 取消按钮
    IDictionary<MessageBoxResult, string> buttonLabels = null, // 自定义按钮文本
    FlowDirection? flowDirection = null,  // 文本方向
    TextAlignment? textAlignment = null   // 文本对齐方式
)
```

### 按钮类型（MessageBoxButton）

| 枚举值 | 显示的按钮 | 包含的返回值 |
|--------|------------|--------------|
| OK | 确定 | OK |
| OKCancel | 确定、取消 | OK、Cancel |
| YesNo | 是、否 | Yes、No |
| YesNoCancel | 是、否、取消 | Yes、No、Cancel |

### 图标类型（MessageBoxImage）

| 枚举值 | 图标 | 系统声音 |
|--------|------|----------|
| None | 无图标 | 无声音 |
| Error | 错误图标 | 错误提示音 |
| Question | 问号图标 | 问题提示音 |
| Exclamation | 感叹号图标 | 警告提示音 |
| Information | 信息图标 | 信息提示音 |

### 使用示例

```csharp
// 确认对话框
var result = this.windowManager.ShowMessageBox(
    "确定要删除选中的项目吗？此操作不可撤销。",
    "确认删除",
    MessageBoxButton.YesNo,
    MessageBoxImage.Question,
    MessageBoxResult.No,  // 默认选择"否"
    MessageBoxResult.No   // ESC键对应"否"
);

if (result == MessageBoxResult.Yes)
{
    // 执行删除操作
}
```

## 返回值处理

### 返回值类型（MessageBoxResult）

```csharp
public enum MessageBoxResult
{
    None = 0,
    OK = 1,
    Cancel = 2,
    Yes = 6,
    No = 7
}
```

### 实际使用示例

```csharp
public void DeleteItem(Item item)
{
    var result = this.windowManager.ShowMessageBox(
        $"确定要删除 \"{item.Name}\" 吗？",
        "确认删除",
        MessageBoxButton.YesNoCancel,
        MessageBoxImage.Warning);

    switch (result)
    {
        case MessageBoxResult.Yes:
            this.DeleteItemConfirmed(item);
            break;
        case MessageBoxResult.No:
            // 用户选择不删除，可能继续选择其他项目
            break;
        case MessageBoxResult.Cancel:
            // 用户取消操作，返回主流程
            break;
    }
}
```

## 自定义MessageBox

### 自定义按钮文本（本地化）

```csharp
public void ShowLocalizedMessage()
{
    var buttonLabels = new Dictionary<MessageBoxResult, string>
    {
        { MessageBoxResult.Yes, "确认" },
        { MessageBoxResult.No, "取消" },
        { MessageBoxResult.OK, "知道了" }
    };

    this.windowManager.ShowMessageBox(
        "操作已完成",
        "提示",
        MessageBoxButton.OK,
        MessageBoxImage.Information,
        buttonLabels: buttonLabels);
}
```

### 全局本地化配置

在应用程序启动时配置全局按钮文本：

```csharp
// 在Bootstrapper中配置
protected override void Configure()
{
    // 设置中文按钮文本
    MessageBoxViewModel.ButtonLabels = new Dictionary<MessageBoxResult, string>
    {
        { MessageBoxResult.OK, "确定" },
        { MessageBoxResult.Cancel, "取消" },
        { MessageBoxResult.Yes, "是" },
        { MessageBoxResult.No, "否" }
    };

    // 设置从右到左的文本方向（如阿拉伯语、希伯来语）
    MessageBoxViewModel.DefaultFlowDirection = FlowDirection.RightToLeft;
    
    // 设置文本对齐方式
    MessageBoxViewModel.DefaultTextAlignment = TextAlignment.Center;
}
```

### 自定义MessageBox视图模型

创建自定义的MessageBox视图模型：

```csharp
public class CustomMessageBoxViewModel : Screen, IMessageBoxViewModel
{
    public MessageBoxResult ClickedButton { get; private set; }

    public void Setup(string messageBoxText, string caption = null, 
        MessageBoxButton buttons = MessageBoxButton.OK, 
        MessageBoxImage icon = MessageBoxImage.None,
        MessageBoxResult defaultResult = MessageBoxResult.None,
        MessageBoxResult cancelResult = MessageBoxResult.None,
        IDictionary<MessageBoxResult, string> buttonLabels = null,
        FlowDirection? flowDirection = null,
        TextAlignment? textAlignment = null)
    {
        // 自定义实现
    }

    public void ButtonClicked(MessageBoxResult button)
    {
        this.ClickedButton = button;
        this.RequestClose(true);
    }
}
```

### 注册自定义MessageBox

在Bootstrapper中注册：

```csharp
protected override void ConfigureIoC(IStyletIoCBuilder builder)
{
    builder.Bind<IMessageBoxViewModel>().To<CustomMessageBoxViewModel>();
}
```

## 高级用法示例

### 错误处理

```csharp
public async Task SaveDataAsync()
{
    try
    {
        await this.dataService.SaveAsync(this.Data);
        this.windowManager.ShowMessageBox(
            "数据保存成功！",
            "保存成功",
            MessageBoxButton.OK,
            MessageBoxImage.Information);
    }
    catch (Exception ex)
    {
        this.logger.Error(ex, "保存数据失败");
        this.windowManager.ShowMessageBox(
            $"保存失败：{ex.Message}\n\n请检查网络连接后重试。",
            "保存失败",
            MessageBoxButton.OK,
            MessageBoxImage.Error);
    }
}
```

### 批量操作确认

```csharp
public void DeleteSelectedItems(List<Item> items)
{
    if (items.Count == 0)
    {
        this.windowManager.ShowMessageBox(
            "请先选择要删除的项目",
            "提示",
            MessageBoxButton.OK,
            MessageBoxImage.Information);
        return;
    }

    string message = items.Count == 1
        ? $"确定要删除 \"{items[0].Name}\" 吗？"
        : $"确定要删除选中的 {items.Count} 个项目吗？";

    var result = this.windowManager.ShowMessageBox(
        message,
        "确认删除",
        MessageBoxButton.YesNo,
        MessageBoxImage.Question);

    if (result == MessageBoxResult.Yes)
    {
        this.PerformBatchDelete(items);
    }
}
```

### 带超时的消息框（扩展实现）

```csharp
public class TimeoutMessageBoxViewModel : MessageBoxViewModel
{
    private readonly DispatcherTimer timer;
    private int remainingSeconds;

    public TimeoutMessageBoxViewModel(int timeoutSeconds)
    {
        this.remainingSeconds = timeoutSeconds;
        this.timer = new DispatcherTimer
        {
            Interval = TimeSpan.FromSeconds(1)
        };
        this.timer.Tick += this.OnTimerTick;
    }

    private void OnTimerTick(object sender, EventArgs e)
    {
        this.remainingSeconds--;
        if (this.remainingSeconds <= 0)
        {
            this.timer.Stop();
            this.ButtonClicked(MessageBoxResult.OK);
        }
    }

    protected override void OnViewLoaded()
    {
        base.OnViewLoaded();
        this.timer.Start();
    }
}
```

## 设计原理分析

### MVVM架构设计

Stylet的MessageBox设计遵循MVVM模式：

1. **视图分离**：MessageBoxView.xaml负责UI展示
2. **视图模型**：MessageBoxViewModel负责业务逻辑
3. **服务定位**：通过IWindowManager接口提供服务

### 依赖注入集成

```csharp
// WindowManager构造函数
public WindowManager(
    IViewManager viewManager,
    Func<IMessageBoxViewModel> messageBoxViewModelFactory,
    IWindowManagerConfig config)
```

- `messageBoxViewModelFactory`：工厂模式创建视图模型
- 支持IoC容器注入自定义实现

### 生命周期管理

1. **创建**：通过工厂创建IMessageBoxViewModel实例
2. **配置**：调用Setup方法配置参数
3. **显示**：通过ShowDialog显示模态对话框
4. **返回**：用户操作后返回MessageBoxResult

### 样式系统

MessageBoxView.xaml中的关键设计：

```xml
<!-- 图标显示 -->
<Image Source="{Binding ImageIcon, Converter={x:Static s:IconToBitmapSourceConverter.Instance}}"
       Visibility="{Binding ImageIcon, Converter={x:Static s:BoolToVisibilityConverter.Instance}}"/>

<!-- 文本显示 -->
<TextBlock Text="{Binding Text}" TextWrapping="Wrap" TextAlignment="{Binding TextAlignment}"/>

<!-- 按钮生成 -->
<ItemsControl ItemsSource="{Binding ButtonList}">
    <ItemsControl.ItemTemplate>
        <DataTemplate>
            <Button Content="{Binding Label}"
                    Command="{s:Action ButtonClicked}"
                    CommandParameter="{Binding Value}"/>
        </DataTemplate>
    </ItemsControl.ItemTemplate>
</ItemsControl>
```

## 最佳实践

### 1. 错误消息标准化

```csharp
public static class MessageBoxHelper
{
    public static void ShowError(this IWindowManager windowManager, string message)
    {
        windowManager.ShowMessageBox(message, "错误", MessageBoxButton.OK, MessageBoxImage.Error);
    }

    public static void ShowWarning(this IWindowManager windowManager, string message)
    {
        windowManager.ShowMessageBox(message, "警告", MessageBoxButton.OK, MessageBoxImage.Warning);
    }

    public static void ShowInfo(this IWindowManager windowManager, string message)
    {
        windowManager.ShowMessageBox(message, "提示", MessageBoxButton.OK, MessageBoxImage.Information);
    }

    public static bool ShowConfirm(this IWindowManager windowManager, string message)
    {
        return windowManager.ShowMessageBox(message, "确认", MessageBoxButton.YesNo, MessageBoxImage.Question) == MessageBoxResult.Yes;
    }
}
```

### 2. 异步操作的用户体验

```csharp
public async Task PerformLongRunningOperation()
{
    using (var busy = this.ShowBusy("正在处理，请稍候..."))
    {
        try
        {
            await this.DoLongRunningWorkAsync();
            this.windowManager.ShowMessageBox(
                "操作完成！",
                "成功",
                MessageBoxButton.OK,
                MessageBoxImage.Information);
        }
        catch (OperationCanceledException)
        {
            this.windowManager.ShowMessageBox(
                "操作已取消",
                "取消",
                MessageBoxButton.OK,
                MessageBoxImage.Information);
        }
        catch (Exception ex)
        {
            this.windowManager.ShowMessageBox(
                $"操作失败：{ex.Message}",
                "错误",
                MessageBoxButton.OK,
                MessageBoxImage.Error);
        }
    }
}
```

### 3. 输入验证反馈

```csharp
public void SaveSettings()
{
    var validationResult = this.ValidateSettings();
    if (!validationResult.IsValid)
    {
        var errorMessage = string.Join("\n", validationResult.Errors);
        this.windowManager.ShowMessageBox(
            $"请修正以下错误：\n\n{errorMessage}",
            "验证失败",
            MessageBoxButton.OK,
            MessageBoxImage.Warning);
        return;
    }

    this.settingsService.Save(this.Settings);
    this.windowManager.ShowMessageBox(
        "设置已保存",
        "成功",
        MessageBoxButton.OK,
        MessageBoxImage.Information);
}
```

## 常见问题解答

### Q1: 如何更改MessageBox的默认样式？

A: 可以通过以下方式自定义样式：

1. 创建自定义的MessageBoxView.xaml
2. 在Bootstrapper中注册自定义视图：

```csharp
protected override void ConfigureIoC(IStyletIoCBuilder builder)
{
    builder.Bind<IMessageBoxViewModel>().To<CustomMessageBoxViewModel>();
}
```

### Q2: 如何支持多语言？

A: 使用资源文件进行本地化：

```csharp
// 在Bootstrapper中配置
protected override void Configure()
{
    var resources = new ResourceManager("MyApp.Resources.Messages", typeof(App).Assembly);
    
    MessageBoxViewModel.ButtonLabels = new Dictionary<MessageBoxResult, string>
    {
        { MessageBoxResult.OK, resources.GetString("Button_OK") },
        { MessageBoxResult.Cancel, resources.GetString("Button_Cancel") },
        { MessageBoxResult.Yes, resources.GetString("Button_Yes") },
        { MessageBoxResult.No, resources.GetString("Button_No") }
    };
}
```

### Q3: 如何测试包含MessageBox的ViewModel？

A: 使用mock框架：

```csharp
[Test]
public void DeleteItem_ShowsConfirmation()
{
    // Arrange
    var mockWindowManager = new Mock<IWindowManager>();
    mockWindowManager.Setup(x => x.ShowMessageBox(
            It.IsAny<string>(),
            It.IsAny<string>(),
            It.IsAny<MessageBoxButton>(),
            It.IsAny<MessageBoxImage>()))
        .Returns(MessageBoxResult.Yes);

    var viewModel = new ItemViewModel(mockWindowManager.Object);
    var item = new Item { Name = "Test Item" };

    // Act
    viewModel.DeleteItem(item);

    // Assert
    mockWindowManager.Verify(x => x.ShowMessageBox(
        "确定要删除 \"Test Item\" 吗？",
        "确认删除",
        MessageBoxButton.YesNo,
        MessageBoxImage.Question), Times.Once);
}
```

### Q4: 如何处理ESC键和关闭按钮？

A: 通过cancelResult参数控制：

```csharp
var result = this.windowManager.ShowMessageBox(
    "确定要保存更改吗？",
    "保存确认",
    MessageBoxButton.YesNoCancel,
    MessageBoxImage.Question,
    cancelResult: MessageBoxResult.Cancel);

// ESC键或关闭窗口将返回Cancel
```

### Q5: 如何显示富文本内容？

A: 需要创建自定义的MessageBoxViewModel和视图：

```csharp
public class RichTextMessageBoxViewModel : MessageBoxViewModel
{
    public FlowDocument Document { get; set; }
    
    // 重写Text属性以支持富文本
}
```

## 总结

Stylet的MessageBox实现提供了：

1. **完整的MVVM支持**：通过IWindowManager接口使用
2. **高度可定制**：支持自定义视图模型和视图
3. **本地化友好**：支持按钮文本的自定义
4. **易于测试**：接口设计便于单元测试
5. **一致的用户体验**：与应用程序其他部分保持风格统一

通过本文档的学习，你应该能够熟练地在Stylet应用程序中使用MessageBox，并根据需要进行自定义扩展。