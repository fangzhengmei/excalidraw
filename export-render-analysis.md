# Excalidraw 导出渲染分支分析

## 概述

Excalidraw 提供了两种主要的图形导出模式：**PNG（Canvas 导出）** 和 **SVG（矢量导出）**。两种模式在数据获取、样式处理、边界裁剪等方面存在显著差异。

---

## 一、导出入口与调用链

### 1.1 PNG 导出链路
```
exportToBlob()
    ↓
exportToCanvas() [utils/src/export.ts]
    ↓
_exportToCanvas() [scene/export.ts]
    ↓
renderStaticScene() [renderer/staticScene.ts]
    ↓
renderElement() [element/renderElement.ts]
```

### 1.2 SVG 导出链路
```
exportToSvg() [utils/src/export.ts]
    ↓
_exportToSvg() [scene/export.ts:289]
    ↓
renderSceneToSvg() [renderer/staticSvgScene.ts:708]
    ↓
renderElementToSvg() [renderer/staticSvgScene.ts:87]
```

---

## 二、数据获取层差异

### 2.1 元素预处理（prepareElementsForRender）
**位置：** `scene/export.ts:146-174`

两种导出模式共享相同的元素预处理逻辑：

```typescript
const prepareElementsForRender = ({
  elements,
  exportingFrame,     // 是否导出单个 Frame
  frameRendering,     // Frame 渲染配置
  exportWithDarkMode, // 暗色模式
}) => {
  if (exportingFrame) {
    // 导出单个 Frame：只获取与 Frame 重叠的元素
    return getElementsOverlappingFrame(elements, exportingFrame, arrayToMap(elements));
  } else if (frameRendering.enabled && frameRendering.name) {
    // 启用 Frame 名称：添加 Frame 名称作为文本元素
    return addFrameLabelsAsTextElements(elements, { exportWithDarkMode });
  }
  return elements;
};
```

**关键点：**
- ✅ 两种模式共享此逻辑
- ✅ Frame 名称通过创建临时文本元素实现
- ✅ 单个 Frame 导出时只导出重叠元素

### 2.2 尺寸计算（getCanvasSize）
**位置：** `scene/export.ts:564-573`

```typescript
const getCanvasSize = (elements, exportPadding) => {
  const [minX, minY, maxX, maxY] = getCommonBounds(elements);
  const width = distance(minX, maxX) + exportPadding * 2;
  const height = distance(minY, maxY) + exportPadding * 2;
  return [minX, minY, width, height];
};
```

**差异点：**
- **PNG：** `exportPadding` 默认为 `DEFAULT_EXPORT_PADDING`
- **SVG：** 同样使用，但导出单个 Frame 时 `exportPadding = 0`

### 2.3 图像资源处理
| 模式 | 处理方式 | 位置 |
|------|---------|------|
| **PNG** | 使用 `updateImageCache` 构建缓存，Canvas 直接绘制 dataURL | `scene/export.ts:237-243` |
| **SVG** | 创建 `<symbol>` 复用图像，通过 `<use>` 引用，支持裁剪 mask | `staticSvgScene.ts:437-596` |

---

## 三、样式处理层差异

### 3.1 渲染引擎
| 模式 | 技术栈 | 核心 API |
|------|---------|---------|
| **PNG** | HTML5 Canvas + Rough.js | `CanvasRenderingContext2D`, `rough.canvas()` |
| **SVG** | SVG DOM + Rough.js | `document.createElementNS()`, `rough.svg()` |

### 3.2 暗色主题处理
**共同逻辑：** `applyDarkModeFilter()` 颜色转换

**差异实现：**

**PNG 渲染：**
```typescript
// staticScene.ts:250-258
bootstrapCanvas({
  theme: appState.theme, // THEME.DARK / THEME.LIGHT
  isExporting,
  viewBackgroundColor: appState.viewBackgroundColor,
});
```
- 在 Canvas 初始化时应用主题
- 元素渲染时动态转换颜色

**SVG 渲染：**
```typescript
// staticSvgScene.ts:680-683
text.setAttribute(
  "fill",
  renderConfig.theme === THEME.DARK 
    ? applyDarkModeFilter(element.strokeColor) 
    : element.strokeColor
);
```
- 每个元素单独设置 fill/stroke 属性
- SVG 图片使用 CSS filter: `DARK_THEME_FILTER`

### 3.3 透明度处理
**PNG：**
```typescript
// 在 renderElement 内部处理
context.globalAlpha = element.opacity / 100;
```

**SVG：**
```typescript
// staticSvgScene.ts:137-140
const opacity = ((frame.opacity ?? 100) * element.opacity) / 10000;
node.setAttribute("stroke-opacity", `${opacity}`);
node.setAttribute("fill-opacity", `${opacity}`);
```
- ✅ SVG 支持 Frame 透明度叠加
- ❌ PNG 仅处理元素自身透明度

### 3.4 字体处理
| 模式 | 处理方式 |
|------|---------|
| **PNG** | 预加载字体 `Fonts.loadElementsFonts()` |
| **SVG** | 内联字体声明 `Fonts.generateFontFaceDeclarations()`，嵌入到 `<defs><style>` |

### 3.5 变换矩阵（旋转/平移）
**PNG：**
```typescript
context.translate(x, y);
context.rotate(angle);
```
- 使用 Canvas context 状态栈
- `save()` / `restore()` 包裹每个元素

**SVG：**
```typescript
node.setAttribute(
  "transform",
  `translate(${offsetX} ${offsetY}) rotate(${degree} ${cx} ${cy})`
);
```
- 直接设置 SVG `transform` 属性
- 中心点计算更复杂

---

## 四、边界裁剪（Frame Clip）处理

### 4.1 PNG Canvas 裁剪
**位置：** `staticScene.ts:132-156`

```typescript
export const frameClip = (frame, context, renderConfig, appState) => {
  context.translate(frame.x + appState.scrollX, frame.y + appState.scrollY);
  context.beginPath();
  if (context.roundRect) {
    context.roundRect(0, 0, frame.width, frame.height, radius);
  } else {
    context.rect(0, 0, frame.width, frame.height);
  }
  context.clip(); // ✨ 核心裁剪 API
  context.translate(-(frame.x + appState.scrollX), -(frame.y + appState.scrollY));
};
```

**调用时机：**
```typescript
// staticScene.ts:318-335
if (frameId && appState.frameRendering.enabled && appState.frameRendering.clip) {
  const frame = getTargetFrame(element, elementsMap, appState);
  if (frame && shouldApplyFrameClip(element, frame, appState, elementsMap, inFrameGroupsMap)) {
    frameClip(frame, context, renderConfig, appState);
  }
  renderElement(...);
}
```

**裁剪特性：**
- ✅ 使用 Canvas 原生 `clip()`
- ✅ 裁剪前 `save()`，裁剪后 `restore()`
- ✅ 支持圆角（`roundRect`）
- ✅ 每个元素渲染前判断是否需要裁剪
- ❌ 导出单个 Frame 时 **禁用** 裁剪（`scene/export.ts:213-215`）

### 4.2 SVG ClipPath 裁剪
**位置：** `staticSvgScene.ts:66-85, 393-429`

**Step 1: 预创建所有 Frame 的 clipPath**
```typescript
// scene/export.ts:393-429
for (const frame of frameElements) {
  const clipPath = svgRoot.ownerDocument.createElementNS(SVG_NS, "clipPath");
  clipPath.setAttribute("id", frame.id);
  
  const rect = svgRoot.ownerDocument.createElementNS(SVG_NS, "rect");
  rect.setAttribute("transform", `translate(${frame.x + offsetX} ${frame.y + offsetY}) rotate(...)`);
  rect.setAttribute("width", `${frame.width}`);
  rect.setAttribute("height", `${frame.height}`);
  rect.setAttribute("rx", `${FRAME_STYLE.radius}`);
  
  clipPath.appendChild(rect);
  defsElement.appendChild(clipPath);
}
```

**Step 2: 元素渲染时引用 clipPath**
```typescript
// staticSvgScene.ts:66-85
const maybeWrapNodesInFrameClipPath = (element, root, nodes, frameRendering, elementsMap) => {
  if (!frameRendering.enabled || !frameRendering.clip) {
    return null;
  }
  const frame = getContainingFrame(element, elementsMap);
  if (frame) {
    const g = root.ownerDocument.createElementNS(SVG_NS, "g");
    g.setAttributeNS(SVG_NS, "clip-path", `url(#${frame.id})`);
    nodes.forEach((node) => g.appendChild(node));
    return g;
  }
  return null;
};
```

**裁剪特性：**
- ✅ 使用 SVG `<clipPath>` 定义，`<g clip-path="url(#id)">` 引用
- ✅ 预创建所有 Frame 裁剪路径，复用率高
- ✅ 支持圆角（`rx`/`ry` 属性）
- ✅ 导出单个 Frame 时 **启用** 裁剪
- ✅ 通过 `<g>` 包装元素，不影响元素自身变换

### 4.3 Frame 裁剪差异对比

| 特性 | PNG (Canvas) | SVG |
|------|-------------|-----|
| **技术实现** | `context.clip()` | `<clipPath>` + `clip-path` 属性 |
| **作用时机** | 渲染每个元素前 | 预定义 + 渲染时引用 |
| **圆角支持** | `roundRect()`（浏览器依赖） | `rx`/`ry` 属性（标准） |
| **导出单个 Frame 时** | ❌ 禁用裁剪 | ✅ 启用裁剪 |
| **透明度叠加** | ❌ 仅元素 opacity | ✅ Frame + Element opacity |
| **变换影响** | 与 context 变换栈耦合 | 独立，不影响元素 transform |
| **性能** | 单次裁剪，速度快 | DOM 节点多，内存占用大 |

---

## 五、特殊元素渲染差异

### 5.1 图片元素（Image）
**PNG：**
- 直接 `context.drawImage()`
- 裁剪使用 `sourceRect` 参数
- 圆角使用额外 `clip()`

**SVG：**
```typescript
// staticSvgScene.ts:437-596
<symbol id="image-xxx">
  <image href="dataURL" preserveAspectRatio="none"/>
</symbol>
<use href="#image-xxx" transform="..."/>
```
- ✅ 使用 `<symbol>` 复用相同图片
- ✅ 裁剪使用 `<mask>` 实现
- ✅ 圆角使用 `<clipPath>`
- ✅ 镜像翻转使用 `scale(-1, 1)`

### 5.2 手绘线条（Freedraw）
**PNG：**
- 直接路径绘制 `context.stroke()`

**SVG：**
```typescript
// staticSvgScene.ts:377-436
// 背景层：rough.js 手绘效果
// 前景层：<path> 精确路径（SVGPathString）
<g transform="...">
  <path fill="..." d="..."/>  <!-- 前景精确路径 -->
  <!-- rough.js 手绘背景 -->
</g>
```

### 5.3 文本元素（Text）
**PNG：**
- `context.fillText()` 逐行绘制
- 依赖 Canvas 字体渲染

**SVG：**
```xml
<g transform="...">
  <text x="..." y="..." font-family="..." font-size="..." 
        fill="..." text-anchor="..." direction="...">line 1</text>
  <text ...>line 2</text>
</g>
```
- ✅ 每行一个 `<text>` 元素
- ✅ 支持 RTL（从右到左）文本
- ✅ 保留文本可编辑性

### 5.4 嵌入元素（Embeddable/Iframe）
**PNG：**
- 只渲染占位框 + 标签
- 不渲染实际 iframe 内容

**SVG：**
- 默认只渲染占位
- `renderEmbeddables=true` 时：
  ```xml
  <foreignObject>
    <div xmlns="http://www.w3.org/1999/xhtml">
      <iframe src="..."/>
    </div>
  </foreignObject>
  ```
- ❌ 文档类型嵌入替换为超链接（SVG 兼容性问题）

---

## 六、导出元数据与附加功能

### 6.1 场景数据嵌入
| 模式 | 嵌入方式 |
|------|---------|
| **PNG** | tEXt chunk 编码 JSON 元数据 |
| **SVG** | `<metadata>` + base64 编码 payload |

**SVG 嵌入结构：**
```xml
<svg>
  <metadata>
    <!-- payload-type:application/json -->
    <!-- payload-version:2 -->
    <!-- payload-start -->
    base64_encoded_json
    <!-- payload-end -->
  </metadata>
</svg>
```

### 6.2 超链接支持
**PNG：** 不保留链接（位图无交互）
**SVG：**
```xml
<a href="https://...">
  <!-- 元素节点 -->
</a>
```
- ✅ 元素级别超链接
- ✅ 区分外部链接 / 元素内部链接

---

## 七、边界情况兜底策略

### 7.1 元素渲染异常
```typescript
// staticSvgScene.ts:734-761
try {
  renderElementToSvg(...);
} catch (error) {
  console.error(error);
  // ✅ 静默失败，跳过该元素
}
```

### 7.2 未实现元素类型
```typescript
// staticSvgScene.ts:701-702
default: {
  if (isTextElement(element)) {
    // ...
  } else {
    throw new Error(`Unimplemented type ${element.type}`);
  }
}
```

### 7.3 字体加载失败
**PNG：** 浏览器自动降级到系统字体
**SVG：** 嵌入 font-face 声明，确保离线可用

### 7.4 图片加载失败
**PNG：** 不渲染，静默跳过
**SVG：** 不渲染，静默跳过

---

## 八、性能对比

| 维度 | PNG (Canvas) | SVG |
|------|-------------|-----|
| **渲染速度** | 快（GPU 加速） | 慢（DOM 操作） |
| **内存占用** | 低（位图缓冲区） | 高（大量 DOM 节点） |
| **文件大小** | 大（像素数据） | 小（矢量指令） |
| **缩放质量** | 模糊 | 无损 |
| **可编辑性** | 否 | 是（文本/形状） |

---

## 九、代码复用与设计模式

### 9.1 共享逻辑
✅ `prepareElementsForRender()` - 元素预处理  
✅ `getCanvasSize()` - 尺寸计算  
✅ `applyDarkModeFilter()` - 颜色转换  
✅ `ShapeCache.generateElementShape()` - Rough.js 形状生成  
✅ Frame 渲染配置逻辑

### 9.2 策略模式应用
```
RenderStrategy
    ├─ CanvasRenderStrategy (PNG)
    │   └─ 使用 context API + Rough.canvas
    └─ SvgRenderStrategy (SVG)
        └─ 使用 DOM API + Rough.svg
```

### 9.3 关注点分离
- **导出层** (`export.ts`)：处理导出参数、尺寸计算、资源准备
- **渲染层** (`staticScene.ts` / `staticSvgScene.ts`)：纯渲染逻辑，无导出副作用
- **元素层** (`renderElement.ts`)：单元素渲染实现

---

## 十、关键改进点（潜在优化方向）

1. **Frame 透明度叠加**：PNG 导出未考虑 Frame 透明度，与 SVG 行为不一致
2. **错误处理一致性**：PNG 异常元素静默跳过，SVG 抛出错误
3. **裁剪逻辑统一**：导出单个 Frame 时 PNG 禁用裁剪，SVG 启用裁剪，行为不一致
4. **图片复用策略**：SVG 的 `<symbol>` 复用策略可移植到 Canvas（精灵图）
5. **字体嵌入优化**：PNG 无法嵌入字体，导出后可能字体不一致

---

## 总结

| 维度 | PNG (Canvas 导出) | SVG (矢量导出) |
|------|------------------|---------------|
| **最佳场景** | 快速预览、分享、打印 | 编辑复用、无损缩放、网页嵌入 |
| **核心技术** | Canvas 2D + Rough.js | SVG DOM + Rough.js |
| **Frame 裁剪** | `context.clip()` | `<clipPath>` 属性引用 |
| **暗色模式** | Canvas 初始化时应用 | 元素级 fill/stroke 转换 |
| **文本处理** | `fillText()` 位图渲染 | `<text>` 元素保留可编辑性 |
| **图片处理** | `drawImage()` 直接绘制 | `<symbol>` + `<use>` 复用 |
| **元数据** | PNG tEXt chunk | SVG `<metadata>` base64 |
| **交互性** | 无 | 支持超链接 |

两种导出模式在**数据获取层**高度复用，但在**渲染引擎**、**样式应用**、**边界裁剪**等方面采用了完全不同的技术路径，这是由各自的输出格式特性决定的。
