# 画板指针手势与拖拽状态机协同关系分析

## 1. 核心架构概述

Excalidraw 的指针手势系统采用**事件驱动的状态机**设计，通过三个核心阶段（按下-移动-抬起）驱动整个交互流程。系统的核心组件包括：

| 组件 | 职责 | 核心文件 |
|------|------|----------|
| 指针事件分发器 | 接收 DOM 事件，分发给对应处理器 | `App.tsx:7688-8146` |
| PointerDownState | 单次交互的状态快照 | `types.ts:853-904` |
| 手势计算器 | 多指手势、缩放计算 | `gesture.ts:1-15` |
| 拖拽引擎 | 元素拖拽、新建、对齐逻辑 | `dragElements.ts:35-353` |
| 历史记录管理器 | 撤销/重做栈管理 | `history.ts:90-240` |
| Store 增量捕获器 | 快照-增量计算、撤销入栈 | `element/src/store.ts:78-427` |
| 协作同步器 | 指针位置广播 | `excalidraw-app/collab/Collab.tsx:914-925` |
| 清理补偿器 | pointerup 丢失时的兜底处理 | `App.tsx:8246-8249` |

---

## 2. 核心数据结构：PointerDownState

### 2.1 状态快照定义 (`types.ts:853-904`)

```typescript
type PointerDownState = {
  origin: { x: number; y: number };           // 按下时的场景坐标
  originInGrid: { x: number; y: number };     // 对齐网格后的坐标
  lastCoords: { x: number; y: number };       // 上一次指针位置
  originalElements: Map<string, Element>;     // 所有元素的冻结快照
  
  resize: {
    handleType: TransformHandleType | false;  // 当前拖拽的变换手柄
    isResizing: boolean;                      // 是否处于缩放状态
    offset: { x: number; y: number };         // 缩放偏移量
    arrowDirection: "origin" | "end";         // 箭头拖拽方向
    center: { x: number; y: number };         // 旋转中心点
  };
  
  hit: {
    element: Element | null;                  // 点击命中的元素
    allHitElements: Element[];                // 所有命中元素（层级）
    wasAddedToSelection: boolean;             // 是否已加入选择集
    hasBeenDuplicated: boolean;               // 是否已复制（Alt拖拽）
    hasHitCommonBoundingBoxOfSelectedElements: boolean;
  };
  
  drag: {
    hasOccurred: boolean;                     // 是否已发生拖拽
    offset: { x: number; y: number } | null;  // 拖拽偏移
    origin: { x: number; y: number };         // 拖拽起始点
    blockDragging: boolean;                   // 阻止拖拽（如套索后）
  };
  
  boxSelection: {
    hasOccurred: boolean;                     // 是否发生框选
  };
  
  eventListeners: {
    onMove: Function | null;
    onUp: Function | null;
    onKeyDown: Function | null;
    onKeyUp: Function | null;
  };
};
```

### 2.2 状态机设计原则

1. **快照不变性**：`originalElements` 在 `pointerDown` 时深拷贝所有元素，确保拖拽过程中可随时回滚
2. **单一流转**：每次 `pointerDown` 创建新的 `PointerDownState`，生命周期严格绑定到 `pointerUp`
3. **副作用隔离**：拖拽过程中的中间状态通过 `pointerDownState` 传递，不直接污染全局 `appState`

---

## 3. 三阶段事件处理流程

### 3.1 第一阶段：PointerDown (`App.tsx:7688-8146`)

**执行流程：**

```
事件捕获
    ↓
7701: viewportCoordsToSceneCoords() - 视口坐标转场景坐标
    ↓
7711: setPointerCapture() - 捕获后续指针事件到 canvas
    ↓
7715: maybeCleanupAfterMissingPointerUp() - 清理上一次可能丢失的 pointerUp
    ↓
7753: updateGestureOnPointerDown() - 更新手势状态（多指检测）
    ↓
7838: setState({ cursorButton: "down" }) - 标记按钮按下
    ↓
7842: savePointer() - 广播指针位置给协作端
    ↓
7903: 多指过滤 → if (gesture.pointers.size > 1) return
    ↓
7909: initialPointerDownState() - 创建状态快照
    ↓
工具分支处理：
  ├─ selection 工具 → handleSelectionOnPointerDown()
  ├─ lasso 工具     → 套索选择逻辑
  ├─ text 工具      → handleTextOnPointerDown()
  ├─ arrow/line 工具 → handleLinearElementOnPointerDown()
  ├─ freedraw 工具  → handleFreeDrawElementOnPointerDown()
  ├─ frame 工具     → createFrameElementOnPointerDown()
  ├─ laser 工具     → 激光轨迹 startPath()
  └─ 其他绘图工具   → createGenericElementOnPointerDown()
    ↓
8123-8127: 创建动态事件处理器
  onPointerMove = onPointerMoveFromPointerDownHandler(state)
  onPointerUp = onPointerUpFromPointerDownHandler(state)
    ↓
8132-8134: 注册补偿清理监听
  this.missingPointerEventCleanupEmitter.once((_event) =>
    onPointerUp(_event || event.nativeEvent),
  );
    ↓
8137-8140: 绑定 window 级事件监听器
```

**关键点：**
- **事件捕获**：使用 `setPointerCapture` 确保后续事件都由 canvas 处理，即使指针移出画布
- **多指过滤**：`gesture.pointers.size > 1` 时阻止单选/绘制操作（`App.tsx:7903`）
- **橡皮笔切换**：检测到橡皮笔按钮时自动切换工具，抬起后还原
- **自由绘制多指特殊处理** (`App.tsx:7760-7793`)：
  - 触屏且当前正在绘制 freedraw 时，第二指按下会触发特殊逻辑
  - 点数 < 10：删除元素（认为是误触尖峰），`captureUpdate: NEVER`
  - 点数 >= 10：保留元素，重置 newElement 等待第二指抬起后 finalize

### 3.2 第二阶段：PointerMove

**⚠️ 重要修正：三层 pointerMove 处理链路，两层节流**

Excalidraw 有**三条独立的 pointerMove 处理链路**，分布在两个层级：

| 层级 | 处理器 | 代码位置 | 节流方式 | 频率 | 职责 |
|------|--------|---------|---------|------|------|
| 核心包 | `handleCanvasPointerMove` | `App.tsx:6888` | **无节流**（React 原生事件） | 60-120Hz（浏览器决定） | 多指手势追踪 + 协作指针同步入口 |
| 核心包 | `onPointerMoveFromPointerDownHandler` | `App.tsx:9675` | `withBatchedUpdatesThrottled` | **~60fps**（16.6ms） | 元素拖拽/绘制逻辑 |
| 协作层 | `Collab.onPointerUpdate` | `Collab.tsx:914` | `lodash.throttle` | **~30fps**（33ms） | 协作端指针广播 |

**第一层：handleCanvasPointerMove (无节流)**

```typescript
// App.tsx:6888-6905
private handleCanvasPointerMove = (
  event: React.PointerEvent<HTMLCanvasElement>,
) => {
  // 无节流！浏览器 pointermove 原生频率 60-120Hz
  this.savePointer(event.clientX, event.clientY, this.state.cursorButton);
  this.lastPointerMoveEvent = event.nativeEvent;
  const scenePointer = viewportCoordsToSceneCoords(event, this.state);
  
  // 更新 gesture.pointers 中该指针的坐标
  if (gesture.pointers.has(event.pointerId)) {
    gesture.pointers.set(event.pointerId, {
      x: event.clientX,
      y: event.clientY,
    });
  }
  
  // 双指缩放检测 (gesture.pointers.size === 2)
  // ...
};
```

**第二层：onPointerMoveFromPointerDownHandler (throttled @60fps)**

```typescript
// App.tsx:9675-9678
private onPointerMoveFromPointerDownHandler(pointerDownState) {
  return withBatchedUpdatesThrottled((event) => {
    // throttleRAF + batchedUpdates，限制到 ~60fps
    // 返回的函数带有 flush() 方法
  });
}
```

**节流实现 (`common/src/utils.ts:155-195` + `reactUtils.ts:23-32`)**

```typescript
export const throttleRAF = (fn) => {
  // requestAnimationFrame 节流，约 16.6ms 一次
  // 同一帧内多次调用只执行最后一次
  // 返回值带有 flush() 方法可强制执行
};

export const withBatchedUpdatesThrottled = (func) => {
  return throttleRAF((event) => {
    unstable_batchedUpdates(func, event);
  });
};
```

**第三层：Collab.onPointerUpdate (throttled @30fps)** - 协作层

```typescript
// excalidraw-app/app_constants.ts:8
export const CURSOR_SYNC_TIMEOUT = 33; // ~30fps

// excalidraw-app/collab/Collab.tsx:914-925
import throttle from "lodash.throttle";

onPointerUpdate = throttle(
  (payload) => {
    // ⚠️ 真实多指过滤条件：单指才广播
    payload.pointersMap.size < 2 &&
      this.portal.socket &&
      this.portal.broadcastMouseLocation(payload);
  },
  CURSOR_SYNC_TIMEOUT,  // 33ms → ~30fps
);
```

**完整协作端同步链路：**

```
浏览器 pointermove (60-120Hz)
    ↓
App.tsx: handleCanvasPointerMove (无节流)
    ↓
App.tsx: this.savePointer(x, y, button)
    ├─ 坐标转换 viewport → scene
    └─ this.props.onPointerUpdate?.({
        pointer, button, pointersMap: gesture.pointers
       })
    ↓
excalidraw-app/App.tsx:916 → collabAPI?.onPointerUpdate
    ↓
Collab.tsx: lodash.throttle(fn, 33ms) → ~30fps
    ↓
真实过滤: payload.pointersMap.size < 2 ?
    ├─ 是 → Portal.broadcastMouseLocation() → WebSocket volatile 消息
    └─ 否 → 丢弃，不广播
```

**执行流程总览：**

```
浏览器 pointermove 事件（60-120Hz）
    │
    ├─→ handleCanvasPointerMove（无节流）
    │       ├─ savePointer() → 协作同步入口（60-120Hz）
    │       ├─ 更新 gesture.pointers 坐标
    │       └─ 双指缩放计算
    │
    └─→ onPointerMoveFromPointerDownHandler（throttled @60fps）
            ├─ 坐标转换 → 场景坐标
            ├─ 懒初始化 drag.offset
            └─ 工具分支处理：
                ├─ 橡皮擦 → handleEraser()
                ├─ 激光笔 → laserTrails.addPointToPath()
                ├─ 缩放状态 → maybeHandleResize()
                ├─ 裁剪状态 → maybeHandleCrop()
                ├─ 线性元素编辑 → LinearElementEditor.handlePointDragging()
                └─ 普通拖拽 → dragSelectedElements()
```

**拖拽核心逻辑 (`dragElements.ts:35-171`)**

```typescript
export const dragSelectedElements = (
  pointerDownState, selectedElements, offset, scene, snapOffset, gridSize
) => {
  // 1. 特殊处理：两端绑定的肘形箭头单独移动时不跟随
  // 2. 处理框架与内部元素的关系
  // 3. 从 originalElements 获取原始位置
  // 4. 计算网格对齐偏移
  // 5. 更新每个元素的 x, y 坐标
  // 6. 处理箭头绑定关系（未同时选中绑定目标时解绑）
};
```

**关键状态标志：**
- `pointerDownState.drag.hasOccurred` - 首次移动超过阈值时设为 `true`
- `selectedElementsAreBeingDragged` - 全局拖拽状态，影响渲染和撤销

### 3.3 第三阶段：PointerUp (`App.tsx:10598-10900`)

**执行流程：**

```
10604: removePointer() - 移除手势追踪
    ↓
10606-10607: pointerDownState.eventListeners.onMove.flush()
    ↓ （⚠️ 强制刷新节流中的 move 事件，确保拖拽状态不丢失）
10620: 重置临时状态
  { isResizing, isRotating, isCropping, resizingElement, 
    selectionElement, snapLines, cursorButton: "up" }
    ↓
10634: lassoTrail.endPath() - 结束套索轨迹
    ↓
10640: savePointer() - 广播抬起状态
    ↓
10811: missingPointerEventCleanupEmitter.clear() - 正常结束，清空补偿监听
    ↓
10813-10828: 移除 window 事件监听器
    ↓
元素分支处理：
  ├─ freedraw → 追加终点 → actionFinalize
  ├─ 线性元素 → 见下文 4.2 节详细分支
  ├─ 文本元素 → handleTextWysiwyg() 进入编辑模式
  ├─ 其他新元素 →
        10991: isInvisiblySmallElement() → 删除（captureUpdate: NEVER）
        否则 → actionFinalize
  └─ frame → getElementsInNewFrame() 收集内部元素
    ↓
10830: onPointerUpEmitter.trigger() - 通知外部
```

---

## 4. 不同工具下的状态流转（修正版）

### 4.1 Selection 工具状态机

```
pointerDown
    ↓
initialPointerDownState()
    ↜─ 检测 transformHandle → resize.isResizing = true
    ↜─ 命中元素 → hit.element = element
    ↔ 按住 Shift → 添加到选择集
    ↔ 按住 Cmd/Ctrl → 切换选中状态
    ↓
pointerMove
    ├─ resize.isResizing = true → 缩放/旋转
    ├─ 拖拽已选中元素 → dragSelectedElements()
    ├─ 拖拽空白区域 → 创建 selectionElement（框选）
    └─ 按住 Alt → 复制元素并拖拽
    ↓
pointerUp
    ├─ 已拖拽 → store.scheduleCapture() → 入撤销栈
    ├─ 框选 → 更新选择集
    └─ 单击 → 单选/取消
```

### 4.2 Linear 工具状态机（Arrow/Line）- 重要修正

**⚠️ 关键修正：线性元素小拖拽不会删除，而是进入多点编辑模式**

```
pointerDown
    ↓
handleLinearElementOnPointerDown()
    ├─ 命中已有箭头端点 → 进入编辑模式
    └─ 空白处 → 创建 multiElement（多点绘制模式）
    ↓
pointerMove
    ├─ 编辑模式 → handlePointDragging()
    │   ├─ 拖拽端点 → 移动箭头端点
    │   ├─ 检测绑定目标 → suggestedBinding
    │   └─ 实时更新箭头绑定
    └─ 新建模式 → dragNewElement() 实时更新箭头
    ↓
pointerUp (`App.tsx:10867-10963`)
    │
    ├─ 【提前入栈检查】points.length > 1 且第二点非原点 → scheduleCapture()
    │
    ├─ 【分支1：小拖拽】!drag.hasOccurred || dragDistance < MINIMUM_ARROW_SIZE
    │       │
    │       ├─ 触屏 (isTouchScreen)：
    │       │   ├─ 设置 FIXED_DELTA_X = min(width*0.7/zoom, 100)
    │       │   ├─ mutate: x -= FIXED_DELTA_X/2, points = [[0,0], [FIXED_DELTA_X,0]]
    │       │   └─ actionFinalize() → 直接完成
    │       │
    │       └─ 非触屏：
    │           ├─ mutate: points = [p0, [pointerCoords.x - x, pointerCoords.y - y]]
    │           └─ setState({ multiElement: newElement, newElement }) → 进入多点编辑
    │
    └─ 【分支2：正常拖拽】drag.hasOccurred && !multiElement
            ├─ actionFinalize() → 完成绘制
            ├─ 非 locked 工具：
            │   ├─ 切换回 selection 工具
            │   ├─ selectedElementIds 加入新元素
            │   └─ selectedLinearElement = new LinearElementEditor(...)
            └─ locked 工具：仅 newElement = null
```

**线性元素与其他元素的关键区别：**

| 行为 | 线性元素 (arrow/line) | 其他元素 (rect/ellipse 等) |
|------|----------------------|---------------------------|
| 小拖拽处理 | 进入多点编辑或固定长度 | 检测 `isInvisiblySmallElement()` 并删除 |
| 入栈时机 | `points.length > 1` 时提前 `scheduleCapture()` | `actionFinalize` 时入栈 |
| 删除条件 | 永不自动删除（除非显式删除） | `width<1 && height<1` 时删除 |
| 删除时 captureUpdate | N/A | `CaptureUpdateAction.NEVER`（避免入撤销栈） |

### 4.3 绘图工具状态机（Rectangle/Ellipse/Diamond 等）

```
pointerDown
    ↓
createGenericElementOnPointerDown()
    ├─ newElement = 新建元素（width=0, height=0）
    └─ setState({ newElement })
    ↓
pointerMove（throttled @60fps）
    ↓
dragNewElement()
    ├─ 根据拖拽方向计算 width/height
    ├─ 按住 Shift → 等比例缩放
    ├─ 按住 Alt → 从中心向外绘制
    └─ scene.mutateElement() 更新元素
    ↓
pointerUp
    ↓
【小元素检测】isInvisiblySmallElement(newElement) →
    ├─ updateScene 过滤删除该元素
    └─ captureUpdate: NEVER（不进入撤销栈）
    ↓
actionFinalize()
    ├─ newElement = null
    ├─ 切换回 selection 工具（除非 locked）
    └─ captureUpdate: IMMEDIATELY → 入撤销栈
```

### 4.4 Hand 工具/视图模式状态机

```
pointerDown
    ↓
handleCanvasPanUsingWheelOrSpaceDrag()
    ├─ 多指检查：gesture.pointers.size <= 1 → 否则不进入平移
    ├─ isPanning = true
    ├─ setCursor(GRABBING)
    └─ 绑定 onPointerMove/onPointerUp
    ↓
pointerMove (withBatchedUpdatesThrottled @60fps)
    ↓
translateCanvas()
    ├─ scrollX -= deltaX / zoom
    └─ scrollY -= deltaY / zoom
    ↓
pointerUp / lastPointerUp()
    ↓
isPanning = false, 恢复光标
```

---

## 5. 多指场景下的过滤与节流规则（最终修正版）

### 5.1 Gesture 对象定义 (`App.tsx:614-619` + `types.ts:512-517`)

```typescript
// 模块级单例，整个 App 共享
const gesture: Gesture = {
  pointers: new Map<number, PointerCoords>(),  // pointerId → {x, y}
  lastCenter: { x: number; y: number } | null,
  initialDistance: number | null,
  initialScale: number | null,
};
```

### 5.2 多指手势过滤规则 - 真实代码条件

| 检测点 | 代码位置 | 真实条件 | 行为 |
|--------|----------|---------|------|
| 选择阻止 | `App.tsx:7903` | `gesture.pointers.size > 1` | pointerDown 直接 return，不进入选择/绘制 |
| 平移阻止 | `App.tsx:8257` | `gesture.pointers.size > 1` | Hand 工具不进入平移模式 |
| 双指缩放触发 | `App.tsx:6909` | `gesture.pointers.size === 2` && `gesture.lastCenter` && `initialScale` && `gesture.initialDistance` | 进入双指缩放平移模式 |
| 缩放初始化 | `App.tsx:8375` | `gesture.pointers.size === 2` | 记录 `initialScale`、`initialDistance`、`lastCenter` |
| 点击阻止 | `App.tsx:1373` | `gesture.pointers.size < 2` | 阻止 iframe 点击交互（<2 才允许） |
| 触屏检测 | `App.tsx:5632` | `gesture.pointers.size >= 2` | 判定为触屏设备，阻止取消选择 |
| 自由绘制多指 | `App.tsx:7760-7793` | `event.pointerType === "touch"` && `newElement.type === "freedraw"` | 短尖峰删除，长轨迹保留 |
| freedraw 缩放禁用 | `App.tsx:6921-6923` | `activeTool.type === "freedraw"` && `penMode` | `scaleFactor = 1`，禁用双指缩放 |
| 空格键切换光标 | `App.tsx:5250` | `gesture.pointers.size === 0` | 按下空格键时才允许切换到抓取光标 |
| **协作端广播阻止** | **`Collab.tsx:920`** | **`payload.pointersMap.size >= 2`** | **多指时不广播指针位置给协作端** |

### 5.3 协作端指针同步 - 真实过滤条件与节流频率

**⚠️ 最终事实：协作端同步是**两层节流 + 多指过滤**的组合**

```typescript
// 节流常量定义
// excalidraw-app/app_constants.ts:8
export const CURSOR_SYNC_TIMEOUT = 33; // ~30fps (1000/33 ≈ 30.3)

// 协作层节流与过滤
// excalidraw-app/collab/Collab.tsx:29, 914-925
import throttle from "lodash.throttle";  // lodash 默认 leading:true, trailing:true

onPointerUpdate = throttle(
  (payload) => {
    // ⚠️ 真实多指过滤条件：严格小于 2 才广播
    payload.pointersMap.size < 2 &&
      this.portal.socket &&
      this.portal.broadcastMouseLocation(payload);
  },
  CURSOR_SYNC_TIMEOUT,  // 33ms
);
```

**三层节流频率对比表：**

| 节流点 | 实现方式 | 常量值 | 真实频率 | 影响范围 |
|--------|----------|--------|---------|----------|
| handleCanvasPointerMove | 无节流（React 原生） | N/A | 60-120Hz | 多指追踪、协作同步入口 |
| onPointerMoveFromPointerDownHandler | `throttleRAF` + `batchedUpdates` | 16.6ms (RAF) | **~60fps** | 拖拽、绘制、平移操作 |
| **Collab.onPointerUpdate** | **`lodash.throttle`** | **CURSOR_SYNC_TIMEOUT = 33ms** | **~30fps** | **协作端指针广播** |
| image refresh | `lodash.throttle` | 可配置 | 可配置 | 图片元素刷新 |
| React 更新 | `unstable_batchedUpdates` | N/A | 批量 | 同一帧内多次 setState 合并 |

**协作端同步完整链路图：**

```
浏览器 pointermove 事件
    │ 频率：60-120Hz（硬件决定）
    ▼
┌─────────────────────────────────────────┐
│ App.tsx: handleCanvasPointerMove         │
│ （无节流，60-120Hz）                     │
│  - 更新 gesture.pointers                 │
│  - 调用 savePointer() → onPointerUpdate  │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ excalidraw-app/collab/Collab.tsx        │
│ onPointerUpdate = throttle(fn, 33ms)    │
│ （lodash.throttle，~30fps）             │
│                                         │
│  过滤条件：                              │
│  payload.pointersMap.size < 2 ?         │
│    ├─ 是 → 广播给协作端                  │
│    └─ 否 → 丢弃，不广播                  │
└─────────────────────────────────────────┘
                    │
                    ▼
           Portal.broadcastMouseLocation()
           WebSocket volatile 消息
```

**lodash.throttle 行为说明：**
- 默认配置：`leading: true`, `trailing: true`
- 第一次调用立即执行（leading edge）
- 33ms 窗口内的后续调用被忽略
- 窗口结束时如果有待执行调用则执行最后一次（trailing edge）
- 返回的函数带有 `flush()` 和 `cancel()` 方法

### 5.4 协作端多指数据广播

```typescript
// 广播数据结构
{
  pointer: { x, y, tool },     // 当前事件触发指针坐标（主指针）
  button: "up" | "down",       // 按钮状态
  pointersMap: Map<number, {x, y}>,  // 所有活跃指针对象，多指时 size >= 2
}
```

---

## 6. 撤销栈交互机制（最终修正版）

### 6.1 Store 核心 (`element/src/store.ts:38-69`)

```typescript
export const CaptureUpdateAction = {
  IMMEDIATELY: "IMMEDIATELY",  // 立即入撤销栈（本地用户操作）
  NEVER: "NEVER",              // 永不入栈（远程更新、小元素删除）
  EVENTUALLY: "EVENTUALLY",    // 延迟入栈（异步多步流程）
} as const;
```

### 6.2 ⚠️ 最终修正：NEVER 分支与 snapshot 更新关系

**之前的错误描述**：NEVER 分支 "只通知，不更新 snapshot"

**✅ 真实语义** (`store.ts:362-385`)

```typescript
try {
  switch (action) {
    case CaptureUpdateAction.IMMEDIATELY:
      this.emitDurableIncrement(nextSnapshot, change, delta);  // 入撤销栈
      break;
    case CaptureUpdateAction.NEVER:
    case CaptureUpdateAction.EVENTUALLY:
      this.emitEphemeralIncrement(nextSnapshot, change);     // 只通知，不入栈
      break;
  }
} finally {
  // ⚠️ 更新 snapshot 在 finally 块中，无论什么分支都会执行！
  switch (action) {
    // ✅ IMMEDIATELY 和 NEVER 都更新 snapshot！
    case CaptureUpdateAction.IMMEDIATELY:
    case CaptureUpdateAction.NEVER:
      this.snapshot = nextSnapshot;  // NEVER 也更新 snapshot！
      break;
    // ❌ 只有 EVENTUALLY 不更新 snapshot
  }
}
```

**三个分支的完整对比：**

| 行为 | IMMEDIATELY | NEVER | EVENTUALLY |
|------|-------------|-------|------------|
| 入撤销栈 | ✅ 是 | ❌ 否 | ❌ 否 |
| 更新 snapshot | ✅ 是 | ✅ 是 | ❌ 否 |
| 发出通知 | DurableIncrement | EphemeralIncrement | EphemeralIncrement |
| 优先级 | 最高 | 中等 | 最低 |

**NEVER 分支的设计意图：**
1. **更新 snapshot**：将变更固化到快照，确保后续 delta 计算不会重复包含
2. **不入撤销栈**：用户无法撤销这个操作（通常是远程变更、系统自动清理）
3. **防止后续捕获**：snapshot 已更新，后续 IMMEDIATELY 操作计算 delta 时不会再包含这个变更

### 6.3 Store 捕获链路（最终修正版）

```
scheduleCapture()
    ↓ （store.ts:110-112）
scheduleAction(IMMEDIATELY)
    ↓
commit(elements, appState)
    ├─ flushMicroActions() - 先执行所有 micro 动作
    ├─ getScheduledMacroAction() - 获取优先级最高的 action
    │   优先级: IMMEDIATELY > NEVER > EVENTUALLY
    └─ processAction()
        ├─ maybeCloneSnapshot() - 克隆检测变更
        ├─ switch(action)
        │   ├─ IMMEDIATELY → emitDurableIncrement() → 计算 delta
        │   │                           ↓
        │   │                       onDurableIncrementEmitter.trigger()
        │   │                           ↓
        │   │                       history.record(delta) → 入 undoStack
        │   ├─ NEVER → emitEphemeralIncrement() - 只通知，不入栈
        │   └─ EVENTUALLY → emitEphemeralIncrement() - 只通知，不入栈
        └─ finally: 更新 snapshot
            ├─ IMMEDIATELY/NEVER → this.snapshot = nextSnapshot
            └─ EVENTUALLY → 不更新
```

### 6.4 小元素删除的 NEVER 语义（`App.tsx:10988-11006`）

```typescript
if (newElement && isInvisiblySmallElement(newElement)) {
  // 注释说明：update the store snapshot, so that invisible elements are not captured by the store
  this.updateScene({
    elements: 过滤删除该元素,
    appState: { newElement: null },
    captureUpdate: CaptureUpdateAction.NEVER,  // ⚠️ NEVER = 更新 snapshot，但不入栈
  });
  return;
}
```

**NEVER 在这里的作用：**
1. **更新 snapshot**：将"删除该元素"这个变更记录到 snapshot 中
2. **不入撤销栈**：用户无法撤销这个删除操作
3. **防止后续捕获**：snapshot 已更新，后续 IMMEDIATELY 操作计算 delta 时不会再包含这个元素

### 6.5 历史记录核心 (`history.ts:90-240`)

```typescript
class History {
  undoStack: HistoryDelta[];   // 撤销栈
  redoStack: HistoryDelta[];   // 重做栈
  
  record(delta: StoreDelta) {
    const historyDelta = HistoryDelta.inverse(delta);  // 转为反向 delta
    this.undoStack.push(historyDelta);
    // 元素变更时清空 redoStack
    if (!historyDelta.elements.isEmpty()) {
      this.redoStack.length = 0;
    }
  }
}
```

### 6.6 撤销时机与条件 (`actionHistory.tsx:31-39`)

```typescript
if (
  !appState.multiElement &&           // 不在多点绘制中
  !appState.resizingElement &&        // 不在缩放中
  !appState.editingTextElement &&     // 不在文本编辑中
  !appState.newElement &&             // 不在新建元素中
  !appState.selectedElementsAreBeingDragged &&  // 不在拖拽中
  !appState.selectionElement &&       // 不在框选中
  !app.flowChartCreator.isCreatingChart
) {
  // 允许撤销/重做
}
```

### 6.7 拖拽与撤销栈的交互

**关键时间点：**

1. **PointerDown** - `originalElements` 深拷贝，作为撤销基准
2. **PointerMove** - 实时 `mutateElement`，Store snapshot 不更新（EVENTUALLY）
3. **PointerUp** - `actionFinalize()` 触发 `CaptureUpdateAction.IMMEDIATELY`
4. **Store.commit()** - 计算 delta = 当前状态 - 上一次 snapshot，`emitDurableIncrement()`
5. **History.record()** - 反向 delta 压入 `undoStack`

**拖拽过程中的撤销安全保证：**
- 拖拽中 `selectedElementsAreBeingDragged = true`，阻塞撤销操作
- `originalElements` 保存了拖拽前的完整状态，确保可精确回滚
- 每次 `mutateElement` 都通过 Store 追踪，但暂不入栈

---

## 7. PointerUp 丢失的清理补偿机制（最终深度解析版）

### 7.1 问题场景

PointerUp 事件可能在以下情况丢失：
- 用户切换浏览器标签页
- 用户切换到其他应用
- 浏览器进程崩溃
- 指针移出浏览器窗口后释放

### 7.2 核心清理机制 (`App.tsx:8246-8249`)

```typescript
private maybeCleanupAfterMissingPointerUp = (event: PointerEvent | null) => {
  lastPointerUp?.();  // 调用上一次的 teardown 函数（平移/滚动条拖拽）
  this.missingPointerEventCleanupEmitter.trigger(event).clear();
};
```

### 7.3 触发时机

| 触发点 | 代码位置 | 说明 |
|--------|----------|------|
| 新 pointerDown 前 | `App.tsx:7715` | 每次按下前先清理上一次可能丢失的 pointerUp |
| 窗口获得焦点时 | `App.tsx:3334` | 切换标签页/应用后回到窗口时强制清理 |

### 7.4 事件监听器双重注册模式

```typescript
// App.tsx:8123-8134
const onPointerUp = this.onPointerUpFromPointerDownHandler(pointerDownState);

// 1. 正常 window 监听
window.addEventListener(EVENT.POINTER_UP, onPointerUp, { once: true });

// 2. 补偿清理监听（兜底）
this.missingPointerEventCleanupEmitter.once((_event) =>
  onPointerUp(_event || event.nativeEvent),  // event 为 null 时用原始 pointerDown 事件替代
);

// 特殊：橡皮擦场景下用 rAF 延迟注册，避免同一帧内触发
requestAnimationFrame(() => {
  unsubCleanup = this.missingPointerEventCleanupEmitter.once(onPointerUp);
});
```

### 7.5 对撤销捕获链路的影响（最终深度解析版）

**⚠️ 关键修正：两层 flush 机制分别影响不同链路**

```
【正常流程】
pointerDown → 拖拽 mutate（EVENTUALLY，snapshot 不更新）
    ↓
pointerUp 事件到达
    ↓
├─ 10606: onMove.flush() → 强制执行最后一次 throttleRAF 中的 move 事件
│   （作用域：拖拽绘制逻辑 @60fps）
├─ Collab.onPointerUpdate.flush() → 执行最后一次 lodash.throttle 中的广播
│   （作用域：协作同步 @30fps，需显式调用）
    ↓
10811: missingPointerEventCleanupEmitter.clear() → 取消补偿监听
    ↓
元素处理分支：
  ├─ 小元素删除 → updateScene(NEVER) → snapshot 更新，不入栈
  └─ 正常元素 → actionFinalize → scheduleCapture(IMMEDIATELY)
    ↓
Store.commit() →
  ├─ flushMicroActions() → 先执行 NEVER micro 动作（如果有）
  ├─ 计算 delta = 当前状态 - 上一次 snapshot
  ├─ emitDurableIncrement → history.record() → 入栈
  └─ finally: snapshot = nextSnapshot
```

```
【补偿流程（pointerUp 丢失）】
pointerDown → 拖拽 mutate（EVENTUALLY，snapshot 不更新）
    ↓
（pointerUp 丢失，状态停留在拖拽状态）
    ↓
（可能插入其他操作：如远程协作者修改、小元素删除 NEVER 操作等）
    ↓
下次 pointerDown 或 window focus →
maybeCleanupAfterMissingPointerUp() →
missingPointerEventCleanupEmitter.trigger(event|null) →
调用 onPointerUp 闭包 →
    ↓
├─ 10606: onMove.flush() → 强制执行最后一次 throttleRAF 中的 move 事件
│   （⚠️ 补偿时确保拖拽最后状态被捕获）
├─ （协作端 throttle 可能需要显式 flush()）
    ↓
10811: missingPointerEventCleanupEmitter.clear() → 清除补偿监听
    ↓
元素处理分支：
  ├─ 小元素删除 → updateScene(NEVER) → snapshot 更新，不入栈
  └─ 正常元素 → actionFinalize → scheduleCapture(IMMEDIATELY)
    ↓
Store.commit() →
  ├─ flushMicroActions() → 先执行所有 micro 动作
  ├─ 计算 delta = 当前状态 - 上一次 snapshot
  ├─ emitDurableIncrement → history.record() → 入栈
  └─ finally: snapshot = nextSnapshot
```

### 7.6 关键差异点 - 恢复边界分析（最终版）

**⚠️ 恢复边界 = 上一次 snapshot 更新点**

snapshot 被更新的操作：
1. **IMMEDIATELY** 操作（本地用户拖拽完成等）
2. **NEVER** 操作（远程协作者修改、小元素删除等）
3. **EVENTUALLY** 操作 **不**更新 snapshot

**1. 事件对象差异**：
- 正常：真实的 PointerUp 事件，`event.clientX/clientY` 是抬起位置
- 补偿：可能为 `null`，回退到原始 pointerDown 事件，坐标是按下位置

**2. 时间延迟**：
- 正常：即时触发，snapshot 是连续的
- 补偿：可能延迟数秒甚至数分钟，中间可能插入其他操作

**⚠️ 3. NEVER 操作对恢复边界的影响（最终关键！）**

如果 pointerup 丢失期间发生了 **NEVER 操作**（如远程协作者修改、小元素删除）：

```
pointerDown → 拖拽 mutate（EVENTUALLY）
    ↓
pointerUp 丢失
    ↓
【插入：远程协作者修改元素 → updateScene(NEVER)
    → snapshot 被更新（因为 NEVER 更新 snapshot）
    ↓
补偿触发 → onPointerUp → onMove.flush() → scheduleCapture(IMMEDIATELY)
    ↓
Store.commit()
  ├─ delta = 当前状态 - 上一次 snapshot（已被 NEVER 更新过）
  └─ 结果：delta 只包含补偿触发时与上一次 snapshot 的差
```

**恢复边界总结表（最终版）：**

| 场景 | 恢复边界 | delta 包含内容 | 协作端同步状态 |
|------|---------|--------------|---------------|
| 无中间操作 | pointerDown 时的 snapshot | 完整拖拽变更 | 最后 ~33ms 的移动可能因 throttle trailing 已广播 |
| 中间有 IMMEDIATELY | 上一次 IMMEDIATELY 的 snapshot | 从该点之后的拖拽变更 | 取决于时间间隔 |
| 中间有 NEVER | 上一次 NEVER 的 snapshot | 从该点之后的拖拽变更 | 取决于时间间隔 |
| 中间有 EVENTUALLY | pointerDown 时的 snapshot | 完整拖拽变更 | 取决于时间间隔 |

**4. onMove.flush() 的关键作用** (`App.tsx:10606-10607`)：
```typescript
if (pointerDownState.eventListeners.onMove) {
  pointerDownState.eventListeners.onMove.flush();
}
```
- `onMove` 是 `withBatchedUpdatesThrottled` 返回的函数（`throttleRAF` 封装），有 `flush()` 方法
- 补偿触发时强制执行最后一次 throttle 中的 move 事件，确保拖拽的最后状态被捕获
- 如果没有 flush，最后几帧的拖拽变更可能丢失（最多 16.6ms 的移动量）

**⚠️ 5. 协作端 throttle 的 flush 问题**：
- `Collab.onPointerUpdate` 是 `lodash.throttle`，也有 `.flush()` 方法
- **但补偿流程中没有显式调用** `Collab.onPointerUpdate.flush()`
- 这意味着：pointerup 丢失时，最后 ~33ms 的指针移动可能**不会**广播给协作端
- 协作端看到的最后指针位置可能比实际位置落后最多 33ms

**6. emitter.clear() 的作用**：
- 触发后立即清空所有注册的补偿监听
- 避免同一轮交互被多次补偿触发
- 正常 pointerUp 时也会调用 `clear()` 取消补偿

### 7.7 Store 补偿捕获的正确性保证

```typescript
// store.ts:317-386 processAction()
private processAction(params) {
  const nextSnapshot = this.maybeCloneSnapshot(action, elements, appState);
  
  if (!nextSnapshot) return;  // 无变更则跳过
  
  // ⚠️ delta 始终与上一次 snapshot 比较，不管时间过了多久
  const delta = StoreDelta.calculate(prevSnapshot, nextSnapshot);
  
  if (!delta.isEmpty()) {
    this.emitDurableIncrement(nextSnapshot, change, delta);
  }
}
```

**保证机制**：
- Store 始终与上一次 snapshot 比较，而非与 pointerDown 时比较
- 延迟触发时，中间的远程变更（NEVER）已更新了 snapshot
- 最终 delta 只反映**当前状态与上一次 snapshot 的差**
- **但这也意味着：如果中间有 NEVER 操作更新了 snapshot，恢复时的 delta 不会包含 NEVER 操作之前的拖拽变更**

### 7.8 恢复边界与协作端同步的关系（新增）

```
pointerDown → 拖拽开始
    ↓
pointerMove (60-120Hz)
    ├─ 拖拽绘制逻辑：throttleRAF @60fps → element mutate
    └─ 协作同步：lodash.throttle @30fps → WebSocket 广播
    ↓
pointerUp 丢失
    ↓
（协作端可能已收到最后一次广播，但落后实际位置最多 33ms）
    ↓
补偿触发 → maybeCleanupAfterMissingPointerUp()
    ├─ onMove.flush() → 执行最后一次拖拽绘制（最多 16.6ms 丢失）
    └─ （无 Collab.onPointerUpdate.flush()）→ 协作端可能落后最多 33ms
    ↓
Store.commit() → delta = 当前状态 - 上一次 snapshot
    ↓
（撤销捕获基于元素状态，不依赖协作端同步状态）
```

**协作端恢复边界：**
- 协作端看到的指针位置可能落后实际位置最多 33ms（CURSOR_SYNC_TIMEOUT）
- 但元素最终状态是一致的（通过元素同步机制，不是指针同步）
- 指针同步是**最佳努力**的，不影响元素状态的最终一致性

---

## 8. 协作端事件传递

### 8.1 指针位置广播 (`App.tsx:12906-12930` + `Collab.tsx:914-925`)

广播时机：
- `pointerMove` - 每次移动（经两层节流：核心层无节流 → 协作层 @30fps）
- `pointerDown` - 按下时
- `pointerUp` - 抬起时

### 8.2 远程指针渲染 (`InteractiveCanvas.tsx:107-136`)

```typescript
props.appState.collaborators.forEach((user, socketId) => {
  // 场景坐标转视口坐标
  remotePointerViewportCoords.set(socketId, sceneToViewport(...));
  // 按钮状态
  remotePointerButton.set(socketId, user.button);
  // 选中元素高亮
  if (user.selectedElementIds) {
    remoteSelectedElementIds.set(id, [socketId, ...]);
  }
});
```

---

## 9. 关键设计模式

### 9.1 闭包状态机
`onPointerMoveFromPointerDownHandler` 和 `onPointerUpFromPointerDownHandler` 通过闭包捕获 `pointerDownState`，实现：
- 状态在事件间安全传递
- 无需全局变量存储中间状态
- 每次交互的状态完全隔离

### 9.2 快照-增量模式
- `pointerDown` 时创建完整元素快照（用于回滚基准）
- `pointerMove` 时仅记录增量变更（EVENTUALLY，不更新 snapshot）
- `pointerUp` 时将增量一次性入撤销栈（IMMEDIATELY）

### 9.3 双重监听补偿模式
- 正常事件路径：`window.addEventListener('pointerup')`
- 补偿事件路径：`missingPointerEventCleanupEmitter.once()`
- 任一触发后立即清除另一路径，避免重复执行

### 9.4 多层节流与批量更新
- `throttleRAF` 确保拖拽绘制 60fps
- `lodash.throttle` 确保协作同步 30fps
- Store 层自动合并连续变更，减少撤销栈条目
- `unstable_batchedUpdates` 合并 React 重渲染

### 9.5 防御式编程
- `maybeCleanupAfterMissingPointerUp` 处理指针丢失
- `setPointerCapture` 确保事件不丢失
- `isInvisiblySmallElement` 自动清理脏数据
- `captureUpdate: NEVER` 防止误操作入栈（但更新 snapshot）

---

## 10. 修正后的核心状态流转图（最终版）

```
┌─────────────────────────────────────────────────────────────┐
│                     空闲状态 (idle)                         │
│  cursorButton: "up", newElement: null,                      │
│  selectedElementsAreBeingDragged: false                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼ pointerDown
┌─────────────────────────────────────────────────────────────┐
│              maybeCleanupAfterMissingPointerUp()            │
│              （清理上一次可能丢失的 pointerUp）               │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                创建 PointerDownState 快照                   │
│  - originalElements 深拷贝所有元素                          │
│  - 多指检测：gesture.pointers.size > 1 → return             │
│  - 根据工具类型进入对应分支                                  │
│  - 注册 window pointerUp + emitter 补偿监听                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│           三层 pointerMove 并行处理                           │
│                                                             │
│  第一层：handleCanvasPointerMove（无节流，60-120Hz）        │
│    ├─ savePointer() → 协作同步入口                          │
│    └─ 更新 gesture.pointers 坐标                            │
│                                                             │
│  第二层：onPointerMoveFromPointerDownHandler（@60fps）      │
│    ├─ 多指分支:                                             │
│    │   ├─ size === 2 → 双指缩放平移                         │
│    │   └─ size > 2  → 忽略                                 │
│    └─ 单指工具分支:                                         │
│        ├─ Selection → dragSelectedElements()               │
│        ├─ 绘图工具 → dragNewElement()                      │
│        ├─ Linear → handlePointDragging()                   │
│        ├─ Hand → translateCanvas()                         │
│        └─ Eraser → handleEraser()                          │
│                                                             │
│  第三层：Collab.onPointerUpdate（@30fps，协作层）           │
│    └─ 过滤: pointersMap.size < 2 ? 广播 : 丢弃              │
└───────────────────────────┬─────────────────────────────────┘
                            │
          ┌─────────────────┴─────────────────┐
          ▼                                   ▼
┌──────────────────────┐           ┌──────────────────────┐
│   正常 pointerUp     │           │  pointerUp 丢失       │
│  window 事件触发      │           │  下次 pointerDown     │
│                      │           │  或 window focus      │
└───────────┬──────────┘           └──────────┬───────────┘
            │                                 │
            ▼                                 ▼
┌─────────────────────────────────────────────────────────────┐
│                      onPointerUp()                          │
│  - onMove.flush() 强制刷新 throttleRAF 中 move 事件           │
│  - （协作端 throttle 无显式 flush，可能落后最多 33ms）        │
│  - emitter.clear() 取消补偿监听                              │
│  - 移除 window 事件监听器                                    │
│  - 线性元素分支:                                             │
│    ├─ 小拖拽 → 多点编辑(非触屏)/固定长度(触屏)               │
│    └─ 正常拖拽 → actionFinalize                              │
│  - 其他元素分支:                                             │
│    ├─ 过小元素 → 删除 + captureUpdate: NEVER                 │
│    │             → snapshot 更新，但不入栈                     │
│    └─ 正常 → actionFinalize + captureUpdate: IMMEDIATELY     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Store.commit()                            │
│  - flushMicroActions()                                      │
│  - IMMEDIATELY → emitDurableIncrement()                     │
│  - delta = 当前状态 - 上一次 snapshot                        │
│  - ⚠️ 恢复边界 = 上一次 IMMEDIATELY/NEVER 更新点              │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                history.record(delta)                        │
│  - inverse(delta) 压入 undoStack                             │
│  - 元素变更清空 redoStack                                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
                    回到空闲状态
```

---

## 11. 代码位置索引（最终版）

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| handleCanvasPointerDown | `components/App.tsx` | 7688 |
| handleCanvasPointerMove | `components/App.tsx` | 6888 |
| handleCanvasPointerUp | `components/App.tsx` | 8148 |
| onPointerMoveFromPointerDownHandler | `components/App.tsx` | 9675 |
| onPointerUpFromPointerDownHandler | `components/App.tsx` | 10598 |
| maybeCleanupAfterMissingPointerUp | `components/App.tsx` | 8246 |
| missingPointerEventCleanupEmitter | `components/App.tsx` | 740 |
| gesture 模块级单例 | `components/App.tsx` | 614 |
| Gesture 类型定义 | `types.ts` | 512 |
| 线性元素小拖拽分支 | `components/App.tsx` | 10867-10963 |
| 小元素检测删除 | `components/App.tsx` | 10988-11006 |
| 多指过滤（选择阻止） | `components/App.tsx` | 7903 |
| 双指缩放触发 | `components/App.tsx` | 6909 |
| 缩放初始化 | `components/App.tsx` | 8375 |
| 自由绘制多指处理 | `components/App.tsx` | 7760-7793 |
| initialPointerDownState | `components/App.tsx` | 8384 |
| dragSelectedElements | `element/src/dragElements.ts` | 35 |
| dragNewElement | `element/src/dragElements.ts` | 231 |
| Store 类 | `element/src/store.ts` | 78 |
| scheduleCapture | `element/src/store.ts` | 110 |
| commit | `element/src/store.ts` | 183 |
| processAction | `element/src/store.ts` | 317 |
| CaptureUpdateAction | `element/src/store.ts` | 38 |
| ⚠️ NEVER 更新 snapshot | `element/src/store.ts` | 377-383 |
| History 类 | `history.ts` | 90 |
| actionFinalize | `actions/actionFinalize.tsx` | 52 |
| undo/redo action | `actions/actionHistory.tsx` | 31-39 |
| savePointer | `components/App.tsx` | 12906 |
| ⚠️ Collab.onPointerUpdate 节流 | `excalidraw-app/collab/Collab.tsx` | 914-925 |
| ⚠️ CURSOR_SYNC_TIMEOUT 常量 | `excalidraw-app/app_constants.ts` | 8 |
| ⚠️ 协作端多指过滤条件 | `excalidraw-app/collab/Collab.tsx` | 920 |
| broadcastMouseLocation | `excalidraw-app/collab/Portal.tsx` | 202 |
| onPointerUpdate 类型定义 | `types.ts` | 601-605 |
| PointerDownState 类型 | `types.ts` | 853 |
| gesture 工具函数 | `gesture.ts` | 1 |
| throttleRAF 实现 | `common/src/utils.ts` | 155 |
| withBatchedUpdatesThrottled | `reactUtils.ts` | 23 |
| MINIMUM_ARROW_SIZE | 常量定义 | - |
| isInvisiblySmallElement | 工具函数 | - |
| onMove.flush() 调用 | `components/App.tsx` | 10606 |

