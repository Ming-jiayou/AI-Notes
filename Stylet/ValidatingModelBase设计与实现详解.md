# Stylet ValidatingModelBase 设计与实现详解

## 概述

ValidatingModelBase 是 Stylet 框架中提供属性验证功能的基类，它继承自 PropertyChangedBase 并实现了 INotifyDataErrorInfo 接口。这个类为 WPF MVVM 应用程序提供了完整的验证基础设施，支持同步和异步验证、属性级和实体级验证，以及与 WPF 错误模板的无缝集成。

## 设计目标

1. **标准兼容**：实现 INotifyDataErrorInfo 接口，与 WPF 验证系统兼容
2. **灵活验证**：支持自定义验证器和多种验证策略
3. **性能优化**：智能验证触发和错误缓存机制
4. **线程安全**：支持多线程环境下的验证操作
5. **异步支持**：提供异步验证方法，不阻塞 UI 线程
6. **自动验证**：支持属性变更时的自动验证

## 核心接口和依赖

### INotifyDataErrorInfo 接口实现
```csharp
public class ValidatingModelBase : PropertyChangedBase, INotifyDataErrorInfo
```

**接口成员：**
- `event EventHandler<DataErrorsChangedEventArgs> ErrorsChanged` - 错误变更事件
- `bool HasErrors { get; }` - 是否存在验证错误
- `IEnumerable GetErrors(string propertyName)` - 获取指定属性的错误

### IModelValidator 验证器接口
```csharp
protected virtual IModelValidator Validator { get; set; }
```

**设计特点：**
- 虚拟属性允许子类覆盖
- 设置验证器时自动初始化
- 支持任何验证框架的适配器模式

## 核心功能详解

### 1. 验证器管理

```csharp
protected virtual IModelValidator Validator
{
    get => this._validator;
    set
    {
        this._validator = value;
        if (this._validator != null)
            this._validator.Initialize(this);
    }
}
```

**初始化机制：**
- 验证器设置时自动初始化
- 传入当前实例作为验证上下文
- 支持验证器的延迟配置

### 2. 自动验证机制

```csharp
protected override async void OnPropertyChanged(string propertyName)
{
    base.OnPropertyChanged(propertyName);

    // 避免 HasErrors 属性变更时的递归验证
    if (this.Validator != null && this.AutoValidate && propertyName != "HasErrors")
        await this.ValidatePropertyAsync(propertyName);
}
```

**智能验证触发：**
- 属性变更时自动触发验证
- 避免递归验证（跳过 HasErrors 属性）
- 可配置的自动验证开关

### 3. 错误存储和管理

```csharp
private readonly Dictionary<string, string[]> propertyErrors = new();
private readonly SemaphoreSlim propertyErrorsLock = new(1, 1);
```

**线程安全设计：**
- 使用 Dictionary 存储属性错误
- SemaphoreSlim 提供异步锁机制
- 支持并发验证操作

### 4. 错误比较优化

```csharp
private bool ErrorsEqual(string[] e1, string[] e2)
{
    if (e1 == null && e2 == null)
        return true;
    if (e1 == null || e2 == null)
        return false;
    return e1.SequenceEqual(e2);
}
```

**性能优化：**
- 避免不必要的错误变更通知
- 精确的错误内容比较
- 减少 UI 刷新次数

## 验证方法详解

### 1. 全属性验证 - ValidateAsync

```csharp
protected virtual async Task<bool> ValidateAsync()
{
    if (this.Validator == null)
        throw new InvalidOperationException("Can't run validation if a validator hasn't been set");

    // 异步验证，避免阻塞
    Dictionary<string, IEnumerable<string>> results = await this.Validator.ValidateAllPropertiesAsync().ConfigureAwait(false);
    if (results == null)
        results = new Dictionary<string, IEnumerable<string>>();

    var changedProperties = new List<string>();
    await this.propertyErrorsLock.WaitAsync().ConfigureAwait(false);
    try
    {
        // 更新验证结果
        foreach (KeyValuePair<string, IEnumerable<string>> kvp in results)
        {
            string[] newErrors = kvp.Value?.ToArray();
            if (!this.propertyErrors.ContainsKey(kvp.Key))
                this.propertyErrors[kvp.Key] = newErrors;
            else if (this.ErrorsEqual(this.propertyErrors[kvp.Key], newErrors))
                continue; // 错误未变化，跳过
            else
                this.propertyErrors[kvp.Key] = newErrors;
            changedProperties.Add(kvp.Key);
        }

        // 处理已移除的验证结果
        foreach (string removedKey in this.propertyErrors.Keys.Except(results.Keys).ToArray())
        {
            this.propertyErrors[removedKey] = null;
            changedProperties.Add(removedKey);
        }
    }
    finally
    {
        this.propertyErrorsLock.Release();
    }

    if (changedProperties.Count > 0)
        this.OnValidationStateChanged(changedProperties);

    return !this.HasErrors;
}
```

**设计亮点：**
- **异步验证**：使用 ConfigureAwait(false) 避免死锁
- **线程安全**：使用异步锁保护共享数据
- **增量更新**：只通知实际变更的属性
- **清理机制**：自动清理已移除的验证结果

### 2. 单属性验证 - ValidatePropertyAsync

```csharp
protected virtual async Task<bool> ValidatePropertyAsync([CallerMemberName] string propertyName = null)
{
    if (this.Validator == null)
        throw new InvalidOperationException("Can't run validation if a validator hasn't been set");

    if (propertyName == null)
        propertyName = string.Empty;

    // 异步验证单个属性
    IEnumerable<string> newErrorsRaw = await this.Validator.ValidatePropertyAsync(propertyName).ConfigureAwait(false);
    string[] newErrors = newErrorsRaw?.ToArray();
    bool propertyErrorsChanged = false;

    await this.propertyErrorsLock.WaitAsync().ConfigureAwait(false);
    try
    {
        if (!this.propertyErrors.ContainsKey(propertyName))
            this.propertyErrors.Add(propertyName, null);

        if (!this.ErrorsEqual(this.propertyErrors[propertyName], newErrors))
        {
            this.propertyErrors[propertyName] = newErrors;
            propertyErrorsChanged = true;
        }
    }
    finally
    {
        this.propertyErrorsLock.Release();
    }

    if (propertyErrorsChanged)
        this.OnValidationStateChanged(new[] { propertyName });

    return newErrors == null || newErrors.Length == 0;
}
```

**特点：**
- **CallerMemberName**：自动获取属性名称
- **空值处理**：支持实体级验证（propertyName 为空）
- **增量更新**：只处理指定属性的验证结果

### 3. 手动错误记录 - RecordPropertyError

```csharp
protected virtual void RecordPropertyError(string propertyName, string[] errors)
{
    if (propertyName == null)
        propertyName = string.Empty;

    bool changed = false;
    this.propertyErrorsLock.Wait();
    try
    {
        string[] existingErrors;
        if (!this.propertyErrors.TryGetValue(propertyName, out existingErrors) || !this.ErrorsEqual(errors, existingErrors))
        {
            this.propertyErrors[propertyName] = errors;
            changed = true;
        }
    }
    finally
    {
        this.propertyErrorsLock.Release();
    }

    if (changed)
    {
        this.OnValidationStateChanged(new[] { propertyName });
    }
}
```

**用途：**
- 独立于验证器的手动错误设置
- 支持自定义验证逻辑
- 实体级错误管理

### 4. 错误清理 - ClearAllPropertyErrors

```csharp
protected virtual void ClearAllPropertyErrors()
{
    List<string> changedProperties;

    this.propertyErrorsLock.Wait();
    try
    {
        changedProperties = this.propertyErrors.Keys.ToList();
        this.propertyErrors.Clear();
    }
    finally
    {
        this.propertyErrorsLock.Release();
    }

    if (changedProperties.Count > 0)
    {
        this.OnValidationStateChanged(changedProperties);
    }
}
```

**功能：**
- 一次性清除所有属性错误
- 通知所有相关属性状态变更
- 支持重置验证状态

## 错误状态管理

### 验证状态变更通知

```csharp
protected virtual void OnValidationStateChanged(IEnumerable<string> changedProperties)
{
    this.NotifyOfPropertyChange(nameof(this.HasErrors));
    foreach (string property in changedProperties)
    {
        this.RaiseErrorsChanged(property);
    }
}
```

**通知机制：**
- 通知 HasErrors 属性变更
- 为每个变更的属性触发 ErrorsChanged 事件
- 使用 PropertyChangedDispatcher 确保线程安全

### 错误事件触发

```csharp
protected virtual void RaiseErrorsChanged(string propertyName)
{
    EventHandler<DataErrorsChangedEventArgs> handler = this.ErrorsChanged;
    if (handler != null)
        this.PropertyChangedDispatcher(() => handler(this, new DataErrorsChangedEventArgs(propertyName)));
}
```

**线程安全：**
- 使用 PropertyChangedDispatcher 调度事件
- 支持跨线程的错误通知
- 保持与 WPF 验证系统的兼容性

## 错误查询接口

### GetErrors 方法

```csharp
public virtual IEnumerable GetErrors(string propertyName)
{
    string[] errors;

    if (propertyName == null)
        propertyName = string.Empty;

    // 同步等待，但使用锁防止死锁
    this.propertyErrorsLock.Wait();
    try
    {
        this.propertyErrors.TryGetValue(propertyName, out errors);
    }
    finally
    {
        this.propertyErrorsLock.Release();
    }
    
    return errors;
}
```

**设计特点：**
- 支持实体级错误（propertyName 为空）
- 同步访问但考虑异步场景
- 返回 IEnumerable 保持接口兼容性

### HasErrors 属性

```csharp
public virtual bool HasErrors => this.propertyErrors.Values.Any(x => x != null && x.Length > 0);
```

**实时计算：**
- 动态检查是否存在错误
- 支持 INotifyDataErrorInfo 接口要求
- 用于 UI 状态控制

## 同步验证支持

### 同步包装方法

```csharp
protected bool Validate()
{
    try
    {
        return this.ValidateAsync().Result;
    }
    catch (AggregateException e)
    {
        // 解包异常，提供更好的错误信息
        throw e.InnerException;
    }
}

protected bool ValidateProperty([CallerMemberName] string propertyName = null)
{
    try
    {
        return this.ValidatePropertyAsync(propertyName).Result;
    }
    catch (AggregateException e)
    {
        throw e.InnerException;
    }
}
```

**设计考虑：**
- 为不支持异步的场景提供同步方法
- 异常解包提供更好的调试体验
- 保持 API 一致性

## 配置选项

### 自动验证开关

```csharp
protected bool AutoValidate { get; set; }

public ValidatingModelBase()
{
    this.AutoValidate = true;
}
```

**灵活性：**
- 可控制的自动验证行为
- 在构造函数中默认启用
- 支持性能优化场景

## 使用模式

### 基本验证实现

```csharp
public class MyViewModel : ValidatingModelBase
{
    private string _name;
    
    public string Name
    {
        get => _name;
        set => SetAndNotify(ref _name, value); // 自动触发验证
    }

    public MyViewModel()
    {
        // 设置自定义验证器
        this.Validator = new MyCustomValidator();
    }
}
```

### 手动错误设置

```csharp
public class ManualValidationViewModel : ValidatingModelBase
{
    public void ValidateCustomLogic()
    {
        if (someCondition)
        {
            this.RecordPropertyError(() => this.SomeProperty, new[] { "自定义错误消息" });
        }
        else
        {
            this.RecordPropertyError(() => this.SomeProperty, null); // 清除错误
        }
    }
}
```

### 异步验证

```csharp
public class AsyncValidationViewModel : ValidatingModelBase
{
    public async Task<bool> ValidateDataAsync()
    {
        // 验证所有属性
        return await this.ValidateAsync();
    }

    public async Task<bool> ValidateSpecificPropertyAsync()
    {
        // 验证特定属性
        return await this.ValidatePropertyAsync(() => this.Email);
    }
}
```

## 最佳实践

### 1. 验证器选择
- **自定义验证器**：实现 IModelValidator 接口
- **第三方集成**：适配流行的验证框架（FluentValidation、DataAnnotations 等）
- **性能考虑**：异步验证避免 UI 阻塞

### 2. 错误管理
- **及时清理**：在适当的时候清除错误状态
- **增量更新**：利用增量错误通知减少 UI 刷新
- **实体级错误**：使用空属性名处理实体级验证错误

### 3. 性能优化
- **自动验证控制**：在需要时关闭自动验证
- **批量验证**：使用 ValidateAsync 进行批量验证
- **错误缓存**：利用内置的错误缓存机制

### 4. 线程安全
- **异步方法优先**：优先使用异步验证方法
- **避免死锁**：理解 ConfigureAwait(false) 的重要性
- **并发控制**：利用内置的锁机制处理并发验证

## 与 WPF 集成

### XAML 错误模板

```xml
<TextBox Text="{Binding Name, ValidatesOnNotifyDataErrors=True, UpdateSourceTrigger=PropertyChanged}">
    <Validation.ErrorTemplate>
        <ControlTemplate>
            <StackPanel>
                <AdornedElementPlaceholder />
                <TextBlock Text="{Binding [0].ErrorContent}" Foreground="Red" />
            </StackPanel>
        </ControlTemplate>
    </Validation.ErrorTemplate>
</TextBox>
```

### 错误样式

```xml
<Style TargetType="TextBox">
    <Style.Triggers>
        <Trigger Property="Validation.HasError" Value="True">
            <Setter Property="BorderBrush" Value="Red"/>
            <Setter Property="ToolTip" Value="{Binding (Validation.Errors)[0].ErrorContent}"/>
        </Trigger>
    </Style.Triggers>
</Style>
```

## 总结

ValidatingModelBase 通过实现 INotifyDataErrorInfo 接口，为 Stylet 应用程序提供了完整的验证基础设施。它的设计充分考虑了性能、线程安全、异步操作和可扩展性，支持自定义验证器和多种验证策略。通过智能的错误管理、增量更新机制和与 WPF 验证系统的无缝集成，它为构建具有专业级验证功能的 MVVM 应用程序提供了坚实的基础。

该类的设计体现了对验证复杂性、用户体验和开发者友好性的深度考虑，是 Stylet 框架中处理数据验证的核心组件。