# Kanban Card / TaskDetail 执行状态调查记录

> 记录时间：2026-09-12
> 
> 目的：记录 Kanban Card 封面与 TaskDetail 对 Agent 执行状态、Run/Rerun 按钮和运行历史的现状调查，作为后续 UI 简化讨论的备份。
> 
> 本记录只描述现状和分析。本轮没有修改业务代码，也没有决定最终的状态模型。

## 一、调查范围

重点检查了以下内容：

- Card 封面上的状态徽标和 Run/Rerun 按钮。
- 点开 Card 后，TaskDetail 的 Execution 区。
- TaskDetail 的 Runs / Activity 区。
- Task、Lane Session、ACP Session 三类状态之间的关系。
- workspace `arknights` 中 TASK-300 的实际数据。

相关实现：

- `src/app/workspace/[workspaceId]/kanban/kanban-card.tsx`
- `src/app/workspace/[workspaceId]/kanban/kanban-card-detail.tsx`
- `src/app/workspace/[workspaceId]/kanban/kanban-card-activity.tsx`
- `src/core/kanban/effective-task-automation.ts`
- `src/core/task-run-ledger.ts`
- `crates/routa-server/src/application/tasks.rs`
- `crates/routa-server/src/api/tasks_automation.rs`
- `crates/routa-server/src/api/kanban.rs`

## 二、Run / Rerun 的实际功能

Card 封面和 TaskDetail 最终都调用同一个 `onRetryTrigger(task.id)`。

后端收到 `retryTrigger: true` 后会：

1. 清除旧的 `triggerSessionId`。
2. 创建新的 Agent Session。
3. 使用当前 Lane/Card 的自动化配置启动 Agent。
4. 将新 Session 写入 `sessionIds` 和 `laneSessions`。
5. 将新 Session 设置为当前 `triggerSessionId`。

它不是恢复旧 Session，也不负责移动 Card、完成 Card 或直接执行测试。

因此两个名称表达的本意是：

- `Run`：首次手动启动当前 Card 的 Agent 自动化。
- `Rerun`：再次创建一个 Agent Session 执行当前 Card。

更准确的功能描述应接近 `Start Agent` / `Start New Agent`。只有在系统明确知道上一次运行失败时，`Retry Agent` 才是准确说法。

## 三、Card 封面的现状

Card 封面的状态徽标由 `linkedSession?.acpStatus` 和队列状态生成：

- `connecting` -> `STARTING`
- `ready` -> `LIVE`
- `error` -> `FAILED`
- 无状态 -> `IDLE`
- 有队列位置 -> `QUEUED #n`

封面按钮的显示逻辑：

```ts
canRetry = effectiveAutomation.canRun && (
  sessionStatus === "error"
  || (!task.triggerSessionId && task.columnId === "dev")
) && !queuePosition;

canRun = effectiveAutomation.canRun
  && !task.triggerSessionId
  && task.columnId !== "done"
  && !queuePosition;
```

封面按钮因此只有在以下条件下出现：

- 当前 Lane 或 Card 有可执行的自动化配置。
- Card 不在队列中。
- `Run` 还要求没有 `triggerSessionId`，且不在 `done`。
- `Rerun` 还要求 ACP 状态为 `error`，或者位于 `dev` 且没有 `triggerSessionId`。

封面不会显示按钮的常见情况：

- 已有 `triggerSessionId`。
- Card 正在排队。
- 没有可解析的 Lane/Card 自动化配置。
- Card 已在 `done`。
- Session 不是明确的 `error`，但已有残留 Session ID。

封面存在一个根本问题：它把 `triggerSessionId` 当作“仍有活动 Session”的充分条件，但状态徽标又只看 ACP 状态。因此可能出现：

```text
Task 有 triggerSessionId
ACP Session 没有有效 acpStatus
Card 显示 IDLE
Run 和 Rerun 都不显示
```

## 四、TaskDetail Execution 区的现状

TaskDetail 的 Execution 区使用更宽松的判断：

```ts
canRunTask = effectiveAutomation.canRun && task.columnId !== "done";
```

它不检查：

- `triggerSessionId` 是否存在。
- Session 是否真实运行。
- Session 是否已经失败。
- Card 是否排队。

所以只要当前 Lane 可执行，且不在 `done`，详情页就可能显示执行按钮。

TaskDetail 的按钮文案为：

```ts
needsLiveRunRecovery
  ? "Recover Live Run"
  : hasRecordedRuns
    ? "Rerun"
    : "Run";
```

这里的 `hasRecordedRuns` 只表示 `sessionIds` 中存在历史记录，不表示最近一次运行失败，也不表示当前 Session 已停止。因此：

- 从未启动过 -> `Run`
- 只要历史上启动过 -> `Rerun`
- 特定的 embedded ACP 恢复错误 -> `Recover Live Run`

这导致详情页和封面可能同时出现以下不一致：

- 封面没有按钮，详情页有按钮。
- 封面显示 `IDLE`，详情页按钮显示 `Rerun`。
- 详情页显示 `Rerun`，但最近一次运行其实仍是 `running`。

详情页只有在以下情况才显示失败提示：

- `sessionInfo.acpStatus === "error"` 且存在 `acpError`。
- 或 Task 上有 `lastSyncError`。

如果 Session 已停止、状态为空、运行记录残留为 `running`，但没有错误字段，详情页不会给出失败或停止原因。

详情页另外会展示：

- Lane Pipeline。
- Current Run 的 Provider、Role、Specialist 和 Transport。
- Card Session Override。
- Fallback Agent Chain。
- 当前/下一阶段所需的 Gate Artifacts。
- Evidence 状态，例如 `Review blocked`。

但这些信息目前没有统一驱动执行按钮的状态或文案。

## 五、TaskDetail Runs / Activity 区的现状

Runs Tab 会读取运行台账，并展示每次历史运行：

- `Running`
- `Completed`
- `Failed`
- `Timed out`
- `Transitioned`
- `Unknown`

每次运行可以显示：

- Run 序号。
- 类型：ACP、Runner ACP 或 A2A。
- 所属 Lane。
- Transport。
- Step / Specialist。
- Provider / Role。
- Session ID 或外部任务 ID。
- 工作目录或 A2A Context。
- 当前运行标志。

用户可以点击历史项切换到对应 Session，但 Runs Tab 没有提供：

- 停止 Session。
- 继续旧 Session。
- 标记为卡住。
- 将残留的 `running` 修正为停止/超时。
- 根据 Gate 阻塞原因决定下一步动作。

运行状态的推导同时使用 `laneSession.status` 和 ACP Session 状态。当两者不一致时，UI 可能继续显示 `Running`，或者退化为 `Unknown`，但没有明确的 `Stopped` / `Stuck` 状态。

## 六、TASK-300 实际案例

查询接口：workspace `arknights`，Card 标题为 `TASK-300：实现并本地验证 ActionsRuntime`。

实际数据：

- `columnId`: `dev`
- `status`: `IN_PROGRESS`
- Dev Lane automation: `enabled: true`
- `triggerSessionId`: `47112b07-74b6-4099-999f-3f3ce67c36ae`
- 当前 Lane Session: `running`
- Session `acpStatus`: `null`
- Session `continuityStatus`: `restorable`
- Session transcript: 空
- `lastSyncError`: `null`
- 运行历史总数：6
- 当前 Evidence：缺少 `screenshot`，因此 `requiredSatisfied: false`

因此当前界面逻辑会产生：

### Card 封面

```text
状态徽标：IDLE
Run：不显示，因为 triggerSessionId 存在
Rerun：不显示，因为 acpStatus 不是 error，且 triggerSessionId 存在
```

### TaskDetail Execution

```text
执行按钮：显示
按钮文案：Rerun，因为有历史 Session
失败说明：不显示，因为没有 acpError，也没有 lastSyncError
```

### TaskDetail Runs

```text
最新运行：Running
当前 Session：可被选中
运行历史：可查看 6 次
```

### Evidence / Gate

```text
Review blocked
缺少 screenshot 工件
```

但 Gate 阻塞没有改变 Execution 区的按钮文案，也没有产生 `Needs attention` 或类似状态。

## 七、核心问题总结

当前实际上有三套状态来源：

1. **Task 状态**：`status`、`columnId`、`triggerSessionId`。
2. **Lane 运行记录**：`laneSessions[].status`，例如 `running`、`timed_out`、`transitioned`。
3. **ACP Session 状态**：`acpStatus`，例如 `connecting`、`ready`、`error` 或空。

它们没有形成一个统一的“当前执行状态”。特别是：

- `triggerSessionId` 存在不等于 Session 仍在运行。
- `acpStatus` 为空不等于 Agent 已正常结束。
- `laneSession.status = running` 不等于运行时仍然活跃。
- 有历史运行不等于应该显示 `Rerun`。
- Evidence / Gate 阻塞不等于 Agent 失败，但当前 UI 没有把它作为独立的注意状态处理。

因此用户可能看到：

```text
Card：IDLE
TaskDetail：Rerun
Runs：Running
Session：无实际活动状态
Evidence：Review blocked
```

这正是“看不出来是停了，还是卡了”的根本原因。

## 八、对 Run / Rerun 的判断

当前 `Rerun` 把以下不同情况混在了一起：

- 上一次明确失败。
- Session 已停止但没有错误信息。
- Session 信息可恢复。
- 运行记录残留为 `running`。
- Agent 已完成工作，但因为 Gate 缺少工件无法流转。
- 用户只是想再次创建一个新的 Agent Session。

这些情况的用户动作并不相同：

- 明确失败：`Retry Agent`。
- 可恢复的旧 Session：`Continue Agent`。
- 已停止且不可恢复：`Start New Agent`。
- Gate 阻塞：先处理缺少的证据/工件，不应默认再次运行 Agent。
- 仍在运行：不应提供新的启动按钮，应该提供查看当前 Session。

所以当前 `Rerun` 不是一个可靠的状态名称，而只是“再次创建新 Session”的历史遗留文案。

## 九、当前简化讨论的建议边界

本记录不直接决定实现方案。后续若从简处理，至少需要先明确以下原则：

1. TaskDetail 是执行操作的唯一入口，Card 封面不承担复杂的运行控制逻辑。
2. 不把“有历史记录”直接等同于 `Rerun`。
3. 不把 `triggerSessionId` 直接等同于“正在运行”。
4. 对无法确认的状态，显示一个明确的未知/需处理状态，不能继续伪装成 `IDLE` 或 `Running`。
5. Gate 阻塞、Agent 失败、Session 停止和 Session 仍运行应至少在语义上区分。
6. 如果暂时不重构后端状态模型，按钮应被视为“启动一个新的 Agent Session”，而不是“重试当前运行”。

## 十、阶段性补充：Lane、Step、Session 与右侧 Tab

本节记录后续对 Detail 弹窗右侧 Session 展示的代码复查结果。讨论范围仍然是 Task 本身的执行记录，不把 Column 位置、Task.status、Evidence 或 Gate 判定混作执行状态。

### 10.1 一个 Lane 可以包含多个 Step，laneSession 是 Step 的执行记录

Lane 的自动化配置有有序的 `steps` 数组，语义是“在同一个 Lane 内按顺序执行的自动化步骤”。

真实测试场景：`task-steps-1` 从 Backlog 进入 Todo Lane 后：

1. 第一个 Session 执行 `triage`，`stepIndex: 0`，Session 为 `session-todo-1`。
2. 第一个 Session 完成后，Task 仍在 Todo Lane。
3. 编排器使用同一个 `columnId: "todo"` 创建第二个 Session，执行 `plan`，`stepIndex: 1`，Session 为 `session-todo-2`。

因此真实关系是：

```text
Todo Lane
  Step 1: triage -> Session todo-1
  Step 2: plan   -> Session todo-2
```

从实际业务语义看，`laneSession` 就是某个 Step 的一次执行记录：它以 `sessionId` 标识实际运行的 Session，同时保存该 Session 所属 Lane 的 `columnId/columnName` 和所执行 Step 的 `stepId/stepIndex/stepName`。

正常路径可以理解为：

```text
一个 Step 执行 -> 一个 Session -> 一个 laneSession
```

因此在理想执行场景下，Step、Session、laneSession 是一一对应的。严格从历史数据看，这不是永久的唯一约束：未执行的 Step 没有 laneSession；同一个 Step 在恢复或重试时可能产生新的 Session 和新的 laneSession。

### 10.2 右侧顶部 Tab 的真实单位和文案

右侧顶部按钮是按 `orderedSessionIds` 遍历生成的，因此：

```text
一个 Tab = 一个 laneSession（即一个 Step 的一次 Session 执行记录）
Tab 文案 = 该记录的 Lane 文本 + Step 文本
```

Tab 不是按 Lane 去重的导航，也不是只展示 Lane 的导航。一个 Lane 内的每个 Step 执行记录都保留为独立 Tab。例如：

```text
Backlog · step1
Backlog · step2
Todo · step1
Dev · step1
Dev · step2
Dev · step3
```

目标设计移除 Lane 图标、计算序号和状态 Pill，使用文字 Tab。Lane 取 `columnName`，Step 取 `stepName`。`stepName` 缺失时不把 `stepId` 或计算出的 `Step N` 自动变成用户可见文案；应保留真实可用文本或明确处理为空的情况。

点击 Tab 后，下方选中信息区继续显示该 Session 的详细元数据。Tab 的 `title` 和无障碍描述可以包含完整的 Session 标识，但不改变可见标签的 Lane + Step 语义。

相关实现：

- `src/app/workspace/[workspaceId]/kanban/kanban-card-activity.tsx`
- `src/app/workspace/[workspaceId]/kanban/kanban-card-session-utils.ts`
- `src/core/kanban/workflow-orchestrator.ts`

### 10.3 `selectedRunId` 与 `selectedLaneSession`

`selectedRunId` 虽然变量名包含 `Run`，实际保存的是当前选中 Tab 对应的 Session ID：

1. 如果传入的 `currentSessionId` 在 Tab 列表中，优先选它。
2. 否则选择 `orderedSessionIds` 的最后一个。

随后使用同一个 ID 查找两个对象：

```text
selectedLaneSession = task.laneSessions 中 sessionId 匹配的记录
selectedRun         = Run ledger 中 sessionId / id 匹配的记录
```

因此 `selectedLaneSession` 不是独立数据源，而是从 `task.laneSessions` 选出的当前 Session 历史记录。如果 Tab 来源于 Run ledger、但没有对应的 `laneSessions` 条目，`selectedLaneSession` 可以是 `undefined`。

### 10.4 一条 `TaskLaneSession` 的字段

核心模型字段如下：

```text
sessionId

columnId / columnName
stepId / stepIndex / stepName

provider / role / specialistId / specialistName
transport

routaAgentId / worktreeId / cwd
externalTaskId / contextId

attempt / loopMode / completionRequirement / objective
lastActivityAt
recoveredFromSessionId / recoveryReason

status / startedAt / completedAt
```

其中：

- `columnId`、`columnName` 表示该 Session 所属 Lane。
- `stepId`、`stepIndex`、`stepName` 表示该 Session 执行的是 Lane 内哪个 Step。
- `transport` 表示 `acp` 或 `a2a` 等传输方式。
- `status` 是 LaneSession 自身的生命周期状态，例如 `running`、`completed`、`failed`、`timed_out`、`transitioned`。

### 10.5 Tab 下方标签行的字段映射

例如页面 DOM 中看到：

```text
Backlog / acp / Completed / Backlog Refiner / completed
```

它实际由五个独立渲染分支组成：

| UI 标签 | 数据来源 | 含义 |
| --- | --- | --- |
| `Backlog` | `selectedLaneSession.columnName` | 当前 Session 所属 Lane |
| `acp` | `selectedLaneSession.transport` | Session 的传输协议 |
| 第一个 `Completed` | `selectedRun.status` | Run ledger 的执行状态 |
| `Backlog Refiner` | `selectedLaneSession.stepName` | 当前 Session 的 Step 名称 |
| 第二个 `completed` | `selectedLaneSession.status` | LaneSession 自身状态 |

两个完成状态来自不同对象，但正常情况下通常相同，因为 Run ledger 的状态主要由 `laneSession.status` 推导。当前 UI 因此可能显示两个语义重复的 `Completed` 标签。页面视觉上的大写来自 CSS `uppercase`，不代表底层字段一定是大写。

### 10.6 当前阶段的简化模型

```text
Lane
  -> 有序 Step
  -> 每次 Step 执行产生一个 Session
  -> 每个 Session 进入 laneSessions 历史
  -> 每个 laneSession 对应一个右侧 Tab
```

因此，Lane 是执行所属的上下文，Step 是 Lane 内的执行定义，`laneSession` 是 Step 的一次执行记录。右侧 Tab 的数据实体是 `laneSession/Session`，但其业务显示语义是 `Lane + Step`。

### 10.7 右侧顶部 Tab 改造方案

只改右侧顶部的 `laneSession` 导航，不改变 `laneSessions` 数据模型、Session 创建流程、历史顺序或下方详情。

1. 使用真正的 Tabs 语义：容器为 `role="tablist"`，每个条目为 `role="tab"`，使用 `aria-selected`、`aria-controls` 和 `tabIndex`。
2. 复用项目现有的底部边框 Tab 设计：`border-b` 分隔线、`border-b-2` 选中指示、`desktop-border`、`desktop-bg-active`、`desktop-text-primary/secondary`、`desktop-accent`。
3. 可见文本固定为 Lane + Step：`columnName · stepName`。不使用 Icon Font，不显示 `index + 1`，不使用 `stepId` 或 `Step N` 作为未经确认的 fallback。
4. 保留 `orderedSessionIds` 的实际历史顺序。默认不加入 `Backlog > Todo` 箭头；项目没有 Tab 专用箭头 token，且箭头会把 Session 历史误读为 Lane 面包屑。
5. Tab 选中态只表达“当前选中的 Session”，不再用 `running/completed/failed` 状态色改变整个 Tab 的外观；执行状态继续在下方详情区表达。

### 10.8 Session、Step 与 Run ledger 的关系澄清

本节进一步收敛“一个 Step 到底对应多少个 Session”和“Run ledger 是否是权威数据源”的语义。讨论范围是已经触发自动化的 Step 执行，不包含未执行或人工执行场景。

#### 10.8.1 一个已触发的 Step 会产生一个 Session

对一次具体的 Step 执行，系统会创建一个新的 Agent Session，并用新的 `sessionId` 记录这次执行：

```text
一次 Step 执行 -> 一个 Session
```

但这不意味着一个 Step 定义在整个生命周期内只能有一个 Session。相同 Step 再次执行、恢复或重试时，会创建新的 Session。准确基数关系是：

```text
Lane 1 -> N Step 定义
Step 定义 1 -> N Session 执行记录
一次 Step 执行 1 -> 1 Session
```

新 Session 不会替换旧 Session。TypeScript 的 `upsertTaskLaneSession` 只按 `sessionId` 查找既有记录：相同 `sessionId` 才会原地更新，不同 `sessionId` 会追加到 `task.laneSessions`。旧 Session 因此保留在历史中，新 Session 成为另一条历史记录。

#### 10.8.2 “当前 Session”是 Task 级指针，不是 Step 级字段

Task 只有一个 `triggerSessionId`，它表示当前任务正在触发或关联的 Session。`sessionIds` 保存任务关联的 Session ID 历史，`laneSessions` 保存带 Lane/Step 元数据的持久化执行历史。

模型没有为每个 Step 单独保存 `currentSessionId`。因此不能说“每个 Step 始终拥有一个 Current Session”；更准确的表达是：

```text
Step 的历史：task.laneSessions 中所有 stepId/stepIndex 对应的记录
Task 的当前指针：task.triggerSessionId
```

例如同一 Step 先后执行三次，历史中可以有三个 Session；只有其中一个可能被 Task 的 `triggerSessionId` 指向为当前 Session。旧记录仍然保留，不会被替换。

#### 10.8.3 `task.laneSessions` 的顺序不是严格时间排序契约

新增记录通常通过 `push` 放到 `task.laneSessions` 末尾，因此在正常串行自动化中，数组顺序通常等于 Session 创建顺序，也通常接近实际发生顺序。但模型没有声明该数组按 `startedAt` 严格排序：已有记录更新不会重新排序，导入、恢复或其他写入路径也可能影响插入顺序。

当前顶部 Session 导航使用 `getOrderedSessionIds(task)`，优先保留 `task.laneSessions` 的数组顺序。顶部显示的 `1/2/3` 是该列表中的位置，不是 `stepIndex`、`attempt` 或基于时间计算的序号。

#### 10.8.4 Run ledger 的定义和权威性边界

`Run ledger`（运行台账）是面向 API/UI 的运行历史读模型，不是另一种 Session，也不是独立的持久化实体。它在请求或构建时由以下信息整理得到：

```text
TaskLaneSession 历史记录
    + ACP Session 的补充元数据
    -> TaskRunInfo / TaskRunLedgerEntry
```

Run ledger 统一提供：

```text
id / sessionId
kind: embedded_acp | runner_acp | a2a_task
status: running | completed | failed | timed_out | transitioned | unknown
columnId / stepId / stepName
provider / specialistName
startedAt / completedAt
resumeTarget
```

TypeScript 实现位于 `src/core/task-run-ledger.ts`；Rust 的任务 Runs API 在 `crates/routa-server/src/api/tasks/evidence.rs` 中从 `task.lane_sessions` 动态构建同类结果。Run ledger 会按 `startedAt` 排序（通常最新在前），而不是复用 `task.laneSessions` 的原始数组顺序。

因此应区分三种角色：

```text
TaskLaneSession  = 权威的持久化执行历史及 Lane/Step 归属
Task.triggerSessionId = Task 级当前 Session 指针
Run ledger       = 运行历史的整理/排序展示投影
```

Run ledger 更适合作为 Runs UI、状态摘要和历史排序的数据源，但不能取代 `TaskLaneSession` 作为领域事实，也不能取代 `triggerSessionId` 判断 Task 当前指向的 Session。当前右侧上部的 Session Tab 如需表达历史实体，应以 `laneSession/session` 为实体；Run ledger 可用于补充状态和排序信息。
