# 选择框命中测试与多选状态管理分析报告

## 目录
1. [命中测试算法](#1-命中测试算法)
2. [多选状态管理](#2-多选状态管理)
3. [单击与框选的优先级机制](#3-单击与框选的优先级机制)

---

## 1. 命中测试算法

### 1.1 单点命中测试 (`hitElementItself`)

**核心函数**: `packages/element/src/collision.ts:119-192`

```typescript
export const hitElementItself = ({
  point,
  element,
  threshold,
  elementsMap,
  frameNameBound = null,
  overrideShouldTestInside = false,
}: HitTestArgs) => {
  // 1. AABB快速过滤（性能优化）
  const bounds = getElementBounds(element, elementsMap, true);
  const hitBounds = isPointInRotatedBounds(point, bounds, element.angle, threshold);
  
  // 2. 精确命中测试
  const hitElement = (overrideShouldTestInside ? true : shouldTestInside(element))
    ? isPointInElement(point, element, elementsMap) || 
      isPointOnElementOutline(point, element, elementsMap, threshold)
    : isPointOnElementOutline(point, element, elementsMap, threshold);
    
  // 3. 帧名称特殊处理
  const hitFrameName = frameNameBound ? isPointWithinBounds(...) : false;
  
  return hitElement || hitFrameName;
};
```

**命中测试层级**:
- **层1 (快速)**: 旋转边界框 (`isPointInRotatedBounds`) - 过滤99%不相关元素
- **层2 (精确)**: 元素内部 (`isPointInElement`) - 射线奇偶性算法
- **层3 (精确)**: 元素轮廓 (`isPointOnElementOutline`) - 距离阈值判断
- **层4 (特殊)**: 帧名称区域 - 独立边界框测试

### 1.2 选择框命中测试 (`getElementsWithinSelection`)

**核心函数**: `packages/element/src/selection.ts:90-367`

```typescript
export const getElementsWithinSelection = (
  elements,
  selection,
  elementsMap,
  excludeElementsInFrames = true,
  boxSelectionMode = "contain"
) => {
  // 1. 构建选择框几何信息
  const selectionBounds = [x1, y1, x2, y2] as Bounds;
  const selectionEdges = [top, right, bottom, left];  // 四条边界线段
  
  for (const element of elements) {
    // 2. 帧可见区域裁剪：只测试元素在帧内的可见部分
    const associatedFrame = getContainingFrame(element, elementsMap);
    if (associatedFrame && elementOverlapsWithFrame(element, associatedFrame, elementsMap)) {
      const frameAABB = getElementBounds(associatedFrame, elementsMap);
      elementAABB = [
        Math.max(elementAABB[0], frameAABB[0]),
        Math.max(elementAABB[1], frameAABB[1]),
        Math.min(elementAABB[2], frameAABB[2]),
        Math.min(elementAABB[3], frameAABB[3]),
      ] as Bounds;
    }
    
    // 3. AABB包含测试（优先，性能最高）
    if (boundsContainBounds(selectionBounds, commonAABB)) {
      if (framesInSelection && isFrameLikeElement(element)) {
        framesInSelection.add(element.id);
      }
      elementsInSelection.add(element);
      continue;
    }
    
    // 4. overlap模式：边界相交测试
    if (boxSelectionMode === "overlap" && 
        doBoundsIntersect(selectionBounds, elementAABB)) {
      // 4.1 特殊形状点测试（线性/自由绘制）
      // 4.2 线段相交测试
      const hasIntersection = selectionEdges.some(edge =>
        intersectElementWithLineSegment(element, elementsMap, edge, ...).length > 0
      );
    }
  }
  
  // 5. 帧-元素互斥过滤：帧和其包含元素不能同时选中
  if (framesInSelection) {
    elementsInSelection.forEach((element) => {
      if (element.frameId && framesInSelection.has(element.frameId)) {
        elementsInSelection.delete(element);
      }
    });
  }
  
  // 6. 组选择一致性保证
  if (boxSelectionMode === "contain") {
    // 组必须全部被包含才选中，否则全部不选中
    elementsInSelection.forEach((element) => {
      const groupId = element.groupIds.at(-1);
      const group = groupId ? groups[groupId] : null;
      if (group && !group.every((groupElement) => elementsInSelection.has(groupElement))) {
        elementsInSelection.delete(element);
      }
    });
  } else if (boxSelectionMode === "overlap") {
    // 组任意元素相交则全部选中
    Array.from(elementsInSelection).forEach((element) => {
      const groupId = element.groupIds.at(-1);
      const group = groupId ? groups[groupId] : null;
      group?.forEach((groupElement) => elementsInSelection.add(groupElement));
    });
  }
};
```

### 1.3 帧裁剪与组状态回写对最终选中集合的影响

#### 1.3.1 帧可见区域裁剪机制

**裁剪时机**: 命中测试前，`getElementsWithinSelection` 第180-201行

```typescript
// 裁剪元素边界框到帧的可见区域
const associatedFrame = getContainingFrame(element, elementsMap);
if (associatedFrame && elementOverlapsWithFrame(element, associatedFrame, elementsMap)) {
  const frameAABB = getElementBounds(associatedFrame, elementsMap);
  elementAABB = [
    Math.max(elementAABB[0], frameAABB[0]),   // 左边界取交集
    Math.max(elementAABB[1], frameAABB[1]),   // 上边界取交集
    Math.min(elementAABB[2], frameAABB[2]),   // 右边界取交集
    Math.min(elementAABB[3], frameAABB[3]),   // 下边界取交集
  ] as Bounds;
}
```

**影响**:
- **包含模式**: 元素整体在选择框内，但可见部分被帧裁剪 → 可能无法被选中
- **相交模式**: 元素边界与选择框相交，但可见部分被帧裁剪 → 可能被错误选中
- **标签文本**: 箭头绑定文本也需同样裁剪处理

#### 1.3.2 组状态回写的两阶段处理

`selectGroupsForSelectedElements` (`packages/element/src/groups.ts:65-208`) 执行两阶段回写：

**阶段1: 组ID收集** (`_selectGroups` 第91-106行)
```typescript
// 遍历所有已选元素，收集顶层组ID
for (const selectedElement of selectedElements) {
  let groupIds = selectedElement.groupIds;
  if (appState.editingGroupId) {
    // 编辑组内时裁剪嵌套层级
    const indexOfEditingGroup = groupIds.indexOf(appState.editingGroupId);
    if (indexOfEditingGroup > -1) {
      groupIds = groupIds.slice(0, indexOfEditingGroup);
    }
  }
  if (groupIds.length > 0) {
    const lastSelectedGroup = groupIds[groupIds.length - 1];
    selectedGroupIds[lastSelectedGroup] = true;
  }
}
```

**阶段2: 组员反向回写** (`_selectGroups` 第108-131行)
```typescript
// 根据收集的组ID，反向将所有组员加入选中集合
const selectedElementIdsInGroups = elements.reduce((acc, element) => {
  if (element.isDeleted) { return acc; }
  const groupId = element.groupIds.find((id) => selectedGroupIds[id]);
  if (groupId) {
    acc[element.id] = true;  // 组员全部加入选中
  }
  return acc;
}, {});
```

**关键一致性保证**:
- 单元素组自动降级：如果组内只有一个元素，不视为组选择
- 编辑组隔离：正在编辑的组不参与外层组选择回写

#### 1.3.3 组合影响的典型场景

| 场景 | 帧裁剪影响 | 组回写影响 | 最终结果 |
|-----|-----------|-----------|---------|
| **组内部分元素在帧外** | 帧外元素AABB被裁剪 | 相交模式：只要帧内元素被选中 → 整个组被选中 | 帧外元素也被选中（看似与选择框无交集） |
| **选择框只包住帧** | 帧被选中，触发帧-元素互斥 | 组包含帧时，帧选中会回写组内其他元素 | 组内帧外元素也被选中 |
| **嵌套组跨帧** | 内层元素被帧裁剪 | 当前仅支持顶层组，嵌套不处理 | 外层组决定最终选择 |

### 1.3 元素获取层级 (`getElementAtPosition`)

**核心函数**: `packages/excalidraw/components/App.tsx:5974-6031`

```typescript
private getElementAtPosition(x, y, opts) {
  // 1. 获取所有命中元素（按z-index排序）
  const allHitElements = this.getElementsAtPosition(x, y, opts);
  
  if (allHitElements.length > 1) {
    // 2. 优先选中已选中的元素（preferSelected）
    if (opts?.preferSelected) { ... }
    
    // 3. 最高z-index元素精确测试
    return hitElementItself(...) 
      ? elementWithHighestZIndex 
      : allHitElements[allHitElements.length - 2];
  }
  return allHitElements[0] || null;
}
```

---

## 2. 多选状态管理

### 2.1 状态数据结构

```typescript
interface SelectionState {
  // 选中元素集合
  selectedElementIds: Record<ExcalidrawElement["id"], true>;
  
  // 选择框元素
  selectionElement: NonDeletedExcalidrawElement | null;
  
  // 框选模式
  boxSelectionMode: "contain" | "overlap";
  
  // 组状态
  selectedGroupIds: Record<string, true>;
  editingGroupId: string | null;
  
  // 线性元素编辑器
  selectedLinearElement: LinearElementEditor | null;
}
```

### 2.2 多选状态机流程

**状态转换触发点**:
1. `pointerDown` - 初始化选择状态
2. `pointerMove` - 更新选择框范围，实时计算选中元素
3. `pointerUp` - 最终确定选择状态

**状态流转**:

```
初始状态 (无选择)
     ↓ pointerDown（空白区域）
创建 selectionElement 开始框选
     ↓ pointerMove
实时计算 elementsWithinSelection
更新 selectedElementIds（预览状态）
     ↓ pointerUp
销毁 selectionElement
固定 selectedElementIds
     ↓
最终多选状态
```

### 2.3 组选择逻辑 (`selectGroupsForSelectedElements`)

- **完全包含模式**: 组内所有元素必须都在选择框内才选中整个组
- **相交模式**: 组内任一元素与选择框相交则选中整个组
- **嵌套组处理**: 当前仅支持顶层组选择

### 2.4 帧与元素的互斥选择

**核心规则** (`excludeElementsInFramesFromSelection`):
```typescript
// 帧及其包含的元素不能同时被选中
if (element.frameId && framesInSelection.has(element.frameId)) {
  elementsInSelection.delete(element);
}
```

---

## 3. 单击与拖拽分支的决策澄清

### 3.1 三事件状态标志的时序拆解

**核心修正**: Ctrl/Cmd 分支**不仅在 pointerDown 触发**，在 pointerMove 阶段也有独立分支逻辑。三个事件的状态标志完全解耦：

| 事件 | 状态标志 | 设置时机 | 语义 |
|-----|---------|---------|-----|
| **pointerDown** | `hit.element` | 命中测试后 | 指针按下时命中的元素 |
| | `hit.wasAddedToSelection` | Ctrl分支中 | 该元素是否因Ctrl被加入选择 |
| | `withCmdOrCtrl` | pointerDown初始化 | **状态快照**，记录按下时是否按Ctrl |
| **pointerMove** | `boxSelection.hasOccurred` | 进入选择分支时 | 是否发生了框选拖动 |
| | `selectedElementsAreBeingDragged` | 进入拖拽分支时 | 是否在拖拽已选中元素 |
| **pointerUp** | （无新增） | - | 最终确认阶段 |

---

### 3.2 优先级重定义：PointerMove 阶段的 Ctrl 独立分支

#### 3.2.1 PointerDown 阶段：组穿透的前置处理

**位置**: `packages/excalidraw/components/App.tsx:8766-8794`

```typescript
// == deep selection ==
// on CMD/CTRL, drill down to hit element regardless of groups etc.
if (event[KEYS.CTRL_OR_CMD]) {
  if (event.altKey) {
    this.lassoTrail.startPath(...);
    this.setActiveTool({ type: "lasso", fromSelection: true });
    return false;
  }
  if (!this.state.selectedElementIds[hitElement.id]) {
    pointerDownState.hit.wasAddedToSelection = true;
  }
  this.setState((prevState) => ({
    ...editGroupForSelectedElement(prevState, hitElement),
    previousSelectedElementIds: this.state.selectedElementIds,
  }));
  // ⚠️ 关键：返回 false 表示未完全处理，pointerMove 阶段会继续
  return false;
}
```

**pointerDown 的实际作用**:
- ✅ 穿透组层级选中单个元素
- ✅ 设置 `editingGroupId` 进入组编辑模式
- ❌ **不阻止后续流程**：返回 `false` 允许 pointerMove 继续判断

#### 3.2.2 PointerMove 阶段：Ctrl 分支独立触发（核心修正）

**位置**: `packages/excalidraw/components/App.tsx:10477-10494`

```typescript
// 这段代码在 pointerMove 的框选分支内执行！
if (!event.shiftKey && isSomeElementSelected(elements, this.state)) {
  // ⚠️ withCmdOrCtrl 是 pointerDown 时的状态快照，不是当前按键状态！
  if (pointerDownState.withCmdOrCtrl && pointerDownState.hit.element) {
    // 独立分支：Ctrl+单击触发的特殊选择模式
    // 只保留 pointerDown 命中的单个元素，清除所有其他选择
    nextSelectedElementIds = {
      [pointerDownState.hit.element!.id]: true,
    };
  } else {
    // 普通框选: 清空之前选择
    shouldReuseSelection = false;
  }
}
```

**关键澄清**:
1. **分支触发时机**: 这段代码在 `pointerMove` 的框选分支内执行，**不是** pointerDown
2. **状态快照特性**: `withCmdOrCtrl` 记录的是 pointerDown 瞬间的按键状态，拖动过程中松开 Ctrl **不影响**此分支
3. **不等于普通单击**: 这是 `pointerMove` 阶段的独立分支，不是 pointerDown 单击逻辑的延续

---

### 3.3 完整决策流程与优先级澄清

#### 真实优先级排序（修正后）

| 优先级 | 分支 | 触发事件 | 行为特征 |
|-------|-----|---------|---------|
| **1** | **Ctrl+Alt 套索** | pointerDown | 切换工具，完全接管后续流程 |
| **2** | **拖拽已选中元素** | pointerMove | 平移拖拽，选中集合不变 |
| **3** | **Ctrl+拖动命中元素** | pointerMove | 特殊分支：只保留单个命中元素 |
| **4** | **Shift+框选（追加）** | pointerMove | 保留原有选择，追加新元素 |
| **5** | **普通框选** | pointerMove | 清空原有选择，重选框内元素 |

#### 优先级分支的代码证据

**分支2（拖拽已选中元素）优先于框选**：
```typescript
// packages/excalidraw/components/App.tsx:10130-10175
// 在 pointerMove 中，先判断是否应该拖拽已选中元素
if (
  !event.shiftKey &&
  this.scene.shouldDraggingSelectedElements(event, pointerDownState)
) {
  // 进入拖拽模式，直接调用 dragSelectedElements
  // 清除 selectionElement，设置 selectedElementsAreBeingDragged
  // 跳过后续框选逻辑
  return;
}
```

**分支3（Ctrl+拖动命中元素）的真实执行顺序**：

⚠️ **之前描述不准确**：Ctrl 分支**不会绕过** `getElementsWithinSelection`，真实执行顺序是：

```typescript
// 第1步：条件判断（第10477行）
if (!event.shiftKey && isSomeElementSelected(elements, this.state)) {
  if (pointerDownState.withCmdOrCtrl && pointerDownState.hit.element) {
    // ⚠️ 第一次 setState：立即设置只包含命中元素
    // ✅ 立即调用 selectGroupsForSelectedElements 组回写
    this.setState((prevState) =>
      selectGroupsForSelectedElements(
        {
          ...prevState,
          selectedElementIds: {
            [pointerDownState.hit.element!.id]: true,  // 只保留命中元素
          },
        },
        this.scene.getNonDeletedElements(),
        prevState,
        this,
      ),
    );
  } else {
    shouldReuseSelection = false;
  }
}

// 第2步：计算框选元素（第10499行）
// ⚠️ 这行代码总是执行，即使 Ctrl 分支已触发！
const elementsWithinSelection = this.state.selectionElement
  ? getElementsWithinSelection(...)
  : [];

// 第3步：第二次 setState，构建最终结果（第10509行）
this.setState((prevState) => {
  // 合并已有选择 + 框选元素
  const nextSelectedElementIds = {
    ...(shouldReuseSelection && prevState.selectedElementIds),
    ...elementsWithinSelection.reduce(...),
  };

  // 命中元素增删互斥处理
  if (pointerDownState.hit.element) {
    if (!elementsWithinSelection.length) {
      // 框选无结果：追加命中元素
      nextSelectedElementIds[pointerDownState.hit.element.id] = true;
    } else {
      // 框选有结果：删除命中元素（起始点元素）
      delete nextSelectedElementIds[pointerDownState.hit.element.id];
    }
  }

  // ⚠️ 第二次组回写：覆盖第一次的结果！
  return {
    ...selectGroupsForSelectedElements(...),  // 再次组回写
    // ...其他字段
  };
});
```

**关键修正**:
1. ❌ **错误认知**: Ctrl 分支直接设置 nextSelectedElementIds，绕过框选计算
2. ✅ **实际机制**: Ctrl 分支触发**第一次** setState 立即设置单个元素，但**不会 return 退出**
3. ✅ 框选计算 `getElementsWithinSelection` **总是在之后执行**
4. ✅ **第二次** setState 构建最终结果，会**覆盖**第一次的设置
5. ✅ 命中元素的增删互斥逻辑**在第二次 setState 中执行**

---

### 3.4 三个反例：为什么"Ctrl/Cmd+单击优先"是错误结论

#### 反例1：按下 Ctrl 拖动已选中的组元素

**操作步骤**:
1. 创建矩形 A 和矩形 B，编为一组
2. 框选整个组（A、B 都被选中）
3. 按住 Ctrl，鼠标放到 A 上，按住左键拖动

**错误预测**（Ctrl+单击优先）:
- Ctrl 穿透组，单独选中 A
- 拖动时只移动 A

**实际行为**:
1. ✅ pointerDown: Ctrl 生效，A 被单独选中，B 取消选中
2. ✅ pointerMove: 检测到 A 已选中 → 进入**拖拽分支**（优先级2）
3. ❌ **没有触发 Ctrl 分支**（优先级3），因为拖拽分支优先级更高
4. ✅ 最终：只移动 A，B 留在原地

**结论**: 拖拽已选中元素（优先级2）比 Ctrl 分支（优先级3）优先级更高

---

#### 反例2：先按 Ctrl，拖动过程中松开 Ctrl

**操作步骤**:
1. 画布上有元素 A、B、C（均未选中）
2. 按住 Ctrl，鼠标点击空白处开始拖动（框选）
3. 拖动过程中保持左键按下，松开 Ctrl
4. 继续拖动直到框住 A、B

**错误预测**（Ctrl+单击优先）:
- 松开 Ctrl 后应该切换到普通框选模式

**实际行为**:
1. ✅ pointerDown: 空白处点击，`hit.element = null`
2. ✅ pointerMove: 超过拖动阈值，进入框选分支
3. ✅ `withCmdOrCtrl = true`（pointerDown 时的状态快照）
4. ❌ **但由于 hit.element 为空，不触发 Ctrl 分支条件**
5. ✅ 执行普通框选逻辑，A、B 同时被选中
6. ✅ 松开 Ctrl 对结果**完全无影响**

**结论**: `withCmdOrCtrl` 是状态快照，只在 pointerDown 时记录；拖动过程中按键变化不影响已进入的分支

---

#### 反例3：Ctrl + Shift 组合键的优先级冲突

**操作步骤**:
1. 元素 A 已选中
2. 同时按住 Ctrl + Shift
3. 鼠标点击元素 B 并拖动一小段距离（框选）

**错误预测**（Ctrl+单击优先）:
- Ctrl 穿透，只选中 B

**实际行为**:
1. ✅ pointerDown: `hit.element = B`，`withCmdOrCtrl = true`，`event.shiftKey = true`
2. ✅ pointerMove: 进入框选分支
3. ⚠️ **关键判断**: `if (!event.shiftKey && ...)` —— Shift 为 true，**不进入内层判断**
4. ❌ **Ctrl 分支完全被跳过**！因为外层条件要求 `!event.shiftKey`
5. ✅ 最终：Shift 追加模式生效，A、B 同时被选中

**代码证据**:
```typescript
// 外层条件：Shift 按下时直接跳过 Ctrl 分支！
if (!event.shiftKey && isSomeElementSelected(elements, this.state)) {
  // Ctrl 分支只在这个 if 块内
  if (pointerDownState.withCmdOrCtrl && pointerDownState.hit.element) {
    // ... 永远不会执行，因为 shiftKey = true
  }
}
```

**结论**: Shift 键的优先级**高于** Ctrl 键，同时按下时 Ctrl 分支完全不执行

---

### 3.5 Ctrl/Cmd 分支路径的最小可复现反例（完整追踪）

#### 实验设置

**画布元素**:
- 矩形 A（id: `"rect-a"`），位置: (100, 100)，尺寸: 50×50
- 矩形 B（id: `"rect-b"`），位置: (200, 100)，尺寸: 50×50
- 矩形 C（id: `"rect-c"`），位置: (300, 100)，尺寸: 50×50
- **编组**: B 和 C 编为一组，组 ID: `"group-1"`

**初始状态**:
- `selectedElementIds = { "rect-a": true }` （只有 A 被选中）
- `selectedGroupIds = {}` （无组选中）
- 鼠标指针在 B 中心 (225, 125)

**操作步骤**:
1. 按住 **Ctrl** 键（保持按下状态）
2. 按住鼠标左键（pointerDown 发生在 B 上）
3. 向右拖动鼠标约 150px，使选择框刚好框住 **B 和 C**（框选范围: (200, 100) → (375, 175)）
4. 保持 Ctrl 按下，松开鼠标左键（pointerUp）

---

#### 逐步骤追踪

##### 🔴 阶段 1: pointerDown 事件
```
// 命中测试
hit.element = B (rect-b)
hit.allHitElements = [B]

// Ctrl 状态
withCmdOrCtrl = true  (pointerDown 时的状态快照)
event.altKey = false  (不触发套索)

// pointerDown 阶段的 Ctrl 组穿透
this.setState({
  selectedElementIds: { "rect-b": true },
  editingGroupId: null,
  selectedGroupIds: {},
  previousSelectedElementIds: { "rect-a": true }
})

// ⚠️ 关键：返回 false，不阻止后续 pointerMove 流程
return false
```

**pointerDown 结束时的状态**:
- `selectedElementIds = { "rect-b": true }` （组穿透后只选中 B）
- `withCmdOrCtrl = true` （状态快照已记录）
- `hit.element = B` （命中元素记录）

---

##### 🟡 阶段 2: pointerMove 事件（拖动超过阈值）

```
boxSelection.hasOccurred = true
进入框选分支
```

**子步骤 2.1: Ctrl 分支条件判断**（第 10477 行）
```typescript
if (!event.shiftKey && isSomeElementSelected(elements, this.state)) {
  // ✅ !event.shiftKey = true
  // ✅ isSomeElementSelected = true (B 已被 pointerDown 选中)
  
  if (pointerDownState.withCmdOrCtrl && pointerDownState.hit.element) {
    // ✅ withCmdOrCtrl = true
    // ✅ hit.element = B 存在
    
    // ⚠️ 第一次 setState
    this.setState((prevState) =>
      selectGroupsForSelectedElements(
        {
          ...prevState,
          selectedElementIds: { "rect-b": true },  // 只保留 B
        },
        elements,
        prevState,
        this,
      ),
    );
  }
}
```

**第一次 setState 后的中间状态**:
```
// 传入 selectGroupsForSelectedElements:
selectedElementIds = { "rect-b": true }

// 组回写逻辑:
B.groupIds = ["group-1"]
→ 找到 B 的顶层组 "group-1"
→ 回写组内所有成员

// 第一次 setState 后的结果
selectedElementIds = { "rect-b": true, "rect-c": true }  (组 B+C 都被选中)
selectedGroupIds = { "group-1": true }
editingGroupId = null
```

---

**子步骤 2.2: 计算框选命中集合**（第 10499 行，总是执行！）
```typescript
const elementsWithinSelection = this.state.selectionElement
  ? getElementsWithinSelection(...)
  : [];
```

**框选计算过程**:
- 选择框范围: (200, 100) → (375, 175)
- 命中检测:
  - B (200, 100, 50×50): ✅ 完全在选区内
  - C (300, 100, 50×50): ✅ 完全在选区内
  - A (100, 100, 50×50): ❌ 不在选区内

**框选结果**:
```
elementsWithinSelection = [B, C]  (顺序按 z-index)
```

---

**子步骤 2.3: 第二次 setState，构建最终结果**（第 10509 行）

```typescript
this.setState((prevState) => {
  // ==============================================
  // 步骤 A: 合并已有选择 + 框选元素
  // ==============================================
  const nextSelectedElementIds = {
    // shouldReuseSelection = true  (未进入 else 分支)
    ...(shouldReuseSelection && prevState.selectedElementIds),
    // prevState.selectedElementIds = { "rect-b": true, "rect-c": true }  (来自第一次 setState)
    
    ...elementsWithinSelection.reduce((acc, element) => {
      acc[element.id] = true;
      return acc;
    }, {}),
    // 展开框选结果: { "rect-b": true, "rect-c": true }
  };
  
  // 合并结果: { "rect-b": true, "rect-c": true }  (无变化)
```

---

```typescript
  // ==============================================
  // 步骤 B: 命中元素增删互斥处理 （最终裁决者！）
  // ==============================================
  if (pointerDownState.hit.element) {
    // pointerDownState.hit.element = B 存在
    
    if (!elementsWithinSelection.length) {
      // 框选无结果：追加命中元素
      nextSelectedElementIds[pointerDownState.hit.element.id] = true;
    } else {
      // ⚠️ 框选有结果：删除命中元素（起始点元素）！
      delete nextSelectedElementIds["rect-b"];
    }
  }
  
  // 互斥处理后的集合:
  // 删除 B 之前: { "rect-b": true, "rect-c": true }
  // 删除 B 之后: { "rect-c": true }
```

---

```typescript
  // ==============================================
  // 步骤 C: 第二次组回写（覆盖第一次结果！）
  // ==============================================
  return {
    ...selectGroupsForSelectedElements(
      {
        editingGroupId: prevState.editingGroupId,  // = null
        selectedElementIds: nextSelectedElementIds,  // = { "rect-c": true }
      },
      this.scene.getNonDeletedElements(),
      prevState,
      this,
    ),
    // ...其他字段
  };
});
```

**第二次组回写过程**:
```
输入: selectedElementIds = { "rect-c": true }
     C.groupIds = ["group-1"]
     → 找到顶层组 "group-1"
     → 组内包含 B 和 C
     → 回写组内所有成员

最终回写结果:
selectedElementIds = { "rect-b": true, "rect-c": true }  (B 又回来了！)
selectedGroupIds = { "group-1": true }
```

---

##### 🟢 阶段 3: pointerUp 事件
```
boxSelection.hasOccurred = true
→ 保持第二次 setState 的结果
→ 清除 selectionElement
→ 提交历史记录
```

---

#### 📊 完整状态变化追踪表

| 阶段 | selectedElementIds | 说明 |
|-----|---------------------|------|
| **初始状态** | `{ A }` | 只有 A 被选中 |
| **pointerDown 后** | `{ B }` | Ctrl 组穿透，只选中 B |
| **第一次 setState 后** | `{ B, C }` | 组回写，B 的组 C 也被加入 |
| **框选计算后** | `[B, C]` | elementsWithinSelection 结果 |
| **互斥处理后** | `{ C }` | 框选有结果，删除起始点命中的 B |
| **第二次组回写后** | `{ B, C }` | 🔄 组回写，C 的组 B 又被加回来了！ |
| **最终结果** | `{ B, C }` | pointerUp 确认 |

---

#### 🎯 关键结论

| 断言 | 结果 | 证据 |
|-----|------|------|
| Ctrl 分支绕过框选计算？ | ❌ 否 | elementsWithinSelection 总是被计算 |
| 第一次 setState 是最终结果？ | ❌ 否 | 被第二次 setState 覆盖 |
| 互斥逻辑删除 B 后就结束了？ | ❌ 否 | 第二次组回写把 B 又加回来了 |
| `withCmdOrCtrl` 是实时按键？ | ❌ 否 | pointerDown 时的状态快照 |
| Ctrl+单击 优先于框选？ | ❌ 否 | 框选结果经过组回写后覆盖了互斥删除 |

**最反直觉的发现**: 即使命中元素增删逻辑删除了 B，第二次组回写会因为 C 仍被选中而把 B 重新加回来！最终 B、C 都被选中。

---

### 3.6 最终时序确认总结

```
用户按下鼠标（pointerDown）
    ↓
┌─────────────────────────────────────────────────────┐
│  1. 命中测试 → hit.element                           │
│  2. Ctrl+Alt → 套索工具（终止流程）                  │
│  3. Ctrl → 组穿透选中单个元素（但不终止流程）       │
│  4. 记录 withCmdOrCtrl = true/false 状态快照        │
└─────────────────────────────────────────────────────┘
    ↓（拖动超过阈值 pointerMove）
┌─────────────────────────────────────────────────────┐
│  1. 已选中元素且未按Shift？                          │
│     ├─ 是 → 拖拽分支（优先级2）→ 结束               │
│     └─ 否 → 进入框选分支                             │
│              设置 boxSelection.hasOccurred = true   │
│              创建 selectionElement                   │
│     ↓                                               │
│  2. 未按 Shift 且已有选中？                          │
│     ├─ 是 + withCmdOrCtrl + 有命中元素 → Ctrl 分支  │
│     │                  只保留单个命中元素            │
│     ├─ 是 + 其他 → shouldReuseSelection = false     │
│     │                清空原有选择                    │
│     └─ 否 → 保留原有选择（追加模式）                 │
│     ↓                                               │
│  3. 调用 getElementsWithinSelection()               │
│  4. 二次互斥处理：框选有结果则删除起始命中元素       │
│  5. selectGroupsForSelectedElements() 组回写        │
└─────────────────────────────────────────────────────┘
    ↓（松开鼠标 pointerUp）
┌─────────────────────────────────────────────────────┐
│  1. boxSelection.hasOccurred ?                      │
│     ├─ true → 保持框选结果                          │
│     └─ false → 纯单击，处理锁定/高亮等               │
│  2. 清除 selectionElement                           │
│  3. 提交历史记录                                    │
└─────────────────────────────────────────────────────┘
```

**核心修正点总结**:
1. ✅ Ctrl 分支在 **pointerMove** 阶段触发，不是 pointerDown
2. ✅ `withCmdOrCtrl` 是**状态快照**，不是实时按键
3. ✅ 拖拽已选中元素优先级 **高于** Ctrl 分支
4. ✅ Shift 键按下时 **完全跳过** Ctrl 分支（外层 `!event.shiftKey` 条件）
5. ✅ Ctrl 分支**不会绕过** `getElementsWithinSelection` 计算
6. ✅ 两次 setState 机制：第一次被第二次**覆盖**
7. ✅ 最终裁决者是**命中元素增删互斥逻辑**（第10521-10528行）
8. ❌ "Ctrl/Cmd+单击优先" 是不成立的简化结论

---

## 关键设计洞察

### 1. 性能优化策略
- **双层命中测试**: AABB快速过滤 + 精确几何计算
- **缓存机制**: `hitElementItself` 缓存相同点的命中结果
- **提前终止**: 线段相交测试找到第一个交点即返回

### 2. 一致性保证
- **组选择原子性**: 组要么全选，要么不选
- **帧-元素互斥**: 帧和其内部元素不同时选中
- **绑定文本处理**: 箭头绑定文本作为箭头的一部分参与命中

### 3. 用户体验考量
- **已选中元素优先**: `preferSelected` 标志确保拖动已选中元素时的操作一致性
- **阈值自适应**: 命中阈值随缩放级别动态调整
- **边界框命中**: 已选中元素使用边界框命中，便于拖动调整

### 4. 状态机设计优点
- **单一事实来源**: `selectedElementIds` 是选择状态的唯一权威
- **引用保持**: `makeNextSelectedElementIds` 确保无变化时保持引用一致
- **预览机制**: 拖动过程中实时更新选择预览，`pointerUp` 才最终确定
