# Excalidraw 协作下撤销范围界定

## 核心问题

多人协作时，用户 A 执行撤销操作：

1. **撤销范围如何界定？** — 只撤销 A 自己的操作，还是影响所有人？
2. **哪些状态会被还原？** — 元素属性、绑定关系、分组、Frame 从属关系？
3. **与远程操作冲突时如何处理？** — 不同元素、同元素不同属性、同属性冲突？

---

## 基础机制

### 1. 历史栈架构

每个客户端独立维护两个栈：

```
Undo Stack: [历史记录 1, 历史记录 2, ...]  ← 可撤销的操作
Redo Stack: [历史记录 1, 历史记录 2, ...]  ← 已撤销的操作
```

**每条历史记录** = `StoreDelta` = 操作的双向增量（deleted → inserted）：

```
{
  elements: {
    added: {},        // 新增元素（isDeleted: false）
    removed: {},      // 删除元素（isDeleted: true）
    updated: {}       // 修改元素（isDeleted 状态不变）
  },
  appState: { ... }   // 被追踪的应用状态
}
```

### 2. 撤销核心流程

```
用户 A 执行撤销
    ↓
1. applyLatestChanges → 用当前最新状态（含远程修改）更新历史记录
    ↓
2. applyTo → 将反向增量应用到当前状态
    ↓
3. scheduleMicroAction(IMMEDIATELY) → 触发事件，同步给协作者
    ↓
4. 将操作推入 redoStack
    ↓
其他客户端收到同步：
    → 作为新操作处理（CaptureUpdateAction.NEVER，不进入对方历史栈）
```

### 3. 关键设计原则

| 原则 | 说明 |
|-----|------|
| **独立历史栈** | 每个客户端独立维护，不共享 |
| **撤销 = 反向新操作** | 不是时间回滚，而是执行反向操作，产生新版本号 |
| **applyLatestChanges** | 撤销前先用远程更新刷新历史记录 |
| **远程操作 NEVER** | 远程更新不进入本地历史栈 |

---

## 场景一：不同元素的并发修改

### 描述

- 用户 A：创建/修改元素 X
- 用户 B：创建/修改元素 Y

### 回放结果

**A 撤销 → 只影响 X，Y 保持不变**

### 测试用例佐证

**测试 1: 不同元素互不干扰**

`packages/excalidraw/tests/history.test.tsx:2125-2167`

```typescript
it("should not override remote changes on different elements", async () => {
  // 本地创建元素并设置背景色
  UI.createElement("rectangle", { x: 10 });
  togglePopover("Background");
  UI.clickOnTestId("color-red");
  
  // 远程更新：同时修改另一元素
  API.updateScene({
    elements: [
      h.elements[0],  // 本地元素不变
      rect,           // 远程新增元素
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 撤销
  Keyboard.undo();
  
  // 结果：本地元素恢复，但远程元素不受影响
  expect(h.elements).toEqual([
    expect.objectContaining({ id: h.elements[0].id, backgroundColor: transparent }),
    expect.objectContaining({ id: rect.id }),  // 远程元素保留
  ]);
});
```

### 结论

| 维度 | 结果 |
|-----|------|
| 本地元素 | ✅ 被撤销 |
| 远程元素 | ✅ 完全保留 |
| 协作者看到 | 本地元素消失/恢复，远程元素不变 |

---

## 场景二：同元素不同属性

### 描述

- 用户 A：修改元素 X 的属性 A（如 `backgroundColor`）
- 用户 B：修改同一元素 X 的属性 B（如 `strokeColor`）

### 回放结果

**A 撤销 → 属性 A 恢复，属性 B 保留远程修改**

### 测试用例佐证

**测试 2: 同一元素不同属性独立处理**

`packages/excalidraw/tests/history.test.tsx:2169-2203`

```typescript
it("should not override remote changes on different properties", async () => {
  // 本地创建元素并设置背景色为 red
  UI.createElement("rectangle", { x: 10 });
  togglePopover("Background");
  UI.clickOnTestId("color-red");
  
  // 远程更新：设置同一元素的边框色为 yellow
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], {
        backgroundColor: red,      // 保持本地值
        strokeColor: yellow,       // 远程新增修改
      }),
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 撤销
  Keyboard.undo();
  
  // 结果：背景色恢复 transparent，但边框色保持 yellow
  expect(h.elements).toEqual([
    expect.objectContaining({
      backgroundColor: transparent,  // 本地撤销生效
      strokeColor: yellow,           // 远程修改保留
    }),
  ]);
});
```

### 实现原理

`applyLatestChanges`（`packages/element/src/delta.ts:1301-1386`）：

```typescript
public applyLatestChanges(prevElements, nextElements, modifierOptions) {
  const modifier = (prevElement, nextElement) => 
    (partial, partialType) => {
      let element = partialType === "deleted" ? prevElement : nextElement;
      
      const latestPartial = {};
      for (const key of Object.keys(partial)) {
        // 用当前最新值更新每个属性
        latestPartial[key] = element[key];
      }
      return latestPartial;
    };
  
  // 逐个属性更新历史记录
  // backgroundColor: red → transparent（历史记录不变）
  // strokeColor: 被 remote 更新为 yellow，但历史记录中没有这个属性
  // 所以撤销时不影响 strokeColor
}
```

### 结论

| 维度 | 结果 |
|-----|------|
| 本地修改的属性 | ✅ 被撤销 |
| 远程修改的不同属性 | ✅ 完全保留 |
| 协作者看到 | 属性 A 恢复，属性 B 保持黄色 |

---

## 场景三：同元素同一属性冲突

### 描述

- 用户 A：将元素 X 的属性 P 从 `old` 改为 `localNew`
- 用户 B：将同一元素 X 的同一属性 P 从 `localNew` 改为 `remoteNew`

### 回放结果

**A 撤销 → 历史记录被更新，撤销到 `old`，而不是 `localNew`**

### 测试用例佐证

**测试 3: 同一属性冲突时更新历史记录**

`packages/excalidraw/tests/history.test.tsx:2205-2367`

```typescript
it("should update history entries after remote changes on the same properties", async () => {
  // 本地创建元素（backgroundColor: transparent）
  UI.createElement("rectangle", { x: 10 });
  
  // 本地修改：transparent → red
  togglePopover("Background");
  UI.clickOnTestId("color-red");
  
  // 远程修改：red → yellow
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], { backgroundColor: yellow }),
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 历史记录原本是：transparent → red
  // applyLatestChanges 更新为：transparent → yellow
  
  // 撤销
  Keyboard.undo();
  
  // 结果：yellow → transparent（撤销到历史记录的 deleted 端）
  expect(h.elements).toEqual([
    expect.objectContaining({ backgroundColor: transparent }),
  ]);
  
  // 重做
  Keyboard.redo();
  
  // 结果：transparent → yellow（使用更新后的历史记录）
  expect(h.elements).toEqual([
    expect.objectContaining({ backgroundColor: yellow }),
  ]);
});
```

### 实现原理

历史记录更新流程：

```
初始状态:  { backgroundColor: transparent }
A 操作:    { backgroundColor: red }
  → 历史记录: { deleted: { transparent }, inserted: { red } }

B 远程修改: { backgroundColor: yellow }
  → 当前状态: yellow

A 撤销:
  applyLatestChanges 刷新历史记录:
    { 
      deleted: { transparent },  // 不变（prevElement）
      inserted: { yellow }        // 更新为 nextElement 的最新值
    }
  
  应用反向增量: yellow → transparent
```

### 结论

| 维度 | 结果 |
|-----|------|
| 本地修改被远程覆盖 | ✅ 历史记录动态更新 |
| 撤销效果 | 撤销到历史记录的起点（old），而不是本地修改值 |
| 重做效果 | 使用更新后的远程值（remoteNew） |
| 协作者看到 | 该属性值从 remoteNew 变为 old |

---

## 场景四：分组（groupIds）的特殊处理

### 描述

- 用户 A：将元素 X、Y 加入分组 A
- 用户 B：将同一元素 X、Y 加入分组 B

### 回放结果

**A 撤销 → 远程分组被覆盖，但重做时恢复**

### 测试用例佐证

**测试 4: 远程分组被撤销覆盖**

`packages/excalidraw/tests/history.test.tsx:2371-2423`

```typescript
it("should override remotely added groups on undo, but restore them on redo", async () => {
  // 本地创建两个元素
  const rect1 = API.createElement({ type: "rectangle" });
  const rect2 = API.createElement({ type: "rectangle" });
  
  // 本地操作：加入分组 A
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], { groupIds: ["A"] }),
      newElementWith(h.elements[1], { groupIds: ["A"] }),
    ],
    captureUpdate: CaptureUpdateAction.IMMEDIATELY,
  });
  
  // 远程操作：加入分组 B
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], { groupIds: ["A", "B"] }),
      newElementWith(h.elements[1], { groupIds: ["A", "B"] }),
      rect3,  // 远程新增元素，groupIds: ["B"]
      rect4,
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 撤销
  Keyboard.undo();
  
  // 结果：本地分组 A 被撤销，远程分组 B 也被覆盖
  expect(h.elements).toEqual([
    expect.objectContaining({ id: rect1.id, groupIds: [] }),  // 覆盖了 ["A", "B"]
    expect.objectContaining({ id: rect2.id, groupIds: [] }),
    expect.objectContaining({ id: rect3.id, groupIds: ["B"] }),  // 远程元素保留
    expect.objectContaining({ id: rect4.id, groupIds: ["B"] }),
  ]);
  
  // 重做
  Keyboard.redo();
  
  // 结果：分组 A 恢复，远程分组 B 也恢复
  expect(h.elements).toEqual([
    expect.objectContaining({ id: rect1.id, groupIds: ["A", "B"] }),
    expect.objectContaining({ id: rect2.id, groupIds: ["A", "B"] }),
    expect.objectContaining({ id: rect3.id, groupIds: ["B"] }),
    expect.objectContaining({ id: rect4.id, groupIds: ["B"] }),
  ]);
});
```

### 为什么分组会被覆盖？

代码注释说明（`packages/excalidraw/tests/history.test.tsx:2369-2370`）：

```typescript
// TODO: #7348 ideally we should not override, but since the order of groupIds matters,
// right now we cannot ensure that with postprocessed groupIds the order will be consistent
// after series or undos/redos, we don't postprocess them at all
//
// 理想情况下不应该覆盖，但由于 groupIds 顺序很重要，
// 目前无法保证后处理后顺序一致，所以不做合并处理
```

`applyLatestChanges` 中 `groupIds` 会被完全替换（`delta.ts:1340-1342`）：

```typescript
for (const key of Object.keys(partial)) {
  default:
    latestPartial[key] = element[key];  // 完全替换
}
```

### 结论

| 维度 | 结果 |
|-----|------|
| 本地分组 | ✅ 被撤销 |
| 远程分组 | ❌ 被覆盖（但重做时恢复） |
| 远程新增元素的分组 | ✅ 保留 |
| 协作者看到 | 分组从 `["A", "B"]` 变成 `[]`，重做后恢复 |

---

## 场景五：线性元素的 points 数组

### 描述

- 用户 A：创建箭头，有 3 个点
- 用户 B：同一箭头，远程增加到 5 个点

### 回放结果

**A 撤销 → 远程 points 被覆盖，但重做时恢复**

### 测试用例佐证

**测试 5: 远程 points 被撤销覆盖**

`packages/excalidraw/tests/history.test.tsx:2425-2503`

```typescript
it("should override remotely added points on undo, but restore them on redo", async () => {
  // 本地创建 3 点箭头
  UI.clickTool("arrow");
  mouse.click(0, 0);
  mouse.click(10, 10);
  mouse.click(20, 20);
  Keyboard.keyPress(KEYS.ENTER);
  
  // 远程更新：增加到 5 个点
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], {
        points: [
          pointFrom(0, 0),
          pointFrom(5, 5),    // 远程新增
          pointFrom(10, 10),
          pointFrom(15, 15),  // 远程新增
          pointFrom(20, 20),
        ],
      }),
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 撤销
  Keyboard.undo();
  
  // 结果：远程 points 被覆盖，回到本地的 2 个点
  expect(h.elements).toEqual([
    expect.objectContaining({
      points: [
        [0, 0],
        // 覆盖所有远程点
        [10, 10],
      ],
    }),
  ]);
  
  // 重做
  Keyboard.redo();
  
  // 结果：远程的 5 个点恢复
  expect(h.elements).toEqual([
    expect.objectContaining({
      points: [
        [0, 0],
        [5, 5],
        [10, 10],
        [15, 15],
        [20, 20],
      ],
    }),
  ]);
});
```

### 为什么 points 不合并？

代码注释说明（`packages/element/src/delta.ts:2025-2027`）：

```typescript
// don't diff the points as:
// - we can't ensure the multiplayer order consistency without fractional index on each point
// - we prefer to not merge the points, as it might just lead to unexpected / inconsistent results
//
// 不做 points 差异计算，因为：
// - 每个点没有分数索引，无法保证多人协作时的顺序一致性
// - 不合并可能导致意外/不一致的结果
```

### 结论

| 维度 | 结果 |
|-----|------|
| 本地 points | ✅ 被撤销 |
| 远程新增的 points | ❌ 被覆盖（重做时恢复） |
| 协作者看到 | points 从 5 个变成 2 个，重做后恢复 5 个 |

---

## 场景六：绑定关系（bindings）

### 6.1 容器 ↔ 文本绑定

#### 测试 6: 远程绑定的文本在容器恢复时被保留

`packages/excalidraw/tests/history.test.tsx:4005-4109`

```typescript
it("should preserve latest remotely added binding and unbind previous one when the container is added through the history", async () => {
  // 本地创建容器
  API.updateScene({
    elements: [container],
    captureUpdate: CaptureUpdateAction.IMMEDIATELY,
  });
  
  // 远程：绑定文本 A 到容器
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], { boundElements: [{ id: text.id, type: "text" }] }),
      newElementWith(text, { containerId: container.id }),
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 撤销（容器删除，文本解绑）
  Keyboard.undo();
  // 结果：container.isDeleted=true, text.containerId=null
  
  // 远程：绑定新文本 B 到容器，同时恢复容器
  const remoteText = API.createElement({ type: "text", containerId: container.id });
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], {
        boundElements: [{ id: remoteText.id, type: "text" }],
        isDeleted: false,
      }),
      remoteText,
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 重做（恢复本地容器）
  Keyboard.redo();
  
  // 结果：远程的新绑定被保留，旧文本未重新绑定
  expect(h.elements).toEqual([
    expect.objectContaining({
      id: container.id,
      boundElements: [{ id: remoteText.id, type: "text" }],  // 远程绑定保留
      isDeleted: false,
    }),
    expect.objectContaining({
      id: text.id,
      containerId: null,  // 旧文本未绑定
    }),
    expect.objectContaining({
      id: remoteText.id,
      containerId: container.id,  // 新文本绑定
    }),
  ]);
});
```

### 6.2 箭头绑定

#### 测试 7: 远程移动元素后，箭头重做时重新计算点

`packages/excalidraw/tests/history.test.tsx:5106-5195`

```typescript
it("should update bound element points when rectangle was remotely moved and arrow is added back through the history", async () => {
  // 创建两个矩形，绑定箭头
  UI.clickTool("arrow");
  mouse.down(0, 0);
  mouse.moveTo(25, 0);
  mouse.moveTo(47, 0);
  mouse.up(47, 0);  // 连接两个矩形
  
  const arrowId = h.elements[2].id;
  
  // 撤销箭头
  Keyboard.undo();
  
  // 远程移动目标矩形
  API.updateScene({
    elements: [
      h.elements[0],
      newElementWith(h.elements[1], { x: 500, y: -500 }),  // 远程移动
      h.elements[2],
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 重做箭头
  Keyboard.redo();
  
  // 结果：箭头点被重新计算，指向新位置
  const points = (h.elements[2] as ExcalidrawLinearElement).points[1];
  expect([
    roundToNearestHundred(points[0]),
    roundToNearestHundred(points[1]),
  ]).toEqual([500, -400]);  // 指向远程移动后的新位置
});
```

### 6.3 绑定关系边界情况

| 场景 | 结果 | 测试 |
|-----|------|-----|
| 远程删除被绑定元素 | 自动解绑 | `packages/excalidraw/tests/history.test.tsx:4217-4329` |
| 容器恢复时，远程新增的绑定 | 保留新绑定，旧文本不重新绑定 | `packages/excalidraw/tests/history.test.tsx:4005-4109` |
| 绑定目标被远程移动 | 箭头重做时重新计算点 | `packages/excalidraw/tests/history.test.tsx:5106-5195` |
| Frame 被远程删除，子元素恢复 | 不重新绑定到 Frame | `packages/excalidraw/tests/history.test.tsx:5221-5304` |

### 绑定关系结论

| 维度 | 结果 |
|-----|------|
| 远程新增的绑定 | ✅ 保留（最新绑定优先） |
| 远程删除的绑定 | ✅ 自动解绑 |
| 绑定目标被远程移动 | ✅ 箭头重做时重新计算点 |
| Frame 被远程删除 | ❌ 子元素重做时不重新绑定 |

---

## 场景七：Frame 从属关系

### 描述

- 用户 A：将元素 X 放入 Frame F
- 用户 B：远程删除 Frame F

### 回放结果

**A 重做 → 元素 X 不重新绑定到已删除的 Frame**

### 测试用例佐证

**测试 8: Frame 远程删除后，子元素不重新绑定**

`packages/excalidraw/tests/history.test.tsx:5221-5304`

```typescript
it("should not rebind frame child with frame when frame was remotely deleted and frame child is added back through the history", async () => {
  // 本地创建 Frame 和矩形
  API.updateScene({ elements: [frame], captureUpdate: CaptureUpdateAction.NEVER });
  API.updateScene({ elements: [rect, h.elements[0]], captureUpdate: CaptureUpdateAction.IMMEDIATELY });
  
  // 本地：将矩形放入 Frame
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], { frameId: frame.id }),
      h.elements[1],
    ],
    captureUpdate: CaptureUpdateAction.IMMEDIATELY,
  });
  
  // 撤销两次（矩形移除 + 矩形删除）
  Keyboard.undo();
  Keyboard.undo();
  
  // 远程：删除 Frame
  API.updateScene({
    elements: [
      h.elements[0],
      newElementWith(h.elements[1], { isDeleted: true }),
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 重做两次
  Keyboard.redo();
  Keyboard.redo();
  
  // 结果：矩形恢复，但不绑定到已删除的 Frame
  expect(h.elements).toEqual([
    expect.objectContaining({
      id: rect.id,
      frameId: null,  // 不重新绑定
      isDeleted: false,
    }),
    expect.objectContaining({
      id: frame.id,
      isDeleted: true,
    }),
  ]);
});
```

### Frame 结论

| 维度 | 结果 |
|-----|------|
| Frame 存在时 | ✅ 子元素正常绑定/解绑 |
| Frame 被远程删除后 | ❌ 子元素重做时不重新绑定 |
| 协作者看到 | 子元素恢复但不在 Frame 内 |

---

## 场景八：远程删除元素

### 8.1 单个元素被远程删除

#### 测试 9: 元素被远程删除，撤销跳过不可见变更

`packages/excalidraw/tests/history.test.tsx:2584-2636`

```typescript
it("should iterate through the history when when element change relates to remotely deleted element", async () => {
  // 本地创建矩形，设置背景色为 red
  UI.createElement("rectangle", { x: 10 });
  togglePopover("Background");
  UI.clickOnTestId("color-red");
  
  expect(API.getUndoStack().length).toBe(2);  // 创建 + 修改颜色
  
  // 远程：删除该元素，同时修改背景色为 yellow
  API.updateScene({
    elements: [
      newElementWith(h.elements[0], {
        backgroundColor: yellow,
        isDeleted: true,
      }),
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 撤销（应该跳过颜色修改，直接撤销创建）
  Keyboard.undo();
  
  // undoStack 从 2 → 0，说明跳过了颜色修改
  expect(API.getUndoStack().length).toBe(0);
  expect(API.getRedoStack().length).toBe(2);
  
  // 结果：元素保持删除状态，但颜色被修改
  expect(h.elements).toEqual([
    expect.objectContaining({
      backgroundColor: transparent,  // 颜色从 yellow → transparent
      isDeleted: true,               // 仍保持删除状态
    }),
  ]);
});
```

### 8.2 多个元素被远程删除

#### 测试 10: 部分元素被删除，撤销部分生效

`packages/excalidraw/tests/history.test.tsx:2638-2713`

```typescript
it("should iterate through the history when element changes relate only to remotely deleted elements", async () => {
  const rect1 = UI.createElement("rectangle", { x: 10 });
  const rect2 = UI.createElement("rectangle", { x: 20 });
  
  // 修改 rect2 颜色
  togglePopover("Background");
  UI.clickOnTestId("color-red");
  
  const rect3 = UI.createElement("rectangle", { x: 30, y: 30 });
  // 移动 rect3
  mouse.downAt(35, 35);
  mouse.moveTo(55, 55);
  mouse.upAt(55, 55);
  
  expect(API.getUndoStack().length).toBe(5);
  
  // 远程：删除 rect2 和 rect3
  API.updateScene({
    elements: [
      h.elements[0],
      newElementWith(h.elements[1], { isDeleted: true }),
      newElementWith(h.elements[2], { isDeleted: true }),
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 撤销
  Keyboard.undo();
  
  // 跳过移动 rect3（已删除），跳到选中 rect1
  expect(API.getUndoStack().length).toBe(1);
  expect(API.getRedoStack().length).toBe(4);
  
  expect(API.getSelectedElements()).toEqual([
    expect.objectContaining({ id: rect1.id }),  // 选中 rect1
  ]);
  
  expect(h.elements).toEqual([
    expect.objectContaining({ id: rect1.id, isDeleted: false }),
    expect.objectContaining({
      id: rect2.id,
      isDeleted: true,
      backgroundColor: transparent,  // 颜色被修改但元素仍删除
    }),
    expect.objectContaining({
      id: rect3.id,
      isDeleted: true,
      x: 30, y: 30,  // 位置被修改但元素仍删除
    }),
  ]);
});
```

### 8.3 AppState 相关的远程删除

#### 测试 11: 选中元素被远程删除，撤销跳过

`packages/excalidraw/tests/history.test.tsx:2715-2803`

```typescript
it("should iterate through the history when selected elements relate only to remotely deleted elements", async () => {
  // 创建 3 个元素
  const rect1 = API.createElement({ type: "rectangle", x: 10 });
  const rect2 = API.createElement({ type: "rectangle", x: 20 });
  const rect3 = API.createElement({ type: "rectangle", x: 30 });
  
  // 依次选中
  mouse.select(rect1);
  mouse.select([rect2, rect3]);
  
  expect(API.getUndoStack().length).toBe(3);
  expect(API.getSelectedElements()).toEqual([
    expect.objectContaining({ id: rect2.id }),
    expect.objectContaining({ id: rect3.id }),
  ]);
  
  // 远程：删除 rect2 和 rect3
  API.updateScene({
    elements: [
      h.elements[0],
      newElementWith(h.elements[1], { isDeleted: true }),
      newElementWith(h.elements[2], { isDeleted: true }),
    ],
    captureUpdate: CaptureUpdateAction.NEVER,
  });
  
  // 撤销
  Keyboard.undo();
  
  // 跳过选中 rect2/rect3（已删除），跳到选中 rect1
  expect(API.getUndoStack().length).toBe(1);
  expect(API.getRedoStack().length).toBe(2);
  expect(API.getSelectedElements()).toEqual([
    expect.objectContaining({ id: rect1.id }),
  ]);
});
```

#### 测试 12: 分组/编辑组被远程删除

`packages/excalidraw/tests/history.test.tsx:2805-2900`（分组）和 `2902-2972`（编辑组）

```typescript
// 分组测试核心逻辑：
// - 撤销时，如果所有元素都被远程删除，分组选择会被跳过
// - 重做时，如果元素被远程恢复，分组选择会生效
```

### 远程删除结论

| 场景 | 结果 |
|-----|------|
| 元素被远程删除 | 撤销跳过该元素的可见变更（属性修改），但仍应用属性变化 |
| 选中元素被删除 | 撤销跳过多条记录，直到找到可见变更 |
| 分组/编辑组被删除 | 撤销跳过相关选择状态，直到找到有效元素 |
| 部分元素有效 | 撤销部分生效，有效元素的变更被处理 |
| 协作者看到 | 被删除元素保持删除状态，但其属性可能被修改 |

---

## 总结：协作下的撤销范围

### 1. 按元素维度

| 场景 | 本地撤销效果 | 远程修改保留？ | 协作者看到 |
|-----|-------------|---------------|-----------|
| **不同元素** | 只撤销本地元素 | ✅ 完全保留 | 本地元素变化，远程元素不变 |
| **同元素不同属性** | 撤销本地修改的属性 | ✅ 保留远程属性 | 本地属性恢复，远程属性保持 |
| **同元素同一属性** | 撤销到历史起点 | ❌ 被覆盖 | 属性值从远程值变成本地历史值 |
| **分组 groupIds** | 撤销本地分组 | ❌ 被覆盖（重做恢复） | 分组从 `["A","B"]` 变成 `[]` |
| **线性元素 points** | 撤销本地 points | ❌ 被覆盖（重做恢复） | points 数量变化 |

### 2. 按关系维度

| 关系类型 | 本地撤销/重做效果 | 远程修改保留？ |
|---------|-----------------|---------------|
| **容器↔文本绑定** | 恢复时保留最新绑定 | ✅ 新绑定优先，旧文本解绑 |
| **箭头绑定** | 重做时重新计算点 | ✅ 目标移动后，箭头重新计算 |
| **Frame 从属** | Frame 删除后不重新绑定 | ❌ 子元素重做时无 Frame |

### 3. 远程删除的边界

| 情况 | 处理方式 |
|-----|---------|
| **元素被远程删除** | 撤销跳过可见变更，但属性变化仍应用 |
| **选中元素被删除** | 撤销跳过多条记录，找有效元素 |
| **分组元素被删除** | 撤销跳过分组选择 |
| **部分元素有效** | 撤销部分生效 |
| **重做时元素恢复** | 选择/分组状态恢复生效 |

### 4. 关键原则修正

**❌ 错误结论：远程修改都会保留**

**✅ 正确结论：远程修改的保留取决于属性类型**

| 属性类型 | 远程修改保留？ | 原因 |
|---------|---------------|------|
| 独立属性（backgroundColor, x, y 等） | ✅ 保留 | 按属性独立处理 |
| 数组引用（boundElements） | ✅ 保留 | 专门不更新 `boundElements` |
| 有序数组（groupIds, points） | ❌ 不保留 | 无法保证顺序一致性，直接覆盖 |
| Frame 从属关系 | 视情况 | Frame 删除后不重新绑定 |

### 5. 本地撤销是否影响他人

**短答案：会影响，但不是"时间回滚"，而是"同步新操作"**

| 维度 | 对协作者的影响 |
|-----|---------------|
| 撤销创建元素 | 协作者看到该元素被删除 |
| 撤销修改属性 | 协作者看到该属性恢复旧值 |
| 远程修改的独立属性 | 协作者仍看到自己的修改 |
| 远程修改的 groupIds/points | 协作者的修改被覆盖（重做时恢复） |
| 远程删除的元素 | 协作者看到元素保持删除，但其属性被修改 |

**核心设计：**
- 撤销 = 执行反向操作 = 产生新版本号
- 协作者收到的是新操作，按 `CaptureUpdateAction.NEVER` 处理
- 不会影响协作者的历史栈

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
| 容器文本绑定 | `history.test.tsx:3884, 3945, 4005` |
| 远程删除绑定 | `history.test.tsx:4217, 4274` |
| 箭头绑定 | `history.test.tsx:4896, 4988, 5106` |
| Frame 从属 | `history.test.tsx:5221` |
