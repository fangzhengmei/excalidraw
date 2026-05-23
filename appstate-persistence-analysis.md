# AppState 持久化与本地偏好链路分析

## 一、整体架构概览

Excalidraw 的状态持久化采用**分层存储 + 三级合并**的策略：

```
┌─────────────────────────────────────────────────────────────┐
│                     初始化优先级链                           │
├─────────────────────────────────────────────────────────────┤
│  1. getDefaultAppState()   →  代码内定默认值                 │
│  2. localStorage           →  浏览器本地偏好（高优先级）      │
│  3. importedData           →  文件/链接导入数据（最高优先级）  │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     存储类型过滤                             │
├─────────────────────────────────────────────────────────────┤
│  browser  →  localStorage / IndexedDB（session 间持久化）   │
│  export   →  .excalidraw 文件导出                            │
│  server   →  协作 / 分享链接服务器                           │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、核心文件清单

| 文件路径 | 职责 |
|---------|------|
| `packages/excalidraw/appState.ts` | AppState 默认值、存储过滤配置 |
| `packages/excalidraw/data/restore.ts` | 状态恢复、三级合并逻辑 |
| `excalidraw-app/data/localStorage.ts` | 从 localStorage 读取数据 |
| `excalidraw-app/data/LocalData.ts` | 持久化写入、防抖、IndexedDB |
| `excalidraw-app/data/tabSync.ts` | 跨标签页版本同步 |
| `excalidraw-app/useHandleAppTheme.ts` | 主题偏好单独处理 |
| `packages/excalidraw/data/EditorLocalStorage.ts` | 编辑器级 localStorage 工具类 |
| `packages/common/src/constants.ts` | `EDITOR_LS_KEYS`、存储 key 常量 |
| `excalidraw-app/app_constants.ts` | `STORAGE_KEYS`、超时常量 |

---

## 三、AppState 默认值定义

### 3.1 getDefaultAppState()

**位置**：`packages/excalidraw/appState.ts:22-132`

```typescript
export const getDefaultAppState = (): Omit<
  AppState,
  "offsetTop" | "offsetLeft" | "width" | "height"
> => {
  return {
    showWelcomeScreen: false,
    theme: THEME.LIGHT,
    currentItemBackgroundColor: DEFAULT_ELEMENT_PROPS.backgroundColor,
    currentItemStrokeColor: DEFAULT_ELEMENT_PROPS.strokeColor,
    activeTool: { type: "selection", /* ... */ },
    penMode: false,
    gridModeEnabled: false,
    zenModeEnabled: false,
    viewBackgroundColor: COLOR_PALETTE.white,
    zoom: { value: 1 as NormalizedZoomValue },
    // ... 约 70+ 个字段
  };
};
```

**关键点**：
- 排除了 `offsetTop/offsetLeft/width/height` 四个视口相关字段
- 某些默认值是动态的（如 `exportScale` 基于 `devicePixelRatio`）
- 测试环境下 `currentItemRoundness` 为 `"sharp"`，否则为 `"round"`

---

## 四、存储过滤机制

### 4.1 APP_STATE_STORAGE_CONF 配置

**位置**：`packages/excalidraw/appState.ts:138-257`

每个 AppState 字段都有三个存储标志位：

```typescript
const APP_STATE_STORAGE_CONF = {
  theme: { browser: true, export: false, server: false },
  currentItemStrokeColor: { browser: true, export: false, server: false },
  gridSize: { browser: true, export: true, server: true },
  viewBackgroundColor: { browser: true, export: true, server: true },
  collaborators: { browser: false, export: false, server: false },
  isLoading: { browser: false, export: false, server: false },
  // ... 所有字段
};
```

### 4.2 过滤函数

**位置**：`packages/excalidraw/appState.ts:259-293`

```typescript
// 用于 localStorage 持久化
export const clearAppStateForLocalStorage = (appState: Partial<AppState>) => {
  return _clearAppStateForStorage(appState, "browser");
};

// 用于文件导出
export const cleanAppStateForExport = (appState: Partial<AppState>) => {
  return _clearAppStateForStorage(appState, "export");
};

// 用于服务器存储
export const clearAppStateForDatabase = (appState: Partial<AppState>) => {
  return _clearAppStateForStorage(appState, "server");
};
```

### 4.3 browser 存储字段清单（可持久化到 localStorage）

标记为 `browser: true` 的字段包括：

| 类别 | 字段 |
|------|------|
| **UI 偏好** | `theme`, `zenModeEnabled`, `gridModeEnabled`, `showWelcomeScreen`, `openSidebar`, `openMenu`, `stats`, `defaultSidebarDockedPreference` |
| **工具状态** | `activeTool`, `preferredSelectionTool`, `penMode`, `penDetected`, `currentItem*`（颜色、线宽、字体等约 15 个） |
| **画布状态** | `scrollX`, `scrollY`, `zoom`, `viewBackgroundColor`, `gridSize`, `gridStep` |
| **功能开关** | `isBindingEnabled`, `bindingPreference`, `isMidpointSnappingEnabled`, `objectsSnapModeEnabled`, `bindMode`, `boxSelectionMode` |
| **导出偏好** | `exportBackground`, `exportScale`, `exportEmbedScene`, `exportWithDarkMode` |
| **选择状态** | `selectedElementIds`, `selectedGroupIds`, `previousSelectedElementIds`, `editingGroupId`, `selectedLinearElement`, `lockedMultiSelections` |
| **其他** | `cursorButton`, `lastPointerDownWith`, `name`, `scrolledOutside`, `shouldCacheIgnoreZoom` |

**不持久化**的字段（`browser: false`）：
- 临时状态：`collaborators`, `isLoading`, `isResizing`, `isRotating`
- 交互状态：`newElement`, `resizingElement`, `multiElement`, `selectionElement`, `editingTextElement`, `contextMenu`, `openPopup`, `openDialog`
- 瞬态数据：`toast`, `errorMessage`, `suggestedBinding`, `snapLines`, `originSnapOffset`
- 视口尺寸：`width`, `height`, `offsetTop`, `offsetLeft`
- 协作相关：`userToFollow`, `followedBy`, `activeEmbeddable`, `editingFrame`, `frameToHighlight`, `elementsToHighlight`, `activeLockedId`, `isCropping`, `croppingElementId`, `searchMatches`

---

## 五、持久化写入链路

### 5.1 保存触发时机

**位置**：`excalidraw-app/App.tsx:678-717`

```typescript
const onChange = (
  elements: readonly OrderedExcalidrawElement[],
  appState: AppState,
  files: BinaryFiles,
) => {
  if (collabAPI?.isCollaborating()) {
    collabAPI.syncElements(elements);
  }

  if (!LocalData.isSavePaused()) {
    LocalData.save(elements, appState, files, () => { /* ... */ });
  }
};
```

**保存暂停条件**（`LocalData.isSavePaused()`）：
- 页面隐藏（`document.hidden`）
- 协作模式锁定（`Locker<SavingLockTypes>`）

### 5.2 防抖保存

**位置**：`excalidraw-app/data/LocalData.ts:117-147`

```typescript
private static _save = debounce(
  async (elements, appState, files, onFilesSaved) => {
    saveDataStateToLocalStorage(elements, appState);
    await this.fileStorage.saveFiles({ elements, files });
    onFilesSaved();
  },
  SAVE_TO_LOCAL_STORAGE_TIMEOUT,  // 300ms
);

static save = (elements, appState, files, onFilesSaved) => {
  if (!this.isSavePaused()) {
    this._save(elements, appState, files, onFilesSaved);
  }
};
```

### 5.3 localStorage 写入逻辑

**位置**：`excalidraw-app/data/LocalData.ts:73-109`

```typescript
const saveDataStateToLocalStorage = (
  elements: readonly ExcalidrawElement[],
  appState: AppState,
) => {
  try {
    // 1. 过滤：只保留 browser: true 的字段
    const _appState = clearAppStateForLocalStorage(appState);

    // 2. 特殊处理：不保存 canvas search tab 的 sidebar 状态
    if (
      _appState.openSidebar?.name === DEFAULT_SIDEBAR.name &&
      _appState.openSidebar.tab === CANVAS_SEARCH_TAB
    ) {
      _appState.openSidebar = null;
    }

    // 3. 写入 localStorage
    localStorage.setItem(
      STORAGE_KEYS.LOCAL_STORAGE_ELEMENTS,    // "excalidraw"
      JSON.stringify(getNonDeletedElements(elements)),
    );
    localStorage.setItem(
      STORAGE_KEYS.LOCAL_STORAGE_APP_STATE,   // "excalidraw-state"
      JSON.stringify(_appState),
    );

    // 4. 更新版本号用于跨标签页同步
    updateBrowserStateVersion(STORAGE_KEYS.VERSION_DATA_STATE);
  } catch (error) {
    // QuotaExceededError 处理
    if (isQuotaExceededError(error)) {
      appJotaiStore.set(localStorageQuotaExceededAtom, true);
    }
  }
};
```

### 5.4 强制刷新保存

**位置**：`excalidraw-app/App.tsx:619-621, 654-656`

```typescript
// 页面卸载时强制刷新
const onUnload = () => {
  LocalData.flushSave();
};

// 页面失焦时强制刷新
const visibilityChange = (event) => {
  if (event.type === EVENT.BLUR || document.hidden) {
    LocalData.flushSave();
  }
};
```

---

## 六、初始化读取链路

### 6.1 启动流程

**位置**：`excalidraw-app/App.tsx:216-372`

```typescript
const initializeScene = async (opts) => {
  // 1. 从 localStorage 读取
  const localDataState = importFromLocalStorage();

  // 2. 基础恢复（只传 localStorage 数据）
  let scene = {
    elements: restoreElements(localDataState?.elements, null, {
      repairBindings: true,
      deleteInvisibleElements: true,
    }),
    appState: restoreAppState(localDataState?.appState, null),
  };

  // 3. 如果是外部链接导入（?id= 或 #json=）
  if (isExternalScene) {
    // 从后端加载数据后，二次恢复
    scene = {
      elements: bumpElementVersions(
        restoreElements(imported.elements, null, { /* ... */ }),
        localDataState?.elements,
      ),
      // 关键：localDataState.appState 作为第二参数传入
      appState: restoreAppState(
        imported.appState,
        localDataState?.appState,  // ← 本地偏好用于覆盖默认值
      ),
    };
  }

  // 4. 协作场景的三次恢复
  if (roomLinkData && opts.collabAPI) {
    scene = {
      appState: {
        ...restoreAppState(
          {
            ...scene?.appState,
            // 主题优先使用本地存储的
            theme: localDataState?.appState?.theme || scene?.appState?.theme,
          },
          excalidrawAPI.getAppState(),  // ← 已有状态
        ),
        isLoading: false,
      },
      elements: reconcileElements(/* ... */),
    };
  }
};
```

### 6.2 importFromLocalStorage()

**位置**：`excalidraw-app/data/localStorage.ts:37-74`

```typescript
export const importFromLocalStorage = () => {
  let savedElements = null;
  let savedState = null;

  try {
    savedElements = localStorage.getItem(STORAGE_KEYS.LOCAL_STORAGE_ELEMENTS);
    savedState = localStorage.getItem(STORAGE_KEYS.LOCAL_STORAGE_APP_STATE);
  } catch (error) { /* ... */ }

  let elements: ExcalidrawElement[] = [];
  if (savedElements) {
    try { elements = JSON.parse(savedElements); } catch (e) { /* ... */ }
  }

  let appState = null;
  if (savedState) {
    try {
      appState = {
        ...getDefaultAppState(),                              // 第一层：默认值
        ...clearAppStateForLocalStorage(                       // 第二层：localStorage
          JSON.parse(savedState) as Partial<AppState>,
        ),
      };
    } catch (e) { /* ... */ }
  }

  return { elements, appState };
};
```

---

## 七、restoreAppState() - 三级合并核心

**位置**：`packages/excalidraw/data/restore.ts:1013-1098`

### 7.1 核心合并逻辑

```typescript
export const restoreAppState = (
  appState: ImportedDataState["appState"],     // 第一优先级：导入数据
  localAppState: Partial<AppState> | null,    // 第二优先级：localStorage
): RestoredAppState => {
  appState = appState || {};
  const defaultAppState = getDefaultAppState();  // 第三优先级：默认值
  const nextAppState = {} as typeof defaultAppState;

  // 1. 迁移遗留字段
  for (const legacyKey of Object.keys(LegacyAppStateMigrations)) {
    if (legacyKey in appState) {
      const [nextKey, nextValue] = LegacyAppStateMigrations[legacyKey](
        appState, defaultAppState,
      );
      (nextAppState as any)[nextKey] = nextValue;
    }
  }

  // 2. 核心：三级合并
  for (const [key, defaultValue] of Object.entries(defaultAppState)) {
    const suppliedValue = appState[key];     // 导入数据
    const localValue = localAppState ? localAppState[key] : undefined;

    (nextAppState as any)[key] =
      suppliedValue !== undefined
        ? suppliedValue           // 最高优先级：导入数据
        : localValue !== undefined
        ? localValue              // 次优先级：本地偏好
        : defaultValue;           // 最低优先级：代码默认值
  }

  // 3. 特殊字段处理
  return {
    ...nextAppState,
    // 强制从 localAppState 读取，不允许导入覆盖
    cursorButton: localAppState?.cursorButton || "up",
    // 根据 penMode 重置 penDetected
    penDetected: localAppState?.penDetected ??
      (appState.penMode ? appState.penDetected ?? false : false),
    // 验证并重置 activeTool
    activeTool: {
      ...updateActiveTool(
        defaultAppState,
        nextAppState.activeTool.type &&
          AllowedExcalidrawActiveTools[nextAppState.activeTool.type]
          ? nextAppState.activeTool
          : { type: "selection" },
      ),
      lastActiveTool: null,
      locked: nextAppState.activeTool.locked ?? false,
    },
    // 兼容旧版本：zoom 从 number 转为 { value: number }
    zoom: {
      value: getNormalizedZoom(
        isFiniteNumber(appState.zoom)
          ? appState.zoom
          : appState.zoom?.value ?? defaultAppState.zoom.value,
      ),
    },
    // 兼容旧版本：openSidebar 从 string 转为 { name: string }
    openSidebar:
      typeof (appState.openSidebar as any) === "string"
        ? { name: DEFAULT_SIDEBAR.name }
        : nextAppState.openSidebar,
    // 规范化 grid 尺寸
    gridSize: getNormalizedGridSize(/* ... */),
    gridStep: getNormalizedGridStep(/* ... */),
    editingFrame: null,  // 始终重置
  };
};
```

### 7.2 优先级可视化

```
┌──────────────────────────────────────────────────────────┐
│  优先级从高到低                                          │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. imported appState (文件/链接/协作数据)               │
│     └─ 例外：cursorButton 强制从 localStorage 读取        │
│                                                          │
│  2. localAppState (localStorage 中的用户偏好)             │
│     └─ 例如：theme、activeTool、画笔颜色、字体等           │
│                                                          │
│  3. getDefaultAppState() (代码内定默认值)                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 八、本地偏好的特殊处理

### 8.1 主题偏好 - 独立存储

**位置**：`excalidraw-app/useHandleAppTheme.ts`

主题采用**独立存储 + 系统偏好检测**的策略，不经过 AppState 持久化链路：

```typescript
const useHandleAppTheme = () => {
  // 初始化：直接从独立的 localStorage key 读取
  const [appTheme, setAppTheme] = useState<Theme | "system">(() => {
    return (
      localStorage.getItem(STORAGE_KEYS.LOCAL_STORAGE_THEME) as
        | Theme | "system" | null
    ) || THEME.LIGHT;
  });

  // 变化时：直接写入独立的 localStorage key
  useLayoutEffect(() => {
    localStorage.setItem(STORAGE_KEYS.LOCAL_STORAGE_THEME, appTheme);

    if (appTheme === "system") {
      // 监听系统主题变化
      setEditorTheme(
        window.matchMedia("(prefers-color-scheme: dark)").matches
          ? THEME.DARK : THEME.LIGHT,
      );
    } else {
      setEditorTheme(appTheme);
    }
  }, [appTheme]);

  return { editorTheme, appTheme, setAppTheme };
};
```

**存储 key**：`STORAGE_KEYS.LOCAL_STORAGE_THEME = "excalidraw-theme"`

### 8.2 协作场景的主题覆盖

**位置**：`excalidraw-app/App.tsx:339-347`

```typescript
appState: {
  ...restoreAppState(
    {
      ...scene?.appState,
      // 协作时：本地主题优先于服务器传回的主题
      theme: localDataState?.appState?.theme || scene?.appState?.theme,
    },
    excalidrawAPI.getAppState(),
  ),
  isLoading: false,
},
```

---

## 九、跨标签页同步机制

### 9.1 版本号跟踪

**位置**：`excalidraw-app/data/tabSync.ts`

```typescript
// 内存中的版本号（当前标签页）
const LOCAL_STATE_VERSIONS = {
  [STORAGE_KEYS.VERSION_DATA_STATE]: -1,
  [STORAGE_KEYS.VERSION_FILES]: -1,
};

// 写入时更新版本号
export const updateBrowserStateVersion = (type) => {
  const timestamp = Date.now();
  localStorage.setItem(type, JSON.stringify(timestamp));
  LOCAL_STATE_VERSIONS[type] = timestamp;
};

// 检测是否其他标签页更新了数据
export const isBrowserStorageStateNewer = (type) => {
  const storageTimestamp = JSON.parse(localStorage.getItem(type) || "-1");
  return storageTimestamp > LOCAL_STATE_VERSIONS[type];
};
```

### 9.2 同步触发

**位置**：`excalidraw-app/App.tsx:560-617`

```typescript
const syncData = debounce(() => {
  if (!document.hidden && /* 非协作模式 */) {
    // 检测数据状态是否更新
    if (isBrowserStorageStateNewer(STORAGE_KEYS.VERSION_DATA_STATE)) {
      const localDataState = importFromLocalStorage();
      // 从 localStorage 重新加载
      excalidrawAPI.updateScene({
        ...localDataState,
        captureUpdate: CaptureUpdateAction.NEVER,
      });
      // 重新加载库数据
      LibraryIndexedDBAdapter.load().then((data) => { /* ... */ });
    }

    // 检测文件状态是否更新
    if (isBrowserStorageStateNewer(STORAGE_KEYS.VERSION_FILES)) {
      // 从 IndexedDB 加载新文件
    }
  }
}, SYNC_BROWSER_TABS_TIMEOUT);  // 50ms

// 监听 visibilitychange 和 focus 事件触发同步
```

---

## 十、其他 localStorage 使用

### 10.1 EditorLocalStorage - 编辑器级存储

**位置**：`packages/excalidraw/data/EditorLocalStorage.ts`

用于非 AppState 的编辑器偏好：

```typescript
export const EDITOR_LS_KEYS = {
  OAI_API_KEY: "excalidraw-oai-api-key",
  MERMAID_TO_EXCALIDRAW: "mermaid-to-excalidraw",
  PUBLISH_LIBRARY: "publish-library-data",
} as const;

export class EditorLocalStorage {
  static get<T extends JSONValue>(key) { /* JSON.parse */ }
  static set(key, value) { /* JSON.stringify */ }
  static has(key) { /* ... */ }
  static delete(key) { /* ... */ }
}
```

### 10.2 协作用户名

**位置**：`excalidraw-app/data/localStorage.ts:11-35`

```typescript
export const saveUsernameToLocalStorage = (username: string) => {
  localStorage.setItem(
    STORAGE_KEYS.LOCAL_STORAGE_COLLAB,  // "excalidraw-collab"
    JSON.stringify({ username }),
  );
};
```

### 10.3 调试状态

**位置**：`excalidraw-app/app_constants.ts:44`

```typescript
LOCAL_STORAGE_DEBUG: "excalidraw-debug",
```

---

## 十一、数据流时序图

### 11.1 冷启动（新用户首次访问）

```
getDefaultAppState()
       │
       ▼
  restoreAppState(null, null)
       │
       ▼
  全部使用默认值
       │
       ▼
  用户操作 → onChange → LocalData.save()
       │
       ▼
  300ms 防抖后写入 localStorage
```

### 11.2 热启动（老用户回访）

```
localStorage.getItem("excalidraw-state")
       │
       ▼
  importFromLocalStorage()
       ├─ ① JSON.parse
       ├─ ② clearAppStateForLocalStorage() 过滤
       └─ ③ { ...defaults, ...filtered }
       │
       ▼
  restoreAppState(localAppState, null)
       ├─ 三级合并：imported (空) → local → default
       └─ 特殊字段规范化
       │
       ▼
  初始化完成，用户操作继续写入 localStorage
```

### 11.3 打开分享链接

```
从 URL 解析 id/key → 从后端加载 importedData
       │
       ▼
  importFromLocalStorage() → localAppState
       │
       ▼
  restoreAppState(importedData.appState, localAppState)
       ├─ 三级合并：
       │    importedData  → 用于画布数据（元素、gridSize等）
       │    localAppState → 用于用户偏好（theme、画笔颜色等）
       │    defaults      → 兜底
       └─ 特殊：theme 强制用 local
       │
       ▼
  用户操作写入 localStorage（协作场景除外）
```

---

## 十二、容易混淆的点

### 12.1 `clearAppStateForLocalStorage` 的双重调用

- **读取时**：`importFromLocalStorage()` 中调用，确保旧数据中包含的 `browser: false` 字段被过滤
- **写入时**：`saveDataStateToLocalStorage()` 中调用，确保只持久化允许的字段

### 12.2 两种本地数据的区分

| 概念 | 存储位置 | 生命周期 | 用途 |
|------|---------|---------|------|
| `localDataState` | localStorage | 跨 session | 上次访问的完整画布状态 |
| `localAppState` | restore 函数参数 | 单次恢复过程 | 作为恢复时的偏好源 |

### 12.3 `restoreAppState` 的两个参数

```typescript
restoreAppState(
  appState,        // ← 导入数据：优先级最高，但不覆盖 cursorButton 等
  localAppState,   // ← 本地偏好：用于 imported 未定义的字段
)
```

当 `localAppState = null` 时，实际上只使用 `appState` + `defaults` 两层合并。

### 12.4 为什么主题要独立存储？

主题在 AppState 中有字段，但应用层选择独立管理，原因：
1. 主题是**应用级**偏好，不是**画布级**状态
2. 需要支持 `system` 模式（跟随系统），这在 AppState 中未建模
3. 协作时本地主题不应被其他用户覆盖
4. 需要在 Excalidraw 组件初始化前就确定主题（避免闪烁）

---

## 十三、关键代码定位速查表

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 默认值定义 | `packages/excalidraw/appState.ts` | 22-132 |
| 存储过滤配置 | `packages/excalidraw/appState.ts` | 138-257 |
| 三级合并逻辑 | `packages/excalidraw/data/restore.ts` | 1013-1098 |
| localStorage 读取 | `excalidraw-app/data/localStorage.ts` | 37-74 |
| localStorage 写入 | `excalidraw-app/data/LocalData.ts` | 73-109 |
| 防抖保存 | `excalidraw-app/data/LocalData.ts` | 117-147 |
| onChange 回调 | `excalidraw-app/App.tsx` | 678-717 |
| 启动初始化 | `excalidraw-app/App.tsx` | 216-372 |
| 跨标签页同步 | `excalidraw-app/App.tsx` | 560-617 |
| 主题独立存储 | `excalidraw-app/useHandleAppTheme.ts` | 12-70 |
| Tab 版本同步 | `excalidraw-app/data/tabSync.ts` | 1-39 |
