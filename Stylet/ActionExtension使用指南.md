# ActionExtension 使用指南  
> 一句话：把按钮、菜单、事件直接“指”到 ViewModel 的方法，不写 Command、不写事件处理器。

---

## 1. 它是干嘛的？

| 问题 | 传统做法 | ActionExtension 做法 |
|---|---|---|
| 按钮点击后要执行 ViewModel 方法 | ① 在 VM 里写 `ICommand` 属性<br>② XAML 绑定 `{Binding MyCommand}` | XAML 里直接写 `{s:Action MyMethod}` |
| 需要 CanExecute 逻辑 | 再写一个 `bool CanMyMethod` 并在命令里手动触发 `CanExecuteChanged` | 只要命名 `CanMyMethod`，自动关联 |
| 事件（如 MouseDoubleClick）要调 VM 方法 | 写 Code-behind 事件处理器 | 同样 `{s:Action MyMethod}` |

**结论**：`ActionExtension` 是一个 **MarkupExtension**，把“方法名”翻译成 **ICommand** 或 **Delegate**，自动完成绑定、可用性同步、异常处理。

---

## 2. 基本用法

### 2.1 绑定到按钮（最常见）

```xml
<!-- 引入 Stylet 标记前缀 -->
<Window ...
        xmlns:s="https://github.com/canton7/Stylet">

    <Button Content="保存"
            Command="{s:Action Save}" />
</Window>
```

对应 ViewModel：

```csharp
public class MainViewModel : Screen
{
    public void Save() { /* ... */ }
    public bool CanSave => !HasErrors;   // 可选，自动启用/禁用按钮
}
```

### 2.2 绑定到事件

```xml
<ListBox s:Action.Target="{Binding}"
         MouseDoubleClick="{s:Action OpenDetail}" />
```

- 事件处理器不需要参数，或接受 `(object sender, EventArgs e)`。
- 如果方法签名不符，运行时会抛出 `ActionSignatureInvalidException`。

---

## 3. 高级用法

### 3.1 指定显式目标

当控件不在常规视觉树（如 `ContextMenu`、`Popup`）时，需要手动指定 `ActionTarget`：

```xml
<ContextMenu>
    <MenuItem Header="删除"
              Command="{s:Action Delete}"
              s:View.ActionTarget="{Binding PlacementTarget.DataContext,
                                           RelativeSource={RelativeSource Self}}" />
</ContextMenu>
```

### 3.2 自定义异常行为

| 属性 | 说明 | 可选值 |
|---|---|---|
| `NullTarget` | ActionTarget 为 null 时 | `Enable` / `Disable` / `Throw`（默认） |
| `ActionNotFound` | 找不到方法时 | `Enable` / `Disable` / `Throw`（默认） |

示例：

```xml
<Button Command="{s:Action NonExistMethod,
                          NullTarget=Disable,
                          ActionNotFound=Disable}" />
```

### 3.3 带参数的命令

```xml
<Button Command="{s:Action Delete}"
        CommandParameter="{Binding SelectedItem}" />
```

ViewModel：

```csharp
public void Delete(Item item) { ... }
```

---

## 4. 命名约定

| 方法名 | 可选守卫属性 | 作用 |
|---|---|---|
| `Save()` | `CanSave` | 当 `CanSave == false` 时按钮自动禁用 |
| `FooAsync()` | `CanFooAsync` | 异步方法同样适用 |
| `DoWork(int x)` | `CanDoWork` | 支持参数，参数来自 `CommandParameter` |

---

## 5. 常见错误与排查

| 现象 | 原因 | 解决 |
|---|---|---|
| 按钮始终禁用 | 找不到 `CanXxx` 或返回 false | 检查属性名、拼写、返回值 |
| 点击按钮无反应 | ActionTarget 为 null | 在控件或父级设置 `s:View.ActionTarget="{Binding}"` |
| 运行时异常“找不到方法” | 方法签名不匹配 | 方法必须是 `public`、无参或 `(object p)` |
| 设计时异常 | 设计器没有 DataContext | 可忽略，或在 VM 构造函数加 `if (Execute.InDesignMode)` |

---

## 6. 完整示例

### 6.1 View

```xml
<Window x:Class="Demo.MainView"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:s="https://github.com/canton7/Stylet"
        Title="ActionExtension 示例" Height="200" Width="300">
    <StackPanel Margin="10">
        <TextBox Text="{Binding Name, UpdateSourceTrigger=PropertyChanged}" />
        <Button Content="打招呼"
                Command="{s:Action SayHello}"
                Margin="0,10,0,0" />
    </StackPanel>
</Window>
```

### 6.2 ViewModel

```csharp
public class MainViewModel : Screen
{
    private string _name = "";
    public string Name
    {
        get => _name;
        set
        {
            SetAndNotify(ref _name, value);
            NotifyOfPropertyChange(() => CanSayHello);
        }
    }

    public bool CanSayHello => !string.IsNullOrWhiteSpace(Name);

    public void SayHello()
    {
        MessageBox.Show($"你好，{Name}！");
    }
}
```

运行效果：  
- 文本框为空 → 按钮禁用  
- 输入文字 → 按钮启用，点击弹出消息

---

## 7. 小结

| 一句话总结 |
|---|
| `ActionExtension` 让你 **只写业务方法**，其余绑定、可用性、异常 **全部自动化**。 |

记住三件事即可：  
1. XAML 里写 `{s:Action 方法名}`  
2. ViewModel 里写 `public void 方法名()` 和可选 `public bool Can方法名`  
3. 确保视觉树或显式设置了 `s:View.ActionTarget="{Binding}"`