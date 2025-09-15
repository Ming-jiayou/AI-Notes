# WPF `PresentationUI` 目录详解

本文档旨在解释 `wpf/src/Microsoft.DotNet.Wpf/src/PresentationUI` 目录的内容和该程序集在 WPF 中的作用。

## `PresentationUI` 概述

与 `PresentationCore` 和 `PresentationFramework` 这些包含大量实现代码的核心程序集不同，`PresentationUI` 在现代 .NET (Core) 版本的 WPF 中扮演着一个非常特殊的角色：它主要是一个**类型转发程序集 (Type Forwarding Assembly)**。

## 什么是类型转发？

类型转发是一种机制，它允许开发者将一个类型从一个程序集移动到另一个程序集，而不会破坏依赖于原始程序集的现有应用程序。

当一个程序集（例如 `PresentationUI.dll`）声明它将某个类型（例如 `System.Windows.Controls.PrintDialog`）转发到另一个程序集（例如 `PresentationFramework.dll`）时，任何试图从 `PresentationUI.dll` 加载该类型的请求都会被公共语言运行时 (CLR) 自动重定向到 `PresentationFramework.dll`。

## `PresentationUI` 的作用

在早期的 .NET Framework 版本中，`PresentationUI.dll` 确实包含了一些具体的 UI 组件实现，最典型的就是与打印相关的对话框。

然而，在 .NET (Core) 的演进过程中，为了更好地组织代码结构，这些类型被合并到了 `PresentationFramework.dll` 中。为了保持与那些为 .NET Framework 编译的、并且引用了 `PresentationUI.dll` 的旧应用程序和库的**二进制兼容性**，`PresentationUI.dll` 被保留了下来，但其内容被清空，只留下了类型转发的声明。

因此，当您查看 `PresentationUI` 的项目文件 (`PresentationUI.csproj`) 时，您会看到类似以下的条目：

```xml
<TypeForwardingEntry Include="System.Windows.Controls.PrintDialog" />
```

这正是告诉编译器和运行时：“`PrintDialog` 这个类型虽然名义上属于我，但它的实际实现请去 `PresentationFramework.dll` 中寻找。”

## 目录结构

`PresentationUI` 目录非常简单，通常只包含：

*   **`PresentationUI.csproj`**: 项目文件，其核心内容就是定义类型转发规则。
*   **`ref/`**: 包含引用程序集，其中也只定义了那些被转发的类型签名。
*   **`Resources/`**: 可能包含一些与程序集相关的资源。

## 总结

`PresentationUI` 是 WPF 为了保持向后兼容性而存在的一个历史产物。它本身不包含活动的功能代码，而是像一个“地址转发表”，确保对旧有类型的引用能够正确定位到它们在 `PresentationFramework` 中的新家。对于学习 WPF 现代源代码来说，了解它的这个作用即可，无需深入研究其内部。
