# 03-CustomRegions 学习笔记 - 自定义区域适配器开发

## 项目概述

03-CustomRegions 是 Prism WPF Samples 中的第三个示例，它在前两个示例的基础上引入了 **自定义区域适配器（Custom Region Adapter）** 的高级概念。这个示例不仅展示了如何扩展 Prism 的标准区域功能，更重要的是为开发者提供了深度定制 UI 布局和行为的能力。

## 核心突破
- **✅ 自定义区域**：为 StackPanel 控件创建专用区域适配器
- **✅ 行为定制**：实现独特的视图添加/移除逻辑
- **✅ 框架扩展**：展现 Prism 的可扩展架构设计

## 项目结构

```
03-CustomRegions/
├── Regions.sln
└── Regions/
    ├── App.config
    ├── App.xaml
    ├── App.xaml.cs
    ├── Regions.csproj
    ├── Views/
    │   ├── MainWindow.xaml
    │   └── MainWindow.xaml.cs
    └── Prism/
        └── StackPanelRegionAdapter.cs
```

## 关键概念：区域适配器架构

### 什么是区域适配器（Region Adapter）？

Prism 的区域适配器是一个 **桥梁模式** 的实现，它负责：
- **连接**：将标准 WPF 控件连接到 Prism 的 Region 系统
- **翻译**：在控件特定行为和 Prism 区域行为之间进行转换
- **扩展**：为控件添加区域感知能力

### 区域适配器继承体系

```
IRegionAdapter (接口)
    |
    +-- RegionAdapterBase<T> (抽象基类)
        |
        +-- ContentControlRegionAdapter (内置)
        +-- ItemsControlRegionAdapter (内置)  
        |
        +-- **自定义**StackPanelRegionAdapter (本示例)
```

## 代码深度分析

### 1. StackPanelRegionAdapter - 自定义适配器实现

```csharp
public class StackPanelRegionAdapter : RegionAdapterBase<StackPanel>
{
    public StackPanelRegionAdapter(IRegionBehaviorFactory regionBehaviorFactory)
        : base(regionBehaviorFactory)
    {
        // 构造函数注入标准行为工厂
    }

    protected override void Adapt(IRegion region, StackPanel regionTarget)
    {
        // ▼▼▼ 区域适配核心逻辑 ▼▼▼
        region.Views.CollectionChanged += (s, e) =>
        {
            if (e.Action == NotifyCollectionChangedAction.Add)
            {
                foreach (FrameworkElement element in e.NewItems)
                {
                    regionTarget.Children.Add(element);
                }
            }

            // TODO: 实现了添加逻辑，但移除逻辑未实现
        };
    }

    protected override IRegion CreateRegion()
    {
        // ▼▼▼ 区域类型策略 ▼▼▼
        return new AllActiveRegion();  // 所有视图都是激活状态
    }
}
```

#### 适配器三大核心方法

| 方法 | 用途 | 本例实现 |
|------|------|----------|
| **构造函数** | 注入行为工厂 | 标准模式 |
| **Adapt()** | 控件与区域连接 | StackPanel 的 Children 管理 |
| **CreateRegion()** | 区域类型创建 | AllActiveRegion - 多激活策略 |

#### 事件处理详解

```csharp
// 当区域视图集合发生变化时触发
region.Views.CollectionChanged += (s, e) =>
{
    // e.Action: 变化类型 (Add, Remove, Reset, Move, Replace)
    switch(e.Action)
    {
        case Add:
            // e.NewItems: 新增的视图集合
            foreach(var view in e.NewItems)
                regionTarget.Children.Add(view as UIElement);
            break;
            
        case Remove:
            // e.OldItems: 被移除的视图集合  
            // TODO: 从 Children 中移除对应子元素
            break;
            
        case Reset:
            // 清空 Children 集合
            regionTarget.Children.Clear(); 
            break;
    }
};
```

### 2. App.xaml.cs - 区域映射注册

```csharp
// ▼▼▼ 关键配置：注册自定义适配器 ▼▼▼
protected override void ConfigureRegionAdapterMappings(
    RegionAdapterMappings regionAdapterMappings)
{
    base.ConfigureRegionAdapterMappings(regionAdapterMappings);
    
    // 将 StackPanel 类型与自定义适配器关联
    regionAdapterMappings.RegisterMapping(
        typeof(StackPanel), 
        Container.Resolve<StackPanelRegionAdapter>());
}
```

**配置要点**：
- **`ConfigureRegionAdapterMappings`**：Prism 为自定义适配器提供的专用配置点
- **类型映射**：`StackPanel → StackPanelRegionAdapter`
- **生命周期管理**：通过容器创建，支持依赖注入

### 3. 区域使用方式的演进

#### 02-Regions vs 03-CustomRegions

| 特性 | 02-Regions | 03-CustomRegions |
|------|------------|------------------|
| **宿主控件** | `ContentControl` | `StackPanel` |
| **区域类型** | `SingleActiveRegion` | `AllActiveRegion` |
| **适配器** | 内置标准适配器 | 自定义适配器 |
| **行为特性** | 单视图显示 | 多视图堆叠 |
| **扩展性** | 受限 | 完全定制 |

#### 控件的区域适应性

| WPF 控件类型 | 内置适配器 | 需自定义适配器 |
|-------------|-----------|---------------|
| **ContentControl** | ✅ 是 | ❌ 否 |
| **ItemsControl** | ✅ 是 | ❌ 否 |
| **Selector** | ✅ 是 | ❌ 否 |
| **StackPanel** | ❌ 否 | ✅ 需要 |
| **DockPanel** | ❌ 否 | ✅ 需要 |
| **Grid** | ❌ 否 | ✅ 需要 |

### 4. MainWindow.xaml - 应用场景展示

```xml
<Window ...>
    <Grid>
        <!-- 关键变化：从ContentControl改为StackPanel -->
        <StackPanel prism:RegionManager.RegionName="ContentRegion" />
    </Grid>
</Window>
```

**场景价值**：
- **垂直布局**：StackPanel 提供自然垂直布局
- **多内容显示**：可同时显示多个视图（堆叠排列）
- **灵活组合**：支持不同大小的子视图

## 区域类型策略深度分析

### IRegion 的实现策略

Prism 提供了三种不同的区域实现策略：

```csharp
// 本示例使用的 AllActiveRegion
public class AllActiveRegion : Region
{
    // 特点：所有添加的视图都处于激活状态
    // 适用：展示多个视图同时活跃（如列表或堆叠容器）
}

public class SingleActiveRegion : Region
{
    // 特点：同一时间只有一个视图激活
    // 适用：单视图场景（如ContentControl）
}
```

| 区域策略 | 激活逻辑 | 适用场景 | 本例选择 |
|----------|----------|----------|----------|
| **SingleActiveRegion** | 单个激活 | 主内容、导航 | ❌ 不适合 |
| **AllActiveRegion** | 全部激活 | 列表、工具栏 | ✅ 理想选择 |
| **Region** | 基础实现 | 自定义策略 | ❌ 额外工作 |

## 扩展实现指南

### 1. 完整的 StackPanel 适配器

基于本示例，可以扩展为完整的生产级适配器：

```csharp
public class StackPanelRegionAdapter : RegionAdapterBase<StackPanel>
{
    private readonly Dictionary<object, UIElement> _viewElementMap = new();
    
    protected override void Adapt(IRegion region, StackPanel regionTarget)
    {
        region.Views.CollectionChanged += (s, e) =>
        {
            switch (e.Action)
            {
                case NotifyCollectionChangedAction.Add:
                    AddViews(regionTarget, e.NewItems);
                    break;
                    
                case NotifyCollectionChangedAction.Remove:
                    RemoveViews(regionTarget, e.OldItems);
                    break;
                    
                case NotifyCollectionChangedAction.Reset:
                    ClearViews(regionTarget);
                    break;
                    
                case NotifyCollectionChangedAction.Move:
                    ReorderViews(regionTarget, e.OldStartingIndex, e.NewStartingIndex);
                    break;
            }
        };
    }
    
    private void AddViews(StackPanel panel, IList newViews)
    {
        foreach (var view in newViews.Cast<FrameworkElement>())
        {
            _viewElementMap[view] = view;
            panel.Children.Add(view);
        }
    }
    // ... 其他方法实现
}
```

### 2. 其他自定义适配器模板

基于相同模式，可以快速创建其他适配器：

```csharp
// DockPanel 适配器
public class DockPanelRegionAdapter : RegionAdapterBase<DockPanel>
{
    protected override void Adapt(IRegion region, DockPanel regionTarget)
    {
        // DockPanel 特有的 Dock 属性支持
    }
}

// WrapPanel 适配器  
public class WrapPanelRegionAdapter : RegionAdapterBase<WrapPanel>
{
    protected override void Adapt(IRegion region, WrapPanel regionTarget)
    {
        // WrapPanel 的流式布局支持
    }
}
```

## 架构价值分析

### Prism 的扩展架构优势

```csharp
// Prism 的开放封闭原则（OCP）
public interface IRegionAdapterMapping
{
    void RegisterMapping(Type controlType, IRegionAdapter adapter);
}

// 开发者可以扩展任何控件
// ✅ 封闭：核心业务逻辑不可修改
// ✅ 开放：针对扩展点可以自由添加新功能
```

### 实际商业应用场景

| 场景类型 | 控件需求 | 适配器价值 |
|----------|----------|------------|
| **仪表盘** | Grid 的复杂布局 | 支持行列动态配置 |
| **工具栏** | StackPanel 垂直堆叠 | 支持多工具按钮 |
| **导航菜单** | TreeView 层级结构 | 支持菜单项动态管理 |

### 性能考量

区域适配器虽然是连接层，但需要注意：

- **内存泄漏**：事件订阅需正确清理  
- **性能开销**：大量视图时的渲染优化
- **线程安全**：跨线程访问的问题

## 与标准模式的比较

| 特性对比 | 标准模式 | 自定义适配器模式 |
|----------|----------|------------------|
| **开发成本** | 低 | 中等 |
| **灵活性** | 受限 | 极高 |
| **维护成本** | 零 | 需要同步维护 |
| **升级风险** | 低 | 框架升级需验证 |
| **性能调优** | 固定 | 可深度优化 |

## 学习收获总结

### 架构认知升级

通过定制区域适配器的学习，理解：

**✅ Prism 的扩展点设计**：明确的扩展入口和生命周期管理

**✅ 适配器模式应用**：WPF 控件与 Prism 区域的完美桥接

**✅ 生产级开发准备**：从玩具项目到企业级应用的跨越

### 技能进阶路径

```mermaid
graph LR
    A[标准区域] --> B[自定义适配器]
    B --> C[高级区域策略]
    C --> D[复合区域管理]
    D --> E[企业级区域架构]
```

本示例的学习标志着：
- **从使用者到扩展者** 的身份转变
- **从配置到定制** 的能力升级
- **从应用到架构** 的思维跃迁

### 实战价值

掌握了自定义区域适配器后，可以：

1. **适应任何 UI 需求**：任何 WPF 控件都可以区域化
2. **构建专业产品**：提供行业标准的用户体验
3. **扩展团队开发**：制定统一的区域适配标准
4. **架构迁移保障**：无缝集成现有 UI 组件库

## 总结

03-CustomRegions 通过学习自定义区域适配器的开发，揭示了 Prism 框架最具价值的特质：**开放且可控的扩展性**。它不仅是技术能力，更是架构思维的体现 - **如何让框架成为创意的助推器，而非限制的枷锁**。

这个示例为从示例学习迈向企业级开发提供了关键的"解锁钥匙"。