# 架构全景 - Claude Code 五层架构详解

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/introduction/architecture-overview) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 五层架构

Claude Code 从上到下分为五个层次，每一层职责清晰、边界分明：

| 层次 | 职责 | 入口源码 | 关键词 |
| --- | --- | --- | --- |
| **交互层** | 终端 UI、用户输入、消息展示 | [`rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI、REPL、PromptInput |
| **编排层** | 多轮对话、会话持久化、成本追踪 | [`runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | ConversationRuntime、Session |
| **核心循环层** | 单轮：发请求 → 拿响应 → 执行工具 → 循环 | [`runtime/src/conversation.rs#L296-L490`](/rust/crates/runtime/src/conversation.rs#L296-L490) | Agentic Loop、turn |
| **工具层** | AI 的"双手"——读写文件、执行命令 | [`tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | ToolExecutor、MCP |
| **通信层** | 与模型 API 的流式通信 | [`api/src/client.rs`](/rust/crates/api/src/client.rs) | Streaming、Provider |

### Rust 实现中的层次边界

在 `claw-code` 中，这五层不是抽象概念，而是 9 个 Cargo crate 的真实模块边界：

- `rusty-claude-cli` 承载**交互层**
- `runtime` 承载**编排层**和**核心循环层**
- `tools` 承载**工具层**
- `api` 承载**通信层**

这种 crate 级别的物理隔离意味着：修改 TUI 渲染不会影响 API 协议，新增模型提供商不需要改动工具系统。

---

## 一条主数据流的源码追踪

整个系统的运转可以浓缩为一条核心数据流。以下是将原文 TypeScript 架构映射到 Rust 实现后的完整源码路径。

### 1. 用户输入 → CLI

[`rusty-claude-cli/src/main.rs#L168-L240`](/rust/crates/rusty-claude-cli/src/main.rs#L168-L240) 的 `run()` 函数是进程级入口。它把命令行参数解析为 `CliAction` 枚举，随后按模式分发：

```rust
fn run() -> Result<(), Box<dyn std::error::Error>> {
    let args: Vec<String> = env::args().skip(1).collect();
    match parse_args(&args)? {
        CliAction::Prompt { prompt, model, output_format, allowed_tools, permission_mode, compact, base_commit, .. } => {
            // ... 单次非交互调用
            LiveCli::new(model, true, allowed_tools, permission_mode)?
                .run_turn_with_output(&effective_prompt, output_format, compact)?;
        }
        CliAction::Repl { model, allowed_tools, permission_mode, base_commit, .. } => {
            run_repl(model, allowed_tools, permission_mode, base_commit)?;
        }
        // ... Agents, Skills, Mcp, Doctor, Export 等
    }
    Ok(())
}
```

#### REPL 分支的数据流

[`run_repl`](/rust/crates/rusty-claude-cli/src/main.rs#L2789-L2838) 创建一个 [`input::LineEditor`](/rust/crates/rusty-claude-cli/src/input.rs) 循环读取终端输入，遇到斜杠命令时由 [`SlashCommand::parse`](/rust/crates/commands/src/lib.rs) 处理，普通文本则进入 `LiveCli::run_turn()`。

#### 管道输入的合并策略

在 [`CliAction::Prompt`](/rust/crates/rusty-claude-cli/src/main.rs#L204-L240) 分支中，系统通过 [`read_piped_stdin`](/rust/crates/rusty-claude-cli/src/main.rs#L110-L128) 检测 stdin 是否为终端。只有当 `permission_mode` 为 `DangerFullAccess`（完全无人值守模式）时，才会读取管道输入并与 prompt 合并——否则 stdin 需要保留给交互式权限确认使用。

---

### 2. 编排层：LiveCli + ConversationRuntime

进入 `LiveCli` 后，真正的编排核心在 [`runtime/src/conversation.rs#L126-L188`](/rust/crates/runtime/src/conversation.rs#L126-L188) 的 `ConversationRuntime<C, T>` 中。

#### LiveCli 到 Runtime 的组装链

在调用 `run_turn` 之前，[`prepare_turn_runtime`](/rust/crates/rusty-claude-cli/src/main.rs#L3442-L3459) 完成运行时的"冷启动"组装：

1. 调用 `build_runtime()` → `build_runtime_plugin_state()` 初始化 `ConfigLoader`、插件系统、`GlobalToolRegistry`
2. 根据 `permission_mode` 和 `tool_registry` 构造 `PermissionPolicy`
3. 创建 `AnthropicRuntimeClient`（封装 `api::ProviderClient` 的异步客户端）
4. 创建 `CliToolExecutor`（将模型请求映射到 `tools` crate 的具体实现）
5. 全部注入 `ConversationRuntime::new_with_features(...)`

#### ConversationRuntime 持有的核心状态

[`ConversationRuntime`](/rust/crates/runtime/src/conversation.rs#L126-L188) 持有 13 个字段：

- `session: Session` —— 消息历史、compaction 信息、持久化路径
- `api_client: C` —— 满足 `ApiClient` trait 的流式客户端
- `tool_executor: T` —— 满足 `ToolExecutor` trait 的工具调度器
- `permission_policy: PermissionPolicy` —— 静态规则 + 动态提示的权限策略
- `system_prompt: Vec<String>` —— 每次 turn 重新注入的系统提示
- `max_iterations: usize` —— 迭代上限保护（默认 `usize::MAX`）
- `usage_tracker: UsageTracker` —— 累计 input/output/cache token 用量
- `hook_runner: HookRunner` —— 插件生命周期钩子执行器
- `auto_compaction_input_tokens_threshold: u32` —— 自动压缩触发阈值（默认 100k）
- `hook_abort_signal: HookAbortSignal` —— hook 中断信号
- `hook_progress_reporter: Option<Box<dyn HookProgressReporter>>` —— hook 进度报告器
- `session_tracer: Option<SessionTracer>` —— 会话追踪器

这些字段构成了编排层的完整状态机。

---

### 3. Agentic Loop（runtime/src/conversation.rs）

[`run_turn`](/rust/crates/runtime/src/conversation.rs#L296-L490) 是核心循环的核心。每个 turn 内部是一个 `loop { ... }`，典型流程如下：

1. **写入用户输入**：`session.push_user_text(user_input)`
2. **组装 API 请求**：`ApiRequest { system_prompt, messages: session.messages.clone() }`
3. **流式调用模型**：`api_client.stream(request)` → `Vec<AssistantEvent>`
4. **构建 Assistant 消息**：`build_assistant_message(events)` 将流事件转换为 `ConversationMessage`
5. **检测 ToolUse**：扫描 `ContentBlock::ToolUse`，提取 `(id, name, input)`
6. **没有 ToolUse → break 循环，返回 TurnSummary**
7. **有 ToolUse → 执行权限检查 + hook + 工具执行 + 结果回写 session → continue 循环**

#### 流事件到消息块的转换

[`build_assistant_message`](/rust/crates/runtime/src/conversation.rs#L676-L723) 将离散的 `AssistantEvent` 聚合成结构化的 `ConversationMessage`：

```rust
fn build_assistant_message(events: Vec<AssistantEvent>) -> Result<(ConversationMessage, Option<TokenUsage>, Vec<PromptCacheEvent>), RuntimeError> {
    let mut text = String::new();
    let mut blocks = Vec::new();
    for event in events {
        match event {
            AssistantEvent::TextDelta(delta) => text.push_str(&delta),
            AssistantEvent::ToolUse { id, name, input } => {
                flush_text_block(&mut text, &mut blocks);
                blocks.push(ContentBlock::ToolUse { id, name, input });
            }
            AssistantEvent::Usage(value) => usage = Some(value),
            AssistantEvent::MessageStop => finished = true,
            // ...
        }
    }
    // 必须包含 MessageStop 且至少有一个 content block
}
```

这意味着流式响应在编排层被"物化"为离散的消息对象后才进入工具执行阶段——上层（如 CLI）负责渲染打字机效果，下层（runtime）负责状态机推进。

#### 权限与 Hook 的双重关卡

在 [`run_turn`](/rust/crates/runtime/src/conversation.rs#L348-L470) 的工具执行循环中，每个 ToolUse 需要经过两个关卡：

1. **Pre-tool-use hook**：`run_pre_tool_use_hook(&tool_name, &input)` —— 插件可以在执行前拦截、修改输入或取消调用
2. **PermissionPolicy**：`permission_policy.authorize_with_context(...)` —— 根据 `PermissionMode` 和规则决定 Allow / Deny / Prompt

只有两者都放行，才会调用 `tool_executor.execute(&tool_name, &effective_input)`。执行完成后还有 `post_tool_use_hook` 可以追加反馈或标记错误。

#### 自动压缩（Auto-compaction）

在每轮 turn 结束时，[`maybe_auto_compact`](/rust/crates/runtime/src/conversation.rs#L525-L548) 会检查 `usage_tracker.cumulative_usage().input_tokens` 是否超过阈值（默认 100,000）。若超过，则调用 [`compact_session`](/rust/crates/runtime/src/compact.rs#L96-L139) 将历史消息压缩为一条系统摘要消息，控制上下文长度和 API 成本。

---

### 4. 工具层（tools/src/lib.rs）

`claw-code` 的工具层实现位于 `tools` crate。[`execute_tool`](/rust/crates/tools/src/lib.rs#L1174-L1265) 是一个巨大的 `match name { ... }`，直接映射 40+ 个内置工具：

```rust
pub fn execute_tool(name: &str, input: &Value) -> Result<String, String> {
    match name {
        "bash" => from_value::<BashCommandInput>(input).and_then(run_bash),
        "read_file" => from_value::<ReadFileInput>(input).and_then(run_read_file),
        "write_file" => from_value::<WriteFileInput>(input).and_then(run_write_file),
        "edit_file" => from_value::<EditFileInput>(input).and_then(run_edit_file),
        "glob_search" => from_value::<GlobSearchInputValue>(input).and_then(run_glob_search),
        "grep_search" => from_value::<GrepSearchInput>(input).and_then(run_grep_search),
        "TodoWrite" => from_value::<TodoWriteInput>(input).and_then(run_todo_write),
        "Agent" => from_value::<AgentInput>(input).and_then(run_agent),
        "Skill" => from_value::<SkillInput>(input).and_then(run_skill),
        // ... MCP、Task、Worker、LSP、NotebookEdit、WebFetch、WebSearch 等
        _ => Err(format!("unsupported tool: {name}")),
    }
}
```

#### ToolExecutor trait 的实现

在 CLI 路径中，工具执行器是 `CliToolExecutor`（在 `main.rs` 内部定义，通过 `build_runtime_with_plugin_state` 注入）。在子 Agent 路径中，工具执行器是 [`SubagentToolExecutor`](/rust/crates/tools/src/lib.rs#L3951-L3975) —— 它会先检查 `allowed_tools` 白名单，再调用 `execute_tool_with_enforcer`。

这种 trait 抽象让同一套 runtime 可以同时支持：
- 主进程的全功能工具执行
- 受限子 Agent 的沙箱化工具执行

#### Bash 的安全设计

[`runtime/src/bash.rs#L19-L31`](/rust/crates/runtime/src/bash.rs#L19-L31) 定义了 `BashCommandInput` schema：

```rust
pub struct BashCommandInput {
    pub command: String,
    pub timeout: Option<u64>,
    pub description: Option<String>,
    pub run_in_background: Option<bool>,
    pub dangerously_disable_sandbox: Option<bool>,
    pub isolate_network: Option<bool>,
    pub filesystem_mode: Option<FilesystemIsolationMode>,
    pub allowed_mounts: Option<Vec<String>>,
}
```

沙箱状态由 [`sandbox_status_for_input`](/rust/crates/runtime/src/bash.rs#L188-L203) 根据运行时配置和模型请求共同决定。默认情况下命令在受限环境中执行；模型可以显式请求关闭沙箱（需要更高权限）。异步执行使用 `tokio::process::Command` + `tokio::time::timeout`，输出超过阈值时会被截断。

#### 文件操作的工作区边界

[`runtime/src/file_ops.rs#L32-L44`](/rust/crates/runtime/src/file_ops.rs#L32-L44) 定义了 `validate_workspace_boundary`：

```rust
fn validate_workspace_boundary(resolved: &Path, workspace_root: &Path) -> io::Result<()> {
    if !resolved.starts_with(workspace_root) {
        return Err(io::Error::new(
            io::ErrorKind::PermissionDenied,
            format!("path {} escapes workspace boundary {}", ...),
        ));
    }
    Ok(())
}
```

同时 [`is_symlink_escape`](/rust/crates/runtime/src/file_ops.rs#L610-L620) 检测符号链接是否解析到工作区之外。读、写、编辑三个操作都有对应的 `*_in_workspace` 变体，在 `PermissionEnforcer` 路径中被调用。

---

### 5. 通信层（api/src/client.rs）

`api` crate 的职责是与上游模型接口通信。[`ProviderClient`](/rust/crates/api/src/client.rs#L13-L19) 是一个 enum，覆盖三种后端：

```rust
pub enum ProviderClient {
    Anthropic(AnthropicClient),
    Xai(OpenAiCompatClient),
    OpenAi(OpenAiCompatClient),
}
```

#### 模型分发逻辑

[`ProviderClient::from_model_with_anthropic_auth`](/rust/crates/api/src/client.rs#L21-L47) 根据 `detect_provider_kind()` 决定路由：

- 以 `claude-` 开头的模型 → `AnthropicClient`（原生 Messages API，支持 prompt cache）
- 以 `grok-` 开头的模型 → `OpenAiCompatClient` + xAI 配置
- 以 `gpt-` 或 `openai/` 开头的模型 → `OpenAiCompatClient` + OpenAI 配置
- 以 `qwen-` 开头的模型 → `OpenAiCompatClient` + DashScope 配置

这意味着新增一个 OpenAI 兼容端点时，只需在 `api` crate 增加一条 `OpenAiCompatConfig`，上层无需改动。

#### Provider trait 抽象

所有后端都实现 [`Provider`](/rust/crates/api/src/providers/mod.rs#L14-L29) trait：

```rust
pub trait Provider {
    type Stream;
    fn send_message<'a>(&'a self, request: &'a MessageRequest) -> ProviderFuture<'a, MessageResponse>;
    fn stream_message<'a>(&'a self, request: &'a MessageRequest) -> ProviderFuture<'a, Self::Stream>;
}
```

#### 流式响应的终端渲染

在 CLI 侧，[`AnthropicRuntimeClient::consume_stream`](/rust/crates/rusty-claude-cli/src/main.rs#L6531-L6695) 把 `stream.next_event()` 的 SSE 事件实时转换为终端输出：

- `ContentBlockDelta::TextDelta` → 通过 [`MarkdownStreamState::push`](/rust/crates/rusty-claude-cli/src/render.rs#L601-L620) 增量渲染为 ANSI Markdown
- `ContentBlockDelta::InputJsonDelta` → 累加到 `pending_tool` 的 input JSON
- `ContentBlockStop` → 刷新 Markdown 缓冲区，若存在 pending tool 则输出工具调用提示
- `MessageDelta` → 记录 token usage
- `MessageStop` → 结束本次流式接收

若流意外中断（如 post-tool stall），`consume_stream` 会检测并在超时时重试一次 continuation nudge。

---

## 四个核心设计原则

### 流式优先 (Streaming-first)

所有 API 通信默认走流式路径：`api_client.stream(request)` 内部调用 `Provider::stream_message`，返回 `MessageStream`（SSE 事件迭代器）。CLI 层通过 `MarkdownStreamState::push` 将文本 delta 实时渲染到终端，用户能逐字看到 AI 的回答。

在 [`consume_stream`](/rust/crates/rusty-claude-cli/src/main.rs#L6531-L6695) 中，文本增量到达后立即调用 `write!(out, "{rendered}")` 并 `flush`，实现真正的低延迟打字机效果。

### 工具即能力 (Tool as Capability)

每个工具不是配置，而是可执行代码。`tools/src/lib.rs` 的 `execute_tool` 使用原生 Rust `match` 将工具名直接映射到实现函数。工具白名单由 `allowed_tools: Option<BTreeSet<String>>` 控制 —— `SubagentToolExecutor` 会前置过滤，`CliToolExecutor` 则允许全部（受权限策略约束）。

MCP 工具通过 `McpToolRegistry` 动态发现，运行时组装进 `GlobalToolRegistry`，实现"外部工具即原生工具"的能力扩展。

### 权限即边界 (Permission as Boundary)

权限检查发生在两个位置：

1. **运行时层**：`PermissionPolicy::authorize_with_context` 评估 `PermissionMode`、`allow_rules`、`deny_rules`、`ask_rules`
2. **工具层**：`PermissionEnforcer` 在 `execute_tool_with_enforcer` 中对特定工具（如 bash、write_file）做二次边界检查

[`PermissionMode`](/rust/crates/runtime/src/permissions.rs#L9-L15) 定义了五级模式：

```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,
    Allow,
}
```

`Prompt` 模式下，任何需要 escalated 权限的操作都会调用 `PermissionPrompter::decide()`，由 `CliPermissionPrompter` 在终端弹出 `Allow? [y/N]` 的交互确认。

#### 审视角：crate 边界是否过度隔离？

将 TUI、Runtime、Tools、API 拆分为独立 crate 确实降低了耦合，但也引入了跨 crate 重构的成本。例如，修改 `ContentBlock` 的字段需要同时更新 `api`、`runtime`、`tools` 和 `rusty-claude-cli` 四个 crate 的编译单元，任何类型不匹配都会在编译期暴露，这也意味着小范围的原型验证需要更长的编译反馈循环。对于个人贡献者或小团队，这种物理隔离的净收益可能在项目早期为负。

### 上下文即记忆 (Context as Memory)

System Prompt 由 [`load_system_prompt`](/rust/crates/runtime/src/prompt.rs#L432-L445) 动态组装，包含：

- 固定系统指令（`get_simple_system_section`）
- 环境上下文（OS、工作目录、日期）
- 项目上下文：向上遍历目录树收集 `CLAUDE.md`、`.claw/CLAUDE.md`、`.claw/instructions.md`
- Git 上下文：分支、最近 commit、暂存文件（[`GitContext::detect`](/rust/crates/runtime/src/git_context.rs#L26-L42)）
- 运行时配置：`.claw.json` / `.claw/settings.json` 的加载记录

会话记忆由 `Session` 持久化到 `项目目录 `.claw/sessions/<workspace_hash>/*.jsonl``，支持 `--resume` 恢复和 `fork` 分支。

---

## 入口与引导

| 入口 | 文件 | 说明 |
| --- | --- | --- |
| **CLI 启动** | [`rusty-claude-cli/src/main.rs#L1-L44`](/rust/crates/rusty-claude-cli/src/main.rs#L1-L44) | `fn main()`，全局导入，错误统一退出处理 |
| **参数解析** | [`rusty-claude-cli/src/main.rs#L371-L520`](/rust/crates/rusty-claude-cli/src/main.rs#L371-L520) | `parse_args()`，构建 `CliAction` |
| **REPL 循环** | [`rusty-claude-cli/src/main.rs#L2789-L2838`](/rust/crates/rusty-claude-cli/src/main.rs#L2789-L2838) | `run_repl()`，`LineEditor` 读取终端输入 |
| **单次 Prompt** | [`rusty-claude-cli/src/main.rs#L204-L240`](/rust/crates/rusty-claude-cli/src/main.rs#L204-L240) | `CliAction::Prompt` 分支，管道输入合并 |
| **运行时组装** | [`rusty-claude-cli/src/main.rs#L6235-L6310`](/rust/crates/rusty-claude-cli/src/main.rs#L6235-L6309) | `build_runtime()` → `ConversationRuntime` 组装 |

### 管道模式的可组合性

```bash
echo "总结这个文件" | claw "总结这个文件"
```

当 `permission_mode` 为 `DangerFullAccess` 时，[`read_piped_stdin`](/rust/crates/rusty-claude-cli/src/main.rs#L110-L128) 会将 stdin 内容附加到 prompt 之后。这让 `claw-code` 能无缝嵌入 CI/CD 脚本和自动化流程。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、参数解析、`CliAction`、REPL 循环 |
| [`/rust/crates/rusty-claude-cli/src/render.rs`](/rust/crates/rusty-claude-cli/src/render.rs) | `MarkdownStreamState`、ANSI 实时渲染、Spinner |
| [`/rust/crates/rusty-claude-cli/src/input.rs`](/rust/crates/rusty-claude-cli/src/input.rs) | `LineEditor`、终端输入读取与历史 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime`、`run_turn`、agentic loop |
| [`/rust/crates/runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) | `Session`、`ContentBlock`、消息持久化、fork |
| [`/rust/crates/runtime/src/compact.rs`](/rust/crates/runtime/src/compact.rs) | `compact_session`、上下文压缩与摘要 |
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode`、`PermissionPolicy`、规则匹配 |
| [`/rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs) | `BashCommandInput`、沙箱、超时、后台任务 |
| [`/rust/crates/runtime/src/file_ops.rs`](/rust/crates/runtime/src/file_ops.rs) | 文件读写编辑、工作区边界检查、glob/grep |
| [`/rust/crates/runtime/src/git_context.rs`](/rust/crates/runtime/src/git_context.rs) | `GitContext`、分支/commit/暂存文件检测 |
| [`/rust/crates/runtime/src/prompt.rs`](/rust/crates/runtime/src/prompt.rs) | `load_system_prompt`、`SystemPromptBuilder`、指令发现 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `execute_tool`、40+ 工具实现、`SubagentToolExecutor` |
| [`/rust/crates/api/src/client.rs`](/rust/crates/api/src/client.rs) | `ProviderClient`、模型路由与多后端分发 |
| [`/rust/crates/api/src/providers/mod.rs`](/rust/crates/api/src/providers/mod.rs) | `Provider` trait、`MODEL_REGISTRY`、别名解析 |
| [`/rust/crates/api/src/providers/anthropic.rs`](/rust/crates/api/src/providers/anthropic.rs) | `AnthropicClient`、原生 Messages API + Prompt Cache |
| [`/rust/crates/api/src/types.rs`](/rust/crates/api/src/types.rs) | `MessageRequest`、`StreamEvent`、`ContentBlockDelta` |
