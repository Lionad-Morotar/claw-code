# Unit 14 Critical Review Findings

**Review scope**: `09-shell-execution.md`, `06-multi-turn.md`, `52-context-collapse.md`, `55-tier3-stubs.md`  
**Method**: static source-code alignment against current HEAD + working-tree files  
**Date**: 2026-04-10

---

## 1. `09-shell-execution.md`

### CRITICAL

1. **Broken anchor — `describe_tool_progress`**
   - Claimed: `rust/crates/rusty-claude-cli/src/main.rs#L6169-L6184`
   - Actual: lines 6169-6184 are `export` markdown-transcript writing code.
   - `fn describe_tool_progress()` is at **line 6666**.
   - The code block shown in the report *does* match the real function, but the line anchor is off by ~500 lines.

2. **Broken anchor — CLI second truncation**
   - Claimed: `rust/crates/rusty-claude-cli/src/main.rs#L7142-L7157` for "CLI 渲染层的第二次截断（按显示行数）"
   - Actual: lines 7142-7157 are inside API stream `ContentBlockDelta` event handling (`InputJsonDelta`, `ThinkingDelta`, `SignatureDelta`).
   - The display-line truncation function `truncate_output_for_display` and its Bash-tool call sites are around **lines 7758-7986**.

### MINOR

3. **Line-range mismatch in `bash_validation.rs`**
   - Markdown link text says `bash_validation.rs#L99-L131`, but the source anchor URL says `...#L99-L130`.
   - The `validate_read_only` function starts at line 99; line 131 does not exist inside the function (it ends earlier). The anchor should be `L99-L130` consistently.

4. **Incomplete code snippet in `sandbox.rs`**
   - The quote for `sandbox.rs#L211-L255` stops mid-function and omits the final `Some(LinuxSandboxCommand { program, args, env })` return statement (line 256+). The anchor itself is acceptable, but the block is truncated.

---

## 2. `06-multi-turn.md`

### CRITICAL

5. **Broken anchor — `set_model()`**
   - Claimed: `rust/crates/rusty-claude-cli/src/main.rs#L3831-L3878`
   - Actual: these lines contain `build_runtime`, `replace_runtime`, and `run_turn`.
   - `fn set_model()` starts at **line 4213**.

6. **Broken anchor — `clear_session()`**
   - Claimed: `rust/crates/rusty-claude-cli/src/main.rs#L3926-L3959`
   - Actual: these lines build the JSON output block including `"estimated_cost"` formatting.
   - `fn clear_session()` starts at **line 4308**.

7. **Broken anchor — `print_cost()`**
   - Claimed: `rust/crates/rusty-claude-cli/src/main.rs#L3961-L3964`
   - Actual: line 3961-3964 is `SlashCommand::Status => { self.print_status(); ... }`.
   - `fn print_cost()` is at **line 4343**.

8. **Broken anchor — `resume_session()`**
   - Claimed: `rust/crates/rusty-claude-cli/src/main.rs#L3966-L4004`
   - Actual: these lines are REPL `SlashCommand` match arms (`Bughunter`, `Commit`, `Pr`, `Issue`, `Ultraplan`, etc.).
   - `fn resume_session()` starts at **line 4348**.

9. **Broken anchor — `estimated_cost` JSON output**
   - Claimed: `rust/crates/rusty-claude-cli/src/main.rs#L3565-L3572`
   - Actual: these lines initialize `runtime_tools` from MCP discovery.
   - The `"estimated_cost"` field formatting is around **lines 3938-3948**.

10. **Broken anchor — `append_persisted_message()`**
    - Claimed: `rust/crates/runtime/src/session.rs#L517-L531`
    - Actual: lines 517-531 are inside `render_jsonl_snapshot()` (the `lines.extend(...)` calls).
    - `fn append_persisted_message()` starts at **line 533**.
    - The code block shown in the report *does* match `append_persisted_message`, but the anchor is ~16 lines too early.

### MAJOR

11. **Stale struct excerpt — `Session` missing `model` field**
    - Claimed: `session.rs#L88-L99` shows the complete `Session` struct ending with `persistence: Option<SessionPersistence>`.
    - Actual: line 99 is the start of a doc comment for the **`model`** field (`pub model: Option<String>`), which appears on line 101. The struct closes around line 103.
    - The report silently drops a persisted field that affects model-resume behavior.

### MINOR

12. **Imprecise end line for `run_turn()`**
    - Claimed: `conversation.rs#L296-L490`
    - Actual: `pub fn run_turn` ends at line 485; line 490 is already inside `pub fn compact`. The anchor overruns by ~5 lines.

---

## 3. `52-context-collapse.md`

### PASS

- All source anchors point to the upstream TypeScript sub-project (`packages/ccb/src/...`). The report explicitly warns readers that these files are upstream-only and not present in the Rust rewrite.
- Spot-checked anchors (`query.ts`, `autoCompact.ts`, `contextCollapse/index.ts`, `operations.ts`, `persist.ts`, `postCompactCleanup.ts`, `setup.ts`, `tools.ts`, `types/logs.ts`) all align with the on-disk files.
- The "缺失模块索引" correctly identifies three missing files (`CtxInspectTool`, `SnipTool`, `force-snip.js`).

*(Note: `packages/ccb/src/` files exist in the working tree but are untracked by git, so `git show HEAD:...` fails; verification was done against the on-disk files.)*

---

## 4. `55-tier3-stubs.md`

### PASS

- Like `52-context-collapse.md`, this report documents upstream Tier-3 feature stubs in `packages/ccb/src/`.
- Spot-checked files (`reactiveCompact.ts`, `MonitorTool.ts`, `udsClient.ts`, `concurrentSessions.ts`, `extractMemories.ts`, `tools.ts`) match their claimed content and line numbers.
- Missing-directory claims (`ListPeersTool` expected path does not exist) were confirmed accurate.

---

## Cross-Report Observations

- **No contradictions** were found between the four reports or with other units.
- `06-multi-turn.md` and `52-context-collapse.md` both discuss "compaction" but at different abstraction layers (Rust auto-compaction vs upstream TypeScript Context Collapse). Their descriptions are consistent and do not conflict.

---

## Summary

| Severity | Count | Files affected |
|----------|-------|----------------|
| CRITICAL | 8     | `09-shell-execution.md` (2), `06-multi-turn.md` (6) |
| MAJOR    | 1     | `06-multi-turn.md` (1) |
| MINOR    | 3     | `09-shell-execution.md` (2), `06-multi-turn.md` (1) |
| PASS     | 2     | `52-context-collapse.md`, `55-tier3-stubs.md` |

**Overall assessment**: `52-context-collapse.md` and `55-tier3-stubs.md` are accurate. `09-shell-execution.md` and `06-multi-turn.md` suffer from widespread line-number drift in `main.rs` and `session.rs`, likely caused by post-report code changes. The `Session` struct excerpt in `06-multi-turn.md` is incomplete (missing the `model` field) and should be refreshed.
