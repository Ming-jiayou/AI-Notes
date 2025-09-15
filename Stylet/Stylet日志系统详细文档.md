# Stylet日志系统详细文档

## 概述

Stylet提供了一个轻量级、灵活的日志系统，专为框架内部使用而设计。该系统采用基于接口的简单方法，支持可插拔的实现，允许开发者集成他们偏好的日志框架或使用内置选项。

## 架构设计

Stylet日志系统由四个主要组件构成：

1. **ILogger** - 核心日志接口
2. **LogManager** - 日志创建和配置的中央管理器
3. **NullLogger** - 空实现（默认）
4. **TraceLogger** - 使用System.Diagnostics.Trace的调试输出实现

## 核心组件详解

### 1. ILogger接口

[`ILogger`](Stylet/Logging/ILogger.cs:8)接口定义了Stylet使用的所有日志记录器的契约。它提供了三个日志级别：

```csharp
public interface ILogger
{
    void Info(string format, params object[] args);
    void Warn(string format, params object[] args);
    void Error(Exception exception, string message = null);
}
```

#### 方法说明：
- **Info**: 记录信息性消息，用于一般应用程序流程
- **Warn**: 记录警告消息，用于潜在的问题情况
- **Error**: 记录错误消息，包含异常详情用于调试

所有方法都通过`format`参数和`args`数组支持字符串格式化，遵循标准的.NET字符串格式化约定。

### 2. LogManager管理器

[`LogManager`](Stylet/Logging/LogManager.cs:8)是一个静态类，作为日志创建和配置的中央枢纽。它提供：

#### 关键属性：
- **Enabled** (bool): 控制日志是否激活
  - 默认值：`false`
  - 当为`false`时：返回`NullLogger`实例（不记录日志）
  - 当为`true`时：使用`LoggerFactory`创建实际日志记录器

- **LoggerFactory** (Func<string, ILogger>): 用于创建日志记录器的工厂委托
  - 默认值：`name => new TraceLogger(name)`
  - 可以替换为自定义实现

#### 方法：
- **GetLogger(Type type)**: 使用类型的全名创建日志记录器
- **GetLogger(string name)**: 使用特定名称创建日志记录器

#### 使用模式：
```csharp
// 启用日志记录
LogManager.Enabled = true;

// 使用自定义日志工厂
LogManager.LoggerFactory = name => new MyCustomLogger(name);

// 获取日志记录器
var logger = LogManager.GetLogger(typeof(MyClass));
```

### 3. NullLogger空实现

[`NullLogger`](Stylet/Logging/NullLogger.cs:8)是`ILogger`的空实现，会丢弃所有日志消息。

#### 特点：
- 实现了`ILogger`接口的所有方法
- 所有方法体为空，不执行任何操作
- 用作默认日志记录器，避免空引用异常
- 零性能开销，适合生产环境中禁用日志的场景

#### 实现代码：
```csharp
public class NullLogger : ILogger
{
    public void Info(string format, params object[] args) { }
    public void Warn(string format, params object[] args) { }
    public void Error(Exception exception, string message = null) { }
}
```

### 4. TraceLogger实现

[`TraceLogger`](Stylet/Logging/TraceLogger.cs:9)是`ILogger`的具体实现，使用`System.Diagnostics.Trace`将日志输出到调试输出。

#### 特点：
- 使用`Trace.WriteLine`输出日志
- 包含日志级别、类名和格式化消息
- 所有输出都带有"Stylet"类别标识
- 适合开发和调试阶段使用

#### 输出格式：
- **Info**: `INFO [类名] 消息内容`
- **Warn**: `WARN [类名] 消息内容`
- **Error**: `ERROR [类名] 异常信息` 或 `ERROR [类名] 消息 异常信息`

#### 实现细节：
```csharp
public void Info(string format, params object[] args)
{
    Trace.WriteLine(string.Format("INFO [{1}] {0}", string.Format(format, args), this.name), "Stylet");
}

public void Warn(string format, params object[] args)
{
    Trace.WriteLine(string.Format("WARN [{1}] {0}", string.Format(format, args), this.name), "Stylet");
}

public void Error(Exception exception, string message = null)
{
    if (message == null)
        Trace.WriteLine(string.Format("ERROR [{1}] {0}", exception, this.name), "Stylet");
    else
        Trace.WriteLine(string.Format("ERROR [{2}] {0} {1}", message, exception, this.name), "Stylet");
}
```

## 使用场景和最佳实践

### 1. 开发阶段
在开发过程中，启用日志记录有助于调试：
```csharp
// 在应用程序启动时
LogManager.Enabled = true;
// 使用默认的TraceLogger
```

### 2. 生产环境
在生产环境中，可以禁用日志或使用自定义实现：
```csharp
// 禁用日志（默认行为）
LogManager.Enabled = false;

// 或使用自定义日志框架
LogManager.LoggerFactory = name => new NLogLogger(name);
```

### 3. 集成第三方日志框架
可以轻松集成Serilog、NLog、log4net等：

```csharp
// 集成Serilog示例
LogManager.LoggerFactory = name => new SerilogLogger(name);
```

### 4. 在Stylet组件中使用
Stylet内部组件通过以下方式获取日志记录器：
```csharp
private static readonly ILogger logger = LogManager.GetLogger(typeof(MyClass));

// 使用示例
logger.Info("组件已初始化");
logger.Warn("检测到潜在问题: {0}", warningDetails);
logger.Error(exception, "操作失败");
```

## 扩展性

Stylet的日志系统设计具有良好的扩展性：

1. **自定义日志实现**：通过实现`ILogger`接口创建自定义日志记录器
2. **日志级别扩展**：虽然接口只定义了三个级别，但自定义实现可以添加更多级别
3. **日志目标**：可以轻松将日志输出到文件、数据库、云服务等
4. **格式化控制**：自定义实现可以完全控制日志消息的格式

## 总结

Stylet的日志系统采用了简洁而有效的设计：
- 通过接口抽象确保灵活性
- 默认提供空实现避免性能开销
- 内置调试输出实现满足基本需求
- 易于扩展和集成第三方框架
- 全局配置管理简化使用

这种设计使得Stylet既能满足框架内部的日志需求，又不会给应用程序增加不必要的依赖或性能负担。