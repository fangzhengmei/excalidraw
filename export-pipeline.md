# Excalidraw 导出管道分析报告

## 概述

Excalidraw 的导出系统是一个精心设计的模块化架构，支持 PNG、SVG 两种主要导出格式，以及剪贴板复制功能。在暗色主题下，系统会自动对颜色进行反色处理，确保导出内容在不同主题下的视觉一致性。

---

## 1. 渲染器复用机制

导出管道的核心设计理念是**渲染逻辑复用**，确保编辑器内渲染与导出渲染的一致性。

### 1.1 渲染器层级架构

```
┌─────────────────────────────────────────────────────────┐
│                  导出入口层 (packages/utils/src)        │
│     exportToCanvas()  |  exportToSvg()  |  exportToBlob()│
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              场景渲染层 (packages/excalidraw/scene)      │
│              export.ts  -  导出准备与协调               │
└─────────────────────┬───────────────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        ▼                           ▼
┌───────────────────┐     ┌───────────────────┐
│  Canvas 渲染器    │     │   SVG 渲染器      │
│  staticScene.ts   │     │ staticSvgScene.ts │
└───────────────────┘     └───────────────────┘
        │                           │
        └─────────────┬─────────────┘
                      ▼
              ┌───────────────────┐
              │  元素形状生成    │
              │  ShapeCache       │
              └───────────────────┘
```

### 1.2 形状缓存的实际行为

**ShapeCache 核心机制（shape.ts:81-164）**

```typescript
public static generateElementShape = (element, renderConfig) => {
  // ⚠️ 导出时总是重新生成，绕过缓存读取
  const cachedShape = renderConfig?.isExporting
    ? undefined
    : ShapeCache.get(element, renderConfig?.theme || null);

  if (cachedShape !== undefined) {
    return cachedShape;
  }

  const shape = _generateElementShape(element, ...);

  // ⚠️ 导出时不写入缓存
  if (!renderConfig?.isExporting) {
    ShapeCache.cache.set(element, { shape, theme: renderConfig?.theme });
  }

  return shape;
};
```

**导出阶段缓存行为分析**

| 场景 | 缓存读取 | 缓存写入 | 行为模式 |
|-----|---------|---------|---------|
| 编辑器渲染 (isExporting=false) | ✅ 优先读取缓存 | ✅ 生成后写入 | 正常缓存复用 |
| PNG 导出 (isExporting=true) | ❌ 强制绕过 | ❌ 不写入缓存 | 每次完整重算 |
| SVG 导出 (isExporting=true) | ❌ 强制绕过 | ❌ 不写入缓存 | 每次完整重算 |

**重算触发原因（代码注释）**

> "when exporting, always regenerated to guarantee the latest shape"

设计意图：
1. 规避编辑器交互过程中可能出现的缓存与实际状态不一致问题
2. 确保导出结果反映元素的最新属性（位置、颜色、样式等）
3. 导出通常是低频操作，性能开销可接受

**渲染配置标准化**

- `StaticCanvasRenderConfig` 和 `SVGRenderConfig` 共享相同的主题配置接口
- 两种渲染器使用相同的 `THEME` 枚举和 `applyDarkModeFilter` 函数
- 坐标变换逻辑（平移、旋转、缩放）在两个渲染器中保持一致

**字体处理统一**

- 导出前统一通过 `Fonts.loadElementsFonts()` 加载字体
- SVG 导出通过 `Fonts.generateFontFaceDeclarations()` 内联字体声明
- Canvas 渲染依赖浏览器字体加载完成后的测量结果

---

## 2. 暗色主题反色策略

### 2.1 颜色变换算法

反色处理不是简单的 RGB 取反，而是采用了与 CSS `filter: invert() hue-rotate()` 等效的两步变换：

```typescript
// packages/common/src/colors.ts:83-110
export const applyDarkModeFilter = (color: string): string => {
  // 步骤1: 93% 反色（不完全反色，保留一些对比度）
  const inverted = cssInvert(rgb.r, rgb.g, rgb.b, 93);
  
  // 步骤2: 色相旋转 180 度
  const rotated = cssHueRotate(inverted.r, inverted.g, inverted.b, 180);
  
  return rgbToHex(rotated.r, rotated.g, rotated.b, alpha);
};
```

### 2.2 颜色反色的数学原理

**cssInvert 函数**
```typescript
// 反色公式：inverted = color * (1 - p) + (255 - color) * p
// p = 0.93 表示 93% 反色程度
```

使用 93% 而非 100% 的原因：
- 100% 反色会导致纯黑变白、纯白变黑，中间灰度正好反转
- 93% 保留轻微的原色倾向，避免过渡刺眼
- 视觉上更接近真实的暗色主题体验

**cssHueRotate 函数**
- 使用标准的 3x3 色相旋转矩阵
- 180 度旋转将颜色映射到色轮的对面
- 补偿反色操作带来的色相偏移

### 2.3 应用时机与范围

反色处理在渲染管道的多个层级中应用：

| 层级 | 应用位置 | 处理内容 |
|------|---------|---------|
| 元素级 | `renderElementToSvg()` / `renderElement()` | 元素描边色、填充色 |
| 背景级 | `exportToSvg()` / `exportToCanvas()` | 画布背景色 |
| 系统级 | `staticScene.ts` | 网格线颜色、框架边框色 |

**SVG 导出中的应用示例**
```typescript
// staticSvgScene.ts:389-392
path.setAttribute(
  "fill",
  renderConfig.theme === THEME.DARK
    ? applyDarkModeFilter(element.strokeColor)
    : element.strokeColor,
);
```

**Canvas 导出中的应用示例**
```typescript
// exportToCanvas 渲染配置传递
renderConfig: {
  theme: appState.exportWithDarkMode ? THEME.DARK : THEME.LIGHT,
  // ...
}
```

### 2.4 性能优化：颜色缓存

为避免重复计算相同颜色的反色值，系统实现了缓存机制：

```typescript
const DARK_MODE_COLORS_CACHE: Map<string, string> | null =
  typeof window !== "undefined" ? new Map() : null;
```

- 缓存 key 是原始颜色字符串
- 缓存 value 是计算后的暗色主题颜色
- 仅在浏览器环境启用，避免服务器内存泄漏
- 大幅提升包含大量相同颜色元素的导出性能

---

## 3. 剪贴板格式协商

### 3.1 剪贴板操作的三种格式

Excalidraw 剪贴板系统支持三种数据格式，根据操作类型和浏览器支持情况自动选择：

| 格式类型 | MIME 类型 | 适用场景 | 数据内容 |
|---------|-----------|---------|---------|
| PNG 图像 | `image/png` | 复制为图片 | Canvas 渲染的二进制 Blob |
| SVG 文本 | `image/svg+xml` | 复制为矢量图 | SVG DOM 序列化字符串 |
| JSON 数据 | `text/plain` + 自定义 | 跨画板复制 | 完整元素数据 + 文件引用 |

### 3.2 导出到剪贴板的决策流程

```typescript
// packages/utils/src/export.ts:199-216
export const exportToClipboard = async (opts) => {
  switch (opts.type) {
    case "svg":
      const svg = await exportToSvg(opts);
      await copyTextToSystemClipboard(svg.outerHTML);
      break;
    case "png":
      await copyBlobToClipboardAsPng(exportToBlob(opts));
      break;
    case "json":
      await copyToClipboard(opts.elements, opts.files);
      break;
  }
};
```

### 3.3 PNG 剪贴板的兼容性处理

由于 `ClipboardItem` API 在不同浏览器中的实现差异，系统设计了双重 Fallback 机制：

```typescript
// packages/excalidraw/clipboard.ts:556-584
export const copyBlobToClipboardAsPng = async (blob: Blob | Promise<Blob>) => {
  try {
    // 方法1: 使用 Promise 作为 ClipboardItem 数据源
    // Safari 要求同步构造 ClipboardItem
    await navigator.clipboard.write([
      new ClipboardItem({ [MIME_TYPES.png]: blob }),
    ]);
  } catch (error) {
    // 方法2: 先 resolve Promise，再构造 ClipboardItem
    // Firefox 不支持 Promise 数据源
    if (isPromiseLike(blob)) {
      await navigator.clipboard.write([
        new ClipboardItem({ [MIME_TYPES.png]: await blob }),
      ]);
    } else {
      throw error;
    }
  }
};
```

### 3.4 文本剪贴板的三级 Fallback 策略

```typescript
// packages/excalidraw/clipboard.ts:586-636
export const copyTextToSystemClipboard = async (text, clipboardEvent?) => {
  // 1. 优先使用 clipboardEvent（来自用户交互事件）
  // 兼容性最好，支持多种 MIME 类型
  if (clipboardEvent) {
    clipboardEvent.clipboardData?.setData(mimeType, value);
    return;
  }

  // 2. 使用 navigator.clipboard.writeText API
  // 需要 HTTPS 和页面聚焦，支持异步操作
  if (probablySupportsClipboardWriteText) {
    try {
      await navigator.clipboard.writeText(text);
      return;
    } catch (error) {
      console.error(error);
    }
  }

  // 3. Fallback 到 document.execCommand('copy')
  // 兼容性最广，但需要临时 DOM 元素注入
  if (!copyTextViaExecCommand(text)) {
    throw new Error("Error copying to clipboard.");
  }
};
```

### 3.5 JSON 数据的多格式写入

当复制元素数据时，系统同时写入两种格式：

```typescript
// packages/excalidraw/clipboard.ts:194-209
export const copyToClipboard = async (elements, files, clipboardEvent?) => {
  const json = serializeAsClipboardJSON({ elements, files });

  await copyTextToSystemClipboard(
    {
      // 自定义 MIME 类型，供 Excalidraw 内部识别
      [MIME_TYPES.excalidrawClipboard]: json,
      // text/plain 作为 Fallback，支持粘贴到其他应用
      [MIME_TYPES.text]: json,
    },
    clipboardEvent,
  );
};
```

**剪贴板数据结构**
```typescript
type ElementsClipboard = {
  type: "excalidraw/clipboard";       // 类型标识
  elements: readonly ExcalidrawElement[];  // 元素数组
  files: BinaryFiles | undefined;     // 关联的图片文件
};
```

---

## 4. 导出流程时序图

### 4.1 PNG 导出流程

```
用户操作
    ↓
exportToClipboard({ type: "png" })
    ↓
exportToBlob()
    ├─> 恢复元素状态 (restoreElements)
    ├─> exportToCanvas()
    │    ├─> 加载字体 (Fonts.loadElementsFonts)
    │    ├─> 准备元素 (prepareElementsForRender)
    │    ├─> 计算边界与尺寸 (getCanvasSize)
    │    ├─> 创建 Canvas 并设置缩放
    │    └─> renderStaticScene()
    │         ├─> 渲染背景
    │         ├─> 应用暗色主题滤镜 (如需要)
    │         ├─> ⚠️ 遍历元素，每个元素完整重算形状
    │         │    (isExporting=true 绕过 ShapeCache)
    │         └─> 绘制元素
    └─> canvas.toBlob()
         └─> (可选) 嵌入场景数据到 PNG metadata
    ↓
copyBlobToClipboardAsPng()
    ├─> 尝试 ClipboardItem + Promise
    └─> Fallback 到 resolved Blob
    ↓
写入系统剪贴板
```

### 4.2 SVG 导出流程

```
用户操作
    ↓
exportToClipboard({ type: "svg" })
    ↓
exportToSvg()
    ├─> 恢复元素状态
    ├─> 准备元素（添加框架标签）
    ├─> 计算边界与偏移
    ├─> 创建 SVG 根元素
    ├─> 嵌入字体声明到 <defs>
    ├─> 创建框架裁剪路径
    ├─> 渲染背景（应用反色如需要）
    └─> renderSceneToSvg()
         └─> 遍历调用 renderElementToSvg()
              ├─> ⚠️ 每个元素完整重算形状
              │    (isExporting=true 绕过 ShapeCache)
              ├─> 对每个颜色应用 applyDarkModeFilter
              ├─> 使用 Rough.js 绘制手绘风格
              └─> 处理框架裁剪、链接、图片嵌入
    ↓
copyTextToSystemClipboard(svg.outerHTML)
    ↓
写入系统剪贴板
```

---

## 5. 关键设计决策总结

| 决策点 | 选择方案 | 权衡考量 |
|-------|---------|---------|
| 反色算法 | invert(93%) + hue-rotate(180°) | 视觉舒适度 vs 精确反转 |
| **形状缓存** | **导出时强制绕过，编辑器内正常复用** | **导出结果准确性 vs 导出性能** |
| 剪贴板策略 | 三级 Fallback 机制 | 兼容性 vs 功能完整性 |
| 颜色缓存 | Map<string, string> 浏览器端缓存 | 性能 vs 内存占用 |
| 主题切换 | 渲染时动态反色 | 实现简单 vs 导出性能 |

## 6. 性能优化要点

1. **颜色缓存**：避免重复计算相同颜色的暗色模式值
2. **编辑器内形状缓存**：交互渲染时复用 ShapeCache，提升流畅度
3. **导出时重算保证准确性**：虽牺牲性能，但确保导出结果与编辑器一致
4. **图片去重**：SVG 导出中使用 `<symbol>` 复用相同图片
5. **字体预加载**：导出前确保所有字体加载完成
6. **边界计算优化**：一次性计算所有元素的公共边界

---

## 文件索引

| 功能模块 | 文件路径 |
|---------|---------|
| 导出核心 | `packages/excalidraw/scene/export.ts` |
| SVG 渲染器 | `packages/excalidraw/renderer/staticSvgScene.ts` |
| Canvas 渲染器 | `packages/excalidraw/renderer/staticScene.ts` |
| 剪贴板处理 | `packages/excalidraw/clipboard.ts` |
| 导出工具函数 | `packages/utils/src/export.ts` |
| 颜色反色算法 | `packages/common/src/colors.ts` |
| 形状缓存实现 | `packages/element/src/shape.ts` |
