# 12-UsingCompositeCommands 学习笔记 - 复合命令架构

## 项目概述

12-UsingCompositeCommands 展示了 Prism 的 **CompositeCommand** 复合命令系统，实现了**跨模块命令聚合**，支持**全局操作**如"保存所有"、"刷新全部"等企业级应用场景。

## 项目结构

```
12-UsingCompositeCommands/
├── UsingCompositeCommands.Core/     # 共享命令接口
│   ├── IApplicationCommands.cs      # 全局命令契约
│   └── ApplicationCommands.cs       # CompositeCommand实现
├── UsingCompositeCommands/          # 主程序
│   └── ViewModels/
│       └── MainWindowViewModel.cs   # 全局命令消费者
├── ModuleA/                         # 功能模块
│   └── ViewModels/
│       └── TabViewModel.cs          # 子命令提供者
└── UsingCompositeCommands.sln
```

## 核心架构

### 1. 全局命令契约

#### ApplicationCommands.cs - 复合命令中枢
```csharp
public interface IApplicationCommands
{
    CompositeCommand SaveCommand { get; }
}

public class ApplicationCommands : IApplicationCommands
{
    private CompositeCommand _saveCommand = new CompositeCommand();
    public CompositeCommand SaveCommand
    {
        get { return _saveCommand; }
    }
}
```

### 2. 子命令注册

#### TabViewModel.cs - 模块内命令聚合
```csharp
public class TabViewModel : BindableBase
{
    private IApplicationCommands _applicationCommands;
    public DelegateCommand UpdateCommand { get; private set; }

    public TabViewModel(IApplicationCommands applicationCommands)
    {
        _applicationCommands = applicationCommands;
        
        // 创建本地命令
        UpdateCommand = new DelegateCommand(Update)
            .ObservesCanExecute(() => CanUpdate);
        
        // 注册到全局复合命令
        _applicationCommands.SaveCommand.RegisterCommand(UpdateCommand);
    }

    private void Update()
    {
        UpdateText = $"Updated: {DateTime.Now}";
    }
}
```

### 3. 全局命令消费

#### MainWindowViewModel.cs - 统一执行点
```csharp
public class MainWindowViewModel : BindableBase
{
    public IApplicationCommands ApplicationCommands { get; set; }

    public MainWindowViewModel(IApplicationCommands applicationCommands)
    {
        ApplicationCommands = applicationCommands;
        // 全局SaveCommand绑定到UI按钮
        // 执行时将触发所有注册的子命令
    }
}
```

## 复合命令工作原理

### 执行流程
```
用户点击"保存"按钮
    ↓
CompositeCommand.Execute() 被调用
    ↓
遍历所有注册的DelegateCommand
    ↓
对每个CanExecute为true的子命令执行Execute()
    ↓
所有模块的保存操作完成
```

### 生命周期管理
```csharp
// 自动管理子命令注册/注销
_compositeCommand.RegisterCommand(childCommand);
_compositeCommand.UnregisterCommand(childCommand);
```

## 架构优势

| 特性 | 单一命令 | 复合命令 | 企业价值 |
|------|----------|----------|----------|
| **作用范围** | 模块内 | 全局跨模块 | ✅ 统一操作 |
| **执行控制** | 独立 | 聚合管理 | ✅ 集中控制 |
| **模块解耦** | 紧耦合 | 松耦合 | ✅ 插件式架构 |
| **扩展性** | 困难 | 简单注册 | ✅ 动态扩展 |

## 实际应用场景

### 1. 全局保存
```csharp
// 所有打开的文档统一保存
public interface IDocumentCommands
{
    CompositeCommand SaveAllCommand { get; }
    CompositeCommand CloseAllCommand { get; }
}
```

### 2. 批量操作
```csharp
// 批量处理功能
public interface IBatchCommands
{
    CompositeCommand ProcessAllCommand { get; }
    CompositeCommand ValidateAllCommand { get; }
}
```

### 3. 系统级操作
```csharp
// 系统全局功能
public interface ISystemCommands
{
    CompositeCommand RefreshAllCommand { get; }
    CompositeCommand ResetAllCommand { get; }
}
```

## 模块化命令架构价值

**CompositeCommand** 实现了：
- **命令聚合**：多个子命令统一执行
- **模块解耦**：各模块独立注册，无需相互引用
- **动态扩展**：运行时动态添加/移除命令
- **集中控制**：统一入口管理分散功能

此示例代表了**企业级命令架构**，为**模块化应用**的**全局操作管理**奠定了**复合命令模式**的基础。