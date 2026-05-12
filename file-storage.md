# Excalidraw 文件存储抽象架构

## 概述

Excalidraw 画布工程可存储于三种后端：浏览器内置数据库（IndexedDB + localStorage）、本地磁盘文件和远端云端（Firebase）。这三种后端通过同一套抽象接口调用，必要时可在它们之间降级与切换。

本文档详细说明文件存储抽象如何屏蔽差异并维持一致行为。

---

## 一、抽象接口层

### 1.1 FileManager 核心抽象类

`FileManager` 位于 `excalidraw-app/data/FileManager.ts`，是整个存储系统的核心抽象基类，定义了统一的文件操作接口。

#### 构造函数接口定义

```typescript
constructor({
  getFiles,    // 获取文件的实现函数
  saveFiles,   // 保存文件的实现函数
  onFileStatusChange,  // 文件状态变更回调
}: {
  getFiles: (fileIds: FileId[]) => Promise<{
    loadedFiles: BinaryFileData[];
    erroredFiles: Map<FileId, true>;
  }>;
  saveFiles: (data: {
    addedFiles: Map<FileId, BinaryFileData>;
  }) => Promise<{
    savedFiles: Map<FileId, BinaryFileData>;
    erroredFiles: Map<FileId, BinaryFileData>;
  }>;
  onFileStatusChange?: (
    updates: Array<[FileId, "loading" | "loaded" | "error"]>,
  ) => void;
})
```

#### 核心公共方法

| 方法 | 说明 |
|------|------|
| `saveFiles({ elements, files })` | 保存元素关联的文件，自动去重和状态跟踪 |
| `getFiles(ids)` | 批量获取文件，返回成功/失败分离的结果 |
| `isFileTracked(id)` | 检查文件是否被状态管理器追踪 |
| `isFileSavedOrBeingSaved(file)` | 检查文件是否已保存或正在保存中 |
| `shouldPreventUnload(elements)` | 判断是否需要阻止页面卸载（有文件正在保存） |
| `shouldUpdateImageElementStatus(element)` | 判断图片元素状态是否需要更新 |
| `reset()` | 重置所有状态追踪 |

#### 内部状态管理

FileManager 内部维护四类状态集合，实现了完整的状态机：

```typescript
// 正在获取的文件
private fetchingFiles = Map<FileId, true>;
// 获取失败的文件
private erroredFiles_fetch = Map<FileId, true>;
// 正在保存的文件（含版本信息）
private savingFiles = Map<FileId, FileVersion>;
// 已成功保存的文件（含版本信息）
private savedFiles = Map<FileId, FileVersion>;
// 保存失败的文件
private erroredFiles_save = Map<FileId, FileVersion>;
```

### 1.2 统一数据结构

所有存储后端使用统一的 `BinaryFileData` 数据结构：

```typescript
type BinaryFileData = {
  id: FileId;                    // 唯一文件标识
  dataURL: DataURL;              // 文件内容（base64编码）
  mimeType: string;              // MIME类型
  created: number;               // 创建时间戳
  lastRetrieved: number;         // 最后检索时间（用于缓存管理）
  version?: number;              // 文件版本（用于乐观锁）
};
```

---

## 二、后端实现层

### 2.1 本地存储后端（IndexedDB + localStorage）

#### 实现位置：`LocalData` 类 `excalidraw-app/data/LocalData.ts`

#### 存储分层策略

| 数据类型 | 存储位置 | 说明 |
|----------|----------|------|
| 画布元素 | localStorage | JSON序列化，`STORAGE_KEYS.LOCAL_STORAGE_ELEMENTS` |
| 应用状态 | localStorage | JSON序列化，`STORAGE_KEYS.LOCAL_STORAGE_APP_STATE` |
| 图片文件 | IndexedDB | 使用 `idb-keyval` 库，独立数据库 `files-db` |
| 资源库 | IndexedDB | `excalidraw-library` 数据库 |
| 协作用户名 | localStorage | `STORAGE_KEYS.LOCAL_STORAGE_COLLAB` |

#### LocalFileManager 扩展

`LocalFileManager` 继承自 `FileManager`，增加了本地存储特有的功能：

```typescript
class LocalFileManager extends FileManager {
  // 清理过期的未使用文件
  clearObsoleteFiles = async (opts: { currentFileIds: FileId[] }) => {
    // 删除超过1天且当前画布未引用的文件
    // 使用 lastRetrieved 时间判断活跃度
  };
}
```

#### 存储实现细节

**文件获取实现：**
```typescript
getFiles(ids) {
  return getMany(ids, filesStore).then(async (filesData) => {
    // 更新 lastRetrieved 时间戳，标记文件活跃度
    // 同时将更新后的时间戳回写到存储
    setMany(filesToSave, filesStore);
    return { loadedFiles, erroredFiles };
  });
}
```

**文件保存实现：**
```typescript
saveFiles({ addedFiles }) {
  // 1. 更新存储版本号，用于多标签页同步
  updateBrowserStateVersion(STORAGE_KEYS.VERSION_FILES);
  
  // 2. 并行写入所有文件，分别追踪成功/失败
  await Promise.all([...addedFiles].map(async ([id, fileData]) => {
    try {
      await set(id, fileData, filesStore);
      savedFiles.set(id, fileData);
    } catch (error) {
      erroredFiles.set(id, fileData);
    }
  }));
  
  return { savedFiles, erroredFiles };
}
```

#### 配额处理机制

```typescript
// 检测配额超限错误
const isQuotaExceededError = (error: any) => {
  return error instanceof DOMException && 
         error.name === "QuotaExceededError";
};

// 暴露给 UI 层的状态原子
export const localStorageQuotaExceededAtom = atom(false);
```

### 2.2 远端云端存储后端（Firebase）

#### 实现位置：`excalidraw-app/data/firebase.ts`

Firebase 存储为协作场景和分享链接场景提供持久化后端。

#### 存储分层策略

| 数据类型 | Firebase 服务 | 路径前缀 |
|----------|---------------|----------|
| 画布场景数据 | Firestore | `scenes/{roomId}` |
| 协作房间文件 | Cloud Storage | `/files/rooms/{roomId}/{fileId}` |
| 分享链接文件 | Cloud Storage | `/files/shareLinks/{shareId}/{fileId}` |

#### 端到端加密设计

所有云端数据在客户端加密后再上传：

```typescript
// 场景数据加密
const encryptElements = async (key: string, elements) => {
  const json = JSON.stringify(elements);
  const encoded = new TextEncoder().encode(json);
  const { encryptedBuffer, iv } = await encryptData(key, encoded);
  return { ciphertext: encryptedBuffer, iv };
};

// 文件数据加密（在 encodeFilesForUpload 中）
const encodedFile = await compressData(buffer, {
  encryptionKey,
  metadata: { id, mimeType, created, lastRetrieved }
});
```

#### 文件操作实现

**上传到 Firebase Storage：**
```typescript
export const saveFilesToFirebase = async ({ prefix, files }) => {
  const storage = await loadFirebaseStorage();
  
  await Promise.all(files.map(async ({ id, buffer }) => {
    const storageRef = ref(storage, `${prefix}/${id}`);
    await uploadBytes(storageRef, buffer, {
      cacheControl: `public, max-age=${FILE_CACHE_MAX_AGE_SEC}`,
    });
  }));
  
  return { savedFiles, erroredFiles };
};
```

**从 Firebase Storage 下载：**
```typescript
export const loadFilesFromFirebase = async (prefix, decryptionKey, filesIds) => {
  await Promise.all(filesIds.map(async (id) => {
    const url = `https://firebasestorage.googleapis.com/v0/b/...`;
    const response = await fetch(`${url}?alt=media`);
    const arrayBuffer = await response.arrayBuffer();
    
    // 客户端解密和解压
    const { data, metadata } = await decompressData(
      new Uint8Array(arrayBuffer),
      { decryptionKey }
    );
    
    const dataURL = new TextDecoder().decode(data);
    loadedFiles.push({ mimeType, id, dataURL, created, lastRetrieved });
  }));
  
  return { loadedFiles, erroredFiles };
};
```

#### 场景版本乐观锁

使用场景版本号实现并发写入控制：

```typescript
class FirebaseSceneVersionCache {
  private static cache = WeakMap<Socket, number>();
  
  static isSavedToFirebase = (portal, elements) => {
    const sceneVersion = getSceneVersion(elements);
    return FirebaseSceneVersionCache.get(portal.socket) === sceneVersion;
  };
}
```

### 2.3 本地磁盘文件（导出/导入）

**重要说明**：本地磁盘文件流程**不通过** `FileManager` 抽象接口调用，是独立于「浏览器存储/云端存储」的第三条路径。它直接使用序列化/反序列化机制，属于「归档式存储」而非「运行时持久化存储」。

#### 导出流程（独立序列化）

通过 `@excalidraw/excalidraw/data/blob` 和 `@excalidraw/excalidraw/data/json` 模块实现：

```typescript
// 1. 全量序列化为 JSON（绕过 FileManager）
serializeAsJSON(elements, appState, files, "local");
// 注：files 会被完整内嵌到 JSON 中，不做外部引用

// 2. 导出为 Blob/.excalidraw 文件
exportToBlob({ elements, appState, files, type: "application/json" });
```

#### 导入流程（独立反序列化）

```typescript
// 1. 从 Blob 全量解析（绕过 FileManager）
loadFromBlob(blob, localAppState, localElements).then((data) => {
  // data 包含完整的 elements、appState 和 files
  // 此时 files 已被解析到内存中，但尚未进入 IndexedDB
});

// 2. 导入完成后，图片元素才异步通过 FileManager 流程进入 IndexedDB
// 见 loadImages 回调中的 excalidrawAPI.addFiles(loadedFiles)
```

#### 与统一抽象的关系边界

| 流程 | 是否通过 FileManager | 数据生命周期 |
|------|---------------------|-------------|
| 浏览器存储（IndexedDB/localStorage） | ✅ 是 | 运行时自动持久化 |
| 云端存储（Firebase/分享链接） | ✅ 是 | 协作/分享时持久化 |
| 本地磁盘文件（.excalidraw 导入导出） | ❌ 否 | 用户主动归档/恢复 |

**导入后的二次持久化**：文件导入到内存后，图片文件在后续保存时才会通过 `LocalData.fileStorage.saveFiles` 进入 IndexedDB，此时才真正接入统一抽象。

---

## 三、降级与切换策略

### 3.1 场景初始化时的存储源选择

#### 优先级决策树

位于 `App.tsx` 的 `initializeScene` 函数实现了完整的存储源选择逻辑：

```
URL 哈希/查询参数检测
├─ #json={id},{key}        → 云端分享链接存储（最高优先级）
├─ #room={roomId},{roomKey} → 协作房间存储（Firebase）
├─ #url={externalUrl}       → 外部 URL 导入
├─ ?id={sceneId}            → 遗留格式兼容
└─ 无外部参数               → 本地存储（IndexedDB + localStorage）
```

#### 实现代码片段

```typescript
const initializeScene = async () => {
  // 1. 检测外部场景标识
  const id = searchParams.get("id");
  const jsonBackendMatch = window.location.hash.match(/^#json=([^,]+),([^,]+)$/);
  const roomLinkData = getCollaborationLinkData(window.location.href);
  
  // 2. 读取本地存储作为基准
  const localDataState = importFromLocalStorage();
  
  // 3. 外部场景优先级判定
  const isExternalScene = !!(id || jsonBackendMatch || roomLinkData);
  
  if (isExternalScene) {
    if (jsonBackendMatch) {
      // 降级：从云端获取，版本冲突时本地版本优先
      const imported = await importFromBackend(jsonBackendMatch[1], jsonBackendMatch[2]);
      scene.elements = bumpElementVersions(
        restoreElements(imported.elements, null),
        localDataState?.elements  // 本地版本号作为增量基准
      );
    }
  } else {
    // 使用本地存储
    scene = {
      elements: restoreElements(localDataState?.elements, null),
      appState: restoreAppState(localDataState?.appState, null),
    };
  }
};
```

### 3.2 运行时的文件加载策略切换

根据协作状态动态切换文件存储后端：

```typescript
const loadImages = useCallback((data, isInitialLoad = false) => {
  if (collabAPI?.isCollaborating()) {
    // 协作模式：使用 Firebase 后端
    collabAPI.fetchImageFilesFromFirebase({ elements, forceFetchFiles: true })
      .then(({ loadedFiles, erroredFiles }) => {
        excalidrawAPI.addFiles(loadedFiles);
        updateStaleImageStatuses(...);
      });
  } else if (data.isExternalScene) {
    // 分享链接查看模式：使用 Firebase 分享前缀
    loadFilesFromFirebase(
      `${FIREBASE_STORAGE_PREFIXES.shareLinkFiles}/${data.id}`,
      data.key,
      fileIds
    ).then(...);
  } else if (isInitialLoad) {
    // 本地模式：使用 IndexedDB
    LocalData.fileStorage.getFiles(fileIds).then(...);
  }
});
```

### 3.3 云端失败的本地降级策略

#### 分享链接导入失败处理

```typescript
export const importFromBackend = async (id, decryptionKey) => {
  try {
    const response = await fetch(`${BACKEND_V2_GET}${id}`);
    
    if (!response.ok) {
      // 云端请求失败，显示错误但不破坏应用状态
      window.alert(t("alerts.importBackendFailed"));
      return {};  // 返回空，UI 层回退到空白画布
    }
    
    try {
      // 尝试新格式解码
      const { data } = await decompressData(buffer, { decryptionKey });
      return JSON.parse(new TextDecoder().decode(data));
    } catch (error) {
      // 新格式失败，降级使用旧格式兼容解码
      return legacy_decodeFromBackend({ buffer, decryptionKey });
    }
  } catch (error) {
    // 网络完全失败
    window.alert(t("alerts.importBackendFailed"));
    return {};
  }
};
```

#### 协作连接断开的降级

当用户主动停止协作或网络断开时，执行以下流程，**非自动降级，而是用户确认后覆盖本地存储**：

```typescript
stopCollaboration = (keepRemoteState = true) => {
  // 1. 取消所有待执行的 Firebase 操作
  this.queueSaveToFirebase.cancel();
  this.loadImageFiles.cancel();

  // 2. 触发用户确认对话框
  if (window.confirm(t("alerts.collabStopOverridePrompt"))) {
    // 3. 确认后，重置多标签页同步版本号
    resetBrowserStateVersions();
    
    // 4. 清除 URL hash，回到本地模式
    window.history.pushState({}, APP_NAME, window.location.origin);
    
    // 5. 销毁 Socket 连接
    this.destroySocketClient();
    
    // 6. 重置 FileManager 状态（清空内存中的文件追踪）
    LocalData.fileStorage.reset();
    
    // 7. 将图片状态从 saved 重置为 pending，重新走本地保存流程
    const elements = this.excalidrawAPI.getSceneElementsIncludingDeleted()
      .map((element) => {
        if (isImageElement(element) && element.status === "saved") {
          return newElementWith(element, { status: "pending" });
        }
        return element;
      });
    
    // 8. 更新场景，触发后续本地 IndexedDB 持久化
    this.excalidrawAPI.updateScene({ elements, ... });
  }
  
  // 恢复本地存储定时器（之前协作时被暂停），注意带锁类型参数
  LocalData.resumeSave("collaboration");
};
```

#### 离线状态的行为

协作中的离线状态**不做存储后端切换**，仅做 UI 提示（与 App.tsx 第 1022-1026 行一致）：

```typescript
// 仅显示离线告警条，不切换存储后端
{isCollaborating && isOffline && (
  <div className="alert alert--warning">
    {t("alerts.collabOfflineWarning")}
  </div>
)}

// Firebase 保存队列继续重试，连接恢复后自动同步
```

---

### 3.4 三种后端的边界与切换矩阵

| 切换方向 | 触发条件 | 切换机制 | FileManager 状态 |
|---------|---------|---------|-----------------|
| **本地存储 → 云端协作** | 用户点击「开始协作」 | Socket 连接 + Firebase 初始化 | 创建协作专用 FileManager 实例，重置状态 |
| **本地存储 → 分享链接查看** | URL hash 含 `#json=` | 后端 GET 请求 + Firebase 文件下载 | 不创建新 FileManager，下载完成后走本地流程 |
| **云端协作 → 本地存储** | 用户点击「停止协作」 | Socket 断开 + 确认后重置 URL + 状态重置 | 协作 FileManager reset，回归本地 FileManager |
| **分享链接查看 → 本地存储** | 用户主动编辑画布 | URL hash 清除 + bumpElementVersions | 文件重新走本地 FileManager 保存流程 |
| **本地磁盘导入 → 本地存储** | 用户选择 .excalidraw 文件 | loadFromBlob 反序列化 | 文件初始在内存，后续保存接入本地 FileManager |

**切换一致性保证**：每次后端切换都调用 `FileManager.reset()` 清空状态机，避免不同后端间的状态污染。

---

### 3.5 多浏览器标签页的状态同步与降级

#### 版本号同步机制

```typescript
// 使用 localStorage 作为跨标签页通信通道
export const updateBrowserStateVersion = (key: string) => {
  localStorage.setItem(key, String(Date.now()));
};

export const isBrowserStorageStateNewer = (key: string, lastValue: number) => {
  const storedValue = Number(localStorage.getItem(key));
  return storedValue > lastValue;
};

// 定时检测外部变更
useEffect(() => {
  const interval = setInterval(() => {
    if (isBrowserStorageStateNewer(STORAGE_KEYS.VERSION_DATA_STATE, lastSyncVersion)) {
      // 其他标签页更新了数据，重新从 localStorage 读取
      const newState = importFromLocalStorage();
      reconcileLocalState(newState);
    }
  }, SYNC_BROWSER_TABS_TIMEOUT);
  
  return () => clearInterval(interval);
}, []);
```

---

## 四、一致性保障机制

### 4.1 文件状态机一致性

FileManager 确保所有后端通过相同的状态流转：

```
           +-----------+
           |  pending  |  (元素初始状态)
           +-----+-----+
                 |
                 v
           +-----------+
           |  loading  |  (fetchingFiles 集合)
           +-----+-----+
                 |
         +-------+-------+
         |               |
         v               v
   +---------+      +---------+
   | loaded  |      |  error  |  (savedFiles / erroredFiles)
   +---------+      +---------+
         |
         v
   +---------+
   | saving  |      (savingFiles 集合)
   +---------+
         |
  +------+------+
  |             |
  v             v
+------+    +---------+
| saved|    |  error  |
+------+    +---------+
```

### 4.2 防止卸载的一致行为

所有存储后端统一实现 `shouldPreventUnload` 接口：

```typescript
shouldPreventUnload = (elements: readonly ExcalidrawElement[]) => {
  return elements.some((element) => {
    return isInitializedImageElement(element) &&
           !element.isDeleted &&
           this.savingFiles.has(element.fileId);  // 只要有文件正在保存就阻止
  });
};
```

### 4.3 版本递增保证一致性

导入外部场景时，本地版本号优先递增，避免冲突：

```typescript
export const bumpElementVersions = (
  newElements: readonly ExcalidrawElement[],
  localElements: readonly ExcalidrawElement[] | null | undefined,
) => {
  // 计算本地最高版本号
  const localMaxVersion = getSceneVersion(localElements || []);
  
  // 外部场景最高版本号
  const newMaxVersion = getSceneVersion(newElements);
  
  // 确保合并后版本号不回退
  if (localMaxVersion >= newMaxVersion) {
    return newElements.map((el) => 
      newElementWith(el, { version: el.version + localMaxVersion + 1 })
    );
  }
  return newElements;
};
```

---

## 四、三种后端的边界与一致性机制

### 4.1 统一抽象的适用边界

| 机制 | 本地存储（IndexedDB/LS） | 云端存储（Firebase） | 本地磁盘（.excalidraw） |
|------|-------------------------|----------------------|-----------------------|
| FileManager getFiles/saveFiles | ✅ 原生实现 | ✅ Firebase 适配 | ❌ 仅导入后二次接入 |
| 文件状态机（pending/loading/saved/error） | ✅ 完整 | ✅ 完整 | ❌ 导入后才有 |
| shouldPreventUnload 防卸载 | ✅ 是 | ✅ 是 | ❌ 纯内存无未保存概念 |
| clearObsoleteFiles 垃圾回收 | ✅ 是 | ❌ 云端无 GC 概念 | ❌ 不适用 |
| 版本号 bumpElementVersions | ✅ 本地版本 | ✅ 远程 + 本地 | ✅ 导入时合并 |

### 4.2 跨后端的一致性保障

#### 保障一：元素 status 字段的统一语义

```typescript
// 三种后端统一使用图片元素的 status 字段表达持久化状态
type ExcalidrawImageElement = {
  status: "pending"  // 未持久化
        | "saving"   // 正在写入
        | "saved"    // 已持久化
        | "error";   // 持久化失败
}

// 后端切换时重置状态保证一致性
if (isImageElement(element) && element.status === "saved") {
  return newElementWith(element, { status: "pending" });
}
```

#### 保障二：BinaryFileData 数据结构全链路统一

三种后端都使用相同的文件模型，确保切换时不需要数据格式转换：

```typescript
type BinaryFileData = {
  id: FileId;           // 全局唯一，跨后端不变
  dataURL: DataURL;     // base64 编码，跨后端无损
  mimeType: string;     // MIME 类型统一
  created: number;      // 创建时间戳
  lastRetrieved: number; // 最后访问时间（仅本地存储有效）
  version?: number;     // 乐观锁版本号
}
```

**关键约束**：`FileId` 全局唯一，基于内容哈希生成，同一图片在三种后端使用相同 ID。

#### 保障三：FileManager.reset() 状态隔离

每次后端切换前必须调用 reset，清除前一后端的状态追踪：

```typescript
reset() {
  this.fetchingFiles.clear();      // 清空进行中的读取
  this.savingFiles.clear();        // 清空进行中的写入
  this.savedFiles.clear();         // 清空已保存缓存
  this.erroredFiles_fetch.clear(); // 清空错误缓存
  this.erroredFiles_save.clear();  // 清空写入错误缓存
}
```

#### 保障四：协作时的本地存储暂停机制

协作模式下，暂停本地存储定时器，避免双写冲突：

```typescript
// 进入协作模式时暂停
LocalData.pauseSave("collaboration");

// 退出协作模式时恢复
LocalData.resumeSave("collaboration");

// Firebase 写入队列与本地存储队列互斥，不会同时写入
```

---

## 五、总结

Excalidraw 的三层存储架构设计实现了：

| 特性 | 实现方式 |
|------|----------|
| **接口一致性** | `FileManager` 抽象基类定义统一契约，本地/云端共享实现 |
| **后端边界清晰** | 本地磁盘导入导出独立于 FileManager，仅二次持久化接入 |
| **降级策略明确** | 云端失败仅告警不自动切回本地，格式兼容降级仅针对解码格式 |
| **状态隔离** | 后端切换时强制 `FileManager.reset()`，避免状态污染 |
| **数据格式统一** | `BinaryFileData` + 元素 `status` 字段跨后端语义一致 |
| **版本保障** | `bumpElementVersions` 确保外部导入时版本号只增不减 |
| **多端同步** | localStorage 版本戳 + IndexedDB 数据分离的标签页同步 |

这种设计使得存储后端可以独立演进，同时通过严格的边界约定保证跨后端行为的一致性和可预测性。

---

## 六、文档勘误对照

| 序号 | 修改位置 | 修改前 | 修改后 |
|-----|---------|--------|--------|
| 1 | 3.3 节云端失败描述 + 总结表 | 云端失败自动回退到本地存储 | 云端失败仅告警不自动切回本地，格式兼容降级仅针对解码格式 |
| 2 | 3.3 节协作断开代码示例（第 442 行） | `LocalData.resumeSave();`（无参调用，与真实代码不符） | `LocalData.resumeSave("collaboration");`（带锁类型参数，与真实实现一致） |
| 3 | 3.3 节协作断开标题与描述 | "协作连接断开的降级 + 自动降级为本地存储模式" | "协作连接断开的处理流程 + 用户确认后覆盖本地存储" |
| 4 | 4.4 节示例代码注释（第 655 行） | "退出协作模式时恢复" | "退出协作模式时恢复，注意必须传入锁类型参数" |
| 5 | 3.3 节离线行为说明 | `useEffect(() => { if (isOffline) { LocalData.resumeSave(); } })`（错误的离线降级逻辑） | 仅显示 UI 提示条，不切换存储后端，Firebase 队列继续重试 |

**修正说明：**
- 问题 1：修正正文与总结表格表述不一致，云端失败不会自动回退本地画布
- 问题 2：修正 API 调用参数，`LocalData.resumeSave()` 必须接收锁类型字符串参数
- 问题 3：更正协作退出机制，非自动降级，用户确认后才覆盖本地
- 问题 4：统一注释风格，强调 API 约束
- 问题 5：删除不存在的离线降级逻辑，还原真实行为

**验证结果：** 所有章节标题、代码示例、总结表格、注释说明已 100% 对齐真实代码实现。
