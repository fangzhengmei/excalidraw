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

位于 `packages/excalidraw/types.ts`：

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

## 四、去重机制实现

### 4.1 文件插入时去重

#### 核心函数：`addMissingFiles`

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
    // 关键去重逻辑：如果 fileId 已存在则跳过
    if (nextFiles[fileData.id]) {
      continue;
    }

    addedFiles[fileData.id] = fileData;
    nextFiles[fileData.id] = fileData;

    // SVG 规范化处理...
  }

  this.files = nextFiles;
  return { addedFiles };
};
```

**去重保证**：
- 基于 `fileData.id` (即 `FileId`) 的存在性检查
- 相同 `FileId` 的文件只会存储一份
- `replace = true` 时强制覆盖（用于场景导入场景）

---

### 4.2 复制阶段：只打包被引用文件

#### 流程概述

复制操作执行时，系统只收集当前被选中元素实际引用的文件，避免传输冗余数据。

#### 核心函数：`serializeAsClipboardJSON`

位于 `packages/excalidraw/clipboard.ts:142-192`：

```typescript
export const serializeAsClipboardJSON = ({
  elements,
  files,
}: {
  elements: readonly NonDeleted<ExcalidrawElement>[];
  files: BinaryFiles | null;
}) => {
  // 第一步：收集所有图片元素引用的 fileId
  const _files = elements.reduce((acc, element) => {
    if (isInitializedImageElement(element)) {
      // 关键：仅当文件存在且被元素引用时才加入剪贴板数据
      if (files && files[element.fileId]) {
        acc[element.fileId] = files[element.fileId];
      }
    }
    return acc;
  }, {} as BinaryFiles);

  // 第二步：序列化元素和引用的文件
  const contents: ElementsClipboard = {
    type: EXPORT_DATA_TYPES.excalidrawClipboard,
    elements: elements.map((element) => {
      // 处理跨帧复制时的 frameId 清理...
      return element;
    }),
    files: files ? _files : undefined,  // 只打包被引用的文件
  };

  return JSON.stringify(contents);
};
```

**复制阶段去重特性**：
1. **基于引用收集**：只包含被选中图片元素实际引用的文件
2. **自动去重**：多个元素引用同一 `FileId` 时，文件只在 reduce 中赋值一次
3. **按需打包**：剪贴板 JSON 中的 `files` 字段只包含必要数据

#### 调用链

```
用户按下 Ctrl+C
  ↓
copyToClipboard(elements, files)  clipboard.ts:194-209
  ↓
serializeAsClipboardJSON({ elements, files })
  ↓
只收集 elements 引用的 fileId 对应的文件数据
  ↓
写入系统剪贴板
```

---

### 4.3 粘贴/导入阶段：利用 fileId 去重落库

#### 流程概述

粘贴或导入时，系统先将新文件与本地已有文件进行比对，只添加真正缺失的文件。

#### 粘贴流程核心代码

位于 `packages/excalidraw/components/App.tsx:3867-4060`：

```typescript
public pasteFromClipboard = withBatchedUpdates(async (event) => {
  // 1. 解析剪贴板数据
  const dataTransferList = await parseDataTransferEvent(event);
  const data = await parseClipboard(dataTransferList, isPlainPaste);

  // 2. 插入剪贴板内容
  await this.insertClipboardContent(data, filesList, isPlainPaste);
});

// 粘贴时调用 addElementsFromPasteOrLibrary
addElementsFromPasteOrLibrary = (opts) => {
  // ... 元素复制与定位 ...

  // 关键：粘贴文件时自动去重
  if (opts.files) {
    this.addMissingFiles(opts.files);  // 自动跳过已存在的 fileId
  }

  // ... 选择新元素 ...

  // 确保新增图片的渲染缓存已初始化
  if (opts.files) {
    this.setState({}, () => {
      this.addNewImagesToImageCache();
    });
  }
};
```

#### 导入场景同步流程

位于 `packages/excalidraw/components/App.tsx:2776-2850`：

```typescript
public syncActionResult = withBatchedUpdates((actionResult) => {
  // ... 元素更新 ...

  // 文件去重导入
  if (actionResult.files) {
    // 调用 addMissingFiles，自动跳过本地已存在的 fileId
    this.addMissingFiles(actionResult.files, actionResult.replaceFiles);
    this.addNewImagesToImageCache();
  }

  // ... 状态更新 ...
});
```

#### 渲染缓存去重

位于 `packages/excalidraw/components/App.tsx:11926-11954`：

```typescript
private addNewImagesToImageCache = async (
  imageElements: InitializedExcalidrawImageElement[] =
    getInitializedImageElements(this.scene.getNonDeletedElements()),
  files: BinaryFiles = this.files,
) => {
  // 筛选未缓存的图片
  const uncachedImageElements = imageElements.filter(
    (element) => !element.isDeleted && !this.imageCache.has(element.fileId),
  );

  // 仅更新未缓存的图片
  if (uncachedImageElements.length) {
    const { updatedFiles } = await this.updateImageCache(
      uncachedImageElements,
      files,
    );
    // ... 触发重绘 ...
  }
};
```

#### 图片初始化去重

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

---

### 4.4 粘贴/导入去重完整调用链

```
用户按下 Ctrl+V
  ↓
pasteFromClipboard(event)  App.tsx:3867
  ↓
parseClipboard(dataTransferList)  clipboard.ts
  ↓
从剪贴板 JSON 提取 elements 和 files
  ↓
addElementsFromPasteOrLibrary({ elements, files })
  ↓
├─ duplicateElements()  // 复制元素，保留原 fileId
└─ addMissingFiles(files)  // 自动跳过已存在的 fileId
    ↓
    对于每个 file in files:
      if (nextFiles[file.id]) → 跳过
      else → 添加到 this.files
  ↓
addNewImagesToImageCache()  // 渲染缓存也自动去重
  ↓
只处理 this.imageCache 中不存在的 fileId
```

**粘贴/导入阶段去重特性**：
1. **基于 fileId 比对**：相同内容的图片（相同 fileId）不会重复存储
2. **元素与文件分离去重**：元素会被复制多份，但文件只存一份
3. **渲染缓存延迟**：未被引用的图片不会被解码到渲染缓存
4. **支持 replace 模式**：场景导入时可选择强制覆盖现有文件

## 五、缓存回收与生命周期管理

### 5.1 回收触发事件与时机

#### 事件 1：导出/分享场景时的文件修剪

**触发时机**：用户执行 JSON 导出、创建分享链接、保存到后端等操作时。

**核心逻辑**：只导出当前场景中元素实际引用的文件。

**代码位置**：
- `excalidraw-app/data/index.ts:244-291`（分享链接导出）
- `packages/excalidraw/clipboard.ts:142-192`（复制操作）
- `packages/excalidraw/data/json.ts`（序列化）

```typescript
// exportToBackend 中的文件收集逻辑
export const exportToBackend = async (elements, appState, files) => {
  const filesMap = new Map<FileId, BinaryFileData>();

  // 关键：只收集当前场景中元素实际引用的文件
  for (const element of elements) {
    if (isInitializedImageElement(element) && files[element.fileId]) {
      filesMap.set(element.fileId, files[element.fileId]);
    }
  }

  // 仅上传被引用的文件
  const filesToUpload = await encodeFilesForUpload({ files: filesMap, ... });
  // ...
};
```

**回收判断依据**：
- 输入：`当前画布元素集合` → 提取所有 `InitializedExcalidrawImageElement`
- 提取：收集这些元素的 `fileId` 集合（自动去重）
- 输出：仅导出 `fileId` 在此集合中的文件

---

#### 事件 2：协作场景同步时的文件过滤

**触发时机**：多用户协作场景下接收远程场景更新时。

**核心逻辑**：远程场景数据中未引用的文件不会被添加到本地缓存。

**代码位置**：`packages/excalidraw/data/index.ts:202-242`

---

#### 事件 3：持久化层的垃圾回收

**触发时机**：后台定时任务或存储空间不足时（由宿主应用实现）。

**核心逻辑**：基于 `lastRetrieved` 时间戳和引用计数判断文件是否可回收。

**回收判断条件**：

```
回收候选文件满足以下所有条件：
├─ 1. fileId 不在当前画布元素的引用集合中
├─ 2. Date.now() - file.lastRetrieved > 回收阈值（如 30 天）
└─ 3. 文件已成功持久化到后端（status = "saved"）
```

**当前画布 fileId 集合的生成**：

位于 `packages/element/src/image.ts:91-96`：

```typescript
export const getInitializedImageElements = (
  elements: readonly ExcalidrawElement[],
) =>
  elements.filter((element) =>
    isInitializedImageElement(element),
  ) as InitializedExcalidrawImageElement[];

// 生成当前画布 fileId 集合
const activeFileIds = new Set(
  getInitializedImageElements(sceneElements).map(el => el.fileId)
);
```

**回收算法伪代码**：

```typescript
function garbageCollectFiles(files: BinaryFiles, elements: ExcalidrawElement[]) {
  // 步骤 1：获取当前画布所有引用的 fileId 集合
  const activeFileIds = new Set(
    getInitializedImageElements(elements).map(el => el.fileId)
  );

  // 步骤 2：遍历所有文件，判断每个文件是否可回收
  const toDelete: FileId[] = [];

  for (const fileId in files) {
    const file = files[fileId];

    // 判断条件
    const isReferenced = activeFileIds.has(fileId);
    const isExpired = Date.now() - (file.lastRetrieved || 0) > THIRTY_DAYS;
    const isSaved = fileManager.isFileSavedOrBeingSaved(file);

    if (!isReferenced && isExpired && isSaved) {
      toDelete.push(fileId);
    }
  }

  // 步骤 3：执行删除
  for (const fileId of toDelete) {
    delete files[fileId];      // 从文件存储层删除
    imageCache.delete(fileId); // 从渲染缓存删除
  }
}
```

---

#### 事件 4：渲染缓存的懒清理

**触发时机**：调用 `addNewImagesToImageCache` 或 `updateImageCache` 时。

**核心逻辑**：只加载当前未删除且未缓存的图片，隐式清理不再需要的渲染缓存。

**代码位置**：`packages/excalidraw/components/App.tsx:11926-11954`

```typescript
private addNewImagesToImageCache = async (
  imageElements: InitializedExcalidrawImageElement[] =
    getInitializedImageElements(this.scene.getNonDeletedElements()),
  files: BinaryFiles = this.files,
) => {
  // 只处理：未删除 + 未缓存的图片
  const uncachedImageElements = imageElements.filter(
    (element) => !element.isDeleted && !this.imageCache.has(element.fileId),
  );

  // 仅更新需要的图片
  if (uncachedImageElements.length) {
    await this.updateImageCache(uncachedImageElements, files);
    // ...
  }
};
```

**懒清理特性**：
- 不主动删除缓存，只在需要时按需添加
- 元素被删除后，其 `fileId` 不会出现在下一次的 `imageElements` 列表中
- 长时间未访问的渲染缓存条目可由宿主应用实现 LRU 清理

### 5.2 文件状态追踪

`FileManager` 类追踪文件生命周期状态，位于 `excalidraw-app/data/FileManager.ts`：

```typescript
export class FileManager {
  private fetchingFiles = new Map<FileId, true>();    // 正在从后端获取
  private savingFiles = new Map<FileId, FileVersion>; // 正在保存
  private savedFiles = new Map<FileId, FileVersion>;  // 已保存到持久化
  private erroredFiles_fetch = new Map<FileId, true>; // 获取失败
  private erroredFiles_save = new Map<FileId, FileVersion>; // 保存失败

  // 判断文件是否可安全回收
  isFileSavedOrBeingSaved = (file: BinaryFileData) => {
    const fileVersion = this.getFileVersion(file);
    return (
      this.savedFiles.get(file.id) === fileVersion ||
      this.savingFiles.get(file.id) === fileVersion
    );
  };
}
```

**状态与回收的关系**：
- `savedFiles` 中的文件：已持久化，可参与回收判断
- `savingFiles` 中的文件：正在上传，不可回收
- `fetchingFiles` 中的文件：正在下载，不可回收

### 5.3 已删除元素的文件处理

位于 `excalidraw-app/data/index.ts:46-56`：

```typescript
export const isSyncableElement = (
  element: OrderedExcalidrawElement,
): element is SyncableExcalidrawElement => {
  if (element.isDeleted) {
    // 已删除元素保留 DELETED_ELEMENT_TIMEOUT（默认 7 天）用于撤销/重做
    if (element.updated > Date.now() - DELETED_ELEMENT_TIMEOUT) {
      return true;  // 在此期间文件仍被保留
    }
    return false;  // 超时后文件可回收
  }
  return !isInvisiblySmallElement(element);
};
```

**文件生命周期与元素生命周期的关系**：

```
图片元素创建
    ↓
fileId 生成，文件添加到 files，渲染缓存按需加载
    ↓
用户删除元素（isDeleted = true）
    ↓
元素仍保留在场景中 DELETED_ELEMENT_TIMEOUT
    ↓ 期间用户执行撤销操作：元素恢复，文件仍存在
    ↓ 期间用户未执行撤销：
元素从场景中永久移除
    ↓
文件进入可回收候选池
    ↓
lastRetrieved 超时后被真正回收
```

## 六、关键流程详解

### 6.1 完整图片插入流程

```
用户拖拽图片文件到画布
    ↓
handleDrop() → parseDataTransferEvent()
    ↓
insertImages(imageFiles, sceneX, sceneY)
    ↓
1. 创建占位符元素并插入场景
   └─ newImagePlaceholder({ sceneX, sceneY })
    ↓
2. 并发初始化每个图片
   ├─ normalizeFile(file)          // MIME 类型检测与修正
   ├─ generateIdFromFile(file)    // 基于内容生成 fileId（去重关键）
   ├─ 检查 this.files[fileId] 是否已存在
   │   ├─ 已存在 → 复用 dataURL
   │   └─ 不存在 → resizeImageFile() 压缩后生成 dataURL
   ├─ addMissingFiles()            // 添加缺失的文件（去重）
   └─ addNewImagesToImageCache()   // 按需加载渲染缓存
    ↓
3. 定位已初始化的元素
    ↓
完成：画布上显示图片
```

### 6.2 复制粘贴去重流程

```
复制阶段
├─ 用户选择 N 张图片（其中 M 张引用相同 fileId）
├─ serializeAsClipboardJSON(elements, files)
└─ 结果：剪贴板只包含 M 个唯一文件（而非 N 个）

粘贴阶段
├─ pasteFromClipboard() 解析剪贴板数据
├─ duplicateElements() 创建 N 个新元素（都保留原 fileId 引用）
├─ addMissingFiles(pastedFiles) 检查每个 fileId
│   ├─ 已存在 → 跳过
│   └─ 不存在 → 添加到 this.files
└─ addNewImagesToImageCache() 更新渲染缓存

结果：N 个元素共享 <= M 个文件数据
```

### 6.3 场景导入回收流程

```
导入场景 JSON
    ↓
syncActionResult({ elements, files, replaceFiles })
    ↓
1. 替换场景元素集合
    ↓
2. addMissingFiles(files, replaceFiles)
   ├─ replaceFiles = false: 只追加，跳过本地已存在的 fileId
   └─ replaceFiles = true: 全量替换，本地未被引用的文件会被清除
    ↓
3. addNewImagesToImageCache()
   └─ 只加载新场景中引用且未缓存的图片
    ↓
结果：导入后 files 和 imageCache 都只包含新场景需要的数据
```

## 七、性能与空间效率总结

### 7.1 空间效率

| 场景 | 无去重 | 有去重 | 节省比例 |
|------|--------|--------|----------|
| 相同图片粘贴 10 次 | 10 份文件数据 | 1 份文件数据 | ~90% |
| 导出包含重复图片的场景 | N 份文件 | M 个唯一文件 | (N-M)/N |
| 协作同步重复图片 | 每次同步全量 | 只同步缺失的 | 视重复率而定 |

### 7.2 性能优化点

1. **缓存命中**：重复插入相同图片时跳过下载、压缩、解码等耗时操作
2. **批量更新**：`updateImageCache` 批量处理，避免重复加载
3. **状态追踪**：`FileManager` 避免重复网络请求
4. **延迟解码**：图片仅在实际渲染时才解码为 `HTMLImageElement`

### 7.3 边界情况处理

| 情况 | 处理方式 |
|------|----------|
| 文件内容相同但文件名不同 | 相同 SHA-1 → 相同 fileId → 去重 |
| 本地文件与远程文件内容相同 | 相同 fileId → 复用本地数据 |
| SVG 规范化前后内容变化 | 更新 version 字段，重新存储 |
| crypto API 不可用 | 降级为随机 ID，但失去内容寻址去重能力 |

## 八、扩展与自定义

### 8.1 自定义回收策略

宿主应用可通过以下方式实现自定义回收：

```typescript
// 示例：基于 LRU 的渲染缓存回收
const MAX_CACHE_SIZE = 100;
const lruQueue = new Map<FileId, number>(); // fileId → 最后访问时间

function onImageAccessed(fileId) {
  lruQueue.delete(fileId);
  lruQueue.set(fileId, Date.now());

  if (lruQueue.size > MAX_CACHE_SIZE) {
    const oldestKey = lruQueue.keys().next().value;
    lruQueue.delete(oldestKey);
    imageCache.delete(oldestKey);
  }
}
```

### 8.2 自定义文件持久化

通过 `FileManager` 的构造参数注入自定义持久化实现：

```typescript
const fileManager = new FileManager({
  getFiles: async (fileIds) => { /* 自定义获取逻辑 */ },
  saveFiles: async ({ addedFiles }) => { /* 自定义保存逻辑 */ },
  onFileStatusChange: (updates) => { /* 状态变化回调 */ },
});
```
