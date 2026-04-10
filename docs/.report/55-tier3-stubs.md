# Unit 55: Tier3 Stubs 技术报告

**原始页面**: https://ccb.agent-aura.top/docs/features/tier3-stubs  
**报告生成**: 2026-04-09  
**状态**: Tier 3 — 纯 Stub / N/A 低优先级 Feature 概览  
**源码位置**: `packages/ccb/src/` (子项目)


> **源码映射说明**：Tier3 Stubs 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。---

## 一、概览

本文档汇总 claw-code 代码库中所有 Tier 3 feature。这些功能分为三类：

1. **纯 Stub** — 所有函数返回空值或空实现
2. **N/A 内部基础设施** — Anthropic 内部使用，外部无法运行
3. **引用量极低的辅助功能** — 需要大量工作才能实现或尚在概念阶段

---

## 二、核心 Tier 3 Feature 源码分析

### 2.1 MONITOR_TOOL — 纯 Stub

**状态**: Stub  
**引用数**: 13  
**类别**: 工具

**源码位置**: `packages/ccb/src/tools/MonitorTool/MonitorTool.ts`

```typescript
// Auto-generated stub — replace with real implementation
export {};
export const MonitorTool: Record<string, unknown> = {};
```

**分析**: 完全的 auto-generated stub 文件，仅导出空对象。设计用途是文件/进程监控与变更通知，但当前无任何实现。

**引用点**:
- `packages/ccb/src/tools.ts#L37-L38` — 条件加载
- `packages/ccb/src/tools.ts#L235` — 加入工具列表
- `packages/ccb/src/tools/AgentTool/runAgent.ts#L849` — feature 检查
- `packages/ccb/src/tools/BashTool/prompt.ts#L312-L320` — prompt 注入

---

### 2.2 UDS_INBOX — Unix 域套接字消息通信

**状态**: Stub (部分实现)  
**引用数**: 17  
**类别**: 消息通信

**核心实现**: `packages/ccb/src/utils/udsClient.ts`

```typescript
// Auto-generated stub — replace with real implementation
export const sendToUdsSocket: (target: string, message: string) => Promise<void> = async () => {};
export const listAllLiveSessions: () => Promise<Array<{ kind?: string; sessionId?: string }>> = async () => [];
```

**分析**:
- `sendToUdsSocket()` — 空实现，设计用于向 UDS socket 发送消息
- `listAllLiveSessions()` — 返回空数组，设计用于列出所有活跃会话

**集成点**:
- `packages/ccb/src/tools/SendMessageTool/SendMessageTool.ts#L72` — 输入 schema 描述扩展
- `packages/ccb/src/tools/SendMessageTool/SendMessageTool.ts#L586,L631,L658,L685,L742` — 消息路由逻辑
- `packages/ccb/src/tools/SendMessageTool/prompt.ts#L6,L10` — prompt 扩展
- `packages/ccb/src/utils/concurrentSessions.ts#L86` — PID 文件写入 messagingSocketPath
- `packages/ccb/src/utils/conversationRecovery.ts#L497` — 动态导入

**ListPeersTool**: 引用于 `packages/ccb/src/tools.ts#L124-L125`，但目录不存在 — 预期为独立工具文件

---

### 2.3 BG_SESSIONS — 后台会话管理

**状态**: 部分实现  
**引用数**: 11  
**类别**: 会话管理

**核心实现**: `packages/ccb/src/utils/concurrentSessions.ts` (204 行)

**已实现功能**:
- `registerSession()` — 写入 PID 文件到 `~/.claude/sessions/<pid>.json` #L59-L109
- `updateSessionName()` — 更新会话名称 #L131-L136
- `updateSessionBridgeId()` — 记录 Remote Control session ID #L144-L148
- `updateSessionActivity()` — 推送活动状态 #L155-L161
- `countConcurrentSessions()` — 统计并发会话数 #L168-L204
- `isBgSession()` — 判断是否为后台会话 #L44-L46

**环境门控**:
```typescript
function envSessionKind(): SessionKind | undefined {
  if (feature('BG_SESSIONS')) {
    const k = process.env.CLAUDE_CODE_SESSION_KIND
    if (k === 'bg' || k === 'daemon' || k === 'daemon-worker') return k
  }
  return undefined
}
```

**CLI 入口**: `packages/ccb/src/entrypoints/cli.tsx` — 处理 `--bg`, `ps`, `logs`, `attach`, `kill` 等命令

---

### 2.4 CHICAGO_MCP — 内部 MCP 基础设施

**状态**: N/A (Anthropic 内部)  
**引用数**: 16  
**类别**: 内部基础设施

**分析**: Computer Use 功能的 feature flag，控制 MCP server 注册和清理逻辑。

**关键引用**:
- `packages/ccb/src/services/mcp/client.ts#L242,L246,L927,L1985` — ComputerUseWrapper 集成
- `packages/ccb/src/services/mcp/config.ts#L641,L1513` — 内置 server 配置
- `packages/ccb/src/utils/computerUse/wrapper.tsx#L15` — 运行时使能控制
- `packages/ccb/src/utils/computerUse/cleanup.ts#L24` — 清理逻辑
- `packages/ccb/src/state/AppStateStore.ts#L258` — AppState 计算机使用状态
- `packages/ccb/src/query.ts#L1036,L1492` — 工具调用权限检查
- `packages/ccb/src/query/stopHooks.ts#L164` — stop hooks 集成
- `packages/ccb/src/entrypoints/cli.tsx#L108` — CLI 快速路径
- `packages/ccb/src/main.tsx#L2355,L2515` — main.tsx 命令处理
- `packages/ccb/src/services/analytics/metadata.ts#L130` — 遥测标记

**实际实现**: `packages/@ant/computer-use-mcp/` 包含完整的 MCP server 实现（macOS + Windows）

---

### 2.5 LODESTONE — 内部基础设施

**状态**: N/A (Anthropic 内部)  
**引用数**: 6  
**类别**: 内部基础设施

**关键引用**:
- `packages/ccb/src/main.tsx#L968,L5474` — 主 CLI 逻辑
- `packages/ccb/src/utils/backgroundHousekeeping.ts#L10-L11,L39-L40` — 深度链接协议注册
- `packages/ccb/src/utils/deepLink/terminalPreference.ts#L6` — 终端偏好设置
- `packages/ccb/src/utils/settings/types.ts#L815` — 设置类型
- `packages/ccb/src/interactiveHelpers.tsx#L244` — 交互式助手

**功能**: 深链接协议注册 (`claude-code://` URI scheme)，用于从浏览器/其他应用启动 Claude Code

---

### 2.6 EXTRACT_MEMORIES — 自动记忆提取

**状态**: 完整实现 (feature-gated)  
**引用数**: 7  
**类别**: 记忆

**源码位置**: `packages/ccb/src/services/extractMemories/extractMemories.ts` (615 行)

**核心功能**:
- `initExtractMemories()` — 初始化提取系统 #L296
- `executeExtractMemories()` — 在 query loop 结束时运行 #L598
- `drainPendingExtraction()` — 等待所有进行中的提取 #L611
- `createAutoMemCanUseTool()` — 创建受限的 canUseTool 函数 #L171

**工作流程**:
1. 每个 turn 结束时通过 `handleStopHooks` 触发
2. 使用 `runForkedAgent()` 创建 forked agent（共享 prompt cache）
3. 读取现有记忆 manifest #L398
4. 调用 forked agent 执行提取 #L415
5. 写入记忆文件到 `~/.claude/projects/<path>/memory/`

**门控条件**:
- `feature('EXTRACT_MEMORIES')` 启用
- `tengu_passport_quail` GrowthBook flag 为 true
- 非远程模式
- 非子 agent

**引用点**:
- `packages/ccb/src/utils/backgroundHousekeeping.ts#L7-L8,L34-L35` — 初始化
- `packages/ccb/src/cli/print.ts#L368-L369,L961-L962` —  draining
- `packages/ccb/src/query/stopHooks.ts#L42-L43,L142,L149` — stop hooks 集成

---

### 2.7 SHOT_STATS — 逐 prompt 统计

**状态**: 完整实现 (feature-gated)  
**引用数**: 10  
**类别**: 统计

**核心实现**:
- `packages/ccb/src/utils/stats.ts#L84,L131,L214,L364,L610,L829` — 统计计算
- `packages/ccb/src/utils/statsCache.ts#L137,L172,L194,L196,L384` — 缓存迁移
- `packages/ccb/src/components/Stats.tsx#L317-L341,L463-L479` — UI 渲染

**功能**: 收集和展示 prompt 分布统计（按 token 区间分桶）

---

### 2.8 TEMPLATES — 项目/提示模板系统

**状态**: Stub (部分集成)  
**引用数**: 6  
**类别**: 项目管理

**引用点**:
- `packages/ccb/src/utils/markdownConfigLoader.ts#L35` — markdown 配置加载
- `packages/ccb/src/utils/permissions/filesystem.ts#L1521` — 权限系统
- `packages/ccb/src/entrypoints/cli.tsx#L234` — CLI 命令
- `packages/ccb/src/query.ts#L69` — job classifier
- `packages/ccb/src/query/stopHooks.ts#L45,L109` — job classifier 模块

---

### 2.9 REACTIVE_COMPACT — 响应式压缩

**状态**: 纯 Stub  
**引用数**: 4  
**类别**: 压缩

**源码位置**: `packages/ccb/src/services/compact/reactiveCompact.ts` (22 行)

```typescript
// Auto-generated stub — replace with real implementation
export {};

import type { Message } from 'src/types/message';
import type { CompactionResult } from './compact.js';

export const isReactiveOnlyMode: () => boolean = () => false;
export const reactiveCompactOnPromptTooLong: (...) => Promise<...> = async () => ({ ok: false });
export const isReactiveCompactEnabled: () => boolean = () => false;
export const isWithheldPromptTooLong: (message: Message) => boolean = () => false;
export const isWithheldMediaSizeError: (message: Message) => boolean = () => false;
export const tryReactiveCompact: (...) => Promise<CompactionResult | null> = async () => null;
```

**引用点**:
- `packages/ccb/src/commands/compact/compact.ts#L35-L36,L87,L92,L143,L175` — compact 命令集成
- `packages/ccb/src/query.ts#L15-L16,L173,L209,L275,L316` — query 流程集成
- `packages/ccb/src/utils/analyzeContext.ts#L1117` — 上下文分析
- `packages/ccb/src/components/TokenWarning.tsx#L98` — Token 警告 UI

---

### 2.10 CCR 系列 — 远程控制

| Feature | 引用 | 状态 | 说明 |
|---------|------|------|------|
| CCR_AUTO_CONNECT | 3 | 部分实现 | 自动建立远程控制会话 |
| CCR_MIRROR | 4 | 部分实现 | 会话状态同步 |
| CCR_REMOTE_SETUP | 1 | Stub | 初始化远程控制配置 |

**核心引用**:
- `packages/ccb/src/bridge/remoteBridgeCore.ts#L732,L748` — CCR_MIRROR  outbound-only 模式
- `packages/ccb/src/bridge/bridgeEnabled.ts#L186,L198` — bridge 启用检查
- `packages/ccb/src/utils/config.ts#L39,L1097` — 配置读取
- `packages/ccb/src/commands.ts#L91` — web 命令
- `packages/ccb/src/main.tsx#L4272` — CLI 处理

---

### 2.11 MEMORY_SHAPE_TELEMETRY — 记忆形态遥测

**状态**: 部分实现  
**引用数**: 3  
**类别**: 遥测

**引用点**:
- `packages/ccb/src/memdir/findRelevantMemories.ts#L66,L69` — memoryShapeTelemetry 动态导入
- `packages/ccb/src/utils/sessionFileAccessHooks.ts#L38-L39,L210,L217` — 文件访问 hook 记录

---

## 三、单引用 Feature（40+ 个）

以下 feature 各只有 1 处引用，多为内部标记或实验性功能：

```
UNATTENDED_RETRY, ULTRATHINK, TORCH, SLOW_OPERATION_LOGGING, SKILL_IMPROVEMENT,
SELF_HOSTED_RUNNER, RUN_SKILL_GENERATOR, PERFETTO_TRACING, NATIVE_CLIENT_ATTESTATION,
KAIROS_DREAM, IS_LIBC_MUSL, IS_LIBC_GLIBC, DUMP_SYSTEM_PROMPT,
COMPACTION_REMINDERS, BYOC_ENVIRONMENT_RUNNER, BUILTIN_EXPLORE_PLAN_AGENTS,
BUILDING_CLAUDE_APPS, ANTI_DISTILLATION_CC, AGENT_TRIGGERS, ABLATION_BASELINE
```

**其他单引用 features** (来自原文档):
- STREAMLINED_OUTPUT — 精简输出模式
- HOOK_PROMPTS — Hook 提示词注入
- NATIVE_CLIPBOARD_IMAGE — 原生剪贴板图片
- CONNECTOR_TEXT — 连接器文本
- COMMIT_ATTRIBUTION — Commit 归因
- CACHED_MICROCOMPACT — 缓存微压缩
- PROMPT_CACHE_BREAK_DETECTION — Prompt cache 中断检测
- QUICK_SEARCH — 快速搜索
- MESSAGE_ACTIONS — 消息操作
- DOWNLOAD_USER_SETTINGS — 下载用户设置
- DIRECT_CONNECT — 直连模式
- VERIFICATION_AGENT — 验证 Agent
- TERMINAL_PANEL — 终端面板
- SSH_REMOTE — SSH 远程
- REVIEW_ARTIFACT — 审查产出物
- FILE_PERSISTENCE — 文件持久化
- TREE_SITTER_BASH_SHADOW — Bash AST Shadow 模式
- UPLOAD_USER_SETTINGS — 上传用户设置
- POWERSHELL_AUTO_MODE — PowerShell 自动模式
- OVERFLOW_TEST_TOOL — 溢出测试工具
- NEW_INIT — 新版初始化流程
- HARD_FAIL — 硬失败模式
- ENHANCED_TELEMETRY_BETA — 增强遥测 Beta
- COWORKER_TYPE_TELEMETRY — 协作者类型遥测
- BREAK_CACHE_COMMAND — 中断缓存命令
- AWAY_SUMMARY — 离开摘要
- AUTO_THEME — 自动主题
- ALLOW_TEST_VERSIONS — 允许测试版本
- AGENT_TRIGGERS_REMOTE — Agent 远程触发
- AGENT_MEMORY_SNAPSHOT — Agent 记忆快照

---

## 四、分类说明

这些 feature 被列为 Tier 3 的原因：

| 原因 | Feature 示例 |
|------|-------------|
| 内部基础设施 | CHICAGO_MCP, LODESTONE — 外部无法直接运行 |
| 纯 Stub 且引用低 | UDS_INBOX, MONITOR_TOOL, REACTIVE_COMPACT — 需要完整后端实现 |
| 实验性功能 | SHOT_STATS, EXTRACT_MEMORIES — Gate 控制严格，引用范围有限 |
| 辅助功能 | STREAMLINED_OUTPUT, HOOK_PROMPTS — 影响范围小 |
| 依赖远程控制 | CCR 系列 — 需要 BRIDGE_MODE 完善 |

---

## 五、源码文件索引

| 文件 | 行数 | 状态 |
|------|------|------|
| `src/tools/MonitorTool/MonitorTool.ts` | 3 | 纯 Stub |
| `src/utils/udsClient.ts` | 3 | 纯 Stub |
| `src/services/compact/reactiveCompact.ts` | 22 | 纯 Stub |
| `src/utils/concurrentSessions.ts` | 204 | 部分实现 |
| `src/services/extractMemories/extractMemories.ts` | 615 | 完整实现 |
| `src/utils/stats.ts` | ~1061 | 完整实现 |
| `src/utils/backgroundHousekeeping.ts` | 94 | 完整集成 |

---

## 六、结论

Tier 3 features 包含大量引用量低或处于实验/内部状态的开关。其中：

- 部分为 **Anthropic 内部基础设施**（`CHICAGO_MCP`、`LODESTONE` 等），外部仓库无法直接运行；
- 部分为 **纯 Stub 或骨架布线**（`MONITOR_TOOL`、`UDS_INBOX`、`REACTIVE_COMPACT`、Context Collapse 核心模块等），需要完整后端实现才能投入使用；
- 另有少量功能本身已实现（如 `EXTRACT_MEMORIES`、`SHOT_STATS`），但 Gate 控制严格、引用范围有限，目前仍处于实验或小流量阶段。

如需深入了解某个 Tier 3 feature，可在 `packages/ccb/src/` 中搜索 `feature('FEATURE_NAME')` 查看具体使用场景。
