# 上下文压缩 - Compaction 三层策略与边界机制

深度解析 claw-code 上下文压缩的完整实现：会话摘要压缩（Session Compaction）、自动阈值触发压缩，以及 Token 预算控制策略，涵盖 `compact.rs`、`conversation.rs`、`session.rs` 和 `usage.rs` 中的关键源码位置。

---

## 压缩的触发时机

### 压缩入口的优先级链

在 claw-code 的 Rust runtime 中，压缩触发点集中在 `ConversationRuntime` 的单轮结束后。系统首先检查本轮累积的输入 token 是否超过阈值，再决定是否调用自动压缩。

```rust
// conversation.rs:521-545
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
    Some(AutoCompactionEvent {
        removed_message_count: result.removed_message_count,
    })
}
```

触发优先级链：

1. **阈值检查**：`input_tokens` 必须超过 `auto_compaction_input_tokens_threshold`（默认 100,000）。
2. **紧凑性检查**：`compact_session` 内部调用 `should_compact`，只有同时满足 `preserve_recent_messages` 数量下限和 `max_estimated_tokens` 上限双重条件才会真正压缩。
3. **状态替换**：压缩成功后，原 `session` 被替换为 `compacted_session`。

#### 源码映射

- 自动压缩触发逻辑：[conversation.rs](/rust/crates/runtime/src/conversation.rs#L525-L548)
- 默认阈值常量：[conversation.rs](/rust/crates/runtime/src/conversation.rs#L18)
- 阈值解析（支持环境变量覆盖）：[conversation.rs](/rust/crates/runtime/src/conversation.rs#L669-L674)

---

## 第一层：Session-level 紧凑判断 — `should_compact`

`should_compact` 是压缩的边界守卫。它会跳过已经存在于消息链顶部的 compaction system message，只评估后续消息是否满足压缩条件：

```rust
// compact.rs:41-51
pub fn should_compact(session: &Session, config: CompactionConfig) -> bool {
    let start = compacted_summary_prefix_len(session);
    let compactable = &session.messages[start..];

    compactable.len() > config.preserve_recent_messages
        && compactable
            .iter()
            .map(estimate_message_tokens)
            .sum::<usize>()
            >= config.max_estimated_tokens
}
```

关键边界机制：

- **忽略已有摘要前缀**：`compacted_summary_prefix_len` 检查首条消息是否是之前压缩产生的 system summary，避免对摘要本身重复评估。
- **双条件**：消息数量必须超过 `preserve_recent_messages`（默认保留 4 条），且可压缩部分的 token 估算值必须达到 `max_estimated_tokens`（默认 10,000）。
- `maybe_auto_compact` 将 `max_estimated_tokens` 设为 `0`，意味着只要可压缩消息数超过保留窗口即可触发压缩。

#### 源码映射

- 紧凑条件判断：[compact.rs](/rust/crates/runtime/src/compact.rs#L41-L51)
- 已有摘要前缀长度计算：[compact.rs](/rust/crates/runtime/src/compact.rs#L186-L193)
- 默认配置：`preserve_recent_messages = 4`，`max_estimated_tokens = 10,000`：[compact.rs](/rust/crates/runtime/src/compact.rs#L15-L22)

---

## 第二层：`compact_session` — 核心压缩实现

### 保留窗口的计算

```rust
// compact.rs:96-139
pub fn compact_session(session: &Session, config: CompactionConfig) -> CompactionResult {
    if !should_compact(session, config) {
        return CompactionResult { /* no-op */ };
    }

    let existing_summary = session
        .messages
        .first()
        .and_then(extract_existing_compacted_summary);
    let compacted_prefix_len = usize::from(existing_summary.is_some());
    let keep_from = session
        .messages
        .len()
        .saturating_sub(config.preserve_recent_messages);
    let removed = &session.messages[compacted_prefix_len..keep_from];
    let preserved = session.messages[keep_from..].to_vec();
    let summary =
        merge_compact_summaries(existing_summary.as_deref(), &summarize_messages(removed));
    // ... 构建 continuation system message 并替换 session
}
```

算法逻辑：

1. **提取已有摘要**：如果首条消息是 compaction summary，则保留并作为叠加上下文。
2. **计算保留窗口**：从尾部保留 `preserve_recent_messages` 条消息。
3. **压缩中间段**：`compacted_prefix_len..keep_from` 区间的消息被移除，并生成新的结构化摘要。
4. **合并摘要**：`merge_compact_summaries` 将新旧摘要合并为层级式总结（Previously compacted context / Newly compacted context / Key timeline）。
5. **构造 continuation message**：通过 `get_compact_continuation_message` 生成 system message，附加 `COMPACT_DIRECT_RESUME_INSTRUCTION`，防止模型在压缩后向用户提额外问题。
6. **持久化元数据**：调用 `session.record_compaction(summary, removed.len())` 写入 `SessionCompaction`。

#### 源码映射

- 核心压缩函数：[compact.rs](/rust/crates/runtime/src/compact.rs#L96-L139)
- 摘要合并：[compact.rs](/rust/crates/runtime/src/compact.rs#L283-L311)
- continuation message 生成：[compact.rs](/rust/crates/runtime/src/compact.rs#L71-L92)
- 工具对边界保护：[compact.rs](/rust/crates/runtime/src/compact.rs#L96-L139)
- 会话元数据记录：[session.rs](/rust/crates/runtime/src/session.rs#L240-L248)

---

## 第三层：摘要内容生成策略 — `summarize_messages`

`compact_session` 不使用外部 API 调用生成摘要，而是基于规则在本地构造结构化文本。摘要包含以下维度：

```rust
// compact.rs:151-236
fn summarize_messages(messages: &[ConversationMessage]) -> String {
    // 1. 统计各角色消息数量
    // 2. 收集去重后的工具名称
    // 3. 提取最近 3 条用户请求
    // 4. 推断 pending work（todo/next/pending/follow up/remaining）
    // 5. 收集关键文件路径（按扩展名白名单过滤）
    // 6. 推断当前工作（最近非空文本 block）
    // 7. 按时间线逐条 summarize_block（截断到 160 字符）
}
```

### 工具对完整性保护

目前 claw-code 的实现中，工具结果完整性通过**保留尾部消息窗口**与显式的**工具对边界保护**共同保证。`compact_session` 在计算 `keep_from` 后会额外检查：如果第一个被保留的消息以 `ToolResult` 开头，则向前回溯，确保配对的 `ToolUse` 消息也一同保留，避免在 compaction 边界处产生孤立的 tool 角色消息。这彻底避免了 "tool_result 缺少对应 tool_use" 的 API 错误。中间段消息整体摘要化，不保留原始 block。

#### 源码映射

- 消息摘要生成器：[compact.rs](/rust/crates/runtime/src/compact.rs#L151-L236)
- block 摘要与截断：[compact.rs](/rust/crates/runtime/src/compact.rs#L318-L333)
- 关键文件提取（扩展名白名单：rs, ts, tsx, js, json, md）：[compact.rs](/rust/crates/runtime/src/compact.rs#L376-L400)
- pending work 推断（关键词匹配）：[compact.rs](/rust/crates/runtime/src/compact.rs#L353-L375)

---

## Token 估算与预算管理

### `estimate_session_tokens`

claw-code 采用字符长度除以 4 的简化策略估算 token，不对每条消息调用真正的 tokenizer：

```rust
// compact.rs:35-37
pub fn estimate_session_tokens(session: &Session) -> usize {
    session.messages.iter().map(estimate_message_tokens).sum()
}
```

```rust
// compact.rs:399-411
fn estimate_message_tokens(message: &ConversationMessage) -> usize {
    message
        .blocks
        .iter()
        .map(|block| match block {
            ContentBlock::Text { text } => text.len() / 4 + 1,
            ContentBlock::ToolUse { name, input, .. } => (name.len() + input.len()) / 4 + 1,
            ContentBlock::ToolResult { tool_name, output, .. }
                => (tool_name.len() + output.len()) / 4 + 1,
        })
        .sum()
}
```

### `UsageTracker` — 真实 token 累计

与估算值不同，`UsageTracker` 从模型的 API 响应中读取真实的 `TokenUsage`，用于：

- 向用户展示精确的成本和 token 统计。
- 作为 `maybe_auto_compact` 的触发依据（`cumulative_usage().input_tokens`）。

```rust
// usage.rs:168-215
pub struct UsageTracker {
    latest_turn: TokenUsage,
    cumulative: TokenUsage,
    turns: u32,
}
```

#### 源码映射

- 会话级 token 估算：[compact.rs](/rust/crates/runtime/src/compact.rs#L35-L37)
- 单条消息 token 估算：[compact.rs](/rust/crates/runtime/src/compact.rs#L399-L411)
- 真实 token 累计器：`UsageTracker`：[usage.rs](/rust/crates/runtime/src/usage.rs#L168-L215)
- 自动压缩使用真实 `input_tokens` 作为阈值：[conversation.rs](/rust/crates/runtime/src/conversation.rs#L521-L528)

---

## 压缩边界与元数据持久化

### `SessionCompaction`

每次压缩后，会话对象会记录压缩元数据，用于持久化到 JSON/JSONL 并在会话恢复时重建状态：

```rust
// session.rs:53-58
pub struct SessionCompaction {
    pub count: u32,
    pub removed_message_count: usize,
    pub summary: String,
}
```

- `count`：该会话经历过多少次压缩（叠加上一层摘要时递增）。
- `removed_message_count`：本轮压缩实际移除的消息数量。
- `summary`：本轮压缩生成的原始摘要文本。

### 序列化与反序列化

`Session` 在 `to_json`/`from_json` 以及 JSONL 导出/导入流程中，都会完整携带 `compaction` 字段，保证会话重载后不会丢失压缩历史。

#### 源码映射

- 压缩元数据定义：[session.rs](/rust/crates/runtime/src/session.rs#L53-L58)
- 记录压缩行为：[session.rs](/rust/crates/runtime/src/session.rs#L240-L248)
- JSON 序列化中的 `compaction` 字段：[session.rs](/rust/crates/runtime/src/session.rs#L302-L303)
- JSON 反序列化恢复：[session.rs](/rust/crates/runtime/src/session.rs#L355-L380)
- JSONL 导出支持：[session.rs](/rust/crates/runtime/src/session.rs#L515-L516)

---

## Hook 与扩展机制

压缩前后可以通过 `HookRunner` 注入自定义行为。当前 `ConversationRuntime` 中 Hook 的调用点集中在工具调用前后（`run_pre_tool_use_hook`、`run_post_tool_use_hook`）。压缩流程本身未暴露独立的 `pre_compact` / `post_compact` Hook 入口，但外部编排层可以在检测到 `AutoCompactionEvent` 后追加扩展逻辑。

#### 源码映射

- Hook 执行器定义：[hooks.rs](/rust/crates/runtime/src/hooks.rs)
- `ConversationRuntime` 中 Hook 调用：[conversation.rs](/rust/crates/runtime/src/conversation.rs#L260-L320)

---

## 测试覆盖

`compact.rs` 包含针对压缩核心逻辑的单元测试，覆盖以下边界：

- 小会话不压缩（`leaves_small_sessions_unchanged`）
- 旧消息被正确汇总为 system summary（`compacts_older_messages_into_a_system_summary`）
- 多次压缩保留前次摘要上下文（`keeps_previous_compacted_context_when_compacting_again`）
- 已有 compaction summary 被正确忽略（`ignores_existing_compacted_summary_when_deciding_to_recompact`）
- 长 block 截断与 key files 提取验证

#### 源码映射

- 压缩模块测试：[compact.rs](/rust/crates/runtime/src/compact.rs#L519-L695)
- 会话持久化元数据测试：[session.rs](/rust/crates/runtime/src/session.rs#L1230-L1244)
- 自动压缩阈值解析测试：[conversation.rs](/rust/crates/runtime/src/conversation.rs#L1568-L1582)
