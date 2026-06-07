# 02 · 宿主组件：`ViewNativeComponent` 与 ViewConfig

> 聚焦 [01](./01-render-pipeline.md) 链路中的 ① ——「JS 侧那个 `<View>` 到底是什么」。
> 相关：[03 协调器](./03-reconciler-and-fiber-tree.md)（协调器如何处理这个宿主组件）

---

## 核心设计：`ViewNativeComponent` 的类型本质

`get()` 最后一行（`NativeComponentRegistry.js:111`）：

```js
// $FlowFixMe[incompatible-type] `NativeComponent` is actually string!
return name;
```

在底层实现中，`ViewNativeComponent` 并不是 React 组件对象，而是**字符串 `'RCTView'` 本身**。
代码中写 `<ViewNativeComponent {...props}/>` 等价于 `<'RCTView' {...props}/>`。

---

## 运行时的两个核心逻辑

`get('RCTView', configFactory)` 在被引入（import）时会触发以下操作：

**1. 注册延迟加载的 ViewConfig 工厂至 JS 侧注册表**
```js
// NativeComponentRegistry.js:55
ReactNativeViewConfigRegistry.register(name, () => { ... return viewConfig; });
```
第二个参数为回调函数，采用懒加载机制——仅在渲染器首次解析 `'RCTView'` 配置时被触发（即 [03](./03-reconciler-and-fiber-tree.md) 中的 `getViewConfigForType`）。

**2. 返回字符串 `'RCTView'` 作为宿主组件类型**（`return name`）。

因此，`ViewNativeComponent` 在形式上等价于**组件名字符串与对应 ViewConfig Schema 的声明组合**，用于元数据登记，本身不承担运行时逻辑。

---

## ViewConfig 里有什么（`ViewConfig.js:24`）

```js
{
  uiViewClassName,      // 'RCTView'
  Commands: {},
  bubblingEventTypes,   // 哪些事件冒泡
  directEventTypes,     // 哪些事件直达（如 onLayout）
  validAttributes,      // 哪些 prop 合法 + 怎么 diff —— 渲染器运行时会用！
}
```

`validAttributes` 不是死信息：**渲染器每次 diff props 时都用它**决定哪些 prop 往原生传、怎么处理。
但它是「说明书（schema）」，不是「货物（数据）」。

---

## 常见问题梳理

| 概念描述 | 结论 | 说明 |
|---|---|---|
| 仅用于定义组件名称映射与属性 | 基本正确 | 更准确的描述是其执行了“注册延迟 ViewConfig Schema”和“返回组件名字符串”两个步骤 |
| 实际不直接与 C++ 或原生层交互 | 正确 | 此文件自身不直接发起交互，但 Legacy 反射路径存在特例（详见下文） |
| 不作为数据传输通道 | 正确 | Props 数据的实际传输不经过此文件 |

**非数据通道说明**：`<View style={{width:100}}/>` 中的 `style` 数据并不流经此文件。其真实流动路径如下：
```
协调器获取 props → 查询 ViewConfig.validAttributes（根据 Schema 验证）
  → 调用 Fabric JSI 绑定 createNode(tag, 'RCTView', surfaceId, props, ...) → 进入 C++ 层
```
`ViewNativeComponent` 仅声明组件名称和 Schema，不参与具体数据的传输。

**Legacy 兼容路径**（`NativeComponentRegistry.js:62`）：在旧架构（非 Bridgeless 模式，即 `!global.RN$Bridgeless`）下，config 回调会通过 `getNativeComponentAttributes(name)` 向原生反射获取配置。但这仅限于旧架构，且只有在该延迟回调被执行时才会发生。

---

## 总结

`ViewNativeComponent` 的本质是组件标识符（字符串 `'RCTView'`）以及在 `ReactNativeViewConfigRegistry` 中懒加载注册的配置 Schema。它负责声明组件元数据（名称、合法属性、支持的事件），而不直接参与渲染时的指令执行或数据流传递。
