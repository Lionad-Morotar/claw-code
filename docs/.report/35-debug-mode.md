# Debug 模式

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/features/debug-mode) 的原始主题，映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 概述

上游文档主要描述如何使用 Bun 的 `--inspect-wait` 对 TypeScript REPL 进行 VS Code attach 调试。但在 `claw-code` 的 Rust 实现中，调试能力以另一种形态存在：

1. **没有 `debug.rs` 文件** — Rust 代码库中没有独立的 debug 模块，调试能力分散在 CLI slash commands、telemetry 子系统和错误渲染路径中。
2. **没有传统 logging 框架** — 未使用 `tracing`、`env_logger` 或 `log` crate，而是自建了 `telemetry` crate 做结构化事件追踪。
3. **CLI 侧有一个内建的 debug 工具**：`/debug-tool-call` slash command，用于在 REPL 中回放最后一次工具调用的详细输入输出。

---

## 调试能力一：`/debug-tool-call`（REPL 级工具调试）

在 REPL 交互模式下，用户可以输入 `/debug-tool-call` 来查看最近一次 AI 工具调用的完整细节。这在排查"模型调了什么工具、传了什么参数、得到什么结果"时非常有用。

### 源码位置

 SlashCommand 定义：[`commands/src/lib.rs#L1075`](/rust/crates/commands/src/lib.rs#L1075)

```rust
pub enum SlashCommand {
    // ...
    DebugToolCall,
    // ...
}
```

解析逻辑：[`commands/src/lib.rs#L1274-L1277`](/rust/crates/commands/src/lib.rs#L1274-L1277)

```rust
"debug-tool-call" => {
    validate_no_args(command, &args)?;
    SlashCommand::DebugToolCall
}
```

说明该命令**不接受任何参数**，格式严格为 `/debug-tool-call`。

### CLI 执行入口

在 `rusty-claude-cli` 的 REPL 循环中，当检测到 `SlashCommand::DebugToolCall` 时，调用 [`main.rs#L3589`](/rust/crates/rusty-claude-cli/src/main.rs#L3589)：`self.run_debug_tool_call(None)?`。

具体实现位于 [`main.rs#L4330-L4334`](/rust/crates/rusty-claude-cli/src/main.rs#L4330-L4334)：

```rust
fn run_debug_tool_call(&self, args: Option<&str>) -> Result<(), Box<dyn std::error::Error>> {
    validate_no_args("/debug-tool-call", args)?;
    println!("{}", render_last_tool_debug_report(self.runtime.session())?);
    Ok(())
}
```

### 报告渲染逻辑

核心函数 `render_last_tool_debug_report` 位于 [`main.rs#L5219-L5262`](/rust/crates/rusty-claude-cli/src/main.rs#L5219-L5262)：

```rust
fn render_last_tool_debug_report(session: &Session) -> Result<String, Box<dyn std::error::Error>> {
    let last_tool_use = session
        .messages
        .iter()
        .rev()
        .find_map(|message| {
            message.blocks.iter().rev().find_map(|block| match block {
                ContentBlock::ToolUse { id, name, input } => {
                    Some((id.clone(), name.clone(), input.clone()))
                }
                _ => None,
            })
        })
        .ok_or_else(|| "no prior tool call found in session".to_string())?;
    // ... 再查找对应的 ToolResult，组装成可读的调试报告
}
```

它从 Session 的消息链末尾反向扫描：
1. 找到最后一个 `ContentBlock::ToolUse`，提取 `id`、`name`、`input`。
2. 再找到匹配的 `ContentBlock::ToolResult`（通过 `tool_use_id` 关联）。
3. 将两者格式化为对齐文本块输出，包含：
   - `Tool id`
   - `Tool name`
   - `Input`（完整 JSON 参数）
   - `Result`（工具执行结果，或 `missing tool result` / `error` 标记）

这是 claw-code 中最接近"Debug 模式"的用户可见功能——**无需 attach 调试器，直接在终端内检查 agent 的工具调用链路**。

---

## 调试能力二：Telemetry（结构化运行时追踪）

`claw-code` 没有采用 Rust 生态常见的 `tracing` + `RUST_LOG` 日志体系，而是实现了一个精简的 `telemetry` crate，位于 [`/rust/crates/telemetry/src/lib.rs`](/rust/crates/telemetry/src/lib.rs)。

### 设计哲学

- **结构化**：所有事件都是 JSON 可序列化的 `TelemetryEvent` 枚举。
- **可持久化**：通过 `JsonlTelemetrySink` 将事件追加到 `.jsonl` 文件。
- **无侵入**：`ConversationRuntime` 仅在持有 `SessionTracer` 时才会记录事件；不持有则全部跳过。

### 核心类型

`TelemetryEvent` 枚举：[`telemetry/src/lib.rs#L170-L203`](/rust/crates/telemetry/src/lib.rs#L170-L203)

```rust
pub enum TelemetryEvent {
    HttpRequestStarted { session_id, attempt, method, path, attributes },
    HttpRequestSucceeded { session_id, attempt, method, path, status, request_id, attributes },
    HttpRequestFailed { session_id, attempt, method, path, error, retryable, attributes },
    Analytics(AnalyticsEvent),
    SessionTrace(SessionTraceRecord),
}
```

`SessionTracer`：[`telemetry/src/lib.rs#L279-L406`](/rust/crates/telemetry/src/lib.rs#L279-L406)

```rust
pub struct SessionTracer {
    session_id: String,
    sequence: Arc<AtomicU64>,
    sink: Arc<dyn TelemetrySink>,
}
```

它通过自增 `sequence` 保证事件顺序，支持 `record_analytics`、`record_http_request_*` 等高阶 API。

### Sink 实现

- `MemoryTelemetrySink`：测试用，内存中保存事件列表。
- `JsonlTelemetrySink`：生产用，每条事件一行 JSON，append 到文件。参见 [`telemetry/src/lib.rs#L233-L277`](/rust/crates/telemetry/src/lib.rs#L233-L277)。

### 生命周期挂载点

`ConversationRuntime` 在核心链路中埋了 6 个追踪点：

| 方法 | 位置 | 记录时机 |
|------|------|----------|
| `record_turn_started` | `conversation.rs#L547-L556` | 用户输入被接收 |
| `record_assistant_iteration` | `conversation.rs#L565-L579` | 每次 assistant 响应解析完成 |
| `record_tool_started` | `conversation.rs#L583-L593` | 工具执行前 |
| `record_tool_finished` | `conversation.rs#L597-L614` | 工具执行后 |
| `record_turn_completed` | `conversation.rs#L618-L639` | 整轮对话正常结束 |
| `record_turn_failed` | `conversation.rs#L643-L650` | 整轮对话因错误中断 |

例如 `record_tool_started` 的源码（[`conversation.rs#L583-L593`](/rust/crates/runtime/src/conversation.rs#L583-L593)）：

```rust
fn record_tool_started(&self, iteration: usize, tool_name: &str) {
    let Some(session_tracer) = &self.session_tracer else { return; };
    let mut attributes = Map::new();
    attributes.insert("iteration".to_string(), Value::from(iteration as u64));
    attributes.insert("tool_name".to_string(), Value::String(tool_name.to_string()));
    session_tracer.record("tool_execution_started", attributes);
}
```

这种设计让"verbose 输出"不再是往 stderr 打印文本，而是产出结构化事件流，便于后续分析和回放。

---

## 调试能力三：API Client 的 `request_id` 链路追踪

当 API 请求失败时，`api` crate 会将 `request_id` 携带到错误信息中。这个 `request_id` 是 provider 侧返回的 trace ID，对跨团队排查服务端问题至关重要。

### 错误渲染中的 Trace 字段

在 [`main.rs#L6732-L6735`](/rust/crates/rusty-claude-cli/src/main.rs#L6732-L6735) 附近：

```rust
if let Some(request_id) = error.request_id() {
    lines.push(format!("  Trace            {request_id}"));
}
```

CLI 的错误摘要会显式输出 `Trace <request_id>`，用户可以直接用这个 ID 向 Anthropic / OpenAI 支持团队定位请求。

### 错误类型中的 `request_id`

`ApiError` 结构：[`api/src/error.rs`](/rust/crates/api/src/error.rs) 中定义了多种错误变体，其中网络/HTTP 错误会带上可选的 `request_id`。这一设计保证即使模型返回异常，也能保留完整的服务端追踪上下文。

---

## 为什么没有 Bun `--inspect-wait`？

上游 TypeScript 实现依赖 Bun 运行时的 Chrome DevTools Protocol（CDP）inspect 服务，原因有二：

1. **上游是 TypeScript + Bun**，Bun 原生支持 `--inspect-wait`。
2. **TUI REPL 需要真实终端**，VS Code 直接 launch 无法提供交互式 tty，所以采用 attach 模式。

`claw-code` 作为纯 Rust 项目，可直接用 `rust-analyzer` + VS Code CodeLLDB / `launch.json` 启动调试，无需额外的 inspect 服务。Cargo 产物是普通 ELF/Mach-O 二进制，调试体验与常规 Rust CLI 完全一致。这意味着 Rust 实现**没有也不需要一个等价的 `bun --inspect-wait` 机制**。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `SlashCommand::DebugToolCall` 定义与解析 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | `run_debug_tool_call`、`render_last_tool_debug_report` |
| [`/rust/crates/telemetry/src/lib.rs`](/rust/crates/telemetry/src/lib.rs) | `TelemetryEvent`、`SessionTracer`、`JsonlTelemetrySink` |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime` 生命周期追踪点（turn/tool 记录） |
| [`/rust/crates/api/src/error.rs`](/rust/crates/api/src/error.rs) | `ApiError` 及 `request_id` 携带逻辑 |
