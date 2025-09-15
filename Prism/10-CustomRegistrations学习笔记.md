# 10-CustomRegistrations 学习笔记 - 自定义ViewModel注册

## 项目概述

10-CustomRegistrations 展示了如何**手动注册特定的ViewModel绑定关系**，通过`ViewModelLocationProvider.Register`实现**精确的视图到ViewModel映射控制**，为复杂场景提供**显式绑定管理**能力。

## 核心机制

### 1. 显式注册模式

#### App.xaml.cs - 多种注册方式
```csharp
protected override void ConfigureViewModelLocator()
{
    base.ConfigureViewModelLocator();

    // 方式1: 类型/类型注册
    //ViewModelLocationProvider.Register(typeof(MainWindow).ToString(), typeof(CustomViewModel));

    // 方式2: 类型/工厂注册  
    //ViewModelLocationProvider.Register(typeof(MainWindow).ToString(), () => Container.Resolve<CustomViewModel>());

    // 方式3: 泛型工厂注册
    //ViewModelLocationProvider.Register<MainWindow>(() => Container.Resolve<CustomViewModel>());

    // 方式4: 泛型类型注册 (本例使用)
    ViewModelLocationProvider.Register<MainWindow, CustomViewModel>();
}
```

### 2. 自定义ViewModel

#### CustomViewModel.cs - 独立实现
```csharp
public class CustomViewModel : BindableBase
{
    private string _title = "Custom ViewModel Application";
    public string Title
    {
        get { return _title; }
        set { SetProperty(ref _title, value); }
    }
}
```

## 注册方式对比

| 注册方式 | 语法形式 | 适用场景 | 灵活性 |
|----------|----------|----------|--------|
| **类型/类型** | `Register(type, type)` | 简单映射 | 中等 |
| **类型/工厂** | `Register(type, factory)` | 需要依赖注入 | 高 |
| **泛型工厂** | `Register<T>(() => ...)` | 类型安全 | 高 |
| **泛型类型** | `Register<TView, TViewModel>()` | 最简洁 | ✅ 最高 |

## 实际应用场景

### 1. 约定覆盖
```csharp
// 当约定不满足时，显式指定
ViewModelLocationProvider.Register<ComplexView, SimpleViewModel>();
```

### 2. 条件绑定
```csharp
// 基于运行时条件选择ViewModel
ViewModelLocationProvider.Register<MainWindow>(() => 
    isDebugMode ? new DebugViewModel() : new ReleaseViewModel());
```

### 3. 依赖注入集成
```csharp
// 工厂模式支持完整DI
ViewModelLocationProvider.Register<CustomerView>(() => 
    Container.Resolve<CustomerViewModel>(new ParameterOverride("repository", customerRepo)));
```

## 核心价值

**显式控制** = **精确管理**

- ✅ 覆盖默认约定
- ✅ 支持复杂依赖
- ✅ 运行时条件选择
- ✅ 测试替身注入

此示例代表了**ViewModelLocator的高级用法**，为**复杂企业应用**提供了**显式绑定管理**的最佳实践。