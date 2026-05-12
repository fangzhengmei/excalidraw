# Excalidraw 文本测量与换行流程分析

## 概述

Excalidraw 的文本编辑系统采用了 **Canvas 测量 + 智能换行 + 实时几何回填** 的三层架构，确保在双击进入文本编辑时，光标位置计算、自动换行、字号变化以及最终几何更新都能精确同步。输入法候选词通过浏览器原生 composition 机制被排除在宽度测量之外。

---

## 一、字体探测：基于 Canvas 的文本测量层

### 1.1 核心测量机制

字体探测的核心位于 `packages/element/src/textMeasurements.ts`，通过 **Canvas 2D API** 实现高精度文本测量：

```typescript
class CanvasTextMetricsProvider implements TextMetricsProvider {
  private canvas: HTMLCanvasElement;

  constructor() {
    this.canvas = document.createElement("canvas");
  }

  public getLineWidth(text: string, fontString: FontString): number {
    const context = this.canvas.getContext("2d")!;
    context.font = fontString;
    const metrics = context.measureText(text);
    const advanceWidth = metrics.width;  // 关键：使用 advanceWidth 而非 bounding box
    return advanceWidth;
  }
}
```

**设计要点**：
- **advanceWidth 优先**：使用 glyph 的步进宽度而非实际渲染边界，这与浏览器换行算法一致，避免了字间距导致的宽度计算偏差
- **字体字符串规范化**：通过 `getFontString()` 统一字体格式（字号 + 字体族），确保 Canvas 与 CSS 渲染一致性
- **单字符缓存优化**：对单个字符（CJK、空格等）的宽度进行缓存，减少重复测量开销

### 1.2 字体字符串构造

位于 `@excalidraw/common` 的 `getFontString()` 函数确保跨模块字体描述一致：

```typescript
// 构造格式：`${fontSize}px ${fontFamily}`
export const getFontString = (element: {
  fontSize: number;
  fontFamily: number;
}) => `${element.fontSize}px ${getFontFamilyString(element)}`;
```

### 1.3 行高探测算法

当从旧版本文件加载或恢复文本时，通过反向推导恢复行高：

```typescript
export const detectLineHeight = (textElement: ExcalidrawTextElement) => {
  const lineCount = splitIntoLines(textElement.text).length;
  return (textElement.height / lineCount / textElement.fontSize) as ExcalidrawTextElement["lineHeight"];
};
```

**原理**：通过实际高度 ÷ 行数 ÷ 字号 = 无单位行高倍数，与 CSS `line-height` 规范对齐。

---

## 二、行高与宽度估算：Unicode 感知的换行引擎

### 2.1 分词正则表达式架构

文本换行位于 `packages/element/src/textWrapping.ts`，采用 **Unicode-aware 分词** 策略，支持：

- **西方语言**：空格、连字符处断词
- **CJK 中日韩**：字符间可断行
- **Emoji**：完整 emoji 序列作为不可分割单元
- **标点符号**：特定规则处理开/闭括号、引号等

```typescript
// 高级断行正则（支持 Safari <16.4 降级）
const getLineBreakRegexAdvanced = () => Regex.or(
  getEmojiRegex(),                    // Emoji 序列
  Break.Before(COMMON.WHITESPACE),    // 空格前可断
  Break.After(COMMON.WHITESPACE, COMMON.HYPHEN),  // 空格、连字符后可断
  Break.Before(CJK.CHAR, CJK.CURRENCY)  // CJK 字符前可断
    .NotPrecededBy(COMMON.OPENING, CJK.OPENING),  // 但开括号后不可断
  // ... 更多规则
);
```

### 2.2 行宽估算与缓存策略

```typescript
export const charWidth = (() => {
  const cachedCharWidth: { [key: FontString]: Array<number> } = {};

  const calculate = (char: string, font: FontString) => {
    const unicode = char.charCodeAt(0);
    if (!cachedCharWidth[font]) {
      cachedCharWidth[font] = [];
    }
    if (!cachedCharWidth[font][unicode]) {
      cachedCharWidth[font][unicode] = getLineWidth(char, font);
    }
    return cachedCharWidth[font][unicode];
  };

  return { calculate, getCache, clearCache };
})();
```

**优化点**：
- 单字符宽度按 Unicode 码位缓存
- 多字符单词整体测量以保留 kerning 信息
- CJK 字符因无 kerning 可安全使用缓存值

### 2.3 换行算法流程

```
原始文本 → 硬换行分割 → 按正则分词 → 逐 token 累积宽度 → 超宽则断行 → 尾部空格修剪
     ↓          ↓           ↓            ↓               ↓           ↓
  original   hard line    breakable   current width   soft wrap   trim trailing
```

核心实现 `wrapLine()` 函数关键点：

1. **预分词**：`parseTokens()` 将硬行拆分为可断词元
2. **渐进累积**：逐个 token 累加宽度，直到超过 `maxWidth`
3. **超长词处理**：单个词超宽时拆分为字符级（但保持 emoji 完整）
4. **空格修剪**：软换行处移除尾部空格以对齐浏览器行为

### 2.4 输入法候选词的处理

输入法候选词**不参与宽度计算**的机制：

- **浏览器原生 Composition**：`textarea` 的 `isComposing` 状态下，输入法候选词仅作为视觉覆盖，不写入 `value`
- **事件过滤**：`onkeydown` 中检测 `event.isComposing || event.keyCode === 229` 时跳过处理
- **延迟测量**：直到 `compositionend` 事件触发后，才对最终提交的文本进行测量

---

## 三、提交时几何回填：编辑器与画布的同步

### 3.1 WYSIWYG 编辑器的实时更新流程

位于 `packages/excalidraw/wysiwyg/textWysiwyg.tsx` 的编辑生命周期：

```
双击进入编辑 → 创建 textarea 覆盖 → 初始化样式 → 输入时 oninput → 调用 onChange → 触发场景更新 → redrawTextBoundingBox
     ↓              ↓                ↓            ↓              ↓               ↓               ↓
  pointerdown   position:absolue   font+size     每次按键     normalizeText   mutateElement   重算宽高位置
```

### 3.2 关键函数：`redrawTextBoundingBox`

几何回填的核心算法位于 `packages/element/src/textElement.ts:46-140`：

```typescript
export const redrawTextBoundingBox = (
  textElement: ExcalidrawTextElement,
  container: ExcalidrawElement | null,
  scene: Scene,
) => {
  // 1. 决定最大换行宽度
  const maxWidth = container 
    ? getBoundTextMaxWidth(container, textElement)  // 容器内文本
    : textElement.width;                             // 自由文本

  // 2. 重新换行处理
  const wrappedText = wrapText(
    textElement.originalText,  // 注意：使用原始未换行文本！
    getFontString(textElement),
    maxWidth,
  );

  // 3. 测量新尺寸
  const metrics = measureText(
    wrappedText,
    getFontString(textElement),
    textElement.lineHeight,
  );

  // 4. 自动调整容器大小（如果文本超出）
  if (container && !isArrowElement(container)) {
    if (metrics.height > getBoundTextMaxHeight(container, textElement)) {
      const nextHeight = computeContainerDimensionForBoundText(
        metrics.height, container.type
      );
      scene.mutateElement(container, { height: nextHeight });
    }
  }

  // 5. 计算文本在容器内的对齐位置
  const { x, y } = computeBoundTextPosition(container, textElement, elementsMap);

  // 6. 提交更新到场景
  scene.mutateElement(textElement, {
    text: wrappedText,
    width: metrics.width,
    height: metrics.height,
    x, y,
    angle: container ? container.angle : textElement.angle,
  });
};
```

### 3.3 容器维度换算公式

不同形状容器的可用文本区域计算：

| 容器类型 | 宽度公式 | 高度公式 | 说明 |
|---------|---------|---------|------|
| Rectangle | `width - 2*padding` | `height - 2*padding` | 简单矩形内边距 |
| Ellipse | `(width/2)*√2 - 2*padding` | `(height/2)*√2 - 2*padding` | 椭圆内接最大矩形 |
| Diamond | `width/2 - 2*padding` | `height/2 - 2*padding` | 菱形内接最大矩形 |
| Arrow | `max(0.7*width, fontSize*8)` | `height - 16*padding` | 箭头标签特殊处理 |

### 3.4 对齐与旋转处理

`computeBoundTextPosition()` 实现：

1. **水平对齐**：LEFT/RIGHT/CENTER 基于容器可用宽度与文本宽度的差值
2. **垂直对齐**：TOP/BOTTOM/MIDDLE 基于容器可用高度与文本高度的差值
3. **旋转变换**：先计算未旋转坐标，再绕容器中心旋转相同角度

---

## 四、数据流与协调机制

### 4.1 编辑过程中的数据流

```
用户输入
    ↓
textarea.value → normalizeText() → 制表符转空格、换行规范化
    ↓
onChange(originalText) → App 层存储原始文本（不换行）
    ↓
scene.onUpdate → 触发 updateWysiwygStyle()
    ↓
wrapText(originalText, font, maxWidth) → 生成带软换行的显示文本
    ↓
measureText() → 计算文本实际宽高
    ↓
redrawTextBoundingBox() → 更新元素几何属性
    ↓
Canvas 重绘 → 用户看到更新后的文本框
```

### 4.2 关键不变量

1. **`originalText` vs `text`**：`originalText` 始终存储用户输入的原始文本（无自动换行），`text` 字段是根据当前宽度计算的换行后显示文本
2. **`autoResize` 标志**：为 `true` 时宽度随文本变化；为 `false` 时固定宽度（触发换行）
3. **容器缓存**：`originalContainerCache` 记录容器初始高度，文本删除时容器可回缩到原始大小

---

## 五、光标位置计算的精度保证

### 5.1 点击定位算法

`getCaretIndexFromInitialSceneCoords()` 实现精确的点击→光标映射：

1. **坐标变换**：将点击坐标反向旋转，消除容器旋转影响
2. **行定位**：通过 `y ÷ lineHeight` 确定落在第几视觉行
3. **字符定位**：调用 `getLineCaretOffsetFromNativeLayout()` 精确计算 x 坐标对应的字符偏移

### 5.2 原生布局校准

为消除 Canvas 测量与实际文本渲染的亚像素偏差，使用隐藏 DOM 元素校准：

```typescript
// 创建镜像元素，使用浏览器原生 Range API 测量 caret 位置
const mirror = document.createElement("div");
mirror.style.font = font;
mirror.style.lineHeight = `${lineHeightPx}px`;
mirror.textContent = text;

// 使用 document.createRange() 逐个字符测量边界
const range = document.createRange();
for (const offset of offsets) {
  range.setStart(textNode, offset);
  const caretRect = range.getBoundingClientRect();
  positions.push(caretRect.left);
}
```

**这确保了光标视觉位置与实际编辑位置的精确对齐**，解决了纯 Canvas 测量无法捕捉字体渲染细微差异的问题。

---

## 总结

Excalidraw 文本测量系统的三层架构：

| 层级 | 职责 | 关键技术 |
|-----|------|---------|
| **字体探测层** | 精确测量文本宽度、高度 | Canvas 2D API、advanceWidth、字符缓存 |
| **换行引擎层** | Unicode 智能分词、换行、行高估算 | 正则分词、CJK 规则、emoji 保护、渐进累积 |
| **几何回填层** | 编辑提交后同步更新元素边界 | 容器尺寸换算、对齐算法、旋转变换、原生校准 |

该设计既保证了跨浏览器渲染一致性，又通过分层缓存和渐进计算在性能与精度之间取得了良好平衡。
