# 14-UsingEventAggregator 学习笔记 - 事件聚合器基础

## 项目概述

14-UsingEventAggregator 展示了 Prism 的 **EventAggregator** 事件聚合系统，实现了**模块间松耦合通信**，支持**发布-订阅模式**的事件传递，为**模块化架构**提供了**事件驱动通信**的基础能力。

## 项目结构

```
14-UsingEventAggregator/
├── UsingEventAggregator.Core/     # 共享事件定义
│   └── MessageSentEvent.cs        # 事件契约
├── ModuleA/                       # 事件发布者
│   └── ViewModels/
│       └── MessageViewModel.cs    # 发布消息
├── ModuleB/                       # 事件订阅者  
│   └── ViewModels/
│       └── MessageListViewModel.cs # 接收消息
└── UsingEventAggregator/          # 主程序
    └── ViewModels/
        └── MainWindowViewModel.cs
```

## 核心机制

### 1. 事件定义

#### MessageSentEvent.cs - 事件契约
```csharp
using Prism.Events;

namespace UsingEventAggregator.Core
{
    // 定义事件：继承PubSubEvent<T>
    public class MessageSentEvent : PubSubEvent<string>
    {
    }
}
```

### 2. 事件发布

#### MessageViewModel.cs - 发布者实现
```csharp
public class MessageViewModel : BindableBase
{
    private IEventAggregator _eventAggregator;

    public DelegateCommand SendMessageCommand { get; private set; }

    public MessageViewModel(IEventAggregator eventAggregator)
    {
        _eventAggregator = eventAggregator;
        SendMessageCommand = new DelegateCommand(SendMessage);
    }

    private void SendMessage()
    {
        // 发布事件：所有订阅者都会收到
        _eventAggregator.GetEvent<MessageSentEvent>().Publish(Message);
    }
}
```

### 3. 事件订阅

#### MessageListViewModel.cs - 订阅者实现
```csharp
public class MessageListViewModel : BindableBase
{
    private IEventAggregator _eventAggregator;
    public ObservableCollection<string> Messages { get; set; }

    public MessageListViewModel(IEventAggregator eventAggregator)
    {
        _eventAggregator = eventAggregator;
        Messages = new ObservableCollection<string>();

        // 订阅事件：自动接收所有发布
        _eventAggregator.GetEvent<MessageSentEvent>().Subscribe(MessageReceived);
    }

    private void MessageReceived(string message)
    {
        Messages.Add(message);
    }
}
```

## 事件聚合架构

### 发布-订阅模式
```
ModuleA (发布者)          EventAggregator          ModuleB (订阅者)
     ↓                           ↓                         ↓
Publish(MessageSentEvent)  →  事件总线  →  Subscribe(MessageSentEvent)
     ↓                           ↓                         ↓
发送消息"Hello"              广播事件           接收消息"Hello"
```

### 生命周期管理
```csharp
// 自动注入：Prism自动管理IEventAggregator生命周期
public MessageViewModel(IEventAggregator eventAggregator)
{
    _eventAggregator = eventAggregator;
}
```

## 事件类型支持

| 事件类型 | 声明方式 | 适用场景 | 示例 |
|----------|----------|----------|------|
| **简单事件** | `PubSubEvent<string>` | 单值传递 | 消息通知 |
| **复杂事件** | `PubSubEvent<CustomEventArgs>` | 多值传递 | 状态变更 |
| **无参数事件** | `PubSubEvent` | 信号通知 | 刷新请求 |

## 松耦合通信价值

| 通信方式 | 耦合度 | 模块依赖 | 扩展性 | 适用场景 |
|----------|--------|----------|--------|----------|
| **直接调用** | 高耦合 | 强依赖 | 差 | 简单场景 |
| **接口注入** | 中耦合 | 接口依赖 | 中 | 服务调用 |
| **事件聚合** | **✅ 低耦合** | **✅ 零依赖** | **✅ 极高** | **✅ 模块化通信** |

## 实际应用场景

### 1. 跨模块通知
```csharp
// 用户登录事件
public class UserLoggedInEvent : PubSubEvent<UserInfo>
{
}

// 模块A：用户登录后发布
_eventAggregator.GetEvent<UserLoggedInEvent>().Publish(currentUser);

// 模块B、C、D：自动接收登录通知
_eventAggregator.GetEvent<UserLoggedInEvent>().Subscribe(OnUserLoggedIn);
```

### 2. 状态同步
```csharp
// 配置变更事件
public class ConfigurationChangedEvent : PubSubEvent<ConfigData>
{
}

// 配置模块：发布变更
_eventAggregator.GetEvent<ConfigurationChangedEvent>().Publish(newConfig);

// 所有模块：自动同步新配置
_eventAggregator.GetEvent<ConfigurationChangedEvent>().Subscribe(UpdateConfiguration);
```

### 3. 广播消息
```csharp
// 系统消息事件
public class SystemMessageEvent : PubSubEvent<string>
{
}

// 任意模块：广播消息
_eventAggregator.GetEvent<SystemMessageEvent>().Publish("系统维护中...");

// 所有UI模块：显示系统消息
_eventAggregator.GetEvent<SystemMessageEvent>().Subscribe(ShowSystemMessage);
```

## 架构优势

**EventAggregator** 实现了：
- **零耦合通信**：发布者和订阅者完全独立
- **动态扩展**：运行时动态添加/移除订阅
- **类型安全**：强类型事件定义
- **生命周期管理**：自动清理和依赖注入

此示例代表了**模块化通信**的**事件驱动模式**，为**松耦合架构**奠定了**事件总线**的基础。