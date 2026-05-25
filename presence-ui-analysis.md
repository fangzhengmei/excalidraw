# Excalidraw 多人协作 Presence 数据接入与光标渲染分析

## 一、核心数据结构定义

### 1.1 Collaborator 类型

**文件**: `packages/excalidraw/types.ts:70-90`

```typescript
export type Collaborator = Readonly<{
  pointer?: CollaboratorPointer;      // 坐标位置、工具类型
  button?: "up" | "down";             // 鼠标按键状态
  selectedElementIds?: Record<string, boolean>;  // 选中的元素
  username?: string | null;
  userState?: UserIdleState;          // ACTIVE | IDLE | AWAY
  color?: { background: string; stroke: string };
  avatarUrl?: string;
  id?: string;                        // 用户ID（用于去重）
  socketId?: SocketId;                // WebSocket连接ID
  isCurrentUser?: boolean;
  isInCall?: boolean;
  isSpeaking?: boolean;
  isMuted?: boolean;
}>;
```

### 1.2 AppState 中的存储位置

**文件**: `packages/excalidraw/appState.ts:29`

```typescript
export const getDefaultAppState = (): Omit<AppState, ...> => {
  return {
    collaborators: new Map(),  // Map<SocketId, Collaborator>
    // ...
  };
};
```

> **注意**: `collaborators` 在 `APP_STATE_STORAGE_CONF` 中被标记为 `browser: false, export: false, server: false`，不会持久化存储。

---

## 二、数据流向全景图

```
WebSocket 消息
      ↓
Collab.handleSocketMessage()  [Collab.tsx:610-626]
      ↓  (WS_SUBTYPES.MOUSE_LOCATION)
Collab.updateCollaborator()   [Collab.tsx:884-900]
      ↓  (更新 this.collaborators Map)
excalidrawAPI.updateScene({ collaborators })
      ↓
App.updateScene()             [App.tsx:4573-4621]
      ↓  (this.setState({ collaborators }))
      ├───────────────────────────────────┐
      ↓                                   ↓
UserList 组件                     InteractiveCanvas 组件
[LayerUI.tsx:404-408]            [InteractiveCanvas.tsx:96-136]
      ↓                                   ↓
显示用户头像 + 状态            转换为 renderConfig 并驱动 Canvas 渲染
                                  ↓
                            renderInteractiveScene()
                                  ↓
                            renderRemoteCursors()  [clients.ts:57-261]
                                  ↓
                            Canvas 2D 绘制光标 + 用户名标签
```

---

## 三、远端 Presence 数据接入本地提示组件（UserList）

### 3.1 数据接收与更新

**文件**: `excalidraw-app/collab/Collab.tsx:610-626`

```typescript
case WS_SUBTYPES.MOUSE_LOCATION: {
  const { pointer, button, username, selectedElementIds } = decryptedData.payload;
  const socketId = decryptedData.payload.socketId;
  
  this.updateCollaborator(socketId, {
    pointer,
    button,
    selectedElementIds,
    username,
  });
  break;
}
```

### 3.2 updateCollaborator 核心逻辑

**文件**: `excalidraw-app/collab/Collab.tsx:884-900`

```typescript
updateCollaborator = (socketId: SocketId, updates: Partial<Collaborator>) => {
  // 1. 创建新 Map（不可变更新）
  const collaborators = new Map(this.collaborators);
  // 2. 合并更新，标记是否为当前用户
  const user = Object.assign({}, collaborators.get(socketId), updates, {
    isCurrentUser: socketId === this.portal.socket?.id,
  });
  // 3. 更新 Map
  collaborators.set(socketId, user);
  this.collaborators = collaborators;
  // 4. 推送到 App state
  this.excalidrawAPI.updateScene({ collaborators });
};
```

### 3.3 App.updateScene 接收

**文件**: `packages/excalidraw/components/App.tsx:4573-4621`

```typescript
public updateScene = withBatchedUpdates(
  <K extends keyof AppState>(sceneData: {
    appState?: Pick<AppState, K> | null;
    collaborators?: SceneData["collaborators"];
    // ...
  }) => {
    const { elements, appState, collaborators, captureUpdate } = sceneData;
    // ...
    if (collaborators) {
      this.setState({ collaborators });  // 直接 setState
    }
  }
);
```

### 3.4 UserList 组件接入

**文件**: `packages/excalidraw/components/LayerUI.tsx:404-408`

```tsx
{appState.collaborators.size > 0 && (
  <UserList
    collaborators={appState.collaborators}
    userToFollow={appState.userToFollow?.socketId || null}
  />
)}
```

**文件**: `packages/excalidraw/components/UserList.tsx:112-264`

UserList 组件关键处理：

1. **去重逻辑**（`UserList.tsx:116-128`）：
   - 按 `userId`（优先）或 `socketId`（兜底）去重
   - 解决同一用户多端登录问题

2. **渲染每个协作者**（`UserList.tsx:50-82`）：
   - 调用 `actionManager.renderAction("goToCollaborator", data)`
   - 由 `actionGoToCollaborator.PanelComponent` 实际渲染

3. **goToCollaborator Action**（`actions/actionNavigate.tsx:21-144`）：
   - `perform`：设置 `appState.userToFollow`，开启跟随模式
   - `PanelComponent`：渲染头像 + 用户名 + 状态图标（通话中/说话中/静音）

---

## 四、光标渲染机制

### 4.1 InteractiveCanvas 数据转换层

**文件**: `packages/excalidraw/components/canvases/InteractiveCanvas.tsx:96-136`

这是最容易看漏的关键转换层！在 `useEffect` 中，每次 `appState` 变化时：

```typescript
useEffect(() => {
  // 初始化 4 个 Map
  const remotePointerButton = new Map();
  const remotePointerViewportCoords = new Map();
  const remoteSelectedElementIds = new Map();
  const remotePointerUsernames = new Map();
  const remotePointerUserStates = new Map();

  // 遍历 collaborators，逐个提取数据
  props.appState.collaborators.forEach((user, socketId) => {
    // 1. 收集选中元素（用于高亮显示谁在选哪个元素）
    if (user.selectedElementIds) {
      for (const id of Object.keys(user.selectedElementIds)) {
        if (!remoteSelectedElementIds.has(id)) {
          remoteSelectedElementIds.set(id, []);
        }
        remoteSelectedElementIds.get(id)!.push(socketId);
      }
    }
    
    // 2. 过滤：没有 pointer 或 renderCursor=false 则跳过
    if (!user.pointer || user.pointer.renderCursor === false) {
      return;
    }
    
    // 3. 提取用户名、用户状态
    if (user.username) {
      remotePointerUsernames.set(socketId, user.username);
    }
    if (user.userState) {
      remotePointerUserStates.set(socketId, user.userState);
    }
    
    // 4. 关键：场景坐标 → 视口坐标转换
    remotePointerViewportCoords.set(
      socketId,
      sceneCoordsToViewportCoords(
        { sceneX: user.pointer.x, sceneY: user.pointer.y },
        props.appState,
      ),
    );
    
    // 5. 鼠标按键状态
    remotePointerButton.set(socketId, user.button);
  });

  // 组装 renderConfig，触发渲染
  rendererParams.current = {
    // ...
    renderConfig: {
      remotePointerViewportCoords,
      remotePointerButton,
      remoteSelectedElementIds,
      remotePointerUsernames,
      remotePointerUserStates,
      // ...
    },
  };
});
```

> **关键点**:
> - `sceneCoordsToViewportCoords` 进行坐标转换，考虑 `scrollX/scrollY/zoom`
> - 这 5 个 Map 是 `InteractiveCanvasRenderConfig` 的一部分，定义在 `scene/types.ts:61-74`

### 4.2 渲染配置类型定义

**文件**: `packages/excalidraw/scene/types.ts:61-74`

```typescript
export type InteractiveCanvasRenderConfig = {
  remoteSelectedElementIds: Map<ExcalidrawElement["id"], SocketId[]>;
  remotePointerViewportCoords: Map<SocketId, { x: number; y: number }>;
  remotePointerUserStates: Map<SocketId, UserIdleState>;
  remotePointerUsernames: Map<SocketId, string>;
  remotePointerButton: Map<SocketId, string | undefined>;
  selectionColor: string;
  lastViewportPosition: { x: number; y: number };
  renderScrollbars?: boolean;
};
```

### 4.3 renderRemoteCursors 核心渲染

**文件**: `packages/excalidraw/clients.ts:57-261`

```typescript
export const renderRemoteCursors = ({
  context, renderConfig, appState, normalizedWidth, normalizedHeight
}) => {
  // 遍历所有远端指针
  for (const [socketId, pointer] of renderConfig.remotePointerViewportCoords) {
    let { x, y } = pointer;
    
    // 1. 获取协作者信息（用于颜色、说话状态等）
    const collaborator = appState.collaborators.get(socketId);
    
    // 2. 坐标修正（减去画布偏移）
    x -= appState.offsetLeft;
    y -= appState.offsetTop;
    
    // 3. 边界检测与裁剪
    const isOutOfBounds = x < 0 || x > normalizedWidth - width || 
                          y < 0 || y > normalizedHeight - height;
    x = Math.max(x, 0);
    x = Math.min(x, normalizedWidth - width);
    // ...
    
    // 4. 计算用户颜色（基于 socketId 哈希）
    const background = getClientColor(socketId, collaborator);
    
    // 5. 状态判断：是否不活跃（超出边界/IDLE/AWAY）
    const userState = renderConfig.remotePointerUserStates.get(socketId);
    const isInactive = isOutOfBounds ||
                       userState === UserIdleState.IDLE ||
                       userState === UserIdleState.AWAY;
    if (isInactive) {
      context.globalAlpha = 0.3;  // 半透明显示
    }
    
    // 6. 按下状态：绘制环形指示
    if (renderConfig.remotePointerButton.get(socketId) === "down") {
      context.beginPath();
      context.arc(x, y, 15, 0, 2 * Math.PI, false);
      context.lineWidth = 3;
      context.strokeStyle = "#ffffff88";
      context.stroke();
      // ...
    }
    
    // 7. 说话状态：特殊绿色边框 + 声波图标
    const isSpeaking = collaborator?.isSpeaking;
    if (isSpeaking) {
      context.fillStyle = IS_SPEAKING_COLOR;
      // 绘制光标外框
      // 绘制三个竖条表示说话中
      context.fillRect(boxX + boxWidth + margin, ..., 2, barheight);
      context.fillRect(boxX + boxWidth + margin + gap, ..., 2, barheight * 2);
      context.fillRect(boxX + boxWidth + margin + gap * 2, ..., 2, barheight);
    }
    
    // 8. 绘制箭头光标（三层：说话外框 → 白色描边 → 彩色填充）
    // 白色描边层
    context.fillStyle = COLOR_WHITE;
    context.lineWidth = 6;
    context.beginPath();
    context.moveTo(x, y);
    context.lineTo(x + 0, y + 14);
    context.lineTo(x + 4, y + 9);
    context.lineTo(x + 11, y + 8);
    context.closePath();
    context.stroke();
    context.fill();
    
    // 彩色填充层
    context.fillStyle = background;
    context.lineWidth = 2;
    // ...
    
    // 9. 绘制用户名标签
    const username = renderConfig.remotePointerUsernames.get(socketId) || "";
    if (!isOutOfBounds && username) {
      context.font = "600 12px sans-serif";
      // 绘制圆角背景框
      roundRect(context, boxX, boxY, boxWidth, boxHeight, 8, COLOR_WHITE);
      context.fillStyle = COLOR_CHARCOAL_BLACK;
      context.fillText(username, offsetX + paddingHorizontal + 1, ...);
    }
  }
};
```

### 4.4 颜色生成算法

**文件**: `packages/excalidraw/clients.ts:30-44`

```typescript
export const getClientColor = (socketId: SocketId, collaborator: Collaborator | undefined) => {
  // 对 id 做哈希，确保均匀分布
  const hash = Math.abs(hashToInteger(collaborator?.id || socketId));
  // 生成 0-360 范围内步长为 10 的色相值（共 37 种）
  const hue = (hash % 37) * 10;
  const saturation = 100;
  const lightness = 83;  // 浅色调，保证可读性
  return `hsl(${hue}, ${saturation}%, ${lightness}%)`;
};
```

---

## 五、容易看漏的关键点

### 5.1 数据转换层

`InteractiveCanvas.tsx` 的 `useEffect` 是最容易被忽略的关键层：
- 它不是直接使用 `appState.collaborators`
- 而是将其拆分为 5 个独立的 Map，结构更适合渲染
- 这里完成了 `场景坐标 → 视口坐标` 的转换

### 5.2 过滤机制

- `user.pointer.renderCursor === false` 时跳过光标渲染（只渲染激光轨迹）
- 超出画布边界时 `isOutOfBounds = true`，用户名标签不显示

### 5.3 渲染时机

- `InteractiveCanvas` 是 `React.memo` 组件，通过 `areEqual` 函数比较
- 关键比较项包含 `collaborators`（在 `getRelevantAppStateProps` 中）
- 动画由 `AnimationController` 驱动，每帧调用 `renderInteractiveScene`

### 5.4 状态优先级

```
isSpeaking (说话中) > button === "down" (按下中) > isInactive (不活跃)
```

不同状态会叠加不同的视觉效果。

---

## 六、本地上报流程（发送方）

### 6.1 指针移动节流上报

**文件**: `excalidraw-app/collab/Collab.tsx:914-925`

```typescript
onPointerUpdate = throttle(
  (payload: {
    pointer: { x: number; y: number; tool: "pointer" | "laser" };
    button: "up" | "down";
    pointersMap: Gesture["pointers"];
  }) => {
    payload.pointersMap.size < 2 &&
      this.portal.socket &&
      this.portal.broadcastMouseLocation(payload);
  },
  CURSOR_SYNC_TIMEOUT,  // 默认 50ms
);
```

### 6.2 节流时间常量

**文件**: `excalidraw-app/app_constants.ts`

```typescript
export const CURSOR_SYNC_TIMEOUT = 50;  // 50ms = 20fps
```

---

## 七、总结

| 层级 | 模块 | 职责 |
|------|------|------|
| 网络层 | `Collab.tsx` | 接收 WebSocket 消息，维护 `collaborators` Map |
| 状态层 | `App.tsx` | `appState.collaborators` 作为唯一数据源 |
| UI 层 | `UserList.tsx` | 显示在线用户头像，点击发起跟随 |
| 转换层 | `InteractiveCanvas.tsx` | **关键**：拆分为 5 个渲染专用 Map + 坐标转换 |
| 渲染层 | `renderRemoteCursors()` | Canvas 2D 绘制光标、用户名、状态指示 |
| 工具层 | `getClientColor()` | 生成用户专属颜色 |

> **最容易看漏**: `InteractiveCanvas.tsx` 中的 `useEffect` 转换层，它是连接 `appState.collaborators` 和 Canvas 渲染的桥梁。
