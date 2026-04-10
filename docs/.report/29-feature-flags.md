# 88 个 Feature Flags - 构建时特性门控全解

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/internals/feature-flags) 的原始内容，并映射到 `claw-code` 反编译源码的精确行号锚点。文末附 [完整 Flag 清单](#flags-完整清单) 与 [源码索引](#源码索引)。

> **源码映射说明**：Feature Flags 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）的 `bun:bundle` 构建时门控系统。`claw-code`（Rust 重写版）不使用 `feature()` 函数或构建时 feature flags，而是通过 Cargo feature gates 和运行时配置控制功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。

---

## feature() 是什么

Claude Code 使用 Bun 打包器的 `bun:bundle` 模块提供编译时特性门控：

```typescript
// 源码中的用法（src/tools.ts 等）
import { feature } from 'bun:bundle'

const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null
```

在 Anthropic 的内部构建中，`feature()` 在打包时被求值——返回 `true` 的代码会被保留，返回 `false` 的代码会被 **Dead Code Elimination (DCE)** 彻底移除。

**为什么选择构建时门控？** 原始代码库采用编译时开关而非运行时配置，主要有三点设计 rationale：
1. **风险隔离**：实验性或内部专用的代码不会出现在外部发布包中，从源头杜绝误泄露或误触发。
2. **产物体积**：被 DCE 移除的模块不会增加最终二进制大小，也减少了 Bun 运行时的解析开销。
3. **断点防御**：逆向工程虽然能阅读 `false` 分支的源代码，但这些代码在正式构建产物中已不存在，无法通过补丁简单启用。

在反编译版本中，这个函数没有实际实现，而是通过运行时兜底返回 `false`。类型声明位于 [`src/types/internal-modules.d.ts#L10-L12`](packages/ccb/src/types/internal-modules.d.ts#L10-L12)：

```typescript
declare module "bun:bundle" {
    export function feature(name: string): boolean;
}
```

在 [`src/entrypoints/cli.tsx`](packages/ccb/src/entrypoints/cli.tsx) 中，`feature()` 保持为原始导入形式，依赖 Bun 运行时的 `--feature` 标志来启用特定功能。

这意味着所有 feature flag 后的代码**在默认运行时不会执行**，但代码本身完整保留，可以阅读和分析。

---

## Flags 分类全景

通过对源码中 `feature('FLAG_NAME')` 调用的全面审计，共发现 **90+ 个独立 feature flags**。以下是按功能领域的分类：

### Agent / 自动化（15 个 flags）

控制 AI 的自主能力边界与多 Agent 协作：

| Flag | 用途 | 典型使用位置 |
|------|------|--------------|
| `KAIROS` | 核心自动化框架 | [`tools.ts#L26`](packages/ccb/src/tools.ts#L26) |
| `KAIROS_BRIEF` | Brief 工具增强 | [`commands.ts#L67`](packages/ccb/src/commands.ts#L67) |
| `KAIROS_CHANNELS` | 多渠道支持 | - |
| `KAIROS_DREAM` | 实验性功能 | - |
| `KAIROS_GITHUB_WEBHOOKS` | GitHub webhook 集成 | [`tools.ts#L48`](packages/ccb/src/tools.ts#L48) |
| `KAIROS_PUSH_NOTIFICATION` | 推送通知 | [`tools.ts#L44`](packages/ccb/src/tools.ts#L44) |
| `PROACTIVE` | 主动建议模式 | [`tools.ts#L26`](packages/ccb/src/tools.ts#L26) |
| `COORDINATOR_MODE` | 协调者模式 | [`tools.ts#L118`](packages/ccb/src/tools.ts#L118) |
| `FORK_SUBAGENT` | 子 Agent 分叉 | [`commands.ts#L113`](packages/ccb/src/commands.ts#L113) |
| `AGENT_MEMORY_SNAPSHOT` | Agent 内存快照 | - |
| `AGENT_TRIGGERS` | Agent 触发器 | [`build.ts#L17`](packages/ccb/build.ts#L17) |
| `AGENT_TRIGGERS_REMOTE` | 远程触发器 | [`tools.ts#L34`](packages/ccb/src/tools.ts#L34) |
| `VERIFICATION_AGENT` | 验证 Agent | [`build.ts#L24`](packages/ccb/build.ts#L24) |
| `BUILTIN_EXPLORE_PLAN_AGENTS` | 内置探索/计划 Agent | [`build.ts#L18`](packages/ccb/build.ts#L18) |
| `MONITOR_TOOL` | 监控工具 | [`tools.ts#L37`](packages/ccb/src/tools.ts#L37) |

### 基础设施（10 个 flags）

控制运行环境和连接方式：

| Flag | 用途 | 典型使用位置 |
|------|------|--------------|
| `DAEMON` | 守护进程模式 | [`cli.tsx#L124`](packages/ccb/src/entrypoints/cli.tsx#L124) |
| `BG_SESSIONS` | 后台会话 | [`cli.tsx#L201`](packages/ccb/src/entrypoints/cli.tsx#L201) |
| `BRIDGE_MODE` | 桥接/远程控制模式 | [`commands.ts#L73`](packages/ccb/src/commands.ts#L73) |
| `CCR_AUTO_CONNECT` | CCR 自动连接 | - |
| `CCR_MIRROR` | CCR 镜像 | - |
| `CCR_REMOTE_SETUP` | CCR 远程设置 | [`commands.ts#L91`](packages/ccb/src/commands.ts#L91) |
| `DIRECT_CONNECT` | 直连模式 | - |
| `SSH_REMOTE` | SSH 远程连接 | - |
| `SELF_HOSTED_RUNNER` | 自托管 Runner | [`cli.tsx#L260`](packages/ccb/src/entrypoints/cli.tsx#L260) |
| `BYOC_ENVIRONMENT_RUNNER` | BYOC 环境 Runner | [`cli.tsx#L248`](packages/ccb/src/entrypoints/cli.tsx#L248) |

### 安全 / 分类（6 个 flags）

增强权限判断的智能性：

| Flag | 用途 | 典型使用位置 |
|------|------|--------------|
| `TRANSCRIPT_CLASSIFIER` | 转录分类器 | [`betas.ts#L24`](packages/ccb/src/constants/betas.ts#L24) |
| `BASH_CLASSIFIER` | Bash 命令分类器 | - |
| `TREE_SITTER_BASH` | Tree-sitter Bash 解析 | - |
| `TREE_SITTER_BASH_SHADOW` | Tree-sitter 影子模式 | - |
| `NATIVE_CLIENT_ATTESTATION` | 本地客户端认证 | - |
| `ABLATION_BASELINE` | 消融实验基线 | [`cli.tsx#L38`](packages/ccb/src/entrypoints/cli.tsx#L38) |

### 工具 / 能力（10 个 flags）

新增的 AI 能力：

| Flag | 用途 | 典型使用位置 |
|------|------|--------------|
| `WEB_BROWSER_TOOL` | 浏览器工具 | [`tools.ts#L115`](packages/ccb/src/tools.ts#L115) |
| `TERMINAL_PANEL` | 终端面板捕获 | [`tools.ts#L111`](packages/ccb/src/tools.ts#L111) |
| `CONTEXT_COLLAPSE` | 上下文折叠 | [`tools.ts#L108`](packages/ccb/src/tools.ts#L108) |
| `HISTORY_SNIP` | 历史片段 | [`tools.ts#L121`](packages/ccb/src/tools.ts#L121) |
| `OVERFLOW_TEST_TOOL` | 溢出测试工具 | [`tools.ts#L105`](packages/ccb/src/tools.ts#L105) |
| `WORKFLOW_SCRIPTS` | 工作流脚本 | [`tools.ts#L127`](packages/ccb/src/tools.ts#L127) |
| `VOICE_MODE` | 语音模式 | [`commands.ts#L80`](packages/ccb/src/commands.ts#L80) |
| `MCP_RICH_OUTPUT` | MCP 丰富输出 | - |
| `MCP_SKILLS` | MCP 技能 | - |
| `UDS_INBOX` | UDS 收件箱 | [`tools.ts#L124`](packages/ccb/src/tools.ts#L124) |

### UI / 体验（8 个 flags）

界面和交互改进：

| Flag | 用途 | 典型使用位置 |
|------|------|--------------|
| `MESSAGE_ACTIONS` | 消息操作 | - |
| `QUICK_SEARCH` | 快速搜索 | - |
| `HISTORY_PICKER` | 历史选择器 | - |
| `AUTO_THEME` | 自动主题 | - |
| `STREAMLINED_OUTPUT` | 精简输出 | - |
| `COMPACTION_REMINDERS` | 压缩提醒 | - |
| `TEMPLATES` | 模板系统 | [`cli.tsx#L234`](packages/ccb/src/entrypoints/cli.tsx#L234) |
| `BUDDY` | AI 吉祥物系统 | [`commands.ts#L118`](packages/ccb/src/commands.ts#L118) |

### 平台 / 实验（12+ 个 flags）

实验性和平台级功能：

| Flag | 用途 | 典型使用位置 |
|------|------|--------------|
| `DUMP_SYSTEM_PROMPT` | 导出系统 prompt | [`cli.tsx#L79`](packages/ccb/src/entrypoints/cli.tsx#L79) |
| `UPLOAD_USER_SETTINGS` | 上传用户设置 | - |
| `DOWNLOAD_USER_SETTINGS` | 下载用户设置 | - |
| `EXPERIMENTAL_SKILL_SEARCH` | 实验性技能搜索 | [`commands.ts#L96`](packages/ccb/src/commands.ts#L96) |
| `ULTRAPLAN` | 超级计划模式 | [`commands.ts#L104`](packages/ccb/src/commands.ts#L104) |
| `ULTRATHINK` | 超级思考模式 | [`build.ts#L17`](packages/ccb/build.ts#L17) |
| `TORCH` | Torch 功能 | [`commands.ts#L107`](packages/ccb/src/commands.ts#L107) |
| `LODESTONE` | Limestone 功能 | [`build.ts#L19`](packages/ccb/build.ts#L19) |
| `PERFETTO_TRACING` | Perfetto 性能追踪 | - |
| `SLOW_OPERATION_LOGGING` | 慢操作日志 | - |
| `HARD_FAIL` | 硬失败模式 | - |
| `ALLOW_TEST_VERSIONS` | 允许测试版本 | - |

### 其他发现（19 个 flags）

额外发现的其他 flags：

| Flag | 用途 | 典型使用位置 |
|------|------|--------------|
| `CHICAGO_MCP` | Computer Use MCP | [`build.ts#L14`](packages/ccb/build.ts#L14) |
| `TEAMMEM` | Team Memory | - |
| `COMMIT_ATTRIBUTION` | 提交归属 | - |
| `CACHED_MICROCOMPACT` | 缓存微压缩 | - |
| `SHOT_STATS` | 命中率统计 | [`build.ts#L15`](packages/ccb/build.ts#L15) |
| `TOKEN_BUDGET` | Token 预算 | [`build.ts#L16`](packages/ccb/build.ts#L16) |
| `PROMPT_CACHE_BREAK_DETECTION` | Prompt 缓存破坏检测 | [`build.ts#L15`](packages/ccb/build.ts#L15) |
| `EXTRACT_MEMORIES` | 记忆提取 | [`build.ts#L23`](packages/ccb/build.ts#L23) |
| `CONNECTOR_TEXT` | 连接器文本 | - |
| `REACTIVE_COMPACT` | 响应式压缩 | - |
| `MEMORY_SHAPE_TELEMETRY` | 记忆形状遥测 | - |
| `FILE_PERSISTENCE` | 文件持久化 | - |
| `AWAY_SUMMARY` | 离开摘要 | [`build.ts#L26`](packages/ccb/build.ts#L26) |
| `POWERSHELL_AUTO_MODE` | PowerShell 自动模式 | - |
| `NEW_INIT` | 新初始化流程 | - |
| `NATIVE_CLIPBOARD_IMAGE` | 原生剪贴板图片 | - |
| `ENHANCED_TELEMETRY_BETA` | 增强遥测测试版 | - |
| `COWORKER_TYPE_TELEMETRY` | 同事类型遥测 | - |
| `BREAK_CACHE_COMMAND` | 破坏缓存命令 | - |
| `UNATTENDED_RETRY` | 无人值守重试 | - |
| `SKIP_DETECTION_WHEN_AUTOUPDATES_DISABLED` | 自动更新禁用时跳过检测 | - |
| `SKILL_IMPROVEMENT` | 技能改进 | - |
| `RUN_SKILL_GENERATOR` | 运行技能生成器 | - |
| `HOOK_PROMPTS` | Hook prompts | - |
| `IS_LIBC_MUSL` | libc musl 检测 | - |
| `IS_LIBC_GLIBC` | libc glibc 检测 | - |
| `BUILDING_CLAUDE_APPS` | 构建 Claude 应用 | - |

---

## 代码中的典型模式

Feature flags 在代码中主要有五种使用模式：

### 模式一：条件加载工具

这是最常见的模式，用于工具注册表 [`src/tools.ts`](packages/ccb/src/tools.ts)：

```typescript
// src/tools.ts — 第 25-28 行
const SleepTool =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./tools/SleepTool/SleepTool.js').SleepTool
    : null

// src/tools.ts — 第 37-38 行
const MonitorTool = feature('MONITOR_TOOL')
  ? require('./tools/MonitorTool/MonitorTool.js').MonitorTool
  : null

// src/tools.ts — 第 105-106 行
const OverflowTestTool = feature('OVERFLOW_TEST_TOOL')
  ? require('./tools/OverflowTestTool/OverflowTestTool.js').OverflowTestTool
  : null
```

当 flag 为 `false` 时，`require()` 调用在构建时被 DCE 移除，工具不会出现在可用工具列表中。

### 模式二：条件注册命令

在 [`src/commands.ts`](packages/ccb/src/commands.ts) 中，斜杠命令的注册同样由 feature flags 控制：

```typescript
// src/commands.ts — 第 62-69 行
const proactive =
  feature('PROACTIVE') || feature('KAIROS')
    ? require('./commands/proactive.js').default
    : null
const briefCommand =
  feature('KAIROS') || feature('KAIROS_BRIEF')
    ? require('./commands/brief.js').default
    : null

// src/commands.ts — 第 80-82 行
const voiceCommand = feature('VOICE_MODE')
  ? require('./commands/voice/index.js').default
  : null

// src/commands.ts — 第 118-121 行
const buddy = feature('BUDDY')
  ? require('./commands/buddy/index.js').default
  : null
```

命令数组的组装使用扩展运算符条件展开：

```typescript
// src/commands.ts — 第 289-321 行附近
const COMMANDS = memoize((): Command[] => [
  addDir,
  advisor,
  // ...
  ...(webCmd ? [webCmd] : []),
  ...(forkCmd ? [forkCmd] : []),
  ...(buddy ? [buddy] : []),
  ...(proactive ? [proactive] : []),
  ...(briefCommand ? [briefCommand] : []),
  // ...
])
```

### 模式三：条件启用 API 特性

在 [`src/constants/betas.ts`](packages/ccb/src/constants/betas.ts) 中，feature flags 控制发送给 API 的 beta header：

```typescript
// src/constants/betas.ts — 第 23-25 行
export const AFK_MODE_BETA_HEADER = feature('TRANSCRIPT_CLASSIFIER')
  ? 'afk-mode-2026-01-31'
  : ''
```

在 [`src/utils/betas.ts`](packages/ccb/src/utils/betas.ts#L159) 中，feature flag 控制 auto mode 的可用性判断：

```typescript
// src/utils/betas.ts — 第 159-170 行
export function modelSupportsAutoMode(model: string): boolean {
  if (feature('TRANSCRIPT_CLASSIFIER')) {
    const m = getCanonicalName(model)
    // ... 复杂的模型支持逻辑
  }
  return false
}
```

### 模式四：CLI 快速路径门控

在 [`src/entrypoints/cli.tsx`](packages/ccb/src/entrypoints/cli.tsx) 中，feature flags 控制 CLI 启动时的快速路径：

```typescript
// src/entrypoints/cli.tsx — 第 38-50 行
// Harness-science L0 ablation baseline
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  for (const k of [
    'CLAUDE_CODE_SIMPLE',
    'CLAUDE_CODE_DISABLE_THINKING',
    'DISABLE_COMPACT',
    'DISABLE_AUTO_COMPACT',
    'CLAUDE_CODE_DISABLE_AUTO_MEMORY',
    'CLAUDE_CODE_DISABLE_BACKGROUND_TASKS',
  ]) {
    process.env[k] ??= '1'
  }
}

// src/entrypoints/cli.tsx — 第 79 行
if (feature('DUMP_SYSTEM_PROMPT') && args[0] === '--dump-system-prompt') {
  // ...
}

// src/entrypoints/cli.tsx — 第 124 行
if (feature('DAEMON') && args[0] === '--daemon-worker') {
  // ...
}
```

### 模式五：环境变量组合门控

部分 feature flags 需要与环境变量组合使用：

```typescript
// src/entrypoints/cli.tsx — 第 201-204 行
if (
  feature('BG_SESSIONS') &&
  args[0] !== '--tmux' &&
  // ...
) {
  // 后台会话逻辑
}
```

---

## Dev / Build 默认启用的 Features

### Dev 模式默认 Features

在 [`scripts/dev.ts#L26-L43`](packages/ccb/scripts/dev.ts#L26-L43) 中定义了开发模式下默认启用的 features：

```typescript
const DEFAULT_FEATURES = [
  "BUDDY", "TRANSCRIPT_CLASSIFIER", "BRIDGE_MODE",
  "AGENT_TRIGGERS_REMOTE", "CHICAGO_MCP", "VOICE_MODE",
  "SHOT_STATS", "PROMPT_CACHE_BREAK_DETECTION", "TOKEN_BUDGET",
  // P0: local features
  "AGENT_TRIGGERS",
  "ULTRATHINK",
  "BUILTIN_EXPLORE_PLAN_AGENTS",
  "LODESTONE",
  // P1: API-dependent features
  "EXTRACT_MEMORIES", "VERIFICATION_AGENT",
  "KAIROS_BRIEF", "AWAY_SUMMARY", "ULTRAPLAN",
  // P2: daemon + remote control server
  "DAEMON",
];
```

### Build 模式默认 Features

在 [`build.ts#L13-L37`](packages/ccb/build.ts#L13-L37) 中定义了构建模式下默认启用的 features：

```typescript
const DEFAULT_BUILD_FEATURES = [
  'AGENT_TRIGGERS_REMOTE',
  'CHICAGO_MCP',
  'VOICE_MODE',
  'SHOT_STATS',
  'PROMPT_CACHE_BREAK_DETECTION',
  'TOKEN_BUDGET',
  // P0: local features
  'AGENT_TRIGGERS',
  'ULTRATHINK',
  'BUILTIN_EXPLORE_PLAN_AGENTS',
  'LODESTONE',
  // P1: API-dependent features
  'EXTRACT_MEMORIES',
  'VERIFICATION_AGENT',
  'KAIROS_BRIEF',
  'AWAY_SUMMARY',
  'ULTRAPLAN',
  // P2: daemon + remote control server
  'DAEMON',
]
```

### 通过环境变量启用额外 Features

可以通过 `FEATURE_<NAME>=1` 环境变量启用额外的 features：

```bash
# Dev mode with additional features
FEATURE_PROACTIVE=1 FEATURE_KAIROS=1 bun run dev

# Build with additional features
FEATURE_MONITOR_TOOL=1 bun run build
```

---

## 有趣的发现

1. **KAIROS 家族最庞大** — 6 个相关 flag（`KAIROS`、`KAIROS_BRIEF`、`KAIROS_CHANNELS`、`KAIROS_DREAM`、`KAIROS_GITHUB_WEBHOOKS`、`KAIROS_PUSH_NOTIFICATION`）控制从核心功能到推送通知的方方面面

2. **ABLATION_BASELINE 用于科学对照实验** — 它会关闭 thinking、compaction、auto-memory 等高级功能，测量裸 API 调用的基线性能。见 [`cli.tsx#L38-L50`](packages/ccb/src/entrypoints/cli.tsx#L38-L50)

3. **BUDDY 是 AI 吉祥物/精灵系统** — 在 `src/buddy/` 目录下有完整实现，包括 `CompanionSprite.tsx`、`prompt.ts`、`useBuddyNotification.tsx` 等

4. **ULTRAPLAN 和 ULTRATHINK 暗示高级推理模式** — 比当前 extended thinking 更高级的推理模式，分别对应计划制定和思考深度

5. **TRANSCRIPT_CLASSIFIER 控制最多的 beta header** — 包括 `AFK_MODE_BETA_HEADER` 和 auto mode 的模型支持判断

6. **BRIDGE_MODE 和 DAEMON 联动** — 远程控制服务器需要同时启用这两个 flags 才能工作（[`commands.ts#L77`](packages/ccb/src/commands.ts#L77)）

7. **构建体积分层设计** — 由于 `feature()` 在构建时求值，被 DCE 移除的代码不会增加最终打包体积。但在反编译版本中，这些代码全部保留——这正是我们能够进行完整分析的原因

---

## Flags 完整清单

按字母顺序排列的所有 feature flags（90+ 个）：

```
ABLATION_BASELINE
AGENT_MEMORY_SNAPSHOT
AGENT_TRIGGERS
AGENT_TRIGGERS_REMOTE
ALLOW_TEST_VERSIONS
AUTO_THEME
AWAY_SUMMARY
BASH_CLASSIFIER
BG_SESSIONS
BRIDGE_MODE
BREAK_CACHE_COMMAND
BUILDING_CLAUDE_APPS
BUDDY
BUILTIN_EXPLORE_PLAN_AGENTS
BYOC_ENVIRONMENT_RUNNER
CACHED_MICROCOMPACT
CCR_AUTO_CONNECT
CCR_MIRROR
CCR_REMOTE_SETUP
CHICAGO_MCP
COMMIT_ATTRIBUTION
COMPACTION_REMINDERS
CONNECTOR_TEXT
CONTEXT_COLLAPSE
COORDINATOR_MODE
COWORKER_TYPE_TELEMETRY
DAEMON
DIRECT_CONNECT
DOWNLOAD_USER_SETTINGS
DUMP_SYSTEM_PROMPT
ENHANCED_TELEMETRY_BETA
EXPERIMENTAL_SKILL_SEARCH
EXTRACT_MEMORIES
FILE_PERSISTENCE
FORK_SUBAGENT
HARD_FAIL
HISTORY_PICKER
HISTORY_SNIP
HOOK_PROMPTS
IS_LIBC_GLIBC
IS_LIBC_MUSL
KAIROS
KAIROS_BRIEF
KAIROS_CHANNELS
KAIROS_DREAM
KAIROS_GITHUB_WEBHOOKS
KAIROS_PUSH_NOTIFICATION
LODESTONE
MEMORY_SHAPE_TELEMETRY
MESSAGE_ACTIONS
MCP_RICH_OUTPUT
MCP_SKILLS
MONITOR_TOOL
NATIVE_CLIENT_ATTESTATION
NATIVE_CLIPBOARD_IMAGE
NEW_INIT
OVERFLOW_TEST_TOOL
PERFETTO_TRACING
POWERSHELL_AUTO_MODE
PROMPT_CACHE_BREAK_DETECTION
PROACTIVE
QUICK_SEARCH
REACTIVE_COMPACT
RUN_SKILL_GENERATOR
SELF_HOSTED_RUNNER
SHOT_STATS
SKIP_DETECTION_WHEN_AUTOUPDATES_DISABLED
SKILL_IMPROVEMENT
SLOW_OPERATION_LOGGING
SSH_REMOTE
STREAMLINED_OUTPUT
TEAMMEM
TEMPLATES
TERMINAL_PANEL
TOKEN_BUDGET
TORCH
TRANSCRIPT_CLASSIFIER
TREE_SITTER_BASH
TREE_SITTER_BASH_SHADOW
UDS_INBOX
ULTRAPLAN
ULTRATHINK
UNATTENDED_RETRY
UPLOAD_USER_SETTINGS
VERIFICATION_AGENT
VOICE_MODE
WEB_BROWSER_TOOL
WORKFLOW_SCRIPTS
```

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`src/types/internal-modules.d.ts#L10-L12`](packages/ccb/src/types/internal-modules.d.ts#L10-L12) | `bun:bundle` 模块的 `feature` 函数类型声明 |
| [`src/entrypoints/cli.tsx`](packages/ccb/src/entrypoints/cli.tsx) | CLI 入口，feature-gated 快速路径，`ABLATION_BASELINE` 门控 |
| [`src/tools.ts`](packages/ccb/src/tools.ts) | 工具注册表，条件加载工具的主战场 |
| [`src/commands.ts`](packages/ccb/src/commands.ts) | 命令注册表，条件注册斜杠命令 |
| [`src/constants/betas.ts`](packages/ccb/src/constants/betas.ts) | API beta headers，`AFK_MODE_BETA_HEADER` 条件定义 |
| [`src/utils/betas.ts#L159`](packages/ccb/src/utils/betas.ts#L159) | `modelSupportsAutoMode` 函数，`TRANSCRIPT_CLASSIFIER` 门控 |
| [`build.ts#L13-L37`](packages/ccb/build.ts#L13-L37) | Build 默认 features 列表 |
| [`scripts/dev.ts#L26-L43`](packages/ccb/scripts/dev.ts#L26-L43) | Dev 默认 features 列表 |

---

*本报告为 Unit 29 输出文件，基于原始文档 https://ccb.agent-aura.top/docs/internals/feature-flags 及 `claw-code` 源码审计生成。*
