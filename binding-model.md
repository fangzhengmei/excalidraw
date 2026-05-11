# Excalidraw 绑定关系模型分析报告

## 1. 概述

Excalidraw 中的绑定关系主要分为两类：
1. **箭头绑定**：箭头的端点可以绑定到其他图形（BindableElement）上
2. **文本绑定**：文本元素可以附着在形状或箭头内部

这两类绑定关系都采用**双向引用**机制来维护，确保图形移动、旋转、缩放或删除时，绑定关系能够正确联动。

---

## 2. 核心数据结构

### 2.1 FixedPoint（固定点）

在 `packages/element/src/types.ts:280` 中定义：

```typescript
export type FixedPoint = [number, number];
```

- 是一个归一化的坐标比例值 [x, y]，范围在 0.0-1.0 之间
- 通过 `fixedPoint[0] * element.width` 和 `fixedPoint[1] * element.height` 计算实际坐标
- 使用 `normalizeFixedPoint` 函数进行规范化，避免精确等于 0.5 导致的浮点精度问题

### 2.2 FixedPointBinding（固定点绑定）

在 `packages/element/src/types.ts:284-296` 中定义：

```typescript
export type FixedPointBinding = {
  elementId: ExcalidrawBindableElement["id"];
  fixedPoint: FixedPoint;  // 归一化的固定点位置
  mode: BindMode;           // "inside" | "orbit" | "skip"
};
```

**BindMode 说明**：
- `"inside"`：箭头端点可以完全进入图形内部，直到固定点
- `"orbit"`：箭头端点保持在图形轮廓外，自动吸附到边缘
- `"skip"`：特殊绑定模式，用于跳过某些操作

### 2.3 箭头元素的绑定字段

在 `packages/element/src/types.ts:333-341` 中定义：

```typescript
export type ExcalidrawLinearElement = _ExcalidrawElementBase & Readonly<{
  type: "line" | "arrow";
  points: readonly LocalPoint[];
  startBinding: FixedPointBinding | null;  // 起点绑定
  endBinding: FixedPointBinding | null;    // 终点绑定
  startArrowhead: Arrowhead | null;
  endArrowhead: Arrowhead | null;
}>;
```

### 2.4 可绑定元素（BindableElement）的绑定字段

在 `packages/element/src/types.ts:75-76` 中定义：

```typescript
/** other elements that are bound to this element */
boundElements: readonly BoundElement[] | null;
```

其中 `BoundElement` 是一个简单的引用结构，包含：
- `id`：被绑定元素的 ID
- `type`：被绑定元素的类型（"arrow" 或 "text"）

### 2.5 文本元素的绑定字段

在 `packages/element/src/types.ts:276-278` 中定义：

```typescript
export type ExcalidrawTextElementWithContainer = {
  containerId: ExcalidrawTextContainer["id"];
} & ExcalidrawTextElement;
```

---

## 3. 绑定关系的双向维护机制

### 3.1 双向引用结构

Excalidraw 采用**双向引用**来维护绑定关系：

**箭头绑定的双向引用**：
```
Arrow.startBinding.elementId → BindableElement.id
Arrow.endBinding.elementId → BindableElement.id
BindableElement.boundElements[].id → Arrow.id
```

**文本绑定的双向引用**：
```
TextElement.containerId → Container.id
Container.boundElements[].id → TextElement.id
```

### 3.2 维护类

在 `packages/element/src/binding.ts:2187-2408` 中定义了两个核心维护类：

#### BoundElement 类

用于处理**被绑定元素**（箭头、文本）的绑定关系：

- `unbindAffected()`：解绑受影响的可绑定元素（从 `boundElements` 中移除）
- `rebindAffected()`：重新绑定受影响的可绑定元素

#### BindableElement 类

用于处理**可绑定元素**（形状、箭头）的绑定关系：

- `unbindAffected()`：解绑受影响的被绑定元素（将 `startBinding`/`endBinding`/`containerId` 置为 null）
- `rebindAffected()`：重新绑定受影响的被绑定元素

### 3.3 访问器模式

代码中使用了访问器模式（Visitor Pattern）来遍历绑定关系：

```typescript
// 遍历可绑定元素的 boundElements
const boundElementsVisitor = (elements, element, visit) => {
  if (isBindableElement(element)) {
    const boundElements = element.boundElements?.slice() ?? [];
    boundElements.forEach(({ id }) => {
      visit(elements.get(id), "boundElements", id);
    });
  }
};

// 遍历被绑定元素的绑定关系
const bindableElementsVisitor = (elements, element, visit) => {
  const result = [];
  
  if (element.frameId) {
    result.push(visit(elements.get(id), "frameId", id));
  }
  
  if (isBoundToContainer(element)) {
    result.push(visit(elements.get(id), "containerId", id));
  }
  
  if (isArrowElement(element)) {
    if (element.startBinding) {
      result.push(visit(elements.get(id), "startBinding", id));
    }
    if (element.endBinding) {
      result.push(visit(elements.get(id), "endBinding", id));
    }
  }
  
  return result;
};
```

---

## 4. 箭头绑定的建立

### 4.1 绑定策略

在 `packages/element/src/binding.ts:87-105` 中定义了 `BindingStrategy` 类型：

```typescript
export type BindingStrategy =
  | { mode: BindMode; element: ...; focusPoint: ...; }  // 创建新绑定
  | { mode: null; }                                    // 断开绑定
  | { mode: undefined; }                               // 保持现有绑定
```

### 4.2 绑定建立的流程

1. **检测绑定目标**：通过 `getHoveredElementForBinding()` 检测鼠标位置附近的可绑定元素

2. **计算绑定模式**：
   - 鼠标在图形内部 → 可能创建 `"inside"` 绑定
   - 鼠标在图形附近但外部 → 创建 `"orbit"` 绑定
   - 按住 Alt 键 → 强制创建 `"inside"` 绑定

3. **计算 FixedPoint**：
   - 对于普通箭头：`calculateFixedPointForNonElbowArrowBinding()`
   - 对于肘形箭头：`calculateFixedPointForElbowArrowBinding()`

4. **执行绑定**：`bindBindingElement()` 函数（`binding.ts:1016-1068`）

### 4.3 bindBindingElement 核心逻辑

```typescript
export const bindBindingElement = (
  arrow: NonDeleted<ExcalidrawArrowElement>,
  hoveredElement: ExcalidrawBindableElement,
  mode: BindMode,
  startOrEnd: "start" | "end",
  scene: Scene,
  focusPoint?: GlobalPoint,
  shouldSnapToOutline = true,
): void => {
  // 1. 计算 FixedPoint
  const binding: FixedPointBinding = {
    elementId: hoveredElement.id,
    mode,
    ...calculateFixedPoint...
  };

  // 2. 更新箭头的绑定字段
  scene.mutateElement(arrow, {
    [startOrEnd === "start" ? "startBinding" : "endBinding"]: binding,
  });

  // 3. 更新目标元素的 boundElements（双向引用）
  const boundElementsMap = arrayToMap(hoveredElement.boundElements || []);
  if (!boundElementsMap.has(arrow.id)) {
    scene.mutateElement(hoveredElement, {
      boundElements: (hoveredElement.boundElements || []).concat({
        id: arrow.id,
        type: "arrow",
      }),
    });
  }
};
```

### 4.4 绑定断开

`unbindBindingElement()` 函数（`binding.ts:1070-1100`）：

```typescript
export const unbindBindingElement = (
  arrow: NonDeleted<ExcalidrawArrowElement>,
  startOrEnd: "start" | "end",
  scene: Scene,
): ExcalidrawBindableElement["id"] | null => {
  const binding = arrow[startOrEnd === "start" ? "startBinding" : "endBinding"];
  
  if (binding == null) return null;

  // 只有当两端不都绑定到同一元素时，才从目标元素移除引用
  if (!oppositeBinding || oppositeBinding.elementId !== binding.elementId) {
    scene.mutateElement(boundElement, {
      boundElements: boundElement.boundElements?.filter(
        (element) => element.id !== arrow.id,
      ),
    });
  }

  // 清除箭头的绑定字段
  scene.mutateElement(arrow, { [field]: null });
  return binding.elementId;
};
```

---

## 5. 文本绑定的建立

### 5.1 手动绑定

通过 `actionBindText` 操作（`packages/excalidraw/actions/actionBoundText.tsx:109-181`）：

**前提条件**：
- 选中 2 个元素
- 一个是文本元素
- 一个是可绑定容器（形状或箭头）
- 容器没有已绑定的文本

**绑定流程**：
```typescript
// 1. 设置文本的 containerId
scene.mutateElement(textElement, {
  containerId: container.id,
  verticalAlign: VERTICAL_ALIGN.MIDDLE,
  textAlign: TEXT_ALIGN.CENTER,
  autoResize: true,
  angle: isArrowElement(container) ? 0 : container.angle,
});

// 2. 设置容器的 boundElements
scene.mutateElement(container, {
  boundElements: (container.boundElements || []).concat({
    type: "text",
    id: textElement.id,
  }),
});

// 3. 重新计算文本位置和尺寸
redrawTextBoundingBox(textElement, container, scene);
```

### 5.2 自动创建容器

通过 `actionWrapTextInContainer` 操作（`packages/excalidraw/actions/actionBoundText.tsx:223-338`）：

- 自动为选中的文本创建一个矩形容器
- 建立双向绑定关系
- 同时迁移原文本上的箭头绑定到新容器

### 5.3 文本位置计算

`computeBoundTextPosition()` 函数（`packages/element/src/textElement.ts:222-278`）：

- 对于普通容器：根据 `textAlign` 和 `verticalAlign` 计算居中位置
- 对于箭头容器：使用 `LinearElementEditor.getBoundTextElementPosition()` 计算在箭头中间位置
- 考虑容器的旋转角度

---

## 6. 被绑元素移动时的联动

### 6.1 核心更新函数

`updateBoundElements()` 函数（`packages/element/src/binding.ts:1104-1204`）：

```typescript
export const updateBoundElements = (
  changedElement: NonDeletedExcalidrawElement,
  scene: Scene,
  options?: {
    simultaneouslyUpdated?: readonly ExcalidrawElement[];
    changedElements?: Map<string, ExcalidrawElement>;
  },
) => {
  if (!isBindableElement(changedElement)) return;

  // 遍历所有绑定到该元素的元素
  const visitor = (element: ExcalidrawElement | undefined) => {
    if (!isArrowElement(element) || element.isDeleted) return;
    if (!doesNeedUpdate(element, changedElement)) return;

    // 跳过同时被移动的元素（如整体框选移动）
    if (simultaneouslyUpdatedElementIds.has(element.id)) return;

    // 更新箭头的绑定点位置
    const updates = bindableElementsVisitor(
      elementsMap,
      element,
      (bindableElement, bindingProp) => {
        if (changedElement.id === element[bindingProp]?.elementId) {
          const point = updateBoundPoint(
            element,
            bindingProp,
            element[bindingProp],
            bindableElement,
            elementsMap,
          );
          if (point) {
            return [bindingProp === "startBinding" ? 0 : element.points.length - 1, { point }];
          }
        }
        return null;
      },
    ).filter(u => u !== null);

    // 移动箭头端点
    LinearElementEditor.movePoints(element, scene, new Map(updates), {
      moveMidPointsWithElement: 两端都绑定到同一元素时,
    });

    // 同时更新绑定的文本
    const boundText = getBoundTextElement(element, elementsMap);
    if (boundText && !boundText.isDeleted) {
      handleBindTextResize(element, scene, false);
    }
  };

  boundElementsVisitor(elementsMap, changedElement, visitor);
};
```

### 6.2 updateBoundPoint 的详细逻辑

`updateBoundPoint()` 函数（`packages/element/src/binding.ts:1746-1902`）：

1. **计算新的固定点位置**：
   ```typescript
   const focusPoint = getGlobalFixedPointForBindableElement(
     normalizeFixedPoint(binding.fixedPoint),
     bindableElement,
     elementsMap,
   );
   ```

2. **"inside" 模式**：直接使用固定点位置

3. **"orbit" 模式**：需要计算箭头端点应该吸附到的轮廓位置：
   - 构建从固定点到另一端的线段
   - 计算该线段与图形轮廓的交点
   - 应用绑定间隙（`bindingGap`）
   - 处理箭头过短的特殊情况

### 6.3 触发时机

在 `packages/element/src/dragElements.ts:139-141` 中：

```typescript
updateBoundElements(element, scene, {
  simultaneouslyUpdated: Array.from(elementsToUpdate),
});
```

- 每次拖动元素时都会调用
- `simultaneouslyUpdated` 参数用于跳过框选时整体移动的箭头

### 6.4 文本的联动

- 文本元素本身不直接监听容器移动
- 文本的位置在渲染时通过 `computeBoundTextPosition()` 动态计算
- 容器缩放时通过 `handleBindTextResize()` 调整文本换行和位置

---

## 7. 元素删除时的联动

### 7.1 核心清理函数

`fixBindingsAfterDeletion()` 函数（`packages/element/src/binding.ts:2061-2075`）：

```typescript
export const fixBindingsAfterDeletion = (
  sceneElements: readonly ExcalidrawElement[],
  deletedElements: readonly ExcalidrawElement[],
): void => {
  const elements = arrayToMap(sceneElements);

  for (const element of deletedElements) {
    // 清理被删除元素作为「被绑定元素」时的引用
    BoundElement.unbindAffected(elements, element, (element, updates) =>
      mutateElement(element, elements, updates),
    );
    // 清理被删除元素作为「可绑定元素」时的引用
    BindableElement.unbindAffected(elements, element, (element, updates) =>
      mutateElement(element, elements, updates),
    );
  }
};
```

### 7.2 删除场景

**场景 1：删除箭头（被绑定元素）**

通过 `BoundElement.unbindAffected()`：
- 遍历箭头的 `startBinding` 和 `endBinding` 指向的元素
- 从这些元素的 `boundElements` 中移除该箭头的引用

**场景 2：删除形状（可绑定元素）**

通过 `BindableElement.unbindAffected()`：
- 遍历形状的 `boundElements`
- 将绑定到该形状的所有箭头的 `startBinding`/`endBinding` 置为 null
- 将绑定到该形状的文本的 `containerId` 置为 null

### 7.3 调用位置

1. 在删除操作中（`packages/excalidraw/actions/actionDeleteSelected.tsx:278-281`）：
   ```typescript
   fixBindingsAfterDeletion(
     nextElements,
     nextElements.filter((el) => el.isDeleted),
   );
   ```

2. 在元素更新/同步中（`packages/element/src/delta.ts:1850-1854`）：
   ```typescript
   BoundElement.unbindAffected(nextElements, prevElement(), updater);
   BoundElement.unbindAffected(nextElements, nextElement(), updater);
   BindableElement.unbindAffected(nextElements, prevElement(), updater);
   BindableElement.unbindAffected(nextElements, nextElement(), updater);
   ```

---

## 8. 绑定关系的重建与同步

### 8.1 复制时的绑定修复

`fixDuplicatedBindingsAfterDuplication()` 函数（`packages/element/src/binding.ts:1989-2059`）：

当复制一组绑定元素时：
1. 更新 `boundElements` 中的引用 ID
2. 更新 `containerId` 引用
3. 更新 `startBinding.elementId` 和 `endBinding.elementId` 引用
4. 对于肘形箭头，重新计算拐点位置

### 8.2 增量同步

在 `packages/element/src/delta.ts` 中，每次元素变化时：

**删除/更新前**：`unbindAffected()` 清除旧的绑定关系

**更新后**：`rebindAffected()` 重建新的绑定关系

```typescript
// 更新时的处理
BoundElement.unbindAffected(nextElements, prevElement(), updater);
BoundElement.rebindAffected(nextElements, nextElement(), updater);

BindableElement.unbindAffected(nextElements, prevElement(), updater);
BindableElement.rebindAffected(nextElements, nextElement(), updater);
```

---

## 9. 关键设计特点

### 9.1 双向引用的优缺点

**优点**：
- 可以从任意方向快速遍历绑定关系
- 被绑定元素移动时，能高效找到所有需要更新的箭头
- 删除元素时，能高效清理所有相关引用

**缺点**：
- 需要维护双向一致性
- 每次绑定/解绑都要操作两个对象

### 9.2 FixedPoint 的设计优势

使用归一化比例而非绝对坐标：
- 图形缩放时，绑定点自动按比例缩放
- 不需要重新计算绑定关系
- 旋转时通过 `pointRotateRads` 自动处理

### 9.3 延迟计算策略

- 箭头端点位置在需要时才计算（`updateBoundPoint`）
- 文本位置在渲染/调整大小时动态计算
- 避免了每一帧都计算所有绑定位置

---

## 10. 总结

| 特性 | 实现方式 |
|------|----------|
| 绑定关系存储 | 双向引用（`startBinding`/`endBinding` ↔ `boundElements`） |
| 绑定位置记录 | FixedPoint（归一化比例 [0-1]） |
| 绑定模式 | inside（内部）/ orbit（轮廓吸附） |
| 移动联动 | `updateBoundElements()` → `updateBoundPoint()` |
| 删除清理 | `fixBindingsAfterDeletion()` → `unbindAffected()` |
| 文本绑定 | `containerId` ↔ `boundElements` |
| 文本位置 | `computeBoundTextPosition()` 动态计算 |

整个绑定系统的核心在于 **FixedPoint 机制** 和 **双向引用维护**，配合访问器模式实现了高效的绑定关系遍历和更新。
