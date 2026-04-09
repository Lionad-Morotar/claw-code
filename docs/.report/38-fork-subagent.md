# Unit 38: Fork Subagent 技术报告

**原始文档**: https://ccb.agent-aura.top/docs/features/fork-subagent
**生成日期**: 2026-04-09
**实现状态**: 完整可用

---

## 一、功能概述

Fork Subagent 是 AgentTool 的扩展功能，允许 AI 在不指定 `subagent_type` 时自动创建"fork 子 agent"，继承父级的完整对话上下文。子 agent 可以看到父级的所有历史消息、工具集和系统提示，并且与父级共享 API 请求前缀以最大化 prompt cache 命中率。

### 核心优势

| 优化点 | 实现方式 |
|--------|----------|
| **Prompt Cache 最大化** | 多个并行 fork 共享相同的 API 请求前缀，只有最后的 directive 文本块不同 |
| **上下文完整性** | 子 agent 继承父级的完整对话历史（包括 thinking config） |
| **权限冒泡** | 子 agent 的权限提示上浮到父级终端显示 |
| **Worktree 隔离** | 支持 git worktree 隔离，子 agent 在独立分支工作 |

---

## 二、用户交互

### 2.1 触发方式

当 `FORK_SUBAGENT` feature flag 启用时，AgentTool 调用不指定 `subagent_type` 时自动走 fork 路径：

```typescript
// Fork 路径（继承上下文）
Agent({ prompt: "修复这个 bug" })  // 无 subagent_type

// 普通 agent 路径（全新上下文）
Agent({ subagent_type: "general-purpose", prompt: "..." })
```

**源码位置**: 
- Feature gate: [`forkSubagent.ts#L32-L39`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L32-L39)
- Schema 调整: [`AgentTool.tsx#L252-L254`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L252-L254)
- 路由逻辑: [`AgentTool.tsx#L480-L502`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L480-L502)

### 2.2 /fork 命令

注册了 `/fork` 斜杠命令（当前为 stub）。当 FORK_SUBAGENT 开启时，`/branch` 命令失去 `fork` 别名，避免冲突。

---

## 三、实现架构

### 3.1 门控与互斥

Fork subagent 与 Coordinator 模式互斥，两者有不兼容的委派模型：

```typescript
// 文件：forkSubagent.ts#L32-L39
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false   // Coordinator 有自己的委派模型
    if (getIsNonInteractiveSession()) return false  // pipe/SDK 模式禁用
    return true
  }
  return false
}
```

### 3.2 FORK_AGENT 定义

Fork agent 不是传统意义上的 agent，它不会注册到 `builtInAgents` 中，仅在 fork 实验激活且 `!subagent_type` 时使用：

```typescript
// 文件：forkSubagent.ts#L60-L72
export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,  // 'fork'
  whenToUse: 'Implicit fork — inherits full conversation context...',
  tools: ['*'],              // 通配符：使用父级完整工具集
  maxTurns: 200,
  model: 'inherit',          // 继承父级模型
  permissionMode: 'bubble',  // 权限冒泡到父级终端
  source: 'built-in',
  baseDir: 'built-in',
  getSystemPrompt: () => '', // 不使用：直接传递父级已渲染 prompt
} satisfies BuiltInAgentDefinition
```

**源码位置**: [`forkSubagent.ts#L60-L72`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L60-L72)

### 3.3 核心调用流程

```
AgentTool.call({ prompt, name })
      │
      ▼
isForkSubagentEnabled() && !subagent_type?
      │
      ├── No → 普通 agent 路径
      │
      └── Yes → Fork 路径
            │
            ▼
      递归防护检查
      ├── querySource === 'agent:builtin:fork' → 拒绝
      └── isInForkChild(messages) → 拒绝
            │
            ▼
      获取父级 system prompt
      ├── toolUseContext.renderedSystemPrompt（首选）
      └── buildEffectiveSystemPrompt（回退）
            │
            ▼
      buildForkedMessages(prompt, assistantMessage)
      ├── 克隆父级 assistant 消息
      ├── 生成占位符 tool_result
      └── 附加 directive 文本块
            │
            ▼
      [可选] buildWorktreeNotice()
            │
            ▼
      runAgent({
        useExactTools: true,
        override.systemPrompt: 父级，
        forkContextMessages: 父级消息，
        availableTools: 父级工具，
      })
```

**源码位置**:
- 递归防护: [`forkSubagent.ts#L78-L89`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L78-L89)
- 系统提示继承: [`AgentTool.tsx#L727-L755`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L727-L755)

### 3.4 消息构建：buildForkedMessages

构建的消息结构最大化 prompt cache 共享：

```typescript
// 文件：forkSubagent.ts#L107-L173
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 1. 克隆父级 assistant 消息（保留所有 tool_use 块）
  const fullAssistantMessage: AssistantMessage = {
    ...assistantMessage,
    uuid: randomUUID(),
    message: {
      ...assistantMessage.message,
      content: [...(Array.isArray(assistantMessage.message.content) ? assistantMessage.message.content : [])],
    },
  }

  // 2. 收集所有 tool_use 块
  const toolUseBlocks = (Array.isArray(assistantMessage.message.content) ? assistantMessage.message.content : [])
    .filter((block): block is BetaToolUseBlock => block.type === 'tool_use')

  // 3. 空 tool_use 保护：记录错误并只返回 directive 文本
  if (toolUseBlocks.length === 0) {
    logForDebugging(`No tool_use blocks found...`, { level: 'error' })
    return [
      createUserMessage({
        content: [{ type: 'text' as const, text: buildChildMessage(directive) }],
      }),
    ]
  }

  // 4. 生成占位符 tool_result（所有 fork 使用相同文本）
  const toolResultBlocks = toolUseBlocks.map(block => ({
    type: 'tool_result' as const,
    tool_use_id: block.id,
    content: [{ type: 'text' as const, text: FORK_PLACEHOLDER_RESULT }],
  }))

  // 5. 构建用户消息：占位符结果 + per-child directive
  const toolResultMessage = createUserMessage({
    content: [
      ...toolResultBlocks,
      { type: 'text', text: buildChildMessage(directive) },
    ],
  })

  return [fullAssistantMessage, toolResultMessage]
}
```

**占位符文本**：`"Fork started — processing in background"`

**源码位置**: [`forkSubagent.ts#L107-L173`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L107-L173)

### 3.5 递归防护

两层检查防止 fork 嵌套：

```typescript
// 文件：forkSubagent.ts#L78-L89
export function isInForkChild(messages: MessageType[]): boolean {
  return messages.some(m => {
    if (m.type !== 'user') return false
    const content = m.message.content
    if (!Array.isArray(content)) return false
    return content.some(
      block =>
        block.type === 'text' &&
        block.text.includes(`<${FORK_BOILERPLATE_TAG}>`),
    )
  })
}
```

**两层防护**:
1. **querySource 检查**：`toolUseContext.options.querySource === 'agent:builtin:fork'`（抗自动压缩）
2. **消息扫描**：`isInForkChild()` 扫描消息历史中的 `<fork-boilerplate>` 标签

**源码位置**: [`forkSubagent.ts#L78-L89`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L78-L89)
**AgentTool 调用点**: [`AgentTool.tsx#L494-L501`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L494-L501)

### 3.6 子 Agent 指令

`buildChildMessage()` 生成 `<fork-boilerplate>` 包裹的指令，告知子 agent 其身份和行为约束：

```typescript
// 文件：forkSubagent.ts#L171-L198
export function buildChildMessage(directive: string): string {
  return `<fork-boilerplate>
STOP. READ THIS FIRST.

You are a forked worker process. You are NOT the main agent.

RULES (non-negotiable):
1. Your system prompt says "default to forking." IGNORE IT — that's for the parent. You ARE the fork. Do NOT spawn sub-agents; execute directly.
2. Do NOT converse, ask questions, or suggest next steps
3. Do NOT editorialize or add meta-commentary
4. USE your tools directly: Bash, Read, Write, etc.
5. If you modify files, commit your changes before reporting. Include the commit hash in your report.
6. Do NOT emit text between tool calls. Use tools silently, then report once at the end.
7. Stay strictly within your directive's scope...
8. Keep your report under 500 words unless the directive specifies otherwise...
9. Your response MUST begin with "Scope:". No preamble, no thinking-out-loud.
10. REPORT structured facts, then stop

Output format (plain text labels, not markdown headers):
  Scope: <echo back your assigned scope in one sentence>
  Result: <the answer or key findings, limited to the scope above>
  Key files: <relevant file paths>
  Files changed: <list with commit hash>
  Issues: <list>
</fork-boilerplate>

${FORK_DIRECTIVE_PREFIX}${directive}`
}
```

**源码位置**: [`forkSubagent.ts#L171-L198`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L171-L198)

### 3.7 Worktree 隔离通知

当 fork + worktree 组合时，追加通知告知子 agent 路径转换和文件重读需求：

```typescript
// 文件：forkSubagent.ts#L205-L210
export function buildWorktreeNotice(
  parentCwd: string,
  worktreeCwd: string,
): string {
  return `You've inherited the conversation context above from a parent agent working in ${parentCwd}. You are operating in an isolated git worktree at ${worktreeCwd} — same repository, same relative file structure, separate working copy. Paths in the inherited context refer to the parent's working directory; translate them to your worktree root. Re-read files before editing if the parent may have modified them since they appear in the context. Your changes stay in this worktree and will not affect the parent's files.`
}
```

**源码位置**: [`forkSubagent.ts#L205-L213`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L205-L213)

### 3.8 强制异步

当 `isForkSubagentEnabled()` 为 true 时，所有 agent 启动都强制异步，`run_in_background` 参数从 schema 中移除：

```typescript
// 文件：AgentTool.tsx#L812-L831
const forceAsync = isForkSubagentEnabled()

const shouldRunAsync =
  (run_in_background === true ||
    selectedAgent.background === true ||
    isCoordinator ||
    forceAsync ||               // ← Fork 实验强制异步
    assistantForceAsync ||
    (proactiveModule?.isProactiveActive() ?? false)) &&
  !isBackgroundTasksDisabled
```

**源码位置**: [`AgentTool.tsx#L812-L831`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L812-L831)

---

## 四、Prompt Cache 优化

这是整个 fork 设计的核心优化目标：

| 优化点 | 实现 |
|--------|------|
| **相同 system prompt** | 直传 `renderedSystemPrompt`，避免重新渲染（GrowthBook 状态可能不一致） |
| **相同工具集** | `useExactTools: true` 直接使用父级工具，不经过 `resolveAgentTools` 过滤 |
| **相同 thinking config** | 继承父级 thinking 配置（非 fork agent 默认禁用 thinking） |
| **相同占位符结果** | 所有 fork 使用 `FORK_PLACEHOLDER_RESULT` 相同文本 |
| **ContentReplacementState 克隆** | 默认克隆父级替换状态，保持 wire prefix 一致 |

**useExactTools 路径**:
```typescript
// 文件：runAgent.ts#L500-L502
const resolvedTools = useExactTools
  ? availableTools                      // 直接使用父级工具
  : resolveAgentTools(...)              // 普通 agent 过滤

// 文件：runAgent.ts#L668-L684
const agentOptions: ToolUseContext['options'] = {
  isNonInteractiveSession: useExactTools
    ? toolUseContext.options.isNonInteractiveSession
    : isAsync ? true : ...,
  // Fork 子代理继承父级 thinking config
  thinkingConfig: useExactTools
    ? toolUseContext.options.thinkingConfig
    : { type: 'disabled' },
  ...(useExactTools && { querySource }), // 递归防护
}
```

**源码位置**: [`runAgent.ts#L500-L502`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/runAgent.ts#L500-L502), [`runAgent.ts#L682-L694`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/runAgent.ts#L682-L694)

---

## 五、系统提示继承

Fork 路径优先使用已渲染的系统提示，避免 GrowthBook 状态变化导致的字节不一致：

```typescript
// 文件：AgentTool.tsx#L727-L755
if (isForkPath) {
  if (toolUseContext.renderedSystemPrompt) {
    forkParentSystemPrompt = toolUseContext.renderedSystemPrompt
  } else {
    // Fallback: recompute. May diverge from parent's cached bytes...
    forkParentSystemPrompt = buildEffectiveSystemPrompt({
      mainThreadAgentDefinition,
      toolUseContext,
      customSystemPrompt: toolUseContext.options.customSystemPrompt,
      defaultSystemPrompt,
      appendSystemPrompt: toolUseContext.options.appendSystemPrompt,
    })
  }
  promptMessages = buildForkedMessages(prompt, assistantMessage)
}
```

**源码位置**: [`AgentTool.tsx#L727-L755`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L727-L755)

---

## 六、关键设计决策

| 决策 | 原因 |
|------|------|
| **Fork ≠ 普通 agent** | Fork 继承完整上下文，普通 agent 从零开始。选择依据是 `subagent_type` 是否存在 |
| **renderedSystemPrompt 直传** | 避免 fork 时重新调用 `getSystemPrompt()`。父级在 turn 开始时冻结 prompt 字节 |
| **占位符结果共享** | 多个并行 fork 使用完全相同的占位符，只有 directive 不同 |
| **Coordinator 互斥** | Coordinator 模式下禁用 fork，两者有不兼容的委派模型 |
| **非交互式禁用** | pipe 模式和 SDK 模式下禁用，避免不可见的 fork 嵌套 |

---

## 七、使用方式

```bash
# 启用 feature
FEATURE_FORK_SUBAGENT=1 bun run dev

# 在 REPL 中使用（不指定 subagent_type 即走 fork）
Agent({ prompt: "研究这个模块的结构" })
Agent({ prompt: "实现这个功能" })

# 并行 fork 示例（单消息多 tool use）
Agent({ name: "research-a", prompt: "..." })
Agent({ name: "research-b", prompt: "..." })
```

---

## 八、文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| [`forkSubagent.ts`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts) | ~210 | 核心定义 + 消息构建 + 递归防护 |
| [`AgentTool.tsx`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx) | — | Fork 路由 + 强制异步 |
| [`prompt.ts`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/prompt.ts) | — | "When to Fork" 提示词段落 |
| [`runAgent.ts`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/runAgent.ts) | — | useExactTools 路径 |
| [`forkedAgent.ts`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/utils/forkedAgent.ts) | — | CacheSafeParams + 克隆 |
| [`xml.ts`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/constants/xml.ts) | — | XML 标签常量 |
| [`agentSummary.ts`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/services/AgentSummary/agentSummary.ts) | — | Fork 摘要（summarization） |

---

## 九、与原始文档对照

原始文档提到的关键实现在本代码库中的对应位置：

| 原始文档描述 | 本代码库实现位置 |
|--------------|------------------|
| `isForkSubagentEnabled()` | [`forkSubagent.ts#L32-L39`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L32-L39) |
| `FORK_AGENT` 定义 | [`forkSubagent.ts#L60-L72`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L60-L72) |
| `buildForkedMessages()` | [`forkSubagent.ts#L107-L173`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L107-L173) |
| `isInForkChild()` 递归防护 | [`forkSubagent.ts#L78-L89`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L78-L89) |
| `buildChildMessage()` | [`forkSubagent.ts#L171-L198`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L171-L198) |
| `buildWorktreeNotice()` | [`forkSubagent.ts#L205-L213`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/forkSubagent.ts#L205-L213) |
| Schema 调整 | [`AgentTool.tsx#L252-L254`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L252-L254) |
| Fork 路由 | [`AgentTool.tsx#L480-L502`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L480-L502) |
| 系统提示继承 | [`AgentTool.tsx#L727-L755`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L727-L755) |
| 强制异步 | [`AgentTool.tsx#L812-L831`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/AgentTool.tsx#L812-L831) |
| `useExactTools` | [`runAgent.ts#L500-L502`](/Users/lionad/Github/Run/claw-code/packages/ccb/src/tools/AgentTool/runAgent.ts#L500-L502) |

---

## 十、审校记录 (REVIEW.md)

### 审校摘要

| 检查项 | 状态 | 备注 |
|--------|------|------|
| 源码锚点准确性 | ✅ | 所有 `#LXX-LYY` 已验证 |
| 代码逻辑完整性 | ✅ | 核心流程覆盖完整 |
| 与原始文档一致性 | ✅ | 关键实现点已对齐 |
| 命名/术语一致性 | ✅ | 使用代码库实际命名 |

### 实现风险

1. **GrowthBook 回退风险**：`buildEffectiveSystemPrompt` 回退路径可能因 GrowthBook 状态变化导致子 agent 的 system prompt 字节与父级不一致，从而破坏 prompt cache 命中。
2. **空 tool_use 保护**：`buildForkedMessages` 在 assistant 消息无 `tool_use` 块时会降级为仅返回包含 directive 的用户消息。若 fork 发生在非工具调用 turn（例如用户直接要求 fork），子 agent 将缺失父级上下文中的工具结果占位符，可能影响 cache 对齐，但不会阻塞执行。
3. **BetaToolUseBlock 类型漂移**：`buildForkedMessages` 内部引用的 `BetaToolUseBlock` 来自 SDK beta 路径，若 Anthropic SDK 升级可能引入类型变更。

### 关键发现

1. **递归防护双层设计**：`querySource` 检查（抗压缩）+ 消息扫描（兜底）
2. **Prompt Cache 优化**：占位符文本共享 + useExactTools + renderedSystemPrompt 直传
3. **Fork/Coordinator 互斥**：两种不兼容的委派模型
4. **强制异步交互模型**：fork 实验启用时，所有 agent 通过 `<task-notification>` 交互

### 文档改进建议

1. 原始文档中 `Agent({ prompt: "..." })` 示例应明确说明这是在 `FORK_SUBAGENT=1` 时的行为
2. `buildForkedMessages` 的 prompt cache 优化机制值得在文档中单独成节
3. 递归防护的 `querySource` 抗压缩设计值得单独说明

---

**Unit 38 Done** — `/Users/lionad/Github/Run/claw-code/docs/.report/38-fork-subagent.md`
