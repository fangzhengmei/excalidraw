# Excalidraw 指令派发机制详解

## 目录
- [概述](#概述)
- [一、入口来源](#一入口来源)
- [二、动作注册](#二动作注册)
- [三、快捷键解析](#三快捷键解析)
- [四、运行时上下文判定](#四运行时上下文判定)
- [五、执行落点](#五执行落点)
- [六、端到端完整示例](#六端到端完整示例)
- [七、核心文件索引](#七核心文件索引)

---

## 概述

Excalidraw 的指令派发系统是一个**"一次定义，多处复用"**的统一动作执行架构。它将**命令面板**、**右键菜单**、**工具栏**、**快捷键**四个完全独立的入口点，通过统一的 `Action` 接口和 `ActionManager` 引擎连接到同一个执行落点。

**核心设计思想：**
- 动作定义与 UI 表现分离
- 所有入口共享相同的业务逻辑（`perform` 函数）
- 不同入口有不同的上下文判定机制
- 追踪来源但不改变核心执行路径

```
┌─────────────────────────────────────────────────────────────────────┐
│                          四个独立入口                                 │
├─────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐       │
│  │   命令面板     │  │   右键菜单     │  │    工具栏      │  快捷键  │
│  │ Cmd+Shift+P   │  │  RightClick    │  │  Toolbar      │ KeyDown│
│  └───────┬────────┘  └───────┬────────┘  └───────┬────────┘  ──┬──  │
└───────────┼─────────────────────┼─────────────────────┼──────────────┼──┘
            │                     │                     │              │
            ▼                     ▼                     ▼              ▼
  固定动作集合+额外命令      predicate 过滤      renderAction渲染   keyTest匹配
  (非全部Action遍历)         (显示/隐藏)         PanelComponent    (键盘事件)
            │                     │                     │              │
            └─────────────────────┴─────────────────────┴──────────────┘
                                  │
                                  ▼
                        ┌─────────────────────┐
                        │   Action 定义        │
                        │  (一次定义)          │
                        │  - name             │
                        │  - perform()        │ ← 核心业务逻辑唯一定义
                        │  - keyTest()        │
                        │  - predicate()      │
                        │  - PanelComponent   │
                        │  - viewMode         │
                        └───────────┬─────────┘
                                    │
                                    ▼
                        ┌─────────────────────┐
                        │   ActionManager     │
                        │  (统一引擎)         │
                        │  - executeAction()  │ ← 命令面板/右键菜单入口
                        │  - handleKeyDown()  │ ← 快捷键专属入口
                        │  - renderAction()   │ ← 工具栏专属入口
                        └───────────┬─────────┘
                                    │
                                    ▼
                        ┌─────────────────────┐
                        │     updater()       │  ← 统一执行落点
                        │  更新 elements       │
                        │  更新 appState       │
                        └─────────────────────┘
```

---

## 一、入口来源

Excalidraw 有四个独立的动作触发入口，它们最终都指向同一个 Action 执行引擎，但触发路径和判定机制有显著差异。

### 1.1 命令面板 (Command Palette)

**触发方式：** `Cmd/Ctrl + Shift + P` 或 `Cmd/Ctrl + /`

**位置：** `packages/excalidraw/components/CommandPalette/CommandPalette.tsx`

**核心机制 - 固定动作集合 + 额外命令：**

命令面板**不是遍历全部 Action**，而是手动构建了几个固定的动作集合，再加上额外的自定义命令，最后进行可用性过滤：

```typescript
// 第 1 步：构建 4 个固定的动作集合
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

// 第 2 步：合并固定动作集合 + 额外的自定义命令
commandsFromActions = [
  ...elementsCommands,      // 元素操作命令
  ...editorCommands,        // 编辑器命令
  { /* clearCanvas 自定义命令 */ },  // 非标准 Action，直接定义
  { /* exportImage 自定义命令 */ },  // 非标准 Action，直接定义
  ...exportCommands,        // 导出命令
];

// 第 3 步：additionalCommands - 与 Action 无关的纯自定义命令
const additionalCommands: CommandPaletteItem[] = [
  { label: "Library", /* ... */ },          // 打开侧边栏
  { label: "Search", /* ... */ },           // 搜索
  { label: "Shape Switch", /* ... */ },     // 形状切换
  { label: "Change Stroke", /* ... */ },    // 改颜色
  { label: "Change Background", /* ... */ }, // 改背景
  { label: "Canvas Background", /* ... */ }, // 画布背景
  ...SHAPES.reduce((acc, shape) => { /* 转换为命令 */ }, []),  // 形状工具命令
];

// 第 4 步：最终合并 + 可用性过滤
const allCommands = [
  ...commandsFromActions,     // 来自 Action 的命令
  ...additionalCommands,      // 额外自定义命令
  ...(customCommandPaletteItems || []),  // 用户自定义命令
].filter(isCommandAvailable);  // 可用性过滤
```

**actionToCommand 转换器：**

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
      actionManager.executeAction(action, "commandPalette");  // 执行
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
    actionManager.executeAction(item, "contextMenu");  // 通过统一入口执行
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

  // ⚠️ 注意：快捷键路径没有调用 predicate！
  // 如果需要上下文判定，必须在 perform 内部实现

  // 第 3 层：直接调用 perform 执行
  event.preventDefault();
  event.stopPropagation();
  this.updater(action.perform(elements, appState, null, this.app));  // 直接调用
  return true;
}
```

---

## 二、动作注册

所有动作都通过统一的注册机制添加到系统中。

### 2.1 Action 接口定义

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

### 2.2 注册函数

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

### 2.3 ActionManager 初始化

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

## 三、快捷键解析

快捷键系统实现了跨平台、可冲突解决的键盘事件映射，但需要注意它有独立的判定机制。

### 3.1 快捷键映射表

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

### 3.2 跨平台快捷键适配

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

### 3.3 快捷键匹配算法详解

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

| 判定层级 | 快捷键 | 命令面板 | 右键菜单 | 工具栏 |
|---------|--------|---------|---------|-------|
| **keyTest** | ✅ filter 阶段匹配 | ❌ 无 | ❌ 无 | ❌ 无 |
| **viewMode** | ✅ 单独判断 | ✅ `isCommandAvailable` | ✅ filter 阶段 | ❌ 外部控制 |
| **predicate** | ❌ **不自动调用** | ✅ `isCommandAvailable` | ✅ filter 阶段 | ⚠️ 外部或组件内控制 |
| **perform 内部判断** | ✅ 依赖此层 | ✅ 最终兜底 | ✅ 最终兜底 | ✅ 最终兜底 |

---

## 四、运行时上下文判定

上下文判定决定了动作在当前状态下是否可用，但**不同入口的判定机制差异显著**。

### 4.1 predicate 判定函数

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

### 4.2 各入口的上下文判定实现

#### 4.2.1 命令面板判定

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

#### 4.2.2 右键菜单判定

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

#### 4.2.3 工具栏判定

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

#### 4.2.4 快捷键判定（再次强调）

```typescript
// ❌ 快捷键路径不会自动调用 predicate！
// 只有这两层判定：
// 1. keyTest - 按键匹配
// 2. viewMode - 查看模式检查
// 3. perform 内部 - 最终兜底（如果有）
```

**示例：deleteSelectedElements 的快捷键路径**

```typescript
// keyTest 只检查按键，不检查是否有选中元素
keyTest: (event, appState, elements) =>
  (event.key === KEYS.BACKSPACE || event.key === KEYS.DELETE) &&
  !event[KEYS.CTRL_OR_CMD],

// ⚠️ 是否有选中元素的检查在 perform 内部
perform: (elements, appState, formData, app) => {
  // 如果没有选中元素，删除逻辑会自然不执行或返回 false
  if (/* 没有选中元素 */) {
    return false;  // 不执行任何更新
  }
  // ... 删除逻辑
}
```

### 4.3 ActionManager 统一判定接口

**位置：** `packages/excalidraw/actions/manager.tsx`

```typescript
// 仅供外部调用，快捷键派发内部不使用此方法
isActionEnabled = (action: Action) => {
  const elements = this.getElementsIncludingDeleted();
  const appState = this.getAppState();

  return (
    !action.predicate ||
    action.predicate(elements, appState, this.app.props, this.app)
  );
};
```

---

## 五、执行落点

无论从哪个入口触发，最终都调用 `action.perform()`，但**到达 perform 的路径有显著差异**。

### 5.1 四个入口的执行路径对比

| 入口 | 执行路径 | 调用链 |
|------|---------|--------|
| **命令面板** | 命令项 `perform` → `executeAction` → `action.perform` → `updater` | 完整路径 |
| **右键菜单** | `onClick` → `executeAction` → `action.perform` → `updater` | 完整路径 |
| **工具栏** | `PanelComponent.updateData` → **直接调用 `action.perform`** → `updater` | ❌ 跳过 `executeAction` |
| **快捷键** | `handleKeyDown` → **直接调用 `action.perform`** → `updater` | ❌ 跳过 `executeAction` |

### 5.2 executeAction 统一入口（命令面板/右键菜单专用）

**位置：** `packages/excalidraw/actions/manager.tsx`

```typescript
// ⚠️ 只有命令面板和右键菜单走这个入口！
executeAction<T extends Action>(
  action: T,
  source: ActionSource = "api",  // "ui" | "keyboard" | "contextMenu" | "api" | "commandPalette"
  value: Parameters<T["perform"]>[2] = null,
) {
  const elements = this.getElementsIncludingDeleted();
  const appState = this.getAppState();

  // 1. 统一事件追踪（区分来源）
  trackAction(action, source, appState, elements, this.app, value);

  // 2. 调用 action.perform() 执行业务逻辑
  // 3. 将结果传递给 updater 更新状态
  this.updater(action.perform(elements, appState, value, this.app));
}
```

### 5.3 工具栏专属路径（renderAction → PanelComponent → updateData）

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

### 5.4 快捷键专属路径（handleKeyDown 直接调用）

```typescript
// handleKeyDown 内部
trackAction(action, "keyboard", appState, elements, this.app, null);

event.preventDefault();
event.stopPropagation();
// ⚠️ 直接调用 perform，跳过 executeAction
this.updater(action.perform(elements, appState, null, this.app));
```

### 5.5 基于源码的完整调用链对照表

| 步骤 | 命令面板 | 右键菜单 | 工具栏 | 快捷键 |
|------|---------|---------|-------|--------|
| **1. 触发点** | 用户点击命令项 | 用户右键点击菜单项 | 用户点击工具栏按钮 | 用户按下快捷键 |
| **2. 触发函数** | `command.perform()` | `onClick` 回调 | `PanelComponent` 内部 `onClick` | `window.addEventListener('keydown')` |
| **3. 调用 ActionManager** | `actionManager.executeAction(action, "commandPalette")` | `actionManager.executeAction(item, "contextMenu")` | ❌ **不调用** | ❌ **不调用** |
| **4. trackEvent 位置** | `executeAction` 内部 | `executeAction` 内部 | `renderAction` → `updateData` 内部 | `handleKeyDown` 内部 |
| **5. source 参数** | `"commandPalette"` | `"contextMenu"` | `"ui"` | `"keyboard"` |
| **6. 调用 perform** | ✅ `executeAction` 内部调用 | ✅ `executeAction` 内部调用 | ✅ `updateData` 直接调用 | ✅ `handleKeyDown` 直接调用 |
| **7. 到达 updater** | ✅ 统一落点 | ✅ 统一落点 | ✅ 统一落点 | ✅ 统一落点 |
| **8. predicate 判定位置** | 显示前 `isCommandAvailable` 过滤 | 渲染前 `reduce` 过滤 | 外部条件渲染 / `disabled` | ❌ **不在派发层调用，依赖 perform 内部** |
| **9. keyTest 判定** | ❌ 无 | ❌ 无 | ❌ 无 | ✅ `handleKeyDown` filter 阶段 |
| **10. viewMode 判定位置** | `isCommandAvailable` 内部 | ⚠️ 依赖 predicate 或执行失败 | ⚠️ 外部控制 | ✅ `handleKeyDown` 单独判断 |

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                              调用链对比图                                      │
├───────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐│
│  │  命令面板    │    │  右键菜单    │    │    工具栏     │    │    快捷键    ││
│  │  Command     │    │  ContextMenu │    │  Toolbar     │    │ KeyDown      ││
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘│
│         │                    │                    │                    │        │
│         ▼                    ▼                    │                    │        │
│  ┌────────────────────────────────────┐           │                    │        │
│  │  executeAction(action, source)     │           │                    │        │
│  │  source = "commandPalette"         │           │                    │        │
│  │  source = "contextMenu"            │           │                    │        │
│  └────────────────┬───────────────────┘           │                    │        │
│                   │                                │                    │        │
│                   └────────────────────────────────┼────────────────────┘        │
│                                                    │                             │
│                                                    ▼                             │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                         action.perform()                                 │  │
│  │                                                                          │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │  │
│  │  │ trackEvent      │◄─│ 业务逻辑       │◄─│ 结果返回        │         │  │
│  │  │ (区分 source)   │  │ (唯一核心)     │  │ ActionResult    │         │  │
│  │  └─────────────────┘  └─────────────────┘  └────────┬────────┘         │  │
│  └──────────────────────────────────────────────────────┼──────────────────┘  │
│                                                          │                      │
│                                                          ▼                      │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │                            updater(result)                                │  │
│  │                    (更新 elements / appState 唯一落点)                    │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                               │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 5.6 ActionResult 返回值

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

## 六、端到端完整示例

让我们以 **"删除选中元素"** (`actionDeleteSelected`) 动作为例，完整跟踪从四个入口触发到执行的全过程。

### 6.1 Action 定义：actionDeleteSelected

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

### 6.2 入口 1：快捷键 Delete 键（完整路径）

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
updater(actionResult)
    ↓
更新画布状态
```

### 6.3 入口 2：右键菜单 → 删除（完整路径）

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
│ actionManager.executeAction(item, "contextMenu")  │
│ ✅ 走统一入口                                 │
└──────────────────────┬──────────────────────┘
                       ↓
trackAction(action, "contextMenu", ...)
    ↓
action.perform(elements, appState, null, app)
    ↓
updater(actionResult)
    ↓
更新画布状态
```

### 6.4 入口 3：工具栏 → 删除按钮（完整路径）

```
选中至少一个元素
    ↓
Actions 组件渲染
    ↓
┌─────────────────────────────────────────────┐
│ 外部条件渲染（父组件控制）                   │
│ canChangeStrokeColor(...) && renderAction() │
└──────────────────────┬──────────────────────┘
                       ↓
renderAction("deleteSelectedElements") 调用
    ↓
检查 Action 有 PanelComponent（有）
    ↓
构建 updateData 回调函数
    ↓
渲染 PanelComponent → 显示垃圾桶图标按钮
    ↓
┌─────────────────────────────────────────────┐
│ PanelComponent 内部 disabled 控制           │
│ disabled={!isSomeElementSelected(...)}      │
└──────────────────────┬──────────────────────┘
                       ↓
用户点击垃圾桶按钮
    ↓
PanelComponent 内部 onClick 触发
    ↓
updateData(null) 调用
    ↓
┌─────────────────────────────────────────────┐
│ ⚠️ 直接调用 action.perform()                 │
│ ❌ 跳过 executeAction 统一入口              │
└──────────────────────┬──────────────────────┘
                       ↓
trackAction(action, "ui", ...)  ← 在 updateData 内部调用
    ↓
updater(actionResult)
    ↓
更新画布状态
```

### 6.5 入口 4：命令面板 → Delete（完整路径）

```
用户按下 Cmd+Shift+P 打开命令面板
    ↓
CommandPalette 组件渲染
    ↓
┌─────────────────────────────────────────────┐
│ ⚠️ 不是遍历所有 Action！                     │
│ 手动构建 4 个固定动作集合：                  │
│ - elementsCommands (约 30 个)               │
│ - toolCommands (3 个)                       │
│ - editorCommands (约 15 个)                 │
│ - exportCommands (4 个)                     │
└──────────────────────┬──────────────────────┘
                       ↓
┌─────────────────────────────────────────────┐
│ actionDeleteSelected 在 elementsCommands 中 │
│ 通过 actionToCommand 转换为命令项           │
│                                                             │
│ ⚠️ elementsCommands 有特殊 transformer！     │
│ 如果 Action 没有 predicate，自动添加一个：   │
│   predicate = () => selectedElements.length > 0 │
└──────────────────────┬──────────────────────┘
                       ↓
添加额外自定义命令（clearCanvas、exportImage 等）
    ↓
添加 additionalCommands（Library、Search、形状工具等）
    ↓
┌─────────────────────────────────────────────┐
│ isCommandAvailable 可用性过滤               │
│ 1. 检查 viewMode                            │
│ 2. 执行 predicate（transformer 添加的）     │
└──────────────────────┬──────────────────────┘
                       ↓
用户输入 "delete" 进行搜索
    ↓
模糊匹配命中 "Delete" 命令
    ↓
用户点击或按 Enter 选择
    ↓
executeCommand(command, event)
    ↓
command.perform({ actionManager, event })
    ↓
┌─────────────────────────────────────────────┐
│ actionManager.executeAction(..., "commandPalette") │
│ ✅ 走统一入口                                 │
└──────────────────────┬──────────────────────┘
                       ↓
trackAction(action, "commandPalette", ...)
    ↓
action.perform(elements, appState, null, app)
    ↓
updater(actionResult)
    ↓
更新画布状态
```

### 6.6 四个入口执行路径对比总结

```
┌───────────────────────────────────────────────────────────────────────┐
│                        四个独立入口触发                                 │
├───────────────┬───────────────┬───────────────┬───────────────────┤
│   Delete 键   │  右键菜单     │  工具栏按钮   │   命令面板        │
│   (快捷键)    │  "Delete"     │  (垃圾桶)     │  "Delete"         │
└───────┬───────┴───────┬───────┴───────┬───────┴─────────┬─────────┘
        │               │               │                 │
        │               │               │                 │
        ▼               ▼               │                 ▼
 handleKeyDown     onClick 菜单项        │          选中命令项
        │               │               │                 │
        │               │               │                 │
        │               └───────────────┼─────────────────┘
        │                               │
        │                               ▼
        │                    executeAction("deleteSelectedElements")
        │                        source: "contextMenu"
        │                        source: "commandPalette"
        │                               │
        └───────────────────────────────┘
                        │
                        ▼
              ┌─────────────────────┐
              │ action.perform()    │  ← 业务逻辑只在这里！
              │ (删除元素逻辑)      │
              └───────────┬─────────┘
                          │
                          ▼
              ┌─────────────────────┐
              │     updater()       │  ← 统一执行落点
              │ 更新 elements       │
              │ 更新 appState       │
              └─────────────────────┘
```

**核心结论：**

| 特性 | 快捷键 | 命令面板 | 右键菜单 | 工具栏 |
|------|--------|---------|---------|-------|
| **遍历全部 Action** | ✅ 是 | ❌ 固定集合 | ❌ 传入 items | ❌ 按 name 查找 |
| **走 executeAction** | ❌ 否 | ✅ 是 | ✅ 是 | ❌ 否 |
| **自动调用 predicate** | ❌ 否 | ✅ 是 | ✅ 是 | ⚠️ 分散控制 |
| **有 keyTest 匹配** | ✅ 是 | ❌ 否 | ❌ 否 | ❌ 否 |
| **业务逻辑 perform** | ✅ 唯一定义 | ✅ 唯一定义 | ✅ 唯一定义 | ✅ 唯一定义 |

---

## 七、核心文件索引

| 文件路径 | 核心职责 |
|---------|---------|
| `packages/excalidraw/actions/types.ts` | Action 接口定义、ActionName 枚举、ActionResult 类型 |
| `packages/excalidraw/actions/register.ts` | 动作注册函数 |
| `packages/excalidraw/actions/manager.tsx` | ActionManager 核心引擎（executeAction、handleKeyDown、renderAction） |
| `packages/excalidraw/actions/shortcuts.ts` | 快捷键映射表、getShortcutFromShortcutName 工具 |
| `packages/excalidraw/shortcut.ts` | 跨平台快捷键适配工具（getShortcutKey） |
| `packages/excalidraw/components/CommandPalette/CommandPalette.tsx` | 命令面板实现 - 固定动作集合 + 额外命令模式 |
| `packages/excalidraw/components/CommandPalette/defaultCommandPaletteItems.ts` | 默认命令面板项 |
| `packages/excalidraw/components/ContextMenu.tsx` | 右键菜单实现 - predicate 过滤模式 |
| `packages/excalidraw/components/Actions.tsx` | 工具栏动作渲染 - renderAction + PanelComponent 模式 |
| `packages/excalidraw/actions/actionDeleteSelected.tsx` | 示例：删除选中元素 Action 定义 |
| `packages/excalidraw/actions/*.tsx` | 其他 60+ 个具体 Action 定义 |

---

## 总结

Excalidraw 的指令派发机制是一个设计精良但也相当复杂的架构：

### ✅ 统一的方面
1. **业务逻辑唯一** - `action.perform()` 只定义一次，所有入口共享
2. **执行落点统一** - 无论从哪来，最终都通过 `updater` 更新状态
3. **Action 定义统一** - 所有动作遵循相同的接口规范

### ⚠️ 不统一的方面（容易踩坑）
1. **执行入口不统一**
   - 命令面板/右键菜单 → `executeAction` 统一入口
   - 工具栏/快捷键 → 直接调用 `perform`，跳过 `executeAction`

2. **上下文判定不统一**
   - 命令面板 → `isCommandAvailable` 统一过滤
   - 右键菜单 → 渲染前 `predicate` 过滤
   - 工具栏 → 分散在外部条件渲染 + PanelComponent disabled
   - 快捷键 → ❌ 不自动调用 `predicate`，依赖 `keyTest` + `perform` 内部

3. **命令面板不是遍历全部 Action**
   - 手动维护了 `elementsCommands`、`toolCommands`、`editorCommands`、`exportCommands` 四个固定集合
   - 额外添加 `additionalCommands` 自定义命令
   - 新 Action 不会自动出现在命令面板中，需要手动添加到对应集合

### 💡 架构设计启示
这种设计的好处是**灵活性高**（每个入口可以有自己的判定逻辑），但代价是**一致性差**（容易出现"命令面板能用但快捷键不能用"之类的不一致问题）。

如果追求更统一的架构，可以考虑：
1. 所有入口都走 `executeAction` 统一入口
2. `executeAction` 内部统一调用 `predicate` 进行可用性检查
3. 命令面板自动遍历所有 Action 而不是手动维护集合
