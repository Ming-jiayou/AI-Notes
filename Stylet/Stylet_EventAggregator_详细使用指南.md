# Stylet EventAggregator 详细使用指南

## 概述

EventAggregator是Stylet框架提供的**事件聚合器**实现，它实现了**发布-订阅模式**（Publish-Subscribe Pattern），用于在应用程序的不同组件之间进行**松耦合通信**。通过EventAggregator，组件可以发布事件而无需知道谁会接收，也可以订阅事件而无需知道事件来源。

### 核心特性
- **弱引用订阅**：自动内存管理，防止内存泄漏
- **类型安全**：基于泛型的事件处理
- **多通道支持**：支持事件通道（channels）概念
- **线程安全**：支持UI线程调度
- **灵活调度**：支持同步/异步事件处理

## 核心接口与类

### 1. IHandle接口体系

#### IHandle（标记接口）
```csharp
public interface IHandle { }
```
- 空标记接口，用于标识可以处理事件的类型

#### IHandle<TMessageType>（泛型处理接口）
```csharp
public interface IHandle<in TMessageType> : IHandle
{
    void Handle(TMessageType message);
}
```
- 泛型接口，定义具体事件类型的处理方法
- 支持逆变（in关键字），允许处理派生类型事件

### 2. IEventAggregator接口

```csharp
public interface IEventAggregator
{
    void Subscribe(IHandle handler, params string[] channels);
    void Unsubscribe(IHandle handler, params string[] channels);
    void PublishWithDispatcher(object message, Action<Action> dispatcher, params string[] channels);
}
```

### 3. EventAggregator类

Stylet提供的默认实现，包含：
- **弱引用管理**：使用`WeakReference`防止内存泄漏
- **通道机制**：支持事件分组
- **表达式树优化**：使用表达式树编译快速调用

## 基本使用流程

### 步骤1：定义事件类

```csharp
// 定义一个简单的事件
public class UserLoggedInEvent
{
    public string Username { get; set; }
    public DateTime LoginTime { get; set; }
}

// 定义带业务数据的事件
public class OrderCreatedEvent
{
    public int OrderId { get; set; }
    public decimal Amount { get; set; }
    public string CustomerName { get; set; }
}
```

### 步骤2：创建事件订阅者

```csharp
public class UserNotificationViewModel : Screen, IHandle<UserLoggedInEvent>, IHandle<OrderCreatedEvent>
{
    private readonly IEventAggregator _events;
    
    public UserNotificationViewModel(IEventAggregator events)
    {
        _events = events;
        // 订阅事件
        _events.Subscribe(this);
    }
    
    // 处理用户登录事件
    public void Handle(UserLoggedInEvent message)
    {
        ShowNotification($"欢迎回来，{message.Username}！");
    }
    
    // 处理订单创建事件
    public void Handle(OrderCreatedEvent message)
    {
        ShowNotification($"新订单 #{message.OrderId} 已创建，金额：¥{message.Amount}");
    }
    
    private void ShowNotification(string message)
    {
        // 显示通知逻辑
    }
    
    protected override void OnDeactivate()
    {
        // 清理订阅
        _events.Unsubscribe(this);
        base.OnDeactivate();
    }
}
```

### 步骤3：发布事件

```csharp
public class LoginViewModel : Screen
{
    private readonly IEventAggregator _events;
    
    public LoginViewModel(IEventAggregator events)
    {
        _events = events;
    }
    
    public void Login()
    {
        // 登录逻辑...
        
        // 发布事件
        _events.PublishOnUIThread(new UserLoggedInEvent
        {
            Username = this.Username,
            LoginTime = DateTime.Now
        });
    }
}
```

## 高级用法

### 1. 通道（Channels）机制

通道允许将事件分组，实现更精细的事件路由：

```csharp
// 定义通道常量
public static class EventChannels
{
    public const string UserManagement = "UserManagement";
    public const string OrderManagement = "OrderManagement";
    public const string SystemNotifications = "SystemNotifications";
}

// 订阅特定通道
public class UserEventHandler : IHandle<UserCreatedEvent>, IHandle<UserDeletedEvent>
{
    public UserEventHandler(IEventAggregator events)
    {
        // 只订阅用户管理通道的事件
        events.Subscribe(this, EventChannels.UserManagement);
    }
    
    public void Handle(UserCreatedEvent message) { /* ... */ }
    public void Handle(UserDeletedEvent message) { /* ... */ }
}

// 发布到特定通道
public class UserService
{
    private readonly IEventAggregator _events;
    
    public void CreateUser(User user)
    {
        // 创建用户逻辑...
        
        // 发布到用户管理通道
        _events.PublishOnUIThread(
            new UserCreatedEvent { User = user }, 
            EventChannels.UserManagement
        );
    }
}
```

### 2. 线程调度选项

EventAggregator提供多种线程调度方式：

#### PublishOnUIThread（UI线程）
```csharp
// 在UI线程上执行所有订阅者的Handle方法
_events.PublishOnUIThread(new MyEvent());
```

#### Publish（当前线程）
```csharp
// 在当前线程同步执行所有订阅者的Handle方法
_events.Publish(new MyEvent());
```

#### PublishWithDispatcher（自定义调度）
```csharp
// 使用自定义调度器
_events.PublishWithDispatcher(
    new MyEvent(), 
    action => Task.Run(action) // 在后台线程执行
);
```

### 3. 继承层次的事件处理

利用逆变特性，可以处理继承层次中的事件：

```csharp
// 基础事件类
public class DomainEvent { }

// 具体事件类
public class UserCreatedEvent : DomainEvent { }
public class OrderCreatedEvent : DomainEvent { }

// 通用事件处理器
public class LoggingEventHandler : IHandle<DomainEvent>
{
    public void Handle(DomainEvent message)
    {
        // 记录所有领域事件
        _logger.LogInformation($"Event: {message.GetType().Name}");
    }
}

// 注册后，LoggingEventHandler将接收所有DomainEvent及其子类事件
```

## 实际应用示例

### 示例1：Reddit浏览器中的事件通信

基于Stylet官方示例，展示实际应用场景：

```csharp
// 定义事件
public class OpenSubredditEvent
{
    public string Subreddit { get; set; }
    public SortMode SortMode { get; set; }
}

// 事件发布者 - 任务栏视图模型
public class TaskbarViewModel : Screen
{
    private readonly IEventAggregator _events;
    
    public string Subreddit { get; set; }
    public SortMode SelectedSortMode { get; set; }
    
    public TaskbarViewModel(IEventAggregator events)
    {
        _events = events;
    }
    
    public bool CanOpen => !string.IsNullOrWhiteSpace(this.Subreddit);
    
    public void Open()
    {
        // 发布事件，通知打开新的子版块
        _events.Publish(new OpenSubredditEvent
        {
            Subreddit = this.Subreddit,
            SortMode = this.SelectedSortMode
        });
    }
}

// 事件订阅者 - Shell视图模型
public class ShellViewModel : Conductor<IScreen>.Collection.OneActive, IHandle<OpenSubredditEvent>
{
    private readonly IEventAggregator _events;
    private readonly ISubredditViewModelFactory _subredditFactory;
    
    public ShellViewModel(IEventAggregator events, ISubredditViewModelFactory subredditFactory)
    {
        _events = events;
        _subredditFactory = subredditFactory;
        
        // 订阅事件
        events.Subscribe(this);
    }
    
    public void Handle(OpenSubredditEvent message)
    {
        // 处理打开子版块事件
        var subredditVM = _subredditFactory.CreateSubredditViewModel();
        subredditVM.Subreddit = message.Subreddit;
        subredditVM.SortMode = message.SortMode;
        
        // 激活新的子版块视图
        this.ActivateItem(subredditVM);
    }
}
```

### 示例2：模块化应用程序中的跨模块通信

```csharp
// 模块A：用户管理模块
public class UserModule
{
    private readonly IEventAggregator _events;
    
    public void DeleteUser(int userId)
    {
        // 删除用户逻辑...
        
        // 发布用户删除事件
        _events.PublishOnUIThread(new UserDeletedEvent
        {
            UserId = userId,
            DeletedAt = DateTime.Now
        });
    }
}

// 模块B：权限管理模块
public class PermissionModule : IHandle<UserDeletedEvent>
{
    private readonly IEventAggregator _events;
    
    public PermissionModule(IEventAggregator events)
    {
        _events = events;
        _events.Subscribe(this);
    }
    
    public void Handle(UserDeletedEvent message)
    {
        // 用户删除后，清理相关权限
        _permissionService.RevokeAllPermissions(message.UserId);
    }
}

// 模块C：日志模块
public class LoggingModule : IHandle<UserDeletedEvent>
{
    public void Handle(UserDeletedEvent message)
    {
        // 记录用户删除日志
        _logger.LogInformation($"User {message.UserId} deleted at {message.DeletedAt}");
    }
}
```

## 内存管理与生命周期

### 弱引用机制

EventAggregator使用`WeakReference`来持有订阅者引用，这意味着：

1. **自动清理**：当订阅者被垃圾回收时，EventAggregator会自动清理相关订阅
2. **无需手动取消订阅**：在大多数情况下，不需要显式调用`Unsubscribe`
3. **防止内存泄漏**：不会因为事件订阅而阻止对象被垃圾回收

### 最佳实践

```csharp
// 推荐做法：Screen/ViewModel基类已处理清理
public class MyViewModel : Screen, IHandle<MyEvent>
{
    public MyViewModel(IEventAggregator events)
    {
        events.Subscribe(this);
    }
    
    public void Handle(MyEvent message) { /* ... */ }
    
    // Screen基类会在适当时候处理清理
}

// 如果需要手动控制
public class ServiceWithLongLifetime : IHandle<MyEvent>, IDisposable
{
    private readonly IEventAggregator _events;
    
    public ServiceWithLongLifetime(IEventAggregator events)
    {
        _events = events;
        _events.Subscribe(this);
    }
    
    public void Handle(MyEvent message) { /* ... */ }
    
    public void Dispose()
    {
        // 显式取消订阅
        _events.Unsubscribe(this);
    }
}
```

## 性能优化建议

### 1. 事件粒度设计
- **细粒度事件**：针对特定操作，减少无关订阅者的处理开销
- **粗粒度事件**：减少事件数量，适合频繁操作

```csharp
// 细粒度：针对特定操作
public class UserPasswordChangedEvent { public int UserId { get; set; } }

// 粗粒度：通用更新事件
public class UserUpdatedEvent { public User User { get; set; } }
```

### 2. 异步处理
对于耗时操作，使用异步处理避免阻塞UI线程：

```csharp
public class ReportGenerator : IHandle<GenerateReportEvent>
{
    public async void Handle(GenerateReportEvent message)
    {
        // 注意：async void仅适用于事件处理器
        try
        {
            var report = await GenerateReportAsync(message.ReportId);
            _events.PublishOnUIThread(new ReportGeneratedEvent { Report = report });
        }
        catch (Exception ex)
        {
            _events.PublishOnUIThread(new ReportGenerationFailedEvent 
            { 
                ReportId = message.ReportId,
                Error = ex.Message 
            });
        }
    }
}
```

### 3. 批量事件处理
对于高频事件，考虑批量处理：

```csharp
public class BatchEventHandler : IHandle<ItemChangedEvent>
{
    private readonly List<ItemChangedEvent> _pendingEvents = new();
    private readonly Timer _batchTimer;
    
    public BatchEventHandler(IEventAggregator events)
    {
        events.Subscribe(this);
        _batchTimer = new Timer(ProcessBatch, null, Timeout.Infinite, Timeout.Infinite);
    }
    
    public void Handle(ItemChangedEvent message)
    {
        lock (_pendingEvents)
        {
            _pendingEvents.Add(message);
            _batchTimer.Change(100, Timeout.Infinite); // 100ms后处理
        }
    }
    
    private void ProcessBatch(object state)
    {
        List<ItemChangedEvent> toProcess;
        lock (_pendingEvents)
        {
            toProcess = new List<ItemChangedEvent>(_pendingEvents);
            _pendingEvents.Clear();
        }
        
        if (toProcess.Any())
        {
            _events.PublishOnUIThread(new ItemsBatchChangedEvent 
            { 
                Items = toProcess.Select(e => e.Item).ToList() 
            });
        }
    }
}
```

## 调试与故障排除

### 1. 事件流监控

创建调试处理器监控所有事件：

```csharp
public class EventDebugger : IHandle<object>
{
    private readonly ILogger<EventDebugger> _logger;
    
    public EventDebugger(IEventAggregator events, ILogger<EventDebugger> logger)
    {
        _logger = logger;
        events.Subscribe(this);
    }
    
    public void Handle(object message)
    {
        _logger.LogDebug($"Event published: {message.GetType().Name}");
    }
}
```

### 2. 订阅者诊断

检查当前订阅状态：

```csharp
public static class EventAggregatorDiagnostics
{
    public static void LogSubscriptions(IEventAggregator events)
    {
        // 使用反射获取内部状态（仅调试使用）
        var eventAggregator = events as EventAggregator;
        if (eventAggregator != null)
        {
            var handlersField = typeof(EventAggregator)
                .GetField("handlers", BindingFlags.NonPublic | BindingFlags.Instance);
            var handlers = handlersField?.GetValue(eventAggregator) as List<object>;
            
            Debug.WriteLine($"Total subscribers: {handlers?.Count ?? 0}");
        }
    }
}
```

## 常见陷阱与解决方案

### 1. 事件循环
**问题**：A发布事件→B处理并发布事件→A处理，形成循环

**解决方案**：
```csharp
public class SafeEventHandler : IHandle<MyEvent>
{
    private readonly ThreadLocal<bool> _isProcessing = new();
    
    public void Handle(MyEvent message)
    {
        if (_isProcessing.Value)
            return; // 避免循环
            
        _isProcessing.Value = true;
        try
        {
            // 处理事件
        }
        finally
        {
            _isProcessing.Value = false;
        }
    }
}
```

### 2. UI线程阻塞
**问题**：在Handle方法中执行耗时操作阻塞UI

**解决方案**：
```csharp
public class AsyncEventHandler : IHandle<HeavyWorkEvent>
{
    public void Handle(HeavyWorkEvent message)
    {
        // 立即返回，后台处理
        Task.Run(() => DoHeavyWork(message));
    }
    
    private async Task DoHeavyWork(HeavyWorkEvent message)
    {
        // 耗时操作
        var result = await ProcessDataAsync(message.Data);
        
        // 完成后回到UI线程通知
        _events.PublishOnUIThread(new WorkCompletedEvent { Result = result });
    }
}
```

### 3. 内存泄漏（特殊情况）
虽然EventAggregator使用弱引用，但以下情况仍需注意：

```csharp
// 问题：匿名方法捕获外部变量
public class ProblematicClass
{
    public void SetupHandler(IEventAggregator events)
    {
        // 错误：匿名方法捕获this，可能导致内存泄漏
        events.Subscribe(new AnonymousHandler(e => this.HandleEvent(e)));
    }
    
    private void HandleEvent(MyEvent e) { /* ... */ }
    
    private class AnonymousHandler : IHandle<MyEvent>
    {
        private readonly Action<MyEvent> _handler;
        public AnonymousHandler(Action<MyEvent> handler) => _handler = handler;
        public void Handle(MyEvent message) => _handler(message);
    }
}
```

## 与IoC容器集成

### 1. 自动注册

在Bootstrapper中配置：

```csharp
public class Bootstrapper : Bootstrapper<ShellViewModel>
{
    protected override void ConfigureIoC(IStyletIoCBuilder builder)
    {
        // 注册EventAggregator为单例
        builder.Bind<IEventAggregator>().To<EventAggregator>().InSingletonScope();
        
        // 自动注册所有IHandle<>实现
        var handlerTypes = Assembly.GetExecutingAssembly()
            .GetTypes()
            .Where(t => t.GetInterfaces().Any(i => 
                i.IsGenericType && i.GetGenericTypeDefinition() == typeof(IHandle<>)));
                
        foreach (var type in handlerTypes)
        {
            builder.Bind(type).ToSelf().InSingletonScope();
        }
    }
}
```

### 2. 自动订阅

创建基类自动处理订阅：

```csharp
public abstract class AutoSubscribingScreen : Screen
{
    protected IEventAggregator Events { get; }
    
    protected AutoSubscribingScreen(IEventAggregator events)
    {
        Events = events;
    }
    
    protected override void OnActivate()
    {
        base.OnActivate();
        Events.Subscribe(this);
    }
    
    protected override void OnDeactivate()
    {
        Events.Unsubscribe(this);
        base.OnDeactivate();
    }
}

// 使用示例
public class MyViewModel : AutoSubscribingScreen, IHandle<MyEvent>
{
    public MyViewModel(IEventAggregator events) : base(events) { }
    
    public void Handle(MyEvent message) { /* ... */ }
}
```

## 总结

Stylet的EventAggregator提供了强大而灵活的事件通信机制，通过遵循本指南的最佳实践，可以构建松耦合、可维护的应用程序。记住以下关键点：

1. **定义清晰的事件类型**：使用具体的事件类而非原始类型
2. **合理使用通道**：根据业务逻辑分组事件
3. **注意线程安全**：使用适当的调度方式
4. **避免内存泄漏**：虽然EventAggregator使用弱引用，但仍需注意特殊情况
5. **保持处理器简洁**：复杂的业务逻辑应该委托给专门的服务

通过合理使用EventAggregator，可以显著降低组件间的耦合度，提高代码的可测试性和可维护性。