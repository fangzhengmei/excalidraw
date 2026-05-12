# Excalidraw 手绘与激光指针实现分析

## 一、架构全景：分离与共享

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           共享公共模块                                    │
├─────────────────────────────────────────────────────────────────────────┤
│  AnimationFrameHandler 统一调度  +  getSvgPathFromStroke (common)        │
│         ▲                               ▲                                │
│         │                               │                                │
├───────────────────────────────────┬─────────────────────────────────────┤
│        手绘渲染管线 (Freedraw)    │    激光渲染管线 (Laser)            │
├───────────────────────────────────┼─────────────────────────────────────┤
│  perfect-freehand → getStroke     │ @excalidraw/laser-pointer          │
│  + 本地 SVG 生成函数              │    ↳ LaserPointer 类                │
│  + ShapeCache 缓存               │    ↳ getStrokeOutline               │
│  + Rough.js 手绘质感              │    ↳ 内置 streamline/simplify       │
│  + points-on-curve simplify      │  + sizeMapping 动态衰减              │
│                                  │  + 每帧重算机制                      │
├───────────────────────────────────┼─────────────────────────────────────┤
│        Canvas 2D 渲染            │      SVG 直接渲染                    │
│        保存为 ExcalidrawElement  │      仅内存中轨迹                    │
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

### 2.3 perfect-freehand 工作原理

```
perfect-freehand 内部处理链：

Input Points → [x, y, pressure]
    ↓
1. streamline 阶段
   ├─ 计算每个点的移动方向
   ├─ 基于相邻点修正点位置
   └─ 消除抖动，让轨迹更流畅
    ↓
2. smoothing 阶段
   ├─ 高斯模糊-like 的点平滑
   └─ 减少尖锐拐角
    ↓
3. thinning 阶段
   ├─ pressure → 笔触宽度
   ├─ easing 函数变换压力曲线
   └─ 计算每个点的左右偏移量
    ↓
4. 输出轮廓点
   └─ 闭合多边形顶点 [x, y][]
```

---

## 三、激光衰减管线 (Time Decay Pipeline)

### 3.1 @excalidraw/laser-pointer 内部架构

```typescript
// 内部类结构（推导自使用方式）
class LaserPointer {
  // 原始点存储
  originalPoints: [number, number, number][];  // [x, y, timestamp]
  
  // 配置
  options: LaserPointerOptions;
  
  constructor(options: Partial<LaserPointerOptions>) {
    this.options = {
      size: 8,
      thinning: 0.5,
      smoothing: 0.5,
      streamline: 0.4,
      simplify: 0,          // 激光：不简化
      sizeMapping: undefined,
      keepHead: true,
      ...options,
    };
  }
  
  addPoint(point: [number, number, number]): void {
    this.originalPoints.push(point);
  }
  
  close(): void {
    this.options.keepHead = false;
  }
  
  // 核心：生成带衰减的笔触轮廓
  getStrokeOutline(scale: number = 1): [number, number][] {
    // 1. 对每个点计算动态大小
    //    sizeMapping(pressure, currentIndex, totalLength)
    // 2. 内置 streamline + smoothing 处理
    // 3. 计算轮廓偏移
    // 4. 返回闭合多边形点
  }
}

interface LaserPointerOptions {
  size: number;
  thinning: number;
  smoothing: number;
  streamline: number;
  simplify: number;
  
  // 动态大小映射（激光核心！）
  sizeMapping?: (context: {
    pressure: number;        // 存储的 timestamp
    currentIndex: number;    // 当前点索引
    totalLength: number;     // 总点数
  }) => number;
  
  keepHead?: boolean;        // 保持头部不消失
}
```

### 3.2 激光几何平滑细节

**位置: `packages/excalidraw/laser-trails.ts:31-50`**

```typescript
private getTrailOptions() {
  return {
    // ── 几何平滑参数 ─────────────────────────────
    simplify: 0,           // 激光：完全不简化，保留所有点
                           // 原因：轨迹短暂，需要精确跟随鼠标
                           // 手绘：0.75，大幅减少点数量
    
    streamline: 0.4,       // 轻度流线化，平衡响应速度与平滑
                           // 手绘：0.5，更注重平滑
    
    // ── 动态衰减映射（激光独有） ──────────────────
    sizeMapping: (c) => {
      const DECAY_TIME = 1000;     // 1秒完全消失
      const DECAY_LENGTH = 50;     // 尾部50个点淡出
      
      // 双重衰减机制
      // 1. 时间衰减：按点添加时间计算剩余可见度
      const t = Math.max(
        0,
        1 - (performance.now() - c.pressure) / DECAY_TIME,
        //           ↑ 这里 pressure 实际存储的是 timestamp！
      );
      
      // 2. 位置衰减：从头部到尾部自然淡出
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

### 3.3 AnimatedTrail 帧渲染循环

**位置: `packages/excalidraw/animated-trail.ts:139-197`**

```typescript
export class AnimatedTrail implements Trail {
  private currentTrail?: LaserPointer;   // 当前绘制中
  private pastTrails: LaserPointer[] = []; // 已结束但仍在衰减
  
  private onFrame() {
    const paths: string[] = [];

    // ── 阶段1：渲染历史轨迹（继续衰减） ─────────
    for (const trail of this.pastTrails) {
      paths.push(this.drawTrail(trail, this.app.state));
    }

    // ── 阶段2：渲染当前轨迹 ────────────────────
    if (this.currentTrail) {
      const currentPath = this.drawTrail(this.currentTrail, this.app.state);
      paths.push(currentPath);
    }

    // ── 阶段3：清理已完全消失的轨迹 ────────────
    this.pastTrails = this.pastTrails.filter((trail) => {
      // 轮廓为空 = 已完全消失
      return trail.getStrokeOutline().length !== 0;
    });

    // ── 阶段4：合并路径更新 SVG ────────────────
    const svgPaths = paths.join(" ").trim();
    this.trailElement.setAttribute("d", svgPaths);
    
    // 注意：每帧都重新计算所有点！
    // 因为 sizeMapping 依赖 performance.now()，是动态的
  }

  private drawTrail(trail: LaserPointer, state: AppState): string {
    // 1. 调用 laser-pointer 获取带衰减的轮廓
    const _stroke = trail
      .getStrokeOutline(trail.options.size / state.zoom.value)
      .map(([x, y]) => {
        // 2. 坐标变换：场景坐标 → 视口坐标
        const result = sceneCoordsToViewportCoords(
          { sceneX: x, sceneY: y },
          state,
        );
        return [result.x, result.y];
      });

    // 3. 动画模式：只使用一半点（虚线效果）
    const stroke = this.trailAnimation
      ? _stroke.slice(0, _stroke.length / 2)
      : _stroke;

    // 4. 调用 common 包共享的 SVG 路径生成
    return getSvgPathFromStroke(stroke, true);
  }
}
```

---

## 四、真实共享模块详解

### 4.1 getSvgPathFromStroke - 唯一真正共享的路径函数

**位置: `packages/common/src/utils.ts:1103-1134`**

```typescript
/**
 * 唯一真正共享的 SVG 路径生成函数
 * 与手绘本地版本的区别：使用 T 命令的二次贝塞尔平滑
 */
export function getSvgPathFromStroke(points: number[][], closed = true) {
  const len = points.length;

  if (len < 4) {
    return ``;
  }

  let a = points[0];
  let b = points[1];
  const c = points[2];

  // 关键区别：使用 Q + T 命令链，而非手绘版本的 reduce 模式
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

**两个 SVG 函数对比：**

| 特性 | 手绘本地版本 (shape.ts) | 共享版本 (common/utils.ts) |
|------|-------------------------|---------------------------|
| 实现方式 | reduce + 中点逐点构建 | Q + T 命令链 |
| 平滑度 | 每点显式控制点 | 利用 T 命令的反射特性 |
| 精度处理 | 正则替换统一处理 | 生成时直接 toFixed |
| 使用者 | 手绘 SVG 导出 | 激光轨迹渲染 |

### 4.2 AnimationFrameHandler - 统一动画调度

**位置: `packages/excalidraw/animation-frame-handler.ts:8-79`**

```typescript
export class AnimationFrameHandler {
  // 使用 WeakMap，避免阻止 GC
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
}

// 典型使用：
// LaserTrails 注册自身的 onFrame
// AnimatedTrail 注册自身的 onFrame
// 两者共享同一个调度器，但独立启停
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
    // LaserTrails 自身也注册到动画帧
    // 用于：每帧更新协作者轨迹
    this.animationFrameHandler.register(this, this.onFrame.bind(this));

    // 创建本地用户轨迹
    this.localTrail = new AnimatedTrail(animationFrameHandler, app, {
      ...this.getTrailOptions(),
      fill: () => DEFAULT_LASER_COLOR,
    });
  }

  onFrame() {
    // 每帧都执行：检查并更新所有协作者轨迹
    this.updateCollabTrails();
  }

  private updateCollabTrails() {
    // ── 快速路径：无协作者直接返回 ──────
    if (!this.container || this.app.state.collaborators.size === 0) {
      return;
    }

    // ── 阶段1：遍历所有协作者 ────────────
    for (const [key, collaborator] of this.app.state.collaborators.entries()) {
      let trail!: AnimatedTrail;

      // ── 阶段2：按需创建新轨迹实例 ──────
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

      // ── 阶段3：根据指针状态驱动轨迹 ────
      if (collaborator.pointer && collaborator.pointer.tool === "laser") {
        // 按下：开始新路径
        if (collaborator.button === "down" && !trail.hasCurrentTrail) {
          trail.startPath(collaborator.pointer.x, collaborator.pointer.y);
        }

        // 按下且已有路径：追加点（防重复）
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

    // ── 阶段4：清理离开的协作者 ──────────
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

### 5.2 协作轨迹状态机

```
协作轨迹生命周期：

collaborator 加入
    ↓
创建 AnimatedTrail 实例 → 注册到 AnimationFrameHandler
    ↓
[ 循环 ]
    ├─ pointer.tool === laser
    │   ├─ button down, no trail → startPath()
    │   ├─ button down, has trail → addPointToPath() (去重)
    │   └─ button up, has trail → endPath()
    └─ 否则：忽略
    ↓
collaborator 离开
    ↓
stop() 动画 + 从 Map 删除
    ↓
WeakMap 自动 GC 清理
```

---

## 六、关键差异对照表

| 维度 | 手绘 (Freedraw) | 激光指针 (Laser) |
|------|----------------|-----------------|
| **底层库** | `perfect-freehand` 独立库 | `@excalidraw/laser-pointer` 专用包 |
| **点简化** | `simplify(points, 0.75)` 大幅简化 | `simplify: 0` 不简化 |
| **流线化** | `streamline: 0.5` | `streamline: 0.4` |
| **大小映射** | 基于压力/速度的静态粗细 | `sizeMapping` 动态时间+位置衰减 |
| **SVG 函数** | 本地实现：reduce + 逐点 Q 命令 | 共享 common 版本：Q + T 命令链 |
| **渲染方式** | Canvas + Rough.js 手绘质感 | SVG path 直接填充 |
| **计算时机** | 元素变更时计算一次，缓存结果 | 每帧全部重新计算（动态衰减） |
| **持久化** | 保存为 ExcalidrawElement | 仅内存中，无状态 |
| **协作处理** | 元素同步 + 增量更新 | 指针事件流 + 独立轨迹实例 |
| **缓存策略** | ShapeCache 弱引用缓存 | 无缓存，每帧实时生成 |
| **pressure 含义** | 真实/模拟笔压 0-1 | 存储 timestamp 用于衰减 |

---

## 七、代码引用位置速查表

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| perfect-freehand 手绘笔触生成 | `packages/element/src/shape.ts` | 1181-1200 |
| 手绘本地 SVG 路径函数 | `packages/element/src/shape.ts` | 1211-1231 |
| 共享 SVG 路径函数 | `packages/common/src/utils.ts` | 1103-1134 |
| AnimatedTrail 基类 | `packages/excalidraw/animated-trail.ts` | 30-198 |
| 激光衰减配置 | `packages/excalidraw/laser-trails.ts` | 31-50 |
| LaserTrails 协作管理器 | `packages/excalidraw/laser-trails.ts` | 80-129 |
| AnimationFrameHandler | `packages/excalidraw/animation-frame-handler.ts` | 8-79 |
| points-on-curve simplify | `packages/element/src/shape.ts` | 688-691 |
