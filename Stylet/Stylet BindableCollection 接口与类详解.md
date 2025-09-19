# Stylet BindableCollection 接口与类详解

本文详细解读 Stylet 中的可观察集合实现，包括接口与具体类的设计目的、行为特性、线程调度策略以及最佳实践。源码位置见 [Stylet/BindableCollection.cs](Stylet/BindableCollection.cs)。

参考的核心语言构件（可点击直达源码行号）：
- 接口定义
  - [C#.IObservableCollection<T>()](Stylet/BindableCollection.cs:12)
  - [C#.IReadOnlyObservableCollection<out T>()](Stylet/BindableCollection.cs:31)
- 类与成员
  - [C#.BindableCollection<T>()](Stylet/BindableCollection.cs:38)
  - [C#.isNotifying](Stylet/BindableCollection.cs:43)
  - [C#.CollectionChanging](Stylet/BindableCollection.cs:60)
  - [C#.OnPropertyChanged()](Stylet/BindableCollection.cs:66)
  - [C#.OnCollectionChanging()](Stylet/BindableCollection.cs:77)
  - [C#.OnCollectionChanged()](Stylet/BindableCollection.cs:96)
  - [C#.AddRange()](Stylet/BindableCollection.cs:106)
  - [C#.RemoveRange()](Stylet/BindableCollection.cs:132)
  - [C#.Refresh()](Stylet/BindableCollection.cs:157)
  - [C#.InsertItem()](Stylet/BindableCollection.cs:174)
  - [C#.SetItem()](Stylet/BindableCollection.cs:189)
  - [C#.RemoveItem()](Stylet/BindableCollection.cs:203)
  - [C#.ClearItems()](Stylet/BindableCollection.cs:216)
- 相关文件参考： [Stylet/INotifyCollectionChanging.cs](Stylet/INotifyCollectionChanging.cs)

## 1. 概述

.NET 的 ObservableCollection<T> 缺少高效的批量操作（AddRange/RemoveRange）。Stylet 提供的 BindableCollection<T> 在保持 WPF 数据绑定兼容性的同时，新增了批量操作且确保所有集合变更在 UI 线程上执行，避免跨线程 UI 更新异常。此外，它额外暴露了 “将要变化” 事件（CollectionChanging）以实现更细粒度的生命周期钩子和防重入保护。

- 核心目标
  - 提供 AddRange / RemoveRange 批量变更
  - 在批量期间抑制逐项通知，最终以 Reset 触发一次集合变更
  - 确保所有变更在 UI 线程执行
  - 暴露 CollectionChanging 事件（INotifyCollectionChanging 支持）

## 2. 接口

### 2.1 IObservableCollection<T>
定位与用途：在可观察集合上增加批量操作能力。定义为
- [C#.IObservableCollection<T>()](Stylet/BindableCollection.cs:12)
  - [C#.AddRange()](Stylet/BindableCollection.cs:18)
  - [C#.RemoveRange()](Stylet/BindableCollection.cs:24)

继承自 IList<T>、INotifyPropertyChanged、INotifyCollectionChanged，保持与 WPF 绑定系统的兼容性。

### 2.2 IReadOnlyObservableCollection<out T>
定位与用途：只读集合视图，既是 IReadOnlyList<T>，又实现 INotifyCollectionChanged 与 INotifyCollectionChanging（后者在 Stylet 中定义，参考 [Stylet/INotifyCollectionChanging.cs](Stylet/INotifyCollectionChanging.cs)）。
- [C#.IReadOnlyObservableCollection<out T>()](Stylet/BindableCollection.cs:31)

适合只读暴露 BindableCollection 的场景，防止写操作渗透到外部。

## 3. 类 BindableCollection<T>

### 3.1 定义与继承
- [C#.BindableCollection<T>()](Stylet/BindableCollection.cs:38) 继承 ObservableCollection<T>，实现 IObservableCollection<T> 与 IReadOnlyObservableCollection<T>。

扩展点：
- 新增事件 [C#.CollectionChanging](Stylet/BindableCollection.cs:60)
- 新增批量方法 [C#.AddRange()](Stylet/BindableCollection.cs:106)、[C#.RemoveRange()](Stylet/BindableCollection.cs:132)、以及 [C#.Refresh()](Stylet/BindableCollection.cs:157)
- 重写变更通知方法以注入 UI 线程调度与重入保护

### 3.2 关键字段 isNotifying
- [C#.isNotifying](Stylet/BindableCollection.cs:43) 控制通知是否派发
  - 批量操作中置为 false，抑制逐项通知
  - 结束后恢复并派发必要的 Reset/属性变更通知
  - 目的：避免大量 UI 通知导致性能问题与闪烁

### 3.3 构造函数
- 默认构造：初始化为空集合
- 集合构造：从现有 IEnumerable<T> 初始化

对应源码：
- 默认构造/集合构造见 [C#.BindableCollection<T>()](Stylet/BindableCollection.cs:38)

## 4. 事件与通知流程

### 4.1 CollectionChanging 事件
- [C#.CollectionChanging](Stylet/BindableCollection.cs:60) 在集合将要改变时触发
- 内部触发封装于 [C#.OnCollectionChanging()](Stylet/BindableCollection.cs:77)
  - 在 isNotifying=true 时才会派发
  - 使用 BlockReentrancy() 包裹，避免重入事件导致的状态不一致

使用场景：
- 监听即将发生的 Add/Remove/Reset 以做前置校验、取消或清理工作（注意：该接口不提供取消，更多是通知/时序用途）

### 4.2 覆盖系统通知
- [C#.OnPropertyChanged()](Stylet/BindableCollection.cs:66) 与 [C#.OnCollectionChanged()](Stylet/BindableCollection.cs:96) 覆盖父类逻辑，仅在 isNotifying=true 时转发到基类，控制通知时机

## 5. UI 线程调度

所有集合变更均通过 Execute.OnUIThreadSync 包裹执行（源文件中直接调用，避免了跨线程 UI 更新异常）：
- 插入：[C#.InsertItem()](Stylet/BindableCollection.cs:174)
- 替换：[C#.SetItem()](Stylet/BindableCollection.cs:189)
- 删除：[C#.RemoveItem()](Stylet/BindableCollection.cs:203)
- 清空：[C#.ClearItems()](Stylet/BindableCollection.cs:216)
- 批量操作：[C#.AddRange()](Stylet/BindableCollection.cs:106)、[C#.RemoveRange()](Stylet/BindableCollection.cs:132)
- 刷新：[C#.Refresh()](Stylet/BindableCollection.cs:157)

注意：
- Execute.OnUIThreadSync 位于 Stylet 基础设施中（文件 Stylet/Execute.cs），确保在正确的调度器/线程同步上下文上执行 UI 操作。

## 6. 批量操作语义

### 6.1 AddRange
- [C#.AddRange()](Stylet/BindableCollection.cs:106)
- 核心流程：
  1. 触发 CollectionChanging(Reset)（提前通知）
  2. 暂存并关闭通知 isNotifying=false
  3. 使用 base.InsertItem 逐项插入（避免基类逐项通知）
  4. 恢复通知 isNotifying=true
  5. 触发 PropertyChanged("Count") 与 PropertyChanged("Item[]")
  6. 最终发出 CollectionChanged(Reset)（WPF 对范围变更不支持 AddRange/RemoveRange 的复合事件，此处以 Reset 语义统一）

- 设计理由：WPF Binding 引擎对范围操作不识别，采用 Reset 表达 “全量变更”，代价是 UI 通常会重绘整个集合视图，但可显著减少大量单项通知的开销。

### 6.2 RemoveRange
- [C#.RemoveRange()](Stylet/BindableCollection.cs:132)
- 流程与 AddRange 类似：
  1. CollectionChanging(Reset)
  2. isNotifying=false
  3. 遍历 items，定位索引并 base.RemoveItem(index)
  4. 恢复通知
  5. PropertyChanged("Count" / "Item[]")
  6. CollectionChanged(Reset)

### 6.3 Refresh
- [C#.Refresh()](Stylet/BindableCollection.cs:157)
- 用途：通知数据绑定 “集合内容可能整体变化（即使未改动集合元素）”
- 流程：依次引发 PropertyChanged("Count"/"Item[]")、CollectionChanging(Reset)、CollectionChanged(Reset)
- 适用：集合元素内部属性变化，但视图使用了局部缓存或无法感知，需要强制刷新绑定

## 7. 单项变更覆盖（与基类契约一致）

- 插入：[C#.InsertItem()](Stylet/BindableCollection.cs:174)
  - 先触发 CollectionChanging(Add) 再调用 base.InsertItem
- 替换：[C#.SetItem()](Stylet/BindableCollection.cs:189)
  - 先触发 CollectionChanging(Replace) 包含新旧值，再 base.SetItem
- 删除：[C#.RemoveItem()](Stylet/BindableCollection.cs:203)
  - 先触发 CollectionChanging(Remove) 再 base.RemoveItem
- 清空：[C#.ClearItems()](Stylet/BindableCollection.cs:216)
  - 先触发 CollectionChanging(Reset) 再 base.ClearItems

以上操作均在 UI 线程上同步执行，保证与 WPF 视觉树交互安全。

## 8. 线程模型与重入保护

- UI 线程保证：所有修改都经 Execute.OnUIThreadSync 执行，调用方无需关心当前线程是否为 UI 线程
- 重入保护：OnCollectionChanging 中使用 BlockReentrancy()，防止监听器在处理事件时再次修改集合导致 reentrancy（该保护与基类 ObservableCollection 的重入策略一致）

## 9. 与 ObservableCollection<T> 的差异

- 新增：
  - 批量操作 AddRange、RemoveRange
  - 事件 CollectionChanging（INotifyCollectionChanging 扩展）
  - Refresh 一键刷新
- 语义差异：
  - 批量操作以 Reset 通知集合改变（ObservableCollection 不支持范围变更）
  - 单项变更前置触发 “将要变化” 事件
- 线程语义：
  - 强制在 UI 线程执行所有修改（ObservableCollection 不保证调用线程）

## 10. 使用示例

### 10.1 基本增删（单项）
```csharp
var items = new BindableCollection<string>();
items.Add("A");         // 单项添加（内部走 InsertItem）
items.RemoveAt(0);      // 单项删除（内部走 RemoveItem）
```

### 10.2 批量添加与删除
```csharp
var items = new BindableCollection<int>();
items.AddRange(new[] { 1, 2, 3, 4 });  // 触发一次 Reset
items.RemoveRange(new[] { 2, 4 });     // 触发一次 Reset
```

### 10.3 刷新绑定（元素内部变化）
```csharp
// 假设集合元素属性变化但绑定未自动响应
items.Refresh(); // 强制 Count/Item[] 与 Reset，常用于列表需整体重绘的场景
```

### 10.4 订阅 “将要变化” 事件
```csharp
items.CollectionChanging += (s, e) =>
{
    // e.Action: Add / Remove / Replace / Reset
    // 在修改前得到通知，可进行状态清理、日志、快照等
};
```

## 11. 性能与兼容性建议

- 大量变更时优先使用 AddRange/RemoveRange，减少 UI 重绘与通知风暴
- 需要整体刷新或排序/过滤后建议调用 Refresh
- 仅在 UI 线程外触发变更调用时，依赖内部的 Execute.OnUIThreadSync 调度即可，无需额外 Dispatcher 操作
- 暴露只读视图时建议面向 IReadOnlyObservableCollection<T>（防止外部写入）

## 12. 常见问题

- 为什么批量操作使用 Reset 而不是多个 Add/Remove？
  - WPF 的 ObservableCollection 不支持范围通知，多个单项通知会造成巨大开销和 UI 闪烁
- 可以在非 UI 线程调用 Add/Remove/… 吗？
  - 可以。内部会通过 Execute.OnUIThreadSync 切换到 UI 线程安全执行
- CollectionChanging 能取消操作吗？
  - 不能。它仅提供即将发生的通知（对等于 CollectionChanged 的 “之后” 通知）

## 13. 与 INotifyCollectionChanging 的关系

BindableCollection<T> 通过事件 CollectionChanging 提供 “将要变化” 通知，并使 IReadOnlyObservableCollection<T> 承诺该能力，接口文件见 [Stylet/INotifyCollectionChanging.cs](Stylet/INotifyCollectionChanging.cs)。

## 14. 小结

BindableCollection<T> 在 WPF 生态下对集合变更体验做了三个关键增强：
- 批量操作：以 Reset 合理建模大规模变更，避免单项通知风暴
- 线程保障：所有更改统一在 UI 线程执行
- 生命周期钩子：增加 CollectionChanging 实现“之前”通知

配合 Stylet 的 View/Screen/Conductor 体系，可在复杂 MVVM 应用中提供高性能、稳定且可预测的集合更新行为。

附：快捷跳转
- 接口 [C#.IObservableCollection<T>()](Stylet/BindableCollection.cs:12) / [C#.IReadOnlyObservableCollection<out T>()](Stylet/BindableCollection.cs:31)
- 类 [C#.BindableCollection<T>()](Stylet/BindableCollection.cs:38)
- 批量 [C#.AddRange()](Stylet/BindableCollection.cs:106) / [C#.RemoveRange()](Stylet/BindableCollection.cs:132) / [C#.Refresh()](Stylet/BindableCollection.cs:157)
- 单项 [C#.InsertItem()](Stylet/BindableCollection.cs:174) / [C#.SetItem()](Stylet/BindableCollection.cs:189) / [C#.RemoveItem()](Stylet/BindableCollection.cs:203) / [C#.ClearItems()](Stylet/BindableCollection.cs:216)