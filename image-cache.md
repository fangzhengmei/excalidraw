# Excalidraw 图片缓存机制

本文档详细说明 Excalidraw 中图片元素与底层文件缓存的引用关系、去重策略以及回收机制。

## 一、整体架构

### 1.1 三层引用结构

```
┌───────────────────────────────────────────────────────────────┐
│                     图片元素层 (Element Layer)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │ ImageElem A  │  │ ImageElem B  │  │ ImageElem C  │        │
│  │ fileId: X   │  │ fileId: X   │  │ fileId: Y   │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│                     文件数据层 (File Layer)                     │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  files: {                                               │  │
│  │    X: { dataURL: "data:image/png;base64,...", ... },   │  │
│  │    Y: { dataURL: "data:image/jpg;base64,...", ... }    │  │
│  │  }                                                      │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────┬────────────────────────────────────┘
                           │
┌──────────────────────────▼────────────────────────────────────┐
│                  渲染缓存层 (Render Cache Layer)                │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  imageCache: Map<FileId, {                              │  │
│  │    image: HTMLImageElement | Promise<HTMLImageElement> │  │
│  │    mimeType: string                                     │  │
│  │  }>                                                     │  │
│  └────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────┘
```

### 1.2 核心设计原则

- **内容寻址**：通过文件内容 SHA-1 哈希生成 `FileId`，相同内容必然相同 ID
- **引用共享**：多个图片元素可指向同一 `FileId`，实现数据去重
- **分层缓存**：原始文件数据与解码后的渲染对象分离存储
- **延迟解码**：图片仅在实际渲染需要时才解码为 `HTMLImageElement`

## 二、图片上传与缓存键生成

### 2.1 FileId 生成算法

位于 `packages/excalidraw/data/blob.ts:260-272`：

```typescript
export const generateIdFromFile = async (file: File): Promise<FileId> => {
  try {
    // 使用 SHA-1 哈希文件内容，确保相同内容生成相同 ID
    const hashBuffer = await window.crypto.subtle.digest(
      "SHA-1",
      await blobToArrayBuffer(file),
    );
    return bytesToHexString(new Uint8Array(hashBuffer)) as FileId;
  } catch (error: any) {
    console.error(error);
    // 降级方案：生成 40 字符随机字符串，与 SHA-1 长度一致
    return nanoid(40) as FileId;
  }
};
```

**关键特性**：
- **确定性**：相同内容 → 相同哈希 → 相同 `FileId`
- **时机**：在图片压缩、调整大小之前生成，保证 ID 可移植性
- **兼容性**：支持浏览器 crypto API 降级方案

### 2.2 自定义生成器支持

宿主应用可通过 `generateIdForFile` prop 覆盖默认行为，位于 `packages/excalidraw/types.ts:652`：

```typescript
generateIdForFile?: (file: File) => string | Promise<string>;
```

## 三、元素到文件的引用关系

### 3.1 图片元素结构

位于 `packages/element/src/types.ts`：

```typescript
export type ExcalidrawImageElement = _ExcalidrawElementBase &
  Readonly<{
    type: "image";
    fileId: FileId | null;          // 指向二进制文件的引用
    status: "pending" | "saved" | "error";  // 文件持久化状态
    scale: [number, number];        // 缩放因子，用于翻转
    crop: ImageCrop | null;         // 裁剪信息
  }>;

// 已初始化的图片元素（fileId 非空）
export type InitializedExcalidrawImageElement = MarkNonNullable<
  ExcalidrawImageElement,
  "fileId"
>;
```

### 3.2 二进制文件数据结构

位于 `packages/excalidraw/types.ts:125-132`：

```typescript
export type BinaryFileData = {
  mimeType: ValueOf<typeof IMAGE_MIME_TYPES>;
  id: FileId;                        // 与元素的 fileId 对应
  dataURL: DataURL;                  // base64 编码的图片数据
  created: number;                   // 创建时间戳（ms）
  lastRetrieved?: number;            // 最后获取时间，用于回收判断
  version?: number;                  // 版本号，用于更新检测
};

export type BinaryFiles = Record<ExcalidrawElement["id"], BinaryFileData>;
```

### 3.3 渲染缓存结构

位于 `packages/excalidraw/components/App.tsx:795`：

```typescript
imageCache: Map<
  FileId,
  {
    image: HTMLImageElement | Promise<HTMLImageElement>;
    mimeType: ValueOf<typeof IMAGE_MIME_TYPES>;
  }
>;
```

## 四、复制阶段：完整调用链与去重

### 4.1 复制完整调用链（按真实代码排序）

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. 事件监听注册                                                    │
│     位置：App.tsx:3295                                             │
│     addEventListener(document, EVENT.COPY, this.onCopy, {})        │
└──────────────────────────┬──────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│  2. 复制事件入口：onCopy                                            │
│     位置：App.tsx:3606-3616                                         │
│     this.actionManager.executeAction(actionCopy, "keyboard", event) │
└──────────────────────────┬──────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│  3. Action 执行：actionCopy.perform                                 │
│     位置：actions/actionClipboard.tsx:28-52                        │
│     步骤：                                                          │
│     - 获取选中元素：app.scene.getSelectedElements({...})           │
│     - 调用：copyToClipboard(elementsToCopy, app.files, event)      │
└──────────────────────────┬──────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│  4. 剪贴板封装入口：copyToClipboard                                  │
│     位置：clipboard.ts:194-209                                      │
│     ┌─────────────────────────────────────────────────────────────┐ │
│     │  子步骤 4a) **去重发生点**：serializeAsClipboardJSON           │ │
│     │     位置：clipboard.ts:142-192                                │ │
│     │     核心逻辑：reduce 遍历元素，相同 fileId 只保留一份         │ │
│     │     输出：序列化 json 字符串（包含去重后的 files 集合）        │ │
│     └─────────────────────────────────────────────────────────────┘ │
│                          ↓                                           │
│     子步骤 4b) 构造双 MIME 参数对象：                               │
│     {                                                               │
│       [MIME_TYPES.excalidrawClipboard]: json,                      │
│       [MIME_TYPES.text]: json,                                     │
│     }                                                               │
│                          ↓                                           │
│     子步骤 4c) 调用 copyTextToSystemClipboard(对象, clipboardEvent) │
└──────────────────────────┬──────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────────┐
│  5. 写入系统剪贴板：copyTextToSystemClipboard                       │
│     位置：clipboard.ts:202-208（调用），clipboard.ts:586-636（定义）│
│     遍历对象的每个 MIME 键值对，调用 clipboardData.setData          │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 4.2 复制入口与代码位置对应表

| 调用顺序 | 函数/方法 | 代码位置 | 职责 | 关键参数 |
|---------|----------|---------|------|----------|
| 1 | `onCopy` | `App.tsx:3606-3616` | 事件入口，判断是否激活 Excalidraw | `event: ClipboardEvent` |
| 2 | `actionCopy.perform` | `actions/actionClipboard.tsx:28-52` | 获取选中元素，调用复制逻辑 | 传递 `clipboardEvent` |
| 3 | `copyToClipboard` | `clipboard.ts:194-209` | 封装剪贴板数据格式 | `elements, files, clipboardEvent` |
| 4 | `serializeAsClipboardJSON` | `clipboard.ts:142-192` | **去重发生点**，打包文件 | 返回序列化 `json` |
| 5 | `copyTextToSystemClipboard` | `clipboard.ts:202-208`（调用）<br>`clipboard.ts:586-636`（定义） | 双 MIME 类型写入系统剪贴板 | **对象参数**：<br>`{ excalidrawClipboard: json, text: json }, clipboardEvent` |

---

### 4.3 去重发生点核心代码

位于 `packages/excalidraw/clipboard.ts:142-192`：

```typescript
export const serializeAsClipboardJSON = ({
  elements,
  files,
}: {
  elements: readonly NonDeleted<ExcalidrawElement>[];
  files: BinaryFiles | null;
}) => {
  // ┌─────────────────────────────────────────────────────────┐
  // │  **去重发生点核心逻辑**                                  │
  // │  reduce 遍历元素，相同 fileId 多次出现时只赋值一次        │
  // │  acc[element.fileId] = files[element.fileId]             │
  // │  由于对象属性覆盖特性，相同 fileId 最终只保留一份         │
  // └─────────────────────────────────────────────────────────┘
  const _files = elements.reduce((acc, element) => {
    if (isInitializedImageElement(element)) {
      if (files && files[element.fileId]) {
        // 相同 fileId 重复引用时，这里会重复赋值
        // 但由于值相同，最终结果是幂等的，只保留一份
        acc[element.fileId] = files[element.fileId];
      }
    }
    return acc;
  }, {} as BinaryFiles);

  const contents: ElementsClipboard = {
    type: EXPORT_DATA_TYPES.excalidrawClipboard,
    elements: elements.map((element) => {
      // frameId 清理逻辑...
      return element;
    }),
    files: files ? _files : undefined,  // 去重后的文件集合
  };

  return JSON.stringify(contents);
};
```

**复制阶段去重特性**：
- **输入**：N 个图片元素（可能 M 个唯一 fileId）
- **输出**：M 个文件数据（fileId 唯一）
- **机制**：reduce + 对象属性覆盖自动去重
- **结果**：剪贴板中无冗余文件数据

---

### 4.4 双 MIME 类型写入机制

#### 真实调用前后关系

```
前置步骤（去重完成）
    ↓
serializeAsClipboardJSON({ elements, files })  → 返回字符串 json
    ↓
copyToClipboard 内部构造双 MIME 对象
    ↓
┌─────────────────────────────────────────────────────────────┐
│  **真实参数结构（对象）**：                                   │
│  {                                                           │
│    [MIME_TYPES.excalidrawClipboard]: json,  // 键 1：专用格式 │
│    [MIME_TYPES.text]: json,                 // 键 2：纯文本格式 │
│  }                                                           │
│  两份值完全相同，都是同一份序列化后的 json 字符串              │
└─────────────────────────────────────────────────────────────┘
    ↓
copyTextToSystemClipboard(对象, clipboardEvent)
    ↓
clipboardData.setData(mimeType, value)  // 遍历两个键分别写入
    ↓
写入系统剪贴板完成
```

#### 核心代码位置

**调用处（clipboard.ts:194-209）**：
```typescript
export const copyToClipboard = async (
  elements: readonly NonDeletedExcalidrawElement[],
  files: BinaryFiles | null,
  clipboardEvent?: ClipboardEvent | null,
) => {
  const json = serializeAsClipboardJSON({ elements, files });

  await copyTextToSystemClipboard(
    {
      [MIME_TYPES.excalidrawClipboard]: json,
      [MIME_TYPES.text]: json,
    },
    clipboardEvent,
  );
};
```

**定义处（clipboard.ts:586-636）**：
```typescript
export const copyTextToSystemClipboard = async <
  MimeType extends ValueOf<typeof STRING_MIME_TYPES>,
>(
  text: string | { [K in MimeType]: string } | null,  // ← 支持对象参数
  clipboardEvent?: ClipboardEvent | null,
) => {
  text = text || "";

  const entries = Object.entries(
    typeof text === "string" ? { [MIME_TYPES.text]: text } : text,
  );

  // 遍历对象的每个键值对，分别写入剪贴板
  if (clipboardEvent) {
    for (const [mimeType, value] of entries) {
      clipboardEvent.clipboardData?.setData(mimeType, value);
    }
    return;
  }
  // ... navigator.clipboard 降级方案
};
```

**双 MIME 设计目的**：
- `excalidrawClipboard`：供 Excalidraw 自身识别，可完整还原元素+文件
- `text`：纯文本格式，供其他应用粘贴（如文本编辑器、邮件等）
- 两份值完全相同，保证兼容性的同时不增加数据量

---

### 4.5 复制流程图示

```
用户选中图片元素（Img1、Img2、Img3）
    ↓  其中 Img1.fileId = Img2.fileId = X, Img3.fileId = Y
用户按下 Ctrl+C
    ↓
document COPY 事件触发 → onCopy()  App.tsx:3606
    ↓
actionCopy.perform()  actionClipboard.tsx:28
    ↓
copyToClipboard([Img1, Img2, Img3], files, event)  clipboard.ts:194
    ↓
serializeAsClipboardJSON({ elements, files })  clipboard.ts:142
    ↓
┌─────────────────────────────────────────────────────┐
│  reduce 遍历：                                        │
│  Img1 (fileId=X) → acc[X] = files[X]                │
│  Img2 (fileId=X) → acc[X] = files[X]  (覆盖，值相同) │
│  Img3 (fileId=Y) → acc[Y] = files[Y]                │
│  结果：acc = { X: data, Y: data }  ← 去重完成       │
└─────────────────────────────────────────────────────┘
    ↓
copyTextToSystemClipboard(
  {
    [MIME_TYPES.excalidrawClipboard]: json,  // 专用格式
    [MIME_TYPES.text]: json,                 // 兼容格式
  },
  event
)  clipboard.ts:202-208
    ↓
同时写入两个 MIME 键到系统剪贴板完成
```

## 五、粘贴阶段：去重落库调用链

### 5.1 粘贴完整调用链（按真实代码排序）

```
用户按下 Ctrl+V
    ↓
pasteFromClipboard(event)  App.tsx:3867-3918
    ↓
parseClipboard(dataTransferList, isPlainPaste)  clipboard.ts
    ↓
extract elements 和 files 从剪贴板数据
    ↓
insertClipboardContent(data, filesList, isPlainPaste)  App.tsx:3700-3908
    ↓
判断粘贴类型（图片/元素/文本等）
    ↓
if (data.elements) → 执行元素粘贴
    ↓
addElementsFromPasteOrLibrary({ elements, files, ... })  App.tsx:3920-4020
    ↓
┌─────────────────────────────────────────────────────────┐
│  复制元素，保留 fileId 引用                               │
│  duplicateElements({ elements, ... })                    │
│  → 元素 id 会变，但 fileId 引用保持不变                   │
└──────────────────────────┬───────────────────────────────┘
                           ↓
将 duplicatedElements 添加到场景
    ↓
┌─────────────────────────────────────────────────────────┐
│  **去重发生点 2：文件去重落库**                           │
│  if (opts.files) {                                       │
│    this.addMissingFiles(opts.files);  // 关键            │
│  }                                                       │
│  addMissingFiles(files)  App.tsx:4536-4573               │
│  → 遍历每个 file，if (nextFiles[file.id]) → 跳过         │
│  → 不存在则添加到 this.files                              │
└──────────────────────────┬───────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  **去重发生点 3：渲染缓存去重**                           │
│  this.setState({}, () => {                               │
│    this.addNewImagesToImageCache();  // 关键              │
│  });                                                     │
│  addNewImagesToImageCache()  App.tsx:11926-11954         │
│  → 只处理 !element.isDeleted && !imageCache.has(fileId)  │
└─────────────────────────────────────────────────────────┘
    ↓
选中新粘贴的元素
    ↓
粘贴完成
```

---

### 5.2 粘贴去重核心代码

**去重发生点 2：文件数据去重落库**

位于 `packages/excalidraw/components/App.tsx:4536-4573`：

```typescript
private addMissingFiles = (
  files: BinaryFiles | BinaryFileData[],
  replace = false,
) => {
  const nextFiles = replace ? {} : { ...this.files };
  const addedFiles: BinaryFiles = {};

  const _files = Array.isArray(files) ? files : Object.values(files);

  for (const fileData of _files) {
    // 关键：基于 fileId 的存在性检查
    if (nextFiles[fileData.id]) {
      continue;  // 已存在，跳过，实现去重
    }

    addedFiles[fileData.id] = fileData;
    nextFiles[fileData.id] = fileData;

    // SVG 规范化处理...
  }

  this.files = nextFiles;
  return { addedFiles };
};
```

**去重发生点 3：渲染缓存去重**

位于 `packages/excalidraw/components/App.tsx:11926-11954`：

```typescript
private addNewImagesToImageCache = async (
  imageElements: InitializedExcalidrawImageElement[] =
    getInitializedImageElements(this.scene.getNonDeletedElements()),
  files: BinaryFiles = this.files,
) => {
  // 只处理未删除且未缓存的图片
  const uncachedImageElements = imageElements.filter(
    (element) => !element.isDeleted && !this.imageCache.has(element.fileId),
  );

  // 仅更新需要的图片
  if (uncachedImageElements.length) {
    const { updatedFiles } = await this.updateImageCache(
      uncachedImageElements,
      files,
    );
    // 触发重绘...
  }
};
```

**粘贴阶段去重特性**：
- 元素会被复制多份（生成新 id），但 fileId 引用保持不变
- `addMissingFiles` 基于 fileId 比对，相同内容不重复存储
- `addNewImagesToImageCache` 渲染缓存也自动去重
- 最终：N 个相同图片元素只占 1 份文件存储空间

---

### 5.3 粘贴去重流程图示

```
剪贴板数据：elements = [ImgA(X), ImgB(X), ImgC(Y)], files = { X: data, Y: data }
    ↓
pasteFromClipboard()  App.tsx:3867
    ↓
parseClipboard() → 解析出 elements 和 files
    ↓
insertClipboardContent()  App.tsx:3700
    ↓
addElementsFromPasteOrLibrary({ elements, files })  App.tsx:3920
    ↓
duplicateElements()
    → [ImgA'(id=new, fileId=X), ImgB'(id=new, fileId=X), ImgC'(id=new, fileId=Y)]
    ↓
addMissingFiles({ X: data, Y: data })
    ├─ if !this.files[X] → 添加
    └─ if !this.files[Y] → 添加
    ↓
addNewImagesToImageCache()
    ├─ if !imageCache.has(X) → 加载并缓存
    └─ if !imageCache.has(Y) → 加载并缓存
    ↓
结果：3 个元素共享 2 个文件数据，无冗余
```

## 六、缓存回收机制（基于真实代码）

### 6.1 回收触发时序

#### 触发事件：场景初次加载完成后

**触发位置**：`excalidraw-app/App.tsx:512-516`

```typescript
// loadImages 函数内
if (isInitialLoad) {
  // on fresh load, clear unused files from IDB (from previous session)
  LocalData.fileStorage.clearObsoleteFiles({
    currentFileIds: fileIds,
  });
}
```

**完整调用链路**：

```
页面刷新 / 初次加载
    ↓
initializeScene({ collabAPI, excalidrawAPI })  App.tsx:528-531
    ↓
恢复 elements 和 appState 到场景
    ↓
loadImages(data, /* isInitialLoad */ true)  App.tsx:529
    ↓
┌─────────────────────────────────────────────────────────────┐
│  计算当前场景引用的 fileId 集合：                             │
│  const fileIds = elements?.reduce((acc, element) => {       │
│    if (isInitializedImageElement(element)) {                 │
│      return acc.concat(element.fileId);                      │
│    }                                                         │
│    return acc;                                               │
│  }, [] as FileId[]) || [];                                   │
└─────────────────────────────────────────────────────────────┘
    ↓
从 IDB 加载文件数据并更新 lastRetrieved 时间戳
    ↓
┌─────────────────────────────────────────────────────────────┐
│  **回收触发点**：                                            │
│  if (isInitialLoad) {                                        │
│    LocalData.fileStorage.clearObsoleteFiles({                │
│      currentFileIds: fileIds,  // ← 当前场景引用的 fileId    │
│    });                                                       │
│  }                                                           │
└─────────────────────────────────────────────────────────────┘
```

---

### 6.2 回收判断逻辑（真实代码）

位于 `excalidraw-app/data/LocalData.ts:54-70`：

```typescript
class LocalFileManager extends FileManager {
  clearObsoleteFiles = async (opts: { currentFileIds: FileId[] }) => {
    await entries(filesStore).then((entries) => {
      for (const [id, imageData] of entries as [FileId, BinaryFileData][]) {
        // if image is unused (not on canvas) & is older than 1 day, delete
        // it from storage. We check `lastRetrieved` we care about the last
        // time the image was used (loaded on canvas), not when it was
        // initially created.

        // ┌─────────────────────────────────────────────────────┐
        // │  **回收判断条件（AND 关系）**：                      │
        // │  1. (!imageData.lastRetrieved                      │
        // │      || Date.now() - imageData.lastRetrieved       │
        // │         > 24 * 3600 * 1000)  // 超过 1 天          │
        // │  AND                                                │
        // │  2. !opts.currentFileIds.includes(id)  // 不在当前场景引用 │
        // └─────────────────────────────────────────────────────┘
        if (
          (!imageData.lastRetrieved ||
            Date.now() - imageData.lastRetrieved > 24 * 3600 * 1000) &&
          !opts.currentFileIds.includes(id as FileId)
        ) {
          del(id, filesStore);  // 从 IndexedDB 中删除
        }
      }
    });
  };
}
```

---

### 6.3 lastRetrieved 更新时机

位于 `excalidraw-app/data/LocalData.ts:179-198`：

```typescript
getFiles(ids) {
  return getMany(ids, filesStore).then(async (filesData) => {
    const filesToSave: [FileId, BinaryFileData][] = [];

    filesData.forEach((data, index) => {
      const id = ids[index];
      if (data) {
        const _data: BinaryFileData = {
          ...data,
          lastRetrieved: Date.now(),  // ← 更新最后访问时间
        };
        filesToSave.push([id, _data]);
      }
    });

    try {
      // save loaded files back to storage with updated `lastRetrieved`
      setMany(filesToSave, filesStore);  // 回写更新后的时间戳
    } catch (error) {
      console.warn(error);
    }

    return { loadedFiles, erroredFiles };
  });
}
```

**关键设计**：
- 每次从 IDB 加载文件时，自动更新 `lastRetrieved` 为当前时间
- 加载完成后立即回写到 IDB，确保下一次回收判断准确
- 只要图片被加载（被使用），就会重置 1 天的回收计时器

---

### 6.4 回收机制总结

| 项目 | 真实值 |
|------|--------|
| **触发时机** | 场景初次加载完成后（`isInitialLoad = true`） |
| **触发位置** | `excalidraw-app/App.tsx:514-516` |
| **判断条件 1** | `!lastRetrieved` 或 `Date.now() - lastRetrieved > 1 天` |
| **判断条件 2** | `fileId` 不在 `currentFileIds`（当前场景引用集合）中 |
| **删除范围** | IndexedDB 中的文件存储（不影响内存中的 `this.files`） |
| **重置机制** | 每次加载文件时更新 `lastRetrieved = Date.now()` |

**回收流程图**：

```
页面刷新 / 初次加载场景
    ↓
┌─────────────────────────────────────────┐
│  计算 currentFileIds = 场景中所有图片的 fileId  │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  遍历 IDB 中所有文件 entries            │
│  for each [id, imageData] in entries:   │
└─────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────┐
│  判断回收条件（AND）：                   │
│  ├─ (无 lastRetrieved  OR 超过 1 天)    │
│  └─ AND 不在 currentFileIds 中          │
└─────────────────────────────────────────┘
         │              │
      满足           不满足
         │              │
         ▼              ▼
  del(id, filesStore)  保留
```

## 七、文件插入去重机制

### 7.1 图片初始化时去重

位于 `packages/excalidraw/components/App.tsx:11684-11804`：

```typescript
private initializeImage = async (placeholder, imageFile) => {
  // 1. 基于文件内容生成 FileId
  const fileId = await generateIdFromFile(imageFile);

  // 2. 检查文件是否已存在
  const existingFileData = this.files[fileId];
  if (!existingFileData?.dataURL) {
    // 文件不存在：进行压缩、大小检查等处理
  }

  // 3. 复用已有 dataURL 或生成新的
  const dataURL = this.files[fileId]?.dataURL || await getDataURL(imageFile);

  // 4. 仅添加缺失的文件（自动去重）
  this.addMissingFiles([{ mimeType, id: fileId, dataURL, ... }]);

  // 5. 渲染缓存去重：仅当缓存中不存在时才更新
  if (!this.imageCache.get(fileId)) {
    this.addNewImagesToImageCache();
    await this.updateImageCache([initializedImageElement]);
  }

  // ...
};
```

## 八、性能与空间效率总结

### 8.1 空间效率

| 场景 | 无去重 | 有去重 | 节省比例 |
|------|--------|--------|----------|
| 相同图片粘贴 10 次 | 10 份文件数据 | 1 份文件数据 | ~90% |
| 导出包含重复图片的场景 | N 份文件 | M 个唯一文件 | (N-M)/N |
| 协作同步重复图片 | 每次同步全量 | 只同步缺失的 | 视重复率而定 |

### 8.2 性能优化点

1. **缓存命中**：重复插入相同图片时跳过下载、压缩、解码等耗时操作
2. **批量更新**：`updateImageCache` 批量处理，避免重复加载
3. **状态追踪**：`FileManager` 避免重复网络请求
4. **延迟解码**：图片仅在实际渲染时才解码为 `HTMLImageElement`

### 8.3 边界情况处理

| 情况 | 处理方式 |
|------|----------|
| 文件内容相同但文件名不同 | 相同 SHA-1 → 相同 fileId → 去重 |
| 本地文件与远程文件内容相同 | 相同 fileId → 复用本地数据 |
| SVG 规范化前后内容变化 | 更新 version 字段，重新存储 |
| crypto API 不可用 | 降级为随机 ID，但失去内容寻址去重能力 |
