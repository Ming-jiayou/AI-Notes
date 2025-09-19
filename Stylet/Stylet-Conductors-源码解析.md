# Stylet Conductors 源码解析（Conductor 家族七类）

本文针对以下文件中的类与接口进行系统化解析：[`Stylet/Conductor.cs`](Stylet/Conductor.cs) [`Stylet/ConductorBase.cs`](Stylet/ConductorBase.cs) [`Stylet/ConductorAllActive.cs`](Stylet/ConductorAllActive.cs) [`Stylet/ConductorOneActive.cs`](Stylet/ConductorOneActive.cs) [`Stylet/ConductorNavigating.cs`](Stylet/ConductorNavigating.cs) [`Stylet/ConductorBaseWithActiveItem.cs`](Stylet/ConductorBaseWithActiveItem.cs) [`Stylet/IConductor.cs`](Stylet/IConductor.cs)。

目标是帮助你准确理解 Conductor 家族的职责、生命周期管理、激活/关闭流程以及不同变体（单一活动、多活动、导航栈）的差异与适用场景。

- 适合读者：正在使用或打算扩展 Stylet Conductors 的开发者
- 关注重点：激活/停用/关闭（Activate/Deactivate/Close）语义、一致性约束、资源释放（DisposeChildren）、集合变更的副作用控制、页面导航的历史栈管理

---

## 1. 角色与结构概览

核心接口与基类（职责与关系）：
- 父子关系与活动项
  - IParent：[`IParent<out T>`](Stylet/IConductor.cs) 提供 [`IParent.GetChildren()`](Stylet/IConductor.cs:15)，让外部能遍历父级的子项集合。
  - IHaveActiveItem：[`IHaveActiveItem<T>`](Stylet/IConductor.cs) 揭示“当前活动项” [`IHaveActiveItem.ActiveItem`](Stylet/IConductor.cs:28) 的统一访问。
  - IChildDelegate：[`IChildDelegate`](Stylet/IConductor.cs) 允许子项请求父级关闭自己（[`IChildDelegate.CloseItem(object,bool?)`](Stylet/IConductor.cs:41)）。
  - IConductor：[`IConductor<T>`](Stylet/IConductor.cs) 统一规范父级对子项的生命周期操作：[`IConductor.DisposeChildren`](Stylet/IConductor.cs:54) [`IConductor.ActivateItem(T)`](Stylet/IConductor.cs:60) [`IConductor.DeactivateItem(T)`](Stylet/IConductor.cs:66) [`IConductor.CloseItem(T)`](Stylet/IConductor.cs:72)。

- Conductor 抽象基类
  - [`ConductorBase<T>`](Stylet/ConductorBase.cs) 继承自 Screen，统一实现“子项的准备、关闭检查、按序关闭”等共性逻辑。关键点：
    - 释放策略：[`ConductorBase<T>.DisposeChildren`](Stylet/ConductorBase.cs:20)
    - 子项准备：[`ConductorBase<T>.EnsureItem(T)`](Stylet/ConductorBase.cs:54)
    - 关闭检查：[`ConductorBase<T>.CanCloseItem(T)`](Stylet/ConductorBase.cs:85)、[`ConductorBase<T>.CanAllItemsCloseAsync(IEnumerable<T>)`](Stylet/ConductorBase.cs:67)
    - 子项请求关闭的回调：[`IChildDelegate.CloseItem`](Stylet/ConductorBase.cs:98)

- 单活动基类
  - [`ConductorBaseWithActiveItem<T>`](Stylet/ConductorBaseWithActiveItem.cs) 在 ConductorBase 基础上增加“单一 ActiveItem”的内聚管理：
    - 活动项属性：[`ActiveItem`](Stylet/ConductorBaseWithActiveItem.cs:16)（setter 会转为激活调用）
    - 切换活动项核心：[`ChangeActiveItem(T,bool)`](Stylet/ConductorBaseWithActiveItem.cs:36)
    - 与自身生命周期联动：[`OnActivate()`](Stylet/ConductorBaseWithActiveItem.cs:60) [`OnDeactivate()`](Stylet/ConductorBaseWithActiveItem.cs:68) [`OnClose()`](Stylet/ConductorBaseWithActiveItem.cs:76)

- 具体变体
  - 单子项（无集合）：[`Conductor<T>`](Stylet/Conductor.cs)
  - 多子项·全激活：[`Conductor<T>.Collection.AllActive`](Stylet/ConductorAllActive.cs:19)
  - 多子项·仅一激活：[`Conductor<T>.Collection.OneActive`](Stylet/ConductorOneActive.cs:15)
  - 单子项·导航栈：[`Conductor<T>.StackNavigation`](Stylet/ConductorNavigating.cs:12)

辅助类：
- 屏幕生命周期桥接：[`ScreenExtensions`](Stylet/ScreenExtensions.cs)（用于 TryActivate / TryDeactivate 调用，文件已在仓库中）

---

## 2. 统一的生命周期与关闭语义

- Activate / Deactivate / Close 三元语义
  - Activate：将 Screen 置为活动状态，通常对应“显示/前台”。
  - Deactivate：将 Screen 置为非活动状态，保持已创建但不在前台。
  - Close：彻底移除（可能伴随 Dispose），对应资源释放与从父集合移除。
- 关闭守卫（Guard Close）
  - 若子项实现 IGuardClose，则会通过 [`ConductorBase<T>.CanCloseItem(T)`](Stylet/ConductorBase.cs:85) 异步询问是否可关闭。
  - 多子项关闭采用顺序检查，避免并发弹窗：[`CanAllItemsCloseAsync`](Stylet/ConductorBase.cs:67)。
- 父子关联
  - 确保子项的 Parent 指向当前 Conductor：[`EnsureItem`](Stylet/ConductorBase.cs:54)。
- 与自身 Screen 生命周期联动
  - 典型联动在单活动基类中：[`ConductorBaseWithActiveItem.OnActivate`](Stylet/ConductorBaseWithActiveItem.cs:60) 会激活当前 ActiveItem，Deactivate/Close 同理。

---

## 3. ConductorBase<T>：共性能力沉淀

关键成员与要点：
- 释放策略开关
  - [`DisposeChildren`](Stylet/ConductorBase.cs:20)：关闭子项时是否调用 Dispose，默认 true，可被派生类覆盖。
- 子项准备
  - [`EnsureItem(T)`](Stylet/ConductorBase.cs:54)：
    - 断言非空
    - 若子项实现 IChild 且 Parent 非自身，则修正 Parent = this
- 关闭检查
  - 单项：[`CanCloseItem(T)`](Stylet/ConductorBase.cs:85)（IGuardClose 优先）
  - 多项顺序：[`CanAllItemsCloseAsync(IEnumerable<T>)`](Stylet/ConductorBase.cs:67)（逐个 await，防止并发 UI 干扰）
- 子项请求关闭入口
  - [`IChildDelegate.CloseItem(object,bool?)`](Stylet/ConductorBase.cs:98)：子项可通过 IChildDelegate 反向请求父级关闭自身
- 抽象契约
  - [`GetChildren()`](Stylet/ConductorBase.cs:30) [`ActivateItem(T)`](Stylet/ConductorBase.cs:36) [`DeactivateItem(T)`](Stylet/ConductorBase.cs:42) [`CloseItem(T)`](Stylet/ConductorBase.cs:48)

实践建议：
- 若需要自定义“子项入场策略”（例如依赖注入、预加载数据），覆盖 [`EnsureItem`](Stylet/ConductorBase.cs:54) 是最佳位置。
- 对多子项关闭做整体验证时，应优先复用 [`CanAllItemsCloseAsync`](Stylet/ConductorBase.cs:67)。

---

## 4. ConductorBaseWithActiveItem<T>：单活动核心

单活动场景核心实现，内聚“活动项切换”逻辑。

- ActiveItem 属性拦截
  - [`ActiveItem`](Stylet/ConductorBaseWithActiveItem.cs:16) 的 setter 直接转发到 [`ActivateItem(T)`](Stylet/IConductor.cs:60) 接口，屏蔽外部对字段的直接赋值副作用。
- 活动项切换算法
  - [`ChangeActiveItem(T newItem, bool closePrevious)`](Stylet/ConductorBaseWithActiveItem.cs:36)：
    1. TryDeactivate 旧 ActiveItem
    2. 若 closePrevious 为 true：关闭并清理旧 ActiveItem（受 [`DisposeChildren`](Stylet/ConductorBase.cs:20) 影响）
    3. 置新 ActiveItem 字段
    4. 若 newItem 非空：先 [`EnsureItem`](Stylet/ConductorBase.cs:54) 绑定 Parent，再依据自身 IsActive 状态 TryActivate/TryDeactivate 新项
    5. Raise PropertyChanged(ActiveItem)
- 与自身生命周期联动
  - 激活时：[`OnActivate`](Stylet/ConductorBaseWithActiveItem.cs:60) → 激活 ActiveItem
  - 停用时：[`OnDeactivate`](Stylet/ConductorBaseWithActiveItem.cs:68) → 停用 ActiveItem
  - 关闭时：[`OnClose`](Stylet/ConductorBaseWithActiveItem.cs:76) → 关闭并清理 ActiveItem

常见扩展点：
- 可在派生类暴露“是否关闭旧项”的策略参数，再调用 [`ChangeActiveItem`](Stylet/ConductorBaseWithActiveItem.cs:36)。

---

## 5. Conductor<T>（单子项，无集合）

适用于仅在任一时刻持有至多一个子项的场景。核心逻辑在：
- 激活：[`Conductor<T>.ActivateItem(T)`](Stylet/Conductor.cs:15)
  - 若 item == ActiveItem 且自身已激活 → TryActivate(ActiveItem)
  - 否则先判断旧项是否可关（[`CanCloseItem`](Stylet/ConductorBase.cs:85)），可关则 [`ChangeActiveItem(item, true)`](Stylet/ConductorBaseWithActiveItem.cs:36)
- 停用：[`Conductor<T>.DeactivateItem(T)`](Stylet/Conductor.cs:34)
  - 仅当 item == ActiveItem 时才 TryDeactivate
- 关闭子项：[`Conductor<T>.CloseItem(T)`](Stylet/Conductor.cs:44)
  - 若 item == ActiveItem 且可关 → [`ChangeActiveItem(default, true)`](Stylet/ConductorBaseWithActiveItem.cs:36)（移除当前活动项）
- 自身可关闭判断：[`CanCloseAsync()`](Stylet/Conductor.cs:57)
  - 先兼容旧的 `CanClose`，再检查当前 ActiveItem 是否可关

使用建议：
- 典型于“嵌套详情页”的宿主：每次切换详情都关闭旧详情，保证上下文清爽。
- 若切换频繁且希望缓存旧项，可考虑 OneActive 或 StackNavigation。

---

## 6. Collection.AllActive（多子项，全部激活）

类定义：[`Conductor<T>.Collection.AllActive`](Stylet/ConductorAllActive.cs:19)

特征：
- 内部集合：`BindableCollection<T> items`，对外暴露 [`Items`](Stylet/ConductorAllActive.cs:28)。
- 集合事件双阶段处理：
  - Changing 捕捉 Reset 前快照（[`itemsBeforeReset`](Stylet/ConductorAllActive.cs:23)）
  - Changed 中按 Action 分派：
    - Add：[`ActivateAndSetParent`](Stylet/ConductorAllActive.cs:75)（批量 SetParent 并根据宿主 IsActive 决定 TryActivate/TryDeactivate）
    - Remove：[`CloseAndCleanUp`](Stylet/ConductorAllActive.cs:109)（按当前 DisposeChildren 策略）
    - Replace：先激活新，再清理旧
    - Reset：以差集计算新增与移除，分别激活/清理
- 宿主生命周期联动：
  - Activated：对集合中全部 IScreenState 执行 Activate（[`OnActivate`](Stylet/ConductorAllActive.cs:83)）
  - Deactivated：对全部执行 Deactivate（[`OnDeactivate`](Stylet/ConductorAllActive.cs:96)）
  - Closed：对全部执行 CloseAndCleanUp 并清空集合（[`OnClose`](Stylet/ConductorAllActive.cs:109)）
- 单项操作：
  - [`ActivateItem(T)`](Stylet/ConductorAllActive.cs:140)：EnsureItem 后，依据宿主 IsActive 决定 TryActivate/TryDeactivate（AllActive 语义：加入集合即受宿主活跃度牵引）
  - [`DeactivateItem(T)`](Stylet/ConductorAllActive.cs:157)：直接 TryDeactivate，不移除
  - [`CloseItem(T)`](Stylet/ConductorAllActive.cs:166)：可关闭则 CloseAndCleanUp 并从集合移除
- 关闭检查：[`CanCloseAsync`](Stylet/ConductorAllActive.cs:126) → 顺序检查集合中的所有项
- EnsureItem 覆盖：若集合未包含该项则先 Add 再委派基类（[`EnsureItem`](Stylet/ConductorAllActive.cs:191)）

适用场景：
- 同屏多模块同时活动，例如“停靠式面板、工具条、多文档同时活动但各自自管理激活”。

注意事项：
- 集合事件可能引发 re-entrancy，源码通过复制快照（ToList）规避遍历时修改集合的副作用。

---

## 7. Collection.OneActive（多子项，仅一激活）

类定义：[`Conductor<T>.Collection.OneActive`](Stylet/ConductorOneActive.cs:15)

特征：
- 内部集合：`BindableCollection<T> items`，对外暴露 [`Items`](Stylet/ConductorOneActive.cs:24)。
- 集合事件处理重点在“活动项可能被移除”的一致性修正：
  - Add：[`SetParentAndSetActive(newItems,false)`](Stylet/ConductorOneActive.cs:46)（设置 Parent，不立即激活新项）
  - Remove / Replace / Reset：先调用 [`ActiveItemMayHaveBeenRemovedFromItems()`](Stylet/ConductorOneActive.cs:79) 以确保 ActiveItem 仍合法，再按需要清理旧项与设置新项
- 活动项一致性修正：
  - [`ActiveItemMayHaveBeenRemovedFromItems`](Stylet/ConductorOneActive.cs:79)：
    - 若 ActiveItem 不在集合中，选择一个“合理”的新 ActiveItem（[`DetermineNextItemToActivate`](Stylet/ConductorOneActive.cs:161)）并调用 [`ChangeActiveItem(next, items.Contains(oldActive))`](Stylet/ConductorBaseWithActiveItem.cs:36)
- 单项操作：
  - 激活：[`ActivateItem(T)`](Stylet/ConductorOneActive.cs:102)
    - 若 item == ActiveItem 且宿主已激活 → TryActivate
    - 否则直接 [`ChangeActiveItem(item,false)`](Stylet/ConductorBaseWithActiveItem.cs:36)（不关闭旧项，由更高层策略决定）
  - 停用：[`DeactivateItem(T)`](Stylet/ConductorOneActive.cs:119)
    - 若停用的是当前 ActiveItem → 选出下一个项并切换为活动
    - 否则仅 TryDeactivate(item)
  - 关闭：[`CloseItem(T)`](Stylet/ConductorOneActive.cs:139)
    - 若是活动项：先选出下一项并切换，再从集合移除（避免双重关闭）
    - 非活动项：直接从集合移除（移除行为会触发清理）
- 选择下一个活动项：
  - [`DetermineNextItemToActivate`](Stylet/ConductorOneActive.cs:161)：优先选择“当前项之前一位”，否则取集合第一个，集合不足则 default
- 自身关闭：
  - 清空集合即触发所有子项关闭（[`OnClose`](Stylet/ConductorOneActive.cs:202)）

适用场景：
- 典型 Tab 场景：任一时刻仅一个页签处于活动，切换时不必总是关闭旧页签（保留状态提高体验）。

注意事项：
- 源码中特别避免“先 Close 再 Deactivate 导致的反激活”问题，因此在 Replace/Remove/Reset 的处理顺序上极为谨慎。

---

## 8. StackNavigation（单子项 + 历史栈导航）

类定义：[`Conductor<T>.StackNavigation`](Stylet/ConductorNavigating.cs:12)

特征：
- 维护“历史列表”（后进先出）：`List<T> history`
- 激活：[`ActivateItem(T)`](Stylet/ConductorNavigating.cs:21)
  - 若激活项与当前相同且宿主已激活 → TryActivate
  - 否则若当前存在 ActiveItem → 推入 history，随后 [`ChangeActiveItem(item,false)`](Stylet/ConductorBaseWithActiveItem.cs:36)
- 停用：[`DeactivateItem(T)`](Stylet/ConductorNavigating.cs:40) 直接 TryDeactivate
- 回退入口：
  - [`GoBack()`](Stylet/ConductorNavigating.cs:48) 内部调用 `CloseItem(ActiveItem)`
- 清空历史：
  - [`Clear()`](Stylet/ConductorNavigating.cs:56)：关闭并清理 history 中所有项后清空列表
- 关闭子项：[`CloseItem(T)`](Stylet/ConductorNavigating.cs:69)
  - 若关闭的是 ActiveItem：
    - 从 history 取出最后一项作为新的 ActiveItem（如有），再 [`ChangeActiveItem(newItem,true)`](Stylet/ConductorBaseWithActiveItem.cs:36)
  - 若关闭的是历史中的某项：仅 CloseAndCleanUp 并自 history 移除
- 自身关闭：[`OnClose`](Stylet/ConductorNavigating.cs:108) 先关闭 history，最后关闭 ActiveItem
- 可关闭检查：[`CanCloseAsync`](Stylet/ConductorNavigating.cs:95) 同时覆盖 ActiveItem + history 全部项

适用场景：
- 向导/页面前进后退：支持“后退激活上一页”的用户体验。
- 对比 OneActive：StackNavigation 额外维护历史回退轨迹。

---

## 9. Conductor 家族差异与选择建议

- 仅单项且切换即关闭旧项
  - 使用 [`Conductor<T>`](Stylet/Conductor.cs)：简洁直接，内存与状态清爽。
- 多项但全部同时活动（例如并列模块）
  - 使用 [`Collection.AllActive`](Stylet/ConductorAllActive.cs:19)：集合变更即自动跟随宿主活跃度。
- 多项且仅一个活动（Tab 典型）
  - 使用 [`Collection.OneActive`](Stylet/ConductorOneActive.cs:15)：切换活跃项但可保留非活动项状态。
- 单项 + 可回退
  - 使用 [`StackNavigation`](Stylet/ConductorNavigating.cs:12)：历史轨迹可精细控制（`GoBack`/`Clear`）。

通用建议：
- 当子项含资源（文件句柄/网络连接）且不再需要时，保持 [`DisposeChildren=true`](Stylet/ConductorBase.cs:20)。
- 若需要缓存，考虑关闭策略或选择 OneActive/StackNavigation 的非立即关闭流派。

---

## 10. 关键实现细节与坑点提示

- 关闭顺序与异步对话
  - 多项关闭必须顺序 await（[`CanAllItemsCloseAsync`](Stylet/ConductorBase.cs:67)），避免同时弹出多个“确认关闭”对话。
- 集合变更的再入（re-entrancy）
  - AllActive/OneActive 处理集合事件时均通过复制快照（ToList/Except）来避免遍历期间的集合修改。
- 避免“双重关闭”
  - OneActive 在关闭活动项时，先切换 ActiveItem 再从集合移除，并且不重复调用 CloseAndCleanUp（[`CloseItem`](Stylet/ConductorOneActive.cs:139) 内注释有特别说明）。
- Parent 绑定一致性
  - 子项进入集合或成为活动项前，确保 Parent 正确（[`EnsureItem`](Stylet/ConductorBase.cs:54)），避免子 → 父回调失败。
- 激活与宿主状态一致性
  - 新项加入时是否立即激活取决于宿主的 IsActive（AllActive）或策略（OneActive 的 `SetParentAndSetActive(..., false)`）。
- TryActivate/TryDeactivate 的容错
  - 通过 [`ScreenExtensions`](Stylet/ScreenExtensions.cs) 调用，避免直接依赖子项具体实现，增强健壮性。

---

## 11. 扩展与自定义建议

- 自定义子项入场策略
  - 覆盖 [`EnsureItem`](Stylet/ConductorBase.cs:54) 注入依赖、建立事件订阅或预加载数据。
- 自定义活动项切换策略
  - 基于 [`ChangeActiveItem`](Stylet/ConductorBaseWithActiveItem.cs:36) 包装“是否关闭旧项”的策略开关（如脏数据提示时保留旧项）。
- 自定义“下一个活动项”选择
  - 在 OneActive 覆盖 [`DetermineNextItemToActivate`](Stylet/ConductorOneActive.cs:161) 实现“就近/最近使用/固定顺序”等策略。
- 导航栈增强
  - 在 StackNavigation 上层组合 Command（GoBack/Forward）、限制历史深度、或对不同页面类型施加差异化的回退策略。

---

## 12. 参考调用路径速查

- 切换活动项（单活动基类）：
  - 入口：[`ActiveItem.set`](Stylet/ConductorBaseWithActiveItem.cs:16) → 接口 [`ActivateItem(T)`](Stylet/IConductor.cs:60)
  - 实现：派生类的 `ActivateItem` 根据情况调用 [`ChangeActiveItem`](Stylet/ConductorBaseWithActiveItem.cs:36)
- 关闭请求（子项触发）：
  - 子项经由 [`IChildDelegate.CloseItem`](Stylet/IConductor.cs:41) → 基类显式实现 [`IChildDelegate.CloseItem`](Stylet/ConductorBase.cs:98) → 调用派生类的 [`CloseItem(T)`](Stylet/IConductor.cs:72)

---

## 13. 结语

Conductor 家族通过“接口统一 + 基类沉淀 + 变体定制”，为常见 MVVM 多屏生命周期管理提供了稳健且可扩展的骨架。理解其关闭检查的顺序语义、集合变更的副作用控制以及活动项切换的约束，是安全扩展与定制的关键。

当你的业务需要在“内存占用、用户体验、状态保留”之间平衡时，可以结合本解析选择最合适的 Conductor 变体，或在基类提供的可扩展点上实现定制策略。