# 剪贴板粘贴数据处理链路分析

## 一、整体架构概览

外部内容粘贴到 Excalidraw 画板后，会经过一条完整的**识别 → 解析 → 归一化 → 转换**链路。整个流程的入口是 `pasteFromClipboard` 方法，出口是将标准化的内部元素插入画布。

### 核心文件
| 文件 | 主要职责 |
|------|----------|
| `packages/excalidraw/clipboard.ts` | 剪贴板事件解析、数据类型识别、MIME 类型处理 |
| `packages/excalidraw/components/App.tsx` | 粘贴事件入口、内容分发、各类型处理逻辑 |
| `packages/excalidraw/data/restore.ts` | 元素归一化、数据修复、属性补全 |
| `packages/excalidraw/charts/charts.parse.ts` | 表格/电子表格数据解析 |
| `packages/excalidraw/mermaid.ts` | Mermaid 图表识别 |
| `packages/element/src/transform.ts` | 骨架元素转换为完整 Excalidraw 元素 |
| `packages/common/src/constants.ts` | MIME 类型定义、导出数据类型常量 |

---

## 二、完整处理链路

### 阶段 1：事件捕获与数据提取

**入口**：`App.tsx:3865` → `pasteFromClipboard`

```
用户按下 Ctrl+V → 触发 document paste 事件 → pasteFromClipboard()
```

#### 关键校验
1. **焦点校验**：检查 Excalidraw 容器是否为活动元素 (`App.tsx:3870-3875`)
2. **画布校验**：检查鼠标位置是否在画布上，且不在可输入元素中 (`App.tsx:3877-3887`)

#### 数据提取
**函数**：`clipboard.ts:466` → `parseDataTransferEvent()`

```typescript
// 同步提取剪贴板数据（必须在同一帧，否则 Firefox 会清空 clipboardData）
const dataTransferList = await parseDataTransferEvent(event);
```

该函数遍历 `DataTransferItemList`，将每个项目标准化为：
- `{ kind: "file", type: "image/png", file: File, fileHandle }`
- `{ kind: "string", type: "text/plain", value: string }`

同时会调用 `normalizeFile()` 对文件进行标准化处理。

---

### 阶段 2：数据类型识别与解析

**函数**：`clipboard.ts:522` → `parseClipboard(dataList, isPlainPaste)`

#### 2.1 支持的 MIME 类型
定义于 `constants.ts:273-277`：
```typescript
ALLOWED_PASTE_MIME_TYPES = [
  "text/plain",        // 纯文本
  "text/html",         // HTML 富文本
  "image/svg+xml",     // SVG 图片
  "image/png",         // PNG 图片
  "image/jpeg",        // JPEG 图片
  "image/gif",         // GIF 图片
  "image/webp",        // WebP 图片
  // ... 其他图片格式
]
```

#### 2.2 文本数据解析
**函数**：`clipboard.ts:330` → `parseClipboardEventTextData()`

**优先级**：
1. **HTML 解析** (`clipboard.ts:232-250`)：
   - 优先检查 `text/html` 类型
   - 使用 `DOMParser` 解析 HTML 树
   - `parseHTMLTree()` 递归遍历 DOM 节点，提取：
     - 文本节点 → `{ type: "text", value: string }`
     - 图片节点 (`<img src="http...">`) → `{ type: "imageUrl", value: url }`
   
2. **纯文本降级**：如果 HTML 解析失败或全为文本，回退到 `text/plain`

#### 2.3 Excalidraw 内部格式识别
**函数**：`clipboard.ts:73-87` → `clipboardContainsElements()`

检查 JSON 数据的 `type` 字段：
```typescript
// 可识别的内部类型
EXPORT_DATA_TYPES = {
  excalidraw: "excalidraw",                          // 普通导出
  excalidrawClipboard: "excalidraw/clipboard",       // 剪贴板导出
  excalidrawClipboardWithAPI: "excalidraw-api/clipboard"  // API 导出
}
```

如果匹配成功，直接返回 `{ elements, files, programmaticAPI }`。

---

### 阶段 3：内容分发与类型判断

**函数**：`App.tsx:3698` → `insertClipboardContent(data, filesList, isPlainPaste)`

> **关键机制**：内容按**优先级**依次匹配，匹配成功则提前返回。

#### 内容类型判断优先级（从上到下）

##### 1. 错误消息 (`App.tsx:3711-3715`)
```typescript
if (data.errorMessage) {
  this.setState({ errorMessage: data.errorMessage });
  return;
}
```

##### 2. 混合内容 (HTML 中的文本+图片) (`App.tsx:3717-3725`)
```typescript
if (dataTransferFiles.length === 0 && !isPlainPaste && data.mixedContent) {
  await this.addElementsFromMixedContentPaste(data.mixedContent, { ... });
  return;
}
```

##### 3. 电子表格数据 (`App.tsx:3727-3741`)
**函数**：`charts.parse.ts:131` → `tryParseSpreadsheet()`

**解析逻辑**：
- 尝试三种分隔符：`\t` (Tab) > `,` (CSV) > `;` (分号)
- 评分标准：列数一致 > 列数最多 > 分隔符优先级
- 数据验证：
  - 数字列必须全部可解析为数字
  - 至少 2 行数据
  - 1 列 / 2 列 / 多列格式分别处理

**支持的格式**：
| 列数 | 格式说明 |
|------|----------|
| 1 列 | 单列数据（标题 + 数值） |
| 2 列 | 标签 + 数值 |
| >2 列 | 标签 + 多系列数值（支持宽表转置） |

匹配成功后弹出图表选择对话框。

##### 4. 图片或 SVG 代码 (`App.tsx:3743-3762`)
```typescript
// 检测粘贴的 SVG 代码
if (trimmedText.startsWith("<svg") && trimmedText.endsWith("</svg>")) {
  imageFiles.push(SVGStringToFile(trimmedText));
}
```

支持：
- 剪贴板中的图片文件（PNG/JPG/SVG/GIF 等）
- 粘贴的 SVG 代码字符串

调用 `insertImages()` 插入图片元素。

##### 5. Excalidraw 元素 (`App.tsx:3764-3783`)
```typescript
if (data.elements) {
  const elements = data.programmaticAPI
    ? convertToExcalidrawElements(data.elements)  // API 骨架数据
    : data.elements;                              // 完整元素数据
  
  this.addElementsFromPasteOrLibrary({ elements, ... });
  return;
}
```

##### 6. Mermaid 图表 (`App.tsx:3790-3814`)
**函数**：`mermaid.ts:2` → `isMaybeMermaidDefinition()`

通过正则匹配开头的图表类型关键字：
```typescript
/^(?:%%{.*?}%%[\s\n]*)?\b(?:\s*flowchart(-beta)?|\s*graph(-beta)?|\s*sequenceDiagram|...)\b/
```

支持的图表类型：`flowchart`, `graph`, `sequenceDiagram`, `classDiagram`, `stateDiagram`, `erDiagram`, `gantt`, `pie`, `mindmap`, `timeline` 等 20+ 种。

匹配成功后调用 `@excalidraw/mermaid-to-excalidraw` 解析为元素骨架，再通过 `convertToExcalidrawElements()` 转换。

##### 7. 可嵌入 URL (`App.tsx:3816-3859`)
判断条件：
- 所有非空行都是有效的可嵌入 URL
- 支持 YouTube、Vimeo、Figma 等视频/嵌入链接
- 每行一个 URL，自动横向排列

##### 8. 纯文本 (`App.tsx:3861-3862`)
最后回退到 `addTextFromPaste()` 创建文本元素。

---

### 阶段 4：各类型内容专门处理

#### 4.1 纯文本处理 (`App.tsx:4123`) → `addTextFromPaste()`
- 按换行符分割为多行
- 长文本自动换行（最大宽度 `min(画布宽度*0.5, 800px)`，最小 200px）
- 多行文本纵向排列，行间距 10px
- 应用当前工具栏样式（颜色、字体、字号等）

#### 4.2 混合内容处理 (`App.tsx:4072`) → `addElementsFromMixedContentPaste()`
- 提取所有 `imageUrl` 类型，下载后作为图片插入
- 剩余文本合并后作为文本插入

#### 4.3 骨架元素转换 (`transform.ts:509`) → `convertToExcalidrawElements()`
将简化的元素骨架转换为完整 Excalidraw 元素：
```typescript
// 为每种元素类型补全默认属性
switch (element.type) {
  case "rectangle":
  case "ellipse":
  case "diamond":
    return newElement({ width, height, ...element });
  case "line":
    return newLinearElement({ points: [[0,0], [width,height]], ...element });
  case "arrow":
    return newArrowElement({ endArrowhead: "arrow", ...element });
  case "text":
    return newTextElement({ metrics, lineHeight, ...element });
  case "image":
    return newImageElement({ width, height, ...element });
}
```

---

### 阶段 5：元素归一化（最复杂的一步）

**函数**：`restore.ts:764` → `restoreElements(elements, existingElements, opts)`

> **设计意图**：确保粘贴/导入的元素符合当前版本的数据格式，修复损坏数据，补全缺失属性，保证数据一致性。

#### 5.1 核心归一化流程

##### 第一步：逐个元素归一化 (`restore.ts:413`) → `restoreElement()`

针对每种元素类型进行专门处理：

**通用属性归一化** (`restore.ts:327-411`) → `restoreElementWithProperties()`
```typescript
const base = {
  type: extra.type || element.type,
  version: element.version || 1,
  id: element.id || randomId(),
  fillStyle: element.fillStyle || DEFAULT_ELEMENT_PROPS.fillStyle,
  strokeWidth: element.strokeWidth || DEFAULT_ELEMENT_PROPS.strokeWidth,
  strokeStyle: element.strokeStyle ?? DEFAULT_ELEMENT_PROPS.strokeStyle,
  roughness: element.roughness ?? DEFAULT_ELEMENT_PROPS.roughness,
  opacity: element.opacity ?? DEFAULT_ELEMENT_PROPS.opacity,
  angle: element.angle || 0,
  x: extra.x ?? element.x ?? 0,
  y: extra.y ?? element.y ?? 0,
  strokeColor: element.strokeColor || DEFAULT_ELEMENT_PROPS.strokeColor,
  backgroundColor: element.backgroundColor || DEFAULT_ELEMENT_PROPS.backgroundColor,
  width: element.width || 0,
  height: element.height || 0,
  seed: element.seed ?? 1,
  groupIds: element.groupIds ?? [],
  frameId: element.frameId ?? null,
  roundness: normalizeRoundness(element),  // legacy strokeSharpness → roundness
  boundElements: normalizeBoundElements(element),  // legacy boundElementIds → boundElements
  updated: element.updated ?? Date.now(),
  link: element.link ? normalizeLink(element.link) : null,
  locked: element.locked ?? false,
};
```

**文本元素归一化** (`restore.ts:427-476`)
- 修复 `font` 字符串属性 → `fontSize` + `fontFamily`
- 检测/补全行高 `lineHeight`
- 空文本标记为删除（`isDeleted: true`）
- 清理 legacy `rawText` 属性

**线条/箭头元素归一化** (`restore.ts:496-634`)
- 修复/规范化 `points` 数组（至少 2 个有效点）
- 坐标归一化：确保第一个点为 `[0, 0]`，调整 `x/y` 偏移
- 箭头样式标准化：`normalizeArrowhead()`
- 绑定修复：`repairBinding()` 处理新旧绑定格式迁移
- 超大箭头（>75000px）强制标记为删除

**手绘线条归一化** (`restore.ts:477-487`)
- 修复 `points` 和 `pressures` 数组
- 压力值标准化到 `[0, 1]` 范围

**图片元素归一化** (`restore.ts:489-495`)
- 补全 `status: "pending"`
- 补全 `scale: [1, 1]`
- 补全 `crop: null`

##### 第二步：绑定关系修复 (`restore.ts:837-944`)

当 `opts.repairBindings = true` 时执行：

1. **容器-文本绑定修复** (`repairContainerElement`, `repairBoundElement`)
   - 移除重复绑定
   - 修复 `containerId` 反向引用
   - 清理已删除元素的绑定

2. **Frame 成员关系修复** (`repairFrameMembership`)
   - 移除不存在的 Frame 的 `frameId` 引用

3. **线性元素绑定清理**
   - 移除指向不存在元素的 `startBinding` / `endBinding`

4. **Elbow 箭头特殊修复**
   - 自绑定且坐标异常的箭头重置路径
   - 无效折点的箭头重计算路径

##### 第三步：ID 去重与版本同步
```typescript
// 检测重复 ID，自动生成新 ID
if (existingIds.has(migratedElement.id)) {
  migratedElement = { ...migratedElement, id: randomId() };
}

// 不可见元素标记为删除
if (opts.deleteInvisibleElements && isInvisiblySmallElement(element)) {
  migratedElement = { ...migratedElement, isDeleted: true };
}
```

---

### 阶段 6：元素插入画布

**函数**：`App.tsx:3918` → `addElementsFromPasteOrLibrary()`

#### 6.1 坐标转换
```typescript
// 计算元素中心点
const [minX, minY, maxX, maxY] = getCommonBounds(elements);
const elementsCenterX = (minX + maxX) / 2;
const elementsCenterY = (minY + maxY) / 2;

// 转换为场景坐标
const { x, y } = viewportCoordsToSceneCoords({ clientX, clientY }, this.state);

// 对齐到网格
const [gridX, gridY] = getGridPoint(dx, dy, this.getEffectiveGridSize());
```

#### 6.2 元素复制
**函数**：`duplicateElements()`
- 为每个元素生成新 ID（`randomizeSeed: !retainSeed`）
- 保留 Group 关系
- 调整坐标到目标位置

#### 6.3 Frame 自动包含
```typescript
const topLayerFrame = this.getTopLayerFrameAtSceneCoords({ x, y });
if (topLayerFrame) {
  const eligibleElements = filterElementsEligibleAsFrameChildren(duplicatedElements, topLayerFrame);
  nextElements = addElementsToFrame(nextElements, eligibleElements, topLayerFrame);
}
```

#### 6.4 状态更新
- 替换场景元素
- 加载字体（Safari 特殊处理）
- 添加图片文件到缓存
- 选中新插入的元素
- 触发历史记录捕获

---

## 三、Plain Paste vs Normal Paste

通过 `IS_PLAIN_PASTE` 全局变量区分（通常由 `Shift+Ctrl+V` 触发）：

| 处理步骤 | Normal Paste | Plain Paste |
|---------|-------------|-------------|
| HTML 解析 | ✅ 启用 | ❌ 禁用 |
| 表格识别 | ✅ 启用 | ❌ 禁用 |
| SVG 代码识别 | ✅ 启用 | ❌ 禁用 |
| Mermaid 识别 | ✅ 启用 | ❌ 禁用 |
| URL 嵌入识别 | ✅ 启用 | ❌ 禁用 |
| 多行文本分割 | ✅ 分割为多个元素 | ❌ 保留为单个元素 |
| 元素种子随机化 | ✅ 随机化 | ❌ 保留原种子 |

---

## 四、关键设计决策

### 1. 同步提取 + 异步解析
- `parseDataTransferEvent()` 必须同步调用（浏览器安全限制）
- 后续解析、归一化、插入可异步执行

### 2. 优先级匹配机制
按「结构化程度从高到低」匹配，确保最精确的解析被优先使用。

### 3. 防御式归一化
`restoreElements` 是整个系统的「数据守门员」：
- 从不信任外部输入
- 缺失属性补全默认值
- 损坏数据尽力修复
- 无法修复的数据标记删除而非抛出异常

### 4. 向前兼容
通过迁移逻辑支持历史版本数据：
- `strokeSharpness` → `roundness`
- `boundElementIds` → `boundElements`
- `font` 字符串 → `fontSize` + `fontFamily`
- 旧版箭头绑定 → 新版固定点绑定

---

## 五、常见问题解惑

### Q: 为什么粘贴的文字有时候变成多个文本框？
A: 正常粘贴模式下，换行符会被分割为独立的文本元素。使用 `Shift+Ctrl+V` 纯文本粘贴可保留为单个元素。

### Q: 为什么粘贴的 Excalidraw JSON 有时不生效？
A: 检查 JSON 的 `type` 字段，必须是 `excalidraw` / `excalidraw/clipboard` / `excalidraw-api/clipboard` 之一。

### Q: 为什么粘贴的表格有时识别不出来？
A: 表格解析要求行列数一致，且数据列必须是纯数字。混合文本和数字的表格会降级为普通文本。

### Q: 为什么有些粘贴的元素会消失？
A: `restoreElements` 会将不可见元素（尺寸 < 1px）、空文本、超大箭头（>75000px）标记为 `isDeleted: true`，但元素仍在数据中。
