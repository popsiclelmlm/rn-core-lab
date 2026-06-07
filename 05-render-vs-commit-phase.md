# 05 · Render 阶段 vs Commit 阶段

> 把「你 render 函数返回了什么」翻译成「对原生树的增删改」的完整时间线。
> 相关：[03 协调器](./03-reconciler-and-fiber-tree.md) · [04 卸载](./04-mount-and-unmount.md) · [06 Fiber 内核](./06-fiber-internals.md)（遍历与可中断的算法细节）

---

## 为什么分两个阶段

并发渲染需要「算到一半能被打断、甚至整个丢弃重来」。所以工作切成两段：

| 阶段 | 能否被打断 | 碰不碰真实界面 | 类比 |
|---|---|---|---|
| **Render（渲染）** | ✅ 可中断、可丢弃、可重来 | ❌ 不碰屏幕，只打草稿 | 草稿纸上反复改方案 |
| **Commit（提交）** | ❌ 一口气做完，不可中断 | ✅ 真正改界面 | 方案定了，一次性施工 |

> 口诀：**Render 可中断、无副作用、只搭草稿；Commit 不可中断、做真正副作用、一次性落地。**
> （为什么 commit 不可中断、它和 Android VSync 帧渲染的关系 → 见 [06](./06-fiber-internals.md)）

---

## 双缓冲：两棵树

- `current` 树 = 现在屏幕上显示的（已提交）
- `workInProgress` 树 = 后台正在搭的草稿
- commit 后**指针一换**：workInProgress 变成新的 current（类比双缓冲渲染 / Android RenderThread）。
- 每个 fiber 的 `alternate` 指向另一棵树里对应的自己。

---

## Render 阶段：搭草稿

触发：`setState`/初次渲染 → 标记 fiber 为脏 → scheduler 排渲染任务。

work loop 一个 fiber 一个 fiber 处理（算法细节见 [06](./06-fiber-internals.md)），两个动作深度优先交替：

- **`beginWork`（往下钻）** `:9198`：函数组件 → **调用组件函数**拿 JSX；diff children、创建/复用子 fiber；给变化的 fiber **打 flag**（Placement 新增 / Update / Deletion）。
- **`completeWork`（往上爬）** `:9951`：host 组件在这里 `createNode` 建 **C++ ShadowNode**（`:10056`）。

**Render 阶段结束**：workInProgress 草稿树搭好、flag 标好、C++ ShadowNode 建好——**但屏幕一点没变**（ShadowNode 是离屏的）。这就是「无副作用」的精髓：建的全是离屏、可丢弃的东西。

---

## Commit 阶段：一次性落地

`commitRoot`（`:14325`）开始，不可中断。React 内部三小段：

1. **before mutation**：`getSnapshotBeforeUpdate` 等。
2. **mutation**：执行 render 阶段攒的所有 flag。对 Fabric 来说核心是把新 ShadowNode 根树交给 C++：
   `completeRoot(containerTag, newChildren)`（`:15963`）。
3. **layout**：`useLayoutEffect`、`componentDidMount/Update`、`ref` 赋值。

完后**翻面**：workInProgress → current。
`useEffect`（被动副作用）在 commit **之后异步**执行——这是它和 `useLayoutEffect` 的时机本质区别。

---

## 注意：其实有「两次 commit」

```
React 的 commit（JS 侧）
  → completeRoot(containerTag, 新ShadowNode树)   ← 把新树交给 C++
  ─────────────────────────────────────────────
C++ Fabric 的 commit（ShadowTree::commit）
  → Differentiator diff 旧树 vs 新树
  → 产出 mutation（Create/Insert/Update/Remove/Delete）
  → JNI → 主线程 → SurfaceMountingManager → 原生 view 增删改
```

- **React 的 commit** 负责「把搭好的新树交出去」。
- **C++ 的 commit** 才负责「diff 出最小变更、真正动原生 view」（即 [01](./01-render-pipeline.md) 的 ⑤⑥⑦⑧）。

---

## 完整例子：count 0→1，render 里有 `{count > 0 && <Badge/>}`

```
onPress → setCount(1) → 标记 fiber 脏 → 排渲染任务

【Render 阶段 —— 搭草稿，屏幕不动】
  beginWork 到组件 → 调用组件函数 → count=1 → 返回 JSX 多了 <Badge/>
  diff children：发现新增 Badge → 打【Placement】flag
  往下钻进 Badge 内部 host 组件
  completeWork 往上爬：给 Badge 内部 host createNode 建 ShadowNode
  ⚠️ 此刻屏幕上还没有 Badge

【Commit 阶段 —— 一次性，不可中断】
  completeRoot 把新 ShadowNode 树交给 C++
  跑 layout effect；翻面 workInProgress→current

【C++ Fabric】
  Differentiator diff → Create + Insert mutation
  → JNI → 主线程 → SurfaceMountingManager → new view + addView

【下一帧】Badge 上屏 ✅
```

反过来 `count 1→0`：`<Badge/>` 从输出消失 → render 阶段打 Deletion、跑 cleanup → commit → C++ diff 出
Remove+Delete → removeView（详见 [04](./04-mount-and-unmount.md)）。

---

## 总口诀

> **Render**：beginWork 往下钻（调用组件、diff、打 flag）+ completeWork 往上爬（host createNode 建离屏 ShadowNode）。可中断、不碰屏幕。
> **Commit**：completeRoot 交树、跑 layout effect、翻面。不可中断。
> 然后 **C++ 再 commit 一次**：Differentiator diff → mutation → JNI → 原生增删改 → 上屏。
