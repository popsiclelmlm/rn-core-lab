# 07 · 协调剪枝：基于 Lanes 的子树跳过（Bailout）机制

> 核心机制：在包含大量节点的视图树中，当触发局部状态更新（如 `setState`）时，React 并非盲目遍历整棵树。本篇阐述 React 如何通过 `lanes` 与 `childLanes` 的更新标记机制实现高效率的协调剪枝（Bailout），仅更新必要的节点路径。
> 相关：[06 Fiber 内核](./06-fiber-internals.md) · [04 挂载卸载](./04-mount-and-unmount.md)

---

## 剪枝机制概述

在触发 `setState` 更新时，React 会从当前更新源节点**向上追溯**，在每个祖先节点的 `childLanes` 中标记子树存在待处理更新；在随后的 Render 阶段，协调算法从根节点（Root）**向下遍历**，若某节点的子树路径上没有更新标记，则该子树会被整体跳过（Bailout），直接返回 `null`。

此机制核心对应以下两部分源码实现：
- `markUpdateLaneFromFiberToRoot`（`ReactFabric-dev.js:5073`）：用于向上标记更新路径。
- `bailoutOnAlreadyFinishedWork`（`ReactFabric-dev.js:9031`，关键判断位于 `:9039`）：用于执行剪枝跳过。

---

## 协调更新流程实例分析

### 示例视图树结构（状态维护在 `Card2`，文本 `"B"` 更新为 `"B2"`）

```
root (HostRoot)
└─ App
   └─ Screen
      ├─ Header
      │  └─ Text "标题"
      └─ List
         ├─ Card1
         │  ├─ Image
         │  └─ Text "A"
         └─ Card2          ← 状态更新源节点，"B" → "B2"
            ├─ Image
            └─ Text "B"
```

### 阶段一：触发更新
在 `Card2` 组件内部调用 `setState`。React 框架需要首先定位并标记包含更新的节点路径。

### 阶段二：向上标记更新路径（`markUpdateLaneFromFiberToRoot`）

```js
sourceFiber.lanes |= lane;                 // 更新源节点：标记自身存在待处理工作
for (parent = sourceFiber.return; null !== parent; ) {
  parent.childLanes |= lane;               // 祖先节点：标记子树中存在待处理工作
  parent = parent.return;
}
```

| 步 | 节点 | 标记状态 |
|---|---|---|
| 1 | `Card2` | 标记 `lanes`（自身待处理） |
| 2 | `List` | 标记 `childLanes`（子树待处理） |
| 3 | `Screen` | 标记 `childLanes`（子树待处理） |
| 4 | `App` | 标记 `childLanes`（子树待处理） |
| 5 | `root` | 标记 `childLanes`（子树待处理） |

标记完成后的视图树状态（`childLanes` 表示子树有更新，`lanes` 表示节点自身有更新，`clean` 表示无待处理更新）：

```
root        [childLanes]
└─ App      [childLanes]
   └─ Screen   [childLanes]
      ├─ Header        [clean]
      │  └─ Text "标题"
      └─ List     [childLanes]
         ├─ Card1      [clean]
         └─ Card2  [lanes]   ← 待更新源节点
```

**更新路径特征**：自根节点到 `Card2` 形成了一条由 `childLanes` 指向的活跃更新路径，而 `Header` 与 `Card1` 分支则保持干净状态。

### 阶段三：Render 阶段的自顶向下节点决策

当遍历访问每个 Fiber 节点时，协调器会评估两个指标：**1. 节点自身是否有待处理任务（`lanes`）？ 2. 该节点的子树中是否有待处理任务（`childLanes`）？**（剪枝逻辑对应源码 `:9039` 的 `0 === (renderLanes & childLanes)` 行）

| 步 | 访问节点 | 自身待处理 (lanes) | 子树待处理 (childLanes) | 遍历决策 | 下一步骤 |
|---|---|---|---|---|---|
| 1 | `root` | 否 | 是 | 继续向下遍历 | 访问 App |
| 2 | `App` | 否 | 是 | 克隆子节点，向下遍历 | 访问 Screen |
| 3 | `Screen` | 否 | 是 | 克隆子节点，向下遍历 | 优先访问 Header |
| 4 | `Header` | 否 | 否 | **执行 Bailout，跳过子树** | 跳过 `Header` 及其子树（不访问 `Text "标题"`） |
| 5 | — | | | 转向兄弟节点 | 访问 List |
| 6 | `List` | 否 | 是 | 克隆子节点，向下遍历 | 优先访问 Card1 |
| 7 | `Card1` | 否 | 否 | **执行 Bailout，跳过子树** | 跳过 `Card1` 及其子树（不访问 `Image` 和 `Text "A"`） |
| 8 | — | | | 转向兄弟节点 | 访问 Card2 |
| 9 | `Card2` | **是** | — | **执行重渲染** | 调用 `Card2()` 组件函数 |

### 阶段四：执行更新源组件的协调逻辑

| 步 | 详细操作 |
|---|---|
| 10 | 协调 `Card2` 子节点：检测 `Image` 属性无变化，直接复用；检测 `Text` 文本发生 `"B" → "B2"` 变化，标记 `Update` 标志 |
| 11 | 向下遍历 `Text` 宿主节点，回溯调用 `completeWork` 以记录文本更新 |
| 12 | 向上回溯至根节点（Root），完成整个 Render 阶段的计算 |

### 阶段五：Commit 提交与原生视图应用

| 步 | 详细操作 |
|---|---|
| 13 | 触发 React Commit 阶段：调用 `completeRoot` 将新构建的 C++ ShadowTree 提交给 C++ 层 |
| 14 | C++ 层的 `Differentiator` 比对新旧树，最终仅输出**一条 `Update` 变更指令**（仅包含文本节点更新） |
| 15 | 指令通过 JNI 下发给 UI 线程，由 `SurfaceMountingManager` 触发对应的原生更新，最终仅更新 **1 个** TextView |

### 性能指标评估

- **遍历节点数**：实际访问的 Fiber 节点数量非常有限（仅 root, App, Screen, Header, List, Card1, Card2 及 Card2 内部节点）。
- **剪枝跳过节点数**：`Header` 子树及 `Card1` 子树被整体跳过，完全不触发其子节点的协调计算。
- **重渲染组件数**：仅调用了 `Card2` 的组件函数。
- **原生视图变更数**：仅修改了 1 个原生视图。

即使在拥有大规模节点的树结构中，只要未受影响的分支保持干净（即没有更新标记），React 的协调剪枝算法即可通过 `return null` 整体过滤该子树，有效避免了全量遍历。

---

## 节点遍历的决策结果分类

在协调过程中，每个 Fiber 节点被访问时，其最终处理路径可分为以下三类：

1. **子树跳过（Bailout）**：当节点本身无待处理工作且其 `childLanes` 干净时，React 整体跳过其子树（直接返回 `null`）。
2. **向下透传（Bailout 但克隆直接子节点）**：当节点自身无待处理工作，但 `childLanes` 存在更新标记时，克隆其直接子节点，继续向下寻找更新路径。
3. **节点重渲染**：当节点自身 `lanes` 包含待处理工作时，执行该组件的渲染函数。

---

## 架构建议：状态管理层次对性能的影响

在实际项目中，重渲染的范围大小直接取决于状态（State）所处的层级位置：

| 状态层级位置 | 渲染结果与性能表现 |
|---|---|
| **就近/底层管理（推荐）** | 状态维护在最靠近目标视图的局部组件中。更新触发时仅重渲染极小的子树，其他大面积分支均触发 Bailout 跳过。性能极佳。 |
| **顶层管理（非推荐）** | 状态维护在顶层组件（例如持有大型列表的父级）。局部数据变更触发顶层重渲染，导致重新评估并映射大量的子元素，引起不必要的 Diff 比较。 |

针对顶层管理中大量子组件的 Diff 开销，可以通过以下优化手段进行缓解：
- **`React.memo` 配合稳定引用**：使子组件的 Props 保持稳定引用，当父组件重渲染时，子组件能够命中 Memo 并实现 Bailout 跳过（此时子组件函数不执行）。
- **未应用 Memo 或引用不稳定**：即使最终原生层不发生修改，JavaScript 侧依然会重新调用所有子组件函数进行计算，从而产生大量多余计算。

> 💡 **补充说明**：Bailout 跳过的是该节点下方的深度子树。而在向下透传时，React 会对当前节点的直接子节点（直接兄弟级）执行浅层的克隆操作（Clone）。因此，“单层平铺上万子节点”是一种极具挑战的扁平设计，而“适当深度的多层树状嵌套”由于能够更好地触发 Bailout，其协调开销往往更小。

---

## 性能优化工程实践

1. **状态下沉与就近原则**：将 `setState` 限制在尽可能底层的组件内，从而缩短更新标记路径，减小重渲染子树的范围。
2. **`React.memo` 与稳定引用（或稳定的 `key`）**：确保父级重渲染时，未受影响的子组件能够命中 Bailout 机制。
3. **组件虚拟化（如 FlatList / VirtualizedList）**：避免在视图层挂载超大规模的节点，仅挂载并渲染当前视口可见的极少数节点（参见 [04](./04-mount-and-unmount.md)）。

> 💡 **架构本质**：因为底层的 C++ Differentiator 机制保证了只有发生实际变化的节点才会修改原生视图，所以性能优化的主要目标在于**最小化 JavaScript 侧的协调与重渲染开销**。
