# WPF `PresentationCore` 目录详解

本文档旨在详细解释 `wpf/src/Microsoft.DotNet.Wpf/src/PresentationCore` 目录下的内容，帮助您理解 WPF 这一核心程序集的功能和结构。

## `PresentationCore` 概述

`PresentationCore` 是 WPF 框架的基石之一，它提供了 WPF 的核心服务和基础类型。它位于 `WindowsBase` 之上，并被 `PresentationFramework` 所依赖。可以说，`PresentationCore` 搭建了 WPF 的可视化和输入系统的桥梁。

该程序集主要负责：

*   **可视化层 (Visual Layer)**: 定义了 `Visual` 和 `UIElement` 类，它们是所有 WPF 控件和图形的渲染基础。
*   **输入系统**: 处理来自鼠标、键盘、触摸和手写笔的输入。
*   **事件系统**: 定义了路由事件 (Routed Events) 的机制。
*   **布局系统**: 提供了布局的核心接口和基元。
*   **图形和动画**: 包含 2D/3D 图形、画刷、几何图形和动画系统的核心实现。
*   **XAML 支持**: 提供了与 XAML 解析相关的底层支持。

## 目录结构详解

以下是 `PresentationCore` 目录中关键文件和子目录的说明：

*   **`PresentationCore.csproj`**: 项目文件，定义了该程序集的编译方式、依赖项和包含的文件。

*   **`System/Windows/`**: 这是 `PresentationCore` 最核心的部分，包含了绝大多数公共 API 的实现。
    *   **`UIElement.cs`**: 定义了 `UIElement` 类，这是所有 WPF 可视化元素的基类。它添加了对布局、输入、焦点和事件的支持。
    *   **`Input/`**: 包含了 WPF 输入系统的所有相关代码，例如：
        *   `Mouse.cs`, `Keyboard.cs`, `Stylus.cs`: 处理特定设备的输入。
        *   `InputManager.cs`: 协调所有输入设备。
        *   `RoutedCommand.cs`: 命令系统的实现。
    *   **`Media/`**: 包含了 WPF 的媒体集成、图形和动画系统。
        *   `Visual.cs`: 所有可渲染对象的基类。
        *   `DrawingContext.cs`: 用于绘制 2D 图形。
        *   `Brushes.cs`, `Pens.cs`: 定义了画刷和画笔。
        *   `Animation/`: 包含了动画的核心类，如 `AnimationTimeline` 和 `Clock`。
        *   `Media3D/`: 包含了所有 3D 图形相关的类。
    *   **`Markup/`**: 包含了与 XAML 处理相关的类，如 `XamlReader` 和 `XamlWriter` 的底层支持。
    *   **`Automation/`**: 包含了对 UI 自动化的支持，用于辅助功能和自动化测试。
    *   **`Documents/`**: 包含了与文档和文本布局相关的底层类，如 `TextRun` 和 `GlyphRun`。
    *   **`EventManager.cs`**: 路由事件系统的核心实现。
    *   **`DependencyObject.cs` 和 `DependencyProperty.cs`**: 虽然在 `WindowsBase` 中定义，但 `PresentationCore` 大量使用并扩展了依赖属性系统。

*   **`MS/`**: 包含了微软内部使用的非公共类。这些类通常是公共 API 的辅助实现，不应直接依赖。
    *   `MS/Internal/`: 内部工具和辅助类。
    *   `MS/Win32/`: 与 Win32 API 互操作的内部代码。

*   **`Fonts/`**: 包含了 WPF 使用的默认或备用字体文件。

*   **`Resources/`**: 包含了程序集使用的资源，例如本地化的字符串。

*   **`ref/`**: 包含了程序集的引用程序集 (reference assembly)，用于编译时类型检查。

## 总结

`PresentationCore` 是理解 WPF 工作原理的关键。它连接了底层的 `WindowsBase` 和 `milcore.dll` (非托管渲染引擎)，并为上层的 `PresentationFramework` 提供了一个丰富的、面向对象的框架，用于构建复杂的用户界面。研究此目录中的代码有助于深入理解 WPF 的渲染、输入和事件处理机制。
