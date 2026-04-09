# Unit 52: Context Collapse (`CONTEXT_COLLAPSE` / `HISTORY_SNIP`)

**Feature Flags**: `FEATURE_CONTEXT_COLLAPSE=1`, `FEATURE_HISTORY_SNIP=1`  
**实现状态**: 核心逻辑全部 Stub，布线（plumbing）完整  
**引用计数**: `CONTEXT_COLLAPSE` 20 处 + `HISTORY_SNIP` 16 处 ≈ 36 处集成点

---

## 1. 功能概述

Context Collapse 让模型在上下文窗口接近上限时，自动将旧消息折叠为压缩摘要，释放 token 空间。它不是简单截断，而是通过在后台调用 LLM 生成摘要来保留关键信息。

- **`CONTEXT_COLLAPSE`**: 上下文折叠引擎（自动折叠）
- **`HISTORY_SNIP`**: SnipTool — 手动标记消息进行折叠/修剪

---

## 2. 核心源码地图（精确锚点）

### 2.1 折叠引擎 Stub（接口完整，实现为空）

| 文件 | 职责 | 状态 | 关键行号 |
|------|------|------|----------|
| `packages/ccb/src/services/contextCollapse/index.ts` | 折叠主入口 | **Stub** | `#L1-L67` |
| `packages/ccb/src/services/contextCollapse/operations.ts` | `projectView` 投影 | **Stub** | `#L1-L4` |
| `packages/ccb/src/services/contextCollapse/persist.ts` | 持久化恢复 | **Stub** | `#L1-L3` |

**`index.ts` 的关键接口定义（Stub 实现）**:

```ts
// packages/ccb/src/services/contextCollapse/index.ts#L14-L28
export interface ContextCollapseStats {
  collapsedSpans: number
  collapsedMessages: number
  stagedSpans: number
  health: ContextCollapseHealth
}
export interface CollapseResult { messages: Message[] }
export interface DrainResult { committed: number; messages: Message[] }
```

所有导出函数均为空操作（恒等/返回默认值）：
- `isContextCollapseEnabled()` → `false`  `#L43`
- `applyCollapsesIfNeeded(messages, ...)` → 透传 `messages`  `#L47-L51`
- `recoverFromOverflow(messages, ...)` → `{ committed: 0, messages }`  `#L59-L62`
- `initContextCollapse()` / `resetContextCollapse()` → 空函数  `#L64-L66`

**`projectView` 投影函数**：

```ts
// packages/ccb/src/services/contextCollapse/operations.ts#L3-L4
export const projectView: (messages: Message[]) => Message[] = (messages) => messages;
```

---

### 2.2 与主查询循环 `query.ts` 的集成

`packages/ccb/src/query.ts` 是折叠逻辑最核心的调用点，所有关键调用均通过条件 `require()` 导入：

```ts
// packages/ccb/src/query.ts#L17-L22
const contextCollapse = feature('CONTEXT_COLLAPSE')
  ? (require('./services/contextCollapse/index.js') as typeof import('./services/contextCollapse/index.js'))
  : null
```

**溢出检测处的折叠应用**  
在消息即将发送给 API 之前，先应用自动折叠：

```ts
// packages/ccb/src/query.ts#L436-L446
if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
  const collapseResult = await contextCollapse.applyCollapsesIfNeeded(
    messagesForQuery,
    toolUseContext,
    querySource,
  )
  messagesForQuery = collapseResult.messages
}
```

> 注释解释了设计要点：折叠后的视图是一个**读取时投影**（read-time projection），摘要消息存在于 collapse store 而非 REPL 消息数组中，因此能跨 turn 持久化。

**413 (Prompt Too Long) 恢复路径**  
当 API 返回 413 时，折叠系统尝试紧急释放（drain staged collapses）：

```ts
// packages/ccb/src/query.ts#L1089-L1108
if (
  feature('CONTEXT_COLLAPSE') &&
  contextCollapse &&
  state.transition?.reason !== 'collapse_drain_retry'
) {
  const drained = contextCollapse.recoverFromOverflow(
    messagesForQuery,
    querySource,
  )
  if (drained.committed > 0) {
    const next: State = {
      messages: drained.messages,
      // ...
    }
    yield systemTransitionMessage('collapse_drain_retry')
    return next
  }
}
```

**Stream 中的错误抑制**  
在 SSE 流解析错误时，如果是可恢复的 prompt-too-long，先 withhold（暂存）该错误，待 collapse drain 后再决定是否抛出：

```ts
// packages/ccb/src/query.ts#L796-L812
let withheld = false
if (feature('CONTEXT_COLLAPSE')) {
  if (
    contextCollapse?.isWithheldPromptTooLong(
      message as Message,
      isPromptTooLongMessage,
      querySource,
    )
  ) {
    withheld = true
  }
}
```

**Reactive Compact 与 Collapse 的协同门控**  
为防止 synthetic preempt 在 API 调用前就返回，导致 collapse 的 413 恢复路径被饿死，有一个专门的条件将 `collapseOwnsIt` 考虑在内：

```ts
// packages/ccb/src/query.ts#L609-L640
let collapseOwnsIt = false
if (feature('CONTEXT_COLLAPSE')) {
  collapseOwnsIt =
    (contextCollapse?.isContextCollapseEnabled() ?? false) &&
    isAutoCompactEnabled()
}
// 后续条件中: !collapseOwnsIt
```

---

### 2.3 与 `QueryEngine.ts` 的集成（Snip/HISTORY_SNIP）

`packages/ccb/src/QueryEngine.ts` 管理对话状态和 turn-level 的书账。

**条件导入 snip 模块**：

```ts
// packages/ccb/src/QueryEngine.ts#L122-L130
const snipModule = feature('HISTORY_SNIP')
  ? (require('./services/compact/snipCompact.js') as typeof import('./services/compact/snipCompact.js'))
  : null
const snipProjection = feature('HISTORY_SNIP')
  ? (require('./services/compact/snipProjection.js') as typeof import('./services/compact/snipProjection.js'))
  : null
```

**在 `buildSystemMessages()` 中注入 snip 边界提示**：

```ts
// packages/ccb/src/QueryEngine.ts#L1299-L1311
...(feature('HISTORY_SNIP')
  ? [
      {
        type: 'text',
        text: snipModule!.buildSnipSystemPromptFragment(
          snipProjection!.getSnipBoundaryCount(store.messages),
        ),
      } as TextBlockParam,
    ]
  : []),
```

---

### 2.4 自动压缩 `autoCompact.ts` 与 Collapse 的协作抑制

`packages/ccb/src/services/compact/autoCompact.ts` 中，如果 `CONTEXT_COLLAPSE` 已启用且处于激活状态，则主动抑制 proactive autocompact（避免两者竞争同一个 headroom）：

```ts
// packages/ccb/src/services/compact/autoCompact.ts#L201-L226
if (feature('CONTEXT_COLAPSE')) {
  const { isContextCollapseEnabled } =
    require('../contextCollapse/index.js') as typeof import('../contextCollapse/index.js')
  if (isContextCollapseEnabled()) {
    return false
  }
}
```

此外，`marble_origami`（ctx-agent）源流也禁用了 autocompact，因为 compact 会重置主线程的 collapse 状态：

```ts
// packages/ccb/src/services/compact/autoCompact.ts#L174-L183
if (feature('CONTEXT_COLLAPSE')) {
  if (querySource === 'marble_origami') {
    return false
  }
}
```

---

### 2.5 持久化与会话恢复

折叠状态通过日志条目持久化到磁盘，会话恢复时重载。

**类型定义**（`packages/ccb/src/types/logs.ts`）：

```ts
// packages/ccb/src/types/logs.ts#L255-L282
export type ContextCollapseCommitEntry = {
  type: 'marble-origami-commit'
  sessionId: UUID
  collapseId: string
  summaryUuid: string
  summaryContent: string
  summary: string
  firstArchivedUuid: string
  lastArchivedUuid: string
}

export type ContextCollapseSnapshotEntry = {
  type: 'marble-origami-snapshot'
  sessionId: UUID
  staged: Array<{
    startUuid: string
    endUuid: string
    summary: string
    risk: number
    stagedAt: number
  }>
  armed: boolean
  lastSpawnTokens: number
}
```

**恢复调用点**：

- `packages/ccb/src/utils/sessionRestore.ts#L121-L136` — `restoreSessionStateFromLog()` 路径
- `packages/ccb/src/utils/sessionRestore.ts#L494-L503` — CLI `--continue/--resume` 路径
- `packages/ccb/src/screens/ResumeConversation.tsx#L317-L324` — 交互式 `/resume` 路径
- `packages/ccb/src/utils/sessionStorage.ts#L2306-L2351` / `L2979-L3050` — 磁盘序列化/反序列化逻辑

---

### 2.6 Compact 后的清理 `postCompactCleanup.ts`

在自动或手动 compact 后，需要重置主线程的 context-collapse 状态，防止子 agent compact 污染主线程的 module-level store：

```ts
// packages/ccb/src/services/compact/postCompactCleanup.ts#L31-L50
export function runPostCompactCleanup(querySource?: QuerySource): void {
  const isMainThreadCompact =
    querySource === undefined ||
    querySource.startsWith('repl_main_thread') ||
    querySource === 'sdk'
  resetMicrocompactState()
  if (feature('CONTEXT_COLLAPSE')) {
    if (isMainThreadCompact) {
      (
        require('../contextCollapse/index.js') as typeof import('../contextCollapse/index.js')
      ).resetContextCollapse()
    }
  }
  // ...
}
```

---

### 2.7 UI 集成：TokenWarning & ContextVisualization

**`TokenWarning.tsx`**（实时状态显示）  
在上下文使用警告条中，如果 collapse 模式启用，显示折叠进度：

```ts
// packages/ccb/src/components/TokenWarning.tsx#L25-L72
function CollapseLabel({ upgradeMessage }: { upgradeMessage: string | null }): React.ReactNode {
  const { getStats, subscribe } =
    require('../services/contextCollapse/index.js') as typeof import('../services/contextCollapse/index.js')
  const snapshot = useSyncExternalStore(subscribe, () => {
    const s = getStats()
    return `${s.collapsedSpans}|${s.stagedSpans}|...`
  })
  // 渲染: "x / y summarized"
}
```

同时，该组件在 collapse mode 下会重新计算 `displayPercentLeft`（直接基于 effective window，而不是 autocompact threshold）：

```ts
// packages/ccb/src/components/TokenWarning.tsx#L96-L118
if (feature('CONTEXT_COLLAPSE')) {
  const { isContextCollapseEnabled } = ...
  if (isContextCollapseEnabled()) collapseMode = true
}
if (reactiveOnlyMode || collapseMode) {
  displayPercentLeft = Math.max(0, Math.round(((effectiveWindow - tokenUsage) / effectiveWindow) * 100))
}
```

**`ContextVisualization.tsx`**（`/context` 命令输出）  
在上下文可视化面板的图例中，增加 `CollapseStatus` 子组件：

```ts
// packages/ccb/src/components/ContextVisualization.tsx#L24-L73
function CollapseStatus(): React.ReactNode {
  const { getStats, isContextCollapseEnabled } =
    require('../services/contextCollapse/index.js') as typeof import('../services/contextCollapse/index.js')
  // 显示 summarized spans / staged spans / errors / idle warnings
}
```

---

### 2.8 `/context` 命令中的 API 视图投影

`packages/ccb/src/commands/context/context.tsx` 负责 `/context` 命令，为了让用户看到与 API 实际接收一致的消息视图，它调用 `projectView`：

```ts
// packages/ccb/src/commands/context/context.tsx#L18-L28
function toApiView(messages: Message[]): Message[] {
  let view = getMessagesAfterCompactBoundary(messages)
  if (feature('CONTEXT_COLLAPSE')) {
    const { projectView } =
      require('../../services/contextCollapse/operations.js') as typeof import('../../services/contextCollapse/operations.js')
    view = projectView(view)
  }
  return view
}
```

---

### 2.9 `collapseReadSearch.ts` — Snip 的静默吸收

`packages/ccb/src/utils/collapseReadSearch.ts` 已有完整实现，将 `HISTORY_SNIP` 的 Snip tool 作为"静默吸收操作"处理（不中断搜索/读取折叠组）：

```ts
// packages/ccb/src/utils/collapseReadSearch.ts#L53-L62
const SNIP_TOOL_NAME = feature('HISTORY_SNIP')
  ? (
      require('../tools/SnipTool/prompt.js') as typeof import('../tools/SnipTool/prompt.js')
    ).SNIP_TOOL_NAME
  : null
```

```ts
// packages/ccb/src/utils/collapseReadSearch.ts#L197-L213
if (
  (feature('HISTORY_SNIP') && toolName === SNIP_TOOL_NAME) ||
  (isFullscreenEnvEnabled() && toolName === TOOL_SEARCH_TOOL_NAME)
) {
  return {
    isCollapsible: true,
    // ...
    isAbsorbedSilently: true,
  }
}
```

---

### 2.10 `analyzeContext.ts` 中的保留缓冲区跳过

在上下文分析中，如果 collapse 启用，保留缓冲区（reserved buffer）的显示被跳过，因为 collapse 自己管理 threshold ladder：

```ts
// packages/ccb/src/utils/analyzeContext.ts#L1122-L1136
if (feature('CONTEXT_COLLAPSE')) {
  const { isContextCollapseEnabled } =
    require('../services/contextCollapse/index.js') as typeof import('../services/contextCollapse/index.js')
  if (isContextCollapseEnabled()) {
    skipReservedBuffer = true
  }
}
```

---

### 2.11 初始化与 Rewind 重置

**初始化**：`setup.ts` 在应用启动时调用 `initContextCollapse`：

```ts
// packages/ccb/src/setup.ts#L295-L301
if (feature('CONTEXT_COLLAPSE')) {
  (
    require('./services/contextCollapse/index.js') as typeof import('./services/contextCollapse/index.js')
  ).initContextCollapse()
}
```

**Rewind 重置**：用户在 REPL 中执行 rewind（截断历史）时，必须重置 collapse 状态，否则 staged queue 中会引用已不存在的 UUID：

```ts
// packages/ccb/src/screens/REPL.tsx#L4318-L4330
if (feature('CONTEXT_COLLAPSE')) {
  (
    require('../services/contextCollapse/index.js') as typeof import('../services/contextCollapse/index.js')
  ).resetContextCollapse()
}
```

---

## 3. 缺失模块索引

| 模块 | 期望路径 | 状态 |
|------|----------|------|
| `CtxInspectTool` | `src/tools/CtxInspectTool/CtxInspectTool.js` | **缺失** — 目录不存在 |
| `SnipTool` 实现 | `src/tools/SnipTool/SnipTool.ts` | **缺失** — 目录不存在 |
| `force-snip` 命令 | `src/commands/force-snip.js` | **缺失** — 文件不存在 |

`tools.ts` 中已预留了 `CtxInspectTool` 的条件加载入口：

```ts
// packages/ccb/src/tools.ts#L108-L109
const CtxInspectTool = feature('CONTEXT_COLLAPSE')
  ? require('./tools/CtxInspectTool/CtxInspectTool.js').CtxInspectTool
  : null
```

以及 `commands.ts` 中的 `forceSnip`：

```ts
// packages/ccb/src/commands.ts#L83
const forceSnip = feature('HISTORY_SNIP')
  ? require('./commands/force-snip.js').default
  : null
```

---

## 4. 数据流总览

```
对话持续增长
      │
      ▼
query.ts:L440 检测上下文接近限制
      │
      ├── applyCollapsesIfNeeded(messages, toolUseContext, querySource)
      │      (contextCollapse/index.ts#L47)
      │
      ├── 后台 LLM 调用压缩旧消息（目前 Stub，待实现）
      ├── 保留关键信息（决策、文件路径、错误）
      └── 替换旧消息为压缩摘要
      │
      ├── 413 恢复流: recoverFromOverflow() (query.ts:L1093, L1179)
      │   └── 紧急 drain staged collapses
      │
      ▼
operations.ts:projectView() 过滤/重放折叠后视图
      │
      ▼
API 接收压缩后的消息序列
      │
      ▼
Compact/rewind → resetContextCollapse()
      │
      ▼
会话保存 → contextCollapseCommits / contextCollapseSnapshot
      │
      ▼
会话恢复 → restoreFromEntries()
```

---

## 5. 设计决策摘要

1. **读取时投影**：折叠不是修改 REPL 的 `messages` 数组，而是通过 `projectView()` 在每次 API 调用前重放。这保证了 collapse state 持久且可恢复。
2. **与 autocompact 互斥**：当 collapse 启用时， proactive autocompact 被抑制（`autoCompact.ts` 返回 `false`），避免两者竞争 headroom。但 reactive compact 仍保留作为 413 fallback。
3. **413 恢复优先 drain staged**：API 返回 prompt-too-long 时，先尝试 drain staged collapse，失败后才会 fallback 到 reactive compact/truncation。
4. **子 Agent 隔离**：`runPostCompactCleanup` 和 `autoCompact` 都会判断 `querySource`，防止子 agent（`agent:*` / `marble_origami`）的 compact 污染主线程的 module-level collapse store。
5. **持久化格式**：采用双结构 — `marble-origami-commit`（按顺序重放）和 `marble-origami-snapshot`（last-wins，仅最新一条生效）。

---

## 6. 需要补全的实现清单

| 优先级 | 模块 | 工作量 | 说明 |
|--------|------|--------|------|
| 1 | `services/contextCollapse/index.ts` | 大 | 折叠状态机、后台 LLM spawn、 staged/commit 生命周期 |
| 2 | `services/contextCollapse/operations.ts` | 中 | `projectView()` 的重放逻辑（按 commit log 替换消息范围） |
| 3 | `services/contextCollapse/persist.ts` | 小 | `restoreFromEntries()` 的磁盘反序列化 |
| 4 | `tools/CtxInspectTool/` | 中 | Token 计数内省、已折叠范围查询 |
| 5 | `tools/SnipTool/SnipTool.ts` | 中 | Snip 工具实现 |
| 6 | `commands/force-snip.js` | 小 | `/force-snip` CLI 命令 |

---

*Report generated for docs/.report/52-context-collapse.md*
