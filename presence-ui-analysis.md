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

---

## 十、房间成员变更到在线列表更新的完整链路

### 10.1 服务端推送入口

**文件**: `excalidraw-app/collab/Portal.tsx:56-58`

```typescript
this.socket.on("room-user-change", (clients: SocketId[]) => {
  this.collab.setCollaborators(clients);
});
```

当房间成员发生变化（有人加入/离开），服务端通过 Socket.IO 推送 `room-user-change` 事件，携带当前房间内所有 `SocketId[]`。

### 10.2 setCollaborators 构建在线 Map

**文件**: `excalidraw-app/collab/Collab.tsx:869-882`

```typescript
setCollaborators(sockets: SocketId[]) {
  const collaborators: InstanceType<typeof Collab>["collaborators"] = new Map();
  for (const socketId of sockets) {
    collaborators.set(
      socketId,
      Object.assign({}, this.collaborators.get(socketId), {
        isCurrentUser: socketId === this.portal.socket?.id,
      }),
    );
  }
  this.collaborators = collaborators;
  this.excalidrawAPI.updateScene({ collaborators });
}
```

> **容易混淆**: 这里不是增量更新，而是**全量重建** `collaborators` Map。但通过 `Object.assign({}, this.collaborators.get(socketId), ...)` 保留了已有用户的 `pointer`、`button`、`userState` 等字段。

### 10.3 跟随模式的联动清理

**文件**: `packages/excalidraw/components/App.tsx:3422-3424`

```typescript
const hasFollowedPersonLeft =
  prevState.userToFollow &&
  !this.state.collaborators.has(prevState.userToFollow.socketId);

if (hasFollowedPersonLeft) {
  this.maybeUnfollowRemoteUser();
}
```

`componentDidUpdate` 中检测被跟随的用户是否离开，如果离开则自动取消跟随。

### 10.4 UserList 在线列表渲染

**文件**: `packages/excalidraw/components/UserList.tsx:50-82`

```tsx
export const UserList = ({ collaborators, userToFollow }) => {
  const uniqueCollaborators = Array.from(collaborators.values()).filter(
    (collaborator, index, self) => {
      const firstIndex = self.findIndex(
        (c) => c.id === collaborator.id || c.socketId === collaborator.socketId,
      );
      return firstIndex === index;
    },
  );

  return (
    <div className="excalidraw-user-list">
      {uniqueCollaborators.map((collaborator) => {
        const data = { collaborator, userToFollow };
        return actionManager.renderAction("goToCollaborator", data);
      })}
    </div>
  );
};
```

### 10.5 完整链路图

```
服务端 room-user-change 事件 (SocketId[])
      ↓
Portal.socket.on("room-user-change")  [Portal.tsx:56]
      ↓
Collab.setCollaborators(clients)  [Collab.tsx:869]
      │  ├─ 创建新 Map（全量重建）
      │  ├─ 保留已有用户的 pointer/button/userState
      │  └─ 标记 isCurrentUser
      ↓
excalidrawAPI.updateScene({ collaborators })
      ↓
App.setState({ collaborators })  [App.tsx:4617]
      │
      ├─────────────────────────────────────────┐
      ↓                                         ↓
App.componentDidUpdate                    LayerUI 渲染
  ├─ 检查 followedPersonLeft?               ├─ userToFollow 存在?
  └─ 离开则取消跟随                           └─ 渲染 FollowMode 组件
                                                  │
                                                  ↓
                                            UserList 组件
                                              ├─ 按 id/socketId 去重
                                              └─ 渲染头像 + 用户名 + 状态
```

---

## 十一、IDLE 状态消息到光标透明度变化的完整链路

### 11.1 本地 IDLE 检测与上报

**文件**: `excalidraw-app/collab/Collab.tsx:860-867`

```typescript
private reportIdle = () => {
  this.onIdleStateChange(UserIdleState.IDLE);
};

private reportActive = () => {
  this.onIdleStateChange(UserIdleState.ACTIVE);
};
```

**空闲检测启动**（`Collab.tsx:810-850`）:

```typescript
onUserActivity = () => {
  if (this.idleTimeoutId) {
    window.clearTimeout(this.idleTimeoutId);
    this.idleTimeoutId = null;
  }
  // 60秒后进入 IDLE
  this.idleTimeoutId = window.setTimeout(this.reportIdle, IDLE_THRESHOLD);

  if (!this.activeIntervalId) {
    // 每3秒上报一次 ACTIVE（心跳）
    this.activeIntervalId = window.setInterval(this.reportActive, ACTIVE_THRESHOLD);
  }
};
```

### 11.2 IDLE 消息通过 volatile 通道发送

**文件**: `excalidraw-app/collab/Portal.tsx:185-199`

```typescript
broadcastIdleChange = (userState: UserIdleState) => {
  if (this.socket?.id) {
    const data: SocketUpdateDataSource["IDLE_STATUS"] = {
      type: WS_SUBTYPES.IDLE_STATUS,
      payload: {
        socketId: this.socket.id as SocketId,
        userState,
        username: this.collab.state.username,
      },
    };
    return this._broadcastSocketData(
      data as SocketUpdateData,
      true, // volatile — ⚠️ 这是 volatile 消息！
    );
  }
};
```

### 11.3 远端接收 IDLE 状态

**文件**: `excalidraw-app/collab/Collab.tsx:663-670`

```typescript
case WS_SUBTYPES.IDLE_STATUS: {
  const { userState, socketId, username } = decryptedData.payload;
  this.updateCollaborator(socketId, {
    userState,
    username,
  });
  break;
}
```

### 11.4 updateCollaborator 更新 AppState

**文件**: `excalidraw-app/collab/Collab.tsx:884-900`

```typescript
updateCollaborator = (socketId: SocketId, updates: Partial<Collaborator>) => {
  const collaborators = new Map(this.collaborators);
  const user: Mutable<Collaborator> = Object.assign(
    {},
    collaborators.get(socketId),
    updates,  // { userState: "idle", username: "..." }
    { isCurrentUser: socketId === this.portal.socket?.id },
  );
  collaborators.set(socketId, user);
  this.collaborators = collaborators;
  this.excalidrawAPI.updateScene({ collaborators });
};
```

### 11.5 InteractiveCanvas 转换层提取 userState

**文件**: `packages/excalidraw/components/canvases/InteractiveCanvas.tsx:120-121`

```typescript
if (user.userState) {
  remotePointerUserStates.set(socketId, user.userState);
}
```

### 11.6 renderRemoteCursors 应用透明度

**文件**: `packages/excalidraw/clients.ts:209-215`

```typescript
const userState = renderConfig.remotePointerUserStates.get(socketId);
const isInactive =
  isOutOfBounds ||
  userState === UserIdleState.IDLE ||    // ⬅️ IDLE 状态命中
  userState === UserIdleState.AWAY;       // ⬅️ AWAY 状态也命中

if (isInactive) {
  context.globalAlpha = 0.3;  // 半透明显示
}
```

### 11.7 完整链路图

```
本地: 60秒无操作
      ↓
Collab.reportIdle()  [Collab.tsx:860]
      ↓
Collab.onIdleStateChange(UserIdleState.IDLE)
      ↓
Portal.broadcastIdleChange(userState)  [Portal.tsx:185]
      ↓
_broadcastSocketData(data, volatile=true)  → WS_EVENTS.SERVER_VOLATILE
      │
      │  ⚠️ 注意：这是 volatile 消息，可能丢包！
      │
      ↓ (WebSocket 传输)
远端: client-broadcast 事件
      ↓
Collab.handleSocketMessage()
  └─ case WS_SUBTYPES.IDLE_STATUS  [Collab.tsx:663]
      ↓
Collab.updateCollaborator(socketId, { userState, username })
      ↓
excalidrawAPI.updateScene({ collaborators })
      ↓
App.setState({ collaborators })
      ↓
React 重渲染 → InteractiveCanvas.memo 比较 collaborators
      ↓
InteractiveCanvas.useEffect() 提取到 remotePointerUserStates
      ↓
AnimationController 驱动下一帧渲染
      ↓
renderRemoteCursors()
  └─ isInactive = (userState === IDLE || AWAY)
      └─ context.globalAlpha = 0.3  ← 光标半透明
```

---

## 十二、Laser 轨迹与普通光标的分离渲染

### 12.1 工具类型区分

**文件**: `excalidraw-app/data/index.ts:95-104`

```typescript
MOUSE_LOCATION: {
  type: WS_SUBTYPES.MOUSE_LOCATION;
  payload: {
    socketId: SocketId;
    pointer: { x: number; y: number; tool: "pointer" | "laser" };  // ⬅️ 关键
    button: "down" | "up";
    selectedElementIds: AppState["selectedElementIds"];
    username: string;
  };
};
```

### 12.2 LaserTrails：SVG 管线（独立于 Canvas）

**文件**: `packages/excalidraw/laser-trails.ts:13-129`

```typescript
export class LaserTrails implements Trail {
  public localTrail: AnimatedTrail;              // 本地激光轨迹
  private collabTrails = new Map<SocketId, AnimatedTrail>();  // 远端激光轨迹
  private container?: SVGSVGElement;              // SVG 容器

  constructor(private animationFrameHandler: AnimationFrameHandler, private app: App) {
    this.animationFrameHandler.register(this, this.onFrame.bind(this));
    this.localTrail = new AnimatedTrail(animationFrameHandler, app, {
      ...this.getTrailOptions(),
      fill: () => DEFAULT_LASER_COLOR,
    });
  }
```

### 12.3 LaserTrails.updateCollabTrails 核心逻辑

**文件**: `packages/excalidraw/laser-trails.ts:80-129`

```typescript
private updateCollabTrails() {
  if (!this.container || this.app.state.collaborators.size === 0) {
    return;
  }

  for (const [key, collaborator] of this.app.state.collaborators.entries()) {
    let trail!: AnimatedTrail;

    if (!this.collabTrails.has(key)) {
      // 首次遇到该用户，创建 SVG 轨迹对象
      trail = new AnimatedTrail(this.animationFrameHandler, this.app, {
        ...this.getTrailOptions(),
        fill: () =>
          collaborator.pointer?.laserColor ||   // 优先使用远端指定颜色
          getClientColor(key, collaborator),    // 否则使用哈希颜色
      });
      trail.start(this.container);
      this.collabTrails.set(key, trail);
    } else {
      trail = this.collabTrails.get(key)!;
    }

    // 只在 laser 工具且按下时添加点
    if (collaborator.pointer && collaborator.pointer.tool === "laser") {
      if (collaborator.button === "down" && !trail.hasCurrentTrail) {
        trail.startPath(collaborator.pointer.x, collaborator.pointer.y);
      }
      if (collaborator.button === "down" && trail.hasCurrentTrail &&
          !trail.hasLastPoint(collaborator.pointer.x, collaborator.pointer.y)) {
        trail.addPointToPath(collaborator.pointer.x, collaborator.pointer.y);
      }
      if (collaborator.button === "up" && trail.hasCurrentTrail) {
        trail.addPointToPath(collaborator.pointer.x, collaborator.pointer.y);
        trail.endPath();
      }
    }
  }

  // 清理已离开用户的轨迹
  for (const key of this.collabTrails.keys()) {
    if (!this.app.state.collaborators.has(key)) {
      const trail = this.collabTrails.get(key)!;
      trail.stop();
      this.collabTrails.delete(key);
    }
  }
}
```

### 12.4 Laser 轨迹衰减参数

**文件**: `packages/excalidraw/laser-trails.ts:31-50`

```typescript
private getTrailOptions() {
  return {
    simplify: 0,
    streamline: 0.4,
    sizeMapping: (c) => {
      const DECAY_TIME = 1000;   // 1秒内轨迹逐渐消失
      const DECAY_LENGTH = 50;   // 轨迹最多保留50个点
      const t = Math.max(0, 1 - (performance.now() - c.pressure) / DECAY_TIME);
      const l = (DECAY_LENGTH - Math.min(DECAY_LENGTH, c.totalLength - c.currentIndex)) / DECAY_LENGTH;
      return Math.min(easeOut(l), easeOut(t));
    },
  } as Partial<LaserPointerOptions>;
}
```

> **关键**: 激光轨迹在 1 秒内会渐隐消失（`DECAY_TIME = 1000ms`），最多保留 50 个点。

### 12.5 InteractiveCanvas 中 renderCursor 过滤

**文件**: `packages/excalidraw/components/canvases/InteractiveCanvas.tsx:110-112`

```typescript
// 过滤：没有 pointer 或 renderCursor=false 则跳过光标渲染
if (!user.pointer || user.pointer.renderCursor === false) {
  return;
}
```

> **分离渲染的关键**: 当 `tool === "laser"` 时，`renderCursor` 被设为 `false`，因此 Canvas 光标不会渲染，只有 SVG LaserTrails 会渲染轨迹。

### 12.6 双管线架构对比

| 特性 | 普通光标 (pointer) | 激光轨迹 (laser) |
|------|---------------------|-------------------|
| **渲染管线** | Canvas 2D (`renderRemoteCursors`) | SVG (`LaserTrails` → `AnimatedTrail`) |
| **驱动方式** | `AnimationController` + `requestAnimationFrame` | `AnimationFrameHandler`（不同的动画控制器！） |
| **坐标系统** | 视口坐标（已转换） | 场景坐标（直接使用） |
| **渲染频率** | 60fps（由 AnimationController 控制） | 60fps（由 AnimationFrameHandler 控制） |
| **可见性** | 始终可见（除非 IDLE/AWAY） | 按下时才绘制，松开后 1 秒内渐隐 |
| **颜色来源** | `getClientColor()` 哈希 | `collaborator.pointer.laserColor` 或 `getClientColor()` |
| **过滤条件** | `renderCursor !== false` | `pointer.tool === "laser"` |

> **容易混淆**: 两者使用**不同的动画控制器**！Cursor 走 `AnimationController`（`animation.ts`），Laser 走 `AnimationFrameHandler`（`animation-frame-handler.ts`）。

### 12.7 渲染来源对比图

```
appState.collaborators (Map<SocketId, Collaborator>)
      │
      ├─────────────────────────────────────────────┐
      ↓                                             ↓
InteractiveCanvas.useEffect()              LaserTrails.onFrame()
  ├─ 遍历 collaborators                      ├─ 遍历 collaborators
  ├─ 过滤: renderCursor === false?           ├─ 过滤: pointer.tool === "laser"?
  ├─ 提取到 5 个渲染 Map                      ├─ button === "down" → 添加路径点
  └─ 组装 renderConfig                        └─ button === "up" → 结束路径
      ↓                                             ↓
renderInteractiveScene()                 AnimatedTrail 绘制 SVG 路径
  └─ renderRemoteCursors()                    └─ 1秒内渐隐消失
      ↓                                             ↓
Canvas 2D 绘制                            SVG 元素渲染
  ├─ 箭头光标（3层叠加）
  ├─ 用户名标签
  └─ 状态指示（按下环/说话框）
```

---

## 十三、Volatile 消息丢包对可见节奏的影响

### 13.1 Volatile vs Non-Volatile 分发

**文件**: `excalidraw-app/collab/Portal.tsx:85-101`

```typescript
async _broadcastSocketData(
  data: SocketUpdateData,
  volatile: boolean = false,
  roomId?: string,
) {
  if (this.isOpen()) {
    const json = JSON.stringify(data);
    const encoded = new TextEncoder().encode(json);
    const { encryptedBuffer, iv } = await encryptData(this.roomKey!, encoded);

    this.socket?.emit(
      volatile ? WS_EVENTS.SERVER_VOLATILE : WS_EVENTS.SERVER,  // ⬅️ 关键分发
      roomId ?? this.roomId,
      encryptedBuffer,
      iv,
    );
  }
}
```

### 13.2 各消息类型的 Volatile 属性

| 消息类型 | Volatile? | 丢包影响 |
|----------|-----------|----------|
| `MOUSE_LOCATION` (指针位置) | ✅ **是** | 光标暂时冻结，下一帧恢复 |
| `IDLE_STATUS` (空闲状态) | ✅ **是** | 状态短暂不一致，下次心跳修复 |
| `USER_VISIBLE_SCENE_BOUNDS` (视口边界) | ✅ **是** | 跟随模式视口短暂不更新 |
| `SCENE_INIT` (场景初始化) | ❌ **否** | 必须送达，否则无法加入协作 |
| `SCENE_UPDATE` (元素更新) | ❌ **否** | 必须送达，否则画板不同步 |

### 13.3 各 volatile 消息丢包的具体影响

#### 13.3.1 MOUSE_LOCATION 丢包

**场景**: 用户 A 移动光标，某一帧位置消息丢失

```
正常节奏:  [pos1] → [pos2] → [pos3] → [pos4] → ...  (每 33ms 一帧)
丢包后:   [pos1] → [丢了] → [pos3] → [pos4] → ...
                                                          ↑
                                              A 的光标在 pos1 停留 33ms
                                              然后跳到 pos3（用户感觉轻微卡顿）
```

**影响程度**: 低。每帧之间独立，丢一帧只会导致光标在旧位置多停留 33ms。

**修复机制**: 下一帧自动修正。

#### 13.3.2 IDLE_STATUS 丢包

**场景**: 用户 A 进入 IDLE 状态，消息丢失

```
正常节奏:  A ACTIVE → [IDLE 消息] → 其他用户看到 A 光标半透明
丢包后:   A ACTIVE → [IDLE 丢了] → 其他用户仍看到 A 光标正常明亮
                                                              ↑
                                              最多延迟 3 秒（下一次 ACTIVE 心跳后
                                              A 重新确认 ACTIVE，但 IDLE 状态丢失）
```

**影响程度**: 中等。其他用户看到 A 的光标状态与实际不符，可能误导协作决策。

**修复机制**: 依赖 `ACTIVE_THRESHOLD = 3000ms` 心跳，但心跳只上报 ACTIVE，IDLE 状态丢失后需要等到 A 再次活动才能恢复。

#### 13.3.3 USER_VISIBLE_SCENE_BOUNDS 丢包

**场景**: 用户 B 正在跟随用户 A，A 滚动视口，边界消息丢失

```
正常节奏:  A 滚动 → [bounds] → B 自动跟随滚动到新位置
丢包后:   A 滚动 → [bounds 丢了] → B 视口停留在旧位置
                                                    ↑
                                          B 看到的内容与 A 不一致
                                          直到下一次 bounds 消息（throttleRAF 限制频率）
```

**影响程度**: 中高。跟随模式下视口可能长时间不一致。

**修复机制**: `throttleRAF`（requestAnimationFrame 节流）会在下一帧重新发送。

### 13.4 为什么这些消息可以是 Volatile

**设计考量**:

1. **MOUSE_LOCATION**: 每帧独立，丢了下一帧自动修正，状态不累积
2. **IDLE_STATUS**: 用户状态有心跳机制兜底，最坏情况是短暂显示错误
3. **USER_VISIBLE_SCENE_BOUNDS**: 跟随模式下视口会持续刷新，丢一帧不致命

**核心原则**:
- **有状态累积的消息**（元素创建/删除）→ **Non-Volatile**，必须可靠送达
- **瞬时状态消息**（位置/状态/视口）→ **Volatile**，允许丢包以换取带宽效率

### 13.5 Volatile 消息对渲染节奏的整体影响

```
渲染帧率 (~60fps)
    ↑
    │  Canvas 每帧都在渲染
    │  但 collaborators.pointer 只有收到新 MOUSE_LOCATION 才更新
    │
    │  ┌──── 33ms 节流窗口 ────┐
    │  │                        │
    │  │ 帧1: pointer 更新 ✓    │  帧2: pointer 未更新
    │  │ 光标移动到新位置        │  光标停留在旧位置
    │  │                        │
    │  └────────────────────────┘
    │
    └────────────────────────────── 时间 →
```

**即使 MOUSE_LOCATION 以 30fps 发送**，Canvas 仍以 60fps 渲染，结果是：
- 光标每 2 帧才移动一次（33ms/帧 × 2 = 66ms 两帧）
- 在两帧之间光标位置不变（视觉上是 30fps 的光标）
- 这是**正常的**，不是丢包

**只有当连续多帧 MOUSE_LOCATION 都丢失**时，才会出现明显卡顿：
- 连续丢 3 帧：光标停留 ~100ms（可感知的卡顿）
- 连续丢 6 帧：光标停留 ~200ms（明显卡顿）

### 13.6 完整的 Volatile 消息处理流程

```
发送端 (Portal.broadcastMouseLocation)
      │
      │  _broadcastSocketData(data, volatile=true)
      │
      ↓
socket.emit("server-volatile-broadcast", roomId, encryptedBuffer, iv)
      │
      │  Socket.IO 的 volatile 标志:
      │  - 若底层传输忙/缓冲区满 → 直接丢弃
      │  - 不保证送达顺序
      │  - 不重传
      │
      ↓ (WebSocket 传输)
服务端: server-volatile-broadcast 事件
      │  转发给房间内其他用户
      │
      ↓
接收端: socket.on("client-broadcast", ...)
      │  解密 → handleSocketMessage()
      │
      ↓
Collab.handleSocketMessage()
  └─ case WS_SUBTYPES.MOUSE_LOCATION
      ↓
updateCollaborator() → excalidrawAPI.updateScene()
      ↓
App.setState() → React 重渲染
      │
      ├─ UserList: 头像状态可能更新
      └─ InteractiveCanvas: useEffect 提取新的 pointer
          ↓
      AnimationController 下一帧:
          renderRemoteCursors() 绘制新位置
```

---

## 十四、总结（完整链路速查表）

| 链路 | 入口 | 出口 | 关键信号 |
|------|------|------|----------|
| 房间成员变更 | `room-user-change` 事件 | `UserList` 头像列表 | `setCollaborators` 全量重建 Map |
| IDLE → 光标透明 | `IDLE_STATUS` volatile 消息 | `context.globalAlpha = 0.3` | `userState` → `isInactive` |
| 指针位置同步 | `MOUSE_LOCATION` volatile 消息 | Canvas 光标位置 | 33ms 节流，~30fps 有效更新 |
| 激光轨迹渲染 | `MOUSE_LOCATION` (tool=laser) | SVG 渐隐轨迹 | `LaserTrails` + `AnimatedTrail` |
| 光标不渲染 laser | `renderCursor = false` | Canvas 跳过该用户 | InteractiveCanvas useEffect 过滤 |
| Volatile 丢包 | 网络层丢弃 | 光标/状态短暂不一致 | 下一帧或心跳自动修复 |
