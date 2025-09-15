# Stylet中s:Action第三阶段：生成命令（CommandAction）实现详解

## 概述

在Stylet框架中，`s:Action`的第三阶段是**生成命令**，这个阶段的核心任务是实现`ICommand`接口，并通过反射查找目标对象上的方法（如`SayHello`）和对应的条件方法（如`CanSayHello`）。这个过程由`CommandAction`类完成，它是连接ViewModel方法与UI命令的桥梁。

## 核心组件：CommandAction

### 1. 类定义与继承结构

[`CommandAction`](Stylet/Xaml/CommandAction.cs:19)继承自`ActionBase`并实现`ICommand`接口：

```csharp
public class CommandAction : ActionBase, ICommand
```

### 2. 核心字段

[`CommandAction`](Stylet/Xaml/CommandAction.cs:21)包含以下关键字段：

```csharp
private static readonly ILogger logger = LogManager.GetLogger(typeof(CommandAction));
private Func<bool> guardPropertyGetter;  // 缓存的条件方法委托
private string guardName => "Can" + this.MethodName;  // 条件方法名规则
```

## 构造函数与初始化

### 1. 使用View.ActionTarget的构造函数

[`CommandAction`](Stylet/Xaml/CommandAction.cs:36)提供使用视觉树查找目标的构造函数：

```csharp
public CommandAction(DependencyObject subject, DependencyObject backupSubject, string methodName, 
    ActionUnavailableBehaviour targetNullBehaviour, ActionUnavailableBehaviour actionNonExistentBehaviour)
    : base(subject, backupSubject, methodName, targetNullBehaviour, actionNonExistentBehaviour, logger)
{ }
```

### 2. 显式目标的构造函数

[`CommandAction`](Stylet/Xaml/CommandAction.cs:47)提供使用显式目标的构造函数：

```csharp
public CommandAction(object target, string methodName, 
    ActionUnavailableBehaviour targetNullBehaviour, ActionUnavailableBehaviour actionNonExistentBehaviour)
    : base(target, methodName, targetNullBehaviour, actionNonExistentBehaviour, logger)
{ }
```

## 方法查找机制

### 1. 方法签名验证

[`AssertTargetMethodInfo`](Stylet/Xaml/CommandAction.cs:58)验证目标方法的签名：

```csharp
private protected override void AssertTargetMethodInfo(MethodInfo targetMethodInfo, Type newTargetType)
{
    ParameterInfo[] methodParameters = targetMethodInfo.GetParameters();
    if (methodParameters.Length > 1)
    {
        var e = new ActionSignatureInvalidException(
            string.Format("Method {0} on {1} must have zero or one parameters", 
            this.MethodName, newTargetType.Name));
        logger.Error(e);
        throw e;
    }
}
```

### 2. 方法查找规则

- **实例方法**：`BindingFlags.Public | BindingFlags.Instance`
- **静态方法**：`BindingFlags.Public | BindingFlags.Static`
- **参数限制**：0个或1个参数
- **返回类型**：无限制（支持`async Task`）

## 条件方法（Guard Method）机制

### 1. 条件方法命名约定

条件方法遵循固定命名规则：
- 主方法：`SayHello`
- 条件方法：`CanSayHello`

### 2. 条件方法查找与缓存

在[`OnTargetChanged`](Stylet/Xaml/CommandAction.cs:74)中查找并缓存条件方法：

```csharp
private protected override void OnTargetChanged(object oldTarget, object newTarget)
{
    // 移除旧的事件监听
    if (oldTarget is INotifyPropertyChanged oldInpc)
        PropertyChangedEventManager.RemoveHandler(oldInpc, this.PropertyChangedHandler, this.guardName);

    this.guardPropertyGetter = null;
    
    // 查找条件属性
    PropertyInfo guardPropertyInfo = newTarget?.GetType().GetProperty(this.guardName);
    if (guardPropertyInfo != null && guardPropertyInfo.PropertyType == typeof(bool))
    {
        // 使用表达式树编译高性能委托
        Expressions.ConstantExpression targetExpression = Expressions.Expression.Constant(newTarget);
        Expressions.MemberExpression propertyAccess = 
            Expressions.Expression.Property(targetExpression, guardPropertyInfo);
        this.guardPropertyGetter = 
            Expressions.Expression.Lambda<Func<bool>>(propertyAccess).Compile();
    }
    
    // 设置属性变更监听
    if (this.guardPropertyGetter != null && newTarget is INotifyPropertyChanged inpc)
    {
        PropertyChangedEventManager.AddHandler(inpc, this.PropertyChangedHandler, this.guardName);
    }
    
    this.UpdateCanExecute();
}
```

### 3. 条件方法使用示例

```csharp
public class MyViewModel : PropertyChangedBase
{
    public void SayHello()
    {
        // 实际执行逻辑
    }
    
    public bool CanSayHello
    {
        get { return !string.IsNullOrEmpty(Name); }
    }
    
    private string name;
    public string Name
    {
        get => name;
        set 
        {
            name = value;
            NotifyOfPropertyChange();
            NotifyOfPropertyChange(nameof(CanSayHello)); // 触发CanExecuteChanged
        }
    }
}
```

## ICommand接口实现

### 1. CanExecute方法

[`CanExecute`](Stylet/Xaml/CommandAction.cs:126)实现命令可用性检查：

```csharp
public bool CanExecute(object parameter)
{
    // 处理ActionTarget未设置的情况
    if (this.Target == View.InitialActionTarget)
        return true;

    // 处理目标为null的情况
    if (this.Target == null)
        return this.TargetNullBehaviour != ActionUnavailableBehaviour.Disable;

    // 处理方法不存在的情况
    if (this.TargetMethodInfo == null)
        return this.ActionNonExistentBehaviour != ActionUnavailableBehaviour.Disable;

    // 使用条件方法检查结果
    return this.guardPropertyGetter?.Invoke() ?? true;
}
```

### 2. Execute方法

[`Execute`](Stylet/Xaml/CommandAction.cs:160)实现命令执行：

```csharp
public void Execute(object parameter)
{
    this.AssertTargetSet(); // 验证目标已设置
    
    if (this.Target == null || this.TargetMethodInfo == null)
        return;

    // 构建参数数组
    object[] parameters = this.TargetMethodInfo.GetParameters().Length == 1 
        ? new[] { parameter } 
        : null;
    
    this.InvokeTargetMethod(parameters);
}
```

### 3. CanExecuteChanged事件

[`CanExecuteChanged`](Stylet/Xaml/CommandAction.cs:154)事件用于通知UI更新：

```csharp
public event EventHandler CanExecuteChanged;

private void UpdateCanExecute()
{
    EventHandler handler = this.CanExecuteChanged;
    if (handler != null)
        Stylet.Execute.OnUIThread(() => handler(this, EventArgs.Empty));
}
```

## 异步方法支持

### 1. Task返回类型处理

在[`InvokeTargetMethod`](Stylet/Xaml/ActionBase.cs:224)中支持异步方法：

```csharp
private protected void InvokeTargetMethod(object[] parameters)
{
    try
    {
        object target = this.TargetMethodInfo.IsStatic ? null : this.Target;
        object result = this.TargetMethodInfo.Invoke(target, parameters);
        
        // 处理异步方法
        if (result is Task task)
        {
            AwaitTask(task);
        }
    }
    catch (TargetInvocationException e)
    {
        // 异常包装和重新抛出
        ExceptionDispatchInfo.Capture(e.InnerException).Throw();
    }

    async void AwaitTask(Task t) => await t;
}
```

### 2. 异步方法示例

```csharp
public async Task SaveAsync()
{
    await dataService.SaveAsync();
}

public bool CanSaveAsync => !IsBusy;
```

## 性能优化

### 1. 表达式树编译

使用`System.Linq.Expressions`编译条件方法，避免反射性能开销：

```csharp
// 传统反射调用
(bool)guardPropertyInfo.GetValue(target);

// 表达式树编译（性能提升10倍以上）
Func<bool> compiledDelegate = Expression.Lambda<Func<bool>>(
    Expression.Property(Expression.Constant(target), guardPropertyInfo)
).Compile();
```

### 2. 事件监听优化

使用`PropertyChangedEventManager`弱事件模式，避免内存泄漏：

```csharp
// 添加监听
PropertyChangedEventManager.AddHandler(inpc, this.PropertyChangedHandler, this.guardName);

// 移除监听
PropertyChangedEventManager.RemoveHandler(oldInpc, this.PropertyChangedHandler, this.guardName);
```

## 错误处理与调试

### 1. 详细日志记录

[`CommandAction`](Stylet/Xaml/CommandAction.cs:21)使用日志记录关键操作：

```csharp
logger.Info("Invoking method {0} on {1} with parameters ({2})", 
    this.MethodName, this.TargetName(), 
    parameters == null ? "none" : string.Join(", ", parameters));
```

### 2. 异常处理

- **方法签名无效**：`ActionSignatureInvalidException`
- **方法不存在**：`ActionNotFoundException`
- **目标未设置**：`ActionNotSetException`
- **目标为null**：`ActionTargetNullException`

### 3. 调试信息

在调试模式下提供详细信息：

```csharp
logger.Warn("Found guard property {0} for action {1} on target {2}, but its return type wasn't bool", 
    this.guardName, this.MethodName, newTarget);
```

## 高级特性

### 1. 参数传递

支持命令参数传递：

```xml
<Button Content="删除" 
        Command="{s:Action DeleteItem}" 
        CommandParameter="{Binding SelectedItem}" />
```

```csharp
public void DeleteItem(object item)
{
    Items.Remove(item);
}

public bool CanDeleteItem => SelectedItem != null;
```

### 2. 静态方法支持

支持调用静态方法：

```csharp
public static void GlobalAction()
{
    // 静态方法实现
}

// XAML中使用
<Button Command="{s:Action GlobalAction}" />
```

### 3. 泛型方法支持

通过反射支持泛型方法调用（需要显式指定类型参数）。

## 实际应用示例

### 1. 基本命令绑定

```csharp
public class ShellViewModel : Screen
{
    public void SayHello()
    {
        MessageBox.Show("Hello from ViewModel!");
    }
    
    public bool CanSayHello => true;
}
```

### 2. 带参数的命令

```csharp
public void SaveDocument(Document doc)
{
    documentService.Save(doc);
}

public bool CanSaveDocument(Document doc) => doc?.IsModified == true;
```

### 3. 异步命令

```csharp
public async Task LoadDataAsync()
{
    IsBusy = true;
    try
    {
        Data = await dataService.LoadAsync();
    }
    finally
    {
        IsBusy = false;
    }
}

public bool CanLoadDataAsync => !IsBusy;
```

## 总结

Stylet的第三阶段通过`CommandAction`类实现了完整的命令生成机制，将ViewModel中的方法转换为WPF可用的`ICommand`对象。其设计特点包括：

1. **智能方法查找**：自动发现方法和对应的条件方法
2. **高性能**：使用表达式树编译避免反射开销
3. **内存安全**：弱事件模式避免内存泄漏
4. **异步支持**：完整支持async/await模式
5. **调试友好**：详细的日志和错误信息
6. **灵活配置**：多种行为配置选项

这一机制使得开发者可以专注于业务逻辑，而无需关心命令系统的底层实现细节。