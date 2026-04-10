# EXPERIMENTAL_SKILL_SEARCH — 技能语义搜索实现分析

**Unit**: 50  
**Feature Flag**: `EXPERIMENTAL_SKILL_SEARCH`  
**分析日期**: 2026-04-09  
**原始文档**: https://ccb.agent-aura.top/docs/features/experimental-skill-search


> **源码映射说明**：Experimental Skill Search 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。---

## 一、功能概述

EXPERIMENTAL_SKILL_SEARCH 是一个实验性功能，旨在为模型提供根据任务语义自动发现和推荐相关技能的能力。该功能包含两个核心场景：

1. **Turn-0 发现**：用户首次输入时，分析输入内容并推荐相关技能
2. **Inter-turn 发现**：在对话过程中，检测到写操作拐点时预取相关技能

当前实现状态：**8 个 Stub 模块 + 完整布线**。这意味着虽然编译与启动路径已打通，但任何实际语义搜索能力均缺失；若在产品中误开启此 flag，模型将调用一个空名称工具或收到空附件，导致困惑循环。所有核心模块均已占位并完成了与主流程的集成，但具体实现逻辑大多为 Stub（空操作）。

---

## 二、架构与模块状态

### 2.1 模块清单

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| DiscoverSkillsTool | `packages/ccb/src/tools/DiscoverSkillsTool/prompt.ts` | **Stub** | 工具名称为空字符串 |
| 预取 | `packages/ccb/src/services/skillSearch/prefetch.ts` | **Stub** | 3 个函数全部空操作 |
| 远程加载 | `packages/ccb/src/services/skillSearch/remoteSkillLoader.ts` | **Stub** | 返回空结果 |
| 远程状态 | `packages/ccb/src/services/skillSearch/remoteSkillState.ts` | **Stub** | 返回 null/undefined |
| 信号 | `packages/ccb/src/services/skillSearch/signals.ts` | **Stub** | `DiscoverySignal = any` |
| 遥测 | `packages/ccb/src/services/skillSearch/telemetry.ts` | **Stub** | 空操作日志 |
| 本地搜索 | `packages/ccb/src/services/skillSearch/localSearch.ts` | **Stub** | 空操作缓存 |
| 功能检查 | `packages/ccb/src/services/skillSearch/featureCheck.ts` | **Stub** | `isSkillSearchEnabled => false` |

### 2.2 集成点

| 集成点 | 文件 | 状态 | 说明 |
|--------|------|------|------|
| SkillTool 集成 | `packages/ccb/src/tools/SkillTool/SkillTool.ts` | **布线完成** | 动态加载所有远程技能模块 |
| 提示集成 | `packages/ccb/src/constants/prompts.ts` | **布线完成** | DiscoverSkills schema 注入 |
| 附件系统 | `packages/ccb/src/utils/attachments.ts` | **布线完成** | Turn-0 和 Inter-turn 发现 |
| Query 引擎 | `packages/ccb/src/query.ts` | **布线完成** | 预取和收集机制 |
| Tool 上下文 | `packages/ccb/src/Tool.ts` | **布线完成** | `discoveredSkillNames` 字段 |

---

## 三、核心实现分析

### 3.1 Feature Flag 门控

功能通过 `feature('EXPERIMENTAL_SKILL_SEARCH')` 进行门控，该模式支持 DCE（Dead Code Elimination）：

**源码**: `packages/ccb/src/utils/attachments.ts#L95-L102`
```typescript
const skillSearchModules = feature('EXPERIMENTAL_SKILL_SEARCH')
  ? {
      featureCheck:
        require('../services/skillSearch/featureCheck.js') as typeof import('../services/skillSearch/featureCheck.js'),
      prefetch:
        require('../services/skillSearch/prefetch.js') as typeof import('../services/skillSearch/prefetch.js'),
    }
  : null
```

这种设计使得在外部构建中，整个技能搜索模块可以被完全移除，包括 `'skill_discovery'` 字符串字面量。

### 3.2 DiscoverSkillsTool Stub

**源码**: `packages/ccb/src/tools/DiscoverSkillsTool/prompt.ts#L1-L3`
```typescript
// Auto-generated stub — replace with real implementation
export {};
export const DISCOVER_SKILLS_TOOL_NAME: string = '';
```

该工具是当前功能的入口，但目前仅导出空名称。

### 3.3 预取机制 (prefetch.ts)

预取机制设计用于在用户提交输入前分析消息内容，提前搜索相关技能。

**源码**: `packages/ccb/src/services/skillSearch/prefetch.ts#L6-L18`
```typescript
export const startSkillDiscoveryPrefetch: (
  input: string | null,
  messages: Message[],
  toolUseContext: ToolUseContext,
) => Promise<Attachment[]> = (async () => []);

export const collectSkillDiscoveryPrefetch: (
  pending: Promise<Attachment[]>,
) => Promise<Attachment[]> = (async (pending) => pending);

export const getTurnZeroSkillDiscovery: (
  input: string,
  messages: Message[],
  context: ToolUseContext,
) => Promise<Attachment | null> = (async () => null);
```

### 3.4 SkillTool 集成

SkillTool 是执行技能的核心工具，已完整集成 EXPERIMENTAL_SKILL_SEARCH 的远程技能加载逻辑。

#### 3.4.1 动态模块加载

**源码**: `packages/ccb/src/tools/SkillTool/SkillTool.ts#L108-L117`
```typescript
/* eslint-disable @typescript-eslint/no-require-imports */
const remoteSkillModules = feature('EXPERIMENTAL_SKILL_SEARCH')
  ? {
      ...(require('../../services/skillSearch/remoteSkillState.js') as typeof import('../../services/skillSearch/remoteSkillState.js')),
      ...(require('../../services/skillSearch/remoteSkillLoader.js') as typeof import('../../services/skillSearch/remoteSkillLoader.js')),
      ...(require('../../services/skillSearch/telemetry.js') as typeof import('../../services/skillSearch/telemetry.js')),
      ...(require('../../services/skillSearch/featureCheck.js') as typeof import('../../services/skillSearch/featureCheck.js')),
    }
  : null
/* eslint-enable @typescript-eslint/no-require-imports */
```

#### 3.4.2 远程规范技能验证

**源码**: `packages/ccb/src/tools/SkillTool/SkillTool.ts#L382-L398`
```typescript
if (
  feature('EXPERIMENTAL_SKILL_SEARCH') &&
  process.env.USER_TYPE === 'ant'
) {
  const slug = remoteSkillModules!.stripCanonicalPrefix(
    normalizedCommandName,
  )
  if (slug !== null) {
    const meta = remoteSkillModules!.getDiscoveredRemoteSkill(slug)
    if (!meta) {
      return {
        result: false,
        message: `Remote skill ${slug} was not discovered in this session. Use DiscoverSkills to find remote skills first.`,
        errorCode: 6,
      }
    }
    // Discovered remote skill — valid. Loading happens in call().
    return { result: true }
  }
}
```

远程技能使用 `_canonical_<slug>` 命名约定，通过 AKI/GCS 后端加载 SKILL.md 内容。

#### 3.4.3 Telemetry 集成

技能调用时会记录 `was_discovered` 字段，用于追踪技能是否通过发现机制被找到。

**源码**: `packages/ccb/src/tools/SkillTool/SkillTool.ts#L139-L142`
```typescript
const wasDiscoveredField =
  feature('EXPERIMENTAL_SKILL_SEARCH') &&
  remoteSkillModules!.isSkillSearchEnabled()
    ? {
        was_discovered:
          context.discoveredSkillNames?.has(commandName) ?? false,
      }
    : {}
```

### 3.5 Attachments 系统集成

#### 3.5.1 Turn-0 技能发现

在用户首次输入时，`processUserInputAttachments` 会调用 `getTurnZeroSkillDiscovery`。

**源码**: `packages/ccb/src/utils/attachments.ts#L792-L812`
```typescript
...(feature('EXPERIMENTAL_SKILL_SEARCH') &&
skillSearchModules &&
!options?.skipSkillDiscovery
  ? [
      maybe('skill_discovery', async () => {
        const result = await skillSearchModules.prefetch.getTurnZeroSkillDiscovery(
          input,
          messages ?? [],
          context,
        )
        return result ? [result] : []
      }),
    ]
  : [])
```

注意 `skipSkillDiscovery` 门控：当技能被调用时，其 SKILL.md 内容会作为 `input` 传入以提取 `@` 提及，但这些内容不是用户意图，必须避免触发发现查询。

#### 3.5.2 Skill Listing 过滤

当技能搜索激活时，`skill_listing` 附件会过滤为仅包含 bundled + MCP 技能，用户/项目/插件技能通过发现机制获取。

**源码**: `packages/ccb/src/utils/attachments.ts#L2693-L2698`
```typescript
if (
  feature('EXPERIMENTAL_SKILL_SEARCH') &&
  skillSearchModules?.featureCheck.isSkillSearchEnabled()
) {
  allCommands = filterToBundledAndMcp(allCommands)
}
```

### 3.6 Query 引擎集成

Query 引擎负责在每轮对话中预取技能发现结果。

**源码**: `packages/ccb/src/query.ts#L331-L335`
```typescript
const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(
  null,
  messages,
  toolUseContext,
)
```

预取在模型流式响应和工具执行时并发运行，在工具结果收集阶段等待。

**源码**: `packages/ccb/src/query.ts#L1620-L1626`
```typescript
// Inject prefetched skill discovery. collectSkillDiscoveryPrefetch emits
// hidden_by_main_turn — true when the prefetch resolved before this point
// (should be >98% at AKI@250ms / Haiku@573ms vs turn durations of 2-30s).
if (skillPrefetch && pendingSkillPrefetch) {
  const skillAttachments =
    await skillPrefetch.collectSkillDiscoveryPrefetch(pendingSkillPrefetch)
  for (const att of skillAttachments) {
    const msg = createAttachmentMessage(att)
    yield msg
    toolResults.push(msg)
  }
}
```

### 3.7 Tool 上下文

`ToolUseContext` 包含 `discoveredSkillNames` 字段，用于记录会话中通过技能发现机制找到的技能名称。

**源码**: `packages/ccb/src/Tool.ts#L224-L225`
```typescript
/** Skill names surfaced via skill_discovery this session. Telemetry only (feeds was_discovered). */
discoveredSkillNames?: Set<string>
```

### 3.8 系统提示集成

系统提示中包含技能发现的指导说明。

**源码**: `packages/ccb/src/constants/prompts.ts#L334-L342`
```typescript
function getDiscoverSkillsGuidance(): string | null {
  if (
    feature('EXPERIMENTAL_SKILL_SEARCH') &&
    DISCOVER_SKILLS_TOOL_NAME !== null
  ) {
    return `Relevant skills are automatically surfaced each turn as "Skills relevant to your task:" reminders. If you're about to do something those don't cover — a mid-task pivot, an unusual workflow, a multi-step plan — call ${DISCOVER_SKILLS_TOOL_NAME} with a specific description of what you're doing. Skills already visible or loaded are filtered automatically. Skip this if the surfaced skills already cover your next action.`
  }
  return null
}
```

### 3.9 远程技能状态管理

**源码**: `packages/ccb/src/services/skillSearch/remoteSkillState.ts#L2-L3`
```typescript
export function stripCanonicalPrefix(_name: string): string | null { return null; }
export function getDiscoveredRemoteSkill(_slug: string): { url: string } | undefined { return undefined; }
```

### 3.10 功能检查

**源码**: `packages/ccb/src/services/skillSearch/featureCheck.ts#L2-L3`
```typescript
export {};
export const isSkillSearchEnabled: () => boolean = () => false;
```

---

## 四、数据流

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        用户输入/对话轮次                                 │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     EXPERIMENTAL_SKILL_SEARCH 检查                       │
│                   feature('EXPERIMENTAL_SKILL_SEARCH')                   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
                    ▼                               ▼
        ┌───────────────────┐           ┌───────────────────┐
        │     Turn-0        │           │   Inter-turn      │
        │   (blocking)      │           │   (prefetch)      │
        │                   │           │                   │
        │ getTurnZeroSkill  │           │ startSkillDis-    │
        │   Discovery()     │           │ coveryPrefetch()  │
        │                   │           │                   │
        │ attachments.ts    │           │ query.ts          │
        └───────────────────┘           └───────────────────┘
                    │                               │
                    │                               ▼
                    │                   ┌───────────────────┐
                    │                   │ collectSkillDis-  │
                    │                   │ coveryPrefetch()  │
                    │                   └───────────────────┘
                    │                               │
                    └───────────────┬───────────────┘
                                    ▼
                    ┌───────────────────────────────────┐
                    │     AttachmentMessage 注入        │
                    │     type: 'skill_discovery'       │
                    │     content: "Skills relevant     │
                    │              to your task:"       │
                    └───────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────────┐
                    │     模型看到推荐的技能列表         │
                    │     决定是否使用 DiscoverSkills   │
                    │     工具或直接调用 SkillTool      │
                    └───────────────────────────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────────┐
                    │     SkillTool 执行技能            │
                    │     - 本地技能: processPrompt-    │
                    │       SlashCommand                │
                    │     - 远程技能: executeRemote-    │
                    │       Skill (AKI/GCS)             │
                    │     - Fork 技能：executeForked-   │
                    │       Skill (子代理)              │
                    └───────────────────────────────────┘
```

---

## 五、需要补全的内容

| 优先级 | 模块 | 工作量 | 说明 |
|--------|------|--------|------|
| 1 | `DiscoverSkillsTool` | 大 | 语义搜索工具 schema + 执行 |
| 2 | `skillSearch/prefetch.ts` | 中 | 用户输入分析和预取逻辑 |
| 3 | `skillSearch/remoteSkillLoader.ts` | 大 | 远程市场/注册表获取 |
| 4 | `skillSearch/remoteSkillState.ts` | 小 | 已发现技能状态管理 |
| 5 | `skillSearch/localSearch.ts` | 中 | 本地索引构建/查询 |
| 6 | `skillSearch/featureCheck.ts` | 小 | GrowthBook/配置门控 |
| 7 | `skillSearch/signals.ts` | 小 | `DiscoverySignal` 类型定义 |

---

## 六、关键设计决策

1. **预取优化**：在用户提交前就开始搜索，减少首次响应延迟
2. **本地 + 远程双搜索**：本地索引快速匹配 + 远程市场深度搜索
3. **SkillTool 集成**：发现的技能通过 SkillTool 调用，不需要新的调用机制
4. **独立于 MCP_SKILLS**：MCP_SKILLS 从 MCP 服务器发现，EXPERIMENTAL_SKILL_SEARCH 从技能市场发现
5. **DCE 友好设计**：所有技能搜索相关字符串字面量都在 feature-gated 模块内，外部构建可完全移除
6. **Skip 门控**：技能调用时跳过发现，避免 110KB SKILL.md 触发 3.3s 的 AKI 查询
7. **Ant-only 实验**：远程规范技能 (`_canonical_<slug>`) 仅限 ant 用户

---

## 七、源码索引表

| 文件 | 行号范围 | 职责 |
|------|----------|------|
| `packages/ccb/src/tools/DiscoverSkillsTool/prompt.ts` | L1-L3 | DiscoverSkills 工具定义 (Stub) |
| `packages/ccb/src/services/skillSearch/prefetch.ts` | L6-L18 | 预取逻辑 (Stub) |
| `packages/ccb/src/services/skillSearch/remoteSkillLoader.ts` | L2-L17 | 远程加载 (Stub) |
| `packages/ccb/src/services/skillSearch/remoteSkillState.ts` | L2-L3 | 远程状态管理 (Stub) |
| `packages/ccb/src/services/skillSearch/signals.ts` | L2 | 信号类型定义 (Stub) |
| `packages/ccb/src/services/skillSearch/telemetry.ts` | L2-L11 | 遥测日志 (Stub) |
| `packages/ccb/src/services/skillSearch/localSearch.ts` | L1-L2 | 本地搜索 (Stub) |
| `packages/ccb/src/services/skillSearch/featureCheck.ts` | L2-L3 | 功能检查 (Stub) |
| `packages/ccb/src/tools/SkillTool/SkillTool.ts` | L108-L117 | 动态模块加载 |
| `packages/ccb/src/tools/SkillTool/SkillTool.ts` | L139-L142 | was_discovered 遥测 |
| `packages/ccb/src/tools/SkillTool/SkillTool.ts` | L382-L398 | 远程规范技能验证 |
| `packages/ccb/src/tools/SkillTool/SkillTool.ts` | L493-L505 | 远程技能自动授权 |
| `packages/ccb/src/tools/SkillTool/SkillTool.ts` | L606-L614 | 远程技能执行路由 |
| `packages/ccb/src/tools/SkillTool/SkillTool.ts` | L662-L669 | was_discovered 字段注入 |
| `packages/ccb/src/tools/SkillTool/SkillTool.ts` | L968-L1109 | executeRemoteSkill 实现 |
| `packages/ccb/src/utils/attachments.ts` | L95-L102 | skillSearchModules 条件加载 |
| `packages/ccb/src/utils/attachments.ts` | L801-L812 | Turn-0 技能发现 |
| `packages/ccb/src/utils/attachments.ts` | L2693-L2698 | Skill Listing 过滤 |
| `packages/ccb/src/query.ts` | L66-L68 | skillPrefetch 条件加载 |
| `packages/ccb/src/query.ts` | L331-L335 | 预取启动 |
| `packages/ccb/src/query.ts` | L1620-L1631 | 预取结果收集 |
| `packages/ccb/src/constants/prompts.ts` | L87-L93 | DISCOVER_SKILLS_TOOL_NAME 加载 |
| `packages/ccb/src/constants/prompts.ts` | L334-L341 | getDiscoverSkillsGuidance |
| `packages/ccb/src/constants/prompts.ts` | L778-L784 | 子代理系统提示集成 |
| `packages/ccb/src/Tool.ts` | L224-L225 | discoveredSkillNames 上下文字段 |
| `packages/ccb/src/QueryEngine.ts` | L199 | discoveredSkillNames 实例字段 |
| `packages/ccb/src/QueryEngine.ts` | L240 | 会话开始时清除 |
| `packages/ccb/src/QueryEngine.ts` | L376/L524 | 注入 ToolUseContext |
| `packages/ccb/src/screens/REPL.tsx` | L2275 | discoveredSkillNamesRef ref |
| `packages/ccb/src/screens/REPL.tsx` | L2855 | 注入 ToolUseContext |
| `packages/ccb/src/commands/clear/conversation.ts` | L52/L60 | clearConversation 参数 |
| `packages/ccb/src/commands/clear/conversation.ts` | L131 | 清除 discoveredSkillNames |

---

*Unit 50 分析完成*
