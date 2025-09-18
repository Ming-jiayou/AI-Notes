# 27-StylingDialog 学习笔记

## 🎯 核心概念：对话框样式定制

StylingDialog项目是Prism框架中用于**定制对话框外观和行为**的示例，它展示了如何通过`prism:Dialog.WindowStyle`附加属性来**控制对话框窗口的样式**，包括位置、大小、可调整性等。

## 🔑 关键技术点

### 1. 对话框窗口样式附加属性
```xml
<prism:Dialog.WindowStyle>
    <Style TargetType="Window">
        <!-- 窗口样式设置 -->
    </Style>
</prism:Dialog.WindowStyle>
```

### 2. 核心样式属性
- `prism:Dialog.WindowStartupLocation` - 窗口启动位置
- `ResizeMode` - 窗口调整大小模式
- `ShowInTaskbar` - 是否在任务栏显示
- `SizeToContent` - 窗口大小自适应内容

## 🏗️ 项目架构分析

### 1. 主窗口结构（与26-UsingDialogService相同）
```xml
<Window x:Class="StylingDialog.Views.MainWindow"
        xmlns:prism="http://prismlibrary.com/"
        prism:ViewModelLocator.AutoWireViewModel="True">
    <!-- 显示对话框按钮 -->
    <Button Command="{Binding Path=ShowDialogCommand}" Content="Show Dialog" />
</Window>
```

### 2. 对话框视图（新增样式定制）
```xml
<UserControl x:Class="StylingDialog.Views.NotificationDialog"
             xmlns:prism="http://prismlibrary.com/"             
             prism:ViewModelLocator.AutoWireViewModel="True"
             Width="300" Height="150">
    <!-- 对话框窗口样式定制 -->
    <prism:Dialog.WindowStyle>
        <Style TargetType="Window">
            <Setter Property="prism:Dialog.WindowStartupLocation" Value="CenterScreen" />
            <Setter Property="ResizeMode" Value="NoResize"/>
            <Setter Property="ShowInTaskbar" Value="False"/>
            <Setter Property="SizeToContent" Value="WidthAndHeight"/>
        </Style>
    </prism:Dialog.WindowStyle>
    
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

## 🔄 对话框样式机制详解

### 1. 应用程序配置（与26-UsingDialogService相同）
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

### 2. 主窗口ViewModel（与26-UsingDialogService相同）
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

### 3. 对话框ViewModel（与26-UsingDialogService相同）
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

## 🎯 样式定制详解

### 1. 窗口启动位置
```xml
<Setter Property="prism:Dialog.WindowStartupLocation" Value="CenterScreen" />
```
- **CenterScreen**：窗口在屏幕中央启动
- **CenterOwner**：窗口在所有者窗口中央启动
- **Manual**：手动指定位置

### 2. 窗口调整大小模式
```xml
<Setter Property="ResizeMode" Value="NoResize"/>
```
- **NoResize**：不允许调整窗口大小
- **CanResize**：允许调整窗口大小
- **CanMinimize**：只允许最小化
- **CanResizeWithGrip**：允许调整大小并显示调整手柄

### 3. 任务栏显示
```xml
<Setter Property="ShowInTaskbar" Value="False"/>
```
- **True**：在任务栏显示窗口
- **False**：不在任务栏显示窗口

### 4. 窗口大小自适应
```xml
<Setter Property="SizeToContent" Value="WidthAndHeight"/>
```
- **WidthAndHeight**：窗口大小自适应内容的宽度和高度
- **Width**：窗口宽度自适应内容
- **Height**：窗口高度自适应内容
- **Manual**：手动指定窗口大小

## 🎯 企业级应用场景

### 1. 专业UI设计
- **品牌一致性**：确保对话框符合企业品牌风格
- **用户体验**：提供一致的用户界面体验
- **视觉效果**：通过样式定制提升视觉效果

### 2. 特定业务需求
- **信息展示**：定制信息展示对话框的外观
- **操作确认**：定制操作确认对话框的行为
- **数据输入**：定制数据输入对话框的布局

### 3. 多平台适配
- **屏幕适配**：根据不同屏幕尺寸调整对话框
- **分辨率适配**：根据分辨率调整对话框大小
- **设备适配**：针对不同设备定制对话框样式

### 4. 辅助功能
- **可访问性**：为视觉障碍用户提供高对比度样式
- **大字体支持**：支持大字体用户的界面需求
- **键盘导航**：优化键盘导航的对话框行为

## 💡 最佳实践

### 1. 样式统一管理
```xml
<!-- 在资源字典中定义通用样式 -->
<prism:Dialog.WindowStyle>
    <Style TargetType="Window" BasedOn="{StaticResource CommonDialogStyle}">
        <Setter Property="prism:Dialog.WindowStartupLocation" Value="CenterScreen" />
        <Setter Property="ResizeMode" Value="NoResize"/>
    </Style>
</prism:Dialog.WindowStyle>
```

### 2. 响应式设计
```xml
<prism:Dialog.WindowStyle>
    <Style TargetType="Window">
        <Setter Property="prism:Dialog.WindowStartupLocation" Value="CenterScreen" />
        <Setter Property="ResizeMode" Value="CanResize"/>
        <Setter Property="SizeToContent" Value="Manual"/>
        <Setter Property="MinWidth" Value="300"/>
        <Setter Property="MinHeight" Value="150"/>
    </Style>
</prism:Dialog.WindowStyle>
```

### 3. 平台特定样式
```xml
<prism:Dialog.WindowStyle>
    <Style TargetType="Window">
        <!-- 通用设置 -->
        <Setter Property="prism:Dialog.WindowStartupLocation" Value="CenterScreen" />
        
        <!-- Windows特定设置 -->
        <Style.Triggers>
            <DataTrigger Binding="{Binding Source={x:Static SystemParameters.HighContrast}}" Value="True">
                <Setter Property="Background" Value="Black"/>
                <Setter Property="Foreground" Value="White"/>
            </DataTrigger>
        </Style.Triggers>
    </Style>
</prism:Dialog.WindowStyle>
```

## 🚀 技术优势

1. **声明式样式**：通过XAML声明式定义样式
2. **灵活定制**：支持窗口位置、大小、行为的全面定制
3. **与DialogService集成**：无缝集成Prism的对话框服务
4. **MVVM友好**：不破坏MVVM架构模式
5. **可重用性**：样式可重用和继承

## 📊 对比其他样式方式

| 特性 | prism:Dialog.WindowStyle | 直接Window样式 | 自定义Window类 |
|------|--------------------------|----------------|----------------|
| **声明式** | ✅ XAML声明式 | ✅ XAML声明式 | ❌ 需要代码 |
| **灵活性** | ✅ 高度灵活 | ✅ 灵活 | ⚠️ 中等灵活 |
| **集成性** | ✅ 与DialogService集成 | ⚠️ 需要手动集成 | ⚠️ 需要手动集成 |
| **MVVM友好** | ✅ 完全MVVM友好 | ✅ MVVM友好 | ⚠️ 部分MVVM友好 |
| **重用性** | ✅ 样式可重用 | ✅ 样式可重用 | ❌ 需要继承 |

## 🎯 总结

**StylingDialog**项目通过`prism:Dialog.WindowStyle`附加属性，展示了**Prism对话框样式定制**的完整机制。它在**26-UsingDialogService**的基础上，增加了**对话框窗口外观和行为的定制能力**，为构建**专业级用户界面**提供了**强大的样式支持**。

**核心价值**：
1. **专业UI设计**：通过样式定制实现专业级用户界面
2. **用户体验提升**：提供一致且美观的对话框体验
3. **灵活配置**：支持窗口位置、大小、行为的全面定制
4. **架构友好**：不破坏MVVM架构，与DialogService无缝集成

**与26-UsingDialogService的主要区别**：
- **功能扩展**：增加了对话框窗口样式定制功能
- **用户体验**：提供更好的视觉效果和交互体验
- **专业性**：更适合企业级应用的专业UI需求