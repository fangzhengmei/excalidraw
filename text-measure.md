# Excalidraw 文本测量与换行流程分析

## 整体流程概览

文本编辑的完整生命周期：

```
用户双击图形 → 光标定位计算 → 创建原生 textarea 覆盖 → 输入内容变化 → 自动换行
                                                                 ↓
                                                         字号变化 → 触发测量重算
                                                                 ↓
                                                      输入法 Composition → 候选词不计入宽度
                                                                 ↓
                                                      提交/失焦 → 几何回填 → 同步更新容器尺寸
```

关键协调机制：
1. **光标定位**：反向旋转消除角度影响 → 行定位 → 原生 DOM Range 校准字符偏移
2. **自动换行**：Unicode 分词 → 渐进宽度累积 → 超宽断行 → 尾部空格修剪
3. **字号变化**：更新字号 → 重新测量宽高 → redrawTextBoundingBox 全量重算
4. **提交回填**：使用 originalText 重新换行 → 测量最终尺寸 → 扩张/回缩容器 → 重排文本位置

---

## 一、字体探测：基于 Canvas 的测量层

### 1.1 核心测量机制

字体探测的核心位于 `packages/element/src/textMeasurements.ts:121-150`，通过 **Canvas 2D API** 实现高精度文本测量：

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
    const advanceWidth = metrics.width;  // 关键：使用 glyph 步进宽度
    return advanceWidth;
  }
}
```

**代码实际行为**：
- **advanceWidth 优先**：使用 `context.measureText(text).width，即 glyph 的步进宽度（advance width），这与浏览器换行算法完全一致
- **字体字符串格式**：`${fontSize}px ${fontFamily}`，通过 `getFontString()` 统一构造
- **测试环境适配**：在测试环境中宽度值 × 10，解决 jsdom 不支持真实测量的问题

### 1.2 字符宽度缓存

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

**代码实际行为**：
- 按 Unicode 码位缓存，同字体下重复字符只测量一次
- 换行算法中对单字符（CJK、空格等）直接使用缓存值
- 多字符单词整体测量以保留 kerning 字距调整信息

### 1.3 行高探测与换算

```typescript
// 行高探测：从元素实际高度反向推导无单位行高
export const detectLineHeight = (textElement: ExcalidrawTextElement) => {
  const lineCount = splitIntoLines(textElement.text).length;
  return (textElement.height / lineCount / textElement.fontSize) as ExcalidrawTextElement["lineHeight"];
};

// 行高换算：无单位行高 → 像素高度
export const getLineHeightInPx = (
  fontSize: ExcalidrawTextElement["fontSize"],
  lineHeight: ExcalidrawTextElement["lineHeight"],
) => {
  return fontSize * lineHeight;
};
```

**代码实际行为**：
- 行高存储为无单位倍数（如 1.25），与 CSS `line-height` 规范完全对齐
- `detectLineHeight` 仅用于旧版本文件恢复，正常编辑流程中行高由字号直接计算
- 文本总高度 = 行数 × 字号 × 行高倍数

---

## 二、行高与宽度估算：Unicode 感知的换行引擎

### 2.1 分词正则表达式架构

文本换行位于 `packages/element/src/textWrapping.ts`，采用 **Unicode-aware 分词** 策略：

```typescript
const getLineBreakRegexAdvanced = () => Regex.or(
  getEmojiRegex(),                         // 完整 emoji 序列
  Break.Before(COMMON.WHITESPACE),       // 空格前可断
  Break.After(COMMON.WHITESPACE, COMMON.HYPHEN),  // 空格、连字符后可断
  Break.Before(CJK.CHAR, CJK.CURRENCY)
    .NotPrecededBy(COMMON.OPENING, CJK.OPENING),  // CJK 字符前可断但开括号后不可断
  Break.After(CJK.CHAR)
    .NotFollowedBy(COMMON.HYPHEN, COMMON.CLOSING, CJK.CLOSING),
  Break.BeforeMany(CJK.OPENING).NotPrecededBy(COMMON.OPENING),
  Break.AfterMany(CJK.CLOSING).NotFollowedBy(COMMON.CLOSING),
  Break.AfterMany(COMMON.CLOSING).FollowedBy(COMMON.OPENING),
);
```

**代码实际行为**：
- 支持 Safari <16.4 降级到简单正则（不支持 lookbehind 断言）
- emoji 作为完整多码位字符
- CJK 字符间可断行，但标点符号有特殊规则避免行首行尾保护

### 2.2 换行算法流程

```typescript
const wrapLine = (line: string, font: FontString, maxWidth: number, lineStart: number): WrappedTextLine[] => {
  const tokens = parseTokens(line);
  let currentLine = "";
  let currentLineWidth = 0;

  while (tokenIndex < tokens.length) {
    const token = tokens[tokenIndex];
    const testLine = currentLine + token;

    // 单字符使用缓存宽度，多字符重新测量
    const testLineWidth = isSingleCharacter(token)
      ? currentLineWidth + charWidth.calculate(token, font)
      : getLineWidth(testLine, font);

    if (/\s/.test(token) || testLineWidth <= maxWidth) {
      currentLine = testLine;
      currentLineWidth = testLineWidth;
      tokenIndex++;
    } else if (!currentLine) {
      // 单个词超宽，拆分为字符级别（emoji保持完整
      const wrappedWord = wrapWord(token, font, maxWidth, tokenStart);
      // ... 处理超宽词
    } else {
      // 当前行已满，推入结果，重置
      lines.push(trimLineEndAtSoftBreak(currentLine, currentLineStart, currentLineEnd));
      currentLine = "";
      currentLineWidth = 0;
    }
  }
  // ... 处理尾部行
};
```

**代码实际行为**：
- **渐进式累积：逐个 token 累加宽度，空格无条件加入直到超宽才断行
- **超宽词处理**：单个 token 超过 maxWidth 时按字符拆分（但 emoji 保持完整）
- **空格修剪**：软换行处调用 trimLineEndAtSoftBreak() 移除行尾空格，与浏览器行为一致

### 2.3 输入法候选词为什么不计入宽度

**代码实际行为**（位于 `textWysiwyg.tsx` 事件处理）：

```typescript
// 在 onkeydown 中检测输入法状态：
if (event.key === KEYS.ENTER && event[KEYS.CTRL_OR_CMD]) {
  event.preventDefault();
  // composition 期间跳过处理
  if (event.isComposing || event.keyCode === 229) {
    return;
  }
  handleSubmit();
}
```

**根本原因**：
1. **浏览器原生 Composition 机制**：输入法候选词阶段，textarea 的 `value` 尚未写入，仅作为视觉覆盖层，不触发 `oninput` 事件
2. **229 键码过滤**：`keyCode === 229 表示 IME 正在处理输入，所有测量逻辑主动跳过
3. **延迟提交**：直到 `compositionend` 事件触发后才对最终确认的文本进行测量和换行计算

这就是为什么输入中文时候选词阶段不会触发宽度变化，确认后才更新。

---

## 三、提交时几何回填：编辑器与画布同步

### 3.1 关键函数：`redrawTextBoundingBox`

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

  // 2. 重新换行处理（关键：始终使用 originalText！
  boundTextUpdates.text = wrapText(
    textElement.originalText,
    getFontString(textElement),
    maxWidth,
  );

  // 3. 测量新尺寸
  const metrics = measureText(
    boundTextUpdates.text,
    getFontString(textElement),
    textElement.lineHeight,
  );

  // 4. 自动调整容器宽度和高度
  if (container) {
    const maxContainerHeight = getBoundTextMaxHeight(container, textElement);
    const maxContainerWidth = getBoundTextMaxWidth(container, textElement);

    // 高度扩张：非箭头容器且文本高度 > 容器可用高度
    if (!isArrowElement(container) && metrics.height > maxContainerHeight) {
      const nextHeight = computeContainerDimensionForBoundText(metrics.height, container.type);
      scene.mutateElement(container, { height: nextHeight });
      updateOriginalContainerCache(container.id, nextHeight);
    }

    // 宽度扩张：所有容器（包括箭头！）文本宽度 > 容器可用宽度
    if (metrics.width > maxContainerWidth) {
      const nextWidth = computeContainerDimensionForBoundText(metrics.width, container.type);
      scene.mutateElement(container, { width: nextWidth });
    }

    // 5. 计算文本在容器内的对齐位置
    const { x, y } = computeBoundTextPosition(container, updatedTextElement, elementsMap);
    boundTextUpdates.x = x;
    boundTextUpdates.y = y;
  }

  // 6. 提交最终更新
  scene.mutateElement(textElement, boundTextUpdates);
};
```

**代码实际行为**：
- **宽度扩张逻辑（所有容器类型都会触发，包括箭头标签）
- **高度扩张逻辑**：仅非箭头容器会触发，箭头容器高度永远不扩张
- **容器回缩**：文本删除时容器可以回缩到 originalContainerCache 记录的原始高度
- **旋转同步**：文本角度始终与容器角度保持一致

### 3.2 容器维度换算公式（修正版）

| 容器类型 | 可用宽度公式 | 可用高度公式 | 说明 |
|---------|-------------|-------------|------|
| Rectangle | `width - 2*padding` | `height - 2*padding` | 简单矩形内边距（padding=5） |
| Ellipse | `(width/2)*√2 - 2*padding` | `(height/2)*√2 - 2*padding` | 椭圆内接最大矩形 |
| Diamond | `width/2 - 2*padding` | `height/2 - 2*padding` | 菱形内接最大矩形 |
| Arrow | `max(0.7*width, fontSize*8)` | **见下方特殊规则 | 箭头标签特殊处理 |

**箭头容器可用高度规则（修正）**：
```typescript
if (isArrowElement(container)) {
  const containerHeight = height - BOUND_TEXT_PADDING * 8 * 2;  // 减去上下各40px
  if (containerHeight <= 0) {
    return boundTextElement.height;  // 容器太小，返回文本当前高度
  }
  return height;  // 否则返回完整容器高度
}
```
关键点：箭头容器高度计算时减去 80px 边距，如果结果 ≤0 直接返回文本元素高度，否则返回容器高度，**箭头容器高度永远不会自动扩张**。

### 3.3 容器外推公式

```typescript
export const computeContainerDimensionForBoundText = (
  dimension: number,
  containerType: string,
) => {
  dimension = Math.ceil(dimension);
  const padding = BOUND_TEXT_PADDING * 2;

  if (containerType === "ellipse") {
    return Math.round(((dimension + padding) / Math.sqrt(2)) * 2;
  }
  if (containerType === "arrow") {
    return dimension + padding * 8;  // 箭头标签加 80px 边距
  }
  if (containerType === "diamond") {
    return 2 * (dimension + padding);
  }
  return dimension + padding;
};
```

**代码实际行为**：不同形状反向外推因子不同，确保文本完整显示在形状内部，包括：
- 椭圆：× 2/√2 ≈ 1.414
- 菱形：× 2
- 箭头：+ 80px
- 矩形：+ 10px

---

## 四、容易误解点

### 误解 1："text 字段存储用户输入的原始文本"

**错误**：认为 `element.text 是用户看到的最终文本

**正确结论**：
- `originalText` 才是用户输入的原始文本，不含自动换行
- `text` 字段是根据当前宽度计算的换行后显示文本
- 提交时永远用 `originalText` 重新换行，确保一致性

### 误解 2："箭头容器的宽高都会自动扩张"

**错误**：认为箭头标签文本太长时箭头会自动变大

**正确结论**：
- 箭头容器**宽度会扩张（第116-122行代码证明）
- 箭头容器**高度永远不会扩张**（第107行 `!isArrowElement(container)` 条件判断）
- 这是设计决策：箭头标签横向扩展，纵向保持箭头大小不变

### 误解 3："Canvas measureText 测量的是实际渲染的 bounding box"

**错误**：认为测量的是文本实际占据的视觉边界

**正确结论**：
- 使用的是 `advanceWidth`，即字形的步进宽度，不是实际 bounding box
- 这与浏览器换行算法完全一致，保证了测量结果与 textarea 换行行为一致
- 这也是为什么有些字体渲染时字可能略微超出测量宽度的根本原因

### 误解 4："autoResize 为 false 时高度也固定"

**错误**：认为关闭自动调整大小后宽高都不变

**正确结论**：
- `autoResize: false` 只固定**宽度**，触发自动换行
- **高度永远随文本行数动态变化，无论 autoResize 无关
- 这是为了防止文本垂直方向溢出

---

## 总结

Excalidraw 文本测量系统的三层架构：

| 层级 | 职责 | 关键技术 |
|-----|------|---------|
| **字体探测层** | 精确测量文本宽度、高度 | Canvas 2D API、advanceWidth、字符缓存 |
| **换行引擎层** | Unicode 智能分词、换行、行高估算 | 正则分词、CJK 规则、emoji 保护、渐进累积 |
| **几何回填层** | 编辑提交后同步更新元素边界 | 容器尺寸换算、对齐算法、旋转变换、原生校准 |

关键设计决策：
1. 原始文本与显示文本分离存储，保证编辑一致性
2. 箭头容器只扩宽度不扩高度，保持箭头形态
3. 使用 advanceWidth 而非 bounding box，与浏览器换行算法对齐
4. 输入法候选词通过原生 composition 机制排除，不参与宽度测量
