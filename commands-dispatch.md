# Excalidraw 指令派发机制详解

## 目录
- [概述](#概述)
- [一、入口来源](#一入口来源)
- [二、命令类型分层](#二命令类型分层)
- [三、非 Action 命令完整盘点（新增）](#三非-action-命令完整盘点新增)
- [四、动作注册](#四动作注册)
- [五、快捷键解析](#五快捷键解析)
- [六、运行时上下文判定](#六运行时上下文判定)
- [七、执行落点](#七执行落点)
- [八、端到端完整示例](#八端到端完整示例)
- [九、核心文件索引](#九核心文件索引)

---

## 概述

Excalidraw 的指令派发系统是一个**"一次定义，多处复用"**的统一动作执行架构。它将**命令面板**、**右键菜单**、**工具栏**、**快捷键**四个完全独立的入口点，通过统一的 `Action` 接口和 `ActionManager` 引擎连接到同一个执行落点。

**核心设计思想：**
- 动作定义与 UI 表现分离
- 所有入口共享相同的业务逻辑（`perform` 函数）
- 不同入口有不同的上下文判定机制
- 命令面板支持两种命令类型：Action 命令和纯 UI 命令
- 追踪来源但不改变核心执行路径

```
┌─────────────────────────────────────────────────────────────────────────┐
│                            四个独立入口                                   │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐           │
│  │   命令面板     │  │   右键菜单     │  │    工具栏      │  快捷键    │
│  │ Cmd+Shift+P   │  │  RightClick    │  │  Toolbar      │ KeyDown    │
│  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘  ──┬────  │
└───────────┼─────────────────────┼─────────────────────┼────────────────┼───┘
            │                     │                     │                │
            ▼                     ▼                     ▼                ▼
  ┌───────────────────────┐  predicate 过滤     renderAction渲染    keyTest匹配
  │  固定动作集合+额外命令 │                     PanelComponent    (键盘事件)
  │  (非全部Action遍历)   │
  │  ┌─────────────────┐ │
  │  │ Layer 1: Action │ │ → 进 action.perform 执行业务逻辑
  │  │   命令          │ │ → 经 executeAction 统一入口
  │  │                 │ │ → 有统一埋点
  │  ├─────────────────┤ │
  │  │ Layer 2: 非     │ │ → 直接执行界面逻辑（打开弹窗/侧边栏）
  │  │   Action 命令   │ │ → 不经过 Action 系统
  │  │                 │ │ → 无统一埋点
  │  └─────────────────┘ │
  └───────────┬───────────┘
              │
              ▼
    ┌─────────────────────────┐
    │   Action 定义           │
    │  (一次定义)             │
    │  - name                │
    │  - perform()           │ ← 核心业务逻辑唯一定义
    │  - keyTest()           │
    │  - predicate()         │
    │  - PanelComponent      │
    │  - viewMode            │
    └─────────────┬───────────┘
                  │
                  ▼
    ┌─────────────────────────┐
    │   ActionManager         │
    │  (统一引擎)             │
    │  - executeAction()      │ ← 命令面板/右键菜单入口
    │  - handleKeyDown()      │ ← 快捷键专属入口
    │  - renderAction()       │ ← 工具栏专属入口
    └─────────────┬───────────┘
                  │
                  ▼
    ┌─────────────────────────┐
    │     updater()           │ ← 统一执行落点
    │  更新 elements           │
    │  更新 appState           │
    └─────────────────────────┘
```

---

## 一、入口来源

Excalidraw 有四个独立的动作触发入口，它们最终都指向同一个 Action 执行引擎，但触发路径和判定机制有显著差异。

### 1.1 命令面板 (Command Palette)

**触发方式：** `Cmd/Ctrl + Shift + P` 或 `Cmd/Ctrl + /`

**位置：** `packages/excalidraw/components/CommandPalette/CommandPalette.tsx`

**核心机制 - 固定动作集合 + 额外命令：**

命令面板**不是遍历全部 Action**，而是手动构建了几个固定的动作集合，再加上额外的自定义命令，最后进行可用性过滤。命令分为两层：

```typescript
// 第 1 步：构建 4 个固定的动作集合（Layer 1: Action 命令）
const elementsCommands: CommandPaletteItem[] = [
  actionManager.actions.group,
  actionManager.actions.ungroup,
  actionManager.actions.cut,
  actionManager.actions.copy,
  actionManager.actions.deleteSelectedElements,
  // ... 约 30 个手动列出的元素相关 Action
].map((action: Action) => actionToCommand(action, DEFAULT_CATEGORIES.elements));

const toolCommands: CommandPaletteItem[] = [
  actionManager.actions.toggleHandTool,
  actionManager.actions.setFrameAsActiveTool,
  actionManager.actions.toggleLassoTool,
].map((action) => actionToCommand(action, DEFAULT_CATEGORIES.tools));

const editorCommands: CommandPaletteItem[] = [
  actionManager.actions.undo,
  actionManager.actions.redo,
  actionManager.actions.zoomIn,
  // ... 约 15 个编辑器相关 Action
].map((action) => actionToCommand(action, DEFAULT_CATEGORIES.editor));

const exportCommands: CommandPaletteItem[] = [
  actionManager.actions.saveToActiveFile,
  actionManager.actions.saveFileToDisk,
  actionManager.actions.copyAsPng,
  actionManager.actions.copyAsSvg,
].map((action) => actionToCommand(action, DEFAULT_CATEGORIES.export));

// 第 2 步：合并固定动作集合 + 额外的自定义命令（Layer 2: 非 Action 命令）
commandsFromActions = [
  ...elementsCommands,      // 元素操作命令（Layer 1）
  ...editorCommands,        // 编辑器命令（Layer 1）
  {
    // clearCanvas 自定义命令（Layer 2：直接执行 UI 逻辑）
    label: getActionLabel(actionClearCanvas),
    icon: getActionIcon(actionClearCanvas),
    viewMode: false,
    perform: () => {
      editorJotaiStore.set(activeConfirmDialogAtom, "clearCanvas");
    },
  },
  {
    // exportImage 自定义命令（Layer 2：直接执行 UI 逻辑）
    label: t("buttons.exportImage"),
    category: DEFAULT_CATEGORIES.export,
    icon: ExportImageIcon,
    perform: () => {
      setAppState({ openDialog: { name: "imageExport" } });
    },
  },
  ...exportCommands,        // 导出命令（Layer 1）
];

// 第 3 步：additionalCommands - 与 Action 无关的纯自定义命令（主要是 Layer 2）
const additionalCommands: CommandPaletteItem[] = [
  {
    // Library（Layer 2：打开/关闭侧边栏）
    label: t("toolBar.library"),
    category: DEFAULT_CATEGORIES.app,
    icon: LibraryIcon,
    viewMode: false,
    perform: () => {
      if (uiAppState.openSidebar) {
        setAppState({ openSidebar: null });
      } else {
        setAppState({
          openSidebar: {
            name: DEFAULT_SIDEBAR.name,
            tab: DEFAULT_SIDEBAR.defaultTab,
          },
        });
      }
    },
  },
  {
    // Search（混合 Layer：内部调用 executeAction）
    label: t("search.title"),
    category: DEFAULT_CATEGORIES.app,
    icon: searchIcon,
    viewMode: true,
    perform: () => {
      actionManager.executeAction(actionToggleSearchMenu);
    },
  },
  // ... 更多形状工具命令（直接 setActiveTool）
];

// 第 4 步：最终合并 + 可用性过滤
const allCommands = [
  ...commandsFromActions,     // 来自 Action 的命令
  ...additionalCommands,      // 额外自定义命令
  ...(customCommandPaletteItems || []),  // 用户自定义命令
].filter(isCommandAvailable);  // 可用性过滤
```

**actionToCommand 转换器（Layer 1 专用）：**

```typescript
const actionToCommand = (
  action: Action,
  category: string,
  transformer?: (command: CommandPaletteItem, action: Action) => CommandPaletteItem,
): CommandPaletteItem => {
  const command: CommandPaletteItem = {
    label: getActionLabel(action),
    icon: getActionIcon(action),
    category,
    shortcut: getShortcutFromShortcutName(action.name as ShortcutName),
    keywords: action.keywords,
    predicate: action.predicate,          // 直接复制 predicate
    viewMode: action.viewMode,            // 直接复制 viewMode
    perform: () => {
      actionManager.executeAction(action, "commandPalette");  // ✅ 走 executeAction → action.perform
    },
  };
  return transformer ? transformer(command, action) : command;
};
```

**关键点：**
- ❌ **不是遍历所有 Action**，而是手动列出的固定集合
- ✅ 命令集合 = 4 类 Action 子集 + 额外自定义命令
- ✅ 可用性判定在展示阶段通过 `isCommandAvailable` 过滤

### 1.2 右键菜单 (Context Menu)

**触发方式：** 画布或元素上右键点击

**位置：** `packages/excalidraw/components/ContextMenu.tsx`

**工作原理：**
- 接收 `items: ContextMenuItems` 参数（Action 数组）
- 渲染前通过 `predicate` 过滤不可用项
- 点击时调用 `actionManager.executeAction(item, "contextMenu")`

**关键代码：**

```typescript
// 过滤不可用项
const filteredItems = items.reduce((acc: ContextMenuItem[], item) => {
  if (
    item &&
    (item === CONTEXT_MENU_SEPARATOR ||
      !item.predicate ||  // 如果没有 predicate，默认可用
      item.predicate(
        elements,
        appState,
        actionManager.app.props,
        actionManager.app,
      ))
  ) {
    acc.push(item);
  }
  return acc;
}, []);

// 点击执行
onClick={() => {
  onClose(() => {
    actionManager.executeAction(item, "contextMenu");  // ✅ 走统一入口
  });
}}
```

### 1.3 工具栏 (Actions Toolbar)

**触发方式：** 点击工具栏中的动作按钮

**位置：** `packages/excalidraw/components/Actions.tsx`

**工作原理：**
- 通过 `actionManager.renderAction(name)` 触发渲染
- 渲染 Action 的 `PanelComponent` 组件
- `PanelComponent` 内部通过 `updateData(formState)` 直接调用 `action.perform()`
- ❌ **不经过 executeAction 统一入口**，直接调用 perform

**关键代码：**

```typescript
// packages/excalidraw/actions/manager.tsx - renderAction
renderAction = (name: ActionName, data?: PanelComponentProps["data"]) => {
  if (
    this.actions[name] &&
    "PanelComponent" in this.actions[name]
  ) {
    const action = this.actions[name];
    const PanelComponent = action.PanelComponent!;
    const updateData = (formState?: any) => {
      // ⚠️ 注意：工具栏直接调用 perform，不经过 executeAction！
      this.updater(
        action.perform(
          this.getElementsIncludingDeleted(),
          this.getAppState(),
          formState,
          this.app,
        ),
      );
    };

    return (
      <PanelComponent
        updateData={updateData}  // updateData 直接调用 perform
        /* ...其他 props */
      />
    );
  }
  return null;
};

// Actions.tsx 中调用
{canChangeStrokeColor(appState, targetElements) &&
  renderAction("changeStrokeColor")}  // 外部先做可用性判定
```

### 1.4 快捷键 (Keyboard Shortcuts)

**触发方式：** 键盘按键组合

**位置：** `packages/excalidraw/actions/manager.tsx` → `handleKeyDown()`

**工作原理：**
- 监听全局 `keydown` 事件
- ❌ **不自动调用 predicate**，判定分三层：
  - 第 1 层：`keyTest()` - 匹配按键组合（filter 阶段）
  - 第 2 层：`viewMode` - 检查查看模式权限（单独判断）
  - 第 3 层：`perform()` 内部逻辑 - 上下文判定（如果有的话）
- 匹配成功后**直接调用 perform**，不经过 executeAction

**关键代码：**

```typescript
handleKeyDown(event: React.KeyboardEvent | KeyboardEvent) {
  const canvasActions = this.app.props.UIOptions.canvasActions;

  // 第 1 层：keyTest() 匹配按键（filter 阶段）
  const data = Object.values(this.actions)
    .sort((a, b) => (b.keyPriority || 0) - (a.keyPriority || 0))
    .filter(
      (action) =>
        (action.name in canvasActions
          ? canvasActions[action.name as keyof typeof canvasActions]
          : true) &&
        action.keyTest &&  // 有 keyTest 函数才参与匹配
        action.keyTest(event, this.getAppState(), elements, this.app),
    );

  // 确保只有一个匹配（避免歧义）
  if (data.length !== 1) {
    if (data.length > 1) {
      console.warn("Canceling as multiple actions match this shortcut", data);
    }
    return false;
  }

  const action = data[0];

  // 第 2 层：viewMode 权限检查（单独判断）
  if (this.getAppState().viewModeEnabled && action.viewMode !== true) {
    return false;
  }

  // 第 3 层：直接调用 perform 执行
  event.preventDefault();
  event.stopPropagation();
  this.updater(action.perform(elements, appState, null, this.app));  // ✅ 直接调用
  return true;
}
```

---

## 二、命令类型分层

命令面板包含两类命令，执行路径有本质区别：

### 2.1 Layer 1: Action 命令

**来源：** `actionToCommand` 转换的 4 个固定动作集合（`elementsCommands` / `toolCommands` / `editorCommands` / `exportCommands`）

**特点：**
- ✅ 有对应的 `Action` 对象定义（在 `packages/excalidraw/actions/*.tsx` 中）
- ✅ 执行时走 `actionManager.executeAction(action, "commandPalette")` 统一入口
- ✅ 经过 `trackEvent` 事件追踪
- ✅ 最终进入 `action.perform()` 执行业务逻辑
- ✅ 返回 `ActionResult` 后通过 `updater` 更新状态
- ✅ 所有入口（命令面板/右键菜单/工具栏/快捷键）共享同一业务逻辑

**执行路径：**
```
用户点击命令项
    ↓
command.perform() 触发
    ↓
actionManager.executeAction(action, "commandPalette")
    ↓
trackEvent(action, "commandPalette", ...)
    ↓
action.perform(elements, appState, value, app)
    ↓
updater(ActionResult)
    ↓
更新 elements / appState
```

**典型示例（约 50+ 个）：**
- `deleteSelectedElements` - 删除选中元素
- `group` / `ungroup` - 组合/取消组合元素
- `undo` / `redo` - 撤销/重做
- `zoomIn` / `zoomOut` / `resetZoom` - 缩放
- `copyAsPng` / `copyAsSvg` - 复制为图片/矢量图
- `toggleHandTool` / `toggleLassoTool` - 切换工具
- `alignTop` / `alignBottom` / `alignCenter` - 对齐操作
- `flipHorizontal` / `flipVertical` - 翻转
- `sendToBack` / `bringToFront` - 层级调整

---

### 2.2 Layer 2: 非 Action 命令

**来源：**
1. `commandsFromActions` 中的手动定义对象（clearCanvas、exportImage）
2. `additionalCommands` 数组中的所有自定义命令
3. `defaultCommandPaletteItems.ts` 中的独立定义

**特点：**
- ❌ 没有对应的 `Action` 对象定义（纯 `CommandPaletteItem` 对象）
- ❌ **不经过 `executeAction` 统一入口**（混合模式除外）
- ❌ **不进入 `action.perform()`**（也根本没有 action）
- ✅ 直接执行 UI 逻辑（打开弹窗、切换侧边栏、设置工具等）
- ✅ 可以调用 `setAppState` 或其他状态管理 API
- ❌ 没有统一的埋点机制（除非手动调用）
- ❌ 无法通过右键菜单、工具栏、快捷键触发（仅命令面板可用）

**执行路径：**
```
用户点击命令项
    ↓
command.perform() 触发
    ↓
┌─────────────────────────────────────────────┐
│  直接执行 UI 逻辑（不经过 Action 系统）      │
│  - 打开弹窗：setAppState({ openDialog })    │
│  - 打开侧边栏：setAppState({ openSidebar }) │
│  - 切换工具：app.setActiveTool({ type })    │
│  - 其他 UI 状态变更                         │
└──────────────────────┬──────────────────────┘
                       ↓
              状态由 React 状态管理系统处理
              （不经过 updater 回调）
```

**典型示例（约 22 个，详见第三节完整清单）：**
- Export Image - 打开导出图片弹窗
- Library - 打开/关闭资源库侧边栏
- Change Stroke / Change Background - 打开颜色选择器
- Shape Switch - 形状快速切换（混合模式）
- Search - 打开搜索菜单（混合模式）
- 8 个形状工具 - 直接设置激活工具
- Lock - 锁定/解锁工具
- Text to Diagram / Mermaid to Excalidraw - AI 转换功能

---

### 2.3 Layer 1.5: 混合模式（非 Action 内部调用 executeAction）

部分非 Action 命令内部会调用 `executeAction`，形成混合模式：

```typescript
{
  label: t("search.title"),
  category: DEFAULT_CATEGORIES.app,
  perform: () => {
    // ⚠️ 非 Action 命令内部调用 executeAction
    actionManager.executeAction(actionToggleSearchMenu);
  },
}
```

这种命令虽然属于 Layer 2（没有通过 `actionToCommand` 转换），但实际执行时会进入 Layer 1 的执行路径，获得统一的埋点和 perform 执行。

---

## 三、非 Action 命令完整盘点（新增）

### 3.1 分类统计

| 来源位置 | 数量 | 说明 |
|---------|------|------|
| `commandsFromActions` 手动定义 | 2 | clearCanvas、exportImage |
| `additionalCommands` App 分类 | 2 | Library、Search |
| `additionalCommands` Elements 分类 | 4 | Shape Switch、Change Stroke、Change Background、Canvas Background |
| `additionalCommands` Tools 分类（形状） | 8 | selection、rectangle、diamond、ellipse、arrow、line、freedraw、text |
| `additionalCommands` Tools 分类（其他） | 3 | Lock、Text to Diagram、Mermaid to Excalidraw |
| `defaultCommandPaletteItems` | 1 | toggleTheme |
| **总计（纯非 Action）** | **20** | |
| **混合模式（内部调用 executeAction）** | 3 | Search、Shape Switch、toggleTheme |

---

### 3.2 非 Action 命令完整清单表

| # | 命令名称 | 来源位置 | 分类 | 执行入口 | 进 action.perform | 经 executeAction | 有统一埋点 | 是否混合模式 | 核心执行逻辑 |
|---|---------|---------|------|---------|-------------------|------------------|-----------|-------------|-------------|
| 1 | Clear canvas | `commandsFromActions` | Editor | `editorJotaiStore.set()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 弹出清空画布确认对话框 |
| 2 | Export Image | `commandsFromActions` | Export | `setAppState()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 打开导出图片弹窗 |
| 3 | Library | `additionalCommands` | App | `setAppState()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 打开/关闭资源库侧边栏 |
| 4 | Search | `additionalCommands` | App | `actionManager.executeAction()` | ✅ 是 | ✅ 是 | ✅ 是 | ✅ 是 | 打开搜索菜单（内部调用 ActionToggleSearchMenu） |
| 5 | Shape Switch | `additionalCommands` | Elements | `actionManager.executeAction()` | ✅ 是 | ✅ 是 | ✅ 是 | ✅ 是 | 快速切换上一个形状工具（内部调用 ActionToggleShapeSwitch） |
| 6 | Change Stroke | `additionalCommands` | Elements | `setAppState()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 打开元素描边颜色选择器弹窗 |
| 7 | Change Background | `additionalCommands` | Elements | `setAppState()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 打开元素背景颜色选择器弹窗 |
| 8 | Canvas Background | `additionalCommands` | Editor | `setAppState()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 打开画布背景颜色选择器弹窗 |
| 9 | Selection | `additionalCommands` → SHAPES | Tools | `app.setActiveTool()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换到选择工具 |
| 10 | Rectangle | `additionalCommands` → SHAPES | Tools | `app.setActiveTool()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换到矩形工具 |
| 11 | Diamond | `additionalCommands` → SHAPES | Tools | `app.setActiveTool()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换到菱形工具 |
| 12 | Ellipse | `additionalCommands` → SHAPES | Tools | `app.setActiveTool()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换到椭圆工具 |
| 13 | Arrow | `additionalCommands` → SHAPES | Tools | `app.setActiveTool()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换到箭头工具 |
| 14 | Line | `additionalCommands` → SHAPES | Tools | `app.setActiveTool()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换到直线工具 |
| 15 | Freedraw | `additionalCommands` → SHAPES | Tools | `app.setActiveTool()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换到手绘工具 |
| 16 | Text | `additionalCommands` → SHAPES | Tools | `app.setActiveTool()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换到文本工具 |
| 17 | Lock | `additionalCommands` | Tools | `app.toggleLock()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 切换工具锁定状态 |
| 18 | Text to Diagram | `additionalCommands` | Tools | `setAppState()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 打开文本转图表 AI 对话框 |
| 19 | Mermaid to Excalidraw | `additionalCommands` | Tools | `setAppState()` | ❌ 否 | ❌ 否 | ❌ 否 | ❌ 否 | 打开 Mermaid 转换 AI 对话框 |
| 20 | Toggle theme | `defaultCommandPaletteItems` | App | `actionManager.executeAction()` | ✅ 是 | ✅ 是 | ✅ 是 | ✅ 是 | 切换明暗主题（内部调用 ActionToggleTheme） |

---

### 3.3 关键源码位置详解

#### 3.3.1 commandsFromActions 中的非 Action 命令（2 个）

**文件位置：** `packages/excalidraw/components/CommandPalette/CommandPalette.tsx:392-422`

```typescript
commandsFromActions = [
  ...elementsCommands,      // Layer 1 Action 命令
  ...editorCommands,        // Layer 1 Action 命令
  {
    // 1. Clear canvas（非 Action）
    label: getActionLabel(actionClearCanvas),  // 借用 Action 的 label/icon
    icon: getActionIcon(actionClearCanvas),
    shortcut: getShortcutFromShortcutName(actionClearCanvas.name as ShortcutName),
    category: DEFAULT_CATEGORIES.editor,
    keywords: ["delete", "destroy"],
    viewMode: false,
    perform: () => {
      // ❌ 直接操作状态，不经过 executeAction
      editorJotaiStore.set(activeConfirmDialogAtom, "clearCanvas");
    },
  },
  {
    // 2. Export Image（非 Action）
    label: t("buttons.exportImage"),
    category: DEFAULT_CATEGORIES.export,
    icon: ExportImageIcon,
    shortcut: getShortcutFromShortcutName("imageExport"),
    keywords: ["export", "image", "png", "jpeg", "svg", "clipboard", "picture"],
    perform: () => {
      // ❌ 直接操作状态，不经过 executeAction
      setAppState({ openDialog: { name: "imageExport" } });
    },
  },
  ...exportCommands,        // Layer 1 Action 命令
];
```

**注意：** `clearCanvas` 虽然借用了 `actionClearCanvas` 的 label、icon、shortcut，但它的 `perform` 是重新定义的，并没有真正调用 `actionClearCanvas.perform()`。

#### 3.3.2 additionalCommands 中的非 Action 命令（17 个 + 1 个来自 defaultCommandPaletteItems）

**文件位置：** `packages/excalidraw/components/CommandPalette/CommandPalette.tsx:426-608`

```typescript
const additionalCommands: CommandPaletteItem[] = [
  // 3. Library（非 Action）
  {
    label: t("toolBar.library"),
    category: DEFAULT_CATEGORIES.app,
    icon: LibraryIcon,
    viewMode: false,
    perform: () => {
      // ❌ 直接 setAppState
      if (uiAppState.openSidebar) {
        setAppState({ openSidebar: null });
      } else {
        setAppState({ openSidebar: { name: DEFAULT_SIDEBAR.name, tab: DEFAULT_SIDEBAR.defaultTab } });
      }
    },
  },
  // 4. Search（混合模式）
  {
    label: t("search.title"),
    category: DEFAULT_CATEGORIES.app,
    icon: searchIcon,
    viewMode: true,
    perform: () => {
      // ✅ 内部调用 executeAction，进入 Layer 1 路径
      actionManager.executeAction(actionToggleSearchMenu);
    },
  },
  // 5. Shape Switch（混合模式）
  {
    label: t("labels.shapeSwitch"),
    category: DEFAULT_CATEGORIES.elements,
    icon: boltIcon,
    perform: () => {
      // ✅ 内部调用 executeAction，进入 Layer 1 路径
      actionManager.executeAction(actionToggleShapeSwitch);
    },
  },
  // 6. Change Stroke（非 Action）
  {
    label: t("labels.changeStroke"),
    keywords: ["color", "outline"],
    category: DEFAULT_CATEGORIES.elements,
    icon: bucketFillIcon,
    viewMode: false,
    predicate: (elements, appState) => { /* ... */ },
    perform: () => {
      // ❌ 直接 setAppState
      setAppState((prevState) => ({ openPopup: "elementStroke" }));
    },
  },
  // 7. Change Background（非 Action）
  {
    label: t("labels.changeBackground"),
    keywords: ["color", "fill"],
    icon: bucketFillIcon,
    category: DEFAULT_CATEGORIES.elements,
    viewMode: false,
    predicate: (elements, appState) => { /* ... */ },
    perform: () => {
      // ❌ 直接 setAppState
      setAppState((prevState) => ({ openPopup: "elementBackground" }));
    },
  },
  // 8. Canvas Background（非 Action）
  {
    label: t("labels.canvasBackground"),
    keywords: ["color"],
    icon: bucketFillIcon,
    category: DEFAULT_CATEGORIES.editor,
    viewMode: false,
    perform: () => {
      // ❌ 直接 setAppState
      setAppState((prevState) => ({
        openMenu: prevState.openMenu === "canvas" ? null : "canvas",
        openPopup: "canvasBackground",
      }));
    },
  },
  // 9-16. 8 个形状工具（非 Action）
  ...SHAPES.reduce((acc: CommandPaletteItem[], shape) => {
    // SHAPES = [selection, rectangle, diamond, ellipse, arrow, line, freedraw, text]
    const { value, icon, key, numericKey } = shape;
    const command: CommandPaletteItem = {
      label: t(`toolBar.${value}`),
      category: DEFAULT_CATEGORIES.tools,
      shortcut: letter || numericKey,
      icon,
      keywords: ["toolbar"],
      viewMode: false,
      perform: ({ event }) => {
        // ❌ 直接 app.setActiveTool，不经过 executeAction
        app.setActiveTool({ type: value });
      },
    };
    acc.push(command);
    return acc;
  }, []),
  // ...toolCommands（这里是 3 个 Layer 1 Action 命令，不计入非 Action）
  // 17. Lock（非 Action）
  {
    label: t("toolBar.lock"),
    category: DEFAULT_CATEGORIES.tools,
    icon: uiAppState.activeTool.locked ? LockedIcon : UnlockedIcon,
    shortcut: KEYS.Q.toLocaleUpperCase(),
    viewMode: false,
    perform: () => {
      // ❌ 直接调用 app.toggleLock()，不经过 Action 系统
      app.toggleLock();
    },
  },
  // 18. Text to Diagram（非 Action，AI 功能）
  {
    label: `${t("labels.textToDiagram")}...`,
    category: DEFAULT_CATEGORIES.tools,
    icon: brainIconThin,
    viewMode: false,
    predicate: appProps.aiEnabled,
    perform: () => {
      // ❌ 直接 setAppState
      setAppState((state) => ({
        ...state,
        openDialog: { name: "ttd", tab: "text-to-diagram" },
      }));
    },
  },
  // 19. Mermaid to Excalidraw（非 Action，AI 功能）
  {
    label: `${t("toolBar.mermaidToExcalidraw")}...`,
    category: DEFAULT_CATEGORIES.tools,
    icon: mermaidLogoIcon,
    viewMode: false,
    predicate: appProps.aiEnabled,
    perform: () => {
      // ❌ 直接 setAppState
      setAppState((state) => ({
        ...state,
        openDialog: { name: "ttd", tab: "mermaid" },
      }));
    },
  },
  // 20. Toggle theme（来自 defaultCommandPaletteItems，混合模式）
  defaultItems.toggleTheme,
];
```

#### 3.3.3 defaultCommandPaletteItems 中的非 Action 命令（1 个）

**文件位置：** `packages/excalidraw/components/CommandPalette/defaultCommandPaletteItems.ts:1-12`

```typescript
import { actionToggleTheme } from "../../actions";
import type { CommandPaletteItem } from "./types";

export const toggleTheme: CommandPaletteItem = {
  ...actionToggleTheme,  // 借用 Action 的属性
  category: "App",
  label: "Toggle theme",
  perform: ({ actionManager }) => {
    // ✅ 内部调用 executeAction，混合模式
    actionManager.executeAction(actionToggleTheme, "commandPalette");
  },
};
```

---

### 3.4 边界结论：可观测性与一致性差异（经校对）

#### 3.4.1 可观测性差异

| 维度 | Layer 1: Action 命令 | Layer 2: 纯非 Action 命令 | Layer 1.5: 混合模式 |
|------|---------------------|---------------------------|---------------------|
| **统一埋点** | ✅ `executeAction` 内部自动调用 `trackEvent`，source="commandPalette" | ❌ 无自动埋点，需要在 perform 内部手动调用 | ✅ 有统一埋点（内部调用 executeAction） |
| **可追踪性** | ✅ 完整追踪：action name、category、调用来源、参数 | ❌ 无法通过 analytics 系统追踪（除非手动埋点） | ✅ 完整追踪 |
| **埋点一致性** | ✅ 四个入口（命令面板/右键菜单/工具栏/快捷键）使用同一埋点规范 | ❌ 命令面板独有的埋点逻辑（如果有） | ✅ 与其他入口一致 |
| **统计覆盖率** | ✅ 100% 覆盖所有核心业务操作 | ❌ 约 17 个命令（纯非 Action）无统计 | ✅ 3 个混合模式命令有统计 |

**可观测性问题示例：**
- 无法统计用户通过命令面板打开导出对话框的频率
- 无法比较"通过工具栏切换形状" vs "通过命令面板切换形状"的用户偏好
- AI 功能（Text to Diagram、Mermaid）的使用情况无法统计

#### 3.4.2 一致性差异

| 维度 | Layer 1: Action 命令 | Layer 2: 纯非 Action 命令 | Layer 1.5: 混合模式 |
|------|---------------------|---------------------------|---------------------|
| **多入口可用** | ✅ 四个入口（命令面板/右键菜单/工具栏/快捷键）都可触发 | ❌ 仅命令面板可用，无法通过其他入口触发 | ✅ 本质是 Layer 1，多入口可用 |
| **权限判定一致** | ✅ `predicate` 在所有入口统一应用（命令面板、右键菜单） | ✅ 命令面板内部应用 `predicate`，但其他入口无对应功能 | ❌ 不存在（不是真正的独立命令） |
| **ViewMode 控制** | ✅ 统一应用 `viewMode` 属性 | ✅ 命令面板内部应用 `viewMode` 属性 | ✅ 继承 Action 的 viewMode |
| **业务逻辑唯一** | ✅ `action.perform()` 只定义一次，所有入口共享 | ❌ 逻辑散落在命令面板 perform 中，无统一管理 | ✅ 共享 Action 的 perform 逻辑 |
| **快捷键支持** | ✅ 有 `keyTest` 就支持 | ❌ 无法支持（没有 Action 就没有 handleKeyDown 匹配） | ✅ 继承 Action 的 keyTest |

**一致性问题示例：**
- 用户无法通过快捷键打开导出对话框（Export Image 没有 Action）
- 用户右键画布无法快速找到"切换颜色"选项（Change Stroke 没有 Action）
- Library 侧边栏只能通过命令面板或工具栏按钮打开，无快捷键
- 新增纯 UI 功能时需要分别在命令面板和工具栏两处实现，容易不一致

#### 3.4.3 架构权衡分析

**当前设计的优点：**
1. **快速迭代**：纯 UI 功能可以快速添加为 Layer 2 命令，无需定义完整 Action
2. **灵活性高**：命令面板可以承载不属于任何 Action 的临时功能或实验性功能
3. **减少 Action 膨胀**：纯 UI 操作（打开弹窗、侧边栏）不需要污染 Action 命名空间

**当前设计的代价：**
1. **可观测性断裂**：约 17 个核心功能的使用情况无法统计，产品决策缺乏数据支撑
2. **入口不一致**：用户需要记住"某些功能只有命令面板才有"，增加认知负担
3. **维护复杂度**：同一功能可能需要在多个地方重复实现（如颜色选择器）
4. **架构理解成本**：新人需要理解为什么有些命令是 Action、有些不是，增加上手难度

**推荐改进方向：**
1. **短期**：为 Layer 2 命令的 perform 函数添加手动埋点，解决可观测性问题
2. **中期**：将高频使用的 Layer 2 命令（如 Export Image、Change Stroke）逐步转化为真正的 Action，实现多入口一致
3. **长期**：建立统一的命令注册机制，所有命令面板项都通过 Action 系统，Layer 2 仅作为快速原型的临时方案

---

## 四、动作注册

所有动作都通过统一的注册机制添加到系统中。

### 4.1 Action 接口定义

**位置：** `packages/excalidraw/actions/types.ts`

```typescript
interface Action<TData = any> {
  // ========== 基础标识 ==========
  name: ActionName;                    // 唯一标识符（枚举）
  label: string | Function;            // 显示标签（支持国际化）
  keywords?: string[];                 // 搜索关键词（命令面板用）
  icon?: ReactNode | Function;         // 图标

  // ========== UI 渲染（工具栏用）==========
  PanelComponent?: React.FC<{
    elements: readonly ExcalidrawElement[];
    appState: AppState;
    updateData: (formData?: TData) => void;  // 回调直接调用 perform
    appProps: ExcalidrawProps;
    app: AppClassProperties;
  }>;

  // ========== 核心执行（所有入口共用）==========
  perform: (
    elements: readonly OrderedExcalidrawElement[],
    appState: Readonly<AppState>,
    formData: TData | undefined,
    app: AppClassProperties,
  ) => ActionResult | Promise<ActionResult>;

  // ========== 快捷键匹配（快捷键专属）==========
  keyPriority?: number;                // 快捷键优先级（解决冲突）
  keyTest?: (
    event: React.KeyboardEvent | KeyboardEvent,
    appState: AppState,
    elements: readonly ExcalidrawElement[],
    app: AppClassProperties,
  ) => boolean;

  // ========== 上下文判定（命令面板/右键菜单用）==========
  predicate?: (
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    appProps: ExcalidrawProps,
    app: AppClassProperties,
  ) => boolean;

  // ========== 查看模式权限（所有入口用）==========
  viewMode?: boolean;                  // 查看模式下是否可用
  checked?: (appState: Readonly<AppState>) => boolean;  // 选中状态

  // ========== 事件追踪 ==========
  trackEvent: false | {
    category: "toolbar" | "element" | "canvas" | "export" | "history" | "menu" | "hyperlink";
    action?: string;
    predicate?: (appState, elements, value) => boolean;
  };
}
```

### 4.2 注册函数

**位置：** `packages/excalidraw/actions/register.ts`

```typescript
export let actions: readonly Action[] = [];

export const register = <TData extends any, T extends Action<TData> = Action<TData>>(
  action: T,
) => {
  actions = actions.concat(action);
  return action;
};
```

### 4.3 ActionManager 初始化

**位置：** `packages/excalidraw/actions/manager.tsx`

```typescript
export class ActionManager {
  actions = {} as Record<ActionName, Action>;
  updater: (actionResult: ActionResult | Promise<ActionResult>) => void;

  constructor(
    updater: UpdaterFn,
    getAppState: () => AppState,
    getElementsIncludingDeleted: () => readonly OrderedExcalidrawElement[],
    app: AppClassProperties,
  ) {
    this.updater = updater;
    // ...
  }

  // 注册单个 Action
  registerAction(action: Action) {
    this.actions[action.name] = action;
  }

  // 批量注册
  registerAll(actions: readonly Action[]) {
    actions.forEach((action) => this.registerAction(action));
  }
}
```

---

## 五、快捷键解析

快捷键系统实现了跨平台、可冲突解决的键盘事件映射，但需要注意它有独立的判定机制。

### 5.1 快捷键映射表

**位置：** `packages/excalidraw/actions/shortcuts.ts`

```typescript
export type ShortcutName = SubtypeOf<ActionName, /* 支持快捷键的 Action */>;

const shortcutMap: Record<ShortcutName, string[]> = {
  deleteSelectedElements: [getShortcutKey("Delete")],
  duplicateSelection: [getShortcutKey("CtrlOrCmd+D"), getShortcutKey("Alt+drag")],
  sendBackward: [getShortcutKey("CtrlOrCmd+[")],
  bringForward: [getShortcutKey("CtrlOrCmd+]")],
  // ... 约 50 个快捷键定义
};

export const getShortcutFromShortcutName = (name: ShortcutName, idx = 0) => {
  const shortcuts = shortcutMap[name];
  return shortcuts && shortcuts.length > 0
    ? shortcuts[idx] || shortcuts[0]
    : "";
};
```

### 5.2 跨平台快捷键适配

**位置：** `packages/excalidraw/shortcut.ts`

```typescript
import { isDarwin } from "@excalidraw/common";
import { t } from "./i18n";

export const getShortcutKey = (shortcut: string): string =>
  shortcut
    // Alt/Option 适配
    .replace(/\b(Opt(?:ion)?|Alt)\b/i, isDarwin ? t("keys.option") : t("keys.alt"))
    // Shift 国际化
    .replace(/\bShift\b/i, t("keys.shift"))
    // Ctrl/Cmd 跨平台适配
    .replace(/\b(Ctrl|Cmd|Command|CtrlOrCmd)\b/gi, isDarwin ? t("keys.cmd") : t("keys.ctrl"))
    // Escape 国际化
    .replace(/\b(Esc(?:ape)?)\b/i, t("keys.escape"))
    // Delete/Backspace 国际化
    .replace(/\b(Del(?:ete)?)\b/i, t("keys.delete"));
```

### 5.3 快捷键匹配算法详解

**位置：** `packages/excalidraw/actions/manager.tsx` → `handleKeyDown()`

快捷键判定分为**三层**，需要特别注意：**predicate 不在快捷键路径中自动调用**。

```typescript
handleKeyDown(event: React.KeyboardEvent | KeyboardEvent) {
  const canvasActions = this.app.props.UIOptions.canvasActions;

  // ┌─────────────────────────────────────────────────────────────┐
  // │ 第 1 层：keyTest() 按键匹配（filter 过滤阶段）              │
  // │ ⚠️ 这是快捷键特有的机制，其他入口没有这一层                 │
  // └─────────────────────────────────────────────────────────────┘
  const data = Object.values(this.actions)
    // 按 keyPriority 降序排序（解决快捷键冲突）
    .sort((a, b) => (b.keyPriority || 0) - (a.keyPriority || 0))
    .filter(
      (action) =>
        // 1. 用户配置允许该动作
        (action.name in canvasActions
          ? canvasActions[action.name as keyof typeof canvasActions]
          : true) &&
        // 2. 有 keyTest 函数才参与匹配
        action.keyTest &&
        // 3. 执行 keyTest 匹配按键
        action.keyTest(event, this.getAppState(), elements, this.app),
    );

  // 确保只有一个匹配（避免歧义）
  if (data.length !== 1) {
    if (data.length > 1) {
      console.warn("Canceling as multiple actions match this shortcut", data);
    }
    return false;
  }

  const action = data[0];

  // ┌─────────────────────────────────────────────────────────────┐
  // │ 第 2 层：viewMode 权限检查（单独判断）                       │
  // │ ⚠️ 注意：这里没有调用 predicate！                            │
  // └─────────────────────────────────────────────────────────────┘
  if (this.getAppState().viewModeEnabled && action.viewMode !== true) {
    return false;
  }

  // ┌─────────────────────────────────────────────────────────────┐
  // │ 第 3 层：perform 内部上下文判定（如果有的话）                 │
  // │ ⚠️ predicate 不在快捷键派发中自动调用！需要在 perform 内处理  │
  // └─────────────────────────────────────────────────────────────┘
  // 示例：deleteSelectedElements 的 perform 内部会检查是否有选中元素
  // 如果没有选中元素，perform 可能返回 false 表示不执行

  // 执行：直接调用 perform，不经过 executeAction 统一入口
  trackAction(action, "keyboard", appState, elements, this.app, null);

  event.preventDefault();
  event.stopPropagation();
  this.updater(action.perform(elements, appState, null, this.app));
  return true;
}
```

**关键点总结：**

| 判定层级 | 快捷键 | 命令面板 Layer 1 | 命令面板 Layer 2 | 右键菜单 | 工具栏 |
|---------|--------|-----------------|-----------------|---------|-------|
| **keyTest** | ✅ filter 阶段匹配 | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 |
| **viewMode** | ✅ 单独判断 | ✅ `isCommandAvailable` | ✅ `isCommandAvailable` | ✅ filter 阶段 | ❌ 外部控制 |
| **predicate** | ❌ 不自动调用 | ✅ `isCommandAvailable` | ✅ `isCommandAvailable`（如有） | ✅ filter 阶段 | ⚠️ 外部或组件内控制 |
| **perform 内部判断** | ✅ 依赖此层 | ✅ 最终兜底 | ❌ 无（直接 UI 操作） | ✅ 最终兜底 | ✅ 最终兜底 |
| **经过 action.perform** | ✅ 是 | ✅ 是 | ❌ 否 | ✅ 是 | ✅ 是 |

---

## 六、运行时上下文判定

上下文判定决定了动作在当前状态下是否可用，但**不同入口的判定机制差异显著**。

### 6.1 predicate 判定函数

`predicate` 是 Action 的可选属性，主要用于命令面板和右键菜单的可用性判定：

```typescript
predicate?: (
  elements: readonly ExcalidrawElement[],
  appState: AppState,
  appProps: ExcalidrawProps,
  app: AppClassProperties,
) => boolean;
```

**典型使用场景：**
- 需要选中元素：`getSelectedElements(elements, appState).length > 0`
- 需要特定工具激活：`appState.activeTool.type === "selection"`
- 需要特定配置：`appProps.aiEnabled`

### 6.2 各入口的上下文判定实现

#### 6.2.1 命令面板判定

**位置：** `packages/excalidraw/components/CommandPalette/CommandPalette.tsx`

```typescript
const isCommandAvailable = useStableCallback(
  (command: CommandPaletteItem) => {
    // 1. 检查查看模式权限
    if (command.viewMode === false && uiAppState.viewModeEnabled) {
      return false;
    }

    // 2. 执行 predicate 判定
    return typeof command.predicate === "function"
      ? command.predicate(
          app.scene.getNonDeletedElements(),
          uiAppState as AppState,
          appProps,
          app,
        )
      : command.predicate === undefined || command.predicate;
  },
);

// 使用：过滤 + 排序
let matchingCommands = allCommands
  .filter(isCommandAvailable)
  .sort((a, b) => a.order - b.order);
```

**特点：**
- ✅ 显式检查 `viewMode`
- ✅ 显式调用 `predicate`
- ✅ 过滤发生在命令显示前
- ✅ Layer 1 和 Layer 2 命令使用相同的过滤逻辑

#### 6.2.2 右键菜单判定

**位置：** `packages/excalidraw/components/ContextMenu.tsx`

```typescript
const filteredItems = items.reduce((acc: ContextMenuItem[], item) => {
  if (
    item &&
    (item === CONTEXT_MENU_SEPARATOR ||
      !item.predicate ||  // 如果没有 predicate，默认可用
      item.predicate(elements, appState, actionManager.app.props, actionManager.app))
  ) {
    acc.push(item);
  }
  return acc;
}, []);
```

**特点：**
- ✅ 显式调用 `predicate`
- ❌ 没有显式检查 `viewMode`（依赖 predicate 内部处理或执行时失败）
- ✅ 过滤发生在菜单渲染前

#### 6.2.3 工具栏判定

**位置：** `packages/excalidraw/components/Actions.tsx`

工具栏判定最分散，通常有多层：

```typescript
// 第 1 层：组件外部先做可用性判定（不渲染就不显示）
{canChangeStrokeColor(appState, targetElements) &&
  renderAction("changeStrokeColor")}

// 第 2 层：PanelComponent 内部通过 disabled 控制
PanelComponent: ({ elements, appState, updateData }) => (
  <ToolButton
    onClick={() => updateData(null)}
    disabled={!isSomeElementSelected(getNonDeletedElements(elements), appState)}
  />
)

// 第 3 层：perform 内部最终判定（兜底）
```

**特点：**
- ❌ 没有统一的判定入口
- ⚠️ 可用性判定分散在多个地方
- ✅ 最灵活但也最容易不一致

#### 6.2.4 快捷键判定（再次强调）

```typescript
// ❌ 快捷键路径不会自动调用 predicate！
// 只有这两层判定：
// 1. keyTest - 按键匹配
// 2. viewMode - 查看模式检查
// 3. perform 内部 - 最终兜底（如果有）
```

---

## 七、执行落点

无论从哪个入口触发，Layer 1 命令最终都调用 `action.perform()`，但**到达 perform 的路径有显著差异**。

### 7.1 四个入口的执行路径对比

| 入口 | 执行路径 | 调用链 |
|------|---------|--------|
| **命令面板 Layer 1** | 命令项 `perform` → `executeAction` → `action.perform` → `updater` | 完整路径 |
| **命令面板 Layer 2** | 命令项 `perform` → 直接 UI 操作 | ❌ 跳过整个 Action 系统 |
| **右键菜单** | `onClick` → `executeAction` → `action.perform` → `updater` | 完整路径 |
| **工具栏** | `PanelComponent.updateData` → **直接调用 `action.perform`** → `updater` | ❌ 跳过 `executeAction` |
| **快捷键** | `handleKeyDown` → **直接调用 `action.perform`** → `updater` | ❌ 跳过 `executeAction` |

### 7.2 executeAction 统一入口（命令面板 Layer 1 / 右键菜单专用）

**位置：** `packages/excalidraw/actions/manager.tsx`

```typescript
// ⚠️ 只有命令面板 Layer 1 和右键菜单走这个入口！
executeAction<T extends Action>(
  action: T,
  source: ActionSource = "api",  // "ui" | "keyboard" | "contextMenu" | "api" | "commandPalette"
  value: Parameters<T["perform"]>[2] = null,
) {
  const elements = this.getElementsIncludingDeleted();
  const appState = this.getAppState();

  // 1. 统一事件追踪（区分来源）
  trackEvent(action, source, appState, elements, this.app, value);

  // 2. 调用 action.perform() 执行业务逻辑
  // 3. 将结果传递给 updater 更新状态
  this.updater(action.perform(elements, appState, value, this.app));
}
```

### 7.3 工具栏专属路径（renderAction → PanelComponent → updateData）

```typescript
// renderAction 内部实现
renderAction = (name: ActionName, data?: any) => {
  if (this.actions[name] && "PanelComponent" in this.actions[name]) {
    const action = this.actions[name];
    const PanelComponent = action.PanelComponent!;

    // ⚠️ 工具栏的 updateData 直接调用 perform，不经过 executeAction
    const updateData = (formState?: any) => {
      // trackEvent 在这里调用，而不是在 executeAction 中
      trackAction(action, "ui", appState, elements, this.app, formState);

      this.updater(action.perform(elements, appState, formState, this.app));
    };

    return <PanelComponent updateData={updateData} />;
  }
  return null;
};
```

### 7.4 快捷键专属路径（handleKeyDown 直接调用）

```typescript
// handleKeyDown 内部
trackAction(action, "keyboard", appState, elements, this.app, null);

event.preventDefault();
event.stopPropagation();
// ⚠️ 直接调用 perform，跳过 executeAction
this.updater(action.perform(elements, appState, null, this.app));
```

### 7.5 基于源码的完整调用链对照表（更新版）

| 步骤 | 命令面板 Layer 1 | 命令面板 Layer 2 | 右键菜单 | 工具栏 | 快捷键 |
|------|-----------------|-----------------|---------|-------|--------|
| **1. 触发点** | 用户点击 Action 命令项 | 用户点击非 Action 命令项 | 用户右键点击菜单项 | 用户点击工具栏按钮 | 用户按下快捷键 |
| **2. 触发函数** | `command.perform()` | `command.perform()` | `onClick` 回调 | `PanelComponent` 内部 `onClick` | `window.addEventListener('keydown')` |
| **3. 调用 ActionManager** | `actionManager.executeAction(action, "commandPalette")` | ❌ 不调用 | `actionManager.executeAction(item, "contextMenu")` | ❌ 不调用 | ❌ 不调用 |
| **4. trackEvent 位置** | `executeAction` 内部 | ❌ 无（直接 UI 操作） | `executeAction` 内部 | `renderAction` → `updateData` 内部 | `handleKeyDown` 内部 |
| **5. source 参数** | `"commandPalette"` | ❌ 无 | `"contextMenu"` | `"ui"` | `"keyboard"` |
| **6. 调用 perform** | ✅ `executeAction` 内部调用 | ❌ **无 perform 可调用（没有 Action 对象）** | ✅ `executeAction` 内部调用 | ✅ `updateData` 直接调用 | ✅ `handleKeyDown` 直接调用 |
| **7. 是否经过 action.perform** | ✅ 是 | ❌ **否，直接 UI 操作** | ✅ 是 | ✅ 是 | ✅ 是 |
| **8. 到达 updater** | ✅ 统一落点 | ❌ 绕过，走状态管理 | ✅ 统一落点 | ✅ 统一落点 | ✅ 统一落点 |
| **9. predicate 判定位置** | 显示前 `isCommandAvailable` 过滤 | 显示前 `isCommandAvailable` 过滤（如有） | 渲染前 `reduce` 过滤 | 外部条件渲染 / `disabled` | ❌ 不在派发层调用，依赖 perform 内部 |
| **10. keyTest 判定** | ❌ 无 | ❌ 无 | ❌ 无 | ❌ 无 | ✅ `handleKeyDown` filter 阶段 |
| **11. viewMode 判定位置** | `isCommandAvailable` 内部 | `isCommandAvailable` 内部 | ⚠️ 依赖 predicate 或执行失败 | ⚠️ 外部控制 | ✅ `handleKeyDown` 单独判断 |
| **12. 返回值类型** | ✅ `ActionResult` | ❌ 无返回值 | ✅ `ActionResult` | ✅ `ActionResult` | ✅ `ActionResult` |

```
┌───────────────────────────────────────────────────────────────────────────────────────┐
│                                    调用链对比图                                        │
├───────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐                   │
│  │ 命令面板 Layer 1 │  │ 命令面板 Layer 2 │  │    右键菜单      │  工具栏  快捷键    │
│  │   Action 命令    │  │   纯 UI 命令      │  │  ContextMenu     │ Toolbar  KeyDown  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  ──┬───  ──┬───   │
│           │                      │                      │               │        │       │
│           ▼                      ▼                      │               │        │       │
│  ┌───────────────────────┐  ┌───────────────────────┐  │               │        │       │
│  │ executeAction         │  │  直接 UI 操作         │  │               │        │       │
│  │ source: commandPalette│  │  - 打开弹窗           │  │               │        │       │
│  └──────────┬────────────┘  │  - 切换侧边栏         │  │               │        │       │
│             │               │  - 其他 UI 状态变更   │  │               │        │       │
│             │               └───────────────────────┘  │               │        │       │
│             │                                          │               │        │       │
│             └──────────────────────────────────────────┘               │        │       │
│                                          │                               │        │       │
│                                          ▼                               │        │       │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                             action.perform()                                    │  │
│  │                                                                                   │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐              │  │
│  │  │ trackEvent      │◄─│ 业务逻辑       │◄─│ 结果返回        │              │  │
│  │  │ (区分 source)   │  │ (唯一核心)     │  │ ActionResult    │              │  │
│  │  └─────────────────┘  └─────────────────┘  └────────┬────────┘              │  │
│  └──────────────────────────────────────────────────────┼───────────────────────────┘  │
│                                                          │                               │
│                                                          ▼                               │
│  ┌─────────────────────────────────────────────────────────────────────────────────┐  │
│  │                               updater(result)                                    │  │
│  │                      (更新 elements / appState 唯一落点)                         │  │
│  └─────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

### 7.6 ActionResult 返回值

```typescript
export type ActionResult =
  | {
      elements?: readonly ExcalidrawElement[] | null;  // 新的元素数组
      appState?: Partial<AppState> | null;             // 新的 appState
      files?: BinaryFiles | null;                       // 文件更新
      captureUpdate: CaptureUpdateActionType;           // 更新策略
      replaceFiles?: boolean;                            // 是否替换文件
    }
  | false;  // 返回 false 表示不执行任何更新（perform 内部判定结果）
```

---

## 八、端到端完整示例

让我们以 **"删除选中元素"** (`actionDeleteSelected`) 动作为例，完整跟踪从四个入口触发到执行的全过程。

### 8.1 Action 定义：actionDeleteSelected

**位置：** `packages/excalidraw/actions/actionDeleteSelected.tsx`

```typescript
export const actionDeleteSelected = register({
  name: "deleteSelectedElements",
  label: "labels.delete",
  icon: TrashIcon,
  trackEvent: { category: "element", action: "delete" },

  // 核心执行逻辑
  perform: (elements, appState, formData, app) => {
    // ⚠️ 快捷键路径依赖这里的内部判定：如果没有选中元素，自然什么也不删
    if (appState.selectedLinearElement?.isEditing) {
      // ... 线性元素点删除逻辑
    }

    // 核心删除逻辑
    let { elements: nextElements, appState: nextAppState } =
      deleteSelectedElements(elements, appState, app);

    fixBindingsAfterDeletion(
      nextElements,
      nextElements.filter((el) => el.isDeleted),
    );

    nextAppState = handleGroupEditingState(nextAppState, nextElements);

    return {
      elements: nextElements,
      appState: {
        ...nextAppState,
        activeTool: updateActiveTool(appState, {
          type: app.state.preferredSelectionTool.type,
        }),
        multiElement: null,
        newElement: null,
        activeEmbeddable: null,
        selectedLinearElement: null,
      },
      captureUpdate: isSomeElementSelected(getNonDeletedElements(elements), appState)
        ? CaptureUpdateAction.IMMEDIATELY
        : CaptureUpdateAction.EVENTUALLY,
    };
  },

  // 快捷键匹配：只检查按键，不检查是否有选中元素
  keyTest: (event, appState, elements) =>
    (event.key === KEYS.BACKSPACE || event.key === KEYS.DELETE) &&
    !event[KEYS.CTRL_OR_CMD],

  // 工具栏面板组件
  PanelComponent: ({ elements, appState, updateData, app }) => (
    <ToolButton
      onClick={() => updateData(null)}
      // ⚠️ 工具栏在组件内部通过 disabled 控制可用性
      disabled={!isSomeElementSelected(getNonDeletedElements(elements), appState)}
    />
  ),

  // ⚠️ 注意：deleteSelectedElements 没有定义 predicate！
  // - 命令面板：elementsCommands 中的 transformer 会额外添加 predicate
  // - 快捷键：依赖 perform 内部逻辑自然处理
  // - 工具栏：依赖 PanelComponent 的 disabled
});
```

### 8.2 入口 1：快捷键 Delete 键（完整路径）

```
用户按下 Delete 键
    ↓
window.addEventListener('keydown') 事件触发
    ↓
ActionManager.handleKeyDown(event)
    ↓
┌─────────────────────────────────────────────┐
│ 第 1 层：遍历所有 Action 做 keyTest 匹配     │
│ keyTest(event) 返回 true                     │
│ (只检查按键，不检查是否有选中元素)          │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│ 第 2 层：检查匹配数量 == 1                   │
│ （避免歧义，多个匹配则取消）                │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│ 第 3 层：viewMode 权限检查                  │
│ viewModeEnabled && action.viewMode !== true │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│ ⚠️ 注意：快捷键路径不会调用 predicate！      │
│ deleteSelectedElements 甚至没有定义 predicate│
└──────────────────────┬──────────────────────┘
                       ↓
trackAction(action, "keyboard", ...)
    ↓
event.preventDefault()
event.stopPropagation()
    ↓
┌─────────────────────────────────────────────┐
│ 直接调用 action.perform()                   │
│ ❌ 跳过 executeAction 统一入口              │
└──────────────────────┬──────────────────────┘
                       ↓
perform 内部执行删除逻辑（自然处理无选中情况）
    ↓
返回 ActionResult（无选中则 elements 不变）
    ↓
updater(ActionResult)
    ↓
更新画布状态
```

### 8.3 入口 2：右键菜单 → 删除（完整路径）

```
用户在选中元素上右键点击
    ↓
ContextMenu 组件渲染
    ↓
items 数组中包含 actionDeleteSelected
    ↓
┌─────────────────────────────────────────────┐
│ predicate 过滤（渲染前）                     │
│ ⚠️ deleteSelectedElements 没有 predicate     │
│ 所以默认显示（总是可用）                     │
└──────────────────────┬──────────────────────┘
                       ↓
显示 "Delete" 菜单项
    ↓
用户点击菜单项
    ↓
onClick 触发
    ↓
onClose(() => { ... }) 回调
    ↓
┌─────────────────────────────────────────────┐
│ actionManager