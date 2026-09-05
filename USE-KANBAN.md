# Kanban 使用指南

本文从最终使用者的角度说明 Routa 的 Kanban 模式。重点是如何在桌面应用中准备工作区、关联项目、创建卡片、观察自动工作流，以及如何验证 Agent 是否按预期工作。

## 1. Kanban 是什么

Routa 的 Kanban 不只是一个任务列表。卡片所在的列代表任务当前的工作流阶段。创建卡片或进入新列会触发该列配置的 Agent；Agent 完成当前列后，会通过 `move_card` 自动推进卡片。

- 卡片的真实位置由 `columnId` 表示。
- `status` 是根据列阶段同步出的状态，不是另一套独立的看板分组。
- 创建卡片或进入自动化列会创建 Agent session（前提是该列启用自动化）。
- 列之间的移动会留下工作流事件和任务历史。
- Review、Done 等列可以配置质量门禁，缺少必要证据时不能直接推进。

典型流程是：

```text
Backlog -> Todo -> Dev -> Review -> Done
               \-> Blocked
```

`Blocked` 用于人工处理阻塞问题，默认不是自动化执行列。

## 2. 使用前准备

开始 Kanban 工作前，需要完成以下准备：

1. 启动 Routa 桌面应用。
2. 创建或选择一个 workspace。
3. 配置至少一个可用的 Agent Provider，并完成所需认证。
4. 在当前 workspace 中添加一个本地项目或克隆一个 GitHub 项目。
5. 确认项目是可访问的 Git 仓库，并且 Routa 对它有读写权限。

首次验证建议使用一个小型项目，例如 `sample-todo`，并单独创建一个 `kanban-demo` workspace。这样不会把验证任务混入其他真实工作。

## 3. 创建 workspace 和添加项目

### 创建 workspace

在首页或 workspace 切换菜单中选择“新建 workspace”，输入名称并创建。创建后，Routa 会将它作为当前工作区。

### 添加项目

有两种常见入口：

- 首页配置区的“代码库”卡片
- 当前 workspace 的“设置 -> 关联仓库”

在仓库选择器中可以：

- 选择本地已有的 Git 项目目录
- 输入或选择 GitHub 仓库并执行克隆

保存后，项目就成为当前 workspace 的一个 codebase。这个操作只是登记项目路径和相关元数据，不会把本地项目复制进 Routa。

### 一个 workspace 可以有多个项目

可以。例如：

```text
电商项目 workspace
  - shop-web
  - shop-api
  - shop-infra
```

添加多个项目后，Kanban 页面顶部的仓库控制区域可以切换当前查看的项目。创建卡片时，卡片表单中的“关联仓库”区域可以选择一个或多个 codebase。

如果卡片没有明确选择仓库，当前产品语义可能会将 workspace 下的所有关联仓库作为任务上下文。因此，执行真实任务时应明确选择目标仓库。

多个 Agent 不应同时直接修改同一个物理目录。需要并行工作时，应使用独立 Git worktree；否则应让任务在同一目录中串行执行。

## 4. 创建一张 Kanban 卡片

进入当前 workspace 的 Kanban 页面，选择创建任务。卡片应尽量在开始时写清楚：

- 任务标题
- 目标和范围
- 验收标准
- 验证命令
- 测试场景
- 优先级和标签
- 关联的仓库

不要只写一句“把这个功能做出来”。Kanban 的后续列会根据卡片内容判断是否具备执行和交付条件。

## 5. 每个列应该做什么

| 列 | 使用者期望 | 典型结果 |
| --- | --- | --- |
| Backlog | 澄清需求，不急于写代码 | 范围、验收标准、验证计划完整 |
| Todo | 确认依赖和执行上下文 | 目标仓库、Provider、worktree 已明确 |
| Dev | 实现功能并运行测试 | 代码变更、测试结果、Git commit |
| Review | 检查质量和验收证据 | QA 结果、截图、测试证据、评审结论 |
| Done | 确认可以交付 | 提交状态、工作区状态、交付报告 |
| Blocked | 记录阻塞并等待处理 | 阻塞原因、恢复条件或 handoff 记录 |

### Backlog

Backlog Agent 的主要职责是整理需求，而不是直接开始大范围开发。它应确认：

- 任务范围是什么
- 验收标准是什么
- 如何验证
- 是否存在依赖或风险

完成后，Backlog Agent 会自动调用 `move_card` 进入 Todo。推荐配置还将 Backlog 的 `autoAdvanceOnSuccess` 设为 `true`，作为成功完成后的系统兜底推进。使用者不需要拖动卡片，只需检查卡片位置和 session 状态。

### Todo

Todo 阶段用于准备执行：确认仓库、分支、依赖、worktree 和执行角色。Todo Agent 完成后会自动调用 `move_card` 进入 Dev。

### Dev

Dev Agent 负责修改代码、运行验证并形成可审查的变更。进入 Review 前，应至少完成：

- 代码已修改
- 测试已运行
- 验证结果已记录
- 变更已提交，或已明确说明为什么不能提交

从 Dev 开始，任务描述通常应保持稳定，后续进展使用 comment、artifact 或验证报告记录。

### Review

Review 可能包含多个质量检查步骤，例如 QA 和 Review Guard。当前推荐配置通常要求测试结果和截图等证据。缺少这些证据时，Agent 的 `move_card` 调用可能被服务端门禁阻止，或留下明确的 warning，取决于列配置。

### Done

Done 表示满足交付条件，而不只是 Agent 说“完成了”。如果列配置了对应策略，还需要满足：

- 已提交代码
- worktree 状态符合要求
- 验收标准全部通过
- 分支具备交付或 PR-ready 条件

## 6. 推荐使用案例

使用一个已经添加到 `kanban-demo` workspace 的小型 `sample-todo` 项目，完成下面的需求：

> 增加 `GET /health` 接口。接口返回 HTTP 200，并返回 `{ "status": "ok" }`。为接口增加自动化测试，并提供一次运行验证的证据。

### 操作步骤

1. 在 Kanban 页面创建卡片，关联 `sample-todo`。
2. 把需求、验收标准和验证命令写入卡片。创建后不要手动拖动卡片。
3. 观察 Backlog Agent 是否自动启动并补全任务信息，然后自动推进到 Todo。
4. 观察 Todo Agent 是否自动确认执行仓库和依赖上下文，并自动推进到 Dev。
5. 等待 Dev Agent 自动实现接口、增加测试并提交变更，然后观察它是否自动推进到 Review。
6. 检查 Review Agent 的测试结果、截图和评审结论；门禁通过后观察卡片是否自动进入 Done。
7. 打开卡片详情，确认列位置、任务状态、lane session、变更记录和验证证据都能追溯。

### 验收标准示例

- `GET /health` 返回 HTTP 200。
- 响应内容包含 `status=ok`。
- 自动化测试覆盖成功响应。
- Git 变更属于 `sample-todo`，没有修改其他仓库。
- Review 阶段能看到测试结果和必要的截图证据。
- Done 阶段的提交和 worktree 状态符合当前列策略。

## 7. Rust-only 桌面运行面的注意事项

在最终 Tauri 桌面运行面中，Rust/Axum 负责 Kanban API、卡片创建和列变更、列进入时的 Agent 触发、任务持久化和 ACP 持久化。默认流程也是自动的：创建卡片后触发当前列 Agent，Agent 在当前列完成后调用 `move_card` 推进，下一列 Agent 随后自动启动。

推荐配置中只有 Backlog 的 `autoAdvanceOnSuccess` 默认开启；Todo、Dev、Review、Done 主要由当前 lane Agent 显式调用 `move_card` 推进。这不是人工流程，也不是关闭自动化，而是避免 Agent 推进和系统完成事件重复推进。

验证自动推进时，使用者只创建 Issue，然后观察：

```text
创建 Issue
  -> 当前列自动化 session
  -> Agent 调用 move_card
  -> 下一列自动化 session
  -> ...
  -> Done
```

使用者只在门禁拒绝、Agent 失败/超时或需要进入 `Blocked` 时人工介入。不要通过手动拖动卡片来代替自动推进验证。

## 8. 如何判断验证成功

不要只看卡片是否换列。建议同时检查：

1. **状态**：`columnId`、`status` 和列阶段是否一致。
2. **自动化**：进入自动化列后，是否创建了正确的 Provider、role 或 specialist session。
3. **门禁**：缺少测试结果、截图、提交或必要字段时，是否被正确阻止。
4. **追溯**：卡片详情中是否能看到 session、comment、artifact、变更和验证记录。
5. **恢复**：Agent 失败、超时或应用重启后，任务是否仍能恢复；进入 Blocked 后是否不会误启动自动化。

### 终态与 Blocked 分支检查

手工验证自动推进时，还要确认正常流程和异常旁路没有被混成一条线性序列：

- 正常流程应为 `Backlog -> Todo -> Dev -> Review -> Done`。
- `Done` 是终止列。Done Agent 完成后不应再计算下一列，也不应继续调用 `move_card`。
- `Blocked` 保留为看板中的人工处理列，但不应作为 `Done` 的下一个列或正常 successor。
- Agent 失败、超时、门禁拒绝、缺少依赖或需要人工暂停时，才应进入 `Blocked`。
- 进入 `Blocked` 时，应能看到阻塞原因和恢复条件；`Blocked` 列不应自动启动新的 Agent。
- 从 `Blocked` 恢复时，应显式指定目标列，不应依赖列的视觉顺序推断恢复位置。

本项的关键验收结果是：正常完成的卡片停在 `Done` 并同步为 `COMPLETED`；只有存在明确阻塞证据的卡片才停在 `Blocked` 并同步为 `BLOCKED`。

## 9. workspace 的清理和删除

当前用户界面没有提供删除 workspace 的按钮。

可以在“Workspace 设置 -> 关联仓库”中移除仓库，但这只会解除 codebase 关联，不会删除 workspace 下的任务、会话、看板或 artifact，也不是“一键清空”。

当前行为可以概括为：

```text
可以创建 workspace
可以切换 workspace
可以添加或移除关联仓库
可以暂时停用某个 workspace
暂时不能在界面中永久删除 workspace
```

因此，功能验证最好使用专用的临时 workspace，例如 `kanban-demo`，验证完成后保留它或移除其中的仓库。永久删除属于破坏性操作，需要产品提供明确的确认、归档和恢复流程后再开放。

## 10. 常见问题

### 页面一开始显示没有 board

首屏加载期间可能暂时显示空状态。先等待 Rust 服务和数据库初始化完成，再确认 Kanban 页面是否加载默认 board。应区分加载中、请求失败和确实没有 board 三种情况。

### 卡片进入列后没有启动 Agent

检查当前列是否启用了自动化、Provider 是否已认证、仓库路径是否有效，以及任务是否已经有正在运行的 lane session。`Blocked` 列默认是人工列，不会自动启动 Agent。

### 多张卡片同时进入 Dev

Kanban 会使用队列和并发限制控制 Agent 数量。即使界面上有多张卡片，也不意味着它们会无限并行执行。若任务修改同一个目录，必须使用 worktree 或改为串行。
