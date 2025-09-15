# Stylet中s:Action第五阶段：命令执行（CommandAction.Execute）实现详解

## 概述

在Stylet框架中，`s:Action`的第五阶段是**命令执行**，这个阶段的核心任务是通过反射调用ViewModel中的方法（如`SayHello()`），并处理各种边界情况和异常。这个过程由`CommandAction.Execute`和`ActionBase.InvokeTargetMethod`方法完成，是整个命令系统的最终执行环节。

## 核心执行流程

### 1. Execute方法入口

[`CommandAction.Execute`](Stylet/Xaml/CommandAction.cs:160)是命令执行的入口点：

```csharp
public void Execute(object parameter)
{
    this.AssertTargetSet();  // 验证目标已设置
    
    // 边界检查
    if (this.Target == null || this.TargetMethodInfo == null)
        return;

    // 构建参数并执行
    object[] parameters = this.TargetMethodInfo.GetParameters().Length == 1 
        ? new[] { parameter } 
        : null;
    
    this.InvokeTargetMethod(parameters);
}
```

### 2. 目标验证机制

[`AssertTargetSet`](Stylet/Xaml/ActionBase.cs:199)确保执行环境正确：

```csharp
private protected void AssertTargetSet()
{
    // 检查ActionTarget是否已设置
    if (this.Target == View.InitialActionTarget)
    {
        var ex = new ActionNotSetException(
            $"View.ActionTarget not set on control {this.Subject} (method {this.MethodName}). " +
            $"This probably means the control hasn't inherited it from a parent...");
        logger.Error(ex);
        throw ex;
    }

    // 检查方法是否存在
    if (this.TargetMethodInfo == null && this.ActionNonExistentBehaviour == ActionUnavailableBehaviour.Throw)
    {
        var ex = new ActionNotFoundException(
            $"Unable to find method {this.MethodName} on {this.TargetName()}");
        logger.Error(ex);
        throw ex;
    }
}
```

## 方法调用实现

### 1. 参数构建

[`InvokeTargetMethod`](Stylet/Xaml/ActionBase.cs:224)负责实际的方法调用：

```csharp
private protected void InvokeTargetMethod(object[] parameters)
{
    logger.Info("Invoking method {0} on {1} with parameters ({2})", 
        this.MethodName, this.TargetName(), 
        parameters == null ? "none" : string.Join(", ", parameters));

    try
    {
        object target = this.TargetMethodInfo.IsStatic ? null : this.Target;
        object result = this.TargetMethodInfo.Invoke(target, parameters);
        
        // 处理异步结果
        if (result is Task task)
        {
            AwaitTask(task);
        }
    }
    catch (TargetInvocationException e)
    {
        // 异常包装处理
        logger.Error(e.InnerException, 
            $"Failed to invoke method {this.MethodName} on {this.TargetName()}");
        ExceptionDispatchInfo.Capture(e.InnerException).Throw();
    }

    async void AwaitTask(Task t) => await t;
}
```

### 2. 方法签名支持

支持多种方法签名：

```csharp
// 无参数方法
public void SayHello() { }

// 带参数方法
public void SayHello(string name) { }

// 异步无参数方法
public async Task SayHelloAsync() { }

// 异步带参数方法
public async Task SayHelloAsync(string name) { }

// 静态方法
public static void StaticMethod() { }
```

## 参数处理机制

### 1. 参数类型转换

WPF自动处理参数类型转换：

```xml
<!-- 字符串参数 -->
<Button Command="{s:Action SetName}" CommandParameter="John" />

<!-- 绑定参数 -->
<Button Command="{s:Action SaveDocument}" 
        CommandParameter="{Binding SelectedDocument}" />

<!-- 数值参数 -->
<Button Command="{s:Action SetAge}" CommandParameter="25" />
```

### 2. 参数验证

```csharp
public void SetAge(int age)
{
    if (age < 0 || age > 150)
        throw new ArgumentOutOfRangeException(nameof(age));
    
    this.Age = age;
}
```

## 异常处理机制

### 1. 异常包装与解包

[`InvokeTargetMethod`](Stylet/Xaml/ActionBase.cs:238)处理目标方法抛出的异常：

```csharp
catch (TargetInvocationException e)
{
    // 解包TargetInvocationException，保留原始堆栈
    ExceptionDispatchInfo.Capture(e.InnerException).Throw();
}
```

### 2. 异常类型映射

- **ActionNotSetException**: ActionTarget未设置
- **ActionTargetNullException**: 目标对象为null
- **ActionNotFoundException**: 方法不存在
- **ActionSignatureInvalidException**: 方法签名无效

### 3. 用户友好的错误处理

```csharp
public void SafeExecute()
{
    try
    {
        // 可能抛出异常的业务逻辑
        RiskyOperation();
    }
    catch (Exception ex)
    {
        // 用户友好的错误处理
        ShowErrorMessage(ex.Message);
        logger.Error(ex, "执行操作时发生错误");
    }
}
```

## 异步执行支持

### 1. Task返回类型处理

支持`async Task`方法的异步执行：

```csharp
public async Task LoadDataAsync()
{
    IsLoading = true;
    try
    {
        var data = await dataService.LoadAsync();
        Items = new ObservableCollection<Item>(data);
    }
    finally
    {
        IsLoading = false;
    }
}

public bool CanLoadDataAsync => !IsLoading;
```

### 2. async void处理

对于`async void`方法的特殊处理：

```csharp
public async void SaveAsync()
{
    await SaveImplementationAsync();
}

// 注意：async void方法无法等待完成，异常需要内部处理
```

### 3. 异步异常处理

```csharp
public async Task ProcessDataAsync()
{
    try
    {
        await ProcessAsync();
    }
    catch (Exception ex)
    {
        // 异步方法中的异常处理
        await ShowErrorDialogAsync(ex.Message);
    }
}
```

## 静态方法调用

### 1. 静态方法支持

支持调用静态方法：

```csharp
public static class GlobalCommands
{
    public static void ShowAbout()
    {
        MessageBox.Show("About this application");
    }
    
    public static bool CanShowAbout => true;
}

// XAML中使用
<Button Command="{s:Action ShowAbout}" />
```

### 2. 静态方法查找

在[`UpdateActionTarget`](Stylet/Xaml/ActionBase.cs:150)中处理静态方法：

```csharp
BindingFlags bindingFlags;
if (newTarget is Type newTargetType)
{
    bindingFlags = BindingFlags.Public | BindingFlags.Static;
}
else
{
    newTargetType = newTarget.GetType();
    bindingFlags = BindingFlags.Public | BindingFlags.Instance;
}
```

## 性能优化

### 1. 方法缓存

反射结果在`TargetMethodInfo`中缓存，避免重复查找：

```csharp
protected MethodInfo TargetMethodInfo { get; private set; }
```

### 2. 参数数组重用

对于无参数方法，重用空数组：

```csharp
object[] parameters = this.TargetMethodInfo.GetParameters().Length == 1 
    ? new[] { parameter } 
    : null;  // 无参数时返回null，避免创建空数组
```

## 调试与诊断

### 1. 详细日志记录

记录每次方法调用的详细信息：

```csharp
logger.Info("Invoking method {0} on {1} with parameters ({2})", 
    this.MethodName, this.TargetName(), 
    parameters == null ? "none" : string.Join(", ", parameters));
```

### 2. 调试输出

在ViewModel中添加调试信息：

```csharp
public void DebugMethod(string parameter)
{
    Debug.WriteLine($"DebugMethod called with parameter: {parameter}");
    Debug.WriteLine($"Current time: {DateTime.Now}");
}
```

### 3. 性能监控

```csharp
public void MonitoredMethod()
{
    var stopwatch = Stopwatch.StartNew();
    
    // 实际业务逻辑
    DoWork();
    
    stopwatch.Stop();
    logger.Info($"MonitoredMethod executed in {stopwatch.ElapsedMilliseconds}ms");
}
```

## 高级特性

### 1. 方法重载支持

支持方法重载，通过参数类型匹配：

```csharp
public void Process(int number) { }
public void Process(string text) { }
public void Process(object data) { }
```

### 2. 泛型方法支持

通过反射支持泛型方法：

```csharp
public void GenericMethod<T>(T value)
{
    // 泛型方法实现
}

// 调用时通过反射处理类型参数
```

### 3. 可选参数支持

支持C#可选参数：

```csharp
public void ProcessData(string data, bool validate = true)
{
    if (validate)
        ValidateData(data);
    
    Process(data);
}
```

## 实际应用示例

### 1. 基础命令执行

```csharp
public class ShellViewModel : Screen
{
    public void ShowMessage()
    {
        MessageBox.Show("Hello from ViewModel!");
    }
    
    public bool CanShowMessage => true;
}
```

### 2. 带参数的命令

```csharp
public class DocumentViewModel : Screen
{
    public void SaveDocument(Document doc)
    {
        if (doc == null)
            throw new ArgumentNullException(nameof(doc));
            
        documentService.Save(doc);
        ShowMessage($"Document {doc.Name} saved successfully");
    }
    
    public bool CanSaveDocument(Document doc) => doc?.IsModified == true;
}
```

### 3. 异步命令执行

```csharp
public class AsyncViewModel : Screen
{
    public async Task LoadDataAsync()
    {
        try
        {
            var data = await dataService.LoadDataAsync();
            Items = new ObservableCollection<Item>(data);
        }
        catch (Exception ex)
        {
            ShowError($"Failed to load data: {ex.Message}");
        }
    }
    
    public bool CanLoadDataAsync => !IsLoading;
}
```

### 4. 错误处理命令

```csharp
public class ErrorHandlingViewModel : Screen
{
    public void ProcessWithValidation()
    {
        try
        {
            ValidateInput();
            ProcessData();
            ShowSuccess("Processing completed successfully");
        }
        catch (ValidationException ex)
        {
            ShowValidationError(ex.Message);
        }
        catch (Exception ex)
        {
            logger.Error(ex, "Unexpected error during processing");
            ShowError("An unexpected error occurred. Please try again.");
        }
    }
    
    public bool CanProcessWithValidation => HasValidInput;
}
```

## 边界情况处理

### 1. 空参数处理

```csharp
public void HandleNullParameter(object data)
{
    if (data == null)
    {
        logger.Warn("Received null parameter");
        return;
    }
    
    ProcessData(data);
}
```

### 2. 类型转换失败

```csharp
public void SetCount(string countText)
{
    if (!int.TryParse(countText, out int count))
    {
        ShowError("Please enter a valid number");
        return;
    }
    
    Count = count;
}
```

### 3. 并发执行保护

```csharp
private int executionCount = 0;

public void ConcurrentSafeMethod()
{
    if (Interlocked.Increment(ref executionCount) > 1)
    {
        ShowWarning("Method already executing");
        return;
    }
    
    try
    {
        // 实际执行逻辑
    }
    finally
    {
        Interlocked.Decrement(ref executionCount);
    }
}
```

## 测试与验证

### 1. 单元测试示例

```csharp
[Test]
public void Execute_ShouldCallTargetMethod()
{
    var mock = new MockViewModel();
    var command = new CommandAction(mock, "TestMethod", 
        ActionUnavailableBehaviour.Throw, ActionUnavailableBehaviour.Throw);
    
    command.Execute(null);
    
    Assert.IsTrue(mock.MethodCalled);
}

[Test]
public void Execute_WithParameter_ShouldPassParameter()
{
    var mock = new MockViewModel();
    var command = new CommandAction(mock, "TestMethodWithParam", 
        ActionUnavailableBehaviour.Throw, ActionUnavailableBehaviour.Throw);
    
    command.Execute("test parameter");
    
    Assert.AreEqual("test parameter", mock.LastParameter);
}
```

## 总结

Stylet的第五阶段通过`CommandAction.Execute`实现了完整的命令执行机制，其设计特点包括：

1. **灵活的方法调用**：支持实例方法、静态方法、异步方法
2. **完善的异常处理**：提供详细的错误信息和调试支持
3. **性能优化**：方法缓存、参数重用、最小化反射开销
4. **类型安全**：自动参数转换和验证
5. **调试友好**：详细的日志记录和调试输出
6. **测试支持**：易于单元测试和验证

这一机制使得开发者可以专注于业务逻辑实现，而无需关心命令执行的底层细节，真正实现了MVVM模式中的命令驱动架构。