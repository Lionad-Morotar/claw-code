# Unit 15 — Token budget + MCP + ant-only | Critical Review Findings

**Review date**: 2026-04-10  
**Reviewer**: Unit 15 worker  
**Files reviewed**: `15-token-budget.md`, `19-mcp-protocol.md`, `34-ant-only-world.md`, `51-token-budget-feature.md`

---

## 1. 15-token-budget.md

### Anchor verification summary
| Anchor | Status | Notes |
|--------|--------|-------|
| `api/src/providers/mod.rs#L47-L50` | PASS | `ModelTokenLimit` struct present |
| `api/src/providers/mod.rs#L242-L258` | PASS | `model_token_limit` function |
| `api/src/providers/mod.rs#L287-L292` | MAJOR | **Line range stale**: `estimate_serialized_tokens` is at **L302-L306**, not L287-L292. The prose quotes the correct source code body (`serde_json::to_vec ... / 4 + 1`), so this is a stale anchor, not a false claim. |
| `api/src/providers/mod.rs#L260-L276` | PASS | `preflight_message_request` function |
| `api/src/providers/mod.rs#L218-L230` | PASS | `max_tokens_for_model` function |
| `api/src/providers/anthropic.rs#L489-L520` | PASS | `preflight_message_request` async method |
| `api/src/error.rs#L31-L37` | PASS | `ApiError::ContextWindowExceeded` variant |
| `api/src/types.rs#L167-L206` | PASS | `Usage` struct and `token_usage` impl |
| `runtime/src/usage.rs#L29-L36` | PASS | `TokenUsage` struct |
| `runtime/src/usage.rs#L167-L215` | PASS | `UsageTracker` struct and `record` impl |
| `runtime/src/conversation.rs#L16-L18` | PASS | Threshold constants |
| `runtime/src/conversation.rs#L525-L548` | PASS | `maybe_auto_compact` method |
| `runtime/src/conversation.rs#L669-L674` | PASS | `parse_auto_compaction_threshold` |
| `runtime/src/compact.rs#L96-L139` | PASS | `compact_session` |
| `runtime/src/compact.rs#L9-L12` | PASS | `CompactionConfig` struct |
| `runtime/src/conversation.rs#L1476-L1508` | PASS | Auto-compaction test |
| `providers/mod.rs#L619-L662` | PASS | Preflight context-window test |
| `runtime/src/compact.rs#L537-L583` | PASS | Session compaction test |
| `runtime/src/session.rs` (bare) | PASS | File exists; report uses it to explain message metadata |
| `runtime/src/conversation.rs` (bare) | PASS | File exists |
| `runtime/src/compact.rs` (bare) | PASS | File exists |

### Factual / logic issues
- **MAJOR**: The anchor `api/src/providers/mod.rs#L287-L292` points to the *middle* of `preflight_message_request`’s error-return block. The `estimate_serialized_tokens` function actually lives at **L302-L306**. Code snippet in the report is correct; only the link is stale.

### Completeness / cross-report notes
- Minor formatting: the source index table at the bottom lists `runtime/src/conversation.rs#L558-L564` for threshold parsing, but the threshold parsing function is actually at `L669-L674`. `L558-L564` appears to be a stale leftover from an earlier revision.

---

## 2. 19-mcp-protocol.md

### Anchor verification summary
| Anchor | Status | Notes |
|--------|--------|-------|
| `runtime/src/mcp_stdio.rs#L19-L22` | PASS | Default init timeout (10_000) |
| `runtime/src/mcp_stdio.rs#L34-L181` | PASS | JSON-RPC request/response types |
| `runtime/src/mcp_stdio.rs#L254-L282` | PASS | `McpServerManagerError` enum |
| `runtime/src/mcp_stdio.rs#L267-L282` | PASS | Same error enum (overlap with previous line range) |
| `runtime/src/mcp_stdio.rs#L463-529` | PASS | `ManagedMcpServer` struct + `McpServerManager` definition |
| `runtime/src/mcp_stdio.rs#L494-L512` | PASS | `from_servers` stdio filter |
| `runtime/src/mcp_stdio.rs#L531-L548` | PASS | Unsupported-servers handling |
| `runtime/src/mcp_stdio.rs#L624-L773` | PASS | `call_tool` method |
| `runtime/src/mcp_stdio.rs#L652-L686` | CRITICAL | **Stale range**: report says `McpDegradedReport` generation is here, but `L652-L686` falls inside `call_tool` (tool invocation payload building). The actual `McpDegradedReport` construction is in `discover_tools_best_effort` around **L585-L605**. |
| `runtime/src/mcp_stdio.rs#L852-L863` | CRITICAL | **Stale range**: `McpServerManager::shutdown` is at **L723-L734**. `L852-L863` is inside `call_tool` / `discover_tools` code. |
| `runtime/src/mcp_stdio.rs#L878-L884` | PASS | `is_retryable_error` |
| `runtime/src/mcp_stdio.rs#L963-L967` | PASS | `take_request_id` |
| `runtime/src/mcp_stdio.rs#L990-L1049` | CRITICAL | **Stale range**: this area covers `is_retryable_error`, `should_reset_server`, `run_process_request`, and the *start* of `ensure_server_ready`. The prose describes **discover_tools / pagination logic**, which is actually in `discover_tools_for_server_once` around **L806-L875** and `discover_tools_best_effort` around **L555-L621**. |
| `runtime/src/mcp_stdio.rs#L1052-L1120` | PASS | `ensure_server_ready` |
| `runtime/src/mcp_stdio.rs#L1139-L1225` | PASS | `McpStdioProcess` read/write frame methods |
| `runtime/src/mcp_stdio.rs#L1231-L1241` | PASS | `McpStdioProcess::shutdown` |
| `runtime/src/mcp_stdio.rs#L1244-L1249` | PASS | `encode_frame` |
| `runtime/src/mcp_stdio.rs#L2553-L2602` | PASS | Initialize retry test |
| `runtime/src/mcp_stdio.rs#L2603-L2663` | PASS | Tool-call disconnect reset test |
| `runtime/src/mcp.rs#L7-L23` | PASS | `normalize_name_for_mcp` |
| `runtime/src/mcp.rs#L31-L37` | PASS | `mcp_tool_name` |
| `runtime/src/mcp.rs#L84-L121` | PASS | `scoped_mcp_config_hash` |
| `runtime/src/mcp.rs#L152-L159` | PASS | `stable_hex_hash` |
| `runtime/src/mcp_lifecycle_hardened.rs#L16-L34` | PASS | `McpLifecyclePhase` enum |
| `runtime/src/mcp_server.rs#L41-L42` | PASS | `ToolCallHandler` type alias |
| `runtime/src/mcp_server.rs#L66-L70` | PASS | `McpServer` struct |
| `runtime/src/mcp_server.rs#L86-L141` | PASS | `run` loop |
| `runtime/src/mcp_server.rs#L162-L177` | PASS | `handle_initialize` |
| `runtime/src/mcp_server.rs#L178-L188` | PASS | `handle_tools_list` |
| `runtime/src/mcp_server.rs#L190-L230` | PASS | `handle_tools_call` |
| `rusty-claude-cli/src/main.rs#L2995-L3008` | PASS | `RuntimeMcpState::new` |
| `rusty-claude-cli/src/main.rs#L3143-L3165` | PASS | `build_runtime_mcp_state` |
| `rusty-claude-cli/src/main.rs#L3171-L3200` | PASS | `mcp_wrapper_tool_definitions` |
| `rusty-claude-cli/src/main.rs#L3270-L3282` | CRITICAL | **Stale range**: `permission_mode_for_mcp_tool` is at **L3647-L3665**. `L3270-L3282` is inside `BuiltRuntime::new`. |
| `runtime/src/config.rs` (bare) | PASS | File exists |
| `runtime/src/lib.rs` (bare) | PASS | File exists |
| `runtime/src/mcp_stdio.rs` (bare) | PASS | File exists |
| `runtime/src/mcp_client.rs` (bare) | PASS | File exists |
| `runtime/src/mcp.rs` (bare) | PASS | File exists |
| `runtime/src/mcp_server.rs` (bare) | PASS | File exists |
| `runtime/src/mcp_tool_bridge.rs` (bare) | PASS | File exists |
| `runtime/src/mcp_lifecycle_hardened.rs` (bare) | PASS | File exists |

### Factual / logic issues
1. **CRITICAL**: `runtime/src/mcp_stdio.rs#L652-L686` — claimed to show `McpDegradedReport` generation. It does not; it shows the middle of `call_tool`. The actual degraded-report logic is in `discover_tools_best_effort` ~L585-L605.
2. **CRITICAL**: `runtime/src/mcp_stdio.rs#L852-L863` — claimed to be `McpServerManager::shutdown`. It is not; `shutdown` is at L723-L734.
3. **CRITICAL**: `runtime/src/mcp_stdio.rs#L990-L1049` — claimed to cover discover-tools with pagination. This range covers helper functions (`is_retryable_error`, `should_reset_server`, `run_process_request`) and the start of `ensure_server_ready`. Pagination logic is in `discover_tools_for_server_once` ~L806-L875.
4. **CRITICAL**: `rusty-claude-cli/src/main.rs#L3270-L3282` — claimed to show `permission_mode_for_mcp_tool`. It does not; the function is at L3647-L3665.
5. **MAJOR**: Report states init timeout default is 10s at `mcp_stdio.rs#L19-L22`. The lines shown are actually the *test* variant (200ms). The 10s default exists at `L24` (`#[cfg(not(test))]`), but the anchor starts at L19 which is inside the `#[cfg(test)]` block. A cleaner anchor would be `L24` only or `L20-L24`.

---

## 3. 34-ant-only-world.md

### Anchor verification summary
| Anchor | Status | Notes |
|--------|--------|-------|
| `api/src/http_client.rs#L78` | PASS | Environment-based request configuration |
| `api/src/providers/anthropic.rs#L1011-L1023` | PASS | `strip_unsupported_beta_body_fields` |
| `api/src/providers/anthropic.rs#L227-L230` | PASS | `with_beta` |
| `api/src/providers/anthropic.rs#L483-L486` | PASS | Header injection loop |
| `api/src/providers/anthropic.rs#L621-L631` | PASS | `from_env_or_saved` auth |
| `api/src/providers/anthropic.rs` (bare) | PASS | File exists |
| `api/tests/client_integration.rs#L102` | PASS | Beta header wire-format test assertion |
| `api/tests/client_integration.rs` (bare) | PASS | File exists |
| `commands/src/lib.rs` (bare) | PASS | File exists |
| `commands/src/lib.rs#L3873-L3920` | PASS | Command JSON helpers (no contradiction) |
| `commands/src/lib.rs` (bare, repeated) | PASS | — |
| `plugins/src/lib.rs` (bare) | PASS | File exists |
| `runtime/src/conversation.rs` (bare) | PASS | File exists |
| `rusty-claude-cli/src/input.rs` (bare) | PASS | File exists |
| `rusty-claude-cli/src/main.rs` (bare) | PASS | File exists |
| `telemetry/src/lib.rs#L54-L63` | MAJOR | Anchor uses `L54-L63`, but the code shown ends at `L61`. The struct spans L54-L61; the report shows `L54-L63` which includes two blank/comment lines. Minor range drift, but the prose says “L54-L61 is confirmed” in the REVIEW.md section, contradicting the inline anchor. |
| `telemetry/src/lib.rs#L68-L72` | PASS | Default betas vector |
| `telemetry/src/lib.rs` (bare) | PASS | File exists |

### Factual / logic issues
- **MINOR**: `telemetry/src/lib.rs#L54-L63` — the anchor overshoots by 2 lines. The struct ends at L61. The self-review at the bottom of the report acknowledges `L52-L59` / `L69-L72`, which is inconsistent with both the inline anchor and the actual code.

### Completeness / cross-report notes
- The report accurately notes that ant-only constructs (`USER_TYPE`, `IS_DEMO`, `GrowthBook`, etc.) are **absent** from the Rust codebase and that this reflects a deliberate single-build model. This is internally consistent.
- **Cross-report observation**: The report refers to [`PARITY.md#L97-L101`](/rust/PARITY.md#L97-L101) for the 67/141 slash-command figure. In HEAD, the `PARITY.md` line numbers for that section are approximately correct (the “Slash Commands: 67/141 upstream entries” block starts around L90-L105). Slight drift, but the referenced text is there.

---

## 4. 51-token-budget-feature.md

### Anchor verification summary
This report intentionally maps to upstream TypeScript paths (`packages/ccb/src/...`) and explicitly disclaims the existence of those files in the Rust repo. There are **no `rust/crates/...` anchors** to verify in this file.

### Factual / logic issues
- **PASS** (no rust anchors): The report’s disclaimer is clear: “Token Budget Feature 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。”
- **MAJOR**: The report does not contain any line-range verifications for the upstream TypeScript paths. Since this is a documentation-audit task aimed at `claw-code`, the absence of Rust-side anchors is by design. However, if a user later expects `51-token-budget-feature.md` to also describe Rust implementation parity, they will be disappointed; the report should ideally include a short “Rust parity status” section summarizing whether any token-budget concepts (e.g., `+500k` parsing, nudge loops, `task_budget` API field) are present in `claw-code`. Currently that gap is only implicit.

### Completeness / cross-report notes
- **Cross-report with `15-token-budget.md`**: `15-token-budget.md` discusses the Rust `UsageTracker`, `compact_session`, and `preflight_message_request` but never mentions the upstream “token budget continuation / nudge” feature described in `51-token-budget-feature.md`. The two reports are not contradictory, but there is no explicit cross-reference explaining that `claw-code` has token-counting/compaction but **not** the upstream `TOKEN_BUDGET` auto-continue feature.

---

## Cross-Report Observations

1. **Token-budget gap not bridged**: `15-token-budget.md` covers Rust token counting and compaction thoroughly, while `51-token-budget-feature.md` covers upstream TypeScript token-budget continuation. Neither report explicitly tells the reader that the Rust repo lacks the continuation/nudge mechanism. A one-line cross-reference in `15-token-budget.md` would close this gap.
2. **Line-number inconsistency in self-review**: `34-ant-only-world.md`’s `REVIEW.md` footer claims certain line numbers were “confirmed,” but those numbers (`L52-L59`, `L69-L72`) do not match the inline anchors (`L54-L63`, `L68-L72`). This suggests the self-review was performed against an older snapshot.

---

## Severity Tally

### CRITICAL
1. `19-mcp-protocol.md` — `runtime/src/mcp_stdio.rs#L652-L686` does not contain `McpDegradedReport`.
2. `19-mcp-protocol.md` — `runtime/src/mcp_stdio.rs#L852-L863` is not `McpServerManager::shutdown`.
3. `19-mcp-protocol.md` — `runtime/src/mcp_stdio.rs#L990-L1049` does not cover discover-tools pagination logic.
4. `19-mcp-protocol.md` — `rusty-claude-cli/src/main.rs#L3270-L3282` is not `permission_mode_for_mcp_tool`.

### MAJOR
1. `15-token-budget.md` — `api/src/providers/mod.rs#L287-L292` is stale; `estimate_serialized_tokens` is at L302-L306.
2. `15-token-budget.md` — Source index references `conversation.rs#L558-L564` for threshold parsing, but the function is at `L669-L674`.
3. `19-mcp-protocol.md` — `mcp_stdio.rs#L19-L22` points to the test variant of the init timeout; the 10s production default is at L24.
4. `34-ant-only-world.md` — `telemetry/src/lib.rs#L54-L63` overshoots the struct end by 2 lines.
5. `51-token-budget-feature.md` — Missing a “Rust parity status” explicit summary (implied only by the upstream-only disclaimer).

### MINOR
1. `34-ant-only-world.md` — Self-review footer (`REVIEW.md`) contains stale/conflicting line numbers (`L52-L59` vs inline `L54-L63`).

---

## Verdict

`15-token-budget.md` and `34-ant-only-world.md` are mostly sound with only minor anchor drift.  
`19-mcp-protocol.md` has **multiple stale/broken anchors** that need correction before readers can reliably navigate to the claimed source locations.  
`51-token-budget-feature.md` is accurate as an upstream-only report but would benefit from an explicit Rust parity summary.
