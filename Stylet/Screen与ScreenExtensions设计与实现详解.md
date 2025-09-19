# Stylet Screen 与 ScreenExtensions 设计与实现详解

## 概述

Stylet 的 Screen 类是 MVVM 模式中 ViewModel 的基类实现，提供了屏幕生命周期管理、状态跟踪、视图关联等核心功能。ScreenExtensions 则提供了一系列便捷的扩展方法，简化了屏幕间的交互和生命周期管理。这两个类共同构成了 Stylet 框架中屏幕管理的基础。

## Screen 类设计分析

### 类层次结构

```csharp
public class Screen : ValidatingModelBase, IScreen
```

Screen 类继承自 `ValidatingModelBase`，实现了 `IScreen` 接口，提供了验证功能和屏幕生命周期管理的组合。

### 核心设计目标

1. **生命周期管理**：提供完整的屏幕激活、停用、关闭生命周期
2. **状态跟踪**：维护屏幕的当前状态
3. **视图关联**：管理与视图的关联关系
4. **父子关系**：支持屏幕间的层级关系
5. **关闭保护**：提供可关闭性检查机制
6. **日志支持**：集成日志功能便于调试

## Screen 类核心功能详解

### 1. 显示名称管理

```csharp
private string _displayName;

public string DisplayName
{
    get => this._displayName;
    set => this.SetAndNotify(ref this._displayName, value);
}
```

**设计特点：**
- 使用 `SetAndNotify` 确保属性变更通知
- 构造函数中默认设置为类的全名
- 支持窗口标题、标签页标题等场景

### 2. 屏幕状态管理

```csharp
private ScreenState _screenState = ScreenState.Deactivated;

public virtual ScreenState ScreenState
{
    get => this._screenState;
    protected set
    {
        if (this.SetAndNotify(ref this._screenState, value))
        {
            this.NotifyOfPropertyChange(nameof(this.IsActive));
        }
    }
}

public bool IsActive => this.ScreenState == ScreenState.Active;
```

**状态机设计：**
- 三种状态：`Active`、`Deactivated`、`Closed`
- 状态转换有严格的顺序要求
- 提供 `IsActive` 便捷属性
- 状态变更时触发相关事件

### 3. 生命周期方法

#### 状态转换核心方法

```csharp
protected virtual void SetState(ScreenState newState, Action<ScreenState, ScreenState> changedHandler)
{
    if (newState == this.ScreenState)
        return;

    ScreenState previousState = this.ScreenState;
    this.ScreenState = newState;

    this.logger.Info("Setting state from {0} to {1}", previousState, newState);

    this.OnStateChanged(previousState, newState);
    changedHandler(previousState, newState);

    EventHandler<ScreenStateChangedEventArgs> handler = this.StateChanged;
    if (handler != null)
        Execute.OnUIThread(() => handler(this, new ScreenStateChangedEventArgs(newState, previousState)));
}
```

**设计亮点：**
- 防止重复状态设置
- 完整的日志记录
- 线程安全的事件触发
- 可扩展的状态变更处理

#### 激活流程

```csharp
void IScreenState.Activate()
{
    this.SetState(ScreenState.Active, (oldState, newState) =>
    {
        bool isInitialActivate = !this.haveActivated;
        if (!this.haveActivated)
        {
            this.OnInitialActivate();
            this.haveActivated = true;
        }

        this.OnActivate();

        EventHandler<ActivationEventArgs> handler = this.Activated;
        if (handler != null)
            Execute.OnUIThread(() => handler(this, new ActivationEventArgs(oldState, isInitialActivate)));
    });
}
```

**生命周期管理：**
- 区分首次激活和后续激活
- 严格的激活顺序：`OnInitialActivate` → `OnActivate`
- 事件在 UI 线程上触发

#### 停用流程

```csharp
void IScreenState.Deactivate()
{
    // 避免从 Closed 直接到 Deactivated
    if (this.ScreenState == ScreenState.Closed)
        ((IScreenState)this).Activate();

    this.SetState(ScreenState.Deactivated, (oldState, newState) =>
    {
        this.OnDeactivate();

        EventHandler<DeactivationEventArgs> handler = this.Deactivated;
        if (handler != null)
            Execute.OnUIThread(() => handler(this, new DeactivationEventArgs(oldState)));
    });
}
```

**状态保护：**
- 防止非法状态转换
- 自动修正状态顺序

#### 关闭流程

```csharp
void IScreenState.Close()
{
    // 避免从 Activated 直接到 Closed
    if (this.ScreenState != ScreenState.Closed)
        ((IScreenState)this).Deactivate();

    this.View = null;
    this.haveActivated = false;

    this.SetState(ScreenState.Closed, (oldState, newState) =>
    {
        this.OnClose();

        EventHandler<CloseEventArgs> handler = this.Closed;
        if (handler != null)
            Execute.OnUIThread(() => handler(this, new CloseEventArgs(oldState)));
    });
}
```

**清理工作：**
- 断开视图关联
- 重置激活状态
- 确保正确的状态转换顺序

### 4. 视图关联管理

```csharp
public UIElement View { get; private set; }

void IViewAware.AttachView(UIElement view)
{
    if (this.View != null)
        throw new InvalidOperationException($"Tried to attach View {view.GetType().Name} to ViewModel {this.GetType().Name}, but it already has a view attached");

    this.View = view;

    this.logger.Info("Attaching view {0}", view);

    if (view is FrameworkElement viewAsFrameworkElement)
    {
        if (viewAsFrameworkElement.IsLoaded)
            this.OnViewLoaded();
        else
            viewAsFrameworkElement.Loaded += (o, e) => this.OnViewLoaded();
    }
}
```

**设计特点：**
- 防止重复附加视图
- 自动处理视图加载事件
- 提供视图加载回调方法

### 5. 关闭保护机制

```csharp
public virtual Task<bool> CanCloseAsync()
{
    return Task.FromResult(this.CanClose());
}

[Obsolete("This method is deprecated, please use CanCloseAsync() instead")]
protected virtual bool CanClose()
{
    return true; 
}
```

**异步设计：**
- 支持异步关闭检查
- 向后兼容同步版本
- 默认允许关闭

### 6. 父子关系管理

```csharp
private object _parent;

public object Parent
{
    get => this._parent;
    set => this.SetAndNotify(ref this._parent, value);
}
```

**用途：**
- 支持屏幕层级结构
- 用于关闭请求的路由
- 支持导体(Conductor)模式

### 7. 关闭请求机制

```csharp
public virtual void RequestClose(bool? dialogResult = null)
{
    if (this.Parent is IChildDelegate conductor)
    {
        this.logger.Info("RequstClose called. Conductor: {0}; DialogResult: {1}", conductor, dialogResult);
        conductor.CloseItem(this, dialogResult);
    }
    else
    {
        var e = new InvalidOperationException($"Unable to close ViewModel {this.GetType()} as it must have a conductor as a parent");
        this.logger.Error(e);
        throw e;
    }
}
```

**设计原则：**
- 必须通过父导体关闭
- 支持对话框结果传递
- 完整的错误处理和日志记录

## ScreenExtensions 扩展方法分析

### 设计哲学

ScreenExtensions 提供了一组便捷的扩展方法，遵循"尝试执行"模式，避免强制类型转换和空值检查。

### 基础生命周期扩展

#### TryActivate、TryDeactivate、TryClose

```csharp
public static void TryActivate(object screen)
{
    if (screen is IScreenState screenAsScreenState)
        screenAsScreenState.Activate();
}

public static void TryDeactivate(object screen)
{
    if (screen is IScreenState screenAsScreenState)
        screenAsScreenState.Deactivate();
}

public static void TryClose(object screen)
{
    if (screen is IScreenState screenAsScreenState)
        screenAsScreenState.Close();
}
```

**设计特点：**
- 安全的类型检查和转换
- 静默失败，不抛出异常
- 统一的命名模式

#### TryDispose

```csharp
public static void TryDispose(object screen)
{
    if (screen is IDisposable screenAsDispose)
        screenAsDispose.Dispose();
}
```

**资源管理：**
- 支持 IDisposable 模式
- 与生命周期管理集成

### 高级关联扩展

#### ActivateWith - 父子激活关联

```csharp
public static void ActivateWith(this IScreenState child, IScreenState parent)
{
    var weakChild = new WeakReference<IScreenState>(child);
    void Handler(object o, ActivationEventArgs e)
    {
        if (weakChild.TryGetTarget(out IScreenState strongChild))
            strongChild.Activate();
        else
            parent.Activated -= Handler;
    }

    parent.Activated += Handler;
}
```

**内存管理：**
- 使用弱引用防止内存泄漏
- 自动清理失效的引用
- 事件处理器自动移除

#### DeactivateWith - 父子停用关联

```csharp
public static void DeactivateWith(this IScreenState child, IScreenState parent)
{
    var weakChild = new WeakReference<IScreenState>(child);
    void Handler(object o, DeactivationEventArgs e)
    {
        if (weakChild.TryGetTarget(out IScreenState strongChild))
            strongChild.Deactivate();
        else
            parent.Deactivated -= Handler;
    }

    parent.Deactivated += Handler;
}
```

#### CloseWith - 父子关闭关联

```csharp
public static void CloseWith(this IScreenState child, IScreenState parent)
{
    var weakChild = new WeakReference<IScreenState>(child);
    void Handler(object o, CloseEventArgs e)
    {
        if (weakChild.TryGetTarget(out IScreenState strongChild))
            TryClose(strongChild);
        else
            parent.Closed -= Handler;
    }

    parent.Closed += Handler;
}
```

#### ConductWith - 完整生命周期关联

```csharp
public static void ConductWith(this IScreenState child, IScreenState parent)
{
    child.ActivateWith(parent);
    child.DeactivateWith(parent);
    child.CloseWith(parent);
}
```

**一站式解决方案：**
- 一次性建立完整的生命周期关联
- 简化父子屏幕的管理
- 统一的内存泄漏防护

## 设计优势

### 1. 生命周期管理
- **严格的状态机**：确保状态转换的正确性
- **事件驱动**：通过事件通知生命周期变化
- **可扩展性**：虚方法允许自定义生命周期行为

### 2. 内存管理
- **弱引用使用**：防止扩展方法中的内存泄漏
- **自动清理**：自动移除失效的事件处理器
- **视图清理**：关闭时断开视图关联

### 3. 错误处理
- **日志集成**：完整的生命周期日志记录
- **异常安全**：防止非法状态转换
- **验证机制**：参数验证和状态检查

### 4. 灵活性
- **多接口支持**：同时支持多个核心接口
- **可重写方法**：允许自定义生命周期行为
- **扩展方法**：提供便捷的操作方式

## 使用模式

### 基本屏幕实现

```csharp
public class MyViewModel : Screen
{
    protected override void OnActivate()
    {
        base.OnActivate();
        // 激活时的初始化逻辑
    }

    protected override void OnDeactivate()
    {
        base.OnDeactivate();
        // 停用时的清理逻辑
    }
}
```

### 父子屏幕关联

```csharp
public class ParentViewModel : Screen
{
    private ChildViewModel child;
    
    public ParentViewModel()
    {
        this.child = new ChildViewModel();
        this.child.ConductWith(this); // 建立完整生命周期关联
    }
}
```

### 关闭确认

```csharp
public class ConfirmCloseViewModel : Screen
{
    public override async Task<bool> CanCloseAsync()
    {
        var result = await this.dialogManager.ShowMessageBox("确定要关闭吗？");
        return result == MessageBoxResult.Yes;
    }
}
```

## 最佳实践

1. **生命周期方法重写**：
   - 总是调用基类方法
   - 在适当的生命周期阶段执行相应操作
   - 避免在生命周期方法中执行耗时操作

2. **状态管理**：
   - 依赖内置的状态管理，不要手动设置状态
   - 使用 `IsActive` 属性检查激活状态
   - 监听相关事件响应状态变化

3. **内存管理**：
   - 使用扩展方法建立父子关系
   - 及时取消事件订阅
   - 在关闭时清理资源

4. **错误处理**：
   - 在 `CanCloseAsync` 中处理用户确认
   - 记录重要的生命周期事件
   - 适当处理异常情况

## 总结

Stylet 的 Screen 和 ScreenExtensions 类提供了完整的屏幕生命周期管理解决方案。Screen 类通过严格的状态机、丰富的事件系统和可扩展的设计，为 ViewModel 提供了强大的生命周期管理能力。ScreenExtensions 则通过智能的扩展方法和内存安全的实现，简化了屏幕间的关联管理。这两个类的设计体现了对 MVVM 模式、内存管理和用户体验的深度考虑，是构建复杂 WPF 应用程序的重要基础设施。