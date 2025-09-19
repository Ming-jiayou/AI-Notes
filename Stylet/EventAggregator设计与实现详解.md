# Stylet EventAggregator 设计与实现详解

## 概述

Stylet 的 EventAggregator 是一个集中式、弱引用的发布/订阅事件管理系统，专为 WPF/MVVM 应用程序设计。它提供了一种解耦的通信机制，允许不同组件之间进行消息传递，而无需直接引用彼此。

## 核心设计目标

1. **解耦通信**：发布者和订阅者无需知道彼此的存在
2. **弱引用管理**：防止内存泄漏，自动清理已销毁的订阅者
3. **类型安全**：通过泛型接口确保类型安全的消息处理
4. **线程安全**：支持多线程环境下的安全操作
5. **通道支持**：允许消息按通道进行分组和过滤
6. **高性能**：使用表达式树编译优化方法调用

## 接口设计

### IHandle 标记接口

```csharp
public interface IHandle
{
}
```

这是一个标记接口，用于标识可能对接收事件感兴趣的类型。它本身不包含任何方法，主要用于类型约束和标识。

### IHandle<TMessageType> 泛型接口

```csharp
public interface IHandle<in TMessageType> : IHandle
{
    void Handle(TMessageType message);
}
```

这是核心的消息处理接口，包含以下设计特点：

- **逆变泛型参数** (`in TMessageType`)：允许处理基类类型的消息
- **单一职责**：每个实现类专注于处理特定类型的消息
- **灵活性**：可以通过实现多个此接口来处理不同类型的消息

### IEventAggregator 接口

```csharp
public interface IEventAggregator
{
    void Subscribe(IHandle handler, params string[] channels);
    void Unsubscribe(IHandle handler, params string[] channels);
    void PublishWithDispatcher(object message, Action<Action> dispatcher, params string[] channels);
}
```

接口设计体现了以下原则：

- **最小化接口**：只暴露必要的操作
- **通道支持**：允许按通道进行订阅和发布
- **调度器抽象**：将执行策略委托给外部调度器

## 核心实现分析

### EventAggregator 主类

#### 线程安全机制

```csharp
private readonly List<Handler> handlers = new();
private readonly object handlersLock = new();

public void Subscribe(IHandle handler, params string[] channels)
{
    lock (this.handlersLock)
    {
        // 线程安全的订阅逻辑
    }
}
```

- 使用专用锁对象 `handlersLock` 保护共享状态
- 所有对 `handlers` 列表的操作都在锁内进行
- 防止并发访问导致的数据不一致

#### 发布机制实现

```csharp
public void PublishWithDispatcher(object message, Action<Action> dispatcher, params string[] channels)
{
    List<HandlerInvoker> invokers;
    lock (this.handlersLock)
    {
        // 清理已销毁的处理器
        this.handlers.RemoveAll(x => !x.IsAlive);
        
        Type messageType = message.GetType();
        invokers = this.handlers.SelectMany(x => x.GetInvokers(messageType, channels)).ToList();
    }

    foreach (HandlerInvoker invoker in invokers)
    {
        dispatcher(() => invoker.Invoke(message));
    }
}
```

关键设计特点：

1. **可重入性支持**：在锁外执行处理器调用，避免死锁
2. **自动清理**：发布时自动移除已销毁的弱引用目标
3. **延迟执行**：收集所有匹配的处理器后在锁外执行

### Handler 内部类

#### 弱引用管理

```csharp
private readonly WeakReference target;
public bool IsAlive => this.target.IsAlive;
```

- 使用 `WeakReference` 避免强引用导致的内存泄漏
- 提供 `IsAlive` 属性快速检查目标是否仍然有效

#### 通道管理

```csharp
private readonly HashSet<string> channels = new();

public void SubscribeToChannels(string[] channels)
{
    this.channels.UnionWith(channels);
}

public bool UnsubscribeFromChannels(string[] channels)
{
    if (channels.Length == 0)
        return true; // 从所有通道取消订阅
    this.channels.ExceptWith(channels);
    return this.channels.Count == 0; // 如果没有订阅的通道，返回true表示可以移除
}
```

- 使用 `HashSet<string>` 高效管理通道订阅
- 支持批量订阅和取消订阅操作
- 智能的取消订阅逻辑，自动判断是否需要完全移除处理器

#### 消息类型匹配

```csharp
public IEnumerable<HandlerInvoker> GetInvokers(Type messageType, string[] channels)
{
    if (!this.IsAlive)
        return Enumerable.Empty<HandlerInvoker>();

    if (channels.Length == 0)
        channels = DefaultChannelArray;

    // 检查通道匹配
    if (!channels.Any(x => this.channels.Contains(x)))
        return Enumerable.Empty<HandlerInvoker>();

    return this.invokers.Where(x => x.CanInvoke(messageType));
}
```

匹配逻辑包含两个层面：

1. **通道匹配**：消息必须发布到订阅者订阅的通道
2. **类型匹配**：消息类型必须可赋值给处理器参数类型

### HandlerInvoker 内部类

#### 高性能方法调用

```csharp
private readonly Action<object, object> invoker;

public HandlerInvoker(WeakReference target, Type targetType, Type messageType, MethodInfo invocationMethod)
{
    this.target = target;
    this.messageType = messageType;
    
    // 使用表达式树构建高性能调用委托
    ParameterExpression targetParam = Expression.Parameter(typeof(object), "target");
    ParameterExpression messageParam = Expression.Parameter(typeof(object), "message");
    UnaryExpression castTarget = Expression.Convert(targetParam, targetType);
    UnaryExpression castMessage = Expression.Convert(messageParam, messageType);
    MethodCallExpression callExpression = Expression.Call(castTarget, invocationMethod, castMessage);
    this.invoker = Expression.Lambda<Action<object, object>>(callExpression, targetParam, messageParam).Compile();
}
```

性能优化策略：

1. **表达式树编译**：将反射调用转换为编译后的委托
2. **类型转换优化**：使用表达式树进行类型转换
3. **缓存编译结果**：每个处理器只需要编译一次

#### 类型兼容性检查

```csharp
public bool CanInvoke(Type messageType)
{
    return this.messageType.IsAssignableFrom(messageType);
}
```

- 支持继承层次结构中的消息处理
- 允许基类处理器处理派生类消息

## 扩展方法设计

### EventAggregatorExtensions

```csharp
public static class EventAggregatorExtensions
{
    public static void PublishOnUIThread(this IEventAggregator eventAggregator, object message, params string[] channels)
    {
        eventAggregator.PublishWithDispatcher(message, Execute.OnUIThread, channels);
    }

    public static void Publish(this IEventAggregator eventAggregator, object message, params string[] channels)
    {
        eventAggregator.PublishWithDispatcher(message, a => a(), channels);
    }
}
```

扩展方法提供了常用的发布模式：

1. **UI线程发布**：确保处理器在UI线程上执行
2. **同步发布**：在当前线程上同步执行处理器

## 设计优势

### 1. 内存管理
- 弱引用防止内存泄漏
- 自动清理已销毁的对象
- 无需手动取消订阅（但建议显式取消）

### 2. 性能优化
- 表达式树编译避免反射开销
- 高效的通道匹配算法
- 延迟执行减少锁竞争

### 3. 灵活性
- 支持多通道消息传递
- 类型安全的泛型接口
- 可插拔的调度器机制

### 4. 线程安全
- 完善的并发控制
- 可重入的发布机制
- 无死锁设计

## 使用模式

### 基本订阅模式

```csharp
public class MyViewModel : IHandle<MyMessage>
{
    private readonly IEventAggregator eventAggregator;
    
    public MyViewModel(IEventAggregator eventAggregator)
    {
        this.eventAggregator = eventAggregator;
        this.eventAggregator.Subscribe(this);
    }
    
    public void Handle(MyMessage message)
    {
        // 处理消息
    }
}
```

### 多消息类型处理

```csharp
public class MultiHandlerViewModel : IHandle<MessageType1>, IHandle<MessageType2>
{
    public void Handle(MessageType1 message) { /* ... */ }
    public void Handle(MessageType2 message) { /* ... */ }
}
```

### 通道化消息

```csharp
// 订阅特定通道
eventAggregator.Subscribe(handler, "Channel1", "Channel2");

// 发布到特定通道
eventAggregator.Publish(message, "Channel1");
```

## 最佳实践

1. **及时取消订阅**：在对象销毁时显式调用 `Unsubscribe`
2. **合理使用通道**：避免过度使用通道导致复杂性增加
3. **消息设计**：保持消息简单，只包含必要数据
4. **错误处理**：在处理器中妥善处理异常，避免影响其他订阅者
5. **性能考虑**：避免在处理器中执行耗时操作，考虑使用异步模式

## 总结

Stylet 的 EventAggregator 实现体现了优秀的设计原则和工程实践。它通过弱引用管理、表达式树优化、线程安全设计等机制，提供了一个高效、可靠、易用的事件聚合系统。其模块化的设计使得它不仅可以用于 WPF 应用程序，也可以作为独立的事件总线组件在其他 .NET 应用程序中使用。