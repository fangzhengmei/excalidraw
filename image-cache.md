# Excalidraw 图片缓存机制

本文档详细说明 Excalidraw 中图片元素与底层文件缓存的引用关系、去重策略以及回收机制。

## 一、图片上传与缓存键生成

### 1.1 文件 ID 生成策略

图片文件的唯一标识（`FileId`）通过 `generateIdFromFile` 函数生成，位于 `packages/excalidraw/data/blob.ts:260-272`：

```typescript
export const generateIdFromFile = async (file: File): Promise<FileId> => {
  try {
    const hashBuffer = await window.crypto.subtle.digest(
      "SHA-1",
      await blobToArrayBuffer(file),
    );
    return bytesToHexString(new Uint8Array(hashBuffer)) as FileId;
  } catch (error: any) {
    console.error(error);
    // length 40 to align with the HEX length of SHA-1 (which is 160 bit)
    return nanoid(40) as FileId;
  }
};
```

**关键特性：**
- **基于内容哈希**：优先使用 SHA-1 哈希算法对文件二进制内容计算哈希值
- **确定性**：相同内容的文件必然生成相同的 `FileId`
- **回退机制**：若 crypto API 不可用，使用 40 字符的 nanoid（与 SHA-1 哈希长度一致）
- **时机**：在图片压缩/调整大小之前生成，保证 ID 可移植性

### 1.2 自定义生成器支持

宿主应用可以通过 `generateIdForFile` prop 自定义 ID 生成逻辑，位于 `packages/excalidraw/types.ts:652`：

```typescript
generateIdForFile?: (file: File) => string | Promise<string>;
```

## 二、图片元素到文件的引用关系

### 2.1 数据结构分层

Excalidraw 使用三层架构实现图片元素与文件的解耦：

| 层级 | 数据结构 | 位置 | 作用 |
|------|----------|------|------|
| 元素层 | `ExcalidrawImageElement` | `packages/element/src/types.ts` | 画布上的图片元素，仅存储 `fileId` 引用 |
| 文件存储层 | `BinaryFiles` | `packages/excalidraw/types.ts` | 二进制文件数据存储，`fileId` 为键 |
| 渲染缓存层 | `imageCache: Map<FileId, { image: HTMLImageElement | Promise<HTMLImageElement>, mimeType }>` | `AppClassProperties` | 已解码的图片对象缓存，用于快速渲染 |

### 2.2 图片元素结构

```typescript
export type ExcalidrawImageElement = _ExcalidrawElementBase &
  Readonly<{
    type: "image";
    fileId: FileId | null;          // 指向二进制文件的引用
    status: "pending" | "saved" | "error";  // 文件持久化状态
    scale: [number, number];        // 缩放因子，用于翻转
    crop: ImageCrop | null;         // 裁剪信息
  }>;
```

**核心设计：**
- 图片元素本身不存储二进制数据，只存储 `fileId` 引用
- 多个图片元素可以共享同一个 `fileId`，指向同一份文件数据
- `status` 字段跟踪文件是否已持久化到后端存储

### 2.3 二进制文件数据结构

```typescript
export type BinaryFileData = {
  mimeType: ValueOf<typeof IMAGE_MIME_TYPES> | typeof MIME_TYPES.binary;
  id: FileId;                        // 与元素的 fileId 对应
  dataURL: DataURL;                  // base64 编码的图片数据
  created: number;                   // 创建时间戳
  lastRetrieved?: number;            // 最后获取时间，用于回收
  version?: number;                  // 版本号，用于更新检测
};

export type BinaryFiles = Record<ExcalidrawElement["id"], BinaryFileData>;
```

## 三、去重机制实现

### 3.1 文件插入时去重

在 `App.tsx:4536-4573` 的 `addMissingFiles` 方法中实现文件级去重：

```typescript
private addMissingFiles = (
  files: BinaryFiles | BinaryFileData[],
  replace = false,
) => {
  const nextFiles = replace ? {} : { ...this.files };
  const addedFiles: BinaryFiles = {};

  const _files = Array.isArray(files) ? files : Object.values(files);

  for (const fileData of _files) {
    // 关键：如果文件已存在则跳过
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

### 3.2 图片初始化时去重

在 `App.tsx:11684-11804` 的 `initializeImage` 方法中：

```typescript
private initializeImage = async (
  placeholderImageElement: ExcalidrawImageElement,
  imageFile: File,
) => {
  // 1. 生成 fileId（基于文件内容哈希）
  const fileId = await ((this.props.generateIdForFile?.(
    imageFile,
  ) as Promise<FileId>) || generateIdFromFile(imageFile));

  // 2. 关键：检查文件是否已存在
  const existingFileData = this.files[fileId];
  if (!existingFileData?.dataURL) {
    // 新文件：进行压缩、大小检查等处理
    // ...
  }

  // 3. 复用已有 dataURL 或生成新的
  const dataURL =
    this.files[fileId]?.dataURL || (await getDataURL(imageFile));

  // 4. 仅添加缺失的文件
  this.addMissingFiles([
    {
      mimeType,
      id: fileId,
      dataURL,
      created: Date.now(),
      lastRetrieved: Date.now(),
    },
  ]);

  // 5. 渲染缓存去重：仅当缓存中不存在时才更新
  if (!this.imageCache.get(fileId)) {
    this.addNewImagesToImageCache();
    await this.updateImageCache([initializedImageElement]);
  }
  // ...
};
```

### 3.3 渲染缓存更新

在 `packages/element/src/image.ts:36-89` 的 `updateImageCache` 中：

```typescript
export const updateImageCache = async ({
  fileIds,
  files,
  imageCache,
}: {
  fileIds: FileId[];
  files: BinaryFiles;
  imageCache: AppClassProperties["imageCache"];
}) => {
  const updatedFiles = new Map<FileId, true>();
  const erroredFiles = new Map<FileId, true>();

  await Promise.all(
    fileIds.reduce((promises, fileId) => {
      const fileData = files[fileId as string];
      // 仅处理未更新的文件
      if (fileData && !updatedFiles.has(fileId)) {
        updatedFiles.set(fileId, true);
        return promises.concat(
          (async () => {
            // 加载图片到缓存...
          })(),
        );
      }
      return promises;
    }, [] as Promise<any>[]),
  );

  return { imageCache, updatedFiles, erroredFiles };
};
```

### 3.4 剪贴板粘贴时的去重

在 `packages/excalidraw/clipboard.ts:142-192` 的 `serializeAsClipboardJSON` 中：

```typescript
export const serializeAsClipboardJSON = ({
  elements,
  files,
}: {
  elements: readonly NonDeleted<ExcalidrawElement>[];
  files: BinaryFiles | null;
}) => {
  const _files = elements.reduce((acc, element) => {
    if (isInitializedImageElement(element)) {
      // 仅收集图片元素引用的文件，重复引用的 fileId 只会收集一次
      if (files && files[element.fileId]) {
        acc[element.fileId] = files[element.fileId];
      }
    }
    return acc;
  }, {} as BinaryFiles);

  // 序列化...
};
```

**粘贴时的去重：**
- 粘贴时通过 `addMissingFiles` 方法自动去重
- 相同 `fileId` 的文件不会重复存储
- 多个粘贴的图片元素共享同一份文件数据

## 四、缓存回收与生命周期管理

### 4.1 文件状态追踪

`FileManager` 类（`excalidraw-app/data/FileManager.ts`）追踪文件生命周期：

```typescript
export class FileManager {
  private fetchingFiles = new Map<FileId, true>();    // 正在获取
  private savingFiles = new Map<FileId, FileVersion>; // 正在保存
  private savedFiles = new Map<FileId, FileVersion>;  // 已保存
  private erroredFiles_fetch = new Map<FileId, true>; // 获取失败
  private erroredFiles_save = new Map<FileId, FileVersion>; // 保存失败
}
```

### 4.2 未使用文件的回收策略

**场景 1：场景导入/导出时的文件修剪**

导出场景时，仅包含当前元素引用的文件：
```typescript
// 在 serializeAsClipboardJSON 中：仅收集元素实际引用的文件
const _files = elements.reduce((acc, element) => {
  if (isInitializedImageElement(element) && files && files[element.fileId]) {
    acc[element.fileId] = files[element.fileId];
  }
  return acc;
}, {} as BinaryFiles);
```

**场景 2：基于访问时间的回收**

`BinaryFileData` 中的 `lastRetrieved` 字段用于标记文件最后访问时间：
```typescript
export type BinaryFileData = {
  // ...
  lastRetrieved?: number;  // 最后获取时间戳
};
```

回收策略（由持久化层实现）：
- 定期扫描 `lastRetrieved` 时间
- 删除长时间未被引用的文件
- 通过 `FileStatusStore` 追踪文件加载状态

### 4.3 图片缓存清理

在 `addFiles` 方法中，当新文件添加时会清理形状缓存：
```typescript
public addFiles: ExcalidrawImperativeAPI["addFiles"] = withBatchedUpdates(
  (files) => {
    const { addedFiles } = this.addMissingFiles(files);
    this.clearImageShapeCache(addedFiles);  // 清理相关渲染缓存
    this.scene.triggerUpdate();
    this.addNewImagesToImageCache();
  },
);
```

## 五、引用关系图示

```
┌─────────────────────────────────────────────────────────────────┐
│                        画布场景 (Scene)                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│  │  ImageElem1  │    │  ImageElem2  │    │  ImageElem3  │    │
│  │  fileId: A  │    │  fileId: A  │    │  fileId: B  │    │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘    │
│         │                    │                    │             │
│         └──────────┬─────────┘                    │             │
│                    │                              │             │
└────────────────────┼──────────────────────────────┼─────────────┘
                     │                              │
┌────────────────────▼──────────────────────────────▼─────────────┐
│                        files 存储对象                             │
├─────────────────────────────────────────────────────────────────┤
│  {                                                               │
│    A: { mimeType: "image/png", dataURL: "...", ... },           │
│    B: { mimeType: "image/jpg", dataURL: "...", ... }            │
│  }                                                               │
└────────────────────┬──────────────────────────────┬─────────────┘
                     │                              │
┌────────────────────▼──────────────────────────────▼─────────────┐
│                      imageCache (Map)                             │
├─────────────────────────────────────────────────────────────────┤
│  A -> { image: HTMLImageElement, mimeType: "image/png" }         │
│  B -> { image: HTMLImageElement, mimeType: "image/jpg" }         │
└─────────────────────────────────────────────────────────────────┘
```

## 六、关键设计总结

### 6.1 去重机制核心
1. **内容寻址**：通过 SHA-1 哈希生成 `FileId`，相同内容必然相同 ID
2. **引用共享**：多个图片元素可指向同一 `FileId`
3. **幂等操作**：`addMissingFiles` 自动跳过已存在文件
4. **缓存分层**：文件数据缓存与渲染对象缓存分离

### 6.2 空间效率
- **一份存储，多处引用**：N 个相同图片元素只占 1 份文件存储空间
- **延迟解码**：图片仅在实际渲染时才解码为 `HTMLImageElement`
- **剪贴板优化**：复制粘贴时不重复传输相同文件数据

### 6.3 性能优化
- **缓存命中**：重复插入相同图片时跳过压缩、解码等耗时操作
- **批量更新**：`updateImageCache` 批量处理，避免重复加载
- **状态追踪**：`FileManager` 避免重复网络请求

### 6.4 可扩展性
- 支持自定义 `generateIdForFile` 生成器
- `FileManager` 可适配不同持久化后端
- `lastRetrieved` 支持灵活的回收策略
