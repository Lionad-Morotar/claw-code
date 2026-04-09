# Token 预算管理 - 上下文窗口动态计算

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/context/token-budget) 的目录结构，映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。

---

## 一句话总结

Token 预算管理是 claw-code 中通过 **本地字节估算** + **远程精确计数** 两级策略，在 API 调用前拦截超出上下文窗口的请求，并在会话累计 token 超过阈值时触发自动压缩的完整机制。

---

## 核心架构

### 上下文窗口：200K 不是全部

claw-code 的默认上下文窗口为 200K tokens，但实际可用于对话的空间需要扣除以下部分：

```
上下文窗口（200K）
├── 系统提示词（~15-25K，缓存后成本低）
├── 工具定义（~10-20K，含 MCP 工具）
├── 用户上下文（CLAUDE.md、git status 等）
├── 输出预留（max_tokens）
│   ├── Opus 模型默认：32K
│   └── Sonnet/Haiku 默认：64K
└── 剩余：对话历史空间（随对话增长）
```

在 `claw-code` 的 Rust 实现中，上下文窗口和输出限额被封装在 [`ModelTokenLimit`](rust/crates/api/src/providers/mod.rs#L47-L50) 结构中：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct ModelTokenLimit {
    pub max_output_tokens: u32,
    pub context_window_tokens: u32,
}
```

模型注册表定义在 [`model_token_limit`](rust/crates/api/src/providers/mod.rs#L242-L258) 函数中：

```rust
pub fn model_token_limit(model: &str) -> Option<ModelTokenLimit> {
    let canonical = resolve_model_alias(model);
    match canonical.as_str() {
        "claude-opus-4-6" => Some(ModelTokenLimit {
            max_output_tokens: 32_000,
            context_window_tokens: 200_000,
        }),
        "claude-sonnet-4-6" | "claude-haiku-4-5-20251213" => Some(ModelTokenLimit {
            max_output_tokens: 64_000,
            context_window_tokens: 200_000,
        }),
        "grok-3" | "grok-3-mini" => Some(ModelTokenLimit {
            max_output_tokens: 64_000,
            context_window_tokens: 131_072,
        }),
        _ => None,
    }
}
```

---

## Token 计数：近似 vs 精确

系统使用两级 token 计数策略：

### 近似估算（毫秒级，热路径使用）

在 [`providers/mod.rs`](rust/crates/api/src/providers/mod.rs#L287-L292) 中实现的 `estimate_serialized_tokens` 函数：

```rust
fn estimate_serialized_tokens<T: Serialize>(value: &T) -> u32 {
    serde_json::to_vec(value)
        .ok()
        .map_or(0, |bytes| (bytes.len() / 4 + 1) as u32)
}
```

这个近似算法假设每 4 字节 ≈ 1 token，与上游 TypeScript 实现的 `roughTokenCountEstimation` 策略一致。对不同内容类型的特殊处理：

- **JSON/JSONL**：字节密度高，但公式统一按 `/4` 处理
- **Tool Use**：序列化 `name + JSON.stringify(input)` 后估算
- **Thinking Block**：按实际文本长度 `/4`

### 精确计数（API 调用，关键决策点使用）

在 [`providers/anthropic.rs`](rust/crates/api/src/providers/anthropic.rs#L489-L520) 中，`preflight_message_request` 方法在本地估算通过后，会尝试调用 Anthropic 的 `beta.messages.countTokens` 端点：

```rust
async fn preflight_message_request(&self, request: &MessageRequest) -> Result<(), ApiError> {
    // 1. 先运行本地字节估算拦截器
    super::preflight_message_request(request)?;

    let Some(limit) = model_token_limit(&request.model) else {
        return Ok(());
    };

    // 2. 最佳努力尝试远程精确计数
    let counted_input_tokens = match self.count_tokens(request).await {
        Ok(count) => count,
        Err(_) => return Ok(()),  // 失败则跳过精确计数，继续后续流程
    };

    // 3. 使用精确计数进行最终拦截
    let estimated_total_tokens = counted_input_tokens.saturating_add(request.max_tokens);
    if estimated_total_tokens > limit.context_window_tokens {
        return Err(ApiError::ContextWindowExceeded { ... });
    }

    Ok(())
}
```

精确计数在以下场景使用：
- 压缩前后 token 对比
- Warning/Error 阈值判断
- 会话预算检查

---

## 3P Provider 的 Token 计数差异

| Provider | 计数方式 | 注意事项 |
| --- | --- | --- |
| Anthropic 直连 | `anthropic.beta.messages.countTokens()` | 标准 API，最准确 |
| AWS Bedrock | `CountTokensCommand` | 需要动态加载 279KB AWS SDK |
| Google Vertex | Anthropic SDK + beta 过滤 | 需要特定 beta headers |
| OpenAI 兼容层 | 无精确计数 | 退回到近似估算 |
| Gemini 兼容层 | 无精确计数 | 退回到近似估算 |

在 `claw-code` 的 Rust 实现中，OpenAI 和 Gemini 兼容层**不支持精确 token 计数**，系统会退回到近似估算。这会影响：
- 自动压缩触发时机（可能略有偏差）
- 压缩前后 token 对比（仅为估算值）

---

## 自动压缩的触发阈值

在 [`runtime/src/conversation.rs`](rust/crates/runtime/src/conversation.rs#L16-L18) 中定义了默认阈值：

```rust
const DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD: u32 = 100_000;
const AUTO_COMPACTION_THRESHOLD_ENV_VAR: &str = "CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS";
```

以 200K 窗口为例：
- **~100K**：自动压缩触发点（默认阈值）
- **可通过环境变量覆盖**：`CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS=50000`

阈值解析逻辑在 [`parse_auto_compact_threshold`](rust/crates/runtime/src/conversation.rs#L669-L674)：

```rust
#[must_use]
fn parse_auto_compaction_threshold(value: Option<&str>) -> u32 {
    value
        .and_then(|raw| raw.trim().parse::<u32>().ok())
        .filter(|threshold| *threshold > 0)
        .unwrap_or(DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD)
}
```

### 逃逸条件

`maybe_auto_compact` 方法（[`conversation.rs#L525-L548`](rust/crates/runtime/src/conversation.rs#L525-L548)）有多个逃逸条件：
- 累计 input tokens 未达到阈值 → 跳过
- 压缩后 `removed_message_count == 0` → 返回 `None`

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
            max_estimated_tokens: 0,  // 强制压缩
            ..CompactionConfig::default()
        },
    );

    if result.removed_message_count == 0 {
        return None;
    }

    self.session = result.compacted_session;
    Some(AutoCompactionEvent {
        removed_message_count: result.removed_message_count,
    })
}
```

---

## TokenUsage 数据结构

在 [`runtime/src/usage.rs`](rust/crates/runtime/src/usage.rs#L29-L36) 中定义了核心计数器：

```rust
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
pub struct TokenUsage {
    pub input_tokens: u32,
    pub output_tokens: u32,
    pub cache_creation_input_tokens: u32,
    pub cache_read_input_tokens: u32,
}
```

在 [`api/src/types.rs`](rust/crates/api/src/types.rs#L167-L206) 中定义了 API 响应结构：

```rust
#[derive(Debug, Clone, Default, PartialEq, Serialize, Deserialize)]
pub struct Usage {
    #[serde(default)]
    pub input_tokens: u32,
    #[serde(default)]
    pub cache_creation_input_tokens: u32,
    #[serde(default)]
    pub cache_read_input_tokens: u32,
    #[serde(default)]
    pub output_tokens: u32,
}

impl Usage {
    pub fn token_usage(&self) -> TokenUsage {
        TokenUsage {
            input_tokens: self.input_tokens,
            output_tokens: self.output_tokens,
            cache_creation_input_tokens: self.cache_creation_input_tokens,
            cache_read_input_tokens: self.cache_read_input_tokens,
        }
    }
}
```

### UsageTracker 累计追踪

在 [`runtime/src/usage.rs`](rust/crates/runtime/src/usage.rs#L167-L215) 中：

```rust
#[derive(Debug, Clone, Default, PartialEq, Eq)]
pub struct UsageTracker {
    latest_turn: TokenUsage,
    cumulative: TokenUsage,
    turns: u32,
}

impl UsageTracker {
    pub fn record(&mut self, usage: TokenUsage) {
        self.latest_turn = usage;
        self.cumulative.input_tokens += usage.input_tokens;
        self.cumulative.output_tokens += usage.output_tokens;
        self.cumulative.cache_creation_input_tokens += usage.cache_creation_input_tokens;
        self.cumulative.cache_read_input_tokens += usage.cache_read_input_tokens;
        self.turns += 1;
    }
}
```

---

## 全量压缩的完整流程

在 [`runtime/src/compact.rs`](rust/crates/runtime/src/compact.rs#L96-L139) 中：

```rust
pub fn compact_session(session: &Session, config: CompactionConfig) -> CompactionResult {
    if !should_compact(session, config) {
        return CompactionResult::no_op(session.clone());  // 无需压缩
    }

    // 1. 检查是否存在已有的压缩摘要
    let existing_summary = session
        .messages
        .first()
        .and_then(extract_existing_compacted_summary);

    // 2. 计算保留尾部的起始位置
    let keep_from = session
        .messages
        .len()
        .saturating_sub(config.preserve_recent_messages);

    // 3. 分离待压缩和保留的消息
    let removed = &session.messages[compacted_prefix_len..keep_from];
    let preserved = session.messages[keep_from..].to_vec();

    // 4. 合并摘要（保留历史压缩上下文）
    let summary = merge_compact_summaries(
        existing_summary.as_deref(),
        &summarize_messages(removed)
    );

    // 5. 构建新的系统消息 + 保留的尾部
    let continuation = get_compact_continuation_message(&summary, true, !preserved.is_empty());
    let mut compacted_messages = vec![ConversationMessage {
        role: MessageRole::System,
        blocks: vec![ContentBlock::Text { text: continuation }],
        usage: None,
    }];
    compacted_messages.extend(preserved);

    // 6. 更新 session 并记录压缩元数据
    let mut compacted_session = session.clone();
    compacted_session.messages = compacted_messages;
    compacted_session.record_compaction(summary.clone(), removed.len());

    CompactionResult {
        compacted_session,
        removed_message_count: removed.len(),
        formatted_summary: summary,
    }
}
```

### CompactionConfig 配置

在 [`compact.rs#L9-L12`](rust/crates/runtime/src/compact.rs#L9-L12)：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct CompactionConfig {
    pub preserve_recent_messages: usize,  // 默认 4 条
    pub max_estimated_tokens: usize,      // 默认 10,000
}
```

---

## Prompt Cache Sharing

压缩 API 调用是整个会话中最昂贵的操作之一。在 `claw-code` 中，压缩通过复用主线程的 prompt cache 前缀（system prompt + tools + context messages），理论上可显著提升缓存命中率。实际效果取决于模型 provider 的缓存实现和请求内容的稳定性。

---

## 输出 Token 的 Slot 优化

在 [`providers/mod.rs`](rust/crates/api/src/providers/mod.rs#L218-L230) 中：

```rust
pub fn max_tokens_for_model(model: &str) -> u32 {
    model_token_limit(model).map_or_else(
        || {
            let canonical = resolve_model_alias(model);
            if canonical.contains("opus") {
                32_000  // Opus 默认 32K
            } else {
                64_000  // 其他默认 64K
            }
        },
        |limit| limit.max_output_tokens,
    )
}
```

这个设计考虑了 API 的 slot 机制：按 `max_tokens` 预留推理容量。较低的默认值（相比 200K 窗口）意味着更多的 slot 容量可用于并发请求。

---

## 上下文窗口拦截器

在 [`providers/mod.rs`](rust/crates/api/src/providers/mod.rs#L260-L276) 中的 `preflight_message_request`：

```rust
pub fn preflight_message_request(request: &MessageRequest) -> Result<(), ApiError> {
    let Some(limit) = model_token_limit(&request.model) else {
        return Ok(());
    };

    let estimated_input_tokens = estimate_message_request_input_tokens(request);
    let estimated_total_tokens = estimated_input_tokens.saturating_add(request.max_tokens);

    if estimated_total_tokens > limit.context_window_tokens {
        return Err(ApiError::ContextWindowExceeded {
            model: resolve_model_alias(&request.model),
            estimated_input_tokens,
            requested_output_tokens: request.max_tokens,
            estimated_total_tokens,
            context_window_tokens: limit.context_window_tokens,
        });
    }

    Ok(())
}
```

错误类型定义在 [`api/src/error.rs`](rust/crates/api/src/error.rs#L31-L37)：

```rust
ContextWindowExceeded {
    model: String,
    estimated_input_tokens: u32,
    requested_output_tokens: u32,
    estimated_total_tokens: u32,
    context_window_tokens: u32,
}
```

---

## 端到端流程

```
用户输入 → ConversationRuntime::run_turn
         ↓
    写入 Session → UsageTracker 累计 input_tokens
         ↓
    检查 auto_compaction_input_tokens_threshold (默认 100K)
         ↓
    [若超过] → compact_session → 压缩旧消息 → 生成摘要
         ↓
    组装 ApiRequest → preflight_message_request
         ↓
    estimate_message_request_input_tokens (本地字节估算)
         ↓
    [若通过] → AnthropicClient::count_tokens (远程精确计数)
         ↓
    [若超过 context_window_tokens] → ApiError::ContextWindowExceeded
         ↓
    [若通过] → 发送 API 请求 → 返回 Usage
         ↓
    UsageTracker::record(usage) → 累计 token 用量
```

---

## 源码索引

| 文件 | 核心内容 | 行号 |
|------|----------|------|
| [`rust/crates/api/src/providers/mod.rs`](rust/crates/api/src/providers/mod.rs) | `ModelTokenLimit`、`model_token_limit`、`preflight_message_request` | L47-L50, L218-L276 |
| [`rust/crates/api/src/providers/anthropic.rs`](rust/crates/api/src/providers/anthropic.rs) | `preflight_message_request`、`count_tokens` | L489-L520 |
| [`rust/crates/api/src/types.rs`](rust/crates/api/src/types.rs) | `Usage`、`MessageRequest::max_tokens` | L8, L167-L206 |
| [`rust/crates/api/src/error.rs`](rust/crates/api/src/error.rs) | `ApiError::ContextWindowExceeded` | L34-L40 |
| [`rust/crates/runtime/src/usage.rs`](rust/crates/runtime/src/usage.rs) | `TokenUsage`、`UsageTracker`、成本估算 | L29-L36, L167-L215 |
| [`rust/crates/runtime/src/conversation.rs`](rust/crates/runtime/src/conversation.rs) | `run_turn`、`maybe_auto_compact`、阈值解析 | L16-L18, L505-L548, L558-L564 |
| [`rust/crates/runtime/src/compact.rs`](rust/crates/runtime/src/compact.rs) | `compact_session`、`estimate_session_tokens`、`should_compact` | L15-L21, L93-L132 |
| [`rust/crates/runtime/src/session.rs`](rust/crates/runtime/src/session.rs) | `ConversationMessage::usage`、`record_compaction` | L47-L51, L240-L248 |

---

## 关键测试用例

### 自动压缩触发测试

[`conversation.rs#L1476-L1508`](rust/crates/runtime/src/conversation.rs#L1476-L1508)：

```rust
#[test]
fn auto_compacts_when_cumulative_input_threshold_is_crossed() {
    struct SimpleApi;
    impl ApiClient for SimpleApi {
        fn stream(...) -> Result<Vec<AssistantEvent>, RuntimeError> {
            Ok(vec![
                AssistantEvent::TextDelta("done".to_string()),
                AssistantEvent::Usage(TokenUsage {
                    input_tokens: 120_000,  // 超过 100K 阈值
                    output_tokens: 4,
                    cache_creation_input_tokens: 0,
                    cache_read_input_tokens: 0,
                }),
                AssistantEvent::MessageStop,
            ])
        }
    }

    let mut runtime = ConversationRuntime::new(...)
        .with_auto_compaction_input_tokens_threshold(100_000);

    let summary = runtime.run_turn("trigger", None).expect("turn should succeed");

    assert_eq!(
        summary.auto_compaction,
        Some(AutoCompactionEvent {
            removed_message_count: 2,
        })
    );
}
```

### 上下文窗口拦截测试

[`providers/mod.rs#L619-L662`](rust/crates/api/src/providers/mod.rs#L619-L662)：

```rust
#[test]
fn preflight_blocks_requests_that_exceed_the_model_context_window() {
    let request = MessageRequest {
        model: "claude-opus-4-6".to_string(),
        max_tokens: 64_000,
        messages: vec![...],  // 构造 150K 估算输入
        system: Some("x".repeat(100_000)),
        ..Default::default()
    };

    match preflight_message_request(&request) {
        Err(ApiError::ContextWindowExceeded {
            context_window_tokens,
            estimated_total_tokens,
            ..
        }) => {
            assert!(estimated_total_tokens > context_window_tokens);
            assert_eq!(context_window_tokens, 200_000);
        }
        other => panic!("expected context-window preflight failure"),
    }
}
```

### 会话压缩测试

[`compact.rs#L537-L583`](rust/crates/runtime/src/compact.rs#L537-L583)：

```rust
#[test]
fn compacts_older_messages_into_a_system_summary() {
    let mut session = Session::new();
    session.messages = vec![
        ConversationMessage::user_text("one ".repeat(200)),
        ConversationMessage::assistant(vec![ContentBlock::Text {
            text: "two ".repeat(200),
        }]),
        ConversationMessage::tool_result("1", "bash", "ok ".repeat(200), false),
        ConversationMessage {
            role: MessageRole::Assistant,
            blocks: vec![ContentBlock::Text { text: "recent".to_string() }],
            usage: None,
        },
    ];

    let result = compact_session(
        &session,
        CompactionConfig {
            preserve_recent_messages: 2,
            max_estimated_tokens: 1,
        },
    );

    assert_eq!(result.removed_message_count, 2);
    assert_eq!(result.compacted_session.messages[0].role, MessageRole::System);
    assert!(result.formatted_summary.contains("Scope:"));
    assert!(result.formatted_summary.contains("Key timeline:"));
}
```

---

## 设计哲学

1. **防御性拦截**：在 API 调用前进行本地估算 + 远程精确计数双重检查，避免浪费 API 配额
2. **渐进式压缩**：先尝试 micro-compact（工具结果），再触发全量压缩
3. **可配置阈值**：通过环境变量 `CLAUDE_CODE_AUTO_COMPACT_INPUT_TOKENS` 允许用户调整触发点
4. **成本透明**：`TokenUsage` 包含 cache creation/read 分离，便于追踪缓存优化效果
5. **多 Provider 兼容**：对不支持精确计数的 Provider（OpenAI/Gemini 兼容层）自动降级到近似估算
