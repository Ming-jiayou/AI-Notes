# Stylet ApplicationLoader 类详解

## 概述

`ApplicationLoader.cs` 文件定义了 `ApplicationLoader` 类，这是 Stylet 框架中的一个重要组件，用于在 WPF 应用程序中加载 Bootstrapper 和 Stylet 的其他资源。该类继承自 `ResourceDictionary`，通常被添加到应用程序的 `App.xaml` 文件中，负责初始化 Stylet 框架并设置应用程序的启动逻辑。

## 类定义

```csharp
public class ApplicationLoader : ResourceDictionary
{
    // 字段、属性和方法实现...
}
```

## 字段详解

### styletResourceDictionary

```csharp
private readonly ResourceDictionary styletResourceDictionary;
```

**用途**：存储 Stylet 框架自身的资源字典。

**说明**：
- 这是一个只读字段，在构造函数中初始化
- 指向 Stylet 框架内置的资源字典文件 `StyletResourceDictionary.xaml`
- 包含 Stylet 框架定义的各种资源，如样式、模板、转换器等
- 通过 `pack://application:,,,/Stylet;component/Xaml/StyletResourceDictionary.xaml` URI 加载

## 属性详解

### Bootstrapper

```csharp
private IBootstrapper _bootstrapper;

public IBootstrapper Bootstrapper
{
    get => this._bootstrapper;
    set
    {
        this._bootstrapper = value;
        this._bootstrapper.Setup(Application.Current);
    }
}
```

**用途**：获取或设置用于启动应用程序的 Bootstrapper 实例。

**类型**：`IBootstrapper`

**说明**：
- 这是 `ApplicationLoader` 的核心属性，必须设置
- 当设置该属性时，会自动调用 Bootstrapper 的 `Setup` 方法，传入当前应用程序实例
- `Setup` 方法负责初始化 Stylet 框架，设置依赖注入容器，创建主视图模型等
- 通过这个属性，开发者可以指定自定义的 Bootstrapper 实现来控制应用程序的启动逻辑

### LoadStyletResources

```csharp
private bool _loadStyletResources;

public bool LoadStyletResources
{
    get => this._loadStyletResources;
    set
    {
        this._loadStyletResources = value;
        if (this._loadStyletResources)
            this.MergedDictionaries.Add(this.styletResourceDictionary);
        else
            this.MergedDictionaries.Remove(this.styletResourceDictionary);
    }
}
```

**用途**：获取或设置是否加载 Stylet 自身的资源。

**类型**：`bool`

**默认值**：`true`

**说明**：
- 控制是否将 Stylet 框架的资源字典添加到应用程序的合并资源字典中
- 当设置为 `true` 时，将 `styletResourceDictionary` 添加到 `MergedDictionaries`
- 当设置为 `false` 时，从 `MergedDictionaries` 中移除 `styletResourceDictionary`
- 默认值为 `true`，确保 Stylet 的资源（如 `StyletConductorTabControl`）可用
- 允许开发者选择是否加载 Stylet 的内置资源，在某些特殊情况下可能需要禁用

## 方法详解

### 构造函数

```csharp
public ApplicationLoader()
{
    this.styletResourceDictionary = new ResourceDictionary() { Source = new Uri("pack://application:,,,/Stylet;component/Xaml/StyletResourceDictionary.xaml", UriKind.Absolute) };
    this.LoadStyletResources = true;
}
```

**用途**：初始化 `ApplicationLoader` 类的新实例。

**说明**：
- 创建并初始化 `styletResourceDictionary`，设置其源为 Stylet 框架的资源字典文件
- 使用 pack URI 格式加载嵌入在 Stylet 程序集中的 XAML 资源
- 将 `LoadStyletResources` 属性设置为默认值 `true`
- 构造函数在 XAML 解析器创建 `ApplicationLoader` 实例时自动调用

## 使用方式

### 在 App.xaml 中使用

```xml
<Application x:Class="MyApp.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:stylet="clr-namespace:Stylet.Xaml;assembly=Stylet">
    <Application.Resources>
        <ResourceDictionary>
            <ResourceDictionary.MergedDictionaries>
                <stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" />
            </ResourceDictionary.MergedDictionaries>
        </ResourceDictionary>
    </Application.Resources>
</Application>
```

**说明**：
- 在 `App.xaml` 的资源部分添加 `ApplicationLoader`
- 通过 `Bootstrapper` 属性指定应用程序的 Bootstrapper 实例
- 通常使用 `x:Static` 标记扩展引用 Bootstrapper 的单例实例
- `ApplicationLoader` 会自动加载 Stylet 资源并初始化 Bootstrapper

### 自定义 Bootstrapper

```csharp
public class Bootstrapper : Bootstrapper<ShellViewModel>
{
    // 自定义 Bootstrapper 实现
}
```

```xml
<stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" />
```

**说明**：
- 创建自定义 Bootstrapper 类，继承自 `Bootstrapper<T>` 其中 T 是主视图模型类型
- 在 Bootstrapper 中可以配置依赖注入、服务、视图模型等
- 在 XAML 中将 `ApplicationLoader` 的 `Bootstrapper` 属性设置为自定义 Bootstrapper 实例

### 控制资源加载

```xml
<!-- 加载 Stylet 资源（默认行为） -->
<stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" LoadStyletResources="true" />

<!-- 不加载 Stylet 资源 -->
<stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" LoadStyletResources="false" />
```

**说明**：
- 通过 `LoadStyletResources` 属性控制是否加载 Stylet 的内置资源
- 如果不需要 Stylet 的内置控件和样式，可以设置为 `false`
- 设置为 `false` 可能会减少应用程序的内存使用，但会失去 Stylet 提供的 UI 功能

## 设计特点

### 1. 资源字典继承

- 继承自 `ResourceDictionary`，使其可以自然地集成到 WPF 的资源系统中
- 可以作为资源字典直接添加到 `App.xaml` 中
- 通过 `MergedDictionaries` 管理子资源字典

### 2. 自动初始化

- 通过属性设置自动触发 Bootstrapper 的初始化
- 无需在代码中显式调用初始化方法
- 利用 WPF 的属性系统实现声明式配置

### 3. 灵活的资源管理

- 允许开发者选择是否加载 Stylet 的内置资源
- 提供默认行为，同时支持自定义
- 通过简单的布尔值控制资源加载

### 4. 声明式配置

- 通过 XAML 声明式地配置应用程序启动逻辑
- 减少代码中的初始化代码
- 提高配置的可读性和可维护性

## 工作原理

### 1. XAML 解析阶段

1. WPF 应用程序启动时解析 `App.xaml`
2. 创建 `ApplicationLoader` 实例
3. 调用构造函数，初始化内部资源字典引用

### 2. 属性设置阶段

1. XAML 解析器设置 `Bootstrapper` 属性
2. 属性设置器调用 Bootstrapper 的 `Setup` 方法
3. Bootstrapper 初始化依赖注入容器和其他服务

### 3. 资源加载阶段

1. 根据 `LoadStyletResources` 属性值决定是否加载 Stylet 资源
2. 如果需要加载，将 Stylet 资源字典添加到合并资源字典中
3. Stylet 的资源（如样式、模板、转换器）在应用程序中可用

### 4. 应用程序运行阶段

1. Bootstrapper 创建主视图模型和主视图
2. 应用程序显示主窗口
3. Stylet 框架管理视图模型的生命周期和导航

## 最佳实践

### 1. 使用单例 Bootstrapper

```csharp
public class Bootstrapper : Bootstrapper<ShellViewModel>
{
    public static Bootstrapper Instance { get; } = new Bootstrapper();
    
    // 其他实现...
}
```

```xml
<stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" />
```

**说明**：
- 将 Bootstrapper 实现为单例，确保整个应用程序中只有一个实例
- 使用 `x:Static` 标记扩展引用单例实例
- 避免多次创建 Bootstrapper 实例导致的资源浪费

### 2. 合理使用资源加载

```xml
<!-- 大多数情况下，使用默认设置 -->
<stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" />

<!-- 如果不需要 Stylet 的 UI 组件，可以禁用资源加载 -->
<stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" LoadStyletResources="false" />
```

**说明**：
- 默认情况下启用 Stylet 资源加载，以使用框架提供的 UI 功能
- 如果应用程序不使用 Stylet 的 UI 组件，可以禁用资源加载以减少内存使用
- 根据实际需求选择合适的资源加载策略

### 3. 自定义资源字典

```xml
<Application.Resources>
    <ResourceDictionary>
        <ResourceDictionary.MergedDictionaries>
            <stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" LoadStyletResources="false" />
            <ResourceDictionary Source="pack://application:,,,/MyAssembly;component/Resources/CustomStyles.xaml" />
        </ResourceDictionary.MergedDictionaries>
    </ResourceDictionary>
</Application.Resources>
```

**说明**：
- 如果禁用了 Stylet 资源加载，可以手动添加自定义资源字典
- 允许完全自定义应用程序的样式和模板
- 提供更大的灵活性，同时保持 Stylet 的核心功能

## 常见问题与解决方案

### 1. Bootstrapper 未设置

**问题**：应用程序启动时出现异常，提示 Bootstrapper 未设置。

**解决方案**：
```xml
<!-- 确保 Bootstrapper 属性已正确设置 -->
<stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" />
```

**说明**：
- 确保 `Bootstrapper` 属性已正确设置为有效的 Bootstrapper 实例
- 检查 Bootstrapper 类是否已正确定义并实现为单例
- 确认 XAML 命名空间是否正确引用

### 2. 资源冲突

**问题**：Stylet 资源与应用程序自定义资源发生冲突。

**解决方案**：
```xml
<!-- 禁用 Stylet 资源加载 -->
<stylet:ApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" LoadStyletResources="false" />

<!-- 手动添加需要的资源 -->
<ResourceDictionary Source="pack://application:,,,/Stylet;component/Xaml/SpecificResource.xaml" />
```

**说明**：
- 禁用 Stylet 资源加载，避免资源冲突
- 手动添加特定的 Stylet 资源，只使用需要的部分
- 或者调整资源加载顺序，确保自定义资源覆盖 Stylet 资源

### 3. 启动顺序问题

**问题**：需要在 Bootstrapper 初始化之前执行某些代码。

**解决方案**：
```csharp
public class CustomApplicationLoader : ApplicationLoader
{
    public CustomApplicationLoader()
    {
        // 在构造函数中执行初始化代码
        InitializeCustomServices();
    }
    
    private void InitializeCustomServices()
    {
        // 自定义初始化逻辑
    }
}
```

```xml
<local:CustomApplicationLoader Bootstrapper="{x:Static local:Bootstrapper.Instance}" />
```

**说明**：
- 创建自定义的 `ApplicationLoader` 子类
- 在构造函数中添加自定义初始化逻辑
- 在 XAML 中使用自定义的加载器类

## 总结

`ApplicationLoader` 类是 Stylet 框架与 WPF 应用程序集成的重要组件，它通过声明式的方式简化了框架的初始化过程。该类的主要职责包括：

1. **加载 Bootstrapper**：通过设置 `Bootstrapper` 属性，自动初始化 Stylet 框架和应用程序的启动逻辑。
2. **管理资源**：控制是否加载 Stylet 框架的内置资源，如样式、模板和转换器。
3. **简化配置**：通过 XAML 声明式地配置应用程序，减少代码中的初始化代码。

通过继承 `ResourceDictionary`，`ApplicationLoader` 自然地集成到 WPF 的资源系统中，提供了一种优雅的方式来初始化 Stylet 应用程序。理解其设计和工作原理，有助于开发者更好地使用 Stylet 框架，构建出结构清晰、易于维护的 WPF 应用程序。