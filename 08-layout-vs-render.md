# 08 · 渲染与布局的解耦：以局部尺寸变更为例

> 核心机制：在 React Native 中，组件的“重渲染（Re-render）”与“重新布局（Re-layout）”是相互解耦的两个过程。当某个子节点的尺寸（如 Text 内容）发生变更时，其父容器及兄弟节点通常并不需要重新执行 JavaScript 重渲染，但会在 C++ 层触发重新布局。
> 相关：[01 渲染链路 ④ Yoga](./01-render-pipeline.md) · [05 两阶段](./05-render-vs-commit-phase.md) · [07 协调剪枝](./07-reconciliation-bailout.md)

### 布局传播的三个方向
1. **向上传播**：影响父节点（触发父节点尺寸的重新计算）。
2. **横向传播**：影响同级兄弟节点（触发同级节点的坐标重定位）。
3. **向下拉伸**：影响自身子节点（触发测量算法重算）。

---

## 视图容器更新的四个维度

| 更新维度 | 父节点是否执行 | 负责模块 | 触发时机 |
|---|---|---|---|
| 组件重渲染 (JavaScript 逻辑重跑) | 否 | React Bailout 机制 | Render 阶段 |
| ShadowNode 结构克隆 (脊柱克隆) | 是 | Fabric `cloneNodeWithNewChildren` | Render 阶段 / completeWork |
| 布局计算 (坐标及尺寸计算) | 是 (如果受子节点影响) | Yoga 引擎的 `markDirtyAndPropagate` 标记 | Commit 阶段的 Layout Pass |
| 原生视图更新 (Native View bounds 更新) | 是 (仅当重新计算的布局发生变化时) | Differentiator 驱动 updateLayout | UI 线程挂载 |

子节点尺寸变更导致父容器尺寸变大的过程，完全是由 Yoga 引擎在 C++ 层通过脏标记的向上冒泡机制计算完成的，与 React 是否重新调用父组件的 JavaScript 渲染函数无关。

---

## 第一层：React 协调层的重渲染决策

根据 [07 的 Bailout 剪枝机制](./07-reconciliation-bailout.md)，当文本内容变化时，仅有该 Text 节点所在的组件函数会被重新执行，**父容器组件的渲染函数并不会被调用**。

## 第二层：ShadowNode 树的结构克隆（脊柱克隆）

由于 ShadowNode 采用不可变设计，为了实现将旧 Text 节点替换为新 Text 节点，其对应的父 ShadowNode 必须被克隆为指向新 Text 的新节点对象；以此类推，其上方的所有祖先节点直至根节点（Root）均需要重新克隆（此过程称为脊柱克隆 / Spine Clone）。

源码参考（completeWork 宿主分支，`ReactFabric-dev.js:10021`）：
```js
? cloneNodeWithNewChildrenAndProps(newProps, _type2)  // 自身 Props 发生改变时的克隆
: cloneNodeWithNewChildren(newProps);                 // 仅子节点发生改变时的克隆
```

此操作并不等同于“父组件的重渲染”：
- **旁支节点结构共享**：未受影响的旁支子树直接共享原始引用，不执行克隆。
- **浅层对象替换开销**：克隆仅限于从更新源节点至根节点的链路，耗时为 O(树深度) 的浅层克隆，不会重新执行这些节点对应的 JavaScript 组件函数。

## 第三层：Yoga 引擎的布局计算

当 Text 节点内容变更并导致测量尺寸改变时，Yoga 引擎会将该节点标记为“脏（Dirty）”，并向上递归标记其所有祖先节点（`yoga/node/Node.cpp:421`）：

```cpp
void Node::markDirtyAndPropagate() {
  if (!isDirty_) {
    setDirty(true);
    if (owner_ != nullptr) {
      owner_->markDirtyAndPropagate();   // 将脏标记向上递归传递至 Root 节点
    }
  }
}
```

- **独立性**：父容器的尺寸重新计算是由 Yoga 在 C++ 层的布局计算阶段（Layout Pass）通过脏标记向上冒泡完成的。此逻辑与 React 的 Render Bailout 分属两个独立的处理阶段。
- **局部重布缓存优化**：Yoga 内部维护布局缓存，仅重新计算被标记为脏的祖先链，未受影响的子树则直接复用历史布局，避免了全量布局计算。

## 第四层：原生视图（Native View）的实际更新

布局计算完成后，Differentiator 模块对新旧 ShadowTree 进行对比：
- 若父节点的 `LayoutMetrics`（坐标及尺寸）发生改变，则生成 **Update（含 updateLayout）** 变更指令，由 UI 主线程调用 `parentView.layout(新 bounds)`。
- 若父节点的布局信息未变，则不为父节点生成任何变更指令。

---

## 样式属性对布局更新传播的控制

| 父节点样式配置 | 子节点尺寸增大时 | 父节点的更新行为 |
|---|---|---|
| **自适应包裹 (flex / auto / wrapContent)** | 父容器尺寸随之增大 | Yoga 重算 → 父节点布局变更 → 生成 updateLayout，修改原生视图 bounds |
| **固定宽高 (width/height 为常量)** | 父容器大小保持不变 | 父节点布局未变更 → 不生成 updateLayout，子节点内容可能产生溢出或被裁切 |

因此，父节点是否更新并不取决于子节点本身的变化，而是取决于**子节点的变化是否穿透了父节点的样式约束规则**。

---

## 兄弟节点的布局更新机制

当某个子节点尺寸增大时，同容器下的兄弟节点（如弹性盒布局 `column` 下方的后续节点）会因空间挤压而改变位置。

此时，兄弟节点虽然会被重新定位，但其内部通常不执行“重新测量（Measure）”，更不涉及“JavaScript 重渲染”。

### 兄弟节点的更新边界与计算优化

Yoga 的脏标记在传递时**仅向上冒泡**，并不会横向扩散至兄弟节点。兄弟节点坐标变化的实质是：
- 尺寸增大的节点将**父容器**标脏。
- 父容器在重新执行 Flexbox 布局计算时，重新分配了各子节点的位置坐标。
- 受影响的兄弟节点坐标发生偏移 → 触发 `y` 坐标变更 → Yoga 内部将其标记为 `hasNewLayout`。

### 源码解析：`YogaLayoutableShadowNode::layout`（`:691`）

```cpp
for (auto childYogaNode : yogaNode_.getChildren()) {
  auto& childNode = shadowNodeFromContext(childYogaNode);
  if (childYogaNode->getHasNewLayout()) {        // Yoga 标记该子节点布局（坐标或尺寸）发生变更
    childNode.ensureUnsealed();                  // 浅克隆以解除只读锁定，允许写入更新
    auto newLayoutMetrics = layoutMetricsFromYogaNode(*childYogaNode);
    if (layoutContext.affectedNodes != nullptr)
      layoutContext.affectedNodes->push_back(&childNode);  // 加入 AffectedNodes 列表，后续触发 onLayout
    childNode.setLayoutMetrics(newLayoutMetrics);          // 写入新的相对坐标或尺寸
    if (newLayoutMetrics.displayType != DisplayType::None)
      childNode.layout(layoutContext);            // 仅对有布局更新的子树递归执行 Layout 流程
  }
}
```

### 局部重定位与重测量的复用

Yoga 的缓存机制极大降低了位置偏移的开销（`CalculateLayout.cpp:2292` 中的 `canUseCachedMeasurement`）：
- **复用测量**：若兄弟节点的内容和自身样式未变更，其内部尺寸测量会被直接缓存复用，跳过内部布局计算。
- **便宜更新**：若仅坐标（位置）发生偏移，则仅触发一次 `ensureUnsealed()`（Spine 克隆）与 `setLayoutMetrics()`，这属于 O(1) 的低开销操作。

### 兄弟节点的更新过程对比

| 渲染及布局层次 | 兄弟节点是否需要 | 执行细节 |
|---|---|---|
| 组件重渲染 (JavaScript 逻辑) | 否 | 触发 React Bailout，不执行组件渲染函数 |
| 尺寸重测量 (内部布局计算) | 否 | 兄弟组件内容及自身尺寸未变时，直接复用 Yoga 布局缓存 |
| 布局坐标写入 (LayoutMetrics) | 是 (仅当位置偏移时) | 触发 `ensureUnsealed` 并通过 `setLayoutMetrics` 写入新坐标 |
| 原生视图更新 (Native View bounds) | 是 (仅当位置偏移时) | 经 Differentiator 输出 `updateLayout` 并调用 `siblingView.layout` |

位置改变的兄弟节点会被加入 `affectedNodes` 列表中，这将在随后触发其 JavaScript 层的 `onLayout` 回调函数。因此，当同级节点尺寸变更时，兄弟节点即便没有重渲染，也依然会收到 `onLayout` 事件。

在长列表等复杂场景下，某一项高度变化会导致其下方所有同级视图发生位移。频繁且大范围的位移会产生大量的原生 `updateLayout` 调用。这种情况在图形学中被称为**布局抖动（Layout Thrashing）**。这属于布局引擎层面的性能问题，与 React 侧的“无效重渲染”属于不同层级的瓶颈。

---

## 渲染与布局解耦的架构价值

- **职能分离**：React 的 Render 过程决定“组件树的逻辑结构”，而 Yoga 引擎则决定“各节点的具体尺寸与空间坐标”。
  - 若父节点自身的样式约束没有被破坏，则无需重新执行父组件的 JavaScript 函数。
  - Yoga 在 C++ 异步线程层完成受影响节点的坐标补算，有效确保了排版的精确性。
- **线程优化**：布局计算运行在后台线程（参见 [01](./01-render-pipeline.md) ④），减少了主线程阻塞；在提交流程中，仅对实际发生尺寸或位置变化的原生视图派发更新指令。
- **性能优化导向**：React Native 开发中的优化任务被清晰划分为两个维度：**React 协调层（通过 Memo、状态就近设计解决“无效重渲染”）**与 **Yoga 排版层（通过约束布局范围、固定尺寸、防止大范围 Reflow 解决“布局抖动”）**。
