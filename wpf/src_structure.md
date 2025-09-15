# WPF `src` 目录结构详解

本文档旨在详细解释 `wpf/src/Microsoft.DotNet.Wpf/src` 目录下的各个子目录，以帮助您更好地理解 WPF 的源代码结构。

## 目录结构概览

`wpf/src/Microsoft.DotNet.Wpf/src` 目录是 WPF 框架核心源代码的所在地。以下是该目录下的主要子目录及其功能说明：

*   **Common/**: 包含跨多个 WPF 程序集共享的通用工具和帮助程序代码。
*   **DirectWriteForwarder/**: 包含与 DirectWrite 相关的代码。DirectWrite 是用于高质量文本呈现的 DirectX API。此目录可能包含一个转发器或包装器，以便在 WPF 中使用 DirectWrite 的功能。
*   **Extensions/**: 包含对 WPF 的扩展，可能提供额外的控件、功能或服务。
*   **PenImc/**: 包含与手写笔输入和输入法编辑器 (Input Method Editor, IMC) 相关的代码，用于处理来自手写笔设备的输入。
*   **PresentationBuildTasks/**: 包含用于构建 WPF 应用程序的 MSBuild 任务。其中最重要的是 XAML 编译任务，它负责将 `.xaml` 文件编译成 `.baml` (二进制 XAML) 文件或代码。
*   **PresentationCore/**: WPF 的核心程序集之一。它提供了 WPF 的基础服务，包括：
    *   可视化系统 (Visual System)
    *   UI 元素基类 (UIElement, Visual)
    *   调度系统 (Dispatcher)
    *   输入处理
*   **PresentationFramework/**: WPF 的另一个核心程序集，它构建于 `PresentationCore` 之上。它提供了更高级别的应用程序框架功能，包括：
    *   一组丰富的控件 (Button, TextBox, etc.)
    *   数据绑定
    *   样式和模板
    *   布局系统
*   **PresentationUI/**: 此程序集包含 `PresentationFramework` 的 UI 部分。
*   **ReachFramework/**: 提供用于创建、查看和管理 XPS (XML Paper Specification) 文档的功能。
*   **Shared/**: 与 `Common/` 类似，包含在多个程序集之间共享的源代码。
*   **System.Printing/**: 包含与打印相关的类型和 API，用于在 WPF 应用程序中实现打印功能。
*   **System.Windows.Controls.Ribbon/**: 包含 Ribbon 控件的实现，这是一种常见的 UI 模式，用于在窗口顶部提供一组选项卡式的命令。
*   **System.Windows.Input.Manipulations/**: 包含用于处理触摸和操作事件的 API，例如平移、缩放和旋转。
*   **System.Windows.Presentation/**: 为表示层提供支持的程序集。
*   **System.Windows.Primitives/**: 包含表示层中使用的基元类型。
*   **System.Xaml/**: .NET XAML 服务的实现，为解析和加载 XAML 提供核心支持。
*   **Themes/**: 包含 WPF 控件的默认主题。每个主题（例如 Aero, Classic, Luna, Royale）都有自己的子目录，其中包含定义控件外观和感觉的资源字典。
*   **UIAutomation/**: 包含 UI 自动化框架的实现。UI 自动化是 Windows 的辅助功能框架，允许屏幕阅读器等辅助技术与应用程序交互，也用于自动化测试。
*   **WindowsBase/**: WPF 的最基础程序集之一。它提供了许多核心的底层服务，包括：
    *   依赖属性 (Dependency Property) 系统
    *   调度程序对象 (DispatcherObject)
    *   线程模型
*   **WindowsFormsIntegration/**: 提供 WPF 和 Windows Forms 之间的互操作性。它允许您在 WPF 应用程序中承载 Windows Forms 控件，反之亦然。
*   **WpfGfx/**: WPF 的低级图形层，负责与 DirectX 进行交互以渲染 UI。这部分代码主要是用 C++/CLI 编写的，作为托管代码和非托管图形 API 之间的桥梁。

希望这份文档能帮助您更好地理解 WPF 的内部工作原理。
