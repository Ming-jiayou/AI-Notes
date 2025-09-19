# Stylet View 静态类详解

## 概述

`View.cs` 文件定义了 `View` 静态类，这是 Stylet 框架中的一个重要辅助类，用于管理 WPF 视图和视图模型之间的关系。该类通过定义附加属性（Attached Properties）来实现视图与视图模型的绑定、动作目标的设置以及视图内容的动态管理。作为静态类，它提供了一组工具方法，使 Stylet 框架能够在 WPF 应用程序中实现 MVVM 模式的各种功能。

## 类定义

```csharp
public static class View
{
    // 常量、字段、属性和方法实现...
}
```

## 常量详解

### ViewManagerResourceKey

```csharp
public const string ViewManagerResourceKey = "b9a38199-8cb3-4103-8526-c6cfcd089df7";
```

**用途**：用于从应用程序资源中检索与当前应用程序关联的 ViewManager 的键。

**说明**：
- 这是一个全局唯一的字符串标识符，用于在应用程序资源中查找 ViewManager 实例
- ViewManager 是 Stylet 框架中负责管理视图和视图模型关系的核心组件
- 使用 GUID 格式确保键的唯一性，避免与其他资源键冲突
- 在 Model 属性的变更回调中，通过此键获取 ViewManager 实例

### InitialActionTarget

```csharp
public static readonly object InitialActionTarget = new();
```

**用途**：ActionTarget 属性的初始值，用作标记。

**说明**：
- 用作 ActionTarget 附加属性的默认值
- 可以作为标记使用，如果属性具有此值，表示它尚未被分配给其他对象
- 通过创建新对象实例作为标记值，避免与实际使用的 ActionTarget 值冲突
- 在 ActionTargetProperty 的注册中用作默认值

## 字段详解

### defaultModelValue

```csharp
private static readonly object defaultModelValue = new();
```

**用途**：ModelProperty 的默认值。

**说明**：
- 用作 Model 附加属性的默认值
- 在 Model 属性的变更回调中，用于检测属性是否被重置为默认值
- 通过创建新对象实例作为默认值，避免与实际使用的 ViewModel 值冲突
- 在 ModelProperty 的注册中用作默认值

## 附加属性详解

### ActionTargetProperty

```csharp
public static readonly DependencyProperty ActionTargetProperty =
    DependencyProperty.RegisterAttached("ActionTarget", typeof(object), typeof(View), new FrameworkPropertyMetadata(InitialActionTarget, FrameworkPropertyMetadataOptions.Inherits));

public static object GetActionTarget(DependencyObject obj)
{
    return obj.GetValue(ActionTargetProperty);
}

public static void SetActionTarget(DependencyObject obj, object value)
{
    obj.SetValue(ActionTargetProperty, value);
}
```

**用途**：指定对象的 ActionTarget，用于确定 ActionExtension 标记扩展调用 Actions 的对象。

**类型**：`object`

**默认值**：`InitialActionTarget`

**继承选项**：`FrameworkPropertyMetadataOptions.Inherits`

**说明**：
- 这是一个附加属性，可以附加到任何 `DependencyObject` 上
- 主要用于 ActionExtension 标记扩展，确定在调用 Actions 时应该使用哪个对象作为目标
- 设置为可继承，意味着子元素会自动继承父元素的 ActionTarget 值
- 在典型的 Stylet 应用程序中，ActionTarget 通常是视图模型，用于处理用户交互触发的动作
- `GetActionTarget` 和 `SetActionTarget` 是标准的附加属性访问器方法

### ModelProperty

```csharp
public static readonly DependencyProperty ModelProperty =
    DependencyProperty.RegisterAttached("Model", typeof(object), typeof(View), new PropertyMetadata(defaultModelValue, (d, e) =>
    {
        if (((FrameworkElement)d).TryFindResource(ViewManagerResourceKey) is not IViewManager viewManager)
        {
            if (Execute.InDesignMode)
            {
                BindingExpression bindingExpression = BindingOperations.GetBindingExpression(d, ModelProperty);
                string text;
                if (bindingExpression == null)
                    text = "View for [Broken Binding]";
                else if (bindingExpression.ResolvedSourcePropertyName == null)
                    text = string.Format("View for child ViewModel on {0}", bindingExpression.DataItem.GetType().Name);
                else
                    text = string.Format("View for {0}.{1}", bindingExpression.DataItem.GetType().Name, bindingExpression.ResolvedSourcePropertyName);
                SetContentProperty(d, new System.Windows.Controls.TextBlock() { Text = text });
            }
            else
            {
                throw new InvalidOperationException("The ViewManager resource is unassigned. This should have been set by the Bootstrapper");
            }
        }
        else
        {
            object newValue = e.NewValue == defaultModelValue ? null : e.NewValue;
            viewManager.OnModelChanged(d, e.OldValue, newValue);
        }
    }));

public static object GetModel(DependencyObject obj)
{
    return obj.GetValue(ModelProperty);
}

public static void SetModel(DependencyObject obj, object value)
{
    obj.SetValue(ModelProperty, value);
}
```

**用途**：指定与给定对象关联的当前 ViewModel。

**类型**：`object`

**默认值**：`defaultModelValue`

**说明**：
- 这是 Stylet 框架中最重要的附加属性之一，用于将视图模型与视图关联起来
- 当 Model 属性发生变化时，会触发属性变更回调，执行以下逻辑：
  1. 尝试从应用程序资源中获取 ViewManager 实例
  2. 如果找不到 ViewManager：
     - 在设计模式下，显示一个文本块，提供有关视图模型的信息
     - 在运行时模式下，抛出异常，提示 ViewManager 资源未分配
  3. 如果找到 ViewManager，调用其 `OnModelChanged` 方法，处理视图模型变更
- 在设计模式下的特殊处理，提供了更好的设计时体验
- `GetModel` 和 `SetModel` 是标准的附加属性访问器方法

## 方法详解

### SetContentProperty

```csharp
public static void SetContentProperty(DependencyObject targetLocation, UIElement view)
{
    Type type = targetLocation.GetType();
    ContentPropertyAttribute attribute = type.GetCustomAttribute<ContentPropertyAttribute>();
    // No attribute? Try a property called 'Content'...
    string propertyName = attribute != null ? attribute.Name : "Content";
    PropertyInfo property = type.GetProperty(propertyName);
    if (property == null)
        throw new InvalidOperationException(string.Format("Unable to find a Content property on type {0}. Make sure you're using 's:View.Model' on a suitable container, e.g. a ContentControl", type.Name));
    property.SetValue(targetLocation, view);
}
```

**用途**：辅助方法，用于将给定对象的 Content 属性设置为特定视图。

**参数**：
- `targetLocation` (DependencyObject): 要设置 Content 属性的对象
- `view` (UIElement): 要设置对象 Content 的视图

**说明**：
- 这是一个辅助方法，用于动态设置对象的 Content 属性
- 首先尝试通过反射获取类型的 `ContentPropertyAttribute`，确定内容属性名称
- 如果没有找到 `ContentPropertyAttribute`，则默认使用 "Content" 作为属性名
- 通过反射获取属性信息，并将视图设置为属性的值
- 如果找不到合适的属性，抛出异常，提示用户使用合适的容器（如 ContentControl）
- 在 Model 属性的设计模式处理中，用于显示占位符文本块

## 使用方式

### 基本用法

```xml
<UserControl x:Class="MyApp.Views.ShellView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:s="https://github.com/canton7/Stylet"
             s:View.Model="{Binding ShellViewModel}">
    <!-- 视图内容 -->
</UserControl>
```

**说明**：
- 使用 `s:View.Model` 附加属性将视图与视图模型关联
- 通常在视图的根元素上设置此属性
- 通过数据绑定将视图模型实例分配给 Model 属性

### 在 ContentControl 中使用

```xml
<ContentControl s:View.Model="{Binding ActiveItem}" />
```

**说明**：
- 在 ContentControl 中使用 Model 属性，实现动态视图切换
- 当 ActiveItem 属性发生变化时，ContentControl 会自动显示对应的视图
- 这是 Stylet 中实现导航和屏幕管理的常用方式

### 在 ItemsControl 中使用

```xml
<ItemsControl ItemsSource="{Binding Items}">
    <ItemsControl.ItemTemplate>
        <DataTemplate>
            <ContentControl s:View.Model="{Binding}" />
        </DataTemplate>
    </ItemsControl.ItemTemplate>
</ItemsControl>
```

**说明**：
- 在 ItemsControl 的 ItemTemplate 中使用 Model 属性
- 为集合中的每个项创建对应的视图
- 常用于显示列表或集合中的视图模型项

### 设置 ActionTarget

```xml
<Button s:View.ActionTarget="{Binding}" Command="{s:Action MyCommand}" />
```

**说明**：
- 使用 `s:View.ActionTarget` 附加属性设置动作目标
- 通常与 `s:Action` 标记扩展一起使用，指定动作处理的目标对象
- 在此示例中，按钮的动作目标是当前数据上下文（通常是视图模型）

## 设计模式

### 1. 附加属性模式

- 使用 WPF 的附加属性机制，为现有控件添加新功能
- 允许在不继承控件的情况下扩展其行为
- 提供了一种灵活的方式来关联视图和视图模型

### 2. 视图定位模式

- 通过 Model 属性的变化，自动定位并加载对应的视图
- ViewManager 负责根据视图模型类型查找和创建视图实例
- 实现了视图模型到视图的动态映射

### 3. 设计时支持模式

- 在设计模式下提供有意义的占位内容
- 显示视图模型信息，帮助开发者在设计时理解视图结构
- 区分设计时和运行时行为，提供不同的用户体验

## 工作原理

### 1. 视图模型绑定

1. 在 XAML 中使用 `s:View.Model` 附加属性绑定视图模型
2. 当绑定生效时，ModelProperty 的值发生变化
3. 触发属性变更回调，执行以下逻辑：
   - 从应用程序资源中获取 ViewManager 实例
   - 调用 ViewManager 的 `OnModelChanged` 方法
   - ViewManager 根据视图模型类型定位并创建对应的视图
   - 将创建的视图设置为对象的 Content 属性

### 2. 动作目标设置

1. 在 XAML 中使用 `s:View.ActionTarget` 附加属性设置动作目标
2. ActionTargetProperty 的值被设置为指定的对象（通常是视图模型）
3. 当使用 `s:Action` 标记扩展时，会查找当前元素或其父元素的 ActionTarget
4. 找到 ActionTarget 后，在其上调用指定的动作方法

### 3. 设计时处理

1. 在设计模式下，ModelProperty 的变更回调会执行特殊逻辑
2. 检查 Model 属性的绑定表达式，提取视图模型信息
3. 创建包含视图模型信息的文本块
4. 调用 `SetContentProperty` 方法，将文本块设置为对象的 Content
5. 在设计器中显示占位内容，提供视觉反馈

## 最佳实践

### 1. 正确使用 Model 属性

```xml
<!-- 推荐：在 ContentControl 或类似容器上使用 Model 属性 -->
<ContentControl s:View.Model="{Binding ActiveItem}" />

<!-- 不推荐：在没有 Content 属性的元素上使用 Model 属性 -->
<Button s:View.Model="{Binding SomeViewModel}" />
```

**说明**：
- 确保 Model 属性设置在具有 Content 属性的元素上
- 优先使用 ContentControl、UserControl 等容器元素
- 避免在不适合的元素上使用 Model 属性，可能导致异常

### 2. 合理设置 ActionTarget

```xml
<!-- 推荐：在容器上设置 ActionTarget，子元素继承 -->
<Grid s:View.ActionTarget="{Binding}">
    <Button Command="{s:Action Save}" />
    <Button Command="{s:Action Cancel}" />
</Grid>

<!-- 不推荐：在每个元素上单独设置 ActionTarget -->
<Button Command="{s:Action Save}" s:View.ActionTarget="{Binding}" />
<Button Command="{s:Action Cancel}" s:View.ActionTarget="{Binding}" />
```

**说明**：
- 利用 ActionTarget 的继承特性，在容器上设置一次
- 子元素会自动继承父元素的 ActionTarget
- 减少重复代码，提高可维护性

### 3. 处理设计时视图

```xml
<!-- 为设计时提供默认数据上下文 -->
<UserControl x:Class="MyApp.Views.ShellView"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:s="https://github.com/canton7/Stylet"
             xmlns:d="http://schemas.microsoft.com/expression/blend/2008"
             xmlns:mc="http://schemas.openxmlformats.org/markup-compatibility/2006"
             d:DataContext="{d:DesignInstance Type=local:ShellViewModel, IsDesignTimeCreatable=True}"
             mc:Ignorable="d"
             s:View.Model="{Binding ShellViewModel}">
    <!-- 视图内容 -->
</UserControl>
```

**说明**：
- 使用设计时数据上下文，提供更好的设计时体验
- 结合 `d:DataContext` 和 `s:View.Model`，确保设计时和运行时都能正常工作
- 使设计器能够显示有意义的占位内容

## 常见问题与解决方案

### 1. ViewManager 资源未分配

**问题**：运行时出现异常，提示 "The ViewManager resource is unassigned"。

**解决方案**：
```csharp
// 确保在 Bootstrapper 中正确设置 ViewManager 资源
protected override void ConfigureIoC(IStyletIoCBuilder builder)
{
    base.ConfigureIoC(builder);
    
    // 其他配置...
}

protected override void OnLaunch()
{
    base.OnLaunch();
    
    // 确保 ViewManager 资源已设置
    this.RootView.Resources.Add(View.ViewManagerResourceKey, this.ViewManager);
}
```

**说明**：
- 确保在 Bootstrapper 中正确设置 ViewManager 资源
- 检查 `OnLaunch` 方法中是否添加了 ViewManager 资源
- 验证资源键是否正确使用了 `View.ViewManagerResourceKey` 常量

### 2. 找不到 Content 属性

**问题**：设置 Model 属性时出现异常，提示 "Unable to find a Content property"。

**解决方案**：
```xml
<!-- 使用具有 Content 属性的容器 -->
<ContentControl s:View.Model="{Binding ActiveItem}" />

<!-- 或者使用 UserControl -->
<UserControl s:View.Model="{Binding ViewModel}">
    <!-- 内容 -->
</UserControl>
```

**说明**：
- 确保 Model 属性设置在具有 Content 属性的元素上
- 常用的容器包括 ContentControl、UserControl、Page 等
- 避免在没有 Content 属性的元素（如 Button、TextBlock）上直接设置 Model 属性

### 3. 动作不触发

**问题**：使用 `s:Action` 标记扩展时，动作方法不被调用。

**解决方案**：
```xml
<!-- 确保设置了正确的 ActionTarget -->
<Grid s:View.ActionTarget="{Binding}">
    <Button Command="{s:Action MyMethod}" />
</Grid>

<!-- 或者直接在按钮上设置 ActionTarget -->
<Button Command="{s:Action MyMethod}" s:View.ActionTarget="{Binding}" />
```

**说明**：
- 确保 ActionTarget 已正确设置为包含动作方法的对象
- 检查动作方法的名称和参数是否正确
- 验证数据上下文是否正确设置

## 总结

`View` 静态类是 Stylet 框架中实现 MVVM 模式的核心组件之一，通过附加属性机制提供了视图与视图模型的绑定、动作目标的设置以及视图内容的动态管理。该类的主要职责包括：

1. **视图模型绑定**：通过 `Model` 附加属性，将视图与视图模型关联起来，实现自动视图定位和加载。
2. **动作目标管理**：通过 `ActionTarget` 附加属性，确定动作处理的目标对象，支持用户交互。
3. **设计时支持**：在设计模式下提供有意义的占位内容，增强开发体验。
4. **内容属性设置**：通过 `SetContentProperty` 辅助方法，动态设置对象的 Content 属性。

通过理解 `View` 静态类的设计和工作原理，开发者可以更好地使用 Stylet 框架，构建出结构清晰、易于维护的 WPF 应用程序。该类提供的附加属性和辅助方法，使 Stylet 框架能够在 WPF 中实现优雅的 MVVM 模式，简化视图和视图模型的管理。