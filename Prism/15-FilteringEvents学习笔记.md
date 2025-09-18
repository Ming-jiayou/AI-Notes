# 15-FilteringEvents 学习笔记 - 事件过滤与高级订阅

## 项目概述

15-FilteringEvents 展示了 **EventAggregator** 的**高级订阅功能**，通过**线程选项**、**保持引用**和**过滤器**等参数，实现了**精确的事件订阅控制**，为**性能优化**和**条件订阅**提供了**企业级事件管理**能力。

## 核心突破

### 与14示例的关键差异
- **14-UsingEventAggregator**：基础订阅，所有消息都接收
- **15-FilteringEvents**：**条件过滤订阅**，只接收满足条件的事件

## 高级订阅功能

### 1. 完整订阅参数

#### MessageListViewModel.cs - 精确订阅控制
```csharp
public MessageListViewModel(IEventAggregator ea)
{
    _ea = ea;
    Messages = new ObservableCollection<string>();

    // 高级订阅：线程选项 + 保持引用 + 过滤器
    _ea.GetEvent<MessageSentEvent>().Subscribe(
        MessageReceived,           // 处理方法
        ThreadOption.PublisherThread, // 执行线程
        false,                     // 保持弱引用
        (filter) => filter.Contains("Brian")  // 过滤器
    );
}
```

## 订阅参数详解

| 参数 | 选项 | 作用 | 适用场景 |
|------|------|------|----------|
| **action** | 处理方法 | 事件处理逻辑 | 必需参数 |
| **threadOption** | PublisherThread/UIThread/BackgroundThread | 执行线程控制 | UI更新/后台处理 |
| **keepSubscriberReferenceAlive** | true/false | 引用生命周期 | 内存管理 |
| **filter** | Func<T, bool> | 事件过滤条件 | 条件订阅 |

### 2. 线程选项对比

| ThreadOption | 执行线程 | 适用场景 | 性能影响 |
|-------------|----------|----------|----------|
| **PublisherThread** | 发布者线程 | 快速处理 | 最低延迟 |
| **UIThread** | UI线程 | UI更新 | 线程安全 |
| **BackgroundThread** | 后台线程 | 耗时操作 | 不阻塞UI |

### 3. 引用管理策略

| keepSubscriberReferenceAlive | 内存管理 | 适用场景 | 风险 |
|-----------------------------|----------|----------|------|
| **false (默认)** | 弱引用，自动回收 | 大多数场景 | 可能提前回收 |
| **true** | 强引用，手动管理 | 长生命周期 | 内存泄漏风险 |

## 过滤器实现

### 条件订阅示例
```csharp
// 只接收包含"Brian"的消息
(filter) => filter.Contains("Brian")

// 只接收特定类型的消息
(filter) => filter.StartsWith("ERROR")

// 复合条件过滤
(filter) => filter.Length > 10 && filter.Contains("urgent")
```

### 实际过滤效果
```
发布消息序列：
1. "Hello World"     → ❌ 被过滤 (不包含"Brian")
2. "Hi Brian"        → ✅ 被接收 (包含"Brian")  
3. "Brian is here"   → ✅ 被接收 (包含"Brian")
4. "Good morning"    → ❌ 被过滤 (不包含"Brian")
```

## 企业级应用场景

### 1. 日志级别过滤
```csharp
// 只接收错误级别日志
_eventAggregator.GetEvent<LogEvent>().Subscribe(
    OnErrorLog,
    ThreadOption.BackgroundThread,
    false,
    (log) => log.Level == LogLevel.Error
);
```

### 2. 用户权限过滤
```csharp
// 只接收当前用户相关事件
_eventAggregator.GetEvent<UserEvent>().Subscribe(
    OnUserEvent,
    ThreadOption.UIThread,
    false,
    (userEvent) => userEvent.UserId == currentUserId
);
```

### 3. 性能优化过滤
```csharp
// 只处理重要更新，忽略频繁的小更新
_eventAggregator.GetEvent<DataUpdateEvent>().Subscribe(
    OnDataUpdate,
    ThreadOption.BackgroundThread,
    false,
    (update) => update.ChangeMagnitude > 0.1
);
```

## 性能与内存优化

### 1. 线程优化
```csharp
// 耗时操作在后台线程
_eventAggregator.GetEvent<CalculationEvent>().Subscribe(
    PerformComplexCalculation,
    ThreadOption.BackgroundThread,  // 不阻塞UI
    false,
    (data) => data.NeedsCalculation
);
```

### 2. 内存优化
```csharp
// 短生命周期对象使用弱引用
_eventAggregator.GetEvent<TempEvent>().Subscribe(
    HandleTempEvent,
    ThreadOption.PublisherThread,
    false,  // 弱引用，允许GC回收
    (temp) => temp.IsValid
);
```

### 3. 过滤优化
```csharp
// 先过滤再处理，减少不必要的处理
_eventAggregator.GetEvent<NotificationEvent>().Subscribe(
    ShowNotification,
    ThreadOption.UIThread,
    false,
    (notification) => notification.Priority > Priority.Normal  // 只处理高优先级
);
```

## 高级过滤模式

### 1. 多条件组合
```csharp
// 复合过滤条件
(filter) => filter.Sender == "System" && 
            filter.Priority >= Priority.High &&
            filter.Timestamp > DateTime.Now.AddHours(-1)
```

### 2. 动态过滤
```csharp
// 基于运行时状态的动态过滤
bool isDebugMode = GetDebugMode();
(filter) => isDebugMode || filter.IsProductionReady
```

### 3. 类型安全过滤
```csharp
// 泛型事件过滤
public class TypedEvent<T> : PubSubEvent<T> where T : IFilterable
{
    // 过滤实现
    (filter) => filter.ShouldProcess()
}
```

此示例代表了**事件系统**的**高级管理能力**，为**企业级应用**提供了**精确的事件订阅控制**和**性能优化**能力。