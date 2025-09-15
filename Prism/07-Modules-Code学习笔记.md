# 07-Modules-Code 学习笔记 - 代码式模块配置管理

## 项目概述

07-Modules-Code 展示了 **代码级模块配置** 的实现方式，通过`ConfigureModuleCatalog`方法实现模块的**程序化注册**。相比XAML配置，提供了更灵活的运行时模块管理能力。

## 项目结构

```
07-Modules-Code/
├── Modules/
│   ├── Modules.csproj
│   ├── App.xaml.cs (配置ModuleCatalog)
│   └── Views/
└── ModuleA/
    ├── ModuleA.csproj
    ├── ModuleAModule.cs (IModule实现)
    └── Views/
```

## 核心差异对比

| 配置方式 | 实现路径 | 适用场景 | 灵活性 |
|----------|----------|----------|--------|
| **XAML配置** | ModuleCatalog.xaml | 静态模块列表 | 低 |
| **代码配置** | ConfigureModuleCatalog | 动态模块加载 | **✅ 高** |

## 代码实现

### 1. 代码式模块注册

#### App.xaml.cs - ConfigureModuleCatalog重写
```csharp
public partial class App : PrismApplication
{
    protected override void ConfigureModuleCatalog(IModuleCatalog moduleCatalog)
    {
        // 直接代码注册模块
        moduleCatalog.AddModule<ModuleA.ModuleAModule>();
        
        // 支持条件注册
        // if (Environment.IsDevelopment)
        //     moduleCatalog.AddModule<DebugModule>();
    }
}
```

### 2. 标准IModule实现

#### ModuleAModule.cs - 相同实现
```csharp
public class ModuleAModule : IModule
{
    public void OnInitialized(IContainerProvider containerProvider)
    {
        var regionManager = containerProvider.Resolve<IRegionManager>();
        regionManager.RegisterViewWithRegion("ContentRegion", typeof(ViewA));
    }

    public void RegisterTypes(IContainerRegistry containerRegistry)
    {
        // 模块内部配置
    }
}
```

## 代码配置的优势

### 动态能力
- **运行时决定**：基于配置/环境决定加载哪些模块
- **条件加载**：debug/release环境选择不同模块

```csharp
protected override void ConfigureModuleCatalog(IModuleCatalog catalog)
{
    // 条件模块加载示例
    if (Configuration.GetSection("Modules")?.GetValue<bool>("EnableStatistics"))
        catalog.AddModule<StatisticsModule>();
        
    if (Configuration.GetSection("Modules")?.GetValue<bool>("EnableAnalytics"))
        catalog.AddModule<AnalyticsModule>();
}
```

### 开发便利性
- **调试友好**：IDE中直接管理模块依赖
- **重构支持**：重命名和移动的智能感知检查
- **编译时验证**：模块类型有效性检查

此示例是模块化开发的**代码驱动方案**，为复杂业务场景提供了灵活的模块管理能力。