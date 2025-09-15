# GlobalEventManager 源码深度阅读笔记
> 基于 .NET WPF `PresentationCore.dll` 内 `System.Windows.GlobalEventManager`（逐段通读 478 行）

---

## 一、定位与职责
`GlobalEventManager` 是 **WPF Routed Event 的「全局注册中心 + 类级处理器仓库」**。  
它负责三件事：

- 生成并缓存所有 `RoutedEvent` 实例（保证全局唯一索引）。  
- 维护每个 OwnerType 的路由事件列表（按名字唯一）。  
- 管理每个类（而非实例）对路由事件的“类级处理器”。

它是**静态工具类**，对外全部 `internal` 方法，使用者是 `EventManager` 的公开包装。

---

## 二、静态核心字段速览

| 字段 | 作用 |
|---|---|
| `_dTypedRoutedEventList` | `DTypeMap` → 只存 `DependencyObject` 及子类的路由事件列表（Key 为 `DependencyObjectType`）。 |
| `_ownerTypedRoutedEventList` | `Hashtable` → 存非 DO（如控件自身）的路由事件列表。 |
| `_dTypedClassListeners` | `DTypeMap` → 存每个 DO 类型的 **ClassHandlersStore**，即该类型及其基类的“类级路由处理器”表。 |
| `_countRoutedEvents` | `int` → 已注册路由事件计数，同时作为 **全局唯一索引发生器**（`ComputedEventIndex` 的来源）。 |
| `s_globalEventIndex` | `uint` → 为 `RoutedEvent & EventPrivateKey` 提供全局 0-2147483646 的索引。 |
| `Synchronized` | `static Lock` → 所有写入操作的全局互斥锁，支持并发注册/查找。 |

---

## 三、路由事件注册流程（RegisterRoutedEvent）

```csharp
internal static RoutedEvent RegisterRoutedEvent(
    string name,
    RoutingStrategy routingStrategy,
    Type handlerType,
    Type ownerType)
```

执行链路（17-44 行）：

1. **Name 唯一性检查**（内部断言）。  
2. **Lock** → **原子递增全局计数** `_countRoutedEvents` → 得到新的 `GlobalIndex`。  
3. `new RoutedEvent(...)`：构造实例并写入全局索引。  
4. **AddOwner**：将事件绑定到 OwnerType 列表。  
5. 返回完成注册的新 `RoutedEvent`。

---

## 四、类级处理器注册流程（RegisterClassHandler）

```csharp
RegisterClassHandler(
    Type classType,
    RoutedEvent routedEvent,
    Delegate handler,
    bool handledEventsToo)
```

要点：

- 仅允许注册在 `UIElement / ContentElement / UIElement3D` 及其派生类上（断言 56-60 行）。  
- 传入 `handler` 必须和注册时指定的 `handlerType` 类型一致（断言）。  
- 内部维护 **ClassHandlersStore**：  
  - **两层索引**  
    • Key → `DependencyObjectType`  
    • Value → 有序存储：`<RoutedEventIndex, RoutedEventHandlerInfoList>`  
- 当基类/子类有新处理器加入，子类 **自动继承并增量更新**（见 82-90 行的遍历更新）。

---

## 五、数据结构解析

1. **DTypeMap & Hashtable**
   - `DTypeMap` 内部是一个 **稀疏整型数组 + FrugalStructList** 的小集合，存 **10 → 100** 级别初始化尺寸，避免大量托管对象。  
   - 非 DO 类型用 `Hashtable`（key=Type, value=`FrugalObjectList<RoutedEvent>`）。

2. **ClassHandlersStore**
   - 本质：`<RoutedEvent, RoutedEventHandlerInfoList>` 的 **索引数组**  
   - 构造时基于启发式容量：  
     • `UIElement/ContentElement` → **80**  
     • 其他 → **1**，避免空位浪费内存。  
   - 通过 `GetUpdatedDTypedClassListeners` 递归扫描**基类处理器**并合成最终链表（382-420 行）。

---

## 六、全局索引与并发安全

- **原子递增** `Interlocked.Increment(ref s_globalEventIndex)` → 预防跨线程注册导致索引溢出（437-442 行）。  
- **全局锁 `Synchronized`** 包裹所有写操作：注册、类级处理器修改、动态 AddOwner。  
- 读路径（如 `GetRoutedEventFromName`）**仅在写后通过锁同步**；内部 FrugalList 结构本身是线程安全读取。

---

## 七、事件查找策略

支持 **名称 + OwnerType（含基类）**：

1. **先查 DTypeMap**（若 owner 为 DO 派生）。  
2. **基类链倒查**（回溯直至找到匹配名称）。  
3. **再查 Hashtable**（非 DO 类型）。  

示例：

```csharp
internal static RoutedEvent GetRoutedEventFromName(
    string name,
    Type ownerType,
    bool includeSupers);
```

实现采用 **while loop 回溯基类或 DType.BaseType**，确保拿到最派生的定义。

---

## 八、获取所有事件列表接口

```csharp
internal static RoutedEvent[] GetRoutedEvents();
```
- 返回 **拷贝数组**，防止外部改写原始列表。  
- 去重算法：遍历 *DTypeMap + Hashtable*，用 `Array.IndexOf` 去重并写入结果数组。

---

## 九、性能亮点

| 技术 | 目的 |
|---|---|
| `FrugalObjectList<RoutedEvent>` | 小列表专用 **线性或哈希压缩**实现，避免大量对象。 |
| `DTypeMap` | **稀疏整数映射** + **最小化空间**；只缓存 DO 类型。 |
| **静态只读结构** | 运行期几乎 **只读**，所有路由事件与类级处理器 **静态声明完成** 后可线程并发读取且零锁开销。 |
| **基类继承级联** | 子类一次性合并父类处理器链表，不重复遍历 **基类链** 多次。 |

---

## 十、开发者示例场景

```csharp
public static readonly RoutedEvent LoadedEvent =
    EventManager.RegisterRoutedEvent(
        "Loaded",
        RoutingStrategy.Direct,
        typeof(RoutedEventHandler),
        typeof(FrameworkElement));
```

实际背后调用链路：

```csharp
GlobalEventManager.RegisterRoutedEvent(...)
    -> AddOwner(...)  → DTypeMap[FrameworkElement] 追加事件
    -> 设置 GlobalIndex -> 返回 RoutedEvent
```

类级添加：

```csharp
EventManager.RegisterClassHandler(
    typeof(Button), ButtonBase.ClickEvent, OnButtonClick, true);
```

→ `GlobalEventManager.RegisterClassHandler`  
自动：

1. 找到 `Button → DType`。  
2. 检查基类级处理链。  
3. 合并新 handler 到 **ClassHandlersStore**，并广播给所有派生类更新**子类链表**。

---

## 十一、内部工具链协作

- **DTypeMap** → `DependencyObjectType` 缓存机制共用，确保 Key 唯一且 O(1) 查找。  
- **ClassHandlersStore** 与 **事件路由系统** 紧耦合：构建路由表（Route）时由 `GlobalEventManager` 提供“类级处理器”列表而非再次反射。

---

## 十二、总结

`GlobalEventManager` 用 **500 行不到的代码** 完成了：

- **全局唯一** 的 `RoutedEvent` 注册与索引。  
- **高并发** 的类级处理器注册与继承级联。  
- **极低内存** 的存储模型（Frugal 系列 + DTypeMap）。  

它是 WPF 路由事件体系的**基石组件**，将“事件定义”与“事件处理”完全解耦，既确保反射查找性能，又保障线程安全。理解它，方能深入 WPF 事件模型的所有细节。