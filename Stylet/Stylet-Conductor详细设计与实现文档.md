# Stylet Conductor 详细设计与实现文档

## 概述

Conductor是Stylet框架中的核心组件之一，它实现了ViewModel的管理和生命周期控制。Conductor负责管理一个或多个子ViewModel（称为Screen），处理它们的激活、停用、关闭等状态转换，以及父子关系的维护。

## 设计目标

1. **统一管理**：提供一个统一的方式来管理多个ViewModel的生命周期
2. **状态控制**：精确控制子ViewModel的激活、停用和关闭状态
3. **导航支持**：支持不同的导航模式（单页面、多页面、堆栈导航等）
4. **生命周期协调**：确保父ViewModel和子ViewModel的生命周期协调一致
5. **资源管理**：自动处理子ViewModel的资源释放和清理

## 核心接口设计

### IParent<T> 接口
```csharp
public interface IParent<out T>
{
    IEnumerable<T> GetChildren();
}
```
- **功能**：定义父组件获取所有子组件的能力
- **用途**：用于遍历和管理子组件集合

### IHaveActiveItem<T> 接口
```csharp
public interface IHaveActiveItem<T>
{
    T ActiveItem { get; set; }
}
```
- **功能**：定义具有单个活动项的组件
- **用途**：用于需要跟踪当前活动子项的场景

### IChildDelegate 接口
```csharp
public interface IChildDelegate
{
    void CloseItem(object item, bool? dialogResult = null);
}
```
- **功能**：允许子组件请求关闭自身
- **用途**：实现子组件向父组件发送关闭请求的机制

### IConductor<T> 接口
```csharp
public interface IConductor<T>
{
    bool DisposeChildren { get; set; }
    void ActivateItem(T item);
    void DeactivateItem(T item);
    void CloseItem(T item);
}
```
- **功能**：定义管理子组件生命周期的核心能力
- **关键特性**：
  - 控制子组件的激活/停用
  - 管理子组件的关闭
  - 配置是否自动释放子组件资源

## 类层次结构

### 1. ConductorBase<T> - 抽象基类
**位置**：`Stylet/ConductorBase.cs`

**核心职责**：
- 实现IConductor<T>、IParent<T>、IChildDelegate接口
- 提供子组件生命周期管理的基础实现
- 处理关闭确认逻辑

**关键方法**：
- `EnsureItem(T newItem)`: 确保子组件准备就绪，设置父子关系
- `CanAllItemsCloseAsync(IEnumerable<T> itemsToClose)`: 检查所有子组件是否可以关闭
- `CanCloseItem(T item)`: 检查特定子组件是否可以关闭
- `CloseItem(object item, bool? dialogResult)`: 处理子组件的关闭请求

### 2. ConductorBaseWithActiveItem<T> - 带活动项的基类
**位置**：`Stylet/ConductorBaseWithActiveItem.cs`

**核心职责**：
- 扩展ConductorBase<T>，添加对单个活动项的支持
- 管理活动项的切换逻辑
- 协调父组件生命周期与子组件生命周期

**关键特性**：
- `ActiveItem`属性：当前活动的子组件
- `ChangeActiveItem(T newItem, bool closePrevious)`: 切换活动项
- 自动处理激活/停用状态同步

### 3. Conductor<T> - 简单导体
**位置**：`Stylet/Conductor.cs`

**特点**：
- 只管理单个活动项
- 激活新项时会关闭前一项
- 最简单的导体实现

### 4. Conductor<T>.Collection.OneActive - 集合单活动导体
**位置**：`Stylet/ConductorOneActive.cs`

**特点**：
- 管理多个子项的集合
- 同一时间只有一个活动项
- 支持Tab界面模式
- 自动处理集合变化（添加、删除、替换）

**关键功能**：
- `Items`集合：存储所有子项
- `DetermineNextItemToActivate(T itemToRemove)`: 智能选择下一个活动项
- 处理集合变化时的状态同步

### 5. Conductor<T>.Collection.AllActive - 集合全活动导体
**位置**：`Stylet/ConductorAllActive.cs`

**特点**：
- 管理多个子项的集合
- 所有子项同时保持活动状态
- 适用于MDI（多文档界面）模式
- 所有子项共享激活/停用状态

### 6. Conductor<T>.StackNavigation - 堆栈导航导体
**位置**：`Stylet/ConductorNavigating.cs`

**特点**：
- 实现类似浏览器的导航历史
- 支持后退导航
- 维护导航历史堆栈
- 支持清除历史记录

**关键功能**：
- `GoBack()`: 返回上一页
- `Clear()`: 清除导航历史
- 历史堆栈管理

## 生命周期管理

### 状态转换图
```
子组件状态：
[Inactive] → [Active] → [Deactivated] → [Closed]
```

### 生命周期协调
1. **父组件激活**：自动激活所有子组件
2. **父组件停用**：自动停用所有子组件
3. **父组件关闭**：自动关闭所有子组件
4. **子组件关闭**：自动从父组件中移除

### 资源管理
- **DisposeChildren属性**：控制是否自动释放子组件资源
- **CloseAndCleanUp方法**：统一处理关闭和资源清理
- **父子关系清理**：自动清除子组件对父组件的引用

## 使用模式与最佳实践

### 1. 简单单页面应用
```csharp
public class ShellViewModel : Conductor<IScreen>
{
    public ShellViewModel()
    {
        this.ActivateItem(new HomeViewModel());
    }
}
```

### 2. Tab界面模式
```csharp
public class ShellViewModel : Conductor<IScreen>.Collection.OneActive
{
    public ShellViewModel(Page1ViewModel page1, Page2ViewModel page2)
    {
        this.Items.Add(page1);
        this.Items.Add(page2);
        this.ActiveItem = page1;
    }
}
```

### 3. 导航模式
```csharp
public class ShellViewModel : Conductor<IScreen>.StackNavigation
{
    public void NavigateToPage<T>() where T : IScreen
    {
        var page = this.container.Get<T>();
        this.ActivateItem(page);
    }
    
    public void GoBack()
    {
        base.GoBack();
    }
}
```

### 4. 事件驱动的导航
```csharp
public class ShellViewModel : Conductor<IScreen>.Collection.OneActive, IHandle<OpenPageEvent>
{
    public void Handle(OpenPageEvent message)
    {
        var page = this.container.Get(message.PageType);
        this.ActivateItem(page);
    }
}
```

## 扩展点

### 1. 自定义关闭确认
```csharp
protected override async Task<bool> CanCloseItem(MyViewModel item)
{
    if (item.HasUnsavedChanges)
    {
        var result = await this.windowManager.ShowMessageBox("有未保存的更改，是否保存？", "确认", MessageBoxButton.YesNoCancel);
        return result == MessageBoxResult.Yes;
    }
    return true;
}
```

### 2. 自定义活动项选择
```csharp
protected override T DetermineNextItemToActivate(T itemToRemove)
{
    // 自定义逻辑选择下一个活动项
    return this.Items.FirstOrDefault(x => x != itemToRemove);
}
```

### 3. 自定义生命周期行为
```csharp
protected override void OnActivate()
{
    base.OnActivate();
    // 自定义激活逻辑
}

protected override void OnDeactivate()
{
    base.OnDeactivate();
    // 自定义停用逻辑
}
```

## 性能考虑

1. **延迟加载**：子组件的激活是按需进行的
2. **资源清理**：自动清理机制防止内存泄漏
3. **状态同步**：批量处理状态变化，减少UI更新次数
4. **集合优化**：使用BindableCollection提供高效的集合操作

## 线程安全

- **UI线程同步**：所有UI操作自动在UI线程执行
- **异步关闭**：支持异步关闭确认
- **状态一致性**：确保状态变化的原子性

## 错误处理

1. **关闭失败处理**：如果子组件拒绝关闭，保持当前状态
2. **异常传播**：生命周期事件中的异常会被适当处理
3. **资源泄漏防护**：即使出现异常也会尝试清理资源

## 测试建议

1. **生命周期测试**：验证激活/停用/关闭的顺序
2. **集合操作测试**：测试添加/删除/替换操作
3. **导航测试**：验证导航历史的正确性
4. **异常测试**：测试异常情况下的行为

## 总结

Stylet的Conductor系统提供了一个强大而灵活的框架来管理ViewModel的生命周期。通过不同的导体实现，可以支持从简单的单页面应用到复杂的导航系统的各种需求。其设计遵循了单一职责原则，每个导体都有明确的使用场景，同时提供了足够的扩展点来满足特定需求。

关键优势：
- 清晰的生命周期管理
- 多种导航模式支持
- 自动资源管理
- 良好的扩展性
- 与MVVM模式完美集成