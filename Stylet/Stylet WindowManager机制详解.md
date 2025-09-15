# Stylet WindowManager机制详解

## 概述

WindowManager是Stylet框架中负责窗口管理的核心组件，它提供了一种优雅的方式来显示ViewModel对应的View作为窗口或对话框。WindowManager遵循MVVM模式，将窗口的显示逻辑从ViewModel中解耦出来，同时提供了强大的窗口生命周期管理功能。

## 核心设计理念

### 1. ViewModel驱动的窗口管理
WindowManager基于ViewModel来创建和管理窗口，开发者只需要提供ViewModel实例，框架会自动定位、创建并绑定对应的View。

### 2. 自动化的View定位和绑定
通过与ViewManager的协作，WindowManager能够自动定位ViewModel对应的View类型，实例化View，并建立View与ViewModel之间的绑定关系。

### 3. 完整的窗口生命周期管理
WindowManager提供了完整的窗口生命周期管理，包括窗口的创建、显示、状态变化、关闭等各个阶段的处理。

### 4. 灵活的所有者关系管理
支持设置窗口的所有者关系，能够智能推断合适的父窗口，确保窗口层次结构的正确性。

## 核心接口设计

### IWindowManager接口

```csharp
/// <summary>
/// 能够获取ViewModel实例，实例化其View并将其显示为对话框或窗口的管理器
/// </summary>
public interface IWindowManager
{
    /// <summary>
    /// 给定一个ViewModel，将其对应的View显示为窗口
    /// </summary>
    /// <param name="viewModel">要显示View的ViewModel</param>
    void ShowWindow(object viewModel);

    /// <summary>
    /// 给定一个ViewModel，将其对应的View显示为窗口，并设置其所有者
    /// </summary>
    /// <param name="viewModel">要显示View的ViewModel</param>
    /// <param name="ownerViewModel">应该拥有此窗口的View的ViewModel</param>
    void ShowWindow(object viewModel, IViewAware ownerViewModel);

    /// <summary>
    /// 给定一个ViewModel，将其对应的View显示为对话框
    /// </summary>
    /// <param name="viewModel">要显示View的ViewModel</param>
    /// <returns>View的DialogResult</returns>
    bool? ShowDialog(object viewModel);

    /// <summary>
    /// 给定一个ViewModel，将其对应的View显示为对话框，并设置其所有者
    /// </summary>
    /// <param name="viewModel">要显示View的ViewModel</param>
    /// <param name="ownerViewModel">应该拥有此对话框的View的ViewModel</param>
    /// <returns>View的DialogResult</returns>
    bool? ShowDialog(object viewModel, IViewAware ownerViewModel);

    /// <summary>
    /// 显示消息框
    /// </summary>
    /// <param name="messageBoxText">指定要显示的文本</param>
    /// <param name="caption">指定要显示的标题栏标题</param>
    /// <param name="buttons">指定要显示的按钮</param>
    /// <param name="icon">指定要显示的图标</param>
    /// <param name="defaultResult">指定消息框的默认结果</param>
    /// <param name="cancelResult">指定消息框的取消结果</param>
    /// <param name="buttonLabels">指定按钮标签的字典（如果需要）</param>
    /// <param name="flowDirection">要使用的FlowDirection</param>
    /// <param name="textAlignment">要使用的TextAlignment</param>
    /// <returns>用户选择的结果</returns>
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

### IWindowManagerConfig接口

```csharp
/// <summary>
/// 传递给WindowManager的配置（通常由BootstrapperBase实现）
/// </summary>
public interface IWindowManagerConfig
{
    /// <summary>
    /// 返回当前显示的窗口，如果没有或无法确定则返回null
    /// </summary>
    /// <returns>当前显示的窗口，或null</returns>
    Window GetActiveWindow();
}
```

## WindowManager核心实现

### 类结构设计

```csharp
/// <summary>
/// IWindowManager的默认实现，能够将ViewModel的View显示为对话框或窗口
/// </summary>
public class WindowManager : IWindowManager
{
    private static readonly ILogger logger = LogManager.GetLogger(typeof(WindowManager));
    private readonly IViewManager viewManager;
    private readonly Func<IMessageBoxViewModel> messageBoxViewModelFactory;
    private readonly Func<Window> getActiveWindow;

    /// <summary>
    /// 使用给定的IViewManager初始化WindowManager类的新实例
    /// </summary>
    /// <param name="viewManager">创建视图时使用的IViewManager</param>
    /// <param name="messageBoxViewModelFactory">调用时返回新IMessageBoxViewModel实例的委托</param>
    /// <param name="config">配置对象</param>
    public WindowManager(IViewManager viewManager, Func<IMessageBoxViewModel> messageBoxViewModelFactory, IWindowManagerConfig config)
    {
        this.viewManager = viewManager;
        this.messageBoxViewModelFactory = messageBoxViewModelFactory;
        this.getActiveWindow = config.GetActiveWindow;
    }
}
```

### 核心方法实现

#### 1. ShowWindow方法 - 显示窗口

```csharp
/// <summary>
/// 给定一个ViewModel，将其对应的View显示为窗口
/// </summary>
/// <param name="viewModel">要显示View的ViewModel</param>
public void ShowWindow(object viewModel)
{
    this.ShowWindow(viewModel, null);
}

/// <summary>
/// 给定一个ViewModel，将其对应的View显示为窗口，并设置其所有者
/// </summary>
/// <param name="viewModel">要显示View的ViewModel</param>
/// <param name="ownerViewModel">应该拥有此窗口的View的ViewModel</param>
public void ShowWindow(object viewModel, IViewAware ownerViewModel)
{
    this.CreateWindow(viewModel, false, ownerViewModel).Show();
}
```

#### 2. ShowDialog方法 - 显示对话框

```csharp
/// <summary>
/// 给定一个ViewModel，将其对应的View显示为对话框
/// </summary>
/// <param name="viewModel">要显示View的ViewModel</param>
/// <returns>View的DialogResult</returns>
public bool? ShowDialog(object viewModel)
{
    return this.ShowDialog(viewModel, null);
}

/// <summary>
/// 给定一个ViewModel，将其对应的View显示为对话框，并设置其所有者
/// </summary>
/// <param name="viewModel">要显示View的ViewModel</param>
/// <param name="ownerViewModel">应该拥有此对话框的View的ViewModel</param>
/// <returns>View的DialogResult</returns>
public bool? ShowDialog(object viewModel, IViewAware ownerViewModel)
{
    return this.CreateWindow(viewModel, true, ownerViewModel).ShowDialog();
}
```

#### 3. ShowMessageBox方法 - 显示消息框

```csharp
/// <summary>
/// 显示消息框
/// </summary>
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

### 核心方法 - CreateWindow

CreateWindow是WindowManager的核心方法，负责创建和配置窗口：

```csharp
/// <summary>
/// 给定一个ViewModel，创建其View，确保它是一个Window，并设置它
/// </summary>
/// <param name="viewModel">要为其创建窗口的ViewModel</param>
/// <param name="isDialog">如果窗口将用作对话框则为true</param>
/// <param name="ownerViewModel">可选地拥有应该拥有此窗口的视图的ViewModel</param>
/// <returns>已创建并设置的窗口</returns>
protected virtual Window CreateWindow(object viewModel, bool isDialog, IViewAware ownerViewModel)
{
    // 1. 创建并绑定View
    UIElement view = this.viewManager.CreateAndBindViewForModelIfNecessary(viewModel);
    
    // 2. 确保View是Window类型
    if (view is not Window window)
    {
        var e = new StyletInvalidViewTypeException(string.Format(
            "WindowManager.ShowWindow or .ShowDialog tried to show a View of type '{0}', " +
            "but that View doesn't derive from the Window class. " +
            "Make sure any Views you display using WindowManager.ShowWindow or .ShowDialog derive from Window (not UserControl, etc)",
            view == null ? "(null)" : view.GetType().Name));
        logger.Error(e);
        throw e;
    }

    // 3. 设置窗口标题
    if (viewModel is IHaveDisplayName haveDisplayName && 
        (string.IsNullOrEmpty(window.Title) || window.Title == view.GetType().Name) && 
        BindingOperations.GetBindingBase(window, Window.TitleProperty) == null)
    {
        var binding = new Binding("DisplayName") { Mode = BindingMode.TwoWay };
        window.SetBinding(Window.TitleProperty, binding);
    }

    // 4. 设置窗口所有者
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

    // 5. 记录日志
    if (isDialog)
    {
        logger.Info("Displaying ViewModel {0} with View {1} as a Dialog", viewModel, window);
    }
    else
    {
        logger.Info("Displaying ViewModel {0} with View {1} as a Window", viewModel, window);
    }

    // 6. 设置窗口启动位置
    if (window.WindowStartupLocation == WindowStartupLocation.Manual && 
        double.IsNaN(window.Top) && double.IsNaN(window.Left) &&
        BindingOperations.GetBinding(window, Window.TopProperty) == null && 
        BindingOperations.GetBinding(window, Window.LeftProperty) == null)
    {
        window.WindowStartupLocation = window.Owner == null ? 
            WindowStartupLocation.CenterScreen : 
            WindowStartupLocation.CenterOwner;
    }

    // 7. 创建窗口导航器
    new WindowConductor(window, viewModel);

    return window;
}
```

### WindowConductor - 窗口生命周期管理

WindowConductor是WindowManager的内部类，负责管理窗口与ViewModel之间的生命周期同步：

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

        // 激活ViewModel
        ScreenExtensions.TryActivate(this.viewModel);

        // 注册事件处理器
        if (this.viewModel is IScreenState viewModelAsScreenState)
        {
            window.StateChanged += this.WindowStateChanged;
            window.Closed += this.WindowClosed;
        }

        if (this.viewModel is IGuardClose)
            window.Closing += this.WindowClosing;
    }
}
```

#### 窗口状态变化处理

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

#### 窗口关闭处理

```csharp
private void WindowClosed(object sender, EventArgs e)
{
    // 取消注册事件处理器
    this.window.StateChanged -= this.WindowStateChanged;
    this.window.Closed -= this.WindowClosed;
    this.window.Closing -= this.WindowClosing;

    // 关闭ViewModel
    ScreenExtensions.TryClose(this.viewModel);
}
```

#### 窗口关闭前处理（支持异步）

```csharp
private async void WindowClosing(object sender, CancelEventArgs e)
{
    if (e.Cancel)
        return;

    logger.Info("ViewModel {0} close requested because its View was closed", this.viewModel);

    // 检查是否可以关闭
    System.Threading.Tasks.Task<bool> task = ((IGuardClose)this.viewModel).CanCloseAsync();
    if (task.IsCompleted)
    {
        // 同步完成
        if (!task.Result)
            logger.Info("Close of ViewModel {0} cancelled because CanCloseAsync returned false", this.viewModel);
        e.Cancel = !task.Result;
    }
    else
    {
        // 异步完成
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

#### ViewModel请求关闭的处理

```csharp
/// <summary>
/// 子项请求关闭
/// </summary>
/// <param name="item">要关闭的项</param>
/// <param name="dialogResult">如果是对话框，要关闭的DialogResult</param>
async void IChildDelegate.CloseItem(object item, bool? dialogResult)
{
    if (item != this.viewModel)
    {
        logger.Warn("IChildDelegate.CloseItem called with item {0} which is _not_ our ViewModel {1}", item, this.viewModel);
        return;
    }

    // 检查是否可以关闭
    if (this.viewModel is IGuardClose guardClose && !await guardClose.CanCloseAsync())
    {
        logger.Info("Close of ViewModel {0} cancelled because CanCloseAsync returned false", this.viewModel);
        return;
    }

    logger.Info("ViewModel {0} close requested with DialogResult {1} because it called RequestClose", this.viewModel, dialogResult);

    // 取消注册事件处理器
    this.window.StateChanged -= this.WindowStateChanged;
    this.window.Closed -= this.WindowClosed;
    this.window.Closing -= this.WindowClosing;

    // 设置DialogResult（必须在取消注册事件处理器之后）
    if (dialogResult != null)
        this.window.DialogResult = dialogResult;

    // 关闭ViewModel
    ScreenExtensions.TryClose(this.viewModel);

    // 关闭窗口
    this.window.Close();
}
```

## 相关接口和类型

### IViewAware接口

```csharp
/// <summary>
/// 感知自己拥有视图的接口
/// </summary>
public interface IViewAware
{
    /// <summary>
    /// 获取与此ViewModel关联的视图
    /// </summary>
    UIElement View { get; }

    /// <summary>
    /// 当应该附加视图时调用。应该设置View属性。
    /// </summary>
    /// <param name="view">要附加的视图</param>
    void AttachView(UIElement view);
}
```

### IHaveDisplayName接口

```csharp
/// <summary>
/// 具有显示名称。实际上，这绑定到窗口标题和TabControl选项卡等内容
/// </summary>
public interface IHaveDisplayName
{
    /// <summary>
    /// 获取或设置此对象的显示名称
    /// </summary>
    string DisplayName { get; set; }
}
```

### IGuardClose接口

```csharp
/// <summary>
/// 对是否应该关闭有意见的接口
/// </summary>
/// <remarks>如果实现，应在关闭对象之前调用CanCloseAsync</remarks>
public interface IGuardClose
{
    /// <summary>
    /// 当有人想要关闭此对象时调用
    /// </summary>
    /// <returns>如果对象可以关闭则为true，否则为false</returns>
    Task<bool> CanCloseAsync();
}
```

## 使用示例

### 1. 基本窗口显示

#### 定义ViewModel和View

```csharp
// ViewModel
public class UserManagementViewModel : Screen
{
    private string userName;
    
    public string UserName
    {
        get => userName;
        set => SetAndNotify(ref userName, value);
    }
    
    public UserManagementViewModel()
    {
        DisplayName = "用户管理";
    }
    
    public void SaveUser()
    {
        // 保存用户逻辑
        MessageBox.Show($"用户 {UserName} 已保存");
    }
}

// View (UserManagementView.xaml)
<Window x:Class="MyApp.Views.UserManagementView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="{Binding DisplayName}" 
        Width="400" Height="300">
    <Grid>
        <StackPanel Margin="20">
            <TextBlock Text="用户名:" />
            <TextBox Text="{Binding UserName}" Margin="0,5,0,10" />
            <Button Content="保存" Command="{s:Action SaveUser}" />
        </StackPanel>
    </Grid>
</Window>
```

#### 使用WindowManager显示窗口

```csharp
public class MainViewModel : Screen
{
    private readonly IWindowManager windowManager;
    
    public MainViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
        DisplayName = "主窗口";
    }
    
    public void ShowUserManagement()
    {
        var userViewModel = new UserManagementViewModel();
        windowManager.ShowWindow(userViewModel);
    }
    
    public void ShowUserManagementAsDialog()
    {
        var userViewModel = new UserManagementViewModel();
        bool? result = windowManager.ShowDialog(userViewModel);
        
        if (result == true)
        {
            // 对话框被确认
            MessageBox.Show("用户管理对话框已确认");
        }
    }
}
```

### 2. 对话框示例

#### 带返回值的对话框ViewModel

```csharp
public class UserInputDialogViewModel : Screen, IGuardClose
{
    private string inputText;
    private bool isValid;
    
    public string InputText
    {
        get => inputText;
        set
        {
            SetAndNotify(ref inputText, value);
            ValidateInput();
        }
    }
    
    public bool IsValid
    {
        get => isValid;
        private set => SetAndNotify(ref isValid, value);
    }
    
    public UserInputDialogViewModel()
    {
        DisplayName = "输入对话框";
    }
    
    private void ValidateInput()
    {
        IsValid = !string.IsNullOrWhiteSpace(InputText) && InputText.Length >= 3;
    }
    
    public void OK()
    {
        if (IsValid)
        {
            RequestClose(true);
        }
    }
    
    public void Cancel()
    {
        RequestClose(false);
    }
    
    public Task<bool> CanCloseAsync()
    {
        // 可以在这里添加关闭前的验证逻辑
        return Task.FromResult(true);
    }
}
```

#### 使用对话框

```csharp
public class MainViewModel : Screen
{
    private readonly IWindowManager windowManager;
    
    public MainViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
    }
    
    public void ShowInputDialog()
    {
        var dialogViewModel = new UserInputDialogViewModel();
        bool? result = windowManager.ShowDialog(dialogViewModel);
        
        if (result == true)
        {
            string userInput = dialogViewModel.InputText;
            MessageBox.Show($"用户输入: {userInput}");
        }
        else
        {
            MessageBox.Show("用户取消了输入");
        }
    }
}
```

### 3. 设置窗口所有者

```csharp
public class ParentViewModel : Screen
{
    private readonly IWindowManager windowManager;
    
    public ParentViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
        DisplayName = "父窗口";
    }
    
    public void ShowChildWindow()
    {
        var childViewModel = new ChildViewModel();
        // 设置当前ViewModel作为子窗口的所有者
        windowManager.ShowWindow(childViewModel, this);
    }
    
    public void ShowChildDialog()
    {
        var childViewModel = new ChildViewModel();
        // 显示为模态对话框，设置所有者
        bool? result = windowManager.ShowDialog(childViewModel, this);
    }
}
```

### 4. 使用MessageBox

```csharp
public class ExampleViewModel : Screen
{
    private readonly IWindowManager windowManager;
    
    public ExampleViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
    }
    
    public void ShowSimpleMessage()
    {
        windowManager.ShowMessageBox("这是一个简单的消息");
    }
    
    public void ShowConfirmDialog()
    {
        var result = windowManager.ShowMessageBox(
            "确定要删除这个项目吗？",
            "确认删除",
            MessageBoxButton.YesNo,
            MessageBoxImage.Question);
            
        if (result == MessageBoxResult.Yes)
        {
            // 执行删除操作
            MessageBox.Show("项目已删除");
        }
    }
    
    public void ShowCustomMessageBox()
    {
        var buttonLabels = new Dictionary<MessageBoxResult, string>
        {
            { MessageBoxResult.Yes, "是的" },
            { MessageBoxResult.No, "不是" },
            { MessageBoxResult.Cancel, "取消" }
        };
        
        var result = windowManager.ShowMessageBox(
            "您想要保存更改吗？",
            "保存确认",
            MessageBoxButton.YesNoCancel,
            MessageBoxImage.Warning,
            MessageBoxResult.Yes,  // 默认按钮
            MessageBoxResult.Cancel,  // 取消按钮
            buttonLabels,  // 自定义按钮文本
            FlowDirection.LeftToRight,
            TextAlignment.Left);
            
        switch (result)
        {
            case MessageBoxResult.Yes:
                // 保存并继续
                break;
            case MessageBoxResult.No:
                // 不保存但继续
                break;
            case MessageBoxResult.Cancel:
                // 取消操作
                break;
        }
    }
}
```

### 5. 高级窗口管理

#### 具有关闭守护的ViewModel

```csharp
public class DocumentViewModel : Screen, IGuardClose
{
    private bool isDirty;
    private readonly IWindowManager windowManager;
    
    public bool IsDirty
    {
        get => isDirty;
        set => SetAndNotify(ref isDirty, value);
    }
    
    public DocumentViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
        DisplayName = "文档编辑器";
    }
    
    public void ModifyDocument()
    {
        // 模拟文档修改
        IsDirty = true;
    }
    
    public void SaveDocument()
    {
        // 保存文档逻辑
        IsDirty = false;
        windowManager.ShowMessageBox("文档已保存", "保存成功", MessageBoxButton.OK, MessageBoxImage.Information);
    }
    
    public async Task<bool> CanCloseAsync()
    {
        if (IsDirty)
        {
            var result = windowManager.ShowMessageBox(
                "文档已修改但未保存，是否要保存？",
                "未保存的更改",
                MessageBoxButton.YesNoCancel,
                MessageBoxImage.Warning);
                
            switch (result)
            {
                case MessageBoxResult.Yes:
                    SaveDocument();
                    return true;
                case MessageBoxResult.No:
                    return true;
                case MessageBoxResult.Cancel:
                default:
                    return false;
            }
        }
        
        return true;
    }
}
```

#### 窗口状态管理

```csharp
public class MainWindowViewModel : Screen, IScreenState
{
    private readonly IWindowManager windowManager;
    private bool isActive;
    
    public bool IsActive
    {
        get => isActive;
        private set => SetAndNotify(ref isActive, value);
    }
    
    public MainWindowViewModel(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
        DisplayName = "主应用窗口";
    }
    
    // IScreenState接口会自动调用这些方法
    protected override void OnActivate()
    {
        base.OnActivate();
        IsActive = true;
        // 窗口被激活时的逻辑
    }
    
    protected override void OnDeactivate(bool close)
    {
        base.OnDeactivate(close);
        IsActive = false;
        // 窗口被停用时的逻辑
        
        if (close)
        {
            // 窗口即将关闭时的清理逻辑
        }
    }
}
```

## 依赖注入配置

### 使用Stylet的内置IoC容器

```csharp
public class Bootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(IStyletIoCBuilder builder)
    {
        // WindowManager会自动注册
        // 注册自定义的消息框ViewModel工厂
        builder.Bind<IMessageBoxViewModel>().To<CustomMessageBoxViewModel>();
    }
}
```

### 使用外部IoC容器（如Autofac）

```csharp
public class AutofacBootstrapper : AutofacBootstrapper<ShellViewModel>
{
    protected override void ConfigureContainer(ContainerBuilder builder)
    {
        base.ConfigureContainer(builder);
        
        // 注册WindowManager
        builder.RegisterType<WindowManager>().As<IWindowManager>().SingleInstance();
        
        // 注册ViewManager
        builder.RegisterType<ViewManager>().As<IViewManager>().SingleInstance();
        
        // 注册消息框ViewModel
        builder.RegisterType<MessageBoxViewModel>().As<IMessageBoxViewModel>();
        
        // 注册配置
        builder.Register<IWindowManagerConfig>(c => this).SingleInstance();
    }
}
```

## 与ViewManager的协作

WindowManager与ViewManager紧密协作来实现完整的View-ViewModel绑定：

### ViewManager的作用

1. **View定位**: 根据ViewModel类型定位对应的View类型
2. **View创建**: 实例化View对象
3. **View初始化**: 调用InitializeComponent等初始化方法
4. **绑定建立**: 建立View与ViewModel之间的绑定关系

### 协作流程

```csharp
// WindowManager中的核心调用
UIElement view = this.viewManager.CreateAndBindViewForModelIfNecessary(viewModel);

// ViewManager的处理流程
public virtual UIElement CreateAndBindViewForModelIfNecessary(object model)
{
    // 1. 检查ViewModel是否已经有View
    if (model is IViewAware modelAsViewAware && modelAsViewAware.View != null)
    {
        return modelAsViewAware.View;
    }

    // 2. 创建并绑定新的View
    return this.CreateAndBindViewForModel(model);
}

protected virtual UIElement CreateAndBindViewForModel(object model)
{
    // 3. 创建View
    UIElement view = this.CreateViewForModel(model);
    
    // 4. 绑定View和ViewModel
    this.BindViewToModel(view, model);
    
    return view;
}
```

## 错误处理和异常管理

### 常见异常类型

#### 1. StyletInvalidViewTypeException

```csharp
// 当试图显示的View不是Window类型时抛出
if (view is not Window window)
{
    var e = new StyletInvalidViewTypeException(
        $"WindowManager.ShowWindow or .ShowDialog tried to show a View of type '{view?.GetType().Name ?? "(null)"}', " +
        "but that View doesn't derive from the Window class. " +
        "Make sure any Views you display using WindowManager.ShowWindow or .ShowDialog derive from Window (not UserControl, etc)");
    logger.Error(e);
    throw e;
}
```

#### 2. View定位失败

```csharp
// ViewManager无法找到对应的View时
try
{
    UIElement view = this.viewManager.CreateAndBindViewForModelIfNecessary(viewModel);
}
catch (StyletViewLocationException ex)
{
    logger.Error(ex, "Failed to locate view for ViewModel {0}", viewModel.GetType().Name);
    // 处理View定位失败的情况
    throw;
}
```

### 异常处理最佳实践

```csharp
public class SafeWindowManager : IWindowManager
{
    private readonly IWindowManager innerWindowManager;
    private readonly ILogger logger;
    
    public SafeWindowManager(IWindowManager innerWindowManager, ILogger logger)
    {
        this.innerWindowManager = innerWindowManager;
        this.logger = logger;
    }
    
    public void ShowWindow(object viewModel)
    {
        try
        {
            innerWindowManager.ShowWindow(viewModel);
        }
        catch (StyletInvalidViewTypeException ex)
        {
            logger.Error(ex, "Invalid view type for ViewModel {0}", viewModel.GetType().Name);
            // 显示错误消息给用户
            MessageBox.Show($"无法显示窗口: {ex.Message}", "错误", MessageBoxButton.OK, MessageBoxImage.Error);
        }
        catch (Exception ex)
        {
            logger.Error(ex, "Unexpected error showing window for ViewModel {0}", viewModel.GetType().Name);
            throw;
        }
    }
    
    // 其他方法的类似包装...
}
```

## 性能优化和最佳实践

### 1. ViewModel生命周期管理

```csharp
public class OptimizedViewModel : Screen, IDisposable
{
    private readonly IEventAggregator eventAggregator;
    private bool disposed;
    
    public OptimizedViewModel(IEventAggregator eventAggregator)
    {
        this.eventAggregator = eventAggregator;
        this.eventAggregator.Subscribe(this);
    }
    
    protected override void OnDeactivate(bool close)
    {
        if (close)
        {
            Dispose();
        }
        base.OnDeactivate(close);
    }
    
    public void Dispose()
    {
        if (disposed) return;
        
        eventAggregator.Unsubscribe(this);
        // 释放其他资源
        disposed = true;
    }
}
```

### 2. 延迟加载和按需创建

```csharp
public class LazyWindowManager
{
    private readonly IWindowManager windowManager;
    private readonly Dictionary<Type, Lazy<object>> viewModelCache;
    
    public LazyWindowManager(IWindowManager windowManager)
    {
        this.windowManager = windowManager;
        this.viewModelCache = new Dictionary<Type, Lazy<object>>();
    }
    
    public void ShowWindow<T>() where T : class, new()
    {
        if (!viewModelCache.ContainsKey(typeof(T)))
        {
            viewModelCache[typeof(T)] = new Lazy<object>(() => new T());
        }
        
        var viewModel = viewModelCache[typeof(T)].Value;
        windowManager.ShowWindow(viewModel);
    }
}
```

### 3. 内存泄漏预防

```csharp
public class MemoryAwareViewModel : Screen
{
    private Timer timer;
    
    protected override void OnActivate()
    {
        base.OnActivate();
        timer = new Timer(OnTimerTick, null, TimeSpan.Zero, TimeSpan.FromSeconds(1));
    }
    
    protected override void OnDeactivate(bool close)
    {
        timer?.Dispose();
        timer = null;
        base.OnDeactivate(close);
    }
    
    private void OnTimerTick(object state)
    {
        // 定时器逻辑
    }
}
```

## 单元测试支持

### 测试WindowManager

```csharp
[TestFixture]
public class WindowManagerTests
{
    private Mock<IViewManager> mockViewManager;
    private Mock<IWindowManagerConfig> mockConfig;
    private Mock<IMessageBoxViewModel> mockMessageBoxViewModel;
    private WindowManager windowManager;
    
    [SetUp]
    public void SetUp()
    {
        mockViewManager = new Mock<IViewManager>();
        mockConfig = new Mock<IWindowManagerConfig>();
        mockMessageBoxViewModel = new Mock<IMessageBoxViewModel>();
        
        windowManager = new WindowManager(
            mockViewManager.Object,
            () => mockMessageBoxViewModel.Object,
            mockConfig.Object);
    }
    
    [Test]
    public void ShowWindow_ShouldCreateAndShowWindow()
    {
        // Arrange
        var viewModel = new TestViewModel();
        var mockWindow = new Mock<Window>();
        
        mockViewManager
            .Setup(vm => vm.CreateAndBindViewForModelIfNecessary(viewModel))
            .Returns(mockWindow.Object);
        
        // Act
        windowManager.ShowWindow(viewModel);
        
        // Assert
        mockWindow.Verify(w => w.Show(), Times.Once);
    }
    
    [Test]
    public void ShowDialog_ShouldReturnDialogResult()
    {
        // Arrange
        var viewModel = new TestViewModel();
        var mockWindow = new Mock<Window>();
        mockWindow.Setup(w => w.ShowDialog()).Returns(true);
        
        mockViewManager
            .Setup(vm => vm.CreateAndBindViewForModelIfNecessary(viewModel))
            .Returns(mockWindow.Object);
        
        // Act
        bool? result = windowManager.ShowDialog(viewModel);
        
        // Assert
        Assert.AreEqual(true, result);
    }
    
    [Test]
    public void ShowMessageBox_ShouldReturnUserChoice()
    {
        // Arrange
        mockMessageBoxViewModel
            .Setup(vm => vm.ClickedButton)
            .Returns(MessageBoxResult.Yes);
        
        // Act
        var result = windowManager.ShowMessageBox("Test message");
        
        // Assert
        Assert.AreEqual(MessageBoxResult.Yes, result);
        mockMessageBoxViewModel.Verify(vm => vm.Setup(
            "Test message", "", MessageBoxButton.OK, MessageBoxImage.None,
            MessageBoxResult.None, MessageBoxResult.None, null, null, null), Times.Once);
    }
}

public class TestViewModel : Screen
{
    public TestViewModel()
    {
        DisplayName = "Test";
    }
}
```

### 集成测试

```csharp
[TestFixture]
public class WindowManagerIntegrationTests
{
    private IWindowManager windowManager;
    private ViewManager viewManager;
    
    [SetUp]
    public void SetUp()
    {
        var config = new ViewManagerConfig
        {
            ViewFactory = type => Activator.CreateInstance(type),
            ViewAssemblies = new List<Assembly> { Assembly.GetExecutingAssembly() }
        };
        
        viewManager = new ViewManager(config);
        
        var windowManagerConfig = new Mock<IWindowManagerConfig>();
        windowManagerConfig.Setup(c => c.GetActiveWindow()).Returns(() => Application.Current?.MainWindow);
        
        windowManager = new WindowManager(
            viewManager,
            () => new MessageBoxViewModel(),
            windowManagerConfig.Object);
    }
    
    [Test]
    public void ShowDialog_WithRealViewModel_ShouldWork()
    {
        // 这个测试需要在STA线程中运行
        var viewModel = new TestDialogViewModel();
        
        // 在后台线程中运行以避免阻塞
        Task.Run(() =>
        {
            Thread.Sleep(1000); // 等待对话框显示
            // 模拟用户关闭对话框
            Application.Current.Dispatcher.Invoke(() =>
            {
                var windows = Application.Current.Windows.OfType<Window>();
                var dialog = windows.FirstOrDefault(w => w.DataContext == viewModel);
                dialog?.Close();
            });
        });
        
        bool? result = windowManager.ShowDialog(viewModel);
        
        // 验证结果
        Assert.IsNotNull(result);
    }
}
```

## 常见问题和解决方案

### 1. View定位失败

**问题**: ViewManager无法找到ViewModel对应的View

**解决方案**:
```csharp
// 确保View的命名约定正确
// ViewModel: UserManagementViewModel
// View: UserManagementView

// 或者使用自定义的View定位逻辑
public class CustomViewManager : ViewManager
{
    protected override Type LocateViewForModel(Type modelType)
    {
        // 自定义View定位逻辑
        string viewName = modelType.Name.Replace("ViewModel", "View");
        string viewNamespace = modelType.Namespace.Replace(".ViewModels", ".Views");
        string fullViewName = $"{viewNamespace}.{viewName}";
        
        return Type.GetType(fullViewName) ?? base.LocateViewForModel(modelType);
    }
}
```

### 2. 窗口所有者设置问题

**问题**: 对话框没有正确的父窗口

**解决方案**:
```csharp
public void ShowChildDialog()
{
    var childViewModel = new ChildViewModel();
    
    // 明确指定所有者
    windowManager.ShowDialog(childViewModel, this);
    
    // 或者确保GetActiveWindow正确实现
}
```

### 3. 内存泄漏问题

**问题**: 窗口关闭后ViewModel没有被回收

**解决方案**:
```csharp
public class ProperViewModel : Screen, IDisposable
{
    public ProperViewModel()
    {
        // 订阅事件
    }
    
    protected override void OnDeactivate(bool close)
    {
        if (close)
        {
            Dispose();
        }
        base.OnDeactivate(close);
    }
    
    public void Dispose()
    {
        // 取消订阅所有事件
        // 释放所有资源
    }
}
```

### 4. 异步关闭处理

**问题**: 异步关闭逻辑导致UI冻结

**解决方案**:
```csharp
public class AsyncCloseViewModel : Screen, IGuardClose
{
    public async Task<bool> CanCloseAsync()
    {
        // 使用ConfigureAwait(false)避免死锁
        bool canClose = await CheckCanCloseAsync().ConfigureAwait(false);
        
        // 如果需要更新UI，切换到UI线程
        if (canClose)
        {
            await Execute.OnUIThreadAsync(() =>
            {
                // UI更新逻辑
            });
        }
        
        return canClose;
    }
    
    private async Task<bool> CheckCanCloseAsync()
    {
        // 实际的异步检查逻辑
        await Task.Delay(100);
        return true;
    }
}
```

## 总结

Stylet的WindowManager提供了一个强大且灵活的窗口管理解决方案，它的主要优势包括：

1. **MVVM友好**: 完全基于ViewModel的窗口管理，符合MVVM模式
2. **自动化程度高**: 自动处理View定位、创建、绑定等复杂过程
3. **生命周期管理**: 提供完整的窗口生命周期管理，包括状态同步
4. **异步支持**: 支持异步的关闭处理，避免UI阻塞
5. **灵活配置**: 支持自定义View定位、消息框等各种扩展点
6. **内存安全**: 通过正确的生命周期管理避免内存泄漏
7. **测试友好**: 设计上支持单元测试和集成测试

通过合理使用WindowManager，可以构建出结构清晰、易于维护且高性能的WPF应用程序界面。
