# 为什么写这份白皮书 - Claude Code 逆向工程分析

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/introduction/why-this-whitepaper) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 这份白皮书是什么

这是对 Anthropic 官方发布的 Claude Code CLI 的**逆向工程分析**。原始 TypeScript 单文件 bundle 经过反编译处理，保留了核心功能模块，但包含大量类型残损。`claw-code` 仓库在此基础上进行了**公开的 Rust 重写**：它不是官方分叉，而是一个基于运行时行为与残留源码结构重建的、可独立编译运行的生产级 agentic system。

在 `claw-code` 仓库中，这种"逆向工程 + 重写"的哲学被明确记录。

### 逆向工程的边界声明

需要明确的是，本白皮书的分析基于**公开可观察的运行时行为**和经反编译处理后的 TypeScript bundle 结构，不涉及对 Anthropic 服务端、专有模型权重或未公开发布的内部文档的访问。`claw-code` 是一个独立的、开源的重新实现，其代码结构、类型设计和错误处理策略均经过重构以适配 Rust 生态。这意味着：

1. **这不是官方 fork**：代码库没有使用任何受版权保护的原始源代码片段作为实现基础。
2. **行为等价 ≠ 代码等价**：Rust 实现通过测试 harness 验证了与上游的外部行为一致性，但内部数据流和模块边界可能与原始 TypeScript 有显著差异。
3. **parity 是目标而非现实**：`PARITY.md` 明确标记了大量仍处在 stub 或未实现状态的模块（如 reactive compact、LSP 完整接入、任务持久化等）。

### Rust 重写的工程定位

`claw-code` 将上游的终端原生架构迁移到了 Rust 生态中，核心workspace 位于 [`/rust/`](/rust/)。截至 2026-04-10，主分支已包含 **9 个 crate**、约 **84,000 行 Rust 源码**（非空行）和 **4,200+ 行测试代码**。这不是一个玩具项目，而是一个通过了 mock parity harness 验证的严肃实现——支持流式响应、多工具调用、权限提示、Bash 沙箱、会话压缩等完整链路。

白皮书的价值在于：它不仅解释 Claude Code "做了什么"，更解释 "为什么这样设计"。而 `claw-code` 的 Rust 源码为这些设计决策提供了**可编译、可调试、可修改**的实体证据。

---

## 逆向过程中最精妙的设计决策

### 1. Agentic Loop 的自愈能力

上游 TypeScript 实现中，`src/query.ts` 的核心循环不是一个简单的 "发请求→收响应" 过程，而是一个**自愈的状态机**。Rust 实现中，这个循环被精确地编码在 [`runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) 的 `ConversationRuntime::run_turn` 中。

#### `run_turn` 的结构：一个真实的 `loop`

参见 [`conversation.rs#L296-L490`](/rust/crates/runtime/src/conversation.rs#L296-L490)：

```rust
pub fn run_turn(
    &mut self,
    user_input: impl Into<String>,
    mut prompter: Option<&mut dyn PermissionPrompter>,
) -> Result<TurnSummary, RuntimeError> {
    // ... 初始化
    loop {
        iterations += 1;
        if iterations > self.max_iterations {
            return Err(RuntimeError::new(
                "conversation loop exceeded the maximum number of iterations",
            ));
        }

        let request = ApiRequest {
            system_prompt: self.system_prompt.clone(),
            messages: self.session.messages.clone(),
        };
        let events = self.api_client.stream(request)?;
        // ... 解析 assistant 消息、检测 ToolUse
        // ... 权限检查 → 工具执行 → 结果回写
    }
}
```

这不是比喻——这是一个真实的 `loop { ... }`。Agentic loop 的自愈能力体现在几个层面：

- **API 错误处理**：`api_client.stream(request)` 的 `Err` 直接触发 `record_turn_failed`，但不会崩溃进程；调用方可以决定是否重试或降级模型（通过 `ProviderFallbackConfig`）。
- **工具执行失败**：`tool_executor.execute()` 返回 `Err` 时，系统不会终止循环，而是将错误内容包装为 `ToolResult` 回传给模型，让 AI **自己决定下一步**。
- **用户中断**：虽然没有直接暴露 "UserInterruptionMessage" 的独立类型，但 Session 的 `push_user_text` 支持在循环的任何迭代中注入新的用户输入，迫使模型重新评估上下文。
- **迭代上限保护**：`max_iterations` 默认为 `usize::MAX`，但可以通过 `with_max_iterations` 设置边界，防止无限循环。

Hooks 系统进一步强化了这个自愈模型。在每次工具调用前后，`run_turn` 都会调用 `run_pre_tool_use_hook` 和 `run_post_tool_use_hook`。如果 hook 返回 `Denied`/`Failed`/`Cancelled`，循环会将该结果作为 `is_error = true` 的 `ToolResult` 回传模型，而不是粗暴终止。参见 [`conversation.rs#L348-L490`](/rust/crates/runtime/src/conversation.rs#L348-L490)。

### 2. 上下文工程的分层策略

Claude Code 的 "记忆" 是精心分层的工程幻觉。`claw-code` 的 Rust 实现中，这一分层在 [`runtime/src/prompt.rs`](/rust/crates/runtime/src/prompt.rs) 和 [`runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) 中有精确映射。

#### Prompt 工程中的缓存策略

[`prompt.rs#L203-L224`](/rust/crates/runtime/src/prompt.rs#L203-L224) 的 `discover_instruction_files` 会向上遍历目录树收集 `CLAUDE.md`、`.claw/instructions.md` 等文件。这些文件内容相对"静态"，而当前对话历史相对"动态"。在 Prompt 组装时，静态内容被放置在前面——这直接服务于 Anthropic API 的 **prompt caching** 机制：前缀不变时可以复用缓存 token，显著降低延迟和成本。

#### 会话状态的结构化存储

[`session.rs#L28-L52`](/rust/crates/runtime/src/session.rs#L28-L51) 定义了 `ContentBlock` 枚举：

```rust
pub enum ContentBlock {
    Text { text: String },
    ToolUse { id: String, name: String, input: String },
    ToolResult { tool_use_id: String, tool_name: String, output: String, is_error: bool },
}
```

注意 `ToolResult` 包含 `tool_use_id` 字段，这保证了工具调用与结果形成严格的**请求-响应对**。在多轮对话中，模型可以精确追踪 "我请求了什么 → 系统返回了什么"。

#### 上下文分层的完整矩阵

| 层级 | Rust 实现位置 | 持久性 | 作用 |
|------|---------------|--------|------|
| System Prompt | `prompt.rs#L432-L450` (`load_system_prompt`) | 每轮重建 | 项目结构、git 状态、CLAUDE.md |
| 对话历史 | `session.rs` (`Session.messages`) | 会话内 | 完整的 User/Assistant/Tool 消息 |
| Compaction | `compact.rs` (`compact_session`) | 压缩后替代原始消息 | 控制 token 预算 |
| Session 持久化 | `session_control.rs` / `session.rs` | 跨会话（文件级） | 恢复整个对话状态 |

### 3. 工具系统的权限双轨制

白皮书中提到的 "双轨制" 在 Rust 实现中被拆分为两个明确的 crate 层次：

- **应用层权限**：[`runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) + [`permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs)
- **OS 层沙箱**：[`runtime/src/sandbox.rs`](/rust/crates/runtime/src/sandbox.rs) + [`runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs)

#### 应用层：`PermissionMode` 五级模型

[`permissions.rs#L9-L15`](/rust/crates/runtime/src/permissions.rs#L9-L15)：

```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,
    Allow,
}
```

`PermissionPolicy::authorize_with_context(...)` 在 `run_turn` 的工具执行路径中被调用（[`conversation.rs#L348-L490`](/rust/crates/runtime/src/conversation.rs#L348-L490)）。如果当前模式是 `Prompt`，系统会调用 `PermissionPrompter` 向用户展示确认对话框。

`PermissionEnforcer` 在 [`permission_enforcer.rs#L44-L134`](/rust/crates/runtime/src/permission_enforcer.rs#L44-L134) 中进一步细化：
- `check_file_write()`：在 `WorkspaceWrite` 模式下验证路径前缀，防止写出工作区
- `check_bash()`：在 `ReadOnly` 模式下仅允许 "白名单命令"（`cat`, `grep`, `ls` 等），并拒绝包含重定向符 `> ` 或 `--in-place` 的调用

#### OS 层：`SandboxConfig` 与 namespace 隔离

[`sandbox.rs#L8-L38`](/rust/crates/runtime/src/sandbox.rs#L8-L38) 定义了 `SandboxConfig` 和 `FilesystemIsolationMode`，支持三种文件系统隔离模式：

```rust
pub enum FilesystemIsolationMode {
    Off,
    WorkspaceOnly,
    AllowList,
}
```

在 Bash 执行路径中，`bash.rs` 会根据输入中的 `dangerously_disable_sandbox`、`isolate_network`、`filesystem_mode` 等字段构造 `SandboxStatus`，最终决定命令是否通过 Linux namespace/unshare 运行（[`bash.rs#L170-L247`](/rust/crates/runtime/src/bash.rs#L170-L247)）。

两层的信任假设完全不同：应用层依赖用户配置和 AI 行为约束；OS 层不信任任何事物，即使 AI 的输入试图绕过权限，沙箱仍会在内核层面限制文件系统、网络和进程可见性。这就是纵深防御。

### 4. Feature Flag 的全局开关

上游 TypeScript 中有一行著名的代码 `const feature = (_name: string) => false;`，它通过编译时注入控制功能可见性。`claw-code` 的 Rust 实现没有直接复刻这行代码，而是将 "功能开关" 工程化为**结构化的 `RuntimeFeatureConfig`**。

不过这里有一个关键差异：TypeScript 的 `feature()` 可以在运行时通过远程配置或 A/B 测试快速翻转，而 Rust 的 `RuntimeFeatureConfig` 要在编译后才能修改。这意味着 `claw-code` 更适合稳定和可控的部署场景，但在需要快速灰度发布或动态 kill switch 的场景下，灵活性显著低于上游。`reload_runtime_features()` 通过重建整个 `ConversationRuntime` 来实现热重载，代价是会话状态需要完整拷贝，而不是原地更新。

#### `RuntimeFeatureConfig`：类型安全的 feature 开关

参见 [`config.rs#L56-L75`](/rust/crates/runtime/src/config.rs#L56-L75)：

```rust
pub struct RuntimeFeatureConfig {
    hooks: RuntimeHookConfig,
    plugins: RuntimePluginConfig,
    mcp: McpConfigCollection,
    oauth: Option<OAuthConfig>,
    model: Option<String>,
    aliases: BTreeMap<String, String>,
    permission_mode: Option<ResolvedPermissionMode>,
    permission_rules: RuntimePermissionRuleConfig,
    sandbox: SandboxConfig,
    provider_fallbacks: ProviderFallbackConfig,
    trusted_roots: Vec<String>,
}
```

这不是运行时字符串匹配，而是编译期强类型的功能配置。`ConfigLoader::load()` 会按 `User → Project → Local` 的优先级合并多个 `.claw.json` / `.claw/settings.json` 文件，最终生成统一的 `RuntimeFeatureConfig`（[`config.rs#L310-L335`](/rust/crates/runtime/src/config.rs#L310-L335)）。

在 CLI 层面，[`main.rs#L4647`](/rust/crates/rusty-claude-cli/src/main.rs#L4647) 的 `reload_runtime_features()` 支持在运行时会话中热重载配置，无需重启进程即可切换权限模式、模型别名、hook 列表等。

### 5. Compaction 的分档策略

上游白皮书提到三种压缩策略：Micro-compact、Auto-compact 和 Reactive-compact。Rust 实现将这些策略编码在 [`runtime/src/compact.rs`](/rust/crates/runtime/src/compact.rs) 和 `conversation.rs` 的协同逻辑中。

#### Micro-compact：单次工具输出截断

[`bash.rs#L288-L304`](/rust/crates/runtime/src/bash.rs#L288-L304) 定义了 `MAX_OUTPUT_BYTES = 16_384`，`truncate_output()` 会在 Bash 输出超过阈值时截断并追加标记：`[output truncated — exceeded 16384 bytes]`。这是 "单次输出过长" 的即时压缩。

#### Auto-compact：对话 token 接近上限时自动压缩历史

[`conversation.rs#L17-L18`](/rust/crates/runtime/src/conversation.rs#L17-L18) 定义了默认阈值：

```rust
const DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD: u32 = 100_000;
const AUTO_COMPACTION_THRESHOLD_ENV_VAR: &str = "CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS";
```

每次 `run_turn` 循环结束后，`maybe_auto_compact()` 会检查累计 input tokens。如果超过阈值，调用 `compact_session()`（[`conversation.rs#L521-L548`](/rust/crates/runtime/src/conversation.rs#L521-L548)）。

#### Reactive-compact：API 返回 token 超限时紧急压缩

当 API 客户端返回 413/429 或 token 超限错误时，上游的 Reactive-compact 策略会触发。在 Rust 实现中，虽然具体的重试-压缩耦合逻辑仍在完善中（参见 `PARITY.md` 的 "Session compaction behavior matching" 项仍标记为未完成），但 `compact_session()` 已经具备被紧急调用的全部能力。

#### `compact_session` 的语义保留机制

[`compact.rs#L96-L139`](/rust/crates/runtime/src/compact.rs#L96-L139)：

```rust
pub fn compact_session(session: &Session, config: CompactionConfig) -> CompactionResult {
    // 保留最近 N 条消息（默认 4 条），压缩更早的消息
    let keep_from = session
        .messages
        .len()
        .saturating_sub(config.preserve_recent_messages);
    let removed = &session.messages[compacted_prefix_len..keep_from];
    let preserved = session.messages[keep_from..].to_vec();
    // ... 生成结构化摘要，而非简单丢弃
}
```

压缩后，会话的消息列表会以一条 `MessageRole::System` 的 "continuation message" 开头，其内容是一段由 `format_compact_summary()` 生成的自由文本摘要，内部包含 `Summary`、`Key timeline`、`Tools mentioned`、`Pending work` 等结构化项目符号（[`compact.rs#L150-L290`](/rust/crates/runtime/src/compact.rs#L150-L288)）。这确保了模型在压缩后仍能理解对话的语义脉络——不是砍掉旧消息，而是用 AI 可读的格式重新编码它们。

---

## 阅读路线图

如果你希望通过 `claw-code` 的 Rust 源码来系统学习 Claude Code 的架构，推荐的阅读顺序如下：

```
什么是 Claude Code (01-what-is-claude-code.md) ← 建立直觉
│
├── 架构全景 ← 五层架构 + 数据流
│   ├── 安全体系 ← 信任与控制
│   ├── 权限模型 ← PermissionMode / PermissionEnforcer
│   ├── 沙箱机制 ← sandbox.rs / bash.rs
│   └── Plan Mode ← 用户主导模式
│
├── 对话引擎 ← AI 如何思考
│   ├── Agentic Loop ← conversation.rs::run_turn
│   ├── 流式响应 ← api/crates 的 Provider trait
│   └── 多轮对话 ← Session / ContentBlock
│
├── 上下文工程 ← 记忆与预算
│   ├── System Prompt ← prompt.rs 的组装逻辑
│   ├── Token 预算 ← usage.rs + auto-compaction
│   └── 项目记忆 ← session_control.rs 持久化
│
├── 工具系统 ← AI 的双手
│   ├── 工具概览 ← tools/src/lib.rs 的 40 个 tool specs
│   ├── Shell 执行 ← bash.rs + bash_validation.rs
│   └── 搜索与导航 ← file_ops.rs (Glob/Grep/Read/Write)
│
└── Agent 与扩展 ← 能力扩展
    ├── 子 Agent ← runtime/src/worker_boot.rs / task_registry.rs
    ├── 自定义 Agent ← plugin_lifecycle.rs
    └── MCP 协议 ← mcp_tool_bridge.rs / mcp_stdio.rs / mcp_client.rs
```

本地源码的 `PARITY.md` 是一份活的进度表：它记录了哪些上游行为已经落地、哪些仍在分支或 stub 状态。建议在阅读任何模块前先查看对应 lane 的合并状态和 harness 覆盖范围。

### 审视角：parity 追踪的隐形成本

`PARITY.md` 的 9-lane checkpoint 模型给人一种"进度可视化"的错觉，但它没有回答一个更根本的问题：**哪些缺失的功能是架构性的，哪些只是工程量的堆积？** 例如，LSP 完整接入的缺失会根本性地限制代码导航能力，而某些 UI 动画的缺失只是体验层面的。属于前者的缺口如果在新功能开发中被持续搁置，会导致 Rust 实现与上游的语义差距不断扩大，最终使 `claw-code` 成为一个"能编译但不好用"的参考实现。这要求读者不仅看"完成了百分之多少"，还要看"缺失的是不是 load-bearing 功能"。

---

## 适合谁读

- **AI Agent 开发者**：`claw-code` 的 `ConversationRuntime` 是一个生产级 agentic loop 的 Rust 参考实现，比阅读 TypeScript bundle 更容易追踪类型和调用链。
- **安全工程师**：`permissions.rs`、`permission_enforcer.rs` 和 `sandbox.rs` 展示了如何在"完整 shell 能力"和"用户可控"之间建立纵深防御。
- **工具构建者**：如果你正在用 Rust/Go/Python 构建 coding assistant，`api` crate 的 `Provider` trait 和 `tools` crate 的 tool spec 注册机制提供了一套清晰的扩展接口。
- **好奇心驱动的开发者**：想知道 Claude Code 的 "打字机效果"、"自动压缩"、"权限提示" 到底怎么工作的——这里有可编译的答案。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime`、`run_turn`、agentic loop、自动压缩触发 |
| [`/rust/crates/runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) | `Session`、`ContentBlock`、消息持久化与结构化存储 |
| [`/rust/crates/runtime/src/compact.rs`](/rust/crates/runtime/src/compact.rs) | `compact_session`、三种 compaction 策略、语义摘要生成 |
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode`、`PermissionPolicy`、应用层权限模型 |
| [`/rust/crates/runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) | `PermissionEnforcer`、文件写入边界检查、Bash 只读白名单 |
| [`/rust/crates/runtime/src/sandbox.rs`](/rust/crates/runtime/src/sandbox.rs) | `SandboxConfig`、`FilesystemIsolationMode`、OS 层沙箱配置 |
| [`/rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs) | Bash 工具 schema、输出截断 (`MAX_OUTPUT_BYTES`)、沙箱执行 |
| [`/rust/crates/runtime/src/prompt.rs`](/rust/crates/runtime/src/prompt.rs) | `load_system_prompt`、指令文件发现、git 上下文注入、缓存策略 |
| [`/rust/crates/runtime/src/config.rs`](/rust/crates/runtime/src/config.rs) | `RuntimeFeatureConfig`、配置合并优先级 (User > Project > Local) |
| [`/rust/crates/runtime/src/hooks.rs`](/rust/crates/runtime/src/hooks.rs) | `HookRunner`、`HookAbortSignal`、工具调用前后生命周期钩子 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、参数解析、`reload_runtime_features`、会话管理 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | 40 个 tool specs、`execute_tool` 分发逻辑 |
| [`/PARITY.md`](/PARITY.md) | Rust port 的完整 parity 状态、9-lane checkpoint、harness 覆盖 |
