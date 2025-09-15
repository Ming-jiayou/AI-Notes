# Stylet ViewModelFirst机制详解

## 概述

Stylet框架采用了ViewModelFirst的设计模式，这意味着开发者首先创建ViewModel，然后框架会自动找到对应的View并建立绑定关系。当在XAML中使用`s:View.Model="{Binding HeaderViewModel}"`这样的语法时，Stylet会自动创建HeaderView并将其显示出来。

## 核心机制原理

### 1. 附加属性（Attached Property）机制

#### View.Model附加属性的定义

```csharp
// 文件：Stylet\Xaml\View.cs
public static readonly DependencyProperty ModelProperty =
    DependencyProperty.RegisterAttached("Model", typeof(object), typeof(View), 
        new PropertyMetadata(defaultModelValue, (d, e) =>
        {
            // 当Model属性变化时的回调处理
            if (((FrameworkElement)d).TryFindResource(ViewManagerResourceKey) is not IViewManager viewManager)
            {
                // 设计时模式处理
                if (Execute.InDesignMode)
                {
                    // 显示设计时占位文本
                    BindingExpression bindingExpression = BindingOperations.GetBindingExpression(d, ModelProperty);
                    string text = "View for " + bindingExpression?.ResolvedSourcePropertyName;
                    SetContentProperty(d, new System.Windows.Controls.TextBlock() { Text = text });
                }
                else
                {
                    throw new InvalidOperationException("The ViewManager resource is unassigned. This should have been set by the Bootstrapper");
                }
            }
            else
            {
                // 运行时处理：委托给ViewManager处理Model变化
                object newValue = e.NewValue == defaultModelValue ? null : e.NewValue;
                viewManager.OnModelChanged(d, e.OldValue, newValue);
            }
        }));
```

**关键点：**
- `ModelProperty`是一个附加属性（Attached Property），可以附加到任何DependencyObject上
- 当属性值发生变化时，会触发回调函数
- 回调函数中会从应用程序资源中获取ViewManager实例，然后委托其处理Model变化

### 2. ViewManager的注册与获取

#### Bootstrapper中的ViewManager注册

```csharp
// 文件：Stylet\BootstrapperBase.cs
public virtual void Start(string[] args)
{
    // 设置参数
    this.Args = args;
    this.OnStart();

    // 配置IoC容器
    this.ConfigureBootstrapper();

    // 关键：将ViewManager注册到应用程序资源中
    this.Application?.Resources.Add(View.ViewManagerResourceKey, this.GetInstance(typeof(IViewManager)));

    this.Configure();
    this.Launch();
    this.OnLaunch();
}
```

**ViewManagerResourceKey的定义：**
```csharp
// 文件：Stylet\Xaml\View.cs
public const string ViewManagerResourceKey = "b9a38199-8cb3-4103-8526-c6cfcd089df7";
```

### 3. ViewManager的核心逻辑

#### Model变化时的处理逻辑

```csharp
// 文件：Stylet\ViewManager.cs
public virtual void OnModelChanged(DependencyObject targetLocation, object oldValue, object newValue)
{
    if (oldValue == newValue)
        return;

    if (newValue != null)
    {
        logger.Info("View.Model changed for {0} from {1} to {2}", targetLocation, oldValue, newValue);
        
        // 1. 为ViewModel创建并绑定View
        UIElement view = this.CreateAndBindViewForModelIfNecessary(newValue);
        
        // 2. 检查View类型（不能是Window）
        if (view is Window)
        {
            var e = new StyletInvalidViewTypeException("s:View.Model不能显示Window类型的View，请使用UserControl或其他控件");
            throw e;
        }
        
        // 3. 将View设置为目标容器的Content
        View.SetContentProperty(targetLocation, view);
    }
    else
    {
        // 清空Model时，清除Content
        logger.Info("View.Model cleared for {0}, from {1}", targetLocation, oldValue);
        View.SetContentProperty(targetLocation, null);
    }
}
```

#### 创建和绑定View的流程

```csharp
// 文件：Stylet\ViewManager.cs
public virtual UIElement CreateAndBindViewForModelIfNecessary(object model)
{
    // 1. 检查ViewModel是否已经有View
    if (model is IViewAware modelAsViewAware && modelAsViewAware.View != null)
    {
        logger.Info("ViewModel {0} already has a View attached to it. Not attaching another", model);
        return modelAsViewAware.View;
    }

    // 2. 创建新的View并绑定
    return this.CreateAndBindViewForModel(model);
}

protected virtual UIElement CreateAndBindViewForModel(object model)
{
    // 注意：需要先绑定再初始化View
    // 否则Command绑定评估时ActionTarget还没有设置
    logger.Info("Instantiating and binding a new View to ViewModel {0}", model);
    
    // 1. 创建View
    UIElement view = this.CreateViewForModel(model);
    
    // 2. 绑定View和ViewModel
    this.BindViewToModel(view, model);
    
    return view;
}
```

### 4. View的定位和创建机制

#### View类型的定位算法

```csharp
// 文件：Stylet\ViewManager.cs
protected virtual Type LocateViewForModel(Type modelType)
{
    // 1. 获取ViewModel的完整类型名
    string modelName = modelType.FullName;
    
    // 2. 转换为View类型名
    string viewName = this.ViewTypeNameForModelTypeName(modelName);
    
    if (modelName == viewName)
        throw new StyletViewLocationException("无法将ViewModel名称转换为合适的View名称");

    // 3. 在程序集中查找View类型
    Type viewType = this.ViewTypeForViewName(viewName, new[] { modelType.Assembly });
    
    if (viewType == null)
    {
        throw new StyletViewLocationException($"无法找到类型为 {viewName} 的View");
    }

    return viewType;
}
```

#### 命名约定转换

```csharp
// 文件：Stylet\ViewManager.cs
protected virtual string ViewTypeNameForModelTypeName(string modelTypeName)
{
    string transformed = modelTypeName;

    // 1. 应用命名空间转换规则
    foreach (KeyValuePair<string, string> transformation in this.NamespaceTransformations)
    {
        if (transformed.StartsWith(transformation.Key + "."))
        {
            transformed = transformation.Value + transformed.Substring(transformation.Key.Length);
            break;
        }
    }

    // 2. 将"ViewModel"后缀替换为"View"后缀
    // 默认：ViewModelNameSuffix = "ViewModel", ViewNameSuffix = "View"
    transformed = Regex.Replace(transformed,
        $@"(?<=.){Regex.Escape(this.ViewModelNameSuffix)}(?=s?\.)|{Regex.Escape(this.ViewModelNameSuffix)}$",
        Regex.Escape(this.ViewNameSuffix));

    return transformed;
}
```

**示例转换：**
- `Stylet.Samples.NavigationController.Pages.HeaderViewModel` 
- → `Stylet.Samples.NavigationController.Pages.HeaderView`

#### View实例的创建

```csharp
// 文件：Stylet\ViewManager.cs
public virtual UIElement CreateViewForModel(object model)
{
    // 1. 定位View类型
    Type viewType = this.LocateViewForModel(model.GetType());

    // 2. 验证View类型
    if (viewType.IsAbstract || !typeof(UIElement).IsAssignableFrom(viewType))
    {
        throw new StyletViewLocationException($"找到的View类型 {viewType.Name} 不是UIElement的派生类");
    }

    // 3. 通过ViewFactory创建View实例
    var view = (UIElement)this.ViewFactory(viewType);

    // 4. 初始化View（调用InitializeComponent等）
    this.InitializeView(view, viewType);

    return view;
}
```

### 5. View和ViewModel的绑定机制

```csharp
// 文件：Stylet\ViewManager.cs
public virtual void BindViewToModel(UIElement view, object viewModel)
{
    // 1. 设置ActionTarget（用于Action绑定）
    logger.Info("Setting {0}'s ActionTarget to {1}", view, viewModel);
    View.SetActionTarget(view, viewModel);

    // 2. 设置DataContext（用于数据绑定）
    if (view is FrameworkElement viewAsFrameworkElement)
    {
        logger.Info("Setting {0}'s DataContext to {1}", view, viewModel);
        viewAsFrameworkElement.DataContext = viewModel;
    }

    // 3. 如果ViewModel实现了IViewAware，通知其View已附加
    if (viewModel is IViewAware viewModelAsViewAware)
    {
        logger.Info("Setting {0}'s View to {1}", viewModel, view);
        viewModelAsViewAware.AttachView(view);
    }
}
```

### 6. Content属性的动态设置

```csharp
// 文件：Stylet\Xaml\View.cs
public static void SetContentProperty(DependencyObject targetLocation, UIElement view)
{
    Type type = targetLocation.GetType();
    
    // 1. 获取Content属性（通过ContentPropertyAttribute或默认"Content"）
    ContentPropertyAttribute attribute = type.GetCustomAttribute<ContentPropertyAttribute>();
    string propertyName = attribute != null ? attribute.Name : "Content";
    
    // 2. 通过反射设置Content属性值
    PropertyInfo property = type.GetProperty(propertyName);
    if (property == null)
        throw new InvalidOperationException($"在类型 {type.Name} 上找不到Content属性");
    
    property.SetValue(targetLocation, view);
}
```

## 完整工作流程示例

让我们以`<ContentControl s:View.Model="{Binding HeaderViewModel}"/>`为例，追踪完整的工作流程：

### 步骤1：XAML解析与绑定建立
```xml
<ContentControl DockPanel.Dock="Top" s:View.Model="{Binding HeaderViewModel}"/>
```

1. XAML解析器识别`s:View.Model`为附加属性
2. 建立数据绑定：`{Binding HeaderViewModel}`
3. 当ShellViewModel的HeaderViewModel属性有值时，触发绑定更新

### 步骤2：Model属性变化回调
```csharp
// View.ModelProperty的PropertyChangedCallback被触发
(d, e) => {
    // d = ContentControl实例
    // e.NewValue = HeaderViewModel实例
    
    // 从应用程序资源获取ViewManager
    IViewManager viewManager = ((FrameworkElement)d).TryFindResource(ViewManagerResourceKey);
    
    // 委托ViewManager处理
    viewManager.OnModelChanged(d, e.OldValue, e.NewValue);
}
```

### 步骤3：ViewManager处理Model变化
```csharp
public virtual void OnModelChanged(DependencyObject targetLocation, object oldValue, object newValue)
{
    // targetLocation = ContentControl实例
    // newValue = HeaderViewModel实例
    
    // 创建并绑定HeaderView
    UIElement view = this.CreateAndBindViewForModelIfNecessary(newValue);
    
    // 设置ContentControl.Content = HeaderView
    View.SetContentProperty(targetLocation, view);
}
```

### 步骤4：View定位与创建
```csharp
// 1. 类型名转换
// "Stylet.Samples.NavigationController.Pages.HeaderViewModel" 
// → "Stylet.Samples.NavigationController.Pages.HeaderView"

// 2. 程序集中查找HeaderView类型
Type viewType = typeof(HeaderView);

// 3. 通过IoC容器创建HeaderView实例
UIElement view = this.ViewFactory(viewType); // new HeaderView()

// 4. 调用HeaderView.InitializeComponent()
this.InitializeView(view, viewType);
```

### 步骤5：绑定建立
```csharp
public virtual void BindViewToModel(UIElement view, object viewModel)
{
    // view = HeaderView实例
    // viewModel = HeaderViewModel实例
    
    // 1. 设置ActionTarget用于Command绑定
    View.SetActionTarget(view, viewModel);
    
    // 2. 设置DataContext用于数据绑定
    ((FrameworkElement)view).DataContext = viewModel;
    
    // 3. 通知ViewModel（如果实现IViewAware）
    if (viewModel is IViewAware viewAware)
        viewAware.AttachView(view);
}
```

### 步骤6：最终结果
```csharp
// ContentControl.Content现在指向HeaderView实例
// HeaderView.DataContext = HeaderViewModel实例
// HeaderView中的绑定开始工作：{s:Action NavigateToPage1} 等
```

## 设计优势

### 1. 约定优于配置
- 通过命名约定自动定位View，减少配置工作
- HeaderViewModel → HeaderView的自动映射

### 2. 松耦合设计
- ViewModel不需要直接引用View
- 通过接口（IViewAware）可选地感知View

### 3. 容器集成
- 通过ViewFactory委托与IoC容器集成
- 支持View的依赖注入

### 4. 设计时支持
- 在设计时显示占位文本，提升开发体验

## 扩展点

### 1. 自定义ViewManager
```csharp
public class CustomViewManager : ViewManager
{
    protected override Type LocateViewForModel(Type modelType)
    {
        // 自定义View定位逻辑
        return base.LocateViewForModel(modelType);
    }
}
```

### 2. 命名空间转换
```csharp
viewManager.NamespaceTransformations.Add("MyApp.ViewModels", "MyApp.Views");
```

### 3. 自定义命名约定
```csharp
viewManager.ViewModelNameSuffix = "VM";
viewManager.ViewNameSuffix = "View";
```

## 总结

Stylet的ViewModelFirst机制通过以下核心组件协同工作：

1. **附加属性系统**：`s:View.Model`提供XAML中的声明式语法
2. **ViewManager**：负责View的定位、创建和绑定逻辑
3. **命名约定**：自动将ViewModel名称转换为对应的View名称
4. **IoC集成**：通过ViewFactory实现View的依赖注入
5. **资源系统**：通过应用程序资源共享ViewManager实例

这种设计使得开发者只需要遵循简单的命名约定，就能实现ViewModel和View的自动关联，大大简化了MVVM模式的实现复杂度。

当我们在XAML中写下`s:View.Model="{Binding HeaderViewModel}"`时，实际上启动了一个复杂而优雅的机制：从属性变化监听，到View定位与创建，再到双向绑定建立，最终实现了声明式的UI组合。
