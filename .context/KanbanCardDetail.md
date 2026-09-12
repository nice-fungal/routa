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

