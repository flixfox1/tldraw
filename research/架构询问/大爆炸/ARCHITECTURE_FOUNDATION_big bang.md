# Matelink 无限画布 — 三层架构基石与重构指南

> 生成日期: 2026-02-17 | 修订日期: 2026-02-18
> 性质: 新架构的基石文档，指导后续 agent 对现系统进行重构
> 用途: 基于此文档与另一个 agent 进一步解耦剖析，形成 Spec，再通过 Spec 驱动重构
> 重构策略: Big Bang（零用户项目，无回滚风险，适配器认知成本 > 重写成本）
> 前置条件: 旧系统功能行为清单（由人工编制，作为新系统验收标准）

---

## 第一部分：问题诊断 — 为什么当前架构失控

### 1.1 核心症状

当前代码库面临的最严峻问题是依赖关系失控：

- **无效依赖**：同一个概念被实现了多次（3 套 LOD、2 套空间索引、2 套性能监控），各自管理自己的依赖，互不通信
- **连续依赖**：20 层 Provider 嵌套形成隐式依赖链，顶层任何一个 Provider 状态变化，整棵树都要 reconciliation
- **过度依赖**：MatelinkCanvas 消费 8+ 个 Context，LayerManager 45+ 个方法，所有人都依赖它们

### 1.2 根因

这些依赖问题的根因是**架构边界的缺失**。

代码库从来没有人定义过"谁能依赖谁"、"一个单元最多能承载多少"。没有这些约束，代码自然会往最方便的方向生长——需要什么就直接 import，一个文件能解决的就不拆两个。日积月累，就变成了现在的样子。

### 1.3 关键数据

| 维度 | 当前值 | 问题 |
|------|--------|------|
| MatelinkCanvas 行数 | ~2,730 | 混合视图、交互、领域、基础设施四种职责 |
| Provider 嵌套层数 | 20 | 隐式依赖链，任何顶层变化波及全树 |
| LayerManager 方法数 | 45+ | God Object，所有人的依赖中心 |
| 数据真相源 | 2 个 | LayerManager + ReactFlow nodes 双重状态 |
| 状态管理方案 | 3 种 | React Context × 17 + signals + useState，无统一方案 |
| 工具实现模式 | 3 种 | HSM + Reducer Hook + 独立引擎 |
| LOD 系统 | 3 套 | lib/lod/ + lib/drawing/lod/ + lib/pen/lod/ |


---

## 第二部分：架构设计的核心原则

### 2.1 架构设计是"宪法"

```
架构设计（宪法）
  │
  │ 决定了
  ▼
目录结构（行政区划）── 架构的物理映射，让边界可见、可检查
  │
  │ 约束了
  ▼
三个具体领域（法律）
  ├── 数据流动 ── 数据怎么从 A 到 B
  ├── 数据存储 ── 数据住在哪里（唯一真相源）
  └── 数据提取 ── 数据怎么被拿出来用
  │
  │ 细化为
  ▼
类型定义（合同条款）── 数据长什么样，接口怎么对接
```

目录树只是架构的"投影"。真正的硬规矩是架构设计本身——它定义了系统的骨骼。

### 2.2 职责边界是架构的根

架构最重要的不是"数据源的层级关系"，而是**职责边界**。数据源的层级关系是职责边界的结果，不是原因。

职责边界的划分依据是**变化的原因**：

> 因为同一个原因而变化的东西，放在一起。

对于无限画布产品，变化的原因分为四类：

| 变化原因 | 举例 | 归属 |
|----------|------|------|
| 用户看到的东西要变 | "按钮换个颜色"、"笔触加个阴影" | 视图层 |
| 用户的操作方式要变 | "双击改成长按"、"加个快捷键" | 交互模块 |
| 业务规则要变 | "笔触要支持压感"、"图层要支持锁定" | 领域模块 |
| 技术方案要变 | "R-tree 换成四叉树"、"存储从本地换成云端" | 基础设施模块 |

**判断方法**：拿到一块代码，问"它会因为什么原因被修改"。如果答案是多个原因，它就需要被拆开。

### 2.3 厚客户端原则

Matelink 是厚客户端架构。所有核心计算（渲染、交互、数据处理）都在浏览器端完成。服务器只在需要跨设备/协作/AI 功能时才介入。

| 维度 | 约束 | 说明 |
|------|------|------|
| Core 层独立性 | Store + Editor 不依赖任何服务器 | 所有业务逻辑、数据操作、工具行为在浏览器内闭环 |
| 离线可用 | 应用可以在完全离线状态下运行（PWA） | 网络断开不影响画布的创建、编辑、保存 |
| 服务器角色 | 服务器是可选的增强，不是必要依赖 | AI 算力、云端存储、实时协作是增值服务，不是基础能力 |

**架构推论**：
- 不允许在 Core 层（`core/store/`、`core/editor/`）中出现任何 `fetch`、`WebSocket`、服务器 URL 等网络依赖
- 网络相关的功能（同步、云存储、AI 调用）以中间件或独立模块的形式存在于 `core/infra/` 中，通过接口注入
- 应用的第一次启动不需要任何网络请求即可进入可用状态

### 2.4 数据传递的三种模式

数据在系统中的传递只有三种模式：

| 模式 | 含义 | 用在哪里 |
|------|------|----------|
| 单发（Command） | 一个发送者 → 一个接收者，做一件事 | 用户的每一个操作意图 |
| 并发（Event） | 一个发送者 → 多个接收者，各自处理 | 数据变化后通知多个消费者 |
| 转发（Query） | 请求者 → 中间层 → 数据源，只读 | 某一层需要另一层的数据 |

一个完整操作的生命周期：

```
单发（Command）── 触发变化
并发（Event）  ── 通知变化
转发（Query）  ── 读取结果
```


---

## 第三部分：三层架构设计

### 3.1 为什么是三层而不是四层

理论上，按变化原因可以分为四层（视图、交互、领域、基础设施）。但在画布软件的实践中，交互和领域的边界很模糊——"用户框选三个节点然后拖拽"这个操作，交互层需要知道元素在哪（领域知识），领域层需要知道用户在拖拽（交互知识）。硬拆成两层会增加不必要的间接性。

市面上成熟的画布软件（tldraw、Excalidraw、Figma）都选择了三层作为对外边界：

```
理论四层                          实践三层
─────────                        ─────────

视图层 ─────────────────────────→ 视图层（View）
                                     │
交互层 ──────┐                       │
             ├──────────────────→ Editor 层
领域层 ──────┘                       │
                                     │
基础设施层 ─────────────────────→ Store 层
```

**四层的概念在三层内部以模块的形式存在**，而不是作为独立的层暴露出来。

### 3.2 市面上画布软件的架构对比

| 维度 | tldraw | Excalidraw | Figma | Matelink（当前） |
|------|--------|------------|-------|-----------------|
| 数据真相源 | Store（1个） | Scene + Jotai（1个） | C++ 文档模型（1个） | LayerManager + ReactFlow nodes（2个） |
| 状态管理方案 | 自研 Store + computed | Jotai atoms | C++ 内存模型 | React Context × 17 + signals + useState |
| 工具模式 | StateNode（1种） | ActionManager（1种） | 命令系统（1种） | HSM + Reducer + 独立引擎（3种） |
| 形状扩展 | ShapeUtil 注册 | 元素类型注册 | 插件系统 | if (type === 'xxx') 分支 |
| 操作入口 | Editor facade | ActionManager | C++ 桥接 | MatelinkCanvas 直接操作 |
| 视图和数据分离 | Record vs ShapeUtil | 元素数据 vs 渲染函数 | C++ vs React | 混在 MatelinkCanvas 里 |

**共同模式**：数据只住一个地方，操作只走一个入口，渲染只管画画。

### 3.2.1 为什么选 Zustand 作为 Store 层的响应式基座

tldraw 自研了 Store 的响应式系统（signia），Excalidraw 用了 Jotai。我们不需要自研——Zustand 已经精确覆盖了 Store 层的所有需求：

| 需求 | Zustand 如何满足 | 自研 Store 的代价 |
|------|-----------------|------------------|
| 唯一真相源 | 一个 store = 一份数据 | 需要自己实现 Map + 事件系统 |
| React 精确订阅 | `useStore(selector)` 只在 selector 返回值变化时重渲染 | 需要自己实现 `useSyncExternalStore` + 浅比较 |
| React 外可访问 | `store.getState()` / `store.setState()` 在任何 JS 代码中可用 | 需要自己暴露 getState API |
| 事务/批量更新 | `set()` 内的多次修改合并为一次通知 | 需要自己实现 beginTransaction/commit |
| 中间件扩展 | `devtools()`、`persist()`、`immer()` 开箱即用 | 需要自己实现 |
| 不依赖 React | `zustand/vanilla` 是纯 JS，零 React 依赖 | — |
| 替代 Provider | Zustand 不需要 Provider 包裹 | — |

**关键决策：Store 层使用 `zustand/vanilla` 创建，不依赖 React。视图层通过 `zustand` 的 React hook 订阅。**

这意味着：
- `core/store/` 目录下的代码 import `zustand/vanilla`，不 import `react`
- `view/hooks/` 目录下的代码 import `zustand` 的 React 绑定来消费 store
- Editor 层、HSM 工具、DrawingEngine 通过 `store.getState()` / `store.setState()` 直接读写，不需要 Context 桥接

```
当前：
  LayerManager (纯 JS) → LayerContext (React 桥接) → MatelinkCanvas useEffect 同步 → ReactFlow nodes
  ToolManager (纯 JS) → ActiveActionContext (React 桥接) → MatelinkCanvas 读取
  DrawingEngine (signals) → DrawingContext (React 桥接) → 组件消费

Zustand 之后：
  CanvasStore (zustand/vanilla) ← Editor / Tools / DrawingEngine 直接 getState/setState
                                → 视图层通过 useCanvasStore(selector) 精确订阅
                                → ReactFlow nodes 是 store 的派生值，不是独立状态
```

**Zustand 消除的东西**：
- 17 个 Context 中的大部分（数据类 + 工具类）→ 合并为 store 的 slices
- 20 层 Provider 嵌套 → Zustand 不需要 Provider
- MatelinkCanvas 里的 useEffect 同步链 → store 是唯一真相源，不需要同步
- `getState()` 让非 React 代码（HSM、DrawingEngine）直接访问状态，不需要 Context 桥接

### 3.3 三层架构全景图

```
┌─────────────────────────────────────────────────────────────┐
│                        视图层 (View)                         │
│                                                              │
│  只做：渲染 + 事件捕获                                       │
│  内部分为两个渲染通道，对 Store/Editor 层完全不可见           │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │          共享视口 (CameraController / signals)        │    │
│  │  zoom, panX, panY — 存在 @preact/signals 里         │    │
│  │  两个渲染通道读同一份 viewport signal                │    │
│  └──────────────┬──────────────────┬───────────────────┘    │
│                 │                  │                          │
│  ┌──────────────▼────────┐  ┌─────▼──────────────────────┐  │
│  │    DOM 渲染通道        │  │    Canvas 2D 渲染通道      │  │
│  │    (React 组件)        │  │    (RAF 渲染循环)          │  │
│  │                        │  │                            │  │
│  │  WorkbenchNode (图片)  │  │  所有笔触 (stroke)        │  │
│  │  TextNode (Tiptap)     │  │  所有画刷 (paintbrush)    │  │
│  │  变换手柄              │  │  绘制中预览               │  │
│  │  UI 面板/工具栏        │  │  选择框/涂鸦轨迹          │  │
│  │                        │  │  背景网格                  │  │
│  └────────────────────────┘  └────────────────────────────┘  │
│                                                              │
│  两个通道通过 z-index 叠加，视觉上是一个画布                │
│  变化原因：视觉需求（"换个颜色"、"加个动画"）                │
│  禁止：包含任何业务判断逻辑                                  │
├──────────────────────────────────────────────────────────────┤
│                        Editor 层                             │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ 交互模块          │  │ 领域模块          │                 │
│  │                   │  │                   │                 │
│  │ 工具状态机(HSM)   │  │ 形状行为(ShapeUtil)│                │
│  │ 手势识别          │  │ 业务规则          │                 │
│  │ 快捷键映射        │  │ 选择/分组/层级    │                 │
│  │                   │  │                   │                 │
│  │ 变化原因：        │  │ 变化原因：        │                 │
│  │ 操作方式变了      │  │ 业务规则变了      │                 │
│  └──────────────────┘  └──────────────────┘                 │
│                                                              │
│  对外是一层，内部按变化原因分模块                             │
│  统一入口：CanvasEditor（Facade 模式）                       │
├──────────────────────────────────────────────────────────────┤
│                   Store 层 (Zustand vanilla)                 │
│                                                              │
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │ 数据模型          │  │ 基础设施          │                 │
│  │                   │  │                   │                 │
│  │ Shape Records    │  │ 空间索引(R-tree)  │                 │
│  │ Page Records     │  │ 持久化            │                 │
│  │ 类型定义          │  │ 资源管理          │                 │
│  │                   │  │                   │                 │
│  │ 变化原因：        │  │ 变化原因：        │                 │
│  │ 数据结构变了      │  │ 技术方案变了      │                 │
│  └──────────────────┘  └──────────────────┘                 │
│                                                              │
│  唯一真相源：Zustand vanilla store                           │
│  响应式系统由 Zustand 提供，不需要自研                       │
│  React 组件通过 selector 精确订阅，非 React 代码用 getState  │
└──────────────────────────────────────────────────────────────┘
```


### 3.4 视图层双渲染通道架构

Canvas 2D 不是架构的一层，是视图层内部的渲染实现细节。Store 层和 Editor 层完全不知道它的存在。

#### 3.4.1 为什么需要双渲染通道

| 元素类型 | 需要 DOM 交互？ | 数量级 | 适合的渲染方式 |
|----------|----------------|--------|---------------|
| 图片节点 (WorkbenchNode) | 是（拖拽手柄、标记、右键菜单） | 几十个 | DOM |
| 文本节点 (TextNode) | 是（Tiptap 富文本编辑、IME 输入法） | 几十个 | DOM |
| 笔触 (Stroke) | 否（纯图形） | 几百到几千条 | Canvas 2D |
| 画刷 (Paintbrush) | 否（纯图形，需合成控制） | 几百条 | Canvas 2D |
| 绘制中预览 | 否（60fps 实时更新） | 1 条 | Canvas 2D |
| 选择框/涂鸦轨迹 | 否（临时图形） | 1-2 个 | Canvas 2D |
| 背景网格 | 否（纯装饰） | 1 个 | Canvas 2D |
| 变换手柄 | 是（拖拽交互） | 1 组 | DOM |
| UI 面板/工具栏 | 是（按钮、输入框） | 固定 | DOM |

规则：需要 DOM 事件/富文本/IME 的走 DOM 通道，纯图形的走 Canvas 2D 通道。

#### 3.4.2 渲染通道与 ShapeUtil 的关系

渲染通道的选择是形状的固有属性，由 ShapeUtil 声明，不是视图层的判断逻辑：

```typescript
interface IShapeUtil<T extends ShapeData = ShapeData> {
  // ... 其他方法 ...
  
  /** 声明这个形状走哪个渲染通道 */
  getRenderTarget(): 'canvas2d' | 'dom';
  
  /** Canvas 2D 通道调用（getRenderTarget() === 'canvas2d' 时必须实现） */
  drawToCanvas?(shape: T, ctx: CanvasRenderingContext2D): void;
  
  /** DOM 通道调用（getRenderTarget() === 'dom' 时必须实现） */
  render?(shape: T): React.ReactNode;
}

// 笔触：走 Canvas 2D
class StrokeShapeUtil implements IShapeUtil<StrokeRecord> {
  getRenderTarget() { return 'canvas2d' as const; }
  drawToCanvas(shape: StrokeRecord, ctx: CanvasRenderingContext2D) {
    ctx.beginPath();
    // 从 shape.points 绘制路径（复用现有的 SVG path 逻辑，翻译为 Canvas API）
    ctx.fill();
  }
}

// 文本：走 DOM
class TextShapeUtil implements IShapeUtil<TextRecord> {
  getRenderTarget() { return 'dom' as const; }
  render(shape: TextRecord) {
    return <TiptapEditor content={shape.content} />;
  }
}
```

新增形状类型时，只需在 ShapeUtil 里声明 `getRenderTarget()`，视图层的渲染循环不需要改。

#### 3.4.3 Canvas 2D 渲染器的数据消费方式

Canvas 2D 渲染器是纯 JS 类（不是 React 组件），通过两个数据源驱动：

```
低频数据（已完成的笔触）← Zustand store（store.getState() + subscribe）
高频数据（正在画的笔触 + viewport）← @preact/signals（直接读 .value）

两者在同一个 RAF 渲染循环里绘制，统一了当前分散在三个 overlay 里的渲染
```

```typescript
// view/components/canvas/CanvasRenderer.ts — 纯 JS，不依赖 React
class CanvasRenderer {
  private dirty = true;
  private rafId: number | null = null;
  
  constructor(
    private ctx: CanvasRenderingContext2D,
    private store: CanvasStore,
    private shapeUtils: Map<string, IShapeUtil>,
    private drawingEngine: DrawingEngine,  // signals 高频数据源
    private cameraController: CameraController,  // viewport signals（高频）
  ) {
    // 订阅 store 变化，标记需要重绘
    this.store.subscribe(
      (state) => state.shapes,
      () => { this.dirty = true; }
    );
  }
  
  start() { this.loop(); }
  stop() { if (this.rafId) cancelAnimationFrame(this.rafId); }
  
  private loop = () => {
    if (this.dirty) {
      this.render();
      this.dirty = false;
    }
    this.rafId = requestAnimationFrame(this.loop);
  };
  
  private render() {
    const { shapes } = this.store.getState();
    const viewport = this.cameraController.viewport.value;  // 从 signals 读取（高频）
    const ctx = this.ctx;
    
    ctx.clearRect(0, 0, ctx.canvas.width, ctx.canvas.height);
    ctx.save();
    ctx.translate(viewport.panX, viewport.panY);
    ctx.scale(viewport.zoom, viewport.zoom);
    
    // 1. 绘制已完成的形状（从 Zustand store）
    for (const shape of shapes.values()) {
      const util = this.shapeUtils.get(shape.type);
      if (util?.getRenderTarget() === 'canvas2d') {
        util.drawToCanvas!(shape, ctx);
      }
    }
    
    // 2. 绘制正在画的笔触（从 signals — 高频路径）
    const currentStroke = this.drawingEngine.currentStroke.value;
    if (currentStroke) {
      this.drawDraftStroke(currentStroke, ctx);
    }
    
    ctx.restore();
  }
}
```

#### 3.4.4 CanvasShell — 两个渲染通道的容器

```typescript
// view/components/canvas/CanvasShell.tsx
function CanvasShell() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  
  // Canvas 2D 渲染循环（纯 JS，不参与 React 渲染周期）
  useEffect(() => {
    const ctx = canvasRef.current?.getContext('2d');
    if (!ctx) return;
    const renderer = new CanvasRenderer(ctx, canvasStore, shapeUtils, drawingEngine, cameraController);
    renderer.start();
    return () => renderer.stop();
  }, []);
  
  return (
    <div className="canvas-shell" style={{ position: 'relative' }}>
      {/* Canvas 2D 通道 — 底层（笔触、画刷、网格、选择框） */}
      <canvas ref={canvasRef} style={{ position: 'absolute', inset: 0, zIndex: 0 }} />
      
      {/* DOM 通道 — 上层（图片、文本、变换手柄、UI） */}
      <DOMNodeLayer style={{ position: 'absolute', inset: 0, zIndex: 1 }} />
    </div>
  );
}
```

#### 3.4.5 当前 overlay 到 Canvas 2D 渲染器的迁移

当前分散在三个独立 overlay 组件里的渲染，统一到一个 Canvas 2D 渲染循环：

```
当前状态                              目标状态
─────────                            ─────────

CurrentStrokeOverlay (SVG)     ──→   CanvasRenderer.drawDraftStroke()
PaintbrushCanvasOverlay (Canvas 2D) → CanvasRenderer.drawDraftStroke()
PaintbrushOverlay (SVG)        ──→   CanvasRenderer.render() 内的 paintbrush 循环
StrokeNode (ReactFlow 节点)    ──→   CanvasRenderer.render() 内的 stroke 循环

三个独立组件 + 一个 ReactFlow 节点 → 一个统一的 RAF 渲染循环
```

这消除了：
- 三个 overlay 各自管理自己的视口同步（现在共享 CameraController 的 viewport signal）
- StrokeNode 作为 ReactFlow 节点的 DOM 开销（现在是 Canvas draw call）
- PaintbrushOverlay 用 SVG group opacity 解决透明度堆叠的 hack（Canvas 2D 原生支持 globalCompositeOperation）

### 3.5 依赖方向

```
视图层 ──→ Editor 层 ──→ Store 层

箭头方向 = "知道"的方向 = import 的方向

✅ 视图层可以 import Editor 层
✅ Editor 层可以 import Store 层
✅ 视图层可以订阅 Store 层的事件（但不能直接写入）
✅ Canvas 2D 渲染器通过 store.getState() 读取数据（和 Editor 层同级的访问方式）
✅ Canvas 2D 渲染器通过 signals .value 读取高频数据（绘图状态 + viewport）

❌ Store 层不能 import Editor 层
❌ Editor 层不能 import 视图层
❌ Store 层不能 import 视图层
❌ Store 层不知道 Canvas 2D 的存在
❌ Editor 层不知道 Canvas 2D 的存在（ShapeUtil.getRenderTarget() 是声明式的，不是命令式的）
```

### 3.5 数据流向

```
用户点击画布
      │
      ▼
┌─────────────────────────────────────────────────────────────┐
│ 视图层                                                       │
│   onClick(event) {                                           │
│     const point = screenToCanvas(event.clientX, event.clientY);
│     editor.currentTool.onPointerDown({ point });             │
│   }                                                          │
│   只做：坐标转换 + 事件转发                                  │
└─────────────────────────────────────────────────────────────┘
      │ 传递：{ type: 'click', point: {x: 100, y: 200} }
      ▼
┌─────────────────────────────────────────────────────────────┐
│ Editor 层                                                    │
│   handleClick(point) {                                       │
│     if (this.currentTool === 'select') {                     │
│       const hits = store.getState().queryByRect(pointToRect);│
│       store.setState(s => ({ selectedIds: hits }));           │
│     }                                                        │
│   }                                                          │
│   做：意图判断 + 业务规则 + 通过 store.setState 修改数据     │
└─────────────────────────────────────────────────────────────┘
      │ Zustand 自动通知所有订阅了 selectedIds 的组件
      ▼
┌─────────────────────────────────────────────────────────────┐
│ 视图层（通过 selector 订阅 store）                           │
│   const selectedIds = useCanvasStore(s => s.selectedIds);    │
│   // Zustand 只在 selectedIds 引用变化时触发重渲染           │
│   // 其他 store 字段变化不会影响这个组件                     │
└─────────────────────────────────────────────────────────────┘
```

**与 React Context 的关键区别**：Context 的 value 变化会触发所有消费者重渲染。Zustand 的 selector 只在选中的 slice 变化时触发，这就是为什么它能替代 20 层 Provider 而不损失性能。

### 3.6 每一层的代码示例

#### Store 层（Zustand vanilla — 不依赖 React）

```typescript
import { createStore } from 'zustand/vanilla';
import { subscribeWithSelector } from 'zustand/middleware';

// CanvasStore 的状态类型
interface CanvasStoreState {
  // 数据
  shapes: Map<string, ShapeRecord>;
  layers: Map<string, LayerRecord>;
  selectedIds: string[];
  
  // 操作（actions）— 所有修改都通过这些方法
  putShape: (shape: ShapeRecord) => void;
  removeShapes: (ids: string[]) => void;
  updateShape: (id: string, changes: Partial<ShapeRecord>) => void;
  setSelection: (ids: string[]) => void;
  
  // 查询（queries）— 派生数据
  getShape: (id: string) => ShapeRecord | undefined;
  getShapesInRect: (rect: Rect) => ShapeRecord[];
  getSelectedShapes: () => ShapeRecord[];
}

// 用 zustand/vanilla 创建，零 React 依赖
export const canvasStore = createStore<CanvasStoreState>()(
  subscribeWithSelector((set, get) => ({
    shapes: new Map(),
    layers: new Map(),
    selectedIds: [],
    
    putShape: (shape) => set(state => {
      const next = new Map(state.shapes);
      next.set(shape.id, shape);
      return { shapes: next };
    }),
    
    removeShapes: (ids) => set(state => {
      const next = new Map(state.shapes);
      ids.forEach(id => next.delete(id));
      return { shapes: next, selectedIds: state.selectedIds.filter(id => !ids.includes(id)) };
    }),
    
    updateShape: (id, changes) => set(state => {
      const existing = state.shapes.get(id);
      if (!existing) return state;
      const next = new Map(state.shapes);
      next.set(id, { ...existing, ...changes });
      return { shapes: next };
    }),
    
    setSelection: (ids) => set({ selectedIds: ids }),
    
    // 查询方法不修改状态，直接从 get() 读取
    getShape: (id) => get().shapes.get(id),
    getShapesInRect: (rect) => {
      // 实际实现会委托给空间索引
      return Array.from(get().shapes.values()).filter(s => intersects(s.bounds, rect));
    },
    getSelectedShapes: () => {
      const { shapes, selectedIds } = get();
      return selectedIds.map(id => shapes.get(id)!).filter(Boolean);
    },
  }))
);

// 类型导出，供其他层使用
export type CanvasStore = typeof canvasStore;
```

**关键设计决策**：
- `zustand/vanilla` 的 `createStore` 不依赖 React，可以在 HSM、DrawingEngine 等纯 JS 代码中使用
- `subscribeWithSelector` 中间件让外部代码可以精确订阅某个 slice 的变化
- actions 和 queries 都定义在 store 内部，保证数据操作的内聚性
- `set()` 内的多次字段修改会合并为一次通知（Zustand 内置行为）

#### Editor 层（依赖 Store，不依赖 React）

```typescript
// 所有操作的统一入口
// 通过 store.getState() 和 store.setState() 与 Store 层交互

class CanvasEditor {
  constructor(
    private store: CanvasStore,
    private spatialIndex: ISpatialIndex,
  ) {}
  
  private currentTool: StateNode;
  
  setCurrentTool(toolId: string) { /* ... */ }
  
  createShape(shape: ShapeData) {
    const record = toShapeRecord(shape);
    this.store.getState().putShape(record);
    this.spatialIndex.insert(record.id, record.bounds);
  }
  
  deleteShapes(ids: string[]) {
    this.store.getState().removeShapes(ids);
    ids.forEach(id => this.spatialIndex.remove(id));
  }
  
  moveShapes(ids: string[], delta: Vector) {
    const state = this.store.getState();
    ids.forEach(id => {
      const shape = state.getShape(id);
      if (!shape) return;
      const newPosition = { x: shape.position.x + delta.x, y: shape.position.y + delta.y };
      state.updateShape(id, { position: newPosition });
      this.spatialIndex.update(id, { ...shape.bounds, x: newPosition.x, y: newPosition.y });
    });
  }
  
  select(ids: string[]) {
    this.store.getState().setSelection(ids);
  }
  
  undo() { /* CommandManager 操作 */ }
  redo() { /* CommandManager 操作 */ }
}
```

**注意**：Editor 层通过 `store.getState()` 访问 store，这是同步的、不依赖 React 的。HSM 工具内部也用同样的方式：

```typescript
// src/tools/select/SelectTool.ts — HSM 工具直接访问 store
class SelectTool extends StateNode {
  onPointerDown(event: CanvasPointerEvent) {
    const hits = this.editor.store.getState().getShapesInRect(
      pointToRect(event.point, 4)
    );
    if (hits.length > 0) {
      this.editor.select(hits.map(h => h.id));
      this.transition('dragging');
    }
  }
}
```

#### 视图层（通过 Zustand React hook 订阅 Store）

```typescript
import { useStore } from 'zustand';
import { canvasStore } from '@/core/store/canvasStore';

// 创建类型安全的 React hook
export const useCanvasStore = <T>(selector: (state: CanvasStoreState) => T): T => {
  return useStore(canvasStore, selector);
};

// ---- 视图组件 ----

function CanvasView() {
  const editor = useEditor();
  // selector 精确订阅：只有 shapes 变化时才重渲染
  const shapes = useCanvasStore(s => Array.from(s.shapes.values()));
  
  return (
    <div onPointerDown={(e) => {
      const point = screenToCanvas(e.clientX, e.clientY);
      editor.currentTool.onPointerDown({ point });
    }}>
      {shapes.map(shape => (
        <ShapeRenderer key={shape.id} shape={shape} />
      ))}
    </div>
  );
}

// 精确订阅示例：这个组件只在 selectedIds 变化时重渲染
function SelectionIndicator() {
  const selectedIds = useCanvasStore(s => s.selectedIds);
  return <div>{selectedIds.length} selected</div>;
}

// 单个 shape 的精确订阅：只在这个 shape 变化时重渲染
function ShapeRenderer({ shapeId }: { shapeId: string }) {
  const shape = useCanvasStore(s => s.shapes.get(shapeId));
  if (!shape) return null;
  
  const util = useShapeUtil(shape.type);
  return util.render(shape);
}
```

**与当前架构的对比**：
- 当前：`LayerContext` 变化 → 所有消费 `useLayerContext()` 的组件重渲染（包括不相关的）
- Zustand：`shapes` 变化 → 只有 selector 返回值变化的组件重渲染
- 当前：HSM 工具需要通过 Context 桥接读取数据 → 需要 Provider 包裹
- Zustand：HSM 工具直接 `store.getState()` → 不需要任何 Provider


---

## 第四部分：目标目录结构

```
src/
├── core/                          # Store 层 + Editor 层（不依赖 React）
│   ├── store/                     # Store 层（Zustand vanilla）
│   │   ├── canvasStore.ts         # Zustand vanilla store — 唯一真相源
│   │   ├── slices/                # Store 按职责拆分的 slices
│   │   │   ├── shapeSlice.ts      # Shape CRUD actions + queries
│   │   │   ├── selectionSlice.ts  # 选择状态
│   │   │   ├── layerSlice.ts      # 图层状态
│   │   │   └── toolSlice.ts       # 工具状态（activeTool 等）
│   │   ├── records/               # 数据模型（纯类型定义）
│   │   │   ├── ShapeRecord.ts
│   │   │   ├── LayerRecord.ts
│   │   │   └── types.ts
│   │   └── middleware/            # Zustand 中间件
│   │       ├── undoMiddleware.ts  # 撤销/重做（替代 CommandManager）
│   │       └── persistMiddleware.ts # 持久化
│   │
│   ├── editor/                    # Editor 层
│   │   ├── CanvasEditor.ts        # Facade，所有操作的统一入口
│   │   ├── tools/                 # 交互模块：工具状态机
│   │   │   ├── SelectTool.ts
│   │   │   ├── PenTool.ts
│   │   │   ├── EraserTool.ts
│   │   │   └── PanTool.ts
│   │   ├── shapes/                # 领域模块：形状行为
│   │   │   ├── StrokeShapeUtil.ts
│   │   │   ├── TextShapeUtil.ts
│   │   │   └── WorkbenchShapeUtil.ts
│   │   └── commands/              # 命令定义（配合 undoMiddleware）
│   │
│   └── infra/                     # 基础设施模块
│       ├── spatial/               # 空间索引（唯一一套）
│       ├── lod/                   # LOD 系统（唯一一套）
│       ├── persistence/           # 持久化（配合 persistMiddleware）
│       └── resource/              # 资源管理
│
├── view/                          # 视图层（依赖 React）
│   ├── hooks/                     # 视图层专用 hooks
│   │   ├── useCanvasStore.ts      # Zustand React 绑定（selector hook）
│   │   ├── useEditor.ts           # Editor 实例访问
│   │   └── useShapeUtil.ts        # ShapeUtil 注册表访问
│   ├── components/                # React 组件
│   │   ├── nodes/                 # DOM 通道：节点渲染组件
│   │   │   ├── WorkbenchNodeView.tsx  # 图片节点（DOM）
│   │   │   └── TextNodeView.tsx       # 文本节点（DOM + Tiptap）
│   │   ├── canvas/                # 画布容器 + Canvas 2D 渲染器
│   │   │   ├── CanvasShell.tsx        # 容器：挂载 DOM 通道 + Canvas 2D 通道
│   │   │   ├── CanvasRenderer.ts      # Canvas 2D 渲染循环（纯 JS，RAF 驱动）
│   │   │   ├── DOMNodeLayer.tsx       # DOM 通道（纯 React，CSS transform 定位）
│   │   │   └── ViewportSync.ts        # 两个通道的视口同步（读 CameraController signals）
│   │   ├── overlays/              # 覆盖层组件（DOM 通道，需要交互的）
│   │   │   └── TransformHandles.tsx   # 变换手柄（需要拖拽交互，必须 DOM）
│   │   └── ui/                    # UI 组件（工具栏、面板）
│   └── providers/                 # 仅保留真正需要 Provider 的（≤5 个）
│       ├── EditorProvider.tsx      # 提供 CanvasEditor 实例
│       └── ThemeProvider.tsx       # 主题（CSS 变量注入，必须用 Provider）
```

---

## 第五部分：当前代码到目标架构的映射

```
当前状态 ──────────────────────→ 目标状态

MatelinkCanvas (2730 行)
  ├── 渲染逻辑 ──────────────→ view/components/canvas/
  ├── 选择逻辑 ──────────────→ core/editor/tools/SelectTool.ts
  ├── 拖拽逻辑 ──────────────→ core/editor/CanvasEditor.ts
  ├── 右键菜单 ──────────────→ view/components/ui/
  ├── 碰撞检测 ──────────────→ core/editor/shapes/*ShapeUtil.ts
  ├── LOD 集成 ──────────────→ core/infra/lod/
  └── 剪贴板操作 ────────────→ core/editor/commands/

LayerManager (45+ 方法)
  ├── 数据 CRUD ─────────────→ core/store/slices/shapeSlice.ts (Zustand actions)
  ├── 空间索引同步 ──────────→ core/infra/spatial/（由 Editor 层协调）
  ├── 序列化 ────────────────→ core/store/middleware/persistMiddleware.ts
  └── 类型特有操作 ──────────→ core/editor/shapes/*ShapeUtil.ts

ReactFlow nodes state ───────→ 删除（Zustand store 是唯一真相源，
                                ReactFlow 完全移除，DOM 节点用 CSS transform 定位）

渲染层迁移（三个 overlay + ReactFlow 节点 → 统一 Canvas 2D 渲染器）
  ├── StrokeNode (ReactFlow 节点) ──→ CanvasRenderer + StrokeShapeUtil.drawToCanvas()
  │   消除：每笔一个 DOM 节点、LRU SVG path 缓存的复杂性
  │
  ├── PaintbrushOverlay (SVG) ──────→ CanvasRenderer + PaintbrushShapeUtil.drawToCanvas()
  │   消除：SVG group opacity hack、独立的视口同步
  │
  ├── CurrentStrokeOverlay (SVG) ──→ CanvasRenderer.drawDraftStroke()（消费 signals）
  │   消除：SVG overlay 的 React 渲染开销
  │
  ├── PaintbrushCanvasOverlay ─────→ CanvasRenderer.drawDraftStroke()（消费 signals）
  │   消除：独立的 Canvas 元素和 RAF 循环
  │
  ├── WorkbenchNode ────────────────→ 保留在 DOM 通道（DOMNodeLayer）
  │   原因：需要标记拖拽、右键菜单等 DOM 交互
  │
  └── TextNode ─────────────────────→ 保留在 DOM 通道（DOMNodeLayer）
      原因：Tiptap 富文本编辑器必须在 DOM 中

17 个 Context
  ├── LayerContext ───────────→ 删除（被 Zustand store 的 shape/layer slices 替代）
  ├── ActiveActionContext ───→ 删除（被 Zustand store 的 tool slice 替代）
  ├── PenContext ────────────→ 删除（被 Zustand store 的 pen slice 替代）
  ├── EraserContext ─────────→ 删除（被 Zustand store 的 eraser slice 替代）
  ├── DrawingContext ────────→ 保留（signals 高频路径，不适合 Zustand）
  ├── TextEditingContext ────→ 删除（被 Zustand store 的 editing slice 替代）
  ├── ThemeContext ──────────→ 保留（CSS 变量注入需要 Provider）
  ├── MinimapContext ────────→ 删除（被 CameraController signals + useCameraViewport() hook 替代）
  └── 其他 UI Context ──────→ 逐个评估，大部分可合并到 store
```


---

## 第六部分：重构策略 — Big Bang

### 6.1 为什么选 Big Bang 而不是绞杀者模式

| 维度 | 绞杀者模式 | Big Bang | 本项目的情况 |
|------|-----------|----------|-------------|
| 核心价值 | 系统始终可用 | 一次性切换到新架构 | 零用户，"始终可用"是自我施加的约束 |
| 认知负担 | 新旧共存 + 适配器层 | 只有新系统 | 1-2 人 + AI agent，维护两套系统更累 |
| 回滚风险 | 低（旧系统还在） | 高（旧系统被替代） | 零用户 = 零回滚风险 |
| 适合团队 | 大团队、生产系统 | 小团队、零用户项目 | ✅ 小团队 + 零用户 |
| Agent 友好度 | 低（模糊边界判断多） | 高（明确的架构规则） | ✅ Agent 擅长按规则生成代码 |

**结论**：绞杀者模式的核心价值（系统始终可用）在零用户项目中没有意义。适配器本身就是新的复杂度来源。Big Bang 给 agent 的指令更清晰——"在新骨架上重新组装已有的肌肉"。

### 6.2 Big Bang 的前置条件

Big Bang 需要一个绞杀者模式不需要的东西：**一份精确的功能行为清单**。

在动手之前，必须把旧系统的每一个用户可见行为列出来，作为新系统的验收标准。否则 Big Bang 会变成"重写了 80% 然后发现剩下 20% 的边界情况要花 80% 的时间"。

功能行为清单的格式：

```
功能域: 画笔
  ├── 基础绘制：按下 → 移动 → 抬起，生成笔触
  ├── 压感支持：笔触宽度随压力变化
  ├── 自动分段：超过 N 个点自动 split
  ├── LOD 裁剪：视口外笔触不渲染
  ├── ...
  
功能域: 选择
  ├── 点选：点击元素选中
  ├── 框选：拖拽矩形框选
  ├── Shift 多选：按住 Shift 追加选择
  ├── ...
```

**这份清单由人工编制，不由 agent 生成。** Agent 不了解产品意图，只了解代码实现。

### 6.3 Big Bang 的执行阶段

```
阶段 0：功能行为清单（人工编制）
  列出旧系统所有用户可见行为，作为验收标准
  ↓
阶段 1：新骨架搭建
  建立 core/store/（Zustand vanilla canvasStore）
  建立 core/editor/（CanvasEditor facade）
  建立 view/ 目录结构（CanvasShell + 双渲染通道）
  建立 core/infra/（空间索引、LOD — 各一套）
  此时新系统可以启动但功能为空
  ↓
阶段 2：功能重建（按功能域逐个迁移）
  每个功能域：
    1. 从旧代码中提取业务逻辑
    2. 在新架构中重新实现
    3. 对照功能行为清单验证
  迁移顺序按依赖关系排列（见 6.4）
  ↓
阶段 3：旧代码删除
  所有功能验证通过后，删除旧的 MatelinkCanvas、LayerManager、
  17 个 Context、3 个 overlay、ReactFlow 节点类型等
```

### 6.4 功能重建顺序

按依赖关系从底层到上层：

```
第 1 步：Store 层骨架
  canvasStore（zustand/vanilla）+ shapeSlice + selectionSlice + layerSlice + toolSlice
  ↓
第 2 步：基础设施
  空间索引（唯一一套）+ LOD 系统（唯一一套）
  ↓
第 3 步：Editor 层骨架
  CanvasEditor + 工具注册机制 + ShapeUtil 注册机制
  ↓
第 4 步：视图层骨架
  CanvasShell + CanvasRenderer（Canvas 2D RAF 循环）+ DOMNodeLayer
  CameraController（@preact/signals viewport）
  useCanvasStore hook + useEditor hook
  ↓
第 5 步：核心工具重建（按功能域）
  SelectTool（点选、框选、多选）→ 验证
  PenTool（绘制、压感、auto-split）→ 验证
  EraserTool（擦除、预览）→ 验证
  PanTool（平移、缩放）→ 验证
  ↓
第 6 步：形状渲染重建
  StrokeShapeUtil（Canvas 2D drawToCanvas）→ 验证
  PaintbrushShapeUtil（Canvas 2D drawToCanvas）→ 验证
  TextShapeUtil（DOM render + Tiptap）→ 验证
  WorkbenchShapeUtil（DOM render）→ 验证
  ↓
第 7 步：辅助功能重建
  快捷键系统、右键菜单、剪贴板、撤销/重做
  ↓
第 8 步：清理
  删除旧代码、验证功能行为清单 100% 通过
```

### 6.5 ReactFlow 的去留

重构完成后，ReactFlow 的价值只剩 viewport 管理和 DOM 节点拖拽（笔触和画刷已迁移到 Canvas 2D）。这两个功能自己实现的成本远低于维护 ReactFlow 的集成复杂度。

**决策：Big Bang 重构中去掉 ReactFlow。**

替代方案：
- viewport 管理 → CameraController（@preact/signals，已在架构中）
- DOM 节点定位 → 自己用 CSS transform 定位（读 CameraController 的 viewport signal）
- DOM 节点拖拽 → Editor 层的 DragTool / SelectTool 处理

这消除了：
- ReactFlow 的 nodes state 作为第二数据源的问题
- ReactFlow 内部的 reconciliation 开销
- ReactFlow API 与自定义交互逻辑的冲突


---

## 第七部分：物理定律 — 机器强制的硬约束

### 7.1 边界法则

用 ESLint `import/no-restricted-paths` 强制层间依赖方向：

```javascript
// .eslintrc.js
rules: {
  'import/no-restricted-paths': ['error', {
    zones: [
      { target: './src/core/store', from: './src/core/editor', message: 'Store 不能依赖 Editor' },
      { target: './src/core/store', from: './src/view', message: 'Store 不能依赖视图层' },
      { target: './src/core/editor', from: './src/view', message: 'Editor 不能依赖视图层' },
      // Zustand 特有：core 层只能用 zustand/vanilla，不能用 zustand 的 React 绑定
      { target: './src/core', from: 'react', message: 'Core 层不能依赖 React' },
      { target: './src/core', from: 'zustand', message: 'Core 层只能用 zustand/vanilla，不能用 zustand 的 React 绑定' },
    ]
  }],
  'no-restricted-imports': ['error', {
    paths: [
      // 禁止在 core 层 import zustand 的 React hook
      { name: 'zustand', message: 'Core 层请使用 zustand/vanilla 的 createStore' },
    ]
  }]
}
```

**Zustand 的两个入口**：
- `zustand/vanilla` → `createStore()` → 纯 JS，用在 core 层
- `zustand` → `useStore()` → React hook，只用在 view 层

### 7.2 复杂度预算

```javascript
rules: {
  'max-lines': ['error', { max: 300, skipBlankLines: true, skipComments: true }],
  'max-lines-per-function': ['error', { max: 40, skipBlankLines: true, skipComments: true }],
}
```

| 维度 | 预算 | 当前违反 |
|------|------|----------|
| 单文件行数 | ≤ 300 | MatelinkCanvas: 2,730 |
| 单类公开方法数 | ≤ 10 | LayerManager: 45+ |
| 单函数行数 | ≤ 40 | — |
| 单组件直接依赖的 Context 数 | ≤ 3 | MatelinkCanvas: 8+ |
| Provider 嵌套层数 | ≤ 5 | 当前: 20（Zustand 消除大部分 Provider） |
| Zustand store slices 数 | ≤ 8 | —（新建） |

### 7.3 统一性法则

```javascript
rules: {
  'no-restricted-imports': ['error', {
    patterns: [
      { group: ['**/drawing/lod/**'], message: '请使用 core/infra/lod/，不要使用 drawing 内部的 LOD' },
      { group: ['**/pen/lod/**'], message: '请使用 core/infra/lod/' },
      { group: ['**/drawing/spatial/**'], message: '请使用 core/infra/spatial/' },
    ]
  }]
}
```

| 问题域 | 唯一解法 | 当前违反 |
|--------|----------|----------|
| 状态管理（低频） | Zustand store (slices) | React Context × 17 + useState 散落 |
| 状态管理（高频） | @preact/signals | 仅 DrawingEngine 使用，其他高频场景未统一 |
| 工具行为建模 | HSM (StateNode) | 3 种模式并存 |
| 空间查询 | core/infra/spatial/ | 3 处实现 |
| LOD 系统 | core/infra/lod/ | 3 套并存 |
| 性能监控 | core/infra/lod/PerformanceMonitor | 2 套并存 |
| 纯图形渲染 | Canvas 2D 渲染器 (CanvasRenderer) | SVG overlay × 3 + ReactFlow 节点混用 |
| DOM 交互渲染 | DOM 通道 (DOMNodeLayer) | 全部混在 ReactFlow 节点里 |

### 7.4 依赖法则

用 `dependency-cruiser` 检查循环依赖：

```javascript
// .dependency-cruiser.js
module.exports = {
  forbidden: [
    { name: 'no-circular', severity: 'error', from: {}, to: { circular: true } },
    { name: 'core-no-react', severity: 'error', from: { path: '^src/core' }, to: { path: 'react' } },
  ]
}
```

### 7.5 实施节奏

```
第一步（1 周）：配置所有规则为 warn，看有多少违反
第二步（持续）：每次重构一个模块，把该模块的规则从 warn 改为 error
第三步（目标）：所有规则都是 error，CI 不通过就不能合并
```


---

## 第八部分：层间接口合同（类型定义）

### 8.1 视图层 → Editor 层

```typescript
// 视图层传给 Editor 的原始事件（不变）
interface CanvasPointerEvent {
  point: Point;
  button: 'left' | 'right' | 'middle';
  modifiers: { shift: boolean; ctrl: boolean; alt: boolean };
}
```

### 8.2 Store 层状态类型（Zustand store 的完整类型）

```typescript
// 所有 slices 合并后的 store 类型
interface CanvasStoreState {
  // === Shape Slice ===
  shapes: Map<string, ShapeRecord>;
  putShape: (shape: ShapeRecord) => void;
  removeShapes: (ids: string[]) => void;
  updateShape: (id: string, changes: Partial<ShapeRecord>) => void;
  getShape: (id: string) => ShapeRecord | undefined;
  
  // === Layer Slice ===
  layers: Map<string, LayerRecord>;
  activeLayerId: string;
  createLayer: (layer: LayerRecord) => void;
  setActiveLayer: (id: string) => void;
  
  // === Selection Slice ===
  selectedIds: string[];
  setSelection: (ids: string[]) => void;
  addToSelection: (ids: string[]) => void;
  clearSelection: () => void;
  
  // === Tool Slice ===
  activeTool: ToolId;
  previousTool: ToolId | null;
  isSpaceHeld: boolean;
  setActiveTool: (tool: ToolId) => void;
  
  // === Pen Slice（低频配置，不包含高频绘图状态）===
  penColor: string;
  penScale: number;
  penOpacity: number;
  penType: PenType;
  setPenConfig: (config: Partial<PenConfig>) => void;
  
  // === Eraser Slice ===
  eraserSize: number;
  setEraserSize: (size: number) => void;
}
```

**注意**：高频状态（draftPoints、currentStroke、viewport）使用 @preact/signals，不放入 Zustand store。Zustand 的 `set()` 虽然高效，但在 60fps 的画笔路径和 pan/zoom 路径上仍然不如 signals 直接驱动渲染。viewport 由 CameraController 管理，低频消费者通过 `useCameraViewport()` hook 桥接到 React。

### 8.3 Editor 层 → Store 层的交互方式

```typescript
// Editor 不通过 Command 对象与 Store 交互，而是直接调用 store actions
// 这比自定义事件系统更简单、类型更安全

class CanvasEditor {
  constructor(private store: CanvasStore) {}
  
  createShape(data: ShapeData) {
    const record = toShapeRecord(data);
    this.store.getState().putShape(record);     // 直接调用 action
    // 空间索引更新也在这里，不在 store 内部
    this.spatialIndex.insert(record.id, record.bounds);
  }
}

// 对比之前的 Command 模式：
// 之前：editor → dispatch({ type: 'CREATE_SHAPE', shape }) → store 解析 command
// 现在：editor → store.getState().putShape(shape)
// 更直接，类型安全，不需要 switch/case
```

### 8.4 Store 层 → 视图层的订阅方式

```typescript
// 视图层通过 Zustand selector 订阅，不需要自定义事件系统

// 精确订阅单个字段
const activeTool = useCanvasStore(s => s.activeTool);

// 精确订阅派生值（用 shallow 比较避免不必要的重渲染）
import { shallow } from 'zustand/shallow';
const { selectedIds, shapes } = useCanvasStore(
  s => ({ selectedIds: s.selectedIds, shapes: s.shapes }),
  shallow
);

// 非 React 代码订阅（HSM、DrawingEngine 等）
const unsubscribe = canvasStore.subscribe(
  (state) => state.activeTool,
  (activeTool, previousTool) => {
    console.log(`Tool changed: ${previousTool} → ${activeTool}`);
  }
);
```

### 8.5 形状行为接口（含渲染通道声明）

```typescript
// 每种形状必须实现的统一接口
interface IShapeUtil<T extends ShapeData = ShapeData> {
  getBounds(shape: T): Rect;
  hitTest(shape: T, point: Point): boolean;
  serialize(shape: T): SerializedShape;
  getLODBehavior(shape: T): LODBehavior;
  
  /** 声明这个形状走哪个渲染通道 */
  getRenderTarget(): 'canvas2d' | 'dom';
  
  /** 
   * Canvas 2D 通道的渲染方法
   * getRenderTarget() === 'canvas2d' 时必须实现
   */
  drawToCanvas?(shape: T, ctx: CanvasRenderingContext2D): void;
  
  /** 
   * DOM 通道的渲染方法
   * getRenderTarget() === 'dom' 时必须实现
   */
  render?(shape: T): React.ReactNode;
}
```

各形状的渲染通道分配：

| ShapeUtil | getRenderTarget() | 渲染方法 | 原因 |
|-----------|-------------------|----------|------|
| StrokeShapeUtil | `'canvas2d'` | drawToCanvas | 纯图形，数量大 |
| PaintbrushShapeUtil | `'canvas2d'` | drawToCanvas | 纯图形，需合成控制 |
| TextShapeUtil | `'dom'` | render | Tiptap 编辑器必须在 DOM |
| WorkbenchShapeUtil | `'dom'` | render | 标记拖拽、右键菜单需要 DOM |

### 8.6 基础设施接口（不变）

```typescript
// 空间索引的抽象接口（可替换实现）
interface ISpatialIndex {
  insert(id: string, bounds: Rect): void;
  remove(id: string): void;
  update(id: string, bounds: Rect): void;
  query(rect: Rect): string[];
}
```

---

## 第九部分：给下一个 Agent 的指引

### 9.1 你的任务

基于本文档和人工编制的功能行为清单，对现有代码库进行解耦剖析，形成可执行的 Spec 文件。Spec 应按 Big Bang 阶段组织（骨架搭建 → 功能重建 → 旧代码删除），每个 Spec 对应一个可独立验证的功能域。

### 9.2 关键约束

1. **三层分离是硬约束**：Store（Zustand vanilla，唯一真相源）、Editor（统一操作入口）、View（纯渲染 + selector 订阅）
2. **Zustand 的双入口规则**：`zustand/vanilla` 用在 core 层（不依赖 React），`zustand` 的 React hook 只用在 view 层
3. **高频路径例外**：画笔绘图的 draftPoints/currentStroke 以及 viewport（zoom, panX, panY）使用 @preact/signals，不放入 Zustand store。判断标准：如果一个状态在 pointerMove 中每帧更新，它属于 signals；否则属于 Zustand
4. **双渲染通道规则**：Canvas 2D 渲染器处理纯图形（笔触、画刷、选择框、网格），DOM 通道处理需要交互的节点（文本、图片、手柄）。渲染通道由 ShapeUtil.getRenderTarget() 声明，视图层不做 type 判断
5. **Canvas 2D 渲染器是纯 JS**：不依赖 React，通过 store.getState() 读取低频数据，通过 signals .value 读取高频数据，RAF 循环驱动
6. **Big Bang 是本次重构的执行策略**：在新骨架上重新组装已有功能，不做新旧共存
7. **功能行为清单是前置条件**：动手之前必须有人工编制的旧系统行为清单，作为验收标准
8. **ReactFlow 在重构中去掉**：viewport 由 CameraController 管理，DOM 节点用 CSS transform 定位
9. **物理定律由工具链强制**：ESLint、dependency-cruiser、CI 拦截

### 9.3 优先级

按 Big Bang 执行阶段排列（见第六部分 6.4）：

```
阶段 1 — 骨架搭建：
  建立 Zustand canvasStore（zustand/vanilla，唯一真相源）
  建立 CanvasEditor（统一操作入口）
  建立 core/infra/（空间索引 × 1、LOD × 1）
  建立视图层骨架（CanvasShell + CanvasRenderer + DOMNodeLayer + CameraController）

阶段 2 — 功能重建：
  核心工具（Select、Pen、Eraser、Pan）
  形状渲染（Stroke → Canvas 2D、Paintbrush → Canvas 2D、Text → DOM、Workbench → DOM）
  辅助功能（快捷键、右键菜单、剪贴板、撤销/重做）

阶段 3 — 清理：
  删除旧代码（MatelinkCanvas、LayerManager、ReactFlow、17 个 Context、3 个 overlay）
  验证功能行为清单 100% 通过
```

### 9.4 验证标准

重构完成后，以下条件必须满足：

- [ ] 数据真相源只有 1 个（Zustand canvasStore）
- [ ] 所有操作通过 CanvasEditor 进入
- [ ] MatelinkCanvas 行数 ≤ 300（瘦身为 CanvasShell + 事件转发）
- [ ] 工具实现模式只有 1 种（HSM）
- [ ] LOD 系统只有 1 套
- [ ] 空间索引只有 1 套
- [ ] Provider 嵌套 ≤ 5 层（只保留 EditorProvider、ThemeProvider 等必要的）
- [ ] 无循环依赖
- [ ] Core 层不依赖 React（只依赖 zustand/vanilla）
- [ ] 高频路径（绘图状态 + viewport）使用 @preact/signals（不被 Zustand 替代）
- [ ] 笔触和画刷通过 Canvas 2D 渲染器绘制（不再是 ReactFlow 节点或 SVG overlay）
- [ ] 文本和图片通过 DOM 通道渲染（保留 DOM 交互能力）
- [ ] 两个渲染通道共享同一个 viewport（来自 CameraController 的 signals）
- [ ] CanvasRenderer 是纯 JS 类，不依赖 React（通过 store.getState() + signals 驱动）
- [ ] 每个 ShapeUtil 声明 getRenderTarget()，视图层不包含 if (type === 'xxx') 路由逻辑
- [ ] ReactFlow 已完全移除，viewport 由 CameraController 管理

---

*本文档是三层架构重构的基石。Store 层基于 Zustand vanilla 实现，高频路径（绘图状态 + viewport）使用 @preact/signals。视图层采用双渲染通道架构：Canvas 2D 渲染器处理笔触/画刷等纯图形（纯 JS，RAF 驱动），DOM 通道处理文本/图片等需要交互的节点。两个通道共享 CameraController 的 viewport signal，对 Store 层和 Editor 层完全透明。重构采用 Big Bang 策略，在新骨架上重新组装旧系统的功能，以人工编制的功能行为清单作为验收标准。持久化、协作、AI 集成等远期能力不在本文档范围内，待重构完成后作为独立产品规划推进。*
