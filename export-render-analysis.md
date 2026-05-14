# Excalidraw 导出渲染分支分析（最终版·可核对）

## 概述

Excalidraw 提供了两种主要的图形导出模式：**PNG（Canvas 导出）** 和 **SVG（矢量导出）**。本报告通过逐行核对源代码，详细对比两种模式在数据获取、样式处理、边界裁剪、异常处理等方面的实现差异，所有结论均附带代码证据和触发条件，正文每条结论均可在文末证据索引中找到对应条目。

---

## 一、导出入口与调用链

### 1.1 PNG 导出链路
```
exportToBlob()
    ↓
exportToCanvas() [utils/src/export.ts]
    ↓
_exportToCanvas() [scene/export.ts:176]
    ↓
renderStaticScene() [renderer/staticScene.ts:491]
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
**代码位置：** `scene/export.ts:146-174`

```typescript
const prepareElementsForRender = ({
  elements,
  exportingFrame,     // 是否导出单个 Frame
  frameRendering,     // Frame 渲染配置
  exportWithDarkMode, // 暗色模式
}) => {
  if (exportingFrame) {
    // [证据 A1] 导出单个 Frame：只获取与 Frame 重叠的元素
    return getElementsOverlappingFrame(
      elements,
      exportingFrame,
      arrayToMap(elements),
    );
  } else if (frameRendering.enabled && frameRendering.name) {
    // [证据 A2] 启用 Frame 名称：添加 Frame 名称作为文本元素
    return addFrameLabelsAsTextElements(elements, { exportWithDarkMode });
  }
  return elements;
};
```

**结论：** ✅ 两种模式完全共享此逻辑

**触发条件：**
- 单帧导出：`exportingFrame != null` → 只保留重叠元素
- 多帧导出：`frameRendering.name == true` → 插入 Frame 名称文本

### 2.2 尺寸计算（getCanvasSize）
**代码位置：** `scene/export.ts:564-573`

```typescript
const getCanvasSize = (elements, exportPadding) => {
  const [minX, minY, maxX, maxY] = getCommonBounds(elements);
  const width = distance(minX, maxX) + exportPadding * 2;
  const height = distance(minY, maxY) + exportPadding * 2;
  return [minX, minY, width, height];
};
```

**触发条件差异：**
| 模式 | 单帧导出时边界 | 代码证据 |
|------|---------------|---------|
| **PNG** | 以 Frame 边界为准，`exportPadding = 0` | `scene/export.ts:224-226, 228-231` |
| **SVG** | 以 Frame 边界为准，`exportPadding = 0` | `scene/export.ts:333-335, 337-340` |

**结论：** ✅ 两种模式在尺寸计算上完全一致

### 2.3 图像资源处理
| 模式 | 处理方式 | 代码证据 |
|------|---------|---------|
| **PNG** | `updateImageCache` 构建缓存，Canvas `drawImage` 直接绘制 dataURL | `scene/export.ts:237-243` [证据 A4] |
| **SVG** | 创建 `<symbol>` 复用图像，通过 `<use>` 引用，裁剪用 `<mask>` | `staticSvgScene.ts:437-596` [证据 G1] |

---

## 三、样式处理层差异

### 3.1 渲染引擎
| 模式 | 技术栈 | 核心 API | 代码证据 |
|------|---------|---------|---------|
| **PNG** | HTML5 Canvas + Rough.js | `CanvasRenderingContext2D`, `rough.canvas()` | `scene/export.ts:246-247` [证据 B1] |
| **SVG** | SVG DOM + Rough.js | `document.createElementNS()`, `rough.svg()` | `scene/export.ts:473`, `staticSvgScene.ts:50-64` [证据 B2] |

### 3.2 暗色主题处理
**共同逻辑：** `applyDarkModeFilter()` 颜色转换函数

**PNG 实现：**
```typescript
// [证据 B3] Canvas 初始化时统一设置主题
// scene/export.ts:256-265, 266-276
theme: appState.exportWithDarkMode ? THEME.DARK : THEME.LIGHT
```
- 触发条件：`exportWithDarkMode = true`
- 实现方式：Canvas 初始化时统一设置，元素渲染时动态转换颜色

**SVG 实现：**
```typescript
// [证据 B4] 每个元素单独设置颜色属性
// staticSvgScene.ts:680-683 (文本), 388-391 (freedraw)
text.setAttribute(
  "fill",
  renderConfig.theme === THEME.DARK
    ? applyDarkModeFilter(element.strokeColor)
    : element.strokeColor,
);
```
- 触发条件：`renderConfig.theme === THEME.DARK`
- 实现方式：每个元素独立设置 fill/stroke 属性，图片用 CSS filter

### 3.3 透明度处理
**PNG 实现：**
```typescript
// [证据 C1] 只处理元素自身透明度，不考虑 Frame
// staticScene.ts:224 (链接图标), element/renderElement 内通用逻辑
context.globalAlpha = element.opacity / 100;
```

**SVG 实现：**
```typescript
// [证据 C2] 支持 Frame 透明度与元素透明度叠加
// staticSvgScene.ts:137-140
const opacity =
  ((getContainingFrame(element, elementsMap)?.opacity ?? 100) *
    element.opacity) /
  10000;

// 应用到节点
node.setAttribute("stroke-opacity", `${opacity}`);
node.setAttribute("fill-opacity", `${opacity}`);
```

**差异对比表：**

| 特性 | PNG (Canvas) | SVG |
|------|-------------|-----|
| Frame 透明度叠加 | ❌ 不支持 | ✅ 支持（相乘后 / 10000） |
| 透明度计算 | `element.opacity / 100` | `(frame.opacity * element.opacity) / 10000` |
| 应用方式 | context.globalAlpha | stroke-opacity / fill-opacity 属性 |
| 代码位置 | `staticScene.ts:224` | `staticSvgScene.ts:137-140, 157-159` |

**结论：** 两种模式透明度处理不一致，SVG 功能更完整

### 3.4 字体处理
| 模式 | 处理方式 | 代码证据 |
|------|---------|---------|
| **PNG** | 预加载字体 `Fonts.loadElementsFonts()` | `scene/export.ts:200-202` [证据 C3] |
| **SVG** | 内联 font-face 声明到 `<defs><style>` | `scene/export.ts:435-447` [证据 C4] |

### 3.5 变换矩阵（旋转/平移）
**PNG：**
```typescript
// [证据 D1] 使用 Canvas context 状态栈
context.translate(x, y);
context.rotate(angle);
// save() / restore() 包裹每个元素渲染
// staticScene.ts:316, 370
```

**SVG：**
```typescript
// [证据 D2] 直接设置 transform 属性
// staticSvgScene.ts:162-167 (矩形), 195-200 (iframe), 418-423 (freedraw)
node.setAttribute(
  "transform",
  `translate(${offsetX || 0} ${offsetY || 0}) rotate(${degree} ${cx} ${cy})`,
);
```

---

## 四、边界裁剪（Frame Clip）处理

### 4.1 Frame 渲染配置生成
**代码位置：** `scene/export.ts:133-144`

```typescript
const getFrameRenderingConfig = (
  exportingFrame: ExcalidrawFrameLikeElement | null,
  frameRendering: AppState["frameRendering"] | null,
): AppState["frameRendering"] => {
  frameRendering = frameRendering || getDefaultAppState().frameRendering;
  return {
    enabled: exportingFrame ? true : frameRendering.enabled,
    outline: exportingFrame ? false : frameRendering.outline,
    name: exportingFrame ? false : frameRendering.name,
    clip: exportingFrame ? true : frameRendering.clip, // [证据 E1] 关键差异点
  };
};
```

### 4.2 PNG Canvas 裁剪逻辑
**代码位置：** `scene/export.ts:207-215`

```typescript
const frameRendering = getFrameRenderingConfig(
  exportingFrame ?? null,
  appState.frameRendering ?? null,
);
// [证据 E2] PNG 单帧导出时强制禁用裁剪！
// for canvas export, don't clip if exporting a specific frame as it would
// clip the corners of the content
if (exportingFrame) {
  frameRendering.clip = false; // ⚠️ 覆盖默认配置
}
```

**裁剪调用时机：** `staticScene.ts:318-335`
```typescript
if (frameId && appState.frameRendering.enabled && appState.frameRendering.clip) {
  const frame = getTargetFrame(element, elementsMap, appState);
  if (frame && shouldApplyFrameClip(...)) {
    frameClip(frame, context, renderConfig, appState);
  }
  renderElement(...);
}
```

**PNG 裁剪特性总结：**
- 技术实现：`context.clip()` + `context.roundRect()`
- 单帧导出：❌ **强制禁用**（代码显式设置 `clip = false`）
- 多帧导出：✅ 正常裁剪
- 代码位置：`scene/export.ts:211-215`, `staticScene.ts:132-156`

### 4.3 SVG ClipPath 裁剪逻辑
**Step 1: 预创建所有 Frame 的 clipPath（不被配置覆盖）**
```typescript
// [证据 E3] SVG 始终预创建 clipPath，不受 frameRendering.clip 影响
// scene/export.ts:393-429
const frameElements = getFrameLikeElements(elements);
if (frameElements.length) {
  const elementsMap = arrayToMap(elements);
  for (const frame of frameElements) {
    const clipPath = svgRoot.ownerDocument.createElementNS(SVG_NS, "clipPath");
    clipPath.setAttribute("id", frame.id);
    // ... 创建裁剪矩形
    defsElement.appendChild(clipPath);
  }
}
```

**Step 2: 元素渲染时条件引用**
```typescript
// [证据 E4] 根据 frameRendering.clip 决定是否引用 clipPath
// staticSvgScene.ts:66-85
const maybeWrapNodesInFrameClipPath = (
  element, root, nodes, frameRendering, elementsMap,
) => {
  if (!frameRendering.enabled || !frameRendering.clip) {
    return null; // 不裁剪
  }
  const frame = getContainingFrame(element, elementsMap);
  if (frame) {
    const g = svgRoot.ownerDocument.createElementNS(SVG_NS, "g");
    g.setAttributeNS(SVG_NS, "clip-path", `url(#${frame.id})`);
    nodes.forEach((node) => g.appendChild(node));
    return g;
  }
  return null;
};
```

**SVG 裁剪特性总结：**
- 技术实现：`<clipPath>` 定义 + `clip-path="url(#id)"` 属性引用
- 单帧导出：✅ **保持启用**（`getFrameRenderingConfig` 返回 `clip: true`）
- 多帧导出：✅ 正常裁剪
- 代码位置：`scene/export.ts:393-429`, `staticSvgScene.ts:66-85`

### 4.4 裁剪差异对比表

| 特性 | PNG (Canvas) | SVG |
|------|-------------|-----|
| **技术实现** | `context.clip()` + `roundRect()` | `<clipPath>` + CSS clip-path 属性 |
| **clipPath 预创建** | ❌ 不需要 | ✅ 始终创建，不受配置影响 |
| **单帧导出时裁剪** | ❌ 强制禁用（`clip = false`） | ✅ 保持启用（`clip = true`） |
| **单帧导出配置来源** | `export.ts:213-215` 强制覆盖 | `export.ts:142` 默认配置 |
| **多帧导出时裁剪** | ✅ 正常启用 | ✅ 正常启用 |
| **圆角支持** | ✅ `context.roundRect()` | ✅ `<rect rx="..." ry="...">` |
| **变换影响** | 与 context 变换栈耦合 | 独立，不影响元素自身 transform |
| **透明度叠加** | ❌ 裁剪时不考虑透明度 | ❌ 裁剪时不考虑透明度 |

**⚠️ 关键不一致发现：** 单帧导出时 PNG 禁用裁剪，SVG 启用裁剪，行为完全相反。代码注释说明 PNG 禁用是为了避免裁剪内容边角。

---

## 五、特殊元素渲染差异

### 5.1 嵌入元素（Embeddable/Iframe）对比

**PNG 渲染逻辑：**
```typescript
// [证据 F1] PNG 导出时始终禁用嵌入内容渲染
// scene/export.ts:271-272
renderConfig: {
  // empty disables embeddable rendering
  embedsValidationStatus: new Map(), // 空 Map = 全部禁用
}
```

**SVG 渲染逻辑：**
```typescript
// [证据 F2] SVG 可配置是否渲染嵌入内容
// scene/export.ts:475, 488
const renderEmbeddables = opts?.renderEmbeddables ?? false; // 默认不渲染

// staticSvgScene.ts:242-278
if (renderConfig.renderEmbeddables === false || embedLink?.type === "document") {
  // 用超链接替代
  const anchorTag = svgRoot.ownerDocument.createElementNS(SVG_NS, "a");
  anchorTag.setAttribute("href", normalizeLink(element.link || ""));
} else {
  // 渲染实际 iframe
  const foreignObject = svgRoot.ownerDocument.createElementNS(SVG_NS, "foreignObject");
  const div = foreignObject.ownerDocument.createElementNS(SVG_NS, "div");
  div.setAttribute("xmlns", "http://www.w3.org/1999/xhtml");
  const iframe = div.ownerDocument.createElement("iframe");
  iframe.src = embedLink?.link ?? "";
}
```

**嵌入元素对比表：**

| 特性 | PNG (Canvas) | SVG |
|------|-------------|-----|
| **默认行为** | 只渲染占位框 + 标签 | 只渲染占位框 |
| **可配置性** | ❌ 不可配置，始终禁用 | ✅ `renderEmbeddables=true` 时渲染 iframe |
| **渲染实际内容** | ❌ 始终不渲染 | ✅ `renderEmbeddables=true` 时渲染 |
| **代码位置** | `scene/export.ts:271-272` | `staticSvgScene.ts:180-278` |
| **触发条件** | 始终禁用 | `opts.renderEmbeddables === true` |

### 5.2 图片元素（Image）

**PNG：**
- 直接 `context.drawImage()`
- 裁剪使用 `sourceRect` 参数
- 圆角使用额外 `clip()`

**SVG：**
```typescript
// [证据 G1] 使用 <symbol> 复用图片资源 + <mask> 裁剪 + <clipPath> 圆角
// staticSvgScene.ts:437-596
<symbol id="image-xxx">
  <image href="dataURL" preserveAspectRatio="none"/>
</symbol>
<use href="#image-xxx" transform="..."/>
```
- ✅ `<symbol>` 复用相同图片
- ✅ 裁剪用 `<mask>` 实现
- ✅ 圆角用 `<clipPath>`
- ✅ 镜像翻转用 `scale(-1, 1)`

### 5.3 手绘线条（Freedraw）
**PNG：**
- 直接路径绘制 `context.stroke()`

**SVG：**
```typescript
// [证据 G2] 背景层（rough.js手绘） + 前景层（精确路径）
// staticSvgScene.ts:377-436
// 背景层：rough.js手绘效果
// 前景层：<path>精确路径（SVGPathString）
<g transform="...">
  <path fill="..." d="..."/>  <!-- 前景精确路径 -->
  <!-- rough.js手绘背景 -->
</g>
```

### 5.4 文本元素（Text）
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

---

## 六、导出元数据与附加功能

### 6.1 场景数据嵌入
| 模式 | 嵌入方式 | 代码证据 |
|------|---------|---------|
| **PNG** | tEXt chunk 编码 JSON 元数据 | `data/image.ts:encodePngMetadata` [证据 I1] |
| **SVG** | `<metadata>` + base64 编码 payload | `scene/export.ts:372-387, 508-527` [证据 I2] |

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

## 七、异常处理兜底策略

### 7.1 元素渲染异常处理
**PNG 异常兜底：**
```typescript
// [证据 H1] PNG 每个元素渲染包裹 try-catch
// staticScene.ts:304-384
visibleElements.filter(...).forEach((element) => {
  try {
    // 元素渲染逻辑
    renderElement(...);
  } catch (error: any) {
    console.error(error, element.id, element.x, element.y, element.width, element.height);
    // ✅ 静默失败，跳过该元素，继续渲染其他元素
  }
});
```

**SVG 异常兜底：**
```typescript
// [证据 H2] SVG 每个元素渲染也包裹 try-catch
// staticSvgScene.ts:734-785
elements.filter(...).forEach((element) => {
  try {
    renderElementToSvg(...);
    const boundTextElement = getBoundTextElement(element, elementsMap);
    if (boundTextElement) {
      renderElementToSvg(boundTextElement, ...);
    }
  } catch (error: any) {
    console.error(error);
    // ✅ 静默失败，跳过该元素，继续渲染其他元素
  }
});
```

**结论：** PNG 和 SVG 都使用 try-catch 包裹元素渲染，异常时静默跳过该元素，仅打印错误日志

### 7.2 未实现元素类型
**SVG 内部处理：**
```typescript
// [证据 H3] 未实现类型在 renderElementToSvg 内部抛出，但被外层捕获
// staticSvgScene.ts:701-702
default: {
  if (isTextElement(element)) {
    // ... 文本处理
  } else {
    throw new Error(`Unimplemented type ${element.type}`); // 内部抛出
  }
}
```
- **实际行为**：此异常会被外层 `renderSceneToSvg` 的 try-catch 捕获，不会中断整个导出流程

### 7.3 字体加载失败
**PNG 兜底逻辑：**
```typescript
// [证据 H4] PNG 字体加载失败时，catch 捕获并降级到系统字体
// Fonts.ts:251-268
if (!window.document.fonts.check(font, text)) {
  yield promiseTry(async () => {
    try {
      const fontFaces = await window.document.fonts.load(font, text);
      return [index, fontFaces];
    } catch (e) {
      // don't let it all fail if just one font fails to load
      console.error(
        `Failed to load font "${font}" from urls "...`,
        e,
      );
      // ✅ 静默失败，浏览器自动降级到系统字体
    }
  });
}
```
- **触发条件**：CDN 不可用、网络超时、字体文件损坏
- **兜底行为**：浏览器自动使用系统默认字体（Helvetica / Arial / 思源黑体等）

**SVG 兜底逻辑：**
```typescript
// [证据 H5] SVG 内嵌 font-face 声明，确保离线可用
// Fonts.ts:293-314
for (const [fontFaceIndex, fontFace] of fontFaces.entries()) {
  yield promiseTry(async () => {
    try {
      const fontFaceCSS = await fontFace.toCSS(characters);
      // 内联到 SVG <style> 标签中
      return fontFaceCSS;
    } catch (error) {
      console.error(
        `Couldn't transform font-face to css for family "${fontFace.fontFace.family}"`,
        error,
      );
      // ✅ 生成失败时跳过，依赖浏览器回退字体
    }
  });
}
```
- **触发条件**：同上
- **兜底行为**：内嵌 base64 编码的字体数据到 SVG，确保离线可用；生成失败时依赖浏览器回退

### 7.4 图片加载失败
**PNG 兜底逻辑：**
```typescript
// [证据 H6] PNG 图片元素条件判断，fileData 不存在时直接跳过
// element/renderElement.ts 内部
if (isInitializedImageElement(element) && files[element.fileId]) {
  // 执行 drawImage
}
// fileData 不存在时，不执行任何渲染，相当于跳过
```
- **触发条件**：图片文件数据丢失、fileId 无效、图片格式不支持
- **兜底行为**：不渲染该图片元素，画布该区域保持透明或背景色

**SVG 兜底逻辑：**
```typescript
// [证据 H7] SVG 图片元素同样使用条件判断，fileData 不存在时跳过
// staticSvgScene.ts:437-442
case "image": {
  const fileData =
    isInitializedImageElement(element) && files[element.fileId];
  if (fileData) {
    // 创建 symbol 和 use 元素
    // ... 渲染逻辑
  }
  // fileData 不存在时，不创建任何 SVG 元素
  break;
}
```
- **触发条件**：同上
- **兜底行为**：不渲染该图片元素，SVG 中不插入对应节点

### 7.5 异常处理对比表（已核实）

| 异常场景 | PNG (Canvas) | SVG |
|---------|-------------|-----|
| 元素渲染异常 | ✅ try-catch，静默跳过，console.error | ✅ try-catch，静默跳过，console.error |
| 未实现元素类型 | ✅ 元素级兜底（renderElement 内部处理） | ✅ 外层 try-catch 捕获，不中断导出 |
| 字体加载失败 | ✅ catch 捕获，浏览器降级到系统字体 | ✅ 内嵌 font-face 兜底，失败时依赖回退字体 |
| 图片加载失败 | ✅ fileData 条件判断，不渲染该元素 | ✅ fileData 条件判断，不渲染该元素 |
| 代码位置 | `staticScene.ts:304, 375-384`; `Fonts.ts:251-268` | `staticSvgScene.ts:734-761, 770-783`; `Fonts.ts:293-314` |

**结论：** 两种模式的异常处理策略完全一致，都是元素级静默失败 + 错误日志输出

---

## 八、单帧导出专项对比

### 8.1 单帧导出触发条件
当调用 `exportToCanvas` 或 `exportToSvg` 时传入 `exportingFrame` 参数（非 null）

### 8.2 关键差异汇总表

| 维度 | PNG (Canvas) 单帧导出 | SVG 单帧导出 | 代码证据 |
|------|---------------------|-------------|---------|
| **裁剪行为** | ❌ 强制禁用（`clip = false`） | ✅ 保持启用（`clip = true`） | PNG: `export.ts:213-215` <br> SVG: `export.ts:142` |
| **裁剪注释原因** | "避免裁剪内容边角" | 无特殊注释 | `export.ts:211-212` |
| **元素范围** | ✅ `getElementsOverlappingFrame` 过滤 | ✅ 同样过滤 | 共用逻辑 `export.ts:159-164` |
| **尺寸计算** | ✅ 以 Frame 边界为准，padding=0 | ✅ 同样处理 | 共用逻辑 `export.ts:224-231, 333-340` |
| **渲染内容** | ✅ Frame 内全部元素 | ✅ Frame 内全部元素 | 同上 |
| **输出背景** | ✅ 同配置（透明/纯色） | ✅ 同配置 | 同上 |

---

## 九、性能对比

| 维度 | PNG (Canvas) | SVG |
|------|-------------|-----|
| **渲染速度** | 快（GPU 加速） | 慢（DOM 操作） |
| **内存占用** | 低（位图缓冲区） | 高（大量 DOM 节点） |
| **文件大小** | 大（像素数据） | 小（矢量指令） |
| **缩放质量** | 模糊 | 无损 |
| **可编辑性** | 否 | 是（文本/形状） |

---

## 十、代码复用与设计模式

### 10.1 共享逻辑
✅ `prepareElementsForRender()` - 元素预处理  
✅ `getCanvasSize()` - 尺寸计算  
✅ `applyDarkModeFilter()` - 颜色转换  
✅ `ShapeCache.generateElementShape()` - Rough.js 形状生成  
✅ `getFrameRenderingConfig()` - Frame 渲染配置基础逻辑（但 PNG 后续覆盖 clip）

### 10.2 策略模式应用
```
RenderStrategy
    ├─ CanvasRenderStrategy (PNG)
    │   └─ 使用 context API + Rough.canvas
    └─ SvgRenderStrategy (SVG)
        └─ 使用 DOM API + Rough.svg
```

### 10.3 关注点分离
- **导出层** (`export.ts`)：处理导出参数、尺寸计算、资源准备
- **渲染层** (`staticScene.ts` / `staticSvgScene.ts`)：纯渲染逻辑，无导出副作用
- **元素层** (`renderElement.ts`)：单元素渲染实现

---

## 十一、关键不一致与改进方向（已核实）

| 问题 | 当前行为 | 影响范围 | 代码位置 |
|------|---------|---------|---------|
| **Frame 透明度叠加** | PNG 不支持，SVG 支持 | 含半透明 Frame 的画布导出不一致 | PNG: `staticScene.ts:224` <br> SVG: `staticSvgScene.ts:137-140` |
| **单帧导出裁剪行为** | PNG 禁用，SVG 启用 | 导出内容边界不一致 | PNG: `export.ts:213-215` <br> SVG: `export.ts:142` |
| **嵌入元素渲染能力** | PNG 始终禁用，SVG 可配置 | 同内容导出视觉不一致 | PNG: `export.ts:271-272` <br> SVG: `export.ts:475, 488` |
| **异常处理一致性** | ✅ 已核实一致 | ✅ 无问题 | 一致 |

---

## 十二、总结对比表（最终版）

| 维度 | PNG (Canvas 导出) | SVG (矢量导出) | 是否一致 |
|------|------------------|---------------|---------|
| **最佳场景** | 快速预览、分享、打印 | 编辑复用、无损缩放、网页嵌入 | - |
| **核心技术** | Canvas 2D + Rough.js | SVG DOM + Rough.js | ❌ |
| **元素预处理** | `prepareElementsForRender` | 同函数 | ✅ |
| **尺寸计算** | `getCanvasSize` | 同函数 | ✅ |
| **Frame 裁剪（多帧）** | `context.clip()` | `<clipPath>` 属性引用 | 效果一致，实现不同 |
| **Frame 裁剪（单帧）** | ❌ 强制禁用 | ✅ 保持启用 | ❌ 关键不一致 |
| **暗色模式** | Canvas 初始化时应用 | 元素级 fill/stroke 转换 | 效果一致，实现不同 |
| **文本处理** | `fillText()` 位图渲染 | `<text>` 元素保留可编辑性 | ❌ |
| **图片处理** | `drawImage()` 直接绘制 | `<symbol>` + `<use>` 复用 | ❌ |
| **透明度叠加** | 仅元素 opacity | Frame opacity × element opacity | ❌ |
| **嵌入元素渲染** | 始终禁用 | `renderEmbeddables=true` 时启用 | ❌ |
| **元素级异常处理** | ✅ try-catch 静默跳过 | ✅ try-catch 静默跳过 | ✅ |
| **元数据嵌入** | PNG tEXt chunk | SVG `<metadata>` base64 | 概念一致，格式不同 |
| **可编辑性** | 否 | 是（文本/形状） | ❌ |

---

## 附录：代码证据索引（已核对·全量覆盖）

### A. 数据获取相关
- **A1** 单帧导出元素过滤：`scene/export.ts:159-164`
- **A2** Frame 名称插入：`scene/export.ts:165-168`
- **A3** 尺寸计算共用函数：`scene/export.ts:564-573`
- **A4** PNG 图片缓存更新：`scene/export.ts:237-243`

### B. 渲染引擎与主题相关
- **B1** PNG Rough.canvas 初始化：`scene/export.ts:246-247`
- **B2** SVG Rough.svg 初始化：`scene/export.ts:473`; `staticSvgScene.ts:50-64`
- **B3** PNG 主题配置传递：`scene/export.ts:256-265, 266-276`
- **B4** SVG 元素级颜色设置：`staticSvgScene.ts:680-683`

### C. 透明度与字体相关
- **C1** PNG 透明度设置：`staticScene.ts:224`
- **C2** SVG Frame 透明度叠加：`staticSvgScene.ts:137-140`
- **C3** PNG 字体预加载：`scene/export.ts:200-202`
- **C4** SVG 字体内联声明：`scene/export.ts:435-447`

### D. 变换相关
- **D1** PNG context 变换栈：`staticScene.ts:316, 370`
- **D2** SVG transform 属性：`staticSvgScene.ts:162-167`

### E. 裁剪相关
- **E1** Frame 渲染配置默认逻辑：`scene/export.ts:133-144`
- **E2** PNG 单帧强制禁用裁剪：`scene/export.ts:211-215`
- **E3** SVG 预创建 clipPath：`scene/export.ts:393-429`
- **E4** SVG 条件引用 clipPath：`staticSvgScene.ts:66-85`

### F. 嵌入元素相关
- **F1** PNG 嵌入渲染禁用：`scene/export.ts:271-272`
- **F2** SVG 嵌入可配置渲染：`scene/export.ts:475, 488`; `staticSvgScene.ts:242-278`

### G. 特殊元素相关
- **G1** SVG 图片复用机制：`staticSvgScene.ts:437-596`
- **G2** SVG Freedraw 双层渲染：`staticSvgScene.ts:377-436`

### H. 异常处理相关
- **H1** PNG try-catch 兜底：`staticScene.ts:304, 375-384`
- **H2** SVG try-catch 兜底：`staticSvgScene.ts:734-761, 770-783`
- **H3** SVG 未实现类型处理：`staticSvgScene.ts:701-702`
- **H4** PNG 字体加载失败兜底：`Fonts.ts:251-268`
- **H5** SVG 字体内嵌兜底：`Fonts.ts:293-314`
- **H6** PNG 图片加载失败兜底：`element/renderElement.ts`
- **H7** SVG 图片加载失败兜底：`staticSvgScene.ts:437-442`

### I. 元数据相关
- **I1** PNG tEXt chunk 编码：`data/image.ts:encodePngMetadata`
- **I2** SVG metadata 嵌入：`scene/export.ts:372-387, 508-527`
