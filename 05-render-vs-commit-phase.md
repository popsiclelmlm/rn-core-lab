# 05 · Render 阶段 vs Commit 阶段

> 核心流程：将组件 Render 函数的返回结果转换为对原生视图树的最小变更（增、删、改）的完整生命周期。
> 相关：[03 协调器](./03-reconciler-and-fiber-tree.md) · [04 卸载](./04-mount-and-unmount.md) · [06 Fiber 内核](./06-fiber-internals.md)

---

## 双阶段设计的背景与目的

并发渲染要求协调计算过程能够被中断，或者在必要时被丢弃重来。为了实现这一机制，React 将更新工作拆分为两个阶段：

| 更新阶段 | 中断特性 | 视图树交互 | 行为定义 |
|---|---|---|---|
| **Render 阶段** | 可中断、可丢弃、可重试 | 不与真实视图树交互 | 在内存中计算变更，生成临时更新结构 |
| **Commit 阶段** | 同步执行，不可中断 | 直接修改真实视图树 | 将 Render 阶段计算好的变更一次性应用到视图中 |

**总结**：Render 阶段是可中断且无副作用的，仅在内存中计算状态变更；Commit 阶段则是不可中断的，负责执行真实的副作用并将变更应用到界面上。（关于 Commit 阶段不可中断的原理及其与 Android VSync 帧渲染的关系，详见 [06](./06-fiber-internals.md)）

> ⚠️ 线程模型修正（详细分析请参考 [实验 D](./experiments/D-cpp-lldb-and-layout-thread.md)）：此处将布局计算简单归为“Shadow 后台线程”是一种简化描述。在 Bridgeless 模式下，Commit 阶段与 Yoga 布局计算实际运行在 JavaScript 线程的 RuntimeScheduler 事件循环中，而非 Android UI 主线程，亦非独立的 Shadow 线程；原生挂载操作则在 Android 主线程执行。详细的线程模型请参见 [09](./09-runtime-and-threading.md)。

## 双缓冲机制

React 通过双缓冲技术来保证渲染性能与一致性：
- `current` 树：对应当前屏幕上已渲染并显示的视图树。
- `workInProgress` 树：在后台构建并接收更新的临时树。
- Commit 提交后，指针进行切换：`workInProgress` 树转换为新的 `current` 树（类似于图形学中的双缓冲机制）。
- 每个 Fiber 节点的 `alternate` 属性指向其在另一棵树中对应的镜像节点。

---

## Render 阶段：计算与状态标记

**触发时机**：当组件调用 `setState` 或执行初次渲染时，React 会将对应 Fiber 节点标记为待更新，并由 Scheduler 调度渲染任务。

工作循环（work loop）采用深度优先遍历（算法细节见 [06](./06-fiber-internals.md)），主要交替执行以下两个操作：
- **`beginWork`（向下钻取）**（`ReactFabric-dev.js:9198`）：调用函数组件获取 JSX，执行子节点 Diff 算法，创建或复用子 Fiber 节点，并为发生变化的 Fiber 节点标记更新类型（如 Placement、Update、Deletion 等 Flags）。
- **`completeWork`（向上回溯）**（`ReactFabric-dev.js:9951`）：在此处针对宿主组件调用 `createNode` 构建对应的 **C++ ShadowNode**（`ReactFabric-dev.js:10056`）。

**Render 阶段结束状态**：此时 `workInProgress` 树构建完毕，各节点更新标记（Flags）已就绪，且对应的 C++ ShadowNode 也已创建完成。在此阶段结束前，屏幕显示内容不会发生任何改变（所有的 C++ ShadowNode 均在离屏内存中）。这是“无副作用”设计的体现：在内存中创建可随时丢弃的临时结构，从而支持并发与中断。

---

## Commit 阶段：副作用执行与视图应用

当调用 `commitRoot`（`ReactFabric-dev.js:14325`）时，更新进入不可中断的 Commit 阶段。React 内部主要执行以下三个子阶段：

1. **Before Mutation 阶段**：触发 `getSnapshotBeforeUpdate` 等生命周期方法。
2. **Mutation 阶段**：执行 Render 阶段标记的所有 Flags 更新。对 Fabric 而言，其核心是将构建好的 C++ ShadowTree 根节点提交给 C++ 层：调用 `completeRoot(containerTag, newChildren)`（`ReactFabric-dev.js:15963`）。
3. **Layout 阶段**：触发 `useLayoutEffect`、`componentDidMount`、`componentDidUpdate` 以及 Refs 赋值。

最后，进行双缓冲树的指针切换（`workInProgress` 树变为 `current` 树）。
`useEffect`（被动副作用）会在 Commit 阶段完成后异步执行，这也是它与 `useLayoutEffect`（在 Commit 的 Layout 子阶段同步执行）的本质区别。

---

## 协调提交与 Fabric 提交的双重 commit 机制

更新过程中存在两次关键的提交（Commit）操作：

```
React 协调器提交（JavaScript 侧）
  → completeRoot(containerTag, 提交新 ShadowTree 根节点)
  ─────────────────────────────────────────────
C++ Fabric 渲染器提交（ShadowTree::commit）
  → Differentiator 模块对比新旧树
  → 生成变更指令集（Create/Insert/Update/Remove/Delete）
  → JNI 跨语言调用 → UI 主线程 → SurfaceMountingManager 消费指令并更新视图
```

- **React 协调器的 Commit**：负责在 JavaScript 侧完成 Fiber 树构建后，将新的 C++ ShadowTree 提交给 Fabric。
- **C++ Fabric 的 Commit**：负责对比新旧 ShadowTree 的差异，计算最小变更集，并向主线程发送相应的原生视图操作指令（即 [01](./01-render-pipeline.md) 中的步骤 ⑤-⑧）。

---

## 渲染与提交流程实例分析

以计数器 `count` 从 `0` 变更为 `1` 为例，假设渲染结构中包含条件组件：`{count > 0 && <Badge/>}`。

1. **更新触发**：用户触发事件并调用 `setCount(1)`，React 标记对应 Fiber 节点为脏（Dirty），并安排调度更新任务。
2. **Render 阶段（内存计算，不影响视图）**：
   - `beginWork` 访问目标组件，调用组件函数，获取包含 `<Badge/>` 的新 JSX。
   - 对比子节点：检测到新增 `<Badge/>` 节点，为其标记 `Placement` 标志。
   - 深入遍历 `<Badge/>` 内部宿主组件。
   - `completeWork` 向上回溯：为 `<Badge/>` 的宿主组件调用 `createNode` 在 C++ 层构建 ShadowNode。
   - *此时屏幕上依然没有显示 Badge*。
3. **Commit 阶段（同步执行，不可中断）**：
   - 调用 `completeRoot` 将新构建的 C++ ShadowTree 提交。
   - 触发 Layout 等 Effect，完成双缓冲指针翻转（`workInProgress` 树转换为 `current` 树）。
4. **C++ Fabric 渲染**：
   - `Differentiator` 进行新旧树比对，生成 `Create` 与 `Insert` 变更指令。
   - 指令通过 JNI 下发给 UI 线程，由 `SurfaceMountingManager` 创建并插入原生视图实例。
5. **视图呈现**：下一帧显示出 `<Badge/>` 视图。

当 `count` 从 `1` 变更为 `0` 时，`<Badge/>` 从渲染输出中消失。Render 阶段会为其标记 `Deletion` 标志并执行 Cleanup 清理；在 Commit 后，C++ 侧对比计算得出 `Remove` 与 `Delete` 指令，最终从原生视图树中移除（详见 [04](./04-mount-and-unmount.md)）。

---

## 总结

- **Render 阶段**：使用 `beginWork` 向下遍历（调用组件函数、Diff 比较、标记 Flags），结合 `completeWork` 向上回溯（为宿主组件创建 C++ ShadowNode）。该阶段可中断，且不操作真实视图树。
- **Commit 阶段**：调用 `completeRoot` 提交新树，同步执行 Layout 阶段 Effect 并切换双缓冲指针。该阶段同步且不可中断。
- **C++ Fabric 提交**：通过 `Differentiator` 比对差异生成最小变更集，通过 JNI 将变更指令下发至原生主线程，最终由原生系统执行视图更新。
