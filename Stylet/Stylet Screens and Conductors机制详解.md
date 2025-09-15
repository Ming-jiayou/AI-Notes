# Stylet中的Screens and Conductors机制详解

在Stylet MVVM框架中，`Screens` 和 `Conductors` 是实现复杂用户界面逻辑、生命周期管理和ViewModel组合的核心概念。理解它们的设计与实现对于高效地使用Stylet至关重要。本文将结合源码，详细解析这些机制。

## 1. `IScreen` 和 `Screen`：ViewModel的基础

在Stylet中，任何具有生命周期和视图的ViewModel都应该实现`IScreen`接口。`Screen`类是`IScreen`的一个便利的基类实现。

### 1.1. `IScreen` 接口

`IScreen`接口聚合了多个更小的接口，每个接口都定义了ViewModel的一种行为：

- **`IViewAware`**: 表示ViewModel拥有一个关联的View。
  - `UIElement View { get; }`: 获取关联的视图。
  - `void AttachView(UIElement view)`: 将视图附加到ViewModel。
- **`IHaveDisplayName`**: 表示ViewModel有一个可供显示的名称（例如，窗口标题或选项卡标题）。
  - `string DisplayName { get; set; }`
- **`IScreenState`**: 定义了ViewModel的生命周期状态。
  - `ScreenState ScreenState { get; }`: 获取当前状态（`Active`, `Deactivated`, `Closed`）。
  - `bool IsActive { get; }`: 判断当前是否为`Active`状态。
  - `void Activate()`: 激活。
  - `void Deactivate()`: 停用。
  - `void Close()`: 关闭。
  - `event EventHandler<ScreenStateChangedEventArgs> StateChanged`: 状态变更时触发的事件。
- **`IChild`**: 表示ViewModel可以作为子级存在于一个父级（通常是Conductor）中。
  - `object Parent { get; set; }`: 获取或设置其父级对象。
- **`IGuardClose`**: 表示ViewModel可以“拒绝”被关闭。
  - `Task<bool> CanCloseAsync()`: 在关闭前被调用，以确定是否可以关闭。返回`false`将阻止关闭。
- **`IRequestClose`**: 表示ViewModel可以请求其父级关闭自己。
  - `void RequestClose(bool? dialogResult = null)`: 向父级发送关闭请求。

**源码: `Stylet/IScreen.cs`**
```csharp
public interface IScreen : IViewAware, IHaveDisplayName, IScreenState, IChild, IGuardClose, IRequestClose
{
}
```

### 1.2. `Screen` 基类

`Screen`类为`IScreen`接口提供了完整的实现，是大多数ViewModel的理想基类。它管理着状态转换，并提供了一系列可供重写的虚方法，以便在生命周期的关键节点执行自定义逻辑。

**关键生命周期方法:**

- `protected virtual void OnInitialActivate()`: 在ViewModel生命周期中第一次被激活时调用。非常适合执行一次性的初始化操作。
- `protected virtual void OnActivate()`: 每次被激活时调用。
- `protected virtual void OnDeactivate()`: 每次被停用时调用。
- `protected virtual void OnClose()`: 被关闭时调用。
- `public virtual Task<bool> CanCloseAsync()`: 重写此方法以实现自定义的关闭逻辑（例如，提示用户保存未保存的更改）。

**源码: `Stylet/Screen.cs`**
```csharp
public class Screen : ValidatingModelBase, IScreen
{
    // ... DisplayName, Parent, View properties ...

    public virtual ScreenState ScreenState { get; protected set; }
    public bool IsActive => this.ScreenState == ScreenState.Active;

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
            // ... fire events ...
        });
    }

    // ... Deactivate and Close implementations ...

    public virtual Task<bool> CanCloseAsync()
    {
        return Task.FromResult(this.CanClose());
    }

    public virtual void RequestClose(bool? dialogResult = null)
    {
        if (this.Parent is IChildDelegate conductor)
        {
            conductor.CloseItem(this, dialogResult);
        }
        // ... error handling ...
    }
}
```
`RequestClose`方法尤其重要，它通过`Parent`属性找到其所属的Conductor，并调用Conductor的`CloseItem`方法来关闭自己，实现了子ViewModel与父容器的解耦。

## 2. Conductors：ViewModel的管理者

Conductor本身也是一个`Screen`，但它的特殊职责是管理一个或多个其他的`Screen`对象（称为`Items`或`Children`）。它负责决定哪个子`Screen`是活动的，并控制它们的生命周期。

### 2.1. `ConductorBase<T>`

这是所有Conductor的抽象基类，它定义了Conductor的核心API。

**源码: `Stylet/ConductorBase.cs`**
```csharp
public abstract class ConductorBase<T> : Screen, IConductor<T>, IParent<T> where T : class
{
    public virtual bool DisposeChildren { get; set; } = true;

    public abstract IEnumerable<T> GetChildren();
    public abstract void ActivateItem(T item);
    public abstract void DeactivateItem(T item);
    public abstract void CloseItem(T item);

    protected virtual void EnsureItem(T newItem)
    {
        if (newItem is IChild newItemAsChild && newItemAsChild.Parent != this)
            newItemAsChild.Parent = this;
    }

    // ... CanCloseItem, CanAllItemsCloseAsync ...
}
```
- `ActivateItem`: 激活一个子项。具体行为取决于Conductor的类型。
- `CloseItem`: 关闭一个子项。
- `EnsureItem`: 确保子项的`Parent`属性被正确设置为当前Conductor。

当一个Conductor被激活或停用时，它会自动管理其子项的生命周期状态。例如，当Conductor被停用时，其活动的子项也会被停用。

Stylet提供了几种内置的Conductor实现，以适应不同的UI场景。

### 2.2. `Conductor<T>`

这是最简单的Conductor，它只管理**一个**活动的`Screen`。当`ActivateItem`被调用时，它会先尝试关闭当前的活动项，然后再激活新的项。

**源码: `Stylet/Conductor.cs`**
```csharp
public partial class Conductor<T> : ConductorBaseWithActiveItem<T> where T : class
{
    public override async void ActivateItem(T item)
    {
        if (item != null && item.Equals(this.ActiveItem))
        {
            if (this.IsActive)
                ScreenExtensions.TryActivate(item); // Re-activate if it's the same item
        }
        else if (await this.CanCloseItem(this.ActiveItem)) 
        {
            this.ChangeActiveItem(item, true); // Close the old one, activate the new one
        }
    }
    // ...
}
```
**用途**: 适用于主窗口或区域，在任何时候只显示一个主内容ViewModel的场景。例如，一个`ShellViewModel`可以继承自`Conductor<IScreen>`来管理其主内容区域。

### 2.3. `Conductor<T>.Collection.OneActive`

这个Conductor管理一个`Screen`**集合**（通过`Items`属性暴露），但在任何时候只有一个`Screen`是活动的（通过`ActiveItem`属性暴露）。

**源码: `Stylet/ConductorOneActive.cs`**
```csharp
public class OneActive : ConductorBaseWithActiveItem<T>
{
    public IObservableCollection<T> Items { get; }

    public override void ActivateItem(T item)
    {
        if (item != null && item.Equals(this.ActiveItem))
        {
            if (this.IsActive)
                ScreenExtensions.TryActivate(this.ActiveItem);
        }
        else
        {
            this.ChangeActiveItem(item, false); // Don't close the old one, just deactivate it
        }
    }

    public override async void CloseItem(T item)
    {
        // ...
        if (item.Equals(this.ActiveItem))
        {
            // Determine the next item to activate
            T nextItem = this.DetermineNextItemToActivate(item);
            this.ChangeActiveItem(nextItem, false);
        }
        this.items.Remove(item); // Removing from the collection will trigger cleanup
    }
}
```
**用途**: 这是实现**选项卡式界面（Tab Control）**的完美选择。`Items`集合可以绑定到`TabControl`的`ItemsSource`，而`ActiveItem`可以双向绑定到`SelectedItem`。当用户切换选项卡时，Stylet会自动停用旧的ViewModel并激活新的ViewModel。

### 2.4. `Conductor<T>.Collection.AllActive`

这个Conductor也管理一个`Screen`**集合**，但它的所有子项**始终都是活动的**。当Conductor被激活时，它会激活所有子项。

**源码: `Stylet/ConductorAllActive.cs`**
```csharp
public class AllActive : ConductorBase<T>
{
    public IObservableCollection<T> Items { get; }

    protected override void OnActivate()
    {
        foreach (IScreenState item in this.items.OfType<IScreenState>())
        {
            item.Activate();
        }
    }

    public override void ActivateItem(T item)
    {
        this.EnsureItem(item); // Adds to the collection if not present

        if (this.IsActive)
            ScreenExtensions.TryActivate(item);
    }
    // ...
}
```
**用途**: 适用于需要同时显示和管理多个独立活动面板的场景，例如仪表盘（Dashboard）或某些主从视图（Master-Detail）。

### 2.5. `Conductor<T>.StackNavigation`

这个Conductor实现了一个基于**栈**的导航模型。它有一个`ActiveItem`，并维护一个历史记录栈。当`ActivateItem`被调用时，旧的`ActiveItem`会被压入历史栈中。

**源码: `Stylet/ConductorNavigating.cs`**
```csharp
public class StackNavigation : ConductorBaseWithActiveItem<T>
{
    private readonly List<T> history = new();

    public override void ActivateItem(T item)
    {
        // ...
        if (this.ActiveItem != null)
            this.history.Add(this.ActiveItem);
        this.ChangeActiveItem(item, false);
    }

    public void GoBack()
    {
        this.CloseItem(this.ActiveItem);
    }

    public override async void CloseItem(T item)
    {
        // ...
        if (item.Equals(this.ActiveItem))
        {
            var newItem = default(T);
            if (this.history.Count > 0)
            {
                newItem = this.history.Last();
                this.history.RemoveAt(this.history.Count - 1);
            }
            this.ChangeActiveItem(newItem, true); // Close the old item, activate the one from history
        }
        // ...
    }
}
```
**用途**: 非常适合创建向导（Wizard）界面或任何需要“后退”按钮的线性导航流程。

## 3. 用法示例

### ShellViewModel (using `Conductor<T>`)
```csharp
public class ShellViewModel : Conductor<IScreen>
{
    private readonly Page1ViewModel page1;
    private readonly Page2ViewModel page2;

    public ShellViewModel(Page1ViewModel page1, Page2ViewModel page2)
    {
        this.page1 = page1;
        this.page2 = page2;

        // Activate the first page initially
        this.ActivateItem(this.page1);
    }

    public void ShowPage1()
    {
        this.ActivateItem(this.page1);
    }

    public void ShowPage2()
    {
        this.ActivateItem(this.page2);
    }
}
```
在`ShellView.xaml`中，你可以有一个`ContentControl`来显示活动的内容：
```xml
<ContentControl s:View.Model="{Binding ActiveItem}" />
```

### TabbedViewModel (using `Conductor<T>.Collection.OneActive`)
```csharp
public class TabbedViewModel : Conductor<IScreen>.Collection.OneActive
{
    public TabbedViewModel(Tab1ViewModel tab1, Tab2ViewModel tab2)
    {
        this.Items.Add(tab1);
        this.Items.Add(tab2);

        // Activate the first tab
        this.ActivateItem(tab1);
    }

    public void AddNewTab()
    {
        var newTab = new Tab3ViewModel();
        this.Items.Add(newTab);
        this.ActivateItem(newTab);
    }
}
```
在`TabbedView.xaml`中，可以这样绑定到`TabControl`:
```xml
<TabControl ItemsSource="{Binding Items}" SelectedItem="{Binding ActiveItem, Mode=TwoWay}" DisplayMemberPath="DisplayName" />
```

## 总结

Stylet的`Screens`和`Conductors`提供了一个强大而灵活的框架来构建复杂的WPF应用程序。
- **`Screen`** 是构建块，封装了ViewModel的状态和行为。
- **`Conductor`** 是协调器，负责管理一个或多个`Screen`的生命周期和激活状态。
- 通过选择不同类型的**`Conductor`**（`Conductor`, `OneActive`, `AllActive`, `StackNavigation`），开发者可以轻松实现各种常见的UI模式，如主内容切换、选项卡、仪表盘和向导等，同时保持代码的清晰和解耦。
