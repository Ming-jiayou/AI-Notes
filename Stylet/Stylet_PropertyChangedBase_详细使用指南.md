# Stylet PropertyChangedBase 详细使用指南

## 概述

`PropertyChangedBase` 是 Stylet 框架中的核心基类，它实现了 `INotifyPropertyChanged` 接口，为 ViewModel 提供了属性变更通知的基础功能。这个类是所有 Stylet ViewModel 的推荐基类，它简化了属性变更通知的实现，使数据绑定更加高效和优雅。

## 源码结构分析

### 核心定义

```csharp
public class PropertyChangedBase : INotifyPropertyChanged, INotifyPropertyChangedDispatcher
```

`PropertyChangedBase` 实现了两个关键接口：
- `INotifyPropertyChanged`: 标准的 .NET 属性变更通知接口
- `INotifyPropertyChangedDispatcher`: Stylet 自定义的调度器接口，用于控制属性变更事件的线程调度

### 主要成员

#### 事件定义

```csharp
public event PropertyChangedEventHandler PropertyChanged;
```

这是标准的属性变更事件，当属性值发生变化时触发。

#### 属性变更调度器

```csharp
public INotifyPropertyChangedDispatcher PropertyChangedDispatcher { get; set; }
```

用于控制属性变更通知的线程调度，确保 UI 线程安全。

## 核心功能详解

### 1. 基本属性变更通知

#### NotifyOfPropertyChange 方法

```csharp
public virtual void NotifyOfPropertyChanged([CallerMemberName] string propertyName = null)
```

**功能说明**：
- 触发指定属性的 PropertyChanged 事件
- 使用 `[CallerMemberName]` 特性自动获取调用者属性名
- 支持手动指定属性名

**使用示例**：

```csharp
public class MyViewModel : PropertyChangedBase
{
    private string _name;
    
    public string Name
    {
        get => _name;
        set
        {
            _name = value;
            NotifyOfPropertyChanged(); // 自动获取属性名 "Name"
        }
    }
}
```

#### NotifyOfPropertyChanged<T> 泛型方法

```csharp
public virtual void NotifyOfPropertyChanged<T>(Expression<Func<T>> property)
```

**功能说明**：
- 使用表达式树提供类型安全的属性名
- 避免硬编码字符串，支持重构

**使用示例**：

```csharp
public class MyViewModel : PropertyChangedBase
{
    private int _age;
    
    public int Age
    {
        get => _age;
        set
        {
            _age = value;
            NotifyOfPropertyChanged(() => Age); // 类型安全
        }
    }
}
```

### 2. SetAndNotify 方法

#### 基本重载

```csharp
protected virtual bool SetAndNotify<T>(ref T field, T value, [CallerMemberName] string propertyName = null)
```

**功能说明**：
- 原子操作：设置字段值并触发属性变更通知
- 自动处理值比较，避免不必要的通知
- 返回布尔值表示是否发生了变更

**参数说明**：
- `ref T field`: 要设置的字段引用
- `T value`: 新值
- `propertyName`: 属性名（自动获取）

**使用示例**：

```csharp
public class PersonViewModel : PropertyChangedBase
{
    private string _firstName;
    private string _lastName;
    
    public string FirstName
    {
        get => _firstName;
        set => SetAndNotify(ref _firstName, value);
    }
    
    public string LastName
    {
        get => _lastName;
        set => SetAndNotify(ref _lastName, value);
    }
}
```

#### 带自定义比较的 SetAndNotify

```csharp
protected virtual bool SetAndNotify<T>(ref T field, T value, IEqualityComparer<T> comparer, [CallerMemberName] string propertyName = null)
```

**功能说明**：
- 允许使用自定义比较器进行值比较
- 适用于特殊类型的比较需求

**使用示例**：

```csharp
public class CollectionViewModel : PropertyChangedBase
{
    private List<string> _items;
    
    public List<string> Items
    {
        get => _items;
        set => SetAndNotify(ref _items, value, new CollectionComparer());
    }
}

public class CollectionComparer : IEqualityComparer<List<string>>
{
    public bool Equals(List<string> x, List<string> y)
    {
        if (x == null && y == null) return true;
        if (x == null || y == null) return false;
        return x.SequenceEqual(y);
    }
    
    public int GetHashCode(List<string> obj) => obj?.GetHashCode() ?? 0;
}
```

#### 带回调的 SetAndNotify

```csharp
protected virtual bool SetAndNotify<T>(ref T field, T value, Action<T> afterChange, [CallerMemberName] string propertyName = null)
```

**功能说明**：
- 在属性值变更后执行自定义操作
- 适用于需要额外处理的场景

**使用示例**：

```csharp
public class SettingsViewModel : PropertyChangedBase
{
    private string _theme;
    
    public string Theme
    {
        get => _theme;
        set => SetAndNotify(ref _theme, value, newValue => 
        {
            // 主题变更后的处理
            ApplyTheme(newValue);
            SaveSettings();
        });
    }
    
    private void ApplyTheme(string theme)
    {
        // 应用主题逻辑
    }
    
    private void SaveSettings()
    {
        // 保存设置逻辑
    }
}
```

### 3. 批量属性通知

#### Refresh 方法

```csharp
public virtual void Refresh()
```

**功能说明**：
- 通知所有属性已变更
- 适用于对象状态整体更新的场景

**使用示例**：

```csharp
public class DataViewModel : PropertyChangedBase
{
    public void ReloadData()
    {
        // 重新加载所有数据
        LoadDataFromDatabase();
        
        // 通知所有属性已更新
        Refresh();
    }
}
```

### 4. 属性依赖通知

#### 计算属性的通知

```csharp
public class OrderViewModel : PropertyChangedBase
{
    private decimal _unitPrice;
    private int _quantity;
    
    public decimal UnitPrice
    {
        get => _unitPrice;
        set
        {
            if (SetAndNotify(ref _unitPrice, value))
            {
                // 当 UnitPrice 变化时，通知 TotalPrice 也变化
                NotifyOfPropertyChanged(() => TotalPrice);
            }
        }
    }
    
    public int Quantity
    {
        get => _quantity;
        set
        {
            if (SetAndNotify(ref _quantity, value))
            {
                // 当 Quantity 变化时，通知 TotalPrice 也变化
                NotifyOfPropertyChanged(() => TotalPrice);
            }
        }
    }
    
    public decimal TotalPrice => UnitPrice * Quantity;
}
```

## 高级用法

### 1. 自定义属性变更调度

```csharp
public class AsyncViewModel : PropertyChangedBase
{
    public AsyncViewModel()
    {
        // 设置自定义调度器
        PropertyChangedDispatcher = new DispatcherWrapper();
    }
}

public class DispatcherWrapper : INotifyPropertyChangedDispatcher
{
    public void Dispatch(Action action)
    {
        // 自定义调度逻辑
        Application.Current.Dispatcher.Invoke(action);
    }
}
```

### 2. 属性变更验证

```csharp
public class ValidatedViewModel : PropertyChangedBase
{
    private string _email;
    
    public string Email
    {
        get => _email;
        set
        {
            if (SetAndNotify(ref _email, value))
            {
                ValidateEmail(value);
            }
        }
    }
    
    private void ValidateEmail(string email)
    {
        // 验证逻辑
        if (!IsValidEmail(email))
        {
            // 处理验证错误
        }
    }
}
```

### 3. 复杂对象的属性通知

```csharp
public class Address : PropertyChangedBase
{
    private string _street;
    private string _city;
    
    public string Street
    {
        get => _street;
        set => SetAndNotify(ref _street, value);
    }
    
    public string City
    {
        get => _city;
        set => SetAndNotify(ref _city, value);
    }
}

public class PersonViewModel : PropertyChangedBase
{
    private string _name;
    private Address _address;
    
    public string Name
    {
        get => _name;
        set => SetAndNotify(ref _name, value);
    }
    
    public Address Address
    {
        get => _address;
        set
        {
            if (SetAndNotify(ref _address, value))
            {
                // 处理地址变更
                if (_address != null)
                {
                    _address.PropertyChanged += OnAddressPropertyChanged;
                }
            }
        }
    }
    
    private void OnAddressPropertyChanged(object sender, PropertyChangedEventArgs e)
    {
        // 处理地址属性的变更
        NotifyOfPropertyChanged(() => FullAddress);
    }
    
    public string FullAddress => $"{Address?.Street}, {Address?.City}";
}
```

## 最佳实践

### 1. 使用 SetAndNotify 简化代码

**推荐做法**：
```csharp
public string Title
{
    get => _title;
    set => SetAndNotify(ref _title, value);
}
```

**不推荐做法**：
```csharp
public string Title
{
    get { return _title; }
    set
    {
        if (_title != value)
        {
            _title = value;
            NotifyOfPropertyChanged();
        }
    }
}
```

### 2. 处理计算属性依赖

```csharp
public class OrderViewModel : PropertyChangedBase
{
    private decimal _price;
    private int _quantity;
    private decimal _discount;
    
    public decimal Price
    {
        get => _price;
        set
        {
            if (SetAndNotify(ref _price, value))
            {
                NotifyOfPropertyChanged(() => Total);
                NotifyOfPropertyChanged(() => DiscountedTotal);
            }
        }
    }
    
    public int Quantity
    {
        get => _quantity;
        set
        {
            if (SetAndNotify(ref _quantity, value))
            {
                NotifyOfPropertyChanged(() => Total);
                NotifyOfPropertyChanged(() => DiscountedTotal);
            }
        }
    }
    
    public decimal Discount
    {
        get => _discount;
        set
        {
            if (SetAndNotify(ref _discount, value))
            {
                NotifyOfPropertyChanged(() => DiscountedTotal);
            }
        }
    }
    
    public decimal Total => Price * Quantity;
    public decimal DiscountedTotal => Total * (1 - Discount / 100);
}
```

### 3. 异步属性更新

```csharp
public class AsyncDataViewModel : PropertyChangedBase
{
    private string _data;
    private bool _isLoading;
    
    public string Data
    {
        get => _data;
        private set => SetAndNotify(ref _data, value);
    }
    
    public bool IsLoading
    {
        get => _isLoading;
        private set => SetAndNotify(ref _isLoading, value);
    }
    
    public async Task LoadDataAsync()
    {
        IsLoading = true;
        try
        {
            var result = await FetchDataFromServerAsync();
            Data = result;
        }
        finally
        {
            IsLoading = false;
        }
    }
}
```

### 4. 错误处理

```csharp
public class RobustViewModel : PropertyChangedBase
{
    private string _data;
    
    public string Data
    {
        get => _data;
        set
        {
            try
            {
                SetAndNotify(ref _data, value);
                ClearError();
            }
            catch (Exception ex)
            {
                SetError(ex.Message);
            }
        }
    }
    
    private void SetError(string message)
    {
        // 设置错误状态
    }
    
    private void ClearError()
    {
        // 清除错误状态
    }
}
```

## 性能优化

### 1. 避免不必要的通知

```csharp
public class OptimizedViewModel : PropertyChangedBase
{
    private string _cachedValue;
    
    public void UpdateValue(string newValue)
    {
        // 只在值真正变化时触发通知
        if (SetAndNotify(ref _cachedValue, newValue))
        {
            // 值确实变化了，执行相关操作
            UpdateDependentData();
        }
    }
}
```

### 2. 批量更新模式

```csharp
public class BatchUpdateViewModel : PropertyChangedBase
{
    private bool _isBatchUpdating;
    
    public void BeginBatchUpdate()
    {
        _isBatchUpdating = true;
    }
    
    public void EndBatchUpdate()
    {
        _isBatchUpdating = false;
        Refresh(); // 通知所有属性更新
    }
    
    protected override bool SetAndNotify<T>(ref T field, T value, [CallerMemberName] string propertyName = null)
    {
        if (_isBatchUpdating)
        {
            field = value;
            return true; // 不立即通知
        }
        return base.SetAndNotify(ref field, value, propertyName);
    }
}
```

## 调试技巧

### 1. 添加调试输出

```csharp
public class DebugViewModel : PropertyChangedBase
{
    protected override bool SetAndNotify<T>(ref T field, T value, [CallerMemberName] string propertyName = null)
    {
        var changed = base.SetAndNotify(ref field, value, propertyName);
        if (changed)
        {
            Debug.WriteLine($"Property {propertyName} changed from {field} to {value}");
        }
        return changed;
    }
}
```

### 2. 使用诊断工具

```csharp
public class DiagnosticViewModel : PropertyChangedBase
{
    public event EventHandler<PropertyChangedEventArgs> PropertyChanging;
    
    protected override bool SetAndNotify<T>(ref T field, T value, [CallerMemberName] string propertyName = null)
    {
        if (!EqualityComparer<T>.Default.Equals(field, value))
        {
            PropertyChanging?.Invoke(this, new PropertyChangedEventArgs(propertyName));
            return base.SetAndNotify(ref field, value, propertyName);
        }
        return false;
    }
}
```

## 完整示例

### 1. 基础 ViewModel 示例

```csharp
public class UserViewModel : PropertyChangedBase
{
    private string _username;
    private string _email;
    private DateTime _birthDate;
    private bool _isActive;
    
    public string Username
    {
        get => _username;
        set => SetAndNotify(ref _username, value);
    }
    
    public string Email
    {
        get => _email;
        set => SetAndNotify(ref _email, value);
    }
    
    public DateTime BirthDate
    {
        get => _birthDate;
        set
        {
            if (SetAndNotify(ref _birthDate, value))
            {
                NotifyOfPropertyChanged(() => Age);
            }
        }
    }
    
    public bool IsActive
    {
        get => _isActive;
        set => SetAndNotify(ref _isActive, value);
    }
    
    public int Age => DateTime.Today.Year - BirthDate.Year;
    
    public void Reset()
    {
        Username = string.Empty;
        Email = string.Empty;
        BirthDate = DateTime.Today;
        IsActive = true;
    }
}
```

### 2. 复杂业务逻辑示例

```csharp
public class ShoppingCartViewModel : PropertyChangedBase
{
    private readonly ObservableCollection<CartItem> _items;
    
    public ShoppingCartViewModel()
    {
        _items = new ObservableCollection<CartItem>();
        _items.CollectionChanged += OnItemsCollectionChanged;
    }
    
    public ObservableCollection<CartItem> Items => _items;
    
    public decimal Subtotal => Items.Sum(item => item.LineTotal);
    public decimal Tax => Subtotal * 0.08m;
    public decimal Total => Subtotal + Tax;
    
    public void AddItem(Product product, int quantity)
    {
        var existingItem = Items.FirstOrDefault(item => item.Product.Id == product.Id);
        if (existingItem != null)
        {
            existingItem.Quantity += quantity;
        }
        else
        {
            Items.Add(new CartItem { Product = product, Quantity = quantity });
        }
    }
    
    public void RemoveItem(CartItem item)
    {
        Items.Remove(item);
    }
    
    private void OnItemsCollectionChanged(object sender, NotifyCollectionChangedEventArgs e)
    {
        NotifyOfPropertyChanged(() => Subtotal);
        NotifyOfPropertyChanged(() => Tax);
        NotifyOfPropertyChanged(() => Total);
    }
}

public class CartItem : PropertyChangedBase
{
    private Product _product;
    private int _quantity;
    
    public Product Product
    {
        get => _product;
        set => SetAndNotify(ref _product, value);
    }
    
    public int Quantity
    {
        get => _quantity;
        set
        {
            if (SetAndNotify(ref _quantity, value))
            {
                NotifyOfPropertyChanged(() => LineTotal);
            }
        }
    }
    
    public decimal LineTotal => Product.Price * Quantity;
}
```

## 常见问题解答

### Q1: 什么时候应该使用 PropertyChangedBase？

**A**: 所有需要数据绑定的 ViewModel 都应该继承自 PropertyChangedBase，它提供了完整的属性变更通知支持。

### Q2: SetAndNotify 和 NotifyOfPropertyChanged 有什么区别？

**A**: 
- `SetAndNotify`: 原子操作，同时设置值和触发通知，自动处理值比较
- `NotifyOfPropertyChanged`: 仅触发通知，需要手动设置值

### Q3: 如何处理循环依赖的属性通知？

**A**: 使用标志变量避免循环：

```csharp
private bool _isUpdating;
public decimal Value1
{
    get => _value1;
    set
    {
        if (_isUpdating) return;
        _isUpdating = true;
        SetAndNotify(ref _value1, value);
        Value2 = value * 2;
        _isUpdating = false;
    }
}
```

### Q4: 如何在异步操作中安全地更新属性？

**A**: Stylet 的 PropertyChangedBase 自动处理线程调度，确保属性变更通知在 UI 线程执行。

## 总结

PropertyChangedBase 是 Stylet 框架中最重要的基类之一，它提供了：

1. **简洁的 API**：通过 SetAndNotify 方法大大简化了属性变更通知的实现
2. **类型安全**：支持表达式树方式的属性名指定
3. **性能优化**：自动避免不必要的属性变更通知
4. **线程安全**：内置的调度器确保 UI 线程安全
5. **扩展性**：支持自定义比较器和变更后回调

掌握 PropertyChangedBase 的使用是构建高质量 Stylet 应用程序的基础，它让 ViewModel 的编写变得简单而优雅。