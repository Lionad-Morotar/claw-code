# 什么是 Claude Code

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/introduction/what-is-claude-code) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 一句话定义

Claude Code 是一个**运行在本地终端中的 agentic coding system**。它不是给建议的聊天机器人——它直接在你的项目目录中读代码、改文件、跑命令、调试程序，拥有完整的 shell 能力。

### 从 TypeScript 到 Rust

`claw-code` 仓库是 Claude Code 的 Rust 重写实现。它将上游的 Terminal-native 架构以 Rust 的内存安全和零成本抽象重新落地，核心代码位于仓库的 [`/rust/`](/rust/) 目录下，采用 Cargo Workspace 组织，包含 `api`、`runtime`、`rusty-claude-cli`、`tools`、`commands`、`telemetry`、`plugins`、`compat-harness`、`mock-anthropic-service` 等 11 个 crate。

---

## 理解关键词

| 定位关键词 | 含义 |
| --- | --- |
| **Terminal-native** | 原生 CLI 应用，不是 IDE 插件、不是 Web 界面、不是 API wrapper |
| **Agentic** | AI 自主决策工具调用链，不是"一问一答"的聊天模式 |
| **Coding system** | 面向软件工程全流程，不是通用问答工具 |

### Terminal-native 的源码含义

在 Rust 实现中，"Terminal-native" 意味着进程直接从 `main()` 启动，通过标准 I/O 与终端交互。参见 [`rusty-claude-cli/src/main.rs#L110-L123`](/rust/crates/rusty-claude-cli/src/main.rs#L110-L123)：

```rust
fn main() {
    if let Err(error) = run() {
        let message = error.to_string();
        // ... 错误输出到 stderr
        std::process::exit(1);
    }
}
```

没有 Electron 外壳、没有浏览器沙箱、没有 IDE 插件宿主。CLI 进程直接 `fork`/`exec` 子进程调用 `git`、`cargo`、`npm` 等工具，这是 Terminal-native 最本质的架构特征。

### Agentic = 自主工具调用链

Agentic 的核心不是"一问一答"，而是一个**带状态的工具执行循环**。在 `claw-code` 中，这个循环被封装在 [`runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) 的 `ConversationRuntime` 里。`run_turn` 方法（[`L296-L487`](/rust/crates/runtime/src/conversation.rs#L296-L487)）的伪代码逻辑如下：

1. 接收用户输入 → 写入 Session
2. 组装 `ApiRequest`（system prompt + 当前消息历史）
3. 调用 `api_client.stream(request)` 获取流式事件
4. 解析模型返回的 `ContentBlock`，检测是否包含 `ToolUse`
5. 若有 `ToolUse`，则经权限检查后调用 `tool_executor.execute(tool_name, input)`
6. 将工具结果以 `ToolResult` 形式回写到 Session
7. 回到步骤 2，再次请求模型，直到模型不再请求工具

这就是 "agentic loop" 在代码里的精确表达——不是比喻，而是 [`conversation.rs`](/rust/crates/runtime/src/conversation.rs) 中一个真实的 `loop { ... }`。

### Coding system 的边界

Claude Code 不是通用问答工具，它的全部设计都围绕"代码"展开。比如 Session 里存储的消息不是纯字符串，而是带有 `ContentBlock` 枚举的结构化消息：

```rust
pub enum ContentBlock {
    Text { text: String },
    ToolUse { id: String, name: String, input: String },
    ToolResult { tool_use_id: String, tool_name: String, output: String, is_error: bool },
}
```

参见 [`runtime/src/session.rs#L28-L44`](/rust/crates/runtime/src/session.rs#L28-L44)。注意这里 `ToolResult` 直接关联 `tool_use_id`，这意味着工具调用和工具结果形成严格的**请求-响应配对**，便于在多轮循环中追踪调用链。这种数据结构天然就是为 Coding system 设计的。

---

## 与同类工具的架构差异

| 工具 | 架构模式 | 运行位置 | 工具执行 |
| --- | --- | --- | --- |
| **Claude Code** | Terminal-native agentic loop | 本地进程 | 直接 shell 执行 |
| Cursor / Copilot | IDE-integrated autocomplete + chat | IDE 进程内 | LSP / IDE API |
| Aider | CLI chat → git patch | 本地进程 | 文件操作为主 |
| ChatGPT / Claude.ai | Cloud chat + artifacts | 浏览器/云端 | 沙箱容器 |

核心差异：Claude Code 拥有**完整的 shell 访问权**——这意味着它可以做任何你在终端里能做的事，但也需要对应的安全机制来约束这个能力。

### Rust 实现中的安全边界设计

由于拥有完整 shell 能力，`claw-code` 在 `runtime` crate 中设计了一个五级权限模型 `PermissionMode`（[`runtime/src/permissions.rs#L8-L16`](/rust/crates/runtime/src/permissions.rs#L8-L16)）：

```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,
    Allow,
}
```

从最严格的 `ReadOnly` 到完全放行的 `Allow`，每一级都对应不同的策略。工具执行前，`ConversationRuntime` 会调用 `permission_policy.authorize_with_context(...)` 进行判定。如果用户配置了 `Prompt` 模式，危险操作会主动弹窗请求确认——这是"完整 shell 能力"和"安全可控"之间的关键平衡。

---

## 端到端示例：从输入到输出

当你在终端中输入 `bun run dev 有个 TypeScript 报错，帮我修一下` 时，系统发生了什么？

以下是将文档中的 TypeScript 上游架构映射到 `claw-code` Rust 源码后的完整链路：

```
┌─────────────────────────────────────────────────────────┐
│ 1. 入口层 (rusty-claude-cli/src/main.rs)                 │
│    parse_args() → CliAction::Prompt / CliAction::Repl   │
├─────────────────────────────────────────────────────────┤
│ 2. 交互层 (REPL / 管道模式)                              │
│    Prompt 输入捕获 → 合并 stdin 管道内容                 │
├─────────────────────────────────────────────────────────┤
│ 3. 编排层 (runtime/src/conversation.rs)                 │
│    ConversationRuntime 管理 turn 生命周期、token 预算    │
│    自动触发 compaction（上下文压缩）                     │
├─────────────────────────────────────────────────────────┤
│ 4. 核心循环 (run_turn — Agentic Loop)                   │
│    组装 ApiRequest → api_client.stream() → 解析事件     │
│    → 权限检查 → tool_executor.execute() → 结果回传 → 循环│
├─────────────────────────────────────────────────────────┤
│ 5. 工具执行 (runtime/src/bash.rs / file_ops.rs / ...)   │
│    Bash、FileOps、Git、Grep、Glob、MCP ...              │
├─────────────────────────────────────────────────────────┤
│ 6. 通信层 (api/src/client.rs / providers/)              │
│    流式 HTTP + SSE，支持 Anthropic / OpenAI 兼容端点    │
└─────────────────────────────────────────────────────────┘
```

### 入口层详解

用户在终端键入命令后，真正的入口是 [`rusty-claude-cli/src/main.rs#L165-L257`](/rust/crates/rusty-claude-cli/src/main.rs#L165-L257) 的 `run()` 函数。它会把命令行参数解析为 `CliAction` 枚举：

```rust
enum CliAction {
    Prompt { prompt: String, model: String, ... },
    Repl { model: String, ... },
    // ... Agents, Skills, Mcp, Doctor, Export 等
}
```

完整定义见 [`main.rs#L259-L347`](/rust/crates/rusty-claude-cli/src/main.rs#L259-L347)。`Prompt` 对应单次非交互调用，`Repl` 对应持续交互的终端会话。

### 管道模式的可组合性

如果你执行的是 `echo "总结这个文件" | claw "总结这个文件"`，`main.rs` 会调用 [`read_piped_stdin`](/rust/crates/rusty-claude-cli/src/main.rs#L131-L143) 检测 stdin 是否为终端；若不是，则通过 [`merge_prompt_with_stdin`](/rust/crates/rusty-claude-cli/src/main.rs#L151-L163) 将管道内容与命令行 prompt 合并。这让 `claw-code` 能无缝嵌入 CI/CD 脚本和自动化流程。

### 编排层：ConversationRuntime

进入 `runtime` crate 后，[`ConversationRuntime`](/rust/crates/runtime/src/conversation.rs#L126-L138) 是 orchestration 的核心。它持有：

- `session: Session` —— 对话状态（包括消息历史、token 用量、compaction 信息）
- `api_client: C` —— 满足 `ApiClient` trait 的流式模型客户端
- `tool_executor: T` —— 满足 `ToolExecutor` trait 的工具调度器
- `permission_policy: PermissionPolicy` —— 权限策略
- `usage_tracker: UsageTracker` —— token 预算追踪

当 token 用量超过阈值（默认 100k input tokens）时，`run_turn` 会自动触发 `compact_session`，将历史消息压缩成摘要，从而控制上下文长度和 API 成本。这个设计直接对应文档中提到的 "token 预算 / compaction 触发"。

### 通信层：ProviderClient 多后端支持

`api` crate 中的 [`ProviderClient`](/rust/crates/api/src/client.rs#L17-L19) 根据模型名称自动选择后端：

```rust
pub fn from_model(model: &str) -> Result<Self, ApiError> {
    Self::from_model_with_anthropic_auth(model, None)
}
```

实际分发逻辑（[`client.rs#L23-L56`](/rust/crates/api/src/client.rs#L23-L56) 附近）会根据 `detect_provider_kind()` 决定使用 `AnthropicClient`（原生 Messages API）还是 `OpenAiCompatClient`（兼容 OpenAI 格式，覆盖 xAI、DashScope 等）。两种后端都实现了 [`Provider`](/rust/crates/api/src/providers/mod.rs#L17-L31) trait：

```rust
pub trait Provider {
    type Stream;
    fn send_message<'a>(&'a self, request: &'a MessageRequest) -> ProviderFuture<'a, MessageResponse>;
    fn stream_message<'a>(...);
}
```

这意味着新增模型提供商时，只需在 `api` crate 增加一个 `Provider` 实现，上层 `ConversationRuntime` 无需任何改动。

---

## 端到端示例：修复 TypeScript 报错

具体到这个报错修复场景，一次典型的 agentic loop 可能包含多轮工具调用：

| Turn | AI 决策 | 工具调用 | 结果 |
| --- | --- | --- | --- |
| 1 | 先看报错信息 | `Bash("bun run dev 2>&1 \| head -30")` | TypeScript 错误输出 |
| 2 | 定位到文件 | `Read("src/utils/foo.ts")` | 源代码内容 |
| 3 | 搜索相关类型定义 | `Grep("interface Foo", "src/")` | 类型定义位置 |
| 4 | 修复代码 | `FileEdit(old, new)` | 代码已修改 |
| 5 | 验证修复 | `Bash("bun run dev 2>&1 \| head -10")` | 编译通过 |

### 这些工具在 Rust 源码中如何实现？

#### Bash 工具

Bash 不是简单的 `std::process::Command` 透传。它的输入输出 schema 定义在 [`runtime/src/bash.rs#L21-L52`](/rust/crates/runtime/src/bash.rs#L21-L52)：

```rust
pub struct BashCommandInput {
    pub command: String,
    pub timeout: Option<u64>,
    pub description: Option<String>,
    pub dangerously_disable_sandbox: Option<bool>,
    pub isolate_network: Option<bool>,
    pub filesystem_mode: Option<FilesystemIsolationMode>,
    pub allowed_mounts: Option<Vec<String>>,
}
```

注意这里有 `dangerously_disable_sandbox` 和 `isolate_network` 字段——模型可以显式请求关闭沙箱或隔离网络。默认情况下，命令会在受限环境中执行。输出超出 16,384 字节阈值时会被截断并在末尾追加 `[output truncated — exceeded 16384 bytes]` 标记。

#### FileOps 工具

FileOps 实现了 `read_file`、`write_file`、`edit_file`、`glob_search`、`grep_search`。所有路径都会先执行**工作区边界检查**（[`runtime/src/file_ops.rs#L32-L41`](/rust/crates/runtime/src/file_ops.rs#L32-L41)）：

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

这防止了 `../etc/passwd` 或符号链接逃逸等路径遍历攻击，是本地进程拥有完整文件系统访问权时的必要防线。

每一步都是 AI 自主决策的——它决定用哪个工具、传什么参数、何时停止。这就是 "agentic" 的含义。

---

## 它不是什么

* **不是 IDE 插件**：没有图形界面，不依赖 VS Code 或任何 IDE
  * 在 Rust 实现中，这意味着 UI 完全由 TUI（Terminal UI）渲染。CLI 通过 `crossterm` 控制光标、颜色和清屏，通过 `pulldown-cmark` + `syntect` 将模型返回的 Markdown 实时渲染为 ANSI 彩色文本。参见 [`rusty-claude-cli/src/render.rs#L602-L607`](/rust/crates/rusty-claude-cli/src/render.rs#L602-L607) 的 `MarkdownStreamState::push`。
* **不是 API wrapper**：它有自己的工具系统、权限模型、上下文工程、会话管理
  * 这些能力全部内建在 `runtime` crate 中，而不是简单转发 OpenAI/Anthropic SDK 的调用。
* **不是聊天机器人**：输出不是纯文本，而是实际的文件修改、命令执行
  * Session 中消息的核心数据结构是 `ContentBlock::ToolUse` 和 `ContentBlock::ToolResult`，而不是单一的 `String`。见 [`runtime/src/session.rs#L28-L44`](/rust/crates/runtime/src/session.rs#L28-L44)。
* **不是无脑执行器**：每个敏感操作都有权限检查和用户确认环节
  * 在 `run_turn` 的工具执行路径中，调用 `tool_executor.execute()` 之前必须先经过 `permission_policy.authorize_with_context()`。此外，还有 `pre_tool_use` hook 可以在工具执行前插入额外策略检查。

---

## 启动入口解剖

在 TypeScript 上游中，真正的代码入口是 `src/entrypoints/cli.tsx`，它做了三件事：注入 `feature()` polyfill、注入构建时 `MACRO`、声明 `BUILD_TARGET`。在 `claw-code` 的 Rust 实现中，入口更加直接——没有复杂的构建时宏系统，一切在 `main.rs` 中显式表达。

### 参数解析与 CliAction 分发

[`rusty-claude-cli/src/main.rs#L165-L257`](/rust/crates/rusty-claude-cli/src/main.rs#L165-L257) 附近的 `run()` 函数首先解析命令行参数，将其映射为 `CliAction` 枚举，随后按不同的 action 进行分发。对于 `Prompt` 和 `Repl` 这两种交互模式，流程如下：

1. `run()` 解析参数并匹配到 `CliAction::Prompt` 或 `CliAction::Repl`
2. 调用 `LiveCli::new()` 创建交互式 CLI 实例
3. `LiveCli::new()` 内部调用 `build_runtime()`，后者通过 `build_runtime_plugin_state()` 初始化 `ConfigLoader`、读取配置、组装 `PluginManager` 和 `ToolRegistry`
4. `build_runtime()` 还负责构造 `AnthropicRuntimeClient`（封装 `ProviderClient` 的 API 客户端）并将其注入 `ConversationRuntime`
5. 若是 `Repl`，进入 REPL 输入循环；若是 `Prompt`，直接调用 `LiveCli::run_turn_with_output()`
6. 在 `CliAction::Prompt` 分支内，只有当 `permission_mode` 为 `DangerFullAccess` 时才会通过 `read_piped_stdin` 合并管道输入（否则 stdin 需保留给交互式权限确认）

### Prompt 组装

在启动 `ConversationRuntime` 之前，系统会调用 [`runtime/src/prompt.rs#L430-L444`](/rust/crates/runtime/src/prompt.rs#L430-L444) 的 `load_system_prompt`：

```rust
pub fn load_system_prompt(cwd, current_date, os_name, os_version) -> Result<Vec<String>, ...> {
    let project_context = ProjectContext::discover_with_git(&cwd, ...)?;
    let config = ConfigLoader::default_for(&cwd).load()?;
    Ok(SystemPromptBuilder::new()
        .with_os(os_name, os_version)
        .with_project_context(project_context)
        .with_runtime_config(config)
        .build())
}
```

`SystemPromptBuilder` 会自动收集：

- 当前目录向上遍历得到的 `CLAUDE.md`、`.claw/instructions.md` 等指令文件（[`prompt.rs#L202-L220`](/rust/crates/runtime/src/prompt.rs#L202-L220)）
- `git status` 快照和最近 commit 摘要
- 本地配置（`.claw.json` 或 `.claw/settings.json`）

这意味着模型在用户刚输入第一句话时，就已经知道：项目约定、git 状态、文件系统布局。这也是"项目原生"在 Prompt 工程层面的体现。

---

## 为什么选择终端

终端不是限制，而是选择。它带来了独特的能力：

* **完整的 shell 访问**：可以运行任何命令行工具，无需为每个能力写插件
  * 在 Rust 源码中，`bash.rs` 直接调用 `tokio::process::Command`，配合 `tokio::time::timeout` 实现异步命令执行和超时控制。
* **项目原生**：直接在项目目录工作，理解文件系统结构、git 状态
  * `prompt.rs` 中的 `discover_instruction_files` 会向上遍历目录树收集约定文件；`read_git_status` 在初始化 `ProjectContext` 时拉取当前 git 快照。
* **可组合性**：管道模式（`echo "..." | claw -p`）允许嵌入 CI/CD 和自动化流程
  * 这在 [`main.rs#L131-L163`](/rust/crates/rusty-claude-cli/src/main.rs#L131-L163) 的 `read_piped_stdin` 和 `merge_prompt_with_stdin` 中显式实现。
* **低延迟**：没有 Electron 开销，TUI 响应极快
  * 渲染层基于 `crossterm` + `pulldown-cmark` + `syntect`，而非 Chromium。流式响应通过 `MarkdownStreamState::push` 增量渲染，每个 token 到达后立即刷新终端，避免整屏重绘。

代价是用户需要适应命令行界面——但也正因如此，它吸引的是需要**真正掌控开发环境**的开发者。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、`CliAction`、参数解析、管道输入处理 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime`、`run_turn`、agentic loop |
| [`/rust/crates/runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) | `Session`、`ContentBlock`、消息持久化 |
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode`、`PermissionPolicy` 五级权限模型 |
| [`/rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs) | Bash 工具输入输出 schema、沙箱与超时 |
| [`/rust/crates/runtime/src/file_ops.rs`](/rust/crates/runtime/src/file_ops.rs) | 文件读写、边界检查、符号链接逃逸检测 |
| [`/rust/crates/api/src/client.rs`](/rust/crates/api/src/client.rs) | `ProviderClient`、多模型 provider 分发 |
| [`/rust/crates/api/src/providers/mod.rs`](/rust/crates/api/src/providers/mod.rs) | `Provider` trait 抽象 |
| [`/rust/crates/rusty-claude-cli/src/render.rs`](/rust/crates/rusty-claude-cli/src/render.rs) | `MarkdownStreamState`、ANSI 实时渲染 |
| [`/rust/crates/runtime/src/prompt.rs`](/rust/crates/runtime/src/prompt.rs) | `load_system_prompt`、指令文件发现、git 上下文注入 |
