# Unit 45: ULTRAPLAN — 增强规划技术报告

**生成日期**: 2026-04-09  
**原始文档**: https://ccb.agent-aura.top/docs/features/ultraplan  
**Feature Flag**: `FEATURE_ULTRAPLAN=1`


> **源码映射说明**：ULTRAPLAN 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。---

## 一、功能概述

ULTRAPLAN 是 Claude Code 的增强计划模式，在用户输入中检测 "ultraplan" 关键字时，自动将任务路由到 **Claude Code on the Web (CCR)** 进行深度规划。相比普通 plan mode，ultraplan 提供更深入的规划能力，支持本地和远程（CCR）执行。

### 核心特性

| 触发方式 | 行为 |
|---------|------|
| 输入含 "ultraplan" 的文本 | 自动重定向到 `/ultraplan` 命令 |
| `/ultraplan` 斜杠命令 | 直接执行 |
| 彩虹高亮 | 输入框中 "ultraplan" 关键字彩虹动画 |

---

## 二、实现架构

### 2.1 模块状态总览

| 模块 | 文件 | 行数 | 状态 |
|------|------|------|------|
| 命令处理器 | `packages/ccb/src/commands/ultraplan.tsx` | 474 | ✅ 完整 |
| CCR 会话 | `packages/ccb/src/utils/ultraplan/ccrSession.ts` | 349 | ✅ 完整 |
| 关键字检测 | `packages/ccb/src/utils/ultraplan/keyword.ts` | 127 | ✅ 完整 |
| 嵌入式提示 | `packages/ccb/src/utils/ultraplan/prompt.txt` | ~20 | ✅ 完整 |
| REPL 对话框 | `packages/ccb/src/screens/REPL.tsx` | — | ✅ 布线 |
| 关键字高亮 | `packages/ccb/src/components/PromptInput/PromptInput.tsx` | — | ✅ 布线 |

### 2.2 关键字检测 — 智能过滤

**文件**: `packages/ccb/src/utils/ultraplan/keyword.ts` [#L1-L127](packages/ccb/src/utils/ultraplan/keyword.ts)

```typescript
// 核心检测函数 — 排除引号内、路径中的 "ultraplan"
function findKeywordTriggerPositions(text: string, keyword: string): TriggerPosition[] {
  // 排除斜杠命令输入
  if (text.startsWith('/')) return []
  
  // 构建 quotedRanges 排除引号/括号/路径内的关键字
  const quotedRanges: Array<{ start: number; end: number }> = []
  // ... 处理 `, ", <, {, [, (, ' 等定界符
  
  // 匹配关键字但排除误触发场景
  const wordRe = new RegExp(`\\b${keyword}\\b`, 'gi')
  const matches = text.matchAll(wordRe)
  for (const match of matches) {
    // 排除 preceded by /, \, - (路径/标识符)
    if (before === '/' || before === '\\' || before === '-') continue
    // 排除 followed by /, \, -, ? (路径/问题)
    if (after === '/' || after === '\\' || after === '-' || after === '?') continue
    if (after === '.' && isWord(text[end + 1])) continue  // 排除文件扩展名
  }
}

export function findUltraplanTriggerPositions(text: string): TriggerPosition[] {
  return findKeywordTriggerPositions(text, 'ultraplan')
}
```

**设计决策**:
- 排除引号内的 "ultraplan"（如 `"ultraplan"` 或 `'ultraplan'`）
- 排除路径中的 "ultraplan"（如 `/path/to/ultraplan/`、`ultraplan.tsx`）
- 排除斜杠命令输入（如 `/rename ultraplan foo`）
- 排除疑问句（如 `how to use ultraplan?`）

### 2.3 CCR 远程会话 — ExitPlanModeScanner

**文件**: `packages/ccb/src/utils/ultraplan/ccrSession.ts` [#L1-L349](packages/ccb/src/utils/ultraplan/ccrSession.ts)

`ExitPlanModeScanner` 类实现完整的事件状态机：

```typescript
export class ExitPlanModeScanner {
  private exitPlanCalls: string[] = []        // ExitPlanMode tool_use 调用 ID
  private results = new Map<string, ToolResultBlockParam>()  // tool_result 映射
  private rejectedIds = new Set<string>()     // 被拒绝的 plan ID
  private terminated: { subtype: string } | null = null
  everSeenPending = false

  //  ingest SDK 消息批次，返回当前状态
  ingest(newEvents: SDKMessage[]): ScanResult {
    for (const m of newEvents) {
      if (m.type === 'assistant') {
        // 收集 ExitPlanMode tool_use 调用
        for (const block of m.message.content) {
          if (block.type !== 'tool_use') continue
          const tu = block as ToolUseBlock
          if (tu.name === EXIT_PLAN_MODE_V2_TOOL_NAME) {
            this.exitPlanCalls.push(tu.id)
          }
        }
      } else if (m.type === 'user') {
        // 收集 tool_result (用户审批/拒绝结果)
        const content = m.message.content
        for (const block of content) {
          if (block.type === 'tool_result') {
            this.results.set(block.tool_use_id, block)
          }
        }
      } else if (m.type === 'result' && m.subtype !== 'success') {
        // 会话终止信号
        this.terminated = { subtype: m.subtype as string }
      }
    }
    
    // 状态优先级：approved > terminated > rejected > pending > unchanged
    // ... 扫描最新的 ExitPlanMode 调用结果
  }
}
```

**轮询函数** `pollForApprovedExitPlanMode` [#L198-L306](packages/ccb/src/utils/ultraplan/ccrSession.ts#L198-L306):

```typescript
export async function pollForApprovedExitPlanMode(
  sessionId: string,
  timeoutMs: number,            // 30 分钟超时
  onPhaseChange?: (phase: UltraplanPhase) => void,
  shouldStop?: () => boolean,
): Promise<PollResult> {
  const deadline = Date.now() + timeoutMs
  const scanner = new ExitPlanModeScanner()
  let cursor: string | null = null
  let failures = 0
  let lastPhase: UltraplanPhase = 'running'

  while (Date.now() < deadline) {
    // 轮询远程会话事件 (3 秒间隔)
    const resp = await pollRemoteSessionEvents(sessionId, cursor)
    newEvents = resp.newEvents
    cursor = resp.lastEventId
    sessionStatus = resp.sessionStatus  // 'idle' | 'running' | 'requires_action'
    
    // 扫描事件流
    let result: ScanResult = scanner.ingest(newEvents)
    
    if (result.kind === 'approved') {
      return { plan: result.plan, rejectCount: scanner.rejectCount, executionTarget: 'remote' }
    }
    if (result.kind === 'teleport') {
      return { plan: result.plan, rejectCount: scanner.rejectCount, executionTarget: 'local' }
    }
    
    // 相位状态转换
    const phase: UltraplanPhase = scanner.hasPendingPlan
      ? 'plan_ready'        // ExitPlanMode 等待浏览器审批
      : quietIdle
        ? 'needs_input'     // 远程询问问题，等待用户回复
        : 'running'
    
    if (phase !== lastPhase) {
      onPhaseChange?.(phase)
    }
    
    await sleep(POLL_INTERVAL_MS)  // 3 秒
  }
}
```

**相位状态** `UltraplanPhase`:
- `running` — 远程正在执行
- `needs_input` — 远程询问问题，等待用户回复
- `plan_ready` — ExitPlanMode 已发出，等待浏览器审批

### 2.4 数据流

```
用户输入 "帮我 ultraplan 重构这个模块"
         │
         ▼
processUserInput 检测 "ultraplan" 关键字
[packages/ccb/src/utils/processUserInput/processUserInput.ts#L467-L493]
         │
         ▼
重定向到 /ultraplan 命令
         │
         ├── 本地执行 → EnterPlanMode
         │
         └── 远程执行 → teleportToRemote → CCR 会话
                │
                ▼
         ExitPlanModeScanner 轮询 (3 秒间隔)
         [packages/ccb/src/utils/ultraplan/ccrSession.ts#L198-L306]
                │
                ▼
         用户在远程审批 → 本地收到结果
                │
                ▼
         UltraplanChoiceDialog 选择执行方式
         [packages/ccb/src/components/ultraplan/UltraplanChoiceDialog.tsx]
```

---

## 三、核心组件详解

### 3.1 命令处理器 — `/ultraplan`

**文件**: `packages/ccb/src/commands/ultraplan.tsx` [#L1-L474](packages/ccb/src/commands/ultraplan.tsx)

**核心函数** `launchUltraplan` [#L258-L317](packages/ccb/src/commands/ultraplan.tsx#L258-L317):

```typescript
export async function launchUltraplan(opts: {
  blurb: string;
  seedPlan?: string;           // 来自计划审批对话框的草稿计划
  getAppState: () => AppState;
  setAppState: (f: (prev: AppState) => AppState) => void;
  signal: AbortSignal;
  disconnectedBridge?: boolean;
  onSessionReady?: (msg: string) => void;  // URL 就绪回调
}): Promise<string> {
  const { blurb, seedPlan, getAppState, setAppState, signal, disconnectedBridge, onSessionReady } = opts;
  const { ultraplanSessionUrl: active, ultraplanLaunching } = getAppState();
  if (active || ultraplanLaunching) {
    return buildAlreadyActiveMessage(active);  // 防止重复启动
  }

  if (!blurb && !seedPlan) {
    // 无参数时返回使用说明（支持 Markdown 渲染）
    return [
      'Usage: /ultraplan \\u003cprompt\\u003e, or include "ultraplan" anywhere in your prompt',
      '',
      'Advanced multi-agent plan mode with our most powerful model (Opus).',
    ].join('\n');
  }

  // 同步设置 launching 状态，防止 teleportToRemote 窗口期重复启动
  setAppState(prev => (prev.ultraplanLaunching ? prev : { ...prev, ultraplanLaunching: true }));

  // detached 执行，失败通过 enqueuePendingNotification 通知
  void launchDetached({ blurb, seedPlan, getAppState, setAppState, signal, onSessionReady });

  return buildLaunchMessage(disconnectedBridge);  // 立即返回启动消息
}
```

**模型选择** [#L42-L44](packages/ccb/src/commands/ultraplan.tsx#L42-L44):

```typescript
function getUltraplanModel(): string {
  return getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_ultraplan_model',
    ALL_MODEL_CONFIGS.opus46.firstParty  // 默认 Opus 4.6
  )
}
```

### 3.2 对话框组件

#### UltraplanLaunchDialog — 启动前确认

**文件**: `packages/ccb/src/components/ultraplan/UltraplanLaunchDialog.tsx` [#L1-L153](packages/ccb/src/components/ultraplan/UltraplanLaunchDialog.tsx)

用户选择是否启动 ultraplan 会话：
- **Run ultraplan** — 禁用 remote control 并启动 CCR 会话
- **Not now** — 取消

#### UltraplanChoiceDialog — 计划审批后选择

**文件**: `packages/ccb/src/components/ultraplan/UltraplanChoiceDialog.tsx` [#L1-L244](packages/ccb/src/components/ultraplan/UltraplanChoiceDialog.tsx)

用户选择计划执行方式：
- **Implement here** — 在当前会话注入计划执行
- **Start new session** — 清空会话，仅用计划开始新会话
- **Cancel** — 保存计划到文件，不执行

```typescript
const handleChoice = async (choice: 'here' | 'fresh' | 'cancel') => {
  switch (choice) {
    case 'here':
      // 注入计划到当前会话
      enqueuePendingNotification({ value: 'Ultraplan approved in browser. Here is the plan:...' })
      break
    case 'fresh':
      // 清空会话，仅用计划开始
      await clearConversation({...})
      enqueuePendingNotification({ value: `Here is the approved implementation plan:\n\n${plan}` })
      break
    case 'cancel':
      // 保存计划到文件
      await writeFile(join(getCwd(), `${getDateStamp()}-ultraplan.md`), plan)
      break
  }
  
  // 标记任务完成，归档远程会话
  updateTaskState(taskId, setAppState, task => ({ ...task, status: 'completed', endTime: Date.now() }))
  archiveRemoteSession(sessionId)
}
```

### 3.3 输入框高亮 — 彩虹动画

**文件**: `packages/ccb/src/components/PromptInput/PromptInput.tsx`

关键字检测和高亮逻辑：

```typescript
// 检测 ultraplan 关键字位置
const ultraplanTriggers = useMemo(
  () =>
    feature('ULTRAPLAN') && !ultraplanSessionUrl && !ultraplanLaunching
      ? findUltraplanTriggerPositions(displayedValue)
      : [],
  [displayedValue, ultraplanSessionUrl, ultraplanLaunching],
)

// 彩虹高亮效果
if (feature('ULTRAPLAN')) {
  for (const trigger of ultraplanTriggers) {
    for (let i = trigger.start; i < trigger.end; i++) {
      highlights.push({
        start: i,
        end: i + 1,
        // ... 彩虹色计算
      })
    }
  }
}

// 通知提示
useEffect(() => {
  if (feature('ULTRAPLAN') && ultraplanTriggers.length) {
    addNotification({
      key: 'ultraplan-active',
      text: 'This prompt will launch an ultraplan session in Claude Code on the web',
      priority: 'immediate',
      timeoutMs: 5000,
    })
  } else {
    removeNotification('ultraplan-active')
  }
}, [ultraplanTriggers.length])
```

### 3.4 processUserInput 集成

**文件**: `packages/ccb/src/utils/processUserInput/processUserInput.ts` [#L467-L493](packages/ccb/src/utils/processUserInput/processUserInput.ts#L467-L493)

关键字检测和重定向逻辑：

```typescript
if (
  feature('ULTRAPLAN') &&
  mode === 'prompt' &&
  !context.options.isNonInteractiveSession &&
  inputString !== null &&
  !effectiveSkipSlash &&
  !inputString.startsWith('/') &&
  !context.getAppState().ultraplanSessionUrl &&
  !context.getAppState().ultraplanLaunching &&
  hasUltraplanKeyword(preExpansionInput ?? inputString)  // 使用 pre-expansion 输入防止粘贴触发
) {
  logEvent('tengu_ultraplan_keyword', {})
  const rewritten = replaceUltraplanKeyword(inputString).trim()  // "ultraplan" → "plan"
  const { processSlashCommand } = await import('./processSlashCommand.js')
  const slashResult = await processSlashCommand(
    `/ultraplan ${rewritten}`,
    precedingInputBlocks,
    imageContentBlocks,
    [],
    context,
    setToolJSX,
    uuid,
    isAlreadyProcessing,
    canUseTool,
  )
  return addImageMetadataMessage(slashResult, imageMetadataTexts)
}
```

### 3.5 AppState 状态管理

**文件**: `packages/ccb/src/state/AppStateStore.ts` [#L428-L443](packages/ccb/src/state/AppStateStore.ts#L428-L443)

Ultraplan 相关状态：

```typescript
interface AppState {
  // 正在启动 ultraplan (teleportToRemote 期间)
  ultraplanLaunching?: boolean
  
  // 活跃的 CCR 会话 URL (设置后禁用关键字触发)
  ultraplanSessionUrl?: string
  
  // 已审批的计划等待用户选择
  ultraplanPendingChoice?: { plan: string; sessionId: string; taskId: string }
  
  // 启动前对话框等待选择
  ultraplanLaunchPending?: { blurb: string }
}
```

### 3.6 RemoteAgentTask 集成

**文件**: `packages/ccb/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`

Ultraplan 任务状态扩展：

```typescript
export type RemoteAgentTaskState = TaskStateBase & {
  type: 'remote_agent'
  isUltraplan?: boolean  // 标记为 ultraplan 任务
  ultraplanPhase?: Exclude<UltraplanPhase, 'running'>  // 相位状态用于 pill 显示
  // ...
}
```

轮询特殊处理 [#L773-L780](packages/ccb/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx#L773-L780):

```typescript
// Ultraplan: result(success) 在每个 CCR turn 后触发，不能用于完成判断
// startDetachedPoll 通过 ExitPlanMode 扫描拥有完成判断权
const result =
  task.isUltraplan || task.isLongRunning
    ? undefined
    : accumulatedLog.findLast(msg => msg.type === 'result')
```

---

## 四、关键设计决策

### 4.1 智能关键字过滤

排除引号和路径中的 "ultraplan"，避免误触发：
- `"ultraplan"` — 引号内的关键字
- `/path/to/ultraplan/` — 文件路径
- `ultraplan.tsx` — 文件扩展名
- `--ultraplan-mode` — CLI 参数
- `how to use ultraplan?` — 疑问句

### 4.2 本地/远程双模式与实现风险

CCR 会话依赖 Bridge Mode 的远程轮询基础设施。若 Bridge worker 崩溃或网络分区，`pollForApprovedExitPlanMode` 的 30 分钟长轮询会造成资源悬挂；此外，`ExitPlanModeScanner` 目前仅处理 `ExitPlanMode` 一种工具，未来若 CCR 侧新增其他协议工具，扫描器必须同步扩展，否则状态机将漏判。

支持两种执行方式：
- **本地执行** — 用户选择 "Implement here" 或 "Start new session"，计划在当前终端执行
- **远程执行 (CCR)** — 用户在浏览器审批后直接在 CCR 执行，本地仅作为监视器

### 4.3 彩虹高亮反馈

输入框中 "ultraplan" 关键字使用彩虹动画高亮，暗示这是特殊功能，同时提供 `ultraplan-active` 通知提示。

### 4.4 Pre-expansion 输入检测

关键字检测使用 `preExpansionInput` 而非扩展后的输入，防止粘贴内容中的 "ultraplan" 意外触发：

```typescript
hasUltraplanKeyword(preExpansionInput ?? inputString)
```

### 4.5 30 分钟超时

Ultraplan 轮询使用 30 分钟超时 (#L33):

```typescript
const ULTRAPLAN_TIMEOUT_MS = 30 * 60 * 1000
```

---

## 五、使用方式

```bash
# 启用 feature
FEATURE_ULTRAPLAN=1 bun run dev

# 在 REPL 中使用
# > ultraplan 重构认证模块
# > /ultraplan 重构认证模块
```

---

## 六、文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/ccb/src/commands/ultraplan.tsx` | 474 | 斜杠命令处理器 |
| `packages/ccb/src/utils/ultraplan/ccrSession.ts` | 349 | CCR 远程会话管理 + ExitPlanModeScanner |
| `packages/ccb/src/utils/ultraplan/keyword.ts` | 127 | 关键字检测和替换 |
| `packages/ccb/src/utils/ultraplan/prompt.txt` | ~20 | 嵌入式提示 |
| `packages/ccb/src/utils/processUserInput/processUserInput.ts` | ~640 | 关键字重定向 (#L467-L493) |
| `packages/ccb/src/components/PromptInput/PromptInput.tsx` | 3175 | 彩虹高亮 |
| `packages/ccb/src/components/ultraplan/UltraplanLaunchDialog.tsx` | 153 | 启动前对话框 |
| `packages/ccb/src/components/ultraplan/UltraplanChoiceDialog.tsx` | 244 | 计划审批后对话框 |
| `packages/ccb/src/screens/REPL.tsx` | 6194 | 对话框渲染集成 |
| `packages/ccb/src/state/AppStateStore.ts` | 569 | 状态定义 |
| `packages/ccb/src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` | 1102 | 远程任务集成 |

---

## 七、事件追踪

Analytics 事件：
- `tengu_ultraplan_keyword` — 关键字触发
- `tengu_ultraplan_launched` — 会话启动
- `tengu_ultraplan_approved` — 计划审批
- `tengu_ultraplan_failed` — 失败
- `tengu_ultraplan_awaiting_input` — 等待用户输入
- `tengu_ultraplan_create_failed` — 创建失败

---

## 八、相关 Feature Flags

- `FEATURE_ULTRAPLAN` — 主开关
- `tengu_ultraplan_model` — 模型选择 (默认 Opus 4.6)
- `BRIDGE_MODE` — Remote Control (与 ultraplan 互斥)
