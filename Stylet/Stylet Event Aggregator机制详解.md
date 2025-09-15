# Stylet Event Aggregator机制详解

## 概述

Event Aggregator（事件聚合器）是Stylet框架中实现松散耦合组件间通信的核心机制。它采用发布-订阅（Publish-Subscribe）模式，允许不同的组件在不直接引用彼此的情况下进行事件通信。这种设计模式在MVVM架构中特别有用，可以实现ViewModel之间、ViewModel与Model之间的解耦通信。

## 核心设计理念

### 1. 松散耦合
Event Aggregator通过事件消息作为中介，实现组件之间的松散耦合。发布者不需要知道订阅者的存在，订阅者也不需要知道发布者的存在。

### 2. 弱引用机制
系统使用弱引用（WeakReference）来存储事件处理器，防止内存泄漏。当对象被垃圾回收时，相关的事件订阅会自动失效。

### 3. 类型安全
通过泛型接口`IHandle<T>`确保类型安全，编译时就能检查消息类型的匹配性。

### 4. 多通道支持
支持多通道（Channel）机制，允许将消息发布到特定的通道，实现更精细的消息路由控制。

## 核心接口设计

### IHandle接口

```csharp
/// <summary>
/// 标记接口，用于标识可能感兴趣的类型
/// </summary>
public interface IHandle
{
}

/// <summary>
/// 实现此接口来处理特定的消息类型
/// </summary>
/// <typeparam name="TMessageType">要处理的消息类型，可以是消息类型的基类</typeparam>
public interface IHandle<in TMessageType> : IHandle
{
    /// <summary>
    /// 当发布TMessageType类型的消息时调用
    /// </summary>
    /// <param name="message">发布的消息</param>
    void Handle(TMessageType message);
}
```

**设计特点：**
- `IHandle`作为标记接口，用于识别事件处理器
- `IHandle<T>`是泛型接口，支持协变（使用`in`关键字）
- 协变支持意味着如果A是B的子类，那么`IHandle<B>`可以处理A类型的消息

### IEventAggregator接口

```csharp
/// <summary>
/// 中心化的、弱引用绑定的发布/订阅事件管理器
/// </summary>
public interface IEventAggregator
{
    /// <summary>
    /// 注册实例以接收事件，需要实现IHandle{T}接口
    /// </summary>
    /// <param name="handler">要注册的实例</param>
    /// <param name="channels">要订阅的通道，如果不指定则默认为DefaultChannel</param>
    void Subscribe(IHandle handler, params string[] channels);

    /// <summary>
    /// 取消注册，实例将不再接收事件
    /// </summary>
    /// <param name="handler">要取消注册的实例</param>
    /// <param name="channels">要取消订阅的通道，如果不指定则取消所有订阅</param>
    void Unsubscribe(IHandle handler, params string[] channels);

    /// <summary>
    /// 使用指定的调度器向所有订阅者发布事件
    /// </summary>
    /// <param name="message">要发布的事件</param>
    /// <param name="dispatcher">用于调用订阅者处理方法的调度器</param>
    /// <param name="channels">要发布消息的通道，如果不指定则默认为DefaultChannel</param>
    void PublishWithDispatcher(object message, Action<Action> dispatcher, params string[] channels);
}
```

## EventAggregator核心实现

### 类结构设计

```csharp
/// <summary>
/// IEventAggregator的默认实现
/// </summary>
public class EventAggregator : IEventAggregator
{
    /// <summary>
    /// 默认通道名称，当未指定通道时使用
    /// </summary>
    public static readonly string DefaultChannel = "DefaultChannel";

    private readonly List<Handler> handlers = new();
    private readonly object handlersLock = new();
    
    // ... 实现方法
}
```

### 核心内部类

#### 1. Handler类 - 处理器包装器

```csharp
private class Handler
{
    private static readonly string[] DefaultChannelArray = new[] { DefaultChannel };

    private readonly WeakReference target;              // 弱引用目标对象
    private readonly List<HandlerInvoker> invokers = new();    // 方法调用器列表
    private readonly HashSet<string> channels = new();         // 订阅的通道集合

    public bool IsAlive => this.target.IsAlive;

    public Handler(object handler, string[] channels)
    {
        Type handlerType = handler.GetType();
        this.target = new WeakReference(handler);

        // 通过反射查找所有实现的IHandle<T>接口
        foreach (Type implementation in handler.GetType().GetInterfaces()
            .Where(x => x.IsGenericType && typeof(IHandle).IsAssignableFrom(x)))
        {
            Type messageType = implementation.GetGenericArguments()[0];
            this.invokers.Add(new HandlerInvoker(this.target, handlerType, messageType, 
                implementation.GetMethod("Handle")));
        }

        if (channels.Length == 0)
            channels = DefaultChannelArray;
        this.SubscribeToChannels(channels);
    }
}
```

**Handler类的职责：**
- 包装目标处理器对象，使用弱引用防止内存泄漏
- 管理订阅的通道集合
- 创建并管理方法调用器（HandlerInvoker）
- 检查对象生命周期状态

#### 2. HandlerInvoker类 - 方法调用器

```csharp
private class HandlerInvoker
{
    private readonly WeakReference target;
    private readonly Type messageType;
    private readonly Action<object, object> invoker;

    public HandlerInvoker(WeakReference target, Type targetType, Type messageType, MethodInfo invocationMethod)
    {
        this.target = target;
        this.messageType = messageType;
        
        // 使用表达式树编译高性能的方法调用器
        ParameterExpression targetParam = Expression.Parameter(typeof(object), "target");
        ParameterExpression messageParam = Expression.Parameter(typeof(object), "message");
        UnaryExpression castTarget = Expression.Convert(targetParam, targetType);
        UnaryExpression castMessage = Expression.Convert(messageParam, messageType);
        MethodCallExpression callExpression = Expression.Call(castTarget, invocationMethod, castMessage);
        this.invoker = Expression.Lambda<Action<object, object>>(callExpression, targetParam, messageParam).Compile();
    }

    public bool CanInvoke(Type messageType)
    {
        return this.messageType.IsAssignableFrom(messageType);
    }

    public void Invoke(object message)
    {
        object target = this.target.Target;
        // 检查目标对象是否还存活
        if (target != null)
            this.invoker(target, message);
    }
}
```

**HandlerInvoker的关键特性：**
- 使用表达式树（Expression Tree）编译高性能的方法调用器
- 支持类型兼容性检查（通过IsAssignableFrom）
- 在调用前检查目标对象是否仍然存活

### 核心方法实现

#### 1. Subscribe方法 - 订阅事件

```csharp
public void Subscribe(IHandle handler, params string[] channels)
{
    lock (this.handlersLock)
    {
        // 检查是否已经订阅
        Handler subscribed = this.handlers.FirstOrDefault(x => x.IsHandlerForInstance(handler));
        if (subscribed == null)
            this.handlers.Add(new Handler(handler, channels));
        else
            subscribed.SubscribeToChannels(channels);
    }
}
```

**Subscribe方法特点：**
- 线程安全（使用lock）
- 防止重复订阅
- 支持增量订阅（可以订阅更多通道）

#### 2. Unsubscribe方法 - 取消订阅

```csharp
public void Unsubscribe(IHandle handler, params string[] channels)
{
    lock (this.handlersLock)
    {
        Handler existingHandler = this.handlers.FirstOrDefault(x => x.IsHandlerForInstance(handler));
        if (existingHandler != null && existingHandler.UnsubscribeFromChannels(channels))
            this.handlers.Remove(existingHandler);
    }
}
```

#### 3. PublishWithDispatcher方法 - 发布事件

```csharp
public void PublishWithDispatcher(object message, Action<Action> dispatcher, params string[] channels)
{
    // 需要支持重入，因为处理器可能触发另一个消息或订阅
    // 这意味着在调用处理器时不能在迭代this.handlers的过程中

    List<HandlerInvoker> invokers;
    lock (this.handlersLock)
    {
        // 首先清理已失效的处理器
        this.handlers.RemoveAll(x => !x.IsAlive);

        Type messageType = message.GetType();
        invokers = this.handlers.SelectMany(x => x.GetInvokers(messageType, channels)).ToList();
    }

    // 在锁外执行调用，支持重入
    foreach (HandlerInvoker invoker in invokers)
    {
        dispatcher(() => invoker.Invoke(message));
    }
}
```

**PublishWithDispatcher的关键设计：**
- 支持重入：处理器可以在Handle方法中发布新消息或订阅事件
- 自动清理：每次发布前自动清理已失效的处理器
- 灵活调度：通过dispatcher参数支持不同的调度策略

## 扩展方法

Stylet提供了便利的扩展方法来简化事件发布：

```csharp
/// <summary>
/// Event Aggregator扩展方法，提供更多调度选项
/// </summary>
public static class EventAggregatorExtensions
{
    /// <summary>
    /// 在UI线程上发布事件
    /// </summary>
    public static void PublishOnUIThread(this IEventAggregator eventAggregator, object message, params string[] channels)
    {
        eventAggregator.PublishWithDispatcher(message, Execute.OnUIThread, channels);
    }

    /// <summary>
    /// 在当前线程同步发布事件
    /// </summary>
    public static void Publish(this IEventAggregator eventAggregator, object message, params string[] channels)
    {
        eventAggregator.PublishWithDispatcher(message, a => a(), channels);
    }
}
```

## 通道机制详解

### 通道的概念
通道（Channel）是Event Aggregator中的消息路由机制，允许将消息发布到特定的"频道"，只有订阅了该频道的处理器才会接收到消息。

### 默认通道
- 当不指定通道时，系统使用`EventAggregator.DefaultChannel`
- 值为字符串`"DefaultChannel"`

### 通道使用示例

```csharp
// 订阅默认通道
eventAggregator.Subscribe(handler);

// 订阅特定通道
eventAggregator.Subscribe(handler, "UserManagement");

// 订阅多个通道
eventAggregator.Subscribe(handler, "UserManagement", "SystemAlert");

// 发布到默认通道
eventAggregator.Publish(new UserCreatedEvent());

// 发布到特定通道
eventAggregator.Publish(new UserCreatedEvent(), "UserManagement");

// 发布到多个通道
eventAggregator.Publish(new SystemAlertEvent(), "UserManagement", "SystemAlert");
```

## 使用示例

### 1. 基本使用示例

#### 定义消息类型

```csharp
// 用户相关事件
public class UserCreatedEvent
{
    public string UserId { get; set; }
    public string UserName { get; set; }
    public DateTime CreatedAt { get; set; }
}

public class UserDeletedEvent
{
    public string UserId { get; set; }
    public DateTime DeletedAt { get; set; }
}

// 系统消息基类
public class SystemMessage
{
    public string Content { get; set; }
    public DateTime Timestamp { get; set; }
}

public class InfoMessage : SystemMessage { }
public class WarningMessage : SystemMessage { }
public class ErrorMessage : SystemMessage { }
```

#### 实现事件处理器

```csharp
// 用户管理服务 - 处理用户事件
public class UserManagementService : IHandle<UserCreatedEvent>, IHandle<UserDeletedEvent>
{
    public void Handle(UserCreatedEvent message)
    {
        Console.WriteLine($"用户管理服务：用户 {message.UserName} 已创建，ID: {message.UserId}");
        // 执行用户创建后的业务逻辑
        // 例如：发送欢迎邮件、初始化用户配置等
    }

    public void Handle(UserDeletedEvent message)
    {
        Console.WriteLine($"用户管理服务：用户 {message.UserId} 已删除");
        // 执行用户删除后的清理工作
        // 例如：清理用户数据、取消相关订阅等
    }
}

// 日志服务 - 处理所有系统消息
public class LoggingService : IHandle<SystemMessage>
{
    public void Handle(SystemMessage message)
    {
        string logLevel = message switch
        {
            InfoMessage => "INFO",
            WarningMessage => "WARN",
            ErrorMessage => "ERROR",
            _ => "UNKNOWN"
        };
        
        Console.WriteLine($"[{logLevel}] {message.Timestamp:yyyy-MM-dd HH:mm:ss} - {message.Content}");
    }
}

// 通知服务 - 只处理警告和错误消息
public class NotificationService : IHandle<WarningMessage>, IHandle<ErrorMessage>
{
    public void Handle(WarningMessage message)
    {
        Console.WriteLine($"⚠️ 警告通知：{message.Content}");
        // 发送警告通知
    }

    public void Handle(ErrorMessage message)
    {
        Console.WriteLine($"❌ 错误通知：{message.Content}");
        // 发送错误通知、可能需要立即处理
    }
}
```

#### 使用Event Aggregator

```csharp
public class ApplicationService
{
    private readonly IEventAggregator eventAggregator;
    
    public ApplicationService(IEventAggregator eventAggregator)
    {
        this.eventAggregator = eventAggregator;
        
        // 注册事件处理器
        var userService = new UserManagementService();
        var loggingService = new LoggingService();
        var notificationService = new NotificationService();
        
        this.eventAggregator.Subscribe(userService);
        this.eventAggregator.Subscribe(loggingService);
        this.eventAggregator.Subscribe(notificationService);
    }
    
    public void CreateUser(string userName)
    {
        // 创建用户的业务逻辑
        var userId = Guid.NewGuid().ToString();
        
        // 发布用户创建事件
        this.eventAggregator.Publish(new UserCreatedEvent
        {
            UserId = userId,
            UserName = userName,
            CreatedAt = DateTime.Now
        });
        
        // 发布信息日志
        this.eventAggregator.Publish(new InfoMessage
        {
            Content = $"新用户 {userName} 创建成功",
            Timestamp = DateTime.Now
        });
    }
    
    public void DeleteUser(string userId)
    {
        // 删除用户的业务逻辑
        
        // 发布用户删除事件
        this.eventAggregator.Publish(new UserDeletedEvent
        {
            UserId = userId,
            DeletedAt = DateTime.Now
        });
        
        // 发布警告日志
        this.eventAggregator.Publish(new WarningMessage
        {
            Content = $"用户 {userId} 已被删除",
            Timestamp = DateTime.Now
        });
    }
}
```

### 2. 通道使用示例

```csharp
public class ChannelExample
{
    private readonly IEventAggregator eventAggregator;
    
    public ChannelExample(IEventAggregator eventAggregator)
    {
        this.eventAggregator = eventAggregator;
        
        var adminHandler = new AdminHandler();
        var userHandler = new UserHandler();
        var auditHandler = new AuditHandler();
        
        // 订阅不同的通道
        this.eventAggregator.Subscribe(adminHandler, "Admin");
        this.eventAggregator.Subscribe(userHandler, "User");
        this.eventAggregator.Subscribe(auditHandler, "Admin", "User", "Audit"); // 订阅多个通道
    }
    
    public void PublishToSpecificChannels()
    {
        var message = new SystemMessage { Content = "测试消息", Timestamp = DateTime.Now };
        
        // 只有AdminHandler和AuditHandler会接收到这个消息
        this.eventAggregator.Publish(message, "Admin");
        
        // 只有UserHandler和AuditHandler会接收到这个消息
        this.eventAggregator.Publish(message, "User");
        
        // 只有AuditHandler会接收到这个消息
        this.eventAggregator.Publish(message, "Audit");
        
        // 向多个通道发布，但每个处理器只会收到一次
        this.eventAggregator.Publish(message, "Admin", "User");
    }
}

public class AdminHandler : IHandle<SystemMessage>
{
    public void Handle(SystemMessage message) => Console.WriteLine($"Admin处理：{message.Content}");
}

public class UserHandler : IHandle<SystemMessage>
{
    public void Handle(SystemMessage message) => Console.WriteLine($"User处理：{message.Content}");
}

public class AuditHandler : IHandle<SystemMessage>
{
    public void Handle(SystemMessage message) => Console.WriteLine($"Audit处理：{message.Content}");
}
```

### 3. 在MVVM中的使用示例

```csharp
// 视图模型基类
public class ViewModelBase : INotifyPropertyChanged
{
    protected readonly IEventAggregator EventAggregator;
    
    public ViewModelBase(IEventAggregator eventAggregator)
    {
        this.EventAggregator = eventAggregator;
        this.EventAggregator.Subscribe(this);
    }
    
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void OnPropertyChanged([CallerMemberName] string propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

// 用户列表视图模型
public class UserListViewModel : ViewModelBase, IHandle<UserCreatedEvent>, IHandle<UserDeletedEvent>
{
    private ObservableCollection<UserViewModel> users;
    
    public ObservableCollection<UserViewModel> Users
    {
        get => users;
        set
        {
            users = value;
            OnPropertyChanged();
        }
    }
    
    public UserListViewModel(IEventAggregator eventAggregator) : base(eventAggregator)
    {
        Users = new ObservableCollection<UserViewModel>();
    }
    
    public void Handle(UserCreatedEvent message)
    {
        // 在UI线程上更新用户列表
        Application.Current.Dispatcher.Invoke(() =>
        {
            Users.Add(new UserViewModel
            {
                UserId = message.UserId,
                UserName = message.UserName,
                CreatedAt = message.CreatedAt
            });
        });
    }
    
    public void Handle(UserDeletedEvent message)
    {
        // 在UI线程上从列表中移除用户
        Application.Current.Dispatcher.Invoke(() =>
        {
            var userToRemove = Users.FirstOrDefault(u => u.UserId == message.UserId);
            if (userToRemove != null)
            {
                Users.Remove(userToRemove);
            }
        });
    }
}

// 用户详情视图模型
public class UserDetailViewModel : ViewModelBase, IHandle<UserCreatedEvent>
{
    private string selectedUserId;
    private UserViewModel currentUser;
    
    public UserViewModel CurrentUser
    {
        get => currentUser;
        set
        {
            currentUser = value;
            OnPropertyChanged();
        }
    }
    
    public UserDetailViewModel(IEventAggregator eventAggregator) : base(eventAggregator)
    {
    }
    
    public void Handle(UserCreatedEvent message)
    {
        // 如果新创建的用户是当前选中的用户，更新详情
        if (selectedUserId == message.UserId)
        {
            Application.Current.Dispatcher.Invoke(() =>
            {
                CurrentUser = new UserViewModel
                {
                    UserId = message.UserId,
                    UserName = message.UserName,
                    CreatedAt = message.CreatedAt
                };
            });
        }
    }
}
```

## 线程调度机制

### 1. 默认调度（同步）

```csharp
// 在当前线程同步执行
eventAggregator.Publish(message);
```

### 2. UI线程调度

```csharp
// 在UI线程执行，适用于更新界面元素
eventAggregator.PublishOnUIThread(message);
```

### 3. 自定义调度器

```csharp
// 使用Task异步执行
eventAggregator.PublishWithDispatcher(message, action => Task.Run(action));

// 使用特定的TaskScheduler
eventAggregator.PublishWithDispatcher(message, action => 
    Task.Factory.StartNew(action, CancellationToken.None, TaskCreationOptions.None, customScheduler));

// 延迟执行
eventAggregator.PublishWithDispatcher(message, action => 
    Timer.DelayCall(TimeSpan.FromSeconds(1), action));
```

## 内存管理和性能优化

### 1. 弱引用机制
- 所有事件处理器都使用弱引用存储，防止内存泄漏
- 当对象被垃圾回收时，相关订阅会自动失效
- 每次发布事件前会自动清理已失效的处理器

### 2. 表达式树优化
- 使用表达式树编译高性能的方法调用器
- 避免反射调用的性能损耗
- 编译后的委托接近直接方法调用的性能

### 3. 线程安全设计
- 使用锁保护共享状态
- 支持重入调用
- 在锁外执行事件处理器，避免死锁

### 4. 最佳实践

```csharp
public class BestPracticesExample : IHandle<MyEvent>, IDisposable
{
    private readonly IEventAggregator eventAggregator;
    private bool isDisposed;
    
    public BestPracticesExample(IEventAggregator eventAggregator)
    {
        this.eventAggregator = eventAggregator;
        // 在构造函数中订阅
        this.eventAggregator.Subscribe(this);
    }
    
    public void Handle(MyEvent message)
    {
        // 检查是否已释放
        if (isDisposed) return;
        
        // 处理事件...
    }
    
    public void Dispose()
    {
        if (isDisposed) return;
        
        // 显式取消订阅（可选，弱引用会自动处理）
        eventAggregator.Unsubscribe(this);
        isDisposed = true;
    }
}
```

## 测试支持

### 单元测试示例

```csharp
[TestFixture]
public class EventAggregatorTests
{
    private EventAggregator eventAggregator;
    private TestHandler testHandler;
    
    [SetUp]
    public void SetUp()
    {
        eventAggregator = new EventAggregator();
        testHandler = new TestHandler();
    }
    
    [Test]
    public void Should_Deliver_Message_To_Subscribed_Handler()
    {
        // Arrange
        eventAggregator.Subscribe(testHandler);
        var message = new TestMessage { Content = "Test" };
        
        // Act
        eventAggregator.Publish(message);
        
        // Assert
        Assert.AreEqual(message, testHandler.ReceivedMessage);
        Assert.AreEqual(1, testHandler.HandleCount);
    }
    
    [Test]
    public void Should_Support_Multiple_Handlers_For_Same_Message()
    {
        // Arrange
        var handler1 = new TestHandler();
        var handler2 = new TestHandler();
        eventAggregator.Subscribe(handler1);
        eventAggregator.Subscribe(handler2);
        
        var message = new TestMessage { Content = "Test" };
        
        // Act
        eventAggregator.Publish(message);
        
        // Assert
        Assert.AreEqual(1, handler1.HandleCount);
        Assert.AreEqual(1, handler2.HandleCount);
    }
    
    [Test]
    public void Should_Handle_Inheritance_Correctly()
    {
        // Arrange
        var baseHandler = new BaseMessageHandler();
        var derivedHandler = new DerivedMessageHandler();
        
        eventAggregator.Subscribe(baseHandler);
        eventAggregator.Subscribe(derivedHandler);
        
        var derivedMessage = new DerivedMessage { Content = "Derived" };
        
        // Act
        eventAggregator.Publish(derivedMessage);
        
        // Assert
        Assert.AreEqual(1, baseHandler.HandleCount); // 应该接收到，因为DerivedMessage继承自BaseMessage
        Assert.AreEqual(1, derivedHandler.BaseHandleCount);
        Assert.AreEqual(1, derivedHandler.DerivedHandleCount);
    }
}

public class TestMessage
{
    public string Content { get; set; }
}

public class BaseMessage
{
    public string Content { get; set; }
}

public class DerivedMessage : BaseMessage
{
    public string AdditionalContent { get; set; }
}

public class TestHandler : IHandle<TestMessage>
{
    public TestMessage ReceivedMessage { get; private set; }
    public int HandleCount { get; private set; }
    
    public void Handle(TestMessage message)
    {
        ReceivedMessage = message;
        HandleCount++;
    }
}

public class BaseMessageHandler : IHandle<BaseMessage>
{
    public int HandleCount { get; private set; }
    
    public void Handle(BaseMessage message)
    {
        HandleCount++;
    }
}

public class DerivedMessageHandler : IHandle<BaseMessage>, IHandle<DerivedMessage>
{
    public int BaseHandleCount { get; private set; }
    public int DerivedHandleCount { get; private set; }
    
    public void Handle(BaseMessage message)
    {
        BaseHandleCount++;
    }
    
    public void Handle(DerivedMessage message)
    {
        DerivedHandleCount++;
    }
}
```

## 常见问题和解决方案

### 1. 内存泄漏问题
**问题：** 担心事件订阅导致内存泄漏
**解决：** Stylet使用弱引用机制，对象被垃圾回收时订阅会自动失效

### 2. 消息顺序问题
**问题：** 无法保证消息处理的顺序
**解决：** Event Aggregator不保证处理顺序，如需顺序处理可使用队列机制

### 3. 异常处理
**问题：** 某个处理器抛出异常影响其他处理器
**解决：** 在处理器中妥善处理异常，或使用装饰器模式包装异常处理

```csharp
public class SafeHandler : IHandle<MyMessage>
{
    public void Handle(MyMessage message)
    {
        try
        {
            // 实际处理逻辑
        }
        catch (Exception ex)
        {
            // 记录日志，不抛出异常
            Logger.LogError(ex, "处理消息时发生错误");
        }
    }
}
```

### 4. 性能考虑
**问题：** 大量事件处理器影响性能
**解决：** 
- 使用通道机制减少不必要的处理器调用
- 避免在处理器中执行耗时操作
- 考虑异步处理

## 总结

Stylet的Event Aggregator提供了一个强大、灵活且高性能的事件通信机制。它的主要优势包括：

1. **松散耦合**：组件间无需直接引用即可通信
2. **类型安全**：编译时检查消息类型匹配
3. **内存安全**：弱引用机制防止内存泄漏
4. **高性能**：表达式树编译的方法调用器
5. **线程安全**：支持多线程环境和重入调用
6. **灵活调度**：支持多种线程调度策略
7. **通道机制**：精细的消息路由控制

通过合理使用Event Aggregator，可以构建出模块化、可维护且高性能的应用程序架构。
