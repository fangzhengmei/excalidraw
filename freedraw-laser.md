# Excalidraw 手绘与激光指针实现分析

## 一、架构全景：分离与共享

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      唯一共享：getSvgPathFromStroke (common)            │
│                           仅 SVG 路径生成阶段共用                         │
├───────────────────────────────────┬─────────────────────────────────────┤
│        手绘渲染管线 (Freedraw)    │    激光渲染管线 (Laser)            │
├───────────────────────────────────┼─────────────────────────────────────┤
│  perfect-freehand → getStroke     │ AnimationFrameHandler 独占调度      │
│  + 本地 SVG 生成函数              │ @excalidraw/laser-pointer          │
│  + ShapeCache 缓存               │    ↳ 每帧动态 sizeMapping            │
│  + Rough.js 手绘质感              │    ↳ 实时衰减重算                   │
│  + points-on-curve simplify      │  + AnimatedTrail 帧渲染循环          │
│                                  │  + LaserTrails 协作管理器            │
├───────────────────────────────────┼─────────────────────────────────────┤
│        Canvas 2D 渲染            │      SVG 直接渲染                    │
│        保存为 ExcalidrawElement  │      仅内存中轨迹                    │
│        无帧动画循环              │      每帧全量重算                    │
└───────────────────────────────────┴─────────────────────────────────────┘
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
  // 1. 输入准备：合并坐标和压力
  const inputPoints = element.simulatePressure
    ? element.points                                // 仅坐标，库内部模拟压力
    : element.points.length
    ? element.points.map(([x, y], i) => [x, y, element.pressures[i]])
    : [[0, 0, 0.5]];

  // 2. 调用 perfect-freehand 核心算法
  return getStroke(inputPoints as number[][], {
    simulatePressure: element.simulatePressure,
    size: element.strokeWidth * 4.25,     // 大小放大系数
    thinning: 0.6,                         // 压力敏感程度
    smoothing: 0.5,                        // 点间平滑度
    streamline: 0.5,                       // 流线化（方向修正）
    easing: (t) => Math.sin((t * Math.PI) / 2),  // easeOutSine
    last: true,
  }) as [number, number][];
};
// └───────────────────────────────────────────────────────┘

// ┌─ 阶段2：生成 SVG 路径 ─────────────────────────────────┐
// 注意：这是 element 包本地函数，非 common 共享版本！
// 位置: packages/element/src/shape.ts:1211-1231
const med = (A: number[], B: number[]) => 
  [(A[0] + B[0]) / 2, (A[1] + B[1]) / 2];

const TO_FIXED_PRECISION = /(\s?[A-Z]?,?-?[0-9]*\.[0-9]{0,2})(([0-9]|e|-)*)/g;

const getSvgPathFromStroke = (points: number[][]): string => {
  if (!points.length) return "";
  
  const max = points.length - 1;
  
  // 二次贝塞尔曲线 + 中点策略
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

// ┌─ 阶段3：碰撞检测形状（简化） ────────────────────────────┐
// 位置: packages/element/src/shape.ts:688-746
// 使用 points-on-curve 库的 simplify 算法
const simplifiedPoints = simplify(
  element.points as Mutable<LocalPoint[]>,
  0.75,  // 容差值：越大点越少
);
// └───────────────────────────────────────────────────────┘
```

### 2.3 perfect-freehand 可观测工作流程

**证据来源：`packages/element/src/shape.ts` + perfect-freehand 库公开文档**

```
从调用参数可观测的处理链：

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

注意：手绘无帧动画循环 → ShapeCache 缓存结果，
      不使用 AnimationFrameHandler！
```

**证据：ShapeCache 验证**
```typescript
// 位置: packages/element/src/shape.ts:81-163
export class ShapeCache {
  private static cache = new WeakMap<
    ExcalidrawElement,
    { shape: ElementShape; theme: AppState["theme"] }
  >();

  // 生成后缓存，后续直接取用
  public static generateElementShape(element, renderConfig) {
    const cachedShape = renderConfig?.isExporting
      ? undefined
      : ShapeCache.get(element, renderConfig?.theme || null);
    
    if (cachedShape !== undefined) {
      return cachedShape;  // 使用缓存，不重算
    }
    // ... 生成形状 ...
  }
}
```

---

## 三、激光衰减管线 (Time Decay Pipeline)

### 3.1 @excalidraw/laser-pointer 可观测行为

**证据来源：`packages/excalidraw/animated-trail.ts` 调用方式**

```typescript
// 从代码调用可观测的类行为（注意：npm 包源码不在仓库内，
// 以下为基于调用接口的行为推导，非内部实现事实）
class LaserPointer {
  // 可观测：能通过 originalPoints 访问原始点
  // 证据: animated-trail.ts:65-71 hasLastPoint() 直接访问
  originalPoints: [number, number, number][];  // [x, y, timestamp]
  
  // 可观测：构造函数接收 options 覆盖
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

// 可观测的 Options 接口（来自 TypeScript 类型标注）
interface LaserPointerOptions {
  size: number;
  thinning: number;
  smoothing: number;
  streamline: number;
  simplify: number;
  
  // 激光核心：动态大小映射
  // 证据: laser-trails.ts:35-48 传入自定义函数
  sizeMapping?: (context: {
    pressure: number;        // 实际存储 timestamp
    currentIndex: number;    // 点索引
    totalLength: number;     // 总长度
  }) => number;
  
  keepHead?: boolean;        // 绘制中保持头部
}
```

> ⚠️ **重要说明**：`@excalidraw/laser-pointer` 为 npm 外部包，
> 其内部默认参数值（如 size 默认 8、thinning 默认 0.5 等）
> 在本仓库源码中不可直接验证，不应作为确定事实陈述。
> 以上仅为基于接口调用的可观测行为推导。

### 3.2 激光几何平滑与衰减配置

**位置: `packages/excalidraw/laser-trails.ts:31-50`**

```typescript
private getTrailOptions() {
  return {
    // ── 几何平滑参数（仓库内可观测的传值） ──────
    simplify: 0,           // 明确传值：不简化
                           // 设计意图：激光轨迹短暂，需要精确跟随
                           // 对比手绘：simplify(points, 0.75) 大幅简化
    
    streamline: 0.4,       // 明确传值：轻度流线化
                           // 平衡鼠标响应与视觉平滑
                           // 对比手绘：streamline: 0.5
    
    // ── 动态衰减映射（激光独有，仓库可观测） ────
    sizeMapping: (c) => {
      const DECAY_TIME = 1000;     // 1秒完全消失
      const DECAY_LENGTH = 50;     // 尾部50个点淡出
      
      // 可观测事实：pressure 字段被用作 timestamp 存储
      // 证据: animated-trail.ts:99 addPoint([x, y, performance.now()])
      const t = Math.max(
        0,
        1 - (performance.now() - c.pressure) / DECAY_TIME,
      );
      
      // 位置衰减：从头部到尾部自然淡出
      const l =
        (DECAY_LENGTH -
          Math.min(DECAY_LENGTH, c.totalLength - c.currentIndex)) /
        DECAY_LENGTH;
      
      // 两者取最小值，确保消失边界明确
      return Math.min(easeOut(l), easeOut(t));
    },
  } as Partial<LaserPointerOptions>;
}

// 缓动函数：先快后慢的二次衰减
// 位置: packages/common/src/utils.ts
export const easeOut = (t: number) => t * (2 - t);
```

### 3.3 AnimatedTrail 帧渲染循环与 AnimationFrameHandler

**位置: `packages/excalidraw/animated-trail.ts:30-79, 139-197`**

```typescript
// AnimationFrameHandler 仅激光链路使用，手绘不使用！
// 证据：全局搜索 AnimationFrameHandler，仅出现于 laser 相关文件

export class AnimatedTrail implements Trail {
  private currentTrail?: LaserPointer;   // 当前绘制中
  private pastTrails: LaserPointer[] = []; // 已结束但仍在衰减
  
  private container?: SVGSVGElement;
  private trailElement: SVGPathElement;

  constructor(
    private animationFrameHandler: AnimationFrameHandler,  // 注入依赖
    protected app: App,
    private options: Partial<LaserPointerOptions> & AnimatedTrailOptions,
  ) {
    // 注册到帧调度器
    this.animationFrameHandler.register(this, this.onFrame.bind(this));
    this.trailElement = document.createElementNS(SVG_NS, "path");
  }

  start(container?: SVGSVGElement) {
    // 启动帧循环
    this.animationFrameHandler.start(this);
  }

  stop() {
    // 停止帧循环
    this.animationFrameHandler.stop(this);
  }

  // ── 帧渲染核心 ─────────────────────────────────
  private onFrame() {
    const paths: string[] = [];

    // 阶段1：渲染历史轨迹（仍在衰减中）
    for (const trail of this.pastTrails) {
      paths.push(this.drawTrail(trail, this.app.state));
    }

    // 阶段2：渲染当前正在绘制的轨迹
    if (this.currentTrail) {
      const currentPath = this.drawTrail(this.currentTrail, this.app.state);
      paths.push(currentPath);
    }

    // 阶段3：清理已完全消失的轨迹
    this.pastTrails = this.pastTrails.filter((trail) => {
      return trail.getStrokeOutline().length !== 0;
    });

    // 阶段4：合并所有路径更新 SVG
    const svgPaths = paths.join(" ").trim();
    this.trailElement.setAttribute("d", svgPaths);
    this.trailElement.setAttribute(
      "fill",
      (this.options.fill ?? (() => "black"))(this),
    );
    
    // 关键特性：每帧都重新计算所有点！
    // 原因：sizeMapping 依赖 performance.now()，是动态函数
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

    const stroke = this.trailAnimation
      ? _stroke.slice(0, _stroke.length / 2)
      : _stroke;

    // 使用 common 包共享的 SVG 路径函数
    return getSvgPathFromStroke(stroke, true);
  }
}
```

---

## 四、共享模块边界准确说明

### 4.1 getSvgPathFromStroke - 唯一真正共享的函数

**位置: `packages/common/src/utils.ts:1103-1134`**

```typescript
/**
 * 唯一真正在手绘与激光间共享的代码模块
 * 
 * 共享边界说明：
 * ✓ 激光轨迹：AnimatedTrail.drawTrail() 第 303 行直接调用
 * ✓ 手绘 SVG 导出：getFreeDrawSvgPath() 间接使用？待验证
 * ✗ 手绘 Canvas 渲染：shape.ts 第 1211 行有本地独立实现
 */
export function getSvgPathFromStroke(points: number[][], closed = true) {
  const len = points.length;

  if (len < 4) {
    return ``;
  }

  let a = points[0];
  let b = points[1];
  const c = points[2];

  // Q + T 命令链实现
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

### 4.2 两个 SVG 函数实现对比表

| 特性 | 手绘本地版本 (shape.ts:1211) | 共享版本 (common/utils.ts:1103) |
|------|-----------------------------|-------------------------------|
| 实现方式 | reduce + 中点逐点构建 Q 命令 | 首点显式 Q，后续 T 命令链 |
| 控制点策略 | 每对相邻点取中点作控制点 | 利用 SVG T 命令的反射特性 |
| 数值精度 | 正则表达式后处理统一 toFixed | 生成时直接 toFixed |
| 使用者 | 手绘元素的 Shape 生成 | 激光轨迹的帧渲染 |
| 是否共享 | ✗ 本地私有 | ✓ 跨模块共享 |

### 4.3 AnimationFrameHandler - 激光独占的帧调度器

**位置: `packages/excalidraw/animation-frame-handler.ts:8-79`**

```typescript
/**
 * 帧动画调度器 - 仅激光链路使用，手绘不使用
 * 
 * 证据：全局搜索 AnimationFrameHandler
 * 1. laser-trails.ts: 注入并管理协作者轨迹
 * 2. animated-trail.ts: 注册 onFrame 回调
 * 3. 无任何 freedraw / shape 相关文件引用
 */
export class AnimationFrameHandler {
  private targets = new WeakMap<object, AnimationTarget>();
  private rafIds = new WeakMap<object, number>();

  register(key: object, callback: AnimationCallback) {
    this.targets.set(key, { callback, stopped: true });
  }

  start(key: object) {
    const target = this.targets.get(key);
    if (!target || this.rafIds.has(key)) return;
    
    this.targets.set(key, { ...target, stopped: false });
    this.scheduleFrame(key);
  }

  stop(key: object) {
    const target = this.targets.get(key);
    if (target && !target.stopped) {
      this.targets.set(key, { ...target, stopped: true });
    }
    this.cancelFrame(key);
  }

  private scheduleFrame(key: object) {
    const rafId = requestAnimationFrame(this.constructFrame(key));
    this.rafIds.set(key, rafId);
  }

  private constructFrame(key: object): FrameRequestCallback {
    return (timestamp: number) => {
      const target = this.targets.get(key);
      if (!target) return;
      
      const shouldAbort = target.callback(timestamp) ?? false;
      
      if (!target.stopped && !shouldAbort) {
        this.scheduleFrame(key);
      } else {
        this.cancelFrame(key);
      }
    };
  }

  private cancelFrame(key: object) {
    if (this.rafIds.has(key)) {
      cancelAnimationFrame(this.rafIds.get(key)!);
    }
    this.rafIds.delete(key);
  }
}
```

---

## 五、协作轨迹处理详解

### 5.1 LaserTrails 协作管理器架构

**位置: `packages/excalidraw/laser-trails.ts:13-129`**

```typescript
export class LaserTrails implements Trail {
  public localTrail: AnimatedTrail;           // 本地用户轨迹
  private collabTrails = new Map<SocketId, AnimatedTrail>();  // 协作者轨迹
  
  private container?: SVGSVGElement;

  constructor(
    private animationFrameHandler: AnimationFrameHandler,
    private app: App,
  ) {
    // LaserTrails 自身也注册到帧调度器
    // 用途：每帧检查并更新所有协作者轨迹状态
    this.animationFrameHandler.register(this, this.onFrame.bind(this));

    this.localTrail = new AnimatedTrail(animationFrameHandler, app, {
      ...this.getTrailOptions(),
      fill: () => DEFAULT_LASER_COLOR,
    });
  }

  onFrame() {
    this.updateCollabTrails();
  }

  private updateCollabTrails() {
    // 快速路径：无协作者直接返回
    if (!this.container || this.app.state.collaborators.size === 0) {
      return;
    }

    // 阶段1：遍历所有协作者
    for (const [key, collaborator] of this.app.state.collaborators.entries()) {
      let trail!: AnimatedTrail;

      // 阶段2：按需创建新轨迹实例
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

      // 阶段3：根据指针状态驱动轨迹
      if (collaborator.pointer && collaborator.pointer.tool === "laser") {
        // 按下：开始新路径
        if (collaborator.button === "down" && !trail.hasCurrentTrail) {
          trail.startPath(collaborator.pointer.x, collaborator.pointer.y);
        }

        // 按下且已有路径：追加点（去重保护）
        if (
          collaborator.button === "down" &&
          trail.hasCurrentTrail &&
          !trail.hasLastPoint(collaborator.pointer.x, collaborator.pointer.y)
        ) {
          trail.addPointToPath(collaborator.pointer.x, collaborator.pointer.y);
        }

        // 抬起：结束路径
        if (collaborator.button === "up" && trail.hasCurrentTrail) {
          trail.addPointToPath(collaborator.pointer.x, collaborator.pointer.y);
          trail.endPath();
        }
      }
    }

    // 阶段4：清理离开的协作者
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

### 5.2 协作轨迹生命周期状态机

**证据来源：`laser-trails.ts` 条件分支分析**

```
协作者轨迹生命周期：

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
    │   ├─ button === down && hasCurrentTrail && !hasLastPoint
    │   │   └─ addPointToPath(x, y)  // 去重保护
    │   └─ button === up && hasCurrentTrail
    │       ├─ addPointToPath(x, y)  // 补充终点
    │       └─ endPath()
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

## 六、关键差异对照表（准确版）

| 维度 | 手绘 (Freedraw) | 激光指针 (Laser) |
|------|----------------|-----------------|
| **底层库** | `perfect-freehand` npm 包 | `@excalidraw/laser-pointer` npm 包 |
| **点简化策略** | `points-on-curve` 的 `simplify(points, 0.75)` | `simplify: 0` 无简化（明确传值） |
| **流线化参数** | `streamline: 0.5`（明确传值） | `streamline: 0.4`（明确传值） |
| **大小映射** | 静态：基于压力/速度一次性计算 | 动态：`sizeMapping` 每帧重算（时间+位置衰减） |
| **SVG 路径函数** | 本地独立实现（shape.ts:1211） | 共享 common 版本（utils.ts:1103） |
| **渲染方式** | Canvas + Rough.js 手绘质感 | SVG path 元素直接填充 |
| **帧调度器** | ❌ 不使用 AnimationFrameHandler | ✅ 独占使用 |
| **计算策略** | 计算一次 → ShapeCache 缓存 | 每帧全部重新计算 |
| **持久化** | 保存为 ExcalidrawElement | 仅内存中，无状态 |
| **协作处理** | 元素同步 + 增量更新 | 指针事件流 + 独立轨迹实例 |
| **缓存机制** | ShapeCache 弱引用缓存 | 无缓存，每帧实时生成 |
| **pressure 字段含义** | 真实/模拟笔压值 0-1 | 被复用存储 timestamp |
| **easing 函数** | easeOutSine (Math.sin) | easeOut 二次函数 (t*(2-t)) |

---

## 七、代码引用位置速查表

| 功能 | 文件位置 | 行号 | 可验证性 |
|------|---------|------|---------|
| perfect-freehand 手绘笔触生成 | `packages/element/src/shape.ts` | 1181-1200 | ✅ 仓库内源码 |
| 手绘本地 SVG 路径函数 | `packages/element/src/shape.ts` | 1211-1231 | ✅ 仓库内源码 |
| 共享 SVG 路径函数 | `packages/common/src/utils.ts` | 1103-1134 | ✅ 仓库内源码 |
| AnimatedTrail 基类 | `packages/excalidraw/animated-trail.ts` | 30-198 | ✅ 仓库内源码 |
| 激光衰减配置 | `packages/excalidraw/laser-trails.ts` | 31-50 | ✅ 仓库内源码 |
| LaserTrails 协作管理器 | `packages/excalidraw/laser-trails.ts` | 80-129 | ✅ 仓库内源码 |
| AnimationFrameHandler | `packages/excalidraw/animation-frame-handler.ts` | 8-79 | ✅ 仓库内源码 |
| points-on-curve simplify | `packages/element/src/shape.ts` | 688-691 | ✅ 仓库内源码 |
| ShapeCache 缓存机制 | `packages/element/src/shape.ts` | 81-163 | ✅ 仓库内源码 |
| @excalidraw/laser-pointer 内部参数 | npm 包外部源码 | N/A | ❌ 仓库内不可见 |
