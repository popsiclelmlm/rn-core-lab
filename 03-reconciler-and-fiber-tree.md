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
   → 不再调用任何 JS 组件函数      ← "停止下降"
   → 建一个 HostComponent fiber
   → 造原生节点 createNode(...)
```

**字符串 type 本身就是停止信号**，不存在「再递归进 ViewNativeComponent 一次」——里面没有函数可进。

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

### 「停止」的精确边界

1. 停的是「对当前这一支继续调用 JS 组件函数」。若 View 有子节点（`<View><Text/></View>`），建完 RCTView host fiber 后会**继续协调 children**，Text 又是复合组件 → 再调 `Text(...)`。
2. 建 host fiber（render 阶段）≠ 造原生 View（commit 阶段）。`createNode` 造的是 **C++ ShadowNode**（仍在 JS/JSI 侧），真正的 `android.view.View` 在更后面经 JNI 到主线程才创建（见 [01](./01-render-pipeline.md) ⑧）。

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

**结论：在某个 surface 第一次 render 时——不是进程启动那刻，也不是 `registerComponent` 那刻。**

```
App 启动，加载 JS bundle
  → AppRegistry.registerComponent('AppName', () => App)   ← 只是"登记工厂"，不建 Fiber
  ─────────────────────────────────────────────────────
  Native 创建一个 surface / root view（Android: ReactSurface / ReactRootView）
  → 回调 JS: AppRegistry.runApplication('AppName', params)   ← 触发点！
  → renderApplication(...)                                   (renderApplication.js)
  → Renderer.renderElement({element, rootTag})              (renderApplication.js:88)
  → ReactFabric.render(element, rootTag, null, true, ...)   (RendererImplementation.js:28)
  → 首次为该 rootTag 创建 FiberRoot（根容器 + HostRoot fiber）
  → schedule 初始渲染 → work loop 自顶向下调用组件函数 → 构建 workInProgress Fiber 树
```

要点：

1. **`registerComponent` 只是登记**，把 `() => App` 存进表，一个 Fiber 都不建。
2. **真正触发是 `runApplication`**，由原生侧创建 surface 时回调进来。
3. **每个 surface 一棵独立的 Fiber 树**（独立 `FiberRoot`），多 Activity/多 root view 互不相干。
4. **双缓冲**：`current`（已提交）与 `workInProgress`（正在构建）两棵；首次构建 workInProgress，commit 后变 current。

> 类比 Android：`registerComponent` ≈ 在 manifest 里声明 Activity；`runApplication` ≈ 系统真正 `onCreate` + `setContentView`。声明 ≠ 实例化。
