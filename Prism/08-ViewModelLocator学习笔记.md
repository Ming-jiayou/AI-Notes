# 08-ViewModelLocator 学习笔记 - 视图模型自动定位器

## 项目概述

08-ViewModelLocator 是 Prism MVVM 架构的核心示例，展示了 **ViewModelLocator** 的自动视图模型绑定机制。这是从手动绑定到**约定优于配置**的范式转变，实现了真正的MVVM解耦。

## 项目结构

```
08-ViewModelLocator/
├── ViewModelLocator/
│   ├── ViewModelLocator.csproj
│   ├── App.xaml.cs (标准启动)
│   ├── Views/
│   │   └── MainWindow.xaml (自动绑定示例)
│   └── ViewModels/
│       └── MainWindowViewModel.cs (标准ViewModel)
```

## 核心机制

### 1. 自动绑定实现

#### MainWindow.xaml - 一行代码实现MVVM
```xml
<Window x:Class="ViewModelLocator.Views.MainWindow"
        xmlns:prism="http://prismlibrary.com/"
        prism:ViewModelLocator.AutoWireViewModel="True"
        Title="{Binding Title}" ...>
    <Grid>
        <ContentControl prism:RegionManager.RegionName="ContentRegion" />
    </Grid>
</Window>
```

#### MainWindowViewModel.cs - 标准ViewModel
```csharp
public class MainWindowViewModel : BindableBase
{
    private string _title = "Prism Unity Application";
    public string Title
    {
        get { return _title; }
        set { SetProperty(ref _title, value); }
    }
}
```

### 2. 约定优于配置

#### 自动定位规则
```
View: MainWindow.xaml (位于Views命名空间)
    ↓ 自动查找
ViewModel: MainWindowViewModel (位于ViewModels命名空间)
```

| 视图位置 | 自动查找的ViewModel位置 |
|----------|------------------------|
| `Views.MainWindow` | `ViewModels.MainWindowViewModel` |
| `Views.CustomerView` | `ViewModels.CustomerViewModel` |

## 核心价值

### 从手动到自动的范式转变

| 传统方式 | ViewModelLocator方式 |
|----------|---------------------|
| **代码绑定** | `DataContext = new ViewModel()` |
| **XAML绑定** | `DataContext="{Binding ViewModel}"` |
| **Prism方式** | `AutoWireViewModel="True"` |

### 解耦优势
- **零代码绑定**：无需手动设置DataContext
- **命名约定**：基于命名空间的智能匹配
- **测试友好**：独立测试View和ViewModel
- **维护简单**：重命名自动同步

## 实际应用

### 标准MVVM项目结构
```
MyApp/
├── Views/
│   ├── MainWindow.xaml
│   └── CustomerView.xaml
├── ViewModels/
│   ├── MainWindowViewModel.cs
│   └── CustomerViewModel.cs
└── Models/
    └── Customer.cs
```

只需设置`AutoWireViewModel="True"`，Prism自动完成所有绑定工作。

此示例代表了**MVVM架构的现代化实现**，为后续复杂应用奠定了**约定优于配置**的坚实基础。