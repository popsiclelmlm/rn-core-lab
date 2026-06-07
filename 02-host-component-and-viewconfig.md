# 02 · 宿主组件：`ViewNativeComponent` 与 ViewConfig

> 聚焦 [01](./01-render-pipeline.md) 链路中的 ① ——「JS 侧那个 `<View>` 到底是什么」。
> 相关：[03 协调器](./03-reconciler-and-fiber-tree.md)（协调器如何处理这个宿主组件）

---

## 啊哈时刻：它就是字符串 `'RCTView'`

`get()` 最后一行（`NativeComponentRegistry.js:111`）：

```js
// $FlowFixMe[incompatible-type] `NativeComponent` is actually string!
return name;
```

**`ViewNativeComponent` 根本不是对象或组件——它就是字符串 `'RCTView'` 本身。**
写 `<ViewNativeComponent {...props}/>` 等价于 `<'RCTView' {...props}/>`。

---

## 它做了两件运行时的事

`get('RCTView', configFactory)` 在被 import 时：

**动作 A：注册一个「懒加载的 ViewConfig 工厂」到 JS 侧注册表**
```js
// NativeComponentRegistry.js:55
ReactNativeViewConfigRegistry.register(name, () => { ... return viewConfig; });
```
第二个参数是**回调**，不立即执行——只有渲染器第一次需要 `'RCTView'` 配置时才调用（即 [03](./03-reconciler-and-fiber-tree.md) 里 `getViewConfigForType`）。

**动作 B：返回字符串 `'RCTView'` 作为宿主组件类型**（`return name`）。

所以 `ViewNativeComponent` ≈ **一个字符串 + 一份懒注册的 schema**，是**声明/登记**，不是运行时执行者。

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

## 三个常见疑问的精确答案

| 判断 | 对错 | 精确说法 |
|---|---|---|
| 只是定义文件，定义名字映射和属性 | ✅ 基本对 | 更准确是「注册懒 schema + 返回字符串」两个动作 |
| 实际不与 C++/原生交互 | ✅ 对（此文件本身） | legacy 反射路径有小例外，见下 |
| 不作为数据传输 | ✅ 完全正确 | props 数据根本不走这里 |

**为什么不是数据通道**：`<View style={{width:100}}/>` 的 `style` 不经过此文件。真实路径：
```
协调器拿 props → 查 ViewConfig.validAttributes（用 schema 决定怎么处理）
  → 调 Fabric JSI binding createNode(tag, 'RCTView', surfaceId, props, ...) → 进 C++
```
`ViewNativeComponent` 只贡献「组件名」和「schema」。**它是路牌，不是路上的车。**

**legacy 小例外**（`NativeComponentRegistry.js:62`）：旧架构 `!global.RN$Bridgeless` 下，config 回调会调
`getNativeComponentAttributes(name)` 向原生反射要配置。但 (1) 只在旧架构；(2) 只在那个懒回调被调用时才发生，不是此文件主动发起。

---

## 一句话总结

> `ViewNativeComponent` 本质是**字符串 `'RCTView'` + 一份懒注册的 ViewConfig schema**。它是**声明**（这个组件叫什么、有哪些合法 props/事件），不是**执行者**，也不是**数据通道**。
> 类比 Android：像在某个 registry 里 `register("RCTView", configProvider)` 并拿到一个 type token——让系统「认识」这个组件，但搬数据、造 View 是别人的活。
