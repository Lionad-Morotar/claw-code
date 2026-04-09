# 流式响应机制 - Claude Code 打字机效果原理

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/conversation/streaming) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 为什么需要流式

想象 AI 需要 30 秒才能生成完整回答——如果等 30 秒后才一次性显示，用户体验是灾难性的。流式响应让用户实时看到 AI 的思考过程：

- 文字逐字出现，用户能提前判断方向是否正确
- 工具调用的参数在生成过程中就能预览
- 长时间任务不会让用户觉得"卡死了"

### Rust 实现中的流式入口

`claw-code` 的流式链路起点在 CLI 的 `AnthropicRuntimeClient::stream` 方法（[`main.rs#L6490-L6510`](/rust/crates/rusty-claude-cli/src/main.rs#L6490-L6510)）。它将 `ApiRequest` 转换为 `MessageRequest`，设置 `stream: true`，然后进入异步消费循环：

```rust
let message_request = MessageRequest {
    model: self.model.clone(),
    max_tokens: max_tokens_for_model(&self.model),
    messages: convert_messages(&request.messages),
    system: (...).then(|| request.system_prompt.join("\n\n")),
    tools: self.enable_tools.then(|| filter_tool_specs(...)),
    tool_choice: self.enable_tools.then_some(ToolChoice::Auto),
    stream: true,
    ..Default::default()
};
```

注意这里的 `emit_output` 标志：当 `claw` 以 `--output-format json` 运行时，`consume_stream` 会把 `out` 指向 `io::sink()`，即不渲染打字机效果，只收集事件。这让同一套流式基础设施同时支持交互式 TUI 和非交互式管道调用。

---

## BetaRawMessageStreamEvent 核心事件类型

在 TypeScript 上游中，流式 API 返回的是 `BetaRawMessageStreamEvent`。`claw-code` 的 `api` crate 把这些事件映射为 Rust 的 `StreamEvent` 枚举（[`api/src/types.rs#L259-L268`](/rust/crates/api/src/types.rs#L259-L268)）：

```rust
pub enum StreamEvent {
    MessageStart(MessageStartEvent),
    MessageDelta(MessageDeltaEvent),
    ContentBlockStart(ContentBlockStartEvent),
    ContentBlockDelta(ContentBlockDeltaEvent),
    ContentBlockStop(ContentBlockStopEvent),
    MessageStop(MessageStopEvent),
}
```

| 事件 | 对应上游 | 含义 |
| --- | --- | --- |
| `MessageStart` | `message_start` | 消息开始，包含 model、初始 usage |
| `ContentBlockStart` | `content_block_start` | 内容块开始（text / tool_use / thinking） |
| `ContentBlockDelta` | `content_block_delta` | 增量数据（text_delta / input_json_delta / thinking_delta） |
| `ContentBlockStop` | `content_block_stop` | 内容块结束 |
| `MessageDelta` | `message_delta` | stop_reason + 最终 usage |
| `MessageStop` | `message_stop` | 流结束标记 |

### 源码映射：SSE 解析到 StreamEvent

HTTP 层返回的是原始 SSE（Server-Sent Events）帧。Anthropic 后端使用通用的 `SseParser`（[`api/src/sse.rs#L4-L80`](/rust/crates/api/src/sse.rs#L4-L80)），逐块读取 `\n\n` 或 `\r\n\r\n` 分隔的帧：

```rust
fn next_frame(&mut self) -> Option<String> {
    let separator = self.buffer.windows(2)
        .position(|window| window == b"\n\n")
        .map(|position| (position, 2))
        .or_else(|| {
            self.buffer.windows(4)
                .position(|window| window == b"\r\n\r\n")
                .map(|position| (position, 4))
        })?;
    // ... drain frame, parse JSON into StreamEvent
}
```

OpenAI 兼容后端（xAI、DashScope、OpenAI 本身）则有自己专用的 `OpenAiSseParser`（[`api/src/providers/openai_compat.rs#L344-L380`](/rust/crates/api/src/providers/openai_compat.rs#L344-L380)），负责把 OpenAI 格式的 `chat.completion.chunk` 事件转译为与 Anthropic 统一的 `StreamEvent`。这是多 Provider 适配的关键：上层代码只认识 `StreamEvent`，不认识底层协议差异。

---

## 事件处理状态机

CLI 中的 `consume_stream`（[`main.rs#L6531-L6695`](/rust/crates/rusty-claude-cli/src/main.rs#L6531-L6695)）实现了一个基于 `match event` 的状态机：

```rust
loop {
    let next = stream.next_event().await?;
    let Some(event) = next else { break; };
    match event {
        ApiStreamEvent::MessageStart(start) => { ... }
        ApiStreamEvent::ContentBlockStart(start) => { ... }
        ApiStreamEvent::ContentBlockDelta(delta) => match delta.delta { ... }
        ApiStreamEvent::ContentBlockStop(_) => { ... }
        ApiStreamEvent::MessageDelta(delta) => { ... }
        ApiStreamEvent::MessageStop(_) => { ... }
    }
}
```

| 事件类型 | 处理逻辑 | 状态变更 |
| --- | --- | --- |
| `MessageStart` | 遍历 `start.message.content`，逐块调用 `push_output_block` | 初始化 pending tool、thinking summary 标志 |
| `ContentBlockStart` | 按类型创建内容块（text / tool_use / thinking） | `pending_tool` 或 `block_has_thinking_summary` 更新 |
| `ContentBlockDelta` | 按子类型追加数据：text 增量渲染、input_json 拼接到 pending tool | 输出到 stdout / 事件收集 |
| `ContentBlockStop` | `markdown_stream.flush()`，若存在 pending tool 则输出工具调用摘要 | `pending_tool` 转为 `events.push(ToolUse{...})` |
| `MessageDelta` | 记录 usage | `events.push(Usage(...))` |
| `MessageStop` | `markdown_stream.flush()`，设置 `saw_stop = true` | `events.push(MessageStop)`，循环即将结束 |

### Runtime 层的事件收敛

`api` crate 产生的 `StreamEvent` 流，在 `rusty-claude-cli` 的 `consume_stream` 中被消费并渲染；而在更高层的 `ConversationRuntime::run_turn`（[`runtime/src/conversation.rs#L296`](/rust/crates/runtime/src/conversation.rs#L296)）中，并不需要感知逐 token 的 delta——它只关心收敛后的 `AssistantEvent` 序列。

这个收敛由 `AnthropicRuntimeClient::stream`（实现 `ApiClient` trait，[`main.rs#L6490-L6510`](/rust/crates/rusty-claude-cli/src/main.rs#L6490-L6510)）完成：先把 `StreamEvent` 消费成 `Vec<AssistantEvent>`，再把整个向量返回给 `ConversationRuntime`。`AssistantEvent` 枚举更精简（[`runtime/src/conversation.rs#L30-L48`](/rust/crates/runtime/src/conversation.rs#L30-L48)）：

```rust
pub enum AssistantEvent {
    TextDelta(String),
    ToolUse { id: String, name: String, input: String },
    Usage(TokenUsage),
    PromptCache(PromptCacheEvent),
    MessageStop,
}
```

`ConversationRuntime` 内部的 `build_assistant_message`（[`runtime/src/conversation.rs#L672-L771`](/rust/crates/runtime/src/conversation.rs#L672-L771)）进一步把 `AssistantEvent` 向量收敛为单条 `ConversationMessage`：

```rust
fn build_assistant_message(events: Vec<AssistantEvent>) -> Result<...> {
    let mut text = String::new();
    let mut blocks = Vec::new();
    for event in events {
        match event {
            AssistantEvent::TextDelta(delta) => text.push_str(&delta),
            AssistantEvent::ToolUse { id, name, input } => {
                flush_text_block(&mut text, &mut blocks);
                blocks.push(ContentBlock::ToolUse { id, name, input });
            }
            // ...
        }
    }
    flush_text_block(&mut text, &mut blocks);
    // ...
}
```

这就是 `claw-code` 的"三层收敛"架构：
1. **HTTP/SSE 层**：`StreamEvent`（协议原生粒度）
2. **CLI 客户端层**：`AssistantEvent`（渲染 + 收集粒度）
3. **Runtime 层**：`ConversationMessage` + `ContentBlock`（业务逻辑粒度）

---

## 内容块类型及其增量数据

`ContentBlockStartEvent` 中的 `content_block` 类型决定了后续 delta 的处理方式：

```rust
pub enum OutputContentBlock {
    Text { text: String },
    ToolUse { id: String, name: String, input: Value },
    Thinking { thinking: String, signature: Option<String> },
    RedactedThinking,
}
```

定义位于 [`api/src/types.rs#L147-L158`](/rust/crates/api/src/types.rs#L147-L158)。对应的 `ContentBlockDelta`（[`api/src/types.rs#L241-L256`](/rust/crates/api/src/types.rs#L241-L256)）：

```rust
pub enum ContentBlockDelta {
    TextDelta { text: String },
    InputJsonDelta { partial_json: String },
    ThinkingDelta { thinking: String },
    SignatureDelta { signature: String },
}
```

| 内容块类型 | Delta 类型 | 累加逻辑 |
| --- | --- | --- |
| `Text` | `TextDelta` | `text.push_str(delta.text)` |
| `ToolUse` | `InputJsonDelta` | `input.push_str(delta.partial_json)`（JSON 字符串增量拼接） |
| `Thinking` | `ThinkingDelta` + `SignatureDelta` | `thinking += delta.thinking`，`signature = delta.signature` |

关键设计：[`push_output_block`](/rust/crates/rusty-claude-cli/src/main.rs#L7398-L7430) 在流式模式下会忽略 `content_block_start` 中 tool use 的空对象 `{}`，将其视为占位符，真正的输入在后续 `input_json_delta` 中拼接。这对应上游文档中提到的 "content_block_start 时所有文本字段初始化为空字符串，只通过 delta 累加"。

---

## 文本 chunk 和 tool_use block 的交织

一次 AI 响应在 `consume_stream` 的视角下可能长这样：

```
MessageStart
ContentBlockStart (Text)        → "我来帮你修复这个 bug。"
ContentBlockDelta (text_delta)  → "首先..."
ContentBlockStop
ContentBlockStart (ToolUse)     → { name: "Read", input: {} }
ContentBlockDelta (input_json)  → '{"file_path":' → '"src/foo.ts"}'
ContentBlockStop                → 输出 "📄 Reading src/foo.ts…"
ContentBlockStart (Text)        → "我已经看到了问题所在..."
ContentBlockStop
MessageDelta                    → stop_reason = end_turn
MessageStop
```

### 源码映射：工具调用的流式组装

在 [`consume_stream`](/rust/crates/rusty-claude-cli/src/main.rs#L6531-L6695) 中：

- `ContentBlockStart(ToolUse)` 触发 `push_output_block`，把 `(id, name, "")` 放入 `pending_tool`（[`main.rs#L7398-L7430`](/rust/crates/rusty-claude-cli/src/main.rs#L7398-L7430)）
- 后续 `ContentBlockDelta(InputJsonDelta)` 不断 `input.push_str(&partial_json)`（[`main.rs#L6640-L6643`](/rust/crates/rusty-claude-cli/src/main.rs#L6640-L6643)）
- `ContentBlockStop` 到来时，调用 `markdown_stream.flush()` 清空残留文本，然后输出格式化后的工具调用摘要（[`main.rs#L6658-L6667`](/rust/crates/rusty-claude-cli/src/main.rs#L6658-L6667)）：

```rust
writeln!(out, "\n{}", format_tool_call_start(&name, &input))
    .and_then(|()| out.flush())?;
events.push(AssistantEvent::ToolUse { id, name, input });
```

`format_tool_call_start`（[`main.rs#L6948-L6995`](/rust/crates/rusty-claude-cli/src/main.rs#L6948-L6995)）会为不同工具生成人类可读的 ANSI 标签，例如 `📄 Reading src/foo.ts…`、`✏️ Writing src/bar.ts (42 lines)`、`📝 Editing src/baz.ts` 等。

stop_reason 要到 `MessageDelta` 才最终确定（可能是 `end_turn`、`tool_use`、`max_tokens` 等），因此 `MessageDelta` 阶段会把 usage 追加到事件中（[`main.rs#L6669-L6671`](/rust/crates/rusty-claude-cli/src/main.rs#L6669-L6671)）。

---

## 流式中的错误处理

### 网络断开

流式连接依赖 HTTP chunked transfer + SSE。当连接中断时，`stream.next_event()` 会提前返回 `None` 或抛出 `ApiError`。`consume_stream` 在循环结束后会做一项关键守卫检查（[`main.rs#L6685-L6700`](/rust/crates/rusty-claude-cli/src/main.rs#L6685-L6700)）：

```rust
if !saw_stop && events.iter().any(|event| has_meaningful_content(event)) {
    events.push(AssistantEvent::MessageStop);
}

if events.iter().any(|event| matches!(event, AssistantEvent::MessageStop)) {
    return Ok(events);
}

// 若完全没有 MessageStop，降级为非流式请求
let response = self.client.send_message(&MessageRequest { stream: false, ... }).await?;
let mut events = response_to_events(response, out)?;
```

这就是说：
1. 如果流已经产生了有效内容但没有 `MessageStop`，系统会**自动补一个 `MessageStop`** 并返回事件。
2. 如果连有效内容也没有，则**降级为非流式请求**（`send_message`），一次性获取完整响应再返回。

### API 限流

`api` crate 的 `send_with_retry`（anthropic: [`api/src/providers/anthropic.rs#L401-L470`](/rust/crates/api/src/providers/anthropic.rs#L401-L470)，openai_compat: [`api/src/providers/openai_compat.rs#L195-L218`](/rust/crates/api/src/providers/openai_compat.rs#L195-L218)）实现了指数退避重试：

```rust
loop {
    attempts += 1;
    match self.send_raw_request(request).await {
        Ok(response) => match expect_success(response).await { ... }
        Err(error) if error.is_retryable() && attempts <= self.max_retries + 1 => {
            tokio::time::sleep(self.jittered_backoff_for_attempt(attempts)?).await;
        }
        Err(error) => return Err(error),
    }
}
```

- 默认最大重试次数：8 次
- 初始退避：1 秒
- 最大退避：128 秒
- 重试状态码：408, 409, 429, 500, 502, 503, 504（[`api/src/providers/anthropic.rs#L489-L495`](/rust/crates/api/src/providers/anthropic.rs#L489-L495)）

退避使用了基于系统时间纳秒 + 单调计数器的伪随机抖动（[`api/src/providers/openai_compat.rs#L257-L285`](/rust/crates/api/src/providers/openai_compat.rs#L257-L285)），以避免并发客户端的"惊群"同步重试。

### Token 超限

`api` crate 在发送请求前会运行 `preflight_message_request`（[`api/src/providers/mod.rs#L260-L280`](/rust/crates/api/src/providers/mod.rs#L260-L280)），做本地上下文窗口预检：

```rust
fn estimate_message_request_input_tokens(request: &MessageRequest) -> u32 {
    let mut estimate = estimate_serialized_tokens(&request.messages);
    estimate = estimate.saturating_add(estimate_serialized_tokens(&request.system));
    estimate = estimate.saturating_add(estimate_serialized_tokens(&request.tools));
    estimate = estimate.saturating_add(estimate_serialized_tokens(&request.tool_choice));
    estimate
}
```

如果 `estimated_total_tokens > context_window_tokens`，直接返回 `ApiError::ContextWindowExceeded`，不会浪费网络请求。Anthropic 后端还会在此之上做一次 `count_tokens` API 精修（[`api/src/providers/anthropic.rs#L471-L535`](/rust/crates/api/src/providers/anthropic.rs#L471-L535)）。

对于流式过程中出现的 `max_tokens` 超限，Rust 实现不会在上游 `message_delta` 中做特殊的现场恢复逻辑；`ConversationRuntime` 会把 `stop_reason` 原样交给调用者处理。这与 TypeScript 上游直接 `yield` 错误消息的策略不同——Rust 实现更倾向于"失败即返回"，把恢复决策交给外层 UI 或 CLI。

---

## 流式停滞检测

在工具执行完成后，模型重新生成响应的第一条事件可能会出现明显延迟（post-tool stall）。`claw-code` 对此有专门的监测：

```rust
const POST_TOOL_STALL_TIMEOUT: Duration = Duration::from_secs(10);
```

定义位于 [`main.rs#L81`](/rust/crates/rusty-claude-cli/src/main.rs#L81)。在 `consume_stream` 中：

```rust
let next = if apply_stall_timeout && !received_any_event {
    match tokio::time::timeout(POST_TOOL_STALL_TIMEOUT, stream.next_event()).await {
        Ok(inner) => inner?,
        Err(_elapsed) => {
            return Err(RuntimeError::new(
                "post-tool stall: model did not respond within timeout",
            ));
        }
    }
} else {
    stream.next_event().await?
};
```

而 `AnthropicRuntimeClient::stream`（[`main.rs#L6490-L6510`](/rust/crates/rusty-claude-cli/src/main.rs#L6490-L6510)）在发现 `post-tool stall` 错误时，会**自动重发请求一次**作为 continuation nudge：

```rust
for attempt in 1..=max_attempts {
    let result = self.consume_stream(&message_request, is_post_tool && attempt == 1).await;
    match result {
        Ok(events) => return Ok(events),
        Err(error) if error.to_string().contains("post-tool stall") && attempt < max_attempts => {
            continue;  // re-send the request
        }
        Err(error) => return Err(error),
    }
}
```

这比 TypeScript 上游的"stall计数 + 总stall时间统计"更精简：Rust 实现只做一件事——检测 10 秒无事件即报错，然后外层自动重试一次。不需要复杂的计数器，因为 tokio 的 `timeout` 足够精确。

---

## 工具执行的流式反馈

Bash 工具等长时间运行命令也有自己的"进度流"。但与 TypeScript 上游不同，`claw-code` 的 Bash 工具目前并不通过 `AsyncGenerator` 向 UI 推送逐行输出；它的进度反馈通过 `CliToolExecutor` 的 `emit_output` 控制，在命令完全结束后一次性返回截断后的输出（参见 [`runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs)）。

但这不意味着工具执行期间终端完全静止。`LiveCli::run_turn`（[`main.rs#L3461-L3488`](/rust/crates/rusty-claude-cli/src/main.rs#L3461-L3488)）在调用 `ConversationRuntime::run_turn` 之前会启动一个 `Spinner`：

```rust
let mut spinner = Spinner::new();
spinner.tick("🦀 Thinking...", TerminalRenderer::new().color_theme(), &mut stdout)?;
```

`Spinner::tick`（[`render.rs#L75-L90`](/rust/crates/rusty-claude-cli/src/render.rs#L75-L90)）用 `crossterm` 的 `SavePosition / Clear / RestorePosition` 在终端底部显示旋转的 Unicode 点阵。当模型开始输出文本或工具调用时，spinner 会被 `finish` 或自然冲刷掉，流式内容接管光标。

---

## 多 Provider 适配

`claw-code` 的 Provider 架构由 `api` crate 统一封装。`ProviderClient` 枚举（[`api/src/client.rs#L10-L14`](/rust/crates/api/src/client.rs#L10-L14)）只有三个变体：

```rust
pub enum ProviderClient {
    Anthropic(AnthropicClient),
    Xai(OpenAiCompatClient),
    OpenAi(OpenAiCompatClient),
}
```

- **Anthropic**：原生 Messages API + SSE（[`api/src/providers/anthropic.rs#L330-L350`](/rust/crates/api/src/providers/anthropic.rs#L330-L350)）
- **xAI / OpenAI / DashScope**：统一走 `OpenAiCompatClient`（[`api/src/providers/openai_compat.rs#L170-L190`](/rust/crates/api/src/providers/openai_compat.rs#L170-L190)）

### Provider 选择

`detect_provider_kind`（[`api/src/providers/mod.rs#L201-L222`](/rust/crates/api/src/providers/mod.rs#L201-L222)）的优先级：

1. 模型别名前缀路由：`claude-*` → Anthropic，`grok*` → xAI，`openai/` 或 `gpt-` → OpenAI，`qwen/` 或 `qwen-` → DashScope
2. 若前缀无法识别，按环境变量嗅探：`ANTHROPIC_API_KEY` → Anthropic，`OPENAI_API_KEY` → OpenAI，`XAI_API_KEY` → xAI
3. 默认 fallback 到 Anthropic

`main.rs` 的 `AnthropicRuntimeClient::new`（[`main.rs#L6395-L6448`](/rust/crates/rusty-claude-cli/src/main.rs#L6395-L6448)）在构建时就会完成 Provider 分派，并把结果固化到 `client` 字段。这意味着一次会话一旦启动，不会在中途切换 Provider。

### OpenAI 兼容层的协议归一化

`OpenAiCompatClient` 需要做两件事：

1. **请求翻译**：`build_chat_completion_request`（[`api/src/providers/openai_compat.rs#L747-L810`](/rust/crates/api/src/providers/openai_compat.rs#L747-L810)）把 `MessageRequest` 转成 OpenAI `/chat/completions` 格式，包括消息角色展平（[`translate_message`](/rust/crates/api/src/providers/openai_compat.rs#L821-L860)）、工具定义转换（[`openai_tool_definition`](/rust/crates/api/src/providers/openai_compat.rs#L915-L928)）、stream usage 选项等。
2. **响应归一化**：`normalize_response`（[`api/src/providers/openai_compat.rs#L943-L990`](/rust/crates/api/src/providers/openai_compat.rs#L943-L990)）把 ChatCompletionResponse 转回 `MessageResponse`；`StreamState::ingest_chunk`（[`api/src/providers/openai_compat.rs#L420-L490`](/rust/crates/api/src/providers/openai_compat.rs#L420-L490)）把 SSE chunk 转回 `StreamEvent`。

一个细节：`normalize_finish_reason`（[`api/src/providers/openai_compat.rs#L1112-L1124`](/rust/crates/api/src/providers/openai_compat.rs#L1112-L1124)）把 OpenAI 的 `"stop"` 映射为 `"end_turn"`，`"tool_calls"` 映射为 `"tool_use"`，从而让 `ConversationRuntime` 无需关心 Provider 差异。

---

## Markdown 流式增量渲染为 ANSI

文本 delta 最终要以"打字机效果"出现在终端上。这由 `render.rs` 中的 `MarkdownStreamState` 负责（[`render.rs#L601-L620`](/rust/crates/rusty-claude-cli/src/render.rs#L601-L620)）：

```rust
pub struct MarkdownStreamState {
    pending: String,
}

impl MarkdownStreamState {
    pub fn push(&mut self, renderer: &TerminalRenderer, delta: &str) -> Option<String> {
        self.pending.push_str(delta);
        let split = find_stream_safe_boundary(&self.pending)?;
        let ready = self.pending[..split].to_string();
        self.pending.drain(..split);
        Some(renderer.markdown_to_ansi(&ready))
    }

    pub fn flush(&mut self, renderer: &TerminalRenderer) -> Option<String> {
        if self.pending.trim().is_empty() { ... }
        Some(renderer.markdown_to_ansi(&std::mem::take(&mut self.pending)))
    }
}
```

`push` 每次收到新文本时执行"缓冲区追加 + 安全边界切割"：

```rust
fn find_stream_safe_boundary(markdown: &str) -> Option<usize> {
    let mut open_fence: Option<FenceMarker> = None;
    let mut last_boundary = None;
    for (offset, line) in markdown.split_inclusive('\n').scan(...) {
        // 追踪代码块 fence  opener/closer
        // 只有在空行处才标记为安全切割点
    }
    last_boundary
}
```

详见 [`render.rs#L810-L840`](/rust/crates/rusty-claude-cli/src/render.rs#L810-L840)。核心规则：

- 如果当前缓冲区处于未闭合的代码块（` ``` ` 或 `~~~`）内部，不能切割——否则会导致代码块高亮被截断。
- 只有在空行处才能切割，保证 `pulldown-cmark` 拿到完整块级结构（段落、列表、引用）后，渲染结果才稳定。

### TerminalRenderer 的 ANSI 输出

切割出的安全文本交给 `TerminalRenderer::markdown_to_ansi`（[`render.rs#L274-L277`](/rust/crates/rusty-claude-cli/src/render.rs#L274-L277)），底层使用 `pulldown-cmark` 解析 Markdown + `syntect` 对代码块进行语法高亮。`consume_stream` 把渲染好的 ANSI 字符串直接 `write!` 到 `stdout` 并 `flush()`，从而实现逐 token 可见的打字机效果。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/api/src/client.rs`](/rust/crates/api/src/client.rs) | `ProviderClient` 枚举、`stream_message` 多后端分发 |
| [`/rust/crates/api/src/providers/mod.rs`](/rust/crates/api/src/providers/mod.rs) | `Provider` trait、`detect_provider_kind`、`preflight_message_request` |
| [`/rust/crates/api/src/providers/anthropic.rs`](/rust/crates/api/src/providers/anthropic.rs) | Anthropic 原生 SSE 流式实现、`MessageStream`、`next_event` |
| [`/rust/crates/api/src/providers/openai_compat.rs`](/rust/crates/api/src/providers/openai_compat.rs) | OpenAI 兼容层请求翻译、SSE 解析、`StreamState` 归一化 |
| [`/rust/crates/api/src/sse.rs`](/rust/crates/api/src/sse.rs) | 通用 `SseParser`、SSE 帧切割与 JSON 反序列化 |
| [`/rust/crates/api/src/types.rs`](/rust/crates/api/src/types.rs) | `StreamEvent`、`AssistantEvent`、`ContentBlockDelta`、`OutputContentBlock` 定义 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime::run_turn`、`build_assistant_message`、事件收敛 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | `AnthropicRuntimeClient`、`consume_stream`、post-tool stall 检测与重试、流式/非流式降级 |
| [`/rust/crates/rusty-claude-cli/src/render.rs`](/rust/crates/rusty-claude-cli/src/render.rs) | `MarkdownStreamState`、`find_stream_safe_boundary`、`TerminalRenderer`、`Spinner` |
