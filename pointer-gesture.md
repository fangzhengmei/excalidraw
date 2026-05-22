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
| 协作同步器 | 指针位置广播 | `App.tsx:12906-12930` |

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
7753: updateGestureOnPointerDown() - 更新手势状态（多指检测）
    ↓
7838: setState({ cursorButton: "down" }) - 标记按钮按下
    ↓
7842: savePointer() - 广播指针位置给协作端
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
8137-8140: 绑定 window 级事件监听器
```

**关键点：**
- **事件捕获**：使用 `setPointerCapture` 确保后续事件都由 canvas 处理，即使指针移出画布
- **手势优先级**：多指手势（`gesture.pointers.size > 1`）会阻止单选操作
- **橡皮笔切换**：检测到橡皮笔按钮时自动切换工具，抬起后还原

### 3.2 第二阶段：PointerMove (`App.tsx:9675-10597`)

**动态闭包处理器：**
```typescript
private onPointerMoveFromPointerDownHandler(pointerDownState) {
  return withBatchedUpdatesThrottled((event) => {
    // 闭包捕获 pointerDownState，实现状态延续
  });
}
```

**执行流程：**

```
事件节流 (withBatchedUpdatesThrottled)
    ↓
9682: 坐标转换 → 场景坐标
    ↓
9750: 懒初始化 drag.offset（确保选择集已更新）
    ↓
工具分支处理：
  ├─ 橡皮擦 → handleEraser()
  ├─ 激光笔 → laserTrails.addPointToPath()
  ├─ 缩放状态 → maybeHandleResize()
  ├─ 裁剪状态 → maybeHandleCrop()
  ├─ 线性元素编辑 → LinearElementEditor.handlePointDragging()
  └─ 普通拖拽 →
        10145: snapDraggedElements() - 计算对齐吸附
        10158: dragSelectedElements() - 执行元素拖拽
        10168: setState({ selectedElementsAreBeingDragged: true })
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
元素创建分支：
  ├─ freedraw → 追加终点 → actionFinalize
  ├─ 线性元素 → 
  │     10880: 检测拖拽距离 < MINIMUM_ARROW_SIZE 则删除
  │     10934: multiElement 模式下进入多点编辑
  │     否则 → actionFinalize
  └─ 其他新元素 →
        11021: isInvisiblySmallElement() 则删除
        否则 → actionFinalize
    ↓
10813-10828: 移除 window 事件监听器
    ↓
10830: onPointerUpEmitter.trigger() - 通知外部
```

---

## 4. 不同工具下的状态流转

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

### 4.2 绘图工具状态机（Rectangle/Ellipse/Diamond 等）

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
actionFinalize()
    ├─ 检测是否过小 → 标记删除
    ├─ newElement = null
    ├─ 切换回 selection 工具（除非 locked）
    └─ captureUpdate: IMMEDIATELY → 入撤销栈
```

### 4.3 Linear 工具状态机（Arrow/Line）

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
pointerUp
    ├─ 编辑模式 → bindOrUnbindBindingElement() 处理绑定
    ├─ 拖拽距离过小 → 删除元素
    ├─ multiElement 模式 → 保持绘制状态等待下一点
    └─ 按 Enter/Esc → actionFinalize 结束绘制
```

### 4.4 Hand 工具/视图模式状态机

```
pointerDown
    ↓
handleCanvasPanUsingWheelOrSpaceDrag()
    ├─ isPanning = true
    ├─ setCursor(GRABBING)
    └─ 绑定 onPointerMove/onPointerUp
    ↓
pointerMove
    ↓
translateCanvas()
    ├─ scrollX -= deltaX / zoom
    └─ scrollY -= deltaY / zoom
    ↓
pointerUp
    ↓
isPanning = false, 恢复光标
```

---

## 5. 撤销栈交互机制

### 5.1 历史记录核心 (`history.ts:90-240`)

```typescript
class History {
  undoStack: HistoryDelta[];   // 撤销栈
  redoStack: HistoryDelta[];   // 重做栈
  store: Store;               // 状态存储引用
  
  record(delta: StoreDelta) {
    // 将增量转为反向 delta 压入 undoStack
    const historyDelta = HistoryDelta.inverse(delta);
    this.undoStack.push(historyDelta);
    // 元素变更时清空 redoStack
    if (!historyDelta.elements.isEmpty()) {
      this.redoStack.length = 0;
    }
  }
  
  undo() {
    // 弹出 undoStack 顶部，应用后压入 redoStack
  }
  
  redo() {
    // 弹出 redoStack 顶部，应用后压入 undoStack
  }
}
```

### 5.2 撤销时机与条件

**禁止撤销的状态（`actionHistory.tsx:31-39`）：**
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
  // 允许撤销
}
```

### 5.3 拖拽与撤销栈的交互

**关键时间点：**

1. **PointerDown** - `originalElements` 深拷贝，作为撤销基准
2. **PointerMove** - 实时 `mutateElement`，但不立即入栈
3. **PointerUp** - `actionFinalize()` 触发 `CaptureUpdateAction.IMMEDIATELY`
4. **Store 捕获** - `store.scheduleCapture()` 生成 `StoreDelta`
5. **History 记录** - `history.record(delta)` 压入撤销栈

**拖拽过程中的撤销安全保证：**
- 拖拽中 `selectedElementsAreBeingDragged = true`，阻塞撤销操作
- `originalElements` 保存了拖拽前的完整状态，确保可精确回滚
- 每次 `mutateElement` 都通过 Store 追踪增量，最终一次性入栈

---

## 6. 协作端事件传递

### 6.1 指针位置广播 (`App.tsx:12906-12930`)

```typescript
private savePointer = (x: number, y: number, button: "up" | "down") => {
  const { x: sceneX, y: sceneY } = viewportCoordsToSceneCoords(
    { clientX: x, clientY: y },
    this.state,
  );
  
  const pointer: CollaboratorPointer = {
    x: sceneX,
    y: sceneY,
    tool: this.state.activeTool.type === "laser" ? "laser" : "pointer",
  };
  
  this.props.onPointerUpdate?.({
    pointer,
    button,
    pointersMap: gesture.pointers,  // 多指手势数据
  });
};
```

### 6.2 广播时机

| 事件 | 时机 | 数据 |
|------|------|------|
| pointerMove | 每次移动 | `savePointer(clientX, clientY, cursorButton)` |
| pointerDown | 按下时 | `savePointer(clientX, clientY, "down")` |
| pointerUp | 抬起时 | `savePointer(clientX, clientY, "up")` |
| 平移画布 | 拖拽中 | 实时更新 collaborator 状态 |

### 6.3 远程指针渲染 (`InteractiveCanvas.tsx:107-136`)

```typescript
// 从 appState.collaborators 读取所有协作者
props.appState.collaborators.forEach((user, socketId) => {
  // 转换场景坐标到视口坐标
  remotePointerViewportCoords.set(
    socketId,
    sceneCoordsToViewportCoords({ sceneX, sceneY }, props.appState)
  );
  // 按钮状态
  remotePointerButton.set(socketId, user.button);
  // 选中元素高亮
  if (user.selectedElementIds) {
    remoteSelectedElementIds.set(id, [socketId, ...]);
  }
});
```

---

## 7. 多指手势支持

### 7.1 手势状态管理 (`gesture.ts`)

```typescript
const gesture = {
  pointers: Map<number, PointerCoords>,  // 活跃指针集合
  lastCenter: { x, y } | null,           // 上次中心点
  initialScale: number | null,           // 初始缩放值
  initialDistance: number | null,        // 两指初始距离
};

export const getCenter = (pointers) => { /* 计算多指中心 */ };
export const getDistance = ([a, b]) => { /* 计算两点距离 */ };
```

### 7.2 双指缩放平移 (`App.tsx:6908-6957`)

```
pointerMove 检测到 gesture.pointers.size === 2
    ↓
计算 center = getCenter(pointers)
计算 distance = getDistance(pointers)
计算 scaleFactor = distance / initialDistance
    ↓
nextZoom = initialScale * scaleFactor
    ↓
translateCanvas({
  zoom: nextZoom,
  scrollX: scrollX + 2 * (deltaX / nextZoom),
  scrollY: scrollY + 2 * (deltaY / nextZoom),
})
```

**注意**：freedraw + penMode 下禁用双指缩放，避免误操作

---

## 8. 关键设计模式

### 8.1 闭包状态机
`onPointerMoveFromPointerDownHandler` 和 `onPointerUpFromPointerDownHandler` 通过闭包捕获 `pointerDownState`，实现：
- 状态在事件间安全传递
- 无需全局变量存储中间状态
- 每次交互的状态完全隔离

### 8.2 快照-增量模式
- `pointerDown` 时创建完整元素快照（用于回滚基准）
- `pointerMove` 时仅记录增量变更
- `pointerUp` 时将增量一次性入撤销栈

### 8.3 事件节流与批量更新
- `withBatchedUpdatesThrottled` 确保 pointerMove 60fps
- Store 层自动合并连续变更，减少撤销栈条目

### 8.4 防御式编程
- `maybeCleanupAfterMissingPointerUp` 处理指针丢失（如切换标签页）
- `setPointerCapture` 确保事件不丢失
- 元素过小自动删除，避免脏数据

---

## 9. 核心状态流转图

```
┌─────────────────────────────────────────────────────────────┐
│                     空闲状态 (idle)                         │
│  cursorButton: "up", newElement: null,                      │
│  selectedElementsAreBeingDragged: false                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼ pointerDown
┌─────────────────────────────────────────────────────────────┐
│                创建 PointerDownState 快照                   │
│  - originalElements 深拷贝所有元素                          │
│  - 检测命中元素、变换手柄                                    │
│  - 根据工具类型进入对应分支                                  │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼ pointerMove
┌─────────────────────────────────────────────────────────────┐
│                    拖拽/绘制进行中                          │
│  工具分支:                                                   │
│  ├─ Selection → dragSelectedElements()                       │
│  ├─ 绘图工具 → dragNewElement()                              │
│  ├─ Linear → handlePointDragging()                           │
│  ├─ Hand → translateCanvas()                                 │
│  └─ Eraser → handleEraser()                                  │
│                                                              │
│  状态标志:                                                   │
│  - drag.hasOccurred = true                                   │
│  - selectedElementsAreBeingDragged = true                    │
│  - newElement 持续更新                                       │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼ pointerUp
┌─────────────────────────────────────────────────────────────┐
│                      操作完成                               │
│  - actionFinalize() 收尾                                    │
│  - 检测元素有效性，过小则删除                                │
│  - store.scheduleCapture() 生成增量                         │
│  - history.record(delta) 压入撤销栈                         │
│  - 清理事件监听器                                           │
│  - 重置临时状态                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
                    回到空闲状态
```

---

## 10. 代码位置索引

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| handleCanvasPointerDown | `components/App.tsx` | 7688 |
| handleCanvasPointerMove | `components/App.tsx` | 6888 |
| handleCanvasPointerUp | `components/App.tsx` | 8148 |
| onPointerMoveFromPointerDownHandler | `components/App.tsx` | 9675 |
| onPointerUpFromPointerDownHandler | `components/App.tsx` | 10598 |
| initialPointerDownState | `components/App.tsx` | 8384 |
| dragSelectedElements | `element/src/dragElements.ts` | 35 |
| dragNewElement | `element/src/dragElements.ts` | 231 |
| History 类 | `history.ts` | 90 |
| actionFinalize | `actions/actionFinalize.tsx` | 52 |
| undo/redo action | `actions/actionHistory.tsx` | 65 |
| savePointer | `components/App.tsx` | 12906 |
| PointerDownState 类型 | `types.ts` | 853 |
| gesture 工具函数 | `gesture.ts` | 1 |
