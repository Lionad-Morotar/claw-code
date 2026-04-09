# Ant 特权世界 — Anthropic 员工专属功能

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/internals/ant-only-world) 的原始内容，映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码现状。由于 claw-code 是对上游的**干净重实现**，大量 ant-only 门控逻辑在上游 TypeScript 中密集存在，但在 Rust 代码库中尚未建立等价体系。
> 文末附 [源码索引](#源码索引) 与 [映射差距说明](#映射差距说明)。

---

## 一句话定义

`USER_TYPE === 'ant'` 是上游 TypeScript 构建时的身份门控常量。当该条件成立时，Claude Code 会解锁一整套**仅供 Anthropic 内部员工使用的工具、命令、API Headers 和调试开关**。外部公开发布版本中，这些代码路径被 Bun 打包器的 Dead Code Elimination（DCE）直接裁切。

---

## 上游架构：什么是 Ant

在上游代码库中，`USER_TYPE` 是一个通过 Bun `--define` 注入的构建时常量：

- 内部构建：`process.env.USER_TYPE === 'ant'` 为 `true`
- 外部构建：相同比较被常量折叠为 `false`，后续代码被 DCE 移除

据文档统计，`USER_TYPE === 'ant'` 在上游出现 **291 次**，`(process.env.USER_TYPE) === 'ant'` 出现 **86 次**，`
!== 'ant'` 出现 **53 次**，总引用量约 **465 处**，控制着工具注册、斜杠命令、Beta Header、UI 渲染、功能开关等方方面面。

### claw-code 中的现状

**claw-code 的 Rust 实现目前不存在 `USER_TYPE` 或 `IS_DEMO` 的等价构建时门控体系**。全局搜索以下关键字在 `rust/crates/` 中均无结果：

- `USER_TYPE`
- `IS_DEMO`
- `ant_only` / `ant-only`
- `GrowthBook`
- `undercover`

这说明 Rust 重写版当前是一个**单一构建产物**，不区分内部/外部版本。所有已实现的功能对所有用户一视同仁，尚未引入员工身份分层。

---

## Ant-Only 工具（上游）

上游仅在 `USER_TYPE === 'ant'` 时加载到工具注册表的工具包括：

| 工具 | 上游代码位置 | 用途 |
|------|-------------|------|
| `REPLTool` | `src/tools/REPLTool/` | 高级 REPL 模式，在 VM 中包装 Bash/Read/Edit/Glob/Grep/Agent 等 |
| `SuggestBackgroundPRTool` | `src/tools/SuggestBackgroundPRTool/` | 建议在后台创建 PR |
| `ConfigTool` | `src/tools/ConfigTool/` | 交互式配置编辑器，含 Gates 标签页用于覆盖 GrowthBook flags |
| `TungstenTool` | `src/tools/TungstenTool/` | 基于 tmux 的终端面板工具 |

### claw-code 映射现状

claw-code 的 `tools` crate 已经实现了**表面覆盖**（spec parity）的 `REPL` 和 `Config` 等工具，参见 [`PARITY.md`](/rust/PARITY.md) 的 Tool Surface 章节：

- `REPL` — `moderate parity`，子进程代码执行已 wired，但无 VM 包装层，也无 ant-only 门控。
- `Config` — `moderate parity`，配置读取已实现，但缺少交互式编辑器与 Gates 标签页。
- `SuggestBackgroundPRTool` / `TungstenTool` — **尚未实现**。

#### 工具注册入口

当前 Rust 代码中的工具注册在运行时统一完成，没有条件分支。相关源码：

- [`commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) — 斜杠命令与工具分发逻辑
- [`plugins/src/lib.rs`](/rust/crates/plugins/src/lib.rs) — 插件工具发现与挂载

---

## Ant-Only 命令（上游）

上游在 `src/commands.ts` 注册了 28 个仅在内部构建可用的斜杠命令（`INTERNAL_ONLY_COMMANDS`），并在 `USER_TYPE === 'ant' && !IS_DEMO` 时加载。

### 内部命令分类

**调试类**
- `breakCache`、`ctx_viz`、`debugToolCall`、`env`、`mockLimits`、`resetLimits`、`resetLimitsNonInteractive`

**实验类**
- `bughunter`、`goodClaude`、`antTrace`、`perfIssue`

**工作流类**
- `commit`、`commitPushPr`、`issue`、`autofixPr`、`share`、`summary`、`subscribePr`、`forceSnip`、`ultraplan`

**基础设施类**
- `backfillSessions`、`bridgeKick`、`oauthRefresh`、`teleport`、`onboarding`、`agentsPlatform`、`version`、`initVerifiers`

### claw-code 映射现状

claw-code 当前已实现 **67 / 141** 个上游斜杠命令条目（参见 [`PARITY.md#L97-L101`](/rust/PARITY.md#L97-L101)）：

- 27 个原始 spec 有**真实 handler**
- 40 个新 spec 为**解析 + stub handler**（"not yet implemented"）
- 剩余约 74 个上游条目为内部模块/对话框/步骤，并非用户可直接调用的 `/commands`

**claw-code 目前没有 `INTERNAL_ONLY_COMMANDS` 的区分概念**。所有命令要么已实现，要么为 stub，但没有针对 "内部员工" 的条件加载逻辑。命令解析与分发入口参见：

- [`rusty-claude-cli/src/input.rs`](/rust/crates/rusty-claude-cli/src/input.rs) — 输入捕获与斜杠命令识别
- [`rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) — CLI 入口与 `CliAction` 分发
- [`commands/src/lib.rs#L4500-L4600`](/rust/crates/commands/src/lib.rs#L4500-L4600) 附近 — 消息组装与工具调用分发

---

## Beta API Headers 体系

上游将 Beta Header 分为四类：公开、模型能力相关、Ant-Only、Feature Flag Gated。

### claw-code 的 Beta Header 实现

Rust 实现中，Beta Header 的管理集中在 `telemetry` crate 的 `AnthropicRequestProfile` 结构体中：

```rust
pub struct AnthropicRequestProfile {
    pub anthropic_version: String,
    pub client_identity: ClientIdentity,
    pub betas: Vec<String>,
    pub extra_body: Map<String, Value>,
}
```

源码位置：[`rust/crates/telemetry/src/lib.rs#L52-L58`](/rust/crates/telemetry/src/lib.rs#L52-L58)

默认构造时注入了两个 public beta：

```rust
betas: vec![
    DEFAULT_AGENTIC_BETA.to_string(),          // "claude-code-20250219"
    DEFAULT_PROMPT_CACHING_SCOPE_BETA.to_string(), // "prompt-caching-scope-2026-01-05"
],
```

源码位置：[`telemetry/src/lib.rs#L69-L72`](/rust/crates/telemetry/src/lib.rs#L69-L72)

`AnthropicClient` 提供 `with_beta(...)` 链式 API 以追加额外 beta：

```rust
pub fn with_beta(mut self, beta: impl Into<String>) -> Self {
    self.request_profile = self.request_profile.with_beta(beta);
    self
}
```

源码位置：[`api/src/providers/anthropic.rs#L227-L230`](/rust/crates/api/src/providers/anthropic.rs#L227-L230)

发送请求时，header 通过 `request_profile.header_pairs()` 注入：

```rust
for (header_name, header_value) in self.request_profile.header_pairs() {
    request_builder = request_builder.header(header_name, header_value);
}
```

源码位置：[`api/src/providers/anthropic.rs#L483-L485`](/rust/crates/api/src/providers/anthropic.rs#L483-L485)

### Beta Body 字段剥离

标准 `/v1/messages` 和 `/v1/messages/count_tokens` 端点**拒绝在 JSON body 中出现 `betas` 字段**。AnthropicClient 在发送前会调用 `strip_unsupported_beta_body_fields`：

```rust
fn strip_unsupported_beta_body_fields(body: &mut Value) {
    if let Some(object) = body.as_object_mut() {
        object.remove("betas");
        object.remove("frequency_penalty");
        object.remove("presence_penalty");
        if let Some(stop_val) = object.remove("stop") {
            if stop_val.as_array().map_or(false, |a| !a.is_empty()) {
                object.insert("stop_sequences".to_string(), stop_val);
            }
        }
    }
}
```

源码位置：[`api/src/providers/anthropic.rs#L1011-L1024`](/rust/crates/api/src/providers/anthropic.rs#L1011-L1024)

该函数同时清理 OpenAI 兼容字段（`frequency_penalty`、`presence_penalty`、`stop`）并做键名转换，保证 Anthropic 原生端点的兼容性。

### 测试中的 Beta Header 验证

集成测试验证了 beta 必须通过 header 而非 body 发送：

```rust
body.get("betas").is_none(),
"betas must travel via the anthropic-beta header, not the request body"
```

源码位置：[`api/tests/client_integration.rs#L101-L102`](/rust/crates/api/tests/client_integration.rs#L101-L102)

---

## 内部代号体系（上游）

上游有浓厚的动物命名文化：

| 代号 | 身份 | 出处 |
|------|------|------|
| **Tengu** | Claude Code 项目代号 | GrowthBook flags 的 `tengu_` 前缀 |
| **Capybara** | 模型代号 | `src/constants/prompts.ts`，被 Undercover Mode 屏蔽 |
| **Fennec** | 已退役模型别名 | `src/migrations/migrateFennecToOpus.ts` |

### claw-code 映射现状

claw-code 中**未出现**任何 `tengu_`、 `capybara`、 `fennec` 或 `undercover` 相关字符串。代号体系的过滤属于上游的 commit/发布流程工程，Rust 重写版尚未建立等价的文化/过滤机制。

---

## 环境变量开关（上游）

上游有一系列精细环境变量控制功能启用/禁用：

**功能禁用**
- `CLAUDE_CODE_SIMPLE` / `CLAUDE_CODE_DISABLE_THINKING` / `DISABLE_INTERLEAVED_THINKING`
- `DISABLE_COMPACT` / `DISABLE_AUTO_COMPACT`
- `CLAUDE_CODE_DISABLE_AUTO_MEMORY`
- `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS`
- `CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS`

**功能启用**
- `CLAUDE_CODE_VERIFY_PLAN` / `ENABLE_LSP_TOOL`
- `CLAUDE_CODE_UNDERCOVER`
- `CLAUDE_CODE_TERMINAL_RECORDING`
- `CLAUDE_CODE_ABLATION_BASELINE`

**环境配置**
- `CLAUDE_CODE_REMOTE` / `CLAUDE_CODE_COORDINATOR_MODE`
- `CLAUDE_INTERNAL_FC_OVERRIDES`（ant-only）
- `IS_DEMO`

### claw-code 中的环境变量使用

Rust 代码中对环境变量的读取更加收敛。`AnthropicClient` 通过 `read_env_non_empty` 读取认证相关的标准变量：

- `ANTHROPIC_API_KEY`
- `ANTHROPIC_AUTH_TOKEN`

源码位置：[`api/src/providers/anthropic.rs#L646-L651`](/rust/crates/api/src/providers/anthropic.rs#L646-L651)

此外，`http_client.rs` 支持部分通过环境变量覆盖请求行为：

源码位置：[`api/src/http_client.rs#L78`](/rust/crates/api/src/http_client.rs#L78)

**但 claw-code 当前尚未实现上述任何功能开关环境变量**。例如：

- 上下文 compaction 行为在 [`runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) 中是硬编码逻辑，没有 `DISABLE_COMPACT` 的旁路开关。
- thinking 模式、background tasks、auto-memory 等功能在 Rust 代码库中**尚未提及**，更无对应环境变量。

---

## 映射差距说明

| 上游 ant-only 概念 | claw-code 现状 | 差距评估 |
|-------------------|----------------|---------|
| `USER_TYPE === 'ant'` 门控 | **不存在** | 大 — 未建立构建时身份分层 |
| `IS_DEMO` 演示模式 | **不存在** | 中 — 无演示环境隔离 |
| Ant-Only 工具（REPLTool/ConfigTool/Gates） | 部分有 spec parity，但**无 ant 门控** | 中 — 功能可用性差异 |
| 28 个内部斜杠命令 | 部分为 stub，部分未实现，**无内部/外部分类** | 中 |
| `cli-internal-2026-02-09` / `token-efficient-tools` Beta Header | **未发送** | 小 — 当前默认 betas 已满足公开功能 |
| GrowthBook feature flags | **不存在** | 大 — 无动态灰度体系 |
| Undercover Mode / 动物代号过滤 | **不存在** | 小 — 属于发布流程而非运行时 |
| 大量 `CLAUDE_CODE_*` 环境变量开关 | **极少** | 中 — 调试/实验灵活性不足 |

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`rust/PARITY.md`](/rust/PARITY.md) | Tool/Command 覆盖度与行为差距清单 |
| [`rust/crates/telemetry/src/lib.rs`](/rust/crates/telemetry/src/lib.rs) | `AnthropicRequestProfile`、默认 beta headers、header pairs |
| [`rust/crates/api/src/providers/anthropic.rs`](/rust/crates/api/src/providers/anthropic.rs) | `with_beta`、请求构建、`strip_unsupported_beta_body_fields` |
| [`rust/crates/api/tests/client_integration.rs`](/rust/crates/api/tests/client_integration.rs) | Beta header 集成测试与 wire format 校验 |
| [`rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、`CliAction`、无身份门控的分发 |
| [`rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | 斜杠命令与工具分发的核心逻辑 |

---

## REVIEW.md

**审查日期**: 2026-04-09

**准确性核对**
- [x] 源码锚点已逐一手动核对，行号与当前 `research` 分支一致
- [x] `telemetry/src/lib.rs#L52-L58`、`L69-L72` 行号确认无误
- [x] `api/src/providers/anthropic.rs#L227-L230`、`L483-L485`、`L1011-L1024` 行号确认无误
- [x] `api/tests/client_integration.rs#L101-L102` 行号确认无误
- [x] PARITY.md 引用内容与仓库 snapshot 一致

**缺失/待补说明**
- 上游的 `ant-only` 功能在 Rust 代码库中确实不存在，报告如实记录了此差距
- 由于 `claw-code` 是重实现，报告的重点放在**已有等价机制**（如 beta header 管理体系）的源码级解剖上
- 若未来引入 `USER_TYPE` 门控，应优先在 `rusty-claude-cli/src/main.rs` 的构建时宏/条件编译层面展开

**质量结论**: 可发布
