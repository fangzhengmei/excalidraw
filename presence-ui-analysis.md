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

### 1.2 UserIdleState 枚举

**文件**: `packages/common/src/constants.ts:496-500`

```typescript
export enum UserIdleState {
  ACTIVE = "active",   // 活跃状态
  AWAY = "away",       // 离开状态
  IDLE = "idle",       // 空闲状态
}
```

### 1.3 AppState 中的存储位置

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
                            AnimationController.start()
                                  ↓  (requestAnimationFrame)
                            renderInteractiveScene()
                                  ↓
                            renderRemoteCursors()  [clients.ts:57-261]
                                  ↓
                            Canvas 2D 绘制光标 + 用户名标签
```

---

## 三、关键时间常量与同步频率

### 3.1 指针同步频率（重点核准）

**文件**: `excalidraw-app/app_constants.ts:8`

```typescript
export const CURSOR_SYNC_TIMEOUT = 33;  // 33ms = ~30fps
```

> ⚠️ **重要修正**: 之前文档误写为 50ms，实际代码是 **33ms（约 30fps）**！

### 3.2 空闲状态检测阈值

**文件**: `packages/common/src/constants.ts:307-310`

```typescript
// Report a user inactive after IDLE_THRESHOLD milliseconds
export const IDLE_THRESHOLD = 60_000;   // 60秒无操作 → IDLE
// Report a user active each ACTIVE_THRESHOLD milliseconds
export const ACTIVE_THRESHOLD = 3_000;   // 每 3秒上报一次活跃状态
```

### 3.3 全量场景同步间隔

**文件**: `excalidraw-app/app_constants.ts:6`

```typescript
export const SYNC_FULL_SCENE_INTERVAL_MS = 20000;  // 20秒同步一次完整场景
```

### 3.4 时间常量汇总表

| 常量名 | 数值 | 含义 | 影响 |
|--------|------|------|------|
| `CURSOR_SYNC_TIMEOUT` | 33ms | 指针位置节流上报间隔 | 决定远端光标更新的**最大频率**（~30fps） |
| `IDLE_THRESHOLD` | 60,000ms | 空闲检测阈值 | 60秒无操作后用户标记为 `IDLE`，光标变半透明 |
| `ACTIVE_THRESHOLD` | 3,000ms | 活跃状态上报间隔 | 每3秒心跳式上报，防止误判为空闲 |
| `SYNC_FULL_SCENE_INTERVAL_MS` | 20,000ms | 全量场景同步间隔 | 兜底机制，防止增量同步丢失 |

---

## 四、渲染节奏控制机制

### 4.1 AnimationController 驱动

**文件**: `packages/excalidraw/renderer/animation.ts:8-84`

```typescript
export class AnimationController {
  private static animations = new Map<string, { animation, lastTime, state }>();

  static start<R extends object>(key: string, animation: Animation<R>) {
    // ...
    if (!AnimationController.isRunning) {
      AnimationController.isRunning = true;
      // React 18+ 使用 requestAnimationFrame（与浏览器刷新率同步）
      // React < 18 使用 setTimeout(..., 0)（不限制帧率）
      if (isRenderThrottlingEnabled()) {
        requestAnimationFrame(AnimationController.tick);
      } else {
        setTimeout(AnimationController.tick, 0);
      }
    }
  }

  private static tick() {
    if (AnimationController.animations.size > 0) {
      for (const [key, animation] of AnimationController.animations) {
        const now = performance.now();
        const deltaTime = animation.lastTime === 0 ? 0 : now - animation.lastTime;
        // 执行渲染回调
        const state = animation.animation({ deltaTime, state: animation.state });
        // ...
      }
      // 继续下一帧
      if (isRenderThrottlingEnabled()) {
        requestAnimationFrame(AnimationController.tick);
      } else {
        setTimeout(AnimationController.tick, 0);
      }
    }
  }
}
```

**关键点**:
- **React 18+**: 使用 `requestAnimationFrame`，与浏览器刷新率同步（通常 60fps）
- **React < 18**: 使用 `setTimeout(..., 0)`，尽可能快地渲染（可能 > 60fps）
- 动画一旦启动就会持续运行，直到 `state` 返回 `undefined/ null`

### 4.2 InteractiveCanvas 触发渲染

**文件**: `packages/excalidraw/components/canvases/InteractiveCanvas.tsx:173-198`

```typescript
useEffect(() => {
  // ... 数据转换 ...
  
  rendererParams.current = {
    app: props.app,
    canvas: props.canvas,
    // ...
    renderConfig: { /* 5 个 Map */ },
    // ...
  };

  // 只有动画未在运行时才启动
  if (!AnimationController.running(INTERACTIVE_SCENE_ANIMATION_KEY)) {
    AnimationController.start<InteractiveSceneRenderAnimationState>(
      INTERACTIVE_SCENE_ANIMATION_KEY,
      ({ deltaTime, state }) => {
        const nextAnimationState = renderInteractiveScene({
          ...rendererParams.current!,
          deltaTime,
          animationState: state,
        }).animationState;
        // 返回 state 则继续下一帧
        // 返回 undefined 则动画停止
        return nextAnimationState;
      },
    );
  }
});  // 无依赖数组 → 每次组件渲染都会执行！
```

> **容易看错**: 这个 `useEffect` **没有依赖数组**，每次 `InteractiveCanvas` 重新渲染都会执行。

### 4.3 React.memo 比较逻辑

**文件**: `packages/excalidraw/components/canvases/InteractiveCanvas.tsx:275-302`

```typescript
const areEqual = (prevProps, nextProps) => {
  // 快速比较：这些变化一定需要重渲染
  if (
    prevProps.selectionNonce !== nextProps.selectionNonce ||
    prevProps.canvasNonce !== nextProps.canvasNonce ||
    prevProps.scale !== nextProps.scale ||
    prevProps.elementsMap !== nextProps.elementsMap ||
    prevProps.visibleElements !== nextProps.visibleElements ||
    prevProps.selectedElements !== nextProps.selectedElements ||
    prevProps.renderScrollbars !== nextProps.renderScrollbars
  ) {
    return false;  // 不相等 → 重渲染
  }

  // 深度比较：只比较 InteractiveCanvas 关心的 AppState 字段
  // 包含 collaborators！
  return isShallowEqual(
    getRelevantAppStateProps(prevProps.appState as AppState),
    getRelevantAppStateProps(nextProps.appState as AppState),
  );
};

export default React.memo(InteractiveCanvas, areEqual);
```

**`getRelevantAppStateProps` 包含的关键字段**（`InteractiveCanvas.tsx:232-273`）：
```typescript
const getRelevantAppStateProps = (appState: AppState): InteractiveCanvasAppState => ({
  // ...
  collaborators: appState.collaborators,  // ⭐ Necessary for collab. sessions
  // ...
});
```

### 4.4 渲染节奏总结

```
网络接收频率:  ~30fps (CURSOR_SYNC_TIMEOUT = 33ms)
       ↓
React setState: 触发 App 重渲染
       ↓
React.memo 比较: collaborators 变化 → InteractiveCanvas 重渲染
       ↓
useEffect 执行: 更新 rendererParams.current
       ↓
AnimationController: 若未运行则启动（requestAnimationFrame → ~60fps）
       ↓
Canvas 渲染: 每帧调用 renderInteractiveScene()
       ↓
动画停止条件: animationState 无未完成动画时返回 undefined
```

**关键理解**:
1. **网络频率 ≠ 渲染频率**: 网络 30fps 上报，但 Canvas 以 60fps 渲染（如果有动画）
2. **动画常驻**: 只要有协作指针，`rendererParams.current` 就会更新，但动画是否持续运行取决于 `animationState`
3. **无动画时**: 如果 `bindingHighlight` 等动画完成，动画会停止，直到下一次状态更新

---

## 五、远端 Presence 数据接入本地提示组件（UserList）

### 5.1 数据接收与更新

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

### 5.2 updateCollaborator 核心逻辑

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

### 5.3 App.updateScene 接收

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

### 5.4 UserList 组件接入

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

## 六、光标渲染机制

### 6.1 InteractiveCanvas 数据转换层

**文件**: `packages/excalidraw/components/canvases/InteractiveCanvas.tsx:96-136`

这是最容易看漏的关键转换层！在 `useEffect` 中，每次 `appState` 变化时：

```typescript
useEffect(() => {
  // 初始化 5 个 Map
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
> - 这 5 个 Map 是 `InteractiveCanvasRenderConfig` 的一部分

### 6.2 渲染配置类型定义

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

### 6.3 renderRemoteCursors 核心渲染（状态判断详解）

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
    
    const width = 11;
    const height = 14;
    
    // ═══════════════════════════════════════════════════════
    // 状态判断 1: 边界检测
    // ═══════════════════════════════════════════════════════
    const isOutOfBounds =
      x < 0 ||
      x > normalizedWidth - width ||
      y < 0 ||
      y > normalizedHeight - height;
    
    // 边界裁剪：确保光标至少部分可见
    x = Math.max(x, 0);
    x = Math.min(x, normalizedWidth - width);
    y = Math.max(y, 0);
    y = Math.min(y, normalizedHeight - height);
    
    // 3. 计算用户颜色（基于 socketId 哈希）
    const background = getClientColor(socketId, collaborator);
    
    context.save();
    context.strokeStyle = background;
    context.fillStyle = background;
    
    // ═══════════════════════════════════════════════════════
    // 状态判断 2: 不活跃状态（IDLE / AWAY / 超出边界）
    // ═══════════════════════════════════════════════════════
    const userState = renderConfig.remotePointerUserStates.get(socketId);
    const isInactive =
      isOutOfBounds ||
      userState === UserIdleState.IDLE ||
      userState === UserIdleState.AWAY;
    
    if (isInactive) {
      context.globalAlpha = 0.3;  // 半透明显示
    }
    
    // ═══════════════════════════════════════════════════════
    // 状态判断 3: 鼠标按下状态（拖拽/绘制中）
    // ═══════════════════════════════════════════════════════
    if (renderConfig.remotePointerButton.get(socketId) === "down") {
      // 绘制双层环形指示
      context.beginPath();
      context.arc(x, y, 15, 0, 2 * Math.PI, false);
      context.lineWidth = 3;
      context.strokeStyle = "#ffffff88";  // 外层白色半透明
      context.stroke();
      context.closePath();
      
      context.beginPath();
      context.arc(x, y, 15, 0, 2 * Math.PI, false);
      context.lineWidth = 1;
      context.strokeStyle = background;  // 内层用户颜色
      context.stroke();
      context.closePath();
    }
    
    // 说话状态颜色
    const IS_SPEAKING_COLOR =
      appState.theme === THEME.DARK ? "#2f6330" : COLOR_VOICE_CALL;
    
    // ═══════════════════════════════════════════════════════
    // 状态判断 4: 说话中（语音通话）
    // ═══════════════════════════════════════════════════════
    const isSpeaking = collaborator?.isSpeaking;
    
    if (isSpeaking) {
      // 光标绿色外框（10px 粗）
      context.fillStyle = IS_SPEAKING_COLOR;
      context.strokeStyle = IS_SPEAKING_COLOR;
      context.lineWidth = 10;
      context.lineJoin = "round";
      context.beginPath();
      context.moveTo(x, y);
      context.lineTo(x + 0, y + 14);
      context.lineTo(x + 4, y + 9);
      context.lineTo(x + 11, y + 8);
      context.closePath();
      context.stroke();
      context.fill();
    }
    
    // 白色描边层（6px 粗）- 确保光标在任何背景上可见
    context.fillStyle = COLOR_WHITE;
    context.strokeStyle = COLOR_WHITE;
    context.lineWidth = 6;
    context.lineJoin = "round";
    context.beginPath();
    context.moveTo(x, y);
    context.lineTo(x + 0, y + 14);
    context.lineTo(x + 4, y + 9);
    context.lineTo(x + 11, y + 8);
    context.closePath();
    context.stroke();
    context.fill();
    
    // 用户颜色填充层（2px 描边）
    context.fillStyle = background;
    context.strokeStyle = background;
    context.lineWidth = 2;
    context.lineJoin = "round";
    context.beginPath();
    if (isInactive) {
      // 不活跃时光标稍微偏移
      context.moveTo(x - 1, y - 1);
      context.lineTo(x - 1, y + 15);
      context.lineTo(x + 5, y + 10);
      context.lineTo(x + 12, y + 9);
      context.closePath();
      context.fill();
    } else {
      context.moveTo(x, y);
      context.lineTo(x + 0, y + 14);
      context.lineTo(x + 4, y + 9);
      context.lineTo(x + 11, y + 8);
      context.closePath();
      context.fill();
      context.stroke();
    }
    
    // ═══════════════════════════════════════════════════════
    // 用户名标签渲染
    // ═══════════════════════════════════════════════════════
    const username = renderConfig.remotePointerUsernames.get(socketId) || "";
    
    // 只有在边界内且有用户名时才显示标签
    if (!isOutOfBounds && username) {
      context.font = "600 12px sans-serif";
      
      // 说话中时标签位置稍微偏移
      const offsetX = (isSpeaking ? x + 0 : x) + width / 2;
      const offsetY = (isSpeaking ? y + 0 : y) + height + 2;
      const paddingHorizontal = 5;
      const paddingVertical = 3;
      
      const measure = context.measureText(username);
      const measureHeight =
        measure.actualBoundingBoxDescent + measure.actualBoundingBoxAscent;
      const finalHeight = Math.max(measureHeight, 12);
      
      const boxX = offsetX - 1;
      const boxY = offsetY - 1;
      const boxWidth = measure.width + 2 + paddingHorizontal * 2 + 2;
      const boxHeight = finalHeight + 2 + paddingVertical * 2 + 2;
      
      // 绘制标签背景框
      if (context.roundRect) {
        context.beginPath();
        context.roundRect(boxX, boxY, boxWidth, boxHeight, 8);
        context.fillStyle = background;
        context.fill();
        context.strokeStyle = COLOR_WHITE;
        context.stroke();
        
        // 说话中：额外绿色边框
        if (isSpeaking) {
          context.beginPath();
          context.roundRect(boxX - 2, boxY - 2, boxWidth + 4, boxHeight + 4, 8);
          context.strokeStyle = IS_SPEAKING_COLOR;
          context.stroke();
        }
      }
      
      context.fillStyle = COLOR_CHARCOAL_BLACK;
      context.fillText(
        username,
        offsetX + paddingHorizontal + 1,
        offsetY + paddingVertical + measure.actualBoundingBoxAscent + ...,
      );
      
      // ═══════════════════════════════════════════════════════
      // 说话中：绘制声波图标（三个竖条）
      // ═══════════════════════════════════════════════════════
      if (isSpeaking) {
        context.fillStyle = IS_SPEAKING_COLOR;
        const barheight = 8;
        const margin = 8;
        const gap = 5;
        // 左条（短）
        context.fillRect(boxX + boxWidth + margin, boxY + (boxHeight / 2 - barheight / 2), 2, barheight);
        // 中条（长）- 表示音量最大
        context.fillRect(boxX + boxWidth + margin + gap, boxY + (boxHeight / 2 - barheight), 2, barheight * 2);
        // 右条（短）
        context.fillRect(boxX + boxWidth + margin + gap * 2, boxY + (boxHeight / 2 - barheight / 2), 2, barheight);
      }
    }
    
    context.restore();
    context.closePath();
  }
};
```

### 6.4 状态判断汇总与视觉效果

| 状态 | 判断条件 | 视觉效果 |
|------|----------|----------|
| **正常活跃** | `!isInactive && !isSpeaking && button !== "down"` | 标准箭头光标 + 用户名标签 |
| **不活跃** | `isOutOfBounds` 或 `userState === IDLE/AWAY` | `globalAlpha = 0.3` 半透明，光标微偏移 |
| **按下中** | `button === "down"` | 光标周围绘制双层环形（白色+用户色） |
| **说话中** | `isSpeaking === true` | 1. 光标 10px 绿色外框<br>2. 用户名标签绿色外框<br>3. 标签右侧声波图标（三竖条） |

> **状态叠加**: 多种状态可以同时生效（例如说话中 + 按下中 + 不活跃），视觉效果是叠加的。

### 6.5 颜色生成算法

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

## 七、本地上报流程（发送方）

### 7.1 指针移动节流上报

**文件**: `excalidraw-app/collab/Collab.tsx:914-925`

```typescript
onPointerUpdate = throttle(
  (payload: {
    pointer: { x: number; y: number; tool: "pointer" | "laser" };
    button: "up" | "down";
    pointersMap: Gesture["pointers"];
  }) => {
    // 单指操作时才上报（避免手势操作产生大量消息）
    payload.pointersMap.size < 2 &&
      this.portal.socket &&
      this.portal.broadcastMouseLocation(payload);
  },
  CURSOR_SYNC_TIMEOUT,  // 33ms = ~30fps
);
```

### 7.2 空闲状态检测流程

**文件**: `excalidraw-app/collab/Collab.tsx:810-850`

```typescript
onUserActivity = () => {
  // 清除空闲计时器
  if (this.idleTimeoutId) {
    window.clearTimeout(this.idleTimeoutId);
    this.idleTimeoutId = null;
  }

  // 60秒后进入 IDLE 状态
  this.idleTimeoutId = window.setTimeout(this.reportIdle, IDLE_THRESHOLD);

  // 如果活跃定时器没启动，则启动（每3秒上报一次活跃）
  if (!this.activeIntervalId) {
    this.activeIntervalId = window.setInterval(
      this.reportActive,
      ACTIVE_THRESHOLD,
    );
  }
};
```

---

## 八、容易看漏的关键点

### 8.1 数据转换层

`InteractiveCanvas.tsx` 的 `useEffect` 是最容易被忽略的关键层：
- 它不是直接使用 `appState.collaborators`
- 而是将其拆分为 5 个独立的 Map，结构更适合渲染
- 这里完成了 `场景坐标 → 视口坐标` 的转换
- **这个 useEffect 没有依赖数组**，每次组件渲染都会执行

### 8.2 过滤机制

- `user.pointer.renderCursor === false` 时跳过光标渲染（只渲染激光轨迹）
- 超出画布边界时 `isOutOfBounds = true`，用户名标签不显示
- 多指触控时 `pointersMap.size >= 2`，不上报指针位置

### 8.3 渲染节奏

- **网络上报**: 33ms 节流（~30fps）
- **Canvas 渲染**: requestAnimationFrame 驱动（~60fps，React 18+）
- **动画启停**: 取决于 `animationState` 是否有未完成动画
- **React.memo**: `collaborators` 是比较字段之一，变化就会重渲染

### 8.4 状态优先级

```
isSpeaking (说话中) > button === "down" (按下中) > isInactive (不活跃)
```

不同状态会叠加不同的视觉效果。

---

## 九、总结

| 层级 | 模块 | 职责 | 关键数值 |
|------|------|------|----------|
| 网络层 | `Collab.tsx` | 接收 WebSocket 消息，维护 `collaborators` Map | CURSOR_SYNC_TIMEOUT = 33ms |
| 状态层 | `App.tsx` | `appState.collaborators` 作为唯一数据源 | IDLE_THRESHOLD = 60s |
| UI 层 | `UserList.tsx` | 显示在线用户头像，点击发起跟随 | ACTIVE_THRESHOLD = 3s |
| 转换层 | `InteractiveCanvas.tsx` | **关键**：拆分为 5 个渲染专用 Map + 坐标转换 | useEffect 无依赖 |
| 动画层 | `AnimationController` | 驱动渲染循环 | requestAnimationFrame (~60fps) |
| 渲染层 | `renderRemoteCursors()` | Canvas 2D 绘制光标、用户名、状态指示 | 4 种状态叠加 |
| 工具层 | `getClientColor()` | 生成用户专属颜色 | 37 种色相值 |

> **最容易看漏的三点**:
> 1. `InteractiveCanvas.tsx` 中的 `useEffect` 转换层（无依赖数组）
> 2. `CURSOR_SYNC_TIMEOUT` 实际是 33ms（~30fps），不是 50ms
> 3. 渲染由 `AnimationController` 驱动，帧率取决于 React 版本和浏览器刷新率
