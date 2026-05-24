# 剪贴板粘贴数据处理链路分析（真实执行路径版）

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

## 二、完整处理链路（真实执行分支）

### 阶段 0：Plain Paste 标志设置

**位置**：`App.tsx:4964-4972`

```typescript
// 在 keydown 事件中检测 Ctrl+V
if (event[KEYS.CTRL_OR_CMD] && event.key.toLowerCase() === KEYS.V) {
  IS_PLAIN_PASTE = event.shiftKey;  // Shift+Ctrl+V → 纯文本粘贴
  clearTimeout(IS_PLAIN_PASTE_TIMER);
  IS_PLAIN_PASTE_TIMER = window.setTimeout(() => {
    IS_PLAIN_PASTE = false;  // 100ms 后自动重置
  }, 100);
}
```

---

### 阶段 1：事件捕获与数据提取

**入口**：`App.tsx:3865` → `pasteFromClipboard`

```
用户按下 Ctrl+V → 触发 document paste 事件 → pasteFromClipboard()
```

#### 关键校验（任一不满足则直接 return）
1. **焦点校验**：检查 Excalidraw 容器是否为活动元素 (`App.tsx:3870-3875`)
2. **画布校验**：检查鼠标位置是否在画布上，且不在可输入元素中 (`App.tsx:3877-3887`)

#### 数据提取
**函数**：`clipboard.ts:466` → `parseDataTransferEvent(event)`

> **⚠️ 浏览器安全限制**：必须在同一帧同步调用，否则 Firefox 会清空 clipboardData。

```typescript
// 同步提取（必须在 await 之前调用）
const dataTransferList = await parseDataTransferEvent(event);
const filesList = dataTransferList.getFiles();  // 提取文件列表
```

该函数遍历 `DataTransferItemList`，将每个项目标准化为：
- `{ kind: "file", type: "image/png", file: File, fileHandle }`
- `{ kind: "string", type: "text/plain", value: string }`

同时调用 `normalizeFile()` 对文件进行标准化处理。

#### 数据解析
**函数**：`clipboard.ts:522` → `parseClipboard(dataList, isPlainPaste)`

#### onPaste 拦截点
**位置**：`App.tsx:3898-3906`

```typescript
if (this.props.onPaste) {
  try {
    if ((await this.props.onPaste(data, event)) === false) {
      return;  // 返回 false 则终止整个粘贴流程
    }
  } catch (error: any) {
    console.error(error);  // 异常只打日志，不终止流程
  }
}
```

> **关键点**：
- 拦截发生在 `parseClipboard` 之后，`insertClipboardContent` 之前
- 返回 `false` → 终止流程
- 抛出异常 → 继续流程

---

### 阶段 2：数据类型识别与解析（clipboard.ts）

**函数**：`clipboard.ts:522` → `parseClipboard(dataList, isPlainPaste)`

#### 2.1 支持的 MIME 类型
定义于 `constants.ts:273-277`：
```typescript
ALLOWED_PASTE_MIME_TYPES = [
  "text/plain", "text/html", "image/svg+xml", "image/png",
  "image/jpeg", "image/gif", "image/webp", ...
]
```

#### 2.2 文本数据解析
**函数**：`clipboard.ts:330` → `parseClipboardEventTextData()`

**真实分支逻辑**：

```
1. 尝试获取 text/html 类型
   ↓
2. !isPlainPaste 且 htmlItem 存在 → 调用 maybeParseHTMLDataItem()
   ↓
3. 如果返回 mixedContent：
   ├─ 全部都是 text 类型 → 降级为 text 类型（用 text/plain 或拼接文本）
   └─ 包含 imageUrl 类型 → 返回 mixedContent 类型
   ↓
4. 否则降级为 text/plain
```

**HTML 解析细节** (`clipboard.ts:212-250`)：
- `parseHTMLTree()` 递归遍历 DOM 节点，提取：
  - 文本节点 → `{ type: "text", value: string }`
  - 图片节点 (`<img src="http...">`) → `{ type: "imageUrl", value: url }`

#### 2.3 parseClipboard 返回逻辑

```typescript
if (parsedEventData.type === "mixedContent") {
  return { mixedContent: parsedEventData.value };
  // ⚠️ 注意：只返回 mixedContent，不返回 text！
}

try {
  const systemClipboardData = JSON.parse(parsedEventData.value);
  if (clipboardContainsElements(systemClipboardData)) {
    return {
      elements: systemClipboardData.elements,
      files: systemClipboardData.files,
      text: isPlainPaste ? JSON.stringify(...) : undefined,
      programmaticAPI,
    };
  }
} catch {}

return { text: parsedEventData.value };
```

#### 2.4 Excalidraw 内部格式识别
**函数**：`clipboard.ts:73-87` → `clipboardContainsElements()`

检查 JSON 数据的 `type` 字段：
```typescript
EXPORT_DATA_TYPES = {
  excalidraw: "excalidraw",                          // 普通导出
  excalidrawClipboard: "excalidraw/clipboard",       // 剪贴板导出
  excalidrawClipboardWithAPI: "excalidraw-api/clipboard"  // API 导出
}
```

---

### 阶段 3：内容分发与类型判断（真实分支顺序）

**函数**：`App.tsx:3698` → `insertClipboardContent(data, filesList, isPlainPaste)`

> **⚠️ 核心机制**：内容按优先级依次匹配，匹配成功则提前 return。

#### 完整的真实判断顺序（从上到下，return 即终止）

##### 1. 错误消息 (`App.tsx:3711-3715`)
```typescript
if (data.errorMessage) → setState({ errorMessage }) → return
```

##### 2. 混合内容（HTML 文本+图片）(`App.tsx:3717-3725`)
```typescript
// ⚠️ 三个条件必须同时满足：
if (
  dataTransferFiles.length === 0 &&  // ① 没有文件
  !isPlainPaste &&               // ② 不是纯文本粘贴
  data.mixedContent               // ③ 有 mixedContent
) {
  await addElementsFromMixedContentPaste(...) → return
}
```

> **关键真实行为**：
> - 如果 `dataTransferFiles.length > 0`（有粘贴的图片文件），**即使有 mixedContent 也会跳过此分支！
> - 当 `data.mixedContent` 存在时，`data.text` 是 `undefined`

##### 3. 电子表格数据 (`App.tsx:3727-3741`)
```typescript
if (!isPlainPaste && data.text) {
  const result = tryParseSpreadsheet(data.text);
  if (result.ok) → 弹出图表对话框 → return
}
```

**解析逻辑**：
- 尝试三种分隔符：`\t` (Tab) > `,` (CSV) > `;` (分号)
- 评分标准：列数一致 > 列数最多 > 分隔符优先级
- 数据验证：数字列必须全部可解析为数字，至少 2 行数据

##### 4. 图片或 SVG 代码 (`App.tsx:3743-3762`)
```typescript
const imageFiles = dataTransferFiles.map((data) => data.file);

// 如果没有文件但有 text，检测是否为 SVG 代码：
if (imageFiles.length === 0 && data.text && !isPlainPaste) {
  if (trimmedText.startsWith("<svg") && trimmedText.endsWith("</svg>")) {
    imageFiles.push(SVGStringToFile(trimmedText));
  }
}

if (imageFiles.length > 0) → insertImages(...) → return
```

支持：
- 剪贴板中的图片文件（PNG/JPG/SVG/GIF 等）
- 粘贴的 SVG 代码字符串

##### 5. Excalidraw 元素 (`App.tsx:3764-3783`)
```typescript
if (data.elements) {
  const elements = data.programmaticAPI
    ? convertToExcalidrawElements(data.elements)  // API 骨架数据
    : data.elements;                              // 完整元素数据
  
  addElementsFromPasteOrLibrary({ elements, ... }) → return
}
```

##### 6. 纯文本空值检查 (`App.tsx:3785-3788`)
```typescript
if (!data.text) → return  // 没有文本直接终止
```

##### 7. Mermaid 图表 (`App.tsx:3790-3814`)
```typescript
if (!isPlainPaste && isMaybeMermaidDefinition(data.text)) {
  try {
    const api = await import("@excalidraw/mermaid-to-excalidraw");
    const { elements: skeletonElements, files = {} } =
      await api.parseMermaidToExcalidraw(data.text);
    
    const elements = convertToExcalidrawElements(skeletonElements, {
      regenerateIds: true,
    });
    
    addElementsFromPasteOrLibrary({ elements, files, ... }) → return
  } catch (err: any) {
    // ⚠️ 解析失败只打 warn 日志，不 return！
    console.warn(`parsing pasted text as mermaid definition failed: ${err.message}`);
  }
}
```

> **⚠️ Mermaid 解析失败后的完整回退顺序**：
> 1. `isMaybeMermaidDefinition` 匹配成功，进入 Mermaid 分支
> 2. `parseMermaidToExcalidraw` 抛出异常 → 进入 catch
> 3. catch 只打日志，**不 return**
> 4. ↓ 继续向下执行
> 5. → 尝试 URL 识别（第 8 步）
> 6. → 如果 URL 识别也不通过
> 7. → 最后回退到纯文本处理（第 9 步）

**Mermaid 识别正则** (`mermaid.ts:2-33`)：
```typescript
/^(?:%%{.*?}%%[\s\n]*)?\b(?:\s*flowchart(-beta)?|\s*graph(-beta)?|\s*sequenceDiagram|...)\b/
```

支持 20+ 种图表类型：`flowchart`, `graph`, `sequenceDiagram`, `classDiagram`, `stateDiagram`, `erDiagram`, `gantt`, `pie`, `mindmap`, `timeline` 等。

##### 8. 可嵌入 URL (`App.tsx:3816-3859`)
```typescript
判断条件：
- !isPlainPaste
- 所有非空行都是有效的可嵌入 URL
- 支持 YouTube、Vimeo、Figma 等视频/嵌入链接
- 每行一个 URL，自动横向排列

→ 全部满足则插入 embeddable 元素 → return
```

##### 9. 纯文本（最后回退）(`App.tsx:3861-3862`)
```typescript
addTextFromPaste(data.text, isPlainPaste)
```

---

### 阶段 4：各类型内容专门处理（真实行为）

#### 4.1 混合内容处理 (`App.tsx:4072`) → `addElementsFromMixedContentPaste()`

**⚠️ **真实分支**：
```typescript
if (
  !isPlainPaste &&
  mixedContent.some(node.type === "imageUrl") &&
  this.isToolSupported("image")
) {
  // ── 有图片 URL 的分支 ──
  // 1. 提取所有 imageUrl
  // 2. Promise.all 并发下载图片
  // 3. 成功的作为图片插入
  // 4. 失败的显示错误消息
  // ⚠️ 文本节点被丢弃！（TODO: "rewrite this to paste both text & images"
} else {
  // ── 没有图片 URL 的分支 ──
  // 1. 提取所有 text 节点
  // 2. 用 "\n\n" 拼接成一个字符串
  // 3. 调用 addTextFromPaste()
}
```

> **已知缺陷**：代码顶部注释 `// TODO rewrite this to paste both text & images at the same time if pasted data contains both`

#### 4.2 纯文本处理 (`App.tsx:4123`) → `addTextFromPaste()`
```typescript
const lines = isPlainPaste ? [text] : text.split("\n");
// 正常粘贴：按换行符分割为多个文本元素
// 纯文本粘贴：保留为单个元素
```

- 长文本自动换行（最大宽度 `min(画布宽度*0.5, 800px)`，最小 200px
- 多行文本纵向排列，行间距 10px
- 应用当前工具栏样式（颜色、字体、字号等）

#### 4.3 骨架元素转换 (`transform.ts:509`) → `convertToExcalidrawElements()`
将简化的元素骨架转换为完整 Excalidraw 元素：
```typescript
switch (element.type) {
  case "rectangle" | "ellipse" | "diamond":
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

### 阶段 5：元素归一化（粘贴时的真实行为）

**函数**：`restore.ts:764` → `restoreElements(elements, existingElements, opts)`

**粘贴时的真实调用参数** (`App.tsx:3926-3928`)：
```typescript
const elements = restoreElements(opts.elements, null, {
  deleteInvisibleElements: true,
  // ⚠️ 注意：没有传 repairBindings 参数！
});
```

> **设计意图**：确保粘贴/导入的元素符合当前版本的数据格式，修复损坏数据，补全缺失属性，保证数据一致性。

#### 5.1 粘贴时实际执行的流程

##### 第一步：逐个元素归一化 (`restore.ts:413`) → `restoreElement()`

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
- **单个箭头绑定修复：`repairBinding()` 处理新旧绑定格式迁移
- 超大箭头（>75000px）强制标记为删除

**手绘线条归一化** (`restore.ts:477-487`)
- 修复 `points` 和 `pressures` 数组
- 压力值标准化到 `[0, 1]` 范围

**图片元素归一化** (`restore.ts:489-495`)
- 补全 `status: "pending"`
- 补全 `scale: [1, 1]`
- 补全 `crop: null`

##### 第二步：ID 去重与不可见元素处理
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

##### ⚠️ 第三步：绑定关系修复 —— 粘贴时被跳过！

**位置**：`restore.ts:830-835`)
```typescript
if (!opts?.repairBindings) {
  // ⚠️ 粘贴时 repairBindings 是 undefined，直接返回！
  return restoredElements;
}
```

> **粘贴时不执行的修复**（`restore.ts:837-944` 全部跳过）：

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

> **重要结论**：
> - 只有单个箭头的绑定格式迁移会执行（在 `restoreElement` 中）
> - 但容器-文本双向绑定完整性校验、Frame 成员关系、绑定清理、Elbow 箭头特殊修复，**在粘贴时都不会执行**

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

## 三、完整的条件分流逻辑

### 有文件时的分流条件

**关键判断点**：`App.tsx:3718`
```typescript
if (dataTransferFiles.length === 0 && !isPlainPaste && data.mixedContent)
```

**真实场景分析**：

| 场景 | dataTransferFiles | data.mixedContent | 执行分支 |
|-----|-------------------|--------------------|----------|
| 复制网页中的图片+文字 | 1（图片文件） | 有 | ✅ 跳过混合内容 → 进入图片分支 |
| 复制网页中的图片+文字 | 0（只有 HTML) | 有 | ✅ 进入混合内容分支 |
| 复制 Excel 表格 | 0 | 无（全是文本降级） | ✅ 进入表格分支 |
| 截图粘贴 | 1（图片文件） | 无 | ✅ 进入图片分支 |
| 复制 Excalidraw 元素 | 0 | 无 | ✅ 进入元素分支 |

> **反直觉行为**：从网页复制带图片的富文本时，如果浏览器会自动把图片转成文件，导致 `dataTransferFiles.length > 0`，此时混合内容分支被跳过，直接进入图片分支，**文本丢失**！

---

## 四、Plain Paste vs Normal Paste

通过 `IS_PLAIN_PASTE` 全局变量区分（`Shift+Ctrl+V` 触发）：

| 处理步骤 | Normal Paste | Plain Paste |
|---------|-------------|-------------|
| HTML 解析 | ✅ 启用 | ❌ 禁用 |
| 表格识别 | ✅ 启用 | ❌ 禁用 |
| SVG 代码识别 | ✅ 启用 | ❌ 禁用 |
| Mermaid 识别 | ✅ 启用 | ❌ 禁用 |
| URL 嵌入识别 | ✅ 启用 | ❌ 禁用 |
| 多行文本分割 | ✅ 分割为多个元素 | ❌ 保留为单个元素 |
| 元素种子随机化 | ✅ 随机化 | ❌ 保留原种子 |
| mixedContent 分支 | ✅ 可能进入 | ❌ 不进入 |

---

## 五、关键设计决策

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

### 5. 绑定修复的选择性执行
- 粘贴时跳过复杂绑定修复，权衡了性能与数据完整性：
- 只做单元素级别的属性修复
- 不做跨元素的绑定关系校验
- 避免潜在风险：粘贴带绑定的元素可能出现引用不一致

---

## 六、完整执行流程图

```
Ctrl+V 按下
   ↓
keydown 事件设置 IS_PLAIN_PASTE = event.shiftKey
   ↓
paste 事件触发 → pasteFromClipboard()
   ├─ 焦点校验失败 → return
   ├─ 画布校验失败 → return
   ├─ parseDataTransferEvent(event) [同步]
   ├─ parseClipboard(dataList, isPlainPaste)
   │   ├─ 解析 HTML → mixedContent 或 text
   │   ├─ 解析 JSON → elements 或 text
   │   └─ 返回 { mixedContent? | elements? | text?
   ├─ onPaste 拦截 → 返回 false? → return
   └─ insertClipboardContent(data, filesList, isPlainPaste)
       ├─ 1. errorMessage → return
       ├─ 2. mixedContent + 无文件 + !plain → 混合内容处理 → return
       │   ├─ 有 imageUrl → 下载图片插入（文本丢弃）
       │   └─ 无 imageUrl → 文本拼接插入
       ├─ 3. !plain + text → tryParseSpreadsheet → ok? → 图表对话框 → return
       ├─ 4. 收集 imageFiles（文件 + SVG 代码) → 有文件? → 插入图片 → return
       ├─ 5. data.elements? → 转换 → addElementsFromPasteOrLibrary → return
       │   └─ restoreElements(elements, null, { deleteInvisibleElements: true })
       │       ├─ 逐个元素属性归一化
       │       ├─ 箭头绑定格式迁移
       │       ├─ ID 去重
       │       └─ ⚠️ 跳过绑定关系修复
       ├─ 6. !data.text → return
       ├─ 7. !plain + isMaybeMermaidDefinition → try parseMermaid
       │   ├─ ✅ 成功 → addElementsFromPasteOrLibrary → return
       │   └─ ❌ 失败 → warn 日志 → 继续向下（不 return)
       ├─ 8. !plain + 全部是 embeddable URL → 插入嵌入元素 → return
       └─ 9. addTextFromPaste(text, isPlainPaste) → return
```

---

## 七、常见问题解惑（真实行为版）

### Q: 为什么从网页复制的带图片富文本，粘贴后只有图片丢失了？
A: 浏览器自动将图片转为文件，`dataTransferFiles.length > 0`，导致混合内容分支条件不满足，直接进入图片分支，文本被丢弃。这是已知问题，代码中有 TODO 注释待修复。

### Q: 为什么粘贴的文字有时候变成多个文本框？
A: 正常粘贴模式下，换行符会被分割为独立的文本元素。使用 `Shift+Ctrl+V` 纯文本粘贴可保留为单个元素。

### Q: 为什么粘贴的 Excalidraw JSON 有时不生效？
A: 检查 JSON 的 `type` 字段，必须是 `excalidraw` / `excalidraw/clipboard` / `excalidraw-api/clipboard` 之一。

### Q: 为什么粘贴的表格有时识别不出来？
A: 表格解析要求行列数一致，且数据列必须是纯数字。混合文本和数字的表格会降级为普通文本。

### Q: 为什么有些粘贴的元素会消失？
A: `restoreElements` 会将不可见元素（尺寸 < 1px）、空文本、超大箭头（>75000px）标记为 `isDeleted: true`，但元素仍在数据中。

### Q: Mermaid 解析失败后会怎样？
A: 只打 warn 日志，不终止流程，继续尝试 URL 识别，最后回退到纯文本。所以语法错误的 Mermaid 代码会作为普通文本粘贴。

### Q: 粘贴带箭头绑定的元素，绑定关系会修复吗？
A: 单个箭头的绑定格式会迁移，但容器-文本双向绑定、Frame 成员关系、绑定完整性校验**都不会执行**。可能出现绑定引用不一致的情况。

### Q: onPaste 回调抛出异常会怎样？
A: 异常会被 catch 住，只打错误日志，粘贴流程继续执行。只有返回 `false` 才会终止。
