# Stylet BindableCollection 详细使用指南

## 概述

`BindableCollection<T>` 是 Stylet 框架提供的一个强大的可观察集合类，它继承自 `ObservableCollection<T>` 并添加了额外的功能。这个类专为 WPF 和 MVVM 模式设计，提供了数据绑定所需的全部功能，同时优化了批量操作的性能。

## 核心特性

### 1. 基础功能
- **完整的 INotifyCollectionChanged 支持**：当集合发生变化时自动通知 UI
- **INotifyPropertyChanged 支持**：当 Count 和 Item[] 属性变化时通知
- **线程安全**：所有操作都在 UI 线程上执行
- **批量操作支持**：提供 AddRange 和 RemoveRange 方法

### 2. 高级功能
- **CollectionChanging 事件**：在集合实际改变之前触发
- **Refresh 方法**：强制刷新所有绑定
- **性能优化**：批量操作时只触发一次 CollectionChanged 事件

## 接口定义

### IObservableCollection<T>
```csharp
public interface IObservableCollection<T> : IList<T>, INotifyPropertyChanged, INotifyCollectionChanged
{
    void AddRange(IEnumerable<T> items);
    void RemoveRange(IEnumerable<T> items);
}
```

### IReadOnlyObservableCollection<T>
```csharp
public interface IReadOnlyObservableCollection<out T> : IReadOnlyList<T>, 
    INotifyCollectionChanged, INotifyCollectionChanging
{ }
```

## 构造函数

### 默认构造函数
```csharp
var collection = new BindableCollection<string>();
```

### 从现有集合初始化
```csharp
var list = new List<string> { "item1", "item2" };
var collection = new BindableCollection<string>(list);
```

## 基本用法

### 创建和初始化
```csharp
// 创建一个员工集合
public class EmployeeViewModel : Screen
{
    public IObservableCollection<EmployeeModel> Employees { get; private set; }

    public EmployeeViewModel()
    {
        Employees = new BindableCollection<EmployeeModel>
        {
            new EmployeeModel { Name = "张三", Age = 30 },
            new EmployeeModel { Name = "李四", Age = 25 }
        };
    }
}
```

### 单个项目操作
```csharp
// 添加单个项目
Employees.Add(new EmployeeModel { Name = "王五", Age = 28 });

// 插入到指定位置
Employees.Insert(0, new EmployeeModel { Name = "赵六", Age = 35 });

// 移除项目
var employee = Employees[0];
Employees.Remove(employee);

// 移除指定位置的项目
Employees.RemoveAt(0);

// 清空集合
Employees.Clear();
```

## 批量操作

### AddRange - 批量添加
```csharp
public void LoadEmployees()
{
    var newEmployees = new List<EmployeeModel>
    {
        new EmployeeModel { Name = "员工1", Age = 20 },
        new EmployeeModel { Name = "员工2", Age = 21 },
        new EmployeeModel { Name = "员工3", Age = 22 }
    };
    
    // 批量添加 - 只触发一次 CollectionChanged 事件
    Employees.AddRange(newEmployees);
}
```

### RemoveRange - 批量移除
```csharp
public void RemoveSelectedEmployees(IEnumerable<EmployeeModel> selectedItems)
{
    // 批量移除选中的员工
    Employees.RemoveRange(selectedItems);
}
```

## 数据绑定示例

### XAML 绑定
```xml
<!-- ListBox 绑定到 BindableCollection -->
<ListBox ItemsSource="{Binding Employees}"
         SelectedItem="{Binding SelectedEmployee}">
    <ListBox.ItemTemplate>
        <DataTemplate>
            <StackPanel Orientation="Horizontal">
                <TextBlock Text="{Binding Name}" Width="100"/>
                <TextBlock Text="{Binding Age}" Width="50"/>
            </StackPanel>
        </DataTemplate>
    </ListBox.ItemTemplate>
</ListBox>

<!-- DataGrid 绑定 -->
<DataGrid ItemsSource="{Binding Employees}"
          AutoGenerateColumns="False"
          CanUserAddRows="False">
    <DataGrid.Columns>
        <DataGridTextColumn Header="姓名" Binding="{Binding Name}"/>
        <DataGridTextColumn Header="年龄" Binding="{Binding Age}"/>
    </DataGrid.Columns>
</DataGrid>
```

### ViewModel 完整示例
```csharp
using Stylet;
using System.Linq;

namespace Stylet.Samples.MasterDetail
{
    public class ShellViewModel : Screen
    {
        public IObservableCollection<EmployeeModel> Employees { get; private set; }

        private EmployeeModel _selectedEmployee;
        public EmployeeModel SelectedEmployee
        {
            get => this._selectedEmployee;
            set => this.SetAndNotify(ref this._selectedEmployee, value);
        }

        public ShellViewModel()
        {
            this.DisplayName = "Master-Detail";

            this.Employees = new BindableCollection<EmployeeModel>
            {
                new EmployeeModel() { Name = "Fred" },
                new EmployeeModel() { Name = "Bob" }
            };

            this.SelectedEmployee = this.Employees.FirstOrDefault();
        }

        public void AddEmployee()
        {
            this.Employees.Add(new EmployeeModel() { Name = "Unnamed" });
        }

        public void RemoveEmployee(EmployeeModel item)
        {
            this.Employees.Remove(item);
        }
    }
}
```

## 事件处理

### CollectionChanged 事件
```csharp
public class EmployeeViewModel : Screen
{
    public BindableCollection<EmployeeModel> Employees { get; private set; }

    public EmployeeViewModel()
    {
        Employees = new BindableCollection<EmployeeModel>();
        
        // 监听集合变化
        Employees.CollectionChanged += OnEmployeesCollectionChanged;
    }

    private void OnEmployeesCollectionChanged(object sender, NotifyCollectionChangedEventArgs e)
    {
        switch (e.Action)
        {
            case NotifyCollectionChangedAction.Add:
                // 处理添加操作
                foreach (EmployeeModel newItem in e.NewItems)
                {
                    // 执行添加后的逻辑
                }
                break;
                
            case NotifyCollectionChangedAction.Remove:
                // 处理移除操作
                foreach (EmployeeModel oldItem in e.OldItems)
                {
                    // 执行移除后的逻辑
                }
                break;
                
            case NotifyCollectionChangedAction.Reset:
                // 处理重置操作
                break;
        }
    }
}
```

### CollectionChanging 事件
```csharp
public class EmployeeViewModel : Screen
{
    public EmployeeViewModel()
    {
        var collection = new BindableCollection<EmployeeModel>();
        
        // 在集合改变前监听
        collection.CollectionChanging += OnCollectionChanging;
    }

    private void OnCollectionChanging(object sender, NotifyCollectionChangedEventArgs e)
    {
        // 在集合实际改变前执行验证或其他操作
        if (e.Action == NotifyCollectionChangedAction.Remove)
        {
            foreach (EmployeeModel item in e.OldItems)
            {
                // 验证是否可以移除
                if (item.IsLocked)
                {
                    // 取消操作或显示警告
                }
            }
        }
    }
}
```

## 性能优化技巧

### 1. 批量操作优于循环
```csharp
// ❌ 不推荐：会触发多次 CollectionChanged 事件
foreach (var item in items)
{
    collection.Add(item);
}

// ✅ 推荐：只触发一次 CollectionChanged 事件
collection.AddRange(items);
```

### 2. 使用 Refresh 方法
```csharp
// 当需要强制刷新所有绑定时
public void RefreshAllBindings()
{
    Employees.Refresh();
}
```

### 3. 延迟通知
```csharp
// 在大量操作时临时禁用通知
public void PerformBulkOperation()
{
    // 注意：Stylet 的 BindableCollection 内部已经优化了批量操作
    // 一般不需要手动禁用通知
    Employees.AddRange(largeListOfItems);
}
```

## 线程安全

### 所有操作都在 UI 线程执行
```csharp
// 即使在后台线程中调用，也会自动切换到 UI 线程
Task.Run(() =>
{
    // 这是安全的，会自动在 UI 线程执行
    Employees.Add(new EmployeeModel { Name = "后台添加的员工" });
});
```

## 高级用法

### 过滤和排序
```csharp
public class EmployeeViewModel : Screen
{
    public BindableCollection<EmployeeModel> AllEmployees { get; private set; }
    public BindableCollection<EmployeeModel> FilteredEmployees { get; private set; }

    private string _searchText;
    public string SearchText
    {
        get => _searchText;
        set
        {
            if (SetAndNotify(ref _searchText, value))
            {
                ApplyFilter();
            }
        }
    }

    private void ApplyFilter()
    {
        var filtered = AllEmployees
            .Where(e => string.IsNullOrEmpty(SearchText) || 
                       e.Name.Contains(SearchText, StringComparison.OrdinalIgnoreCase))
            .OrderBy(e => e.Name);

        FilteredEmployees.Clear();
        FilteredEmployees.AddRange(filtered);
    }
}
```

### 与 Conductor 结合使用
```csharp
public class EmployeeConductor : Conductor<EmployeeViewModel>.Collection.OneActive
{
    public void AddEmployee()
    {
        var employee = new EmployeeModel { Name = "新员工" };
        var employeeVm = new EmployeeViewModel(employee);
        
        // Items 是 BindableCollection<T>
        this.Items.Add(employeeVm);
        this.ActivateItem(employeeVm);
    }
}
```

## 常见错误和解决方案

### 1. 修改集合时未触发 UI 更新
```csharp
// ❌ 错误：直接修改集合中的对象属性
Employees[0].Name = "新名字"; // UI 不会更新

// ✅ 正确：确保 EmployeeModel 实现了 INotifyPropertyChanged
public class EmployeeModel : PropertyChangedBase
{
    private string _name;
    public string Name
    {
        get => _name;
        set => SetAndNotify(ref _name, value);
    }
}
```

### 2. 在非 UI 线程操作集合
```csharp
// ❌ 错误：在非 UI 线程直接操作
Task.Run(() => Employees.Add(item));

// ✅ 正确：Stylet 自动处理线程切换，无需额外代码
```

### 3. 循环引用导致内存泄漏
```csharp
// ❌ 错误：未取消事件订阅
public class EmployeeViewModel : Screen
{
    public EmployeeViewModel()
    {
        Employees.CollectionChanged += OnCollectionChanged;
    }
}

// ✅ 正确：在 Dispose 时取消订阅
protected override void OnDeactivate()
{
    Employees.CollectionChanged -= OnCollectionChanged;
    base.OnDeactivate();
}
```

## 最佳实践

### 1. 使用接口类型声明属性
```csharp
// ✅ 推荐：使用接口类型
public IObservableCollection<EmployeeModel> Employees { get; private set; }

// ❌ 不推荐：使用具体类型
public BindableCollection<EmployeeModel> Employees { get; private set; }
```

### 2. 封装集合操作
```csharp
public class EmployeeViewModel : Screen
{
    private readonly BindableCollection<EmployeeModel> _employees;
    public IObservableCollection<EmployeeModel> Employees => _employees;

    public void AddEmployee(string name, int age)
    {
        var employee = new EmployeeModel { Name = name, Age = age };
        _employees.Add(employee);
    }

    public void RemoveEmployee(EmployeeModel employee)
    {
        _employees.Remove(employee);
    }
}
```

### 3. 使用依赖注入
```csharp
public class EmployeeViewModel : Screen
{
    private readonly IEmployeeService _employeeService;

    public EmployeeViewModel(IEmployeeService employeeService)
    {
        _employeeService = employeeService;
        Employees = new BindableCollection<EmployeeModel>();
    }

    public async Task LoadEmployeesAsync()
    {
        var employees = await _employeeService.GetAllEmployeesAsync();
        Employees.Clear();
        Employees.AddRange(employees);
    }
}
```

## 总结

`BindableCollection<T>` 是 Stylet 框架中处理集合数据绑定的核心组件，它提供了：

1. **完整的 WPF 数据绑定支持**
2. **优化的批量操作性能**
3. **线程安全的操作**
4. **丰富的事件通知机制**
5. **与 Stylet 其他组件的完美集成**

通过合理使用 BindableCollection，可以构建出响应式、高性能的 MVVM 应用程序。记住始终使用接口类型声明公共属性，利用批量操作优化性能，并确保正确的事件订阅和取消订阅以避免内存泄漏。