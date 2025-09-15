# 11-UsingDelegateCommands 学习笔记 - DelegateCommand命令系统

## 项目概述

11-UsingDelegateCommands 展示了 Prism 的 **DelegateCommand** 命令系统，实现了从传统 WPF 命令到**可观察命令**的升级，支持**属性观察**和**参数传递**等高级功能。

## 核心功能

### 1. 基础命令实现

#### MainWindowViewModel.cs - 四种命令模式
```csharp
public class MainWindowViewModel : BindableBase
{
    // 基础命令
    public DelegateCommand ExecuteDelegateCommand { get; private set; }
    
    // 泛型参数命令
    public DelegateCommand<string> ExecuteGenericDelegateCommand { get; private set; }
    
    // 属性观察命令
    public DelegateCommand DelegateCommandObservesProperty { get; private set; }
    
    // CanExecute观察命令
    public DelegateCommand DelegateCommandObservesCanExecute { get; private set; }

    public MainWindowViewModel()
    {
        ExecuteDelegateCommand = new DelegateCommand(Execute, CanExecute);
        
        // 观察属性变化自动更新CanExecute
        DelegateCommandObservesProperty = new DelegateCommand(Execute, CanExecute)
            .ObservesProperty(() => IsEnabled);
        
        // 声明式CanExecute观察
        DelegateCommandObservesCanExecute = new DelegateCommand(Execute)
            .ObservesCanExecute(() => IsEnabled);
        
        // 泛型命令带参数
        ExecuteGenericDelegateCommand = new DelegateCommand<string>(ExecuteGeneric)
            .ObservesCanExecute(() => IsEnabled);
    }
}
```

### 2. 命令绑定界面

#### MainWindow.xaml - 完整命令演示
```xml
<Window ... prism:ViewModelLocator.AutoWireViewModel="True">
    <StackPanel>
        <!-- 控制CanExecute的状态 -->
        <CheckBox IsChecked="{Binding IsEnabled}" 
                  Content="Can Execute Command"/>
        
        <!-- 基础命令 -->
        <Button Command="{Binding ExecuteDelegateCommand}" 
                Content="DelegateCommand"/>
        
        <!-- 属性观察命令 -->
        <Button Command="{Binding DelegateCommandObservesProperty}" 
                Content="DelegateCommand ObservesProperty"/>
        
        <!-- CanExecute观察命令 -->
        <Button Command="{Binding DelegateCommandObservesCanExecute}" 
                Content="DelegateCommand ObservesCanExecute"/>
        
        <!-- 带参数命令 -->
        <Button Command="{Binding ExecuteGenericDelegateCommand}" 
                CommandParameter="Passed Parameter"
                Content="DelegateCommand Generic"/>
        
        <!-- 执行结果显示 -->
        <TextBlock Text="{Binding UpdateText}" FontSize="22"/>
    </StackPanel>
</Window>
```

## 命令类型对比

| 命令类型 | 声明方式 | 核心特性 | 适用场景 |
|----------|----------|----------|----------|
| **DelegateCommand** | `new DelegateCommand(execute, canExecute)` | 基础命令 | 简单操作 |
| **DelegateCommand<T>** | `new DelegateCommand<T>(execute)` | 参数支持 | 需要参数 |
| **ObservesProperty** | `.ObservesProperty(() => prop)` | 属性观察 | 自动CanExecute |
| **ObservesCanExecute** | `.ObservesCanExecute(() => condition)` | 条件观察 | 声明式条件 |

## 高级特性

### 1. 属性观察机制
```csharp
// 当IsEnabled变化时自动调用CanExecute
DelegateCommandObservesProperty = new DelegateCommand(Execute, CanExecute)
    .ObservesProperty(() => IsEnabled);
```

### 2. 声明式条件观察
```csharp
// 更简洁的条件声明
DelegateCommandObservesCanExecute = new DelegateCommand(Execute)
    .ObservesCanExecute(() => IsEnabled);
```

### 3. 参数传递
```csharp
// 支持任意类型参数
ExecuteGenericDelegateCommand = new DelegateCommand<string>(ExecuteGeneric)
    .ObservesCanExecute(() => IsEnabled);

private void ExecuteGeneric(string parameter)
{
    UpdateText = parameter; // "Passed Parameter"
}
```

## 与传统命令对比

| 特性对比 | ICommand | DelegateCommand | 优势 |
|----------|----------|-----------------|------|
| **CanExecute更新** | 手动通知 | 自动观察 | ✅ 智能更新 |
| **参数支持** | 需要转换 | 泛型直接支持 | ✅ 类型安全 |
| **属性观察** | 不支持 | 内置支持 | ✅ 响应式 |
| **生命周期** | 手动管理 | 自动管理 | ✅ 简化代码 |

## 实际应用价值

### 企业级场景
- **表单验证**：字段变化自动更新提交按钮状态
- **权限控制**：用户角色变化自动更新操作可用性
- **异步操作**：长时间运行任务的状态管理
- **条件流程**：基于业务状态的动态命令可用性

此示例代表了**现代MVVM命令系统**，为**响应式用户界面**奠定了**智能命令管理**的基础。