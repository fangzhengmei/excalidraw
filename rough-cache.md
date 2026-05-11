# Excalidraw 手绘风格图形缓存机制分析

## 1. 概述

Excalidraw 使用 [roughjs](https://roughjs.com/) 库生成手绘风格的图形。为了确保**同一图形在重绘时保持一致的手绘效果**，同时又能**在不同图形之间产生随机差异**，项目实现了一套精巧的缓存机制。

核心思路：
- **随机种子固化**：每个元素拥有独立的随机种子，确保重复渲染时手绘效果稳定
- **两级缓存**：ShapeCache（roughjs 形状数据）+ elementWithCanvasCache（渲染结果 Canvas）
- **智能失效**：基于对象引用和属性变化精确控制缓存生命周期

---

## 2. 随机种子固化机制

### 2.1 种子的生成与存储

**位置**：`packages/element/src/newElement.ts:145`

```typescript
// _newElementBase 函数中
const element: Merge<ExcalidrawGenericElement, { type: T["type"] }> = {
  // ...
  seed: rest.seed ?? randomInteger(),
  // ...
};
```

**种子生成**：
- 位置：`packages/common/src/random.ts:9`
- 使用 roughjs 内置的 `Random` 类
- 初始种子基于 `Date.now()`

```typescript
import { Random } from "roughjs/bin/math";

let random = new Random(Date.now());

export const randomInteger = () => Math.floor(random.next() * 2 ** 31);
```

**元素类型定义**：`packages/element/src/types.ts:55-57`

```typescript
/** Random integer used to seed shape generation so that the roughjs shape
    doesn't differ across renders. */
seed: number;
```

### 2.2 种子如何传递给 roughjs

**位置**：`packages/element/src/shape.ts:193-264`

```typescript
export const generateRoughOptions = (
  element: ExcalidrawElement,
  continuousPath = false,
  isDarkMode: boolean = false,
): Options => {
  const options: Options = {
    seed: element.seed,  // 关键：将元素的 seed 传递给 roughjs
    // ... 其他选项
  };
  // ...
};
```

**工作原理**：
1. roughjs 的 `RoughGenerator` 使用 `seed` 作为伪随机数生成器的起点
2. 相同的 `seed + 相同图形参数` → 产生**完全一致**的手绘效果
3. 不同元素的 `seed` 不同 → 产生**随机差异**的手绘效果

### 2.3 种子的特殊处理

**复制元素时**：位置 `packages/element/tests/duplicate.test.tsx:81`

```typescript
expect(copy.seed).not.toBe(element.seed);
```

- 复制元素会生成**新的 seed**
- 这确保复制出的图形与原图有略微不同的手绘效果，增加真实感

**数据恢复时**：位置 `packages/excalidraw/data/restore.ts:308`

```typescript
seed: element.seed ?? 1,
```

- 如果历史数据中没有 seed，使用默认值 1

---

## 3. 缓存键计算方式

Excalidraw 采用**两级缓存架构**，两级缓存的键设计完全不同。

### 3.1 第一级：ShapeCache（roughjs 形状数据缓存）

**位置**：`packages/element/src/shape.ts:81-164`

```typescript
export class ShapeCache {
  private static rg = new RoughGenerator();
  private static cache = new WeakMap<
    ExcalidrawElement,
    { shape: ElementShape; theme: AppState["theme"] }
  >();
}
```

**缓存键设计**：
- **Key**：`ExcalidrawElement` 对象本身（对象引用）
- **Value**：`{ shape: ElementShape; theme: AppState["theme"] }`

**为什么用对象引用作为键？**

1. **自动内存管理**：`WeakMap` 不阻止元素对象被垃圾回收，当元素被删除时缓存自动释放
2. **精确失效**：同一元素对象的属性变化时，通过显式调用 `delete` 精确失效
3. **性能**：对象引用比较是 O(1) 的哈希查找

**缓存获取逻辑**：位置 `shape.ts:92-103`

```typescript
public static get = <T extends ExcalidrawElement>(
  element: T,
  theme: AppState["theme"] | null,
) => {
  const cached = ShapeCache.cache.get(element);
  // 只有当 theme 匹配时才返回缓存
  if (cached && (theme === null || cached.theme === theme)) {
    return cached.shape;
  }
  return undefined;
};
```

**缓存写入逻辑**：位置 `shape.ts:118-163`

```typescript
public static generateElementShape = <...>(element, renderConfig) => {
  // 导出时总是重新生成，不使用缓存
  const cachedShape = renderConfig?.isExporting
    ? undefined
    : ShapeCache.get(element, renderConfig ? renderConfig.theme : null);

  if (cachedShape !== undefined) {
    return cachedShape;
  }

  // 生成新的 shape
  const shape = _generateElementShape(element, ShapeCache.rg, renderConfig);

  // 非导出场景才缓存
  if (!renderConfig?.isExporting) {
    ShapeCache.cache.set(element, {
      shape,
      theme: renderConfig?.theme || THEME.LIGHT,
    });
  }

  return shape;
};
```

### 3.2 第二级：elementWithCanvasCache（渲染结果 Canvas 缓存）

**位置**：`packages/element/src/renderElement.ts:603-606`

```typescript
export const elementWithCanvasCache = new WeakMap<
  ExcalidrawElement,
  ExcalidrawElementWithCanvas
>();
```

**缓存值结构**：位置 `renderElement.ts:134-147`

```typescript
export interface ExcalidrawElementWithCanvas {
  element: ExcalidrawElement | ExcalidrawTextElement;
  canvas: HTMLCanvasElement;
  theme: AppState["theme"];
  scale: number;
  angle: number;
  zoomValue: AppState["zoom"]["value"];
  canvasOffsetX: number;
  canvasOffsetY: number;
  boundTextElementVersion: number | null;
  imageCrop: ExcalidrawImageElement["crop"] | null;
  containingFrameOpacity: number;
  boundTextCanvas: HTMLCanvasElement;
}
```

**缓存命中判断**：位置 `renderElement.ts:631-645`

```typescript
if (
  !prevElementWithCanvas ||
  // zoom 变化时失效（除非 shouldCacheIgnoreZoom）
  shouldRegenerateBecauseZoom ||
  // 主题变化时失效
  prevElementWithCanvas.theme !== appState.theme ||
  // 绑定文本版本变化时失效
  prevElementWithCanvas.boundTextElementVersion !== boundTextElementVersion ||
  // 图片裁剪变化时失效
  prevElementWithCanvas.imageCrop !== imageCrop ||
  // 包含帧的透明度变化时失效
  prevElementWithCanvas.containingFrameOpacity !== containingFrameOpacity ||
  // 带标签的箭头角度变化时失效
  (isArrowElement(element) &&
    boundTextElement &&
    element.angle !== prevElementWithCanvas.angle)
) {
  // 需要重新生成
}
```

---

## 4. 缓存失效场景

### 4.1 ShapeCache 显式失效

`ShapeCache.delete(element)` 会同时删除 ShapeCache 和 elementWithCanvasCache：

**位置**：`shape.ts:105-108`

```typescript
public static delete = (element: ExcalidrawElement) => {
  ShapeCache.cache.delete(element);
  elementWithCanvasCache.delete(element);
};
```

#### 场景 1：元素尺寸/路径/图片变化

**位置**：`packages/element/src/mutateElement.ts:130-137`

```typescript
if (
  typeof updates.height !== "undefined" ||
  typeof updates.width !== "undefined" ||
  typeof fileId != "undefined" ||
  typeof points !== "undefined"
) {
  ShapeCache.delete(element);
}
```

**触发条件**：
- `width` 变化
- `height` 变化  
- `fileId` 变化（图片元素）
- `points` 变化（线性元素、自由绘制）

#### 场景 2：字体加载完成

**位置**：`packages/excalidraw/fonts/Fonts.ts:129-139`

```typescript
if (isTextElement(element)) {
  didUpdate = true;
  ShapeCache.delete(element);
  
  const container = getContainerElement(element, elementsMap);
  if (container) {
    ShapeCache.delete(container);
  }
}
```

**原因**：字体加载前后，文本的测量结果会变化，需要重新计算布局和绘制。

#### 场景 3：样式属性变化

**位置**：`packages/excalidraw/components/LayerUI.tsx:517`

```typescript
ShapeCache.delete(element);
```

**触发场景**：通过属性面板修改颜色等样式时。

#### 场景 4：元素类型转换

**位置**：`packages/excalidraw/components/ConvertElementTypePopup.tsx:827`

```typescript
ShapeCache.delete(element);
```

**原因**：矩形 → 椭圆、直线 → 箭头等类型转换后，shape 生成逻辑完全不同。

#### 场景 5：嵌入元素验证状态变化

**位置**：`packages/excalidraw/components/App.tsx:1517`

```typescript
ShapeCache.delete(element);
```

**触发**：嵌入元素（iframe/embeddable）的验证状态变化时。

#### 场景 6：图片缓存清理

**位置**：`packages/excalidraw/components/App.tsx:3093`

```typescript
ShapeCache.delete(element);
```

**触发**：图片文件从缓存中删除时。

#### 场景 7：全局主题切换

**位置**：`packages/excalidraw/components/App.tsx:3247`

```typescript
this.scene
  .getElementsIncludingDeleted()
  .forEach((element) => ShapeCache.delete(element));
```

**原因**：主题切换时，所有元素的颜色需要重新应用暗色模式滤镜。

#### 场景 8：图片文件更新

**位置**：`packages/excalidraw/components/App.tsx:11945`

```typescript
ShapeCache.delete(element);
```

**触发**：当图片文件内容更新时。

### 4.2 elementWithCanvasCache 隐式失效

除了 `ShapeCache.delete` 连带删除外，`elementWithCanvasCache` 还有独立的失效判断：

**位置**：`renderElement.ts:631-645`

| 失效条件 | 说明 |
|---------|------|
| `zoomValue` 变化 | 缩放变化时重新渲染以获得更清晰的结果 |
| `theme` 变化 | 亮色/暗色模式切换 |
| `boundTextElementVersion` 变化 | 绑定的文本内容变化 |
| `imageCrop` 变化 | 图片裁剪区域变化 |
| `containingFrameOpacity` 变化 | 所在帧的透明度变化 |
| 带标签的箭头 `angle` 变化 | 箭头角度变化需要重新计算标签位置 |

### 4.3 导出时的特殊处理

**位置**：`shape.ts:130-132`

```typescript
const cachedShape = renderConfig?.isExporting
  ? undefined
  : ShapeCache.get(element, renderConfig ? renderConfig.theme : null);
```

**导出时总是重新生成**：
- 不读取缓存
- 不写入缓存
- 确保导出的是最新、最完整的图形数据

---

## 5. 架构图解

```
┌─────────────────────────────────────────────────────────────────┐
│                        渲染请求                                  │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              generateElementShape()                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 检查是否导出模式 (isExporting)                       │   │
│  │     ├─ 是 → 跳过缓存，直接生成                            │   │
│  │     └─ 否 → 继续检查缓存                                  │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  2. 检查 ShapeCache (WeakMap)                            │   │
│  │     Key: element 对象引用                                 │   │
│  │     条件: cached && cached.theme === currentTheme        │   │
│  │     ├─ 命中 → 返回缓存的 shape                           │   │
│  │     └─ 未命中 → 继续生成                                  │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  3. 调用 _generateElementShape()                        │   │
│  │     使用 element.seed 作为 roughjs 的随机种子            │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  4. 非导出模式下写入 ShapeCache                          │   │
│  │     { shape, theme } → WeakMap.set(element, ...)        │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              generateElementWithCanvas()                        │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  1. 检查 elementWithCanvasCache (WeakMap)               │   │
│  │     Key: element 对象引用                                 │   │
│  │     条件:                                              │   │
│  │       - zoomValue 未变化 (或 shouldCacheIgnoreZoom)     │   │
│  │       - theme 未变化                                    │   │
│  │       - boundTextElementVersion 未变化                  │   │
│  │       - imageCrop 未变化                                │   │
│  │       - containingFrameOpacity 未变化                   │   │
│  │       - 带标签箭头的 angle 未变化                        │   │
│  │     ├─ 命中 → 返回缓存的 canvas                         │   │
│  │     └─ 未命中 → 继续生成                                 │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  2. 调用 generateElementCanvas()                        │   │
│  │     创建离屏 Canvas，调用 drawElementOnCanvas()          │   │
│  │     drawElementOnCanvas 内部使用 ShapeCache            │   │
│  ├─────────────────────────────────────────────────────────┤   │
│  │  3. 写入 elementWithCanvasCache                         │   │
│  │     { element, canvas, theme, scale, ... } → WeakMap   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│              drawElementFromCanvas()                            │
│  从缓存的 canvas 复制到主 canvas，应用旋转/缩放/位移              │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键设计决策

### 6.1 为什么用 WeakMap 而不是普通 Map？

| 特性 | WeakMap | Map |
|-----|---------|-----|
| Key 类型 | 只能是对象 | 任意类型 |
| 垃圾回收 | 不阻止 Key 被回收 | 阻止 Key 被回收 |
| 可迭代 | 不可迭代 | 可迭代 |

**Excalidraw 的选择理由**：
1. 元素对象被删除时，缓存自动释放，无需手动清理
2. 避免内存泄漏（画布上可能有成千上万个元素）
3. 缓存生命周期与元素对象生命周期绑定

### 6.2 为什么导出时不使用缓存？

1. **导出完整性**：确保导出的 SVG/PNG 包含最新的所有属性
2. **导出规模**：导出时的渲染配置（如 exportScale）可能与实时渲染不同
3. **可靠性**：避免缓存中的潜在问题影响导出结果

### 6.3 为什么复制元素要重新生成 seed？

1. **手绘真实感**：手绘作品中，两个相同的图形应该有细微差异
2. **避免单调感**：如果所有复制的图形完全一样，会显得机械
3. **可预测性**：用户明确知道"粘贴"会创建一个新的、略有不同的图形

---

## 7. 性能优化要点

1. **离屏 Canvas 缓存**：`elementWithCanvasCache` 将每个元素预渲染到离屏 Canvas，避免每次重绘都调用 roughjs
2. **增量失效**：只有变化的属性才触发缓存删除，而非全量重建
3. **主题感知**：缓存包含 theme 信息，亮色/暗色模式各自独立缓存
4. **缩放优化**：`shouldCacheIgnoreZoom` 模式下，缩放变化不触发重绘，通过 CSS 缩放实现平滑过渡

---

## 8. 文件位置速查

| 功能 | 文件路径 | 关键行号 |
|-----|---------|---------|
| ShapeCache 类 | `packages/element/src/shape.ts` | 81-164 |
| 随机种子生成 | `packages/common/src/random.ts` | 1-16 |
| 元素 seed 定义 | `packages/element/src/types.ts` | 55-57 |
| 种子传递给 roughjs | `packages/element/src/shape.ts` | 193-264 |
| 新元素创建 | `packages/element/src/newElement.ts` | 145 |
| 元素修改缓存失效 | `packages/element/src/mutateElement.ts` | 130-137 |
| Canvas 缓存 | `packages/element/src/renderElement.ts` | 603-663 |
| Canvas 缓存命中判断 | `packages/element/src/renderElement.ts` | 631-645 |
