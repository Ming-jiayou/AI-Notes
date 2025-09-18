# 21-PassingParameters 项目学习笔记

## 项目概述

PassingParameters 项目演示了如何在 Prism 框架中实现导航参数传递（Passing Parameters）。通过使用 [`NavigationParameters`](21-PassingParameters/ModuleA/ViewModels/PersonListViewModel.cs:33) 类，可以在导航过程中在视图之间传递复杂的数据对象，实现主从（Master-Detail）界面模式。

## 核心概念

### 参数传递的作用
导航参数传递允许开发者：
- **传递复杂对象**：在视图之间传递业务实体和数据模型
- **保持类型安全**：保持强类型数据传递，避免类型转换错误
- **支持多种数据类型**：支持基本类型、复杂对象、集合等多种数据类型
- **实现松耦合**：视图之间不需要直接引用，通过参数进行通信

### 关键特性
- **类型安全**：支持强类型参数传递
- **灵活配置**：支持多种参数添加方式
- **双向传递**：支持导航前和导航后的参数传递
- **生命周期管理**：参数在导航生命周期内有效

## 项目结构

```
21-PassingParameters/
├── PassingParameters/            # 主应用程序
│   ├── App.xaml                 # 应用程序入口，配置 Prism
│   ├── App.xaml.cs              # 应用程序逻辑，注册模块
│   ├── Views/
│   │   └── MainWindow.xaml      # 主窗口，包含 ContentRegion
│   └── ViewModels/
│       └── MainWindowViewModel.cs
└── ModuleA/                     # 功能模块
    ├── ModuleAModule.cs         # 模块初始化，注册视图
    ├── Business/
    │   └── Person.cs            # 业务实体类，包含人员信息
    ├── Views/
    │   ├── PersonList.xaml      # 人员列表视图（主视图）
    │   └── PersonDetail.xaml    # 人员详情视图（从视图）
    └── ViewModels/
        ├── PersonListViewModel.cs # 列表视图模型，负责参数传递
        └── PersonDetailViewModel.cs # 详情视图模型，负责参数接收
```

## 关键实现

### 1. 参数传递 (PersonListViewModel.cs)
```csharp
private void PersonSelected(Person person)
{
    var parameters = new NavigationParameters();
    parameters.Add("person", person);

    if (person != null)
        _regionManager.RequestNavigate("PersonDetailsRegion", "PersonDetail", parameters);
}
```

**关键点**：
- 创建 [`NavigationParameters`](21-PassingParameters/ModuleA/ViewModels/PersonListViewModel.cs:33) 实例
- 使用 [`Add`](21-PassingParameters/ModuleA/ViewModels/PersonListViewModel.cs:34) 方法添加参数
- 将参数作为第三个参数传递给 [`RequestNavigate`](21-PassingParameters/ModuleA/ViewModels/PersonListViewModel.cs:37) 方法

### 2. 参数接收 (PersonDetailViewModel.cs)
```csharp
public void OnNavigatedTo(NavigationContext navigationContext)
{
    var person = navigationContext.Parameters["person"] as Person;
    if (person != null)
        SelectedPerson = person;
}
```

**关键点**：
- 通过 [`navigationContext.Parameters`](21-PassingParameters/ModuleA/ViewModels/PersonDetailViewModel.cs:24) 访问参数
- 使用键名获取参数值
- 进行类型安全的类型转换

### 3. 参数在导航目标判断中的使用
```csharp
public bool IsNavigationTarget(NavigationContext navigationContext)
{
    var person = navigationContext.Parameters["person"] as Person;
    if (person != null)
        return SelectedPerson != null && SelectedPerson.LastName == person.LastName;
    else
        return true;
}
```

**关键点**：
- 在 [`IsNavigationTarget`](21-PassingParameters/ModuleA/ViewModels/PersonDetailViewModel.cs:29) 方法中也可以访问参数
- 根据参数内容决定是否重用当前视图实例
- 实现更智能的导航目标判断

### 4. 模块注册和区域配置 (ModuleAModule.cs)
```csharp
public void OnInitialized(IContainerProvider containerProvider)
{
    var regionManager = containerProvider.Resolve<IRegionManager>();
    regionManager.RegisterViewWithRegion("ContentRegion", typeof(PersonList));            
}

public void RegisterTypes(IContainerRegistry containerRegistry)
{
    containerRegistry.RegisterForNavigation<PersonDetail>();
}
```

**关键点**：
- PersonList 通过 [`RegisterViewWithRegion`](21-PassingParameters/ModuleA/ModuleAModule.cs:13) 自动显示
- PersonDetail 通过 [`RegisterForNavigation`](21-PassingParameters/ModuleA/ModuleAModule.cs:18) 注册为可导航视图

## 工作流程

1. **初始化**：应用程序启动时，PersonList 自动显示在 ContentRegion 中

2. **用户选择**：用户在列表中选择人员，触发 [`PersonSelectedCommand`](21-PassingParameters/ModuleA/ViewModels/PersonListViewModel.cs:21)

3. **参数准备**：创建 [`NavigationParameters`](21-PassingParameters/ModuleA/ViewModels/PersonListViewModel.cs:33) 并添加选中的人员对象

4. **导航请求**：调用 [`RequestNavigate`](21-PassingParameters/ModuleA/ViewModels/PersonListViewModel.cs:37) 进行导航，传递参数

5. **参数接收**：PersonDetailViewModel 在 [`OnNavigatedTo`](21-PassingParameters/ModuleA/ViewModels/PersonDetailViewModel.cs:22) 中接收参数

6. **数据显示**：Detail 视图显示接收到的人员信息

## 技术要点

### NavigationParameters 的使用方式
```csharp
// 1. 基本用法
var parameters = new NavigationParameters();
parameters.Add("key1", value1);
parameters.Add("key2", value2);

// 2. 对象初始化器
var parameters = new NavigationParameters
{
    { "person", person },
    { "mode", "edit" },
    { "timestamp", DateTime.Now }
};

// 3. 从字典创建
var dict = new Dictionary<string, object>
{
    { "person", person },
    { "mode", "view" }
};
var parameters = new NavigationParameters(dict);
```

### 参数类型支持
```csharp
// 基本类型
parameters.Add("id", 123);
parameters.Add("name", "John");
parameters.Add("isActive", true);

// 复杂对象
parameters.Add("person", new Person { FirstName = "John", LastName = "Doe" });

// 集合
parameters.Add("items", new List<Item> { item1, item2, item3 });

// 自定义对象
parameters.Add("config", new AppConfig { Theme = "Dark", Language = "zh-CN" });
```

### 参数获取方式
```csharp
public void OnNavigatedTo(NavigationContext navigationContext)
{
    // 1. 直接索引器访问
    var person = navigationContext.Parameters["person"] as Person;
    
    // 2. TryGetValue 方法（推荐）
    if (navigationContext.Parameters.TryGetValue<Person>("person", out var person))
    {
        SelectedPerson = person;
    }
    
    // 3. 获取所有参数
    foreach (var parameter in navigationContext.Parameters)
    {
        Console.WriteLine($"Key: {parameter.Key}, Value: {parameter.Value}");
    }
}
```

### 参数验证
```csharp
public void OnNavigatedTo(NavigationContext navigationContext)
{
    // 验证必需参数
    if (!navigationContext.Parameters.ContainsKey("person"))
    {
        throw new InvalidOperationException("Person parameter is required");
    }
    
    // 类型验证
    if (!(navigationContext.Parameters["person"] is Person person))
    {
        throw new InvalidCastException("Invalid person parameter type");
    }
    
    // 业务验证
    if (person.Age < 0 || person.Age > 150)
    {
        throw new ArgumentOutOfRangeException("Invalid age value");
    }
    
    SelectedPerson = person;
}
```

## 扩展应用

### 多参数传递
```csharp
private void NavigateToDetail(Person person, string mode, DateTime timestamp)
{
    var parameters = new NavigationParameters
    {
        { "person", person },
        { "mode", mode },
        { "timestamp", timestamp },
        { "userId", CurrentUser.Id },
        { "permissions", CurrentUser.Permissions }
    };
    
    _regionManager.RequestNavigate("DetailRegion", "PersonDetail", parameters);
}
```

### 复杂对象传递
```csharp
public class NavigationData
{
    public Person Person { get; set; }
    public string Mode { get; set; }
    public Dictionary<string, object> Metadata { get; set; }
    public List<string> Tags { get; set; }
}

// 传递复杂对象
var navData = new NavigationData
{
    Person = selectedPerson,
    Mode = "edit",
    Metadata = new Dictionary<string, object> { { "source", "list" } },
    Tags = new List<string> { "urgent", "important" }
};

var parameters = new NavigationParameters();
parameters.Add("navData", navData);

_regionManager.RequestNavigate("DetailRegion", "DetailView", parameters);
```

### 参数在回调中的使用
```csharp
private void NavigateWithCallback(Person person)
{
    var parameters = new NavigationParameters();
    parameters.Add("person", person);
    parameters.Add("callbackId", Guid.NewGuid());
    
    _regionManager.RequestNavigate("DetailRegion", "PersonDetail", parameters, (result) =>
    {
        if (result.Result == true)
        {
            Console.WriteLine($"Navigation successful for person: {person.LastName}");
        }
    });
}
```

### 双向参数传递
```csharp
public void OnNavigatedFrom(NavigationContext navigationContext)
{
    // 在离开时将数据传递回导航上下文
    navigationContext.Parameters["returnData"] = new
    {
        lastModified = DateTime.Now,
        changesMade = this.HasChanges,
        selectedTab = this.CurrentTab
    };
}
```

## 高级场景

### 参数序列化
```csharp
// 传递需要序列化的复杂对象
private void NavigateWithSerialization(CustomObject data)
{
    var json = JsonSerializer.Serialize(data);
    var parameters = new NavigationParameters();
    parameters.Add("serializedData", json);
    
    _regionManager.RequestNavigate("TargetRegion", "TargetView", parameters);
}

public void OnNavigatedTo(NavigationContext navigationContext)
{
    if (navigationContext.Parameters.TryGetValue<string>("serializedData", out var json))
    {
        var data = JsonSerializer.Deserialize<CustomObject>(json);
        // 使用反序列化的数据
    }
}
```

### 参数加密
```csharp
private void NavigateWithEncryption(SensitiveData data)
{
    var encryptedData = EncryptData(data);
    var parameters = new NavigationParameters();
    parameters.Add("secureData", encryptedData);
    
    _regionManager.RequestNavigate("SecureRegion", "SecureView", parameters);
}
```

### 参数缓存策略
```csharp
public class ParameterCache
{
    private static readonly Dictionary<string, object> _cache = new();
    
    public static void CacheParameters(string key, NavigationParameters parameters)
    {
        _cache[key] = parameters;
    }
    
    public static NavigationParameters GetCachedParameters(string key)
    {
        return _cache.TryGetValue(key, out var parameters) ? parameters as NavigationParameters : null;
    }
}
```

## 最佳实践

### 1. 使用常量定义参数键
```csharp
public static class NavigationParameterKeys
{
    public const string Person = "person";
    public const string Mode = "mode";
    public const string Id = "id";
    public const string Timestamp = "timestamp";
}

// 使用
parameters.Add(NavigationParameterKeys.Person, person);
var person = navigationContext.Parameters[NavigationParameterKeys.Person] as Person;
```

### 2. 参数验证和错误处理
```csharp
public void OnNavigatedTo(NavigationContext navigationContext)
{
    try
    {
        ValidateParameters(navigationContext);
        ProcessParameters(navigationContext);
    }
    catch (Exception ex)
    {
        HandleParameterError(ex);
    }
}
```

### 3. 参数封装
```csharp
public class NavigationParameterBuilder
{
    private readonly NavigationParameters _parameters = new();
    
    public NavigationParameterBuilder AddPerson(Person person)
    {
        _parameters.Add("person", person);
        return this;
    }
    
    public NavigationParameterBuilder AddMode(string mode)
    {
        _parameters.Add("mode", mode);
        return this;
    }
    
    public NavigationParameters Build() => _parameters;
}

// 使用
var parameters = new NavigationParameterBuilder()
    .AddPerson(selectedPerson)
    .AddMode("edit")
    .Build();
```

## 学习总结

参数传递机制在实际应用中非常重要，它提供了：

1. **数据共享**：在视图之间传递复杂的数据对象
2. **类型安全**：保持强类型数据传递，减少运行时错误
3. **松耦合**：视图之间不需要直接引用，通过参数进行通信
4. **灵活性**：支持各种数据类型和传递方式
5. **可扩展性**：易于扩展和维护的参数传递机制

通过本项目的学习，可以掌握 Prism 框架中参数传递的核心概念和实现方式，为构建复杂的企业级应用奠定基础。参数传递是实现主从界面、详情页面、编辑表单等常见场景的关键技术。