# 08 · 重渲染 ≠ 重布局：子节点变大，父容器怎么跟着变

> 问题：改一个 Text 的文本（字号/长度变大），父容器、兄弟节点会跟着"更新"吗？
> 答案：它们**不需要"重渲染"，但可能需要"重新布局"**——这两件事在 RN 里是解耦的：重渲染归 React，布局归 Yoga。
> 相关：[01 渲染链路 ④ Yoga](./01-render-pipeline.md) · [05 两阶段](./05-render-vs-commit-phase.md) · [07 协调剪枝](./07-reconciliation-bailout.md)

> **布局传播的三个方向**（一个孩子变大时）：① 向上 → 父（重算尺寸）；② 横向 → 兄弟（被挤、重定位）；③ 向下 → 自己子树（重测量）。本篇先讲父，再讲兄弟。

---

## 核心：把"父需要更新"拆成四个独立层面

| 层面 | 父需要吗 | 谁负责 | 何时 |
|---|---|---|---|
| 组件重渲染（JS 函数重跑） | ❌ 不需要 | React bailout | render 阶段 |
| ShadowNode 克隆（脊柱） | ✅ 会克隆（廉价结构操作） | Fabric `cloneNodeWithNewChildren` | render / completeWork |
| 布局重算（size/position） | ✅ 若影响到它 | Yoga `markDirtyAndPropagate` 上传 | commit 的 layout pass |
| 原生 view bounds | ✅ 仅当布局真变 | Differentiator → updateLayout | 主线程挂载 |

> **一句话**：父不"重渲染"，但会被"克隆脊柱 + 重新布局"。`size 变大 → 父变大` 是 Yoga 在 C++ 里把脏标记从孩子一路传到父算出来的，跟 React 要不要重跑父组件函数毫无关系。

---

## 第一层：React 重渲染 —— 父 ❌ 不需要

[07 的 bailout](./07-reconciliation-bailout.md) 依然成立：Text 文本变，只有 Text 所在的组件函数重跑，**父 View 的组件函数不会被调用**。

## 第二层：ShadowNode 克隆（不可变树的"脊柱"）—— 父 ✅ 会被克隆，但很轻

ShadowNode 是**不可变**的。要把父里的旧 Text 换成新 Text，父 ShadowNode 必须**克隆**成"指向新 Text"的新对象；爷爷也得克隆指向新父……**一路克隆到 root**（脊柱克隆 / spine clone）。

证据（completeWork host 分支，`ReactFabric-dev.js:10021`）：
```js
? cloneNodeWithNewChildrenAndProps(newProps, _type2)  // 自己 props 也变了
: cloneNodeWithNewChildren(newProps);                 // 只是孩子变了 → 克隆出新父，指向新孩子
```

为什么这**不是"重渲染父"**：
- **旁支子树按引用共享，不克隆**（结构共享 structural sharing）。
- 克隆的只是"根到 Text"这条**脊柱**上的几个节点对象，O(树深度) 的**浅操作**。克隆一个对象 ≠ 跑一遍组件函数。

> 类比：改链表里一个节点，你得新建从它到头的那几个节点，但其余分支照旧引用——不是把整个链表重算一遍。

## 第三层：Yoga 重新布局 —— ✅ 这才是"父变大"

Text 内容变 → 测量尺寸变 → Yoga 把该节点**标脏**并**向上冒泡到祖先**（`yoga/node/Node.cpp:421`）：

```cpp
void Node::markDirtyAndPropagate() {
  if (!isDirty_) {
    setDirty(true);
    if (owner_ != nullptr) {
      owner_->markDirtyAndPropagate();   // ← 把"脏"向上传给父，一路递归到根
    }
  }
}
```

- "孩子变大 → 父重新算大小" 由 Yoga 在 **C++ 布局 pass** 靠**脏标记向上传播**完成，**和 React 的 render bailout 是两套独立机制**。
- Yoga 有**布局缓存**：只重算被标脏的祖先链，未受影响的子树复用上次结果，**不会全树重新布局**。

## 第四层：原生 view —— 父 ✅ 仅当布局真的变了才更新

布局算完，Differentiator diff 新旧 shadow 树：
- 父的 `LayoutMetrics`（x/y/w/h）变了 → 产出 **Update（含 updateLayout）** → 主线程调 `parentView.layout(新bounds)`。
- 父布局没变 → 父**没有任何 mutation**。

---

## 关键的"取决于"：父变不变，看父自己的样式 🎯

| 父的样式 | 孩子（Text）变大时 | 父会怎样 |
|---|---|---|
| **flex / auto / 包裹内容** | 父跟着撑大 | Yoga 重算 → 父布局变 → updateLayout，**原生父 bounds 改变** |
| **固定 width/height** | 父尺寸不动 | 父布局没变 → 父**无 mutation**，Text 可能溢出 / 被 `overflow` 裁剪 |

所以"父是否更新"不取决于"孩子是否变了"，而取决于"**孩子的变化是否影响到父的布局**"——由父的布局规则决定。

---

## 兄弟节点：会被"重定位"，但通常不"重测量"

一个孩子变大，同容器里的兄弟（比如 `column` 布局下后面的节点）会被往下挤、位置变化。它们也会更新吗？

**会，但只是"重定位（便宜）"，不是"重测量（贵）"，更不"重渲染"。**

### 反直觉点：脏标记不"横向"传给兄弟

Yoga 的脏标记**只向上传播**（孩子→父→…→root），**不会直接把兄弟标脏**。兄弟被重定位是**副作用**：
- 变大的 Text 把**父**标脏；
- 父重新跑 flexbox 时，要给**所有孩子**重新分配位置；
- 后面的兄弟被往下推 → `y` 变了 → Yoga 给它打上 `hasNewLayout`。

### 代码：`YogaLayoutableShadowNode::layout`（`:691`）

```cpp
for (auto childYogaNode : yogaNode_.getChildren()) {
  auto& childNode = shadowNodeFromContext(childYogaNode);
  if (childYogaNode->getHasNewLayout()) {        // Yoga 标记"这孩子布局变了"（含仅位置变）
    childNode.ensureUnsealed();                  // 共享节点在此克隆/解封以便写入
    auto newLayoutMetrics = layoutMetricsFromYogaNode(*childYogaNode);
    if (layoutContext.affectedNodes != nullptr)
      layoutContext.affectedNodes->push_back(&childNode);  // 进 affectedNodes → 触发 onLayout
    childNode.setLayoutMetrics(newLayoutMetrics);          // 写入新位置/尺寸
    if (newLayoutMetrics.displayType != DisplayType::None)
      childNode.layout(layoutContext);            // 只对"有新布局"的孩子继续往下
  }
}
```

### 关键：重定位便宜，重测量被缓存跳过

Yoga 有布局缓存（`CalculateLayout.cpp:2292` `canUseCachedMeasurement`）：
- 兄弟自己**内容/尺寸没变** → **复用子树缓存测量**，不重跑它内部 flexbox（重测量被跳过）。
- 但**位置变了** → `getHasNewLayout()` 为真 → `ensureUnsealed()`(克隆) + `setLayoutMetrics()`(写新坐标)，这是 O(1) 便宜操作。

### 兄弟节点的四层（对照下表）

| 层面 | 兄弟需要吗 | 说明 |
|---|---|---|
| 组件重渲染（JS 函数） | ❌ | bailout，兄弟组件函数不跑 |
| 重测量子树（跑 flexbox） | ❌* | *前提：兄弟自己内容/尺寸没变 → 复用缓存 |
| 写 LayoutMetrics（位置） | ✅ 若位置变 | `ensureUnsealed`(克隆) + `setLayoutMetrics` |
| 原生 view bounds | ✅ 若位置变 | Differentiator → updateLayout → `siblingView.layout(新坐标)` |

> 附带：位置变了的兄弟会进 `affectedNodes`（`:725`）→ **它的 `onLayout` 回调触发**。这就是"邻居变大时我也收到 onLayout"的原因。

> 常见现象：长列表里一项变高，下面所有项往下移——它们全被重定位、全发 updateLayout，但都没被重渲染、没被重测量。若这种 reflow 范围大又频繁就会卡，这是**布局抖动（layout thrashing）**，与"无效重渲染"是两类不同的病。

---

## 为什么这个解耦很重要

- **render（React）** 决定"组件输出什么"，**layout（Yoga）** 决定"每个节点多大、在哪"。两者各管一摊：
  - 父样式不变 → 不必重跑父组件函数（省 JS）。
  - 但只要孩子尺寸影响到父 → Yoga 仍会在 C++ 后台线程把父的布局补算出来（保证视觉正确）。
- 布局在 **shadow 后台线程** 算（见 [01](./01-render-pipeline.md) ④），不占主线程；原生侧只对真正变化的 view 发 updateLayout。
- 这也是为什么 RN 性能优化要分清两类问题：**「无效重渲染」（React 层，用 memo/状态下沉解决）** vs **「布局抖动」（Yoga 层，靠减少影响范围、固定尺寸、避免深层 reflow 解决）**。
