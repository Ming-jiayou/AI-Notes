# Stylet中s:Action第一阶段：XAML解析与ActionExtension实现详解

## 概述

在Stylet框架中，`s:Action`的第一阶段是**XAML解析**，这个阶段的核心任务是将XAML标记中的`{s:Action SayHello}`语法翻译成`CommandAction`实例。这个过程由`ActionExtension`标记扩展类完成，它是整个命令绑定系统的入口点。

## 核心组件：ActionExtension

### 1. 类定义与继承结构

[`ActionExtension`](Stylet/Xaml/ActionExtension.cs:39)是一个标记扩展类，继承自WPF的`MarkupExtension`：

```csharp
public class ActionExtension : MarkupExtension
```

### 2. 核心属性

[`ActionExtension`](Stylet/Xaml/ActionExtension.cs:41)提供了多个配置属性：

```csharp
[ConstructorArgument("method")]
public string Method { get; set; }  // 要调用的方法名

public object Target { get; set; }  // 覆盖View.ActionTarget的显式目标

public ActionUnavailableBehaviour NullTarget { get; set; }      // 目标为null时的行为
public ActionUnavailableBehaviour ActionNotFound { get; set; }  // 方法不存在时的行为
```

### 3. 构造函数

[`ActionExtension`](Stylet/Xaml/ActionExtension.cs:65)提供了两个构造函数：

```csharp
// 无参构造函数
public ActionExtension() { }

// 带方法名的构造函数
public ActionExtension(string method)
{
    this.Method = method;
}
```

## XAML解析流程

### 步骤1：XAML解析器调用ProvideValue

当XAML解析器遇到`{s:Action SayHello}`时，会调用[`ActionExtension.ProvideValue`](Stylet/Xaml/ActionExtension.cs:95)方法：

```csharp
public override object ProvideValue(IServiceProvider serviceProvider)
{
    if (this.Method == null)
        throw new InvalidOperationException("Method has not been set");

    var valueService = (IProvideValueTarget)serviceProvider.GetService(typeof(IProvideValueTarget));
    
    return valueService.TargetObject switch
    {
        DependencyObject targetObject => this.HandleDependencyObject(serviceProvider, valueService, targetObject),
        CommandBinding commandBinding => this.CreateEventAction(serviceProvider, null, ((EventInfo)valueService.TargetProperty).EventHandlerType, isCommandBinding: true),
        _ => this,
    };
}
```

### 步骤2：目标对象类型判断

[`HandleDependencyObject`](Stylet/Xaml/ActionExtension.cs:112)方法根据目标属性的类型进行分支处理：

```csharp
private object HandleDependencyObject(IServiceProvider serviceProvider, IProvideValueTarget valueService, DependencyObject targetObject)
{
    switch (valueService.TargetProperty)
    {
        case DependencyProperty dependencyProperty when dependencyProperty.PropertyType == typeof(ICommand):
            return this.CreateCommandAction(serviceProvider, targetObject);
        case EventInfo eventInfo:
            return this.CreateEventAction(serviceProvider, targetObject, eventInfo.EventHandlerType);
        case MethodInfo methodInfo:
            return this.CreateEventAction(serviceProvider, targetObject, parameters[1].ParameterType);
        default:
            throw new ArgumentException("Can only use ActionExtension with a Command property or an event handler");
    }
}
```

### 步骤3：创建CommandAction实例

对于命令绑定场景，[`CreateCommandAction`](Stylet/Xaml/ActionExtension.cs:135)方法创建`CommandAction`实例：

```csharp
private ICommand CreateCommandAction(IServiceProvider serviceProvider, DependencyObject targetObject)
{
    if (this.Target == null)
    {
        var rootObjectProvider = (IRootObjectProvider)serviceProvider.GetService(typeof(IRootObjectProvider));
        var rootObject = rootObjectProvider?.RootObject as DependencyObject;
        return new CommandAction(targetObject, rootObject, this.Method, 
            this.commandNullTargetBehaviour, this.commandActionNotFoundBehaviour);
    }
    else
    {
        return new CommandAction(this.Target, this.Method, 
            this.commandNullTargetBehaviour, this.commandActionNotFoundBehaviour);
    }
}
```

## 行为配置机制

### 1. 默认行为定义

[`ActionExtension`](Stylet/Xaml/ActionExtension.cs:78)为不同场景定义了默认行为：

```csharp
private ActionUnavailableBehaviour commandNullTargetBehaviour =>
    this.NullTarget == ActionUnavailableBehaviour.Default 
        ? (Execute.InDesignMode ? ActionUnavailableBehaviour.Enable : ActionUnavailableBehaviour.Disable) 
        : this.NullTarget;

private ActionUnavailableBehaviour commandActionNotFoundBehaviour =>
    this.ActionNotFound == ActionUnavailableBehaviour.Default 
        ? ActionUnavailableBehaviour.Throw 
        : this.ActionNotFound;
```

### 2. 行为枚举定义

[`ActionUnavailableBehaviour`](Stylet/Xaml/ActionExtension.cs:13)枚举定义了各种处理行为：

```csharp
public enum ActionUnavailableBehaviour
{
    Default,    // 默认行为
    Enable,     // 启用控件但不执行操作
    Disable,    // 禁用控件
    Throw       // 抛出异常
}
```

## 服务提供者机制

### 1. IProvideValueTarget服务

通过`IProvideValueTarget`服务获取目标信息：

```csharp
var valueService = (IProvideValueTarget)serviceProvider.GetService(typeof(IProvideValueTarget));
// valueService.TargetObject - 目标对象
// valueService.TargetProperty - 目标属性
```

### 2. IRootObjectProvider服务

通过`IRootObjectProvider`服务获取XAML根对象：

```csharp
var rootObjectProvider = (IRootObjectProvider)serviceProvider.GetService(typeof(IRootObjectProvider));
var rootObject = rootObjectProvider?.RootObject as DependencyObject;
```

## 模板化场景处理

### 1. 模板中的延迟处理

在模板化场景中，[`ProvideValue`](Stylet/Xaml/ActionExtension.cs:106)返回`this`以支持延迟处理：

```csharp
_ => this,
```

这是因为模板中的元素在第一次调用`ProvideValue`时可能还没有正确初始化，需要等待后续的真正绑定。

### 2. 设计时支持

[`ActionExtension`](Stylet/Xaml/ActionExtension.cs:79)针对设计时模式提供了特殊处理：

```csharp
Execute.InDesignMode ? ActionUnavailableBehaviour.Enable : ActionUnavailableBehaviour.Disable
```

## 实际应用示例

### 1. 基本命令绑定

```xml
<Button Content="点击我" Command="{s:Action SayHello}" />
```

解析过程：
1. XAML解析器遇到`{s:Action SayHello}`
2. 创建`ActionExtension`实例，设置`Method="SayHello"`
3. 调用`ProvideValue`，检测到目标属性是`ICommand`类型
4. 创建`CommandAction`实例并返回

### 2. 带行为配置的绑定

```xml
<Button Content="点击我" 
        Command="{s:Action SayHello, NullTarget=Disable, ActionNotFound=Throw}" />
```

### 3. 事件绑定

```xml
<Button Content="点击我">
    <i:Interaction.Triggers>
        <i:EventTrigger EventName="Click">
            <i:InvokeCommandAction Command="{s:Action HandleClick}" />
        </i:EventTrigger>
    </i:Interaction.Triggers>
</Button>
```

## 错误处理机制

### 1. 方法名为空检查

```csharp
if (this.Method == null)
    throw new InvalidOperationException("Method has not been set");
```

### 2. 无效目标属性检查

```csharp
default:
    throw new ArgumentException("Can only use ActionExtension with a Command property or an event handler");
```

### 3. 上下文相关错误

对于事件绑定场景，提供了专门的错误类型：

- `ActionNotSetException` - ActionTarget未设置
- `ActionTargetNullException` - 目标为null
- `ActionNotFoundException` - 方法不存在
- `ActionSignatureInvalidException` - 方法签名无效

## 性能优化

### 1. 延迟实例化

标记扩展采用延迟实例化策略，只有在真正需要时才创建`CommandAction`或`EventAction`实例。

### 2. 缓存机制

虽然第一阶段不直接涉及缓存，但为后续阶段的缓存和优化奠定了基础。

## 总结

Stylet的XAML解析阶段通过`ActionExtension`标记扩展实现了从声明式XAML语法到实际命令对象的无缝转换。这个过程充分利用了WPF的标记扩展机制，提供了灵活的配置选项和完善的错误处理，为后续的命令执行阶段奠定了坚实的基础。

关键特点：
- **简洁的语法**：`{s:Action MethodName}`直观易用
- **灵活的配置**：支持多种行为配置选项
- **完善的错误处理**：提供详细的错误信息和调试支持
- **设计时友好**：在设计器中也能正常工作
- **模板支持**：正确处理模板化场景