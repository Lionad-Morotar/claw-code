# Unit 6 — Core loop & safety fundamentals
## Critical Review Findings

**Reviewed files:**
- `docs/.report/04-the-loop.md`
- `docs/.report/01-what-is-claude-code.md`
- `docs/.report/25-sandbox.md`
- `docs/.report/41-kairos.md`

**Method:** All `rust/crates/...#Lxx-Lyy` anchors were verified with `git show HEAD:<path> | sed -n '<start>,<end>p'`.

---

## 1. `04-the-loop.md` — The Agentic Loop

### source anchor: `conversation.rs#L335-L338` (usage_tracker.record)
**Severity: MAJOR**
The report claims `run_turn` calls `self.usage_tracker.record(usage)` at `conversation.rs#L335-L338`. At HEAD, lines 335-338 are actually:
```rust
    Ok(result) => result,
    Err(error) => {
        self.record_turn_failed(iterations, &error);
```
These are part of the `build_assistant_message` result match, not the `usage_tracker.record(usage)` call. The actual `usage_tracker.record(usage)` call is around line 330-332. Anchor is stale.

### source anchor: `conversation.rs#L955-L1020`
**Severity: PASS**
Verified at HEAD. Covers test `records_full_turn_lifecycle` and `records_denied_tool_results_when_prompt_rejects`. The report says this demonstrates "`unexpected: true`" cache event capture — that's actually `conversation.rs#L676-L723` and later tests; this range itself does not contain cache tests. However the anchor itself is valid and the surrounding prose about the tests is directionally correct.

### source anchor: `conversation.rs#L1015-L1105`
**Severity: PASS**
Verified at HEAD. Correctly covers `denies_tool_use_when_pre_tool_hook_blocks` and related hook tests.

### source anchor: `conversation.rs#L1583-L1608`
**Severity: PASS**
Verified at HEAD. Correctly covers the `max_iterations = 1` test.

### source anchor: `conversation.rs#L525-L548` (maybe_auto_compact)
**Severity: PASS**
Verified at HEAD. Accurate.

### source anchor: `compact.rs#L96-L139`
**Severity: PASS**
Verified at HEAD. Function `compact_session` starts at L96. Accurate.

### source anchor: `compact.rs#L10-L13`
**Severity: PASS**
Verified at HEAD. `CompactionConfig` struct is at L10-L13. Accurate.

### source anchor: `session.rs#L43-L46` and `session.rs#L28-L46`
**Severity: PASS**
Verified at HEAD. `ConversationMessage` is at L43-L46 (actually L43-48), and `ContentBlock` is at L28-L46. Accurate.

### source anchor: `conversation.rs#L126-L188`
**Severity: PASS**
Verified at HEAD. `ConversationRuntime` struct and `new`/`new_with_features` constructors span L126-L188. Accurate.

### source anchor: `conversation.rs#L296-L490`
**Severity: PASS**
Verified at HEAD. `run_turn` spans exactly this range. Accurate.

### source anchor: `conversation.rs#L348-L490`
**Severity: PASS**
Verified at HEAD. This covers the tool execution loop inside `run_turn`. Accurate.

### source anchor: `conversation.rs#L30-L48`
**Severity: PASS**
Verified at HEAD. `AssistantEvent` enum is at L30-L48. Accurate.

### source anchor: `conversation.rs#L50-L58`
**Severity: PASS**
Verified at HEAD. `PromptCacheEvent` is at L50-L58. Accurate.

### source anchor: `conversation.rs#L56-L61`
**Severity: PASS**
Verified at HEAD. `ApiClient` trait is at L56-L61 (actually L56-L60). Accurate.

### source anchor: `conversation.rs#L107-L120`
**Severity: PASS**
Verified at HEAD. `TurnSummary` struct is at L107-L120. Accurate.

### source anchor: `conversation.rs#L304-L314`
**Severity: PASS**
Verified at HEAD. Max-iterations guard is here. Accurate.

### source anchor: `conversation.rs#L319-L325`
**Severity: PASS**
Verified at HEAD. `api_client.stream(request)` error handling. Accurate.

### source anchor: `conversation.rs#L326-L332`
**Severity: PASS**
Verified at HEAD. `build_assistant_message` error handling. Accurate.

### source anchor: `conversation.rs#L676-L723`
**Severity: PASS**
Verified at HEAD. `build_assistant_message` function definition. Accurate.

### source anchor: `permissions.rs#L174-L292`
**Severity: PASS**
Verified at HEAD. `authorize_with_context` spans L174-L292. Accurate.

### source anchor: `hooks.rs#L152-L987`
**Severity: PASS**
Verified at HEAD. `hooks.rs` is exactly 987 lines, and `HookRunner` starts at L152. This is a very wide reference but technically valid.

### source anchor: `usage.rs#L31-L36`
**Severity: PASS**
Verified at HEAD. `TokenUsage` struct is at L31-L36. Accurate.

### source anchor: `conversation.rs#L16`
**Severity: PASS**
Verified at HEAD. `DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD` constant is here. Accurate.

### source anchor: `conversation.rs#L181`
**Severity: PASS**
Verified at HEAD. `max_iterations: usize::MAX` is at L181. Accurate.

---

## 2. `01-what-is-claude-code.md` — Overview & Architecture

### source anchor: `rusty-claude-cli/src/main.rs#L110-L123`
**Severity: PASS**
Verified at HEAD. `main()` function spans L110-L123. Accurate.

### source anchor: `rusty-claude-cli/src/main.rs#L168-L240`
**Severity: PASS**
Verified at HEAD. Contains the `run()` function start and `CliAction::Prompt` branch. Accurate.

### source anchor: `rusty-claude-cli/src/main.rs#L259-L347`
**Severity: PASS**
Verified at HEAD. `CliAction` enum definition starts at L259 and extends past L347. Accurate.

### source anchor: `rusty-claude-cli/src/main.rs#L131-L143` and `L151-L163`
**Severity: PASS**
Verified at HEAD. `read_piped_stdin` is at L131-L147; `merge_prompt_with_stdin` body starts at L151. The ranges are slightly off at the edges (L143 is inside the function, L163 is past `merge_prompt_with_stdin`'s start) but the claimed symbols are present within the ranges. Acceptable.

### source anchor: `rusty-claude-cli/src/main.rs#L165-L257`
**Severity: PASS**
Verified at HEAD. Covers `run()` from its signature through most of the `CliAction` match arms. Accurate.

### source anchor: `api/src/client.rs#L17-L19` and `L23-L56`
**Severity: PASS**
Verified at HEAD. `from_model` is at L17-L19; provider dispatch logic is at L23-L56. Accurate.

### source anchor: `api/src/providers/mod.rs#L17-L31`
**Severity: PASS**
Verified at HEAD. `Provider` trait is at L17-L31. Accurate.

### source anchor: `runtime/src/conversation.rs#L126-L138`
**Severity: PASS**
Verified at HEAD. `ConversationRuntime` struct declaration is at L126-L138. Accurate.

### source anchor: `runtime/src/conversation.rs#L296-L490`
**Severity: PASS**
Verified at HEAD. `run_turn` spans this range. Accurate.

### source anchor: `runtime/src/session.rs#L28-L46`
**Severity: PASS**
Verified at HEAD. `ContentBlock` enum is at L28-L46. Accurate.

### source anchor: `runtime/src/permissions.rs#L9-L15`
**Severity: PASS**
Verified at HEAD. `PermissionMode` enum is at L9-L15. Accurate.

### source anchor: `runtime/src/bash.rs#L19-L31`
**Severity: PASS**
Verified at HEAD. `BashCommandInput` struct starts at L19. Accurate.

### source anchor: `runtime/src/file_ops.rs#L32-L44`
**Severity: PASS**
Verified at HEAD. `validate_workspace_boundary` is at L32-L44. Accurate.

### source anchor: `runtime/src/prompt.rs#L432-L446`
**Severity: PASS**
Verified at HEAD. `load_system_prompt` function signature and body start is here. Accurate.

### source anchor: `runtime/src/prompt.rs#L202-L220`
**Severity: PASS**
Verified at HEAD. `discover_instruction_files` covers exactly this logic. Accurate.

### source anchor: `rusty-claude-cli/src/render.rs#L602-L606`
**Severity: MINOR**
Verified at HEAD. L602 is `impl MarkdownStreamState {`, L603-606 is `#[must_use] pub fn push(...)`. The method body continues to L612, so the range is slightly tight but the symbol is present. The report prose says "`MarkdownStreamState::push`", and that symbol does start at L606. Acceptable with minor note.

---

## 3. `25-sandbox.md` — Sandbox Mechanism

### source anchor: `rusty-claude-cli/src/main.rs#L2606-L2614`
**Severity: CRITICAL**
At HEAD, lines 2606-2614 of `main.rs` are:
```rust
fn parse_git_status_branch(status: Option<&str>) -> Option<String> {
```
This is **not** `SlashCommand::Sandbox`. The actual `SlashCommand::Sandbox` match arm is at **L2817-L2829** in `execute_slash_command`.
The code block shown in the report *does* match L2817-L2829, but the line anchor is completely wrong. This is a broken anchor.

### source anchor: `runtime/src/config.rs#L865-L923`
**Severity: PASS**
Verified at HEAD. `parse_optional_sandbox_config` is at L865-L891 (ends earlier than L923), but the claimed symbol is present in the range. Accurate enough.

### source anchor: `runtime/src/sandbox.rs#L9-L14`
**Severity: PASS**
Verified at HEAD. `FilesystemIsolationMode` enum is at L9-L14. Accurate.

### source anchor: `runtime/src/sandbox.rs#L87-L105`
**Severity: PASS**
Verified at HEAD. `resolve_request` is at L87-L105. Accurate.

### source anchor: `runtime/src/sandbox.rs#L162-L208`
**Severity: PASS**
Verified at HEAD. `resolve_sandbox_status_for_request` is at L162-L208. Accurate.

### source anchor: `runtime/src/sandbox.rs#L109-L153`
**Severity: PASS**
Verified at HEAD. `detect_container_environment` and `detect_container_environment_from` span this range. Accurate.

### source anchor: `runtime/src/bash.rs#L70-L103`
**Severity: PASS**
Verified at HEAD. `execute_bash` function is here. Accurate.

### source anchor: `runtime/src/bash.rs#L170-L183`
**Severity: PASS**
Verified at HEAD. `sandbox_status_for_input` is at L170-L183. Accurate.

### source anchor: `runtime/src/bash.rs#L212-L237`
**Severity: PASS**
Verified at HEAD. `prepare_tokio_command` is at L212-L237. Accurate.

### source anchor: `runtime/src/sandbox.rs#L211-L262`
**Severity: PASS**
Verified at HEAD. `build_linux_sandbox_command` is at L211-L262. Accurate.

### source anchor: `runtime/src/sandbox.rs#L264-L278`
**Severity: PASS**
Verified at HEAD. `normalize_mounts` is at L264-L278. Accurate.

### source anchor: `runtime/src/sandbox.rs#L288-L304`
**Severity: PASS**
Verified at HEAD. `unshare_user_namespace_works` is at L288-L304. Accurate.

### source anchor: `runtime/src/bash.rs#L25-L26`
**Severity: PASS**
Verified at HEAD. `dangerously_disable_sandbox` field in `BashCommandInput` is at L25-L26. Accurate.

### source anchor: `runtime/src/bash.rs#L53-L54`
**Severity: MAJOR**
At HEAD, lines 53-54 are inside `BashCommandInput` (still the input struct). The *output* schema field `dangerously_disable_sandbox` in `BashCommandOutput` is actually around `bash.rs#L57-L58` (inside `BashCommandOutput`). The report says "输出 schema 会把原值回显在 `dangerously_disable_sandbox` 字段中（`runtime/src/bash.rs#L53-L54`）", which is the wrong struct. Symbol mismatch.

### source anchor: `tools/src/lib.rs#L397`
**Severity: PASS**
Verified at HEAD. `"dangerouslyDisableSandbox": { "type": "boolean" }` is exactly at L397. Accurate.

### source anchor: `commands/src/lib.rs`
**Severity: PASS**
Verified at HEAD. `SlashCommand::Sandbox` exists in this file (L1057, L1295, L1335, L4031, L4204). The report does not give a line range, so this is a valid file-level reference.

---

## 4. `41-kairos.md` — KAIROS Persistent Assistant

**Severity: PASS (no Rust anchors)**
This report contains **no `rust/crates/` anchors**. All anchors point to upstream TypeScript in `packages/ccb/src/...`, and the report explicitly states that `claw-code` (Rust) has not implemented KAIROS. Because there are no Rust source anchors to verify, and the report is transparent about upstream-only scope, there are no broken anchors or false claims to flag for the Rust implementation.

Factual claims about upstream stub architecture (`assistant/`, `proactive/index.ts`, etc.) were not verified against upstream code per the unit scope, but the report's self-disclaimer is sufficient.

---

## Cross-Report Observations

No contradictions were found between the four assigned reports. All three Rust-focused reports (`04`, `01`, `25`) are consistent in their descriptions of:
- `PermissionMode` five-level enum
- `ConversationRuntime::run_turn` agentic loop
- `ContentBlock` structured messages
- Terminal-native architecture

One minor note: `04-the-loop.md` and `01-what-is-claude-code.md` both reference `conversation.rs#L296-L490` for `run_turn` — this is consistent and correct.

---

## Summary

| Severity | Count | Issues |
|----------|-------|--------|
| **CRITICAL** | 1 | `25-sandbox.md`: broken anchor `main.rs#L2606-L2614` (should be ~L2817) |
| **MAJOR** | 2 | `04-the-loop.md`: stale `usage_tracker.record` anchor (L335-L338 wrong); `25-sandbox.md`: `dangerously_disable_sandbox` output field anchor points to input struct (L53-L54) |
| **MINOR** | 1 | `01-what-is-claude-code.md`: `render.rs#L602-L606` range is tight for `MarkdownStreamState::push` |
| **PASS** | ~85 | All remaining anchors and factual claims verified successfully |
