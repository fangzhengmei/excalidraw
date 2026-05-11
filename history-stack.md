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
