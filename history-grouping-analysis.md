# 撤销重做历史分组策略分析

## 一、核心架构概述

Excalidraw 的撤销重做系统基于 **增量式 delta 快照** 机制实现，核心组件包括：

| 组件 | 职责 | 位置 |
|------|------|------|
| `History` | 管理 undo/redo 栈，执行撤销重做操作 | `packages/excalidraw/history.ts` |
| `Store` | 状态快照管理、delta 计算、增量发射 | `packages/excalidraw/element/src/store.ts` |
| `StoreDelta` | 封装元素与应用状态的变化增量 | `packages/excalidraw/element/src/store.ts` |
| `HistoryDelta` | 历史专用增量，继承自 StoreDelta | `packages/excalidraw/history.ts` |

---

## 二、操作批次划分策略

### 2.1 按用户交互粒度划分

每个用户操作可能产生 **多个历史条目**，典型操作分解：

| 用户操作 | 历史条目数 | 条目中记录的内容 |
|---------|-----------|-----------------|
| 创建矩形 | 2 | 1. 创建元素<br>2. 取消选择 |
| 创建矩形 + 删除 | 6 | 3 次创建 + 2 次选择 + 1 次删除 |
| 双击进入组编辑 | 2 | 1. 组选择<br>2. 进入编辑模式 |
| 修改元素颜色 | 1 | 颜色属性变化 |

**示例：三个矩形创建 + 删除的历史栈分解**

```
undoStack[0] = 创建 rect1
undoStack[1] = 选择 rect1
undoStack[2] = 创建 rect2
undoStack[3] = 选择 rect2
undoStack[4] = 创建 rect3
undoStack[5] = 多选 [rect2, rect3]
undoStack[6] = 删除选中元素
```

### 2.2 选择变化作为独立边界

**选择状态变化总是独立记录**，这是历史分组的关键特性：

```typescript
// store.ts:813-840 - getObservedAppState 定义了需要观察的应用状态
const getObservedAppState = (appState: AppState): ObservedAppState => {
  return {
    name: appState.name,
    editingGroupId: appState.editingGroupId,
    viewBackgroundColor: appState.viewBackgroundColor,
    selectedElementIds: appState.selectedElementIds,  // 选择状态
    selectedGroupIds: appState.selectedGroupIds,      // 组选择状态
    selectedLinearElement: appState.selectedLinearElement,
    croppingElementId: appState.croppingElementId,
    activeLockedId: appState.activeLockedId,
    lockedMultiSelections: appState.lockedMultiSelections,
  };
};
```

**设计意图**：选择状态是用户交互的重要上下文，撤销时需要精确恢复到之前的选择状态。

---

## 三、状态边界保留机制

### 3.1 三种捕获更新动作（CaptureUpdateAction）

这是历史分组的 **核心控制机制**：

```typescript
export const CaptureUpdateAction = {
  /**
   * 立即可撤销
   * - 用于大多数本地更新（除拖拽等临时操作）
   * - 立即进入 undo/redo 栈
   * - 更新快照
   */
  IMMEDIATELY: "IMMEDIATELY",

  /**
   * 永不撤销
   * - 用于远程更新或场景初始化
   * - 永不进入 undo/redo 栈
   * - 更新快照
   */
  NEVER: "NEVER",

  /**
   * 最终可撤销
   * - 用于异步多步过程中的中间更新
   * - 不立即进入历史栈，下一次 IMMEDIATELY 时会被合并
   * - 不更新快照
   */
  EVENTUALLY: "EVENTUALLY",
} as const;
```

### 3.2 动作优先级与调度

```typescript
// store.ts:391-406 - getScheduledMacroAction
private getScheduledMacroAction() {
  if (this.scheduledMacroActions.has(CaptureUpdateAction.IMMEDIATELY)) {
    // IMMEDIATELY 优先级最高
    return CaptureUpdateAction.IMMEDIATELY;
  } else if (this.scheduledMacroActions.has(CaptureUpdateAction.NEVER)) {
    return CaptureUpdateAction.NEVER;
  } else {
    return CaptureUpdateAction.EVENTUALLY;
  }
}
```

### 3.3 快照更新边界

| 动作类型 | 更新快照？ | 进入历史？ | 典型场景 |
|---------|-----------|-----------|---------|
| IMMEDIATELY | ✅ 是 | ✅ 是 | 用户主动操作（创建、删除、修改属性） |
| NEVER | ✅ 是 | ❌ 否 | 远程协作更新、场景初始化 |
| EVENTUALLY | ❌ 否 | ⏳ 延迟 | 拖拽过程中、自由绘制中、异步加载 |

---

## 四、撤销重做执行机制

### 4.1 迭代跳过无可见变化的条目

执行 undo/redo 时，系统会 **自动跳过不产生可见变化的历史条目**：

```typescript
// history.ts:179-219 - perform 方法中的迭代逻辑
while (historyDelta) {
  [nextElements, nextAppState, containsVisibleChange] = 
    historyDelta.applyTo(nextElements, nextAppState, prevSnapshot);
  
  // ... 应用 delta ...
  
  if (containsVisibleChange) {
    break;  // 遇到可见变化，停止迭代
  }
  
  historyDelta = pop();  // 继续弹出下一个条目
}
```

**关键行为**：
- 纯选择变化在某些情况下可能被判断为"无可见变化"
- 系统会连续跳过多个条目，直到找到产生可见变化的条目
- 被跳过的条目仍然会被推入相反方向的栈中（undo→redo，redo→undo）

### 4.2 Redo 栈的清空策略

```typescript
// history.ts:127-131 - record 方法中的 redo 栈清空逻辑
if (!historyDelta.elements.isEmpty()) {
  // 只有当元素发生变化时才清空 redo 栈
  // 纯应用状态变化（如点击取消选择）不会丢失 redo 历史
  this.redoStack.length = 0;
}
```

**重要特性**：
- ✅ **元素变化**：清空 redo 栈（符合用户预期）
- ✅ **纯应用状态变化**：不清空 redo 栈（可以保留重做历史）

示例场景：
```
1. 创建 rect1, rect2        undoStack=[创建1, 选择1, 创建2, 选择2]  redoStack=[]
2. 撤销 2 次                undoStack=[创建1, 选择1]                redoStack=[创建2, 选择2]
3. 点击空白处取消选择       undoStack=[创建1, 选择1, 取消选择]      redoStack=[创建2, 选择2]  ✅ 未清空
4. 重做                     undoStack=[创建1, 选择1, 取消选择, 创建2, 选择2]  redoStack=[]
```

---

## 五、历史条目内部结构

### 5.1 HistoryDelta 组成

```typescript
class HistoryDelta extends StoreDelta {
  elements: ElementsDelta;  // 元素变化（新增、删除、更新）
  appState: AppStateDelta;  // 应用状态变化
  
  applyTo(elements, appState, snapshot): [SceneElementsMap, AppState, boolean]
  // 返回值第三个参数：是否产生可见变化
}
```

### 5.2 增量的反向应用

```typescript
// history.ts:245-248 - push 时自动反转 delta
private static push(stack: HistoryDelta[], entry: HistoryDelta) {
  const inversedEntry = HistoryDelta.inverse(entry);
  return stack.push(inversedEntry);
}
```

**设计要点**：
- undo 栈中的条目是"反向操作"
- 从 undo 栈弹出应用后，再反转推入 redo 栈
- 反之亦然，形成对称的双向操作

---

## 六、分组策略的权衡分析

### ✅ 优点

1. **精确的状态恢复**：每个交互步骤都被独立记录，撤销时能精确恢复
2. **协作友好**：基于增量的设计天然支持多人协作
3. **性能优化**：EVENTUALLY 避免了拖拽等连续操作产生大量历史条目
4. **用户体验优化**：纯选择变化不清空 redo 栈，保留重做可能性

### ⚠️ 潜在问题

1. **历史栈膨胀**：选择变化作为独立条目，导致历史条目数量较多
   - 三个元素的创建 + 删除可能产生 6-7 个条目
   
2. **用户预期差异**：用户可能期望"一次撤销"回退到"上一个有意义的状态"，而不是精确的每一步

3. **无变化迭代**：多次撤销可能需要跳过多个无可见变化的条目才能看到实际效果

### 🔄 可能的优化方向

```
策略 1：操作合并
- 将连续的选择变化与后续操作合并
- 例如：选择元素 + 修改颜色 → 合并为单个条目

策略 2：语义分组
- 在历史条目中标记"语义组"
- 撤销时按语义组回退，而不是单个条目

策略 3：用户可控粒度
- 提供"精细撤销"和"粗粒度撤销"两种模式
```

---

## 七、关键代码位置索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 历史栈核心逻辑 | `packages/excalidraw/history.ts` | 90-250 |
| 捕获动作类型定义 | `packages/excalidraw/element/src/store.ts` | 38-69 |
| Store 提交逻辑 | `packages/excalidraw/element/src/store.ts` | 183-201 |
| 动作优先级调度 | `packages/excalidraw/element/src/store.ts` | 391-406 |
| 观察的应用状态 | `packages/excalidraw/element/src/store.ts` | 813-840 |
| 快照克隆逻辑 | `packages/excalidraw/element/src/store.ts` | 761-811 |
| 撤销迭代跳过逻辑 | `packages/excalidraw/history.ts` | 179-219 |
| Redo 栈清空策略 | `packages/excalidraw/history.ts` | 127-131 |

---

## 八、总结

Excalidraw 的历史分组策略采用 **"精确记录 + 智能跳过"** 的混合模式：

1. **分组原则**：基于用户交互的原子性，每个选择变化都是独立边界
2. **状态保留**：通过三种捕获动作精确控制哪些变化进入历史、哪些更新快照
3. **执行策略**：撤销重做时自动跳过无可见变化的条目
4. **特殊处理**：纯应用状态变化不清空 redo 栈，优化用户体验

这种设计在 **协作支持**、**精确恢复** 和 **性能** 之间取得了平衡，虽然历史条目数量较多，但保证了状态变化的完整性和可追溯性。
