审阅总结
这是一份质量很高的架构重构文档。问题诊断准确，三层架构设计合理，对 tldraw 等成熟画布软件的借鉴到位。但有几个关键点需要讨论和调整。

一、做得好的地方
1. 问题诊断精准

"依赖关系失控"的根因分析非常到位。2730 行的 MatelinkCanvas、45+ 方法的 LayerManager、双数据源、20 层 Provider——这些都是典型的"架构边界缺失"症状。tldraw 早期也经历过类似的演进，最终通过 Store + Editor 的分离解决了这个问题。

2. 三层架构的选择正确

文档正确识别了"理论四层 vs 实践三层"的权衡。tldraw 的 @tldraw/store + @tldraw/editor + UI 层正是这个模式。交互和领域在画布软件中确实难以硬拆——SelectTool 需要知道形状在哪，形状需要知道自己被选中了。

3. 双渲染通道的设计合理

Canvas 2D 处理高频纯图形（笔触、画刷），DOM 处理需要交互的节点（文本、图片）——这个分工符合画布软件的最佳实践。tldraw 虽然用 SVG 渲染形状，但也有类似的分层思想（culling、LOD）。

4. Big Bang 策略的选择合理

零用户项目选 Big Bang 是对的。绞杀者模式的核心价值是"系统始终可用"，在没有用户的情况下这是自我施加的约束。

二、需要讨论的关键问题
问题 1：Zustand vs 自研响应式系统
文档选择 Zustand 作为 Store 层的响应式基座，理由是"不需要自研"。这个决策需要仔细权衡。

tldraw 为什么自研了 signia（@tldraw/state）？

需求	Zustand	signia
细粒度依赖追踪	selector 手动指定	自动追踪（读 .get() 时注册）
computed 缓存	需要手动 memoize	内置（Computed 自动缓存）
事务批量更新	set() 内合并	transact() 显式事务
React 外使用	zustand/vanilla	原生支持（不依赖 React）
增量更新通知	整个 slice 变化	精确到单个 atom 变化
关键差异：Zustand 的 selector 是"手动指定依赖"，signia 是"自动追踪依赖"。

// Zustand：手动指定依赖
const selectedIds = useCanvasStore(s => s.selectedIds);
const shapes = useCanvasStore(s => s.shapes);
// 如果忘了写 selector，整个 store 变化都会触发重渲染

// signia：自动追踪依赖
const selectedShapes = computed('selectedShapes', () => {
  return selectedIds.get().map(id => shapes.get().get(id));
});
// 只有 selectedIds 或相关 shapes 变化时才重算
我的建议：

对于 Matelink 的规模，Zustand 够用。但要注意：

selector 必须精确：文档里的 useCanvasStore(s => Array.from(s.shapes.values())) 每次都创建新数组，会导致不必要的重渲染。需要配合 shallow 或自定义比较函数。

computed 需要手动实现：Zustand 没有内置 computed，派生数据需要用 useMemo 或在 store 里定义 getter。

考虑 zustand-computed 中间件：如果派生数据多，可以用这个中间件。

问题 2：ShapeUtil 的 getRenderTarget() 设计
文档设计了 getRenderTarget(): 'canvas2d' | 'dom' 来声明渲染通道。这个设计有一个潜在问题：

tldraw 的 ShapeUtil 是如何处理渲染的？

// tldraw 的 ShapeUtil
class MyShapeUtil extends ShapeUtil<MyShape> {
  component(shape: MyShape) {
    // 返回 React 组件，由框架决定如何渲染
    return <div>...</div>;
  }
  
  indicator(shape: MyShape) {
    // 选中时的指示器，通常是 SVG
    return <rect ... />;
  }
}
tldraw 没有让 ShapeUtil 声明"走哪个渲染通道"，而是统一返回 React 组件，由框架层面决定渲染策略（SVG、culling、LOD）。

潜在问题：

如果 getRenderTarget() 是形状的固有属性，那么同一种形状在不同场景下无法切换渲染方式。比如：

缩略图/导出时，所有形状都应该走 Canvas 2D（性能）
编辑时，文本需要走 DOM（IME 输入）
建议调整：

interface IShapeUtil<T> {
  // 不声明 getRenderTarget()，而是同时提供两种渲染方法
  
  /** Canvas 2D 渲染（用于缩略图、导出、高性能场景） */
  drawToCanvas(shape: T, ctx: CanvasRenderingContext2D): void;
  
  /** DOM 渲染（用于需要交互的场景） */
  component(shape: T): React.ReactNode;
  
  /** 是否需要 DOM 交互（决定默认渲染通道） */
  needsDOMInteraction(shape: T): boolean;
}
这样视图层可以根据场景选择渲染方式，而不是由形状硬编码。

问题 3：高频路径的 signals vs Zustand 边界
文档说"如果一个状态在 pointerMove 中每帧更新，它属于 signals"。这个判断标准基本正确，但边界需要更清晰。

tldraw 的做法：

tldraw 的 @tldraw/state 统一处理所有状态，不区分高频/低频。它通过以下机制保证性能：

Atom 的细粒度更新：每个 atom 独立通知，不会触发无关组件
Computed 的惰性求值：只有被读取时才计算
React 绑定的批量更新：useValue() 内部用 useSyncExternalStore，自动批量
Matelink 的情况：

文档选择 Zustand（低频）+ @preact/signals（高频）的混合方案。这会带来：

优点：高频路径绕过 Zustand 的 selector 机制，直接驱动渲染
缺点：两套响应式系统，需要在边界处做桥接
建议：

如果坚持混合方案，需要明确定义边界：

状态	归属	理由
shapes（已完成）	Zustand	低频，需要持久化
selectedIds	Zustand	低频，需要 undo/redo
activeTool	Zustand	低频
draftPoints（绘制中）	signals	高频，不需要持久化
currentStroke（绘制中）	signals	高频，不需要持久化
viewport (zoom, pan)	signals	高频，但需要持久化？
注意 viewport：viewport 是高频更新的，但通常也需要持久化（用户下次打开时恢复视口位置）。这里需要一个同步机制：signals 驱动渲染，定期同步到 Zustand 做持久化。

问题 4：CanvasRenderer 的 dirty flag 机制
文档的 CanvasRenderer 用 dirty flag 控制重绘：

private loop = () => {
  if (this.dirty) {
    this.render();
    this.dirty = false;
  }
  this.rafId = requestAnimationFrame(this.loop);
};
问题：这个实现在高频绘图时会有问题。

当用户快速绘制时，draftPoints 每帧都在变化（来自 signals），但 dirty 只在 store 变化时设为 true。这意味着绘制中的笔触不会触发重绘。

建议修改：

private loop = () => {
  // 高频数据直接检查 signals 的值
  const currentStroke = this.drawingEngine.currentStroke.value;
  const viewport = this.cameraController.viewport.value;
  
  // 低频数据检查 dirty flag
  if (this.dirty || currentStroke || this.viewportChanged(viewport)) {
    this.render();
    this.dirty = false;
    this.lastViewport = viewport;
  }
  this.rafId = requestAnimationFrame(this.loop);
};
或者更简单：高频场景下每帧都重绘，用 culling 和 LOD 控制性能，而不是用 dirty flag 跳过帧。

问题 5：去掉 ReactFlow 的风险
文档决定完全去掉 ReactFlow。这个决策需要评估工作量。

ReactFlow 提供的能力：

viewport 管理（pan, zoom, fit）
节点拖拽
节点选择
连线（edges）
minimap
键盘导航
触摸手势
需要自己实现的：

viewport 管理 → CameraController（文档已规划）
节点拖拽 → SelectTool / DragTool（文档已规划）
节点选择 → SelectTool（文档已规划）
minimap → 需要额外实现
触摸手势 → 需要额外实现（pinch zoom 等）
建议：

去掉 ReactFlow 是正确的方向（消除双数据源），但要确保功能行为清单里包含了 ReactFlow 当前提供的所有交互能力，特别是触摸手势和 minimap。

三、补充建议
1. 撤销/重做的实现
文档提到用 undoMiddleware 实现撤销/重做，但没有详细设计。

tldraw 的做法：

tldraw 的 Store 内置了历史记录：

// 每次 put/remove 都会记录到历史
store.put([shape]);

// undo/redo 通过历史记录回放
editor.undo();
editor.redo();
建议：

Zustand 的 undo 中间件（如 zundo）是基于快照的，对于大量形状的画布可能有性能问题。考虑实现基于 command 的 undo：

interface Command {
  execute(): void;
  undo(): void;
}

// 每个操作都是一个 command
class CreateShapeCommand implements Command {
  constructor(private store: CanvasStore, private shape: ShapeRecord) {}
  execute() { this.store.getState().putShape(this.shape); }
  undo() { this.store.getState().removeShapes([this.shape.id]); }
}
2. 形状的 ID 生成
文档没有提到 ID 生成策略。

tldraw 的做法：

tldraw 用 createShapeId() 生成带类型前缀的 ID：

const id = createShapeId(); // "shape:abc123"
这样从 ID 就能知道记录类型，方便调试和序列化。

建议：

定义统一的 ID 生成策略，考虑：

唯一性（UUID 或 nanoid）
可读性（带类型前缀）
协作场景（避免冲突）
3. 形状的 z-index 管理
文档提到了图层（layers），但没有详细说明形状的 z-index 如何管理。

tldraw 的做法：

tldraw 用 index 字段（fractional indexing）管理 z-order：

interface TLShape {
  id: string;
  index: string; // "a0", "a1", "a0V" — 分数索引
  // ...
}
这样插入形状时不需要更新其他形状的 index。

建议：

如果 Matelink 需要频繁调整 z-order（置顶、置底、上移、下移），考虑使用 fractional indexing 而不是整数 index。

四、总结
这份重构文档的整体方向是正确的：

✅ 三层架构（Store / Editor / View）符合画布软件最佳实践 ✅ 双渲染通道（Canvas 2D + DOM）的分工合理 ✅ Big Bang 策略适合零用户项目 ✅ 物理定律（ESLint 规则）确保架构约束可执行

需要注意的点：

⚠️ Zustand selector 需要精确，避免不必要的重渲染 ⚠️ ShapeUtil 的渲染通道声明可以更灵活 ⚠️ 高频/低频状态的边界需要更清晰的定义 ⚠️ CanvasRenderer 的 dirty flag 机制需要处理高频场景 ⚠️ 去掉 ReactFlow 后需要自己实现触摸手势和 minimap

建议在动手之前：

完成功能行为清单（文档已强调）
明确 Zustand + signals 的边界规则
设计 undo/redo 的具体实现方案
评估触摸手势的实现复杂度
