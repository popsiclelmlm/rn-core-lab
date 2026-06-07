# 01 · 渲染链路总览：`<View>` 从 JS 到 Android 屏幕

> 宏观视角：一次渲染要穿过哪几层。细节分散在后续各篇，本篇是「地图」。
> 相关：[02 宿主组件](./02-host-component-and-viewconfig.md) · [03 协调器](./03-reconciler-and-fiber-tree.md) · [05 两阶段](./05-render-vs-commit-phase.md)

---

## 全局图

```
       JS 线程                  Shadow 线程(后台)            UI 线程(主线程)
   ┌──────────────┐        ┌───────────────────┐      ┌────────────────────┐
   │ <View/>      │        │  C++ ShadowTree    │      │  Android 原生 View  │
   │ React 协调器  │──JSI──▶│  ShadowNode + Yoga │      │  ReactViewGroup     │
   │ (Fabric绑定)  │        │  布局 + Diff        │──JNI▶│  SurfaceMounting    │
   └──────────────┘        └───────────────────┘      └────────────────────┘
       ①②                       ③④⑤                        ⑥⑦⑧
```

**工作原理**：布局计算与 Diff 比较均在 C++ 后台线程完成，最终向主线程下发一批「原子变更指令」。主线程主要负责按照指令创建并布置 View 树，不参与复杂的布局计算。

---

## ① JS 层：`<View>` 是什么

`Libraries/Components/View/View.js:26` 是薄包装（处理 `aria-*`、Text 上下文），最后渲染宿主组件：

```js
// View.js:121
const actualView = <ViewNativeComponent {...resolvedProps} />;
```

`ViewNativeComponent`（`ViewNativeComponent.js:18`）本质上**就是字符串 `'RCTView'`**——详见 [02](./02-host-component-and-viewconfig.md)。

## ② JS → C++：协调器通过 JSI 下发

Fabric 渲染器（`Libraries/ReactNative/FabricUIManager.js`）通过 **JSI 同步调用** C++ 构建 ShadowNode 树：
`createNode(tag, "RCTView", surfaceId, props, ...)` / `appendChild` / `completeRoot`。

**JSI 替代 Bridge 的实现方式**：JSI 允许 JS 直接持有 C++ 对象引用，通过同步调用避免了数据序列化的开销。

## ③ C++ 层：ShadowNode 与工厂

用 `"RCTView"` 经 ComponentDescriptor 注册表找到工厂 `ViewComponentDescriptor.h:15`，new 出
`ViewShadowNode`（`ViewShadowNode.h:33`）。

- ShadowNode **不可变**：每次渲染产生新树（旧节点 clone），才能做新旧对比。
- 继承 `YogaLayoutableShadowNode`（`ConcreteViewShadowNode.h:31`）→ View 自带 Flexbox 布局能力。

## ④ C++ 层：Yoga 布局（后台线程）

提交前用 Yoga 算出每个节点的 `LayoutMetrics`（x/y/width/height）。**布局在后台线程，不占主线程。**

## ⑤ C++ 层：Commit + Diff → 变更指令

`ShadowTree::commit()` 调 **Differentiator**（`mounting/Differentiator.cpp`）对比旧树 vs 新树，产出扁平变更列表
`ShadowViewMutation`（`ShadowViewMutation.h:84`）：

```cpp
enum Type : std::uint8_t { Create=1, Delete=2, Insert=4, Remove=8, Update=16 };
```

**关键机制**：React 在 JS 侧对比组件树的变更，而 Fabric 在 C++ 侧将这些变更翻译为针对原生视图树的最小原子操作序列（即变更指令集），主线程最终执行这些指令。

## ⑥ C++ → Android：跨 JNI 边界

C++ 的 `Binding.cpp` 通过 JNI 调 `FabricUIManager.java:909` 的 `scheduleMountItem()`（注释明写
*"Called from C++ via JNI (Binding.cpp)"*）。整批 mutation 打包成 **IntBufferBatchMountItem**（`int[]` + `Object[]` 紧凑编码），一次 JNI 传一整批。

## ⑦ Android：主线程消费队列

`MountItemDispatcher.kt` 在 **UI 线程** 取出 BatchMountItem 逐条执行，分发给 `SurfaceMountingManager`（每 surface 一个）。

## ⑧ Android：真正创建 `android.view.View`

`Create` → `SurfaceMountingManager.createViewUnsafe()`（`SurfaceMountingManager.kt:586`）用组件名找到 ViewManager 造 View：

```kotlin
// SurfaceMountingManager.kt:610
val viewManager = viewManagerRegistry?.get(componentName) as ViewManager<View, *>
viewState.view = viewManager.createView(...)
// → ReactViewManager.createViewInstance() (ReactViewManager.kt:423)
//   → return ReactViewGroup(context)   ← 终点：真正的 android.view.View
```

其余指令：`Insert`→`addViewAt()`（`:312`）；`Update`→`updateProps()`（`:644`）经 `@ReactProp` setter 映射；
`updateLayout`→ 用 ⑤ 算好的 LayoutMetrics 调 `view.layout(...)`。下一帧上屏。

---

## 渲染流程总结

JSX 中的 `<View>`（宿主组件，名称为 `RCTView`）通过 JSI 在 C++ 层同步构建不可变的 ShadowNode 树。Yoga 引擎在后台线程完成布局计算后，Differentiator 对比新旧两棵树，生成包含 Create/Insert/Update/Remove/Delete 等操作的变更指令。这些指令经打包后通过 JNI 传递到 UI 线程，由 `SurfaceMountingManager` 调度对应的 `ReactViewManager` 进行原生 View 的创建与排版，最终完成上屏渲染。
