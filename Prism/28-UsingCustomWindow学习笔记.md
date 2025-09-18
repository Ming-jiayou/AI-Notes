# 28-UsingCustomWindow 学习笔记

## 🎯 核心概念：自定义对话框窗口

UsingCustomWindow项目是Prism框架中用于**使用自定义窗口类**作为对话框容器的示例，它展示了如何通过实现`IDialogWindow`接口来**创建完全自定义的对话框窗口**，从而获得**最大的样式和行为控制能力**。

## 🔑 关键技术点

### 1. 自定义窗口接口
```csharp
public partial class MyCustomWindow : Window, IDialogWindow
```

### 2. 核心接口成员
- `IDialogResult Result { get; set; }` - 对话框结果属性
- 继承自Window类的所有功能

### 3. 注册方法
```csharp
containerRegistry.RegisterDialogWindow<MyCustomWindow>();
```

## 🏗️ 项目架构分析

### 1. 主窗口结构（与之前项目相同）
```xml
<Window x:Class="UsingCustomWindow.Views.MainWindow"
        xmlns:prism="http://prismlibrary.com/"
        prism:ViewModelLocator.AutoWireViewModel="True">
    <!-- 显示对话框按钮 -->
    <Button Command="{Binding Path=ShowDialogCommand}" Content="Show Dialog" />
</Window>
```

### 2. 自定义窗口类
```csharp
public partial class MyCustomWindow : Window, IDialogWindow
{
    // 对话框结果属性
    public IDialogResult Result { get; set; }

    public MyCustomWindow()
    {
        InitializeComponent();
    }
}
```

### 3. 自定义窗口XAML
```xml
<Window x:Class="UsingCustomWindow.Views.MyCustomWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MyCustomWindow" Background="Azure" SizeToContent="WidthAndHeight">
</Window>
```

### 4. 对话框视图（与之前项目相同）
```xml
<UserControl x:Class="UsingCustomWindow.Views.NotificationDialog"
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

## 🔄 自定义窗口机制详解

### 1. 应用程序配置
```csharp
public partial class App
{
    protected override void RegisterTypes(IContainerRegistry containerRegistry)
    {
        // 注册自定义窗口类
        containerRegistry.RegisterDialogWindow<MyCustomWindow>();
        
        // 注册对话框视图和ViewModel
        containerRegistry.RegisterDialog<NotificationDialog, NotificationDialogViewModel>();
    }
}
```

### 2. 主窗口ViewModel（与之前项目相同）
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

### 3. 自定义窗口实现
```csharp
public partial class MyCustomWindow : Window, IDialogWindow
{
    // 实现IDialogWindow接口的Result属性
    public IDialogResult Result { get; set; }

    public MyCustomWindow()
    {
        InitializeComponent();
    }
}
```

### 4. 自定义窗口XAML
```xml
<Window x:Class="UsingCustomWindow.Views.MyCustomWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MyCustomWindow" 
        Background="Azure" 
        SizeToContent="WidthAndHeight">
    <!-- 可以在这里添加自定义的窗口内容 -->
</Window>
```

## 🎯 自定义窗口工作流程

### 1. 窗口注册
```
App.xaml.cs → RegisterDialogWindow<MyCustomWindow>()
    ↓
Prism容器注册自定义窗口类作为默认对话框窗口
```

### 2. 对话框显示
```
MainWindowViewModel → ShowDialog() → _dialogService.ShowDialog()
    ↓
Prism使用注册的MyCustomWindow作为对话框容器
```

### 3. 窗口实例化
```
MyCustomWindow构造函数 → InitializeComponent()
    ↓
创建自定义窗口实例并显示对话框内容
```

### 4. 结果处理
```
CloseDialog() → Result属性设置 → 回调函数处理结果
```

## 🎯 企业级应用场景

### 1. 品牌化对话框
- **企业品牌**：使用企业特定颜色和样式
- **视觉一致性**：确保对话框与主应用风格一致
- **专业外观**：创建专业级用户界面

### 2. 特殊功能窗口
- **无边框窗口**：创建现代化无边框对话框
- **透明窗口**：实现半透明或全透明效果
- **自定义控件**：在窗口中集成自定义控件

### 3. 多语言支持
- **本地化窗口**：根据不同语言调整窗口布局
- **字体适配**：针对不同语言的字体需求调整窗口
- **文化适配**：根据文化习惯调整窗口行为

### 4. 辅助功能
- **高对比度**：为视觉障碍用户提供高对比度窗口
- **屏幕阅读器**：优化屏幕阅读器支持
- **键盘导航**：增强键盘导航功能

## 💡 最佳实践

### 1. 窗口接口实现
```csharp
public partial class MyCustomWindow : Window, IDialogWindow
{
    // 必须实现Result属性
    public IDialogResult Result { get; set; }

    public MyCustomWindow()
    {
        InitializeComponent();
        
        // 可以在这里添加自定义初始化代码
        SetupCustomBehavior();
    }
    
    private void SetupCustomBehavior()
    {
        // 自定义窗口行为
        this.WindowStartupLocation = WindowStartupLocation.CenterScreen;
        this.ResizeMode = ResizeMode.NoResize;
    }
}
```

### 2. XAML样式定制
```xml
<Window x:Class="UsingCustomWindow.Views.MyCustomWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="MyCustomWindow" 
        Background="{DynamicResource CustomBackgroundBrush}" 
        SizeToContent="WidthAndHeight"
        WindowStartupLocation="CenterScreen"
        ResizeMode="NoResize"
        ShowInTaskbar="False">
    <!-- 可以添加自定义窗口装饰 -->
    <WindowChrome.WindowChrome>
        <WindowChrome CaptionHeight="30" 
                      CornerRadius="10" 
                      GlassFrameThickness="0" 
                      UseAeroCaptionButtons="False" />
    </WindowChrome.WindowChrome>
    
    <!-- 自定义标题栏 -->
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="30"/>
            <RowDefinition/>
        </Grid.RowDefinitions>
        
        <Border Grid.Row="0" Background="DarkBlue">
            <TextBlock Text="{Binding Title}" 
                       Foreground="White" 
                       VerticalAlignment="Center" 
                       Margin="10,0"/>
        </Border>
        
        <ContentPresenter Grid.Row="1" x:Name="DialogContent"/>
    </Grid>
</Window>
```

### 3. 多种窗口类型
```csharp
// 注册不同类型的自定义窗口
protected override void RegisterTypes(IContainerRegistry containerRegistry)
{
    // 注册默认自定义窗口
    containerRegistry.RegisterDialogWindow<MyCustomWindow>();
    
    // 注册特定用途的自定义窗口
    containerRegistry.RegisterDialogWindow<WarningDialogWindow>("WarningWindow");
    containerRegistry.RegisterDialogWindow<InformationDialogWindow>("InfoWindow");
}

// 使用特定窗口类型
_dialogService.ShowDialog("NotificationDialog", parameters, callback, "WarningWindow");
```

## 🚀 技术优势

1. **完全控制**：对窗口的外观和行为有完全控制权
2. **灵活性**：可以实现任何复杂的窗口样式和行为
3. **与DialogService集成**：无缝集成Prism的对话框服务
4. **MVVM友好**：不破坏MVVM架构模式
5. **可扩展性**：支持多种自定义窗口类型

## 📊 对比其他窗口方式

| 特性 | 自定义窗口 | prism:Dialog.WindowStyle | 默认窗口 |
|------|------------|--------------------------|----------|
| **控制级别** | ✅ 完全控制 | ⚠️ 部分控制 | ❌ 无控制 |
| **灵活性** | ✅ 极高灵活性 | ⚠️ 中等灵活性 | ❌ 固定样式 |
| **实现复杂度** | ⚠️ 需要代码实现 | ✅ XAML声明式 | ✅ 无需实现 |
| **MVVM友好** | ✅ 完全MVVM友好 | ✅ MVVM友好 | ✅ MVVM友好 |
| **重用性** | ✅ 高度可重用 | ✅ 可重用 | ✅ 可重用 |

## 🎯 总结

**UsingCustomWindow**项目通过实现`IDialogWindow`接口和`RegisterDialogWindow`方法，展示了**Prism自定义对话框窗口**的完整机制。它在**27-StylingDialog**的基础上，提供了**更高级别的窗口定制能力**，允许开发者**完全控制对话框窗口的外观和行为**，为构建**专业级用户界面**提供了**最大的灵活性**。

**核心价值**：
1. **完全控制**：对窗口的外观和行为有完全控制权
2. **专业UI设计**：可以实现任何复杂的窗口样式
3. **灵活扩展**：支持多种自定义窗口类型
4. **架构友好**：不破坏MVVM架构，与DialogService无缝集成

**与之前项目的主要区别**：
- **26-UsingDialogService**：使用默认窗口样式
- **27-StylingDialog**：通过XAML附加属性定制窗口样式
- **28-UsingCustomWindow**：通过自定义窗口类实现完全控制

**适用场景**：
- 需要特殊窗口样式的企业级应用
- 需要自定义窗口行为的复杂应用
- 需要多种不同类型对话框窗口的应用