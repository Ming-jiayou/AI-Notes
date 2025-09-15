# WPF `System.Xaml` 目录详解

本文档旨在详细解释 `wpf/src/Microsoft.DotNet.Wpf/src/System.Xaml` 目录下的内容，帮助您理解 .NET XAML 服务框架的功能和结构。

## `System.Xaml` 概述

`System.Xaml` 是一个核心的 .NET 程序集，它提供了用于处理 XAML 语言的服务。它不仅仅是为 WPF 服务的，也被其他使用 XAML 的技术（如 Windows Workflow Foundation (WF) 和 Windows Communication Foundation (WCF)）所使用。可以将其视为一个通用的 XAML 框架，定义了 XAML 的读取、写入和对象模型表示。

`System.Xaml` 的主要职责包括：

*   **XAML 读取器 (Readers)**: 提供了 `XamlReader` 的抽象基类和具体的实现（如 `XamlObjectReader` 和 `XamlXmlReader`），用于将 XAML 文本或 XML 节点流解析成一个标准的 XAML 节点流 (XAML Node Stream)。
*   **XAML 写入器 (Writers)**: 提供了 `XamlWriter` 的抽象基类和具体的实现（如 `XamlObjectWriter` 和 `XamlXmlWriter`），用于将 XAML 节点流转换成对象图 (Object Graph) 或 XAML 文本。
*   **XAML 模式上下文 (Schema Context)**: `XamlSchemaContext` 是核心概念之一，它负责解析程序集和 CLR 类型，并将其映射到 XAML 类型系统（`XamlType`, `XamlMember` 等）。它充当了 CLR 世界和 XAML 世界之间的桥梁。
*   **XAML 类型系统**: 定义了一组类（`XamlType`, `XamlMember`, `XamlDirective`）来在内存中表示 XAML 的概念，与具体的 CLR 类型系统解耦。
*   **服务提供者**: 提供了一系列接口（如 `IXamlNamespaceResolver`, `IRootObjectProvider`），允许在 XAML 处理管道中查询上下文信息。

## XAML 处理管道

理解 `System.Xaml` 的关键是理解其处理管道：

1.  **加载 (Load Path)**:
    `XAML Text` -> `XamlXmlReader` -> `XAML Node Stream` -> `XamlObjectWriter` -> `Object Graph`

2.  **保存 (Save Path)**:
    `Object Graph` -> `XamlObjectReader` -> `XAML Node Stream` -> `XamlXmlWriter` -> `XAML Text`

`XamlServices` 类提供了这个管道的高级封装，其 `Load()` 和 `Save()` 方法是与 `System.Xaml` 交互的最简单方式。

## 目录结构详解

以下是 `System.Xaml` 目录中关键文件和子目录的说明：

*   **`System.Xaml.csproj`**: 项目文件，定义了程序集的编译方式和依赖。

*   **`System/Xaml/`**: 包含了 `System.Xaml` 的所有核心实现。
    *   **`XamlReader.cs` / `XamlWriter.cs`**: 定义了 XAML 读取器和写入器的抽象基类。
    *   **`XamlXmlReader.cs` / `XamlObjectReader.cs`**: 读取器的具体实现，分别用于从 XML 和从现有对象图读取。
    *   **`XamlXmlWriter.cs` / `XamlObjectWriter.cs`**: 写入器的具体实现，分别用于写入到 XML 和创建对象图。
    *   **`XamlServices.cs`**: 提供了高级静态方法（`Load`, `Save`, `Transform`）来简化对 XAML 处理管道的使用。
    *   **`Schema/`**: 包含了 XAML 模式上下文的核心实现。
        *   `XamlSchemaContext.cs`: 核心的模式上下文类。
        *   `XamlType.cs`, `XamlMember.cs`: XAML 类型系统的核心类，分别代表一个 XAML 类型和一个 XAML 成员（属性或事件）。
    *   **`Parser/`**: 包含了 XAML 的低级解析逻辑。
    *   **`Context/`**: 包含了一些模式上下文相关的类。
    *   **`Runtime/`**: 包含了一些运行时辅助类。
    *   **`XamlLanguage.cs`**: 定义了 XAML 语言的内在指令（如 `x:Key`, `x:Name`, `x:Type`）和内在类型（如 `x:Array`, `x:Null`）。

*   **`ms/`**: 包含了微软内部使用的非公共类。

*   **`ref/`**: 包含了程序集的引用程序集。

## 总结

`System.Xaml` 是 .NET 中 XAML 功能的基石。它提供了一个可扩展的、与具体 UI 框架无关的工具集，用于序列化和反序列化对象图到 XAML。WPF 的 `System.Windows.Markup.XamlReader` 和 `XamlWriter` 在内部就大量依赖于 `System.Xaml` 提供的服务来完成其工作。理解 `System.Xaml` 的处理管道和模式上下文概念，对于深入理解 WPF 的 XAML 加载机制以及进行高级 XAML 定制至关重要。
