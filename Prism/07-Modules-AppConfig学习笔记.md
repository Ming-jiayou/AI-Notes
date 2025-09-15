# 07-Modules-AppConfig 学习笔记 - 基于App.config的模块配置

## 项目概述

07-Modules-AppConfig 展示了 **App.config配置文件** 的模块管理方式，通过 `ConfigurationModuleCatalog` 实现**配置文件驱动的模块发现**，适合运维友好的应用程序部署场景。

## 项目结构

```
07-Modules-AppConfig/
├── Modules/
│   ├── App.config (核心配置文件)
│   ├── Modules.csproj  
│   └── Views/
├── ModuleA/
│   ├── ModuleAModule.cs
│   └── Views/
└── Modules.sln
```

## 核心配置

### 1. App.config - 模块配置声明

```xml
<configuration>
  <configSections>
    <section name="modules" 
             type="Prism.Modularity.ModulesConfigurationSection, Prism.Wpf" />
  </configSections>
  
  <modules>
    <module assemblyFile="ModuleA.dll" 
            moduleType="ModuleA.ModuleAModule, ModuleA" 
            moduleName="ModuleAModule" 
            startupLoaded="True" />
  </modules>
</configuration>
```

### 2. App.xaml.cs - 配置驱动集成

```csharp
public partial class App : PrismApplication
{
    protected override IModuleCatalog CreateModuleCatalog()
    {
        return new ConfigurationModuleCatalog();
    }
}
```

## 配置驱动的运维价值

| 配置方式 | 实现文件 | 部署优势 | 运维价值 |
|----------|----------|----------|----------|
| **App.config** | App.config | 无需重新编译 | ✅ 运维友好 |
| **代码** | App.xaml.cs | 编译时加载 | 开发时便利 |
| **XAML** | ModuleCatalog.xaml | 配置分离 | 设计时清晰 |

## 运维场景应用

### 生产配置示例
```xml
<!-- 生产环境可根据需求配置 -->
<modules>
  <module assemblyFile="ModuleA.dll" moduleName="ModuleA" startupLoaded="True" />
  <module assemblyFile="StatsModule.dll" moduleName="Stats" startupLoaded="False" />
</modules>
```

无需重新编译即可调整模块加载策略，适合**部署后变更**的生产环境。