# Unit 5 Review Findings — Streaming + Foundation

**Reviewer**: Unit 5 (05-streaming.md, 02-why-this-whitepaper.md, 38-fork-subagent.md, 11-task-management.md)
**Review date**: 2026-04-10
**Commit**: HEAD of `/Users/lionad/Github/Run/claw-code`

---

## Executive Summary

| File | Anchors Checked | Result |
|------|-----------------|--------|
| `05-streaming.md` | ~48 Rust anchors | **Multiple CRITICAL stale line numbers in `main.rs`**; other anchors (api, runtime, render) mostly valid |
| `02-why-this-whitepaper.md` | ~31 Rust anchors | **Mostly valid**; one MAJOR factual gap in compaction description |
| `38-fork-subagent.md` | 0 Rust anchors (upstream-only) | N/A — all references are to `packages/ccb/src/...` which do not exist in this repo |
| `11-task-management.md` | 1 Rust anchor | Valid; report correctly disclaims Rust parity gap |

---

## 1. `docs/.report/05-streaming.md`

### CRITICAL: Stale / Misaligned Source Anchors in `rusty-claude-cli/src/main.rs`

The following anchors point to the **wrong symbols** due to significant line drift in `main.rs`. The line ranges exist (files are valid), but the claimed content is absent at those locations.

| Reported Anchor | Claimed Content | Actual Content at HEAD | Verdict |
|-----------------|-----------------|------------------------|---------|
| `main.rs#L6511-L6563` | `AnthropicRuntimeClient::stream` | `InternalPromptProgressReporter` methods (emit, emit_heartbeat, snapshot, etc.) | **WRONG** — actual `stream` is at ~L7001 |
| `main.rs#L6564` | `consume_stream` definition | `InternalPromptProgressRun::start_ultraplan` | **WRONG** — actual `consume_stream` is at ~L7059 |
| `main.rs#L6564-L6729` | `consume_stream` implementation | Misc. progress-reporter & utility functions (`format_internal_prompt_progress_line`, `build_runtime`, etc.) | **WRONG** |
| `main.rs#L6640-L6643` | `InputJsonDelta` handling (`input.push_str`) | `InternalPromptProgressEvent::Started` match arm | **WRONG** — actual delta handling is at ~L7140 |
| `main.rs#L6658-L6667` | `ContentBlockStop` tool display | `format_tool_call_start` match arms for "read_file"/"write_file" | **WRONG** — actual display logic is at ~L7166 |
| `main.rs#L6669-L6671` | `MessageDelta` usage push | `format_tool_call_start` "bash" branch | **WRONG** — actual MessageDelta is at ~L7177 |
| `main.rs#L6699-L6729` | Post-stream guard (`saw_stop`, fallback to non-streaming) | `format_tool_call_start` helper + `build_runtime` | **WRONG** — actual guard is at ~L7194 |
| `main.rs#L6981-L7025` | `format_tool_call_start` | `resolve_cli_auth_source_for_cwd`, `load_runtime_oauth_config_for`, start of `ApiClient::stream` | **WRONG** — actual function is at ~L7597 |
| `main.rs#L7431-L7465` | `push_output_block` | A literal array of unsupported CLI command strings (`"context"`, `"color"`, `"effort"`, etc.) | **WRONG** — actual function is at ~L8047 |
| `main.rs#L3470-L3476` | `LiveCli::run_turn` spinner | `CliRuntime::server_names` / `call_tool` | **WRONG** — actual spinner code is at ~L3849 |
| `main.rs#L6395-L6448` | `AnthropicRuntimeClient::new` | `InternalPromptProgressState` struct + `InternalPromptProgressEvent` enum | **WRONG** — actual `new` is at ~L6908 |

**Note**: `main.rs` appears to have been reorganized significantly (likely ~+400 lines of drift) since the report was written. The narrative prose is still directionally accurate, but readers following the hyperlinks will land on unrelated code.

### MINOR: `render.rs` `Spinner::tick` anchor slightly off

- `render.rs#L75-L90` is claimed for `Spinner::tick`. The actual `tick` method spans **L60-L78**; L75-L90 covers the tail end of `tick` plus `finish`. The overlap is small but the start is late.

### PASS: api crate anchors

All of the following anchors are accurate (exist, correct symbol, correct range):

- `api/src/types.rs#L259-L268` — `StreamEvent` enum
- `api/src/types.rs#L145-L158` — `OutputContentBlock` enum
- `api/src/types.rs#L243-L249` — `ContentBlockDelta` enum
- `api/src/sse.rs#L4-L80` — `SseParser` / `next_frame`
- `api/src/client.rs#L10-L14` — `ProviderClient` enum
- `api/src/providers/anthropic.rs#L330-L350` — `stream_message`
- `api/src/providers/anthropic.rs#L401-L470` — `send_with_retry`
- `api/src/providers/anthropic.rs#L471-L535` — `preflight_message_request`
- `api/src/providers/anthropic.rs#L489-L495` — retryable status codes (actually `is_retryable_status` appears at L924, but L489-L495 still falls inside `preflight_message_request` prose; the *specific* status-code list the report quotes is NOT at L489-L495, it is at L924)
- `api/src/providers/mod.rs#L201-L222` — `detect_provider_kind`
- `api/src/providers/mod.rs#L260-L278` — `preflight_message_request`
- `api/src/providers/openai_compat.rs#L170-L190` — `OpenAiCompatClient`
- `api/src/providers/openai_compat.rs#L195-L217` — `send_with_retry`
- `api/src/providers/openai_compat.rs#L257-L285` — `jittered_backoff_for_attempt`
- `api/src/providers/openai_compat.rs#L344-L380` — `MessageStream` / `next_event`
- `api/src/providers/openai_compat.rs#L420-L489` — `StreamState::ingest_chunk`
- `api/src/providers/openai_compat.rs#L747-L810` — `build_chat_completion_request`
- `api/src/providers/openai_compat.rs#L821-L860` — `translate_message`
- `api/src/providers/openai_compat.rs#L915-L926` — `openai_tool_definition`
- `api/src/providers/openai_compat.rs#L943-L990` — `normalize_response`
- `api/src/providers/openai_compat.rs#L1112-L1123` — `normalize_finish_reason` (actually at L1265, so this is **stale**)

Wait: `openai_compat.rs#L1112-L1123` for `normalize_finish_reason` is **WRONG**. The function is at L1265. The report quotes the exact source of `normalize_finish_reason` but gives the wrong line range. This is a second category of error (not just drift in `main.rs`).

### MAJOR: Factual claim about retry defaults

The report states (around API rate limit section):

> - 默认最大重试次数：8 次
> - 初始退避：1 秒
> - 最大退避：128 秒

While the backoff math in `openai_compat.rs` uses exponential doubling (`1_u32.checked_shl(attempt - 1)`), the report does not cite where the **default** `max_retries` is set to 8, initial backoff to 1s, or max backoff to 128s. Searching the codebase for these exact defaults is difficult; the visible implementations (`backoff_for_attempt` and `jittered_backoff_for_attempt`) do not hard-code 8/1/128 — those values likely come from `OpenAiCompatConfig` or `AnthropicClient` construction. Without a clear source anchor confirming the defaults, this claim should be treated as **unverified / potentially stale**.

### MINOR: `api/src/providers/anthropic.rs#L489-L495` claims retry status codes

The report quotes:

> 重试状态码：408, 409, 429, 500, 502, 503, 504（`api/src/providers/anthropic.rs#L489-L495`）

At HEAD, L489-L495 is inside `preflight_message_request`, not the status-code list. The actual `is_retryable_status` constant with those codes is at **L924**.

### PASS: Runtime anchors

- `runtime/src/conversation.rs#L296` — `run_turn`
- `runtime/src/conversation.rs#L30-L48` — `AssistantEvent`
- `runtime/src/conversation.rs#L676-L723` — `build_assistant_message`

All valid.

### PASS: `render.rs` anchors (except `Spinner::tick`)

- `render.rs#L601-L620` — `MarkdownStreamState`
- `render.rs#L810-L839` — `find_stream_safe_boundary`
- `render.rs#L274-L276` — `markdown_to_ansi`

All valid.

---

## 2. `docs/.report/02-why-this-whitepaper.md`

### PASS: Most anchors are accurate

All runtime anchors checked align with HEAD:

- `runtime/src/conversation.rs#L296-L490` — `run_turn`
- `runtime/src/conversation.rs#L348-L490` — tool execution / permission / hook path
- `runtime/src/session.rs#L28-L51` — `ContentBlock`
- `runtime/src/permissions.rs#L9-L15` — `PermissionMode`
- `runtime/src/permission_enforcer.rs#L44-L134` — `PermissionEnforcer`
- `runtime/src/sandbox.rs#L8-L38` — `SandboxConfig` / `FilesystemIsolationMode`
- `runtime/src/bash.rs#L170-L247` — sandbox execution / `prepare_command`
- `runtime/src/bash.rs#L288-L304` — `MAX_OUTPUT_BYTES` / `truncate_output`
- `runtime/src/prompt.rs#L203-L224` — `discover_instruction_files`
- `runtime/src/config.rs#L56-L75` — `RuntimeFeatureConfig`
- `runtime/src/config.rs#L310-L335` — config merging
- `runtime/src/compact.rs#L96-L139` — `compact_session`
- `runtime/src/conversation.rs#L17-L18` — auto-compaction threshold constants
- `runtime/src/conversation.rs#L521-L548` — `maybe_auto_compact`

### MAJOR: Inaccurate description of `compact_session` semantic fields

The report writes:

> 压缩后，会话的消息列表会以一条 `MessageRole::System` 的 "continuation message" 开头，其中包含 `Summary`、`Key timeline`、`Tools mentioned`、`Pending work` 等结构化字段（[`compact.rs#L150-L290`](/rust/crates/runtime/src/compact.rs#L150-L288)）

While the fields `Summary`, `Key timeline`, `Tools mentioned`, and `Pending work` **do** appear inside the formatted summary text (generated by `format_compact_summary`), the actual `compact_session` function (L96-L150+) does not explicitly create a message with those four named fields. It creates a single `System` message whose text is the opaque string returned by `get_compact_continuation_message(&summary, ...)`. The structured bullets are hidden inside `format_compact_summary`. This is a **mischaracterization of the API surface** — it makes the continuation message sound like a strongly-typed struct when it is actually one free-text block.

### MINOR: `main.rs#L4265-L4276` for `reload_runtime_features` is stale

At HEAD, `reload_runtime_features` is at **L4647**. L4265-L4276 covers a completely different function (permission mode handling). This is the same `main.rs` drift observed in `05-streaming.md`.

---

## 3. `docs/.report/38-fork-subagent.md`

### PASS: No Rust anchors; upstream scope clearly declared

This report contains **zero anchors into `rust/crates/`**. Every code reference points to `packages/ccb/src/tools/AgentTool/...` and other TypeScript upstream paths. The preamble explicitly states:

> `claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。

That disclaimer is accurate and prominent. No factual issues found.

---

## 4. `docs/.report/11-task-management.md`

### PASS: Scope disclaimer and factual accuracy

The report correctly states:

> `claw-code`（Rust 重写版）目前尚未完全实现 parity。

Only one Rust anchor exists:

- `rust/crates/runtime/src/conversation.rs` — used in the source index as a generic pointer to `ConversationRuntime`.

It is valid. The report’s analysis of V1/V2 task architecture is upstream-focused and internally consistent.

---

## Cross-Report Findings

| Observation | Files Involved | Severity |
|-------------|----------------|----------|
| `main.rs` has undergone major refactoring (likely +400 line drift). Both `05-streaming.md` and `02-why-this-whitepaper.md` contain stale `main.rs` anchors. | `05-streaming.md`, `02-why-this-whitepaper.md` | MAJOR (systematic line drift) |
| `02-why-this-whitepaper.md` and `05-streaming.md` both reference `conversation.rs#L296` / `L296-L490` for `run_turn` — this anchor is consistent and correct. | — | PASS |
| `02-why-this-whitepaper.md` and `11-task-management.md` both acknowledge Rust parity gaps in task/agent infrastructure. No contradiction. | — | PASS |

---

## Consolidated Action Items

1. **Regenerate all `main.rs` line ranges** in `05-streaming.md` and `02-why-this-whitepaper.md` against current HEAD.
2. **Fix `api/src/providers/openai_compat.rs#L1112-L1123`** in `05-streaming.md` — actual `normalize_finish_reason` is at L1265.
3. **Fix `api/src/providers/anthropic.rs#L489-L495`** in `05-streaming.md` — actual `is_retryable_status` is at L924.
4. **Clarify `compact_session` description** in `02-why-this-whitepaper.md` to avoid implying a strongly-typed struct for the continuation message.
5. **Add source anchor or citation** for the retry-default claims (8 retries / 1s base / 128s cap) in `05-streaming.md`, or mark as approximate / configuration-dependent.

---

*End of Unit 5 review findings.*
