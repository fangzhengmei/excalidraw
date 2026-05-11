# Excalidraw 历史栈（History Stack）机制分析

## 目录

1. [概述](#概述)
2. [核心架构](#核心架构)
3. [元素操作如何进栈](#元素操作如何进栈)
4. [撤销时哪些状态会还原](#撤销时哪些状态会还原)
5. [协作场景下的撤销机制](#协作场景下的撤销机制)
6. [本地撤销会不会影响他人](#本地撤销会不会影响他人)
7. [关键测试用例说明](#关键测试用例说明)

---

## 概述

Excalidraw 的撤销/重做系统基于 **增量式 Delta 记录** 而非全量快照。每个用户操作（创建、修改、删除元素等）被记录为一个 `StoreDelta`，存储在两个栈中：

- **Undo Stack**：记录可撤销的操作历史
- **Redo Stack**：记录已撤销的操作，用于重做

这套机制的设计参考了 Figma 的多人协作技术架构（见代码注释中的引用），能够智能处理并发协作场景。

---

## 核心架构

### 1. 关键类及其关系

```
Store (存储管理)
  ├── 维护 StoreSnapshot (当前状态快照)
  ├── 计算 StoreDelta (变更增量)
  └── 触发事件：
      ├── onDurableIncrementEmitter → History.record() → 进入历史栈
      └── onStoreIncrementEmitter → 外部订阅者（如协作同步）

History (历史栈管理)
  ├── undoStack: HistoryDelta[]
  ├── redoStack: HistoryDelta[]
  └── 方法：record(), undo(), redo()

HistoryDelta (历史记录单元)
  ├── elements: ElementsDelta
  └── appState: AppStateDelta
```

### 2. StoreSnapshot 快照

`StoreSnapshot` 是某个时刻的完整状态快照，包含：

- **elements**: `SceneElementsMap` — 所有元素（包括已删除的）
- **appState**: `ObservedAppState` — 被观察的应用状态子集

并非所有 AppState 属性都会被历史追踪，只有 `getObservedAppState()` 定义的子集：

```typescript
// 被观察的 AppState 属性（会进入历史栈）
{
  name: null,
  editingGroupId: null,
  viewBackgroundColor: COLOR_PALETTE.white,
  selectedElementIds: {},
  selectedGroupIds: {},
  selectedLinearElement: null,
  croppingElementId: null,
  activeLockedId: null,
  lockedMultiSelections: {},
}
```

### 3. StoreDelta 增量

`StoreDelta` 计算两个快照之间的差异，分为：

#### ElementsDelta（元素变化）

- **added**: 新增的元素（isDeleted: false）
- **removed**: 删除的元素（isDeleted: true）
- **updated**: 修改的元素（isDeleted 状态不变）

每个元素变化用双向 Delta 记录：

```typescript
{
  deleted: { /* 变更前的属性值 */ },
  inserted: { /* 变更后的属性值 */ }
}
```

#### AppStateDelta（应用状态变化）

记录上述 `ObservedAppState` 中属性的变化。

---

## 元素操作如何进栈

### 1. 三种 CaptureUpdateAction

元素更新通过 `captureUpdate` 参数控制是否进入历史栈：

| Action | 说明 | 历史记录 | 快照更新 |
|--------|------|---------|---------|
| `IMMEDIATELY` | 立即可撤销 | ✅ 进入 undo 栈 | ✅ 更新 |
| `NEVER` | 永不撤销 | ❌ 不记录 | ✅ 更新 |
| `EVENTUALLY` | 最终可撤销 | ⏳ 暂不记录 | ❌ 不更新 |

### 2. 进栈流程

```
用户操作 (e.g., 创建矩形)
    ↓
updateScene({ elements, appState, captureUpdate: IMMEDIATELY })
    ↓
Store.commit()
    ↓
计算 StoreDelta = 前快照 → 后快照
    ↓
如果 Delta 非空：
    ├── 触发 onDurableIncrementEmitter
    └── 触发 onStoreIncrementEmitter（用于协作同步）
    ↓
History.record(StoreDelta)
    ↓
将 Delta 取反（inverse）→ HistoryDelta
    ↓
推入 undoStack
    ↓
如果涉及元素变化 → 清空 redoStack
```

### 3. 关键代码片段

**Store.commit() 触发持久化增量** (`packages/element/src/store.ts:183-201`)：

```typescript
public commit(elements, appState): void {
  this.flushMicroActions(); // 先执行微操作
  
  const action = this.getScheduledMacroAction();
  this.processAction({ action, elements, appState });
}
```

**Store.emitDurableIncrement()** (`packages/element/src/store.ts:216-246`)：

```typescript
private emitDurableIncrement(snapshot, change, delta) {
  const prevSnapshot = this.snapshot;
  
  // 计算变化增量
  if (!delta) {
    storeDelta = StoreDelta.calculate(prevSnapshot, snapshot);
  }
  
  if (!storeDelta.isEmpty()) {
    const increment = new DurableIncrement(storeChange, storeDelta);
    this.onDurableIncrementEmitter.trigger(increment);
    this.onStoreIncrementEmitter.trigger(increment);
  }
}
```

**History.record() 记录历史** (`packages/excalidraw/history.ts:117-137`)：

```typescript
public record(delta: StoreDelta) {
  // 忽略空 Delta 和已记录的 HistoryDelta
  if (delta.isEmpty() || delta instanceof HistoryDelta) {
    return;
  }
  
  // 取反后推入 undo 栈（因为撤销时需要反向操作）
  const historyDelta = HistoryDelta.inverse(delta);
  this.undoStack.push(historyDelta);
  
  // 仅当有元素变化时清空 redo 栈
  if (!historyDelta.elements.isEmpty()) {
    this.redoStack.length = 0;
  }
}
```

### 4. Redo 栈清空规则

**只有元素变化才会清空 redo 栈**（`history.ts:127-132`）：

```typescript
if (!historyDelta.elements.isEmpty()) {
  this.redoStack.length = 0;
}
```

这意味着：
- 点击画布取消选择（纯 AppState 变化）→ **不会**清空 redo 栈
- 创建/修改/删除元素 → **会**清空 redo 栈

---

## 撤销时哪些状态会还原

### 1. 撤销流程

```
Ctrl+Z 触发
    ↓
History.undo(elements, appState)
    ↓
从 undoStack 弹出 HistoryDelta
    ↓
HistoryDelta.applyTo(elements, appState, snapshot)
    ↓
应用反向增量 → 还原到上一个状态
    ↓
创建新的 StoreChange 和 StoreDelta（更新后的版本号）
    ↓
触发 onDurableIncrementEmitter（用于协作同步）
    ↓
将新 Delta 推入 redoStack
    ↓
返回新的 [elements, appState]
```

### 2. 还原的状态

撤销时会还原以下两类状态：

#### A. 元素状态（ElementsDelta.applyTo）

对于每个元素变化类型：

| Delta 类型 | 撤销操作 |
|-----------|---------|
| **added** | 将元素标记为 isDeleted: true |
| **removed** | 将元素恢复为 isDeleted: false |
| **updated** | 将属性从 inserted 恢复为 deleted |

#### B. AppState 状态（AppStateDelta.applyTo）

还原以下属性：
- `selectedElementIds` — 选中的元素 ID
- `selectedGroupIds` — 选中的组 ID
- `selectedLinearElement` — 线性元素编辑器状态
- `editingGroupId` — 当前编辑的组
- `viewBackgroundColor` — 画布背景色
- `name` — 文档名称
- `croppingElementId` — 裁切元素 ID
- `activeLockedId` — 锁定 ID
- `lockedMultiSelections` — 多选锁定状态

### 3. 关键代码：History.perform()

`packages/excalidraw/history.ts:157-229` 是撤销/重做的核心：

```typescript
private perform(elements, appState, pop, push) {
  let historyDelta = pop(); // 从 undo/redo 栈弹出
  
  // 持续弹出直到找到可见变化（跳过纯选择变化等）
  while (historyDelta) {
    // 应用 Delta 到当前状态
    [nextElements, nextAppState, containsVisibleChange] = 
      historyDelta.applyTo(nextElements, nextAppState, prevSnapshot);
    
    // 计算新的 Delta（用于 redo 栈，且版本号更新）
    const delta = HistoryDelta.applyLatestChanges(
      historyDelta,
      prevElements,
      nextElements,
    );
    
    if (!delta.isEmpty()) {
      // 立即触发事件，用于协作同步
      this.store.scheduleMicroAction({
        action: CaptureUpdateAction.IMMEDIATELY,
        change,
        delta,
      });
    }
    
    // 将取反后的 Delta 推入对面的栈
    push(historyDelta);
    
    if (containsVisibleChange) {
      break; // 找到可见变化，停止
    }
    
    historyDelta = pop(); // 继续找下一个
  }
  
  return [nextElements, nextAppState];
}
```

### 4. 版本号特殊处理

**撤销/重做时不恢复 `version` 和 `versionNonce`**（`history.ts:32-34`）：

```typescript
{
  excludedProperties: new Set(["version", "versionNonce"]),
}
```

原因：协作场景下，每次撤销/重做都应产生新的版本，让其他客户端认为这是一个新的用户操作。

---

## 协作场景下的撤销机制

### 1. 协作时的核心原则

**每个人的历史栈是独立的**，但通过以下机制协调：

#### A. 远程更新不进入历史栈

远程更新使用 `CaptureUpdateAction.NEVER`，不会记录到本地历史：

`excalidraw-app/collab/Collab.tsx:804-813`:

```typescript
private handleRemoteSceneUpdate = (elements) => {
  this.excalidrawAPI.updateScene({
    elements,
    captureUpdate: CaptureUpdateAction.NEVER, // 永不撤销
  });
};
```

#### B. 本地撤销会同步给协作者

撤销时 `scheduleMicroAction` 使用 `IMMEDIATELY`，会触发 `onStoreIncrementEmitter` 通知协作模块同步：

`history.ts:198-204`:

```typescript
if (!delta.isEmpty()) {
  this.store.scheduleMicroAction({
    action: CaptureUpdateAction.IMMEDIATELY, // 立即同步
    change,
    delta,
  });
}
```

### 2. applyLatestChanges: 智能处理并发

这是多人协作撤销的关键机制。当本地撤销时，会先将历史 Delta 与当前最新状态（可能包含远程修改）进行合并：

`delta.ts:1301-1386`:

```typescript
public applyLatestChanges(prevElements, nextElements, modifierOptions) {
  // 对每个历史记录的元素：
  // 用当前最新的元素属性值更新 Delta
  
  const modifier = (prevElement, nextElement) => 
    (partial, partialType) => {
      let element;
      switch (partialType) {
        case "deleted": element = prevElement; break;
        case "inserted": element = nextElement; break;
      }
      
      // 用最新值更新 partial（除了 boundElements）
      for (const key of Object.keys(partial)) {
        if (key === "boundElements") continue;
        latestPartial[key] = element[key];
      }
      return latestPartial;
    };
  
  // 重新分发 Delta（added/removed/updated 可能互换）
  return ElementsDelta.create(added, removed, updated, {
    shouldRedistribute: true,
  });
}
```

### 3. 协作场景示例

#### 场景 1：不同元素的并发

```
用户 A：创建矩形 A（version=1）
用户 B：创建矩形 B（version=1）[远程更新，NEVER]

A 撤销：
  - 只撤销矩形 A（删除 A）
  - 矩形 B 保持不变
  - 结果：{A: deleted, B: visible}
```

#### 场景 2：同一元素不同属性

```
用户 A：设置矩形背景为红色（backgroundColor: red）
用户 B：设置同一矩形边框为黄色（strokeColor: yellow）[远程]

A 撤销：
  - 用 applyLatestChanges 合并最新状态
  - backgroundColor 恢复为透明
  - strokeColor 保持黄色（远程修改保留）
  - 结果：{backgroundColor: transparent, strokeColor: yellow}
```

#### 场景 3：同一元素同一属性

```
用户 A：背景色 red → blue（历史记录：red→blue）
用户 B：背景色 blue → yellow（远程，NEVER）

A 撤销：
  - applyLatestChanges 更新历史记录
  - 原记录：red → blue
  - 更新为：red → yellow（用最新值）
  - 撤销后：backgroundColor = red
  
再撤销：
  - 远程又改成 violet
  - 重做时记录更新为：violet → yellow
  - 重做后：backgroundColor = yellow
```

---

## 本地撤销会不会影响他人

### 短答案：**会，但通过智能合并保证数据一致性**

### 详细分析

#### 1. 撤销操作会广播

本地用户撤销时，会触发 `onStoreIncrementEmitter`，协作模块监听到后会同步给其他用户。

但关键是：**撤销产生的是新的操作（新版本号）**，而非真正回滚到过去。

#### 2. 不会出现的问题

**不会出现：** 用户 A 撤销后，用户 B 的历史栈也被回退。

**原因：** 每个客户端独立维护历史栈，撤销作为新操作同步给他人。

#### 3. 可能出现的冲突及处理

| 冲突场景 | 处理方式 | 结果 |
|---------|---------|------|
| A 撤销创建元素 | 广播"删除元素"操作 | 所有人看到元素消失 |
| A 撤销修改元素 | 广播"反向修改"操作 | 所有人看到属性恢复 |
| A 撤销删除元素 | 广播"恢复元素"操作 | 所有人看到元素恢复 |
| B 同时修改同一属性 | `applyLatestChanges` 合并 | 保留远程修改的属性 |

#### 4. 特殊情况：被远程删除的元素

如果用户 A 想撤销的元素已被用户 B 远程删除：

```
历史记录指向元素 X（已被远程删除）
    ↓
撤销时 applyTo 从 snapshot 中找不到 X
    ↓
跳过该历史记录（不产生可见变化）
    ↓
继续撤销下一条记录
```

这种情况下，撤销会"穿透"多条历史记录，直到找到可见变化。

---

## 关键测试用例说明

`packages/excalidraw/tests/history.test.tsx` 中的 `multiplayer undo/redo` 测试套件验证了所有协作场景：

| 测试用例 | 验证点 |
|---------|-------|
| `should not override remote changes on different elements` | 撤销不影响其他元素的远程修改 |
| `should not override remote changes on different properties` | 同一元素不同属性独立处理 |
| `should update history entries after remote changes on the same properties` | 同一属性冲突时智能更新历史记录 |
| `should redistribute deltas when element gets removed locally but is restored remotely` | 元素删除/恢复的并发处理 |
| `should iterate through the history when element change relates to remotely deleted element` | 被远程删除的元素会被跳过 |

---

## 总结

### 1. 历史栈核心特点

| 特性 | 实现方式 |
|-----|---------|
| 增量记录 | `StoreDelta` 双向记录变更前后 |
| 独立栈 | 每个客户端独立维护 undo/redo 栈 |
| 版本隔离 | 撤销/重做产生新版本号，不回溯历史 |
| 智能合并 | `applyLatestChanges` 处理并发修改 |

### 2. 协作撤销范围

**本地撤销只会撤销当前用户自己的操作，但：**

1. **撤销结果会同步给他人**（作为新操作）
2. **远程已修改的属性会被保留**（通过 Delta 合并）
3. **远程已删除的元素无法恢复**（撤销会跳过）

### 3. 设计哲学

参考 Figma 的协作架构：
- 撤销不是"时间旅行"，而是"执行反向操作"
- 每次撤销产生新的变更，让协作引擎按正常流程处理
- 这避免了复杂的分布式一致性问题
