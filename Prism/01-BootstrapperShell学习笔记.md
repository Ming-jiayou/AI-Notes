# 01-BootstrapperShell 学习笔记 - Prism WPF 基础启动示例

## 项目概述

01-BootstrapperShell 是 Prism WPF Samples 中的第一个示例，展示了使用 Prism 框架构建 WPF 应用程序的最基本结构。该示例重点演示了如何使用 Bootstrapper 来初始化和启动基于 Prism 的应用程序。

## 项目结构

```
01-BootstrapperShell/
├── BootstrapperShell.sln
└── BootstrapperShell/
    ├── App.config
    ├── App.xaml
    ├── App.xaml.cs
    ├── Bootstrapper.cs
    ├── BootstrapperShell.csproj
    └── Views/
        ├── MainWindow.xaml
        └── MainWindow.xaml.cs
```

## 核心组件解析

### 1. App.xaml

```xml
<Application x:Class="BootstrapperShell.App"
             xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
             xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
             xmlns:local="clr-namespace:BootstrapperShell">
    <Application.Resources>
    </Application.Resources>
</Application>
```

**特点亮点**：
- 这是一个标准的 WPF Application 定义
- 没有设置 StartupUri 属性，意味着应用程序的启动将由代码控制
- 为 Prism 框架的启动留下了完全的控制权

### 2. App.xaml.cs

```csharp
public partial class App : Application
{
    protected override void OnStartup(StartupEventArgs e)
    {
        base.OnStartup(e);

        var bootstrapper = new Bootstrapper();
        bootstrapper.Run();
    }
}
```

**关键分析**：
- 重写了 `OnStartup` 方法，这是 WPF 应用程序的入口点
- 创建并执行 Bootstrapper 实例，这是 Prism 框架的标准启动模式
- 将应用程序的控制权完全移交给 Prism 框架

### 3. Bootstrapper.cs

```csharp
class Bootstrapper : PrismBootstrapper
{
    protected override DependencyObject CreateShell()
    {
        return Container.Resolve<MainWindow>();
    }

    protected override void RegisterTypes(IContainerRegistry containerRegistry)
    {
    }
}
```

**核心概念**：

#### PrismBootstrapper
- Prism 提供的启动器基类，封装了框架初始化的所有复杂细节
- 自动处理模块发现、容器初始化、服务注册等基础工作

#### CreateShell 方法
- **作用**：创建应用程序的主窗口（Shell）
- **实现**：使用依赖注入容器解析 `MainWindow` 实例
- **重点**：返回的窗口将成为应用程序的主界面

#### RegisterTypes 方法
- **用途**：用于注册应用程序级别的类型和服务到依赖注入容器
- **当前状态**：此示例中为空实现，为后续扩展预留了空间
- **常见用途**：注册 ViewModels、服务接口、仓储等

### 4. MainWindow (Shell)

#### MainWindow.xaml
```xml
<Window x:Class="BootstrapperShell.Views.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        Title="Shell" Height="350" Width="525">
    <Grid>
        <ContentControl Content="Hello from Prism"  />
    </Grid>
</Window>
```

**设计特点**：
- 使用了 `ContentControl` 作为内容占位符
- 简单的问候文本展示了 Prism 应用的基本工作
- 窗口标题为"Shell"，符合 Prism 的术语规范

#### MainWindow.xaml.cs
```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
    }
}
```
- 这是一个标准的 WPF 窗口代码后置文件
- 没有包含任何业务逻辑，符合 MVVM 模式

### 5. 项目配置

#### BootstrapperShell.csproj
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>$(PrismTargetFramework)</TargetFramework>
    <AssemblyName>BootstrapperShell</AssemblyName>
    <UseWPF>true</UseWPF>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Prism.Unity" />
  </ItemGroup>
</Project>
```

**配置要点**：
- 使用 .NET SDK 风格的项目文件
- 配置为 Windows 可执行应用程序 (WinExe)
- 使用 PrismTargetFramework 变量指定目标框架
- 启用 WPF 支持
- 引用 Prism.Unity 包，表示使用 Unity 作为 IoC 容器

## 运行流程

1. **启动阶段**：
   - 应用程序从 `App` 类的 `OnStartup` 方法开始
   - 实例化并运行 `Bootstrapper`

2. **初始化阶段**：
   - `PrismBootstrapper` 初始化依赖注入容器
   - 调用 `RegisterTypes` 方法注册应用程序服务
   - 调用 `CreateShell` 方法创建主窗口

3. **显示阶段**：
   - 主窗口 (`MainWindow`) 被显示
   - 应用程序进入消息循环，等待用户交互

## 学习要点

### 1. Prism 架构核心理念
- **模块化**：虽然此示例没有展示模块功能，但所有基础结构已经就位
- **依赖注入**：通过容器管理对象的创建和生命周期
- **松耦合**：Shell 的创建通过 DI 容器完成，而非直接实例化

### 2. 与传统 WPF 启动方式的区别

| 传统 WPF | Prism WPF |
|----------|-----------|
| 在 App.xaml 设置 StartupUri | 重写 OnStartup 方法 |
| 直接创建 Window 实例 | 通过 Bootstrapper 启动 |
| 直接在 App.xaml.cs 中编写逻辑 | 通过继承 PrismBootstrapper 实现 |

### 3. 扩展准备
当前的 `RegisterTypes` 方法虽然为空，但已经为以下扩展做好了准备：
- 注册 ViewModels 到 Views 的映射
- 注册服务接口和实现
- 配置区域适配器
- 设置导航服务

## 示例意义

这个示例虽然简单，但它完美地展示了使用 Prism 框架的最小可行配置。开发者可以基于这个基础结构：

1. 添加模块化的业务功能
2. 实现 MVVM 模式
3. 使用区域和导航
4. 整合事件聚合器

这个"Hello World"级别的示例为后续所有复杂的 Prism 功能奠定了坚实基础。

## 总结

01-BootstrapperShell 是理解 Prism 框架工作原理的绝佳起点。它展示了：
- 如何使用 Bootstrapper 模式启动 WPF 应用程序
- 如何将传统的 StartupUri 方式替换为 Prism 的启动流程
- 如何正确设置基本的项目结构和依赖关系

掌握了这个示例后，开发者就可以继续探索 Prism 提供的更多高级功能，如模块化开发、导航、事件聚合等。