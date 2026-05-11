# Excalidraw 多人协作实时同步机制分析报告

## 1. 概述

Excalidraw 的多人协作同步采用了 **三层架构** 设计，通过三条独立的链路协同工作，实现了文档状态、光标位置和用户状态的实时同步。这三条链路分别是：

1. **文档状态同步链路** - 负责画布元素的创建、修改、删除同步
2. **光标 Awareness 同步链路** - 负责鼠标/指针位置、选中状态和用户状态的实时感知
3. **房间分发链路** - 负责 WebSocket 连接管理、房间加入/退出和消息广播

## 2. 文档状态同步链路

### 2.1 职责
负责所有画布元素（形状、线条、文本、图片等）的创建、修改、删除和版本同步。

### 2.2 核心流程

#### 2.2.1 本地修改 → 广播
1. **触发点** (`excalidraw-app/App.tsx:678-685`)
   - 当用户在画布上进行任何操作时，`Excalidraw` 组件的 `onChange` 回调被触发
   - 检查 `collabAPI?.isCollaborating()`，如果正在协作则调用 `syncElements(elements)`

2. **同步处理** (`excalidraw-app/collab/Collab.tsx:955-958`)
   - `syncElements` 方法被调用，传入当前所有元素
   - 首先调用 `broadcastElements` 进行增量广播
   - 然后调用 `queueSaveToFirebase` 异步保存到 Firebase

3. **增量广播逻辑** (`excalidraw-app/collab/Collab.tsx:944-953`)
   - 检查当前场景版本 `getSceneVersion(elements)`
   - 如果版本号大于最后一次广播/接收的版本，则进行广播
   - 调用 `portal.broadcastScene(WS_SUBTYPES.UPDATE, elements, false)`

4. **全量定期同步** (`excalidraw-app/collab/Collab.tsx:960-972`)
   - 使用 `throttle` 包装，每 `SYNC_FULL_SCENE_INTERVAL_MS` (20秒) 触发一次
   - 广播所有元素（包括已删除但未超时的元素），确保所有客户端状态一致
   - 更新 `lastBroadcastedOrReceivedSceneVersion`

5. **元素筛选与版本追踪** (`excalidraw-app/collab/Portal.tsx:142-183`)
   - `broadcastScene` 方法过滤需要同步的元素：
     - `syncAll=true`（初始化场景时）：同步所有可同步元素
     - `syncAll=false`（增量更新时）：只同步版本号大于 `broadcastedElementVersions` 的元素
   - 只同步 `isSyncableElement(element)` 为 true 的元素：
     - 非删除元素且不是不可见的微小元素
     - 已删除元素但删除时间不超过 24 小时
   - 更新 `broadcastedElementVersions` Map，记录每个元素的最新版本号

6. **加密与发送** (`excalidraw-app/collab/Portal.tsx:85-102`)
   - `_broadcastSocketData` 方法：
     - 将数据序列化为 JSON → 编码为 Uint8Array
     - 使用 `roomKey` 进行 AES 加密，生成 `encryptedBuffer` 和 `iv`
     - 通过 Socket.IO 发送 `server-broadcast` 事件到指定房间

#### 2.2.2 接收远程更新 → 本地应用
1. **Socket 监听** (`excalidraw-app/collab/Collab.tsx:568-677`)
   - 监听 `client-broadcast` 事件，接收加密数据和 IV
   - 调用 `decryptPayload` 解密：
     - 使用 `roomKey` 解密数据
     - JSON.parse 解析为消息对象

2. **场景初始化消息 (INIT)** (`excalidraw-app/collab/Collab.tsx:584-599`)
   - 当新用户加入房间时，现有用户发送完整场景
   - 调用 `initializeRoom({ fetchScene: false })` 标记房间已初始化
   - 调用 `_reconcileElements` 合并远程元素与本地元素
   - `handleRemoteSceneUpdate` 更新本地场景

3. **场景更新消息 (UPDATE)** (`excalidraw-app/collab/Collab.tsx:601-609`)
   - 调用 `_reconcileElements` 处理远程更新

4. **元素合并逻辑** (`excalidraw-app/collab/Collab.tsx:755-787`)
   - `_reconcileElements` 方法：
     - 先调用 `restoreElements` 恢复远程元素（处理引用、类型等）
     - 调用 `reconcileElements` 进行三方合并（本地版本 vs 远程版本）
     - `bumpElementVersions` 增加版本号防止重复同步
     - **关键**：更新 `lastBroadcastedOrReceivedSceneVersion`，避免将刚收到的更新再次广播出去

5. **应用到画布** (`excalidraw-app/collab/Collab.tsx:804-813`)
   - `handleRemoteSceneUpdate` 调用 `excalidrawAPI.updateScene`
   - `captureUpdate: CaptureUpdateAction.NEVER` 表示这是远程更新，不记录到本地历史栈

### 2.3 Firebase 持久化
- **定时保存** (`excalidraw-app/collab/Collab.tsx:974-986`)
  - `queueSaveToFirebase` 每 20 秒保存一次到 Firebase Firestore
  - 保存时检查 `portal.socketInitialized`
  - 调用 `saveCollabRoomToFirebase(getSyncableElements(...))`

- **保存逻辑** (`excalidraw-app/collab/Collab.tsx:315-355`)
  - `saveToFirebase` 使用 `runTransaction` 确保原子性
  - 只有当本地版本号大于 Firebase 中存储的版本号时才更新
  - 数据加密后存储为 `ciphertext` + `iv`

- **加载逻辑** (`excalidraw-app/collab/Collab.tsx:727-741`)
  - 加入房间时尝试从 Firebase 加载
  - 如果失败，依靠其他客户端通过 INIT 消息同步

## 3. 光标 Awareness 同步链路

### 3.1 职责
负责用户指针（鼠标/触摸）位置、按钮状态、选中元素、用户活跃状态和视口边界的实时同步，实现"看到对方在做什么"的协作体验。

### 3.2 核心流程

#### 3.2.1 指针位置同步
1. **本地触发** (`packages/excalidraw/components/App.tsx:12919-12929`)
   - 当用户移动鼠标/触摸时，`App` 组件内部触发指针更新
   - 构建 `pointer` 对象：`{ x: sceneX, y: sceneY, tool: "pointer" | "laser" }`
   - 调用 `this.props.onPointerUpdate?.(...)` 回调

2. **节流处理** (`excalidraw-app/collab/Collab.tsx:914-925`)
   - `onPointerUpdate` 使用 `throttle` 包装，节流时间 `CURSOR_SYNC_TIMEOUT = 33ms`（约 30fps）
   - 检查手势指针数：`payload.pointersMap.size < 2`（多指触摸时不同步，避免冲突）
   - 调用 `portal.broadcastMouseLocation(payload)`

3. **广播** (`excalidraw-app/collab/Portal.tsx:202-224`)
   - 构建 `MOUSE_LOCATION` 类型消息，包含：
     - `socketId`: 当前用户的 socket 标识
     - `pointer`: 指针位置和工具类型
     - `button`: "up" 或 "down"
     - `selectedElementIds`: 当前选中的元素 ID
     - `username`: 用户名
   - 使用 `volatile: true` 发送（`server-volatile-broadcast` 事件）
     - Volatile 消息不保证送达，适合高频、可丢失的指针位置更新

4. **接收与显示** (`excalidraw-app/collab/Collab.tsx:610-627`)
   - 监听 `client-broadcast` 事件中的 `MOUSE_LOCATION` 子类型
   - 调用 `updateCollaborator(socketId, { pointer, button, selectedElementIds, username })`
   - 更新本地 `collaborators` Map

5. **渲染远程光标** (`packages/excalidraw/components/canvases/InteractiveCanvas.tsx:96-136`)
   - 每次渲染时遍历 `appState.collaborators`
   - 转换坐标：`sceneCoordsToViewportCoords` 将场景坐标转换为视口坐标
   - 收集以下数据传递给渲染器：
     - `remotePointerViewportCoords`: 各用户指针位置
     - `remotePointerButton`: 按钮状态（控制光标样式）
     - `remotePointerUsernames`: 用户名标签
     - `remotePointerUserStates`: 用户状态（活跃/空闲/离开）
     - `remoteSelectedElementIds`: 选中的元素（用于高亮显示）

#### 3.2.2 空闲状态同步
1. **状态检测** (`excalidraw-app/collab/Collab.tsx:815-862`)
   - `initializeIdleDetector` 监听 `pointermove` 和 `visibilitychange` 事件
   - `onPointerMove`: 清除空闲超时，设置活跃定时器
   - `onVisibilityChange`: 页面隐藏时标记为 AWAY，显示时恢复

2. **状态定义** (`packages/common/src/constants.ts:495-499, 307-310`)
   - `ACTIVE`: 用户活跃（指针移动后每 3 秒 `ACTIVE_THRESHOLD` 报告一次）
   - `IDLE`: 用户空闲（60 秒 `IDLE_THRESHOLD` 无操作后报告为空闲）
   - `AWAY`: 用户离开（页面不可见时立即报告）

3. **广播** (`excalidraw-app/collab/Collab.tsx:940-942`)
   - `onIdleStateChange(userState)` 调用 `portal.broadcastIdleChange(userState)`
   - 消息类型 `IDLE_STATUS`，同样使用 volatile 发送

4. **接收** (`excalidraw-app/collab/Collab.tsx:663-670`)
   - 调用 `updateCollaborator(socketId, { userState, username })`

#### 3.2.3 视口边界同步（跟随模式）
1. **触发时机** (`excalidraw-app/collab/Collab.tsx:217-222, 927-938`)
   - 滚动/缩放变化时，`onScrollChange` 触发 `relayVisibleSceneBounds`
   - 只有当 `appState.followedBy.size > 0`（有人在跟随我）时才广播
   - `WS_EVENTS.USER_FOLLOW_ROOM_CHANGE` 事件触发时强制广播

2. **广播** (`excalidraw-app/collab/Portal.tsx:226-248`)
   - 构建 `USER_VISIBLE_SCENE_BOUNDS` 消息
   - 发送到特定房间：`follow@${socketId}`（只有跟随者所在的子房间）

3. **接收与跟随** (`excalidraw-app/collab/Collab.tsx:629-661`)
   - 检查是否正在跟随该用户：`appState.userToFollow?.socketId === socketId`
   - 排除互跟情况：`followedBy.has(userToFollow.socketId)`
   - 调用 `zoomToFitBounds` 自动调整视口，跟随对方的视野

## 4. 房间分发链路

### 4.1 职责
负责：
- WebSocket 连接管理（建立、断开、重连）
- 房间加入/退出和用户列表管理
- 消息的加密/解密和路由分发
- 新用户加入时的场景初始化

### 4.2 核心流程

#### 4.2.1 启动协作
1. **创建/加入房间** (`excalidraw-app/collab/Collab.tsx:471-506`)
   - `startCollaboration(existingRoomLinkData)`：
     - 已有链接：从 URL hash 解析 `roomId` 和 `roomKey`
     - 新房间：调用 `generateCollaborationLinkData()` 生成随机的 `roomId`（10字节）和 `roomKey`（22字符的 Base64URL 编码字符串，对应 `ENCRYPTION_KEY_BITS = 128` 位的 AES-128-GCM 密钥）
   - 新房间会 `pushState` 更新 URL，链接格式：`#room={roomId},{roomKey}`
   - **端到端加密**：`roomKey` 只存在于 URL hash 中，不会发送到服务器

> **参数说明**：加密强度（128位）和空闲阈值（60秒）的选择直接影响协作体验。128位密钥在安全与性能间取得平衡，加密/解密操作不会成为实时同步的瓶颈；60秒空闲阈值既能及时反映用户活跃度变化，又避免了过于频繁的状态切换带来的网络开销和渲染闪烁。

2. **建立 Socket 连接** (`excalidraw-app/collab/Collab.tsx:508-536`)
   - 动态导入 `socket.io-client`
   - 连接到 `VITE_APP_WS_SERVER_URL`
   - 传输方式：`["websocket", "polling"]`

3. **初始化 Portal** (`excalidraw-app/collab/Collab.tsx:523-529`)
   - `portal.open(socket, roomId, roomKey)`

4. **房间加入流程** (`excalidraw-app/collab/Portal.tsx:37-61`)
   - 监听 `init-room` 事件：服务器确认连接后发送
   - 收到后 `emit("join-room", roomId)` 正式加入房间
   - 监听 `new-user` 事件：有新用户加入时
     - 立即 `broadcastScene(WS_SUBTYPES.INIT, allElements, true)` 发送完整场景
   - 监听 `room-user-change` 事件：用户列表变化时
     - 调用 `collab.setCollaborators(clients)` 更新本地用户列表

5. **房间初始化策略** (`excalidraw-app/collab/Collab.tsx:560-565, 679-688`)
   - `first-in-room` 事件：如果是房间第一个人
     - 从 Firebase 加载场景
   - 回退定时器：`INITIAL_SCENE_UPDATE_TIMEOUT = 5000ms`
     - 如果 5 秒内没收到 INIT 消息，主动从 Firebase 加载
   - 双重保障：WebSocket 实时同步 + Firebase 持久化

#### 4.2.2 消息广播与接收
1. **发送消息** (`excalidraw-app/collab/Portal.tsx:85-102`)
   - `_broadcastSocketData(data, volatile, roomId?)`
   - 两种发送模式：
     - **可靠消息** (`volatile: false`)：`server-broadcast` 事件
       - 用于：`SCENE_INIT`、`SCENE_UPDATE`
       - Socket.IO 保证送达和顺序
     - **易失消息** (`volatile: true`)：`server-volatile-broadcast` 事件
       - 用于：`MOUSE_LOCATION`、`IDLE_STATUS`、`USER_VISIBLE_SCENE_BOUNDS`
       - 不保证送达，但延迟更低

2. **接收消息** (`excalidraw-app/collab/Collab.tsx:568-677`)
   - 统一监听 `client-broadcast` 事件
   - 所有消息都包含 `type` 字段，通过 `switch (decryptedData.type)` 分发：
     - `SCENE_INIT` → 初始化场景
     - `SCENE_UPDATE` → 增量更新
     - `MOUSE_LOCATION` → 更新协作者指针
     - `USER_VISIBLE_SCENE_BOUNDS` → 跟随模式视口同步
     - `IDLE_STATUS` → 用户状态更新

3. **协作 API 暴露** (`excalidraw-app/collab/Collab.tsx:230-243`)
   - 通过 Jotai atom `collabAPIAtom` 暴露给应用其他部分
   - 暴露的方法：
     - `isCollaborating()`: 检查是否在协作中
     - `onPointerUpdate`: 指针更新回调
     - `startCollaboration` / `stopCollaboration`: 协作生命周期
     - `syncElements`: 元素同步
     - `fetchImageFilesFromFirebase`: 图片文件加载
     - `setUsername` / `getUsername`: 用户名管理
     - `getActiveRoomLink`: 获取当前房间链接
     - `setCollabError`: 错误提示

## 5. 三条链路的衔接与协作

### 5.1 架构关系图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        协作系统整体架构                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    excalidraw-app/App.tsx                      │  │
│  │                                                               │  │
│  │  onChange (元素变化) ──────► collabAPI.syncElements()          │  │
│  │  onPointerUpdate (指针) ───► collabAPI.onPointerUpdate()       │  │
│  │                                                               │  │
│  └────────────────────────┬──────────────────────────────────────┘  │
│                           │                                          │
│                           ▼                                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              excalidraw-app/collab/Collab.tsx                  │  │
│  │                                                               │  │
│  │  文档状态同步              │              光标 Awareness        │  │
│  │  ──────────────              │              ────────────       │  │
│  │  • syncElements()            │              • onPointerUpdate()│  │
│  │  • broadcastElements()       │              • 节流 33ms        │  │
│  │  • 每20s全量同步             │              • 空闲检测          │  │
│  │  • Firebase 持久化           │              • 视口边界          │  │
│  │                                                               │  │
│  └────────────────────────┬──────────────────────────────────────┘  │
│                           │                                          │
│                           ▼                                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              excalidraw-app/collab/Portal.tsx                  │  │
│  │                                                               │  │
│  │                    房间分发链路                                 │  │
│  │                    ────────────                                │  │
│  │  • Socket.IO 连接管理                                          │  │
│  │  • 房间加入/退出 (join-room)                                   │  │
│  │  • 新用户初始化 (new-user → SCENE_INIT)                        │  │
│  │  • 消息加密/解密 (AES-GCM)                                     │  │
│  │  • 可靠 vs 易失消息路由                                        │  │
│  │                                                               │  │
│  └────────────────────────┬──────────────────────────────────────┘  │
│                           │                                          │
│                           ▼                                          │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      WebSocket 服务器                           │  │
│  │                                                               │  │
│  │  • 房间管理 (room-user-change)                                │  │
│  │  • 消息广播 (client-broadcast)                                │  │
│  │  • first-in-room 通知                                         │  │
│  │                                                               │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 数据流总结

| 链路 | 触发源 | 传输类型 | 加密 | 保证送达 | 频率 |
|------|--------|----------|------|----------|------|
| 文档状态同步 | `onChange` 回调 | `SCENE_UPDATE` / `SCENE_INIT` | AES-GCM | ✅ 可靠 | 实时 + 20s全量 |
| 光标位置 | `onPointerUpdate` 回调 | `MOUSE_LOCATION` | AES-GCM | ❌ 易失 | ~30fps (节流33ms) |
| 用户状态 | 指针移动/页面可见性 | `IDLE_STATUS` | AES-GCM | ❌ 易失 | 状态变化时 |
| 视口边界 | 滚动/缩放变化 | `USER_VISIBLE_SCENE_BOUNDS` | AES-GCM | ❌ 易失 | 视口变化时 |

### 5.3 关键衔接点

1. **版本号控制** (`lastBroadcastedOrReceivedSceneVersion`)
   - 文档状态链路中，接收远程更新后立即更新版本号
   - 防止"收到的更新被再次广播出去"的循环问题

2. **初始化优先级**
   - 新用户加入时：优先接收其他用户的 `SCENE_INIT`
   - 超时回退：从 Firebase 加载
   - 双重保障确保场景一致性

3. **加密边界**
   - `roomKey` 只存在于客户端（URL hash）
   - Portal 层负责所有消息的加密/解密
   - 服务器无法看到明文内容

4. **消息分类策略**
   - 状态修改（元素）：可靠传输，保证一致性
   - 感知数据（光标、状态）：易失传输，追求低延迟

## 6. 关键文件索引

| 文件路径 | 主要职责 |
|----------|----------|
| `excalidraw-app/collab/Collab.tsx` | 协作状态管理、元素同步、指针同步、空闲检测 |
| `excalidraw-app/collab/Portal.tsx` | Socket 连接管理、房间操作、消息加密/广播 |
| `excalidraw-app/App.tsx` | 将 Excalidraw 与协作系统集成 |
| `excalidraw-app/data/firebase.ts` | Firebase 场景持久化、事务保存/加载 |
| `excalidraw-app/data/index.ts` | 房间链接生成/解析、数据类型定义 |
| `excalidraw-app/app_constants.ts` | 时间常量、事件类型定义 |
| `packages/excalidraw/components/App.tsx` | 画布事件处理、指针更新触发 |
| `packages/excalidraw/components/canvases/InteractiveCanvas.tsx` | 远程光标渲染 |

## 7. 核心技术点

1. **端到端加密**：所有消息使用 AES-GCM 加密，密钥只在客户端通过 URL hash 分享
2. **增量 + 全量同步**：实时增量更新 + 定期全量同步（20秒），兼顾效率和一致性
3. **版本向量**：每个元素有 `version` 字段，`reconcileElements` 基于版本号进行冲突解决
4. **双重持久化**：WebSocket 实时同步 + Firebase 持久化存储，断网重连也能恢复
5. **消息分类**：状态数据用可靠传输，感知数据用易失传输，平衡一致性和延迟
6. **协作 API 抽象**：通过 Jotai atom 暴露协作能力，与 UI 层解耦
