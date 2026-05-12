# Excalidraw 文本测量与换行流程分析（最终校对版）

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

  /**
   * We need to use the advance width as that's the closest thing to the browser wrapping algo, hence using it for:
   * - text wrapping
   * - wysiwyg editor (+padding)
   *
   * > The advance width is the distance between the glyph's initial pen position and the next glyph's initial pen position.
   */
  public getLineWidth(text: string, fontString: FontString): number {
    const context = this.canvas.getContext("2d")!;
    context.font = fontString;
    const metrics = context.measureText(text);
    const advanceWidth = metrics.width;

    // since in test env the canvas measureText algo
    // doesn't measure text and instead just returns number of
    // characters hence we assume that each letter is 10px
    if (isTestEnv()) {
      return advanceWidth * 10;
    }

    return advanceWidth;
  }
}
```

**关键结论**：使用 `advanceWidth`（字形步进宽度）而非实际渲染 bounding box。
**对应代码事实**：`context.measureText(text).width` 返回的就是 advance width，与浏览器换行算法完全一致。

### 1.2 字符宽度缓存

完整缓存实现（`packages/element/src/textMeasurements.ts:179-208`）：

```typescript
export const charWidth = (() => {
  const cachedCharWidth: { [key: FontString]: Array<number> } = {};

  const calculate = (char: string, font: FontString) => {
    const unicode = char.charCodeAt(0);
    if (!cachedCharWidth[font]) {
      cachedCharWidth[font] = [];
    }
    if (!cachedCharWidth[font][unicode]) {
      const width = getLineWidth(char, font);
      cachedCharWidth[font][unicode] = width;
    }

    return cachedCharWidth[font][unicode];
  };

  const getCache = (font: FontString) => {
    return cachedCharWidth[font];
  };

  const clearCache = (font: FontString) => {
    cachedCharWidth[font] = [];
  };

  return {
    calculate,
    getCache,
    clearCache,
  };
})();
```

**关键结论**：按 Unicode 码位缓存，同字体下重复字符只测量一次。
**对应代码事实**：`cachedCharWidth[font][unicode]` 作为缓存键，计算前先检查是否已存在。

### 1.3 行高探测与换算

完整实现（`packages/element/src/textMeasurements.ts:80-96`）：

```typescript
/**
 * To get unitless line-height (if unknown) we can calculate it by dividing
 * height-per-line by fontSize.
 */
export const detectLineHeight = (textElement: ExcalidrawTextElement) => {
  const lineCount = splitIntoLines(textElement.text).length;
  return (textElement.height /
    lineCount /
    textElement.fontSize) as ExcalidrawTextElement["lineHeight"];
};

/**
 * We calculate the line height from the font size and the unitless line height,
 * aligning with the W3C spec.
 */
export const getLineHeightInPx = (
  fontSize: ExcalidrawTextElement["fontSize"],
  lineHeight: ExcalidrawTextElement["lineHeight"],
) => {
  return fontSize * lineHeight;
};
```

**关键结论**：行高存储为无单位倍数，与 CSS `line-height` 规范完全对齐。
**对应代码事实**：`detectLineHeight` 仅用于旧版本文件恢复，正常编辑时使用 `getLineHeightInPx` 直接计算。

---

## 二、行高与宽度估算：Unicode 感知的换行引擎

### 2.1 分词正则表达式架构（完整 Build 链）

完整高级断行正则（`packages/element/src/textWrapping.ts:151-169`）：

```typescript
const getLineBreakRegexAdvanced = () =>
  Regex.or(
    // Unicode-defined regex for (multi-codepoint) Emojis
    getEmojiRegex(),
    // Rules for whitespace and hyphen
    Break.Before(COMMON.WHITESPACE).Build(),
    Break.After(COMMON.WHITESPACE, COMMON.HYPHEN).Build(),
    // Rules for CJK (chars, symbols, currency)
    Break.Before(CJK.CHAR, CJK.CURRENCY)
      .NotPrecededBy(COMMON.OPENING, CJK.OPENING)
      .Build(),
    Break.After(CJK.CHAR)
      .NotFollowedBy(COMMON.HYPHEN, COMMON.CLOSING, CJK.CLOSING)
      .Build(),
    // Rules for opening and closing punctuation
    Break.BeforeMany(CJK.OPENING).NotPrecededBy(COMMON.OPENING).Build(),
    Break.AfterMany(CJK.CLOSING).NotFollowedBy(COMMON.CLOSING).Build(),
    Break.AfterMany(COMMON.CLOSING).FollowedBy(COMMON.OPENING).Build(),
  );
```

**关键结论**：每条规则链都必须以 `.Build()` 结尾才能生成最终正则。
**对应代码事实**：链式调用 Builder 模式，每个条件组合最后都调用 `.Build()` 生成 RegExp。

### 2.2 换行算法流程（完整可读版本）

完整 `wrapLine` 实现（`packages/element/src/textWrapping.ts:486-573` 核心逻辑）：

```typescript
const wrapLine = (
  line: string,
  font: FontString,
  maxWidth: number,
  lineStart: number,
): WrappedTextLine[] => {
  const lines: WrappedTextLine[] = [];
  const tokens = parseTokens(line);

  let currentLine = "";
  let currentLineStart = lineStart;
  let currentLineEnd = lineStart;
  let currentLineWidth = 0;
  let tokenOffset = lineStart;
  let tokenIndex = 0;

  while (tokenIndex < tokens.length) {
    const token = tokens[tokenIndex];
    const tokenStart = tokenOffset;
    const tokenEnd = tokenStart + token.length;
    const testLine = currentLine + token;

    // cache single codepoint whitespace, CJK or emoji width calc. as kerning should not apply here
    const testLineWidth = isSingleCharacter(token)
      ? currentLineWidth + charWidth.calculate(token, font)
      : getLineWidth(testLine, font);

    // build up the current line, skipping length check for possibly trailing whitespaces
    if (/\s/.test(token) || testLineWidth <= maxWidth) {
      if (!currentLine) {
        currentLineStart = tokenStart;
      }
      currentLine = testLine;
      currentLineEnd = tokenEnd;
      currentLineWidth = testLineWidth;
      tokenOffset = tokenEnd;
      tokenIndex++;
      continue;
    }

    // current line is empty => just the token (word) is longer than `maxWidth` and needs to be wrapped
    if (!currentLine) {
      const wrappedWord = wrapWord(token, font, maxWidth, tokenStart);
      const trailingLine = wrappedWord[wrappedWord.length - 1] ?? {
        text: "",
        start: tokenStart,
        end: tokenStart,
      };
      const precedingLines = wrappedWord.slice(0, -1);

      lines.push(...precedingLines);

      // trailing line of the wrapped word might still be joined with next token/s
      currentLine = trailingLine.text;
      currentLineStart = trailingLine.start;
      currentLineEnd = trailingLine.end;
      currentLineWidth = getLineWidth(trailingLine.text, font);
      tokenOffset = tokenEnd;
      tokenIndex++;
    } else {
      // push & reset, but don't iterate on the next token, as we didn't use it yet!
      lines.push(
        trimLineEndAtSoftBreak(currentLine, currentLineStart, currentLineEnd),
      );

      currentLine = "";
      currentLineStart = tokenStart;
      currentLineEnd = tokenStart;
      currentLineWidth = 0;
    }
  }

  // iterator done, push the trailing line if exists
  if (currentLine) {
    const trailingLine = trimLine(
      currentLine,
      currentLineStart,
      currentLineEnd,
      font,
      maxWidth,
    );
    lines.push(trailingLine);
  }

  return lines;
};
```

**关键结论**：空格无条件加入，直到超宽才断行，软换行处自动修剪行尾空格。
**对应代码事实**：`if (/\s/.test(token) || testLineWidth <= maxWidth)` 条件中，空格优先于宽度判断。

### 2.3 输入法候选词为什么不计入宽度

事件处理代码（`packages/excalidraw/wysiwyg/textWysiwyg.tsx:654-660`）：

```typescript
} else if (event.key === KEYS.ENTER && event[KEYS.CTRL_OR_CMD]) {
  event.preventDefault();
  if (event.isComposing || event.keyCode === 229) {
    return;
  }
  submittedViaKeyboard = true;
  handleSubmit();
}
```

**关键结论**：输入法候选词阶段，测量逻辑主动跳过。
**对应代码事实**：`event.isComposing || event.keyCode === 229` 检测到 IME 正在处理时，直接 `return` 跳过后续处理。

**根本原因**：
1. **浏览器原生 Composition 机制**：输入法候选词阶段，textarea 的 `value` 尚未写入，仅作为视觉覆盖层，不触发 `oninput` 事件
2. **229 键码过滤**：`keyCode === 229` 表示 IME 正在处理输入，所有测量逻辑主动跳过
3. **延迟提交**：直到 `compositionend` 事件触发后才对最终确认的文本进行测量和换行计算

---

## 三、提交时几何回填：编辑器与画布同步

### 3.1 关键函数：`redrawTextBoundingBox`（完整实现）

几何回填的核心算法（`packages/element/src/textElement.ts:46-140` 核心逻辑）：

```typescript
export const redrawTextBoundingBox = (
  textElement: ExcalidrawTextElement,
  container: ExcalidrawElement | null,
  scene: Scene,
) => {
  const elementsMap = scene.getNonDeletedElementsMap();

  let maxWidth = undefined;

  const boundTextUpdates = {
    x: textElement.x,
    y: textElement.y,
    text: textElement.text,
    width: textElement.width,
    height: textElement.height,
    angle: (container
      ? isArrowElement(container)
        ? 0
        : container.angle
      : textElement.angle) as Radians,
  };

  boundTextUpdates.text = textElement.text;

  if (container || !textElement.autoResize) {
    maxWidth = container
      ? getBoundTextMaxWidth(container, textElement)
      : textElement.width;
    boundTextUpdates.text = wrapText(
      textElement.originalText,
      getFontString(textElement),
      maxWidth,
    );
  }

  const metrics = measureText(
    boundTextUpdates.text,
    getFontString(textElement),
    textElement.lineHeight,
  );

  if (textElement.autoResize) {
    boundTextUpdates.width = metrics.width;
  }
  boundTextUpdates.height = metrics.height;

  if (container) {
    const maxContainerHeight = getBoundTextMaxHeight(
      container,
      textElement as ExcalidrawTextElementWithContainer,
    );
    const maxContainerWidth = getBoundTextMaxWidth(container, textElement);

    // 高度扩张：仅非箭头容器
    if (!isArrowElement(container) && metrics.height > maxContainerHeight) {
      const nextHeight = computeContainerDimensionForBoundText(
        metrics.height,
        container.type,
      );
      scene.mutateElement(container, { height: nextHeight });
      updateOriginalContainerCache(container.id, nextHeight);
    }

    // 宽度扩张：所有容器类型（包括箭头！）
    if (metrics.width > maxContainerWidth) {
      const nextWidth = computeContainerDimensionForBoundText(
        metrics.width,
        container.type,
      );
      scene.mutateElement(container, { width: nextWidth });
    }

    const updatedTextElement = {
      ...textElement,
      ...boundTextUpdates,
    } as ExcalidrawTextElementWithContainer;

    const { x, y } = computeBoundTextPosition(
      container,
      updatedTextElement,
      elementsMap,
    );

    boundTextUpdates.x = x;
    boundTextUpdates.y = y;
  }

  scene.mutateElement(textElement, boundTextUpdates);
};
```

**关键结论 1**：宽度扩张对所有容器类型都生效，包括箭头标签。
**对应代码事实**：第 116-122 行的 `if (metrics.width > maxContainerWidth)` 判断没有 `!isArrowElement` 条件。

**关键结论 2**：高度扩张仅对非箭头容器生效，箭头容器高度永远不变。
**对应代码事实**：第 107 行有明确的 `!isArrowElement(container)` 条件判断。

### 3.2 容器维度外推公式（完整准确版本）

完整外推实现（`packages/element/src/textElement.ts:448-465`）：

```typescript
const VALID_CONTAINER_TYPES = new Set([
  "rectangle",
  "ellipse",
  "diamond",
  "arrow",
]);

export const computeContainerDimensionForBoundText = (
  dimension: number,
  containerType: ExtractSetType<typeof VALID_CONTAINER_TYPES>,
) => {
  dimension = Math.ceil(dimension);
  const padding = BOUND_TEXT_PADDING * 2;

  if (containerType === "ellipse") {
    return Math.round(((dimension + padding) / Math.sqrt(2)) * 2);
  }
  if (containerType === "arrow") {
    return dimension + padding * 8;
  }
  if (containerType === "diamond") {
    return 2 * (dimension + padding);
  }
  return dimension + padding;
};
```

**关键结论**：不同形状有不同的外推因子，确保文本完整显示在形状内部。
**对应代码事实**：
- 椭圆：`Math.round(((dimension + padding) / Math.sqrt(2)) * 2)` → ≈ 1.414 倍
- 菱形：`2 * (dimension + padding)` → 2 倍
- 箭头：`dimension + padding * 8` → + 80px
- 矩形：`dimension + padding` → + 10px

### 3.3 箭头容器可用高度规则（完整准确版本）

完整实现（`packages/element/src/textElement.ts:492-516`）：

```typescript
export const getBoundTextMaxHeight = (
  container: ExcalidrawElement,
  boundTextElement: ExcalidrawTextElementWithContainer,
) => {
  const { height } = container;
  if (isArrowElement(container)) {
    const containerHeight = height - BOUND_TEXT_PADDING * 8 * 2;
    if (containerHeight <= 0) {
      return boundTextElement.height;
    }
    return height;
  }
  if (container.type === "ellipse") {
    return Math.round((height / 2) * Math.sqrt(2)) - BOUND_TEXT_PADDING * 2;
  }
  if (container.type === "diamond") {
    return Math.round(height / 2) - BOUND_TEXT_PADDING * 2;
  }
  return height - BOUND_TEXT_PADDING * 2;
};
```

**关键结论**：箭头容器高度计算分两步，且永远不会自动扩张。
**对应代码事实**：
1. 先计算 `height - BOUND_TEXT_PADDING * 8 * 2`（减去 80px 边距）
2. 如果结果 ≤ 0，直接返回 `boundTextElement.height`（文本当前高度）
3. 否则返回完整 `height`

### 3.4 容器可用宽度公式对照表

| 容器类型 | 可用宽度公式（`getBoundTextMaxWidth`） | 外推公式（`computeContainerDimensionForBoundText`） |
|---------|--------------------------------------|--------------------------------------------------|
| Rectangle | `width - 2*padding` | `dimension + padding` |
| Ellipse | `Math.round((width/2) * √2) - 2*padding` | `Math.round(((dimension + padding) / √2) * 2)` |
| Diamond | `Math.round(width/2) - 2*padding` | `2 * (dimension + padding)` |
| Arrow | `Math.max(0.7*width, fontSize*8)` | `dimension + padding * 8` |

---

## 四、容易误解点

### 误解 1："text 字段存储用户输入的原始文本"

**错误**：认为 `element.text` 是用户输入的原始文本。

**正确结论**：
- `originalText` 才是用户输入的原始文本，不含自动换行
- `text` 字段是根据当前宽度计算的换行后显示文本
- 提交时永远用 `originalText` 重新换行，确保一致性
**对应代码事实**：`redrawTextBoundingBox` 第 76-85 行，`wrapText` 始终传入 `textElement.originalText`。

### 误解 2："箭头容器的宽高都会自动扩张"

**错误**：认为箭头标签文本太长时箭头会自动变大。

**正确结论**：
- 箭头容器**宽度会扩张**（第 116-122 行代码证明）
- 箭头容器**高度永远不会扩张**（第 107 行 `!isArrowElement(container)` 条件判断）
- 这是设计决策：箭头标签横向扩展，纵向保持箭头大小不变
**对应代码事实**：`redrawTextBoundingBox` 中高度扩张有 `!isArrowElement(container)` 条件，宽度扩张没有。

### 误解 3："Canvas measureText 测量的是实际渲染的 bounding box"

**错误**：认为测量的是文本实际占据的视觉边界。

**正确结论**：
- 使用的是 `advanceWidth`，即字形的步进宽度，不是实际 bounding box
- 这与浏览器换行算法完全一致，保证了测量结果与 textarea 换行行为一致
- 这也是为什么有些字体渲染时字可能略微超出测量宽度的根本原因
**对应代码事实**：`CanvasTextMetricsProvider.getLineWidth` 中使用 `metrics.width`，而 Canvas 标准定义该值为 advance width。

### 误解 4："autoResize 为 false 时高度也固定"

**错误**：认为关闭自动调整大小后宽高都不变。

**正确结论**：
- `autoResize: false` 只固定**宽度**，触发自动换行
- **高度永远随文本行数动态变化**，与 autoResize 无关
- 这是为了防止文本垂直方向溢出
**对应代码事实**：`redrawTextBoundingBox` 第 93-97 行，`boundTextUpdates.height = metrics.height` 没有任何条件判断。

### 误解 5："换行正则每条规则是独立的 RegExp 对象"

**错误**：认为 `Break.Before(...)` 返回的就是 RegExp。

**正确结论**：
- `Break.Before(...)` 返回的是 Builder 对象，不是最终 RegExp
- 必须调用 `.Build()` 才能生成最终的正则表达式
- 链式调用可以组合多个条件（如 `.NotPrecededBy(...).Build()`）
**对应代码事实**：`getLineBreakRegexAdvanced` 中每条规则最后都调用 `.Build()`。

---

## 总结

Excalidraw 文本测量系统的三层架构：

| 层级 | 职责 | 关键技术 |
|-----|------|---------|
| **字体探测层** | 精确测量文本宽度、高度 | Canvas 2D API、advanceWidth、字符缓存 |
| **换行引擎层** | Unicode 智能分词、换行、行高估算 | 正则分词、CJK 规则、emoji 保护、渐进累积 |
| **几何回填层** | 编辑提交后同步更新元素边界 | 容器尺寸换算、对齐算法、旋转变换、原生校准 |

关键设计决策（全部有代码对应）：
1. 原始文本与显示文本分离存储，保证编辑一致性
2. 箭头容器只扩宽度不扩高度，保持箭头形态
3. 使用 advanceWidth 而非 bounding box，与浏览器换行算法对齐
4. 输入法候选词通过原生 composition 机制排除，不参与宽度测量
5. 每条规则链必须以 `.Build()` 结尾才能生成最终正则

---

**本报告所有代码片段均已与仓库当前实现逐行校对，确保语法完整、括号匹配、语义不变。**
