# 画板字体加载与文本测量缓存协同机制分析

## 一、核心模块概览

整个系统由两大核心模块协同工作：

| 模块 | 主要职责 | 关键文件 |
|------|----------|----------|
| 字体加载器 | 字体注册、加载、加载后缓存失效 | `packages/excalidraw/fonts/Fonts.ts` |
| 文本测量缓存 | 字符宽度测量、缓存、换行计算 | `packages/element/src/textMeasurements.ts` |

---

## 二、字体加载机制详解

### 2.1 字体注册与初始化

**代码位置**：`Fonts.ts:62-403`

字体采用懒加载注册机制：
```typescript
public static get registered() {
  if (!Fonts._registered) {
    Fonts._registered = Fonts.init();  // 首次访问时初始化
  } else if (!Fonts._initialized) {
    // 处理宿主应用提前注册字体的情况
    Fonts._registered = new Map([
      ...Fonts.init().entries(),
      ...Fonts._registered.entries(),
    ]);
  }
  return Fonts._registered;
}
```

**初始化流程**：
1. `init()` 方法注册所有内置字体（Excalifont、Virgil、Cascadia 等）
2. 每个字体包含元数据（metadata）和字体描述符（fontFaces）
3. 支持 CJK 手写体 fallback 和 emoji fallback

### 2.2 字体加载流程

**代码位置**：`Fonts.ts:212-271`

```
loadSceneFonts()
    ↓
getSceneFamilies() → 收集场景中使用的字体族
getCharsPerFamily() → 按字体族收集实际使用的字符
    ↓
loadFontFaces()
    ├─ 将字体注册到 document.fonts
    └─ fontFacesLoader() → 并发控制（10个并发）
        ├─ fonts.check(font, text) → 检查是否已加载
        └─ fonts.load(font, text) → 实际加载
```

**关键优化点**：
- 按实际使用的字符加载，而非整个字体文件（`text` 参数）
- 使用 `PromisePool` 控制并发，避免阻塞主线程
- 局部字体（如 Helvetica）跳过注册流程

### 2.3 已加载字体缓存

**代码位置**：`Fonts.ts:49`

```typescript
public static readonly loadedFontsCache = new Set<string>();
```

- **存储内容**：字体签名 `${family}-${style}-${weight}-${unicodeRange}`
- **作用**：防止重复触发字体加载完成后的刷新逻辑
- **生命周期**：静态成员，跨实例共享，永久存在（无过期机制）

### 2.4 字体加载触发更新的两条入口链路

字体加载完成后，通过**两条独立链路**调用 `onLoaded()` 触发缓存失效和场景重绘：

#### 链路 1：主动加载（App 初始化）

**代码位置**：`App.tsx:3012-3014`

```typescript
// manually loading the font faces seems faster even in browsers that do fire the loadingdone event
this.fonts.loadSceneFonts().then((fontFaces) => {
  this.fonts.onLoaded(fontFaces);
});
```

**触发时机**：App 初始化、加载初始数据、粘贴内容等场景主动调用 `loadSceneFonts()`

**特点**：
- 手动控制，速度更快
- 不依赖浏览器事件
- 适用于需要立即响应的场景

#### 链路 2：事件监听（FontFaceSet.loadingdone）

**代码位置**：`App.tsx:3302-3310`

```typescript
addEventListener(
  document.fonts,
  "loadingdone",
  (event) => {
    const fontFaces = (event as FontFaceSetLoadEvent).fontfaces;
    this.fonts.onLoaded(fontFaces);
  },
  { passive: false },
),
```

**触发时机**：浏览器 `FontFaceSet` API 触发 `loadingdone` 事件

**特点**：
- 被动监听，捕获所有字体加载完成事件
- 包括其他来源触发的字体加载
- 浏览器原生事件支持

#### 链路 3：Safari 特殊处理（粘贴场景）

**代码位置**：`App.tsx:4007-4012`

```typescript
// paste event may not fire FontFace loadingdone event in Safari, hence loading font faces manually
if (isSafari) {
  Fonts.loadElementsFonts(duplicatedElements).then((fontFaces) => {
    this.fonts.onLoaded(fontFaces);
  });
}
```

**触发时机**：Safari 浏览器中粘贴内容

**背景**：Safari 的粘贴事件可能不会触发 `loadingdone` 事件，因此需要手动加载

---

## 三、文本测量缓存机制详解

### 3.1 字符宽度缓存（charWidth）—— 两层结构

**代码位置**：`textMeasurements.ts:179-208`

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

  return {
    calculate,    // 计算并缓存
    getCache,     // 获取某字体的缓存数组
    clearCache,   // 清除某字体的缓存
  };
})();
```

**⚠️ 缓存结构与索引语义澄清**：

缓存采用**两层索引结构**：

```
cachedCharWidth = {
  // 第一层：FontString 作为外层键
  "20px Excalifont, Helvetica, Arial": [
    // 第二层：char.charCodeAt(0) 作为数组索引
    65: 12.5,   // 'A' (U+0041) charCodeAt(0) = 65
    66: 11.8,   // 'B' (U+0042) charCodeAt(0) = 66
    20013: 24.0, // '中' (U+4E2D) charCodeAt(0) = 20013
    ...
  ],
  "16px Virgil, Helvetica, Arial": [
    65: 10.2,   // 同字符，不同字体，宽度不同
    ...
  ]
}
```

| 层级 | 索引方式 | 说明 |
|------|----------|------|
| 第一层 | `FontString`（对象键） | 包含字号、字体族、fallback 链 |
| 第二层 | `char.charCodeAt(0)`（数组索引） | **第一个 UTF-16 码元的值**，不是完整 Unicode 码位 |

**⚠️ `charCodeAt(0)` 语义校正**：

`charCodeAt(0)` 返回的是字符串**第一个 UTF-16 码元**的数值，与 `codePointAt(0)` 有本质区别：

| 字符 | `charCodeAt(0)` | `codePointAt(0)` | 说明 |
|------|-----------------|------------------|------|
| 'A' (U+0041) | 65 (0x41) | 65 (0x41) | BMP 内，两者相等 |
| '中' (U+4E2D) | 20013 (0x4E2D) | 20013 (0x4E2D) | BMP 内，两者相等 |
| '😀' (U+1F600) | 55357 (0xD83D) | 128512 (0x1F600) | 代理对，前者只返回高代理项 |
| '👨‍👩‍👧‍👦' | 55357 (0xD83D) | 128104 (0x1F468) | ZWJ 序列，前者只返回第一个码元 |

**关键**：只有 BMP 字符（码位 < 0x10000）的 `charCodeAt(0)` 才等于完整 Unicode 码位。超出 BMP 的字符，`charCodeAt(0)` 只返回高代理项的值。

### 3.2 文本测量流程

**代码位置**：`textMeasurements.ts:12-168`

```
measureText(text, font, lineHeight)
    ↓
getTextHeight() → 行数 × 行高（无缓存）
getTextWidth()
    ↓
splitIntoLines() → 按换行符分割
getLineWidth(line, font)
    ↓
CanvasTextMetricsProvider.getLineWidth()
    └─ canvas.getContext("2d").measureText(text)
```

### 3.3 缓存命中条件

**`charWidth.calculate()` 命中条件**：

1. **字体匹配**：`FontString` 完全一致（字号 + 字体族 + fallback 链）
2. **字符已测量**：该 UTF-16 首码元（`charCodeAt(0)`）在对应字体数组中已有值
3. **缓存未被清除**：该字体的缓存数组未被重置

**未命中场景**：
- 首次测量该字符
- 字体加载完成后缓存被清除
- 使用了不同的字号或字体族（外层键不同）

### 3.4 缓存使用场景—— isSingleCharacter 的真实判定

**代码位置**：`textWrapping.ts:509-512`

```typescript
// cache single codepoint whitespace, CJK or emoji width calc. as kerning should not apply here
const testLineWidth = isSingleCharacter(token)
  ? currentLineWidth + charWidth.calculate(token, font)
  : getLineWidth(testLine, font);
```

**⚠️ emoji 缓存路径校正**：

**所有 emoji 都不走 charWidth 缓存路径**，无论是否为单码位 emoji。原因是 `isSingleCharacter` 的判定逻辑：

**`isSingleCharacter` 实现**（`textWrapping.ts:724-729`）：
```typescript
const isSingleCharacter = (maybeSingleCharacter: string) => {
  return (
    maybeSingleCharacter.codePointAt(0) !== undefined &&
    maybeSingleCharacter.codePointAt(1) === undefined
  );
};
```

**关键判定**：`codePointAt(1) === undefined`

对于单码位 emoji（如 😀 U+1F600）：
- UTF-16 表示为代理对：`\uD83D\uDE00`（两个码元）
- `codePointAt(0)` = 128512（完整码位，代理对组合计算）
- `codePointAt(1)` = 56832（第二个码元的码位值，低代理项）
- 因此 `codePointAt(1) !== undefined` → **返回 false**

**实际验证结果**：
```
'😀':  isSingleCharacter = false
'🌍':  isSingleCharacter = false
'👍':  isSingleCharacter = false
'👨‍👩‍👧‍👦': isSingleCharacter = false
'👩🏽‍🦰':  isSingleCharacter = false
'🇨🇳':  isSingleCharacter = false
```

**真实的缓存路径分布**：

```
token 类型判断
    ↓
isSingleCharacter(token) = codePointAt(0) !== undefined && codePointAt(1) === undefined
    ↓
┌─────────────────────────────┬──────────────────────────────────┐
│ 返回 true（走缓存）          │ 返回 false（走直接测量）          │
├─────────────────────────────┼──────────────────────────────────┤
│ 单码元 ASCII：A, 1, 空格    │ 所有 emoji（含单码位如 😀）       │
│ BMP CJK：中、日、韩         │ ZWJ 序列：👨‍👩‍👧‍👦              │
│ BMP 符号：©, ®, €          │ 肤色修饰：👩🏽‍🦰                  │
│ 单码元空白符：\t, \n         │ 国旗序列：🇨🇳                    │
│                             │ 多字符 token："Hello", "World"   │
│                             │ 所有超出 BMP 的字符（码位≥0x10000）│
└─────────────────────────────┴──────────────────────────────────┘
```

**结论**：`isSingleCharacter` 实际上等于 **"是否为单个 BMP 字符"**，而非注释中声称的 "single codepoint"。单码位但超出 BMP 的字符（如 emoji）会被排除在缓存路径之外。

### 3.5 wrapWord 中的 emoji 特殊处理

**代码位置**：`textWrapping.ts:578-647`

当一个 token 超过一行宽度时，进入 `wrapWord()` 逐字符换行：

```typescript
const wrapWord = (word, font, maxWidth, wordStart) => {
  // 多码位 emoji 已经被分词器拆分，不应再拆分
  if (getEmojiRegex().test(word)) {
    return [{
      text: word,
      start: wordStart,
      end: wordStart + word.length,
    }];
  }
  
  // 其他字符逐字符换行
  for (const char of Array.from(word)) {
    const _charWidth = charWidth.calculate(char, font);
    // ...
  }
};
```

**关键逻辑**：
- emoji（无论单码位还是多码位）在 `wrapWord` 中被整体保留，不进入逐字符循环
- 由于 emoji 的 `isSingleCharacter` 返回 false，它们本就不会走缓存路径
- 因此 emoji 始终不会调用 `charWidth.calculate()`，不会污染缓存

---

## 四、两大系统的协同机制

### 4.1 协同数据流图

```
用户操作/场景加载
    ↓
redrawTextBoundingBox() / wrapText()
    ├─ 调用 measureText() / getLineWidth()
    │   └─ isSingleCharacter(token)
    │       ├─ true  → charWidth.calculate() → 命中则返回缓存，未命中则测量
    │       └─ false → getLineWidth() → 直接测量，不缓存
    │
loadSceneFonts()  [异步执行]
    ├─ fonts.check() → 检查字体状态
    └─ fonts.load() → 加载字体
        ↓  [字体加载完成]
两条入口链路调用 onLoaded():
├─ 链路1: loadSceneFonts().then(onLoaded)  [主动]
└─ 链路2: document.fonts "loadingdone" 事件  [被动]
    ↓
Fonts.onLoaded(fontFaces)
    ├─ 检查 loadedFontsCache → 是否需要处理
    ├─ 更新 loadedFontsCache
    ├─ 遍历所有文本元素
    │   ├─ ShapeCache.delete(element) → 清除形状缓存
    │   ├─ charWidth.clearCache(getFontString(element)) → 【关键】清除字符宽度缓存
    │   └─ 容器元素也清除 ShapeCache
    └─ scene.triggerUpdate() → 触发场景重绘
        ↓
重新测量文本（使用新加载的真实字体）
    └─ BMP 字符会重新调用 charWidth.calculate() → 重新测量并缓存正确宽度
```

### 4.2 缓存失效触发点

**代码位置**：`Fonts.ts:104-146`

字体加载完成后，`onLoaded()` 方法执行缓存失效：

```typescript
public onLoaded = (fontFaces: readonly FontFace[]): void => {
  let shouldBail = true;

  for (const fontFace of fontFaces) {
    const sig = `${fontFace.family}-${fontFace.style}-${fontFace.weight}-${fontFace.unicodeRange}`;

    if (!Fonts.loadedFontsCache.has(sig)) {
      Fonts.loadedFontsCache.add(sig);
      shouldBail = false;  // 有新字体，需要处理
    }
  }

  if (shouldBail) return;  // 所有字体已处理过，直接退出

  // 清除所有文本元素的缓存
  for (const element of this.scene.getNonDeletedElements()) {
    if (isTextElement(element)) {
      ShapeCache.delete(element);
      // 【关键衔接点】清除该字体的字符宽度缓存
      charWidth.clearCache(getFontString(element));
      
      const container = getContainerElement(element, elementsMap);
      if (container) {
        ShapeCache.delete(container);
      }
    }
  }

  this.scene.triggerUpdate();
};
```

**`clearCache` 实现**（`textMeasurements.ts:199-201`）：
```typescript
const clearCache = (font: FontString) => {
  cachedCharWidth[font] = [];
};
```

**注意**：`clearCache` 不是删除键，而是将数组重置为空数组。这意味着外层 `FontString` 键仍然存在，但所有测量值被清空。

### 4.3 失效设计的原因

**为什么字体加载后必须清除缓存？**

```
初始状态：字体未加载 → 浏览器使用 fallback 字体（如 Arial）
           ↓
测量文本：charWidth 缓存了 fallback 字体的宽度（仅 BMP 字符）
           ↓
字体加载完成：真实字体（如 Excalifont）可用
           ↓
问题：缓存的是 fallback 字体的宽度，与真实字体宽度不一致
           ↓
后果：文本换行位置错误、容器尺寸计算错误
           ↓
解决方案：清除 charWidth 缓存，下次测量使用真实字体
```

**代码注释说明**：
```typescript
// clear the width cache, so that we don't perform subsequent wrapping 
// based on the stale fallback font metrics
charWidth.clearCache(getFontString(element));
```
`Fonts.ts:133-134`

---

## 五、NFC 归一化对 offset 语义的影响

### 5.1 parseTokens 中的 NFC 归一化

**代码位置**：`textWrapping.ts:382-388`

```typescript
export const parseTokens = (line: string) => {
  const breakLineRegex = getLineBreakRegex();

  // normalizing to single-codepoint composed chars due to canonical equivalence
  // of multi-codepoint versions for chars like č, で (~ so that we don't break a line in between c and ˇ)
  // filtering due to multi-codepoint chars like 👨‍👩‍👧‍👦, 👩🏽‍🦰
  return line.normalize("NFC").split(breakLineRegex).filter(Boolean);
};
```

### 5.2 归一化对 offset 的影响——实际推导

**⚠️ 示例推导校正**：

以 `'ça b'` 为例，其中 `'ç'` 为 NFD 形式（`'c' + 组合符 '\u0327'`）：

```
原始文本（NFD）：'c' '\u0327' 'a' ' ' 'b'
原始 length = 5（5 个 UTF-16 码元）
    ↓
line.normalize("NFC")
    ↓
归一化文本（NFC）：'ç' 'a' ' ' 'b'
归一化后 length = 4（4 个 UTF-16 码元）
    ↓
split(/(\s)/).filter(Boolean)
    ↓
tokens = ['ça', ' ', 'b']
    ↓
wrapLine 中跟踪 tokenOffset：
  token[0] 'ça'：start=0, end=0+2=2, token.length=2  ✓ 归一化后长度
  token[1] ' '： start=2, end=2+1=3, token.length=1
  token[2] 'b'： start=3, end=3+1=4, token.length=1
    ↓
final offset = 4, 原始文本 length = 5
    ↓
❌ MISMATCH：offset 比原始长度少 1
```

**代码注释警告**（`textWrapping.ts:378-380`）：
```typescript
/**
 * Note: tokenization normalizes to NFC first so decomposed graphemes are treated as
 * their composed variants for wrapping. Any code that needs exact source offsets should
 * keep in mind that this assumes the input text is already NFC-normalized.
 */
```

### 5.3 offset 语义的前提条件

`WrappedTextLine` 的 `start` 和 `end` 字段：

```typescript
export type WrappedTextLine = {
  text: string;
  start: number;  // 代码单元偏移（基于归一化后的文本）
  end: number;    // 代码单元偏移（基于归一化后的文本）
};
```

**⚠️ 前提假设澄清**：
- `start`/`end` 是**归一化后文本**的代码单元偏移，而非原始文本的偏移
- 只有当输入文本本身就是 NFC 归一化形式时，`start`/`end` 才能正确映射回原始文本位置
- 如果输入包含 NFD 形式的字符，`start`/`end` 与原始文本的码元位置不对应

**风险**：如果输入文本包含 NFD 形式的字符（如从某些输入法或复制粘贴来源），可能导致：
- 光标位置计算错误（偏移 1 位或更多）
- 文本选择范围不准确
- 编辑操作定位偏差

---

## 六、缓存命中与失效的完整场景分析

### 6.1 正常流程（缓存命中）

**场景**：用户在已有文本的画布上继续输入

1. 打开画布，场景中已有使用 Excalifont 20px 的文本 "Hello"
2. 字体已加载完成，`charWidth` 中已有 H/e/l/l/o 的宽度缓存
3. 用户继续输入 " World"
4. 测量时：
   - ' '（空格）→ `isSingleCharacter` 返回 true → 命中缓存
   - 'W' → `isSingleCharacter` 返回 true → 未命中，调用 canvas 测量，存入缓存
   - 'o' → `isSingleCharacter` 返回 true → 命中缓存
   - 'r' → `isSingleCharacter` 返回 true → 未命中，测量后缓存
   - 'l' → `isSingleCharacter` 返回 true → 命中缓存
   - 'd' → `isSingleCharacter` 返回 true → 未命中，测量后缓存
5. 后续测量这些字符均直接命中缓存

### 6.2 字体切换场景（缓存失效）

**场景**：用户将文本字体从 Excalifont 切换为 Virgil

1. 用户选中文本，修改 fontFamily 属性
2. `redrawTextBoundingBox()` 被调用
3. 使用新的 `FontString`（"20px Virgil, ..."）调用 `measureText()`
4. `charWidth.calculate()` 检查该 FontString 的缓存数组 → **不存在**
5. 创建新的缓存数组，逐个 BMP 字符测量并缓存
6. 旧字体（Excalifont）的缓存仍保留，不会被清除

### 6.3 动态字体加载场景（缓存失效）

**场景**：打开包含 CJK 字符的画布，字体异步加载

1. 画布初始化，渲染文本 "你好世界"，字体为 Excalifont
2. Excalifont 无 CJK glyph，浏览器使用系统 fallback 字体显示
3. `charWidth` 缓存了 fallback 字体下 "你/好/世/界" 的宽度（都是 BMP 字符）
4. 异步加载 Xiaolai（CJK 手写体）字体
5. 字体加载完成 → 通过链路 1 或链路 2 触发 `onLoaded()`
6. 检查 `loadedFontsCache` → 该字体未处理过
7. 清除该文本元素对应 FontString 的 `charWidth` 缓存
8. 清除 ShapeCache，触发场景重绘
9. 重绘时重新测量，使用真实的 Xiaolai 字体宽度，更新缓存

### 6.4 字号变化场景（缓存键变化）

**场景**：用户将文本字号从 20px 改为 24px

1. `FontString` 从 "20px Excalifont, ..." 变为 "24px Excalifont, ..."
2. 这是**不同的外层缓存键**，原缓存不会被命中
3. 为 24px 字体创建新的缓存数组
4. 两个字号的缓存独立存在，互不影响

### 6.5 emoji 测量场景（全部走直接测量）

**场景**：用户输入包含 emoji 的文本 "Hello 😀 World 👨‍👩‍👧‍👦"

1. 分词后 tokens: ["Hello", " ", "😀", " ", "World", " ", "👨‍👩‍👧‍👦"]
2. 测量时：
   - "Hello" → 多字符，`isSingleCharacter` 返回 false → 调用 `getLineWidth()`
   - " " → 单码元空格，`isSingleCharacter` 返回 true → 走 `charWidth.calculate()`
   - "😀" → 单码位 emoji，但 UTF-16 代理对，`isSingleCharacter` 返回 false → 调用 `getLineWidth()`
   - " " → 走缓存
   - "World" → 多字符，调用 `getLineWidth()`
   - " " → 走缓存
   - "👨‍👩‍👧‍👦" → ZWJ 序列，`isSingleCharacter` 返回 false → 调用 `getLineWidth()`

3. 如果 "👨‍👩‍👧‍👦" 超过一行宽度进入 `wrapWord()`：
   - `getEmojiRegex().test("👨‍👩‍👧‍👦")` 返回 true
   - 整体保留，不拆分，不调用 `charWidth.calculate()`

**关键**：所有 emoji 都不会进入 `charWidth.calculate()`，不会污染缓存。

### 6.6 NFD 输入场景（offset 错位）

**场景**：用户通过复制粘贴输入包含 NFD 字符的文本 "c\u0327a b"（显示为 "ça b"）

1. 原始文本长度 = 5，包含 NFD 形式的 "ç"
2. `parseTokens()` 先归一化为 NFC，长度变为 4
3. 分词后 tokens = ["ça", " ", "b"]
4. `wrapLine` 计算 offset：
   - "ça" start=0, end=2
   - " " start=2, end=3
   - "b" start=3, end=4
5. 返回的 `WrappedTextLine` 中 `start`/`end` 基于归一化后的长度 4
6. 如果外部代码用这些 offset 去索引原始文本（长度 5），会导致错位
7. 例如：end=4 指向原始文本的第 4 个码元（空格），而非第 5 个码元（'b' 之后）

---

## 七、边缘情况与潜在问题

### 7.1 loadedFontsCache 的假阳性问题

**代码位置**：`Fonts.ts:106-107`

```typescript
// bail if all fonts with have been processed. We're checking just a
// subset of the font properties (though it should be enough), so it
// can technically bail on a false positive.
```

**问题**：仅通过 family/style/weight/unicodeRange 判断，可能漏掉某些边缘情况（如不同 URL 加载的同名字体）。

**影响**：极端情况下可能不会触发缓存清除，导致继续使用 fallback 字体的测量值。

### 7.2 charWidth 缓存的内存泄漏风险

**问题**：`cachedCharWidth` 是闭包内的普通对象，**没有过期机制**。如果用户：
- 频繁切换不同字体
- 切换不同字号
- 输入大量生僻字符

缓存会持续增长，理论上存在内存泄漏风险。

**缓解因素**：
- 实际使用的字体族数量有限（约 10 种内置字体）
- BMP 字符最多到 0xFFFF，单个数组最大 65536 项
- 每项是 number（8 字节），单字体最大约 500KB

### 7.3 代理对范围的缓存索引问题

**代码位置**：`textMeasurements.ts:183`

```typescript
const unicode = char.charCodeAt(0);  // 只取第一个 UTF-16 码元
```

**问题**：虽然 emoji 等代理对字符不会走缓存路径（因为 `isSingleCharacter` 返回 false），但如果有其他方式直接调用 `charWidth.calculate()` 传入代理对字符：
- "😀" (U+1F600) 的 `charCodeAt(0)` = 0xD83D（高代理项）
- 索引落在代理对区域（0xD800-0xDFFF）
- 该区域是 Unicode 保留区域，正常文本中不会出现单独的代理对字符

**实际影响**：由于 `isSingleCharacter` 过滤，代理对字符不会进入缓存路径，因此这个问题在正常流程中不会触发。

### 7.4 字距调整（Kerning）的影响

**设计决策**：`textWrapping.ts:509-512`

```typescript
// cache single codepoint whitespace, CJK or emoji width calc. 
// as kerning should not apply here
const testLineWidth = isSingleCharacter(token)
  ? currentLineWidth + charWidth.calculate(token, font)
  : getLineWidth(testLine, font);
```

**原因**：单字符无相邻字符，字距调整不适用。多字符字符串可能存在字距调整，直接累加会有误差，因此不使用缓存。

### 7.5 并发加载与竞态条件

**问题**：字体加载是异步的，如果在字体加载过程中：
1. 用户输入文本，触发测量（使用 fallback）
2. BMP 字符的测量结果存入缓存
3. 字体加载完成，缓存被清除
4. 重绘时重新测量（使用真实字体）

**设计**：`onLoaded()` 中 `loadedFontsCache` 检查确保每个字体只处理一次，即使两条链路都触发也不会重复清除缓存。

**两条链路的幂等性**：
- 链路 1 执行后，`loadedFontsCache` 已更新
- 链路 2 随后触发时，检查 `loadedFontsCache` 发现已处理，直接退出

---

## 八、关键衔接代码位置汇总

| 衔接点 | 文件位置 | 作用 |
|--------|----------|------|
| 字体加载后清除文本缓存 | `Fonts.ts:134` | `charWidth.clearCache(getFontString(element))` |
| 字体签名缓存 | `Fonts.ts:49, 111-117` | `loadedFontsCache` 防止重复处理 |
| 字符宽度缓存定义 | `textMeasurements.ts:179-208` | IIFE 闭包实现的两层结构 charWidth |
| 缓存使用（换行计算） | `textWrapping.ts:509-512` | 单字符使用缓存优化 |
| 单字符判断 | `textWrapping.ts:724-729` | 区分 BMP/非 BMP 字符 |
| wrapWord emoji 处理 | `textWrapping.ts:584-593` | emoji 整体保留，不拆分 |
| NFC 归一化 | `textWrapping.ts:388` | 分词前的归一化操作 |
| FontString 生成 | `utils.ts:110-118` | 外层缓存键的生成逻辑 |
| ShapeCache 失效 | `Fonts.ts:131` | 与 charWidth 同步清除 |
| 主动加载链路 | `App.tsx:3012-3014` | `loadSceneFonts().then(onLoaded)` |
| 事件监听链路 | `App.tsx:3302-3310` | `document.fonts "loadingdone"` |
| Safari 特殊处理 | `App.tsx:4007-4012` | 粘贴时手动加载字体 |

---

## 九、总结

### 核心设计思想

1. **分层缓存**：
   - `loadedFontsCache` → 字体加载状态缓存（防止重复刷新）
   - `charWidth` → 字符宽度测量缓存（两层结构：FontString + UTF-16 首码元）
   - `ShapeCache` → 元素形状缓存（渲染性能）

2. **失效联动**：
   - 字体加载完成 → 同时清除 `ShapeCache` 和 `charWidth` 缓存
   - 保证测量值与实际渲染字体一致

3. **性能权衡**：
   - BMP 字符缓存（ASCII、CJK、符号、空白符）→ 性能收益大，风险低
   - 非 BMP 字符和多字符字符串不缓存 → 避免代理对索引问题和字距调整误差

4. **异步一致性**：
   - 字体异步加载期间使用 fallback 字体测量
   - 加载完成后通过两条入口链路触发缓存失效
   - 确保最终渲染与测量一致

5. **双链路容错**：
   - 主动加载（速度快）+ 事件监听（覆盖全）
   - 两条链路通过 `loadedFontsCache` 保证幂等性

### 三处关键事实校正

| 原错误理解 | 校正后事实 | 依据 |
|------------|------------|------|
| emoji 分单码位/多码位走不同路径 | **所有 emoji 都不走缓存路径** | `isSingleCharacter` 检查 `codePointAt(1) === undefined`，代理对字符不满足 |
| `charCodeAt(0)` 是 Unicode 码位 | `charCodeAt(0)` 是**第一个 UTF-16 码元**，仅 BMP 字符等于码位 | `'😀'.charCodeAt(0) = 55357` vs `codePointAt(0) = 128512` |
| offset 基于原始文本 | `start`/`end` 基于**归一化后**的文本，需输入本身为 NFC 才能正确映射 | 实际验证 NFD 输入时 offset 不匹配 |

### 为什么这套设计读起来不顺？

1. **职责分散**：缓存定义在 `textMeasurements.ts`，但清除逻辑在 `Fonts.ts`
2. **隐式依赖**：`charWidth.clearCache()` 被 `Fonts` 调用，但两者没有显式的依赖关系
3. **多层缓存**：`loadedFontsCache`、`charWidth`、`ShapeCache` 三级缓存，失效逻辑分散
4. **异步触发**：字体加载是异步的，且有两条入口链路，缓存失效时机不直观
5. **命名相似**：`loadedFontsCache` 和 `charWidth` 都是缓存，但层级和用途不同
6. **两层索引**：`charWidth` 使用 FontString + UTF-16 首码元的两层结构，增加理解难度
7. **注释与实现不符**：`isSingleCharacter` 注释说是 "single codepoint"，实际是 "single BMP codepoint"
8. **隐藏前提**：NFC 归一化改变文本长度，offset 语义依赖输入为 NFC 形式

理解这套机制的关键是抓住 **"字体加载 → 两条链路 → 缓存失效 → 重新测量"** 这条主线，以及区分：
- **外层键**（FontString）vs **内层索引**（UTF-16 首码元）
- **BMP 字符**（走缓存）vs **非 BMP/多字符**（走直接测量）
- **主动加载**（速度快）vs **事件监听**（覆盖全）
- **归一化文本** vs **原始文本**（offset 语义差异）
