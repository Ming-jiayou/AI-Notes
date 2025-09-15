# 09-ChangeConvention 学习笔记 - 自定义ViewModel定位约定

## 项目概述

09-ChangeConvention 展示了如何**自定义ViewModelLocator的绑定约定**，通过`ConfigureViewModelLocator`重写默认的视图到视图模型映射规则，实现**企业级项目结构的灵活适配**。

## 核心变化

### 1. 项目结构调整

#### 非标准命名空间结构
```
ViewModelLocator/
├── Views/
│   └── MainWindow.xaml
└── Views/          ←←← ViewModel也在Views命名空间！
    └── MainWindowViewModel.cs
```

### 2. 自定义约定配置

#### App.xaml.cs - 约定重写
```csharp
protected override void ConfigureViewModelLocator()
{
    base.ConfigureViewModelLocator();

    ViewModelLocationProvider.SetDefaultViewTypeToViewModelTypeResolver((viewType) =>
    {
        var viewName = viewType.FullName;
        var viewAssemblyName = viewType.GetTypeInfo().Assembly.FullName;
        var viewModelName = $"{viewName}ViewModel, {viewAssemblyName}";
        return Type.GetType(viewModelName);
    });
}
```

## 约定对比分析

| 标准约定 | 本例自定义约定 | 适用场景 |
|----------|----------------|----------|
| `Views.MainWindow` → `ViewModels.MainWindowViewModel` | `Views.MainWindow` → `Views.MainWindowViewModel` | 扁平化项目结构 |
| 命名空间分离 | 命名空间统一 | 小型项目简化 |

## 企业级扩展模式

### 1. 多项目结构适配
```csharp
// 跨程序集ViewModel定位
ViewModelLocationProvider.SetDefaultViewTypeToViewModelTypeResolver((viewType) =>
{
    var viewName = viewType.FullName.Replace(".Views.", ".ViewModels.");
    var viewModelName = $"{viewName}ViewModel";
    return Type.GetType(viewModelName);
});
```

### 2. 前缀/后缀定制
```csharp
// 自定义命名规则
// View: CustomerListView -> ViewModel: CustomerListVM
ViewModelLocationProvider.SetDefaultViewTypeToViewModelTypeResolver((viewType) =>
{
    var viewName = viewType.Name.Replace("View", "VM");
    return Type.GetType($"MyApp.ViewModels.{viewName}");
});
```

## 实际价值

**约定灵活性** = **项目结构自由度**

- ✅ 适配现有项目结构
- ✅ 支持团队命名规范  
- ✅ 兼容遗留代码库
- ✅ 渐进式迁移支持

此示例代表了**企业级MVVM架构的定制化能力**，为复杂项目结构提供了**约定适配**的最佳实践。