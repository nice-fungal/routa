# Current Task

## Task Title

将 TaskDetail 的 Story Readiness 合并到 Overview Tab。

## Objective

简化 TaskDetail 的 Tab 导航：移除独立的 `Story Readiness` Tab，将其完整内容移动到 `Overview` Tab 中，按从上到下的顺序统一展示。

保留现有 `StoryReadinessPanel` 组件，不复制或重写其内部逻辑。此次调整只改变详情页内容的组织方式，不改变 Story Readiness 的数据、计算、文案和业务语义。

## Current Structure

TaskDetail 当前通过 `detailTabs` 配置和 `activeTab` 条件渲染内容：

- 第一个 Tab：`overview`
- 第二个 Tab：`readiness`，界面显示名称为 `Story Readiness`
- 后续 Tab：`execution`、`jitContext`、`changes`、`evidence`、`runs`

当前 `Overview` 内容主要包括：

1. Objective 编辑区。
2. Review Feedback。
3. Progress Notes。
4. Test Cases。

当前 `Story Readiness` 内容由独立的 `StoryReadinessPanel` 组件提供，组件位于：

`src/app/workspace/[workspaceId]/kanban/kanban-detail-panels.tsx`

该组件只依赖 `task` 和 `compact`，没有独立编辑状态，也不依赖 Readiness Tab 专属回调，因此可以直接并入 Overview。

## Scope

### 1. 调整 Tab 导航

在 `src/app/workspace/[workspaceId]/kanban/kanban-card-detail.tsx`：

- 从 `KanbanDetailTabId` 移除 `"readiness"`。
- 从 `detailTabs` 移除 `readiness` 配置项。
- 移除 `activeTab === "readiness"` 的独立渲染分支。
- 保留其他 Tab 及其顺序不变。

### 2. 将 Readiness 面板放入 Overview

在 `activeTab === "overview"` 的内容中直接渲染：

```tsx
<DetailSection
  title={t.kanbanDetail.storyReadiness}
  description={compactMode ? undefined : t.kanbanDetail.storyReadinessHint}
  compact={compactMode}
>
  <StoryReadinessPanel task={task} compact={compactMode} />
</DetailSection>
```

推荐顺序：

```text
Objective
Story Readiness
Review Feedback
Progress Notes
Test Cases
```

原因：Story Readiness 检查的是任务定义是否具备进入后续流程的条件，应紧跟 Objective，位于其他反馈和进度信息之前。

### 3. 处理旧的 Readiness Tab 状态

当前 Tab 选择会保存在 `persistedKanbanDetailTabs` 内存 Map 中。合并后，如果旧状态仍然是 `readiness`，必须自动回退到 `overview`，避免出现：

- 当前 Tab ID 仍为 `readiness`，但没有对应内容。
- 详情面板为空。
- 无法通过 Tab 导航返回有效内容。

可采用以下任一等价方式：

- 读取持久化状态时将 `readiness` 归一化为 `overview`。
- Tab 列表变化后校正当前选中的 Tab。
- 初始化 `activeTab` 时仅接受当前仍存在的 Tab ID。

优先选择局部、可读的归一化逻辑，不引入新的全局状态管理。

### 4. 处理顶部摘要徽标

详情页头部当前已有 Story Readiness 的 `MetaBadge` 摘要：

```text
Ready for Dev / Blocked for Dev
```

本 TASK 暂时保留该徽标。合并后允许以下展示结构同时存在：

- 顶部：快速摘要徽标。
- Overview：完整 Story Readiness 检查面板。

是否移除顶部摘要徽标另建 TASK 讨论，本 TASK 不处理。

## Preserve

不得改变以下内容：

- `StoryReadinessPanel` 组件及其内部实现。
- `task.storyReadiness` 和 `task.investValidation` 数据结构。
- Readiness 的字段检查、Invest Validation 检查和状态计算。
- `Ready for Dev`、`Blocked for Dev`、缺失字段和 Gate 文案。
- `Story Readiness` 的 i18n 文案。
- Overview 中 Objective、Review Feedback、Progress Notes、Test Cases 的现有编辑和展示行为。
- 其他 Tab：Execution、JIT Context、Changes、Evidence、Runs。
- 详情页顶部标题、MetaBadge、删除入口、刷新、全屏和 Session 面板行为。
- 后端 API、Task 数据模型和工作流逻辑。

## Explicitly Out Of Scope

- 不修改 Story Readiness 的业务规则或后端计算。
- 不把 `StoryReadinessPanel` 的代码复制到 `kanban-card-detail.tsx`。
- 不将 Readiness 内容移到其他 Tab。
- 不删除顶部 Story Readiness 摘要徽标。
- 不重排 Overview 中其他区块，除非为插入 Story Readiness 所必需。
- 不修改 Execution、Runs、Evidence、Gate 或 Agent 运行逻辑。
- 不修改 Card 封面或主状态徽章。

## Architecture Decision

`Overview` 是 TaskDetail 的基础信息总览，采用从上到下的组合结构。Story Readiness 属于任务定义和流程准备信息，与 Objective 同属 Overview 范畴，不需要单独占用一个 Tab。

保留 `StoryReadinessPanel` 作为独立组件，只移动组件调用位置。这样可以：

- 减少一个 Tab，降低导航复杂度。
- 将 Objective 和其完整准备度检查放在同一上下文。
- 避免复制 Readiness 逻辑。
- 保持组件边界清晰，便于后续单独维护。

代价是 Overview 内容变长，Story Readiness 会在进入 Overview 时直接加载和展示。这是本次简化接受的布局变化。

## Acceptance Criteria

- TaskDetail 不再显示名为 `Story Readiness` 的独立 Tab。
- `detailTabs` 中不存在 `readiness` 项。
- `KanbanDetailTabId` 不再包含 `"readiness"`。
- Overview 中按固定顺序展示 Objective、Story Readiness、Review Feedback、Progress Notes、Test Cases。
- `StoryReadinessPanel` 仍以组件形式渲染，未复制其内部实现。
- 旧的持久化 `readiness` Tab 状态会自动回退到 `overview`。
- 顶部 Story Readiness 摘要徽标仍存在且行为不变。
- Story Readiness 的所有状态、检查项、缺失字段和 i18n 文案保持不变。
- Overview 原有编辑能力保持不变。
- 其他 Tab 的导航和内容保持不变。
- 详情页刷新、关闭、全屏、删除和 Session 相关行为保持不变。

## Test Plan

### TaskDetail Render Tests

- 验证 Tab 列表不再包含 `Story Readiness`。
- 验证默认 Overview 中能看到 Story Readiness 面板。
- 验证 Story Readiness 面板位于 Objective 之后、Review Feedback 之前。
- 验证 `ready` 和 `blocked` 两种 Story Readiness 状态仍正确展示。
- 验证缺失字段、检查项和 Invest Validation 内容仍然存在。
- 验证旧 `readiness` 选中状态会回退到 Overview。

### Regression Tests

- 验证 Overview 中 Objective 编辑、Progress Notes 和 Test Cases 行为不变。
- 验证其他 Tab 仍可正常切换。
- 验证顶部 Story Readiness `MetaBadge` 仍存在。

### Static Checks

- 检查不存在对 `activeTab === "readiness"` 的残留渲染分支。
- 检查 `KanbanDetailTabId`、`detailTabs` 和 Tab 状态归一化逻辑类型一致。
- 运行 TaskDetail 相关聚焦测试和对应 TypeScript/Lint 检查。

## Validation Boundary

本 TASK 只验证 Tab 组织和内容复用：

- Story Readiness 从独立 Tab 移入 Overview。
- Story Readiness 组件和业务逻辑不变。
- 旧 Tab 状态不会造成空面板。
- 其他详情功能不受影响。

