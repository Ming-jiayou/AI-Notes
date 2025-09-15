# WPF `PresentationFramework` 目录详解

本文档旨在详细解释 `wpf/src/Microsoft.DotNet.Wpf/src/PresentationFramework` 目录下的内容，帮助您理解 WPF 应用程序框架层的功能和结构。

## `PresentationFramework` 概述

`PresentationFramework` 是 WPF 框架的顶层程序集，它直接构建于 `PresentationCore` 和 `WindowsBase` 之上。如果说 `PresentationCore` 提供了 WPF 的“骨架”（可视化和输入系统），那么 `PresentationFramework` 则为其添加了“血肉”，提供了构建功能齐全的桌面应用程序所需的所有高级功能和控件。

该程序集主要负责：

*   **控件库 (Controls)**: 提供了一整套丰富的、可自定义的控件，如 `Button`、`TextBox`、`ListBox`、`DataGrid` 等。
*   **应用程序模型**: 定义了 `Application` 和 `Window` 类，管理应用程序的生命周期、窗口和资源。
*   **数据绑定**: 提供了强大的数据绑定引擎，用于在 UI 和数据源之间同步数据。
*   **样式和模板 (Styling and Templating)**: 允许开发者完全自定义控件的外观和行为，而无需重写其核心逻辑。
*   **布局系统**: 提供了多种布局容器（如 `Grid`、`StackPanel`、`DockPanel`），用于灵活地组织 UI 元素。
*   **命令系统 (Commands)**: 提供了一种将用户操作（如点击按钮）与处理逻辑解耦的机制。
*   **导航**: 为基于页面的应用程序（如 `NavigationWindow`）提供导航支持。

## 目录结构详解

以下是 `PresentationFramework` 目录中关键文件和子目录的说明：

*   **`PresentationFramework.csproj`**: 项目文件，定义了该程序集的编译方式、依赖项和包含的文件。

*   **`System/Windows/`**: 这是 `PresentationFramework` 的核心，包含了绝大多数公共 API 的实现。
    *   **`Controls/`**: 这是最大也是最重要的子目录之一，包含了所有标准 WPF 控件的实现。
        *   `Primitives/`: 包含用于构建更复杂控件的基础组件，如 `ButtonBase`、`ScrollBar`、`Thumb`。
        *   `Button.cs`, `TextBox.cs`, `ListBox.cs`, `DataGrid.cs`: 具体控件的实现。
        *   `Control.cs`: 所有控件的基类，定义了 `Template` 属性。
        *   `ContentControl.cs`, `ItemsControl.cs`: 分别是包含单个内容和项目集合的控件的基类。
    *   **`FrameworkElement.cs`**: 一个至关重要的基类，继承自 `UIElement`。它添加了对布局（`Width`, `Height`, `Margin`）、数据上下文（`DataContext`）、样式（`Style`）和工具提示（`ToolTip`）的支持。几乎所有 WPF 控件都直接或间接继承自它。
    *   **`Application.cs`**: 定义了 `Application` 类，是 WPF 应用程序的入口点和中心协调器。
    *   **`Window.cs`**: 定义了 `Window` 类，代表一个顶级窗口。
    *   **`Data/`**: 包含了数据绑定系统的核心类，如 `Binding`、`BindingExpression`。
    *   **`Style.cs`**: 定义了 `Style` 类，用于封装一组属性设置，以便重用。
    *   **`ControlTemplate.cs`**: 定义了 `ControlTemplate` 类，用于完全替换控件的可视化树。
    *   **`DataTemplate.cs`**: 定义了 `DataTemplate` 类，用于指定数据对象的可视化呈现方式。
    *   **`Markup/`**: 包含了与 XAML 相关的扩展，如 `x:Static`、`x:Type` 等的实现。

*   **`MS/`**: 包含了微软内部使用的非公共类，用于支持公共 API 的实现。

*   **`Microsoft/`**: 包含了特定于微软实现的、但可能在其他地方也有用的代码。

*   **`Resources/`**: 包含了程序集使用的资源，例如错误消息字符串。

*   **`ref/`**: 包含了程序集的引用程序集，用于编译时进行类型检查。

## 总结

`PresentationFramework` 是 WPF 开发者最常直接交互的程序集。它将 `PresentationCore` 提供的底层功能封装成一套易于使用、功能强大的控件和应用程序服务。深入理解此程序集中的类，特别是 `FrameworkElement`、`Control`、`ItemsControl` 以及各种具体的控件，是掌握 WPF 开发的关键。同时，对数据绑定、样式和模板系统的理解，将使您能够充分利用 WPF 的灵活性和强大功能。
