# Excalidraw 嵌入桥接机制 (Embed Bridge)

本文档基于源码真实实现，描述 Excalidraw 画板如何嵌入到第三方网页中，宿主应用可用的 API、内部事件外发机制、以及外部命令注入方式。

---

## 一、宿主可用的 API 暴露面 (API Surface)

### 1.1 声明式配置 Props (ExcalidrawProps)

通过 React Props 配置编辑器行为和订阅事件：

```typescript
interface ExcalidrawProps {
  // ========== 数据初始化 ==========
  initialData?: ExcalidrawInitialDataState | null;
  libraryItems?: LibraryItem[] | null;

  // ========== 状态回调 ==========
  onChange?: (
    elements: readonly OrderedExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles,
  ) => void;

  onIncrement?: (event: DurableIncrement | EphemeralIncrement) => void;

  onExcalidrawAPI?: (api: ExcalidrawImperativeAPI | null) => void;

  onInitialize?: (api: ExcalidrawImperativeAPI) => void;

  onMount?: (payload: {
    excalidrawAPI: ExcalidrawImperativeAPI;
    container: HTMLDivElement | null;
  }) => void;

  onUnmount?: () => void;

  // ========== 交互事件 ==========
  onPointerUpdate?: (payload: {
    pointer: { x: number; y: number; tool: "pointer" | "laser" };
    button: "down" | "up";
    pointersMap: Gesture["pointers"];
  }) => void;

  // ========== 嵌入内容 ==========
  validateEmbeddable?:
    | boolean
    | string[]
    | RegExp
    | ((link: string) => boolean | undefined);

  renderEmbeddable?: (
    element: NonDeleted<ExcalidrawEmbeddableElement>,
    appState: AppState,
  ) => JSX.Element | null;

  // ========== 粘贴拦截 ==========
  onPaste?: (
    data: ClipboardData,
    event: ClipboardEvent | null,
  ) => Promise<boolean> | boolean;

  // ========== 导出控制 ==========
  onExport?: (exportOpts: ExportOpts) => void;
  onExportProgress?: (status: OnExportProgress) => void;

  // ========== UI 控制 ==========
  viewModeEnabled?: boolean;
  zenModeEnabled?: boolean;
  gridModeEnabled?: boolean;
  theme?: Theme;
  name?: string;
  UIOptions?: UIOptions;
  langCode?: Language["code"];
  renderCustomStats?: (
    elements: readonly NonDeletedExcalidrawElement[],
    appState: AppState,
  ) => React.ReactNode;

  // ========== 协作 ==========
  isCollaborating?: boolean;
  onPointerDownOnElement?: OnPointerDownOnElementPayload;
  onUserFollow?: OnUserFollowedPayload;

  // ========== 其他 ==========
  detectScroll?: boolean;
  handleKeyboardGlobally?: boolean;
  autoFocus?: boolean;
  generateDiagramToCode?: GenerateDiagramToCode;
  getFormFactor?: (editorWidth: number, editorHeight: number) => FormFactor;
}
```

### 1.2 命令式 API (ExcalidrawImperativeAPI)

通过 `onExcalidrawAPI` 或 `onMount` 获取实例后调用：

```typescript
interface ExcalidrawImperativeAPI {
  // ========== 核心标记 ==========
  isDestroyed: boolean;
  id: string;

  // ========== 场景操作 ==========
  updateScene(opts: {
    elements?: readonly ExcalidrawElement[];
    appState?: DeepPartial<AppState>;
    captureUpdate?: CaptureUpdateActionType;
  }): void;

  applyDeltas(
    elements: readonly ExcalidrawElement[],
    deltas: readonly ElementsMapOrArray,
    {
      mergeIntoAppState,
      captureUpdate,
    }?: {
      mergeIntoAppState?: boolean;
      captureUpdate?: CaptureUpdateActionType;
    },
  ): void;

  mutateElement<T extends Mutable<ExcalidrawElement>>(
    element: T,
    updates: Partial<Omit<T, keyof ExcalidrawElement>> & Partial<ExcalidrawElement>,
    options?: {
      regenerateIds?: boolean;
      preservePrototype?: boolean;
      captureUpdate?: CaptureUpdateActionType;
    },
  ): T;

  resetScene(opts?: {
    resetLoadingState?: boolean;
    preserveSession?: boolean;
  }): void;

  // ========== 历史操作 ==========
  history: {
    clear: () => void;
  };

  // ========== 数据读取 ==========
  getSceneElements(): readonly NonDeletedExcalidrawElement[];
  getSceneElementsIncludingDeleted(): readonly ExcalidrawElement[];
  getSceneElementsMapIncludingDeleted(): Map<string, ExcalidrawElement>;
  getAppState(): AppState;
  getFiles(): BinaryFiles;
  getName(): string;

  // ========== 文件操作 ==========
  addFiles(data: BinaryFileData[]): void;

  // ========== 视图操作 ==========
  scrollToContent(opts?: {
    fitToContent?: boolean;
    animate?: boolean;
    elements?: readonly ExcalidrawElement[];
    scale?: number;
  }): void;

  refresh(): void;

  // ========== 工具操作 ==========
  setActiveTool(
    nextActiveTool:
      | AppState["activeTool"]["type"]
      | {
          type: AppState["activeTool"]["type"];
          lastActiveTool?: AppState["activeTool"]["lastActiveTool"];
          customType?: string;
          locked?: boolean;
        },
  ): void;

  // ========== UI 控制 ==========
  setToast(toast: Toast | null): void;
  toggleSidebar(
    name: SidebarName,
    options?: { tab?: SidebarTabName; force?: boolean },
  ): void;
  setCursor(cursor: string): void;
  resetCursor(): void;
  updateFrameRendering(enabled: boolean): void;

  // ========== 扩展 ==========
  registerAction(action: Action): void;
  updateLibrary(libraryItems: LibraryItems): Promise<void>;
  getEditorInterface(): EditorInterface;

  // ========== 事件订阅 ==========
  onChange(
    callback: (
      elements: readonly ExcalidrawElement[],
      appState: AppState,
      files: BinaryFiles,
    ) => void,
  ): UnsubscribeCallback;

  onIncrement(
    callback: (event: DurableIncrement | EphemeralIncrement) => void,
  ): UnsubscribeCallback;

  onPointerDown(
    callback: (
      activeTool: AppState["activeTool"],
      pointerDownState: PointerDownState,
      event: React.PointerEvent<HTMLElement>,
    ) => void,
  ): UnsubscribeCallback;

  onPointerUp(
    callback: (
      activeTool: AppState["activeTool"],
      pointerDownState: PointerDownState,
      event: PointerEvent,
    ) => void,
  ): UnsubscribeCallback;

  onScrollChange(
    callback: (scrollX: number, scrollY: number, zoom: Zoom) => void,
  ): UnsubscribeCallback;

  onUserFollow(
    callback: (payload: OnUserFollowedPayload) => void,
  ): UnsubscribeCallback;

  onStateChange: OnStateChange; // 见下方详细定义

  onEvent: <K extends keyof ExcalidrawImperativeAPIEventMap>(
    name: K,
    callback: (...args: ExcalidrawImperativeAPIEventMap[K]) => void,
  ) => UnsubscribeCallback;
}
```

### 1.3 onStateChange 详细定义 (重要)

**这是宿主最常用的状态订阅 API，有多种调用形式**：

```typescript
type OnStateChange = {
  // 形式1: 订阅单个属性变化
  <K extends keyof AppState>(
    prop: K,
    callback: (value: AppState[K], appState: AppState) => void,
    opts?: { once: boolean },
  ): UnsubscribeCallback;

  // 形式2: Promise 形式等待单个属性满足条件
  <K extends keyof AppState>(prop: K): Promise<AppState[K]>;

  // 形式3: 订阅多个属性变化
  (
    prop: (keyof AppState)[],
    callback: (currentState: AppState, appState: AppState) => void,
    opts?: { once: boolean },
  ): UnsubscribeCallback;

  // 形式4: Promise 形式等待多个属性满足条件
  (prop: (keyof AppState)[]): Promise<AppState>;

  // 形式5: 通过选择器函数订阅
  <T>(
    prop: (appState: AppState) => T,
    callback: (value: T, appState: AppState) => void,
    opts?: { once: boolean },
  ): UnsubscribeCallback;

  // 形式6: Promise 形式等待选择器满足条件
  <T>(prop: (appState: AppState) => T): Promise<T>;

  // 形式7: 通过 predicate 函数订阅
  (opts: {
    predicate: (appState: AppState) => boolean;
    callback: (appState: AppState) => void;
    once?: boolean;
  }): UnsubscribeCallback;

  // 形式8: Promise 形式等待 predicate 满足
  (opts: { predicate: (appState: AppState) => boolean }): Promise<AppState>;
};
```

**使用示例**：
```typescript
// 订阅主题变化
api.onStateChange("theme", (theme, state) => {
  console.log("Theme changed to:", theme);
});

// 等待侧边栏打开
await api.onStateChange({
  predicate: (state) => state.openSidebar !== null,
});

// 订阅选中元素变化
api.onStateChange(
  (state) => state.selectedElementIds,
  (selectedIds, state) => {
    console.log("Selected elements:", selectedIds.size);
  },
);
```

---

## 二、内部事件外发机制 (Internal Event Forwarding)

### 2.1 onChange - 全量状态变更

**触发时机**：任何影响元素、AppState 或文件的操作都会触发

```typescript
// 宿主侧使用
const unsubscribe = api.onChange((elements, appState, files) => {
  // elements: 所有未删除的画布元素
  // appState: 完整的应用状态
  // files: 所有二进制文件（图片等）
  console.log("Total elements:", elements.length);
  console.log("Current tool:", appState.activeTool.type);
});

// 取消订阅
unsubscribe();
```

### 2.2 onIncrement - 增量变更 (性能优先)

**触发时机**：与 onChange 相同，但只返回变更的增量部分

```typescript
api.onIncrement((event) => {
  if (event.type === "durable") {
    // 持久化变更（元素更新、状态变化等）
    console.log("Durable changes:", event.elements);
  } else {
    // 临时变更（鼠标位置、临时交互状态等）
    console.log("Ephemeral changes");
  }
});
```

### 2.3 onStateChange - 细粒度状态订阅

**实现位置**：`packages/excalidraw/components/AppStateObserver.ts`

**触发机制**：
1. 每次状态变化时调用 `AppStateObserver.flush(prevState)`
2. 遍历所有监听器，调用其 `predicate` 函数判断是否触发
3. 如果满足条件，调用 `callback` 并传入新值
4. 标记 `once: true` 的监听器在触发后自动移除

**自动立即触发**：
如果订阅时当前状态已满足 predicate，会通过 `queueMicrotask` 立即触发回调。

### 2.4 指针事件外发

**onPointerDown / onPointerUp**：
```typescript
api.onPointerDown((activeTool, pointerState, event) => {
  // 广播指针按下事件到协作服务器
  broadcastPointer({
    x: pointerState.origin.x,
    y: pointerState.origin.y,
    tool: activeTool.type,
  });
});
```

### 2.5 滚动变化事件

```typescript
api.onScrollChange((scrollX, scrollY, zoom) => {
  // 同步视口位置给其他协作用户
  syncViewport({ scrollX, scrollY, zoom: zoom.value });
});
```

### 2.6 生命周期事件

```typescript
// 编辑器生命周期事件
type ExcalidrawImperativeAPIEventMap = {
  "editor:mount": [payload: {
    excalidrawAPI: ExcalidrawImperativeAPI;
    container: HTMLDivElement | null;
  }];
  "editor:initialize": [api: ExcalidrawImperativeAPI];
  "editor:unmount": [];
};

api.onEvent("editor:mount", ({ container }) => {
  console.log("Excalidraw mounted on:", container);
});
```

---

## 三、外部命令注入 (External Command Injection)

### 3.1 updateScene - 核心注入 API

**最常用的注入方式**，支持原子化操作：

```typescript
// 注入新元素
api.updateScene({
  elements: [
    ...existingElements,
    {
      type: "rectangle",
      x: 100,
      y: 100,
      width: 200,
      height: 150,
      strokeColor: "#000000",
      backgroundColor: "#ffffff",
      // ... 其他必需属性
    },
  ],
  // CaptureUpdateAction.IMMEDIATELY: 加入历史栈（可撤销，默认）
  // CaptureUpdateAction.EVENTUALLY: 延迟加入历史栈
  // CaptureUpdateAction.NEVER: 不加入历史栈（远程协作更新推荐）
  captureUpdate: CaptureUpdateAction.NEVER,
});

// 只更新应用状态
api.updateScene({
  appState: {
    activeTool: { type: "selection" },
    theme: "dark",
    viewBackgroundColor: "#1a1a1a",
  },
});
```

### 3.2 applyDeltas - 增量更新

适用于协作场景的增量同步：

```typescript
// 只更新变化的属性，而不是全量替换
api.applyDeltas(
  allElements,
  [
    {
      id: "element-id",
      x: 150, // 只更新 x 坐标
      y: 150, // 只更新 y 坐标
    },
  ],
  { captureUpdate: CaptureUpdateAction.NEVER },
);
```

### 3.3 mutateElement - 单个元素更新

更新单个元素的特定属性：

```typescript
api.mutateElement(
  element,
  {
    strokeColor: "#ff0000",
    width: 300,
  },
  {
    captureUpdate: CaptureUpdateAction.IMMEDIATELY, // 允许撤销
  },
);
```

### 3.4 文件注入 (addFiles)

**注意**：输入必须是 `BinaryFileData` 格式，不是原生 File 对象

```typescript
// BinaryFileData 结构
interface BinaryFileData {
  id: FileId;
  dataURL: DataURL; // base64 编码的 data URL
  mimeType: string;
  created: number; // 时间戳
}

// 注入图片文件
api.addFiles([
  {
    id: "file-id-1",
    dataURL: "data:image/png;base64,...",
    mimeType: "image/png",
    created: Date.now(),
  },
]);

// 创建引用该文件的图片元素
api.updateScene({
  elements: [
    {
      type: "image",
      fileId: "file-id-1",
      x: 0,
      y: 0,
      width: 400,
      height: 300,
      // ... 其他必需属性
    },
  ],
});
```

### 3.5 初始数据注入

通过组件 Props 预加载场景：

```typescript
<Excalidraw
  initialData={{
    elements: savedElements,
    appState: { theme: "dark" },
    files: savedFiles,
  }}
  // 或异步加载函数
  initialData={async () => {
    const response = await fetch("/api/scene");
    return response.json();
  }}
/>
```

### 3.6 历史操作控制

```typescript
// 清空历史栈（远程协作时推荐在连接建立后调用）
api.history.clear();

// 重置整个场景
api.resetScene({
  resetLoadingState: true,
  preserveSession: false,
});
```

---

## 四、宿主能力与内部实现边界 (Boundary)

### ✅ 宿主公开可用 (Public API)

| 类别 | 定义位置 | 说明 |
|------|---------|------|
| Props 配置 | `types.ts: ExcalidrawProps` | 组件传入的所有属性 |
| Imperative API | `types.ts: ExcalidrawImperativeAPI` | `api.xxx` 调用的所有方法 |
| 事件回调 | `onChange / onIncrement / onStateChange` 等 | 文档中列出的所有订阅方法 |
| 嵌入验证 | `validateEmbeddable` / `renderEmbeddable` | 自定义嵌入内容的验证和渲染 |
| 粘贴拦截 | `onPaste` | 返回 `true` 阻止默认粘贴行为 |
| 自定义 Action | `registerAction` | 扩展编辑器操作 |

### ⚠️ 内部框架能力 (Internal Only)

**以下是框架内部能力，宿主不应直接使用**：

| 类别 | 定义位置 | 说明 |
|------|---------|------|
| App 类实例 | `components/App.tsx` | 整个编辑器实例，包含大量内部方法 |
| AppClassProperties | `types.ts: AppClassProperties` | 供内部组件使用的 App 属性集合 |
| Scene 类 | `scene/Scene.ts` | 场景状态管理，内部方法不对外 |
| AppStateObserver 内部 | `AppStateObserver.ts` | 除 `onStateChange` 外的内部实现 |
| 内部事件总线 | `AppEventBus` | 不建议宿主直接订阅 |
| 子组件实例方法 | `LinearElementEditor` 等 | 编辑器内部组件方法 |

### ❌ 之前文档中的错误

| 之前描述 | 真实情况 |
|---------|---------|
| `addFiles(files: Array<File | HTMLImageElement | Blob>)` | ❌ 错误，真实签名是 `addFiles(data: BinaryFileData[])` |
| `onStateChange(observer, { fireImmediately })` | ❌ 错误，真实支持多种重载形式（见 1.3） |
| `AppEventBus.on("element:create")` | ❌ 这是内部 API，宿主不应依赖 |
| `api.setPlugins` | ❌ 不在 ExcalidrawImperativeAPI 中，是内部方法 |

---

## 五、嵌入内容通信机制 (Embeddable Communication)

### 5.1 iframe Sandbox 配置

所有嵌入内容默认运行在沙箱中：

```typescript
// packages/excalidraw/components/App.tsx
sandbox={`${
  src?.sandbox?.allowSameOrigin
    ? "allow-same-origin"
    : ""
} allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-presentation allow-downloads`}
```

### 5.2 内置平台双向通信

**YouTube 播放器**：
- 播放器通过 postMessage 发送 `infoDelivery` 事件（包含播放状态）
- 编辑器通过 postMessage 发送命令控制播放/暂停

**Vimeo 播放器**：
- 监听 `paused` 事件
- 发送播放/暂停命令控制

### 5.3 URL 白名单验证

```typescript
// packages/element/src/embeddable.ts
const ALLOWED_DOMAINS = new Set([
  "youtube.com", "youtu.be", "vimeo.com", "player.vimeo.com",
  "drive.google.com", "figma.com", "gist.github.com",
  "twitter.com", "x.com", "*.simplepdf.eu", "stackblitz.com",
  "val.town", "giphy.com", "reddit.com", "forms.microsoft.com",
]);

// 宿主可扩展验证
<Excalidraw
  validateEmbeddable={(url) => {
    // 返回 true 允许，false 拒绝，undefined 使用默认白名单
    return url.includes("internal-company.com");
  }}
/>
```

### 5.4 自定义嵌入渲染

```typescript
<Excalidraw
  renderEmbeddable={(element, appState) => {
    // 返回自定义 React 组件替代默认 iframe
    if (element.link?.includes("custom-app")) {
      return <CustomEmbedComponent link={element.link} />;
    }
    // 返回 null 使用默认渲染
    return null;
  }}
/>
```

---

## 六、典型集成模式 (Typical Integration Patterns)

### 6.1 受控模式

宿主完全控制场景状态：

```typescript
function ControlledExcalidraw() {
  const [api, setApi] = useState<ExcalidrawImperativeAPI | null>(null);

  return (
    <Excalidraw
      onExcalidrawAPI={setApi}
      onChange={(elements, appState, files) => {
        // 同步到宿主状态存储
        saveToHostState({ elements, appState, files });
      }}
    />
  );

  // 在需要时通过 api.updateScene 注入外部变更
}
```

### 6.2 远程协作模式

```typescript
function CollaborativeExcalidraw({ api }) {
  useEffect(() => {
    // 接收远程更新
    socket.on("remote:update", ({ elements, appState }) => {
      api.updateScene({
        elements,
        appState,
        captureUpdate: CaptureUpdateAction.NEVER,
      });
    });

    // 广播本地增量
    const unsubscribe = api.onIncrement((event) => {
      if (event.type === "durable") {
        socket.emit("local:update", event);
      }
    });

    return unsubscribe;
  }, [api, socket]);
}
```

### 6.3 只读预览模式

```typescript
<Excalidraw
  initialData={readOnlyData}
  viewModeEnabled={true}
  // 禁用所有编辑交互
  UIOptions={{
    canvasActions: {
      changeViewBackgroundColor: false,
      clearCanvas: false,
      export: false,
      loadScene: false,
      saveAsImage: false,
      saveSceneToLocalStorage: false,
    },
  }}
/>
```

---

## 文档信息

- **基于源码版本**：Excalidraw v0.17+
- **源码位置**：
  - API 类型定义：`packages/excalidraw/types.ts`
  - AppStateObserver：`packages/excalidraw/components/AppStateObserver.ts`
  - 嵌入相关：`packages/element/src/embeddable.ts`
- **最后更新**：2026-05-12
