# 04 · 组件的挂载与卸载

> 一条统一定律，解释「三元 null」「Navigation pop」「`&&` 短路」为什么都会卸载组件。
> 相关：[03 Fiber 树](./03-reconciler-and-fiber-tree.md) · [05 两阶段](./05-render-vs-commit-phase.md)
> ⚠️ React Navigation / react-native-screens 是**社区库，不在 core 仓库**，但它们的懒加载完全建立在下面这条 React 核心定律上。

---

## 统一定律

> **某个组件元素「上一次 render 输出里有，这一次没有了」→ React 卸载它的 Fiber 子树。**

卸载会：销毁 Fiber 子树、移除原生 view、丢失组件 state、执行 `useEffect` cleanup / `componentWillUnmount`。

### 不是「null」特殊，而是「元素消失」特殊

下面三种写法效果完全相同，都会卸载 `<X/>`：

```jsx
{cond ? <X/> : null}
{cond && <X/>}            // false 时表达式值是 false
{cond ? <X/> : undefined}
```

关键不是返回了 `null`，而是 `<X/>` **不在这次输出里了**。另外两种也触发卸载：

- **切换成不同类型**：`{cond ? <Detail/> : <Other/>}` — 类型变 → 旧的卸载、新的全新挂载，state 不共享。
- **`key` 变了**：即使类型相同，key 变化也让 React 卸载旧的、挂载新的。

---

## 应用一：Navigation 多页面是否一次性全建？

**不会。页面的 Fiber 子树按导航状态懒构建。**

`<Stack.Screen name="Home" component={HomeScreen}/>` 只是**配置**，不直接渲染 `HomeScreen`。Navigator 读当前
**navigation state**，只渲染当前状态里的屏幕。某屏幕组件函数只有进入「被渲染集合」时才被调用 → 那时才建它的 Fiber 子树。

| 类型 | 首次构建时机 |
|---|---|
| **Stack（栈）** | 启动只建初始路由；`push('Detail')` 时 Detail 才进入 state → 这时才建其 Fiber |
| **Tab / Drawer** | 有 `lazy`（bottom-tabs 默认 `true`）；lazy 下某 tab 第一次聚焦才建 Fiber |

---

## 应用二：push（隐藏） vs pop（卸载）

**决定一个屏幕在不在 Fiber 树里的唯一标准是「它在不在 navigation state（栈）里」，与可不可见无关。**

```
push Detail 后:   routes = [Home, Detail]   ← 两个都在 state
pop  Detail 后:   routes = [Home]           ← Detail 离开 state
```

| 操作 | 栈变化 | 那个屏幕的 Fiber |
|---|---|---|
| **push Detail** | `[Home]` → `[Home, Detail]` | Home **仍在栈 → 保持 mounted**（在下面，隐藏但不卸载） |
| **pop Detail** | `[Home, Detail]` → `[Home]` | Detail **离开栈 → 卸载销毁** |

**场景：Home → push Detail → pop 回 Home**，此时 Detail **会被卸载**（Fiber 子树、state、原生 view 全销毁），
只是通常等**退场动画结束**后才真正卸载。

**自证**：`push → pop → 再 push Detail`，第二次是全新 mount（state 重置、effect 重跑），说明第一次确实被销毁了。

```jsx
useEffect(() => {
  return () => console.log('Detail UNMOUNTED');  // pop 后会打印 → 证明卸载
}, []);
```

---

## 隐藏 ≠ 卸载（state 去留的根因）

| 写法 | 结果 | state |
|---|---|---|
| `{cond ? <X/> : null}` / pop | **卸载** Fiber 子树 | 丢失 |
| `<X style={{display: cond?'flex':'none'}}/>` | 保留 mounted，仅不显示 | 保留 |
| react-freeze / `<Activity mode="hidden">` / push 后下层屏幕 | 保留 mounted，停止重渲染 | 保留 |

源码里「隐藏」的实现（`ReactFabric-dev.js:15952`）走的是 clone 出 `display:none` 的新节点，而非删除：

```js
function cloneHiddenInstance(instance) {
  var updatePayload = ...createAttributePayload({ style: { display: "none" } }, ...);
  return { node: cloneNodeWithNewProps(node, updatePayload), ... };
}
```

---

## 两个层面别混：JS Fiber 树 ≠ 原生 View 树

`react-native-screens` 优化的是**原生侧**：即使 React 让不可见屏幕保持 mounted（Fiber 还在 JS 内存），
它可以把对应**原生 view 从视图层级 detach**（`activityState=inactive`），省原生内存——**但不等于销毁 Fiber**。

| 层面 | 谁管 | 不可见页面 |
|---|---|---|
| JS Fiber 树 | React | 已 mount 的保留（除非配置卸载/冻结） |
| 原生 View 树 | react-native-screens | 可被 detach，省原生内存 |

---

## 闭环到底层

JS 层卸载 → C++ Differentiator diff 出 **Remove + Delete** mutation（`ShadowViewMutation.h:84`：`Remove=8, Delete=2`）
→ JNI → 主线程 `SurfaceMountingManager.removeViewAt` → 原生 view 从父 ViewGroup 移除。

> **一句话**：「元素从这次 render 输出里消失」是 React 唯一的卸载触发条件——三元 null、`&&` 短路、类型变、key 变、Navigation pop 都是它的不同外衣。区别只在于你要「真卸载」还是「假隐藏」。
