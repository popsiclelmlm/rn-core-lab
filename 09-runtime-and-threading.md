# 09 · 运行时与线程模型

> 核心机制：本篇介绍 React Native 的运行时与线程模型，阐述 JavaScript 线程、JSI 引擎句柄（Runtime）、RuntimeExecutor 与事件循环（RuntimeScheduler）的协作关系，解析多实例及跨环境通信的底层实现。
> 相关：[05 render/commit](./05-render-vs-commit-phase.md) · [06 Fiber 内核](./06-fiber-internals.md) · [实验 D](./experiments/D-cpp-lldb-and-layout-thread.md)

---

## 1. JS 线程的物理本质

- **线程管理**：在 Android 侧，“JS 线程”在物理上由 Java 层的 `MessageQueueThread`（基于 Looper 实现的线程）管理。
- **共享运行环境**：每个 React Native 实例均分配一条独立的 JS 线程。Hermes 引擎实例以及 C++ 层的核心模块（如 `RuntimeScheduler`、`UIManager`、`ShadowTree` 与 Yoga）均运行在此条线程上。

---

## 2. jsi::Runtime：JavaScript 引擎的控制句柄

- **引擎抽象**：JavaScript 引擎（如 Hermes）是真正解析并执行字节码的底层虚拟机。
- **控制句柄**：`jsi::Runtime` 是 C++ 层操作底层 Hermes VM 实例的接口句柄，用于在 C++ 中执行 JS 代码、封装变量及绑定交互函数。
- **单线程局限**：每一个 `jsi::Runtime` 实例代表一个相互隔离的执行上下文（拥有独立的堆内存）。**`jsi::Runtime` 并非线程安全**，仅允许在分配的 JS 线程中被调用。同时，由其创建的 `jsi::Value`、`jsi::Object` 与 `jsi::Function` 也与该 Runtime 实例强绑定，禁止跨实例传递使用。

---

## 3. JNI 与 JSI 边界对比

| 交互通道 | 交互各方 | 典型场景 |
|---|---|---|
| **JNI** | **Java 与 C++ 交互** | 异步向 JS 线程投递 Native 任务；`FabricUIManager.scheduleMountItem` 的调用 |
| **JSI** | **JavaScript 与 C++ 交互** | 跨语言同步调用，如 `UIManagerBinding` 触发 `completeSurface` 或是 Codegen 自动生成的 TurboModule |

JNI 负责 Java 与 C++ 层之间的互操作，而 JSI 则负责 JS 与 C++ 层之间的直接数据访问，二者的计算任务均可在同一条 JS 线程上被执行。

---

## 4. RuntimeExecutor：跨线程任务调度器

在头文件 `RuntimeExecutor.h` 中，其类型定义如下：
```cpp
using RuntimeExecutor = std::function<void(std::function<void(jsi::Runtime& runtime)>&&)>;
```
在底层构建（如 `ReactInstance.cpp:70`）中，该执行器的本质是将任务投递至 JS 线程队列（通过 `jsThread->runOnQueue(...)` 执行）。当 JS 线程轮询调度到该任务时，会将 `jsi::Runtime` 实例作为实参，同步传入开发者的回调函数中。

### 任务提交与引擎访问的边界

| 交互行为 | 执行线程 | Android 原生类比 |
|---|---|---|
| **提交任务**（执行 `runtimeExecutor(callback)`） | **任意线程**（支持跨线程调用） | `Handler.post(Runnable)` |
| **访问 Runtime**（在 callback 内部执行 `runtime.xxx()`） | **仅允许在 JS 线程** | 在 `runOnUiThread` 中安全更新 UI 元素 |

在调用 `runtimeExecutor(callback)` 时，调用方线程并不直接持有 `runtime` 引用。`runtime` 会以参数的形式传递至回调函数中，仅在 JS 线程开始执行回调时才暴露出来。这种 API 设计在编译期限制了开发者从非 JS 线程直接非法访问 Runtime 的可能性。

常见任务提交场景包括：UI 主线程（响应手势或原生系统回调）、JS 线程自身（处理 setState 或 Promise）、后台工作线程（返回原生模块的异步计算结果）。例如一次点击事件，任务的调度路径往往在 UI 线程与 JS 线程之间流转。

---

## 5. RuntimeScheduler：JavaScript 线程的事件循环驱动

此机制遵循 Web Event Loop 规范，采用按需调度的事件循环模型，并非固定频率的定时器。相关逻辑位于 `RuntimeScheduler_Modern.cpp`：

- **任务队列管理**：基于优先级与过期时间进行排序，数据结构为优先级队列 `std::priority_queue` 结合 `TaskPriorityComparer`。
- **循环触发与任务调度**（`scheduleTask :220`）：仅在任务队列原先为空时启动事件循环；若事件循环已经在执行过程中，则仅将新任务入队，由当前执行的循环在后续迭代中取出。此机制避免了空转开销。
- **事件循环主体**（`runEventLoop :254`）：在事件循环未被同步任务抢占且队列不为空的前提下，持续取出最高优先级的任务并交由 `runEventLoopTick` 处理，当任务被排空时退出循环。
- **Event Loop 的单次循环（Tick）**（`runEventLoopTick :294`）：依次执行以下核心步骤：
  1. `executeTask`：执行队列任务（进 JavaScript 层执行，如 React 的 Commit 与 Layout 计算阶段均在此处触发）。
  2. `performMicrotaskCheckpoint`：清空微任务队列（例如处理 JavaScript 中的 Promise 状态变更）。
  3. `updateRendering`：触发渲染管线的更新步骤。

通过单次循环（Tick）而非固定定时器排空任务的设计，使得 React Native 新架构能够在底层对齐 Promise、微任务（Microtask）及渲染事件的 Web 标准时序规范。

---

## 6. 核心概念术语表

| 核心名词 | 名词释义与架构定位 | Android 机制对比 |
|---|---|---|
| Hermes | 底层负责解析并执行 JS 字节码的虚拟机（VM） | ART / Dalvik 虚拟机 |
| `jsi::Runtime` | C++ 侧访问及操作 JS 执行环境的单线程接口句柄 | 限制在 UI 线程调用的 View 对象 |
| JSI | 供 JavaScript 与 C++ 间进行直接同步调用的接口规范 | 用于实现跨语言同步数据访问 |
| JNI | 供 Java 与 C++ 进行互操作的桥接技术 | 用于实现跨语言任务派发 |
| JS 线程 | 唯一允许直接发起 `jsi::Runtime` 操作的 OS 线程 | UI 主线程 (UI Thread) |
| RuntimeExecutor | 支持跨线程投递并将计算任务分发到 JS 线程执行的调度接口 | `runOnUiThread` / `Handler.post` |
| RuntimeScheduler | 负责在 JS 线程上驱动任务队列、微任务及渲染排版的事件循环 | `Looper` 与优先级队列的组合 |

---

## 7. 多 Runtime 实例与跨实例通信

在工程实践中可以创建多个独立的 `jsi::Runtime` 实例（类似于 Web 中的 Web Worker 隔离机制）。各个 Runtime 实例在内存上互不干扰，**无法直接共享 JavaScript 引用对象**。它们之间的数据通信需要通过“读取数据为纯 C++ 数据类型（或序列化数据）”并“跨线程投递”来完成：

```cpp
// A 线程：自 Runtime A 中读取出 C++ 基础数据（脱离当前环境引用）
int count = aMsg.getProperty(rtA, "count").asNumber();
// 利用 B 的 executor 执行器将数据投递至 Runtime B 的运行线程并执行回调
runtimeExecutorB([count](jsi::Runtime& rtB) {
  rtB.global().getPropertyAsFunction(rtB, "__onMessageFromA").call(rtB, jsi::Value(count));
});
```
- **支持传递的数据**：支持深度拷贝的数据类型或支持被序列化传输的数据。
- **不支持传递的数据**：JSI 引用对象以及携带上下文引用的 JavaScript 闭包（在 `react-native-reanimated` 中，此限制通常通过在编译期将闭包转换为 Worklet 代码解决）。
- **实际应用场景**：`react-native-reanimated` 的双 Runtime 架构（包含主 JavaScript Runtime 与独立的 UI Worklet Runtime，通过 `runOnUI`、`runOnJS` 及 Shared Values 进行状态同步）；Vision Camera 的图像帧处理器（Frame Processor）。

---

## 8. 多 Surface 与多 ReactHost 实例对比

在应用层设计多页面架构时，需要根据业务隔离度选择使用多 Surface 架构还是多 ReactHost 实例架构：

| 架构维度 | 共享 ReactHost（单实例多 Surface） | 多 ReactHost（独立实例） |
|---|---|---|
| Runtime 执行上下文 | **共享同一个 Runtime 实例** | 各实例采用完全独立的 Runtime 运行 |
| 跨页面数据通信 | **在 JS 线程中直接进行同步通信** | **必须通过 Native 原生桥接层转发** |
| 运行内存与 CPU 开销 | 低（共享引擎与核心库） | 高（每个实例均包含独立的 Runtime、Bundle 拷贝及 UIManager 体系） |
| 崩溃及业务隔离度 | 无隔离（共享 JS 线程，一处崩溃会导致全局崩溃） | 强隔离（独立 Bundle，崩溃域相互隔离） |

- **多 Surface 共享实例**：通过 `ReactHost.createSurface(...)` 可以在单实例中挂载**多个 Surface 视图**（由 `ReactHostImpl.attachedSurfaces` 管理），各页面之间共享执行上下文，适用于无需强隔离的单包多页面业务。
- **多实例隔离**：对于大型超级应用（Super App）中各微应用之间需要进行代码隔离或崩溃域隔离的场景，采用多实例机制。在此架构下，跨实例的数据流转需借助**原生消息总线**作为中介进行转发：

```kotlin
object RNMessageBus {
  private val contexts = mutableMapOf<String, ReactContext>()   // 注册表：ID -> 实例 Context
  fun register(id: String, ctx: ReactContext) { contexts[id] = ctx }
  fun send(to: String, event: String, data: WritableMap) {
    contexts[to]?.getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter)?.emit(event, data)
  }
}
```
其调度路径为：`JS 实例 A → 调用本地 TurboModule → 原生 RNMessageBus.send("B", ...) → 原生底层 emit 广播至实例 B 的 Context 上下文 → JS 实例 B 的 DeviceEventEmitter 监听到事件并执行业务逻辑`。

通信的基本原则与跨 Runtime 一致，即：**复制数据内容、避免共享对象引用、通过消息循环进行跨线程/跨实例调度**。由于 React Native 未内置现成的跨实例通信机制，此逻辑需要由开发者利用 Native Module 自行封装实现。

---

## 关键源码索引

| 主题 | 文件 |
|---|---|
| RuntimeExecutor 类型 | `ReactCommon/callinvoker/ReactCommon/RuntimeExecutor.h:22` |
| RuntimeExecutor 构造（投递到 JS 线程） | `ReactCommon/react/runtime/ReactInstance.cpp:70` |
| 事件循环 | `ReactCommon/.../runtimescheduler/RuntimeScheduler_Modern.cpp`（scheduleTask:220 / runEventLoop:254 / runEventLoopTick:294） |
| Surface 管理 | `ReactAndroid/.../ReactHost.kt`（createSurface）· `runtime/ReactHostImpl.kt`（attachedSurfaces） |
