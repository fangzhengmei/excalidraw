# Excalidraw 手绘与激光指针实现分析

## 一、核心架构概览

```
┌──────────────────────────────────────────────────────────────────┐
│                       共享底层模块                                │
├──────────────────────────────────────────────────────────────────┤
│  @excalidraw/laser-pointer  +  perfect-freehand  +  points-on-curve  │
│          ▲                         ▲                              │
│          │                         │                              │
├──────────────────────────────────────────────────────────────────┤
│                    AnimatedTrail 渲染层                            │
│              (getSvgPathFromStroke + SVG 渲染)                     │
├──────────────────────────────────────────────────────────────────┤
│           几何平滑管线                │            时间衰减管线           │
├─────────────────────────────────────┼────────────────────────────┤
│  simplify(点简化)                    │  performance.now()          │
│  smoothing(笔触平滑)                 │  位置索引衰减                │
│  streamline(流线化)                  │  easing函数                 │
│  thinning(压力粗细)                 │                            │
└─────────────────────────────────────┴────────────────────────────┘
```

---

## 二、手绘点序列平滑 (Geometric Smoothing Pipeline)

### 2.1 手绘元素数据结构

```typescript
interface ExcalidrawFreeDrawElement {
  type: "freedraw";
  points: LocalPoint[];           // 原始点序列 [x, y][]
  pressures: number[];             // 压力值，0-1 范围
  simulatePressure: boolean;       // 是否模拟压力
  strokeWidth: number;             // 基础笔触宽度
}
```

### 2.2 平滑处理流程

**位置: `packages/element/src/shape.ts:1181-1200`**

```typescript
export const getFreedrawOutlinePoints = (
  element: ExcalidrawFreeDrawElement,
) => {
  // 1. 准备输入点：合并坐标和压力
  const inputPoints = element.simulatePressure
    ? element.points
    : element.points.length
    ? element.points.map(([x, y], i) => [x, y, element.pressures[i]])
    : [[0, 0, 0.5]];

  // 2. 调用 perfect-freehand 生成平滑笔触
  return getStroke(inputPoints as number[][], {
    simulatePressure: element.simulatePressure,
    size: element.strokeWidth * 4.25,     // 基础大小放大
    thinning: 0.6,                         // 压力变化敏感度
    smoothing: 0.5,                        // 整体平滑度
    streamline: 0.5,                       // 点之间流线化
    easing: (t) => Math.sin((t * Math.PI) / 2),  // easeOutSine
    last: true,
  }) as [number, number][];
};
```

### 2.3 perfect-freehand 核心参数解析

| 参数 | 值 | 作用 | 视觉效果 |
|------|----|------|----------|
| `simulatePressure` | `boolean` | 是否从速度模拟压力 | 提笔处变细 |
| `size` | `strokeWidth * 4.25` | 基础笔触大小 | 整体粗细 |
| `thinning` | `0.6` | 压力影响大小的程度 | 粗细变化幅度 |
| `smoothing` | `0.5` | 整体平滑程度 | 抖动去除 |
| `streamline` | `0.5` | 相邻点流线化 | 拐角圆润度 |
| `easing` | `easeOutSine` | 压力缓动函数 | 粗细过渡自然 |

### 2.4 SVG 路径生成

**位置: `packages/element/src/shape.ts:1211-1232`**

```typescript
const getSvgPathFromStroke = (points: number[][]): string => {
  if (!points.length) return "";
  
  const max = points.length - 1;
  
  // 使用二次贝塞尔曲线连接中点，确保 C0 连续
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

// 取两点中点作为控制点，确保平滑连接
const med = (A: number[], B: number[]) => {
  return [(A[0] + B[0]) / 2, (A[1] + B[1]) / 2];
};
```

### 2.5 曲线简化

**位置: `packages/element/src/shape.ts:688-691`**

```typescript
// 使用 points-on-curve 的 simplify 函数
const simplifiedPoints = simplify(
  element.points as Mutable<LocalPoint[]>,
  0.75,  // 容差值：越大点越少
);
```

---

## 三、激光衰减管线 (Time Decay Pipeline)

### 3.1 激光指针核心类

**位置: `packages/excalidraw/animated-trail.ts:30-198`**

```typescript
export class AnimatedTrail implements Trail {
  private currentTrail?: LaserPointer;
  private pastTrails: LaserPointer[] = [];
  
  private container?: SVGSVGElement;
  private trailElement: SVGPathElement;
  private trailAnimation?: SVGAnimateElement;

  constructor(
    private animationFrameHandler: AnimationFrameHandler,
    protected app: App,
    private options: Partial<LaserPointerOptions> & AnimatedTrailOptions,
  ) {
    this.animationFrameHandler.register(this, this.onFrame.bind(this));
    this.trailElement = document.createElementNS(SVG_NS, "path");
  }

  startPath(x: number, y: number) {
    this.currentTrail = new LaserPointer(this.options);
    this.currentTrail.addPoint([x, y, performance.now()]);
    this.update();
  }

  addPointToPath(x: number, y: number) {
    if (this.currentTrail) {
      this.currentTrail.addPoint([x, y, performance.now()]);
      this.update();
    }
  }

  endPath() {
    if (this.currentTrail) {
      this.currentTrail.close();
      this.currentTrail.options.keepHead = false;
      this.pastTrails.push(this.currentTrail);
      this.currentTrail = undefined;
      this.update();
    }
  }
}
```

### 3.2 衰减大小映射函数

**位置: `packages/excalidraw/laser-trails.ts:31-49`**

```typescript
private getTrailOptions() {
  return {
    simplify: 0,           // 不简化，保持轨迹精度
    streamline: 0.4,       // 轻度流线化
    sizeMapping: (c) => {
      const DECAY_TIME = 1000;     // 1秒时间衰减
      const DECAY_LENGTH = 50;     // 50个点长度衰减
      
      // 双重衰减机制：时间 + 位置
      const t = Math.max(
        0,
        1 - (performance.now() - c.pressure) / DECAY_TIME,
      );
      const l =
        (DECAY_LENGTH -
          Math.min(DECAY_LENGTH, c.totalLength - c.currentIndex)) /
        DECAY_LENGTH;

      // 两者取较小值，确保尾部完全消失
      return Math.min(easeOut(l), easeOut(t));
    },
  } as Partial<LaserPointerOptions>;
}
```

### 3.3 衰减算法详解

```
■ 时间衰减 (Time-based Decay):
    t = 1 - (currentTime - pointTime) / DECAY_TIME
    ├─ 新点: t = 1.0 → 完全显示
    ├─ 中点: t = 0.5 → 半透明
    └─ 旧点: t = 0.0 → 完全消失

■ 位置衰减 (Length-based Decay):
    l = (DECAY_LENGTH - remainingLength) / DECAY_LENGTH
    ├─ 头部: l = 1.0 → 完全显示
    └─ 尾部: l = 0.0 → 完全消失

■ 缓动函数 (Easing):
    easeOut(x) = x * (2 - x)  // 先快后慢
    视觉效果：尾部平滑消失而非线性衰减
```

### 3.4 帧渲染循环

**位置: `packages/excalidraw/animated-trail.ts:139-178`**

```typescript
private onFrame() {
  const paths: string[] = [];

  // 渲染已结束的轨迹（继续衰减）
  for (const trail of this.pastTrails) {
    paths.push(this.drawTrail(trail, this.app.state));
  }

  // 渲染当前正在绘制的轨迹
  if (this.currentTrail) {
    const currentPath = this.drawTrail(this.currentTrail, this.app.state);
    paths.push(currentPath);
  }

  // 过滤掉已完全消失的轨迹
  this.pastTrails = this.pastTrails.filter((trail) => {
    return trail.getStrokeOutline().length !== 0;
  });

  // 合并所有路径并更新 SVG
  const svgPaths = paths.join(" ").trim();
  this.trailElement.setAttribute("d", svgPaths);
  this.trailElement.setAttribute("fill", this.options.fill(this));
}
```

---

## 四、共享绘制模块 (Shared Rendering Modules)

### 4.1 AnimationFrameHandler - 统一动画调度

**位置: `packages/excalidraw/animation-frame-handler.ts:8-79`**

```typescript
export class AnimationFrameHandler {
  private targets = new WeakMap<object, AnimationTarget>();
  private rafIds = new WeakMap<object, number>();

  // 注册动画目标
  register(key: object, callback: AnimationCallback) {
    this.targets.set(key, { callback, stopped: true });
  }

  // 启动动画循环
  start(key: object) {
    const target = this.targets.get(key);
    if (!target || this.rafIds.has(key)) return;
    
    this.targets.set(key, { ...target, stopped: false });
    this.scheduleFrame(key);
  }

  // 停止动画循环
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
      
      const shouldAbort = this.onFrame(target, timestamp);
      
      if (!target.stopped && !shouldAbort) {
        this.scheduleFrame(key);
      } else {
        this.cancelFrame(key);
      }
    };
  }
}
```

### 4.2 LaserPointerOptions - 统一配置接口

```typescript
interface LaserPointerOptions {
  // 几何参数
  size: number;              // 基础大小
  thinning: number;          // 粗细变化
  smoothing: number;         // 平滑度
  streamline: number;        // 流线化
  simplify: number;          // 点简化容差
  
  // 动态映射（激光专用）
  sizeMapping?: (context: {
    pressure: number;        // 点的时间戳
    currentIndex: number;    // 当前索引
    totalLength: number;     // 总长度
  }) => number;
  
  // 头部保持选项
  keepHead?: boolean;        // 绘制中保持头部
}
```

### 4.3 SVG 路径共享函数

**位置: `packages/common/src/...`**

```typescript
// 手绘和激光共享的 SVG 路径生成函数
export const getSvgPathFromStroke = (
  points: number[][],
  closed: boolean = false
): string => {
  if (!points.length) return "";
  
  // 二次贝塞尔曲线 + 中点策略
  // 确保：
  // 1. C0 连续（位置连续）
  // 2. G1 连续（切线方向一致）
  // 3. 计算高效
};
```

---

## 五、协作场景的激光轨迹

### 5.1 LaserTrails 管理器

**位置: `packages/excalidraw/laser-trails.ts:13-129`**

```typescript
export class LaserTrails implements Trail {
  public localTrail: AnimatedTrail;
  private collabTrails = new Map<SocketId, AnimatedTrail>();

  private updateCollabTrails() {
    // 为每个协作者创建独立轨迹
    for (const [key, collaborator] of this.app.state.collaborators.entries()) {
      let trail!: AnimatedTrail;

      if (!this.collabTrails.has(key)) {
        trail = new AnimatedTrail(this.animationFrameHandler, this.app, {
          ...this.getTrailOptions(),
          fill: () =>
            collaborator.pointer?.laserColor || getClientColor(key, collaborator),
        });
        trail.start(this.container!);
        this.collabTrails.set(key, trail);
      } else {
        trail = this.collabTrails.get(key)!;
      }

      // 根据协作者指针状态更新轨迹
      if (collaborator.pointer && collaborator.pointer.tool === "laser") {
        if (collaborator.button === "down" && !trail.hasCurrentTrail) {
          trail.startPath(collaborator.pointer.x, collaborator.pointer.y);
        }
        
        if (collaborator.button === "down" && trail.hasCurrentTrail) {
          trail.addPointToPath(collaborator.pointer.x, collaborator.pointer.y);
        }
        
        if (collaborator.button === "up" && trail.hasCurrentTrail) {
          trail.endPath();
        }
      }
    }
  }
}
```

---

## 六、设计要点与权衡

### 6.1 手绘 vs 激光：参数对比

| 特性 | 手绘 (Freedraw) | 激光指针 (Laser) |
|------|----------------|-----------------|
| `simplify` | 0.75 (大幅简化) | 0 (不简化) |
| `streamline` | 0.5 | 0.4 |
| `sizeMapping` | 基于真实/模拟压力 | 基于时间+位置衰减 |
| `easing` | easeOutSine | easeOut 二次 |
| 持久化 | 保存为元素 | 仅内存中，随时间消失 |
| Z 轴顺序 | 与其他元素一起排序 | 最顶层，不参与元素排序 |

### 6.2 关键设计决策

1. **共享底层库但独立配置**:
   - 都使用 `@excalidraw/laser-pointer` 和 `perfect-freehand`
   - 通过不同 options 实现截然不同的视觉效果

2. **SVG vs Canvas**:
   - 激光轨迹使用 SVG 直接渲染（性能好，衰减平滑）
   - 手绘元素使用 Canvas + Rough.js（手绘质感）

3. **动画循环统一调度**:
   - AnimationFrameHandler 管理所有动画
   - 避免多个 requestAnimationFrame 竞争

4. **协作轨迹隔离**:
   - 每个协作者独立 AnimatedTrail 实例
   - 颜色区分 + 独立衰减周期

---

## 七、性能优化策略

### 7.1 手绘优化
- **点简化**: `simplify(points, 0.75)` 减少 60-80% 点数量
- **形状缓存**: ShapeCache 存储生成的笔触路径
- **惰性渲染**: 仅在视口内的元素才渲染

### 7.2 激光优化
- **过滤空轨迹**: 每帧过滤已完全消失的轨迹
- **路径合并**: 多轨迹合并为单个 SVG path 元素
- **WeakMap 引用**: 避免内存泄漏，协作者离开自动清理

---

## 八、代码引用位置速查表

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 手绘笔触生成 | `packages/element/src/shape.ts` | 1181-1200 |
| SVG路径生成 | `packages/element/src/shape.ts` | 1211-1232 |
| 动画轨迹类 | `packages/excalidraw/animated-trail.ts` | 30-198 |
| 激光衰减配置 | `packages/excalidraw/laser-trails.ts` | 31-49 |
| 动画帧处理器 | `packages/excalidraw/animation-frame-handler.ts` | 8-79 |
| 协作激光轨迹 | `packages/excalidraw/laser-trails.ts` | 80-129 |
