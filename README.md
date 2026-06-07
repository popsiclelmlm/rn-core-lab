# rn-core-lab

React Native 框架内核学习笔记（基于官方仓库 `0.86-stable` 真实源码）。
所有文件路径相对于 `packages/react-native/`，行号以阅读时源码为准（可能随版本微调）。

---

## 全局架构模型

新架构（Fabric）采用多线程环境配合双树结构进行视图管理：

```
        JS 线程                  Shadow 线程(后台)            UI 线程(主线程)
    ┌──────────────┐        ┌───────────────────┐      ┌────────────────────┐
    │ <View/>      │        │  C++ ShadowTree    │      │  Android 原生 View  │
    │ React 协调器  │──JSI──▶│  ShadowNode + Yoga │      │  ReactViewGroup     │
    │ (Fabric绑定)  │        │  布局 + Diff        │──JNI▶│  SurfaceMounting    │
    └──────────────┘        └───────────────────┘      └────────────────────┘
```

- **双树结构**：JavaScript 侧的 Fiber 树与 C++ 侧的不可变 ShadowNode 树。
- **布局与 Diff 计算**：布局计算与 Diff 比对均在 C++ 后台线程完成，主线程仅负责根据最终的变更指令集进行原生 View 的创建与排版布局。
- **组件标识符映射**：组件名字符串（如 `'RCTView'`）作为贯穿三层架构的标识符，在 JS 注册表、C++ ComponentDescriptor 注册表以及 Android ViewManager 注册表中依次匹配，用于路由查找对应的组件工厂。

---

## 文档目录（建议阅读顺序）

| # | 核心文档 | 核心摘要 |
|---|---|---|
| 1 | [01-render-pipeline.md](./01-render-pipeline.md) | **全局渲染链路**：梳理 `<View>` 从 JSX 声明直至映射为原生 View 呈现的八个核心阶段 |
| 2 | [02-host-component-and-viewconfig.md](./02-host-component-and-viewconfig.md) | **宿主组件原理**：剖析 `ViewNativeComponent` 的类型本质与延迟加载的 ViewConfig Schema 注册机制 |
| 3 | [03-reconciler-and-fiber-tree.md](./03-reconciler-and-fiber-tree.md) | **协调器运行时**：对比复合组件与宿主组件的逻辑分流，阐述 Fiber 树的按需构建时机 |
| 4 | [04-mount-and-unmount.md](./04-mount-and-unmount.md) | **组件挂载与卸载**：基于 React 卸载定律，剖析导航栈操作（Pop）、条件渲染及视图隐藏的底层差异 |
| 5 | [05-render-vs-commit-phase.md](./05-render-vs-commit-phase.md) | **渲染与提交双阶段**：详解可中断的 Render 阶段与同步执行的 Commit 阶段的完整生命周期 |
| 6 | [06-fiber-internals.md](./06-fiber-internals.md) | **Fiber 架构内核**：介绍 Fiber 数据结构、非递归的链表迭代 DFS 遍历算法以及协作式时间分片原理 |
| 7 | [07-reconciliation-bailout.md](./07-reconciliation-bailout.md) | **协调剪枝机制**：分析如何通过 `lanes` 与 `childLanes` 标记子树待处理任务以实现局部 Bailout 跳过 |
| 8 | [08-layout-vs-render.md](./08-layout-vs-render.md) | **渲染与布局解耦**：阐述子节点尺寸变更时，布局如何在 C++ 层向上传播，并引发兄弟节点低成本重定位的过程 |
| 9 | [09-runtime-and-threading.md](./09-runtime-and-threading.md) | **运行时与线程模型**：剖析单线程 `jsi::Runtime` 约束、跨线程任务执行器（RuntimeExecutor）以及事件循环（RuntimeScheduler）的实现 |

### 阅读建议
1. **宏观流程建立**：推荐阅读 [01](./01-render-pipeline.md) 至 [03](./03-reconciler-and-fiber-tree.md)，以建立数据流与渲染管线的宏观图景。
2. **核心机制剖析**：推荐阅读 [04](./04-mount-and-unmount.md) 至 [06](./06-fiber-internals.md)，深入分析 React 协调器内部的任务循环、非递归遍历以及可中断计算实现。
3. **性能调优与物理细节**：[07](./07-reconciliation-bailout.md) 与 [08](./08-layout-vs-render.md) 从性能视角出发，阐述剪枝策略与布局抖动的优化；[09](./09-runtime-and-threading.md) 则补充了底层线程交互、Runtime 管理的物理细节。

### 🔬 实战实验记录

理论之外的“手动验证记录”保存在 [`experiments/`](./experiments/) 目录中。我们使用真机及 Debugger 断点对相关内部状态进行了跟踪和印证。

| 实验编号 | 主题内容 | 状态 |
|---|---|---|
| [environment-setup](./experiments/00-environment-setup.md) | Android Studio 构建配置与 JDK 等依赖排坑记录 | 已完成 |
| [A](./experiments/A-view-creation-and-preallocate.md) | 原生 View 实例创建、预分配机制与帧渲染链路分析（附真实调用栈） | 已完成 |
| [B](./experiments/B-prop-update.md) | Props 属性的最小化更新逻辑（剪枝过程） | 已完成 |
| [C](./experiments/C-layout-propagation.md) | 兄弟节点的 `updateLayout` 布局传播路径验证 | 已完成 |
| [D](./experiments/D-cpp-lldb-and-layout-thread.md) | C++ 层 LLDB 动态调试与 JS 线程中布局阶段的实测判定 | 进度：部分完成（重测量条件待补） |

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
