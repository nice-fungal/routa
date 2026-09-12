# Current Task

## Task Title

移除 Kanban CardDetail 的 JIT Context / History Memory Tab

## Objective

从 Kanban CardDetail 中完整移除 `JIT Context` Tab。该 Tab 当前面向用户显示为 `History Memory`，内部 Tab id 为 `jitContext`，主体组件为 `JitContextPanel`。

本任务删除：

```text
CardDetail 的 History Memory Tab
  |- JIT 检索结果展示
  |- 手动刷新 History Memory
  |- Load into current session
  +- Open History Analysis
```

移除后 CardDetail 只保留：

```text
Overview
Execution
Activity
```

采用“删除 CardDetail 产品入口，保留底层能力和数据兼容”的边界。不会顺带删除 Task-Adaptive Harness、Task 数据字段、数据库列、自动 Session preload、History Memory Policy、MCP tools 或 `history-summary-analyst` Specialist。

## Confirmed Decisions

1. 删除整个 CardDetail `JIT Context / History Memory` Tab，不迁移其内容。
2. 删除 `JitContextPanel` 以及该面板独占的展示、刷新、手动注入和 History Analysis 启动逻辑。
3. 删除 CardDetail 到 `kanban-tab-panels.tsx` 的 JIT 专用 props/callback 链。
4. 删除当前 UI 对 `POST /api/harness/task-adaptive` 和 `/history-summary` 的手动调用入口。
5. 删除当前 UI 创建 `history-summary-analyst` 独立 ACP Session 的入口。
6. 保留 `history-summary-analyst` Specialist 定义；删除 Tab 不能证明它没有其他入口。
7. 保留 `Task.contextSearchSpec`、`Task.jitContextSnapshot`、`TaskJitContextAnalysis` 及数据库映射。
8. 保留 Task-Adaptive Harness API/core、History Summary API 和相关 MCP tools。
9. 保留当前 Task 的 Saved History Memory prompt 注入和跨 Task Relevant History Memory 自动 preload。
10. 保留 Board `historyMemoryPolicy` 及其设置 UI。
11. 保留 Agent trigger、context preload、lane experience、reasoning memory 等非 Tab 消费者。
12. 不清理已有 `jit_context_snapshot` 数据，不新增数据库迁移。
13. 只删除确认由该 Tab 独占的 i18n keys 和 UI tests；共享 key 按实际引用处理。
14. 旧的进程内 persisted tab 值 `jitContext` 必须回退到 Overview，不得出现空白详情页。

## Architecture Boundary

当前 Tab 混合了两种能力：

```text
JIT / Task-Adaptive Harness
  = 不调用 LLM 的规则检索、提取、评分和快照生成

History Analysis / history-summary-analyst
  = 独立 LLM Agent 对预加载材料做语义分析
```

基础 JIT 数据：

```text
Task.jitContextSnapshot
  -> PostgreSQL tasks.jit_context_snapshot JSONB
  -> SQLite tasks.jit_context_snapshot TEXT(JSON)
```

可选 LLM 结果：

```text
Task.jitContextSnapshot.analysis
```

删除 Tab 只删除两类能力在 CardDetail 中的可见入口和手动操作，不删除底层生成、保存或自动消费能力。

## Current UI Structure

### Tab Shell

入口：

```text
src/app/workspace/[workspaceId]/kanban/kanban-card-detail.tsx
```

当前 Tab：

```text
overview | execution | jitContext | activity
```

`jitContext` 导航使用 `t.kanbanDetail.jitContext`，英文为 `History Memory`。面板分支是：

```tsx
activeTab === "jitContext"
  -> <JitContextPanel />
```

### JIT Panel Responsibilities

`JitContextPanel` 位于：

```text
src/app/workspace/[workspaceId]/kanban/kanban-detail-panels.tsx
```

它当前负责：

- 从 `Task.jitContextSnapshot` 恢复显示数据。
- 首次展开时延迟调用 `POST /api/harness/task-adaptive`。
- 将 `TaskAdaptiveHarnessPack` 转换为 `TaskJitContextSnapshot` 并 PATCH 回 Task。
- 展示匹配置信度、原因、候选文件、命中 Session、失败、重复读取和 warnings。
- 展示已保存的 `jitContextSnapshot.analysis`。
- 将 JIT Prompt 手动发送到当前 ACP Session。
- 调用 `/api/harness/task-adaptive/history-summary` 预加载摘要。
- 构造 History Analysis Prompt。
- 请求父层创建独立 `history-summary-analyst` ACP Session。
- 对未 refinement 的 Backlog Task 清理推测性旧快照。

### Parent Callback Chain

`KanbanCardDetail` 当前接收：

```text
jitContextSessionId
onLoadJitContextIntoSession
onOpenJitContextHistoryAnalysis
```

由 `kanban-tab-panels.tsx` 提供：

```text
Load into current session
  -> acp.promptSession(...)

Open History Analysis
  -> resolve repo/branch/provider
  -> buildKanbanTaskAdaptiveHarnessOptions(...)
  -> startKanbanHistoryAnalysisSession(...)
  -> /api/acp session/new
  -> /api/acp session/prompt
```

## Removal Plan

### 1. kanban-card-detail.tsx

删除：

- `JitContextPanel` import。
- `KANBAN_DETAIL_TAB_IDS` 中的 `"jitContext"`。
- `detailTabs` 中的 History Memory 导航项。
- `activeTab === "jitContext"` 渲染分支。
- `KanbanCardDetailProps` 中的 `jitContextSessionId`、`onLoadJitContextIntoSession`、`onOpenJitContextHistoryAnalysis`。
- 对应 props 解构和传递。
- 只服务 JIT Tab 的 `resolvedWorkspaceId`。
- 只服务 JIT Tab 的 `getTaskHistoryRepositoryPath()`。

保留：

- `getTaskRepositoryPath()`，它仍服务 Execution/Repository mismatch。
- `onPatchTask`，其他区块仍使用。
- Overview、Execution、Activity 的导航、布局和 persisted tab 行为。

`persistedKanbanDetailTabs` 是进程内 Map。实现后 `normalizeKanbanDetailTab("jitContext")` 必须返回 `overview`。

### 2. kanban-detail-panels.tsx

删除 `JitContextPanel`，并清理仅由它使用的 helpers/types/imports。

已确认的 JIT helper 集群：

```text
hasTaskAdaptiveSearchHints
getMatchedFileDetails
packFromTaskJitContextSnapshot
buildRecommendedContextSearchSpec
buildTaskJitContextSnapshot
formatMatchConfidenceLabel
formatMatchedFileSeed
formatCompactPathList
buildReadableHistoryOverview
uniquePreserveOrder
formatBulletList
formatOrderedList
buildTranscriptHints
uniqueFailureSignals
resolveHistoryAnalysisSessionIds
PreloadedTaskHistorySummaryResult
buildJitContextSessionPrompt
buildHistorySummaryToolArgs
buildJitHistoryAnalysisPrompt
```

实施时逐项检查同文件引用后再删除。不能按行段整体删除：`SummaryGridItem` 位于该集群之后，但可能被其他 panel 使用。

预计可随 JIT 集群删除的 imports：

```text
AcpTaskAdaptiveHarnessOptions
TaskAdaptiveHarnessPack
TaskAdaptiveHistorySummary
TaskAdaptiveMatchedFileDetail
normalizeTaskJitContextAnalysis
normalizeTaskContextSearchSpec
TaskContextSearchSpec
TaskJitContextSnapshot
buildKanbanTaskAdaptiveHarnessOptions
hasConfirmedKanbanTaskAdaptiveContext
mergeTaskLaneExperienceIntoJitSnapshot
desktopAwareFetch / toErrorMessage（若无其他 panel 使用）
KanbanSpecialistLanguage（若无其他 panel 使用）
无剩余调用者的 React hooks
```

保留 `StoryReadinessPanel`、`EvidenceBundlePanel`、`ReviewFeedbackPanel` 及其 helper/imports。

### 3. kanban-tab-panels.tsx

删除 JIT Tab 独占的手动 Agent 启动集群：

```text
buildKanbanHistoryAnalysisSessionName
buildKanbanHistoryAnalysisCreationError
buildKanbanHistoryAnalysisSessionUrl
startKanbanHistoryAnalysisSession
```

删除传给 `KanbanCardDetail` 的：

```text
jitContextSessionId
onLoadJitContextIntoSession
onOpenJitContextHistoryAnalysis
```

删除 callback 内只为 History Analysis 计算的 repo/worktree/branch/provider、`buildKanbanTaskAdaptiveHarnessOptions()`、JIT 错误文案引用及专用 `/api/acp` 调用。

必须保留文件中其他 Session 创建、选择、恢复和 Agent pane 逻辑；不能因为 endpoint 相同而删除共享 ACP 调用。

### 4. Manual History Analysis Entry

删除 Tab 后不再存在：

```text
Open History Analysis
  -> 预加载 /history-summary
  -> 创建 history-summary-analyst Session
  -> 打开新 Session 页面
  -> 保存 analysis
```

这是本任务明确接受的产品变化。

仍保留 Specialist YAML、ACP 通用 Specialist 支持、`save_history_memory_context`、`summarize_task_history_context` 和 History Summary API。它们仍可被通用入口、MCP、外部客户端或未来 UI 使用。

### 5. Manual Load Into Current Session

删除 Tab 后，用户不能再点击 `Load into current session` 手动注入 JIT Prompt。

自动注入路径保留：

```text
src/app/api/acp/acp-session-create.ts
  -> historyMemoryPolicy
  -> loadRelevantTaskHistoryMemories()
  -> buildRelevantHistoryMemoryPromptSection()

src/core/kanban/agent-trigger.ts
  -> buildSavedHistoryMemoryPromptSection(task)
```

删除手动按钮不等于关闭 Saved/Relevant History Memory preload。

### 6. Snapshot Persistence Consequence

`JitContextPanel` 是基础快照的明确写入者：

```text
POST /api/harness/task-adaptive
  -> buildTaskJitContextSnapshot()
  -> PATCH Task.jitContextSnapshot
```

删除后，“用户打开/刷新 Tab 时生成并持久化基础快照”的路径消失。本任务不在其他 Tab 隐式重建它。

仍可能更新该字段的路径：

- `save_history_memory_context` 写入/合并 `analysis`。
- Task lane experience 合成 `perLaneAnalysis`。
- Agent/MCP task tools 更新 JIT analysis。
- Task Store/API 保持兼容读写。

已有快照继续保留并可被非 UI 消费者读取，不清理数据。

## Explicitly Preserved Capabilities

### Task Model And Storage

保留 `TaskContextSearchSpec`、`TaskJitContextSnapshot`、`TaskJitContextAnalysis`、normalize/parse/merge functions，以及 PostgreSQL/SQLite schema 和 Task Store 映射。不删除 `tasks.jit_context_snapshot`，不创建迁移。

### Harness And APIs

保留：

```text
POST /api/harness/task-adaptive
POST /api/harness/task-adaptive/history-summary
src/core/harness/task-adaptive.ts
src/core/harness/task-adaptive-tool.ts
src/core/harness/transcript-sessions.ts
src/core/harness/task-adaptive-path-signals.ts
```

它们还服务 Session startup、MCP、分析能力和潜在外部调用方。

### MCP And Specialist

保留：

```text
assemble_task_adaptive_harness
summarize_task_history_context
inspect_transcript_turns
save_history_memory_context
history-summary-analyst definitions
kanban-planning MCP registrations
```

### Automatic And Cross-task Consumers

保留：

```text
src/core/kanban/context-preload.ts
src/core/kanban/agent-trigger.ts
src/core/kanban/task-adaptive.ts
src/core/kanban/task-lane-experience.ts
src/core/kanban/task-lane-history.ts
src/app/api/acp/acp-session-create.ts
```

继续支持当前 Task Saved History Analysis、其他 Task Relevant History Memory、policy-controlled preload、session-start hydration、lane experience 和 reasoning-memory hints。

### Board Settings

保留 `historyMemoryPolicy` 及 Kanban Settings 控件，包括 sessions/files/features/confidence thresholds。它控制自动 preload，与手动 JIT Tab 独立。

## I18n Cleanup

删除确认只由 JIT Tab、`JitContextPanel` 和手动 History Analysis 使用的类型声明及中英文 locale 值。

候选 keys：

```text
jitContext / jitContextHint
showJitContext / hideJitContext / refreshJitContext
loadJitContextIntoCurrentSession
loadingJitContextIntoCurrentSession
jitContextLoadedIntoCurrentSession
openJitContextHistoryAnalysis
openingJitContextHistoryAnalysis
jitContextHistoryAnalysisOpened
jitContextHistoryAnalysisFailed
jitContextHistoryAnalysisPopupBlocked
loadingJitContext
matchConfidence / matchReasons
matchConfidenceHigh / matchConfidenceMedium / matchConfidenceLow
jitContextUnavailable / jitContextNoHistorySessions
jitContextNeedsRefinement / jitContextSearchFailed / noJitContext
historySummary
historyReadableInspectFiles
historyReadableNoPriorityFiles
historyReadableRecovered
historyReadableNeedsStrongerHints
savedHistoryAnalysis
analysisSessionLayers / analysisTopLeads
analysisTopFiles / analysisTopSessions
analysisInputIssues / analysisLocationIssues / analysisToolingIssues
analysisContextToInject / analysisReusablePrompts
analysisEvidence / analysisInference
historicalIssues / noHistoricalIssues
historySeedSessions / repeatedReadHotspots
relatedSessions / matchedFeature / matchedFiles
```

实施前再次全仓搜索；被其他 UI 使用的 key 必须保留或迁移，不能机械删除。

明确保留 `historyMemoryPolicy`、`historyMemoryPolicyHint`、mode 和 threshold keys，以及 Agent prompt 中的 Relevant/Saved History Memory 文案。

## Tests And Verification

### CardDetail Tests

更新 `kanban-tab.test.tsx`：

- Tab 只包含 Overview、Execution、Activity。
- 不存在 `data-testid="kanban-detail-tab-jitContext"`。
- `normalizeKanbanDetailTab("jitContext")` 返回 `overview`。
- 其他旧值继续回退 Overview。
- 剩余 Tab 选择和 persisted selection 正常。
- 打开 CardDetail 不请求 Harness 或 history-summary endpoint。

### Removed JIT UI Tests

从 `kanban-tab-detail-and-prompts.test.tsx` 删除只覆盖以下行为的场景：

```text
lazy load JIT Context
Open History Analysis
render saved structured history analysis
persist selected History Memory tab
fresh backlog speculative memory handling in panel
load from search hints
warning/empty states
reset on contextSearchSpec changes
Load into current session
```

保留 Story Readiness、Execution、Activity、Session tab、Review Feedback 等测试。

### Preserved Tests

保留：

```text
src/core/kanban/__tests__/task-adaptive.test.ts
src/core/kanban/__tests__/context-preload.test.ts
src/core/kanban/__tests__/agent-trigger.test.ts
src/core/kanban/__tests__/task-lane-experience.test.ts
src/core/harness/__tests__/*task-adaptive*
src/core/mcp/__tests__/*
src/app/api/harness/task-adaptive/__tests__/*
Task API 的 jitContextSnapshot persistence tests
Kanban Settings historyMemoryPolicy tests
```

这些验证保留的 domain/API/automatic preload，不因 UI 删除而移除。

### Static Checks

- 生产代码不再 import 或渲染 `JitContextPanel`。
- `KanbanDetailTabId` 不再接受 `jitContext`。
- `KanbanCardDetailProps` 不再包含 JIT props。
- tab panels 不再包含 CardDetail History Analysis launcher。
- CardDetail 不再请求 Harness/history-summary endpoints。
- CardDetail 不再通过 JIT UI 创建或 prompt ACP analysis Session。
- 没有 JIT-only helper、import、locale key、test mock 或 suppression 残留。
- Specialist、MCP、Harness API、Task fields、automatic preload 和 Board policy 仍有有效引用。

### Validation Level

实施代码后至少运行：

```text
受影响的 Kanban CardDetail / tab / prompt tests
TypeScript typecheck
Task-Adaptive Harness API tests
context-preload / agent-trigger / task-lane-experience tests
MCP save/summarize history context tests
```

该变更修改共享 CardDetail 和 Session callback shell。先运行 `entrix graph impact` / `entrix graph test-radius`，再执行相应 fast 或 normal tier。

## Acceptance Criteria

- CardDetail 只显示 Overview、Execution、Activity。
- 用户不能再从 CardDetail 打开、刷新或查看 History Memory/JIT Context。
- 用户不能再从 CardDetail 手动 Load into current session。
- 用户不能再从 CardDetail 手动启动 `history-summary-analyst`。
- `jitContext` 旧 persisted tab 值安全回退 Overview。
- `JitContextPanel` 无生产调用者并被删除。
- CardDetail 与 tab panels 不再保留 JIT props/callback。
- CardDetail 不再触发 JIT/history-summary fetch 或 manual ACP analysis flow。
- Overview、Execution、Activity 不受影响。
- Task JIT fields 和数据库列保持不变。
- Harness API/core、MCP tools 和 Specialist 保持可用。
- Saved/Relevant History Memory 自动 preload 与 Board policy 保持现有行为。
- 已有 snapshot 数据不被删除或迁移。
- JIT-only UI tests、locale keys、imports 和 helpers 被清理，保留能力的测试继续通过。

## Implementation Sequence

1. 运行 `entrix graph impact` 和 `entrix graph test-radius`。
2. 更新 characterization tests，锁定三个目标 Tab 和 `jitContext -> overview` fallback。
3. 从 `kanban-card-detail.tsx` 删除 JIT Tab、panel、props 和 JIT-only helpers。
4. 从 `kanban-tab-panels.tsx` 删除 JIT props、手动 prompt callback 和 History Analysis launcher。
5. 从 `kanban-detail-panels.tsx` 删除 `JitContextPanel`，按真实引用清理 helper/type/import。
6. 清理 JIT UI tests 和 fetch mocks，保留其他 panel tests。
7. 删除确认无其他消费者的 JIT Tab i18n keys；保留 Board policy 和 Agent prompt 文案。
8. 全仓检查 `jitContext`、`JitContextPanel`、`onOpenJitContextHistoryAnalysis`、`onLoadJitContextIntoSession`、`startKanbanHistoryAnalysisSession` 残留。
9. 验证 Harness、MCP、Task persistence、context preload、agent trigger、lane experience 和 Board policy 未被误删。
10. 运行测试、typecheck 和 Entrix 对应 tier。
11. 若未来彻底移除 JIT/History Memory，另起数据/API/自动 preload 清理任务。

## Explicitly Out Of Scope

- 删除 `Task.contextSearchSpec` 或 `Task.jitContextSnapshot`。
- 删除 `tasks.jit_context_snapshot` 或迁移现有数据。
- 删除 Task-Adaptive Harness、transcript parser 或 Feature Surface inference。
- 删除 Harness 或 History Summary API。
- 删除 `history-summary-analyst` Specialist。
- 删除 `save_history_memory_context`、`summarize_task_history_context`、`inspect_transcript_turns` 或 `assemble_task_adaptive_harness`。
- 删除或关闭 Saved/Relevant History Memory 自动 preload。
- 删除 Board `historyMemoryPolicy` 或设置控件。
- 修改 Agent workflow prompts 中的 Relevant History Memory 指令。
- 修改 lane experience、reasoning memory 或 Agent trigger consumers。
- 将 JIT 内容迁移到其他 Tab 或新页面。
- 新增替代的 History Analysis 启动入口。
- 做全产品术语重命名。
- 清除数据库中已有 snapshot/analysis。
