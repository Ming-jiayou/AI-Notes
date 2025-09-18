# 16-RegionContext 项目学习笔记

## 项目概述

RegionContext 是 Prism 框架中的一个重要概念，用于在区域（Region）之间传递上下文数据。本项目演示了如何使用 RegionContext 在父视图和子视图之间共享数据，实现主从（Master-Detail）界面模式。

## 核心概念

### RegionContext 的作用
RegionContext 允许将数据从一个视图传递到另一个视图，特别是在嵌套区域的情况下。它提供了一种机制，让父视图可以将当前选中的数据项传递给子区域中的视图。

### 关键特性
- **数据共享**：在区域之间传递上下文数据
- **自动更新**：当上下文数据变化时，自动通知相关视图
- **解耦合**：视图之间不需要直接引用，通过 RegionContext 进行通信

## 项目结构

```
16-RegionContext/
├── RegionContext/              # 主应用程序
│   ├── App.xaml               # 应用程序入口，配置 Prism
│   ├── App.xaml.cs            # 应用程序逻辑，注册模块
│   ├── Views/
│   │   └── MainWindow.xaml    # 主窗口，包含 ContentRegion
│   └── ViewModels/
│       └── MainWindowViewModel.cs
└── ModuleA/                   # 功能模块
    ├── ModuleAModule.cs       # 模块初始化，注册视图
    ├── Business/
    │   └── Person.cs          # 业务实体类
    ├── Views/
    │   ├── PersonList.xaml    # 人员列表视图（主视图）
    │   └── PersonDetail.xaml  # 人员详情视图（从视图）
    └── ViewModels/
        ├── PersonListViewModel.cs
        └── PersonDetailViewModel.cs
```

## 关键实现

### 1. 主窗口布局 (MainWindow.xaml)
```xml
<ContentControl prism:RegionManager.RegionName="ContentRegion" />
```

### 2. 人员列表视图 (PersonList.xaml)
```xml
<ListBox x:Name="_listOfPeople" ItemsSource="{Binding People}"/>
<ContentControl Grid.Row="1" 
                prism:RegionManager.RegionName="PersonDetailsRegion"
                prism:RegionManager.RegionContext="{Binding SelectedItem, ElementName=_listOfPeople}"/>
```

**关键点**：
- `RegionContext` 绑定到 ListBox 的 `SelectedItem`
- 当选择变化时，自动将选中的人员对象传递给详情区域

### 3. 人员详情视图 (PersonDetail.xaml.cs)
```csharp
public PersonDetail()
{
    InitializeComponent();
    RegionContext.GetObservableContext(this).PropertyChanged += PersonDetail_PropertyChanged;
}

private void PersonDetail_PropertyChanged(object sender, PropertyChangedEventArgs e)
{
    var context = (ObservableObject<object>)sender;
    var selectedPerson = (Person)context.Value;
    (DataContext as PersonDetailViewModel).SelectedPerson = selectedPerson;
}
```

**关键点**：
- 监听 RegionContext 的变化事件
- 当上下文数据变化时，更新 ViewModel 中的选中人员

### 4. 模块注册 (ModuleAModule.cs)
```csharp
public void OnInitialized(IContainerProvider containerProvider)
{
    var regionManager = containerProvider.Resolve<IRegionManager>();
    regionManager.RegisterViewWithRegion("ContentRegion", typeof(PersonList));
    regionManager.RegisterViewWithRegion("PersonDetailsRegion", typeof(PersonDetail));
}
```

## 工作流程

1. **初始化**：应用程序启动时，ModuleA 将 `PersonList` 注册到 `ContentRegion`，将 `PersonDetail` 注册到 `PersonDetailsRegion`

2. **数据显示**：`PersonList` 显示人员列表，并在下方区域加载 `PersonDetail` 视图

3. **选择变化**：当用户在列表中选择不同人员时，`RegionContext` 自动将选中的人员对象传递给 `PersonDetailsRegion`

4. **数据更新**：`PersonDetail` 视图接收到新的上下文数据，更新显示选中人员的详细信息

## 技术要点

### RegionContext 的使用方式
- **设置上下文**：`prism:RegionManager.RegionContext="{Binding SelectedItem}"`
- **获取上下文**：`RegionContext.GetObservableContext(this)`
- **监听变化**：订阅 `PropertyChanged` 事件

### 数据绑定
- 主视图通过 `RegionContext` 传递数据
- 从视图通过事件监听获取数据变化
- ViewModel 负责实际的数据存储和业务逻辑

## 学习总结

RegionContext 提供了一种优雅的方式来实现区域间的数据共享，特别适用于主从界面模式。通过本项目的学习，可以掌握：

1. 如何在 XAML 中设置 RegionContext
2. 如何在代码中监听 RegionContext 的变化
3. 如何实现主从视图之间的数据传递
4. 如何保持视图之间的松耦合关系

这种模式在实际应用中非常常见，如订单列表与订单详情、产品列表与产品信息等场景。