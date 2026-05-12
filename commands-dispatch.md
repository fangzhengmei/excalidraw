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
- 所有入口共享相同的业务逻辑
- 上下文判定逻辑集中管理
- 追踪来源但不改变执行路径

```
┌───────────────────────────────────────────────────────────────┐
│                        四个独立入口                             │
├───────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  命令面板    │  │  右键菜单    │  │   工具栏     │  快捷键 │
│  │ Cmd+Shift+P  │  │  RightClick  │  │  Toolbar    │  KeyDown│
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  ──┬── │
└─────────┼───────────────────┼───────────────────┼────────────┼──┘
          │                   │                   │            │
          └───────────────────┴───────────────────┴────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   Action 定义      │
                    │  (一次定义)        │
                    │  - name           │
                    │  - perform()      │
                    │  - keyTest()      │
                    │  - predicate()    │
                    │  - PanelComponent │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  ActionManager    │
                    │  (统一引擎)       │
                    │  - executeAction()│
                    │  - handleKeyDown()│
                    │  - renderAction() │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │    updater()      │
                    │  (统一执行落点)   │
                    │  更新 elements    │
                    │  更新 appState    │
                    └───────────────────┘
```

---

## 一、入口来源

Excalidraw 有四个独立的动作触发入口，它们最终都指向同一个 Action 执行引擎。

### 1.1 命令面板 (Command Palette)

**触发方式：** `Cmd/Ctrl + Shift + P` 或 `Cmd/Ctrl + /`

**位置：** `packages/excalidraw/components/CommandPalette/CommandPalette.tsx`

**工作原理：**
- 打开时遍历所有已注册的 Action
- 通过 `actionToCommand()` 将 Action 转换为命令面板项
- 支持模糊搜索（基于 label 和 keywords）
- 点击执行时调用 `actionManager.executeAction(action, "commandPalette")`

**关键代码：**
```typescript
// 将 Action 转换为命令面板项
const actionToCommand = (action: Action, category: string): CommandPaletteItem => {
  return {
    label: getActionLabel(action),           // 显示标签
    icon: getActionIcon(action),             // 图标
    category: category,                      // 分类
    shortcut: getShortcutFromShortcutName(action.name),  // 快捷键提示
    keywords: action.keywords,               // 搜索关键词
    predicate: action.predicate,             // 可用性判定
    viewMode: action.viewMode,               // 查看模式权限
    perform: () => {
      actionManager.executeAction(action, "commandPalette");  // 执行
    },
  };
};
```

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
      !item.predicate ||
      item.predicate(elements, appState, actionManager.app.props, actionManager.app))
  ) {
    acc.push(item);
  }
  return acc;
}, []);

// 点击执行
onClick={() => {
  onClose(() => {
    actionManager.executeAction(item, "contextMenu");
  });
}}
```

### 1.3 工具栏 (Actions Toolbar)

**触发方式：** 点击工具栏中的动作按钮

**位置：** `packages/excalidraw/components/Actions.tsx`

**工作原理：**
- 通过 `actionManager.renderAction(name)` 渲染 Action 的 `PanelComponent`
- `PanelComponent` 内部通过 `updateData()` 触发执行
- 支持条件渲染（如只在选中元素时显示）

**关键代码：**
```typescript
// 有条件地渲染动作
{canChangeStrokeColor(appState, targetElements) &&
  renderAction("changeStrokeColor")}

// renderAction 内部实现
renderAction = (name: ActionName, data?: any) => {
  if (this.actions[name] && "PanelComponent" in this.actions[name]) {
    const action = this.actions[name];
    const PanelComponent = action.PanelComponent!;
    const updateData = (formState?: any) => {
      this.updater(action.perform(elements, appState, formState, this.app));
    };
    return <PanelComponent updateData={updateData} />;
  }
  return null;
};
```

### 1.4 快捷键 (Keyboard Shortcuts)

**触发方式：** 键盘按键组合

**位置：** `packages/excalidraw/actions/manager.tsx` → `handleKeyDown()`

**工作原理：**
- 监听全局 `keydown` 事件
- 遍历所有 Action 执行 `keyTest()` 匹配
- 匹配成功后调用 `action.perform()` 直接执行

**关键代码：**
```typescript
handleKeyDown(event: React.KeyboardEvent | KeyboardEvent) {
  // 1. 按优先级排序所有 Action
  // 2. 执行 keyTest() 匹配快捷键
  const data = Object.values(this.actions)
    .sort((a, b) => (b.keyPriority || 0) - (a.keyPriority || 0))
    .filter(
      (action) =>
        action.keyTest &&
        action.keyTest(event, this.getAppState(), this.getElementsIncludingDeleted(), this.app),
    );

  // 3. 确保只有一个 Action 匹配（避免冲突）
  if (data.length !== 1) {
    if (data.length > 1) {
      console.warn("Canceling as multiple actions match this shortcut", data);
    }
    return false;
  }

  // 4. 执行动作
  event.preventDefault();
  event.stopPropagation();
  this.updater(data[0].perform(/* ... */));
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

  // ========== UI 渲染 ==========
  PanelComponent?: React.FC<{         // 工具栏面板组件
    elements: readonly ExcalidrawElement[];
    appState: AppState;
    updateData: (formData?: TData) => void;
    appProps: ExcalidrawProps;
    app: AppClassProperties;
  }>;

  // ========== 核心执行 ==========
  perform: (
    elements: readonly OrderedExcalidrawElement[],
    appState: Readonly<AppState>,
    formData: TData | undefined,
    app: AppClassProperties,
  ) => ActionResult | Promise<ActionResult>;

  // ========== 快捷键匹配 ==========
  keyPriority?: number;                // 快捷键优先级（解决冲突）
  keyTest?: (
    event: React.KeyboardEvent | KeyboardEvent,
    appState: AppState,
    elements: readonly ExcalidrawElement[],
    app: AppClassProperties,
  ) => boolean;

  // ========== 上下文判定 ==========
  predicate?: (
    elements: readonly ExcalidrawElement[],
    appState: AppState,
    appProps: ExcalidrawProps,
    app: AppClassProperties,
  ) => boolean;

  // ========== 状态与权限 ==========
  checked?: (appState: Readonly<AppState>) => boolean;  // 是否选中状态
  viewMode?: boolean;                  // 查看模式下是否可用

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
  getAppState: () => Readonly<AppState>;
  getElementsIncludingDeleted: () => readonly OrderedExcalidrawElement[];
  app: AppClassProperties;

  constructor(
    updater: UpdaterFn,
    getAppState: () => AppState,
    getElementsIncludingDeleted: () => readonly OrderedExcalidrawElement[],
    app: AppClassProperties,
  ) {
    this.updater = updater;
    this.getAppState = getAppState;
    this.getElementsIncludingDeleted = getElementsIncludingDeleted;
    this.app = app;
  }

  registerAction(action: Action) {
    this.actions[action.name] = action;
  }

  registerAll(actions: readonly Action[]) {
    actions.forEach((action) => this.registerAction(action));
  }
}
```

---

## 三、快捷键解析

快捷键系统实现了跨平台、可冲突解决的键盘事件映射。

### 3.1 快捷键映射表

**位置：** `packages/excalidraw/actions/shortcuts.ts`

```typescript
export type ShortcutName = SubtypeOf<ActionName, /* 支持快捷键的 Action */>;

const shortcutMap: Record<ShortcutName, string[]> = {
  deleteSelectedElements: [getShortcutKey("Delete")],
  duplicateSelection: [getShortcutKey("CtrlOrCmd+D"), getShortcutKey("Alt+drag")],
  sendBackward: [getShortcutKey("CtrlOrCmd+[")],
  bringForward: [getShortcutKey("CtrlOrCmd+]")],
  sendToBack: [
    isDarwin
      ? getShortcutKey("CtrlOrCmd+Alt+[")
      : getShortcutKey("CtrlOrCmd+Shift+["),
  ],
  bringToFront: [
    isDarwin
      ? getShortcutKey("CtrlOrCmd+Alt+]")
      : getShortcutKey("CtrlOrCmd+Shift+]"),
  ],
  copyAsPng: [getShortcutKey("Shift+Alt+C")],
  group: [getShortcutKey("CtrlOrCmd+G")],
  ungroup: [getShortcutKey("CtrlOrCmd+Shift+G")],
  gridMode: [getShortcutKey("CtrlOrCmd+'")],
  zenMode: [getShortcutKey("Alt+Z")],
  objectsSnapMode: [getShortcutKey("Alt+S")],
  stats: [getShortcutKey("Alt+/")],
  flipHorizontal: [getShortcutKey("Shift+H")],
  flipVertical: [getShortcutKey("Shift+V")],
  hyperlink: [getShortcutKey("CtrlOrCmd+K")],
  toggleElementLock: [getShortcutKey("CtrlOrCmd+Shift+L")],
  resetZoom: [getShortcutKey("CtrlOrCmd+0")],
  zoomOut: [getShortcutKey("CtrlOrCmd+-")],
  zoomIn: [getShortcutKey("CtrlOrCmd++")],
  zoomToFit: [getShortcutKey("Shift+1")],
  zoomToFitSelection: [getShortcutKey("Shift+2")],
  zoomToFitSelectionInViewport: [getShortcutKey("Shift+3")],
  toggleEraserTool: [getShortcutKey("E")],
  toggleHandTool: [getShortcutKey("H")],
  setFrameAsActiveTool: [getShortcutKey("F")],
  saveFileToDisk: [getShortcutKey("CtrlOrCmd+S")],
  saveToActiveFile: [getShortcutKey("CtrlOrCmd+S")],
  toggleShortcuts: [getShortcutKey("?")],
  searchMenu: [getShortcutKey("CtrlOrCmd+F")],
  toolLock: [getShortcutKey("Q")],
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
    // Enter/Return 适配
    .replace(/\b(Enter|Return)\b/i, t("keys.enter"))
    // Ctrl/Cmd 跨平台适配
    .replace(/\b(Ctrl|Cmd|Command|CtrlOrCmd)\b/gi, isDarwin ? t("keys.cmd") : t("keys.ctrl"))
    // Escape 国际化
    .replace(/\b(Esc(?:ape)?)\b/i, t("keys.escape"))
    // Delete/Backspace 国际化
    .replace(/\b(Del(?:ete)?)\b/i, t("keys.delete"));
```

### 3.3 快捷键匹配算法

**位置：** `packages/excalidraw/actions/manager.tsx` → `handleKeyDown()`

```typescript
handleKeyDown(event: React.KeyboardEvent | KeyboardEvent) {
  const canvasActions = this.app.props.UIOptions.canvasActions;

  // 步骤 1: 按优先级排序 + 过滤匹配
  const data = Object.values(this.actions)
    // 按 keyPriority 降序排序（解决快捷键冲突）
    .sort((a, b) => (b.keyPriority || 0) - (a.keyPriority || 0))
    // 过滤条件：
    // 1. 用户配置允许该动作
    // 2. 有 keyTest 函数
    // 3. keyTest 返回 true
    .filter(
      (action) =>
        (action.name in canvasActions
          ? canvasActions[action.name as keyof typeof canvasActions]
          : true) &&
        action.keyTest &&
        action.keyTest(
          event,
          this.getAppState(),
          this.getElementsIncludingDeleted(),
          this.app,
        ),
    );

  // 步骤 2: 确保只有一个匹配（避免歧义）
  if (data.length !== 1) {
    if (data.length > 1) {
      console.warn("Canceling as multiple actions match this shortcut", data);
    }
    return false;
  }

  const action = data[0];

  // 步骤 3: 检查查看模式权限
  if (this.getAppState().viewModeEnabled && action.viewMode !== true) {
    return false;
  }

  // 步骤 4: 追踪事件 + 执行
  const elements = this.getElementsIncludingDeleted();
  const appState = this.getAppState();

  trackAction(action, "keyboard", appState, elements, this.app, null);

  event.preventDefault();
  event.stopPropagation();
  this.updater(action.perform(elements, appState, null, this.app));
  return true;
}
```

---

## 四、运行时上下文判定

上下文判定决定了动作在当前状态下是否可用，所有入口共享相同的判定逻辑。

### 4.1 predicate 判定函数

`predicate` 是 Action 的核心属性，所有 UI 入口都会使用它来判断是否显示/启用该动作。

**典型判定场景：**

| 场景 | 判定逻辑 |
|------|---------|
| 需要选中元素 | `getSelectedElements(elements, appState).length > 0` |
| 需要选中多个元素 | `selectedElements.length >= 2` |
| 需要特定工具激活 | `appState.activeTool.type === "selection"` |
| 仅编辑模式可用 | `!appState.viewModeEnabled` |
| 需要编辑文本 | `!!appState.editingTextElement` |

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

// 使用判定结果过滤显示
let matchingCommands = allCommands
  .filter(isCommandAvailable)
  .sort((a, b) => a.order - b.order);
```

#### 4.2.2 右键菜单判定

**位置：** `packages/excalidraw/components/ContextMenu.tsx`

```typescript
const filteredItems = items.reduce((acc: ContextMenuItem[], item) => {
  if (
    item &&
    (item === CONTEXT_MENU_SEPARATOR ||
      // 如果没有 predicate，默认可用
      !item.predicate ||
      // 有 predicate，执行判定
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
```

#### 4.2.3 工具栏判定

**位置：** `packages/excalidraw/components/Actions.tsx`

工具栏通常在组件外部先判定，再决定是否渲染：

```typescript
// 示例：只有选中元素且可以改变描边颜色时才渲染
{canChangeStrokeColor(appState, targetElements) &&
  renderAction("changeStrokeColor")}

// 也可以在 PanelComponent 内部通过 disabled 控制
PanelComponent: ({ elements, appState, updateData }) => (
  <ToolButton
    onClick={() => updateData(null)}
    disabled={!isSomeElementSelected(getNonDeletedElements(elements), appState)}
  />
)
```

#### 4.2.4 快捷键判定

快捷键有两层判定：
1. `keyTest()` - 匹配按键组合
2. `predicate()` - 检查上下文可用性（隐含在 perform 执行中）

### 4.3 ActionManager 统一判定接口

**位置：** `packages/excalidraw/actions/manager.tsx`

```typescript
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

无论从哪个入口触发，最终都通过统一的执行路径到达 `updater` 回调。

### 5.1 executeAction 统一执行入口

**位置：** `packages/excalidraw/actions/manager.tsx`

```typescript
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

### 5.2 ActionResult 返回值

```typescript
export type ActionResult =
  | {
      elements?: readonly ExcalidrawElement[] | null;  // 新的元素数组
      appState?: Partial<AppState> | null;             // 新的 appState
      files?: BinaryFiles | null;                       // 文件更新
      captureUpdate: CaptureUpdateActionType;           // 更新策略
      replaceFiles?: boolean;                            // 是否替换文件
    }
  | false;  // 返回 false 表示不执行任何更新
```

### 5.3 trackEvent 事件追踪

```typescript
const trackAction = (
  action: Action,
  source: ActionSource,        // 来源标识
  appState: Readonly<AppState>,
  elements: readonly ExcalidrawElement[],
  app: AppClassProperties,
  value: any,
) => {
  if (action.trackEvent) {
    try {
      if (typeof action.trackEvent === "object") {
        const shouldTrack = action.trackEvent.predicate
          ? action.trackEvent.predicate(appState, elements, value)
          : true;
        if (shouldTrack) {
          trackEvent(
            action.trackEvent.category,
            action.trackEvent.action || action.name,
            `${source} (${app.editorInterface.formFactor === "phone" ? "mobile" : "desktop"})`,
          );
        }
      }
    } catch (error) {
      console.error("error while logging action:", error);
    }
  }
};
```

### 5.4 四个入口的执行路径对比

| 入口 | 执行路径 |
|------|---------|
| **命令面板** | `command.perform()` → `actionManager.executeAction(action, "commandPalette")` → `action.perform()` → `updater()` |
| **右键菜单** | `onClick` → `actionManager.executeAction(item, "contextMenu")` → `action.perform()` → `updater()` |
| **工具栏** | `PanelComponent.updateData()` → `action.perform()` → `updater()` |
| **快捷键** | `handleKeyDown()` → `action.perform()` → `updater()` |

**关键点：** 所有路径最终都调用 `action.perform()`，业务逻辑只定义一次。

---

## 六、端到端完整示例

让我们以 **"删除选中元素"** 动作为例，完整跟踪从四个入口触发到执行的全过程。

### 6.1 Action 定义：actionDeleteSelected

**位置：** `packages/excalidraw/actions/actionDeleteSelected.tsx`

```typescript
export const actionDeleteSelected = register({
  // ========== 基础标识 ==========
  name: "deleteSelectedElements",
  label: "labels.delete",
  icon: TrashIcon,
  trackEvent: { category: "element", action: "delete" },

  // ========== 核心执行逻辑 ==========
  perform: (elements, appState, formData, app) => {
    if (appState.selectedLinearElement?.isEditing) {
      // 特殊情况：正在编辑线性元素点
      // ... 删除点的逻辑
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

  // ========== 快捷键匹配 ==========
  keyTest: (event, appState, elements) =>
    (event.key === KEYS.BACKSPACE || event.key === KEYS.DELETE) &&
    !event[KEYS.CTRL_OR_CMD],

  // ========== 工具栏面板 ==========
  PanelComponent: ({ elements, appState, updateData, app }) => {
    const isMobile = useStylesPanelMode() === "mobile";

    return (
      <ToolButton
        type="button"
        icon={TrashIcon}
        title={t("labels.delete")}
        aria-label={t("labels.delete")}
        onClick={() => updateData(null)}
        disabled={!isSomeElementSelected(getNonDeletedElements(elements), appState)}
        style={{
          ...(isMobile && appState.openPopup !== "compactOtherProperties"
            ? MOBILE_ACTION_BUTTON_BG
            : {}),
        }}
      />
    );
  },
});
```

### 6.2 入口 1：快捷键 Delete 键

```
用户按下 Delete 键
    ↓
window.addEventListener('keydown') 事件触发
    ↓
ActionManager.handleKeyDown(event)
    ↓
遍历所有 Action 执行 keyTest()
    ↓
actionDeleteSelected.keyTest(event) 返回 true
    ↓
检查是否有多个匹配（这里只有一个）
    ↓
检查 viewMode 权限（delete 不可用，必须是编辑模式）
    ↓
trackAction(action, "keyboard", ...)
    ↓
action.perform(elements, appState, null, app)
    ↓
返回 { elements: nextElements, appState: nextAppState, captureUpdate }
    ↓
updater(actionResult)
    ↓
更新画布状态 → 元素被删除
```

### 6.3 入口 2：右键菜单 → 删除

```
用户在选中元素上右键点击
    ↓
ContextMenu 组件渲染
    ↓
items 数组中包含 actionDeleteSelected
    ↓
执行 predicate 过滤（无 predicate，默认通过）
    ↓
显示 "Delete" 菜单项
    ↓
用户点击菜单项
    ↓
onClick 触发
    ↓
onClose(() => {
  actionManager.executeAction(item, "contextMenu")
})
    ↓
trackAction(action, "contextMenu", ...)
    ↓
action.perform(elements, appState, null, app)
    ↓
返回 { elements: nextElements, appState: nextAppState }
    ↓
updater(actionResult)
    ↓
更新画布状态 → 元素被删除
```

### 6.4 入口 3：工具栏 → 删除按钮

```
选中至少一个元素
    ↓
Actions 组件渲染
    ↓
renderAction("deleteSelectedElements") 被调用
    ↓
检查 Action 是否有 PanelComponent（有）
    ↓
渲染 PanelComponent → 显示垃圾桶图标按钮
    ↓
用户点击垃圾桶按钮
    ↓
PanelComponent 内部 onClick 触发
    ↓
updateData(null) 被调用
    ↓
action.perform(elements, appState, null, app)
    ↓
返回 { elements: nextElements, appState: nextAppState }
    ↓
updater(actionResult)
    ↓
更新画布状态 → 元素被删除
```

### 6.5 入口 4：命令面板 → Delete

```
用户按下 Cmd+Shift+P 打开命令面板
    ↓
CommandPalette 组件渲染
    ↓
遍历所有 Action，通过 actionToCommand() 转换为命令项
    ↓
deleteSelectedElements 被转换为命令项
  - label: "Delete"
  - icon: TrashIcon
  - category: "Elements"
  - shortcut: "Delete"
  - perform: () => actionManager.executeAction(...)
    ↓
执行 isCommandAvailable 判定（有选中元素才可用）
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
actionManager.executeAction(actionDeleteSelected, "commandPalette")
    ↓
trackAction(action, "commandPalette", ...)
    ↓
action.perform(elements, appState, null, app)
    ↓
返回 { elements: nextElements, appState: nextAppState }
    ↓
updater(actionResult)
    ↓
更新画布状态 → 元素被删除
```

### 6.6 四个入口执行路径对比图

```
┌───────────────────────────────────────────────────────────────────────┐
│                        四个独立入口触发                                 │
├───────────────┬───────────────┬───────────────┬───────────────────┤
│   Delete 键   │  右键菜单     │  工具栏按钮   │   命令面板        │
│   (快捷键)    │  "Delete"     │  (垃圾桶)     │  "Delete"         │
└───────┬───────┴───────┬───────┴───────┬───────┴─────────┬─────────┘
        │               │               │                 │
        │               │               │                 │
        ▼               ▼               ▼                 ▼
  handleKeyDown     onClick 菜单项   onClick 按钮    选中命令项
        │               │               │                 │
        │               └───────────────┼─────────────────┘
        │                               │
        │                               ▼
        │                   executeAction("deleteSelectedElements")
        │                       source: "contextMenu"
        │                       source: "ui"
        │                       source: "commandPalette"
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

**核心结论：** 四个入口的业务逻辑完全相同，都调用同一个 `action.perform()` 方法。差异只在于：
- 触发方式不同
- `trackEvent` 的 `source` 参数不同（用于分析用户习惯）

---

## 七、核心文件索引

| 文件路径 | 核心职责 |
|---------|---------|
| `packages/excalidraw/actions/types.ts` | Action 接口定义、ActionName 枚举、ActionResult 类型 |
| `packages/excalidraw/actions/register.ts` | 动作注册函数 |
| `packages/excalidraw/actions/manager.tsx` | ActionManager 核心引擎（executeAction、handleKeyDown、renderAction） |
| `packages/excalidraw/actions/shortcuts.ts` | 快捷键映射表、getShortcutFromShortcutName 工具 |
| `packages/excalidraw/shortcut.ts` | 跨平台快捷键适配工具（getShortcutKey） |
| `packages/excalidraw/components/CommandPalette/CommandPalette.tsx` | 命令面板实现 |
| `packages/excalidraw/components/CommandPalette/defaultCommandPaletteItems.ts` | 默认命令面板项 |
| `packages/excalidraw/components/ContextMenu.tsx` | 右键菜单实现 |
| `packages/excalidraw/components/Actions.tsx` | 工具栏动作渲染、SelectedShapeActions 等 |
| `packages/excalidraw/actions/actionDeleteSelected.tsx` | 示例：删除选中元素 Action 定义 |
| `packages/excalidraw/actions/*.tsx` | 其他 60+ 个具体 Action 定义 |

---

## 总结

Excalidraw 的指令派发机制是一个设计精良的架构典范：

1. **"一次定义，多处复用"** - Action 只定义一次，所有入口共享
2. **"声明式上下文"** - 通过 predicate 声明可用性，所有入口自动应用
3. **"来源追踪但不改变逻辑"** - 通过 source 参数追踪用户行为，但不影响执行路径
4. **"优先级解决冲突"** - keyPriority 机制优雅地解决快捷键冲突
5. **"统一执行落点"** - 无论从哪来，最终都通过 updater 更新状态

这种设计使得添加新动作非常简单：只需要定义一个 Action，它会自动出现在命令面板、右键菜单（如果配置）、工具栏（如果有 PanelComponent），并支持快捷键（如果有 keyTest）。
