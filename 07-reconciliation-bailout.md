# 07 · 协调剪枝：改一个组件，会遍历整棵树吗？

> 性能续集。问题：一个有上万节点的页面，改其中一个 Text 的文本（一次 setState），React 会遍历整棵树吗？
> 答案：**不会。** 靠 `lanes`/`childLanes` 面包屑 + bailout 剪枝，只走必要路径。
> 相关：[06 Fiber 内核](./06-fiber-internals.md)（遍历/可中断）· [04 挂载卸载](./04-mount-and-unmount.md)（虚拟化）

---

## 机制一句话

> setState 时从改动点**向上**给每个祖先埋"子树里有活"的面包屑；render 时从 root **向下**，没面包屑的子树整片 `return null` 跳过。

涉及两段真实代码：
- `markUpdateLaneFromFiberToRoot`（`ReactFabric-dev.js:5073`）——埋面包屑
- `bailoutOnAlreadyFinishedWork`（`:9031`，关键判断在 `:9039`）——剪枝

---

## 一步一步走一遍

### 示例树（state 住在 `Card2`，文本 `"B"` → `"B2"`）

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
         └─ Card2          ← 这里 setState，"B" → "B2"
            ├─ Image
            └─ Text "B"
```

### 阶段一：起点——一次 setState
你在 `Card2` 里调用 `setState`。React 先要标记"哪里有活"。

### 阶段二：向上埋面包屑（`markUpdateLaneFromFiberToRoot` :5073）

```js
sourceFiber.lanes |= lane;                 // 改动点：我自己有活 ●
for (parent = sourceFiber.return; null !== parent; ) {
  parent.childLanes |= lane;               // 每个祖先：我子树里有活 ✓
  parent = parent.return;
}
```

| 步 | 节点 | 标记 |
|---|---|---|
| 1 | `Card2` | `lanes` ●（自己有活） |
| 2 | `List` | `childLanes` ✓ |
| 3 | `Screen` | `childLanes` ✓ |
| 4 | `App` | `childLanes` ✓ |
| 5 | `root` | `childLanes` ✓ |

标记后（✓=子树有活，●=自己有活，空=干净）：

```
root        ✓
└─ App      ✓
   └─ Screen   ✓
      ├─ Header        （空，干净）
      │  └─ Text "标题"
      └─ List     ✓
         ├─ Card1      （空，干净）
         └─ Card2  ●   ← 有活
```

**从 root 到 Card2 形成一条 ✓ 面包屑路径；Header、Card1 是干净旁支。**

### 阶段三：render 从 root 往下，逐个 fiber 决策

每访问一个 fiber 问两件事：**①我自己有活吗（`lanes`）？②子树有活吗（`childLanes`）？**（剪枝判断对应 `:9039` 的 `0 === (renderLanes & childLanes)`）

| 步 | 访问 | 我有活? | 子树有活? | 决策 | 结果 |
|---|---|---|---|---|---|
| 1 | `root` | 否 | ✓ | 下钻 | 去 App |
| 2 | `App` | 否 | ✓ | clone 孩子、下钻 | 去 Screen |
| 3 | `Screen` | 否 | ✓ | clone 孩子、下钻 | 先去 Header |
| 4 | `Header` | 否 | 空 | **🟥 跳过 `return null`** | Header 子树不碰（`Text "标题"` 连访问都没有） |
| 5 | — | | | 转兄弟 | 去 List |
| 6 | `List` | 否 | ✓ | clone 孩子、下钻 | 先去 Card1 |
| 7 | `Card1` | 否 | 空 | **🟥 跳过 `return null`** | Card1 子树不碰（`Image`、`Text "A"` 都没访问） |
| 8 | — | | | 转兄弟 | 去 Card2 |
| 9 | `Card2` | **● 是！** | — | **🟩 真正重渲染**：调用 `Card2()` | 产出新 `Image` + `Text "B2"` |

### 阶段四：进入真正重渲染的 Card2 内部

| 步 | 动作 |
|---|---|
| 10 | 协调 Card2 的孩子：`Image` props 没变 → 无变化；`Text` 文本 `"B"→"B2"` → 打 **Update** flag |
| 11 | 下钻 `Text` host → `completeWork` → 记录文本更新 |
| 12 | 一路 `completeWork` 回到 root，render 阶段结束 |

### 阶段五：commit + 原生

| 步 | 动作 |
|---|---|
| 13 | React commit → `completeRoot` 把新 ShadowNode 树交给 C++ |
| 14 | C++ Differentiator diff → 只产出**一条 `Update` mutation**（Text 文本） |
| 15 | JNI → 主线程 → `SurfaceMountingManager` → 原生只改 **1 个** TextView |

### 数一数

- **真正访问到的 fiber** ≈ 10 个（root, App, Screen, Header, List, Card1, Card2 + Card2 内部几个）。
- **被整片跳过的**：Header 子树、Card1 子树（叶子连访问都没有）。
- **真正重新执行组件函数的**：只有 **Card2**。
- **原生只改 1 个 view**。

即使这棵树有上万叶子，只要它们在干净旁支里，就全被 `return null` 剪掉——**不会遍历整棵树**。

---

## 每个 fiber 被访问时，结局只有三种

1. **🟥 跳过**（`childLanes` 干净）→ 子树整片不动，`return null`。
2. **🟦 下钻但不重渲染**（自己没变、子树有面包屑）→ clone 直接孩子，沿面包屑继续。
3. **🟩 真正重渲染**（自己 `lanes` 有活）→ 调用组件函数。

---

## 关键："重渲染多少"取决于 state 在哪一层

| 场景 | state 位置 | 结果 |
|---|---|---|
| **A（理想）** | 就在 Text 附近的小组件 | 只重渲染那一小块，其余全 bailout。成本极小 |
| **B（常见坑）** | 很高层（顶层数组持有上万项，改其中一项） | 高层组件重渲染 → `.map()` 重产出上万子元素 → React 必须**逐个对比**这上万孩子 |

场景 B 里"对比上万孩子"省不掉，但"是否深入每个孩子"靠优化决定：
- `React.memo` + 列表项**引用稳定**（只有改动项是新引用）→ 其余 memo 命中、bailout（**子组件函数不调用**），只有 1 个真正重渲染。
- 没 memo / 每次 map 新建对象 → 上万子组件函数**全部重跑**（无效渲染，结果却一样）。

> 补充精确点：bailout 跳过的是**子树（深度方向）**；但"下钻"时会 clone 当前节点的**直接孩子**（浅操作）。所以"一层平铺上万兄弟"是 O(N) 浅扫的病态结构，"嵌套深而窄"则极便宜。

---

## 工程结论（性能两大抓手）

1. **状态下沉、就近**：让 setState 发生在尽量低的组件里 → 面包屑路径短、重渲染子树小。
2. **`React.memo` + 稳定引用 / 稳定 `key`**：让高层重渲染时，无关孩子能 bailout。
3. **虚拟化（FlatList / VirtualizedList）**：根本不挂上万节点，只挂可见的约 10~20 个（见 [04](./04-mount-and-unmount.md)）。

> 原生侧无论如何只改实际变化的那几个 view——**成本在 JS 协调，不在原生挂载**。所以优化的战场是"减少 JS 侧的无效重渲染"。
