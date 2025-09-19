# StyletIoC Container 类详解

## 概述

`Container.cs` 文件定义了 `Container` 类，这是 StyletIoC 容器的核心实现类。该类实现了 `IContainer` 和 `IRegistrationContext` 接口，负责管理依赖注入的注册和解析。作为内部类，它是 StyletIoC 框架的核心组件，提供了完整的服务注册、解析和生命周期管理功能。

## 类定义

```csharp
internal class Container : IContainer, IRegistrationContext
{
    // 字段和实现...
}
```

## 字段详解

### 核心数据结构

```csharp
private readonly List<Assembly> autobindAssemblies;
private readonly ConcurrentDictionary<TypeKey, IRegistrationCollection> registrations = new();
private readonly ConcurrentDictionary<TypeKey, IRegistration> getAllRegistrations = new();
private readonly Dictionary<TypeKey, List<UnboundGeneric>> unboundGenerics = new();
private readonly ConcurrentDictionary<RuntimeTypeHandle, BuilderUpper> builderUppers = new();
```

**autobindAssemblies**：
- 存储用于自动绑定的程序集列表
- 在构造函数中初始化，如果传入 null 则创建空列表
- 用于限制自绑定的范围，避免尝试创建 BCL 类等无法创建的类型

**registrations**：
- 将 [type, key] 对映射到该键对的注册集合
- 使用 `ConcurrentDictionary` 确保线程安全
- 是容器的主要注册存储，可以通过类型和键检索注册
- 每个注册集合包含一个或多个 `IRegistration` 实例

**getAllRegistrations**：
- 将 [type, key] 对映射到可以创建 IEnumerable 实现的注册
- 与 `registrations` 分离，因为某些代码路径（如 Get()）不会搜索它
- 用于处理集合类型的解析，如 `IEnumerable<T>`

**unboundGenerics**：
- 将未绑定泛型（如 `IValidator<>`）映射到可以创建特定类型注册的对象
- 允许处理开放泛型类型的绑定
- 在构建后只进行并发读取，因此使用普通的 `Dictionary` 和 `List`

**builderUppers**：
- 将类型映射到可以构建该类型的 `BuilderUpper`
- `BuilderUpper` 可以创建表达式/委托来构建该类型
- 使用 `ConcurrentDictionary` 确保线程安全

### 辅助组件

```csharp
private AbstractFactoryBuilder abstractFactoryBuilder;
public event EventHandler Disposing;
private bool disposed;
```

**abstractFactoryBuilder**：
- 用于构建抽象工厂的构建器
- 延迟初始化，只有在需要时才创建

**Disposing**：
- 容器被要求释放时触发的事件
- 允许其他组件响应容器的释放事件

**disposed**：
- 标记容器是否已被释放
- 用于防止在释放后使用容器

## 方法详解

### 构造函数

```csharp
public Container(List<Assembly> autobindAssemblies)
{
    this.autobindAssemblies = autobindAssemblies ?? new List<Assembly>();
}
```

**用途**：初始化容器实例。

**参数**：
- `autobindAssemblies` (List<Assembly>): 用于自动绑定的程序集列表，如果为 null 则创建空列表

**说明**：
- 简单的构造函数，主要初始化自动绑定程序集列表
- 不进行其他复杂的初始化，大部分工作在首次使用时完成

### 编译方法

```csharp
public void Compile(bool throwOnError = true)
{
    foreach (IRegistrationCollection value in this.registrations.Values)
    {
        foreach (IRegistration registration in value.GetAll())
        {
            try
            {
                registration.GetGenerator();
            }
            catch (StyletIoCFindConstructorException)
            {
                if (throwOnError)
                    throw;
            }
        }
    }
}
```

**用途**：编译所有已知绑定，检查依赖图的一致性。

**参数**：
- `throwOnError` (bool, 可选, 默认为 true): 如果为 true，则在编译类型失败时抛出异常

**说明**：
- 遍历所有注册，调用其 `GetGenerator()` 方法
- 这会预先编译所有绑定，而不是在需要时才编译
- 可以提前发现依赖注入问题，如构造函数参数无法解析等
- 如果 `throwOnError` 为 false，则在遇到构造函数查找异常时不会抛出

### 服务解析方法

#### Get 方法

```csharp
public object Get(Type type, string key = null)
{
    if (type == null)
        throw new ArgumentNullException(nameof(type));
    return this.Get(type.TypeHandle, key, type);
}

public T Get<T>(string key = null)
{
    return (T)this.Get(typeof(T).TypeHandle, key);
}

private object Get(RuntimeTypeHandle typeHandle, string key = null, Type typeIfAvailable = null)
{
    Func<IRegistrationContext, object> generator = this.GetRegistrations(new TypeKey(typeHandle, key), false, typeIfAvailable).GetSingle().GetGenerator();
    return generator(this);
}
```

**用途**：获取指定类型的单个实例。

**参数**：
- `type` (Type): 要获取的服务类型
- `key` (string, 可选, 默认为 null): 服务注册时使用的键
- `T` (泛型): 要获取的服务类型

**返回值**：
- 请求的服务实例

**说明**：
- 公共方法进行参数验证，然后调用私有实现
- 私有实现使用 `RuntimeTypeHandle` 提高性能
- 通过 `GetRegistrations` 获取注册集合，然后调用 `GetSingle()` 获取单个注册
- 最后调用注册的生成器创建实例
- 泛型版本提供类型安全的解析方式

#### GetAll 方法

```csharp
public IEnumerable<object> GetAll(Type type, string key = null)
{
    if (type == null)
        throw new ArgumentNullException(nameof(type));
    return this.GetAll(type.TypeHandle, key, type);
}

public IEnumerable<T> GetAll<T>(string key = null)
{
    return this.GetAll(typeof(T).TypeHandle, key).Cast<T>();
}

private IEnumerable<object> GetAll(RuntimeTypeHandle typeHandle, string key = null, Type elementTypeIfAvailable = null)
{
    var typeKey = new TypeKey(typeHandle, key);
    IRegistration registration;
    bool result = this.TryRetrieveGetAllRegistrationFromElementType(typeKey, null, out registration, elementTypeIfAvailable);
    Debug.Assert(result);
    Func<IRegistrationContext, object> generator = registration.GetGenerator();
    return (IEnumerable<object>)generator(this);
}
```

**用途**：获取实现指定服务的所有类型的实例。

**参数**：
- `type` (Type): 要获取实现的服务类型
- `key` (string, 可选, 默认为 null): 服务注册时使用的键
- `T` (泛型): 要获取实现的服务类型

**返回值**：
- 请求的服务的所有实现

**说明**：
- 公共方法进行参数验证，然后调用私有实现
- 私有实现使用 `RuntimeTypeHandle` 提高性能
- 通过 `TryRetrieveGetAllRegistrationFromElementType` 获取可以创建集合的注册
- 调用注册的生成器创建集合实例
- 泛型版本提供类型安全的解析方式

#### GetTypeOrAll 方法

```csharp
public object GetTypeOrAll(Type type, string key = null)
{
    if (type == null)
        throw new ArgumentNullException(nameof(type));
    return this.GetTypeOrAll(type.TypeHandle, key, type);
}

public T GetTypeOrAll<T>(string key = null)
{
    return (T)this.GetTypeOrAll(typeof(T).TypeHandle, key);
}

private object GetTypeOrAll(RuntimeTypeHandle typeHandle, string key = null, Type typeIfAvailable = null)
{
    Func<IRegistrationContext, object> generator = this.GetRegistrations(new TypeKey(typeHandle, key), true, typeIfAvailable).GetSingle().GetGenerator();
    return generator(this);
}
```

**用途**：根据类型决定是获取单个实例还是所有实例。

**参数**：
- `type` (Type): 如果是 IEnumerable{T}，将获取 T 的所有实现，否则获取单个 T
- `key` (string, 可选, 默认为 null): 服务注册时使用的键
- `T` (泛型): 如果是 IEnumerable{T}，将获取 T 的所有实现，否则获取单个 T

**返回值**：
- 解析结果

**说明**：
- 智能解析方法，根据请求的类型自动决定使用 Get 还是 GetAll
- 与 `Get` 方法类似，但调用 `GetRegistrations` 时传递 `searchGetAllTypes = true`
- 这允许在找不到单个注册时尝试查找集合注册
- 泛型版本提供类型安全的解析方式

### 对象构建方法

```csharp
public void BuildUp(object item)
{
    BuilderUpper builderUpper = this.GetBuilderUpper(item.GetType());
    builderUpper.GetImplementor()(this, item);
}
```

**用途**：对带有 [Inject] 特性的属性/字段进行依赖注入。

**参数**：
- `item` (object): 要构建的对象

**说明**：
- 获取对象类型的 `BuilderUpper`
- 调用 `BuilderUpper` 的实现器来执行属性/字段注入
- 用于对已存在的对象进行依赖注入，而不是创建新对象

### 注册解析方法

#### CanResolve 方法

```csharp
bool IRegistrationContext.CanResolve(Type type, string key)
{
    IRegistrationCollection registrations;

    if (this.registrations.TryGetValue(new TypeKey(type.TypeHandle, key), out registrations) ||
        this.TryCreateFuncFactory(type, key, out registrations) ||
        this.TryCreateGenericTypesForUnboundGeneric(type, key, out registrations) ||
        this.TryCreateSelfBinding(type, key, out registrations))
    {
        return true;
    }

    IRegistration registration;
    return this.TryRetrieveGetAllRegistration(type, key, out registration);
}
```

**用途**：确定是否可以解析特定类型。

**参数**：
- `type` (Type): 要检查的类型
- `key` (string): 关联的键

**返回值**：
- 如果可以解析该类型，返回 true；否则返回 false

**说明**：
- 实现 `IRegistrationContext` 接口的方法
- 按顺序尝试多种方式解析类型：
  1. 检查现有注册
  2. 尝试创建 Func 工厂
  3. 尝试创建泛型类型
  4. 尝试创建自绑定
  5. 尝试检索集合注册
- 不实际创建实例，只检查是否可以解析

#### GetRegistrations 方法

```csharp
internal IReadOnlyRegistrationCollection GetRegistrations(TypeKey typeKey, bool searchGetAllTypes, Type typeIfAvailable = null)
{
    this.CheckDisposed();

    IReadOnlyRegistrationCollection readOnlyRegistrations;

    IRegistrationCollection registrations;
    if (this.registrations.TryGetValue(typeKey, out registrations))
    {
        readOnlyRegistrations = registrations;
    }
    else
    {
        Type type = typeIfAvailable ?? Type.GetTypeFromHandle(typeKey.TypeHandle);
        if (this.TryCreateFuncFactory(type, typeKey.Key, out registrations) ||
            this.TryCreateGenericTypesForUnboundGeneric(type, typeKey.Key, out registrations) ||
            this.TryCreateSelfBinding(type, typeKey.Key, out registrations))
        {
            readOnlyRegistrations = registrations;
        }
        else if (searchGetAllTypes)
        {
            IRegistration registration;
            if (!this.TryRetrieveGetAllRegistration(type, typeKey.Key, out registration))
                throw new StyletIoCRegistrationException(string.Format("No registrations found for service {0}.", type.GetDescription()));

            readOnlyRegistrations = new SingleRegistration(registration);
        }
        else
        {
            readOnlyRegistrations = new EmptyRegistrationCollection(type);
        }
    }

    return readOnlyRegistrations;
}
```

**用途**：获取指定类型的注册。

**参数**：
- `typeKey` (TypeKey): 类型和键的组合
- `searchGetAllTypes` (bool): 是否搜索集合类型
- `typeIfAvailable` (Type, 可选): 如果可用，对应的类型，用于优化

**返回值**：
- 只读注册集合

**说明**：
- 首先检查容器是否已释放
- 尝试从现有注册中获取
- 如果找不到，尝试动态创建注册：
  1. 尝试创建 Func 工厂
  2. 尝试创建泛型类型
  3. 尝试创建自绑定
- 如果允许搜索集合类型且找到集合注册，则返回包含该注册的集合
- 如果找不到任何注册，返回空注册集合
- 这是容器解析服务的核心方法

### 注册管理方法

#### AddRegistration 方法

```csharp
internal IRegistrationCollection AddRegistration(TypeKey typeKey, IRegistration registration)
{
    try
    {
        return this.registrations.AddOrUpdate(typeKey, x => new SingleRegistration(registration), (x, c) => c.AddRegistration(registration));
    }
    catch (StyletIoCRegistrationException e)
    {
        throw new StyletIoCRegistrationException(string.Format("{0} Service type: {1}, key: '{2}'", e.Message, Type.GetTypeFromHandle(typeKey.TypeHandle).GetDescription(), typeKey.Key), e);
    }
}
```

**用途**：添加注册到容器。

**参数**：
- `typeKey` (TypeKey): 类型和键的组合
- `registration` (IRegistration): 要添加的注册

**返回值**：
- 添加或更新后的注册集合

**说明**：
- 使用 `ConcurrentDictionary.AddOrUpdate` 方法确保线程安全
- 如果键不存在，创建新的 `SingleRegistration`
- 如果键已存在，将新注册添加到现有集合
- 捕获并包装异常，添加更多上下文信息

#### AddUnboundGeneric 方法

```csharp
internal void AddUnboundGeneric(TypeKey typeKey, UnboundGeneric unboundGeneric)
{
    List<UnboundGeneric> unboundGenerics;
    if (!this.unboundGenerics.TryGetValue(typeKey, out unboundGenerics))
    {
        unboundGenerics = new List<UnboundGeneric>();
        this.unboundGenerics.Add(typeKey, unboundGenerics);
    }
    if (unboundGenerics.Any(x => x.Type == unboundGeneric.Type))
        throw new StyletIoCRegistrationException(string.Format("Multiple registrations for type {0} found", Type.GetTypeFromHandle(typeKey.TypeHandle).GetDescription()));

    unboundGenerics.Add(unboundGeneric);
}
```

**用途**：添加未绑定泛型到容器。

**参数**：
- `typeKey` (TypeKey): 类型和键的组合
- `unboundGeneric` (UnboundGeneric): 要添加的未绑定泛型

**说明**：
- 如果键不存在，创建新的列表并添加到字典
- 检查是否已存在相同类型的注册，避免重复
- 将新的未绑定泛型添加到列表
- 此方法只在设置期间调用，不保证线程安全

### 辅助方法

#### GetElementTypeFromCollectionType 方法

```csharp
private Type GetElementTypeFromCollectionType(Type type)
{
    if (!type.IsGenericType || !type.Implements(typeof(IEnumerable<>)))
        return null;
    return type.GenericTypeArguments[0];
}
```

**用途**：从集合类型中提取元素类型。

**参数**：
- `type` (Type): 要检查的集合类型

**返回值**：
- 如果是集合类型，返回元素类型；否则返回 null

**说明**：
- 检查类型是否为泛型类型并实现 `IEnumerable<>`
- 如果是，返回第一个泛型参数（元素类型）
- 用于处理 `IEnumerable<T>` 等集合类型的解析

#### TryRetrieveGetAllRegistrationFromElementType 方法

```csharp
private bool TryRetrieveGetAllRegistrationFromElementType(TypeKey elementTypeKey, Type collectionTypeOrNull, out IRegistration registration, Type elementTypeIfAvailable = null)
{
    if (this.getAllRegistrations.TryGetValue(elementTypeKey, out registration))
        return true;

    Type elementType = elementTypeIfAvailable ?? Type.GetTypeFromHandle(elementTypeKey.TypeHandle);
    Type listType = typeof(List<>).MakeGenericType(elementType);
    if (collectionTypeOrNull != null && !collectionTypeOrNull.IsAssignableFrom(listType))
        return false;

    registration = this.getAllRegistrations.GetOrAdd(elementTypeKey, x => new GetAllRegistration(listType.TypeHandle, this, elementTypeKey.Key));
    return true;
}
```

**用途**：尝试创建或获取可以创建 IEnumerable 实现的注册。

**参数**：
- `elementTypeKey` (TypeKey): 元素类型和键
- `collectionTypeOrNull` (Type, 可选): 如果提供，确保生成的集合实现与此类型兼容
- `registration` (out IRegistration): 返回的注册
- `elementTypeIfAvailable` (Type, 可选): 对应于 elementTypeKey 的类型，用于优化

**返回值**：
- 如果成功创建或获取注册，返回 true；否则返回 false

**说明**：
- 首先尝试从缓存中获取现有注册
- 如果不存在，创建新的 `GetAllRegistration`
- 检查集合类型兼容性
- 使用 `GetOrAdd` 确保线程安全

#### TryCreateSelfBinding 方法

```csharp
private bool TryCreateSelfBinding(Type type, string key, out IRegistrationCollection registrations)
{
    registrations = null;

    if (type.IsAbstract || !type.IsClass)
        return false;

    InjectAttribute injectAttribute = type.GetCustomAttribute<InjectAttribute>(true);
    if (injectAttribute != null && injectAttribute.Key != key)
        return false;

    if (!this.autobindAssemblies.Contains(type.Assembly))
        return false;

    var typeKey = new TypeKey(type.TypeHandle, key);
    registrations = this.AddRegistration(typeKey, new TransientRegistration(new TypeCreator(type, this)));
    return true;
}
```

**用途**：尝试创建自绑定。

**参数**：
- `type` (Type): 要自绑定的类型
- `key` (string): 关联的键
- `registrations` (out IRegistrationCollection): 返回的注册集合

**返回值**：
- 如果成功创建自绑定，返回 true；否则返回 false

**说明**：
- 只对具体类（非抽象、非接口）进行自绑定
- 检查 [Inject] 特性，确保键匹配
- 只对白名单程序集中的类型进行自绑定，避免尝试创建 BCL 类
- 创建瞬态注册，使用 `TypeCreator` 创建实例

#### TryCreateFuncFactory 方法

```csharp
private bool TryCreateFuncFactory(Type type, string key, out IRegistrationCollection registrations)
{
    registrations = null;

    if (!type.IsGenericType)
        return false;

    Type genericType = type.GetGenericTypeDefinition();
    Type[] genericArguments = type.GetGenericArguments();

    if (genericType == typeof(Func<>))
    {
        var typeKey = new TypeKey(type.TypeHandle, key);
        foreach (IRegistration registration in this.GetRegistrations(new TypeKey(genericArguments[0].TypeHandle, key), true, genericArguments[0]).GetAll())
        {
            registrations = this.AddRegistration(typeKey, new FuncRegistration(registration));
        }
    }
    else
    {
        return false;
    }

    return true;
}
```

**用途**：如果给定类型是 Func{T}，获取可以创建其实例的注册。

**参数**：
- `type` (Type): 要检查的类型
- `key` (string): 关联的键
- `registrations` (out IRegistrationCollection): 返回的注册集合

**返回值**：
- 如果成功创建 Func 工厂，返回 true；否则返回 false

**说明**：
- 只处理 `Func<>` 泛型类型
- 获取 Func 参数类型的注册
- 为每个注册创建 `FuncRegistration`
- 允许解析 `Func<T>` 类型的依赖，实现延迟解析

#### TryCreateGenericTypesForUnboundGeneric 方法

```csharp
private bool TryCreateGenericTypesForUnboundGeneric(Type type, string key, out IRegistrationCollection registrations)
{
    registrations = null;

    if (!type.IsGenericType || type.GenericTypeArguments.Length == 0)
        return false;

    Type unboundGenericType = type.GetGenericTypeDefinition();

    List<UnboundGeneric> unboundGenerics;
    if (!this.unboundGenerics.TryGetValue(new TypeKey(unboundGenericType.TypeHandle, key), out unboundGenerics))
        return false;

    foreach (UnboundGeneric unboundGeneric in unboundGenerics)
    {
        Type newType;
        if (unboundGeneric.Type == unboundGenericType)
        {
            newType = type;
        }
        else
        {
            Type implOfUnboundGenericType = unboundGeneric.Type.GetBaseTypesAndInterfaces().Single(x => x.Name == unboundGenericType.Name);
            var mapping = implOfUnboundGenericType.GenericTypeArguments.Zip(type.GenericTypeArguments, (n, t) => new { Type = t, Name = n });

            newType = unboundGeneric.Type.MakeGenericType(unboundGeneric.Type.GetTypeInfo().GenericTypeParameters.Select(x => mapping.Single(t => t.Name.Name == x.Name).Type).ToArray());
        }

        Debug.Assert(type.IsAssignableFrom(newType));

        IRegistration registration = unboundGeneric.CreateRegistrationForTypeAndKey(newType, key);
        registrations = this.AddRegistration(new TypeKey(type.TypeHandle, key), registration);
    }

    return registrations != null;
}
```

**用途**：尝试从未绑定泛型注册创建可以实现给定泛型类型的注册集合。

**参数**：
- `type` (Type): 要创建的泛型类型
- `key` (string): 关联的键
- `registrations` (out IRegistrationCollection): 返回的注册集合

**返回值**：
- 如果成功创建泛型类型注册，返回 true；否则返回 false

**说明**：
- 处理复杂泛型类型映射，如 `IC<T, U>` 到 `C<U, T>`
- 查找未绑定泛型注册
- 为每个未绑定泛型创建特定类型的注册
- 处理泛型参数映射，确保类型兼容性
- 这是 StyletIoC 处理开放泛型的核心方法

### 工厂和构建器方法

#### GetFactoryForType 方法

```csharp
internal Type GetFactoryForType(Type serviceType)
{
    if (this.abstractFactoryBuilder == null)
        this.abstractFactoryBuilder = new AbstractFactoryBuilder();

    return this.abstractFactoryBuilder.GetFactoryForType(serviceType);
}
```

**用途**：获取类型的工厂。

**参数**：
- `serviceType` (Type): 服务类型

**返回值**：
- 工厂类型

**说明**：
- 延迟初始化 `AbstractFactoryBuilder`
- 委托给抽象工厂构建器获取工厂类型
- 用于实现抽象工厂模式

#### GetBuilderUpper 方法

```csharp
public BuilderUpper GetBuilderUpper(Type type)
{
    RuntimeTypeHandle typeHandle = type.TypeHandle;
    return this.builderUppers.GetOrAdd(typeHandle, x => new BuilderUpper(typeHandle, this));
}
```

**用途**：获取类型的 BuilderUpper。

**参数**：
- `type` (Type): 类型

**返回值**：
- 类型的 BuilderUpper

**说明**：
- 使用 `RuntimeTypeHandle` 提高性能
- 使用 `GetOrAdd` 确保线程安全
- `BuilderUpper` 可以创建表达式/委托来构建该类型
- 用于实现属性/字段注入

### 资源管理方法

#### Dispose 方法

```csharp
public void Dispose()
{
    this.disposed = true;
    this.Disposing?.Invoke(this, EventArgs.Empty);
}
```

**用途**：释放容器资源。

**说明**：
- 设置 `disposed` 标志为 true
- 触发 `Disposing` 事件，通知其他组件
- 不释放其他资源，因为容器主要管理对象生命周期，而不是非托管资源

#### CheckDisposed 方法

```csharp
private void CheckDisposed()
{
    if (this.disposed)
        throw new ObjectDisposedException("IContainer");
}
```

**用途**：检查容器是否已被释放。

**说明**：
- 如果容器已释放，抛出 `ObjectDisposedException`
- 在关键操作前调用此方法，确保容器未被释放

## 设计特点

### 1. 线程安全

- 使用 `ConcurrentDictionary` 确保主要数据结构的线程安全
- 使用 `GetOrAdd` 和 `AddOrUpdate` 等原子操作
- 在构建后只进行并发读取的数据结构使用普通集合

### 2. 性能优化

- 使用 `RuntimeTypeHandle` 而不是 `Type` 减少内存分配
- 延迟初始化组件，如 `AbstractFactoryBuilder`
- 缓存注册和构建器，避免重复计算

### 3. 灵活的注册系统

- 支持多种注册类型：单例、瞬态、实例等
- 支持键控注册，允许同一类型的多个实现
- 支持泛型类型和未绑定泛型
- 支持自动绑定和自绑定

### 4. 智能解析

- 自动处理集合类型（如 `IEnumerable<T>`）
- 支持 `Func<T>` 延迟解析
- 智能选择单个或多个实例解析
- 支持属性/字段注入

### 5. 扩展性

- 通过 `IRegistrationContext` 接口提供内部功能
- 支持模块化配置
- 允许自定义注册和创建逻辑

## 使用示例

### 基本服务解析

```csharp
// 创建容器
var builder = new StyletIoCBuilder();
builder.Bind<IMyService>().To<MyService>();
var container = builder.BuildContainer();

// 解析服务
var service = container.Get<IMyService>();
```

### 带键的服务解析

```csharp
// 注册带键的服务
builder.Bind<IMyService>().To<MyService1>().WithKey("Service1");
builder.Bind<IMyService>().To<MyService2>().WithKey("Service2");

// 解析特定键的服务
var service1 = container.Get<IMyService>("Service1");
var service2 = container.Get<IMyService>("Service2");
```

### 解析多个实现

```csharp
// 注册多个实现
builder.Bind<IMyService>().To<MyService1>();
builder.Bind<IMyService>().To<MyService2>();

// 解析所有实现
var allServices = container.GetAll<IMyService>();
```

### 对象构建

```csharp
// 创建对象
var viewModel = new MyViewModel();

// 进行属性注入
container.BuildUp(viewModel);
```

### 泛型类型解析

```csharp
// 注册未绑定泛型
builder.Bind(typeof(IValidator<>)).To(typeof(Validator<>));

// 解析特定泛型类型
var stringValidator = container.Get<IValidator<string>>();
var intValidator = container.Get<IValidator<int>>();
```

### 延迟解析

```csharp
// 注册服务
builder.Bind<IMyService>().To<MyService>();

// 解析 Func<T> 进行延迟解析
var serviceFactory = container.Get<Func<IMyService>>();
var service = serviceFactory(); // 实际解析在这里发生
```

## 总结

`Container` 类是 StyletIoC 框架的核心实现，提供了完整的依赖注入功能。通过精心设计的数据结构和方法，它实现了高效的服务注册、解析和生命周期管理。其线程安全设计、性能优化和灵活的注册系统使其成为构建松耦合、可测试应用程序的理想选择。

该类的主要职责包括：
- 管理服务注册和解析
- 处理各种类型的依赖注入场景
- 提供线程安全的服务访问
- 支持泛型类型和集合类型
- 实现属性/字段注入

通过理解 `Container` 类的设计和实现，开发者可以更好地使用 StyletIoC 框架，构建出更加灵活和可维护的应用程序架构。