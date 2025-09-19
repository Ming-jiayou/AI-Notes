# Stylet ViewManager 设计与实现详解

## 概述

ViewManager 是 Stylet 框架中负责管理视图的核心组件，它处理 ViewModel 与 View 之间的映射、创建、初始化和绑定。该类实现了 ViewModel-First 架构模式的核心逻辑，通过约定优于配置的方式自动定位和实例化对应的视图，并处理视图与视图模型之间的绑定关系。

## 设计目标

1. **ViewModel-First 支持**：实现真正的 ViewModel 优先架构
2. **约定优于配置**：通过命名约定自动映射视图和视图模型
3. **灵活扩展**：支持自定义命名转换和视图定位策略
4. **生命周期管理**：正确处理视图的创建、初始化和绑定
5. **错误处理**：提供详细的视图定位错误信息
6. **性能优化**：缓存和优化视图创建过程

## 核心接口设计

### IViewManager 接口

```csharp
public interface IViewManager
{
    void OnModelChanged(DependencyObject targetLocation, object oldValue, object newValue);
    UIElement CreateViewForModel(object model);
    void BindViewToModel(UIElement view, object viewModel);
    UIElement CreateAndBindViewForModelIfNecessary(object model);
}
```

**接口设计原则：**
- **最小化接口**：只暴露核心操作
- **清晰职责**：每个方法有明确的功能边界
- **灵活性**：支持不同的视图管理策略

### ViewManagerConfig 配置类

```csharp
public class ViewManagerConfig
{
    public Func<Type, object> ViewFactory { get; set; }
    public List<Assembly> ViewAssemblies { get; set; }
}
```

**配置分离：**
- 将配置与实现分离
- 支持依赖注入配置
- 提供默认配置选项

## 核心功能详解

### 1. 视图命名转换策略

#### 基本命名规则

```csharp
protected virtual string ViewTypeNameForModelTypeName(string modelTypeName)
{
    string transformed = modelTypeName;

    // 命名空间转换
    foreach (KeyValuePair<string, string> transformation in this.NamespaceTransformations)
    {
        if (transformed.StartsWith(transformation.Key + "."))
        {
            transformed = transformation.Value + transformed.Substring(transformation.Key.Length);
            break;
        }
    }

    // 后缀替换：ViewModel -> View
    transformed = Regex.Replace(transformed,
        string.Format(@"(?<=.){0}(?=s?\.)|{0}$", Regex.Escape(this.ViewModelNameSuffix)),
        Regex.Escape(this.ViewNameSuffix));

    return transformed;
}
```

**转换逻辑：**
1. **命名空间转换**：支持命名空间映射（如 `MyApp.ViewModels` -> `MyApp.Views`）
2. **后缀替换**：将 `ViewModel` 后缀替换为 `View` 后缀
3. **正则表达式**：智能处理各种边界情况

#### 配置属性

```csharp
private string _viewNameSuffix = "View";
private string _viewModelNameSuffix = "ViewModel";

public string ViewNameSuffix { get; set; }
public string ViewModelNameSuffix { get; set; }
```

**可定制性：**
- 支持自定义后缀
- 适应不同的命名约定
- 保持向后兼容性

### 2. 视图定位机制

#### 视图类型定位

```csharp
protected virtual Type LocateViewForModel(Type modelType)
{
    string modelName = modelType.FullName;
    string viewName = this.ViewTypeNameForModelTypeName(modelName);
    if (modelName == viewName)
        throw new StyletViewLocationException(string.Format("Unable to transform ViewModel name {0} into a suitable View name", modelName), viewName);

    // 包含 ViewModel 的程序集，提供额外帮助
    Type viewType = this.ViewTypeForViewName(viewName, new[] { modelType.Assembly });
    if (viewType == null)
    {
        var e = new StyletViewLocationException(string.Format("Unable to find a View with type {0}", viewName), viewName);
        logger.Error(e);
        throw e;
    }
    else
    {
        logger.Info("Searching for a View with name {0}, and found {1}", viewName, viewType);
    }

    return viewType;
}
```

**定位策略：**
1. **类型名称转换**：将 ViewModel 类型名转换为 View 类型名
2. **多程序集搜索**：在配置的程序集中搜索视图类型
3. **错误处理**：提供详细的错误信息和日志记录
4. **性能考虑**：找到第一个匹配项即返回

#### 视图类型搜索

```csharp
protected virtual Type ViewTypeForViewName(string viewName, IEnumerable<Assembly> extraAssemblies)
{
    return this.ViewAssemblies.Concat(extraAssemblies).Select(x => x.GetType(viewName)).FirstOrDefault(x => x != null);
}
```

**搜索逻辑：**
- 遍历所有配置的程序集
- 使用 Type.GetType 进行类型查找
- 优先搜索主程序集，再搜索额外程序集

### 3. 视图创建和初始化

#### 视图创建流程

```csharp
public virtual UIElement CreateViewForModel(object model)
{
    Type viewType = this.LocateViewForModel(model.GetType());

    if (viewType.IsAbstract || !typeof(UIElement).IsAssignableFrom(viewType))
    {
        var e = new StyletViewLocationException(string.Format("Found type for view: {0}, but it wasn't a class derived from UIElement", viewType.Name), viewType.Name);
        logger.Error(e);
        throw e;
    }

    var view = (UIElement)this.ViewFactory(viewType);

    this.InitializeView(view, viewType);

    return view;
}
```

**创建步骤：**
1. **类型定位**：找到对应的视图类型
2. **类型验证**：确保类型是具体的 UIElement 派生类
3. **实例创建**：使用 ViewFactory 创建实例
4. **视图初始化**：调用 InitializeComponent 方法

#### 视图初始化

```csharp
public virtual void InitializeView(UIElement view, Type viewType)
{
    // 如果视图没有代码隐藏，这个方法不会被调用
    // 必须使用反射，因为 InitializeComponent 是视图的方法，而不是其基类的方法
    MethodInfo initializer = viewType.GetMethod("InitializeComponent", BindingFlags.Public | BindingFlags.Instance);
    if (initializer != null)
        initializer.Invoke(view, null);
}
```

**初始化逻辑：**
- 使用反射查找 InitializeComponent 方法
- 仅在方法存在时调用
- 支持 XAML 视图的自动初始化

### 4. 视图-模型绑定

#### 绑定流程

```csharp
public virtual void BindViewToModel(UIElement view, object viewModel)
{
    logger.Info("Setting {0}'s ActionTarget to {1}", view, viewModel);
    View.SetActionTarget(view, viewModel);

    if (view is FrameworkElement viewAsFrameworkElement)
    {
        logger.Info("Setting {0}'s DataContext to {1}", view, viewModel);
        viewAsFrameworkElement.DataContext = viewModel;
    }

    if (viewModel is IViewAware viewModelAsViewAware)
    {
        logger.Info("Setting {0}'s View to {1}", viewModel, view);
        viewModelAsViewAware.AttachView(view);
    }
}
```

**绑定步骤：**
1. **ActionTarget 设置**：设置命令和动作的目标
2. **DataContext 绑定**：建立数据绑定上下文
3. **视图关联**：通知视图模型视图已附加

#### 智能绑定策略

```csharp
public virtual UIElement CreateAndBindViewForModelIfNecessary(object model)
{
    if (model is IViewAware modelAsViewAware && modelAsViewAware.View != null)
    {
        logger.Info("ViewModel {0} already has a View attached to it. Not attaching another", model);
        return modelAsViewAware.View;
    }

    return this.CreateAndBindViewForModel(model);
}
```

**优化策略：**
- 检查是否已存在关联视图
- 避免重复创建和绑定
- 支持视图重用场景

### 5. 模型变更处理

#### 模型变更响应

```csharp
public virtual void OnModelChanged(DependencyObject targetLocation, object oldValue, object newValue)
{
    if (oldValue == newValue)
        return;

    if (newValue != null)
    {
        logger.Info("View.Model changed for {0} from {1} to {2}", targetLocation, oldValue, newValue);
        UIElement view = this.CreateAndBindViewForModelIfNecessary(newValue);
        if (view is Window)
        {
            var e = new StyletInvalidViewTypeException(string.Format("s:View.Model=\"...\" tried to show a View of type '{0}', but that View derives from the Window class. " +
            "Make sure any Views you display using s:View.Model=\"...\" do not derive from Window (use UserControl or similar)", view.GetType().Name));
            logger.Error(e);
            throw e;
        }
        View.SetContentProperty(targetLocation, view);
    }
    else
    {
        logger.Info("View.Model cleared for {0}, from {1}", targetLocation, oldValue);
        View.SetContentProperty(targetLocation, null);
    }
}
```

**处理逻辑：**
- **值比较**：避免不必要的重复处理
- **视图创建**：为新模型创建和绑定视图
- **类型检查**：防止 Window 类型视图的非法使用
- **内容设置**：使用 SetContentProperty 设置视图内容
- **清理处理**：正确处理模型清空的情况

### 6. 命名空间转换

#### 转换配置

```csharp
private Dictionary<string, string> _namespaceTransformations = new();

public Dictionary<string, string> NamespaceTransformations { get; set; }
```

**转换机制：**
- 支持命名空间映射配置
- 在类型名称转换时应用
- 支持复杂的命名空间重构场景

#### 转换示例

```csharp
// 配置命名空间转换
viewManager.NamespaceTransformations["MyApp.ViewModels"] = "MyApp.Views";
viewManager.NamespaceTransformations["MyApp.Core.ViewModels"] = "MyApp.Presentation.Views";
```

## 异常处理设计

### 自定义异常类型

#### StyletViewLocationException

```csharp
public class StyletViewLocationException : Exception
{
    public readonly string ViewTypeName;

    public StyletViewLocationException(string message, string viewTypeName)
        : base(message)
    {
        this.ViewTypeName = viewTypeName;
    }
}
```

**异常信息：**
- 包含视图类型名称
- 提供详细的错误上下文
- 支持调试和错误诊断

#### StyletInvalidViewTypeException

```csharp
public class StyletInvalidViewTypeException : Exception
{
    public StyletInvalidViewTypeException(string message)
        : base(message)
    { }
}
```

**用途：**
- 处理视图类型不合法的情况
- 防止 Window 类型被错误使用
- 提供明确的错误指导

## 配置和使用

### 基本配置

```csharp
var config = new ViewManagerConfig
{
    ViewFactory = container.GetInstance,
    ViewAssemblies = new List<Assembly> { Assembly.GetExecutingAssembly() }
};

var viewManager = new ViewManager(config);
```

### 高级配置

```csharp
viewManager.ViewNameSuffix = "View";
viewManager.ViewModelNameSuffix = "ViewModel";
viewManager.NamespaceTransformations["ViewModels"] = "Views";
```

### 使用示例

```csharp
// 创建视图
var view = viewManager.CreateViewForModel(viewModel);

// 绑定现有视图
viewManager.BindViewToModel(view, viewModel);

// 处理模型变更
viewManager.OnModelChanged(targetElement, oldModel, newModel);
```

## 设计优势

### 1. 约定优于配置
- **自动映射**：通过命名约定自动映射视图和视图模型
- **可定制性**：支持自定义命名规则和转换
- **减少配置**：最小化手动配置需求

### 2. 性能优化
- **视图重用**：智能检查已存在的视图关联
- **缓存机制**：避免重复的类型查找和创建
- **增量处理**：只处理实际变更的部分

### 3. 错误处理
- **详细异常**：提供丰富的错误信息和上下文
- **日志记录**：完整的操作日志便于调试
- **类型验证**：确保视图类型的合法性

### 4. 灵活性
- **多程序集支持**：支持跨程序集的视图查找
- **命名空间转换**：支持复杂的项目结构
- **自定义工厂**：支持自定义视图创建逻辑

## 最佳实践

### 1. 命名约定
- **一致性**：保持项目中命名约定的一致性
- **清晰性**：使用清晰的命名后缀
- **可预测性**：确保命名转换的可预测性

### 2. 程序集组织
- **逻辑分组**：按功能模块组织视图和视图模型
- **依赖管理**：最小化程序集间的依赖
- **性能考虑**：合理控制程序集数量

### 3. 错误处理
- **异常处理**：适当处理视图定位异常
- **日志记录**：利用日志进行调试和监控
- **用户反馈**：向用户提供友好的错误信息

### 4. 性能优化
- **视图缓存**：在适当场景下重用视图
- **延迟加载**：考虑视图的延迟创建
- **资源管理**：正确处置不再使用的视图

## 总结

ViewManager 是 Stylet 框架中实现 ViewModel-First 架构的核心组件，它通过智能的命名约定、灵活的配置选项和 robust 的错误处理，为 WPF MVVM 应用程序提供了强大的视图管理功能。其设计充分体现了约定优于配置的原则，同时保持了足够的灵活性来适应不同的项目需求。通过完善的日志记录、异常处理和性能优化，ViewManager 为构建可维护、可扩展的 MVVM 应用程序提供了坚实的基础。