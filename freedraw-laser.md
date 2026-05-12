# Excalidraw 手绘与激光指针实现分析

## 一、架构全景：分离与共享

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         共享模块边界（最终结论）                                    │
├──────────────────────────────────────────────────────────────────────────────────┤
│  1. getSvgPathFromStroke (@excalidraw/common) - 手绘/激光/套索/橡皮擦共用         │
│  2. AnimationFrameHandler - 帧调度器，激光 + 套索 + 橡皮擦 共用，手绘不用          │
│  3. AnimatedTrail - 轨迹基类，激光 + 套索 + 橡皮擦 共用，手绘不用                 │
├──────────────────────────────────┬───────────────────────────────────────────────┤
│        手绘渲染管线              │    动态轨迹渲染管线（激光/套索/橡皮擦）         │
├──────────────────────────────────┼───────────────────────────────────────────────┤
│  perfect-freehand → getStroke     │ @excalidraw/laser-pointer                     │
│  + 本地 SVG 生成函数 (shape.ts)   │    ↳ 内置 streamline 平滑                    │
│  + ShapeCache 静态缓存            │    ↳ sizeMapping 动态衰减（每帧重算）          │
│  + Rough.js 手绘质感             │    ↳ 无缓存，实时计算                          │
│  + points-on-curve simplify       │                                              │
├──────────────────────────────────┼───────────────────────────────────────────────┤
│        Canvas 2D 渲染           │      SVG path 直接填充                         │
│        保存为 Element           │      仅内存中轨迹                              │
│        无帧动画循环             │      AnimationFrameHandler 统一调度            │
└──────────────────────────────────┴───────────────────────────────────────────────┘
```

---

## 二、手绘平滑管线 (Geometric Smoothing Pipeline)

### 2.1 手绘元素数据结构

```typescript
interface ExcalidrawFreeDrawElement {
  type: "freedraw";
  points: LocalPoint[];           // 原始点序列 [x, y][]
  pressures: number[];             // 压力值，0-1 范围（数位笔输入）
  simulatePressure: boolean;       // 是否从移动速度模拟压力
  strokeWidth: number;             // 基础笔触宽度
}
```

### 2.2 手绘渲染完整流程

**位置: `packages/element/src/shape.ts:1181-1200`**

```typescript
// ┌─ 阶段1：生成平滑轮廓点 ─────────────────────────────────┐
export const getFreedrawOutlinePoints = (
  element: ExcalidrawFreeDrawElement,
) => {
  // 输入准备：合并坐标和压力
  const inputPoints = element.simulatePressure
    ? element.points                                // 仅坐标，库内部模拟压力
    : element.points.length
    ? element.points.map(([x, y], i) => [x, y, element.pressures[i]])
    : [[0, 0, 0.5]];

  // 调用 perfect-freehand 核心算法
  return getStroke(inputPoints as number[][], {
    simulatePressure: element.simulatePressure,
    size: element.strokeWidth * 4.25,
    thinning: 0.6,
    smoothing: 0.5,
    streamline: 0.5,
    easing: (t) => Math.sin((t * Math.PI) / 2),  // easeOutSine
    last: true,
  }) as [number, number][];
};
// └───────────────────────────────────────────────────────┘

// ┌─ 阶段2：SVG 导出路径（调用 common 共享版本）───────────┐
// 位置: packages/element/src/shape.ts:1175-1179
// 可验证结论：第1176行调用的是从 @excalidraw/common 导入的共享版本
const getFreeDrawSvgPath = (element: ExcalidrawFreeDrawElement) => {
  return getSvgPathFromStroke(
    getFreedrawOutlinePoints(element),
  ) as SVGPathString;
};
// └───────────────────────────────────────────────────────┘

// ┌─ 阶段3：本地 SVG 路径函数（用于 Canvas 渲染）──────────┐
// 位置: packages/element/src/shape.ts:1211-1231
// 可验证结论：这是 element 包本地独立实现，与 common 版本不同
const med = (A: number[], B: number[]) =>
  [(A[0] + B[0]) / 2, (A[1] + B[1]) / 2];

const TO_FIXED_PRECISION = /(\s?[A-Z]?,?-?[0-9]*\.[0-9]{0,2})(([0-9]|e|-)*)/g;

const getSvgPathFromStroke = (points: number[][]): string => {
  if (!points.length) return "";

  const max = points.length - 1;

  // reduce + 中点逐点构建 Q 命令（与 common 版本的 Q+T 链不同）
  return points
    .reduce(
      (acc, point, i, arr) => {
        if (i === max) {
          acc.push(point, med(point, arr[0]), "L", arr[0], "Z");
        } else {
          acc.push(point, med(point, arr[i + 1]));
        }
        return acc;
      },
      ["M", points[0], "Q"],
    )
    .join(" ")
    .replace(TO_FIXED_PRECISION, "$1");
};
// └───────────────────────────────────────────────────────┘

// ┌─ 阶段4：碰撞检测简化 ──────────────────────────────────┐
// 位置: packages/element/src/shape.ts:688-691
// 使用 points-on-curve 库的 simplify 算法
const simplifiedPoints = simplify(
  element.points as Mutable<LocalPoint[]>,
  0.75,  // 容差值
);
// └───────────────────────────────────────────────────────┘
```

### 2.3 perfect-freehand 可观测工作流程

**证据来源: `packages/element/src/shape.ts` + perfect-freehand 公开文档**

```
可观测处理链（仓库内可验证）：

Input Points → [x, y, pressure]
    ↓
1. streamline 阶段（参数值：0.5）
   ├─ 基于相邻点方向修正点位置
   └─ 消除鼠标抖动
    ↓
2. smoothing 阶段（参数值：0.5）
   └─ 点间插值平滑，减少尖锐拐角
    ↓
3. thinning 阶段（参数值：0.6）
   ├─ pressure 映射到笔触宽度
   ├─ easing: easeOutSine 变换压力曲线
   └─ 计算每个点的左右轮廓偏移量
    ↓
4. 输出：闭合多边形顶点 [x, y][]

关键特性（可验证）：
✓ 手绘不使用 AnimationFrameHandler（全局搜索无引用）
✓ ShapeCache 静态缓存计算结果
✓ pressure 字段真实表示笔压值 0-1
```

---

## 三、动态轨迹渲染管线（激光/套索/橡皮擦）

### 3.1 @excalidraw/laser-pointer 可观测行为

**证据来源: `packages/excalidraw/animated-trail.ts` 调用方式**

```typescript
// 基于接口调用的可观测行为推导（非内部实现事实）
// 注意：npm 包源码不在仓库内，以下仅为外部接口行为

class LaserPointer {
  // 可观测：能通过 originalPoints 访问原始点
  // 证据: animated-trail.ts:65-71 hasLastPoint() 直接访问
  originalPoints: [number, number, number][];  // [x, y, timestamp]

  // 可观测：构造函数接受 options 覆盖
  // 证据: animated-trail.ts:97 new LaserPointer(this.options)
  constructor(options: Partial<LaserPointerOptions>);

  // 可观测：能添加新点
  // 证据: animated-trail.ts:99, 106 调用
  addPoint(point: [number, number, number]): void;

  // 可观测：能关闭 keepHead
  // 证据: animated-trail.ts:114 trail.close()
  close(): void;

  // 可观测：核心输出，接收 scale 参数
  // 证据: animated-trail.ts:181-182 调用
  getStrokeOutline(scale: number = 1): [number, number][];
}

// 可观测 Options 接口（来自 TypeScript 类型标注）
interface LaserPointerOptions {
  size: number;
  thinning: number;
  smoothing: number;
  streamline: number;
  simplify: number;

  // 动态大小映射
  // 证据: laser-trails.ts:35-48, lasso/index.ts:54-66, eraser/index.ts:51-63
  sizeMapping?: (context: {
    pressure: number;        // 实际存储 timestamp
    currentIndex: number;    // 点索引
    totalLength: number;     // 总长度
  }) => number;

  keepHead?: boolean;        // 保持头部
}
```

> ⚠️ **重要说明**: `@excalidraw/laser-pointer` 为 npm 外部包，
> 其内部默认参数值在本仓库源码中不可直接验证，
> 以上仅为基于调用接口的可观测行为推导。

### 3.2 hasLastPoint 去重逻辑（仓库内可验证）

**位置: `packages/excalidraw/animated-trail.ts:64-74`**

```typescript
hasLastPoint(x: number, y: number) {
  if (this.currentTrail) {
    const len = this.currentTrail.originalPoints.length;
    // 真实去重条件：比较最后一个点的坐标
    return (
      this.currentTrail.originalPoints[len - 1][0] === x &&
      this.currentTrail.originalPoints[len - 1][1] === y
    );
  }

  return false;
}
```

### 3.3 三种动态轨迹参数对比（仓库内可验证）

| 参数 | 激光 | 套索 | 橡皮擦 | 验证位置 |
|------|------|------|--------|---------|
| `streamline` | 0.4 | 0.4 | 0.2 | laser-trails.ts:33, lasso:53, eraser:48 |
| `simplify` | 0 | (未显式设置) | (未显式设置) | laser-trails.ts:33 |
| `DECAY_TIME` | 1000 | Infinity | 200 | laser:36, lasso:55, eraser:52 |
| `DECAY_LENGTH` | 50 | 5000 | 10 | laser:37, lasso:56, eraser:53 |
| `size` | (默认) | (默认) | 5 | eraser:49 |
| `keepHead` | (默认) | (未显式设置) | true | eraser:50 |
| `animateTrail` | false | true | false | lasso:52 |
| 颜色 | 固定激光色 | 主题色半透明 | 主题色半透明 | 各文件 fill/stroke |

### 3.4 AnimationFrameHandler 适用范围（最终结论）

**可验证事实**：AnimationFrameHandler 用于：
1. ✓ 激光轨迹（laser-trails.ts）
2. ✓ 套索选择（lasso/index.ts）
3. ✓ 橡皮擦轨迹（eraser/index.ts）
4. ✗ 手绘工具（无引用，使用 ShapeCache 静态缓存）

**位置: `packages/excalidraw/animation-frame-handler.ts:8-79`**

```typescript
export class AnimationFrameHandler {
  private targets = new WeakMap<object, AnimationTarget>();
  private rafIds = new WeakMap<object, number>();

  register(key: object, callback: AnimationCallback) {
    this.targets.set(key, { callback, stopped: true });
  }

  start(key: object) { /* 启动帧循环 */ }
  stop(key: object) { /* 停止帧循环 */ }

  private scheduleFrame(key: object) {
    const rafId = requestAnimationFrame(this.constructFrame(key));
    this.rafIds.set(key, rafId);
  }
}
```

### 3.5 AnimatedTrail 基类帧渲染循环

**位置: `packages/excalidraw/animated-trail.ts:30-197`**

```typescript
export class AnimatedTrail implements Trail {
  private currentTrail?: LaserPointer;
  private pastTrails: LaserPointer[] = [];

  constructor(
    private animationFrameHandler: AnimationFrameHandler,  // 注入依赖
    protected app: App,
    private options: Partial<LaserPointerOptions> & AnimatedTrailOptions,
  ) {
    this.animationFrameHandler.register(this, this.onFrame.bind(this));
  }

  private onFrame() {
    const paths: string[] = [];

    // 渲染历史轨迹
    for (const trail of this.pastTrails) {
      paths.push(this.drawTrail(trail, this.app.state));
    }

    // 渲染当前轨迹
    if (this.currentTrail) {
      const currentPath = this.drawTrail(this.currentTrail, this.app.state);
      paths.push(currentPath);
    }

    // 清理已消失轨迹
    this.pastTrails = this.pastTrails.filter((trail) => {
      return trail.getStrokeOutline().length !== 0;
    });

    // 更新 SVG
    const svgPaths = paths.join(" ").trim();
    this.trailElement.setAttribute("d", svgPaths);

    // 关键：每帧都重新计算所有点！
    // 因为 sizeMapping 依赖 performance.now()
    // 对比手绘：计算一次后 ShapeCache 缓存
  }

  private drawTrail(trail: LaserPointer, state: AppState): string {
    const _stroke = trail
      .getStrokeOutline(trail.options.size / state.zoom.value)
      .map(([x, y]) => {
        const result = sceneCoordsToViewportCoords(
          { sceneX: x, sceneY: y },
          state,
        );
        return [result.x, result.y];
      });

    // 动画模式：取一半点
    const stroke = this.trailAnimation
      ? _stroke.slice(0, _stroke.length / 2)
      : _stroke;

    // 调用 common 包共享的 SVG 路径函数
    return getSvgPathFromStroke(stroke, true);
  }
}
```

---

## 四、共享模块边界（最终结论，无待验证）

### 4.1 getSvgPathFromStroke - 全场景共享的路径生成

**位置: `packages/common/src/utils.ts:1103-1134`**

```typescript
/**
 * 唯一真正跨模块共享的代码
 *
 * 已验证调用方：
 * ✓ 激光：AnimatedTrail.drawTrail() - animated-trail.ts:303
 * ✓ 套索：继承 AnimatedTrail，间接使用
 * ✓ 橡皮擦：继承 AnimatedTrail，间接使用
 * ✓ 手绘 SVG 导出：getFreeDrawSvgPath() - shape.ts:1176
 *
 * 注意：手绘还有本地独立实现（shape.ts:1211）用于 Canvas 渲染
 */
export function getSvgPathFromStroke(points: number[][], closed = true) {
  const len = points.length;

  if (len < 4) {
    return ``;
  }

  let a = points[0];
  let b = points[1];
  const c = points[2];

  // Q + T 命令链实现（与手绘本地版本不同）
  let result = `M${a[0].toFixed(2)},${a[1].toFixed(2)} Q${b[0].toFixed(
    2,
  )},${b[1].toFixed(2)} ${average(b[0], c[0]).toFixed(2)},${average(
    b[1],
    c[1],
  ).toFixed(2)} T`;

  for (let i = 2, max = len - 1; i < max; i++) {
    a = points[i];
    b = points[i + 1];
    result += `${average(a[0], b[0]).toFixed(2)},${average(a[1], b[1]).toFixed(
      2,
    )} `;
  }

  if (closed) {
    result += "Z";
  }

  return result;
}

function average(a: number, b: number) {
  return (a + b) / 2;
}
```

### 4.2 两个 SVG 路径函数实现对比（仓库内可验证）

| 特性 | 手绘本地版本 (shape.ts:1211) | 共享版本 (common/utils.ts:1103) |
|------|-----------------------------|-------------------------------|
| 实现方式 | reduce + 中点逐点构建 Q 命令 | 首点显式 Q，后续 T 命令链 |
| 控制点策略 | 每对相邻点取中点作控制点 | 利用 SVG T 命令的反射特性 |
| 数值精度 | 正则表达式后处理统一 toFixed | 生成时直接 toFixed |
| 使用方 | 手绘 Canvas 渲染 | 激光/套索/橡皮擦帧渲染 + 手绘 SVG 导出 |
| 是否共享 | ✗ 本地私有 | ✓ 四工具共享 |

### 4.3 共享继承关系（仓库内可验证）

```
AnimatedTrail 基类 (animated-trail.ts)
    ├─ LaserTrail (laser-trails.ts)
    ├─ LassoTrail (lasso/index.ts)
    └─ EraserTrail (eraser/index.ts)
         ↓
         都使用 AnimationFrameHandler 进行帧调度
         都调用 getSvgPathFromStroke (common) 生成路径
         都使用 @excalidraw/laser-pointer 生成轮廓
```

---

## 五、协作轨迹处理详解

### 5.1 LaserTrails 协作管理器架构（与源码完全对齐）

**位置: `packages/excalidraw/laser-trails.ts:102-128`**

```typescript
export class LaserTrails implements Trail {
  public localTrail: AnimatedTrail;           // 本地用户轨迹
  private collabTrails = new Map<SocketId, AnimatedTrail>();  // 协作者轨迹

  private updateCollabTrails() {
    // 遍历所有协作者
    for (const [key, collaborator] of this.app.state.collaborators.entries()) {
      let trail!: AnimatedTrail;

      // 按需创建新轨迹实例
      if (!this.collabTrails.has(key)) {
        trail = new AnimatedTrail(this.animationFrameHandler, this.app, {
          ...this.getTrailOptions(),
          fill: () =>
            collaborator.pointer?.laserColor ||
            getClientColor(key, collaborator),  // 每个协作者独立颜色
        });
        trail.start(this.container);
        this.collabTrails.set(key, trail);
      } else {
        trail = this.collabTrails.get(key)!;
      }

      // 根据指针状态驱动轨迹（与源码完全一致）
      if (collaborator.pointer && collaborator.pointer.tool === "laser") {
        // 1. 按下且无当前轨迹 → 开始
        if (collaborator.button === "down" && !trail.hasCurrentTrail) {
          trail.startPath(collaborator.pointer.x, collaborator.pointer.y);
        }

        // 2. 按下且有当前轨迹且坐标不重复 → 追加点
        if (
          collaborator.button === "down" &&
          trail.hasCurrentTrail &&
          !trail.hasLastPoint(collaborator.pointer.x, collaborator.pointer.y)
        ) {
          trail.addPointToPath(collaborator.pointer.x, collaborator.pointer.y);
        }

        // 3. 抬起且有当前轨迹 → 先追加点再结束
        // 顺序对齐源码：先 addPointToPath，再 endPath
        if (collaborator.button === "up" && trail.hasCurrentTrail) {
          trail.addPointToPath(collaborator.pointer.x, collaborator.pointer.y);
          trail.endPath();
        }
      }
    }

    // 清理离开的协作者
    for (const key of this.collabTrails.keys()) {
      if (!this.app.state.collaborators.has(key)) {
        const trail = this.collabTrails.get(key)!;
        trail.stop();
        this.collabTrails.delete(key);
      }
    }
  }
}
```

### 5.2 协作轨迹状态机（准确版）

```
协作者轨迹生命周期（源码可验证）：

collaborator 加入 collaborators Map
    ↓
首次检测到 → 创建 AnimatedTrail 实例
    ↓
    → 注入 AnimationFrameHandler
    → 调用 trail.start(container) 启动帧循环
    ↓
[ 每帧循环检查 ]
    ├─ pointer.tool === laser
    │   ├─ button === down && !hasCurrentTrail
    │   │   └─ startPath(x, y)
    │   ├─ button === down && hasCurrentTrail && !hasLastPoint(x, y)
    │   │   └─ addPointToPath(x, y)  // 去重条件：坐标比较
    │   └─ button === up && hasCurrentTrail
    │       ├─ addPointToPath(x, y)  // 顺序：先追加点
    │       └─ endPath()              // 再结束
    └─ 其他工具：忽略，轨迹自然衰减消失
    ↓
collaborator 离开 collaborators Map
    ↓
调用 trail.stop() → 停止帧循环
    ↓
从 collabTrails Map 删除
    ↓
WeakMap 引用自动释放 → GC 清理
```

---

## 六、关键差异对照表（最终准确版，全文一致）

| 维度 | 手绘 (Freedraw) | 动态轨迹（激光/套索/橡皮擦） |
|------|----------------|---------------------------|
| **底层库** | `perfect-freehand` npm 包 | `@excalidraw/laser-pointer` npm 包 |
| **点简化策略** | `points-on-curve` 的 `simplify(points, 0.75)` | 激光 `simplify: 0`，套索/橡皮擦（未显式设置） |
| **流线化参数** | `streamline: 0.5` | 激光 0.4、套索 0.4、橡皮擦 0.2 |
| **大小映射** | 静态：基于压力/速度一次性计算 | 动态：`sizeMapping` 每帧重算（时间+位置衰减） |
| **SVG 路径函数** | 本地独立实现 + common 共享版本 | 仅使用 common 共享版本 |
| **渲染方式** | Canvas + Rough.js 手绘质感 | SVG path 元素直接填充 |
| **帧调度器** | ❌ 不使用 AnimationFrameHandler | ✅ 激光、套索、橡皮擦 都使用 |
| **计算策略** | 计算一次 → ShapeCache 缓存 | 每帧全部重新计算 |
| **持久化** | 保存为 ExcalidrawElement | 仅内存中，无状态 |
| **协作处理** | 元素同步 + 增量更新 | 指针事件流 + 独立轨迹实例（仅激光） |
| **缓存机制** | ShapeCache 弱引用缓存 | 无缓存，每帧实时生成 |
| **pressure 字段含义** | 真实/模拟笔压值 0-1 | 被复用存储 timestamp |
| **easing 函数** | easeOutSine (Math.sin) | easeOut 二次函数 (t*(2-t)) |
| **轨迹基类** | 不继承 AnimatedTrail | 都继承 AnimatedTrail |
| **去重逻辑** | 无（点简化代替） | hasLastPoint(x, y) 坐标比较（仅协作场景） |

---

## 七、可验证性总结

### ✅ 仓库内可直接证实的事实

1. `getSvgPathFromStroke`（common 包）被手绘、激光、套索、橡皮擦共用
2. `AnimationFrameHandler` 被激光、套索、橡皮擦使用，手绘不使用
3. `AnimatedTrail` 是激光、套索、橡皮擦的共同基类
4. 手绘在 shape.ts 中有本地独立的 SVG 路径生成函数
5. 手绘使用 perfect-freehand，动态轨迹使用 @excalidraw/laser-pointer
6. 手绘使用 ShapeCache 静态缓存，动态轨迹每帧重算无缓存
7. 三种动态轨迹的 streamline、DECAY_TIME、DECAY_LENGTH 参数值
8. `hasLastPoint` 去重逻辑：坐标比较（animated-trail.ts:64-74）
9. `button === up` 时协作轨迹顺序：先 addPointToPath 再 endPath
10. 协作轨迹状态机的完整条件分支

### ⚠️ 基于调用接口的推导（仓库内不可见内部实现）

1. `@excalidraw/laser-pointer` 的内部默认参数值（size、thinning 等）
2. `@excalidraw/laser-pointer` 内部的平滑算法实现细节
3. perfect-freehand 库的具体算法实现细节

---

## 八、代码引用位置速查表

| 功能 | 文件位置 | 行号 | 可验证性 |
|------|---------|------|---------|
| perfect-freehand 手绘笔触生成 | `packages/element/src/shape.ts` | 1181-1200 | ✅ 仓库内源码 |
| 手绘本地 SVG 路径函数 | `packages/element/src/shape.ts` | 1211-1231 | ✅ 仓库内源码 |
| 共享 SVG 路径函数 | `packages/common/src/utils.ts` | 1103-1134 | ✅ 仓库内源码 |
| AnimatedTrail 基类 | `packages/excalidraw/animated-trail.ts` | 30-197 | ✅ 仓库内源码 |
| hasLastPoint 去重逻辑 | `packages/excalidraw/animated-trail.ts` | 64-74 | ✅ 仓库内源码 |
| 激光衰减配置 | `packages/excalidraw/laser-trails.ts` | 31-50 | ✅ 仓库内源码 |
| 激光协作管理器（完整版） | `packages/excalidraw/laser-trails.ts` | 102-128 | ✅ 仓库内源码 |
| AnimationFrameHandler | `packages/excalidraw/animation-frame-handler.ts` | 8-79 | ✅ 仓库内源码 |
| LassoTrail 实现 | `packages/excalidraw/lasso/index.ts` | 42-71 | ✅ 仓库内源码 |
| EraserTrail 实现 | `packages/excalidraw/eraser/index.ts` | 42-70 | ✅ 仓库内源码 |
| points-on-curve simplify | `packages/element/src/shape.ts` | 688-691 | ✅ 仓库内源码 |
| ShapeCache 缓存机制 | `packages/element/src/shape.ts` | 81-163 | ✅ 仓库内源码 |
| @excalidraw/laser-pointer 内部参数 | npm 包外部源码 | N/A | ❌ 仓库内不可见 |
