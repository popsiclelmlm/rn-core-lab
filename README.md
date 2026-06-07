# rn-core-lab

React Native 框架内核学习笔记（基于官方仓库 `0.86-stable` 真实源码）。
所有文件路径相对于 `packages/react-native/`，行号以阅读时源码为准（可能随版本微调）。

---

## 全局心智模型（先记住这个）

新架构（Fabric）里有 **三个线程、两套树**：

```
       JS 线程                  Shadow 线程(后台)            UI 线程(主线程)
   ┌──────────────┐        ┌───────────────────┐      ┌────────────────────┐
   │ <View/>      │        │  C++ ShadowTree    │      │  Android 原生 View  │
   │ React 协调器  │──JSI──▶│  ShadowNode + Yoga │      │  ReactViewGroup     │
   │ (Fabric绑定)  │        │  布局 + Diff        │──JNI▶│  SurfaceMounting    │
   └──────────────┘        └───────────────────┘      └────────────────────┘
```

- **两套树**：JS 侧 React 元素/Fiber 树；C++ 侧不可变的 ShadowNode 树。
- **布局和 diff 全在 C++ 后台线程算完**，主线程只负责「照单创建/摆放 View」。
- 贯穿三层的主线是组件名字符串 **`'RCTView'`**：JS 注册表 → C++ ComponentDescriptor 注册表 → Android ViewManager 注册表，每层都用它查「用哪个工厂创建」。

---

## 文档目录（建议阅读顺序）

| # | 文档 | 一句话 |
|---|---|---|
| 1 | [01-render-pipeline.md](./01-render-pipeline.md) | **宏观链路**：`<View>` 从 JSX 到屏幕上一个 `android.view.View` 的 8 个阶段 |
| 2 | [02-host-component-and-viewconfig.md](./02-host-component-and-viewconfig.md) | **宿主组件**：`ViewNativeComponent` 其实就是字符串 `'RCTView'` + 一份懒注册 ViewConfig |
| 3 | [03-reconciler-and-fiber-tree.md](./03-reconciler-and-fiber-tree.md) | **协调器**：复合组件 vs 宿主组件；Fiber 树何时、由谁触发构建 |
| 4 | [04-mount-and-unmount.md](./04-mount-and-unmount.md) | **挂载/卸载**：导航懒加载、pop=卸载、三元 null=卸载、隐藏 vs 卸载 |
| 5 | [05-render-vs-commit-phase.md](./05-render-vs-commit-phase.md) | **两阶段**：render（可中断打草稿）vs commit（一次性落地）完整时间线 |
| 6 | [06-fiber-internals.md](./06-fiber-internals.md) | **Fiber 内核**：数据结构、迭代遍历、协作式可中断的原理 |

> 阅读建议：1→2→3 建立「数据怎么流」的宏观图；4→5→6 深入「React 协调器内部怎么运转」。
> 6 是骨架，理解了它前面所有点都能串起来。

---

## 关键源码文件总表

| 层 | 文件 |
|---|---|
| JS - View 组件 | `Libraries/Components/View/View.js` |
| JS - 宿主组件声明 | `Libraries/Components/View/ViewNativeComponent.js` |
| JS - 组件注册表 | `Libraries/NativeComponent/NativeComponentRegistry.js` |
| JS - ViewConfig | `Libraries/NativeComponent/ViewConfig.js` |
| JS - Fabric 渲染器实现 | `Libraries/Renderer/implementations/ReactFabric-dev.js` |
| JS - 渲染入口 | `Libraries/ReactNative/{AppRegistry,renderApplication,RendererImplementation}.js` |
| C++ - ShadowNode | `ReactCommon/react/renderer/components/view/ViewShadowNode.h` |
| C++ - ShadowNode 模板 | `ReactCommon/react/renderer/components/view/ConcreteViewShadowNode.h` |
| C++ - 工厂 | `ReactCommon/react/renderer/components/view/ViewComponentDescriptor.h` |
| C++ - diff 算法 | `ReactCommon/react/renderer/mounting/Differentiator.cpp` |
| C++ - 变更指令 | `ReactCommon/react/renderer/mounting/ShadowViewMutation.h` |
| C++ - 调度器 | `ReactCommon/react/renderer/runtimescheduler/RuntimeScheduler*.{h,cpp}` |
| Android - JNI 入口 | `ReactAndroid/.../fabric/FabricUIManager.java` |
| Android - 挂载管理 | `ReactAndroid/.../fabric/mounting/SurfaceMountingManager.kt` |
| Android - 指令分发 | `ReactAndroid/.../fabric/mounting/MountItemDispatcher.kt` |
| Android - View 管理器 | `ReactAndroid/.../views/view/ReactViewManager.kt` |

---

## 待办 / 下一步

- [ ] 在 rn-tester 给 `ReactViewManager.createViewInstance` / `setPointerEvents` 下断点，跑通一次真实调用栈
- [ ] 深入 `Differentiator.cpp`：diff 怎么处理 reorder / flatten / virtual view
- [ ] 深入 JSI / Bridgeless：`installBindings` 怎么把 C++ 函数挂到 JS
- [ ] 深入 Codegen：`'RCTView'` 的 props 规格如何生成 C++ `ViewProps` + Android `@ReactProp`
