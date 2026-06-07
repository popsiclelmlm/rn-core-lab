# 03 · React 协调器与 Fiber 树

> 两个问题：(A) 协调器怎么区分「复合组件 View」和「宿主组件 RCTView」？(B) Fiber 树何时、由谁触发构建？
> 相关：[02 宿主组件](./02-host-component-and-viewconfig.md) · [05 两阶段](./05-render-vs-commit-phase.md) · [06 Fiber 内核](./06-fiber-internals.md)

---

## A. 复合组件 vs 宿主组件

协调器有两类节点，处理方式完全不同：

| Fiber 类型 | type 是什么 | React 怎么处理 |
|---|---|---|
| **复合组件**（如 `View`） | 函数 | **调用这个函数**，拿返回值，继续往下协调 |
| **宿主组件**（如 `'RCTView'`） | 字符串 | **不调用任何东西**，直接建 host fiber，造原生节点 |

```
React 调用 View(props)            ← 函数组件，被"调用"
   ↓ 返回
<'RCTView' {...props}/>           ← type 是字符串 = ViewNativeComponent
   ↓
React 看到 type 是字符串
    → 不再调用任何 JS 组件函数
    → 建一个 HostComponent fiber
    → 造原生节点 createNode(...)
```

在 React 协调中，字符串类型的 `type` 是停止向下深钻（调用组件函数）的信号，因为字符串代表宿主组件，底层无须且无法作为函数进行调用。

### 真实代码：host fiber 在 completeWork 里造节点

`ReactFabric-dev.js:10045`（`case 5` = HostComponent 分支）：

```js
_type2 = getViewConfigForType(_type2);                 // ① 用 'RCTView' 查 ViewConfig（[02] 的懒回调在此触发）
for (keepChildren in _type2.validAttributes) ...        // ② 用 schema 过滤 props
keepChildren = ReactNativePrivateInterface
    .createAttributePayload(newProps, _type2.validAttributes);
current = {
  node: createNode(                                    // ③ JSI 进 C++！
    renderLanes,                  // reactTag
    _type2.uiViewClassName,       // "RCTView"
    current.containerTag,         // surfaceId
    keepChildren,                 // 过滤后的 props
    workInProgress
  ), ...
};
```

文本节点同理：`createTextInstance` → `createNode(tag, "RCTRawText", ...)`（`:15902`）。

### 流程终止的边界与区别

1. 停止调用组件函数仅针对当前分支。如果 `<View>` 包含子节点（如 `<View><Text/></View>`），在创建 `RCTView` 的 HostFiber 后，React 会继续对子节点进行协调，如果子节点 `Text` 是复合组件，则继续调用 `Text(...)`。
2. 构建 HostFiber 与创建原生 View 分属不同阶段。Render 阶段的 `createNode` 用于构建 C++ 层的 ShadowNode，而原生 `android.view.View` 的创建发生在 Commit 阶段，通过 JNI 在主线程执行（参见 [01](./01-render-pipeline.md) 中的步骤 ⑧）。

### 完整 Fiber 链（`<View><Text>hi</Text></View>`）

```
View   (函数组件, 被调用)
 └─ RCTView   (host fiber, type='RCTView', createNode→C++)
     └─ Text   (函数组件, 被调用)
         └─ RCTText   (host fiber, createNode→C++)
             └─ "hi"   (host text fiber, createTextInstance→createNode "RCTRawText")
```

---

## B. Fiber 树何时开始构建？

**触发时机**：Fiber 树的构建始于 Surface 的首次渲染（Render），并非发生在进程启动或执行 `AppRegistry.registerComponent` 时。

```
App 启动，加载 JS bundle
  → AppRegistry.registerComponent('AppName', () => App)
  ─────────────────────────────────────────────────────
  Native 创建一个 surface / root view（Android: ReactSurface / ReactRootView）
  → 回调 JS: AppRegistry.runApplication('AppName', params)
  → renderApplication(...)                                   (renderApplication.js)
  → Renderer.renderElement({element, rootTag})              (renderApplication.js:88)
  → ReactFabric.render(element, rootTag, null, true, ...)   (RendererImplementation.js:28)
  → 首次为该 rootTag 创建 FiberRoot（根容器 + HostRoot fiber）
  → schedule 初始渲染 → work loop 自顶向下调用组件函数 → 构建 workInProgress Fiber 树
```

### 核心要点

1. **`registerComponent` 仅作注册**：该方法仅将 `() => App` 存入注册表，此时不创建任何 Fiber 节点。
2. **`runApplication` 触发构建**：当原生侧创建 Surface 完毕后，会回调 `runApplication`，此时才真正开始构建 Fiber 树。
3. **独立的 Fiber 树**：每一个 Surface 均拥有独立的 Fiber 树（即独立的 `FiberRoot`），多个 Activity 或 Root View 之间的渲染上下文互不干扰。
4. **双缓冲机制**：React 维护 `current`（已提交）与 `workInProgress`（构建中）两棵树。首次渲染会构建 `workInProgress` 树，在 Commit 提交后其将转换为 `current` 树。
