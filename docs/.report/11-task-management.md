# 任务管理系统 - TodoWrite 与 Tasks 双轨架构

> 本报告将 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/tools/task-management) 中描述的双轨任务系统，映射到 `claw-code`（Claude Code 的重写实现）源码层。所有源码路径均指向 `packages/ccb`。
> 文末附 [源码索引](#源码索引)。

---

## 双轨架构：TodoWrite V1 与 Tasks V2

Claude Code 的任务管理不是一个单一系统，而是两个并存、按运行模式切换的实现：

| 维度 | V1: TodoWrite | V2: TaskCreate / TaskUpdate / TaskList / TaskGet |
|------|---------------|---------------------------------------------------|
| 启用条件 | 非交互式（pipe/SDK）或 `isTodoV2Enabled()` 返回 false | 交互式 REPL（默认）或 `CLAUDE_CODE_ENABLE_TASKS=1` |
| 存储 | 内存中 `AppState.todos[sessionId]`（Zustand store） | 文件系统 `~/.claude/tasks/<taskListId>/<id>.json` |
| 数据模型 | `{content, status, activeForm}` — 扁平三元组 | `{id, subject, description, activeForm, owner, status, blocks[], blockedBy[], metadata}` — 完整实体 |
| 持久化 | 进程退出即丢失 | 跨进程存活，支持多 Agent 并发访问 |
| 并发安全 | 无（单会话单写者） | 文件锁 + 高水位标记 + TOCTOU 防护 |

切换逻辑位于 `isTodoV2Enabled()`（[`packages/ccb/src/utils/tasks.ts#L133-L139`](/packages/ccb/src/utils/tasks.ts#L133-L139)）：

```typescript
export function isTodoV2Enabled(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_ENABLE_TASKS)) {
    return true
  }
  return !getIsNonInteractiveSession()
}
```

交互式会话默认启用 V2，SDK/pipe 模式回落 V1。两者互斥 —— `TodoWriteTool.isEnabled` 返回 `!isTodoV2Enabled()`（[`packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts#L52-L54`](/packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts#L52-L54)），而 `TaskCreateTool.isEnabled` 返回 `isTodoV2Enabled()`（[`packages/ccb/src/tools/TaskCreateTool/TaskCreateTool.ts#L68-L70`](/packages/ccb/src/tools/TaskCreateTool/TaskCreateTool.ts#L68-L70)）。

---

## V1：TodoWrite 的极简设计

### 全量替换与智能清空

`TodoWrite` 本质是一个全量替换操作 —— 每次调用传入完整的 `todos[]` 数组，完全覆盖之前的状态：

```typescript
async call({ todos }, context) {
  const todoKey = context.agentId ?? getSessionId()
  const oldTodos = appState.todos[todoKey] ?? []
  const allDone = todos.every(_ => _.status === 'completed')
  const newTodos = allDone ? [] : todos
  // ...
}
```

参见 [`packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts#L65-L70`](/packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts#L65-L70)。当所有任务都 `completed` 时，`newTodos` 被设为空数组（而非保留 completed 列表）。这确保 UI 上不会有"已完成"的视觉噪音。

### 验证推动（Verification Nudge）

V1 包含一个验证推动机制：当主线程 Agent 完成 3+ 个任务且没有任何一个是验证步骤时，系统在 `tool_result` 中追加提示，催促 Agent 派生验证子 Agent：

```typescript
if (
  feature('VERIFICATION_AGENT') &&
  getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false) &&
  !context.agentId &&
  allDone &&
  todos.length >= 3 &&
  !todos.some(t => /verif/i.test(t.content))
) {
  verificationNudgeNeeded = true
}
```

参见 [`packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts#L76-L86`](/packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts#L76-L86)。最终追加的提示文本见 [`L104-L108`](/packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts#L104-L108)。这是防止 Agent "自说自话地宣布完成"的防御性设计 —— 通过结构性推动而非硬约束。

---

## V2：文件系统持久化的任务系统

### 数据模型

每个任务是一个独立 JSON 文件，路径为 `~/.claude/tasks/<taskListId>/<id>.json`。`TaskSchema` 定义在 [`packages/ccb/src/utils/tasks.ts#L76-L89`](/packages/ccb/src/utils/tasks.ts#L76-L89)：

```typescript
export const TaskSchema = lazySchema(() =>
  z.object({
    id: z.string(),
    subject: z.string(),
    description: z.string(),
    activeForm: z.string().optional(), // "Running tests"
    owner: z.string().optional(), // agent ID
    status: TaskStatusSchema(), // "pending" | "in_progress" | "completed"
    blocks: z.array(z.string()), // 此任务阻塞哪些任务 ID
    blockedBy: z.array(z.string()), // 哪些任务 ID 阻塞此任务
    metadata: z.record(z.string(), z.unknown()).optional(),
  }),
)
```

### 任务列表 ID 的解析优先级

`getTaskListId()` 按 5 级优先级解析任务归属，位于 [`packages/ccb/src/utils/tasks.ts#L199-L210`](/packages/ccb/src/utils/tasks.ts#L199-L210)：

1. `CLAUDE_CODE_TASK_LIST_ID` 环境变量（显式覆盖）
2. 进程内 teammate 上下文的 `teamName`（共享 leader 的任务列表）
3. `CLAUDE_CODE_TEAM_NAME` 环境变量（进程级 teammate）
4. Leader 通过 `setLeaderTeamName()` 设置的 `teamName`
5. `getSessionId()`（独立会话的兜底）

这意味着多 Agent 团队模式下，所有 teammate 自动共享同一个任务列表，无需额外协调。

### ID 分配与高水位标记

任务 ID 是简单的递增整数，但在并发场景下需要防止竞争。`createTask()` 在 [`packages/ccb/src/utils/tasks.ts#L284-L308`](/packages/ccb/src/utils/tasks.ts#L284-L308) 中实现：

```typescript
export async function createTask(
  taskListId: string,
  taskData: Omit<Task, 'id'>,
): Promise<string> {
  const lockPath = await ensureTaskListLockFile(taskListId)
  let release: (() => Promise<void>) | undefined
  try {
    release = await lockfile.lock(lockPath, LOCK_OPTIONS)
    const highestId = await findHighestTaskId(taskListId)
    const id = String(highestId + 1)
    const task: Task = { id, ...taskData }
    const path = getTaskPath(taskListId, id)
    await writeFile(path, jsonStringify(task, null, 2))
    notifyTasksUpdated()
    return id
  } finally {
    if (release) { await release() }
  }
}
```

锁配置 `LOCK_OPTIONS` 在 [`L102-L108`](/packages/ccb/src/utils/tasks.ts#L102-L108) 定义：指数退避重试 30 次（总计约 2.6 秒），适配 10+ 并发 Agent 的 swarm 场景：

```typescript
const LOCK_OPTIONS = {
  retries: { retries: 30, minTimeout: 5, maxTimeout: 100 },
}
```

高水位标记文件 `.highwatermark` 在 [`L92`](/packages/ccb/src/utils/tasks.ts#L92) 定义，确保删除任务后 ID 不会被重用 —— 即使任务 #5 被删除，下一个新建任务仍然是 #6。相关读写逻辑见 [`L110-L131`](/packages/ccb/src/utils/tasks.ts#L110-L131)。

### 依赖管理：blocks / blockedBy

任务间的依赖通过双向链表式的 `blocks` / `blockedBy` 字段实现。`blockTask()` 在 [`packages/ccb/src/utils/tasks.ts#L458-L486`](/packages/ccb/src/utils/tasks.ts#L458-L486) 中同时维护两端：

```typescript
export async function blockTask(
  taskListId: string,
  fromTaskId: string,
  toTaskId: string,
): Promise<boolean> {
  const [fromTask, toTask] = await Promise.all([
    getTask(taskListId, fromTaskId),
    getTask(taskListId, toTaskId),
  ])
  if (!fromTask || !toTask) return false

  if (!fromTask.blocks.includes(toTaskId)) {
    await updateTask(taskListId, fromTaskId, {
      blocks: [...fromTask.blocks, toTaskId],
    })
  }
  if (!toTask.blockedBy.includes(fromTaskId)) {
    await updateTask(taskListId, toTaskId, {
      blockedBy: [...toTask.blockedBy, fromTaskId],
    })
  }
  return true
}
```

删除任务时，系统自动清理所有指向它的依赖引用。`deleteTask()` 在 [`packages/ccb/src/utils/tasks.ts#L393-L441`](/packages/ccb/src/utils/tasks.ts#L393-L441) 中遍历全部任务移除 `blocks` 和 `blockedBy` 中的引用（见 [`L420-L434`](/packages/ccb/src/utils/tasks.ts#L420-L434)）。

### 任务认领与并发控制

`claimTask()` 是 V2 的核心并发原语，位于 [`packages/ccb/src/utils/tasks.ts#L541-L612`](/packages/ccb/src/utils/tasks.ts#L541-L612)，支持两种锁定粒度：

#### 1. 任务级锁（默认）
仅锁定目标任务文件，适合单 Agent 场景。流程为：读取任务 → 检查 owner → 检查 status → 检查 blockedBy → 写入 owner（使用 `updateTaskUnsafe` 以避免嵌套死锁）。

#### 2. 列表级锁 + Agent 忙碌检查
当 `checkAgentBusy: true` 时，锁定整个任务列表目录（`.lock` 文件），原子化地完成：列出全部任务 → 检查状态 → 检查依赖 → 检查 Agent 是否已拥有其他未完成任务 → 写入 owner。实现见 `claimTaskWithBusyCheck()`（[`L618-L692`](/packages/ccb/src/utils/tasks.ts#L618-L692)）。

认领失败的 4 种原因在 `ClaimTaskResult` 中定义（[`L488-L499`](/packages/ccb/src/utils/tasks.ts#L488-L499)）：

| reason | 含义 |
|--------|------|
| `task_not_found` | 任务 ID 不存在 |
| `already_claimed` | 已被其他 Agent 认领 |
| `already_resolved` | 任务已标记 `completed` |
| `blocked` | `blockedBy` 列表中有未完成的任务 |
| `agent_busy` | 该 Agent 已拥有其他未完成任务（仅 `checkAgentBusy` 模式） |

### Agent 团队的任务生命周期

在 swarm 模式下，任务系统的生命周期是这样的（结合 `TaskUpdateTool` 的代码逻辑）：

1. **Leader 创建团队**
2. **Leader 用 `TaskCreate` 创建任务**（`status=pending`, `owner=undefined`）
3. **Leader 用 `TaskUpdate` 设置依赖关系**（`addBlocks` / `addBlockedBy`）
   - 实际调用 `blockTask()` 见 [`packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts#L301-L324`](/packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts#L301-L324)
4. **Teammate 调用 `TaskList`** → 发现可认领的任务
5. **Teammate 调用 `TaskUpdate(taskId, {status: "in_progress"})`**
   - 自动设置 `owner` 为 teammate 名称（见 [`L188-L199`](/packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts#L188-L199)）
   - Leader 通过 mailbox 收到 `task_assignment` 通知（见 [`L277-L298`](/packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts#L277-L298)）
6. **Teammate 完成工作 → `TaskUpdate(taskId, {status: "completed"})`**
   - `tool_result` 提示 "Call TaskList to find your next available task"
   - 依赖此任务的其他任务自动解锁
7. **Teammate 异常退出 → `unassignTeammateTasks()`**
   - 未完成任务被重置为 `pending` + `owner=undefined`
   - Leader 收到通知并重新分配

`unassignTeammateTasks()` 的完整实现在 [`packages/ccb/src/utils/tasks.ts#L818-L860`](/packages/ccb/src/utils/tasks.ts#L818-L860)。

### Hooks 集成

`TaskCreate` 和 `TaskUpdate` 都集成了 hooks 系统。

**创建时**：`TaskCreateTool.call()` 在写入任务文件后执行 `executeTaskCreatedHooks`：

```typescript
const generator = executeTaskCreatedHooks(
  taskId, subject, description, getAgentName(), getTeamName(),
  undefined, context?.abortController?.signal, undefined, context,
)
for await (const result of generator) {
  if (result.blockingError) {
    blockingErrors.push(getTaskCreatedHookMessage(result.blockingError))
  }
}
if (blockingErrors.length > 0) {
  await deleteTask(getTaskListId(), taskId)
  throw new Error(blockingErrors.join('\n'))
}
```

参见 [`packages/ccb/src/tools/TaskCreateTool/TaskCreateTool.ts#L80-L113`](/packages/ccb/src/tools/TaskCreateTool/TaskCreateTool.ts#L80-L113)。如果 hook 返回 `blockingError`，刚创建的任务会被立即删除。

**完成时**：`TaskUpdateTool.call()` 在将状态设为 `completed` 前执行 `executeTaskCompletedHooks`：

```typescript
if (status === 'completed') {
  const generator = executeTaskCompletedHooks(
    taskId, existingTask.subject, existingTask.description,
    getAgentName(), getTeamName(), undefined,
    context?.abortController?.signal, undefined, context,
  )
  for await (const result of generator) {
    if (result.blockingError) {
      blockingErrors.push(getTaskCompletedHookMessage(result.blockingError))
    }
  }
  if (blockingErrors.length > 0) { /* 阻止完成 */ }
}
```

参见 [`packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts#L232-L264`](/packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts#L232-L264)。

Hooks 的底层声明在 [`packages/ccb/src/utils/hooks.ts#L3896-L3968`](/packages/ccb/src/utils/hooks.ts#L3896-L3968) 附近。这允许外部系统（CI、审批流）参与任务状态机。

### activeForm：终端 UX 的细节

每个任务有两个文案字段：

- `subject`：祈使句，用于任务列表展示（"Fix auth bug"）
- `activeForm`：进行时形式，用于 spinner 动画（"Fixing auth bug…"）

当 `activeForm` 缺省时，spinner 回退显示 `subject`。这个设计确保了用户在等待时看到的是"正在做什么"而非"要做什么"。在 `TaskListV2` 组件中，任务项的渲染逻辑见 [`packages/ccb/src/components/TaskListV2.tsx#L271-L283`](/packages/ccb/src/components/TaskListV2.tsx#L271-L283) 的 `getTaskIcon` 和 [`L285-L360`](/packages/ccb/src/components/TaskListV2.tsx#L285-L360) 的 `TaskItem`。

### Plan Mode 与任务系统的配合

Plan Mode（计划模式）和任务系统是互补但独立的机制：

- Plan Mode 限制工具集为只读（搜索、阅读），迫使 AI 先理解再行动
- AI 在 Plan Mode 中用 `TaskCreate` 建立任务列表
- 用户审批后退出 Plan Mode
- AI 按 `blockedBy` 拓扑序逐项执行，每项用 `TaskUpdate` 标记进度

`shouldDefer: true` 属性确保这些工具调用不会触发权限确认弹窗 —— 任务管理操作始终自动批准，因为它们不产生副作用。参见 `TodoWriteTool` 的 [`L51`](/packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts#L51)、`TaskCreateTool` 的 [`L67`](/packages/ccb/src/tools/TaskCreateTool/TaskCreateTool.ts#L67)、`TaskUpdateTool` 的 [`L107`](/packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts#L107)、`TaskListTool` 的 [`L52`](/packages/ccb/src/tools/TaskListTool/TaskListTool.ts#L52)。

### 终端渲染：TaskListV2

V2 任务列表在终端中的 UI 由 `TaskListV2` 组件负责，位于 [`packages/ccb/src/components/TaskListV2.tsx`](/packages/ccb/src/components/TaskListV2.tsx)。它做了几件关键的事：

1. **最近完成的淡出**：任务刚完成 30 秒内仍显示在列表中（`RECENT_COMPLETED_TTL_MS = 30_000`），随后被折叠到隐藏摘要中。这是通过 `completionTimestampsRef` 和一个 `setTimeout` 强制重绘实现的（见 [`L44-L95`](/packages/ccb/src/components/TaskListV2.tsx#L44-L95)）。

2. **截断与优先级排序**：当终端行数不足时，优先显示：最近完成 > in_progress > pending（未被阻塞的在前）> 较早完成。截断提示形如 `… +1 in progress, 2 pending`（见 [`L152-L216`](/packages/ccb/src/components/TaskListV2.tsx#L152-L216)）。

3. **Teammate 活动摘要**：对于 in_progress 的任务，如果 `appStateTasks` 中存在对应的 teammate 后台任务，会展示该 teammate 的最近活动（例如 "Reading src/utils/tasks.ts"）。这来自 `summarizeRecentActivities`（见 [`L124-L141`](/packages/ccb/src/components/TaskListV2.tsx#L124-L141)）。

4. **响应式布局**：当终端宽度小于 60 列时，隐藏 `@owner` 标签以节省空间（见 [`L303-L307`](/packages/ccb/src/components/TaskListV2.tsx#L303-L307)）。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`packages/ccb/src/utils/tasks.ts`](/packages/ccb/src/utils/tasks.ts) | `isTodoV2Enabled`、`getTaskListId`、`createTask`、`updateTask`、`deleteTask`、`claimTask`、`blockTask`、`unassignTeammateTasks`、高水位标记与文件锁 |
| [`packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts`](/packages/ccb/src/tools/TodoWriteTool/TodoWriteTool.ts) | V1 TodoWrite 全量替换、智能清空、verification nudge |
| [`packages/ccb/src/tools/TaskCreateTool/TaskCreateTool.ts`](/packages/ccb/src/tools/TaskCreateTool/TaskCreateTool.ts) | V2 任务创建、hooks 调用、失败回滚 |
| [`packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts`](/packages/ccb/src/tools/TaskUpdateTool/TaskUpdateTool.ts) | V2 任务更新/删除/依赖设置、自动认领、mailbox 通知 |
| [`packages/ccb/src/tools/TaskListTool/TaskListTool.ts`](/packages/ccb/src/tools/TaskListTool/TaskListTool.ts) | V2 任务列表查询、过滤已完成阻塞项 |
| [`packages/ccb/src/components/TaskListV2.tsx`](/packages/ccb/src/components/TaskListV2.tsx) | 终端内任务列表渲染、 teammate 颜色/活动/截断 |
| [`packages/ccb/src/utils/hooks.ts`](/packages/ccb/src/utils/hooks.ts) | `executeTaskCreatedHooks`、`executeTaskCompletedHooks` 声明 |
