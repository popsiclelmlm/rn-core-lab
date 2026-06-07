# 06 · Fiber 内核：数据结构、迭代遍历、协作式可中断

> 全系列的「骨架」。Fiber 这个数据结构的精妙，**同时**解决了「不用递归遍历树」和「能中断」两件事。
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

**核心认知**：加上 `child / sibling / return` 三个指针后，**一棵树就变成了一张能用单个「当前位置指针」逐个走过去的链表**——不再需要递归。

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

## 二、遍历算法：workInProgress → 生成 C++ Node

### 整个遍历就是一个 while 循环

```js
// ReactFabric-dev.js:13965
function workLoopSync() {
  for (; null !== workInProgress; )   // 只要"当前位置"不为空，就一直处理
    performUnitOfWork(workInProgress);
}
```

`workInProgress` 是全局指针，记录「我现在走到哪个 fiber 了」。**整个遍历状态浓缩在这一个指针上**。

### 往下钻 or 往上爬

```js
// performUnitOfWork :14130
const next = beginWork(fiber);   // 返回"第一个孩子"或 null
if (next === null) completeUnitOfWork(fiber);  // 没孩子 → 往上"完成"
else workInProgress = next;                     // 有孩子 → 移到孩子（往下钻）
```

```js
// completeUnitOfWork :14257（精简）
let node = fiber;
do {
  completeWork(node);                // ★ host 组件在这里 createNode 建 C++ ShadowNode
  if (node.sibling !== null) { workInProgress = node.sibling; return; }  // 转兄弟
  node = node.return;                // 没兄弟 → 爬回父节点
  workInProgress = node;
} while (node !== null);
```

### 拿上面那棵树走一遍（★ = 生成 C++ Node）

| 步 | 当前位置 | 做什么 | 下一步 |
|---|---|---|---|
| 1 | App | beginWork → 有孩子 View | View |
| 2 | View | beginWork → 有孩子 Text | Text |
| 3 | Text | beginWork 无孩子 → complete：**★ createNode(Text)**；有兄弟 Image | Image |
| 4 | Image | complete：**★ createNode(Image)**；无兄弟 → 爬回 View | View |
| 4'| View | complete：**★ createNode(View)** + 把 Text/Image 挂上去（`appendAllChildren` `:10073`）；爬回 App | App |
| 5 | App | complete；无父 → 结束 | null |

**生成顺序 Text → Image → View（自底向上）**：父节点 complete 时把已建好的孩子挂上去。这就是迭代式 DFS——**没有递归，调用栈不会变深**。

---

## 三、可中断：精妙就在这里

### 同步 vs 并发，唯一区别一句话

```js
// 同步：跑到底（紧急更新）           :13965
function workLoopSync() {
  for (; null !== workInProgress; ) performUnitOfWork(workInProgress);
}
// 并发：每处理一个就问"该让出了吗"     :14126
function workLoopConcurrentByScheduler() {
  for (; null !== workInProgress && !shouldYield(); ) performUnitOfWork(workInProgress);
}
```

区别只是多了 `&& !shouldYield()`。

- **暂停 = for 循环退出**。`shouldYield()` 为 true → 循环条件不满足 → 函数返回，控制权交还事件循环。`workInProgress` 指针停在原地。
- **恢复 = 再调一次这个循环**。它读全局 `workInProgress`，从上次离开的 fiber 继续。

### 为什么这么轻松？——你问的「数据结构的精妙」

**整个「我走到哪了」的状态，被压成一个 `workInProgress` 指针 + 堆上那棵 fiber 树。** 暂停=不动指针、退出循环；恢复=读指针、继续循环，**零成本**。

对比**递归**：

```js
function reconcile(fiber) {
  for (const child of fiber.children) reconcile(child); // "进度"埋在引擎调用栈 N 层栈帧里
}
```

递归的进度散落在**引擎自己的调用栈**里，没法存下来再返回事件循环，也没法从栈中间恢复。这正是**老版 React（stack reconciler）无法中断**的根本原因。

> **一句话**：React 把「引擎拥有的、藏在调用栈里、动不了的递归进度」改造成「React 自己拥有的、放在堆上一个指针里、随便存取的链表进度」。这个改造就是 Fiber 的灵魂，可中断只是免费副产品。

---

## 四、这不是「线程切换」，是协作式调度

JS 是单线程，**没有也不需要线程切换**。这是**协作式调度（时间分片）**：

| | 抢占式（OS 线程） | 协作式（React Fiber） |
|---|---|---|
| 谁决定切换 | 操作系统，可在任意指令处强行中断 | React 自己，只在"两个 fiber 之间"主动停 |
| 保存现场 | OS 保存 CPU 寄存器、整个栈 | 啥都不用存——`workInProgress` 指针就是现场 |
| 需要配合吗 | 不需要，被动挨打 | 需要，主动问 `shouldYield()` |

- 单线程从未切换；React 只是把自己的大任务**切成片**，干一片**主动让出**给事件循环，稍后回来干下一片。
- 中断点只能在 **fiber 与 fiber 之间**——这就是每个 fiber 叫 **unit of work（工作单元）** 的原因：它是能让出的最小颗粒。
- `shouldYield()` 看「这片时间预算（约几毫秒）用完没」「有没有更高优先级任务在排队」。RN 里调度器是 C++ 的 `RuntimeScheduler`，有 `getShouldYield()`（`RuntimeScheduler_Modern.h:99`）和 `scheduleTask(SchedulerPriority priority, ...)`。

> 🎁 "Fiber" 这个名字本身就来自系统编程里的 **fiber = 用户态协作式轻量线程**（对应内核态、抢占式的 OS thread）。React Fiber = 在用户态用协作式调度跑渲染工作。

---

## 五、与 Android VSync 帧模型的关系（重要澄清）

Android 帧渲染（`Choreographer.doFrame` → measure/layout/draw）在主线程**同步跑、run-to-completion、不可中断**，超时就掉帧。这点和 React 一致：

- React 能被打断的是 **render / reconciliation（算树，纯计算）**；
- React 的 **commit（真正改 view）和 Android 的 traversal 一样不可中断**。

**为什么 React 需要可中断的算树、而 Android 没有这套机制？**

- Android 每帧 measure/layout/draw 是 **native、很快**，框架假设它能塞进一帧；耗时活靠开发者**手动丢后台线程**，框架不切片。
- React 的 reconciliation 跑**任意 JS 用户代码**（成百上千组件函数 + diff），可能很重，所以框架自己**按优先级切片调度**。

RN 特有：**JS 跑在独立 JS 线程**，reconciliation 卡的是 JS 线程（影响响应新 state/touch），不直接卡 Android 主线程的滚动绘制——这跟 React DOM（JS 与渲染同线程）不同。

---

## 六、可中断 + 可丢弃 = Concurrent 的完整能力

因为 workInProgress 是独立草稿树（双缓冲）+ render 无副作用（见 [05](./05-render-vs-commit-phase.md)），高优先级更新插队时，React 可以**直接丢弃搭了一半的草稿树**重搭——current 树没被动过，零风险。这也解释了「render 为什么必须无副作用、可重入」：**它真的可能被丢弃后重跑**。

---

## 总synthesis

> **遍历**：树靠 `child/sibling/return` 三指针变成 while 循环的迭代 DFS——往下 beginWork、往上 completeWork（顺手 createNode），自底向上拼成 ShadowNode 树。
> **可中断**：进度被压成一个 `workInProgress` 指针放在堆上，暂停=退出循环、恢复=再进循环，零成本。
> **不是线程切换**：单线程上的协作式时间分片——主动在 fiber 之间问「该让了吗」。
> **精妙就在数据结构**：把「引擎调用栈里动不了的递归进度」改成「堆上随便存取的链表指针」——这一步同时买下了「迭代遍历」和「可中断」。
