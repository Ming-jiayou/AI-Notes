# Stylet中sAction第四阶段：状态同步（INotifyPropertyChanged）实现详解

## 概述

在Stylet框架中，`s:Action`的第四阶段是**状态同步**，这个阶段的核心任务是通过`INotifyPropertyChanged`接口监听`CanSayHello`等条件属性的变化，自动更新按钮的可用状态。这个过程实现了ViewModel状态与UI控件的实时同步，是MVVM模式中数据绑定的关键机制。

## 核心机制：INotifyPropertyChanged

### 1. 接口定义与实现

`INotifyPropertyChanged`是.NET标准接口，定义在`System.ComponentModel`命名空间中：

```csharp
public interface INotifyPropertyChanged
{
    event PropertyChangedEventHandler PropertyChanged;
}
```

Stylet通过[`PropertyChangedBase`](Stylet/PropertyChangedBase.cs)类提供了默认实现。

### 2. Stylet中的实现

[`PropertyChangedBase`](Stylet/PropertyChangedBase.cs)提供了完整的`INotifyPropertyChanged`实现：

```csharp
public class PropertyChangedBase : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler PropertyChanged;
    
    protected virtual void NotifyOfPropertyChange([CallerMemberName] string propertyName = null)
    {
        this.PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}
```

## 状态同步架构

### 1. 监听机制设计

[`CommandAction`](Stylet/Xaml/CommandAction.cs:74)通过弱事件模式监听属性变化：

```csharp
private protected override void OnTargetChanged(object oldTarget, object newTarget)
{
    // 移除旧目标的监听
    if (oldTarget is INotifyPropertyChanged oldInpc)
        PropertyChangedEventManager.RemoveHandler(oldInpc, this.PropertyChangedHandler, this.guardName);

    // 设置新目标的监听
    if (newTarget is INotifyPropertyChanged inpc)
        PropertyChangedEventManager.AddHandler(inpc, this.PropertyChangedHandler, this.guardName);
}
```

### 2. 弱事件管理器

使用`PropertyChangedEventManager`避免内存泄漏：

```csharp
// 添加监听
PropertyChangedEventManager.AddHandler(viewModel, handler, "CanSayHello");

// 移除监听
PropertyChangedEventManager.RemoveHandler(viewModel, handler, "CanSayHello");
```

## 条件属性监听流程

### 1. 属性发现阶段

在[`OnTargetChanged`](Stylet/Xaml/CommandAction.cs:80)中查找条件属性：

```csharp
PropertyInfo guardPropertyInfo = newTarget?.GetType().GetProperty(this.guardName);
if (guardPropertyInfo != null && guardPropertyInfo.PropertyType == typeof(bool))
{
    // 找到条件属性，设置监听
    if (newTarget is INotifyPropertyChanged inpc)
    {
        PropertyChangedEventManager.AddHandler(inpc, this.PropertyChangedHandler, this.guardName);
    }
}
```

### 2. 属性变更处理

[`PropertyChangedHandler`](Stylet/Xaml/CommandAction.cs:106)处理属性变更：

```csharp
private void PropertyChangedHandler(object sender, PropertyChangedEventArgs e)
{
    this.UpdateCanExecute();
}
```

### 3. UI更新触发

[`UpdateCanExecute`](Stylet/Xaml/CommandAction.cs:111)触发UI更新：

```csharp
private void UpdateCanExecute()
{
    EventHandler handler = this.CanExecuteChanged;
    if (handler != null)
        Stylet.Execute.OnUIThread(() => handler(this, EventArgs.Empty));
}
```

## 线程安全机制

### 1. UI线程调度

使用`Stylet.Execute.OnUIThread`确保UI更新在UI线程执行：

```csharp
Stylet.Execute.OnUIThread(() => handler(this, EventArgs.Empty));
```

### 2. 跨线程调用处理

支持ViewModel在非UI线程触发属性变更：

```csharp
// ViewModel中
public async Task LoadDataAsync()
{
    IsBusy = true;
    await Task.Run(() => 
    {
        // 后台线程执行
        Thread.Sleep(1000);
    });
    IsBusy = false;  // 自动触发UI更新
}
```

## 实际应用示例

### 1. 基础状态同步

```csharp
public class ShellViewModel : PropertyChangedBase
{
    private string name;
    public string Name
    {
        get => name;
        set
        {
            name = value;
            NotifyOfPropertyChange();  // 触发Name变更
            NotifyOfPropertyChange(nameof(CanSayHello));  // 触发CanSayHello变更
        }
    }
    
    public void SayHello()
    {
        MessageBox.Show($"Hello, {Name}!");
    }
    
    public bool CanSayHello => !string.IsNullOrEmpty(Name);
}
```

### 2. 复杂条件逻辑

```csharp
public class DocumentViewModel : PropertyChangedBase
{
    private string content;
    private bool isModified;
    private bool isReadOnly;
    
    public string Content
    {
        get => content;
        set
        {
            if (content != value)
            {
                content = value;
                IsModified = true;
                NotifyOfPropertyChange();
            }
        }
    }
    
    public bool IsModified
    {
        get => isModified;
        set
        {
            isModified = value;
            NotifyOfPropertyChange();
            NotifyOfPropertyChange(nameof(CanSave));
            NotifyOfPropertyChange(nameof(CanClose));
        }
    }
    
    public bool IsReadOnly
    {
        get => isReadOnly;
        set
        {
            isReadOnly = value;
            NotifyOfPropertyChange();
            NotifyOfPropertyChange(nameof(CanSave));
            NotifyOfPropertyChange(nameof(CanEdit));
        }
    }
    
    public void Save()
    {
        // 保存逻辑
        IsModified = false;
    }
    
    public bool CanSave => IsModified && !IsReadOnly;
    
    public void Edit()
    {
        // 编辑逻辑
    }
    
    public bool CanEdit => !IsReadOnly;
}
```

### 3. 异步状态管理

```csharp
public class AsyncViewModel : PropertyChangedBase
{
    private bool isBusy;
    private string result;
    
    public bool IsBusy
    {
        get => isBusy;
        set
        {
            isBusy = value;
            NotifyOfPropertyChange();
            NotifyOfPropertyChange(nameof(CanExecuteAsyncCommand));
        }
    }
    
    public string Result
    {
        get => result;
        set
        {
            result = value;
            NotifyOfPropertyChange();
            NotifyOfPropertyChange(nameof(CanExecuteAsyncCommand));
        }
    }
    
    public async Task ExecuteAsyncCommand()
    {
        IsBusy = true;
        try
        {
            Result = await SomeAsyncOperation();
        }
        finally
        {
            IsBusy = false;
        }
    }
    
    public bool CanExecuteAsyncCommand => !IsBusy && string.IsNullOrEmpty(Result);
}
```

## 高级特性

### 1. 多属性依赖

一个条件方法可以依赖多个属性：

```csharp
public bool CanSaveDocument => 
    IsDocumentLoaded && 
    IsModified && 
    !IsReadOnly && 
    HasValidContent;

// 任何相关属性变更时都需要通知
public string DocumentContent
{
    get => documentContent;
    set
    {
        documentContent = value;
        NotifyOfPropertyChange();
        NotifyOfPropertyChange(nameof(HasValidContent));
        NotifyOfPropertyChange(nameof(CanSaveDocument));
    }
}
```

### 2. 集合状态同步

监听集合变化影响命令状态：

```csharp
public class ListViewModel : PropertyChangedBase
{
    public BindableCollection<Item> Items { get; } = new BindableCollection<Item>();
    
    public ListViewModel()
    {
        Items.CollectionChanged += (s, e) => 
        {
            NotifyOfPropertyChange(nameof(CanRemoveItem));
            NotifyOfPropertyChange(nameof(CanClearItems));
        };
    }
    
    public void RemoveItem(Item item)
    {
        Items.Remove(item);
    }
    
    public bool CanRemoveItem => Items.Count > 0;
    
    public void ClearItems()
    {
        Items.Clear();
    }
    
    public bool CanClearItems => Items.Count > 0;
}
```

### 3. 嵌套属性监听

监听嵌套对象的属性变化：

```csharp
public class ParentViewModel : PropertyChangedBase
{
    private ChildViewModel child;
    
    public ChildViewModel Child
    {
        get => child;
        set
        {
            if (child != null)
                child.PropertyChanged -= OnChildPropertyChanged;
                
            child = value;
            
            if (child != null)
                child.PropertyChanged += OnChildPropertyChanged;
                
            NotifyOfPropertyChange();
            NotifyOfPropertyChange(nameof(CanExecuteChildCommand));
        }
    }
    
    private void OnChildPropertyChanged(object sender, PropertyChangedEventArgs e)
    {
        if (e.PropertyName == nameof(Child.IsValid))
        {
            NotifyOfPropertyChange(nameof(CanExecuteChildCommand));
        }
    }
    
    public void ExecuteChildCommand()
    {
        Child.DoSomething();
    }
    
    public bool CanExecuteChildCommand => Child?.IsValid == true;
}
```

## 性能优化

### 1. 批量通知

减少不必要的UI更新：

```csharp
public void UpdateMultipleProperties()
{
    // 批量更新，只触发一次通知
    Name = "New Name";
    Email = "new@email.com";
    Age = 25;
    
    // 手动触发相关命令状态更新
    NotifyOfPropertyChange(string.Empty);  // 通知所有属性变更
}
```

### 2. 条件通知

避免重复通知：

```csharp
private string name;
public string Name
{
    get => name;
    set
    {
        if (name != value)
        {
            name = value;
            NotifyOfPropertyChange();
            
            // 只在需要时通知命令状态
            if (string.IsNullOrEmpty(name) || string.IsNullOrEmpty(value))
            {
                NotifyOfPropertyChange(nameof(CanSayHello));
            }
        }
    }
}
```

## 调试与诊断

### 1. 日志记录

启用详细日志记录：

```csharp
// 在Bootstrapper中配置
LogManager.GetLogger = type => new TraceLogger(type);
```

### 2. 调试输出

在ViewModel中添加调试信息：

```csharp
public bool CanSave
{
    get
    {
        var canSave = IsModified && !IsReadOnly;
        Debug.WriteLine($"CanSave evaluated: {canSave} (IsModified: {IsModified}, IsReadOnly: {IsReadOnly})");
        return canSave;
    }
}
```

### 3. 设计时支持

在设计器中提供有意义的状态：

```csharp
public bool CanSayHello
{
    get
    {
        if (Execute.InDesignMode)
            return true;  // 设计时总是可用
            
        return !string.IsNullOrEmpty(Name);
    }
}
```

## 常见问题与解决方案

### 1. 命令状态不更新

**问题**：修改属性后命令状态未更新
**解决**：确保调用`NotifyOfPropertyChange(nameof(CanCommandName))`

### 2. 内存泄漏

**问题**：ViewModel未释放导致内存泄漏
**解决**：使用`PropertyChangedEventManager`弱事件模式

### 3. 跨线程访问

**问题**：非UI线程修改属性导致异常
**解决**：使用`Execute.OnUIThread`或确保属性变更在UI线程触发

## 总结

Stylet的第四阶段通过`INotifyPropertyChanged`实现了ViewModel与UI之间的状态同步，其设计特点包括：

1. **自动监听**：自动发现并监听条件属性变化
2. **弱事件模式**：避免内存泄漏
3. **线程安全**：支持跨线程状态更新
4. **高性能**：最小化UI更新开销
5. **调试友好**：详细的日志和调试支持
6. **灵活扩展**：支持复杂的状态依赖关系

这一机制使得开发者可以专注于业务逻辑，而无需手动管理UI状态更新，真正实现了MVVM模式的数据驱动UI。