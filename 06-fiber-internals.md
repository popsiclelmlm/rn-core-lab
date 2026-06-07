# 06 · Fiber 内核：数据结构、迭代遍历、协作式可中断

> 核心设计：本篇介绍 Fiber 节点的数据结构，阐述其如何通过链表结构替代传统的递归树遍历，并实现非阻塞的协作式可中断渲染。
> 相关：[03 协调器](./03-reconciler-and-fiber-tree.md) · [05 两阶段](./05-render-vs-commit-phase.md)

---

## 一、Fiber 节点的数据结构

一个 Fiber 节点本质是普通 JS 对象，关键是身上的指针：

```js
fiber = {
  // ── 三个结构指针（把树变成可遍历的链表）──
  child:   ...,   // 第一个孩子
  sibling: ...,   // 下一个兄弟
  return:  ...,   // 父节点（叫 return 是因为它像函数的"返回地址"）

  // ── 双缓冲指针 ──
  alternate: ..., // 另一棵树里对应的自己（current ↔ workInProgress）

  // ── 身份与数据 ──
  tag:       5,         // 类型：函数组件=0, 宿主组件=5 ...
  type:      'RCTView', // 组件函数 或 host 字符串
  stateNode: {...},     // 真实实例（host 组件这里挂 C++ ShadowNode）
  flags:     2,         // 副作用标记：Placement/Update/Deletion ...
  memoizedState: ...,   // hooks 状态
}
```

### 树结构向链表的转换

通过引入 `child`、`sibling` 和 `return` 指针，传统的树形结构在遍历时可以被抽象为单向链表。这种设计允许遍历过程使用单一指针进行迭代控制，消除了对系统递归调用栈的依赖。

```
        App
         │ child
         ▼
        View
         │ child
         ▼
        Text ──sibling──▶ Image
  (Text.return=View, Image.return=View, Image.sibling=null)
```

---

## 二、遍历算法：workInProgress 树的非递归遍历

### 基于循环的非递归遍历

```js
// ReactFabric-dev.js:13965
function workLoopSync() {
  for (; null !== workInProgress; )
    performUnitOfWork(workInProgress);
}
```

`workInProgress` 是全局指针，用于指向当前正在处理的 Fiber 节点。遍历的即时进度完全由该指针的值决定。

### 深度优先遍历的演进逻辑

```js
// performUnitOfWork :14130
const next = beginWork(fiber);   // 钻取至第一个子节点，若无子节点则返回 null
if (next === null) completeUnitOfWork(fiber);  // 无子节点时，开始向上回溯并执行完成逻辑
else workInProgress = next;                     // 转移至子节点进行深钻
```

```js
// completeUnitOfWork :14257（精简）
let node = fiber;
do {
  completeWork(node);                // 宿主组件在此调用 createNode 创建 C++ ShadowNode
  if (node.sibling !== null) { workInProgress = node.sibling; return; }  // 转移至兄弟节点
  node = node.return;                // 无兄弟节点时，回溯至父节点
  workInProgress = node;
} while (node !== null);
```

### 遍历步骤模拟（以下步骤中包含 C++ 节点的创建）

| 步 | 当前位置 | 做什么 | 下一步 |
|---|---|---|---|
| 1 | App | beginWork → 发现子节点 View | View |
| 2 | View | beginWork → 发现子节点 Text | Text |
| 3 | Text | beginWork 无子节点 → completeWork：创建 C++ 节点并挂载 Text；发现兄弟节点 Image | Image |
| 4 | Image | completeWork：创建 C++ 节点并挂载 Image；无兄弟节点 → 回溯至 View | View |
| 4'| View | completeWork：创建 C++ 节点并挂载 View，同时将 Text/Image 子树关联（`appendAllChildren` `:10073`）；回溯至 App | App |
| 5 | App | completeWork；已无父节点 → 遍历结束 | null |

**自底向上的节点完成顺序**：子节点构建完成后，父节点在 Complete 阶段通过 `appendAllChildren` 将其挂载。这种设计实现了非递归的迭代式深度优先遍历（DFS），避免了函数调用栈的累积。

---

## 三、非阻塞可中断机制的实现原理

### 同步与并发模式的循环对比

```js
// 同步模式：直接遍历完成                                 :13965
function workLoopSync() {
  for (; null !== workInProgress; ) performUnitOfWork(workInProgress);
}
// 并发模式：在遍历过程中通过 shouldYield 评估是否需要释放线程控制权    :14126
function workLoopConcurrentByScheduler() {
  for (; null !== workInProgress && !shouldYield(); ) performUnitOfWork(workInProgress);
}
```

并发模式在每次迭代前均会评估 `shouldYield()` 的状态。

- **任务暂停**：当 `shouldYield()` 返回 `true` 时，循环条件不再满足，工作循环退出并释放主线程控制权。此时，`workInProgress` 指针保留在当前节点位置，即时进度信息得以保留在堆内存中。
- **任务恢复**：再次触发该工作循环时，系统读取全局 `workInProgress` 指针，即可无缝从上次暂停的节点继续向下执行。

### 堆内存现场保留的优势

在 Fiber 架构中，遍历的上下文信息（当前节点、父节点、兄弟节点）全部被保存在堆内存中的 Fiber 节点属性（`child`、`sibling`、`return`）以及全局指针 `workInProgress` 中。因此，中断和恢复仅需要停止或重启循环控制，不需要额外的现场保护与恢复成本。

作为对比，传统的递归遍历（如早期 React 的 Stack Reconciler）：
```js
function reconcile(fiber) {
  for (const child of fiber.children) reconcile(child); // 遍历进度隐式保存在引擎的调用栈中
}
```
其遍历进度完全依赖于 JS 引擎的调用栈帧。一旦函数调用开始，便无法在不销毁调用栈的情况下暂停并让出主线程。

**架构本质**：Fiber 架构将原先依赖系统调用栈管理的隐式递归过程，重构为由开发者在堆内存中显式维护的单向链表遍历过程。这种数据结构层面的重构，是实现可中断渲染的核心基础。

---

## 四、基于协作式时间分片的单线程调度

JavaScript 采用单线程执行模型，并不涉及操作系统层面的线程上下文切换。Fiber 的中断是基于协作式调度（时间分片）实现的：

| 调度属性 | 操作系统抢占式调度 | React Fiber 协作式调度 |
|---|---|---|
| **控制权切换决策** | 由操作系统强制切换，可在任意指令执行处中断 | 由 React 主动决定，仅在 Fiber 节点交界处中断 |
| **执行上下文保存** | 操作系统保存 CPU 寄存器及系统调用栈现场 | 无需额外保存，`workInProgress` 指针即代表当前现场 |
| **线程协作要求** | 执行线程无感知被动中断 | 需要执行单元主动查询 `shouldYield()` 并让出控制权 |

- 协作式中断：在单线程环境下，React 将庞大的渲染任务拆分为细粒度的工作单元（即单个 Fiber 的处理），在两个单元之间主动调用 `shouldYield()` 评估是否需要释放主线程，从而实现与用户交互、动画等高优先级任务的交替执行。
- 中断的时机只能发生在 Fiber 节点处理的交界处。每个 Fiber 节点即为一个独立的工作单元（Unit of Work）。
- `shouldYield()` 用于判断当前时间片（通常为数毫秒）是否已耗尽，或者是否有更高优先级的任务等待处理。在 React Native 中，该调度器由 C++ 层的 `RuntimeScheduler` 实现，核心接口包括 `getShouldYield()` 和 `scheduleTask`。

> 💡 **术语说明**：在操作系统概念中，纤程（Fiber）是指由用户态控制、采用协作式调度的轻量级线程。React 借用此概念，命名其协作式调度的最小执行单元。

---

## 五、与 Android VSync 帧模型的对比分析

Android 原生框架的帧渲染流程（如 `Choreographer.doFrame` 触发的 Measure、Layout 与 Draw）在 UI 主线程上是同步且一次性执行完成的，在此期间不可中断。这与 React 的 Commit 阶段设计一致：

- React 能够被中断的仅是 **Render 阶段**的 Reconciliation 过程（纯 JS 逻辑的计算过程）。
- React 的 **Commit 阶段**（原生视图更新）与 Android 原生的 Layout/Draw 一样，属于不可中断的同步执行过程。

### 为什么 Android 原生框架无需此类中断机制？

1. Android 原生框架的 Measure/Layout/Draw 过程由 C++ 及 Java 高效执行，框架默认其能在 16.6ms（或更短）的帧周期内完成；对于耗时较长的复杂计算，则建议由开发者将其手动迁移至后台工作线程处理。
2. React 的 Reconciliation 阶段运行的是开发者的 JavaScript 代码，可能包含大规模的组件遍历与 Diff 计算。为了避免阻塞主线程交互，React 在框架层实现了自动的切片调度与中断机制。

在 React Native 中，JavaScript 引擎运行在独立的 JS 线程。Reconciliation 阻塞的是 JS 线程（影响对新 State 和手势事件的响应），并不直接阻塞 Android 主线程的滚动和绘制，这与 Web 端的 React DOM（JS 执行与渲染共享主线程）存在本质区别。

---

## 六、并发模式的核心能力：可中断与可丢弃

借助于双缓冲机制中的 `workInProgress` 树和无副作用的 Render 阶段（参见 [05](./05-render-vs-commit-phase.md)），当调度器接收到更高优先级的更新任务时，React 可以选择丢弃当前正在构建的 `workInProgress` 树，重新为高优先级更新构建新树。由于已提交的 `current` 树在内存中保持不变，因此重置计算过程不会带来任何状态副作用，这也解释了为何组件的 Render 函数必须设计为无副作用的纯函数。

---

## 总结

- **遍历算法**：通过 Fiber 节点的 `child`、`sibling` 和 `return` 指针，将传统的递归树遍历转换为基于循环的迭代深度优先搜索（DFS）过程。子节点在 CompleteWork 阶段构建，并自底向上构建整个 C++ ShadowNode 树。
- **可中断特性**：由于更新的中间状态被完全存储在堆内存的全局指针 `workInProgress` 和 Fiber 链表中，使得工作循环的暂停与恢复可以通过简单的循环控制语句完成，开销极低。
- **调度模式**：利用单线程上的协作式时间分片（协作式调度），在执行单元交界处通过主动查询 `shouldYield()` 来决定是否出让控制权，从而实现非阻塞渲染。
- **设计价值**：将基于系统调用栈管理的隐式递归过程改造为由 JavaScript 堆内存管理的显式链表遍历过程，从而在底层同时实现了非递归遍历与协作式可中断渲染。
