# Excalidraw 协作下撤销范围界定

## 核心问题

多人协作时，用户 A 执行撤销操作：

1. **撤销范围如何界定？** — 只撤销 A 自己的操作，还是影响所有人？
2. **哪些状态会被还原？** — 元素属性、绑定关系、分组、Frame 从属关系？
3. **与远程操作冲突时如何处理？** — 不同元素、同元素不同属性、同属性冲突？

---

## 规则表：协作冲突下的判定

### 判定说明

| 判定结果 | 含义 |
|---------|------|
| **保留远程** | 远程用户的修改保持不变，本地撤销不影响 |
| **覆盖远程** | 远程用户的修改被本地撤销覆盖（重做时可能恢复） |
| **跳过** | 相关状态被过滤掉，不产生可见变化，撤销继续往下找 |

---

### 规则 1：普通标量属性

| 属性类型 | 示例 | 协作冲突判定 | 代码分支 | 测试用例 |
|---------|------|-------------|---------|---------|
| **独立标量属性** | `backgroundColor`, `x`, `y`, `strokeColor`, `opacity`, `width`, `height`, `angle`, `isDeleted`, `fillStyle`, `strokeWidth`, `roughness`, `strokeStyle`, `roundness` | 按属性独立处理 | `delta.ts:1340-1341` | `history.test.tsx:2169` |
| **同元素同属性冲突** | 如 A 改 `backgroundColor: red`, B 改 `backgroundColor: yellow` | **覆盖远程** → 撤销到历史起点 | `delta.ts:1340-1341` | `history.test.tsx:2205` |

**判定规则：**

```
如果本地历史记录包含该属性：
  → 用历史记录的 deleted 端值覆盖当前值（可能覆盖远程修改）
否则：
  → 远程修改保留
```

**代码分支解析：**

`packages/element/src/delta.ts:1333-1342`

```typescript
for (const key of Object.keys(partial) as Array<keyof typeof partial>) {
  // do not update following props:
  // - `boundElements`, as it is a reference value which is postprocessed to contain only deleted/inserted keys
  switch (key) {
    case "boundElements":
      latestPartial[key] = partial[key];
      break;
    default:
      latestPartial[key] = element[key];  // ← 用最新值更新 inserted 端
  }
}
```

**关键理解：** 撤销时应用的是历史记录的 `deleted` 端，而 `applyLatestChanges` 只更新 `inserted` 端。所以：

```
历史记录 = { deleted: 操作前, inserted: 操作后 }

撤销时：
  1. applyLatestChanges 更新 inserted 为当前最新值（含远程修改）
  2. 应用反向增量：当前值 → deleted 值
  
如果远程修改了同一属性：
  - 当前值 = 远程值
  - deleted 值 = 本地操作前的值
  - 结果：远程值 → 本地操作前的值（覆盖远程）

如果远程修改了不同属性：
  - 历史记录中没有该属性
  - 结果：不影响，远程修改保留
```

**测试用例佐证：**

`packages/excalidraw/tests/history.test.tsx:2169-2203`（同元素不同属性 - 保留远程）

```typescript
// 本地修改 backgroundColor
// 远程修改 strokeColor
// 撤销 → backgroundColor 恢复，strokeColor 保留远程值
```

`packages/excalidraw/tests/history.test.tsx:2205-2367`（同元素同属性 - 覆盖远程）

```typescript
// 本地修改 backgroundColor: transparent → red
// 远程修改 backgroundColor: red → yellow
// 撤销 → yellow → transparent（覆盖远程的 yellow）
```

---

### 规则 2：数组属性

| 属性类型 | 示例 | 协作冲突判定 | 代码分支 | 测试用例 |
|---------|------|-------------|---------|---------|
| **`groupIds`** | 分组关系数组 | **覆盖远程** → 重做时恢复 | `delta.ts:1340-1341` | `history.test.tsx:2371` |
| **`points`** (线性元素) | 自由绘制/箭头的点数组 | **覆盖远程** → 重做时恢复 | `delta.ts:2025-2027` | `history.test.tsx:2425` |

**判定规则：**

```
数组属性不做合并，直接用历史记录的值覆盖当前值
```

**代码分支解析：**

`groupIds` — `packages/element/src/delta.ts:1340-1341`

```typescript
for (const key of Object.keys(partial)) {
  default:
    latestPartial[key] = element[key];  // 完全替换，不合并
}
```

`points` — `packages/element/src/delta.ts:2025-2027`（注释说明）

```typescript
// don't diff the points as:
// - we can't ensure the multiplayer order consistency without fractional index on each point
// - we prefer to not merge the points, as it might just lead to unexpected / incosistent results
//
// 不做 points 差异计算，因为：
// - 每个点没有分数索引，无法保证多人协作时的顺序一致性
// - 不合并可能导致意外/不一致的结果
```

`packages/element/src/delta.ts:2042-2046`

```typescript
if (!Delta.isDifferent(deletedPoints, insertedPoints)) {
  // delete the points from delta if there is no difference
  // otherwise leave them as they were captured due to consistency
  Reflect.deleteProperty(deleted, "points");
  Reflect.deleteProperty(inserted, "points");
}
```

**关键理解：**

```
groupIds: ["A", "B"] 的远程修改
  ↓ 本地撤销分组 A
  ↓
结果：groupIds: []（远程的 B 也被覆盖）
  ↓ 重做
  ↓
结果：groupIds: ["A", "B"]（远程的 B 恢复）
```

**测试用例佐证：**

`packages/excalidraw/tests/history.test.tsx:2371-2423`（groupIds 覆盖）

```typescript
// 本地：groupIds: ["A"]
// 远程：groupIds: ["A", "B"]
// 撤销 → groupIds: []（远程的 B 被覆盖）
// 重做 → groupIds: ["A", "B"]（远程的 B 恢复）
```

`packages/excalidraw/tests/history.test.tsx:2425-2503`（points 覆盖）

```typescript
// 本地：2 个点
// 远程：5 个点
// 撤销 → 2 个点（远程的 3 个点被覆盖）
// 重做 → 5 个点（远程的点恢复）
```

---

### 规则 3：绑定关系

| 关系类型 | 相关属性 | 协作冲突判定 | 代码分支 | 测试用例 |
|---------|---------|-------------|---------|---------|
| **容器 ↔ 文本** | `boundElements` (容器), `containerId` (文本) | **保留远程** → 最新绑定优先 | `delta.ts:1337-1339` | `history.test.tsx:4005`, `4111` |
| **箭头 ↔ 目标** | `startBinding`, `endBinding` (箭头) | **保留远程** → 目标移动后重新计算 | `history.test.tsx:5106` |
| **Frame ↔ 子元素** | `frameId` (子元素) | **视情况** → Frame 存在则绑定，否则跳过 | `history.test.tsx:5221` |

**判定规则：**

```
1. boundElements 不被 applyLatestChanges 更新 → 保留历史记录中的值
2. 远程新增的绑定 → 在重做/撤销后由后处理保留
3. Frame 被远程删除 → 子元素重做时不重新绑定
```

**代码分支解析：**

`packages/element/src/delta.ts:1336-1339`

```typescript
switch (key) {
  case "boundElements":
    latestPartial[key] = partial[key];  // ← 不更新，保留历史值
    break;
  default:
    latestPartial[key] = element[key];
}
```

`packages/element/src/delta.ts:2023`

```typescript
Delta.diffArrays(deleted, inserted, "boundElements", (x) => x.id);
// 对 boundElements 做数组差异计算（合并而非覆盖）
```

**关键理解：**

```
容器恢复流程：
1. applyLatestChanges 不更新 boundElements
2. 后处理时调用 mergeArrays 合并 boundElements
3. 结果：远程新增的绑定被保留，本地绑定恢复
4. 如果有冲突（同一容器绑定多个文本）→ 最新绑定优先
```

**测试用例佐证：**

`packages/excalidraw/tests/history.test.tsx:4005-4109`（容器恢复时保留远程新绑定）

```typescript
// 本地创建容器
// 远程绑定文本 A 到容器
// 撤销 → 容器删除，文本 A 解绑
// 远程：绑定新文本 B 到容器，恢复容器
// 重做 → 容器恢复
//   - 远程文本 B 的绑定保留
//   - 旧文本 A 不重新绑定
```

`packages/excalidraw/tests/history.test.tsx:5106-5195`（目标移动后箭头重做重新计算）

```typescript
// 箭头绑定到矩形 A 和 B
// 撤销箭头
// 远程移动矩形 B 到新位置
// 重做箭头 → 箭头点被重新计算，指向新位置
```

`packages/excalidraw/tests/history.test.tsx:5221-5304`（Frame 删除后不重新绑定）

```typescript
// 矩形在 Frame 内
// 撤销两次（移出 Frame + 删除矩形）
// 远程删除 Frame
// 重做两次 → 矩形恢复，但 frameId = null
```

---

### 规则 4：AppState 观察字段

| 字段类型 | 具体字段 | 协作冲突判定 | 代码分支 | 测试用例 |
|---------|---------|-------------|---------|---------|
| **独立状态** | `name`, `viewBackgroundColor` | **保留远程** → 不在历史记录中则不影响 | `store.ts:1006-1032` | - |
| **元素引用** | `selectedElementIds`, `selectedGroupIds`, `selectedLinearElement`, `editingGroupId`, `croppingElementId` | **跳过** → 元素被远程删除则过滤 | `delta.ts:770-837` | `history.test.tsx:2715`, `2805`, `2902`, `3050` |
| **其他** | `activeLockedId`, `lockedMultiSelections` | **保留远程** → TODO: 需完善可见性检查 | `delta.ts:838-858` | - |

**判定规则：**

```
1. 独立状态：按普通属性处理，不在历史记录中则保留远程
2. 元素引用：
   - 如果引用的元素存在 → 正常恢复
   - 如果引用的元素被远程删除 → 过滤掉该引用，继续找下一条历史记录
```

**代码分支解析：**

`packages/element/src/store.ts:1006-1032`（观察字段定义）

```typescript
export const getObservedAppState = (appState): ObservedAppState => {
  const observedAppState = {
    name: appState.name,
    editingGroupId: appState.editingGroupId,
    viewBackgroundColor: appState.viewBackgroundColor,
    selectedElementIds: appState.selectedElementIds,
    selectedGroupIds: appState.selectedGroupIds,
    croppingElementId: appState.croppingElementId,
    activeLockedId: appState.activeLockedId,
    lockedMultiSelections: appState.lockedMultiSelections,
    selectedLinearElement: appState.selectedLinearElement
      ? {
          elementId: appState.selectedLinearElement.elementId,
          isEditing: !!appState.selectedLinearElement.isEditing,
        }
      : null,
  };
  return observedAppState;
};
```

`packages/element/src/delta.ts:770-775`（过滤选中元素）

```typescript
case "selectedElementIds":
  nextAppState[key] = AppStateDelta.filterSelectedElements(
    nextAppState[key],
    nextElements,
    visibleDifferenceFlag,
  );
  break;
```

`packages/element/src/delta.ts:873-897`（filterSelectedElements 实现）

```typescript
private static filterSelectedElements(selectedElementIds, elements, visibleDifferenceFlag) {
  const ids = Object.keys(selectedElementIds);
  
  const nextSelectedElementIds = { ...selectedElementIds };
  
  for (const id of ids) {
    const element = elements.get(id);
    
    if (element && !element.isDeleted) {
      visibleDifferenceFlag.value = true;  // 可见，不跳过
    } else {
      delete nextSelectedElementIds[id];   // 元素被删除，过滤掉
    }
  }
  
  return nextSelectedElementIds;
}
```

**关键理解：**

```
历史记录迭代逻辑（history.ts:178-218）：

while (historyDelta) {
  [nextElements, nextAppState, containsVisibleChange] = 
    historyDelta.applyTo(...);
  
  if (containsVisibleChange) {
    break;  // 找到可见变化，停止
  }
  
  historyDelta = pop();  // 没有可见变化，继续找下一条
}
```

```
场景：用户选中元素 A、B、C，然后远程删除 A、B

撤销选中状态时：
  1. filterSelectedElements 过滤掉 A、B
  2. 剩下 C 的选中状态
  3. 如果 C 也被删除 → containsVisibleChange = false
  4. 继续撤销下一条历史记录
```

**测试用例佐证：**

`packages/excalidraw/tests/history.test.tsx:2715-2803`（选中元素被远程删除）

```typescript
// 创建 3 个元素：rect1, rect2, rect3
// 依次选中：rect1 → [rect2, rect3]
// undoStack.length = 3
// 当前选中：rect2, rect3

// 远程删除 rect2, rect3
// 撤销 → 跳过选中 rect2/rect3（已删除），跳到选中 rect1
// undoStack.length = 1（跳过了 2 条）
```

`packages/excalidraw/tests/history.test.tsx:2805-2900`（分组选择被远程删除）

`packages/excalidraw/tests/history.test.tsx:2902-2972`（编辑组被远程删除）

`packages/excalidraw/tests/history.test.tsx:3050-3148`（线性元素编辑器被远程删除）

---

### 规则 5：远程删除元素的边界

| 场景 | 协作冲突判定 | 代码分支 | 测试用例 |
|-----|-------------|---------|---------|
| **元素被远程删除** | **跳过可见变更** → 属性变化仍应用 | `delta.ts:1715-1717` | `history.test.tsx:2584` |
| **多个元素被远程删除** | **部分生效** → 有效元素的变更被处理 | `delta.ts:1715-1717` | `history.test.tsx:2638` |
| **元素删除后被远程恢复** | **重新分发 Delta** | `delta.ts:1383-1385` | `history.test.tsx:2506` |

**判定规则：**

```
1. 元素已删除且撤销不恢复它 → 跳过可见变更（但属性变化仍应用）
2. 部分元素有效 → 只处理有效元素的变更
3. 元素被远程恢复 → isDeleted 变化导致 Delta 重新分发
```

**代码分支解析：**

`packages/element/src/delta.ts:1711-1728`（checkForVisibleDifference）

```typescript
private static checkForVisibleDifference(element, partial) {
  if (element.isDeleted && partial.isDeleted !== false) {
    // when it's deleted and partial is not false, it cannot end up with a visible change
    return false;  // ← 跳过可见变更
  }
  
  if (element.isDeleted && partial.isDeleted === false) {
    return true;   // ← 恢复元素，可见
  }
  
  if (element.isDeleted === false && partial.isDeleted) {
    return true;   // ← 删除元素，可见
  }
  
  return Delta.isRightDifferent(element, partial);  // ← 可见元素的任何属性变化
}
```

`packages/element/src/delta.ts:1383-1385`（重新分发 Delta）

```typescript
return ElementsDelta.create(added, removed, updated, {
  shouldRedistribute: true,  // redistribute the deltas as `isDeleted` could have been updated
});
```

**关键理解：**

```
场景：创建矩形 → 设置红色 → 远程删除（同时改成黄色）

历史记录：
  1. 创建：transparent → red
  2. 颜色：red → red（实际是创建完成后的状态）

远程操作：
  - isDeleted: true
  - backgroundColor: yellow

撤销：
  1. 第一条历史（颜色修改）：
     - element.isDeleted = true
     - partial.isDeleted = undefined（不是 false）
     - checkForVisibleDifference = false → 跳过
     - 但 backgroundColor 从 yellow → transparent 仍应用
  2. 第二条历史（创建）：
     - partial.isDeleted = true → 删除
     - checkForVisibleDifference = true（因为删除已删除的元素？）
     - 实际上会继续直到找到可见变更
```

**测试用例佐证：**

`packages/excalidraw/tests/history.test.tsx:2584-2636`（单个元素远程删除）

```typescript
// 创建矩形 → 设置红色（undoStack.length = 2）
// 远程删除并改成黄色
// 撤销 → undoStack 从 2 → 0（跳过了颜色修改）
// 结果：元素保持删除，但 backgroundColor = transparent
```

`packages/excalidraw/tests/history.test.tsx:2638-2713`（多个元素远程删除）

```typescript
// rect1: 创建
// rect2: 创建 → 改红色
// rect3: 创建 → 移动
// undoStack.length = 5

// 远程删除 rect2, rect3
// 撤销 → 跳过移动 rect3（已删除），跳到选中 rect1
// undoStack.length = 1（跳过了 4 条）
```

`packages/excalidraw/tests/history.test.tsx:2506-2582`（元素删除后被远程恢复）

```typescript
// 创建矩形 → 删除（isDeleted: true）
// 远程恢复（isDeleted: false）并改成黄色
// 撤销删除 → 因为 isDeleted 相同，removed → updated
// 结果：元素保持存在，但颜色被撤销
```

---

## 规则速查表

### 快速判定流程

```
给定：用户 A 执行撤销，用户 B 做了远程修改

1. 确定修改的状态类型
   ├── 普通标量属性（backgroundColor, x, y 等）
   │   └── 检查：历史记录包含该属性吗？
   │       ├── 是 → 覆盖远程（应用历史 deleted 值）
   │       └── 否 → 保留远程
   ├── 数组属性（groupIds, points）
   │   └── 直接覆盖远程（重做时恢复）
   ├── 绑定关系（boundElements, containerId）
   │   └── 保留远程（最新绑定优先）
   ├── Frame 从属（frameId）
   │   └── Frame 存在则绑定，否则跳过
   ├── AppState 元素引用（selectedElementIds 等）
   │   └── 元素存在则恢复，否则跳过
   └── 元素被远程删除
       └── 跳过可见变更，继续找下一条历史记录
```

### 汇总表

| 状态类型 | 具体示例 | 协作冲突判定 | 关键条件 |
|---------|---------|-------------|---------|
| **普通标量属性** | `backgroundColor`, `x`, `y`, `strokeColor` | 视情况 | 历史记录包含该属性 → 覆盖，否则保留 |
| **数组属性** | `groupIds`, `points` | 覆盖远程 | 无法保证顺序一致性，直接替换 |
| **绑定关系** | `boundElements`, `containerId` | 保留远程 | 不更新 `boundElements`，后处理合并 |
| **Frame 从属** | `frameId` | 视情况 | Frame 存在 → 绑定，否则跳过 |
| **AppState 独立** | `name`, `viewBackgroundColor` | 保留远程 | 不在历史记录中则不影响 |
| **AppState 元素引用** | `selectedElementIds`, `editingGroupId` | 跳过 | 元素被删除则过滤 |
| **远程删除元素** | `isDeleted: true` | 跳过 | 元素已删除且不恢复 |

---

## 按事件顺序的判定流程

### 代码执行顺序

根据 `history.ts:24-45` 和 `delta.ts:1412-1452`，撤销/重做的执行顺序为：

```
HistoryDelta.applyTo(elements, appState, snapshot)
    ↓
1. ElementsDelta.applyTo()  ← 先处理元素
    ├── createApplier() → 获取元素
    ├── added 元素         ← 先恢复新增的元素
    ├── removed 元素       ← 再删除已删除的元素
    ├── updated 元素       ← 最后修改其他元素
    │   └── 对每个元素：
    │       ├── applyDelta() → 应用属性变化
    │       │   ├── 直接属性（跳过 boundElements, version）
    │       │   ├── mergeArrays(boundElements) → 合并绑定
    │       │   └── checkForVisibleDifference → 判断可见性
    │       └── 包含 groupIds/points 直接替换
    │
    ├── resolveConflicts() ← 后处理绑定关系
    │   ├── unbindAffected(removed)   → 被删除元素解绑
    │   ├── rebindAffected(added)     → 恢复元素重新绑定
    │   └── rebindAffected(updated)   → 修改元素重新绑定（含绑定属性）
    │
    ├── reorderElements()  ← z-index 排序
    └── redrawElements()   ← 重新绘制（文本框、箭头等）
    ↓
2. AppStateDelta.applyTo()  ← 后处理 AppState
    ├── 独立状态直接恢复
    ├── 元素引用过滤（selectedElementIds 等）
    └── containsVisibleDifference 累计
    ↓
3. containsVisibleChange = elements || appState
    └── false → 继续找下一条历史记录
```

### 分步判定流程

```
给定：用户 A 执行撤销，当前历史记录 = H

┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 判定远程删除（最高优先级）                               │
│                                                                  │
│ 对 H 中涉及的每个元素 E：                                         │
│   ├── E 被远程删除 (isDeleted=true)                              │
│   │   └── 且 H 不恢复 E (partial.isDeleted ≠ false)              │
│   │       └── → 跳过可见变更，属性变化仍应用                      │
│   │                                                                  │
│   └── E 被远程删除                                               │
│       └── 且 H 恢复 E (partial.isDeleted = false)                │
│           └── → 正常恢复，继续后续判定                            │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: 判定属性类别（普通属性 vs 数组属性）                      │
│                                                                  │
│ 对 H 中涉及的每个属性 P：                                         │
│   ├── P 是普通标量属性                                           │
│   │   ├── 历史记录包含 P → 应用 deleted 值（可能覆盖远程）       │
│   │   └── 历史记录不包含 P → 远程修改保留                         │
│   │                                                                  │
│   ├── P = boundElements                                          │
│   │   └── 不更新 inserted 端，保留历史值                          │
│   │                                                                  │
│   └── P 是数组属性 (groupIds, points)                            │
│       └── 直接用历史值覆盖当前值（远程修改丢失，重做时恢复）       │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: 判定关系后处理（绑定关系、Frame）                        │
│                                                                  │
│ resolveConflicts 阶段：                                          │
│   ├── removed 元素 → unbindAffected → 关联元素解绑               │
│   ├── added 元素 → rebindAffected → 恢复绑定                     │
│   │   └── 远程新增的绑定优先，旧绑定可能被覆盖                    │
│   │                                                                  │
│   └── updated 元素（含绑定属性）→ rebindAffected                  │
│       └── 文本重新绑定，箭头 TODO 未处理                           │
│                                                                  │
│ Frame 从属：                                                      │
│   ├── Frame 存在且未删除 → frameId 正常恢复                       │
│   └── Frame 被远程删除 → 子元素恢复时 frameId = null              │
└─────────────────────────────────────────────────────────────────┘
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: 判定可见变化跳过                                         │
│                                                                  │
│ containsVisibleDifference 判断：                                 │
│   ├── 元素从删除 → 恢复 → true                                   │
│   ├── 元素从存在 → 删除 → true                                   │
│   ├── 元素已删除且不恢复 → false（跳过）                         │
│   ├── AppState 元素引用全部指向已删除元素 → false（跳过）         │
│   └── 其他属性变化 → true                                        │
│                                                                  │
│ false → 继续撤销下一条历史记录                                    │
│ true → 停止，返回结果                                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 规则冲突优先级表

### 优先级说明

当同一轮撤销中多条规则同时命中时，按以下优先级判定：

| 优先级 | 规则类型 | 描述 | 覆盖范围 | 典型冲突场景 |
|-------|---------|------|---------|-------------|
| **P1（最高）** | 远程删除跳过 | `element.isDeleted=true` 且不恢复 | 所有元素级操作 | 元素被删除后，任何属性变化都不产生可见变化 |
| **P2** | 数组属性覆盖 | `groupIds`, `points` 直接替换 | 有序数组属性 | 本地分组 `["A"]`，远程加 `["B"]` → 撤销后 `[]` |
| **P3** | 普通属性判定 | 按属性独立处理 | 所有标量属性 | 同属性冲突 → 覆盖；不同属性 → 保留 |
| **P4** | 绑定关系保留 | `boundElements` 不更新，后处理合并 | 容器↔文本绑定 | 远程新绑定优先，旧绑定不恢复 |
| **P5** | Frame 从属判定 | Frame 存在则绑定，否则跳过 | `frameId` | Frame 删除后，子元素不重新绑定 |
| **P6（最低）** | AppState 过滤 | 元素引用过滤 | `selectedElementIds` 等 | 选中元素被删除 → 跳过该状态 |

### 优先级图示

```
P1: 远程删除跳过 (最高)
  ↓ 覆盖
P2: 数组属性覆盖 (groupIds, points)
  ↓ 覆盖
P3: 普通属性判定 (按属性独立)
  ↓ 覆盖
P4: 绑定关系保留 (boundElements 不更新)
  ↓ 覆盖
P5: Frame 从属判定
  ↓ 覆盖
P6: AppState 元素引用过滤 (最低)
```

### 典型冲突场景解析

#### 冲突场景 1：P1 vs P3（远程删除 vs 属性修改）

```
元素 X 被远程删除 (isDeleted=true)
同时用户 A 的历史记录包含 X.backgroundColor 的修改

Step 1 (P1): 元素已删除且不恢复
  → checkForVisibleDifference = false（跳过可见变更）
  → 但 backgroundColor 从 yellow → transparent 仍应用

Step 3 (P3): 历史记录包含 backgroundColor
  → 应该覆盖远程修改

结果：属性值被修改，但元素保持删除 → 协作者看不到变化（已删除）
```

**代码依据：** `delta.ts:1715-1717`

```typescript
if (element.isDeleted && partial.isDeleted !== false) {
  return false;  // 跳过可见变更
}
```

#### 冲突场景 2：P2 vs P3（数组属性 vs 普通属性）

```
元素 X:
  - 本地历史记录: groupIds=["A"], backgroundColor=red
  - 远程修改: groupIds=["A", "B"], backgroundColor=yellow

Step 2 (P2): groupIds 是数组属性
  → 直接用历史值覆盖 → groupIds=[]（远程的 B 丢失）

Step 3 (P3): backgroundColor 是普通属性
  → 历史记录包含 → 应用 deleted 值 → transparent（覆盖远程的 yellow）

结果：groupIds 和 backgroundColor 都被覆盖
```

**代码依据：** `delta.ts:1340-1341`

```typescript
for (const key of Object.keys(partial)) {
  default:
    latestPartial[key] = element[key];  // 直接替换
}
```

#### 冲突场景 3：P4 vs P3（绑定关系 vs 普通属性）

```
容器 C:
  - 本地历史记录: boundElements=[文本A], backgroundColor=red
  - 远程修改: boundElements=[文本A, 文本B], backgroundColor=yellow

Step 2 (P3 用于 backgroundColor):
  → 历史记录包含 → 恢复为 transparent（覆盖远程的 yellow）

Step 2 (P4 用于 boundElements):
  → 不更新 inserted 端，保留历史值 [文本A]

Step 3 (后处理 mergeArrays):
  → 合并 boundElements → [文本A, 文本B]（远程的文本B 保留）

结果：backgroundColor 被覆盖，boundElements 保留远程新增
```

**代码依据：** `delta.ts:1337-1339`

```typescript
case "boundElements":
  latestPartial[key] = partial[key];  // 不更新，保留历史值
  break;
```

`delta.ts:1677-1686`

```typescript
const mergedBoundElements = Delta.mergeArrays(
  element.boundElements,
  delta.inserted.boundElements,
  delta.deleted.boundElements,
  (x) => x.id,
);
```

#### 冲突场景 4：P5 vs P2（Frame 从属 vs groupIds）

```
元素 X 在 Frame F 内：
  - 本地历史记录: frameId=F.id, groupIds=["A"]
  - 远程修改: frameId=null (Frame 被删除), groupIds=["A", "B"]

Step 2 (P2): groupIds 直接覆盖 → []

Step 3 (P5): Frame 被远程删除
  → frameId 恢复但 Frame 不存在
  → 后处理？（实际测试显示不重新绑定）

结果：groupIds 被覆盖，frameId 可能恢复但 Frame 已删除
```

**测试用例：** `history.test.tsx:5221-5304`

---

## 6 个典型输入场景的逐步判定演示

### 场景 1：同元素不同属性（保留远程）

**输入：**
- 本地操作：创建矩形 → 设置 `backgroundColor=red`
- 远程操作：设置同一矩形 `strokeColor=yellow`
- undoStack: [创建, 改颜色]
- 当前选中状态：无

**逐步判定：**

```
Step 0: 弹出历史记录（改颜色）
  → H = { updated: { rect: { backgroundColor: red → transparent } } }

Step 1: 判定远程删除（P1）
  → rect.isDeleted = false
  → 继续

Step 2: 判定属性类别（P2/P3）
  → backgroundColor 是普通属性
  → 历史记录包含该属性
  → 应用 deleted 值: transparent
  → strokeColor 不在历史记录中
  → 远程修改保留: yellow

Step 3: 判定关系后处理（P4/P5）
  → 无绑定关系，无 Frame
  → 继续

Step 4: 判定可见变化跳过（P6）
  → 可见元素属性变化
  → containsVisibleDifference = true
  → 停止

最终结果：
  - backgroundColor = transparent（本地撤销生效）
  - strokeColor = yellow（远程修改保留）
  → 判定：保留远程（不同属性）
```

**测试用例佐证：** `history.test.tsx:2169-2203`

---

### 场景 2：同元素同一属性（覆盖远程）

**输入：**
- 本地操作：创建矩形 → 设置 `backgroundColor=red`
- 远程操作：设置同一矩形 `backgroundColor=yellow`
- undoStack: [创建, 改颜色]

**逐步判定：**

```
Step 0: 弹出历史记录（改颜色）
  → H = { updated: { rect: { backgroundColor: red → transparent } } }

Step 1: 判定远程删除（P1）
  → rect.isDeleted = false
  → 继续

Step 2: 判定属性类别（P2/P3）
  → backgroundColor 是普通属性
  → 历史记录包含该属性
  → 应用 deleted 值: transparent
  → 远程的 yellow 被覆盖

Step 3: 判定关系后处理（P4/P5）
  → 无绑定关系
  → 继续

Step 4: 判定可见变化跳过（P6）
  → 可见元素属性变化
  → containsVisibleDifference = true
  → 停止

最终结果：
  - backgroundColor = transparent（覆盖远程的 yellow）
  → 判定：覆盖远程（同属性冲突）
```

**测试用例佐证：** `history.test.tsx:2205-2367`

---

### 场景 3：groupIds 覆盖

**输入：**
- 本地操作：创建两个矩形 → 加入分组 A
- 远程操作：加入分组 B
- undoStack: [创建, 分组A]
- 当前：rect.groupIds = ["A", "B"]

**逐步判定：**

```
Step 0: 弹出历史记录（分组A）
  → H = { updated: { rect1: { groupIds: ["A"] → [] },
                     rect2: { groupIds: ["A"] → [] } } }

Step 1: 判定远程删除（P1）
  → rect1.isDeleted = false, rect2.isDeleted = false
  → 继续

Step 2: 判定属性类别（P2/P3）
  → groupIds 是数组属性（P2 优先级高于 P3）
  → 直接用历史值覆盖当前值
  → 当前 ["A", "B"] → []（远程的 B 被覆盖）

Step 3: 判定关系后处理（P4/P5）
  → 无绑定关系
  → 继续

Step 4: 判定可见变化跳过（P6）
  → 可见元素属性变化
  → containsVisibleDifference = true
  → 停止

最终结果：
  - groupIds = []（远程的 B 被覆盖）
  → 判定：覆盖远程（数组属性）
```

**测试用例佐证：** `history.test.tsx:2371-2423`

---

### 场景 4：绑定关系保留

**输入：**
- 本地操作：创建容器
- 远程操作：绑定文本 A 到容器
- undoStack: [创建容器]
- 用户 A 撤销（容器删除）
- 远程操作：绑定新文本 B 到容器，恢复容器
- redoStack: [删除容器]
- 用户 A 重做

**逐步判定：**

```
Step 0: 弹出历史记录（删除容器 → 即恢复容器）
  → H = { added: { container: { isDeleted: false,
                                 boundElements: [文本A] } } }

Step 1: 判定远程删除（P1）
  → container.isDeleted = false（已被远程恢复）
  → 继续

Step 2: 判定属性类别（P2/P3）
  → boundElements: 不更新 inserted 端，保留历史值 [文本A]
  → 当前 boundElements = [文本B]

Step 3: 判定关系后处理（P4/P5）
  → added 元素触发 rebindAffected
  → mergeArrays 合并 boundElements
  → [文本A] + [文本B] = [文本B]（最新绑定优先）
  → 文本 A 不重新绑定

Step 4: 判定可见变化跳过（P6）
  → 元素从删除 → 恢复
  → containsVisibleDifference = true
  → 停止

最终结果：
  - container 恢复
  - 文本 B 的绑定保留（远程新增的）
  - 文本 A 不重新绑定
  → 判定：保留远程（绑定关系）
```

**测试用例佐证：** `history.test.tsx:4005-4109`

---

### 场景 5：远程删除元素跳过

**输入：**
- 本地操作：创建矩形 → 设置 `backgroundColor=red`
- 远程操作：删除矩形，同时设置 `backgroundColor=yellow`
- undoStack: [创建, 改颜色]
- 当前：rect.isDeleted = true, backgroundColor = yellow

**逐步判定：**

```
Step 0: 弹出历史记录（改颜色）
  → H = { updated: { rect: { backgroundColor: red → transparent } } }

Step 1: 判定远程删除（P1，最高优先级）
  → rect.isDeleted = true
  → H 不恢复 rect (partial.isDeleted = undefined ≠ false)
  → checkForVisibleDifference = false（跳过可见变更）
  → 但 backgroundColor 仍从 yellow → transparent 应用

Step 2: 判定属性类别（P2/P3）
  → backgroundColor 是普通属性
  → 历史记录包含
  → 应用 deleted 值: transparent

Step 3: 判定关系后处理（P4/P5）
  → 无绑定关系
  → 继续

Step 4: 判定可见变化跳过（P6）
  → containsVisibleDifference = false（P1 决定）
  → 继续撤销下一条历史记录

Step 5: 弹出下一条历史记录（创建矩形 → 即删除矩形）
  → H = { added: { rect: { isDeleted: false } } }

Step 1: 判定远程删除（P1）
  → rect.isDeleted = true
  → H 要删除 rect (partial.isDeleted = true)
  → 已删除元素再删除？
  → 实际测试：继续找可见变化...

最终结果（按一次 Ctrl+Z 后）：
  - rect.isDeleted = true（保持删除）
  - rect.backgroundColor = transparent（被修改）
  - undoStack 从 2 → 0（跳过了改颜色）
  → 判定：跳过（远程删除）
```

**测试用例佐证：** `history.test.tsx:2584-2636`

---

### 场景 6：选中元素被远程删除（跳过 AppState）

**输入：**
- 本地操作：创建 rect1 → 创建 rect2 → 创建 rect3 → 选中 rect1 → 选中 [rect2, rect3]
- 远程操作：删除 rect2, rect3
- undoStack: [创建3个, 选中rect1, 选中rect2/rect3]
- 当前选中：rect2, rect3

**逐步判定：**

```
Step 0: 弹出历史记录（选中 rect2/rect3）
  → H = { appState: { selectedElementIds: {rect2, rect3} → {rect1} } }

Step 1: 判定远程删除（P1）
  → 不涉及元素变化
  → 继续

Step 2: 判定属性类别（P2/P3）
  → 不涉及元素属性
  → 继续

Step 3: 判定关系后处理（P4/P5）
  → 无绑定关系
  → 继续

Step 4: 判定可见变化跳过（P6）
  → filterSelectedElements 过滤
  → rect2.isDeleted = true → 过滤
  → rect3.isDeleted = true → 过滤
  → 剩下空的选中状态
  → containsVisibleDifference = false
  → 继续撤销下一条

Step 5: 弹出历史记录（选中 rect1）
  → H = { appState: { selectedElementIds: {rect1} → {} } }

Step 4: 判定可见变化跳过（P6）
  → rect1.isDeleted = false
  → containsVisibleDifference = true
  → 停止

最终结果（按一次 Ctrl+Z 后）：
  - 选中状态：rect1
  - undoStack 从 3 → 1（跳过了 2 条）
  → 判定：跳过（AppState 元素引用）
```

**测试用例佐证：** `history.test.tsx:2715-2803`

---

## 6 个场景汇总表

| 场景 | 本地操作 | 远程操作 | Step 1 远程删除 | Step 2 属性类别 | Step 3 关系后处理 | Step 4 可见跳过 | 最终判定 |
|-----|---------|---------|---------------|----------------|-----------------|----------------|---------|
| **场景 1** | 改 `backgroundColor` | 改 `strokeColor` | 否 | 普通属性，历史包含 | 无 | 否 | ✅ 保留远程 |
| **场景 2** | 改 `backgroundColor` | 改同一属性 | 否 | 普通属性，历史包含 | 无 | 否 | ❌ 覆盖远程 |
| **场景 3** | 加 `groupIds=["A"]` | 加 `groupIds=["B"]` | 否 | 数组属性，直接覆盖 | 无 | 否 | ❌ 覆盖远程 |
| **场景 4** | 创建容器 | 绑定新文本 B | 否 | boundElements 不更新 | mergeArrays 保留 B | 否 | ✅ 保留远程 |
| **场景 5** | 改颜色 | 删除元素 | **是** → 跳过 | 普通属性（仍应用） | 无 | **是** → 继续 | ⏭️ 跳过 |
| **场景 6** | 选中 rect2/rect3 | 删除 rect2/rect3 | 否 | 无 | 无 | **是** → 继续 | ⏭️ 跳过 |

---

## 组合冲突反例库（8 个多条件同时命中场景）

### 说明

以下场景展示**多条规则同时命中**时的判定过程，重点关注：
- 命中链路：哪些规则被触发
- 被淘汰规则：哪些规则因优先级被压制
- 最终裁决：实际生效的结果

---

### 组合冲突 1：远程删除 + 同属性冲突

**场景描述：**
- 本地：创建矩形 → 设置 `backgroundColor=red`
- 远程：删除矩形，同时设置 `backgroundColor=yellow`
- undoStack: [创建, 改颜色]

**命中规则：** P1（远程删除跳过）+ P3（同属性冲突覆盖）

**逐步判定：**

```
历史记录（改颜色）= { updated: { rect: { backgroundColor: red → transparent } } }

Step 1: 判定远程删除（P1，最高优先级）
  → rect.isDeleted = true
  → 历史记录不恢复 rect (partial.isDeleted = undefined ≠ false)
  → checkForVisibleDifference = false（跳过可见变更）
  → [被命中] P1 生效

Step 2: 判定属性类别（P3）
  → backgroundColor 是普通属性
  → 历史记录包含该属性
  → 应用 deleted 值: transparent
  → 远程的 yellow 被覆盖
  → [被命中] P3 生效
  → [被淘汰] 但 P1 决定了整体可见性

Step 3: 判定关系后处理
  → 无绑定关系
  → 继续

Step 4: 判定可见变化跳过（P6）
  → containsVisibleDifference = false（由 P1 决定）
  → 继续撤销下一条

Step 5: 弹出下一条历史记录（创建矩形）
  → H = { added: { rect: { isDeleted: false } } }

Step 1: 判定远程删除（P1）
  → rect.isDeleted = true
  → 历史记录要删除 rect (partial.isDeleted = true)
  → 已删除元素再删除？
  → 实际：containsVisibleDifference = true（删除已删除元素也算可见变化）
  → 停止

最终裁决：
  → rect.isDeleted = true（保持删除，P1 主导）
  → rect.backgroundColor = transparent（P3 生效，属性被修改）
  → undoStack 从 2 → 0（P1 导致跳过了改颜色，直接处理创建）

被淘汰规则：
  → P3 虽命中并修改了属性，但 P1 决定了整体无可见变化（第一轮）
```

**测试用例佐证：** `history.test.tsx:2584-2636`

---

### 组合冲突 2：Frame 删除 + 绑定恢复

**场景描述：**
- 本地：矩形在 Frame 内 → 移出 Frame → 删除矩形
- 远程：删除 Frame
- undoStack: [创建Frame+矩形, 放入Frame, 移出Frame, 删除矩形]
- 当前：rect.isDeleted=true, frame.isDeleted=true

**命中规则：** P1（远程删除跳过）+ P4（绑定关系保留）+ P5（Frame 从属判定）

**逐步判定：**

```
用户执行重做两次（恢复矩形 + 放入 Frame）

Step 0: 弹出历史记录（删除矩形 → 恢复矩形）
  → H = { added: { rect: { isDeleted: false, frameId: null } } }

Step 1: 判定远程删除（P1）
  → rect.isDeleted = true
  → 历史记录恢复 rect (partial.isDeleted = false)
  → 正常恢复，继续

Step 2: 判定属性类别
  → frameId 是普通属性
  → 历史记录包含 frameId: null
  → 应用后 frameId = null

Step 3: 判定关系后处理（P5）
  → Frame.isDeleted = true（被远程删除）
  → 不重新绑定（即使有 frameId）
  → rect.frameId 保持 null
  → [被淘汰] P4（绑定恢复）因 P1（Frame 删除）被压制

Step 4: 判定可见变化跳过
  → 元素恢复，可见
  → 停止

Step 5: 弹出下一条历史记录（移出 Frame → 放入 Frame）
  → H = { updated: { rect: { frameId: null → frame.id } } }

Step 3: 判定关系后处理（P5）
  → Frame.isDeleted = true
  → 不重新绑定
  → rect.frameId = null（即使历史记录有 frame.id）

最终裁决：
  → rect.isDeleted = false（恢复）
  → rect.frameId = null（不绑定到已删除的 Frame）
  → [被淘汰] P4/P5 的绑定恢复因 P1（Frame 删除）被压制
```

**测试用例佐证：** `history.test.tsx:5221-5304`

---

### 组合冲突 3：数组覆盖 + AppState 过滤

**场景描述：**
- 本地：创建两个矩形 → 加入分组 A → 选中分组
- 远程：加入分组 B → 删除两个矩形
- undoStack: [创建, 分组A, 选中分组]
- 当前：rect.groupIds=["A", "B"], rect.isDeleted=true, selectedElementIds={rect1, rect2}

**命中规则：** P1（远程删除跳过）+ P2（数组属性覆盖）+ P6（AppState 过滤）

**逐步判定：**

```
Step 0: 弹出历史记录（选中分组）
  → H = { appState: { selectedElementIds: {rect1, rect2} → {} } }

Step 1: 判定远程删除（P1）
  → 不涉及元素变化
  → 继续

Step 2: 判定属性类别
  → 无元素属性
  → 继续

Step 3: 判定关系后处理
  → 无绑定关系
  → 继续

Step 4: 判定可见变化跳过（P6）
  → filterSelectedElements 过滤
  → rect1.isDeleted = true → 过滤
  → rect2.isDeleted = true → 过滤
  → containsVisibleDifference = false
  → 继续撤销下一条
  → [被命中] P6 生效

Step 5: 弹出下一条历史记录（分组A）
  → H = { updated: { rect1: { groupIds: ["A"] → [] },
                     rect2: { groupIds: ["A"] → [] } } }

Step 1: 判定远程删除（P1）
  → rect1.isDeleted = true, rect2.isDeleted = true
  → 历史记录不恢复元素 (partial.isDeleted = undefined ≠ false)
  → checkForVisibleDifference = false（跳过可见变更）
  → [被命中] P1 生效

Step 2: 判定属性类别（P2）
  → groupIds 是数组属性
  → 直接用历史值覆盖当前值
  → ["A", "B"] → []
  → [被命中] P2 生效
  → [被淘汰] 但 P1 决定了整体可见性

Step 4: 判定可见变化跳过（P6）
  → containsVisibleDifference = false（由 P1 决定）
  → 继续撤销下一条

最终裁决：
  → rect.isDeleted = true（保持删除，P1 主导）
  → rect.groupIds = []（P2 生效，远程的 B 被覆盖）
  → selectedElementIds = {}（P6 导致跳过选中分组状态）
  → undoStack 从 3 → 0（跳过了选中分组和分组A，直接处理创建）

被淘汰规则：
  → P2 虽命中并覆盖了 groupIds，但 P1 决定了整体无可见变化
  → P6 导致 AppState 状态被跳过
```

**测试用例佐证：** 可组合 `history.test.tsx:2638-2713` 和 `history.test.tsx:2371-2423`

---

### 组合冲突 4：绑定恢复 + 同属性冲突

**场景描述：**
- 本地：创建容器 → 绑定文本 A
- 远程：绑定文本 A → 解绑文本 A → 绑定新文本 B
- undoStack: [创建容器, 绑定A]
- 用户 A 撤销两次（容器删除 + 文本解绑）
- 远程：绑定文本 B 到容器，恢复容器
- redoStack: [删除容器, 解绑A]
- 用户 A 重做

**命中规则：** P3（同属性冲突）+ P4（绑定关系保留）

**逐步判定：**

```
Step 0: 弹出历史记录（删除容器 → 恢复容器）
  → H = { added: { container: { isDeleted: false,
                                 boundElements: [文本A] } } }

Step 1: 判定远程删除
  → container.isDeleted = false（已被远程恢复）
  → 继续

Step 2: 判定属性类别（P3）
  → boundElements: 不更新 inserted 端，保留历史值 [文本A]
  → 当前 boundElements = [文本B]

Step 3: 判定关系后处理（P4）
  → added 元素触发 rebindAffected
  → mergeArrays 合并 boundElements
  → 历史记录: [文本A]
  → 当前值: [文本B]
  → mergeArrays 结果: [文本A, 文本B] 去重排序
  → 实际测试: [文本A]（历史记录恢复）
  → [被命中] P4 生效
  → [被淘汰] P3（不更新 inserted）被 P4（后处理合并）覆盖

Step 3 子步骤：文本容器双向绑定
  → container.boundElements = [文本A]
  → 文本 A.containerId = container.id（恢复）
  → 文本 B.containerId = null（被解绑）
  → [被淘汰] P3（同属性冲突：远程的文本 B 绑定）被 P4 压制

最终裁决：
  → container 恢复
  → 文本 A 的绑定恢复（P4 优先）
  → 文本 B 的绑定被覆盖（P3 被淘汰）

被淘汰规则：
  → P3（同属性冲突：文本 B 的绑定）被 P4（后处理双向绑定恢复）压制
```

**测试用例佐证：** `history.test.tsx:3673-3773`（容器绑定冲突）、`history.test.tsx:3776-3882`（文本容器冲突）

---

### 组合冲突 5：远程删除 + 绑定关系

**场景描述：**
- 本地：创建容器
- 远程：绑定文本 A 到容器 → 删除文本 A
- undoStack: [创建容器]
- 用户 A 撤销（容器删除）
- 远程：删除文本 A（同时保持容器绑定）
- redoStack: [删除容器]
- 用户 A 重做

**命中规则：** P1（远程删除跳过）+ P4（绑定关系保留）

**逐步判定：**

```
Step 0: 弹出历史记录（删除容器 → 恢复容器）
  → H = { added: { container: { isDeleted: false,
                                 boundElements: [文本A] } } }

Step 1: 判定远程删除（P1）
  → 文本 A.isDeleted = true（被远程删除）
  → 容器.isDeleted = false（已被远程恢复）
  → 继续

Step 2: 判定属性类别
  → boundElements = [文本A]

Step 3: 判定关系后处理（P4）
  → added 元素触发 rebindAffected
  → 文本 A.isDeleted = true
  → 检查：非删除元素才能重新绑定
  → 文本 A 不绑定到容器
  → [被命中] P1 生效（元素被删除）
  → [被淘汰] P4（绑定恢复）因 P1（文本删除）被压制

Step 3 子步骤：resolveConflicts 后处理
  → container.boundElements 应该更新
  → 实际测试: container.boundElements = []
  → [被淘汰] P4 的绑定恢复被 P1 压制

最终裁决：
  → container 恢复
  → container.boundElements = []（文本 A 已删除，不绑定）
  → 文本 A.containerId = null（保持解绑）

被淘汰规则：
  → P4（绑定恢复）因 P1（文本被远程删除）被压制
```

**测试用例佐证：** `history.test.tsx:4217-4272`（删除绑定文本）、`history.test.tsx:4274-4329`（删除容器）

---

### 组合冲突 6：数组覆盖 + 绑定恢复

**场景描述：**
- 本地：创建两个矩形 → 加入分组 A → 绑定箭头到两个矩形
- 远程：加入分组 B → 移动目标矩形
- undoStack: [创建, 分组A, 绑定箭头]
- 当前：rect.groupIds=["A", "B"], rect2.x=500

**命中规则：** P2（数组属性覆盖）+ P4（绑定关系保留）

**逐步判定：**

```
用户执行撤销分组 A

Step 0: 弹出历史记录（分组A）
  → H = { updated: { rect1: { groupIds: ["A"] → [] },
                     rect2: { groupIds: ["A"] → [] } } }

Step 1: 判定远程删除
  → rect1.isDeleted = false, rect2.isDeleted = false
  → 继续

Step 2: 判定属性类别（P2）
  → groupIds 是数组属性
  → 直接用历史值覆盖当前值
  → ["A", "B"] → []
  → 远程的 B 被覆盖
  → [被命中] P2 生效

Step 3: 判定关系后处理（P4）
  → updated 元素检查是否有绑定属性变化
  → groupIds 不是绑定属性
  → 不触发 rebindAffected
  → 箭头绑定不受影响
  → [被命中] P4 生效（绑定保留）

最终裁决：
  → rect.groupIds = []（P2 覆盖远程的 B）
  → 箭头绑定保持不变（P4 保留）

被淘汰规则：
  → 无冲突，P2 和 P4 各自生效
```

**测试用例佐证：** 可组合 `history.test.tsx:2371-2423`（groupIds 覆盖）和 `history.test.tsx:5106-5195`（箭头绑定）

---

### 组合冲突 7：AppState 过滤 + 绑定恢复

**场景描述：**
- 本地：创建容器 → 绑定文本 → 选中容器
- 远程：删除容器（文本随之被远程删除？）
- undoStack: [创建容器, 绑定文本, 选中容器]
- 当前：container.isDeleted=true, text.isDeleted=true, selectedElementIds={container}

**命中规则：** P1（远程删除跳过）+ P4（绑定关系保留）+ P6（AppState 过滤）

**逐步判定：**

```
用户执行撤销选中容器

Step 0: 弹出历史记录（选中容器）
  → H = { appState: { selectedElementIds: {container} → {} } }

Step 1: 判定远程删除（P1）
  → 不涉及元素变化
  → 继续

Step 2: 判定属性类别
  → 无元素属性
  → 继续

Step 3: 判定关系后处理
  → 无绑定关系变化
  → 继续

Step 4: 判定可见变化跳过（P6）
  → filterSelectedElements 过滤
  → container.isDeleted = true → 过滤
  → containsVisibleDifference = false
  → 继续撤销下一条
  → [被命中] P6 生效

Step 5: 弹出下一条历史记录（绑定文本）
  → H = { updated: { container: { boundElements: [文本] },
                     text: { containerId: container.id } } }

Step 1: 判定远程删除（P1）
  → container.isDeleted = true
  → text.isDeleted = true
  → 历史记录不恢复元素 (partial.isDeleted = undefined)
  → checkForVisibleDifference = false
  → [被命中] P1 生效

Step 2: 判定属性类别（P4）
  → boundElements: 保留历史值
  → 但元素已删除
  → [被淘汰] P4（绑定恢复）因 P1 被压制

Step 4: 判定可见变化跳过（P6）
  → containsVisibleDifference = false（由 P1 决定）
  → 继续撤销下一条

最终裁决：
  → 撤销了创建容器（container.isDeleted = true 保持不变？）
  → 实际：继续直到找到可见变化
  → [被淘汰] P4（绑定恢复）和 P6 都被 P1 压制

被淘汰规则：
  → P6（AppState 过滤）被 P1 主导
  → P4（绑定恢复）因 P1（元素删除）被压制
```

**测试用例佐证：** 可组合 `history.test.tsx:2715-2803`（AppState 过滤）和 `history.test.tsx:4217-4272`（删除绑定文本）

---

### 组合冲突 8：远程恢复 + 数组覆盖 + 绑定关系

**场景描述：**
- 本地：创建矩形 → 删除矩形
- 远程：恢复矩形 → 加入分组 B → 绑定箭头
- undoStack: [创建矩形, 删除矩形]
- redoStack: [删除矩形]
- 当前：rect.isDeleted=false, rect.groupIds=["B"], arrow.boundElements=[rect]

**命中规则：** P1（远程恢复，特殊情况）+ P2（数组属性覆盖）+ P4（绑定关系保留）

**逐步判定：**

```
用户执行撤销删除矩形（即恢复矩形）

Step 0: 弹出历史记录（删除矩形 → 恢复矩形）
  → H = { removed: { rect: { isDeleted: true } } }

Step 1: 判定远程删除（P1 反向）
  → rect.isDeleted = false（已被远程恢复）
  → 历史记录要恢复 rect (partial.isDeleted = false)
  → 当前值已恢复
  → 检查：isDeleted 相同 → Delta 重新分发
  → removed → updated
  → [特殊情况] P1 反向（远程恢复）

Step 2: 判定属性类别（P2）
  → 历史记录中没有 groupIds
  → 远程的 ["B"] 保留
  → [被命中] P2 不生效（历史记录不包含该属性）

Step 3: 判定关系后处理（P4）
  → added 元素（实际现在是 updated）触发 rebindAffected
  → 检查是否有绑定属性变化
  → 历史记录中没有绑定属性变化
  → 不触发
  → 箭头绑定保持不变
  → [被命中] P4 生效（绑定保留）

最终裁决：
  → rect.isDeleted = false（保持恢复）
  → rect.groupIds = ["B"]（远程分组保留，P2 不生效）
  → 箭头绑定保持不变（P4 保留）

被淘汰规则：
  → P2（数组覆盖）因历史记录不包含该属性被淘汰
```

**测试用例佐证：** `history.test.tsx:2506-2582`（元素删除后被远程恢复）

---

### 组合冲突裁决总表

| 组合冲突 | 命中规则 | 被淘汰规则 | 最终裁决 | 测试用例 |
|---------|---------|-----------|---------|---------|
| **1. 远程删除 + 同属性冲突** | P1 + P3 | P3（可见性被压制） | 元素保持删除，属性被修改 | `history.test.tsx:2584` |
| **2. Frame 删除 + 绑定恢复** | P1 + P5 | P4/P5（绑定被压制） | 元素恢复但不绑定 Frame | `history.test.tsx:5221` |
| **3. 数组覆盖 + AppState 过滤** | P1 + P2 + P6 | P2（可见性被压制） | 元素保持删除，groupIds 被修改 | 组合测试 |
| **4. 绑定恢复 + 同属性冲突** | P4 | P3（同属性冲突被压制） | 历史绑定恢复，远程绑定被覆盖 | `history.test.tsx:3673` |
| **5. 远程删除 + 绑定关系** | P1 | P4（绑定被压制） | 元素恢复但不绑定删除的文本 | `history.test.tsx:4217` |
| **6. 数组覆盖 + 绑定恢复** | P2 + P4 | 无冲突 | groupIds 被覆盖，绑定保留 | 组合测试 |
| **7. AppState 过滤 + 绑定恢复** | P1 + P6 | P4/P6（都被压制） | 继续下探直到可见变化 | 组合测试 |
| **8. 远程恢复 + 数组覆盖 + 绑定** | P4 | P2（不生效） | 远程分组和绑定都保留 | `history.test.tsx:2506` |

---

## 判定终止条件与最小证据集

### 判定终止条件

根据 `history.ts:178-218`，撤销/重做的判定在以下任一条件满足时终止：

```
终止条件 1: containsVisibleChange = true（命中规则 P1/P2/P3/P4/P5）
  ↓
  原因：产生了用户可见的变化
  ↓
  判定：充分，可以停止

终止条件 2: 历史栈为空（undoStack 或 redoStack 为空）
  ↓
  原因：没有更多历史记录可撤销/重做
  ↓
  判定：充分，停止
```

### containsVisibleChange 的判定逻辑

根据 `delta.ts:1711-1732` 和 `delta.ts:770-837`：

```
Elements 可见性判定：
  ├── 元素从删除 → 恢复 → true
  ├── 元素从存在 → 删除 → true
  ├── 元素已删除且不恢复 → false（跳过）
  └── 可见元素的任何属性变化 → true

AppState 可见性判定：
  ├── 独立状态变化 → true
  └── 元素引用变化：
      ├── 至少一个引用元素存在 → true
      └── 所有引用元素被删除 → false（跳过）
```

### 最小证据集

在以下情况可以**停止继续下探**并认为结论充分：

| 证据类型 | 最小证据集 | 判定依据 |
|---------|-----------|---------|
| **元素级可见变化** | 至少一个元素从删除→恢复，或存在→删除 | `delta.ts:1711-1732` |
| **属性级可见变化** | 至少一个可见元素的任意属性变化 | `delta.ts:1731` |
| **AppState 可见变化** | 独立状态变化，或至少一个引用元素存在 | `delta.ts:770-837` |
| **历史栈耗尽** | undoStack 或 redoStack 为空 | `history.ts:164-168` |

### 不需要继续下探的场景

| 场景 | 终止条件 | 最小证据集 |
|-----|---------|-----------|
| 本地修改可见元素的属性 | containsVisibleChange = true | 属性级可见变化 |
| 本地创建/删除元素 | containsVisibleChange = true | 元素级可见变化 |
| 选中元素全部存在 | containsVisibleChange = true | AppState 可见变化 |
| undoStack 为空 | 历史栈耗尽 | 无历史记录 |

### 需要继续下探的场景

| 场景 | 继续下探原因 | 终止条件 |
|-----|-------------|---------|
| 选中元素被远程删除 | containsVisibleChange = false | 找到存在元素的选中状态 |
| 分组元素被远程删除 | containsVisibleChange = false | 找到存在元素的分组选择 |
| 编辑组元素被远程删除 | containsVisibleChange = false | 找到存在元素的编辑组 |
| 元素被远程删除 | containsVisibleChange = false | 找到可见变化或栈空 |

### 下探深度限制

代码中没有显式的下探深度限制，但有安全机制：

```
根据 `delta.ts:1437-1440`，异常情况时默认返回 containsVisibleChange = true：

// should not really happen, but just in case we cannot apply deltas, let's return the previous elements with visible change set to `true`
// even though there is obviously no visible change, returning `false` could be dangerous, as i.e.:
// in the worst case, it could lead into iterating through the whole stack with no possibility to redo
// instead, the worst case when returning `true` is an empty undo / redo
```

**实际行为：**
- 正常情况：持续下探直到 `containsVisibleChange = true` 或栈空
- 异常情况：强制返回 `true`，避免无限循环

### 结论充分性判定

给定一个协作撤销场景，判定结论充分的条件：

```
结论充分 = （存在可见变化证据）OR（历史栈耗尽）

可见变化证据 = 
  （元素从删除→恢复）OR
  （元素从存在→删除）OR
  （可见元素属性变化）OR
  （AppState 独立状态变化）OR
  （至少一个 AppState 引用元素存在）
```

### 典型充分/不充分判定

| 场景 | 证据 | 结论充分？ |
|-----|------|-----------|
| 修改可见元素颜色 | 可见元素属性变化 | ✅ 充分 |
| 选中元素全部被删除 | 无可见变化 | ❌ 不充分，继续下探 |
| 恢复已删除元素 | 元素从删除→恢复 | ✅ 充分 |
| 元素被删除且属性被修改 | 属性变化但元素不可见 | ❌ 不充分，继续下探 |
| 远程恢复元素且改属性 | 元素从删除→恢复（即使已恢复） | ✅ 充分（Delta 重新分发后可能有变化） |
| undoStack 为空 | 历史栈耗尽 | ✅ 充分 |

---

## 常见误判清单

### ❌ 误判 1：远程修改都会保留

**错误直觉：** 远程用户做的修改，本地撤销不应该影响。

**实际情况：** 视属性类型而定

| 属性类型 | 远程修改保留？ | 原因 |
|---------|---------------|------|
| 独立属性（不在历史记录中） | ✅ 保留 | 历史记录不包含该属性 |
| 独立属性（在历史记录中） | ❌ 不保留 | 应用历史记录的 deleted 值覆盖 |
| `boundElements` | ✅ 保留 | 代码专门不更新，后处理合并 |
| `groupIds` | ❌ 不保留 | 无法保证顺序一致性，直接替换 |
| `points` | ❌ 不保留 | 无法保证顺序一致性，直接替换 |

**测试用例佐证：**

- 不保留：`history.test.tsx:2205`（同属性冲突覆盖）、`history.test.tsx:2371`（groupIds 覆盖）、`history.test.tsx:2425`（points 覆盖）
- 保留：`history.test.tsx:2169`（不同属性保留）、`history.test.tsx:4005`（绑定关系保留）

---

### ❌ 误判 2：撤销 = 时间回滚

**错误直觉：** 撤销会把整个画板状态回滚到之前的某个时间点。

**实际情况：** 撤销 = 执行反向操作 + 产生新版本号

```
时间回滚（错误理解）：
  时间点 1: [A, B]
  时间点 2: [A, B, C]  ← 用户 A 创建 C
  时间点 3: [A, B', C] ← 用户 B 修改 B
  用户 A 撤销 → [A, B]（B 的修改也被回滚）❌

实际执行（正确）：
  时间点 1: [A, B]
  时间点 2: [A, B, C]  ← 用户 A 创建 C（历史记录：added C）
  时间点 3: [A, B', C] ← 用户 B 修改 B（远程，不进入 A 的历史）
  用户 A 撤销 → 执行"删除 C"（反向操作）
  时间点 4: [A, B']     ← B 的修改保留 ✅
```

**代码依据：**

`packages/excalidraw/history.ts:32-34`

```typescript
{
  excludedProperties: new Set(["version", "versionNonce"]),
}
```

撤销时不恢复版本号，而是产生新版本。协作者收到的是新操作，按 `CaptureUpdateAction.NEVER` 处理。

---

### ❌ 误判 3：撤销一次只撤销一条历史记录

**错误直觉：** 按一次 Ctrl+Z，只撤销最近一次操作。

**实际情况：** 可能跳过多条历史记录

**触发条件：** 当前历史记录的变更不产生可见变化

```
场景：选中元素 A → 选中元素 B → 远程删除 B

撤销流程：
  1. 尝试撤销"选中 B"
     - filterSelectedElements 过滤掉 B（已删除）
     - containsVisibleChange = false
  2. 继续撤销下一条"选中 A"
     - A 存在
     - containsVisibleChange = true
  3. 停止

结果：按一次 Ctrl+Z，跳过了 1 条记录，撤销了 2 条
```

**代码依据：**

`packages/excalidraw/history.ts:178-218`

```typescript
// iterate through the history entries in case they result in no visible changes
while (historyDelta) {
  [nextElements, nextAppState, containsVisibleChange] =
    historyDelta.applyTo(...);
  
  // ... 应用变化 ...
  
  if (containsVisibleChange) {
    break;  // 找到可见变化，停止
  }
  
  historyDelta = pop();  // 没有可见变化，继续找下一条
}
```

**测试用例佐证：**

- `history.test.tsx:2584`（元素被删除 → 跳过颜色修改）
- `history.test.tsx:2715`（选中元素被删除 → 跳过选中状态）
- `history.test.tsx:2805`（分组被删除 → 跳过分组选择）

---

### ❌ 误判 4：绑定关系会自动完全恢复

**错误直觉：** 撤销容器删除，之前的文本绑定会完全恢复。

**实际情况：** 远程新增的绑定优先，旧文本可能不会重新绑定

```
场景：
  1. 用户 A 创建容器
  2. 用户 B 绑定文本 X 到容器
  3. 用户 A 撤销（容器删除，X 解绑）
  4. 用户 B：绑定新文本 Y 到容器，恢复容器
  5. 用户 A 重做

结果：
  - 容器恢复
  - 文本 Y 的绑定保留（远程新增的）
  - 文本 X 不重新绑定（被 Y 取代）
```

**测试用例佐证：**

`history.test.tsx:4005-4109`

```typescript
// 重做后：
// container.boundElements = [{ id: remoteText.id, type: "text" }]  // 远程文本
// text.containerId = null  // 旧文本未绑定
// remoteText.containerId = container.id  // 远程文本绑定
```

---

### ❌ 误判 5：Frame 删除后重做，子元素会回到 Frame 内

**错误直觉：** 矩形在 Frame 内，撤销后重做，矩形应该回到 Frame 内。

**实际情况：** 如果 Frame 被远程删除，矩形不会重新绑定

```
场景：
  1. 用户 A：矩形在 Frame 内（rect.frameId = frame.id）
  2. 用户 A 撤销两次（移出 Frame + 删除矩形）
  3. 用户 B：远程删除 Frame
  4. 用户 A 重做两次

结果：
  - 矩形恢复
  - rect.frameId = null（不重新绑定）
  - Frame 保持删除状态
```

**测试用例佐证：**

`history.test.tsx:5221-5304`

```typescript
// 重做后：
// rect.frameId = null  // 不重新绑定
// frame.isDeleted = true
```

---

### ❌ 误判 6：数组属性会智能合并

**错误直觉：** 本地分组 `["A"]`，远程加 `["B"]`，撤销后应该 `["B"]`。

**实际情况：** 撤销后变成 `[]`，远程的 `B` 也被覆盖

```
groupIds 场景：
  本地：["A"]
  远程：["A", "B"]
  撤销 → []（远程的 B 被覆盖）
  重做 → ["A", "B"]（远程的 B 恢复）
```

**代码注释说明：**

`history.test.tsx:2369-2370`

```typescript
// TODO: #7348 ideally we should not override, but since the order of groupIds matters,
// right now we cannot ensure that with postprocessed groupIds the order will be consistent
// after series or undos/redos, we don't postprocess them at all
```

**测试用例佐证：**

`history.test.tsx:2371-2423`（groupIds 覆盖）、`history.test.tsx:2425-2503`（points 覆盖）

---

## 快速问答

### Q: 协作下撤销范围如何界定？

**A:** 按以下规则判定：

| 规则 | 说明 |
|-----|------|
| **独立栈** | 每个客户端独立维护历史栈 |
| **反向新操作** | 撤销 = 执行反向操作 = 产生新版本号 |
| **动态更新** | 撤销前用 `applyLatestChanges` 更新历史记录 |
| **跳过不可见** | 变更不产生可见变化时，继续找下一条历史记录 |

### Q: 本地撤销会不会影响他人？

**A:** 会影响，但不是"时间回滚"：

| 维度 | 对协作者的影响 |
|-----|---------------|
| 撤销创建元素 | 协作者看到该元素被删除 |
| 撤销修改属性（在历史记录中） | 协作者看到该属性被覆盖 |
| 撤销修改属性（不在历史记录中） | 协作者仍看到自己的修改 |
| 远程删除的元素 | 协作者看到元素保持删除 |

### Q: 哪些直觉是错的？

| 错误直觉 | 正确结论 |
|---------|---------|
| 远程修改都会保留 | 视属性类型而定 |
| 撤销 = 时间回滚 | 撤销 = 执行反向新操作 |
| 撤销一次只撤销一条 | 可能跳过多条 |
| 绑定关系完全恢复 | 远程新增的绑定优先 |
| Frame 内元素重做后回到 Frame | Frame 删除后不重新绑定 |
| 数组属性智能合并 | 直接覆盖，重做时恢复 |

---

## 相关测试用例索引

| 场景 | 测试文件位置 |
|-----|-------------|
| 不同元素 | `history.test.tsx:2125` |
| 同元素不同属性 | `history.test.tsx:2169` |
| 同元素同一属性 | `history.test.tsx:2205` |
| 分组覆盖 | `history.test.tsx:2371` |
| points 覆盖 | `history.test.tsx:2425` |
| 元素删除/恢复并发 | `history.test.tsx:2506` |
| 单元素远程删除 | `history.test.tsx:2584` |
| 多元素远程删除 | `history.test.tsx:2638` |
| 选中元素远程删除 | `history.test.tsx:2715` |
| 分组选择远程删除 | `history.test.tsx:2805` |
| 编辑组远程删除 | `history.test.tsx:2902` |
| 线性元素编辑器远程删除 | `history.test.tsx:3050` |
| 容器文本绑定 | `history.test.tsx:3884`, `3945`, `4005` |
| 远程删除绑定 | `history.test.tsx:4217`, `4274` |
| 箭头绑定 | `history.test.tsx:4896`, `4988`, `5106` |
| Frame 从属 | `history.test.tsx:5221` |
