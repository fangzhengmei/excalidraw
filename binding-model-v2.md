# Excalidraw 绑定关系模型分析报告 (v2)

## 1. 核心数据结构

### 1.1 FixedPoint（固定点比例）

**定义位置**：`packages/element/src/types.ts:280`

```typescript
export type FixedPoint = [number, number];
```

**关键修正：取值范围**

FixedPoint 是一个比例值，但**不限于 [0, 1] 范围**，可以是任意实数。

**越界场景来源**：

1. **orbit 模式的轮廓吸附计算**
   - 当箭头绑定点在形状轮廓外时，`bindPointToSnapToElementOutline()` 会计算吸附点
   - 对于肘形箭头，`avoidRectangularCorner()` 会将点移动到角点外
   - 例如：矩形左边缘外的点会有 `fixedPoint[0] < 0`

2. **`getBindingGap` 导致的偏移**
   - 绑定间隙 = `BASE_BINDING_GAP + strokeWidth / 2`
   - 这个间隙会使固定点落在形状边界外
   - 代码位置：`packages/element/src/binding.ts:117-125`

3. **旋转变换后的计算**
   - 当形状有角度时，通过 `pointRotateRads` 旋转
   - 旋转后再计算比例，可能产生越界值

4. **手工编辑的 JSON 数据**
   - 外部导入或手工修改的场景数据

**使用时的处理**：

代码中**没有 clamp 限制**，直接使用原始值：

```typescript
// packages/element/src/binding.ts:2410-2425
export const getGlobalFixedPointForBindableElement = (
  fixedPointRatio: FixedPoint,
  element: ExcalidrawBindableElement,
  elementsMap: ElementsMap,
): GlobalPoint => {
  const [fixedX, fixedY] = normalizeFixedPoint(fixedPointRatio);
  // 直接计算，不做 clamp
  return pointRotateRads(
    pointFrom(
      element.x + element.width * fixedX,  // fixedX 可以 < 0 或 > 1
      element.y + element.height * fixedY, // fixedY 可以 < 0 或 > 1
    ),
    elementCenterPoint(element, elementsMap),
    element.angle,
  );
};
```

`normalizeFixedPoint` 只做两件事：
1. 验证格式有效性
2. 避免精确等于 0.5（防止浮点精度导致的方向判断问题）

**示例**：
- `[0.5, 0.5]` → 形状中心
- `[-0.1, 0.5]` → 左边缘外 10% 宽度处
- `[1.2, 0.5]` → 右边缘外 20% 宽度处
- `[0.5, -0.2]` → 上边缘外 20% 高度处

---

### 1.2 FixedPointBinding（固定点绑定）

**定义位置**：`packages/element/src/types.ts:284-296`

```typescript
export type FixedPointBinding = {
  elementId: ExcalidrawBindableElement["id"];
  fixedPoint: FixedPoint;  // 比例值，可越界
  mode: BindMode;          // "inside" | "orbit" | "skip"
};
```

**BindMode 说明**：

| 模式 | 行为 | 典型场景 |
|------|------|----------|
| `"inside"` | 箭头端点指向固定点，不吸附到轮廓 | 箭头两端都在形状内部 |
| `"orbit"` | 箭头端点吸附到形状轮廓，固定点只是"方向锚点" | 箭头连接到形状边缘 |
| `"skip"` | 特殊模式，用于跳过某些操作 | 复杂绑定场景 |

### 1.3 双向引用结构

**箭头绑定**：
```
Arrow.startBinding.elementId → BindableElement.id
Arrow.endBinding.elementId → BindableElement.id
BindableElement.boundElements[].id → Arrow.id (type: "arrow")
```

**文本绑定**：
```
TextElement.containerId → Container.id
Container.boundElements[].id → TextElement.id (type: "text")
```

---

## 2. 主线一：建立绑定

### 2.1 入口函数概览

| 场景 | 入口函数 | 文件位置 |
|------|----------|----------|
| 新建箭头时初始绑定 | `bindOrUnbindBindingElement` (pointer down) | `App.tsx:9410` |
| 拖动箭头端点时 | `bindOrUnbindBindingElement` | `actionFinalize.tsx:111` |
| 释放鼠标/键盘移动后 | `bindOrUnbindBindingElements` | `App.tsx:5413, 11563` |
| 手动绑定文本 | `actionBindText.perform` | `actionBoundText.tsx:140` |
| 文本自动创建容器 | `actionWrapTextInContainer.perform` | `actionBoundText.tsx:234` |

### 2.2 完整调用链：新建箭头绑定

**入口**：`App.handlePointerDown()` → 新建箭头元素

```
App.tsx:9408-9425
  ↓
1. insertNewElement(element)  // 插入新箭头
  ↓
2. bindOrUnbindBindingElement(
     element,
     new Map([[0, { point: [0,0], isDragging: false }]]),
     sceneX, sceneY,
     scene, appState,
     { newArrow: true, initialBinding: true }
   )
     ↓
   binding.ts:145-222
     ↓
   2.1 getBindingStrategyForDraggingBindingElementEndpoints(...)
        ↓
      计算绑定策略：
      - 检测鼠标位置附近的可绑定元素
      - 判断是 inside 还是 orbit 模式
      - 返回 { mode, element, focusPoint }
     ↓
   2.2 bindOrUnbindBindingElementEdge(arrow, strategy, "start"|"end", scene, ...)
        ↓
      根据 strategy.mode:
      - mode === null → unbindBindingElement(...)
      - mode !== undefined → bindBindingElement(...)
```

### 2.3 核心函数：bindBindingElement

**位置**：`packages/element/src/binding.ts:1016-1068`

**执行步骤**：

```
1. 计算 FixedPoint
   ├─ 肘形箭头: calculateFixedPointForElbowArrowBinding()
   │   └─ bindPointToSnapToElementOutline() → 吸附到轮廓
   │       └─ 可能产生越界的 fixedPoint（如 < 0 或 > 1）
   └─ 普通箭头: calculateFixedPointForNonElbowArrowBinding()
       └─ 将鼠标位置转换为比例值
           fixedPointX = (point.x - element.x) / element.width
           fixedPointY = (point.y - element.y) / element.height

2. 更新箭头的绑定字段
   scene.mutateElement(arrow, {
     startBinding: {
       elementId: hoveredElement.id,
       fixedPoint: [...],  // 可能越界
       mode: "inside" | "orbit"
     }
   })

3. 更新目标元素的 boundElements（建立双向引用）
   const boundElementsMap = arrayToMap(hoveredElement.boundElements || []);
   if (!boundElementsMap.has(arrow.id)) {
     scene.mutateElement(hoveredElement, {
       boundElements: [...].concat({
         id: arrow.id,
         type: "arrow"
       })
     })
   }
```

**双向引用变化**：

```
绑定前：
  Arrow.startBinding = null
  Shape.boundElements = null 或不含此箭头

绑定后：
  Arrow.startBinding = { elementId: Shape.id, fixedPoint: [...], mode: "orbit" }
  Shape.boundElements = [..., { id: Arrow.id, type: "arrow" }]
```

### 2.4 完整调用链：手动绑定文本

**入口**：`actionBindText.perform()` → 用户选中文本 + 容器后执行

```
actionBoundText.tsx:140-180
  ↓
1. 识别 textElement 和 container
  ↓
2. 更新文本的 containerId（建立正向引用）
   scene.mutateElement(textElement, {
     containerId: container.id,
     verticalAlign: VERTICAL_ALIGN.MIDDLE,
     textAlign: TEXT_ALIGN.CENTER,
     autoResize: true,
     angle: isArrowElement(container) ? 0 : container.angle
   })

3. 更新容器的 boundElements（建立反向引用）
   scene.mutateElement(container, {
     boundElements: (container.boundElements || []).concat({
       type: "text",
       id: textElement.id
     })
   })

4. 重新计算文本位置和尺寸
   redrawTextBoundingBox(textElement, container, scene)
   └─ computeBoundTextPosition() → 计算居中位置
   └─ 考虑容器旋转角度
```

**双向引用变化**：

```
绑定前：
  TextElement.containerId = null
  Container.boundElements = null 或不含此文本

绑定后：
  TextElement.containerId = Container.id
  Container.boundElements = [..., { id: TextElement.id, type: "text" }]
```

### 2.5 调用链图示

```
用户操作
   │
   ├─ 新建箭头
   │   └─ App.handlePointerDown()
   │       └─ bindOrUnbindBindingElement(..., { newArrow: true })
   │           └─ bindBindingElement()
   │               ├─ 计算 FixedPoint（可能越界）
   │               ├─ 设置 Arrow.startBinding
   │               └─ 设置 Shape.boundElements
   │
   ├─ 拖动箭头端点
   │   └─ actionFinalize.execute()
   │       └─ bindOrUnbindBindingElement(..., { newArrow: false })
   │           └─ getBindingStrategyForDraggingBindingElementEndpoints()
   │               └─ 判断是绑定/解绑/保持
   │
   └─ 手动绑定文本
       └─ actionBindText.perform()
           ├─ 设置 Text.containerId
           └─ 设置 Container.boundElements
```

---

## 3. 主线二：移动联动

### 3.1 入口函数概览

| 场景 | 入口函数 | 文件位置 |
|------|----------|----------|
| 鼠标拖动元素 | `dragSelectedElements()` | `dragElements.ts:139` |
| 键盘方向键移动 | `updateBoundElements()` | `App.tsx:5197` |
| 文本编辑后更新 | `updateBoundElements()` | `App.tsx:5765` |
| 复制元素后 | `updateBoundElements()` | `App.tsx:10315` |
| 裁剪图片后 | `updateBoundElements()` | `App.tsx:12546` |

### 3.2 完整调用链：鼠标拖动形状

**入口**：`App.handlePointerMove()` → 检测到拖动

```
App.tsx:10160-10167
  ↓
dragSelectedElements(
  pointerDownState,
  selectedElements,
  dragOffset,
  scene,
  snapOffset,
  gridSize
)
  ↓
dragElements.ts:123-141
  ↓
1. 更新形状位置
   updateElementCoords(pointerDownState, element, scene, adjustedOffset)
   └─ scene.mutateElement(element, { x: nextX, y: nextY })

2. 更新绑定的文本位置（直接跟随移动）
   const textElement = getBoundTextElement(element, elementsMap)
   if (textElement) {
     updateElementCoords(pointerDownState, textElement, scene, adjustedOffset)
   }

3. 核心：更新绑定的箭头
   updateBoundElements(element, scene, {
     simultaneouslyUpdated: Array.from(elementsToUpdate)
   })
     ↓
   binding.ts:1104-1204
```

### 3.3 核心函数：updateBoundElements

**位置**：`packages/element/src/binding.ts:1104-1204`

**执行步骤**：

```
1. 检查是否为可绑定元素
   if (!isBindableElement(changedElement)) return;

2. 构建 elementsMap（可能包含已变更的元素）
   let elementsMap = scene.getNonDeletedElementsMap();
   if (options?.changedElements) {
     // 合并本次变更
     options.changedElements.forEach(e => elementsMap.set(e.id, e));
   }

3. 遍历所有绑定到该元素的元素
   boundElementsVisitor(elementsMap, changedElement, visitor)
   └─ 遍历 changedElement.boundElements
       └─ 对每个被绑定元素执行 visitor()

4. visitor 函数核心逻辑
   visitor = (element) => {
     // 跳过非箭头或已删除元素
     if (!isArrowElement(element) || element.isDeleted) return;
     
     // 检查是否需要更新（绑定关系是否涉及当前元素）
     if (!doesNeedUpdate(element, changedElement)) return;
     
     // 跳过同时被移动的元素（如框选整体移动）
     if (simultaneouslyUpdatedElementIds.has(element.id)) return;
     
     // 计算箭头端点的新位置
     const updates = bindableElementsVisitor(
       elementsMap, element,
       (bindableElement, bindingProp) => {
         if (changedElement.id === element[bindingProp]?.elementId) {
           // 关键：重新计算端点位置
           const point = updateBoundPoint(
             element,
             bindingProp,           // "startBinding" | "endBinding"
             element[bindingProp],  // FixedPointBinding
             bindableElement,       // 目标形状
             elementsMap
           );
           if (point) {
             return [pointIdx, { point }];  // pointIdx = 0 或 points.length-1
           }
         }
         return null;
       }
     ).filter(u => u !== null);
     
     // 移动箭头端点
     LinearElementEditor.movePoints(element, scene, new Map(updates), {
       moveMidPointsWithElement: 两端都绑定到同一元素时
     });
     
     // 同时更新箭头上绑定的文本
     const boundText = getBoundTextElement(element, elementsMap);
     if (boundText && !boundText.isDeleted) {
       handleBindTextResize(element, scene, false);
     }
   }
```

### 3.4 核心函数：updateBoundPoint

**位置**：`packages/element/src/binding.ts:1746-1902`

**执行步骤**：

```
1. 计算新的焦点位置（FixedPoint → GlobalPoint）
   const focusPoint = getGlobalFixedPointForBindableElement(
     normalizeFixedPoint(binding.fixedPoint),  // 只做格式校验，不 clamp
     bindableElement,
     elementsMap
   );
   // 即使 fixedPoint 越界（如 [-0.1, 0.5]），也会正确计算到形状外

2. "inside" 模式：直接使用焦点位置
   if (binding.mode === "inside") {
     return LinearElementEditor.createPointAt(arrow, elementsMap, focusPoint[0], focusPoint[1], null);
   }

3. "orbit" 模式：吸附到形状轮廓
   // 获取另一端的位置或绑定点
   const { element: otherBindable, focusPoint: otherFocusPoint } = extractBinding(...);
   const otherArrowPoint = LinearElementEditor.getPointAtIndexGlobalCoordinates(...);
   
   // 构建线段：当前焦点 → 另一端
   const intersector = lineSegment(focusPoint, otherFocusPointOrArrowPoint);
   
   // 计算线段与形状轮廓的交点（考虑 bindingGap）
   const outlinePoint = intersectElementWithLineSegment(
     bindableElement, elementsMap, intersector, getBindingGap(...)
   ).sort(...)[0];
   
   // 处理特殊情况：箭头过短、端点在另一形状内等
   // ...
   
   return LinearElementEditor.createPointAt(
     arrow, elementsMap,
     outlinePoint?.[0] || focusPoint[0],
     outlinePoint?.[1] || focusPoint[1],
     null
   );
```

### 3.5 双向引用变化

**移动时，绑定关系本身不改变，只更新位置**：

```
移动前：
  Shape.x = 100, Shape.y = 100
  Arrow.startBinding = { elementId: Shape.id, fixedPoint: [0.5, 0.5], mode: "orbit" }
  Arrow.points[0] = [0, 0]  // 局部坐标
  Shape.boundElements = [{ id: Arrow.id, type: "arrow" }]

移动后（Shape 右移 100）：
  Shape.x = 200, Shape.y = 100
  Arrow.startBinding = 不变（仍指向同一个 Shape）
  Arrow.points[0] = 更新后的局部坐标
  Shape.boundElements = 不变（仍包含此箭头）
```

### 3.6 文本的联动特殊性

**绑定到形状的文本**：
- 在 `dragSelectedElements()` 中直接跟随移动（第 2 步）
- 不需要 `updateBoundElements` 处理

**绑定到箭头的文本**：
- 箭头端点更新后，调用 `handleBindTextResize()`
- 文本位置在 `computeBoundTextPosition()` 中动态计算

### 3.7 调用链图示

```
用户拖动形状
   │
   └─ dragSelectedElements()
       │
       ├─ 1. updateElementCoords(Shape) → 移动形状
       │
       ├─ 2. updateElementCoords(BoundText) → 直接移动绑定文本
       │
       └─ 3. updateBoundElements(Shape)
           │
           └─ boundElementsVisitor(Shape)
               │
               ├─ 对每个绑定的 Arrow：
               │   │
               │   ├─ 跳过同时被选中的元素
               │   │
               │   └─ bindableElementsVisitor(Arrow)
               │       │
               │       └─ updateBoundPoint(Arrow, "startBinding", ...)
               │           │
               │           ├─ getGlobalFixedPointForBindableElement()
               │           │   └─ 使用 fixedPoint（可能越界）计算位置
               │           │
               │           └─ "orbit" 模式：吸附到轮廓
               │               └─ intersectElementWithLineSegment()
               │
               └─ LinearElementEditor.movePoints(Arrow)
                   │
                   └─ 同时更新箭头绑定的文本
                       └─ handleBindTextResize()
```

---

## 4. 主线三：删除清理

### 4.1 入口函数概览

| 场景 | 入口函数 | 文件位置 |
|------|----------|----------|
| 键盘删除 (Backspace/Delete) | `fixBindingsAfterDeletion()` | `actionDeleteSelected.tsx:278` |
| 文本编辑为空时删除 | `fixBindingsAfterDeletion()` | `App.tsx:5797` |
| Delta 同步（协作/历史） | `BoundElement.unbindAffected()` + `BindableElement.unbindAffected()` | `delta.ts:1850-1854` |

### 4.2 完整调用链：键盘删除

**入口**：`actionDeleteSelected.perform()`

```
actionDeleteSelected.tsx:275-281
  ↓
1. 执行删除
   let { elements: nextElements, appState: nextAppState } = 
     deleteSelectedElements(elements, appState, app);
   // 标记 isDeleted = true，但不物理删除

2. 清理绑定关系
   fixBindingsAfterDeletion(
     nextElements,
     nextElements.filter(el => el.isDeleted)
   )
     ↓
   binding.ts:2061-2075
```

### 4.3 核心函数：fixBindingsAfterDeletion

**位置**：`packages/element/src/binding.ts:2061-2075`

```typescript
export const fixBindingsAfterDeletion = (
  sceneElements: readonly ExcalidrawElement[],
  deletedElements: readonly ExcalidrawElement[],
): void => {
  const elements = arrayToMap(sceneElements);

  for (const element of deletedElements) {
    // 清理：被删除元素作为「被绑定元素」时的引用
    // 例如：删除箭头 → 需要从目标形状的 boundElements 中移除
    BoundElement.unbindAffected(elements, element, (element, updates) =>
      mutateElement(element, elements, updates),
    );
    
    // 清理：被删除元素作为「可绑定元素」时的引用
    // 例如：删除形状 → 需要将绑定箭头的 startBinding/endBinding 置 null
    BindableElement.unbindAffected(elements, element, (element, updates) =>
      mutateElement(element, elements, updates),
    );
  }
};
```

### 4.4 场景一：删除箭头（被绑定元素）

**类**：`BoundElement` → `unbindAffected()`

**位置**：`packages/element/src/binding.ts:2193-2226`

**执行逻辑**：

```
对于被删除的 Arrow：
  └─ bindableElementsVisitor(elements, Arrow, callback)
      └─ 遍历 Arrow 的绑定关系：
          ├─ frameId（如果有）
          ├─ containerId（如果有，文本绑定到箭头）
          ├─ startBinding.elementId（箭头起点绑定的形状）
          └─ endBinding.elementId（箭头终点绑定的形状）
      
      对每个绑定目标 Shape：
        └─ boundElementsVisitor(elements, Shape, innerCallback)
            └─ 遍历 Shape.boundElements
                └─ 如果找到 Arrow.id
                    └─ 从 Shape.boundElements 中移除
```

**双向引用变化**：

```
删除前：
  Arrow.startBinding = { elementId: Shape.id, ... }
  Shape.boundElements = [..., { id: Arrow.id, type: "arrow" }]

删除后（Arrow.isDeleted = true）：
  Arrow.startBinding = 不变（但 Arrow 已被标记删除）
  Shape.boundElements = 已移除 Arrow.id
```

### 4.5 场景二：删除形状（可绑定元素）

**类**：`BindableElement` → `unbindAffected()`

**位置**：`packages/element/src/binding.ts:2309-2338`

**执行逻辑**：

```
对于被删除的 Shape：
  └─ boundElementsVisitor(elements, Shape, callback)
      └─ 遍历 Shape.boundElements：
          ├─ 类型为 "arrow" 的元素
          └─ 类型为 "text" 的元素
      
      对每个被绑定元素：
        └─ bindableElementsVisitor(elements, BoundElement, innerCallback)
            └─ 检查该元素的绑定关系是否指向 Shape
                ├─ 如果是 Arrow：
                │   ├─ startBinding.elementId === Shape.id → startBinding = null
                │   └─ endBinding.elementId === Shape.id → endBinding = null
                │
                └─ 如果是 Text：
                    └─ containerId === Shape.id → containerId = null
```

**双向引用变化**：

```
删除前：
  Shape.boundElements = [
    { id: Arrow.id, type: "arrow" },
    { id: Text.id, type: "text" }
  ]
  Arrow.startBinding = { elementId: Shape.id, ... }
  Text.containerId = Shape.id

删除后（Shape.isDeleted = true）：
  Shape.boundElements = 不变（但 Shape 已被标记删除）
  Arrow.startBinding = null  ← 已清理
  Text.containerId = null    ← 已清理
```

### 4.6 Delta 同步的特殊处理

**位置**：`packages/element/src/delta.ts:1845-1880`

```
// 删除元素时
BoundElement.unbindAffected(nextElements, prevElement(), updater);
BoundElement.unbindAffected(nextElements, nextElement(), updater);
BindableElement.unbindAffected(nextElements, prevElement(), updater);
BindableElement.unbindAffected(nextElements, nextElement(), updater);

// 更新元素时（先解绑旧的，再绑定新的）
BoundElement.unbindAffected(nextElements, prevElement(), updater);
BoundElement.rebindAffected(nextElements, nextElement(), updater);
BindableElement.unbindAffected(nextElements, prevElement(), updater);
BindableElement.rebindAffected(nextElements, nextElement(), updater);
```

### 4.7 调用链图示

```
用户按 Delete 键
   │
   └─ actionDeleteSelected.perform()
       │
       ├─ 1. deleteSelectedElements()
       │   └─ 标记元素 isDeleted = true
       │
       └─ 2. fixBindingsAfterDeletion(deletedElements)
           │
           └─ 对每个被删除元素：
               │
               ├─ a) BoundElement.unbindAffected(DeletedElement)
               │   │
               │   └─ 场景：删除箭头
               │       │
               │       └─ 遍历箭头的 startBinding/endBinding
               │           │
               │           └─ 从目标形状的 boundElements 中移除
               │
               └─ b) BindableElement.unbindAffected(DeletedElement)
                   │
                   └─ 场景：删除形状
                       │
                       └─ 遍历形状的 boundElements
                           │
                           ├─ 对每个 Arrow：startBinding/endBinding = null
                           │
                           └─ 对每个 Text：containerId = null
```

---

## 5. 关键设计总结

### 5.1 FixedPoint 越界设计

| 方面 | 说明 |
|------|------|
| **允许越界** | 比例值可以是任意实数，不限于 [0, 1] |
| **越界来源** | orbit 模式轮廓吸附、绑定间隙、旋转变换 |
| **使用方式** | 直接 `element.x + element.width * fixedPoint[0]` 计算 |
| **不做 clamp** | 保持用户原始意图，即使点在形状外 |

### 5.2 三条主线对比

| 主线 | 入口 | 核心函数 | 双向引用变化 |
|------|------|----------|--------------|
| **建立绑定** | 新建箭头/拖动端点/手动操作 | `bindBindingElement()` / `actionBindText` | 建立双向引用 |
| **移动联动** | 拖动/键盘移动/编辑 | `updateBoundElements()` → `updateBoundPoint()` | 引用不变，位置更新 |
| **删除清理** | Delete 键/编辑为空 | `fixBindingsAfterDeletion()` → `unbindAffected()` | 清除双向引用 |

### 5.3 双向引用维护机制

```
绑定操作：
  Arrow.startBinding ←→ Shape.boundElements
  Text.containerId ←→ Container.boundElements

移动操作：
  引用关系保持不变，只更新坐标计算

删除操作（清理方向）：
  删除 Arrow → 从 Shape.boundElements 移除（BoundElement.unbindAffected）
  删除 Shape → 将 Arrow.startBinding 置 null（BindableElement.unbindAffected）
```

### 5.4 核心文件速查

| 功能 | 文件 | 关键函数 |
|------|------|----------|
| 绑定核心逻辑 | `packages/element/src/binding.ts` | `bindBindingElement`, `unbindBindingElement`, `updateBoundElements`, `updateBoundPoint` |
| 绑定关系维护类 | `packages/element/src/binding.ts` | `BoundElement`, `BindableElement` |
| 拖动时调用 | `packages/element/src/dragElements.ts` | `dragSelectedElements` |
| 文本绑定操作 | `packages/excalidraw/actions/actionBoundText.tsx` | `actionBindText`, `actionWrapTextInContainer` |
| 文本位置计算 | `packages/element/src/textElement.ts` | `computeBoundTextPosition`, `getBoundTextElement` |
| 删除时清理 | `packages/excalidraw/actions/actionDeleteSelected.tsx` | `fixBindingsAfterDeletion` 调用处 |
| 类型定义 | `packages/element/src/types.ts` | `FixedPoint`, `FixedPointBinding` |
| 主应用调用 | `packages/excalidraw/components/App.tsx` | 各种入口调用点 |
