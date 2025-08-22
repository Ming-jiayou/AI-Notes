# Stylet Execute 详细使用指南

## 概述

`Execute` 是 Stylet 框架中的一个核心工具类，提供了在 UI 线程上执行操作的便捷方法。它解决了 WPF 应用程序中常见的跨线程访问 UI 元素的问题，确保所有 UI 相关的操作都在正确的线程上执行。

## 核心功能

`Execute` 类主要提供以下功能：

1. **线程调度**：将操作调度到 UI 线程执行
2. **同步/异步执行**：支持同步和异步两种执行模式
3. **设计模式检测**：提供设计时模式检测功能
4. **默认属性变更调度器**：为 PropertyChanged 事件提供默认的线程调度机制

## 源码结构分析

### 主要类和方法

#### Execute 类
位于 `Stylet/Execute.cs`，是一个静态类，提供以下核心方法：

- `PostToUIThread(Action action)` - 异步投递到 UI 线程
- `PostToUIThreadAsync(Action action)` - 异步投递并返回 Task
- `OnUIThread(Action action)` - 在 UI 线程执行（同步或异步）
- `OnUIThreadSync(Action action)` - 同步执行并阻塞直到完成
- `OnUIThreadAsync(Action action)` - 异步执行并返回 Task
- `InDesignMode` - 设计模式检测属性

#### IDispatcher 接口
位于 `Stylet/IDispatcher.cs`，定义了调度器的基本契约：

```csharp
public interface IDispatcher
{
    void Post(Action action);      // 异步执行
    void Send(Action action);      // 同步执行
    bool IsCurrent { get; }        // 当前线程检查
}
```

#### 调度器实现

1. **ApplicationDispatcher**：使用 WPF 的 Dispatcher
2. **SynchronousDispatcher**：同步执行，主要用于单元测试

## 使用方法详解

### 1. 基本线程调度

#### 在 UI 线程执行操作
```csharp
// 确保操作在 UI 线程执行
Execute.OnUIThread(() => 
{
    // 更新 UI 元素
    this.SomeProperty = newValue;
    this.NotifyOfPropertyChange();
});
```

#### 异步执行不阻塞
```csharp
// 异步投递，不等待完成
Execute.PostToUIThread(() => 
{
    // 后台更新 UI
    StatusText = "操作完成";
});
```

#### 同步执行并等待
```csharp
// 阻塞直到 UI 线程操作完成
Execute.OnUIThreadSync(() => 
{
    // 需要立即完成的 UI 操作
    MainWindow.Title = "新标题";
});
```

### 2. 异步编程模式

#### 使用 Task 的异步模式
```csharp
public async Task LoadDataAsync()
{
    var data = await GetDataFromServerAsync();
    
    // 确保 UI 更新在正确线程
    await Execute.OnUIThreadAsync(() => 
    {
        Items.Clear();
        Items.AddRange(data);
    });
}
```

#### 异常处理
```csharp
try
{
    await Execute.OnUIThreadAsync(() => 
    {
        // 可能抛出异常的 UI 操作
        throw new InvalidOperationException("测试异常");
    });
}
catch (Exception ex)
{
    // 处理异常
    logger.Error(ex, "UI 操作失败");
}
```

### 3. 设计模式检测

#### 避免设计时错误
```csharp
public class MyViewModel : Screen
{
    public MyViewModel()
    {
        // 避免设计时执行实际逻辑
        if (!Execute.InDesignMode)
        {
            InitializeRealData();
        }
        else
        {
            // 提供设计时数据
            InitializeDesignData();
        }
    }
}
```

### 4. 自定义调度器

#### 实现自定义 IDispatcher
```csharp
public class CustomDispatcher : IDispatcher
{
    public void Post(Action action)
    {
        // 自定义异步调度逻辑
        Task.Run(action);
    }

    public void Send(Action action)
    {
        // 自定义同步调度逻辑
        action();
    }

    public bool IsCurrent => Thread.CurrentThread.ManagedThreadId == 1;
}

// 使用自定义调度器
Execute.Dispatcher = new CustomDispatcher();
```

## 实际应用场景

### 场景1：后台线程更新 UI
```csharp
public class DataService
{
    public async Task ProcessDataAsync()
    {
        // 在后台线程处理数据
        var result = await Task.Run(() => HeavyComputation());
        
        // 回到 UI 线程更新结果
        Execute.OnUIThread(() => 
        {
            ResultText = $"处理完成：{result}";
            IsProcessing = false;
        });
    }
}
```

### 场景2：事件处理中的线程安全
```csharp
public class NetworkViewModel : Screen
{
    private NetworkClient _client;
    
    public NetworkViewModel()
    {
        _client = new NetworkClient();
        _client.MessageReceived += OnMessageReceived;
    }
    
    private void OnMessageReceived(object sender, MessageEventArgs e)
    {
        // 网络事件可能在非 UI 线程触发
        Execute.OnUIThread(() => 
        {
            Messages.Add(new MessageViewModel(e.Message));
            UnreadCount++;
        });
    }
}
```

### 场景3：批量 UI 更新
```csharp
public void UpdateMultipleProperties()
{
    Execute.OnUIThreadSync(() => 
    {
        // 确保所有属性更新在一个 UI 操作周期内完成
        using (this.DeferRefresh())
        {
            Name = "新名称";
            Age = 25;
            Address = "新地址";
            IsModified = true;
        }
    });
}
```

## 最佳实践

### 1. 避免死锁
```csharp
// 错误做法 - 可能导致死锁
public async Task BadPracticeAsync()
{
    // 不要阻塞等待 UI 线程操作
    Execute.OnUIThreadSync(() => SomeUIOperation());
    
    // 继续其他操作...
}

// 正确做法
public async Task GoodPracticeAsync()
{
    // 使用异步版本
    await Execute.OnUIThreadAsync(() => SomeUIOperation());
    
    // 或者使用 Post 不等待
    Execute.PostToUIThread(() => SomeUIOperation());
}
```

### 2. 性能优化
```csharp
// 批量操作减少线程切换
public void BatchUpdate(List<Item> items)
{
    Execute.OnUIThread(() => 
    {
        // 一次性完成所有更新
        Items.Clear();
        Items.AddRange(items);
    });
}

// 避免在循环中频繁调度
public void UpdateItemsIndividually(List<Item> items)
{
    // 错误做法
    foreach (var item in items)
    {
        Execute.OnUIThread(() => Items.Add(item)); // 频繁线程切换
    }
    
    // 正确做法
    Execute.OnUIThread(() => 
    {
        foreach (var item in items)
        {
            Items.Add(item);
        }
    });
}
```

### 3. 异常处理策略
```csharp
public async Task SafeUIUpdateAsync()
{
    try
    {
        await Execute.OnUIThreadAsync(() => 
        {
            // UI 更新逻辑
            UpdateUI();
        });
    }
    catch (TargetInvocationException ex)
    {
        // 处理 UI 线程中的异常
        logger.Error(ex.InnerException ?? ex, "UI 更新失败");
        
        // 可以显示用户友好的错误信息
        Execute.OnUIThread(() => 
        {
            ErrorMessage = "操作失败，请稍后重试";
        });
    }
}
```

## 与其他 Stylet 组件的集成

### 1. 与 PropertyChangedBase 集成
```csharp
public class MyViewModel : PropertyChangedBase
{
    public MyViewModel()
    {
        // 使用 Execute 的默认调度器
        this.PropertyChangedDispatcher = Execute.DefaultPropertyChangedDispatcher;
    }
}
```

### 2. 与 Screen 生命周期集成
```csharp
public class MyScreen : Screen
{
    protected override void OnActivate()
    {
        base.OnActivate();
        
        // 确保激活事件在 UI 线程处理
        Execute.OnUIThread(() => 
        {
            LoadInitialData();
        });
    }
}
```

### 3. 与 BindableCollection 集成
```csharp
public class MyViewModel : Screen
{
    public BindableCollection<Item> Items { get; private set; }
    
    public MyViewModel()
    {
        // BindableCollection 自动使用 Execute 进行线程安全操作
        Items = new BindableCollection<Item>();
    }
    
    public void AddItem(Item item)
    {
        // 自动在 UI 线程执行
        Items.Add(item);
    }
}
```

## 测试注意事项

### 单元测试中的使用
```csharp
[TestClass]
public class MyViewModelTests
{
    [TestInitialize]
    public void Setup()
    {
        // 测试时使用同步调度器
        Execute.Dispatcher = SynchronousDispatcher.Instance;
    }
    
    [TestMethod]
    public void PropertyUpdate_ShouldWorkInTest()
    {
        var vm = new MyViewModel();
        
        // 现在可以直接测试，无需担心线程问题
        vm.UpdateProperty("test");
        
        Assert.AreEqual("test", vm.Property);
    }
}
```

## 总结

`Execute` 类是 Stylet 框架中处理线程调度的核心工具，它提供了：

1. **简洁的 API**：通过静态方法提供易用的线程调度功能
2. **多种执行模式**：支持同步、异步、阻塞和非阻塞等多种模式
3. **线程安全**：确保 UI 操作始终在正确的线程上执行
4. **测试友好**：通过可替换的调度器支持单元测试
5. **框架集成**：与 Stylet 的其他组件深度集成

正确使用 `Execute` 可以大大简化 WPF 应用程序中的线程管理，避免常见的跨线程访问异常，提高代码的可维护性和可靠性。