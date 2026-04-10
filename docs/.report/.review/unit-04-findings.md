# Unit 4 Critical Review Findings

**Unit assignment**: Integration & audit heavy  
**Files reviewed**: `57-lsp-integration.md`, `59-telemetry-remote-config-audit.md`, `50-experimental-skill-search.md`, `35-debug-mode.md`  
**Review date**: 2026-04-10  
**Commit tested**: HEAD of `main`

---

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | 0 |
| MAJOR    | 5 |
| MINOR    | 2 |
| PASS     | 2 |

**Notes**: `35-debug-mode.md` contains multiple stale/incorrect line-number anchors leading to completely wrong symbols. `57-lsp-integration.md` and `59-telemetry-remote-config-audit.md` are structurally sound. `50-experimental-skill-search.md` contains no rust anchors (as self-declared upstream-only) and passes cleanly.

---

## 1. `docs/.report/57-lsp-integration.md`

**Overall**: PASS with minor note.

All 25 rust source anchors verified successfully against HEAD. Files exist, line ranges are valid, and claimed symbols (`LspAction`, `LspRegistry::dispatch`, `McpStdioProcess`, `run_lsp`, etc.) are present at the cited locations.

### Findings

| Severity | Line in report | Finding |
|----------|----------------|---------|
| MINOR | Architecture diagram (#L28) | Diagram labels `run_lsp() #L1606-L1620` and `LspInput #L2303-L2312` but does not also label `global_lsp_registry() #L35-L42` which is called inside `run_lsp`. This is a completeness gap, not an error. |
| PASS | — | All prose claims about placeholder dispatch, extension mappings, and transport framing match the verified source. |

---

## 2. `docs/.report/59-telemetry-remote-config-audit.md`

**Overall**: PASS.

All 22 rust source anchors verified successfully against HEAD. The report correctly distinguishes upstream TypeScript-only paths from Rust-implemented paths.

### Findings

| Severity | Line in report | Finding |
|----------|----------------|---------|
| PASS | — | All telemetry line ranges in `telemetry/src/lib.rs`, `api/src/providers/anthropic.rs`, `runtime/src/conversation.rs`, `api/tests/client_integration.rs`, and `runtime/src/config.rs` align with current HEAD. |
| PASS | — | Factual claim that `runtime/src/config.rs` validation rejects unknown top-level key `"telemetry"` is confirmed by test at L1970–L1990. |

---

## 3. `docs/.report/50-experimental-skill-search.md`

**Overall**: PASS.

This report is explicitly upstream-only (all anchors point to `packages/ccb/src/...`). Because those paths do not exist in the `claw-code` repo, no HEAD anchor verification is possible. The report itself is consistent and makes no incorrect claims about Rust code.

### Findings

| Severity | Line in report | Finding |
|----------|----------------|---------|
| PASS | — | No rust anchors present; no factual claims about Rust implementation to falsify. |

---

## 4. `docs/.report/35-debug-mode.md`

**Overall**: MAJOR issues — multiple stale/broken anchors and symbol mismatches.

### Findings

| Severity | Section | Finding |
|----------|---------|---------|
| MAJOR | SlashCommand definition | Anchor `commands/src/lib.rs#L1276` claims "`SlashCommand::DebugToolCall` 定义与解析". Verified HEAD line 1276 is `Self::Plan { .. } => "/plan",` inside `fmt`. The actual `SlashCommand::DebugToolCall` enum variant is at **L1075**; the parsing logic is at **L1352–L1356**; the display string is at **L1250**. |
| MAJOR | CLI execution entry | Anchor `main.rs#L3615` claims "调用 `self.run_debug_tool_call(None)?`". Verified HEAD line 3615 is inside a `RuntimeToolDefinition` block (`name: "ListMcpResourcesTool"`). The actual call site is at **L3992** (`SlashCommand::DebugToolCall => { self.run_debug_tool_call(None)?; ... }`). |
| MAJOR | `run_debug_tool_call` implementation | Anchor `main.rs#L4356-L4360` claims to show `fn run_debug_tool_call`. Verified HEAD line 4356 is in `run_review` (`let handle = resolve_session_reference(&session_ref)?; ...`). The actual function is at **L4738–L4743**. |
| MAJOR | `render_last_tool_debug_report` | Anchor `main.rs#L5245-L5294` claims to show `render_last_tool_debug_report`. Verified HEAD lines 5245–5294 are `format_sandbox_report` and surrounding helpers. The actual function is at **L5736–L5783**. |
| MAJOR | Error rendering trace field | Anchor `main.rs#L6765-L6768` claims "错误渲染中的 Trace 字段". Verified HEAD lines 6765–6768 are inside `build_runtime` (`system_prompt: Vec<String>, ... if session.model.is_none() { session.model = Some(model.clone()); }`). The actual `Trace {request_id}` rendering is at **L7262**. |
| MINOR | Telemetry lifecycle table | The table maps `record_turn_started` to `conversation.rs#L550-L561`, but the actual verified range is **L549–L560** (a 1-line shift). Other rows in the same table show similar ~1-line drift versus HEAD, though still covering the correct symbols. |
| MINOR | Cross-report drift | `35-debug-mode.md` and `59-telemetry-remote-config-audit.md` cite overlapping telemetry symbols with different line ranges (e.g. `TelemetryEvent` is `#L171-L203` in 35-debug-mode and `#L170-L203` in 59-telemetry-remote-config-audit). Neither is wildly wrong, but only `59-telemetry-remote-config-audit.md` matches HEAD exactly. This indicates `35-debug-mode.md` was generated against an earlier commit. |

---

## Cross-Report Contradictions

| Concerning files | Topic | Contradiction |
|------------------|-------|---------------|
| `35-debug-mode.md` vs `59-telemetry-remote-config-audit.md` | `telemetry/src/lib.rs` line ranges | `35-debug-mode.md` cites `TelemetryEvent` as `L171-L203` (HEAD: `L170-L203`), `JsonlTelemetrySink` as `L233-L277` (HEAD: `L233-L277` — actually OK), and `SessionTracer` methods as `L280-L407` (HEAD: `L280-L407` — OK). The `TelemetryEvent` start line is off by 1. `59-telemetry-remote-config-audit.md` correctly cites `L170-L203`. |
| `35-debug-mode.md` vs `59-telemetry-remote-config-audit.md` | `conversation.rs` lifecycle hooks | `35-debug-mode.md` consistently cites ranges that are 1–2 lines higher than HEAD and than `59-telemetry-remote-config-audit.md`. For example `record_turn_started` is `L550-L561` in 35-debug-mode but `L550-L560` in 59-telemetry-remote-config-audit (HEAD: `L549-L560`). |

**Root cause hypothesis**: `35-debug-mode.md` was generated against an earlier commit where line numbers were shifted by ~1 line, and the large structural mismatches (`main.rs` off by hundreds of lines) suggest it may have been manually edited or generated from a significantly different tree.

---

## Recommended Fixes

1. **`35-debug-mode.md`** — Re-generate or hand-correct the following anchors to match current HEAD:
   - `commands/src/lib.rs#L1276` → `L1075` (enum variant)
   - `commands/src/lib.rs#L1274-L1277` → `L1352-L1356` (parse logic) or `L1248-L1252` (display)
   - `main.rs#L3615` → `L3992`
   - `main.rs#L4356-L4360` → `L4738-L4743`
   - `main.rs#L5245-L5294` → `L5736-L5783`
   - `main.rs#L6765-L6768` → `L7262`
2. **`35-debug-mode.md`** — Align `conversation.rs` and `telemetry/src/lib.rs` ranges with the verified HEAD ranges used in `59-telemetry-remote-config-audit.md`.
3. **All reports** — Consider adding a CI check (or pre-publish script) that runs `git show HEAD:<path> | sed -n '<start>,<end>p'` on every rust anchor to prevent line-number rot.

---

*Review completed by Unit 4 reviewer.*
