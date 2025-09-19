# StyletIoCBuilder 接口与类详解

## 概述

`StyletIoCBuilder.cs` 文件定义了 StyletIoC 容器的构建器接口 `IStyletIoCBuilder` 和其实现类 `StyletIoCBuilder`。这个构建器是创建和配置 `IContainer` 实例的唯一方式，通过它开发者可以注册各种绑定，然后生成一个可用的 IoC 容器。

## IStyletIoCBuilder 接口

```csharp
public interface IStyletIoCBuilder
{
    List<Assembly> Assemblies { get; set; }
    IBindTo Bind(Type serviceType);
    IBindTo Bind<TService>();
    void Autobind(IEnumerable<Assembly> assemblies);
    void Autobind(params Assembly[] assemblies);
    void AddModule(StyletIoCModule module);
    void AddModules(params StyletIoCModule[] modules);
    IContainer BuildContainer();
}
```

### 接口成员详解

#### Assemblies 属性

```csharp
List<Assembly> Assemblies { get; set; }
```

**用途**：获取或设置由 `Autobind` 和 `ToAllImplementations` 搜索的程序集列表。

**说明**：
- 这是一个程序集列表，用于自动绑定和发现实现
- 默认情况下包含调用程序集
- 可以通过此属性添加或移除程序集，以控制自动绑定的范围

#### Bind 方法

```csharp
IBindTo Bind(Type serviceType);
IBindTo Bind<TService>();
```

**用途**：绑定指定的服务（接口、抽象类、具体类、未绑定泛型等）到某个实现。

**参数**：
- `serviceType` (Type): 要绑定的服务类型
- `TService` (泛型): 要绑定的服务类型

**返回值**：
- 返回一个 `IBindTo` 接口，用于继续配置绑定的详细信息

**说明**：
- 这是注册依赖注入绑定的主要方法
- 支持绑定各种类型的服务，包括接口、抽象类、具体类等
- 泛型版本提供了类型安全的绑定方式
- 返回的 `IBindTo` 接口允许链式调用，进一步配置绑定细节

#### Autobind 方法

```csharp
void Autobind(IEnumerable<Assembly> assemblies);
void Autobind(params Assembly[] assemblies);
```

**用途**：在指定的程序集（或当前程序集）中搜索具体类型，并自绑定它们。

**参数**：
- `assemblies` (IEnumerable<Assembly> 或 Assembly[]): 要搜索的程序集，如果为空或 null，则搜索当前程序集

**说明**：
- 自动发现指定程序集中的所有具体类型并自绑定
- 如果之前调用过 `Autobind`，则会将新的程序集集合添加到现有集合中
- 参数数组版本提供了更方便的调用方式
- 常用于快速注册程序集中的所有类型

#### AddModule/AddModules 方法

```csharp
void AddModule(StyletIoCModule module);
void AddModules(params StyletIoCModule[] modules);
```

**用途**：向构建器添加一个或多个模块。

**参数**：
- `module` (StyletIoCModule): 要添加的模块
- `modules` (StyletIoCModule[]): 要添加的模块数组

**说明**：
- 模块是组织相关绑定的方式
- 允许将绑定配置分组到不同的模块中，提高代码的组织性
- `AddModules` 方法可以一次性添加多个模块
- 模块会自动调用其 `AddToBuilder` 方法来注册绑定

#### BuildContainer 方法

```csharp
IContainer BuildContainer();
```

**用途**：在所有绑定设置完成后，构建一个 `IContainer`，从中可以获取实例。

**返回值**：
- 一个完全配置好的 `IContainer` 实例

**说明**：
- 这是构建过程的最后一步，生成可用的 IoC 容器
- 一旦调用了此方法，构建器的配置阶段就结束了
- 返回的容器应该用于后续的服务解析操作
- 此方法会处理弱绑定和强绑定的冲突，确保正确的绑定被应用

## StyletIoCBuilder 类

```csharp
public class StyletIoCBuilder : IStyletIoCBuilder
{
    private readonly List<BuilderBindTo> bindings = new();
    private List<Assembly> autobindAssemblies;
    
    public List<Assembly> Assemblies { get; set; }
    
    public StyletIoCBuilder();
    public StyletIoCBuilder(params StyletIoCModule[] modules);
    
    public IBindTo Bind(Type serviceType);
    public IBindTo Bind<TService>();
    public void Autobind(IEnumerable<Assembly> assemblies);
    public void Autobind(params Assembly[] assemblies);
    public void AddModule(StyletIoCModule module);
    public void AddModules(params StyletIoCModule[] modules);
    public IContainer BuildContainer();
    
    internal void AddBinding(BuilderBindTo binding);
    private IEnumerable<Assembly> GetAssemblies(IEnumerable<Assembly> extras, string methodName);
}
```

### 类成员详解

#### 字段

```csharp
private readonly List<BuilderBindTo> bindings = new();
private List<Assembly> autobindAssemblies;
```

**bindings**：
- 存储所有通过 `Bind` 方法注册的绑定
- 是 `BuilderBindTo` 对象的列表，每个对象代表一个绑定配置

**autobindAssemblies**：
- 存储通过 `Autobind` 方法指定的程序集
- 用于自动绑定过程中的程序集搜索

#### 属性

```csharp
public List<Assembly> Assemblies { get; set; }
```

- 实现了接口中的 `Assemblies` 属性
- 默认构造函数中初始化为包含调用程序集的列表

#### 构造函数

```csharp
public StyletIoCBuilder();
public StyletIoCBuilder(params StyletIoCModule[] modules);
```

**无参构造函数**：
- 初始化一个新的 `StyletIoCBuilder` 实例
- 将 `Assemblies` 属性设置为包含调用程序集的列表

**带模块参数的构造函数**：
- 初始化一个新的 `StyletIoCBuilder` 实例，并包含给定的模块
- 调用无参构造函数，然后添加指定的模块

#### 方法实现

##### Bind 方法

```csharp
public IBindTo Bind(Type serviceType)
{
    var builderBindTo = new BuilderBindTo(serviceType, this.GetAssemblies);
    this.bindings.Add(builderBindTo);
    return builderBindTo;
}

public IBindTo Bind<TService>()
{
    return this.Bind(typeof(TService));
}
```

**实现细节**：
- 创建一个新的 `BuilderBindTo` 实例，传入服务类型和程序集获取方法
- 将绑定添加到内部列表中
- 返回绑定对象以支持链式调用
- 泛型版本简单地调用非泛型版本

##### Autobind 方法

```csharp
public void Autobind(IEnumerable<Assembly> assemblies)
{
    IEnumerable<Assembly> existing = this.autobindAssemblies ?? Enumerable.Empty<Assembly>();
    this.autobindAssemblies = existing.Concat(this.GetAssemblies(assemblies, "Autobind")).Distinct().ToList();
}

public void Autobind(params Assembly[] assemblies)
{
    this.Autobind(assemblies.AsEnumerable());
}
```

**实现细节**：
- 如果之前调用过 `Autobind`，则将新的程序集集合添加到现有集合中
- 使用 `GetAssemblies` 方法获取最终要搜索的程序集列表
- 确保程序集列表中没有重复项
- 参数数组版本将数组转换为 `IEnumerable<Assembly>` 并调用另一个版本

##### AddModule/AddModules 方法

```csharp
public void AddModule(StyletIoCModule module)
{
    module.AddToBuilder(this, this.GetAssemblies);
}

public void AddModules(params StyletIoCModule[] modules)
{
    foreach (StyletIoCModule module in modules)
    {
        this.AddModule(module);
    }
}
```

**实现细节**：
- 调用模块的 `AddToBuilder` 方法，传入构建器实例和程序集获取方法
- `AddModules` 方法遍历所有模块并逐个添加

##### BuildContainer 方法

```csharp
public IContainer BuildContainer()
{
    var container = new Container(this.autobindAssemblies);

    // Just in case they want it
    var bindings = this.bindings.ToList();
    var containerBuilderBindTo = new BuilderBindTo(typeof(IContainer), this.GetAssemblies);
    containerBuilderBindTo.ToInstance(container).DisposeWithContainer(false).AsWeakBinding();
    bindings.Add(containerBuilderBindTo);

    // For each binding which is weak, if another binding exists with any of the same type+key which is strong, we remove this binding
    var groups = (from binding in bindings
                  from serviceType in binding.ServiceTypes
                  select new { ServiceType = serviceType, Binding = binding })
                  .ToLookup(x => x.ServiceType);

    IEnumerable<BuilderBindTo> filtered = from binding in bindings
                   where !(binding.IsWeak &&
                        binding.ServiceTypes.Any(serviceType => groups.Contains(serviceType) && groups[serviceType].Any(groupItem => !groupItem.Binding.IsWeak)))
                   select binding;

    foreach (BuilderBindTo binding in filtered)
    {
        binding.Build(container);
    }
    return container;
}
```

**实现细节**：
1. 创建一个新的 `Container` 实例，传入自动绑定程序集列表
2. 将容器自身注册为一个弱绑定，以便可以在需要时解析容器实例
3. 处理弱绑定和强绑定的冲突：
   - 对于每个弱绑定，如果存在相同类型+键的强绑定，则移除该弱绑定
   - 使用 LINQ 查询来过滤掉冲突的弱绑定
4. 将所有过滤后的绑定构建到容器中
5. 返回完全配置好的容器

##### 内部方法

```csharp
internal void AddBinding(BuilderBindTo binding)
{
    this.bindings.Add(binding);
}

private IEnumerable<Assembly> GetAssemblies(IEnumerable<Assembly> extras, string methodName)
{
    IEnumerable<Assembly> assemblies = this.Assemblies ?? Enumerable.Empty<Assembly>();
    if (extras != null)
        assemblies = assemblies.Concat(extras);
    if (!assemblies.Any())
        throw new StyletIoCRegistrationException(string.Format("{0} called but Assemblies is empty, and no extra assemblies given", methodName));
    return assemblies.Distinct();
}
```

**AddBinding**：
- 内部方法，用于向绑定列表添加新的绑定
- 可能被模块或其他内部组件使用

**GetAssemblies**：
- 私有方法，用于获取要搜索的程序集列表
- 组合 `Assemblies` 属性和额外的程序集参数
- 确保至少有一个程序集可用，否则抛出异常
- 移除重复的程序集

## 使用示例

### 基本绑定

```csharp
// 创建构建器
var builder = new StyletIoCBuilder();

// 绑定接口到实现
builder.Bind<IMyService>().To<MyService>();

// 自绑定具体类型
builder.Bind<MyService>().ToSelf();

// 构建容器
var container = builder.BuildContainer();
```

### 使用泛型方法

```csharp
var builder = new StyletIoCBuilder();

// 使用泛型方法进行绑定
builder.Bind<IMyService>().To<MyService>();
builder.Bind<MyRepository>().ToSelf();

var container = builder.BuildContainer();
```

### 自动绑定

```csharp
var builder = new StyletIoCBuilder();

// 自动绑定当前程序集中的所有具体类型
builder.Autobind();

// 自动绑定指定程序集中的所有具体类型
builder.Autobind(Assembly.GetAssembly(typeof(MyService)));

var container = builder.BuildContainer();
```

### 使用模块

```csharp
// 定义模块
public class MyModule : StyletIoCModule
{
    protected override void Load()
    {
        this.Bind<IMyService>().To<MyService>().InSingletonScope();
        this.Bind<MyRepository>().ToSelf();
    }
}

// 使用模块
var builder = new StyletIoCBuilder(new MyModule());

// 或者添加多个模块
var builder = new StyletIoCBuilder();
builder.AddModules(new MyModule1(), new MyModule2());

var container = builder.BuildContainer();
```

### 配置程序集

```csharp
var builder = new StyletIoCBuilder();

// 配置要搜索的程序集
builder.Assemblies.Add(Assembly.GetAssembly(typeof(MyService)));
builder.Assemblies.Add(Assembly.GetAssembly(typeof(MyRepository)));

// 自动绑定会搜索这些程序集
builder.Autobind();

var container = builder.BuildContainer();
```

### 复杂绑定配置

```csharp
var builder = new StyletIoCBuilder();

// 带键的绑定
builder.Bind<IMyService>().To<MyService1>().WithKey("Service1");
builder.Bind<IMyService>().To<MyService2>().WithKey("Service2");

// 工厂绑定
builder.Bind<IMyService>().ToFactory(c => new MyService(c.Get<IDependency>()));

// 单例作用域
builder.Bind<IMyService>().To<MyService>().InSingletonScope();

// 弱绑定
builder.Bind<IMyService>().To<MyDefaultService>().AsWeakBinding();

var container = builder.BuildContainer();
```

## 设计特点

### 1. 流畅接口设计

- 通过 `IBindTo` 接口支持链式调用
- 使绑定配置代码更加清晰和易读
- 提供了直观的 API 来配置复杂的依赖关系

### 2. 灵活的程序集管理

- 支持配置要搜索的程序集列表
- 允许动态添加程序集
- 为自动绑定和实现发现提供基础

### 3. 模块化支持

- 通过模块组织相关绑定
- 支持添加多个模块
- 提高代码的组织性和可维护性

### 4. 弱绑定机制

- 支持弱绑定和强绑定
- 自动处理绑定冲突
- 允许框架提供默认绑定，同时允许用户覆盖

### 5. 类型安全

- 提供泛型方法版本
- 减少运行时类型转换错误
- 提高代码的可读性和可维护性

## 最佳实践

### 1. 使用模块组织绑定

```csharp
public class ServicesModule : StyletIoCModule
{
    protected override void Load()
    {
        this.Bind<IMyService>().To<MyService>();
        this.Bind<IRepository>().To<Repository>();
    }
}

public class ViewModelModule : StyletIoCModule
{
    protected override void Load()
    {
        this.Bind<MainViewModel>().ToSelf();
        this.Bind<SettingsViewModel>().ToSelf();
    }
}

var builder = new StyletIoCBuilder();
builder.AddModules(new ServicesModule(), new ViewModelModule());
```

### 2. 合理使用自动绑定

```csharp
// 对于小型项目，可以使用自动绑定
var builder = new StyletIoCBuilder();
builder.Autobind();

// 对于大型项目，建议使用显式绑定
var builder = new StyletIoCBuilder();
builder.Bind<IMyService>().To<MyService>();
builder.Bind<IRepository>().To<Repository>();
```

### 3. 配置适当的程序集

```csharp
var builder = new StyletIoCBuilder();

// 只包含必要的程序集
builder.Assemblies.Add(Assembly.GetExecutingAssembly());
builder.Assemblies.Add(Assembly.GetAssembly(typeof(IMyService)));

// 避免包含过多不必要的程序集，以提高性能
```

### 4. 使用弱绑定提供默认实现

```csharp
var builder = new StyletIoCBuilder();

// 框架提供默认实现（弱绑定）
builder.Bind<IMyService>().To<DefaultMyService>().AsWeakBinding();

// 用户可以覆盖默认实现
builder.Bind<IMyService>().To<CustomMyService>();
```

## 总结

`StyletIoCBuilder` 是 StyletIoC 容器的核心构建器，提供了完整的依赖注入配置功能。通过流畅的接口设计、模块化支持和灵活的绑定机制，开发者可以轻松配置复杂的依赖关系。合理使用这个构建器，可以构建出松耦合、易于测试和维护的应用程序架构。

构建器的主要职责是收集和配置绑定，然后生成一个完全配置好的容器实例。通过理解其设计和工作原理，开发者可以更有效地使用 StyletIoC 框架，实现优雅的依赖注入解决方案。