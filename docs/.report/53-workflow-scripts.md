# Unit 53 — Workflow Scripts 技术报告

> Feature Flag: `FEATURE_WORKFLOW_SCRIPTS=1`  
> 文档来源: `packages/ccb/docs/features/workflow-scripts.md`  
> 主项目: `packages/ccb`（claw-code / ccb 子项目）  


> **源码映射说明**：Workflow Scripts 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。---

## 一、功能概述

`WORKFLOW_SCRIPTS` 的设计目标是实现基于文件的多步自动化工作流：用户可以定义 YAML/JSON 格式的工作流描述文件，系统将其解析为可执行的多 agent 步骤序列，并通过 `/workflows` 命令统一管理和触发。

**当前实现状态：全部 Stub（骨架已布线完整）**。核心执行引擎、DSL 解析、内置工作流均未实现，但命令注册、任务系统、工具系统、UI 列表与详情对话框的接入点已全部就绪。

---

## 二、源码结构与实现锚点

### 2.1 整体模块状态

| 模块 | 文件 | 状态 | 源码位置 |
|------|------|------|----------|
| WorkflowTool | `src/tools/WorkflowTool/WorkflowTool.ts` | Stub |  |
| Workflow 权限 | `src/tools/WorkflowTool/WorkflowPermissionRequest.ts` | Stub | [#L1-L4](#workflowpermissionrequest) |
| 常量 | `src/tools/WorkflowTool/constants.ts` | Stub | [#L1-L2](#constants) |
| 命令创建 | `src/tools/WorkflowTool/createWorkflowCommand.ts` | Stub | [#L1-L4](#createworkflowcommand) |
| 内置工作流 | `src/tools/WorkflowTool/bundled/` | **缺失** | 目录不存在 |
| 本地工作流任务 | `src/tasks/LocalWorkflowTask/LocalWorkflowTask.ts` | Stub | [#L1-L12](#localworkflowtask) |
| 详情对话框 | `src/components/tasks/WorkflowDetailDialog.ts` | Stub | [#L1-L4](#workflowdetaildialog) |
| 任务注册 | `src/tasks.ts` | 布线完整 | [#L1-L39](#tasks-registry) |
| 工具注册 | `src/tools.ts` | 布线完整 | [#L127-L132](#tools-registry) |
| 命令注册 | `src/commands.ts` | 布线完整 | [#L86-L90](#commands-registry) |
| 任务类型定义 | `src/Task.ts` | 类型就绪 | [#L1-L126](#task-types) |
| 命令类型扩展 | `src/types/command.ts` | `kind: 'workflow'` | [#L197-L199](#command-types) |

---

## 三、核心源码解析

### 3.1 特征开关与命令入口（`src/commands.ts`）

```typescript
// src/commands.ts#L86-L90
const workflowsCmd = feature('WORKFLOW_SCRIPTS')
  ? (
      require('./commands/workflows/index.js') as typeof import('./commands/workflows/index.js')
    ).default
  : null
```

- 仅在 `FEATURE_WORKFLOW_SCRIPTS=1` 时动态加载 `/workflows` 命令实现。
- 当前 `src/commands/workflows/index.ts` 为自动生成的空对象 stub（[#L1-L4](#commands-workflows-index)）。

命令列表注册位置（加入 `COMMANDS` 数组）：

```typescript
// src/commands.ts#L341-L342
...(workflowsCmd ? [workflowsCmd] : []),
```

所有命令统一加载时，`getWorkflowCommands` 负责扫描磁盘上的工作流定义并生成 `Command[]`：

```typescript
// src/commands.ts#L451-L460
const loadAllCommands = memoize(async (cwd: string): Promise<Command[]> => {
  const [
    { skillDirCommands, pluginSkills, bundledSkills, builtinPluginSkills },
    pluginCommands,
    workflowCommands,
  ] = await Promise.all([
    getSkills(cwd),
    getPluginCommands(),
    getWorkflowCommands ? getWorkflowCommands(cwd) : Promise.resolve([]),
  ])
  return [
    ...bundledSkills,
    ...builtinPluginSkills,
    ...skillDirCommands,
    ...workflowCommands,
    ...pluginCommands,
    ...pluginSkills,
    ...COMMANDS(),
  ]
})
```

### 3.2 命令类型扩展（`src/types/command.ts`）

为区分 workflow 来源的命令，已预留 `kind` 字段：

```typescript
// src/types/command.ts#L197-L199
kind?: 'workflow' // Distinguishes workflow-backed commands (badged in autocomplete)
```

### 3.3 工具注册（`src/tools.ts`）

`WorkflowTool` 被包装在懒加载 IIFE 中，加载时先初始化内置工作流，再返回工具对象：

```typescript
// src/tools.ts#L127-L132
const WorkflowTool = feature('WORKFLOW_SCRIPTS')
  ? (() => {
      require('./tools/WorkflowTool/bundled/index.js').initBundledWorkflows()
      return require('./tools/WorkflowTool/WorkflowTool.js').WorkflowTool
    })()
  : null
```

注意：`bundled/index.js` 目录目前**不存在**，若启用 flag 此处将抛出运行时错误。

### 3.4 任务ID与状态基类（`src/Task.ts`）

工作流任务类型已纳入 `TaskType` 联合：

```typescript
// src/Task.ts#L6-L14
export type TaskType =
  | 'local_bash'
  | 'local_agent'
  | 'remote_agent'
  | 'in_process_teammate'
  | 'local_workflow'
  | 'monitor_mcp'
  | 'dream'
```

任务 ID 前缀分配为 `w`：

```typescript
// src/Task.ts#L79-L87
const TASK_ID_PREFIXES: Record<string, string> = {
  local_bash: 'b',
  local_agent: 'a',
  remote_agent: 'r',
  in_process_teammate: 't',
  local_workflow: 'w',
  monitor_mcp: 'm',
  dream: 'd',
}
```

### 3.5 本地工作流任务（`src/tasks/LocalWorkflowTask/LocalWorkflowTask.ts`）

当前为最小 stub，仅导出类型和三个空操作函数：

```typescript
// src/tasks/LocalWorkflowTask/LocalWorkflowTask.ts#L1-L12
import type { TaskStateBase, SetAppState } from '../../Task.js'

export type LocalWorkflowTaskState = TaskStateBase & {
  type: 'local_workflow'
  summary?: string
  description: string
}
export const killWorkflowTask = (() => {})
export const skipWorkflowAgent = (() => {})
export const retryWorkflowAgent = (() => {})
```

任务注册表通过动态 `require` 加入该任务（`src/tasks.ts`）：

```typescript
// src/tasks.ts#L8-L14
const LocalWorkflowTask: Task | null = feature('WORKFLOW_SCRIPTS')
  ? require('./tasks/LocalWorkflowTask/LocalWorkflowTask.js').LocalWorkflowTask
  : null
```

随后由 `getAllTasks()` 推入任务数组（[#L22-L31](#tasks-all)）。

### 3.6 UI 层 — BackgroundTasksDialog（`src/components/tasks/BackgroundTasksDialog.tsx`）

`BackgroundTasksDialog` 是工作流在 REPL UI 中的主入口。对工作流做了完整的死码消除（DCE）保护：

```typescript
// src/components/tasks/BackgroundTasksDialog.tsx#L132-L142
const WorkflowDetailDialog = feature('WORKFLOW_SCRIPTS')
  ? (
      require('./WorkflowDetailDialog.js') as typeof import('./WorkflowDetailDialog.js')
    ).WorkflowDetailDialog
  : null
const workflowTaskModule = feature('WORKFLOW_SCRIPTS')
  ? (require('src/tasks/LocalWorkflowTask/LocalWorkflowTask.js') as typeof import('src/tasks/LocalWorkflowTask/LocalWorkflowTask.js'))
  : null
const killWorkflowTask = workflowTaskModule?.killWorkflowTask ?? null
const skipWorkflowAgent = workflowTaskModule?.skipWorkflowAgent ?? null
const retryWorkflowAgent = workflowTaskModule?.retryWorkflowAgent ?? null
```

在列表渲染中，工作流任务被单独分组，排序位于 `Local agents` 之后、`Dream` 之前：

```typescript
// src/components/tasks/BackgroundTasksDialog.tsx#L241
const workflows = sorted.filter(item => item.type === 'local_workflow')
```

详情面板为 `local_workflow` 类型渲染 `WorkflowDetailDialog`，并绑定 Kill / Skip / Retry 回调：

```tsx
// src/components/tasks/BackgroundTasksDialog.tsx#L525-L549
case 'local_workflow':
  if (!WorkflowDetailDialog) return null
  return (
    <WorkflowDetailDialog
      workflow={task}
      onDone={onDone}
      onKill={
        task.status === 'running' && killWorkflowTask
          ? () => killWorkflowTask(task.id, setAppState)
          : undefined
      }
      onSkipAgent={
        task.status === 'running' && skipWorkflowAgent
          ? agentId => skipWorkflowAgent(task.id, agentId, setAppState)
          : undefined
      }
      onRetryAgent={
        task.status === 'running' && retryWorkflowAgent
          ? agentId => retryWorkflowAgent(task.id, agentId, setAppState)
          : undefined
      }
      onBack={goBackToList}
      key={`workflow-${task.id}`}
    />
  )
```

列表中工作流分组 JSX：

```tsx
// src/components/tasks/BackgroundTasksDialog.tsx#L804-L830
{workflowTasks.length > 0 && (
  <Box flexDirection="column" marginTop={...}>
    <Text dimColor>
      <Text bold>{'  '}Workflows</Text> ({workflowTasks.length})
    </Text>
    <Box flexDirection="column">
      {workflowTasks.map(item => (
        <Item key={`workflow-${item.id}`} item={item} isSelected={...} />
      ))}
    </Box>
  </Box>
)}
```

### 3.7 UI 层 — BackgroundTask（状态栏/列表项）

`BackgroundTask.tsx` 对工作流任务的显示做了占位支持：

```tsx
// src/components/tasks/BackgroundTask.tsx#L82-L106
case 'local_workflow':
  return (
    <Text>
      {truncate(
        task.workflowName ?? task.summary ?? task.description,
        activityLimit,
        true,
      )}{' '}
      <TaskStatusText
        status={task.status}
        label={
          task.status === 'running'
            ? `${task.agentCount} ${plural(task.agentCount, 'agent')}`
            : task.status === 'completed'
              ? 'done'
              : undefined
        }
        suffix={...}
      />
    </Text>
  )
```

注意：这里引用的 `task.workflowName` 和 `task.agentCount` 在当前的 `LocalWorkflowTaskState` 类型中**尚未声明**，意味着 `BackgroundTask.tsx` 已经提前为完整状态做了 UI 假设。

### 3.8 Workflow 工具与权限 Stub（`src/tools/WorkflowTool/`）

| 文件 | 内容 | 锚点 |
|------|------|------|
| `WorkflowTool.ts` | `export const WorkflowTool: Record<string, unknown> = {};` |  |
| `createWorkflowCommand.ts` | `export const getWorkflowCommands = () => {};` |  |
| `constants.ts` | `export const WORKFLOW_TOOL_NAME: string = '';` |  |
| `WorkflowPermissionRequest.ts` | `export const WorkflowPermissionRequest = () => null;` |  |

这四个文件均为自动生成的空 stub，等待实际实现填充。

---

## 四、预期数据流

根据现有布线推断，完整实现后的数据流应如下：

```
用户定义工作流（YAML/JSON 文件，如 .claude/workflows/*.yaml）
         │
         ▼
/workflows 命令入口 → src/commands/workflows/index.ts
         │
         ▼
getWorkflowCommands() 扫描文件并解析为 Command 对象 [stub]
         │
         ▼
WorkflowTool 被注册到模型工具列表，接收 execute_workflow 调用 [stub]
         │
         ▼
      分发到 LocalWorkflowTask
         │
         ├── step 1: 调度子 Agent（AgentTool / InProcessTeammateTask）
         ├── step 2: 等待结果，决定串行/分支
         └── step N: 汇总输出
         │
         ▼
BackgroundTasksDialog + WorkflowDetailDialog 显示进度/kill/skip/retry
```

---

## 五、关键设计决策与约束

1. **Feature Flag 门控**
   - 所有 workflow 相关模块使用 `feature('WORKFLOW_SCRIPTS')` + 动态 `require` 包裹，确保外部构建可以通过 Bun 的 DCE 完全剔除未使用代码。
   - `BackgroundTasksDialog.tsx` 注释明确说明：使用相对路径 `./WorkflowDetailDialog.js` 而非 `src/...` 路径映射，因为 Bun 的 DCE 能静态解析并消除 `./` 开头的 require，而路径映射字符串会作为死代码字面量保留在 bundle 中。

2. **任务ID体系**
   - 工作流任务使用前缀 `w`，与 bash(`b`)、local agent(`a`)、remote agent(`r`) 等区分。

3. **UI 集成已完整预演**
   - 状态栏（BackgroundTask）已预留 `workflowName`、`agentCount` 字段占位。
   - 背景任务对话框（BackgroundTasksDialog）已预留 Kill / Skip Agent / Retry Agent 三条交互线。
   - 详情对话框（WorkflowDetailDialog）目前为空组件，未来只需实现内部渲染逻辑即可即插即用。

4. **内置工作流入口缺失**
   - `src/tools/WorkflowTool/bundled/index.js` 被工具注册代码直接 require，但该目录不存在。这是当前启用 flag 后**唯一会立即崩溃**的调用点。

---

## 六、需要补全的内容（施工清单）

| 优先级 | 模块 | 工作量 | 说明 |
|--------|------|--------|------|
| 1 | `WorkflowTool.ts` | 大 | Schema 定义 + 多步执行引擎 |
| 2 | `bundled/index.js` | 中 | 内置工作流定义（`initBundledWorkflows`） |
| 3 | `createWorkflowCommand.ts` | 中 | 从 YAML/JSON 文件解析生成 `Command` 对象 |
| 4 | `LocalWorkflowTask.ts` | 大 | 步骤协调、状态流转、kill/skip/retry 实现 |
| 5 | `WorkflowDetailDialog.ts` | 中 | 进度详情 UI（Ink 组件） |
| 6 | `WorkflowPermissionRequest.ts` | 小 | 权限批准 UI |
| 7 | `constants.ts` | 小 | 工具名常量（如 `execute_workflow`） |

---

## 七、文件索引速查

```
packages/ccb/
├── src/
│   ├── commands.ts                              命令注册与加载
│   ├── commands/workflows/index.ts              /workflows 命令 stub
│   ├── types/command.ts                         Command 类型（kind: 'workflow'）
│   ├── Task.ts                                  任务基类与 ID 生成
│   ├── tasks.ts                                 任务注册表
│   ├── tasks/LocalWorkflowTask/LocalWorkflowTask.ts   本地工作流任务 stub
│   ├── tools.ts                                 工具注册表
│   ├── tools/WorkflowTool/
│   │   ├── WorkflowTool.ts                      工具定义 stub
│   │   ├── createWorkflowCommand.ts             命令创建 stub
│   │   ├── constants.ts                         常量 stub
│   │   └── WorkflowPermissionRequest.ts         权限组件 stub
│   ├── components/tasks/
│   │   ├── BackgroundTask.tsx                   状态栏列表项（含 workflow 渲染分支）
│   │   ├── BackgroundTasksDialog.tsx            背景任务总览对话框
│   │   └── WorkflowDetailDialog.ts              工作流详情 UI stub
│   └── types/command.ts                         命令元类型
└── docs/features/workflow-scripts.md            原始设计文档
```

---

*Report generated for claw-code Unit 53 — Workflow Scripts.*
