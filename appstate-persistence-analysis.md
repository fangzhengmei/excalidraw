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
│  通用字段优先级从高到低（theme 字段除外，见第八节）       │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  1. imported appState (文件/链接/协作数据)               │
│     └─ 例外：cursorButton 强制从 localStorage 读取        │
│                                                          │
│  2. localAppState (localStorage 中的用户偏好)             │
│     └─ 例如：activeTool、画笔颜色、字体、gridSize 等      │
│                                                          │
│  3. getDefaultAppState() (代码内定默认值)                 │
│                                                          │
└──────────────────────────────────────────────────────────┘

⚠️  注意：theme 字段有特殊的双重存储机制，优先级完全不同。
        详见第八节"主题偏好的双重存储与冲突"。
```

---

## 八、本地偏好的特殊处理 - 主题偏好的双重存储与冲突

### 8.1 两套主题存储机制

主题是**唯一同时存储在两个 localStorage key** 中的字段，这是混乱的根源：

| 存储位置 | 存储内容 | 数据类型 | 管理方 |
|---------|---------|---------|-------|
| `localStorage["excalidraw-theme"]` | 独立 key | `"light" \| "dark" \| "system"` | `useHandleAppTheme()` hook |
| `localStorage["excalidraw-state"].theme` | AppState 字段 | `"light" \| "dark"` | AppState 持久化链路 |

#### 存储 A：独立 key（excalidraw-theme）

**位置**：`excalidraw-app/useHandleAppTheme.ts`

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

**特点**：
- 支持 `"system"` 模式（跟随系统主题）
- 在 HTML 加载阶段就被内联脚本读取，避免主题闪烁
- `editorTheme` 作为 `props.theme` 传递给 `<Excalidraw>` 组件

#### 存储 B：AppState 字段（excalidraw-state.theme）

**位置**：`packages/excalidraw/appState.ts:151`

```typescript
theme: { browser: true, export: false, server: false },
```

`theme` 标记为 `browser: true`，所以会通过 `clearAppStateForLocalStorage()` 过滤后持久化到 `excalidraw-state`。

**写入时机**：`onChange` 回调 → `LocalData.save()` → `saveDataStateToLocalStorage()`

---

### 8.2 主题优先级核心规则

`props.theme`（来自独立 key）在 **Excalidraw 组件内部**有三处会覆盖 AppState 中的 theme：

#### 规则 1：初始化时覆盖
**位置**：`packages/excalidraw/components/App.tsx:2916-2918, 2964-2966`

```typescript
// 初始化一开始就设置
if (this.props.theme) {
  this.setState({ theme: this.props.theme });
}

// restoreAppState 之后再次强制覆盖
restoredAppState = {
  ...restoredAppState,
  theme: this.props.theme || restoredAppState.theme,  // ← 关键：props.theme 优先
};
```

#### 规则 2：每次状态更新时覆盖
**位置**：`packages/excalidraw/components/App.tsx:2797-2798`

```typescript
const theme =
  actionResult?.appState?.theme || this.props.theme || THEME.LIGHT;
```

每次调用 `syncActionResult()` 时，theme 的取值顺序：
1. `actionResult.appState.theme`（来自 action，如快捷键切换）
2. `this.props.theme`（来自独立 key）
3. `THEME.LIGHT`（兜底）

#### 规则 3：props 变化时直接覆盖
**位置**：`packages/excalidraw/components/App.tsx:3519-3521`

```typescript
if (prevProps.theme !== this.props.theme && this.props.theme) {
  this.setState({ theme: this.props.theme });
}
```

---

### 8.3 各场景下的主题取值与冲突处理

#### 场景 1：冷启动（新用户）
**存储状态**：两个 key 都不存在

1. HTML 内联脚本读取 `excalidraw-theme` → 不存在，返回 `"light"`
2. `useHandleAppTheme()` 初始化 → `editorTheme = "light"`
3. `importFromLocalStorage()` → `appState` 为 `null`
4. `restoreAppState(null, null)` → `theme = THEME.LIGHT`
5. `initialize()` 中：`restoredAppState.theme = "light" || "light" = "light"`
6. **最终取值**：`"light"`
7. **冲突**：无

#### 场景 2：热启动（老用户，两个存储一致）
**存储状态**：
- `excalidraw-theme = "dark"`
- `excalidraw-state.theme = "dark"`

1. HTML 内联脚本 → `"dark"`，设置 `html.dark`
2. `useHandleAppTheme()` → `editorTheme = "dark"`
3. `importFromLocalStorage()` → `appState.theme = "dark"`
4. `restoreAppState(localDataState.appState, null)` → `theme = "dark"`
5. `initialize()` 中：`restoredAppState.theme = "dark" || "dark" = "dark"`
6. **最终取值**：`"dark"`
7. **冲突**：无，两个存储一致

#### 场景 3：热启动（两个存储冲突）
**存储状态**：
- `excalidraw-theme = "dark"`（独立 key）
- `excalidraw-state.theme = "light"`（AppState 字段）

1. HTML 内联脚本 → `"dark"`
2. `useHandleAppTheme()` → `editorTheme = "dark"`
3. `importFromLocalStorage()` → `appState.theme = "light"`
4. `restoreAppState(localDataState.appState, null)` → `theme = "light"`
5. `initialize()` 中：
   ```typescript
   restoredAppState.theme = this.props.theme || restoredAppState.theme
   // "dark" || "light" = "dark"
   ```
6. **最终取值**：**`"dark"`**
7. **冲突解决**：`props.theme`（独立 key）优先级更高，覆盖了 restoredAppState.theme

#### 场景 4：打开分享链接（链接指定 theme = "dark"，本地独立 key = "light"）
**存储状态**：
- `excalidraw-theme = "light"`（独立 key）
- 链接导入的 `imported.appState.theme = "dark"`

1. `useHandleAppTheme()` → `editorTheme = "light"`
2. `restoreAppState(imported.appState, localDataState.appState)`：
   - 三级合并：`suppliedValue = "dark"`（导入）> `localValue`（本地 AppState）> `default`
   - 结果：`restoredAppState.theme = "dark"`
3. **但**，`initialize()` 中：
   ```typescript
   restoredAppState.theme = this.props.theme || restoredAppState.theme
   // "light" || "dark" = "light"
   ```
4. **最终取值**：**`"light"`**
5. **关键点**：即使导入数据指定了 theme，`props.theme`（独立 key）仍然会覆盖它！
6. **结论**：导入的 `appState.theme` 对 excalidraw-app 来说**完全不生效**，因为 `props.theme` 总是有值。

#### 场景 5：协作场景
**位置**：`excalidraw-app/App.tsx:339-347`

```typescript
appState: {
  ...restoreAppState(
    {
      ...scene?.appState,
      theme: localDataState?.appState?.theme || scene?.appState?.theme,
    },
    excalidrawAPI.getAppState(),
  ),
  isLoading: false,
},
```

这里的逻辑：
1. 先用 `localDataState?.appState?.theme`（AppState 存储）覆盖服务器返回的 theme
2. 调用 `restoreAppState` 恢复
3. **但最终**，还是会被 `props.theme`（独立 key）覆盖！

**最终结果**：协作场景下，主题始终使用本地独立 key 的值，不随其他用户的主题变化。

#### 场景 6：用户通过菜单切换主题（推荐路径）
1. 用户点击菜单选择 "dark" → `setAppTheme("dark")`
2. `setAppTheme()` 写入 `localStorage["excalidraw-theme"] = "dark"`
3. `editorTheme` 更新为 `"dark"`，作为 `props.theme` 传入 `<Excalidraw>`
4. `componentDidUpdate` 检测到变化 → `setState({ theme: "dark" })`
5. `onChange` 回调触发 → `LocalData.save()`
6. 写入 `localStorage["excalidraw-state"].theme = "dark"`
7. **最终状态**：两个存储都更新为 `"dark"`，保持一致
8. **无冲突**：推荐的切换方式

#### 场景 7：用户通过快捷键切换主题（Alt+Shift+D）- 潜在 Bug
1. 快捷键触发 `actionToggleTheme.perform()`
2. 返回 `{ appState: { theme: "dark" } }`
3. `syncActionResult` 处理：
   ```typescript
   const theme = actionResult?.appState?.theme || this.props.theme || THEME.LIGHT
   // "dark" || "light" = "dark"  ← 这次生效了
   ```
4. `setState({ theme: "dark" })`
5. `onChange` 回调 → 写入 `localStorage["excalidraw-state"].theme = "dark"`
6. **但**：`localStorage["excalidraw-theme"]` **没有更新！**
7. **当前状态**：AppState 是 "dark"，独立 key 还是 "light"
8. **刷新后**：`props.theme` 是 "light"，会覆盖 AppState 的 "dark"，主题又变回 "light"！
9. **这是一个 Bug**：快捷键切换主题不会更新独立 key，刷新后恢复。

#### 场景 8：用户选择 "system" 主题
1. 用户选择 "system" → `setAppTheme("system")`
2. 写入 `localStorage["excalidraw-theme"] = "system"`
3. 假设系统是 dark 模式，`editorTheme = "dark"`，作为 `props.theme` 传入
4. `componentDidUpdate` → `setState({ theme: "dark" })`
5. `onChange` 回调 → 写入 `localStorage["excalidraw-state"].theme = "dark"`
6. **存储状态**：
   - `excalidraw-theme = "system"`
   - `excalidraw-state.theme = "dark"`
7. **下次刷新时**：
   - 独立 key 是 "system"，解析为实际主题（如 "dark"）
   - AppState 是 "dark"
   - 两者一致，无冲突
8. **系统主题变化时**：如果系统从 dark 变 light，刷新后：
   - `editorTheme = "light"`（重新解析 "system"）
   - `props.theme = "light"` 覆盖 AppState 的 "dark"
   - 主题变为 light

---

### 8.4 主题优先级总览

```
┌───────────────────────────────────────────────────────────────────────┐
│  主题优先级从高到低                                                   │
├───────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. props.theme（来自 excalidraw-theme 独立 key）                     │
│     ├─ 初始化时：this.props.theme || restoredAppState.theme           │
│     ├─ 每次更新时：actionResult.theme || this.props.theme || LIGHT    │
│     └─ props 变化时：直接 setState({ theme: this.props.theme })       │
│                                                                       │
│  2. actionResult.appState.theme（来自 actionToggleTheme 等）          │
│     └─ 仅在没有 props.theme 时生效（excalidraw-app 中不生效）          │
│                                                                       │
│  3. imported.appState.theme（文件/链接/协作导入）                     │
│     └─ 在 restoreAppState 中生效，但随后被 props.theme 覆盖            │
│                                                                       │
│  4. localAppState.theme（来自 excalidraw-state）                      │
│     └─ 在 restoreAppState 中作为偏好源，但随后被 props.theme 覆盖      │
│                                                                       │
│  5. getDefaultAppState().theme（代码默认值）                          │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘

┌───────────────────────────────────────────────────────────────────────┐
│  主题切换的两条路径                                                   │
├───────────────────────────────────────────────────────────────────────┤
│  路径 A（推荐，用于 excalidraw-app）：                                │
│    用户菜单 → setAppTheme() → 更新 excalidraw-theme →                 │
│    props.theme 变化 → Excalidraw setState → onChange →                │
│    更新 excalidraw-state.theme  → 两个存储一致                        │
│                                                                       │
│  路径 B（仅库内部，excalidraw-app 有副作用）：                         │
│    快捷键 → actionToggleTheme → 更新 AppState →                       │
│    onChange → 更新 excalidraw-state.theme →                           │
│    excalidraw-theme 未更新 → 刷新后恢复                               │
│                                                                       │
└───────────────────────────────────────────────────────────────────────┘
```

---

### 8.5 为什么主题要设计成双重存储？

1. **避免主题闪烁**：HTML 加载阶段就需要读取主题，此时 React 还未初始化，必须用独立 key 的内联脚本读取
2. **支持 system 模式**：AppState.theme 只支持 `"light"/"dark"`，不支持 `"system"` 模式
3. **应用级 vs 画布级**：主题是应用级偏好，不是画布级状态，不应该随画布导入导出而变化
4. **协作隔离**：协作时本地主题不应被其他用户覆盖
5. **库的灵活性**：Excalidraw 作为库，可以通过 `props.theme` 让宿主应用完全控制主题

---

### 8.6 设计缺陷与潜在问题

1. **快捷键切换主题的 Bug**：Alt+Shift+D 不会更新独立 key，刷新后恢复。修复需要在 `actionToggleTheme` 中触发外部回调，或者 excalidraw-app 监听 AppState.theme 变化同步更新独立 key。

2. **数据冗余**：同一个信息存储在两个地方，增加了不一致的风险。

3. **导入数据的 theme 无效**：对于 excalidraw-app，导入的 `.excalidraw` 文件中即使包含 `appState.theme`，也会被 `props.theme` 覆盖，用户感知不到。

4. **restoreAppState 中的 theme 处理名存实亡**：虽然 `restoreAppState` 会按照三级合并逻辑处理 theme，但对于 excalidraw-app 来说，结果总是会被 `props.theme` 覆盖。

---

### 8.7 协作场景的主题覆盖（补充）

**位置**：`excalidraw-app/App.tsx:339-347`

```typescript
appState: {
  ...restoreAppState(
    {
      ...scene?.appState,
      // 先用本地 AppState 的 theme 覆盖服务器数据
      theme: localDataState?.appState?.theme || scene?.appState?.theme,
    },
    excalidrawAPI.getAppState(),
  ),
  isLoading: false,
},
```

这层保护是为了防止协作时服务器返回的主题覆盖本地，但实际上：
1. 这里已经用本地 AppState.theme 覆盖了服务器 theme
2. 最后还会被 `props.theme`（独立 key）再覆盖一次
3. 所以即使删除这段代码，最终结果也一样（只是多了一层防御）

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
       │    localAppState → 用于用户偏好（画笔颜色、字体等）
       │    defaults      → 兜底
       └─ ⚠️  theme 字段在 restoreAppState 中按正常逻辑合并
       │
       ▼
  Excalidraw.initialize() 中强制覆盖
       │
       ▼
  restoredAppState.theme = props.theme || restoredAppState.theme
       │
       ▼
  ⚠️  importedData.appState.theme 被 props.theme（独立 key）完全覆盖！
       │
       ▼
  用户操作写入 localStorage（协作场景除外）
```

### 11.4 主题切换（菜单路径 - 推荐）

```
用户点击菜单选择主题 → setAppTheme(theme)
       │
       ▼
  写入 localStorage["excalidraw-theme"] = theme
       │
       ▼
  editorTheme 更新 → 作为 props.theme 传入
       │
       ▼
  componentDidUpdate 检测到 props.theme 变化
       │
       ▼
  setState({ theme: props.theme })
       │
       ▼
  onChange 回调触发 → LocalData.save()
       │
       ▼
  写入 localStorage["excalidraw-state"].theme = theme
       │
       ▼
  ✅  两个存储保持一致
```

### 11.5 主题切换（快捷键路径 - 有 Bug）

```
用户按 Alt+Shift+D → actionToggleTheme.perform()
       │
       ▼
  返回 { appState: { theme: newTheme } }
       │
       ▼
  syncActionResult() 处理
       │
       ▼
  const theme = actionResult.theme || props.theme || LIGHT
       │
       ▼
  setState({ theme: newTheme })  ←  这次生效了
       │
       ▼
  onChange 回调 → 写入 excalidraw-state.theme = newTheme
       │
       ▼
  ⚠️  localStorage["excalidraw-theme"] 未更新！
       │
       ▼
  刷新页面 → props.theme 读取独立 key 的旧值
       │
       ▼
  ❌  主题恢复到切换前的状态
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

### 12.4 主题的双重存储与优先级（最容易混淆的点）

这是之前分析最容易出错的地方：

| 误区 | 事实 |
|------|------|
| theme 不经过 AppState 持久化链路 | ❌ theme 标记为 `browser: true`，会被写入 `excalidraw-state` |
| theme 完全由独立 key 控制 | ✅ 但 AppState 中也有一份，两者可能不一致 |
| 导入数据的 theme 会生效 | ❌ 对于 excalidraw-app，`props.theme` 会覆盖它 |
| 协作时本地 theme 优先 | ✅ 但实际上有两层保护（AppState 层 + props 层） |
| 快捷键切换主题是可靠的 | ❌ 不会更新独立 key，刷新后恢复 |

**核心记忆点**：
- `props.theme` 是**最终裁决者**，在三处覆盖 AppState.theme
- 独立 key `excalidraw-theme` 决定 `props.theme`
- AppState 中的 theme 只是"影子"，刷新后以独立 key 为准
- 只有通过 `setAppTheme()`（菜单切换）才能同时更新两个存储

### 12.5 为什么主题要设计成双重存储？

主题在 AppState 中有字段，但应用层选择独立管理，原因：
1. **避免主题闪烁**：HTML 加载阶段就需要读取主题，此时 React 还未初始化
2. **支持 system 模式**：AppState.theme 只支持 `"light"/"dark"`，不支持 `"system"`
3. **应用级 vs 画布级**：主题是应用级偏好，不是画布级状态
4. **协作隔离**：协作时本地主题不应被其他用户覆盖
5. **库的灵活性**：Excalidraw 作为库，可以通过 `props.theme` 让宿主完全控制主题

### 12.6 主题相关的潜在 Bug

**快捷键切换主题不会更新独立 key**：
- 按 Alt+Shift+D 只会调用 `actionToggleTheme`，更新 AppState.theme
- 独立 key `excalidraw-theme` 不会更新
- 刷新后 `props.theme` 读取独立 key 的旧值，覆盖 AppState.theme
- 主题恢复到切换前的状态

**修复方案**：
1. 在 excalidraw-app 中监听 `appState.theme` 变化，同步更新独立 key
2. 或者禁用默认的 `actionToggleTheme`，完全由外部控制

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
| 主题独立存储 hook | `excalidraw-app/useHandleAppTheme.ts` | 12-70 |
| **主题优先级规则 1：初始化覆盖** | `packages/excalidraw/components/App.tsx` | 2916-2918, 2964-2966 |
| **主题优先级规则 2：状态更新覆盖** | `packages/excalidraw/components/App.tsx` | 2797-2798 |
| **主题优先级规则 3：props 变化覆盖** | `packages/excalidraw/components/App.tsx` | 3519-3521 |
| **HTML 内联脚本预加载主题** | `excalidraw-app/index.html` | 59-87 |
| **切换主题 action（快捷键用）** | `packages/excalidraw/actions/actionCanvas.tsx` | 468-494 |
| **菜单 ToggleTheme 组件** | `packages/excalidraw/components/main-menu/DefaultItems.tsx` | 231-309 |
| Tab 版本同步 | `excalidraw-app/data/tabSync.ts` | 1-39 |
