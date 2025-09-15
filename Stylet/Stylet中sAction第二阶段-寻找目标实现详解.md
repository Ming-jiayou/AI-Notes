# Stylet中s:Action第二阶段：寻找目标（View.ActionTarget）实现详解

## 概述

在Stylet框架中，`s:Action`的第二阶段是**寻找目标**，这个阶段的核心任务是通过`View.ActionTarget`附加属性将ViewModel注入到可视化树中，使得后续阶段能够找到正确的命令执行目标。

## 核心组件

### 1. View.ActionTarget附加属性

`View.ActionTarget`是定义在[`Stylet.Xaml.View`](Stylet/Xaml/View.cs:48)类中的附加属性：

```csharp
public static readonly DependencyProperty ActionTargetProperty =
    DependencyProperty.RegisterAttached(
        "ActionTarget", 
        typeof(object), 
        typeof(View), 
        new FrameworkPropertyMetadata(
            InitialActionTarget, 
            FrameworkPropertyMetadataOptions.Inherits
        )
    );
```

关键特性：
- **类型**：`object`，可以接收任何类型的ViewModel
- **默认值**：`InitialActionTarget`（一个特殊的标记对象）
- **继承性**：`FrameworkPropertyMetadataOptions.Inherits`确保属性值可以沿着可视化树向下传递

### 2. 属性访问器

[`View`](Stylet/Xaml/View.cs:30)类提供了标准的附加属性访问器：

```csharp
public static object GetActionTarget(DependencyObject obj)
{
    return obj.GetValue(ActionTargetProperty);
}

public static void SetActionTarget(DependencyObject obj, object value)
{
    obj.SetValue(ActionTargetProperty, value);
}
```

## 工作流程

### 步骤1：ViewModel绑定到视图

在Stylet中，ViewModel通常通过以下方式绑定到视图：

```xml
<UserControl ...
    s:View.Model="{Binding}">
    <!-- 视图内容 -->
</UserControl>
```

当[`View.Model`](Stylet/Xaml/View.cs:76)属性被设置时，会触发`PropertyChangedCallback`，通过ViewManager建立视图和ViewModel的关联。

### 步骤2：ActionTarget的继承机制

由于`ActionTargetProperty`设置了`FrameworkPropertyMetadataOptions.Inherits`标志，ViewModel会自动沿着可视化树向下传播：

```
Window (ViewModel)
├── UserControl (继承Window的ViewModel)
│   ├── Button (继承UserControl的ViewModel)
│   └── TextBox (继承UserControl的ViewModel)
└── Grid (继承Window的ViewModel)
```

### 步骤3：ActionExtension获取目标

在[`ActionExtension`](Stylet/Xaml/ActionExtension.cs:135)创建`CommandAction`时，会通过以下方式获取目标：

```csharp
private ICommand CreateCommandAction(IServiceProvider serviceProvider, DependencyObject targetObject)
{
    if (this.Target == null)
    {
        var rootObjectProvider = (IRootObjectProvider)serviceProvider.GetService(typeof(IRootObjectProvider));
        var rootObject = rootObjectProvider?.RootObject as DependencyObject;
        return new CommandAction(targetObject, rootObject, this.Method, ...);
    }
}
```

### 步骤4：CommandAction绑定ActionTarget

在[`CommandAction`](Stylet/Xaml/CommandAction.cs:36)的构造函数中，通过数据绑定机制建立与`ActionTarget`的连接：

```csharp
var actionTargetBinding = new Binding()
{
    Path = new PropertyPath(View.ActionTargetProperty),
    Mode = BindingMode.OneWay,
    Source = this.Subject,
};
BindingOperations.SetBinding(this, targetProperty, actionTargetBinding);
```

## 特殊情况处理

### 1. 上下文菜单和弹出窗口

上下文菜单（ContextMenu）和弹出窗口（Popup）不在主视觉树中，无法继承`ActionTarget`。Stylet通过以下方式解决：

- **备份机制**：[`ActionBase`](Stylet/Xaml/ActionBase.cs:86)提供了备份主题机制
- **显式设置**：开发者需要手动设置`s:View.ActionTarget`：

```xml
<ContextMenu s:View.ActionTarget="{Binding PlacementTarget.DataContext, RelativeSource={RelativeSource Self}}">
    <MenuItem Header="删除" Command="{s:Action DeleteItem}" />
</ContextMenu>
```

### 2. 多绑定转换器

当存在备份主题时，[`ActionBase`](Stylet/Xaml/ActionBase.cs:257)使用`MultiBindingToActionTargetConverter`来选择合适的ActionTarget：

```csharp
private class MultiBindingToActionTargetConverter : IMultiValueConverter
{
    public object Convert(object[] values, Type targetType, object parameter, CultureInfo culture)
    {
        // 优先使用主主题的ActionTarget，如果不可用则使用备份主题
        if (values[0] != View.InitialActionTarget)
            return values[0];
        if (values[1] != View.InitialActionTarget)
            return values[1];
        return View.InitialActionTarget;
    }
}
```

## 状态管理

### 1. 初始状态检测

[`CommandAction`](Stylet/Xaml/CommandAction.cs:132)通过检查`InitialActionTarget`来识别未设置的ActionTarget：

```csharp
if (this.Target == View.InitialActionTarget)
    return true; // 显示为启用状态，但点击时会抛出异常
```

### 2. 错误处理

当ActionTarget未正确设置时，Stylet提供详细的错误信息：

```csharp
var ex = new ActionNotSetException(
    $"View.ActionTarget not set on control {this.Subject} (method {this.MethodName}). " +
    $"This probably means the control hasn't inherited it from a parent, e.g. because a ContextMenu or Popup sits in the visual tree. " +
    $"You will need so set 's:View.ActionTarget' explicitly."
);
```

## 实际应用示例

### 基本用法

```xml
<Window x:Class="MyApp.Views.ShellView"
        xmlns:s="https://github.com/canton7/Stylet">
    <Button Content="点击我" Command="{s:Action SayHello}" />
</Window>
```

在这个例子中：
1. Window的ViewModel自动成为ActionTarget
2. Button通过继承机制获取到Window的ViewModel
3. `SayHello`方法在Window的ViewModel中查找并执行

### 显式设置ActionTarget

```xml
<UserControl xmlns:s="https://github.com/canton7/Stylet">
    <UserControl.Resources>
        <local:MyViewModel x:Key="CustomViewModel" />
    </UserControl.Resources>
    
    <Grid s:View.ActionTarget="{StaticResource CustomViewModel}">
        <Button Command="{s:Action CustomAction}" />
    </Grid>
</UserControl>
```

## 总结

Stylet的第二阶段通过巧妙利用WPF的附加属性和依赖属性继承机制，实现了ViewModel在可视化树中的自动传播。这种设计使得开发者无需手动为每个控件指定ViewModel，大大简化了MVVM模式下的命令绑定。同时，框架还提供了完善的错误处理和特殊情况支持，确保在各种复杂场景下都能正确工作。