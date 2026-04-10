# Agentic Loop：AI 自主循环的核心机制

> 本报告基于 [Claude Code 中文文档 — Agentic Loop](https://ccb.agent-aura.top/docs/conversation/the-loop) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 什么是 Agentic Loop

传统聊天机器人：你问一句，它答一句。

Claude Code 不一样：你说一个需求，它可能连续执行十几步操作才给你最终结果。这背后的机制叫做 **Agentic Loop**（智能体循环）。在 `claw-code` 的 Rust 实现中，这一机制被封装在 `ConversationRuntime::run_turn()` 中，是一个显式的 `loop { ... }` 结构，每次迭代代表一次“思考 → 行动 → 观察”的完整周期。

### 源码映射：`run_turn` 就是循环本身

`ConversationRuntime` 定义于 [`runtime/src/conversation.rs#L126-L188`](/rust/crates/runtime/src/conversation.rs#L126-L188)：

```rust
pub struct ConversationRuntime<C, T> {
    session: Session,
    api_client: C,
    tool_executor: T,
    permission_policy: PermissionPolicy,
    system_prompt: Vec<String>,
    max_iterations: usize,
    usage_tracker: UsageTracker,
    hook_runner: HookRunner,
    // ...
}
```

真正驱动循环的是 `run_turn` 方法，起始于 [`conversation.rs#L296`](/rust/crates/runtime/src/conversation.rs#L296)。`run_turn` 实际定义在 [`conversation.rs#L296-L490`](/rust/crates/runtime/src/conversation.rs#L296-L490)。它的结构可以概括为：

1. 将用户输入追加到 `session`
2. 进入 `loop { ... }`
3. 组装 `ApiRequest`，调用 `api_client.stream(request)`
4. 解析流式事件，构建 `assistant_message`
5. 检测 `ContentBlock::ToolUse` —— 有则执行工具，无则 `break`
6. 工具结果以 `ContentBlock::ToolResult` 回写 `session`，继续下一轮
7. 循环结束后执行可选的 `auto_compaction`

这就是 Agentic Loop 在 Rust 源码中的直接映射：没有异步生成器，没有复杂状态机框架，只有一个显式的 `loop`，靠 `session.messages` 的累积来驱动多轮交互。

---

## 循环的完整结构

`run_turn` 的每次迭代包含四个阶段：上下文预处理、流式 API 调用、工具执行、终止判定。

### 源码映射：四个阶段在 `run_turn` 中的位置

以下是 `run_turn` 的相位拆解，对应 [`conversation.rs#L296-L490`](/rust/crates/runtime/src/conversation.rs#L296-L490) 附近的源码逻辑：

```rust
pub fn run_turn(
    &mut self,
    user_input: impl Into<String>,
    mut prompter: Option<&mut dyn PermissionPrompter>,
) -> Result<TurnSummary, RuntimeError> {
    // === 阶段 0：用户输入注入 ===
    self.session.push_user_text(user_input)?;

    loop {
        iterations += 1;
        if iterations > self.max_iterations { return Err(...); }

        // === 阶段 1：上下文预处理 ===
        let request = ApiRequest {
            system_prompt: self.system_prompt.clone(),
            messages: self.session.messages.clone(),
        };

        // === 阶段 2：流式 API 调用 ===
        let events = self.api_client.stream(request)?;
        let (assistant_message, usage, cache_events) = build_assistant_message(events)?;

        // === 阶段 3：ToolUse 提取 ===
        let pending_tool_uses = assistant_message.blocks.iter().filter_map(|b| match b {
            ContentBlock::ToolUse { id, name, input } => Some((id, name, input)),
            _ => None,
        }).collect::<Vec<_>>();

        self.session.push_message(assistant_message.clone())?;
        assistant_messages.push(assistant_message);

        if pending_tool_uses.is_empty() { break; }  // === 阶段 4：终止判定 ===

        // === 阶段 3：工具执行 ===
        for (tool_use_id, tool_name, input) in pending_tool_uses { ... }
    }

    // 退出循环后：auto compaction + 组装 TurnSummary
    let auto_compaction = self.maybe_auto_compact();
    Ok(TurnSummary { ... })
}
```

与 TypeScript 上游的复杂预处理管道（snip → microcompact → collapse → autocompact）不同，`claw-code` 的 `run_turn` 采取了更精简的设计：预处理阶段主要依靠 `ApiRequest` 组装时直接复制 `session.messages.clone()`，压缩/截断策略被下放到 `maybe_auto_compact()`，在**一轮循环结束后**统一触发，而不是每次 API 调用前都执行多级管道。

---

## 阶段 1：上下文预处理（Pre-Processing Pipeline）

在 `claw-code` 中，上下文预处理的核心职责是两项：

1. **维护消息历史**：`session.messages` 按顺序累积所有 `User`、`Assistant`、`Tool` 消息
2. **控制上下文长度**：当累计 input tokens 超过阈值时，自动触发 `compact_session`

### 源码映射：Session 的消息存储结构

消息存储在 `Session::messages` 中，类型为 `Vec<ConversationMessage>`。`ConversationMessage` 定义于 [`runtime/src/session.rs#L43-L47`](/rust/crates/runtime/src/session.rs#L43-L46)：

```rust
pub struct ConversationMessage {
    pub role: MessageRole,
    pub blocks: Vec<ContentBlock>,
    pub usage: Option<TokenUsage>,
}
```

其中 `ContentBlock`（[`session.rs#L28-L47`](/rust/crates/runtime/src/session.rs#L28-L46)）是结构化内容的枚举：

```rust
pub enum ContentBlock {
    Text { text: String },
    ToolUse { id: String, name: String, input: String },
    ToolResult { tool_use_id: String, tool_name: String, output: String, is_error: bool },
}
```

注意 `ToolResult` 直接携带 `tool_use_id` 和 `tool_name`，确保模型发出的每个 `ToolUse` 都有严格的请求-响应配对。

### 源码映射：自动压缩 `maybe_auto_compact`

自动压缩逻辑在 [`conversation.rs#L525-L548`](/rust/crates/runtime/src/conversation.rs#L525-L548)：

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

默认阈值由常量 [`DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD`](/rust/crates/runtime/src/conversation.rs#L16) 设定为 `100_000`，也可以通过环境变量 `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS` 覆盖。这与文档中描述的“autocompact 超出阈值时触发”完全一致，只是触发时机放在了一轮 turn 结束之后，而非每次流式调用之前。

### 源码映射：`compact_session` 的实现

`compact_session` 定义于 [`runtime/src/compact.rs#L96-L139`](/rust/crates/runtime/src/compact.rs#L96-L139)。其策略是：

1. 检查是否需要压缩（`should_compact`）
2. 若已有压缩摘要，则提取后合并
3. 保留最近 `preserve_recent_messages` 条消息作为尾部
4. 将中间被移除的消息送入 `summarize_messages`
5. 生成一条 `MessageRole::System` 的摘要消息，替换到 `compacted_session.messages` 开头

这种设计与文档中的“上下文压缩 / Compaction 三层策略”对应，但在 Rust 实现中被收敛为单一函数调用，配置项由 `CompactionConfig`（[`compact.rs#L10-L13`](/rust/crates/runtime/src/compact.rs#L10-L13)）控制：

```rust
pub struct CompactionConfig {
    pub preserve_recent_messages: usize,
    pub max_estimated_tokens: usize,
}

impl Default for CompactionConfig {
    fn default() -> Self {
        Self {
            preserve_recent_messages: 4,
            max_estimated_tokens: 10_000,
        }
    }
}
```

---

## 阶段 2：流式 API 调用（Streaming Loop）

`claw-code` 的流式处理采用**批量事件模型**：`api_client.stream(request)` 返回一个 `Result<Vec<AssistantEvent>, RuntimeError>`，而不是逐 token 拉取的异步生成器。这是 Rust 实现与 TypeScript 上游在接口层面的重大差异，但语义等价——事件序列仍然完整表达了从文本增量、工具调用到使用量和缓存信号的全部信息。

### 源码映射：`ApiClient` trait 与 `AssistantEvent`

`ApiClient` 定义于 [`conversation.rs#L56-L61`](/rust/crates/runtime/src/conversation.rs#L56-L60)：

```rust
pub trait ApiClient {
    fn stream(&mut self, request: ApiRequest) -> Result<Vec<AssistantEvent>, RuntimeError>;
}
```

`AssistantEvent` 枚举（[`conversation.rs#L30-L48`](/rust/crates/runtime/src/conversation.rs#L30-L48)）涵盖了一次 assistant turn 中可能收到的全部事件：

```rust
pub enum AssistantEvent {
    TextDelta(String),
    ToolUse { id: String, name: String, input: String },
    Usage(TokenUsage),
    PromptCache(PromptCacheEvent),
    MessageStop,
}
```

### 源码映射：事件组装成消息 `build_assistant_message`

`build_assistant_message` 负责将一维事件序列还原为结构化的 `ConversationMessage`，定义于 [`conversation.rs#L676-L723`](/rust/crates/runtime/src/conversation.rs#L676-L723)：

```rust
fn build_assistant_message(events: Vec<AssistantEvent>) -> Result<(ConversationMessage, Option<TokenUsage>, Vec<PromptCacheEvent>), RuntimeError> {
    let mut text = String::new();
    let mut blocks = Vec::new();
    let mut prompt_cache_events = Vec::new();
    let mut finished = false;
    let mut usage = None;

    for event in events {
        match event {
            AssistantEvent::TextDelta(delta) => text.push_str(&delta),
            AssistantEvent::ToolUse { id, name, input } => {
                flush_text_block(&mut text, &mut blocks);
                blocks.push(ContentBlock::ToolUse { id, name, input });
            }
            AssistantEvent::Usage(value) => usage = Some(value),
            AssistantEvent::PromptCache(event) => prompt_cache_events.push(event),
            AssistantEvent::MessageStop => { finished = true; }
        }
    }

    flush_text_block(&mut text, &mut blocks);

    if !finished {
        return Err(RuntimeError::new("assistant stream ended without a message stop event"));
    }
    if blocks.is_empty() {
        return Err(RuntimeError::new("assistant stream produced no content"));
    }

    Ok((ConversationMessage::assistant_with_usage(blocks, usage), usage, prompt_cache_events))
}
```

此处有两个关键守卫：

- **必须收到 `MessageStop`**：否则视为流异常终止
- **必须产生非空 `blocks`**：防止模型返回完全空内容

这与文档中描述的“流式回调中的关键守卫”语义一致，但实现方式更直接——Rust 侧在事件收集阶段就完成校验，而不是在异步生成器的中间状态里做回调拦截。

### 源码映射：Prompt Cache 事件追踪

`PromptCacheEvent`（[`conversation.rs#L50-L58`](/rust/crates/runtime/src/conversation.rs#L50-L57)）记录了缓存异常的诊断信息：

```rust
pub struct PromptCacheEvent {
    pub unexpected: bool,
    pub reason: String,
    pub previous_cache_read_input_tokens: u32,
    pub current_cache_read_input_tokens: u32,
    pub token_drop: u32,
}
```

`run_turn` 会将每个 turn 产生的 `prompt_cache_events` 累积到 `TurnSummary` 中，供上层做缓存稳定性诊断。测试代码（[`conversation.rs#L955-L1020`](/rust/crates/runtime/src/conversation.rs#L955-L1020)）演示了一个 `unexpected: true` 的场景，验证了缓存 token 下降 5,000 时的事件捕获能力。

---

## 阶段 3：工具执行（Tool Execution）

当 assistant message 中包含 `ContentBlock::ToolUse` 时，循环不会终止，而是进入工具执行阶段。`claw-code` 的工具执行链路可以概括为：

**Pre-tool hook → 权限检查 → ToolExecutor 调用 → Post-tool hook → 结果回写 Session**

### 源码映射：工具执行与 Hooks 的完整链路

在 `run_turn` 中，每个 pending tool use 都会经过以下代码块（[`conversation.rs#L348-L490`](/rust/crates/runtime/src/conversation.rs#L348-L490)）。这是一个较宽的范围；hook → 权限 → 执行 → 结果回写的实际循环体从 `L348` 附近开始，到 `L490` 附近完成单次工具调用的处理：

```rust
for (tool_use_id, tool_name, input) in pending_tool_uses {
    // 1. Pre-tool hook
    let pre_hook_result = self.run_pre_tool_use_hook(&tool_name, &input);
    let effective_input = pre_hook_result.updated_input().map_or_else(|| input.clone(), ToOwned::to_owned);
    let permission_context = PermissionContext::new(
        pre_hook_result.permission_override(),
        pre_hook_result.permission_reason().map(ToOwned::to_owned),
    );

    // 2. 权限判定
    let permission_outcome = if pre_hook_result.is_cancelled() { ... Deny ... }
        else if pre_hook_result.is_failed() { ... Deny ... }
        else if pre_hook_result.is_denied() { ... Deny ... }
        else if let Some(prompt) = prompter.as_mut() {
            self.permission_policy.authorize_with_context(&tool_name, &effective_input, &permission_context, Some(*prompt))
        } else {
            self.permission_policy.authorize_with_context(&tool_name, &effective_input, &permission_context, None)
        };

    // 3. 执行与 Post-tool hooks
    let result_message = match permission_outcome {
        PermissionOutcome::Allow => {
            self.record_tool_started(iterations, &tool_name);
            let (mut output, mut is_error) = match self.tool_executor.execute(&tool_name, &effective_input) {
                Ok(output) => (output, false),
                Err(error) => (error.to_string(), true),
            };
            output = merge_hook_feedback(pre_hook_result.messages(), output, false);

            let post_hook_result = if is_error {
                self.run_post_tool_use_failure_hook(&tool_name, &effective_input, &output)
            } else {
                self.run_post_tool_use_hook(&tool_name, &effective_input, &output, false)
            };
            // ... 合并 hook feedback，标记 is_error ...
            ConversationMessage::tool_result(tool_use_id, tool_name, output, is_error)
        }
        PermissionOutcome::Deny { reason } => {
            ConversationMessage::tool_result(tool_use_id, tool_name, merge_hook_feedback(...), true)
        }
    };

    // 4. 回写 Session
    self.session.push_message(result_message.clone())?;
    self.record_tool_finished(iterations, &result_message);
    tool_results.push(result_message);
}
```

这一链路完整对应了文档中“工具执行”阶段的描述，并额外揭示了 Hooks 在权限判定前后的拦截能力。

### 源码映射：`PermissionPolicy` 的三级判定

权限核心逻辑在 [`runtime/src/permissions.rs#L174-L292`](/rust/crates/runtime/src/permissions.rs#L174-L292) 的 `authorize_with_context` 中。它的判定顺序是：

1. **`deny_rules`** 静态拒绝规则（正则匹配）
2. **Hook override**（`PermissionOverride::Deny / Ask / Allow`）
3. **`ask_rules`** 静态询问规则
4. **`allow_rules`** 静态允许规则
5. **`current_mode` vs `required_mode`** 模式兜底

```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,
    Allow,
}
```

这与文档中的 “Allow / Ask / Deny 三级权限体系” 对应，但 Rust 实现扩展为五级枚举，覆盖了从完全只读到完全放行的完整光谱。

### 源码映射：`HookRunner` 的生命周期钩子

`HookRunner` 定义于 [`runtime/src/hooks.rs#L152-L987`](/rust/crates/runtime/src/hooks.rs)。支持三类钩子：

- `PreToolUse`：工具执行前拦截/改写输入
- `PostToolUse`：工具成功执行后追加反馈
- `PostToolUseFailure`：工具报错后追加反馈

钩子通过外部 shell 命令实现，环境变量注入工具上下文：`HOOK_EVENT`、`HOOK_TOOL_NAME`、`HOOK_TOOL_INPUT`、`HOOK_TOOL_OUTPUT`、`HOOK_TOOL_IS_ERROR`。退出码约定：

- `0`：允许（stdout 可被解析为 JSON 以修改输入/追加反馈）
- `2`：拒绝
- 其他非零：失败

测试代码（[`conversation.rs#L1015-L1105`](/rust/crates/runtime/src/conversation.rs#L1015-L1105)）验证了当 PreToolUse 钩子以退出码 2 运行时，工具不会被执行，而是直接生成一个 `is_error: true` 的 `ToolResult`。

---

## 阶段 4：终止或继续

`run_turn` 的终止条件非常直接：`pending_tool_uses.is_empty()` 时 `break`。这与 TypeScript 上游的 7 种终止条件相比更加收敛，因为很多上游的终止场景（如 `model_error`、`abort_streaming`）在 Rust 实现中被统一收敛为 `Result<TurnSummary, RuntimeError>` 的 `Err` 分支提前返回。

### 源码映射：显式终止条件

在 [`conversation.rs#L335-L338`](/rust/crates/runtime/src/conversation.rs#L335-L338)：

```rust
if pending_tool_uses.is_empty() {
    break;
}
```

这意味着只要模型在一次流式响应中没有请求任何工具，本轮 `run_turn` 就结束了。随后 `maybe_auto_compact()` 执行可选压缩，并返回 `TurnSummary`：

```rust
pub struct TurnSummary {
    pub assistant_messages: Vec<ConversationMessage>,
    pub tool_results: Vec<ConversationMessage>,
    pub prompt_cache_events: Vec<PromptCacheEvent>,
    pub iterations: usize,
    pub usage: TokenUsage,
    pub auto_compaction: Option<AutoCompactionEvent>,
}
```

定义见 [`conversation.rs#L107-L120`](/rust/crates/runtime/src/conversation.rs#L107-L120)。

### 源码映射：隐式/错误终止条件

除了正常的 `break`，以下情况也会导致 `run_turn` 提前终止（返回 `Err`）：

| 终止原因 | 源码位置 | 机制 |
|----------|----------|------|
| **max_iterations 超出** | [`conversation.rs#L304-L314`](/rust/crates/runtime/src/conversation.rs#L304-L314) | `iterations > self.max_iterations` → 返回 `RuntimeError` |
| **API 调用失败** | [`conversation.rs#L319-L325`](/rust/crates/runtime/src/conversation.rs#L319-L325) | `api_client.stream(request)` 返回 `Err` → 直接传播 |
| **流事件不完整** | [`conversation.rs#L326-L332`](/rust/crates/runtime/src/conversation.rs#L326-L332) | `build_assistant_message` 失败（无 `MessageStop` 或空 `blocks`）→ 返回 `Err` |

这些隐式终止条件与文档中的 `blocking_limit`、`aborted_streaming`、`model_error` 等场景在语义上等价，但 Rust 实现通过 `Result` 类型系统进行了更紧凑的错误传播。

### 源码映射：max_iterations 安全网

`max_iterations` 默认值为 `usize::MAX`（[`conversation.rs#L181`](/rust/crates/runtime/src/conversation.rs#L181)），但可以通过 builder 方法限制：

```rust
pub fn with_max_iterations(mut self, max_iterations: usize) -> Self {
    self.max_iterations = max_iterations;
    self
}
```

测试代码 [`conversation.rs#L1583-L1608`](/rust/crates/runtime/src/conversation.rs#L1583-L1608) 验证了一个循环 API 客户端在 `max_iterations = 1` 时的行为——当第一个 turn 永远返回 `ToolUse` 时，第二个迭代就会触发超限错误。

---

## 7 种终止条件（源码级）

将 TypeScript 上游的 7 种终止条件映射到 `claw-code` Rust 实现后，对应关系如下：

| 上游名称 | Rust 实现中的等价行为 | 触发逻辑 |
|----------|----------------------|----------|
| `completed` | `pending_tool_uses.is_empty() → break` | 正常完成，返回 `TurnSummary` |
| `blocking_limit` | `api_client.stream()` 返回硬错误，或 `max_iterations` 超限 | `Err(RuntimeError)` 提前返回 |
| `aborted_streaming` | 暂无显式 abort controller；`HookAbortSignal` 可在 hooks 层面取消 | `pre_tool_use_hook` 被取消时生成 `Deny` 结果 |
| `model_error` | `api_client.stream()` 返回 `Err` | 直接传播上游错误 |
| `prompt_too_long` | 依赖 `api_client` 返回错误；`maybe_auto_compact` 在 turn 后压缩 | 长上下文通过自动压缩降级，而非 reactive drain |
| `image_error` | 由 `api_client` 内部处理 | 未显式在 `conversation.rs` 中体现 |
| `stop_hook_prevented` | `pre_hook_result.is_denied() / is_failed() / is_cancelled()` | 工具被 hooks 拦截后直接生成 `ToolResult`（is_error） |

Rust 实现有意识地**收敛了状态空间**：很多上游的细粒度终止条件被统一为 `Result` 的 `Ok`/`Err` 二分，从而降低了循环本身的认知复杂度。相应的复杂度被转移到了 `ApiClient` 和 `ToolExecutor` 的实现中。

---

## 4 种继续条件（恢复路径）

### 1. 正常工具循环

有 `ToolUse` → 执行工具 → `ToolResult` 回写 `session.messages` → `continue`。这是最典型的路径，对应代码中的 `for (tool_use_id, tool_name, input) in pending_tool_uses { ... }` 块。

### 2. 权限拒绝后的继续

当 `PermissionPolicy` 或 Hooks 拒绝工具时，不会中断循环，而是生成一个 `is_error: true` 的 `ToolResult` 回写 Session。模型在下一轮会看到错误反馈并调整策略。这对应上游的 "Stop hook 阻塞重试" 语义。

### 3. 自动压缩后的继续

`maybe_auto_compact()` 在 turn 结束后触发，将历史消息替换为摘要。下一轮循环的 `ApiRequest` 会自动基于压缩后的 `session.messages`，从而在长对话中恢复可用上下文。这对应上游的 "Prompt-Too-Long 恢复" 和 "autocompact" 机制。

### 4. Hook 改写输入后的继续

`run_pre_tool_use_hook` 可以解析 stdout JSON 来修改 `tool_input`（通过 `updated_input` 字段）。修改后的输入进入 `authorize_with_context` 和 `tool_executor.execute()`。这虽然不是显式的“恢复”路径，但也是一种在循环内部修正执行策略的机制。

---

## 状态机：State 对象

TypeScript 上游使用一个不可变的 `State` 对象在每次 `continue` 时传递上下文。`claw-code` 的 Rust 实现没有单独的 `State` struct，但功能等价的上下文被分散在 `ConversationRuntime` 的字段和 `run_turn` 的局部变量中：

| 上游 State 字段 | Rust 等价物 | 说明 |
|----------------|-------------|------|
| `messages` | `self.session.messages` | 当前对话消息历史 |
| `toolUseContext` | `self.permission_policy` + `prompter` 参数 | 工具上下文（含权限策略）|
| `autoCompactTracking` | `self.usage_tracker` + `maybe_auto_compact()` | 压缩跟踪 |
| `maxOutputTokensRecoveryCount` | 无直接等价 | Rust 实现尚未暴露该恢复逻辑 |
| `hasAttemptedReactiveCompact` | 无直接等价 | 自动压缩在 turn 后统一触发 |
| `turnCount` | `iterations` 局部变量 | 轮次计数 |
| `transition` | `TurnSummary` 返回值 | 上一轮的结果摘要 |

Rust 实现采用**可变状态（mutable self）风格**，而不是不可变拷贝。这符合 Rust 的所有权模型：`
`self.session` 和 `self.usage_tracker` 在 `&mut self` 下就地更新，避免了大规模消息数组的克隆开销。

---

## Token Budget（基于累计阈值的实验性实现）

`claw-code` 通过 `UsageTracker`（[`runtime/src/usage.rs`](/rust/crates/runtime/src/usage.rs)）跟踪 token 消耗：

```rust
pub struct UsageTracker {
    latest_turn: TokenUsage,     // 最近一轮的 token 消耗
    cumulative: TokenUsage,      // 会话累计 token 消耗
    turns: u32,                  // 已完成的轮次计数
}
```

其中 `TokenUsage` 定义在 [`usage.rs#L31-L36`](/rust/crates/runtime/src/usage.rs#L31-L36)：

```rust
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub cache_creation_input_tokens: u32,
    pub cache_read_input_tokens: u32,
}
```

每次 `build_assistant_message` 解析出 `Usage` 事件时，`run_turn` 会调用 `self.usage_tracker.record(usage)`（[`conversation.rs#L335-L338`](/rust/crates/runtime/src/conversation.rs#L335-L338)）。`maybe_auto_compact` 以 `cumulative_usage().input_tokens` 作为触发条件，这本质上就是一种基于累计 token 的预算管理。但它与上游 TypeScript 的 `TOKEN_BUDGET` feature（带有收益递减检测和 continuation nudge 注入）仍有差距，当前仅实现了压缩阈值触发。

但与 TypeScript 上游的 `TOKEN_BUDGET` feature（带有 `continuation` / `diminishing_returns` nudge 消息）相比，`claw-code` 目前仅实现了**压缩阈值触发**，尚未实现循环中注入 nudge 消息或收益递减检测。

---

## 为什么不是"一次规划，批量执行"

`claw-code` 选择逐步循环而非一次性批量规划的根本原因在于：工具执行结果对模型来说是**不可预测的观察**。逐步循环的优势在 Rust 源码中体现为以下三点：

1. **真实信息流**：`tool_executor.execute()` 的返回值（命令输出、文件内容、编译错误）不可能被模型预先规划，必须实际执行后才能获得
2. **动态上下文管理**：每轮 `run_turn` 结束后，`maybe_auto_compact()` 根据最新的 `usage_tracker` 重新评估压缩需求——如果上一轮工具返回大量文本，下一轮就可能触发 compaction
3. **错误即时修正**：工具失败或 Hook 拒绝后，`ToolResult` 会立即回写 Session，下一 iteration 的模型请求天然包含错误上下文，无需推倒重来

这种设计与 TypeScript 上游的核心理念完全一致，只是实现层面更加精简：没有 `AsyncGenerator` 的状态维护开销，一个显式的 `loop` 配合 `Result` 错误传播即完成了全部语义。

### 审视角：`Vec<AssistantEvent>` 的内存假设

`api_client.stream(request)` 返回的是一个完全物化的 `Vec<AssistantEvent>`，而不是逐 token 拉取的流。这意味着一次 assistant turn 的所有事件必须在内存中完整收集后，才能进入 `build_assistant_message`。对于输出极长的响应（如模型返回数万字的分析或大量工具调用），这个设计会造成显著的内存峰值。虽然 Rust 的所有权模型避免了拷贝，但 `Vec` 的预分配和扩容仍可能成为长对话中的隐性瓶颈。如果将来需要支持超大规模输出或低内存环境，可能需要将接口从 `Vec<...>` 迁移到迭代器或流式消费者。

---

## 一个完整的迭代示例

用户："帮我找到项目里所有未使用的导入语句，然后删掉它们"

### 迭代 1：思考 → 行动

- **API 调用**：模型返回 `ToolUse { name: "glob_search", input: "**/*.rs" }`
- **工具执行**：返回 42 个 Rust 文件路径
- **继续循环**：`needsFollowUp` 等价于 `pending_tool_uses.len() > 0` → 继续

### 迭代 2：思考 → 行动

- **API 调用**：模型返回 `ToolUse { name: "grep_search", input: "use .*;" }`
- **工具执行**：在 15 个文件中找到 120 条 import 语句
- **继续循环**

### 迭代 3：思考 → 行动（多轮）

- **上下文预处理**：若 120 条结果导致 token 量超过阈值，`maybe_auto_compact` 会压缩摘要
- **API 调用**：模型返回多个 `ToolUse { name: "edit_file" }`
- **工具执行**：删除 5 条未使用导入
- **继续循环**

### 迭代 4：总结

- **API 调用**：模型返回纯文本 `"已清理 3 个文件中的 5 条未使用导入"`
- **终止**：`pending_tool_uses.is_empty()` → `break`
- **收尾**：`maybe_auto_compact()` 检查阈值，组装 `TurnSummary` 返回

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime`、`run_turn`、agentic loop、事件组装、auto compaction |
| [`/rust/crates/runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) | `Session`、`ConversationMessage`、`ContentBlock`、消息持久化JSON |
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode`、`PermissionPolicy`、`authorize_with_context` |
| [`/rust/crates/runtime/src/hooks.rs`](/rust/crates/runtime/src/hooks.rs) | `HookRunner`、`HookRunResult`、Pre/Post tool hooks 生命周期 |
| [`/rust/crates/runtime/src/compact.rs`](/rust/crates/runtime/src/compact.rs) | `compact_session`、`CompactionConfig`、上下文压缩摘要 |
| [`/rust/crates/runtime/src/usage.rs`](/rust/crates/runtime/src/usage.rs) | `UsageTracker`、token 用量累计追踪 |
