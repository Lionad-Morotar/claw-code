# 多轮对话管理 - QueryEngine 会话编排与持久化

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/conversation/multi-turn) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 单轮 vs 多轮：架构层面的差异

- **单轮（一次 Agentic Loop）**：`run_turn()` 函数的一次完整执行——组装上下文 → 调 API → 处理工具调用 → 循环直到结束。
- **多轮（一个 Session）**：`LiveCli` / `ConversationRuntime` 管理的一次会话——跨越数十轮 `run_turn()` 调用，持续数小时。

在 TypeScript 上游中，这个角色由 `QueryEngine` 担任。在 `claw-code` 的 Rust 实现里，等价的会话编排器是 `ConversationRuntime` 及其外层的 `LiveCli`。`ConversationRuntime` 持有跨 turn 累积的全部状态，是“多轮”与“单轮”之间的分水岭。

### 源码映射：`ConversationRuntime` 的状态机与 `Session`

`ConversationRuntime` 定义于 [`runtime/src/conversation.rs#L126-L188`](/rust/crates/runtime/src/conversation.rs#L126-L188)。它内部持有：

- `session: Session` —— 完整对话历史，跨 turn 累积
- `api_client: C` —— 满足 `ApiClient` trait 的流式模型客户端
- `tool_executor: T` —— 满足 `ToolExecutor` trait 的工具调度器
- `permission_policy: PermissionPolicy` —— 权限策略
- `usage_tracker: UsageTracker` —— token 预算追踪
- `system_prompt: Vec<String>` —— 每次 turn 重新注入的 system prompt
- `hook_runner: HookRunner` —— 生命周期 hooks

而真正持久化的消息历史存储在 [`runtime/src/session.rs#L88-L103`](/rust/crates/runtime/src/session.rs#L88-L103) 的 `Session` 结构体中：

```rust
pub struct Session {
    pub version: u32,
    pub session_id: String,
    pub created_at_ms: u64,
    pub updated_at_ms: u64,
    pub messages: Vec<ConversationMessage>,
    pub compaction: Option<SessionCompaction>,
    pub fork: Option<SessionFork>,
    pub workspace_root: Option<PathBuf>,
    pub prompt_history: Vec<SessionPromptEntry>,
    /// The model used in this session, persisted so resumed sessions can
    /// report which model was originally used.
    pub model: Option<String>,
    persistence: Option<SessionPersistence>,
}
```

注意 `messages: Vec<ConversationMessage>` 是会话级状态。每次 `run_turn` 结束时，新的用户消息、assistant 消息和 tool result 都已经追加到 `session.messages` 中。下一 turn 开始时，`ApiRequest` 直接 `clone()` 整个 `messages` 列表送入模型（[`conversation.rs#L321-L324`](/rust/crates/runtime/src/conversation.rs#L321-L324)）。这就是“多轮”的物理实现：所有历史消息作为上下文重新提交。

---

## QueryEngine 的核心方法：`submitMessage()`

TypeScript 上游中，每次用户输入都会调用 `QueryEngine.submitMessage()`。在 Rust 实现中，对应入口是 `LiveCli::run_turn()`（REPL 模式）和 `LiveCli::run_turn_with_output()`（Prompt 模式）。它们都会调用 `ConversationRuntime::run_turn()`。

### 源码映射：`run_turn` 的 turn 初始化链路

`ConversationRuntime::run_turn` 定义于 [`runtime/src/conversation.rs#L296-L485`](/rust/crates/runtime/src/conversation.rs#L296-L485)。简化后的执行链路如下：

```rust
pub fn run_turn(
    &mut self,
    user_input: impl Into<String>,
    mut prompter: Option<&mut dyn PermissionPrompter>,
) -> Result<TurnSummary, RuntimeError> {
    // 1. 记录 turn 开始、将用户消息追加到 Session
    self.session.push_user_text(user_input)?;

    // 2. 进入 agentic loop
    loop {
        // 3. 组装请求：system_prompt + 当前完整消息历史
        let request = ApiRequest {
            system_prompt: self.system_prompt.clone(),
            messages: self.session.messages.clone(),
        };

        // 4. 调用 API 获取流式事件
        let events = self.api_client.stream(request)?;
        let (assistant_message, usage, cache_events) = build_assistant_message(events)?;

        // 5. 累加 token 用量
        if let Some(usage) = usage { self.usage_tracker.record(usage); }

        // 6. 检测 ToolUse，并将 assistant 消息回写 Session
        let pending_tool_uses = extract_tool_uses(&assistant_message);
        self.session.push_message(assistant_message.clone())?;

        if pending_tool_uses.is_empty() { break; }

        // 7. 执行工具、权限检查、hooks，并将 ToolResult 回写 Session
        for (tool_use_id, tool_name, input) in pending_tool_uses {
            // 权限检查 + hooks ...
            let result_message = ...;
            self.session.push_message(result_message)?;
        }
    }

    // 8. 自动触发 compaction（如果满足阈值）
    let auto_compaction = self.maybe_auto_compact();

    Ok(TurnSummary { ... })
}
```

关键设计：`run_turn()` 返回 `Result<TurnSummary, RuntimeError>`，而不是 async generator。`claw-code` 目前采用同步事件收集（`Vec<AssistantEvent>`），由更上层的 `LiveCli` 负责实时渲染输出。这意味着调用方不需要等待整个 turn 结束才展示结果，因为流式事件在 `AnthropicRuntimeClient::stream()` 内部已被解析为事件列表，随后由渲染层逐条刷新。

---

## 会话持久化：JSONL Transcript

`claw-code` 的会话持久化形式是 JSON Lines（`.jsonl`）。每次消息追加、prompt 记录、compaction 元数据都会写入同一文件，无需读取-修改-重写整个文件。

### 源码映射：存储路径与 `SessionStore`

会话文件存储在项目的 `.claw/sessions/<workspace_hash>/` 目录下。这个路径隔离逻辑由 [`runtime/src/session_control.rs#L19-L63`](/rust/crates/runtime/src/session_control.rs#L19-L63) 的 `SessionStore` 负责：

```rust
pub struct SessionStore {
    sessions_root: PathBuf,
    workspace_root: PathBuf,
}

impl SessionStore {
    pub fn from_cwd(cwd: impl AsRef<Path>) -> Result<Self, SessionControlError> {
        let sessions_root = cwd.join(".claw")
            .join("sessions")
            .join(workspace_fingerprint(cwd));
        fs::create_dir_all(&sessions_root)?;
        Ok(Self { sessions_root, workspace_root: cwd.to_path_buf() })
    }
}
```

`<workspace_hash>` 由 `workspace_fingerprint()`（[`session_control.rs#L246-L254`](/rust/crates/runtime/src/session_control.rs#L246-L254)）基于 FNV-1a 64-bit 哈希生成，确保同一项目目录的会话归入同一子目录，而并行打开的不同工作区不会互相覆盖。

### 源码映射：Transcript 写入器

`Session` 的追加写入逻辑在 [`runtime/src/session.rs#L533-L552`](/rust/crates/runtime/src/session.rs#L533-L552)：`push_message` 成功后会调用 `append_persisted_message()`：

```rust
fn append_persisted_message(&self, message: &ConversationMessage) -> Result<(), SessionError> {
    let Some(path) = self.persistence_path() else {
        return Ok(());
    };

    let needs_bootstrap = !path.exists() || fs::metadata(path)?.len() == 0;
    if needs_bootstrap {
        self.save_to_path(path)?; // 首次写入：输出完整 JSONL snapshot
        return Ok(());
    }

    let mut file = OpenOptions::new().append(true).open(path)?;
    writeln!(file, "{}", message_record(message).render())?;
    Ok(())
}
```

`save_to_path()`（[`session.rs#L195-L202`](/rust/crates/runtime/src/session.rs#L195-L202)）会先生成完整的 `render_jsonl_snapshot()`，它按顺序输出：

1. `session_meta` 记录（含 `session_id`、`created_at_ms`、`workspace_root` 等）
2. `compaction` 记录（如果有）
3. `prompt_history` 记录
4. `message` 记录

这种设计与 TypeScript 上游的 `TranscriptWriter` 写队列等价：追加写入是原子操作，并发消息追加不会互相覆盖。`rotate_session_file_if_needed` 会在文件超过 `ROTATE_AFTER_BYTES`（256KB）时进行轮转，最多保留 `MAX_ROTATED_FILES`（3）个历史文件。

### 源码映射：从 JSONL 恢复会话

`Session::load_from_path()`（[`session.rs#L204-L219`](/rust/crates/runtime/src/session.rs#L204-L218)）支持两种格式：

- 以 `"messages"` 为顶层键的完整 JSON 对象（旧格式兼容）
- JSONL 格式：逐行按 `type` 字段解析，拼回 `Session`

解析逻辑在 `from_jsonl()`（[`session.rs#L388-L403`](/rust/crates/runtime/src/session.rs#L388-L403)），对 `session_meta`、`message`、`compaction`、`prompt_history` 四种记录类型作状态机还原：

```rust
match type_ {
    "session_meta" => { ... }
    "message" => { messages.push(ConversationMessage::from_json(...)?); }
    "compaction" => { compaction = Some(SessionCompaction::from_json(...)?); }
    "prompt_history" => { prompt_history.push(...); }
    other => { return Err(...); }
}
```

恢复流程最终由 `LiveCli::resume_session()`（[`rusty-claude-cli/src/main.rs#L4348-L4388`](/rust/crates/rusty-claude-cli/src/main.rs#L4348-L4388)）调用：加载 JSONL 文件 → 重建 `Session` → 通过 `build_runtime()` 创建新的 `ConversationRuntime` → 将 `LiveCli` 的当前 `session` handle 指向恢复的文件。

---

## 成本追踪：从 API Usage 到美元

成本追踪在 `claw-code` 中贯穿记录、累计、展示三个层次。

### 源码映射：记录层 — `TokenUsage` 与 `UsageCostEstimate`

`runtime/src/usage.rs#L30-L55` 定义了原始计数器和美元估算结构：

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub cache_creation_input_tokens: u32,
    pub cache_read_input_tokens: u32,
}

pub struct UsageCostEstimate {
    pub input_cost_usd: f64,
    pub output_cost_usd: f64,
    pub cache_creation_cost_usd: f64,
    pub cache_read_cost_usd: f64,
}
```

每个 API 响应中的 `usage` 字段在 `build_assistant_message()`（[`conversation.rs#L676-L723`](/rust/crates/runtime/src/conversation.rs#L676-L723)）被提取。随后 `run_turn` 通过 `self.usage_tracker.record(usage)` 累加到会话总量（[`conversation.rs#L342`](/rust/crates/runtime/src/conversation.rs#L342)）。

### 源码映射：累计层 — `UsageTracker`

`UsageTracker` 定义于 [`runtime/src/usage.rs#L168-L215`](/rust/crates/runtime/src/usage.rs#L168-L215)：

```rust
#[derive(Debug, Clone, Default, PartialEq, Eq)]
pub struct UsageTracker {
    latest_turn: TokenUsage,
    cumulative: TokenUsage,
    turns: u32,
}
```

它可以从已恢复的 `Session` 重建：`UsageTracker::from_session(&session)` 遍历所有 `messages`，提取 `message.usage` 并逐条 `record()`（[`usage.rs#L182-L190`](/rust/crates/runtime/src/usage.rs#L182-L190)）。这意味着即使 `--resume` 恢复了一个旧会话，累计费用也能正确延续。

费用计算使用模型专属定价（[`usage.rs#L59-L81`](/rust/crates/runtime/src/usage.rs#L59-L81)）：

```rust
pub fn pricing_for_model(model: &str) -> Option<ModelPricing> {
    let normalized = model.to_ascii_lowercase();
    if normalized.contains("haiku") { ... }
    if normalized.contains("opus") { ... }
    if normalized.contains("sonnet") { return Some(ModelPricing::default_sonnet_tier()); }
    None
}
```

CLI 在 REPL 中通过 `/cost` 命令调用 `LiveCli::print_cost()`（[`main.rs#L4343-L4346`](/rust/crates/rusty-claude-cli/src/main.rs#L4343-L4346)），或者在每次 JSON 输出中附加 `estimated_cost`（[`main.rs#L3938-L3948`](/rust/crates/rusty-claude-cli/src/main.rs#L3938-L3948)）。

### 预算熔断

当前 `claw-code` 暂未实现 TypeScript 上游中的 `maxBudgetUsd` 硬性阻断。会话级成本目前是“追踪与提醒”模式： exceeding thresholds 会显示在 `/status` 和 `/cost` 中，但没有在 `ConversationRuntime` 层硬编码的成本上限拦截器。如果需要引入，合理的拦截点应在 `run_turn` 调用 `api_client.stream()` 前检查 `usage_tracker.cumulative_usage()`。

---

## 模型热切换

在一个会话中切换模型不会丢失对话历史——因为 `messages` 与模型选择是完全解耦的。

### 源码映射：`/model` 命令的实现

REPL 中的 `/model sonnet` 由 `LiveCli::set_model()` 处理（[`rusty-claude-cli/src/main.rs#L4213-L4265`](/rust/crates/rusty-claude-cli/src/main.rs#L4213-L4265)）：

```rust
fn set_model(&mut self, model: Option<String>) -> Result<bool, Box<dyn std::error::Error>> {
    let model = resolve_model_alias_with_config(&model);

    let session = self.runtime.session().clone();
    let runtime = build_runtime(
        session,
        &self.session.id,
        model.clone(),      // <-- 新模型
        self.system_prompt.clone(),
        true, true, ..., self.permission_mode, None,
    )?;
    self.replace_runtime(runtime)?;
    self.model.clone_from(&model);
    ...
}
```

关键行为：

1. 保存当前 `Session`（含完整 `messages`）
2. 用新模型重新 `build_runtime()`
3. 在 `build_runtime()` 内部，`AnthropicRuntimeClient` 会根据新的 `model` 字符串重新选择 provider endpoint
4. 新 `runtime` 继续引用同一 `session`，因此历史消息全部保留

`system_prompt` 也可以根据模型能力重新组装。`build_system_prompt()` 在 `LiveCli::new()` 时调用一次，当前实现中不会在 `/model` 后自动刷新 system prompt——架构上虽然完全支持，但这是一个尚未对齐上游的行为差异。

---

## 上下文压缩（Compaction）

当 token 用量超过阈值时，`ConversationRuntime` 会自动触发 compaction，将旧消息压缩成摘要，保留最近若干条消息原样。

### 源码映射：自动压缩触发

阈值默认为 100,000 input tokens，可通过环境变量 `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS` 覆盖（[`conversation.rs#L660-L675`](/rust/crates/runtime/src/conversation.rs#L660-L674)）。触发逻辑在 `maybe_auto_compact()`（[`conversation.rs#L525-L548`](/rust/crates/runtime/src/conversation.rs#L525-L548)）：

```rust
fn maybe_auto_compact(&mut self) -> Option<AutoCompactionEvent> {
    if self.usage_tracker.cumulative_usage().input_tokens
        < self.auto_compaction_input_tokens_threshold
    {
        return None;
    }

    let result = compact_session(
        &self.session,
        CompactionConfig {
            max_estimated_tokens: 0,
            ..CompactionConfig::default()
        },
    );

    if result.removed_message_count == 0 {
        return None;
    }

    self.session = result.compacted_session;
    Some(AutoCompactionEvent { removed_message_count: result.removed_message_count })
}
```

注意这里将 `max_estimated_tokens` 设为 `0`，配合默认的 `preserve_recent_messages: 4`，目的是尽可能压缩（因为已经触发了硬性 token 阈值），同时保留最近 4 条消息的原貌。

### 源码映射：`compact_session` 的实现

`compact_session` 定义于 [`runtime/src/compact.rs#L96-L139`](/rust/crates/runtime/src/compact.rs#L96-L139)。核心算法：

1. 检查 `should_compact()` —— 可压缩消息数超过 `preserve_recent_messages` 且 token 估计超过阈值
2. 提取已有摘要（如果 `messages[0]` 是系统摘要消息）
3. 保留尾部 `preserve_recent_messages` 条消息
4. 对其余消息调用 `summarize_messages()` 生成结构化摘要
5. 合并旧摘要与新摘要
6. 构造一条新的 `System` 消息作为压缩结果，替换被移除的历史

摘要格式示例（经 `format_compact_summary` 处理）：

```
Summary:
Conversation summary:
- Scope: 6 earlier messages compacted (user=2, assistant=2, tool=2).
- Tools mentioned: bash, edit_file.
- Recent user requests:
  - 请帮我修复这个 TypeScript 报错
- Pending work:
  - 更新测试用例
- Key files referenced: src/utils.ts
- Key timeline:
  - user: 请帮我修复这个 TypeScript 报错
  - assistant: I'll check the error first.
  - tool_use: bash({...})
```

如果已有先前压缩的摘要，合并时会保留为 `Previously compacted context:`，新压缩部分作为 `Newly compacted context:`（[`compact.rs#L238-L270`](/rust/crates/runtime/src/compact.rs#L238-L270)）。这避免了反复压缩导致早期上下文彻底丢失。

### 源码映射：System Prompt 与上下文组装

每次 turn 的 `system_prompt` 由 `SystemPromptBuilder` 组装。虽然 `ConversationRuntime` 持有 `system_prompt` 字段，但实际内容通常由上层 `LiveCli` 在初始化时通过 `build_system_prompt()` → `load_system_prompt()` 生成。

`load_system_prompt()` 定义于 [`runtime/src/prompt.rs#L432-L446`](/rust/crates/runtime/src/prompt.rs#L432-L446)：

```rust
pub fn load_system_prompt(
    cwd: impl Into<PathBuf>,
    current_date: impl Into<String>,
    os_name: impl Into<String>,
    os_version: impl Into<String>,
) -> Result<Vec<String>, PromptBuildError> {
    let cwd = cwd.into();
    let project_context = ProjectContext::discover_with_git(&cwd, current_date.into())?;
    let config = ConfigLoader::default_for(&cwd).load()?;
    Ok(SystemPromptBuilder::new()
        .with_os(os_name, os_version)
        .with_project_context(project_context)
        .with_runtime_config(config)
        .build())
}
```

`SystemPromptBuilder::build()`（[`prompt.rs#L134-L150`](/rust/crates/runtime/src/prompt.rs#L134-L150)）输出多个 section：简单介绍、输出风格、系统提示、任务指导、环境上下文、项目上下文、指令文件、运行时配置等。其中也包含一条明确的提示：

> "The system may automatically compress prior messages as context grows."

这向模型提前声明了 compaction 机制的存在。

---

## 会话恢复与 Fork

除了 `--resume` 恢复会话外，`claw-code` 还支持显式 `/clear --confirm` 创建新会话，以及通过 API fork 会话分支。

### 源码映射：`fork` 与 `clear`

`Session::fork()` 定义于 [`runtime/src/session.rs#L251-L267`](/rust/crates/runtime/src/session.rs#L251-L267)：

```rust
pub fn fork(&self, branch_name: Option<String>) -> Self {
    let now = current_time_millis();
    Self {
        version: self.version,
        session_id: generate_session_id(),
        created_at_ms: now,
        updated_at_ms: now,
        messages: self.messages.clone(),
        compaction: self.compaction.clone(),
        fork: Some(SessionFork {
            parent_session_id: self.session_id.clone(),
            branch_name: normalize_optional_string(branch_name),
        }),
        workspace_root: self.workspace_root.clone(),
        prompt_history: self.prompt_history.clone(),
        persistence: None,
    }
}
```

Fork 后的会话继承原有消息历史和 prompt 历史，但拥有新的 `session_id` 和 `created_at_ms`，并通过 `SessionFork` 记录亲缘关系。`SessionStore::fork_session()` 会为新 session 分配持久化路径并立即保存（[`session_control.rs#L218-L238`](/rust/crates/runtime/src/session_control.rs#L218-L238)）。

`/clear --confirm` 的逻辑在 `LiveCli::clear_session()`（[`main.rs#L4308-L4346`](/rust/crates/rusty-claude-cli/src/main.rs#L4308-L4346)）：创建全新 `Session` → 新 `session_id` → 新持久化文件 → 保留当前 model / permission mode。旧会话文件仍然保留在 `.claw/sessions/` 中，可通过 `--resume` 重新加载。

### 审视角：持久化的信任假设

JSONL 格式虽然便于追加和手动检查，但它不对记录进行签名或校验。如果恶意进程或用户修改了 `.claw/sessions/` 下的文件（例如将 `ToolResult` 替换为精心构造的 prompt 注入内容），`Session::load_from_path()` 仍会将其作为合法历史加载到上下文中。当前实现没有验证消息来源的完整性，安全边界完全依赖文件系统权限（`~/.claw/sessions/` 的访问控制）。这在一个多用户共享工作站或 CI 环境中可能构成未被充分认识到的攻击面。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime`、`run_turn`、agentic loop、auto compaction 触发 |
| [`/rust/crates/runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) | `Session`、`ConversationMessage`、`ContentBlock`、JSONL 持久化与恢复 |
| [`/rust/crates/runtime/src/session_control.rs`](/rust/crates/runtime/src/session_control.rs) | `SessionStore`、工作区指纹隔离、会话列表/加载/fork |
| [`/rust/crates/runtime/src/compact.rs`](/rust/crates/runtime/src/compact.rs) | `compact_session`、`CompactionConfig`、摘要合并与格式化 |
| [`/rust/crates/runtime/src/usage.rs`](/rust/crates/runtime/src/usage.rs) | `TokenUsage`、`UsageTracker`、模型定价与美元估算 |
| [`/rust/crates/runtime/src/prompt.rs`](/rust/crates/runtime/src/prompt.rs) | `load_system_prompt`、`SystemPromptBuilder`、project context 注入 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | `LiveCli`、REPL 命令处理、`/model`、`/resume`、`/clear`、`/cost` |
