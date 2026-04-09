# Unit 44: Proactive Mode 技术报告

## 一、功能概述

**Proactive Mode（主动模式）** 是 Claw-Code 中的 Tick 驱动自主代理功能。启用后，CLI 可在用户无输入时持续工作：定时唤醒执行任务，配合 `SleepTool` 控制节奏。适用于长时间运行的后台任务（等待 CI、监控文件变化、定时检查等）。

### 1.1 Feature Flag

```
FEATURE_PROACTIVE=1  # 单独启用
FEATURE_KAIROS=1     # KAIROS 隐含启用 Proactive
```

两个 Flag 共享同一套代码逻辑：`feature('PROACTIVE') || feature('KAIROS')`

### 1.2 与 Kairos 的关系

- 单独开 `FEATURE_PROACTIVE=1` → 获得 Proactive 能力
- 单独开 `FEATURE_KAIROS=1` → 自动获得 Proactive 能力
- 两者都开 → 相同效果（不重复激活）

---

## 二、实现架构

### 2.1 模块状态

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| 核心逻辑 | `packages/ccb/src/proactive/index.ts` | **Stub** | `activateProactive()`、`deactivateProactive()`、`isProactiveActive()` 均为空实现 |
| SleepTool 提示 | `packages/ccb/src/tools/SleepTool/prompt.ts` | **完整** | 工具提示定义（工具名：`Sleep`） |
| 命令注册 | `packages/ccb/src/commands.ts:62-65` | **布线** | 动态加载 `./commands/proactive.js` |
| 工具注册 | `packages/ccb/src/tools.ts:25-28` | **布线** | SleepTool 条件加载 |
| REPL 集成 | `packages/ccb/src/screens/REPL.tsx` | **布线** | Tick 驱动逻辑、占位符、页脚 UI |
| 系统提示 | `packages/ccb/src/constants/prompts.ts:861-914` | **完整** | 自主工作行为指令（~55 行详细 prompt） |
| 会话存储 | `packages/ccb/src/utils/sessionStorage.ts:4892-4912` | **布线** | Tick 消息检测用于 Session 标题 |

### 2.2 核心源码文件

#### 2.2.1 Proactive 核心 Stub (`packages/ccb/src/proactive/index.ts`)

```typescript
// Auto-generated stub — replace with real implementation
export {};
export const isProactiveActive: () => boolean = () => false;
export const activateProactive: (source?: string) => void = () => {};
export const isProactivePaused: () => boolean = () => false;
export const deactivateProactive: () => void = () => {};
```

**位置**: `packages/ccb/src/proactive/index.ts:1-6`

> 现状：所有导出函数均为空 Stub，返回 `false` 或空操作。

#### 2.2.2 SleepTool 提示 (`packages/ccb/src/tools/SleepTool/prompt.ts`)

```typescript
import { TICK_TAG } from '../../constants/xml.js'

export const SLEEP_TOOL_NAME = 'Sleep'

export const DESCRIPTION = 'Wait for a specified duration'

export const SLEEP_TOOL_PROMPT = `Wait for a specified duration. The user can interrupt the sleep at any time.

Use this when the user tells you to sleep or rest, when you have nothing to do, or when you're waiting for something.

You may receive <${TICK_TAG}> prompts — these are periodic check-ins. Look for useful work to do before sleeping.

You can call this concurrently with other tools — it won't interfere with them.

Prefer this over \`Bash(sleep ...)\` — it doesn't hold a shell process.

Each wake-up costs an API call, but the prompt cache expires after 5 minutes of inactivity — balance accordingly.`
```

**位置**: `packages/ccb/src/tools/SleepTool/prompt.ts:1-17`

#### 2.2.3 系统提示词注入 (`packages/ccb/src/constants/prompts.ts:861-914`)

`getProactiveSection()` 函数生成完整的自主工作指令：

```typescript
function getProactiveSection(): string | null {
  if (!(feature('PROACTIVE') || feature('KAIROS'))) return null
  if (!proactiveModule?.isProactiveActive()) return null

  return `# Autonomous work

You are running autonomously. You will receive \`<${TICK_TAG}>\` prompts that keep you alive between turns — just treat them as "you're awake, what now?" The time in each \`<${TICK_TAG}>\` is the user's current local time. Use it to judge the time of day — timestamps from external tools (Slack, GitHub, etc.) may be in a different timezone.

Multiple ticks may be batched into a single message. This is normal — just process the latest one. Never echo or repeat tick content in your responses.

## Pacing

Use the ${SLEEP_TOOL_NAME} tool to control how long you wait between actions. Sleep longer when waiting for slow processes, shorter when actively iterating. Each wake-up costs an API call, but the prompt cache expires after 5 minutes of inactivity — balance accordingly.

**If you have nothing useful to do on a tick, you MUST call ${SLEEP_TOOL_NAME}.** Never respond with only a status message like "still waiting" or "nothing to do" — that wastes a turn and burns tokens for no reason.

## First wake-up

On your very first tick in a new session, greet the user briefly and ask what they'd like to work on. Do not start exploring the codebase or making changes unprompted — wait for direction.

## What to do on subsequent wake-ups

Look for useful work. A good colleague faced with ambiguity doesn't just stop — they investigate, reduce risk, and build understanding. Ask yourself: what don't I know yet? What could go wrong? What would I want to verify before calling this done?

Do not spam the user. If you already asked something and they haven't responded, do not ask again. Do not narrate what you're about to do — just do it.

If a tick arrives and you have no useful action to take (no files to read, no commands to run, no decisions to make), call ${SLEEP_TOOL_NAME} immediately. Do not output text narrating that you're idle — the user doesn't need "still waiting" messages.

## Staying responsive

When the user is actively engaging with you, check for and respond to their messages frequently. Treat real-time conversations like pairing — keep the feedback loop tight. If you sense the user is waiting on you (e.g. they just sent a message, the terminal is focused), prioritize responding over continuing background work.

## Bias toward action

Act on your best judgment rather than asking for confirmation.

- Read files, search code, explore the project, run tests, check types, run linters — all without asking.
- Make code changes. Commit when you reach a good stopping point.
- If you're unsure between two reasonable approaches, pick one and go. You can always course-correct.

## Be concise

Keep your text output brief and high-level. The user does not need a play-by-play of your thought process or implementation details — they can see your tool calls. Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones (e.g. "PR created", "tests passing")
- Errors or blockers that change the plan

Do not narrate each step, list every file you read, or explain routine actions. If you can say it in one sentence, don't use three.

## Terminal focus

The user context may include a \`terminalFocus\` field indicating whether the user's terminal is focused or unfocused. Use this to calibrate how autonomous you are:
- **Unfocused**: The user is away. Lean heavily into autonomous action — make decisions, explore, commit, push. Only pause for genuinely irreversible or high-risk actions.
- **Focused**: The user is watching. Be more collaborative — surface choices, ask before committing to large changes, and keep your output concise so it's easy to follow in real time.${BRIEF_PROACTIVE_SECTION && briefToolModule?.isBriefEnabled() ? `\n\n${BRIEF_PROACTIVE_SECTION}` : ''}`
}
```

**位置**: `packages/ccb/src/constants/prompts.ts:861-914`

#### 2.2.4 Tick 标签定义 (`packages/ccb/src/constants/xml.ts:25`)

```typescript
export const TICK_TAG = 'tick'
```

**位置**: `packages/ccb/src/constants/xml.ts:25`

#### 2.2.5 命令注册 (`packages/ccb/src/commands.ts:62-65`)

```typescript
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null
```

**位置**: `packages/ccb/src/commands.ts:62-65`

命令数组中的注册 (`packages/ccb/src/commands.ts:324`)：

```typescript
...(proactive ? [proactive] : []),
```

**位置**: `packages/ccb/src/commands.ts:324`

#### 2.2.6 工具注册 (`packages/ccb/src/tools.ts:25-28`)

```typescript
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null
```

**位置**: `packages/ccb/src/tools.ts:25-28`

工具列表中的注册 (`packages/ccb/src/tools.ts:232`)：

```typescript
...(SleepTool ? [SleepTool] : []),
```

**位置**: `packages/ccb/src/tools.ts:232`

### 2.3 数据流

```
activateProactive() [Stub - 需要实现]
│
▼
Tick 调度器启动 [未实现]
├── 定时生成 <tick_tag> 消息
├── 包含用户当前本地时间
└── 注入到对话流（sessionStorage）
│
▼
模型处理 tick
├── 有事可做 → 使用工具执行 → 可能再次 Sleep
└── 无事可做 → 必须调用 SleepTool
│
▼
SleepTool 等待 [Stub - 需要实现]
│
▼
下一个 tick 到达
```

### 2.4 REPL 集成

#### 2.4.1 状态订阅 (`packages/ccb/src/screens/REPL.tsx:910`)

```typescript
// Track proactive mode for tools dependency - SleepTool filters by proactive state
const proactiveActive = React.useSyncExternalStore(
  proactiveModule?.subscribeToProactiveChanges ?? PROACTIVE_NO_OP_SUBSCRIBE,
  proactiveModule?.isProactiveActive ?? PROACTIVE_FALSE,
);
```

**位置**: `packages/ccb/src/screens/REPL.tsx:910`

#### 2.4.2 Hook 调用 (`packages/ccb/src/screens/REPL.tsx:4784`)

```typescript
useProactive?.({
  isLoading: isLoading || initialMessage !== null,
  queuedCommandsLength: queuedCommands.length,
  hasActiveLocalJsxUI: isShowingLocalJSXCommand,
  isInPlanMode: toolPermissionContext.mode === 'plan',
  onSubmitTick: (prompt: string) => handleIncomingPrompt(prompt, { isMeta: true }),
  onQueueTick: (prompt: string) => enqueue({ mode: 'prompt', value: prompt, isMeta: true }),
});
```

**位置**: `packages/ccb/src/screens/REPL.tsx:4784`

#### 2.4.3 暂停/恢复控制

暂停 (`packages/ccb/src/screens/REPL.tsx:2466`)：
```typescript
proactiveModule?.pauseProactive();
```

恢复 (`packages/ccb/src/screens/REPL.tsx:3740`)：
```typescript
proactiveModule?.resumeProactive();
```

上下文阻塞控制 (`packages/ccb/src/screens/REPL.tsx`（多处以 `setContextBlocked` 控制 Proactive 上下文阻塞）)：
```typescript
proactiveModule?.setContextBlocked(false);
proactiveModule?.setContextBlocked(true);
```

### 2.5 CLI 参数注册 (`packages/ccb/src/main.tsx:5589-5591`)

```typescript
if (feature("PROACTIVE") || feature("KAIROS")) {
  program.addOption(
    new Option("--proactive", "Start in proactive autonomous mode"),
  );
}
```

**位置**: `packages/ccb/src/main.tsx:5589-5591`

### 2.6 启动时激活 (`packages/ccb/src/main.tsx:3321-3342`)

```typescript
if (
  (feature("PROACTIVE") || feature("KAIROS")) &&
  ((options as { proactive?: boolean }).proactive ||
    isEnvTruthy(process.env.CLAUDE_CODE_PROACTIVE)) &&
  !coordinatorModeModule?.isCoordinatorMode()
) {
  const briefVisibility =
    feature("KAIROS") || feature("KAIROS_BRIEF")
      ? (
          require("./tools/BriefTool/BriefTool.js") as typeof import("./tools/BriefTool/BriefTool.js")
        ).isBriefEnabled()
        ? "Call SendUserMessage at checkpoints to mark where things stand."
        : "The user will see any text you output."
      : "The user will see any text you output.";
  
  const proactivePrompt = `\n# Proactive Mode\n\nYou are in proactive mode. Take initiative — explore, act, and make progress without waiting for instructions.\n\nStart by briefly greeting the user.\n\nYou will receive periodic <tick> prompts. These are check-ins. Do whatever seems most useful, or call Sleep if there's nothing to do. ${briefVisibility}`;
  
  appendSystemPrompt = appendSystemPrompt
    ? `${appendSystemPrompt}\n\n${proactivePrompt}`
    : proactivePrompt;
}
```

**位置**: `packages/ccb/src/main.tsx:3321-3342`

### 2.7 Headless 路径激活 (`packages/ccb/src/main.tsx:2863-2864`)

```typescript
// Activate proactive mode BEFORE getTools() so SleepTool.isEnabled()
// (which returns isProactiveActive()) passes and Sleep is included.
// The later REPL-path maybeActivateProactive() calls are idempotent.
maybeActivateProactive(options);
```

**位置**: `packages/ccb/src/main.tsx:2863-2864`

`maybeActivateProactive` 函数 (`packages/ccb/src/main.tsx:6875-6885`)：

```typescript
function maybeActivateProactive(options: unknown): void {
  if (
    (feature("PROACTIVE") || feature("KAIROS")) &&
    ((options as { proactive?: boolean }).proactive ||
      isEnvTruthy(process.env.CLAUDE_CODE_PROACTIVE))
  ) {
    const proactiveModule = require("./proactive/index.js");
    if (!proactiveModule.isProactiveActive()) {
      proactiveModule.activateProactive("command");
    }
  }
}
```

**位置**: `packages/ccb/src/main.tsx:6875-6885`

### 2.8 Session Storage Tick 注入 (`packages/ccb/src/utils/sessionStorage.ts:4892-4912`)

```typescript
if (SKIP_FIRST_PROMPT_PATTERN.test(result)) {
  if (
    (feature('PROACTIVE') || feature('KAIROS')) &&
    result.startsWith(`<${TICK_TAG}>`)
  )
    hasTickMessages = true
  continue
}
// ...
// Proactive sessions have only tick messages — give them a synthetic prompt
// so they're not filtered out by enrichLogs
if ((feature('PROACTIVE') || feature('KAIROS')) && hasTickMessages)
  return 'Proactive session'
```

**位置**: `packages/ccb/src/utils/sessionStorage.ts:4892-4912`

---

## 三、需要补全的内容

| 优先级 | 模块 | 工作量 | 说明 |
|--------|------|--------|------|
| **1** | `packages/ccb/src/proactive/index.ts` | 中 | Tick 调度器、`activate/deactivate` 状态机、`pause/resume`、`subscribeToProactiveChanges` |
| **2** | `packages/ccb/src/tools/SleepTool/SleepTool.tsx` | 小 | 工具执行（等待指定时间后触发 tick）— **文件不存在，需创建** |
| **3** | `packages/ccb/src/commands/proactive.js` | 小 | `/proactive` 斜杠命令处理器 |
| **4** | `packages/ccb/src/proactive/useProactive.ts` | 中 | React hook（REPL 引用但不存在） |

### 3.1 缺失文件列表

```
src/proactive/useProactive.ts       # 被 REPL.tsx:332 引用
src/commands/proactive.js           # 被 commands.ts:64 引用
src/tools/SleepTool/SleepTool.tsx   # 被 tools.ts:27 引用
```

---

## 四、关键设计决策

### 4.1 Tick 驱动

模型通过 `SleepTool` 自行控制唤醒频率，不是外部事件推送。

### 4.2 空操作必须 Sleep

防止 "still waiting" 类空消息浪费 turn 和 token。

### 4.3 Prompt Cache 考量

SleepTool 提示中提到 cache 5 分钟过期，建议平衡等待时间。

### 4.4 Terminal Focus 感知

模型根据用户是否在看终端调整自主程度：
- **Unfocused**: 用户离开，重度自主行动
- **Focused**: 用户观看，更协作，更多询问

---

## 五、使用方式

```bash
# 单独启用 proactive
FEATURE_PROACTIVE=1 bun run dev

# 通过 KAIROS 间接启用
FEATURE_KAIROS=1 bun run dev

# 组合使用
FEATURE_PROACTIVE=1 FEATURE_KAIROS=1 FEATURE_KAIROS_BRIEF=1 bun run dev

# CLI 参数启动
claude --proactive
```

---

## 六、文件索引

| 文件 | 职责 | 行号 |
|------|------|------|
| `packages/ccb/src/proactive/index.ts` | 核心逻辑（stub） | 1-7 |
| `packages/ccb/src/tools/SleepTool/prompt.ts` | SleepTool 工具提示 | 1-18 |
| `packages/ccb/src/constants/prompts.ts` | 自主工作系统提示 | 861-914 |
| `packages/ccb/src/constants/xml.ts` | TICK_TAG 定义 | 25 |
| `packages/ccb/src/screens/REPL.tsx` | REPL tick 集成 | 907-910, 4784, 2466, 3740 |
| `packages/ccb/src/utils/sessionStorage.ts` | Tick 消息注入 | 4892-4912 |
| `packages/ccb/src/commands.ts` | 命令注册 | 62-65, 324 |
| `packages/ccb/src/tools.ts` | 工具注册 | 25-28, 232 |
| `packages/ccb/src/main.tsx` | CLI 参数/激活逻辑 | 5589-5591, 3321-3342, 6875-6885, 2863-2864 |
| `packages/ccb/src/types/textInputTypes.ts` | QueuePriority 类型 | 277-291 |

---

## 七、状态摘要

| 组件 | 状态 |
|------|------|
| 系统提示 | ✅ 完整 |
| SleepTool 提示 | ✅ 完整 |
| 命令布线 | ✅ 完成 |
| 工具布线 | ✅ 完成 |
| REPL 集成 | ✅ 布线完成 |
| CLI 参数 | ✅ 完成 |
| 核心 Tick 调度器 | ❌ Stub。实现风险最高：需要设计不阻塞主 REPL 输入队列的定时唤醒机制、处理用户并发输入时的 race condition、以及终端失焦/聚焦状态的精确检测。
| SleepTool 执行器 | ❌ **缺失（文件不存在）** |
| useProactive Hook | ❌ 缺失 |
| proactive.js 命令 | ❌ 缺失 |

---

**报告生成时间**: 2026-04-09  
**原始文档**: https://ccb.agent-aura.top/docs/features/proactive
