# Stylet Execute 类设计与实现详解

## 概述

Stylet 的 Execute 类是一个静态工具类，专门用于在 WPF 应用程序中处理 UI 线程调度。它提供了一系列方法来确保代码在正确的线程上执行，特别是在 MVVM 模式下处理属性变更通知和事件聚合器消息时。

## 设计目标

1. **线程安全调度**：确保 UI 操作在 UI 线程上执行
2. **性能优化**：智能判断当前线程，避免不必要的调度
3. **异步支持**：提供异步执行模式
4. **设计时检测**：支持设计时模式的检测
5. **可扩展性**：通过 IDispatcher 接口支持自定义调度器

## 核心组件

### IDispatcher 接口依赖

Execute 类依赖于 `IDispatcher` 接口来实现具体的线程调度功能：

```csharp
private static IDispatcher _dispatcher;

public static IDispatcher Dispatcher
{
    get => _dispatcher ?? SynchronousDispatcher.Instance;
    set => _dispatcher = value ?? throw new ArgumentNullException(nameof(Dispatcher));
}
```

这种设计体现了以下原则：

- **依赖注入**：通过属性注入调度器实现
- **默认实现**：提供同步调度器作为默认回退
- **空值保护**：防止设置 null 调度器

### 默认属性变更调度器

```csharp
public static Action<Action> DefaultPropertyChangedDispatcher { get; set; }

static Execute()
{
    DefaultPropertyChangedDispatcher = a => a();
}
```

- 专门用于属性变更通知的调度
- 默认为同步执行，可被自定义覆盖
- 静态构造函数确保初始化

## 方法详解

### 1. PostToUIThread - 异步发布

```csharp
public static void PostToUIThread(Action action)
{
    Dispatcher.Post(action);
}
```

**特点：**
- 始终异步执行，即使当前线程是 UI 线程
- 使用 `Dispatcher.Post` 方法
- 不等待操作完成

**使用场景：**
- 不需要等待结果的后台操作
- 避免阻塞当前线程

### 2. PostToUIThreadAsync - 异步发布并返回任务

```csharp
public static Task PostToUIThreadAsync(Action action)
{
    return PostOnUIThreadInternalAsync(action);
}
```

**特点：**
- 返回 `Task` 对象，可用于等待完成
- 内部调用私有辅助方法
- 文档警告不要阻塞等待此任务

**使用场景：**
- 需要知道操作何时完成
- 支持异步编程模式

### 3. OnUIThread - 智能 UI 线程执行

```csharp
public static void OnUIThread(Action action)
{
    if (Dispatcher.IsCurrent)
        action();
    else
        Dispatcher.Post(action);
}
```

**核心设计亮点：**
- **智能判断**：检查当前是否已在 UI 线程
- **性能优化**：避免不必要的线程切换
- **异步回退**：非 UI 线程时异步执行

**使用场景：**
- 最常用的 UI 线程调度方法
- 属性变更通知
- UI 元素更新

### 4. OnUIThreadSync - 同步 UI 线程执行

```csharp
public static void OnUIThreadSync(Action action)
{
    Exception exception = null;
    if (Dispatcher.IsCurrent)
    {
        action();
    }
    else
    {
        Dispatcher.Send(() =>
        {
            try
            {
                action();
            }
            catch (Exception e)
            {
                exception = e;
            }
        });

        if (exception != null)
            throw new System.Reflection.TargetInvocationException("An error occurred while dispatching a call to the UI Thread", exception);
    }
}
```

**特点：**
- **阻塞执行**：等待操作完成
- **异常处理**：捕获并重新抛出异常
- **线程安全**：正确处理跨线程异常传递

**使用场景：**
- 必须等待 UI 操作完成
- 需要异常信息的同步操作

### 5. OnUIThreadAsync - 异步 UI 线程执行

```csharp
public static Task OnUIThreadAsync(Action action)
{
    if (Dispatcher.IsCurrent)
    {
        action();
        return Task.FromResult(false);
    }
    else
    {
        return PostOnUIThreadInternalAsync(action);
    }
}
```

**智能优化：**
- 当前线程是 UI 线程时立即执行并返回已完成任务
- 避免非必要的任务创建和调度开销
- 保持异步接口一致性

### 6. 私有辅助方法 PostOnUIThreadInternalAsync

```csharp
private static Task PostOnUIThreadInternalAsync(Action action)
{
    var tcs = new TaskCompletionSource<object>();
    Dispatcher.Post(() =>
    {
        try
        {
            action();
            tcs.SetResult(null);
        }
        catch (Exception e)
        {
            tcs.SetException(e);
        }
    });
    return tcs.Task;
}
```

**实现细节：**
- 使用 `TaskCompletionSource` 创建可控制的 Task
- 完整的异常处理机制
- 统一的异步执行模式

## 设计时支持

### InDesignMode 属性

```csharp
public static bool InDesignMode
{
    get
    {
        if (inDesignMode == null)
        {
            var descriptor = DependencyPropertyDescriptor.FromProperty(DesignerProperties.IsInDesignModeProperty, typeof(FrameworkElement));
            inDesignMode = (bool)descriptor.Metadata.DefaultValue;
        }
        return inDesignMode.Value;
    }
    set => inDesignMode = value;
}
```

**功能：**
- 检测当前是否处于 Visual Studio 设计时
- 使用依赖属性描述符获取设计时状态
- 支持单元测试时的手动设置
- 延迟初始化，避免不必要的性能开销

## 设计优势

### 1. 性能优化
- **智能线程检测**：避免不必要的线程切换
- **延迟初始化**：设计时检测只在需要时进行
- **任务复用**：已完成任务的复用减少内存分配

### 2. 异常安全
- **跨线程异常传递**：正确处理调度过程中的异常
- **异常包装**：使用 `TargetInvocationException` 提供清晰的异常信息
- **异步异常处理**：在 Task 中正确设置异常状态

### 3. 灵活性
- **调度器抽象**：通过 IDispatcher 接口支持不同平台
- **多种执行模式**：同步、异步、阻塞、非阻塞等多种选择
- **可配置性**：允许自定义默认属性变更调度器

### 4. 线程安全
- **无状态设计**：静态方法不维护状态，避免并发问题
- **原子操作**：所有操作都是原子的，无需额外同步

## 使用模式

### 基本 UI 线程调度

```csharp
// 在 ViewModel 中更新 UI
Execute.OnUIThread(() => 
{
    this.SomeProperty = newValue;
});
```

### 异步操作

```csharp
// 异步执行 UI 操作
await Execute.OnUIThreadAsync(() => 
{
    // UI 相关操作
});
```

### 属性变更通知

```csharp
// 使用默认属性变更调度器
Execute.DefaultPropertyChangedDispatcher(() => 
{
    OnPropertyChanged(nameof(SomeProperty));
});
```

### 设计时检测

```csharp
// 避免设计时执行某些操作
if (!Execute.InDesignMode)
{
    // 运行时-only 的代码
}
```

## 最佳实践

1. **选择合适的调度方法**：
   - 需要立即更新 UI：使用 `OnUIThread`
   - 需要等待完成：使用 `OnUIThreadSync`
   - 异步操作：使用 `OnUIThreadAsync`

2. **避免阻塞等待**：
   - 不要在 UI 线程上阻塞等待异步操作
   - 使用 async/await 模式处理异步操作

3. **异常处理**：
   - 在调度的操作中包含适当的异常处理
   - 考虑跨线程异常的传递和包装

4. **性能考虑**：
   - 批量 UI 更新时考虑使用单次调度
   - 避免在频繁调用的代码路径中创建不必要的任务

## 总结

Stylet 的 Execute 类通过精心设计的 API 提供了强大而灵活的 UI 线程调度功能。它的智能线程检测、多种执行模式、完善的异常处理和设计时支持，使其成为 WPF MVVM 应用程序中处理线程问题的理想工具。其设计体现了对性能、可用性和可靠性的深度考虑，是 Stylet 框架中不可或缺的基础设施组件。