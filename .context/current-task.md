# Current Task

## Task Title

将 Kanban CardDetail 改为固定 `4:4:2` 三栏布局，并将 Evidence Tab 移至右侧第三列常驻显示。

## Objective

CardDetail 弹窗内部采用固定三栏结构：

```text
Task 4fr | Session 4fr | Evidence 2fr
```

左侧 TaskDetail 不再包含 Evidence Tab；右侧第三列常驻显示原 Evidence Tab 的全部内容。第三列不依赖 Session，即使当前没有任何运行内容，Evidence 区域仍然存在并显示自身的空状态。

本次同时彻底移除栏宽拖拽能力。栏宽由 CSS Grid 固定，用户不能调整，历史拖拽比例不再读取或保存。

## Confirmed Decisions

1. 三栏比例严格为 `4:4:2`，即 `40% : 40% : 20%`。
2. 不使用 12 栏 `col-span` 体系，直接使用 `fr` 表达比例。
3. 弹窗外部尺寸保持不变：普通模式继续使用 `w-full max-w-7xl h-[88vh]`，全屏模式继续使用 `h-screen max-w-none`。
4. Task、Session、Evidence 三栏始终存在，不因 Session 是否存在改变比例，也不让 Task 栏扩展。
5. Evidence 右侧第三列常驻显示，不由 `activeSessionId` 控制。
6. Evidence 与 Session 无业务依赖。右侧 Evidence 只绑定当前 `activeTask`。
7. 左侧移除 Evidence Tab 按钮及其内容分支。
8. 暂不处理左侧顶部的 Evidence 状态徽标（MetaBadge），保留现状，后续另行决定。
9. 本次不修改 Evidence、Artifact、Task、Session 或 Gate 的数据模型与业务语义。
10. 本次不讨论右侧第三列的视觉优化，只保证组件整体正确挂载。

## Current Architecture

CardDetail 弹窗当前已经存在三列 Grid：

```text
第一列：KanbanCardDetail
第二列：Session / Chat
第三列：Reserved 空容器
```

当前 Grid 模板已经是：

```tsx
grid-cols-[minmax(0,4fr)_minmax(0,4fr)_minmax(0,2fr)]
```

第三列目前只是空的 `aside`，并使用 `aria-hidden="true"`。

左侧 `KanbanCardDetail` 的 Evidence Tab 当前渲染两个独立组件：

```tsx
<EvidenceBundlePanel task={task} />

<KanbanCardArtifacts
  taskId={task.id}
  requiredArtifacts={nextTransitionArtifacts.nextRequiredArtifacts}
  refreshSignal={refreshSignal}
/>
```

相关文件：

- `src/app/workspace/[workspaceId]/kanban/kanban-card-detail.tsx`
- `src/app/workspace/[workspaceId]/kanban/kanban-detail-panels.tsx`
- `src/app/workspace/[workspaceId]/kanban/kanban-card-artifacts.tsx`
- `src/app/workspace/[workspaceId]/kanban/kanban-tab-panels.tsx`

## Evidence Placement Contract

### Left Column

左侧 `KanbanCardDetail` 只保留以下 Tab：

```text
Overview
Execution
JIT Context
Changes
Runs
```

删除：`Evidence Bundle` Tab。

删除 Evidence Tab 不等于删除左侧顶部已有的 Evidence MetaBadge。MetaBadge 是标题区的一行简短摘要，本期保留，不纳入本次迁移范围。

### Right Column

右侧第三列改为 Evidence Pane，常驻渲染：

```text
EvidenceBundlePanel
KanbanCardArtifacts
```

右侧渲染条件只取决于 `activeTask`：

```text
有 activeTask
  -> 常驻显示 Evidence Pane

无 artifacts / 无 evidenceSummary
  -> 由现有组件显示空状态
```

禁止使用以下条件控制 Evidence 是否存在：

- `activeSessionId`
- `sessionInfo`
- `hasSessionPane`
- ACP/A2A Session 状态

Evidence 只与 Task 相关，与 Session 展示无关。

### Data Inputs

右侧 Evidence Pane 使用现有数据：

- `activeTask.evidenceSummary`
- `activeTask.id`
- `board.columns`
- `refreshSignal`

其中 `requiredArtifacts` 继续由当前 Task 所在列的下一目标列配置推导：

```text
board.columns
  + activeTask.columnId
  -> nextTransitionArtifacts.nextRequiredArtifacts
```

不新增 API，不改变 Artifact 查询方式，不改变 Gate 判断。

## Fixed Three-Column Layout

主容器保持：

```tsx
className="grid h-full grid-cols-[minmax(0,4fr)_minmax(0,4fr)_minmax(0,2fr)]"
```

所有直接子项保留 `min-w-0`，需要纵向内容的区域保留 `min-h-0` 和自身滚动边界。

三栏职责：

```text
第一栏 4fr：TaskDetail，不含 Evidence Tab
第二栏 4fr：ACP、A2A 或 Empty Session 内容
第三栏 2fr：Evidence 常驻内容
```

第二栏仍可根据 `hasSessionPane` 决定内部显示哪一种 Session 内容，但不得影响三栏是否存在或第一栏宽度。

第三栏不再使用 `aria-hidden="true"`，因为其中包含用户可见、可访问的 Evidence 内容。建议将测试标识改为：

```text
kanban-detail-evidence-pane
```

## Drag Removal Scope

### `kanban-tab.tsx`

删除：

- `KANBAN_DETAIL_SPLIT_RATIO_KEY`
- `MIN_DETAIL_SPLIT_RATIO`
- `MAX_DETAIL_SPLIT_RATIO`
- `detailSplitRatio` state
- `isDraggingDetailSplit` state
- `detailSplitContainerRef`
- 从 `localStorage` 读取历史比例的 effect
- 将比例写入 `localStorage` 的 effect
- `mousemove` / `mouseup` 拖拽 effect
- 对 `document.body.style.cursor` 和 `userSelect` 的临时修改
- 传递给 `KanbanTaskDetailOverlay` 的拖拽相关 props

不主动删除旧的 localStorage key。停止读取和写入后，旧值不会影响新布局；本任务也不增加缓存迁移逻辑。

### `kanban-tab-panels.tsx`

删除：

- `detailSplitContainerRef` prop 和类型
- `detailSplitRatio` prop 和类型
- `setIsDraggingDetailSplit` prop 和类型
- 主容器上的拖拽 ref
- Task/Session 容器上的动态 `style.width`
- `kanban-detail-split-handle` 节点
- 只服务于拖拽布局的无用 import

保留固定 CSS Grid 三栏结构，并将右侧 reserved 容器替换为 Evidence Pane。

## Evidence Move Scope

### `kanban-card-detail.tsx`

删除或调整：

- Evidence Tab 类型成员
- `detailTabs` 中的 Evidence 条目
- `activeTab === "evidence"` 内容分支
- 仅用于左侧 Evidence Tab 的 imports

保留：

- 顶部 Evidence MetaBadge
- `getEvidenceStatus()` 及其现有摘要逻辑
- Evidence 相关 Task 数据和 API 行为

### `kanban-tab-panels.tsx`

在第三列直接组合现有 Evidence 组件。可以直接挂载，也可以抽取一个很薄的 `KanbanTaskEvidencePane` 组件；无论采用哪种方式，都不得复制或重写 Evidence 业务逻辑。

推荐输入：

```text
task              = activeTask
requiredArtifacts = resolveKanbanTransitionArtifacts(board.columns, activeTask.columnId).nextRequiredArtifacts
refreshSignal     = refreshSignal
```

不得把 Evidence Pane 的显示条件绑定到 Session。

## Behavioral Invariants

本次只改变布局和 Evidence 的挂载位置，不改变：

- CardDetail 打开、关闭和全屏切换
- `activeTaskId`、`activeSessionId` 的选择语义
- ACP、A2A、Empty Session 的显示与恢复逻辑
- Session 历史导航和 ChatPanel 行为
- Task 编辑、Execution、Changes、Runs 和删除操作
- EvidenceSummary 的计算方式
- ArtifactStore、Artifact API 和 MCP Artifact 工具
- `requiredArtifacts`、transition gate 和 `move_card` 行为
- Task、Session、TaskLaneSession、Run ledger 数据模型
- Rust/Next.js API

## Explicitly Out Of Scope

- 不调整右侧第三列的视觉美观或宽度。
- 不处理左侧 Evidence MetaBadge 是否应删除。
- 不修改 Evidence 或 Artifact 的语义。
- 不修改任何运行状态模型或状态文案。
- 不修改 Gate 规则、requiredArtifacts 规则或迁移行为。
- 不让 Evidence 按 Column 或 Session 分组。
- 不新增 Evidence 与 Session 的关联。
- 不新增 API 或持久化字段。
- 不讨论或迁移旧的内存 Tab 状态；每次验证会重启应用，旧内存状态不属于本次范围。
- 不引入第三方 Split Pane 或 Resizable Panel 库。
- 不执行 UI 视觉重设计。

## Implementation Plan

1. 在 `kanban-card-detail.tsx` 移除左侧 Evidence Tab 条目和内容分支，保留顶部 Evidence MetaBadge。
2. 从 `kanban-card-detail.tsx` 移除仅服务于 Evidence Tab 的组件 imports。
3. 在 `kanban-tab-panels.tsx` 引入并挂载 `EvidenceBundlePanel` 与 `KanbanCardArtifacts`。
4. 在右侧第三列使用 `activeTask` 作为唯一业务输入，始终渲染 Evidence Pane；没有证据时由组件显示空状态。
5. 为右侧 Evidence Pane 计算并传入 `nextRequiredArtifacts` 与 `refreshSignal`。
6. 移除右侧容器的 `aria-hidden="true"`，并更新测试定位标识。
7. 按既有固定 `4:4:2` Grid 保留 Task、Session、Evidence 三列。
8. 按既有范围删除栏宽拖拽状态、ref、effect、props、inline width 和拖拽手柄。
9. 更新聚焦测试：左侧不存在 Evidence Tab，右侧常驻显示 Evidence 内容，Evidence 不依赖 Session。

## Acceptance Criteria

- CardDetail 普通模式仍使用 `max-w-7xl h-[88vh]`。
- CardDetail 全屏模式行为保持不变。
- 弹窗内部始终存在 Task、Session、Evidence 三个直接布局区域。
- Grid 模板严格为 `4fr 4fr 2fr`，并使用 `minmax(0, ...)`。
- 左侧不存在 `Evidence Bundle` Tab。
- 左侧顶部 Evidence MetaBadge 暂时保持现状。
- 右侧 Evidence Pane 始终绑定当前 `activeTask`，不依赖 `activeSessionId`。
- 右侧包含原 Evidence Tab 的 `EvidenceBundlePanel` 和 `KanbanCardArtifacts`。
- 没有 artifacts 或 summary 时，右侧仍存在并显示组件提供的空状态。
- 右侧 Evidence Pane 不因 ACP、A2A 或 Empty Session 状态变化而消失。
- `requiredArtifacts` 和 Artifact 列表行为保持不变。
- Task、Session、Evidence 三栏不会因 Session 缺失而改变比例。
- 页面不存在可拖拽分隔手柄。
- 页面不存在 `cursor-col-resize` 交互。
- 鼠标移动不会改变三栏比例。
- 不再读取或写入 `routa:kanban-detail-split-ratio`。
- 源码中不再存在 `detailSplitRatio`、`isDraggingDetailSplit` 或 `detailSplitContainerRef` 的 CardDetail 布局链路。
- Task、Session、ACP、A2A、Evidence 和全屏切换的既有功能保持可用。

## Test Plan

### Component Tests

- 验证左侧 Tab 列表不包含 `Evidence Bundle`。
- 验证右侧 Evidence Pane 始终存在于三栏布局中。
- 验证右侧 Evidence Pane 使用 `activeTask`，不需要 `activeSessionId`。
- 验证没有 artifacts 时仍显示 Evidence 空状态。
- 验证 `EvidenceBundlePanel` 和 `KanbanCardArtifacts` 均被渲染。
- 验证 `requiredArtifacts` 和 `refreshSignal` 正确传递。
- 验证主容器包含固定 `grid-cols-[minmax(0,4fr)_minmax(0,4fr)_minmax(0,2fr)]`。
- 验证 Task、Session、Evidence 三个区域同时存在。
- 验证有 ACP Session 时中栏继续显示 `ChatPanel`。
- 验证 A2A Session 时中栏继续显示 `A2ASessionPane`。
- 验证可运行但尚未创建 Session 时中栏继续显示 `KanbanEmptySessionPane`。
- 验证没有可显示 Session 内容时，中栏和右栏仍存在，Task 不扩展。
- 验证全屏切换不改变 Grid 比例。

### Static Regression Checks

- 搜索确认不存在 `KANBAN_DETAIL_SPLIT_RATIO_KEY`。
- 搜索确认不存在 CardDetail 使用的 `detailSplitRatio`。
- 搜索确认不存在 CardDetail 使用的 `isDraggingDetailSplit`。
- 搜索确认不存在 `kanban-detail-split-handle`。
- 搜索确认 Task/Session 容器不再使用动态 `style.width`。
- 搜索确认左侧 `detailTabs` 不再包含 `evidence`。
- 搜索确认右侧 Evidence 渲染不读取或判断 Session 状态。

### Validation Boundary

本任务按 `DEVELOPMENT.md` 约束执行。验证只覆盖与本次 UI 改动直接相关的组件/静态检查；不引入 E2E、Playwright、Fitness、entrix、Docker 或 Git hooks。
