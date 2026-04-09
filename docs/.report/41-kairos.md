# Unit 41 — KAIROS 常驻助手模式

> 原始页面：https://ccb.agent-aura.top/docs/features/kairos  
> 生成日期：2026-04-09

---

## 1. 功能概述

KAIROS 将 Claude Code CLI 从"问答工具"转变为"常驻助手"（persistent assistant）。开启后，CLI 持续运行在后台，支持跨终端复用 session、后台自主工作、移动端推送、记忆蒸馏与外部频道消息接入。KAIROS 是 codebase 中引用量最大的 feature flag（约 154 处）。

子 Feature 的层级结构如下：

```
KAIROS (主开关)
├── KAIROS_BRIEF         — BriefTool 结构化输出
├── KAIROS_CHANNELS      — 外部频道消息（Slack/Discord/Telegram）
├── KAIROS_PUSH_NOTIFICATION — 移动端推送
├── KAIROS_GITHUB_WEBHOOKS   — GitHub PR webhook
└── KAIROS_DREAM         — 记忆蒸馏（auto-dream）
```

**关键设计点**：所有 `PROACTIVE` 检查都采用 `feature('PROACTIVE') || feature('KAIROS')` 的 OR 语义，因此开启 KAIROS 时自动获得 proactive 能力。

---

## 2. Feature Flag 与门控

### 2.1 Build-time 检查

KAIROS 的核心入口在 `main.tsx` 中通过 `feature('KAIROS')` 分支控制：

- `main.tsx:142-145` — 动态 import assistant 模块（stub）与 kairos gate（stub）
- `main.tsx:1004-1019` — `claude assistant [sessionId]` 子命令的 argv 前处理
- `main.tsx:1793-1839` — `--assistant` 标志处理与 `initializeAssistantTeam()` 调用
- `main.tsx:2550-2660` — `--channels` / `--dangerously-load-development-channels` 解析
- `main.tsx:2663-2675` — `--tools` 中包含 `SendUserMessage` 时的 opt-in 处理
- `main.tsx:3307-3317` — `defaultView === 'chat'` 时的 Brief opt-in
- `main.tsx:3324-3336` — `--proactive` 时追加 proactive prompt
- `main.tsx:3345-3350` — `getAssistantSystemPromptAddendum()` 注入
- `main.tsx:3742-3744` — `assistantActivationPath` 传给 `createQueryEngine()`

### 2.2 GrowthBook 运行时门控

- `src/assistant/gate.ts#L3` — `isKairosEnabled` 为 stub，恒返回 `false`
- `src/tools/BriefTool/BriefTool.ts:88-99` — `isBriefEntitled()`：在 build flag 开启后，进一步检查 `tengu_kairos_brief` GrowthBook 标志或 `CLAUDE_CODE_BRIEF` 环境变量
- `src/tools/BriefTool/BriefTool.ts:126-134` — `isBriefEnabled()`：需要用户显式 opt-in（`--brief`、`defaultView: 'chat'`、`/brief` 命令、`--tools` 包含 `SendUserMessage`）且具备 entitlement
- `src/services/mcp/channelNotification.ts:191-315` — `gateChannelServer()`： channels 的 7 层门控（capability → runtime gate `tengu_harbor` → OAuth auth → org policy → session `--channels` → marketplace verification → allowlist）

### 2.3 状态存储

- `src/bootstrap/state.ts:1085-1111` — `getKairosActive()` / `setKairosActive()` / `getUserMsgOptIn()` / `setUserMsgOptIn()`，用于会话级运行时状态

---

## 3. 系统提示注入

系统提示的 KAIROS/Proactive 段落由 `src/constants/prompts.ts` 组装。

### 3.1 Brief 段落 (`getBriefSection`)

- 位置：`src/constants/prompts.ts#L844-L858`
- 注入条件：`feature('KAIROS') || feature('KAIROS_BRIEF')`，且 `isBriefEnabled()` 为 true，且 proactive 未激活（避免与 `getProactiveSection` 内联的 Brief 内容重复）
- 内容来源：`src/tools/BriefTool/prompt.ts#L12-L22` 中的 `BRIEF_PROACTIVE_SECTION`

### 3.2 Proactive/Autonomous Work 段落 (`getProactiveSection`)

- 位置：`src/constants/prompts.ts#L861-L914`
- 注入条件：`feature('PROACTIVE') || feature('KAIROS')`，且 `isProactiveActive()` 为 true
- 核心指令：
  - Tick 驱动：通过 `<tick>` prompt 保持存活，每个 tick 包含用户当前本地时间
  - 节奏控制：使用 `Sleep` 工具控制等待间隔，prompt cache 5 分钟后过期
  - 空操作时必须 Sleep，禁止输出 "still waiting"
  - 偏向行动：读文件、搜索、修改、commit 都不需询问
  - Terminal Focus 感知：根据 `terminalFocus` 调节自主程度（Unfocused → 高度自主；Focused → 更协作）
  - 若 Brief 可用，在段落末尾追加 `BRIEF_PROACTIVE_SECTION`

### 3.3 主装配点

- `src/constants/prompts.ts#L552-L554` — dynamic sections 列表中注册 `brief` section
- `src/constants/prompts.ts#L488` — `getProactiveSection()` 在 `getSystemPrompt` 内部被调用
- `src/main.tsx#L3324-3336` — `--proactive` flag 也会追加一个简化的 proactive prompt（与系统提示中的完整版并存或互补）

---

## 4. SleepTool — 节奏控制核心

- `src/tools/SleepTool/prompt.ts#L3` — `SLEEP_TOOL_NAME = 'Sleep'`
- `src/tools/SleepTool/prompt.ts#L7-L17` — 工具描述：告诉模型在等待、无事可做或收到 `<tick>` 时使用 Sleep；明确说明醒来消耗 API call，但 5 分钟 inactive 后 prompt cache 会过期
- `src/constants/xml.ts#L25` — `TICK_TAG = 'tick'`
- `src/cli/print.ts#L1842` — proactive tick 的实际内容：`` `<tick>${new Date().toLocaleTimeString()}</tick>` ``

在 REPL 中，`scheduleProactiveTick`（`src/cli/print.ts`）通过 `enqueue()` 将 tick 作为 meta-prompt 推入队列，模型收到后可以选择继续工作，或调用 Sleep 等待下一次 tick。

---

## 5. Bridge 集成与数据流

KAIROS 通过 Bridge Mode 连接到 claude.ai 服务器。Bridge 模块位于 `src/bridge/`（约 35 个文件）。

### 5.1 Bridge API Client

- `src/bridge/bridgeApi.ts#L199-L247` — `pollForWork()`：长轮询 `/v1/environments/{environment_id}/work/poll`，收到 `WorkResponse` 后返回；空响应时累计 `consecutiveEmptyPolls`
- `src/bridge/bridgeApi.ts#L249-L264` — `acknowledgeWork()`：ACK 已接收的 work item，调用端点在 `/v1/environments/{environment_id}/work/{work_id}/ack`

### 5.2 Bridge 主轮询循环

- `src/bridge/bridgeMain.ts#L607` — `api.pollForWork(...)` 位于外层 `while (!loopSignal.aborted)` 循环中
- `src/bridge/bridgeMain.ts#L840` — 收到 session work 后调用 `api.acknowledgeWork(...)`
- `src/bridge/bridgeMain.ts#L711-L722` — at-capacity heartbeat 模式：当活动 session 数达到上限时，进入 heartbeat 子循环，按配置间隔维持 liveness，auth 失败或 poll deadline 到达时跳出

### 5.3 Session Runner

- `src/bridge/sessionRunner.ts#L248-L547` — `createSessionSpawner()` 生成 `SessionSpawner`
- `src/bridge/sessionRunner.ts#L335-L340` — `spawn()` 内部使用 `child_process.spawn` 启动子 CLI 进程，环境变量：`CLAUDE_CODE_ENVIRONMENT_KIND=bridge`、`CLAUDE_CODE_SESSION_ACCESS_TOKEN=...`
- `src/bridge/sessionRunner.ts#L368-L445` — stdout NDJSON 解析：提取 tool_use、text、result、control_request 与 replayed user message，维护 `activities` ring buffer

### 5.4 REPL Bridge（v1/v2 传输层）

- `src/bridge/replBridge.ts#L1945` — REPL bridge 侧的 `pollForWork()`
- `src/bridge/replBridge.ts#L2157` — `acknowledgeWork()`
- `src/bridge/replBridge.ts#L1380-1420` — v1 `HybridTransport`（WebSocket 读 + POST 写）与 v2 `CCRClient`（SSETransport）的建立逻辑

### 5.5 端到端数据流

```
用户从 claude.ai 发送消息
         │
         ▼
Bridge pollForWork() 收到 WorkResponse
         │
         ▼
acknowledgeWork() 确认接收
         │
         ▼
sessionRunner 创建/恢复 REPL session（子进程）
         │
         ▼
用户消息注入 REPL 对话队列
         │
         ▼
模型处理 → 工具调用（可能包含 Sleep / SendUserMessage）
         │
         ▼
结果通过 Bridge API / WebSocket 回传到 claude.ai
```

---

## 6. Assistant 模块（Stub 架构）

`src/assistant/` 目录目前全部是 stub，定义了 KAIROS 的扩展接口但尚未实现内部逻辑：

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/assistant/index.ts` | 9 | `isAssistantMode()`、`initializeAssistantTeam()`、`markAssistantForced()`、`getAssistantSystemPromptAddendum()`、`getAssistantActivationPath()` 等，均为 no-op |
| `src/assistant/gate.ts` | 3 | `isKairosEnabled()` 恒返回 `false` |
| `src/assistant/sessionDiscovery.ts` | — | stub |
| `src/assistant/sessionHistory.ts` | — | stub |
| `src/assistant/AssistantSessionChooser.ts` | — | stub |

这意味着当前代码库中 KAIROS 的核心 assistant 启动路径被占位，实际运行时需要替换为真实实现。

---

## 7. BriefTool 实现

BriefTool（模型可见名为 `SendUserMessage`，别名为 `Brief`）是 KAIROS 的结构化输出通道。

### 7.1 定义与模式

- `src/tools/BriefTool/prompt.ts#L1-L22` — 工具名、描述、prompt、`BRIEF_PROACTIVE_SECTION`
- `src/tools/BriefTool/BriefTool.ts#L20-L63` — Zod input/output schema：
  - `message`: markdown 文本
  - `attachments`: 可选文件路径数组（支持图片、diff、log）
  - `status`: `'normal' | 'proactive'`，用于区分主动触发的消息与被动回复
- `src/tools/BriefTool/BriefTool.ts#L67` — `KAIROS_BRIEF_REFRESH_MS = 5 * 60 * 1000`（GrowthBook 缓存刷新周期）

### 7.2 Entitlement 与 Activation

- `src/tools/BriefTool/BriefTool.ts#L88-L99` — `isBriefEntitled()`: build flag → `getKairosActive()` / `CLAUDE_CODE_BRIEF` env / `tengu_kairos_brief` GB flag
- `src/tools/BriefTool/BriefTool.ts#L126-L134` — `isBriefEnabled()`: 需要 `getKairosActive() || getUserMsgOptIn()` 且具备 entitlement；被 `Tool.isEnabled()` 懒加载调用

### 7.3 调用行为

- `src/tools/BriefTool/BriefTool.ts#L186-L203` — `call()` 方法：记录 `tengu_brief_send` telemetry，解析附件，返回带 `sentAt` ISO 时间戳的结果
- `src/tools/BriefTool/BriefTool.ts#L175-L182` — `mapToolResultToToolResultBlockParam()` 返回 `Message delivered to user.` 作为工具结果给模型

### 7.4 UI 渲染

- `src/tools/BriefTool/UI.tsx` — `renderToolUseMessage` 与 `renderToolResultMessage`（未展开，但源码存在）

---

## 8. Channel Notification（外部频道消息）

- `src/services/mcp/channelNotification.ts` — 完整的 notification handler 与 gate 逻辑（117 行，非 stub）
- `src/services/mcp/channelAllowlist.ts` — allowlist 数据源（ledger / org policy）
- `src/services/mcp/useManageMCPConnections.ts` — MCP 连接管理（channel gate 在连接建立时调用）

关键协议：
- MCP server 声明 `experimental['claude/channel']` capability
- 发送 `notifications/claude/channel` 通知 inbound 消息
- 发送 `notifications/claude/channel/permission` 处理用户通过手机回复的权限审批

消息被包装为：
```xml
<channel source="..." metaKey="...">
内容
</channel>
```
由 `wrapChannelMessage()`（`channelNotification.ts#L106-L116`）生成，随后进入 REPL 消息队列。

---
## 9. 记忆与 Dream（Stub）

- `src/memdir/memdir.ts` — Memory Directory 管理（stub）
- `src/components/tasks/src/tasks/DreamTask/` — Dream 任务（stub）
- 相关文档：`docs/features/auto-dream.md`

---

## 10. 关键源码锚点索引

| 概念 | 文件 | 行号范围 |
|------|------|---------|
| `feature('KAIROS')` 入口分支 | `src/main.tsx` | `142-145`, `1004-1019`, `1793-1839`, `2550-2660`, `3307-3350`, `3742-3744` |
| Kairos / Brief 状态存取 | `src/bootstrap/state.ts` | `1085-1111` |
| `isKairosEnabled` GB gate | `src/assistant/gate.ts` | `3` |
| `isAssistantMode` stub | `src/assistant/index.ts` | `3` |
| `isProactiveActive` stub | `src/proactive/index.ts` | `3` |
| Proactive section 注入 | `src/constants/prompts.ts` | `861-914` |
| Brief section 注入 | `src/constants/prompts.ts` | `844-858` |
| 系统提示装配点 | `src/constants/prompts.ts` | `488`, `552-554` |
| SleepTool prompt | `src/tools/SleepTool/prompt.ts` | `3`, `7-17` |
| Tick 标签定义 | `src/constants/xml.ts` | `25` |
| Tick 生成与入队 | `src/cli/print.ts` | `1836-1849` |
| `pollForWork` API | `src/bridge/bridgeApi.ts` | `199-247` |
| `acknowledgeWork` API | `src/bridge/bridgeApi.ts` | `249-264` |
| Bridge 轮询循环 | `src/bridge/bridgeMain.ts` | `590-880` |
| Session spawner | `src/bridge/sessionRunner.ts` | `248-547` |
| REPL Bridge poll | `src/bridge/replBridge.ts` | `1945`, `2157` |
| Transport 建立 | `src/bridge/replBridge.ts` | `1380-1420` |
| BriefTool entitlement | `src/tools/BriefTool/BriefTool.ts` | `88-99`, `126-134` |
| BriefTool schema / call | `src/tools/BriefTool/BriefTool.ts` | `20-63`, `175-203` |
| Brief prompt & section | `src/tools/BriefTool/prompt.ts` | `1-22` |
| Channel notification gate | `src/services/mcp/channelNotification.ts` | `191-315` |
| Channel message wrap | `src/services/mcp/channelNotification.ts` | `106-116` |

---

## 11. 审校备注

1. **文档与源码一致性**：原始页面提到 `src/assistant/sessionDiscovery.ts`、`sessionHistory.ts` 等为 stub，经源码确认属实；`src/services/mcp/channelNotification.ts` 并非 5 行 stub，而是具有完整 gate 逻辑的 317 行实现。
2. **精确行号更新**：由于源码演进，`main.tsx` 中的 KAIROS 相关行号与原始文档（ Mintlify 页面基于较旧快照）相比存在偏移，本报告已按当前 `research` 分支实际行号重新标注。
3. **Proactive 文档关系**：PROACTIVE 与 KAIROS 在代码层面是 OR 关系，但 `src/proactive/index.ts` 本身也为 stub；proactive 的实际 tick 调度和 Sleep 处理逻辑集中在 `src/cli/print.ts` 和 `src/constants/prompts.ts`。
4. **Brief 显示/行为分离确认**：`/brief` toggle 与 `--brief` flag 仅影响 `isBriefEnabled()` 中的 `getUserMsgOptIn()`，而非是否向模型暴露 `SendUserMessage` 工具（工具可用性由 `isBriefEnabled()` 决定，但 UI 过滤由 `isBriefOnly` 状态控制）。
