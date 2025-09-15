# Stylet Action机制详解

## 概述

Stylet框架提供了一个强大的Action机制，允许开发者在XAML中直接绑定ViewModel的方法到Button的Command属性或其他控件的事件。当在XAML中使用`<Button Command="{s:Action SayHello}">Say Hello</Button>`时，Stylet会自动创建一个ICommand实例，该实例能够调用对应ViewModel中的SayHello方法。

## 核心机制原理

### 1. 标记扩展（Markup Extension）机制

#### ActionExtension的定义

```csharp
// 文件：Stylet\Xaml\ActionExtension.cs
/// <summary>
/// MarkupExtension用于将Command和Event绑定到View.ActionTarget上的方法
/// </summary>
public class ActionExtension : MarkupExtension
{
    /// <summary>
    /// 要调用的方法名称
    /// </summary>
    [ConstructorArgument("method")]
    public string Method { get; set; }

    /// <summary>
    /// 覆盖View.ActionTarget设置的目标对象
    /// </summary>
    public object Target { get; set; }

    /// <summary>
    /// 当View.ActionTarget为null时的行为
    /// </summary>
    public ActionUnavailableBehaviour NullTarget { get; set; }

    /// <summary>
    /// 当action在View.ActionTarget上不存在时的行为
    /// </summary>
    public ActionUnavailableBehaviour ActionNotFound { get; set; }
}
```

**关键点：**
- `ActionExtension`继承自`MarkupExtension`，这使得它可以在XAML中作为标记扩展使用
- `{s:Action SayHello}`会被解析为`new ActionExtension("SayHello")`

### 2. ProvideValue方法 - 核心分发逻辑

```csharp
// 文件：Stylet\Xaml\ActionExtension.cs
public override object ProvideValue(IServiceProvider serviceProvider)
{
    if (this.Method == null)
        throw new InvalidOperationException("Method has not been set");

    var valueService = (IProvideValueTarget)serviceProvider.GetService(typeof(IProvideValueTarget));

    // 根据目标对象类型决定创建什么类型的Action
    return valueService.TargetObject switch
    {
        // 1. 如果目标是DependencyObject（如Button），处理为依赖对象
        DependencyObject targetObject => this.HandleDependencyObject(serviceProvider, valueService, targetObject),
        
        // 2. 如果目标是CommandBinding，创建事件Action
        CommandBinding commandBinding => this.CreateEventAction(serviceProvider, null, 
            ((EventInfo)valueService.TargetProperty).EventHandlerType, isCommandBinding: true),
        
        // 3. 其他情况抛出异常
        _ => throw new InvalidOperationException("Can only use ActionExtension on a DependencyObject")
    };
}
```

### 3. 依赖对象处理逻辑

```csharp
// 文件：Stylet\Xaml\ActionExtension.cs
private object HandleDependencyObject(IServiceProvider serviceProvider, IProvideValueTarget valueService, DependencyObject targetObject)
{
    switch (valueService.TargetProperty)
    {
        // 1. 如果目标属性是ICommand类型（如Button.Command）
        case DependencyProperty dependencyProperty when dependencyProperty.PropertyType == typeof(ICommand):
            return this.CreateCommandAction(serviceProvider, targetObject);
            
        // 2. 如果目标属性是事件（如Button.Click）
        case EventInfo eventInfo:
            return this.CreateEventAction(serviceProvider, targetObject, eventInfo.EventHandlerType);
            
        // 3. 如果是附加事件的方法
        case MethodInfo methodInfo:
            ParameterInfo[] parameters = methodInfo.GetParameters();
            if (parameters.Length == 2 && typeof(Delegate).IsAssignableFrom(parameters[1].ParameterType))
            {
                return this.CreateEventAction(serviceProvider, targetObject, parameters[1].ParameterType);
            }
            throw new ArgumentException("Action用于不符合正常模式的附加事件");
            
        default:
            throw new ArgumentException("ActionExtension只能用于Command属性或事件处理程序");
    }
}
```

### 4. CommandAction的创建和工作机制

#### CommandAction的创建

```csharp
// 文件：Stylet\Xaml\ActionExtension.cs
private ICommand CreateCommandAction(IServiceProvider serviceProvider, DependencyObject targetObject)
{
    if (this.Target == null)
    {
        // 获取根对象（通常是整个Window或UserControl）
        var rootObjectProvider = (IRootObjectProvider)serviceProvider.GetService(typeof(IRootObjectProvider));
        var rootObject = rootObjectProvider?.RootObject as DependencyObject;
        
        // 创建CommandAction，绑定到View.ActionTarget
        return new CommandAction(targetObject, rootObject, this.Method, 
            this.commandNullTargetBehaviour, this.commandActionNotFoundBehaviour);
    }
    else
    {
        // 使用显式指定的目标对象
        return new CommandAction(this.Target, this.Method, 
            this.commandNullTargetBehaviour, this.commandActionNotFoundBehaviour);
    }
}
```

#### CommandAction的结构

```csharp
// 文件：Stylet\Xaml\CommandAction.cs
/// <summary>
/// 由ActionExtension返回的ICommand，用于将按钮等绑定到ViewModel上的方法
/// 如果方法有参数，CommandParameter会被传递
/// </summary>
/// <remarks>
/// 监视当前的View.ActionTarget，查找具有给定名称的方法，在ICommand被调用时调用它
/// 如果存在名为Can{methodName}的bool属性，它将被观察并用于启用/禁用ICommand
/// </remarks>
public class CommandAction : ActionBase, ICommand
{
    /// <summary>
    /// 生成的访问器，用于获取guard属性的值，如果没有则为null
    /// </summary>
    private Func<bool> guardPropertyGetter;

    // ICommand事件
    public event EventHandler CanExecuteChanged;
}
```

### 5. ActionBase - 核心基类逻辑

#### Target绑定机制

```csharp
// 文件：Stylet\Xaml\ActionBase.cs
public ActionBase(DependencyObject subject, DependencyObject backupSubject, string methodName, 
    ActionUnavailableBehaviour targetNullBehaviour, ActionUnavailableBehaviour actionNonExistentBehaviour, ILogger logger)
{
    this.Subject = subject;
    this.MethodName = methodName;
    
    // 设置绑定到View.ActionTarget
    var actionTargetBinding = new Binding()
    {
        Path = new PropertyPath(View.ActionTargetProperty),
        Mode = BindingMode.OneWay,
        Source = this.Subject,  // 通常是Button控件
    };

    if (backupSubject == null)
    {
        // 简单绑定：Button.View.ActionTarget -> CommandAction.Target
        BindingOperations.SetBinding(this, targetProperty, actionTargetBinding);
    }
    else
    {
        // 多重绑定：优先使用subject的ActionTarget，如果没有则使用backupSubject的
        var multiBinding = new MultiBinding();
        multiBinding.Converter = new MultiBindingToActionTargetConverter();
        multiBinding.Bindings.Add(actionTargetBinding);
        multiBinding.Bindings.Add(new Binding()
        {
            Path = new PropertyPath(View.ActionTargetProperty),
            Mode = BindingMode.OneWay,
            Source = backupSubject,
        });
        BindingOperations.SetBinding(this, targetProperty, multiBinding);
    }
}
```

#### Target变化时的处理

```csharp
// 文件：Stylet\Xaml\ActionBase.cs
private void UpdateActionTarget(object oldTarget, object newTarget)
{
    MethodInfo targetMethodInfo = null;

    // 忽略初始值
    if (newTarget == View.InitialActionTarget)
        return;

    if (newTarget == null)
    {
        // 处理null目标的行为
        if (this.TargetNullBehaviour == ActionUnavailableBehaviour.Throw)
        {
            throw new ActionTargetNullException($"ActionTarget为null (方法名是 {this.MethodName})");
        }
    }
    else
    {
        // 查找目标方法
        Type newTargetType = newTarget is Type type ? type : newTarget.GetType();
        BindingFlags bindingFlags = newTarget is Type ? 
            BindingFlags.Public | BindingFlags.Static : 
            BindingFlags.Public | BindingFlags.Instance;

        try
        {
            targetMethodInfo = newTargetType.GetMethod(this.MethodName, bindingFlags);
            
            if (targetMethodInfo == null)
                this.logger.Warn("无法找到方法 {0} 在 {1} 上", this.MethodName, newTargetType.Name);
            else
                this.AssertTargetMethodInfo(targetMethodInfo, newTargetType);
        }
        catch (AmbiguousMatchException e)
        {
            throw new AmbiguousMatchException($"在 {newTargetType.Name} 上找到多个匹配的 {this.MethodName} 方法", e);
        }
    }

    this.TargetMethodInfo = targetMethodInfo;
    this.OnTargetChanged(oldTarget, newTarget);
}
```

### 6. Command的执行逻辑

#### CanExecute方法

```csharp
// 文件：Stylet\Xaml\CommandAction.cs
public bool CanExecute(object parameter)
{
    this.AssertTargetSet();

    // 1. 如果目标为null，根据行为决定
    if (this.Target == null)
        return this.TargetNullBehaviour == ActionUnavailableBehaviour.Enable;

    // 2. 如果方法不存在，根据行为决定
    if (this.TargetMethodInfo == null)
        return this.ActionNonExistentBehaviour == ActionUnavailableBehaviour.Enable;

    // 3. 如果有Can{MethodName}属性，使用其值
    if (this.guardPropertyGetter != null)
    {
        try
        {
            return this.guardPropertyGetter();
        }
        catch (Exception e)
        {
            this.logger.Error(e, "执行guard {0} 时出错", this.guardName);
            return false;
        }
    }

    // 4. 默认返回true
    return true;
}
```

#### Execute方法

```csharp
// 文件：Stylet\Xaml\CommandAction.cs
public void Execute(object parameter)
{
    this.AssertTargetSet();

    // 如果目标或方法信息为null，直接返回
    if (this.Target == null || this.TargetMethodInfo == null)
        return;

    // 根据方法参数决定传递的参数
    object[] parameters = this.TargetMethodInfo.GetParameters().Length == 1 ? new[] { parameter } : null;
    
    // 调用目标方法
    this.InvokeTargetMethod(parameters);
}
```

#### 方法调用的具体实现

```csharp
// 文件：Stylet\Xaml\ActionBase.cs
private protected void InvokeTargetMethod(object[] parameters)
{
    this.logger.Info("调用方法 {0} 在 {1} 上，参数: ({2})", 
        this.MethodName, this.TargetName(), 
        parameters == null ? "无" : string.Join(", ", parameters));

    try
    {
        // 静态方法的target为null，实例方法使用实际target
        object target = this.TargetMethodInfo.IsStatic ? null : this.Target;
        
        // 通过反射调用方法
        object result = this.TargetMethodInfo.Invoke(target, parameters);
        
        // 如果返回Task，等待其完成
        if (result is Task task)
        {
            AwaitTask(task);
        }
    }
    catch (TargetInvocationException e)
    {
        // 解包异常，显示真实的异常信息
        this.logger.Error(e.InnerException, "调用方法失败 {0} 在 {1} 上", this.MethodName, this.TargetName());
        ExceptionDispatchInfo.Capture(e.InnerException).Throw();
    }

    // 异步等待Task完成
    async void AwaitTask(Task t) => await t;
}
```

### 7. Guard属性机制（Can{MethodName}）

#### Guard属性的发现和绑定

```csharp
// 文件：Stylet\Xaml\CommandAction.cs
private protected override void OnTargetChanged(object oldTarget, object newTarget)
{
    // 取消旧目标的PropertyChanged事件订阅
    if (oldTarget is INotifyPropertyChanged oldTargetINPC)
        oldTargetINPC.PropertyChanged -= this.PropertyChangedHandler;

    // 为新目标设置guard属性
    if (newTarget != null && this.TargetMethodInfo != null)
    {
        this.SetupGuardProperty();
        
        // 订阅PropertyChanged事件
        if (newTarget is INotifyPropertyChanged newTargetINPC)
            newTargetINPC.PropertyChanged += this.PropertyChangedHandler;
    }

    this.UpdateCanExecute();
}

private void SetupGuardProperty()
{
    string guardName = "Can" + this.MethodName;  // 例如：CanSayHello
    
    Type targetType = this.Target is Type type ? type : this.Target.GetType();
    BindingFlags bindingFlags = this.Target is Type ? 
        BindingFlags.Public | BindingFlags.Static : 
        BindingFlags.Public | BindingFlags.Instance;

    PropertyInfo guardPropertyInfo = targetType.GetProperty(guardName, bindingFlags);
    if (guardPropertyInfo != null && guardPropertyInfo.PropertyType == typeof(bool) && guardPropertyInfo.CanRead)
    {
        // 生成高效的属性访问器
        this.guardPropertyGetter = (Func<bool>)Delegate.CreateDelegate(typeof(Func<bool>), 
            this.Target is Type ? null : this.Target, guardPropertyInfo.GetGetMethod());
    }
    else
    {
        this.guardPropertyGetter = null;
    }
}
```

#### 属性变化时的CanExecute更新

```csharp
// 文件：Stylet\Xaml\CommandAction.cs
private void PropertyChangedHandler(object sender, PropertyChangedEventArgs e)
{
    // 只有相关属性变化时才更新CanExecute
    if (e.PropertyName == this.guardName || string.IsNullOrEmpty(e.PropertyName))
    {
        this.UpdateCanExecute();
    }
}

private void UpdateCanExecute()
{
    EventHandler handler = this.CanExecuteChanged;
    if (handler != null)
    {
        // 确保在UI线程上触发CanExecuteChanged事件
        Stylet.Execute.OnUIThread(() => handler(this, EventArgs.Empty));
    }
}
```

## 完整工作流程示例

让我们以`<Button Command="{s:Action SayHello}">Say Hello</Button>`为例，追踪完整的工作流程：

### 步骤1：XAML解析
```xml
<Button Command="{s:Action SayHello}">Say Hello</Button>
```

1. XAML解析器遇到`{s:Action SayHello}`
2. 创建`ActionExtension`实例：`new ActionExtension("SayHello")`
3. 调用`ActionExtension.ProvideValue(serviceProvider)`

### 步骤2：ActionExtension.ProvideValue执行
```csharp
public override object ProvideValue(IServiceProvider serviceProvider)
{
    // Method = "SayHello"
    var valueService = (IProvideValueTarget)serviceProvider.GetService(typeof(IProvideValueTarget));
    
    // valueService.TargetObject = Button实例
    // valueService.TargetProperty = Button.CommandProperty (DependencyProperty)
    
    return this.HandleDependencyObject(serviceProvider, valueService, targetObject);
}
```

### 步骤3：处理依赖对象
```csharp
private object HandleDependencyObject(IServiceProvider serviceProvider, IProvideValueTarget valueService, DependencyObject targetObject)
{
    // valueService.TargetProperty = CommandProperty
    // CommandProperty.PropertyType = typeof(ICommand)
    
    return this.CreateCommandAction(serviceProvider, targetObject);
}
```

### 步骤4：创建CommandAction
```csharp
private ICommand CreateCommandAction(IServiceProvider serviceProvider, DependencyObject targetObject)
{
    // targetObject = Button实例
    // this.Target = null (未显式设置)
    
    var rootObjectProvider = (IRootObjectProvider)serviceProvider.GetService(typeof(IRootObjectProvider));
    var rootObject = rootObjectProvider?.RootObject as DependencyObject; // ShellView实例
    
    // 创建CommandAction实例
    return new CommandAction(targetObject, rootObject, "SayHello", 
        commandNullTargetBehaviour, commandActionNotFoundBehaviour);
}
```

### 步骤5：CommandAction初始化
```csharp
public CommandAction(DependencyObject subject, DependencyObject backupSubject, string methodName, ...)
    : base(subject, backupSubject, methodName, ...)
{
    // subject = Button实例
    // backupSubject = ShellView实例  
    // methodName = "SayHello"
    
    // 在ActionBase构造函数中设置绑定
    // Button.View.ActionTarget -> CommandAction.Target
}
```

### 步骤6：绑定建立和Target设置
```csharp
// 当ViewManager.BindViewToModel执行时：
View.SetActionTarget(view, viewModel);  // view = ShellView, viewModel = ShellViewModel

// 这会触发Button继承ActionTarget属性
// Button.View.ActionTarget = ShellViewModel实例

// 触发CommandAction.UpdateActionTarget
private void UpdateActionTarget(object oldTarget, object newTarget)
{
    // newTarget = ShellViewModel实例
    Type targetType = newTarget.GetType(); // typeof(ShellViewModel)
    
    // 查找SayHello方法
    MethodInfo targetMethodInfo = targetType.GetMethod("SayHello", BindingFlags.Public | BindingFlags.Instance);
    
    this.TargetMethodInfo = targetMethodInfo; // ShellViewModel.SayHello方法
    this.OnTargetChanged(oldTarget, newTarget);
}
```

### 步骤7：Guard属性设置
```csharp
private protected override void OnTargetChanged(object oldTarget, object newTarget)
{
    // 查找CanSayHello属性
    PropertyInfo guardProperty = typeof(ShellViewModel).GetProperty("CanSayHello");
    
    // 创建属性访问器
    this.guardPropertyGetter = () => ((ShellViewModel)this.Target).CanSayHello;
    
    // 订阅PropertyChanged事件
    ((ShellViewModel)newTarget).PropertyChanged += this.PropertyChangedHandler;
}
```

### 步骤8：Button.Command设置完成
```csharp
// Button.Command现在指向CommandAction实例
// CommandAction.Target = ShellViewModel实例
// CommandAction.TargetMethodInfo = ShellViewModel.SayHello方法
// CommandAction.guardPropertyGetter = () => viewModel.CanSayHello
```

### 步骤9：用户点击按钮
```csharp
// WPF命令系统调用CommandAction.Execute
public void Execute(object parameter)
{
    // this.Target = ShellViewModel实例
    // this.TargetMethodInfo = SayHello方法
    // parameter = null (Button没有设置CommandParameter)
    
    object[] parameters = null; // SayHello方法无参数
    this.InvokeTargetMethod(parameters);
}
```

### 步骤10：方法调用执行
```csharp
private protected void InvokeTargetMethod(object[] parameters)
{
    // target = ShellViewModel实例
    // this.TargetMethodInfo = SayHello方法
    
    object result = this.TargetMethodInfo.Invoke(this.Target, null);
    
    // 最终调用：shellViewModel.SayHello()
}
```

### 步骤11：CanExecute更新机制
```csharp
// 当ShellViewModel.Name属性变化时：
public string Name
{
    set
    {
        this.SetAndNotify(ref this._name, value);
        this.NotifyOfPropertyChange(() => this.CanSayHello); // 触发PropertyChanged
    }
}

// CommandAction.PropertyChangedHandler被调用
private void PropertyChangedHandler(object sender, PropertyChangedEventArgs e)
{
    if (e.PropertyName == "CanSayHello")  
    {
        this.UpdateCanExecute(); // 触发CanExecuteChanged事件
    }
}

// Button自动更新IsEnabled状态
public bool CanExecute(object parameter)
{
    return this.guardPropertyGetter(); // 返回shellViewModel.CanSayHello的值
}
```

## 设计优势

### 1. 声明式绑定
- 在XAML中直接声明方法绑定，无需在代码后置中编写事件处理程序
- `{s:Action SayHello}`比传统的事件处理程序更简洁

### 2. 自动化Guard属性支持
- 自动查找`Can{MethodName}`属性来控制命令的可用性
- 支持属性变化通知，实时更新按钮状态

### 3. 类型安全和错误处理
- 编译时检查方法是否存在
- 提供详细的运行时错误信息和日志记录

### 4. 异步支持
- 自动处理返回Task的异步方法
- 确保异常正确传播

### 5. 灵活的错误处理策略
- 可配置的行为：Throw、Enable、Disable
- 支持设计时和运行时的不同行为

## 扩展特性

### 1. 方法参数支持
```csharp
// ViewModel中的方法
public void DeleteItem(object parameter)
{
    var item = parameter as MyItem;
    // 删除逻辑
}

// XAML中的使用
<Button Command="{s:Action DeleteItem}" CommandParameter="{Binding SelectedItem}"/>
```

### 2. 静态方法支持
```csharp
// 绑定到静态方法
<Button Command="{s:Action Target={x:Type local:StaticHelper}, Method=DoSomething}"/>
```

### 3. 显式目标指定
```csharp
// 绑定到特定对象的方法
<Button Command="{s:Action Target={Binding SomeProperty}, Method=SomeMethod}"/>
```

### 4. 事件绑定支持
```csharp
// 绑定事件而非命令
<Button Click="{s:Action SayHello}"/>
```

## 总结

Stylet的Action机制通过以下核心组件协同工作：

1. **ActionExtension**：XAML标记扩展，解析`{s:Action MethodName}`语法
2. **CommandAction**：实现ICommand接口，桥接WPF命令系统和ViewModel方法
3. **ActionBase**：提供目标绑定、方法查找和调用的基础功能
4. **View.ActionTarget**：继承的附加属性，提供方法调用的目标对象
5. **Guard属性机制**：自动处理`Can{MethodName}`属性来控制命令可用性

这种设计使得开发者只需要在XAML中写一行`Command="{s:Action SayHello}"`，就能自动完成：
- Command对象的创建
- ViewModel方法的查找和绑定
- Guard属性的自动发现和监听
- 异常处理和日志记录
- 异步方法支持

整个机制体现了Stylet框架"约定优于配置"的设计理念，极大地简化了MVVM模式中的命令绑定工作。
