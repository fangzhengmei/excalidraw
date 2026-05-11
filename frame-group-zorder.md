# Excalidraw 画板渲染协同机制分析

## 1. Frame 容器剪裁机制

### 1.1 剪裁判定逻辑

Frame 剪裁由 `packages/excalidraw/element/src/frame.ts:907-965` 中的 `shouldApplyFrameClip` 函数控制：

```typescript
export const shouldApplyFrameClip = (
  element: ExcalidrawElement,
  frame: ExcalidrawFrameLikeElement,
  appState: StaticCanvasAppState,
  elementsMap: ElementsMap,
  checkedGroups?: Map<string, boolean>,
) => {
  if (!appState.frameRendering || !appState.frameRendering.clip) {
    return false;
  }

  // 1. 元素自身与 Frame 相交或包含 Frame 时需要剪裁
  const shouldClipElementItself =
    isElementIntersectingFrame(element, frame, elementsMap) ||
    isElementContainingFrame(element, frame, elementsMap);

  if (shouldClipElementItself) {
    for (const groupId of element.groupIds) {
      checkedGroups?.set(groupId, true);
    }
    return true;
  }

  // 2. 元素在 Frame 外，但属于有成员在 Frame 内的组，也需要剪裁
  if (
    !shouldClipElementItself &&
    element.groupIds.length > 0 &&
    !elementsAreInFrameBounds([element], frame, elementsMap)
  ) {
    let shouldClip = false;

    if (!appState.selectedElementsAreBeingDragged) {
      // 非拖拽状态下，检查元素是否属于该 Frame
      shouldClip = element.frameId === frame.id;
      for (const groupId of element.groupIds) {
        checkedGroups?.set(groupId, shouldClip);
      }
    } else {
      // 拖拽状态下，执行更复杂的 inFrame 检查
      shouldClip = isElementInFrame(element, elementsMap, appState, {
        targetFrame: frame,
        checkedGroups,
      });
    }

    for (const groupId of element.groupIds) {
      checkedGroups?.set(groupId, shouldClip);
    }

    return shouldClip;
  }

  return false;
};
```

### 1.2 剪裁执行流程

在 `packages/excalidraw/renderer/staticScene.ts:132-156` 中定义 `frameClip` 函数：

```typescript
export const frameClip = (
  frame: ExcalidrawFrameLikeElement,
  context: CanvasRenderingContext2D,
  renderConfig: StaticCanvasRenderConfig,
  appState: StaticCanvasAppState,
) => {
  context.translate(frame.x + appState.scrollX, frame.y + appState.scrollY);
  context.beginPath();
  if (context.roundRect) {
    context.roundRect(
      0,
      0,
      frame.width,
      frame.height,
      FRAME_STYLE.radius / appState.zoom.value,
    );
  } else {
    context.rect(0, 0, frame.width, frame.height);
  }
  context.clip(); // 创建剪裁区域
  context.translate(
    -(frame.x + appState.scrollX),
    -(frame.y + appState.scrollY),
  );
};
```

渲染流程（`staticScene.ts:316-370`）：
```
1. 保存 Canvas 上下文 (save)
2. 检查是否需要 Frame 剪裁 (shouldApplyFrameClip)
3. 如需要，执行 frameClip 创建剪裁路径
4. 渲染元素 (renderElement)
5. 渲染关联的绑定文本
6. 恢复 Canvas 上下文 (restore)
```

### 1.3 Frame 成员判定

`isElementInFrame` 函数（`frame.ts:813-905`）负责判断元素是否在 Frame 内：
- 检查元素是否被选中且在拖拽中
- 检查组内元素是否有重叠
- 处理 Frame 嵌套和包含关系

## 2. Group 组语义机制

### 2.1 组选择机制

在 `packages/excalidraw/element/src/groups.ts` 中定义：

```typescript
export const selectGroupsForSelectedElements = (() => {
  // ... 缓存逻辑
  
  const _selectGroups = (
    selectedElements: readonly NonDeleted<ExcalidrawElement>[],
    elements: readonly NonDeleted<ExcalidrawElement>[],
    appState: Pick<AppState, "selectedElementIds" | "editingGroupId">,
    prevAppState: InteractiveCanvasAppState,
  ) => {
    const selectedGroupIds: Record<GroupId, boolean> = {};
    
    // 1. 收集选中元素所在的组
    for (const selectedElement of selectedElements) {
      let groupIds = selectedElement.groupIds;
      if (appState.editingGroupId) {
        // 在编辑组内时，只考虑该组内的层级
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

    // 2. 收集组内所有元素
    const groupElementsIndex: Record<GroupId, string[]> = {};
    const selectedElementIdsInGroups = elements.reduce(
      (acc: Record<string, true>, element) => {
        if (element.isDeleted) return acc;
        const groupId = element.groupIds.find((id) => selectedGroupIds[id]);
        if (groupId) {
          acc[element.id] = true;
          if (!Array.isArray(groupElementsIndex[groupId])) {
            groupElementsIndex[groupId] = [element.id];
          } else {
            groupElementsIndex[groupId].push(element.id);
          }
        }
        return acc;
      },
      {},
    );

    // 3. 处理单元素组（不视为组）
    for (const groupId of Object.keys(groupElementsIndex)) {
      if (groupElementsIndex[groupId].length < 2) {
        if (selectedGroupIds[groupId]) {
          selectedGroupIds[groupId] = false;
        }
      }
    }

    return {
      editingGroupId: appState.editingGroupId,
      selectedGroupIds,
      selectedElementIds: makeNextSelectedElementIds(
        { ...appState.selectedElementIds, ...selectedElementIdsInGroups },
        prevAppState,
      ),
    };
  };
})();
```

### 2.2 组内元素获取

```typescript
export const getElementsInGroup = (
  elements: ElementsMapOrArray,
  groupId: string,
) => {
  const elementsInGroup: ExcalidrawElement[] = [];
  for (const element of elements.values()) {
    if (isElementInGroup(element, groupId)) {
      elementsInGroup.push(element);
    }
  }
  return elementsInGroup;
};
```

### 2.3 组编辑模式

- `editingGroupId` 标识当前正在编辑的组
- 进入组编辑后，可以单独选择和编辑组内元素
- 组编辑支持嵌套结构

### 2.4 组与 Frame 的交互

在 Frame 剪裁和成员判定中，组作为整体处理：
- 组内只要有一个元素在 Frame 内，整个组都可能被剪裁
- 拖拽组时，所有成员的 Frame 归属会统一更新

## 3. 锁定元素事件穿透机制

### 3.1 锁定状态过滤

在 `packages/excalidraw/components/App.tsx:6048-6056` 中：

```typescript
.filter(
  (element) =>
    (opts?.includeLockedElements || !element.locked) &&
    (opts?.includeBoundTextElement ||
      !(isTextElement(element) && element.containerId)),
)
```

### 3.2 锁定元素的交互限制

在拖拽处理中（`App.tsx:9955-9963`）：
```typescript
if (
  selectedElements.length > 0 &&
  selectedElements.every((element) => element.locked)
) {
  return; // 所有选中元素都锁定，不执行拖拽
}
```

### 3.3 命中测试中的处理

锁定元素仍可被命中测试检测到，但在选择和操作时会被过滤：
- 渲染时正常显示
- 命中测试（`hitElementItself`）正常工作
- 但在选择和操作阶段会被过滤掉

## 4. 叠层重排算法（z-index）

### 4.1 核心移动函数

在 `packages/excalidraw/element/src/zindex.ts` 中定义：

```typescript
// 逐位移动
export const moveOneLeft = (...) => shiftElementsByOne(..., "left", ...);
export const moveOneRight = (...) => shiftElementsByOne(..., "right", ...);

// 移动到两端
export const moveAllLeft = (...) => shiftElementsAccountingForFrames(..., "left", shiftElementsToEnd);
export const moveAllRight = (...) => shiftElementsAccountingForFrames(..., "right", shiftElementsToEnd);
```

### 4.2 上下文感知的移动

`shiftElementsAccountingForFrames` 函数处理 Frame 边界：

```typescript
function shiftElementsAccountingForFrames(
  allElements: readonly ExcalidrawElement[],
  appState: AppState,
  direction: "left" | "right",
  shiftFunction: (...)
) {
  // 1. 区分普通元素和 Frame 内的子元素
  // 2. Frame 子元素在 Frame 内独立排序
  // 3. 普通元素在全局上下文排序
  
  // 处理 Frame 内的子元素
  for (const [frameId, children] of frameChildrenSets) {
    nextElements = shiftFunction(allElements, appState, direction, frameId, children);
  }

  // 处理普通元素
  return shiftFunction(nextElements, appState, direction, null, regularElements);
}
```

### 4.3 目标位置计算

`getTargetIndex` 函数（`zindex.ts:197-303`）考虑：
- 元素删除状态
- 组编辑上下文（组内移动不跨出组）
- Frame 边界（Frame 内移动不超出 Frame）
- 绑定文本的整体移动

### 4.4 分组移动

`toContiguousGroups` 函数将不连续的选中索引分成连续组：
- 每组独立移动
- 保持组内相对顺序
- 处理组间间隙

## 5. 四者协同工作机制

### 5.1 渲染流程协同

```
元素渲染
  ↓
获取元素所在 Frame（getContainingFrame）
  ↓
检查是否需要剪裁（shouldApplyFrameClip）
  ↓
  ├─ 检查元素自身与 Frame 几何关系
  ├─ 检查组内元素与 Frame 关系
  └─ 考虑拖拽状态
  ↓
执行 Frame 剪裁（frameClip）
  ↓
渲染元素（renderElement）
  ↓
考虑元素锁定状态（opacity 调整）
  ↓
按 z-index 顺序叠加
```

### 5.2 交互选择协同

```
指针命中测试
  ↓
从后向前遍历所有元素
  ↓
执行几何命中测试（hitElementItself）
  ↓
过滤锁定元素（除非 includeLockedElements）
  ↓
检查组选择（selectGroupsForSelectedElements）
  ↓
处理 Frame 与子元素互斥（excludeElementsInFramesFromSelection）
  ↓
返回最终选中元素
```

### 5.3 叠层调整协同

叠层调整时考虑：
- **Frame 约束**：Frame 内元素只能在 Frame 内调整层级
- **组整体性**：组内元素保持相对顺序一起移动
- **锁定过滤**：锁定元素不会被移动（除非全部选中）
- **绑定文本**：绑定文本随宿主元素一起移动

### 5.4 拖拽过程中的协同

拖拽元素时的完整流程：
1. 命中测试获取起始元素
2. 扩展到组和绑定文本
3. 检查锁定状态（全部锁定则取消）
4. 拖拽过程中动态更新 Frame 成员关系
5. 根据 Frame 上下文调整渲染剪裁
6. 结束拖拽后同步元素 z-index

## 6. 关键数据结构与算法复杂度

### 6.1 元素组层级结构

每个元素的 `groupIds` 是一个数组，支持嵌套组：
- `groupIds[0]` 是最内层组
- `groupIds[groupIds.length - 1]` 是最外层组
- 空数组表示不在任何组中

### 6.2 Frame 成员关系

- `element.frameId` 指向所属 Frame
- Frame 元素本身的 `frameId` 为 null
- 嵌套 Frame 通过父子关系链维护

### 6.3 算法复杂度

- **命中测试**：O(n)，n 为元素数量
- **组选择**：O(n)，遍历所有元素
- **Frame 成员判定**：O(g)，g 为组内元素数
- **z-index 移动**：O(n)，需要重建元素数组
- **渲染剪裁**：O(1)  per element（Canvas 硬件加速）