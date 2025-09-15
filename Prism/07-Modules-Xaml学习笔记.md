# 07-Modules-Xaml 学习笔记 - 基于XAML的模块化管理

## 项目概述

07-Modules-Xaml 是 Prism 模块化系列的第一个示例，展示了如何使用 **XAML配置文件** 进行模块发现与管理。这是从单体应用架构**向模块化架构**的转变，实现了真正的插件式开发。

## 项目结构

```
07-Modules-Xaml/
├── Modules/
│   └── (Shell 应用)
│       ├── Modules.csproj (主程序)
│       ├── ModuleCatalog.xaml (模块配置文件)
│       ├── App.xaml.cs (使用XamlModuleCatalog)
│       └── Views/
│           └── MainWindow.xaml
│
├── ModuleA/ (独立模块项目)
│   ├── ModuleAModule.cs (IModule实现)
│   ├── ModuleA.csproj  
│   └── Views/
│       └── ViewA.xaml
└── Modules.sln
```

## 核心机制

### 1. 声明式模块配置

#### ModuleCatalog.xaml
```xml
<m:ModuleCatalog xmlns="..." xmlns:m="...">
    <m:ModuleInfo ModuleName="ModuleAModule" 
                  ModuleType="ModuleA.ModuleAModule, ModuleA"/>
</m:ModuleCatalog>
```

#### App.xaml.cs 的模块集成
```csharp
protected override IModuleCatalog CreateModuleCatalog()
{
    return new XamlModuleCatalog(
        new Uri("/Modules;component/ModuleCatalog.xaml", 
        UriKind.Relative));
}
```

### 2. 模块独立实现

#### ModuleAModule.cs - IModule标准实现
```csharp
public class ModuleAModule : IModule
{
    public void OnInitialized(IContainerProvider containerProvider)
    {
        // 模块初始化时注册视图
        var regionManager = containerProvider.Resolve<IRegionManager>();
        regionManager.RegisterViewWithRegion("ContentRegion", typeof(ViewA));
    }

    public void RegisterTypes(IContainerRegistry containerRegistry)
    {
        // 模块内的类型注册
    }
}
```

### 3. Shell 标准架构

#### MainWindow.xaml 保持不变
```xml
<Window ...>
    <Grid>
        <ContentControl prism:RegionManager.RegionName="ContentRegion"/>
    </Grid>
</Window>
```

## 关键创新点

### 从单体到模块化的转变

| 对比维度 | 单体模式 | 模块化模式 | XAML配置优势 |
|----------|----------|------------|-------------|
| **项目结构** | 单项目 | 多项目 | 独立开发部署 |
| **模块发现** | 代码硬编码 | 配置发现 | 动态配置变更 |
| **编译依赖** | 强依赖 | 松散耦合 | 独立版本管理 |
| **维护性** | 复杂 | 简单 | 配置驱动 |

### 配置驱动的关键价值

1. **可配置性**：不需要重新编译即可调整模块列表
2. **可测试性**：独立模块可以完全单独测试
3. **可部署性**：模块可以独立更新，不影响主程序
4. **可扩展性**：新模块只需配置文件即可集成

## 运行流程

```
1. Modules启动 → 读取ModuleCatalog.xaml
2. 发现ModuleA → 加载ModuleA程序集
3. 执行ModuleAModule → 注册ModuleA.ViewA到"ContentRegion"
4. 完成集成 → 区域自动显示ViewA
```

这个示例体现了模块化开发的起点：**从架构设计到工程实践的蜕变**。