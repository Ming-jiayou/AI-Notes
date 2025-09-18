# 17-BasicRegionNavigation 学习笔记 - 基础区域导航

## 项目概述

17-BasicRegionNavigation 展示了 **IRegionManager.RequestNavigate** 的基础导航功能，实现了**视图间的程序化切换**，为**单页面应用(SPA)**和**标签页界面**提供了**导航驱动**的UI切换能力。

## 核心突破
- **✅ 导航驱动UI**：从静态视图到动态导航的转变
- **✅ 区域导航**：基于区域的视图切换机制
- **✅ URI导航**：使用字符串URI进行视图定位

## 核心实现

### 1. 导航命令

#### MainWindowViewModel.cs - 导航控制中心
```csharp
public class MainWindowViewModel : BindableBase
{
    private readonly IRegionManager _regionManager;

    // 导航命令：参数为视图URI
    public DelegateCommand<string> NavigateCommand { get; private set; }

    public MainWindowViewModel(IRegionManager regionManager)
    {
        _regionManager = regionManager;
        NavigateCommand = new DelegateCommand<string>(Navigate);
    }

    private void Navigate(string navigatePath)
    {
        if (navigatePath != null)
            _regionManager.RequestNavigate("ContentRegion", navigatePath);
    }
}
```

### 2. 导航界面

#### MainWindow.xaml - 导航控制面板
```xml
<Window ...>
    <DockPanel>
        <!-- 导航按钮区域 -->
        <StackPanel DockPanel.Dock="Top" Orientation="Horizontal">
            <Button Command="{Binding NavigateCommand}" 
                    CommandParameter="ViewA" 
                    Content="Navigate to View A"/>
            <Button Command="{Binding NavigateCommand}" 
                    CommandParameter="ViewB" 
                    Content="Navigate to View B"/>
        </StackPanel>
        
        <!-- 内容显示区域 -->
        <ContentControl prism:RegionManager.RegionName="ContentRegion"/>
    </DockPanel>
</Window>
```

## 导航工作原理

### 导航执行流程
```
用户点击"Navigate to View A"
    ↓
NavigateCommand.Execute("ViewA")
    ↓  
_regionManager.RequestNavigate("ContentRegion", "ViewA")
    ↓
Prism查找注册的ViewA视图
    ↓
在ContentRegion中显示ViewA
```

### URI解析机制
| URI参数 | 解析结果 | 对应视图 |
|---------|----------|----------|
| **"ViewA"** | ViewA.xaml | ModuleA.Views.ViewA |
| **"ViewB"** | ViewB.xaml | ModuleA.Views.ViewB |
| **"PersonDetail"** | PersonDetail.xaml | ModuleA.Views.PersonDetail |

## 导航类型对比

| 导航方式 | 实现方式 | 适用场景 | 特点 |
|----------|----------|----------|------|
| **视图发现** | RegisterViewWithRegion | 静态内容 | 一次性加载 |
| **视图注入** | region.Add() | 动态内容 | 程序化控制 |
| **区域导航** | RequestNavigate() | **✅ 交互式导航** | **✅ URI驱动** |

## 实际应用场景

### 1. 标签页导航
```csharp
// 类似浏览器的标签切换
public void NavigateToTab(string tabName)
{
    _regionManager.RequestNavigate("TabRegion", tabName);
}
```

### 2. 向导界面
```csharp
// 分步向导导航
public void NextStep()
{
    currentStep++;
    _regionManager.RequestNavigate("WizardRegion", $"Step{currentStep}");
}
```

### 3. 仪表板切换
```csharp
// 仪表板视图切换
public void ShowDashboard(string dashboardType)
{
    _regionManager.RequestNavigate("DashboardRegion", dashboardType);
}
```

## 导航优势

### 1. 松耦合设计
- **URI解耦**：通过字符串URI定位视图
- **无直接引用**：导航双方无需知道彼此
- **动态解析**：运行时动态解析视图

### 2. 可测试性
```csharp
// 导航逻辑可单元测试
[Test]
public void NavigateCommand_Should_Call_RequestNavigate()
{
    // Arrange
    var mockRegionManager = new Mock<IRegionManager>();
    var viewModel = new MainWindowViewModel(mockRegionManager.Object);
    
    // Act
    viewModel.NavigateCommand.Execute("ViewA");
    
    // Assert
    mockRegionManager.Verify(r => r.RequestNavigate("ContentRegion", "ViewA"));
}
```

### 3. 可扩展性
```csharp
// 支持自定义导航逻辑
public interface INavigationService
{
    void NavigateToView(string viewName);
    void GoBack();
    void GoForward();
}
```

## 企业级价值

### 1. 单页面应用(SPA)
```csharp
// 类似Angular/React的路由机制
public class NavigationService
{
    public void NavigateTo(string route)
    {
        _regionManager.RequestNavigate("MainContent", route);
    }
}
```

### 2. 模块化导航
```csharp
// 模块独立导航
public class ModuleNavigationService
{
    public void NavigateWithinModule(string moduleView)
    {
        _regionManager.RequestNavigate("ModuleRegion", moduleView);
    }
}
```

### 3. 深度链接支持
```csharp
// 支持URL深度链接
public void NavigateFromUrl(string url)
{
    var viewName = ParseUrlToView(url);
    _regionManager.RequestNavigate("ContentRegion", viewName);
}
```

此示例代表了**导航功能**的**基础实现**，为**复杂导航场景**和**高级导航模式**奠定了**RequestNavigate**的基础。