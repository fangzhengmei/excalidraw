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
| 协作同步器 | 指针位置广播 | `App.tsx:12906-12930` |
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

### 3.2 第二阶段：PointerMove (`App.tsx:9675-10597`)

**动态闭包处理器：**
```typescript
private onPointerMoveFromPointerDownHandler(pointerDownState) {
  return withBatchedUpdatesThrottled((event) => {
    // 闭包捕获 pointerDownState，实现状态延续
    // withBatchedUpdatesThrottled = throttleRAF + batchedUpdates
  });
}
```

**节流机制 (`reactUtils.ts:23-32`)：**
```typescript
export const withBatchedUpdatesThrottled = (func) => {
  return throttleRAF((event) => {
    unstable_batchedUpdates(func, event);  // 限制到 60fps + React 批量更新
  });
};
```

**执行流程：**

```
事件节流 (withBatchedUpdatesThrottled, ~60fps)
    ↓
9682: 坐标转换 → 场景坐标
    ↓
9750: 懒初始化 drag.offset（确保选择集已更新）
    ↓
双指缩放检测 (gesture.pointers.size === 2)
    ├─ 计算中心点 center = getCenter(pointers)
    ├─ 计算距离 distance = getDistance(pointers)
    ├─ scaleFactor = distance / initialDistance
    └─ translateCanvas({ zoom: initialScale * scaleFactor, scrollX/Y })
    ↓
单指工具分支处理：
  ├─ 橡皮擦 → handleEraser()
  ├─ 激光笔 → laserTrails.addPointToPath()
  ├─ 缩放状态 → maybeHandleResize()
  ├─ 裁剪状态 → maybeHandleCrop()
  ├─ 线性元素编辑 → LinearElementEditor.handlePointDragging()
  └─ 普通拖拽 →
        10145: snapDraggedElements() - 计算对齐吸附
        10158: dragSelectedElements() - 执行元素拖拽
        10168: setState({ selectedElementsAreBeingDragged: true })
    ↓
savePointer() - 协作端指针同步（隐式随 pointerMove 节流到 ~60fps）
```

**拖拽核心逻辑 (`dragElements.ts:35-171`)：**

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
pointerMove
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
pointerMove (withBatchedUpdatesThrottled)
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

## 5. 多指场景下的过滤与节流规则（新增章节）

### 5.1 多指手势过滤规则

| 检测点 | 代码位置 | 条件 | 行为 |
|--------|----------|------|------|
| 选择阻止 | `App.tsx:7903` | `gesture.pointers.size > 1` | pointerDown 直接 return，不进入选择/绘制 |
| 平移阻止 | `App.tsx:8257` | `gesture.pointers.size > 1` | Hand 工具不进入平移模式 |
| 双指缩放触发 | `App.tsx:6909` | `gesture.pointers.size === 2` | 进入双指缩放平移模式 |
| 缩放初始化 | `App.tsx:8375` | `gesture.pointers.size === 2` | 记录 initialScale、initialDistance、lastCenter |
| 点击阻止 | `App.tsx:1373` | `gesture.pointers.size >= 2` | 阻止 iframe 点击交互 |
| 触屏检测 | `App.tsx:5632` | `gesture.pointers.size >= 2` | 判定为触屏设备，阻止取消选择 |
| 自由绘制多指 | `App.tsx:7760-7793` | 触屏 + freedraw 绘制中 + 第二指按下 | 短尖峰删除，长轨迹保留 |
| freedraw 缩放禁用 | `App.tsx:6903` | freedraw + penMode | 禁用双指缩放，避免误操作 |

### 5.2 手势状态管理 (`gesture.ts`)

```typescript
const gesture = {
  pointers: Map<number, PointerCoords>,  // 活跃指针集合（pointerId → 坐标）
  lastCenter: { x, y } | null,           // 上次双指中心点
  initialScale: number | null,           // 双指按下时的缩放值
  initialDistance: number | null,        // 双指初始距离
};

// 工具函数
export const getCenter = (pointers) => { /* 计算多指中心坐标 */ };
export const getDistance = ([a, b]) => { /* 计算两点欧式距离 */ };
```

### 5.3 节流规则汇总

| 节流点 | 实现方式 | 频率 | 影响范围 |
|--------|----------|------|----------|
| pointerMove 处理 | `withBatchedUpdatesThrottled` = `throttleRAF` + `batchedUpdates` | ~60fps | 所有拖拽、绘制、平移操作 |
| 协作指针同步 | 随 pointerMove 隐式节流 | ~60fps | `savePointer()` 广播频率 |
| image refresh | `lodash.throttle` | 可配置 | 图片元素刷新 |
| React 更新 | `unstable_batchedUpdates` | 批量 | 同一帧内多次 setState 合并 |

### 5.4 协作端多指数据广播

```typescript
// App.tsx:12906-12930
private savePointer = (x: number, y: number, button: "up" | "down") => {
  if (!x || !y) return;  // 空坐标过滤
  
  const { x: sceneX, y: sceneY } = viewportCoordsToSceneCoords(...);
  
  const pointer: CollaboratorPointer = {
    x: sceneX,
    y: sceneY,
    tool: activeTool.type === "laser" ? "laser" : "pointer",
  };
  
  this.props.onPointerUpdate?.({
    pointer,        // 主指针位置
    button,         // 按钮状态
    pointersMap: gesture.pointers,  // 完整多指 Map 也广播
  });
};
```

---

## 6. 撤销栈交互机制

### 6.1 Store 核心 (`element/src/store.ts:38-69`)

```typescript
export const CaptureUpdateAction = {
  IMMEDIATELY: "IMMEDIATELY",  // 立即入撤销栈（本地用户操作）
  NEVER: "NEVER",              // 永不入栈（远程更新、小元素删除）
  EVENTUALLY: "EVENTUALLY",    // 延迟入栈（异步多步流程）
} as const;
```

### 6.2 Store 捕获链路

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
        └─ switch(action)
            ├─ IMMEDIATELY → emitDurableIncrement() → 计算 delta
            │                           ↓
            │                       onDurableIncrementEmitter.trigger()
            │                           ↓
            │                       history.record(delta) → 入 undoStack
            ├─ NEVER → emitEphemeralIncrement() - 只通知，不更新快照
            └─ EVENTUALLY → emitEphemeralIncrement() - 不更新快照
```

### 6.3 历史记录核心 (`history.ts:90-240`)

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

### 6.4 撤销时机与条件 (`actionHistory.tsx:31-39`)

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

### 6.5 拖拽与撤销栈的交互

**关键时间点：**

1. **PointerDown** - `originalElements` 深拷贝，作为撤销基准
2. **PointerMove** - 实时 `mutateElement`，Store snapshot 不更新（EVENTUALLY）
3. **PointerUp** - `actionFinalize()` 触发 `CaptureUpdateAction.IMMEDIATELY`
4. **Store.commit()** - 计算 delta，`emitDurableIncrement()`
5. **History.record()** - 反向 delta 压入 `undoStack`

**拖拽过程中的撤销安全保证：**
- 拖拽中 `selectedElementsAreBeingDragged = true`，阻塞撤销操作
- `originalElements` 保存了拖拽前的完整状态，确保可精确回滚
- 每次 `mutateElement` 都通过 Store 追踪，但暂不入栈

---

## 7. PointerUp 丢失的清理补偿机制（新增章节）

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

### 7.5 对撤销捕获链路的影响

**正常流程 vs 补偿流程对比：**

```
【正常流程】
pointerDown → 拖拽 mutate → pointerUp →
  missingPointerEventCleanupEmitter.clear() → （取消补偿监听）
  actionFinalize → store.scheduleCapture() →
  store.commit() → emitDurableIncrement → history.record() → 入栈

【补偿流程（pointerUp 丢失）】
pointerDown → 拖拽 mutate → （pointerUp 丢失）
  → 下次 pointerDown 或 window focus →
  maybeCleanupAfterMissingPointerUp() →
  missingPointerEventCleanupEmitter.trigger(event|null) →
  调用 onPointerUp 闭包 →
  actionFinalize → scheduleCapture → commit → 入栈
```

**关键差异点：**

1. **事件对象差异**：
   - 正常：真实的 PointerUp 事件
   - 补偿：可能为 `null`，回退到原始 pointerDown 事件

2. **时间延迟**：
   - 正常：即时触发
   - 补偿：可能延迟数秒甚至数分钟（取决于用户何时回到窗口）

3. **状态一致性风险**：
   - 补偿触发时，元素可能已被远程协作者修改
   - `Store.commit()` 重新计算当前状态与上一次 snapshot 的 delta
   - 可能包含非预期的中间变更

4. **特殊 teardown 调用**：
   - 平移/滚动条拖拽场景下，`lastPointerUp?.()` 会被调用
   - 重置 `isPanning`、`isDraggingScrollBar` 等标志
   - 恢复光标样式

5. **emitter.clear() 的作用**：
   - 触发后立即清空所有注册的补偿监听
   - 避免同一轮交互被多次补偿触发
   - 正常 pointerUp 时也会调用 `clear()` 取消补偿

### 7.6 Store 补偿捕获的正确性保证

```typescript
// store.ts:317-386 processAction()
private processAction(params) {
  const nextSnapshot = this.maybeCloneSnapshot(action, elements, appState);
  
  if (!nextSnapshot) return;  // 无变更则跳过
  
  // 计算 delta = 当前状态 - 上一次 snapshot
  // 即使延迟触发，delta 也只包含实际变更
  const delta = StoreDelta.calculate(prevSnapshot, nextSnapshot);
  
  if (!delta.isEmpty()) {
    this.emitDurableIncrement(nextSnapshot, change, delta);
  }
}
```

**保证机制**：
- Store 始终与上一次 snapshot 比较，而非与 pointerDown 时比较
- 延迟触发时，中间的远程变更已被包含在 snapshot 中
- 最终 delta 只反映当前用户操作的实际变更

---

## 8. 协作端事件传递

### 8.1 指针位置广播 (`App.tsx:12906-12930`)

广播时机：
- `pointerMove` - 每次移动（随 pointerMove 节流到 ~60fps）
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

### 9.4 事件节流与批量更新
- `withBatchedUpdatesThrottled` 确保 pointerMove 60fps
- Store 层自动合并连续变更，减少撤销栈条目
- `unstable_batchedUpdates` 合并 React 重渲染

### 9.5 防御式编程
- `maybeCleanupAfterMissingPointerUp` 处理指针丢失
- `setPointerCapture` 确保事件不丢失
- `isInvisiblySmallElement` 自动清理脏数据
- `captureUpdate: NEVER` 防止误操作入栈

---

## 10. 修正后的核心状态流转图

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
                            ▼ pointerMove (throttled @60fps)
┌─────────────────────────────────────────────────────────────┐
│                    拖拽/绘制进行中                          │
│  多指分支:                                                   │
│  ├─ size === 2 → 双指缩放平移                                │
│  └─ size > 2  → 忽略                                        │
│                                                              │
│  单指工具分支:                                               │
│  ├─ Selection → dragSelectedElements()                       │
│  ├─ 绘图工具 → dragNewElement()                              │
│  ├─ Linear → handlePointDragging()                           │
│  ├─ Hand → translateCanvas()                                 │
│  └─ Eraser → handleEraser()                                  │
│                                                              │
│  状态标志:                                                   │
│  - drag.hasOccurred = true                                   │
│  - selectedElementsAreBeingDragged = true                    │
│  - savePointer() 协作同步                                    │
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
│  - emitter.clear() 取消补偿监听                              │
│  - 移除 window 事件监听器                                    │
│  - 线性元素分支:                                             │
│    ├─ 小拖拽 → 多点编辑(非触屏)/固定长度(触屏)               │
│    └─ 正常拖拽 → actionFinalize                              │
│  - 其他元素分支:                                             │
│    ├─ 过小元素 → 删除 + captureUpdate: NEVER                 │
│    └─ 正常 → actionFinalize + captureUpdate: IMMEDIATELY     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                   Store.commit()                            │
│  - flushMicroActions()                                      │
│  - IMMEDIATELY → emitDurableIncrement()                     │
│  - delta = 当前状态 - 上一次 snapshot                        │
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

## 11. 代码位置索引

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| handleCanvasPointerDown | `components/App.tsx` | 7688 |
| handleCanvasPointerMove | `components/App.tsx` | 6888 |
| handleCanvasPointerUp | `components/App.tsx` | 8148 |
| onPointerMoveFromPointerDownHandler | `components/App.tsx` | 9675 |
| onPointerUpFromPointerDownHandler | `components/App.tsx` | 10598 |
| maybeCleanupAfterMissingPointerUp | `components/App.tsx` | 8246 |
| missingPointerEventCleanupEmitter | `components/App.tsx` | 740 |
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
| CaptureUpdateAction | `element/src/store.ts` | 38 |
| History 类 | `history.ts` | 90 |
| actionFinalize | `actions/actionFinalize.tsx` | 52 |
| undo/redo action | `actions/actionHistory.tsx` | 31-39 |
| savePointer | `components/App.tsx` | 12906 |
| PointerDownState 类型 | `types.ts` | 853 |
| gesture 工具函数 | `gesture.ts` | 1 |
| withBatchedUpdatesThrottled | `reactUtils.ts` | 23 |
| MINIMUM_ARROW_SIZE | 常量定义 | - |
| isInvisiblySmallElement | 工具函数 | - |
