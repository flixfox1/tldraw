# Big Bang 重构前检查清单

> 基于《重构前注意事项》整理，对照《三层架构基石文档》逐项分析
> 目的：在动手写代码之前，把所有阻塞项和待决策项理清楚

---

## 一、阻塞项（不解决就不能动手）

### 1.1 功能行为清单 — P0 阻塞

**现状**：基石文档明确要求"功能行为清单由人工编制，不由 agent 生成"，但目前这份清单不存在。

**为什么是 P0**：Big Bang 的本质是"在新骨架上重新组装旧功能"。没有清单 = 没有验收标准 = 盲飞。80/20 陷阱（重写 80% 后发现剩余 20% 边界情况要花 80% 时间）是 Big Bang 最常见的失败模式。

**清单需要覆盖的功能域**：

| 功能域 | 关键行为（示例，非完整） |
|--------|------------------------|
| 画笔 (freehand + paintbrush) | 基础绘制、压感、auto-split、LOD 裁剪、合成模式、透明度 |
| 选择 | 点选、框选、涂鸦选、Shift 多选、深度选择 |
| 橡皮擦 | 擦除预览、scribble 模式 |
| 文本节点 | 创建、编辑、富文本、IME 输入法 |
| 图片节点 | 拖入、标记、变换手柄 |
| 快捷键 | 当前所有绑定的完整映射表 |
| 右键菜单 | 所有操作项 |
| 撤销/重做 | 覆盖的操作范围 |
| 视口操作 | pan、zoom、minimap 联动、scroll zoom |
| 图层系统 | 创建、切换、可见性、锁定 |

**行动项**：人工编制，格式参照基石文档 6.2 节的模板。

### 1.2 现有测试的业务断言提取 — P0 阻塞

**现状**：现有测试（如 `layerManager.property.test.ts`）中的断言逻辑描述了旧系统的不变量。Big Bang 会因 import 路径和 API 变化导致所有测试编译失败。

**风险**：直接丢弃测试 = 丢弃已有的行为规约。

**行动项**：在删除旧测试之前，提取所有业务断言作为功能行为清单的补充验证点。特别是 property-based test 中描述的不变量（如图层操作的不变量），这些在新架构中应同样成立。

---

## 二、架构待决策项（影响 Store 骨架实现）

### 2.1 ShapeRecord 类型设计 — P1

**问题**：基石文档定义了 `IShapeUtil` 接口和 `CanvasStoreState` 中的 `shapes: Map<string, ShapeRecord>`，但没有定义 `ShapeRecord` 的完整类型。

**难点**：不同形状的数据结构差异极大：

| 形状类型 | 特有数据 |
|----------|----------|
| 笔触 (Stroke) | points 数组 + 压感数据 |
| 画刷 (Paintbrush) | 合成模式 + 透明度 |
| 文本 (Text) | Tiptap JSON |
| 图片 (Workbench) | URL + 标记列表 |

**待决策**：
- 方案 A：Discriminated Union — `type ShapeRecord = StrokeRecord | TextRecord | WorkbenchRecord | ...`
- 方案 B：泛型 + 类型参数 — `ShapeRecord<T extends ShapeData>`

**影响范围**：store slice 的设计、`putShape` / `updateShape` 的类型签名、ShapeUtil 的泛型约束。

**建议**：Discriminated Union 更适合 Zustand 的 `Map<string, ShapeRecord>` 模式，因为 Map 的 value 类型必须统一，泛型参数无法在 Map 层面表达。tldraw 也采用了类似的 discriminated union 方案。

### 2.2 Map 的不可变更新策略 — P1

**问题**：`putShape` 当前实现是 `new Map(state.shapes)` 全量复制。当笔触数量达到几千条时，每次操作都全量复制 Map 会有性能问题。

**待决策**：
- 方案 A：Immer middleware — 自动处理不可变更新，代码最简洁，但 Immer 对 Map 的 proxy 有额外开销
- 方案 B：自己做 structural sharing — 手动管理引用，性能最优但代码复杂
- 方案 C：按 layer 分桶 — `Map<layerId, Map<shapeId, ShapeRecord>>`，缩小每次复制的范围

**建议**：先用方案 A（Immer）快速搭建骨架，性能瓶颈出现后再切换到方案 C。过早优化会拖慢骨架搭建速度。

### 2.3 撤销/重做策略 — P1

**问题**：基石文档提到用 `undoMiddleware` 替代 `CommandManager`，但没有展开设计。

**两种方案的对比**：

| 维度 | State Snapshot (如 zundo) | Command Object (当前方案) |
|------|--------------------------|--------------------------|
| 实现复杂度 | 低（中间件自动处理） | 高（每个操作需要定义 undo/redo） |
| 内存开销 | 高（每次操作存完整快照） | 低（只存 delta） |
| 粒度控制 | 粗（整个 state） | 细（可以合并连续操作） |
| 画笔场景 | 一笔几百个点 = 几百个快照？ | 一笔 = 一个 command |

**关键问题**：画笔操作中，一笔可能产生几百个 `pointerMove` 事件。如果用 snapshot 方案，需要明确"一笔"的边界（`pointerDown` 到 `pointerUp`），只在 `pointerUp` 时创建快照。

**建议**：混合方案 — 用 snapshot 作为基础（简化实现），但对画笔等高频操作做 debounce/batch，只在操作完成时创建 checkpoint。

### 2.4 DOM 节点批量定位机制 — P2

**问题**：去掉 ReactFlow 后，DOM 通道的节点定位需要自己实现。当前 ReactFlow 帮你做了 viewport transform → CSS transform 的映射。

**基石文档的方案**：CameraController 用 `@preact/signals` 管理 viewport，DOMNodeLayer 读取 viewport signal 做 CSS transform。

**待明确**：
- 是在 container div 上做一次 transform（所有子节点跟随），还是每个节点单独计算？
- 前者性能更好（一次 CSS transform），后者灵活性更高（节点可以有独立的 transform）

**建议**：container div 统一 transform。图片和文本节点的位置是相对于画布坐标系的，viewport transform 应该在容器层面统一处理，节点只需要关心自己在画布坐标系中的位置。

---

## 三、执行优化建议

### 3.1 迁移顺序调整 — 提前视图层骨架

**基石文档的顺序**：Store → Infra → Editor → View → 工具 → 渲染 → 辅助功能

**问题**：在阶段 4（View 骨架）完成之前，没有任何可视化验证手段。阶段 1-3 全部是盲写。

**建议调整为**：

```
Store 骨架 → View 骨架（最小可渲染）→ Editor 骨架 → 按功能域逐个填充
```

具体来说：
1. 搭建 Zustand canvasStore（最小 slice）
2. 搭建 CanvasShell + CanvasRenderer（能画一个矩形就行）+ DOMNodeLayer（能显示一个 div 就行）
3. 搭建 CameraController（能 pan/zoom）
4. 搭建 CanvasEditor 骨架
5. 然后每迁移一个功能域，都能立刻在画布上看到效果

**收益**：每一步都有视觉反馈，避免盲写到后期才发现基础层有问题。

### 3.2 旧代码保留策略

**原则**：Big Bang ≠ 一开始就删旧代码。

**策略**：
- 新代码写在新目录（`src/core/store/`、`src/core/editor/`、`src/view/`）
- 旧代码（`MatelinkCanvas.tsx`、`layerManager.ts`、`contexts/` 等）原地不动
- 两套代码并存期间，旧系统仍可运行，方便随时对照行为
- 功能行为清单 100% 通过后，一次性删除旧代码

**注意**：这不是绞杀者模式。不需要适配器，旧代码只是参考，不与新代码交互。

### 3.3 ESLint 规则时机

**基石文档建议**：先 warn 再逐步升级为 error（第七部分 7.5 节）。

**更好的做法**：在阶段 1 骨架搭建时就对新目录设为 error。Big Bang 的优势就是新代码从第一天就遵守规则，没有历史包袱。

```javascript
// 对新目录：error
// 对旧目录：用 ESLint overrides 豁免
overrides: [
  {
    files: ['src/core/**', 'src/view/**'],
    rules: {
      'import/no-restricted-paths': 'error',
      'max-lines': ['error', { max: 300 }],
    }
  }
]
```

---

## 四、总结：动手前的 Checklist

| # | 项目 | 优先级 | 状态 | 负责人 |
|---|------|--------|------|--------|
| 1 | 编制功能行为清单 | P0 阻塞 | ❌ 未开始 | 人工 |
| 2 | 提取现有测试的业务断言 | P0 阻塞 | ❌ 未开始 | 人工 + Agent |
| 3 | 确定 ShapeRecord 类型设计 | P1 | ❌ 未决策 | 人工 |
| 4 | 确定 Map 不可变更新策略 | P1 | ❌ 未决策 | 人工 |
| 5 | 确定撤销/重做方案 | P1 | ❌ 未决策 | 人工 |
| 6 | 确定 DOM 节点定位机制 | P2 | ❌ 未决策 | 人工 |
| 7 | 调整迁移顺序（View 骨架提前） | P2 | ❌ 未确认 | 人工 |
| 8 | 配置 ESLint 规则（新目录 error） | P2 | ❌ 未开始 | Agent |

**一句话**：功能行为清单是唯一的硬阻塞。ShapeRecord 类型和 undo 策略是影响骨架实现的关键决策。其余都是执行层面的优化，可以边做边调。
