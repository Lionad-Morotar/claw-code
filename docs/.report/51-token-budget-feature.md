# Unit 51 — Token Budget Feature 技术报告

**原始文档**: https://ccb.agent-aura.top/docs/features/token-budget  
**源码仓库**: `packages/ccb/` (子项目)  
**Feature Flag**: `FEATURE_TOKEN_BUDGET=1`

> 注：本章引用的源码路径若无特别说明，均相对于 `packages/ccb/`。


> **源码映射说明**：Token Budget Feature 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。---

## 一、功能概述

TOKEN_BUDGET 允许用户在 prompt 中通过简写语法（如 `+500k`）或自然语言（如 `spend 2M tokens`）指定一个 output token 预算目标。当模型单轮输出尚未用完预算时，系统会在后台自动追加 nudge 消息以提示继续工作，减少用户手动输入的次数。该机制适用于需要长输出的任务，但实际效果取决于模型对重复 nudge 的响应稳定性，并不能保证每次都能精准填满预算。

---

## 二、用户交互语法

### 支持的输入格式

| 格式 | 示例 | 说明 |
|------|------|------|
| 简写（开头） | `+500k` | 输入开头直接写 |
| 简写（结尾） | `帮我重构这个模块 +2m` | 输入末尾追加 |
| 完整语法 | `spend 2M tokens` / `use 1B tokens` | 自然语言嵌入 |

单位支持：`k`（千）、`m`（百万）、`b`（十亿），大小写不敏感。

### 解析层实现

**文件**: `packages/ccb/src/utils/tokenBudget.ts`

三条正则负责解析：

```typescript
// L1-L3
const SHORTHAND_START_RE = /^\s*\+(\d+(?:\.\d+)?)\s*(k|m|b)\b/i

// L4-L7
const SHORTHAND_END_RE = /\s\+(\d+(?:\.\d+)?)\s*(k|m|b)\s*[.!?]?\s*$/i

// L8
const VERBOSE_RE = /\b(?:use|spend)\s+(\d+(?:\.\d+)?)\s*(k|m|b)\s*tokens?\b/i
```

核心函数：

- `parseTokenBudget(text: string): number | null` — L21-L29：按优先级尝试 start / end / verbose 匹配。
- `findTokenBudgetPositions(text: string): Array<{ start, end }>` — L31-L64：返回所有匹配位置，用于输入框高亮。
- `getBudgetContinuationMessage(pct, turnTokens, budget): string` — L66-L73：生成继续工作的 nudge 消息，例如 `"Stopped at 50% of token target (250,000 / 500,000). Keep working — do not summarize."`

测试覆盖见：

- `packages/ccb/src/utils/__tests__/tokenBudget.test.ts`

---

## 三、状态层实现

**文件**: `packages/ccb/src/bootstrap/state.ts`

模块级单例变量追踪当前 turn 的预算状态：

```typescript
// L724-L728
let outputTokensAtTurnStart = 0
export function getTurnOutputTokens(): number {
  return getTotalOutputTokens() - outputTokensAtTurnStart
}

// L724-L730
let currentTurnTokenBudget: number | null = null
export function getCurrentTurnTokenBudget(): number | null {
  return currentTurnTokenBudget
}

// L732-L743
let budgetContinuationCount = 0
export function snapshotOutputTokensForTurn(budget: number | null): void {
  outputTokensAtTurnStart = getTotalOutputTokens()
  currentTurnTokenBudget = budget
  budgetContinuationCount = 0
}
export function getBudgetContinuationCount(): number {
  return budgetContinuationCount
}
export function incrementBudgetContinuationCount(): void {
  budgetContinuationCount++
}
```

`getTotalOutputTokens()` 定义在 L708-L710，从 `STATE.modelUsage` 汇总所有模型的 outputTokens。

会话切换时清空：

```typescript
// L926-L928
outputTokensAtTurnStart = 0
currentTurnTokenBudget = null
budgetContinuationCount = 0
```

---

## 四、决策层实现

**文件**: `packages/ccb/src/query/tokenBudget.ts`

### BudgetTracker

```typescript
// L3-L19
const COMPLETION_THRESHOLD = 0.9
const DIMINISHING_THRESHOLD = 500

export type BudgetTracker = {
  continuationCount: number
  lastDeltaTokens: number
  lastGlobalTurnTokens: number
  startedAt: number
}

export function createBudgetTracker(): BudgetTracker { ... }
```

### checkTokenBudget

```typescript
// L45-L93
export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision
```

**停止条件**（返回 `action: 'stop'`）：
1. 在子 agent 中（`agentId` 存在）— L51
2. 无预算或预算 `<= 0` — L54
3. 当前 turn 产出已达预算的 `90%`（`COMPLETION_THRESHOLD`）— L64
4. **收益递减**：已连续自动续接 `>= 3` 次，且最近两次 delta 均 `< 500 tokens` — L59-L62

**继续条件**（返回 `action: 'continue'`）：
- 未达 90% 且非收益递减 — L70-L75
- 返回的 `nudgeMessage` 通过 `getBudgetContinuationMessage` 格式化

**完成事件**（停止时附带 `completionEvent`）：
- 包含 `continuationCount`、`pct`、`turnTokens`、`budget`、`diminishingReturns`、`durationMs` — L78-L89

---

## 五、主循环集成

**文件**: `packages/ccb/src/query.ts`

### 初始化

```typescript
// L280
const budgetTracker = feature('TOKEN_BUDGET') ? createBudgetTracker() : null
```

### 每轮结束检查

```typescript
// L1311-L1357
if (feature('TOKEN_BUDGET')) {
  const decision = checkTokenBudget(
    budgetTracker!,
    toolUseContext.agentId,
    getCurrentTurnTokenBudget(),
    getTurnOutputTokens(),
  )

  if (decision.action === 'continue') {
    incrementBudgetContinuationCount()
    // L1321-L1322: 打印调试日志
    logForDebugging(
      `Token budget continuation #${decision.continuationCount}: ${decision.pct}% ...`,
    )
    // L1324-L1341: 构建下一状态，注入 user nudge message，continue 回循环顶部
    state = {
      messages: [ ... ],
      transition: { reason: 'token_budget_continuation' },
    }
    continue
  }

  if (decision.completionEvent) {
    // L1346-L1356: 记录完成事件（含 diminishingReturns 标记）
    logEvent('tengu_token_budget_completed', {
      ...decision.completionEvent,
      queryChainId: queryChainIdForAnalytics,
      queryDepth: queryTracking.depth,
    })
  }
}
```

决策为 `stop` 时正常退出循环。

### API task_budget 区分

`query.ts` L193–L197 和 L699–L703 还存在一个**独立的** `taskBudget` 字段（`output_config.task_budget`），属于 Anthropic API Beta 功能（`task-budgets-2026-03-13`），用于让 API 端侧感知剩余预算。该字段与 `+500k` 的 token budget auto-continue 是不同层面的机制：前者是向 API 声明剩余预算的调用参数，后者是本地循环层的自动续接策略。`taskBudget` 通过 `configureTaskBudgetParams` 注入到 API 请求中（见 `src/services/api/claude.ts` L455–L483）。

---

## 六、UI 层实现

### 输入框高亮

**文件**: `packages/ccb/src/components/PromptInput/PromptInput.tsx`

```typescript
// L775-L779
const tokenBudgetTriggers = useMemo(
  () =>
    feature('TOKEN_BUDGET') ? findTokenBudgetPositions(displayedValue) : [],
  [displayedValue],
)

// L912-L918
for (const trigger of tokenBudgetTriggers) {
  highlights.push({
    start: trigger.start,
    end: trigger.end,
    color: 'suggestion',
    priority: 5,
  })
}
```

命中预算语法时，对应文字以 suggestion 颜色高亮。

### Spinner 进度显示

**文件**: `packages/ccb/src/components/Spinner.tsx`

```typescript
// L340-L361
let budgetText: string | null = null
if (feature('TOKEN_BUDGET')) {
  const budget = getCurrentTurnTokenBudget()
  if (budget !== null && budget > 0) {
    const tokens = getTurnOutputTokens()
    if (tokens >= budget) {
      budgetText = `Target: ${formatNumber(tokens)} used (${formatNumber(budget)} min ${figures.tick})`
    } else {
      const pct = Math.round((tokens / budget) * 100)
      const remaining = budget - tokens
      const rate = elapsedSnapshot > 5000 && tokens >= 2000 ? tokens / elapsedSnapshot : 0
      const eta = rate > 0 ? ` · ~${formatDuration(remaining / rate, { mostSignificantOnly: true })}` : ''
      budgetText = `Target: ${formatNumber(tokens)} / ${formatNumber(budget)} (${pct}%)${eta}`
    }
  }
}
```

未完成时显示进度百分比与 ETA；完成后显示 `"used (min ✓)"`。

### REPL 层预算快照与清理

**文件**: `packages/ccb/src/screens/REPL.tsx`

```typescript
// L2482-L2486: 用户取消时清除预算
if (feature('TOKEN_BUDGET')) {
  snapshotOutputTokensForTurn(null)
}

// L3431-L3435: 提交输入时解析预算并快照
if (feature('TOKEN_BUDGET')) {
  const parsedBudget = input ? parseTokenBudget(input) : null
  snapshotOutputTokensForTurn(parsedBudget ?? getCurrentTurnTokenBudget())
}

// L3500-L3515: turn 结束时捕获预算信息用于显示
if (feature('TOKEN_BUDGET')) {
  if (
    getCurrentTurnTokenBudget() !== null &&
    getCurrentTurnTokenBudget()! > 0 &&
    !abortController.signal.aborted
  ) {
    budgetInfo = {
      tokens: getTurnOutputTokens(),
      limit: getCurrentTurnTokenBudget()!,
      nudges: getBudgetContinuationCount(),
    }
  }
  snapshotOutputTokensForTurn(null)
}
```

---

## 七、系统提示注入

**文件**: `packages/ccb/src/constants/prompts.ts`

```typescript
// L539-L551
...(feature('TOKEN_BUDGET')
  ? [
      systemPromptSection(
        'token_budget',
        () =>
          'When the user specifies a token target (e.g., "+500k", "spend 2M tokens", "use 1B tokens"), your output token count will be shown each turn. Keep working until you approach the target — plan your work to fill it productively. The target is a hard minimum, not a suggestion. If you stop early, the system will automatically continue you.',
      ),
    ]
  : []),
```

**关键设计决策**：这段 prompt 被**无条件缓存**（不随 `getCurrentTurnTokenBudget()` 的变化而 toggle）。原因是 `"When the user specifies..."` 的措辞在没有预算时近似空操作；动态开关反而会在每次翻转时产生约 20K token 的 cache miss。

---

## 八、API 附件

**文件**: `packages/ccb/src/utils/attachments.ts`

```typescript
// L3829-L3845
function getOutputTokenUsageAttachment(): Attachment[] {
  if (feature('TOKEN_BUDGET')) {
    const budget = getCurrentTurnTokenBudget()
    if (budget === null || budget <= 0) {
      return []
    }
    return [
      {
        type: 'output_token_usage',
        turn: getTurnOutputTokens(),
        session: getTotalOutputTokens(),
        budget,
      },
    ]
  }
  return []
}
```

该附件每轮 API 调用附带，让模型在上下文中看到自己的产出进度。附件类型列入 `mainThreadAttachments` 中通过 `maybe('output_token_usage', ...)` 生成（L981-L983）。

---

## 九、关键设计决策总结

| 决策 | 说明 |
|------|------|
| **90% 阈值** | `COMPLETION_THRESHOLD = 0.9`，降低最后一轮 nudge 导致远超预算的风险 |
| **收益递减保护** | 连续 3 轮续接且每轮 `< 500 tokens` 时提前终止，防止模型空转消耗 token |
| **子 agent 豁免** | `agentId` 存在时直接跳过预算检查，避免 `AgentTool` 内部子任务重复触发续接 |
| **无条件缓存提示** | `token_budget` 系统提示始终注入，避免动态开关造成的 cache miss |
| **用户取消清预算** | 按 Escape 取消时调用 `snapshotOutputTokensForTurn(null)`，防止残留预算触发不必要的续接 |

---

## 十、Rust 实现覆盖状态

`claw-code`（Rust 重写版）当前**尚未实现** Token Budget 的自动续接（auto-continue / nudge）机制。Rust 代码库中与 token 相关的现有能力集中在**计数、预检与压缩**层面，与上游 TypeScript 的 budget-continuation 属于不同层次的功能：

- **Token 计数**：`api/src/providers/mod.rs` 中的 `estimate_serialized_tokens`、`preflight_message_request` 等函数在请求发送前估算输入 token，防止超出模型上下文窗口。
- **Usage 跟踪**：`runtime/src/usage.rs` 的 `UsageTracker` 维护单轮和累计 token 消耗，并提供费用估算（输入/输出/Cached）。
- **自动压缩**：`runtime/src/compact.rs` 的 `compact_session` 在对话 token 数量超过阈值时自动压缩历史消息，以释放上下文空间。

上述功能由 [`15-token-budget.md`](15-token-budget.md) 详细覆盖。本报告所述的 `TOKEN_BUDGET` 用户语法解析、`+500k` 高亮、nudge 续接循环、`task_budget` API 参数等机制，在 `claw-code` 中暂无对应实现。

---

## 十一、文件索引

| 路径 | 职责 |
|------|------|
| `packages/ccb/src/utils/tokenBudget.ts` | 正则解析、高亮位置计算、续接消息生成 |
| `packages/ccb/src/utils/__tests__/tokenBudget.test.ts` | 解析层单元测试 |
| `packages/ccb/src/bootstrap/state.ts#L708-L743` | Token 计数状态管理 |
| `packages/ccb/src/query/tokenBudget.ts` | 预算决策逻辑（`checkTokenBudget`） |
| `packages/ccb/src/query.ts#L280, L1311-L1357` | 主循环集成 |
| `packages/ccb/src/components/PromptInput/PromptInput.tsx#L775-L918` | 输入框高亮 |
| `packages/ccb/src/components/Spinner.tsx#L340-L361` | Spinner 进度显示 |
| `packages/ccb/src/screens/REPL.tsx#L2482-L2486, L3431-L3435, L3500-L3515` | 预算快照、清理、捕获 |
| `packages/ccb/src/constants/prompts.ts#L539-L551` | 系统提示注入 |
| `packages/ccb/src/utils/attachments.ts#L981-L983, L3829-L3845` | `output_token_usage` 附件构建 |
| `packages/ccb/src/services/api/claude.ts#L455-L483` | API `task_budget` 参数配置（独立机制） |

