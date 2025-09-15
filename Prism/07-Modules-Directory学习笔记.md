# 07-Modules-Directory 学习笔记 - 基于目录的模块自动发现

## 项目概述

07-Modules-Directory 展示了 **目录扫描式模块发现**，通过 `DirectoryModuleCatalog` 自动扫描指定目录下的所有DLL文件，实现了**插件式架构**的终极形态 - **无需配置，自动发现**。

## 项目结构

```
07-Modules-Directory/
├── Modules/
├── ModuleA/
└── [运行时目录扫描核心]
```

## 核心实现

### 1. 目录扫描策略

#### App.xaml.cs - 目录发现集成
```csharp
public partial class App : PrismApplication
{
    protected override IModuleCatalog CreateModuleCatalog()
    {
        return new DirectoryModuleCatalog() 
            { ModulePath = @".\Modules" };
    }
}
```

### 2. 零配置实现

#### 目录结构模式
```
应用程序根目录/
├── Modules.exe (主程序)
└── Modules/    (扫描目录)
    ├── ModuleA.dll ✅ 自动发现
    ├── ModuleB.dll ✅ 自动发现
    └── PluginX.dll ✅ 自动发现
```

## 零配置架构价值

| 特征比较 | Directory vs 其他方式 |
|----------|---------------------|
| **配置需求** | ❌ 零配置 vs ✓ 需配置 |
| **扩展方式** | 复制DLL即可 vs 修改配置文件 |
| **部署便利** | 极高 vs 中等 |
| **运维成本** | 最低 vs 有成本 |

## 插件式架构核心

### 1. 自动发现机制
- **Assembly扫描**：扫描指定目录所有*.dll文件
- **IModule检测**：自动识别实现了IModule接口的类
- **动态加载**：运行时动态加载发现的模块

### 2. 生产部署优势
- **热插拔**：运行时Copy/Paste模块DLL即可生效
- **版本隔离**：独立更新单个模块不影响主程序
- **运维友好**：无需重启即可扩展功能

## 实际应用场景

```
生产部署模式：
└── Production/
    ├── MyApp.exe
    ├── Plugins/
    │   ├── Analytics.dll     (分析模块)
    │   ├── Reporting.dll     (报表模块)  
    │   └── AdvancedFeature.dll (高级功能)
    
只需新增DLL即可扩展功能 -> 完全零配置
```

此示例代表了**模块化架构的最终形态**，为**企业级插件系统**奠定了最佳实践基础。