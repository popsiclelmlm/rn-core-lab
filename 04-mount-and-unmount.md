# 04 · 组件的挂载与卸载

> 核心机制：解析三元表达式（如 `null`）、路由跳转（如 Navigation pop）及逻辑短路等操作时，React 框架内部统一的组件卸载行为。
> 相关：[03 Fiber 树](./03-reconciler-and-fiber-tree.md) · [05 两阶段](./05-render-vs-commit-phase.md)
> ⚠️ React Navigation / react-native-screens 是社区库，并非 React Native Core，但它们的延迟加载与卸载机制依然遵循 React 的核心卸载逻辑。

---

## 核心机制：元素消失与组件卸载

若某个组件元素在当前渲染（Render）阶段的输出中不再存在，而在上一次渲染输出中存在，React 将对其 Fiber 子树执行卸载（Unmount）操作。

执行卸载操作时，框架会销毁对应的 Fiber 子树、移除原生视图、清除组件的 State，并触发 `useEffect` 的清理函数（cleanup）或 `componentWillUnmount` 生命周期方法。

### 组件卸载的判定标准

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

## 导航机制：多页面按需构建

在单页应用导航中，页面的 Fiber 子树往往是根据导航状态延迟构建的。

`<Stack.Screen name="Home" component={HomeScreen}/>` 仅作为声明式配置，并不直接实例化 `HomeScreen`。导航器根据当前维护的 **Navigation State** 确定需要渲染的屏幕。只有当屏幕被纳入活跃渲染集合时，其组件函数才会被调用，进而构建对应的 Fiber 子树。

| 导航器类型 | 首次构建时机 |
|---|---|
| **Stack（栈）** | 启动时仅构建初始路由页面；当执行 `push('Detail')` 且 Detail 页面进入 Navigation State 时，才开始构建其 Fiber 子树。 |
| **Tab / Drawer** | 通常包含 `lazy` 属性（如 bottom-tabs 默认开启）；首次聚焦到特定 Tab 时才开始构建其 Fiber 子树。 |

---

## 导航生命周期：Push（隐藏）与 Pop（卸载）

屏幕组件是否在 Fiber 树中挂载，取决于它是否包含在 Navigation State（栈）中，而与其实际可见性无关。

```
push Detail 后:   routes = [Home, Detail]   （两个页面均在状态栈中）
pop  Detail 后:   routes = [Home]           （Detail 页面移出状态栈）
```

| 操作类型 | 状态栈变化 | Fiber 子树状态 |
|---|---|---|
| **Push Detail** | `[Home]` → `[Home, Detail]` | Home 仍在栈内，保持 Mounted 状态（此时处于底层隐藏状态） |
| **Pop Detail** | `[Home, Detail]` → `[Home]` | Detail 移出栈外，执行卸载并销毁 |

在 **Home → Push Detail → Pop 回 Home** 的场景中，Detail 页面将被完全卸载（销毁其 Fiber 子树、State 以及原生 View），此过程通常在退场动画结束后完成。

**验证方式**：若执行 `push → pop → 再次 push Detail`，第二次进入将触发全新的挂载流程（State 重置，Effect 重新执行），可验证先前实例已被销毁。

```jsx
useEffect(() => {
  return () => console.log('Detail UNMOUNTED');  // pop 后会打印 → 证明卸载
}, []);
```

---

## 隐藏与卸载的对比及状态保持

| 实现方式 | 运行结果 | 组件 State |
|---|---|---|
| 三元表达式（`cond ? <X/> : null`） 或 Pop 路由 | **卸载** Fiber 子树 | 丢失 |
| 样式控制（`<X style={{display: cond ? 'flex' : 'none'}}/>`） | 保持 Mounted，仅在视觉上隐藏 | 保留 |
| 容器冻结（如 react-freeze / `<Activity mode="hidden">` / 处于底部的页面） | 保持 Mounted，但暂停其重渲染过程 | 保留 |

在 React Native 中，视觉隐藏的底层实现（`ReactFabric-dev.js:15952`）通常是通过克隆并应用 `display: none` 属性，而不是执行节点删除：

```js
function cloneHiddenInstance(instance) {
  var updatePayload = ...createAttributePayload({ style: { display: "none" } }, ...);
  return { node: cloneNodeWithNewProps(node, updatePayload), ... };
}
```

---

## 架构分层：JavaScript Fiber 树与原生 View 树

`react-native-screens` 优化主要针对原生视图树：即使 React 依然在 JavaScript 内存中保留不可见屏幕的 Fiber 节点，该组件对应的原生 View 也可以从底层的视图树中分离（Detach，如 `activityState=inactive`）以节省原生内存，这并不等同于销毁 JS 层的 Fiber 树。

| 架构层 | 管理者 | 不可见页面的处理方式 |
|---|---|---|
| JS Fiber 树 | React | 保持 Mounted 状态（除非显式配置卸载或冻结） |
| 原生 View 树 | react-native-screens | 视图从层级结构中 Detach，释放原生资源 |

---

## 底层执行逻辑

当 JS 层触发卸载时，C++ Differentiator 模块对比新旧树并输出 **Remove** 与 **Delete** 变更指令（`ShadowViewMutation.h:84` 中定义了 `Remove=8, Delete=2`），这些指令通过 JNI 传递给 Android UI 线程，由 `SurfaceMountingManager.removeViewAt` 将原生视图从父 `ViewGroup` 中移除。

## 总结

元素在渲染输出中的消失是触发组件卸载的唯一条件。无论是条件渲染、类型变更、Key 的更新，还是导航栈的 Pop 操作，其底层的卸载机制完全一致。开发者应根据是否需要保留状态（State），选择完全卸载或局部隐藏。
