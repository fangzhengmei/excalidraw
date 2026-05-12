# Excalidraw 嵌入桥接机制 (Embed Bridge)

本文档描述 Excalidraw 画板如何嵌入到第三方网页中，宿主如何通过 API 注入命令、监听内部状态变化，以及如何拦截或扩展默认行为。

---

## 1. API 暴露面 (API Surface)

Excalidraw 提供两层 API 供宿主应用集成：声明式 Props 和命令式 Imperative API。

### 1.1 声明式 Props (ExcalidrawProps)

通过 React Props 配置组件行为和订阅事件：

```typescript
interface ExcalidrawProps {
  // 场景变化回调 - 元素、状态、文件变化时触发
  onChange?: (
    elements: readonly OrderedExcalidrawElement[],
    appState: AppState,
    files: BinaryFiles
  ) => void;

  // 增量更新回调 - 细粒度状态变更
  onIncrement?: (event: DurableIncrement | EphemeralIncrement) => void;

  // API 就绪回调 - 编辑器挂载前触发
  onExcalidrawAPI?: (api: ExcalidrawImperativeAPI | null) => void;

  // 编辑器挂载完成回调
  onMount?: (payload: ExcalidrawMountPayload) => void;

  // 编辑器卸载回调
  onUnmount?: () => void;

  // 场景初始化完成回调
  onInitialize?: (api: ExcalidrawImperativeAPI) => void;

  // 指针更新回调
  onPointerUpdate?: (payload: {
    pointer: { x: number; y: number; tool: "pointer" | "laser" };
    button: "down" | "up";
    pointersMap: Gesture["pointers"];
  }) => void;

  // 粘贴事件拦截
  onPaste?: (
    data: ClipboardData,
    event: ClipboardEvent | null
  ) => Promise<boolean> | boolean;

  // 嵌入 URL 验证
  validateEmbeddable?: 
    | boolean 
    | string[] 
    | RegExp 
    | ((link: string) => boolean | undefined);

  // 自定义嵌入元素渲染
  renderEmbeddable?: (
    element: NonDeleted<ExcalidrawEmbeddableElement>,
    appState: AppState
  ) => JSX.Element | null;

  // 导出进度回调
  onExport?: (exportOpts: ExportOpts) => void;
  onExportProgress?: (status: OnExportProgress) => void;
}
```

### 1.2 命令式 API (ExcalidrawImperativeAPI)

通过 `onExcalidrawAPI` 或 `onMount` 获取实例后调用：

```typescript
interface ExcalidrawImperativeAPI {
  // 核心场景操作
  updateScene: (opts: {
    elements?: readonly ExcalidrawElement[];
    appState?: DeepPartial<AppState>;
    captureUpdate?: CaptureUpdateActionType;
  }) => void;

  // 应用状态管理
  setAppState: (appState: DeepPartial<AppState>) => void;

  // 元素操作
  addFiles: (files: Array<File | HTMLImageElement | Blob>) => Promise<BinaryFiles>;
  getSceneElementsIncludingDeleted: () => readonly ExcalidrawElement[];
  getSceneElements: () => readonly NonDeletedExcalidrawElement[];
  getSelectedElements: () => readonly NonDeletedExcalidrawElement[];

  // 历史操作
  history: {
    clear: () => void;
  };

  // 重置画布
  resetScene: (opts?: {
    resetLoadingState?: boolean;
    preserveSession?: boolean;
  }) => void;

  // 刷新尺寸
  refresh: () => void;

  // 事件订阅
  onEvent: <K extends keyof ExcalidrawImperativeAPIEventMap>(
    name: K,
    callback: (...args: ExcalidrawImperativeAPIEventMap[K]) => void
  ) => UnsubscribeCallback;

  onStateChange: (
    observer: OnStateChange,
    options?: { fireImmediately?: boolean }
  ) => UnsubscribeCallback;
}
```

---

## 2. 内部事件外发 (Internal Event Forwarding)

### 2.1 场景状态变化广播

**触发时机**：元素增删改、视图变化、工具切换等任何影响场景的操作

```typescript
// 方式一：全量 onChange 回调
<Excalidraw
  onChange={(elements, appState, files) => {
    // 宿主接收完整最新状态
    console.log("Elements changed:", elements);
    console.log("App state:", appState);
    console.log("Files:", files);
  }}
/>

// 方式二：增量 onIncrement 回调
<Excalidraw
  onIncrement={(increment) => {
    // 接收增量变更（性能更优）
    // DurableIncrement: 持久化变更（元素更新等）
    // EphemeralIncrement: 临时变更（鼠标位置等）
  }}
/>
```

### 2.2 指针协作事件

**触发时机**：鼠标/触摸移动、按下/抬起

```typescript
<Excalidraw
  onPointerUpdate={({ pointer, button, pointersMap }) => {
    // 广播指针位置到协作服务器
    broadcastPointerPosition({
      x: pointer.x,
      y: pointer.y,
      tool: pointer.tool,
      button
    });
  }}
/>
```

### 2.3 状态观察器模式

细粒度订阅特定状态变化：

```typescript
api.onStateChange(
  (state) => {
    // 只响应选中元素变化
    console.log("Selected elements:", state.selectedElementIds);
  },
  { 
    fireImmediately: true,  // 订阅时立即触发一次
    observedNodes: ["selectedElementIds", "activeTool"]  // 只监听特定字段
  }
);
```

### 2.4 生命周期事件

```typescript
api.onEvent("editor:mount", ({ container }) => {
  console.log("Excalidraw mounted on:", container);
});

api.onEvent("editor:initialize", (api) => {
  console.log("Excalidraw ready with API");
});

api.onEvent("editor:unmount", () => {
  console.log("Excalidraw unmounted");
});
```

---

## 3. 外部命令注入 (External Command Injection)

### 3.1 场景注入 (updateScene)

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
      // ...其他属性
    }
  ],
  // CAPTURE: 加入历史栈（可撤销）
  // EVENTUALLY: 延迟加入历史栈
  // NEVER: 不加入历史栈（远程更新推荐）
  captureUpdate: CaptureUpdateAction.NEVER
});

// 只更新应用状态
api.updateScene({
  appState: {
    activeTool: { type: "laser" },
    theme: "dark",
    viewBackgroundColor: "#1a1a1a"
  }
});

// 快捷方式：只更新状态
api.setAppState({
  selectedElementIds: { "element-id": true }
});
```

### 3.2 初始数据注入

通过 `initialData` 预加载场景：

```typescript
<Excalidraw
  initialData={{
    elements: savedElements,
    appState: { theme: "dark" },
    files: savedFiles
  }}
  // 或异步加载
  initialData={async () => {
    const data = await fetchFromServer();
    return data;
  }}
/>
```

### 3.3 文件注入

```typescript
// 注入图片文件
const files = await api.addFiles([
  new File([blob], "image.png", { type: "image/png" })
]);

// 创建图片元素
api.updateScene({
  elements: [
    {
      type: "image",
      fileId: files[0].id,
      x: 0,
      y: 0,
      width: 400,
      height: 300
    }
  ]
});
```

### 3.4 历史操作控制

```typescript
// 清空历史栈（远程协作时常用）
api.history.clear();

// 重置整个场景
api.resetScene({
  resetLoadingState: true,
  preserveSession: false
});
```

---

## 4. 嵌入内容通信 (Embeddable PostMessage Bridge)

### 4.1 iframe Sandbox 配置

嵌入元素通过严格的沙箱配置运行：

```typescript
// packages/excalidraw/components/App.tsx:1849-1853
sandbox={`${
  src?.sandbox?.allowSameOrigin
    ? "allow-same-origin"
    : ""
} allow-scripts allow-forms allow-popups allow-popups-to-escape-sandbox allow-presentation allow-downloads`}
```

### 4.2 内置平台通信

**YouTube 视频控制**：

```typescript
// 监听 YouTube 播放器状态
// packages/excalidraw/components/App.tsx:870-930
case "https://www.youtube.com":
  if (data.event === "infoDelivery" && data.info.playerState) {
    YOUTUBE_VIDEO_STATES.set(data.id, data.info.playerState);
  }
  break;

// 发送播放/暂停命令
iframe.contentWindow.postMessage(
  JSON.stringify({
    event: "command",
    func: "playVideo" | "pauseVideo",
    args: ""
  }),
  "*"
);
```

**Vimeo 视频控制**：

```typescript
// 监听 Vimeo 暂停事件
case "https://player.vimeo.com":
  if (data.method === "paused") {
    source?.postMessage(
      JSON.stringify({
        method: data.value ? "play" : "pause",
        value: true
      }),
      "*"
    );
  }
  break;
```

### 4.3 URL 白名单验证

```typescript
// packages/element/src/embeddable.ts:452-535
const ALLOWED_DOMAINS = new Set([
  "youtube.com", "youtu.be", "vimeo.com", "player.vimeo.com",
  "drive.google.com", "figma.com", "gist.github.com",
  "twitter.com", "x.com", "*.simplepdf.eu", "stackblitz.com",
  "val.town", "giphy.com", "reddit.com", "forms.microsoft.com"
]);

// 宿主可扩展验证
<Excalidraw
  validateEmbeddable={(url) => {
    // 返回 true 允许，false 拒绝，undefined 使用默认白名单
    return url.includes("internal-company.com");
  }}
/>
```

### 4.4 自定义嵌入渲染

```typescript
<Excalidraw
  renderEmbeddable={(element, appState) => {
    // 返回自定义 React 组件替代默认 iframe
    if (element.link?.includes("custom-app")) {
      return (
        <CustomEmbed
          link={element.link}
          onCommand={(cmd) => handleEmbedCommand(element.id, cmd)}
        />
      );
    }
    // 返回 null 使用默认渲染
    return null;
  }}
/>
```

---

## 5. 拦截与扩展机制 (Interception & Extension)

### 5.1 粘贴拦截

```typescript
<Excalidraw
  onPaste={async (data, event) => {
    // 拦截自定义内容
    if (data.text?.startsWith("custom-payload:")) {
      // 注入自定义元素
      api.updateScene({ elements: createCustomElements(data.text) });
      // 返回 true 阻止默认粘贴行为
      return true;
    }
    // 返回 false 使用默认粘贴逻辑
    return false;
  }}
/>
```

### 5.2 事件总线扩展

通过 `AppEventBus` 订阅内部事件：

```typescript
import { AppEventBus } from "@excalidraw/common";

// 订阅全局事件
AppEventBus.on("element:create", (element) => {
  console.log("Element created:", element);
});
```

### 5.3 自定义工具扩展

通过 Plugin API 扩展工具栏和工具：

```typescript
api.setPlugins([
  {
    name: "custom-tool",
    tools: [
      {
        type: "custom",
        icon: CustomIcon,
        name: "Custom Tool",
        onPointerDown: (event, app) => {
          // 自定义工具逻辑
        }
      }
    ]
  }
]);
```

---

## 6. 典型集成模式 (Typical Integration Patterns)

### 6.1 受控模式 (Controlled Mode)

宿主完全控制场景状态：

```typescript
function ControlledExcalidraw() {
  const [elements, setElements] = useState([]);
  const [appState, setAppState] = useState({});

  return (
    <Excalidraw
      initialData={{ elements, appState }}
      onChange={(newElements, newAppState) => {
        // 1. 同步到宿主状态
        setElements(newElements);
        setAppState(newAppState);
        // 2. 同步到服务器
        saveToServer(newElements, newAppState);
      }}
    />
  );
}
```

### 6.2 远程协作模式 (Collaboration Mode)

```typescript
function CollaborativeExcalidraw() {
  useEffect(() => {
    // 接收远程更新
    socket.on("remote:update", ({ elements, appState }) => {
      api.updateScene({
        elements,
        appState,
        captureUpdate: CaptureUpdateAction.NEVER  // 不加入本地历史
      });
    });

    // 广播本地更新
    api.onIncrement((increment) => {
      if (increment.type === "durable") {
        socket.emit("local:update", increment);
      }
    });
  }, [api]);
}
```

### 6.3 只读预览模式 (Read-only Mode)

```typescript
<Excalidraw
  initialData={readOnlyData}
  viewModeEnabled={true}
  // 禁用所有编辑操作
  onPointerUpdate={() => {}}
  onChange={() => {}}
/>
```

---

## 7. 安全注意事项 (Security Considerations)

1. **Sandbox 隔离**：嵌入 iframe 默认启用沙箱，最小权限原则
2. **Origin 验证**：PostMessage 处理前必须验证 `event.origin`
3. **URL 白名单**：嵌入内容必须通过 `validateEmbeddable` 验证
4. **XSS 防护**：嵌入链接经过 HTML 转义处理 (`escapeDoubleQuotes`)
5. **CSP 兼容**：Excalidraw 内容安全策略需允许指定嵌入源

---

## 8. 性能优化建议 (Performance Tips)

1. **优先使用增量回调**：`onIncrement` 比 `onChange` 性能更好
2. **批量更新**：合并多次 `updateScene` 调用减少重渲染
3. **远程更新禁用历史**：使用 `CaptureUpdateAction.NEVER` 避免历史栈膨胀
4. **懒加载嵌入内容**：只有当嵌入元素进入视口时才初始化 iframe

---

**文档版本**：1.0  
**最后更新**：2026-05-12  
**适用版本**：Excalidraw v0.17+
