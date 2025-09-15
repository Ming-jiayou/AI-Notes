# DependencyObject 源码深度阅读笔记  
> 基于 .NET WPF `WindowsBase.dll` 内 `System.Windows.DependencyObject`（逐行通读 3466 行）

---

## 一、定位与职责

`DependencyObject` 是 **WPF 依赖属性系统的“值存储中心”**。  
所有能参与绑定、动画、样式的对象都必须继承它。  
一句话：  
> “把一张『稀疏可变的属性表』压缩成线性的 `_effectiveValues` 数组，并提供线程安全、继承、动画、表达式的实时评估。”

---

## 二、核心静态字段

| 字段 | 用途 |
|---|---|
| `DirectDependencyProperty` | 仅供内部绑定/资源系统建立“直接依赖通知”的假属性（名字 `__Direct`）。 |
| `ExpressionInAlternativeStore` | 占位单例，表示“表达式在其他 Store（如 Style、Template）”。 |
| `DependentListMapField` | 全局 `UncommonField<FrugalMap>`，用于存放「一个 DP 的所有依赖者列表」。 |
| `_getExpressionCore` | `Framework` 注册的回调，用来在「不放入 `_effectiveValues`」时仍能取到 Expression。 |

---

## 三、关键实例字段

| 字段 | 说明 |
|---|---|
| `_dType` | 惰性生成的 `DependencyObjectType`（缓存 CLR 类型 ↔ DType 映射）。 |
| `_effectiveValues` | 按 `GlobalIndex` 升序排序的 **稀疏缓存数组**；每个元素为一个 `EffectiveValueEntry`。 |
| `_packedData` | 一个 32 位位域，打包了： |
| • bit0-9  | `EffectiveValuesCount`    = 当前使用长度 |
| • bit10-18| `InheritableEffectiveValuesCount` |
| • bit19-31| 多种布尔标志：`IsSealed`、`IsSelfInheritanceParent`、`Freezable_Frozen`、动画标志等 |
| `_contextStorage` | 在非冻结场景下保存「从父节点继承值指针」，即 `InheritanceParent`。 |

---

## 四、`EffectiveValueEntry` 与值计算总览

#### 4.1 结构回顾

| 成员 | 作用 |
|---|---|
| `PropertyIndex` | 指向对应 `DependencyProperty.GlobalIndex`。 |
| `BaseValueSourceInternal` | 枚举：Default / Inherited / Local / Style / Template / … |
| `LocalValue` / `ModifiedValue` | 存放「基值」或「经动画、表达式、 coerce 后」的最终值。 |
| 位标志 | IsExpression / IsAnimated / IsCoerced / HasExpressionMarker / IsDeferredReference |

#### 4.2 值优先级链（由低到高）

1. 默认元数据
2. 继承自 `InheritanceParent`
3. 样式、模板触发器
4. 触发器
5. 动态资源
6. **Local**：`SetValue` 传入
7. **Animation**
8. Coerce & CurrentValue

一次有效值的计算 = 沿上述优先级从下到上查找并合并（见 `EvaluateEffectiveValue`, `UpdateEffectiveValue`）。

---

## 五、核心 API 链路

### 5.1 `GetValue(DependencyProperty dp)`

```csharp
public object GetValue(DependencyProperty dp)
{
    VerifyAccess();                                // 线程安全检查
    EntryIndex index = LookupEntry(dp.GlobalIndex); // log₂(n) 二分查表
    return GetValueEntry(index, dp, null, RequestFlags.FullyResolved).Value;
}
```

- `EntryIndex.Found == false` → 走继承 / 默认流程。
- 递归向上查 `inheritanceParent`，避免大量空节点存储内存。

### 5.2 `SetValue(...)`

公共入口：

```csharp
SetValue(dp, value);                        // 普通
SetValue(key, value);                       // 只读 key 版
SetCurrentValue(dp, value);                 // 不改变 ValueSource
```

最终全部收敛至 **`SetValueCommon(...)**：

- **校验线程、只读、`IsSealed`**
- **把 `value == UnsetValue` 重导为 ClearValue**
- **处理 Expression、动画、强制值链**  
- **更新 `_effectiveValues`**

### 5.3 `ClearValue(...)`

与 `SetValue` 共用 `ClearValueCommon`：  
删除对应 `_effectiveValues` 行，并重新计算继承路径或默认值。

---

## 六、索引策略：二分 + 插入排序

为了压缩内存并保持**O(log n)**查找：

```csharp
private EntryIndex LookupEntry(int targetIndex)
{
    if (已找到) return new EntryIndex(iLo);
    else 返回插入点 iLo
}
```

数组始终按 `PropertyIndex` 升序。  
插入新值时通过 `InsertEntry` 把旧元素集体右移 1 位即可。

---

## 七、值的更新与通知链：UpdateEffectiveValue

核心循环（1204+ 行）：

1. **计算新基值**（`EvaluateEffectiveValue`）
2. **重新应用动画/表达式/强制值**（`EvaluateExpression`、`ProcessCoerceValue`）
3. **比对旧/新值并标志是否真正变化**
4. **通知依赖者**
   * 触发 `PropertyChangedCallback`
   * 刷新所有 `DependentList`（绑定、触发器、子属性）
5. **更新 InheritanceContext / 继承子节点**
6. **回收空白槽**（`UnsetEffectiveValue`、`EndPropertyInitialization` 阶段压缩）

---

## 八、继承机制

**InheritanceParent** 字段指向父节点（逻辑树/可视树）。  
只有 `IsInherited==true` 的 DP 会参与。

- 当节点设置过局部值 → 自动调用 `SetIsSelfInheritanceParent()`  
  一次性缓存**所有**可继承值到自身 `_effectiveValues`，此后不再回溯。

- `SynchronizeInheritanceParent` 由元素树变化时调用：  
  处理树移动、逻辑/可视父级切换后的值同步。

---

## 九、并发与线程模型

- **对象级 Dispatcher 校验**：所有公共方法最外层 `VerifyAccess()`  
  保证访问线程与创建线程一致（除冻结后的 `Freezable`）。

- **修改锁**：  
  `CanModifyEffectiveValues` / `DO_Sealed` / `SEALED flag` 在多步调用链内防止重入。

- **DependentList** 使用 `FrugalMap`（稀疏整型数组）减少托管对象数量。

---

## 十、扩展点：如何被子类自定义

| 成员 | 说明 |
|---|---|
| `EvaluateBaseValueCore` | Framework 层重载，加载「样式、模板、触发器」值。 |
| `EvaluateAnimatedValueCore` | Animatable 重载，查询 AnimationStorage。 |
| `OnPropertyChanged` | 公共通知，供子类处理布局、重绘、验证等。 |
| `InheritanceContextChanged` | 事件，用于绑定系统监听父级更替。 |

---

## 十一、性能技巧

1. **有效值缓存稀疏**：不存在的 DP 不占行。
2. **自动压缩**  
   • `InsertEntry` 每次增长系数 `1.2×`(常态) / `2×`(初始化模式)。  
   • `EndPropertyInitialization` 后按 `< 0.8` 利用率裁剪数组。
3. **重用盒子**  
   布尔 / 双精度 / Int32 等常见值用预先创建的 `BooleanBoxes`、`DoubleBoxes` 避免额外分配。
4. **DeferredReference**  
   对于“字典资源”等大型对象实现延迟加载，`SetDeferredValue` 只在真正访问时实例化。

---

## 十二、实际用法小结

```csharp
public class MyControl : DependencyObject
{
    public static readonly DependencyProperty FooProperty =
        DependencyProperty.Register("Foo", typeof(string), typeof(MyControl), new PropertyMetadata("default"));

    public string Foo
    {
        get => (string)GetValue(FooProperty);
        set => SetValue(FooProperty, value);
    }
}

var ctrl = new MyControl();

ctrl.Foo = "hello";          // 向 _effectiveValues 插入 Local 值
ctrl.ClearValue(FooProperty); // 立即删除行，回滚默认值
```

---

## 十三、总结

DependencyObject 在 1 万行不到的源码里构建了 **最密集且可扩展的属性系统**：

- **O(log n)** 的稀疏表 → 支撑百万级元素  
- **继承、动画、强制值、表达式** 统一进入 1 次遍历  
- **无锁、线程绑** → 保证 UI 线程安全的同时最大化吞吐  
- **压缩存储、按需膨胀/释放** → 低内存占用

理解了 `DependencyObject`，才真正跨进了 WPF 世界的大门。