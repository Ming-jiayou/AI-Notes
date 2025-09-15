# 13-IActiveAwareCommands 学习笔记 - 活动感知命令系统

## 项目概述

13-IActiveAwareCommands 展示了 **IActiveAware** 接口在命令系统中的应用，实现了**活动感知的复合命令**，只有**活动模块**的命令才会被执行，为**多文档界面(MDI)**和**标签页应用**提供了**智能命令管理**能力。

## 核心突破

### 与12示例的关键差异
- **12-UsingCompositeCommands**：所有注册命令都会执行
- **13-IActiveAwareCommands**：只有**活动(IActive)**的命令才会执行

## 核心实现

### 1. 活动感知命令

#### ApplicationCommands.cs - 启用活动监控
```csharp
public class ApplicationCommands : IApplicationCommands
{
    // 关键：构造函数参数true启用活动监控
    private CompositeCommand _saveCommand = new CompositeCommand(true);
    public CompositeCommand SaveCommand
    {
        get { return _saveCommand; }
    }
}
```

### 2. IActiveAware实现

#### TabViewModel.cs - 活动状态管理
```csharp
public class TabViewModel : BindableBase, IActiveAware
{
    private bool _isActive;
    public bool IsActive
    {
        get { return _isActive; }
        set
        {
            _isActive = value;
            OnIsActiveChanged();
        }
    }

    private void OnIsActiveChanged()
    {
        // 关键：将ViewModel的活动状态同步到命令
        UpdateCommand.IsActive = IsActive;
        IsActiveChanged?.Invoke(this, new EventArgs());
    }

    public event EventHandler IsActiveChanged;
}
```

## 活动感知架构

### 执行逻辑对比

| 场景 | 12-CompositeCommand | 13-IActiveAwareCommand | 行为差异 |
|------|-------------------|----------------------|----------|
| **3个标签页** | 执行3个命令 | 只执行**活动标签**命令 | ✅ 智能过滤 |
| **无活动项** | 执行所有 | 执行**0个**命令 | ✅ 安全保护 |
| **切换活动** | 不变 | 动态调整执行列表 | ✅ 实时响应 |

### 活动状态流转
```
用户切换标签页
    ↓
TabViewModel.IsActive = true/false
    ↓
UpdateCommand.IsActive = true/false  
    ↓
CompositeCommand只执行IsActive=true的命令
    ↓
只有活动模块的保存操作被执行
```

## 企业级应用场景

### 1. 多文档界面(MDI)
```csharp
// 只有当前活动的文档执行保存
public class DocumentViewModel : IActiveAware
{
    public DelegateCommand SaveCommand { get; private set; }
    
    public bool IsActive 
    { 
        get => _isActive; 
        set 
        {
            _isActive = value;
            SaveCommand.IsActive = value; // 只有活动文档可保存
        }
    }
}
```

### 2. 标签页编辑器
```csharp
// 只有活动的标签页响应全局命令
public class TabEditorViewModel : IActiveAware
{
    public DelegateCommand FormatCommand { get; private set; }
    public DelegateCommand FindReplaceCommand { get; private set; }
    
    // 活动状态自动同步到所有命令
    public bool IsActive 
    { 
        get => _isActive;
        set 
        {
            _isActive = value;
            FormatCommand.IsActive = value;
            FindReplaceCommand.IsActive = value;
        }
    }
}
```

### 3. 面板系统
```csharp
// 工具面板的活动状态管理
public class ToolPanelViewModel : IActiveAware
{
    public DelegateCommand RefreshCommand { get; private set; }
    
    public bool IsActive 
    { 
        get => _isActive;
        set => _isActive = value; // 自动同步到命令
    }
}
```

## 架构优势

| 维度 | 传统复合命令 | 活动感知命令 | 企业价值 |
|------|-------------|--------------|----------|
| **执行精度** | 全部执行 | 仅活动执行 | ✅ 精准控制 |
| **用户体验** | 可能误操作 | 智能过滤 | ✅ 直观友好 |
| **性能优化** | 冗余执行 | 按需执行 | ✅ 高效节能 |
| **状态管理** | 手动控制 | 自动同步 | ✅ 简化开发 |

## 实现要点

### 1. 启用活动监控
```csharp
// 关键：构造函数参数true
new CompositeCommand(true) // 启用IActiveAware支持
```

### 2. 状态同步
```csharp
// 必须手动同步ViewModel和Command的活动状态
public bool IsActive 
{ 
    get => _isActive;
    set 
    {
        _isActive = value;
        Command.IsActive = value; // 关键同步
    }
}
```

### 3. 事件通知
```csharp
// 触发活动状态变更通知
public event EventHandler IsActiveChanged;
```

此示例代表了**智能命令管理**的**高级形态**，为**复杂多模块应用**提供了**活动感知**的**精准命令控制**能力。