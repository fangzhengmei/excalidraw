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

#### 导出后端

通过 `@excalidraw/excalidraw/data/blob` 模块实现：

```typescript
// 序列化为 JSON
serializeAsJSON(elements, appState, files, "local");

// 导出为 .excalidraw 文件
exportToBlob(elements, appState, files, "application/json");
```

#### 导入后端

```typescript
// 从 Blob 加载
loadFromBlob(blob, null, null).then((data) => {
  // 解析为标准的 ExcalidrawInitialDataState 结构
});
```

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

当协作连接断开时，系统自动降级为本地存储模式，保证用户可继续编辑：

```typescript
// 位于 Collab 组件的离线状态检测
const [isOffline, setIsOffline] = useAtom(isOfflineAtom);

// 离线时停止 Firebase 同步，但保留本地 IndexedDB 写入
useEffect(() => {
  if (isOffline) {
    // 暂停云端同步
    LocalData.resumeSave();  // 确保本地存储继续工作
  }
}, [isOffline]);
```

### 3.4 多浏览器标签页的状态同步与降级

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

## 五、总结

Excalidraw 的三层存储架构设计实现了：

| 特性 | 实现方式 |
|------|----------|
| **接口一致性** | `FileManager` 抽象基类定义统一契约 |
| **后端透明性** | 上层调用无需关心具体存储实现 |
| **优雅降级** | 云端失败时自动回退到本地存储 |
| **状态一致性** | 统一的文件状态机 + 版本号机制 |
| **多端同步** | localStorage 版本戳 + IndexedDB 数据分离 |

这种设计使得存储后端可以独立演进，同时保证应用层行为的一致性和可预测性。
