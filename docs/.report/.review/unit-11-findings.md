# Unit 11 Critical Review Findings

## Scope

- `docs/.report/43-bridge-mode.md`
- `docs/.report/26-plan-mode.md`
- `docs/.report/58-external-dependencies.md`
- `docs/.report/40-teammem.md`

---

## 43-bridge-mode.md

### PASS — All source anchors verified

| Anchor | Status | Notes |
|--------|--------|-------|
| `rust/crates/runtime/src/bootstrap.rs#L9` | PASS | `BridgeFastPath` enum variant |
| `rust/crates/runtime/src/bootstrap.rs#L32` | PASS | `BootstrapPhase::BridgeFastPath` in `claude_code_default()` |
| `rust/crates/compat-harness/src/lib.rs#L202` | PASS | `phases.push(BootstrapPhase::BridgeFastPath)` |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L25-L33` | PASS | `McpConnectionStatus` enum |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L64-L72` | PASS | `McpServerState` struct |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L74-L77` | PASS | `McpToolRegistry` struct |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L85` | PASS | `set_manager` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L92` | PASS | `register_server` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L114` | PASS | `get_server` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L119` | PASS | `list_servers` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L124` | PASS | `list_resources` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L140` | PASS | `read_resource` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L161` | PASS | `list_tools` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L177-L238` | PASS | `spawn_tool_call` function |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L240` | PASS | `call_tool` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L275` | PASS | `mcp_tool_name(...)` call site |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L281` | PASS | `set_auth_status` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L295` | PASS | `disconnect` method |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L314-L920` | PASS | Tests module (`mod tests { ... }` through EOF) |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L572` | PASS | `given_connected_server_with_manager_when_calling_tool_then_it_returns_live_result` |
| `rust/crates/runtime/src/mcp_tool_bridge.rs#L815` | PASS | `call_tool_payload_structure` test |
| `rust/crates/runtime/src/mcp.rs#L31` | PASS | `mcp_tool_name` function |
| `rust/crates/runtime/src/mcp_stdio.rs#L457-L460` | PASS | `ToolRoute` struct |
| `rust/crates/runtime/src/mcp_stdio.rs#L463-L468` | PASS | `ManagedMcpServer` struct |
| `rust/crates/runtime/src/mcp_stdio.rs#L480-L486` | PASS | `McpServerManager` struct |
| `rust/crates/runtime/src/plugin_lifecycle.rs#L16-L17` | PASS | `pub type ToolInfo = McpToolInfo;` / `pub type ResourceInfo = McpResourceInfo;` |
| `rust/crates/tools/src/lib.rs#L41-L46` | PASS | `global_mcp_registry()` singleton |
| `rust/crates/tools/src/lib.rs#L1074` | PASS | `ListMcpResources` tool name |
| `rust/crates/tools/src/lib.rs#L1086` | PASS | `ReadMcpResource` tool name |
| `rust/crates/tools/src/lib.rs#L1100` | PASS | `McpAuth` tool name |
| `rust/crates/tools/src/lib.rs#L1129` | PASS | `MCP` tool name |
| `rust/crates/tools/src/lib.rs#L1625` | PASS | `run_list_mcp_resources` handler |
| `rust/crates/tools/src/lib.rs#L1656` | PASS | `run_read_mcp_resource` handler |
| `rust/crates/tools/src/lib.rs#L1677` | PASS | `run_mcp_auth` handler |
| `rust/crates/tools/src/lib.rs#L1760` | PASS | `run_mcp_tool` handler |
| `rust/crates/tools/src/lib.rs#L2438-L2439` | MINOR OFFSET | Claims both `pending_mcp_servers` and `mcp_degraded` lines. L2438 is `pending_mcp_servers`, but L2439 is `mcp_degraded` (only one of the two claimed fields). The prose says "L2438-L2439" representing both fields; this is factually close but technically the line range spans exactly one field declaration plus the next line which is the `mcp_degraded` field. Acceptable but slightly imprecise. |

### MINOR — Preamble wording

- Line 15 says "所有行号均指向当前 `research` 分支." The repo is currently on `main` branch, and there is no `research` branch mentioned in git status. This could confuse readers into thinking the line numbers belong to a different branch. If the report was generated from upstream docs and then mapped to this repo, the text should say `main` or be updated to match the actual tracking branch.

---

## 26-plan-mode.md

### PASS — All source anchors verified

| Anchor | Status | Notes |
|--------|--------|-------|
| `rust/crates/tools/src/lib.rs#L670-L671` | PASS | `EnterPlanMode` tool definition (name + description) |
| `rust/crates/tools/src/lib.rs#L680-L681` | PASS | `ExitPlanMode` tool definition (name + description) |
| `rust/crates/tools/src/lib.rs#L1218` | PASS | Tool dispatch for `EnterPlanMode` |
| `rust/crates/tools/src/lib.rs#L2491-L2496` | PASS | `PlanModeState` struct |
| `rust/crates/tools/src/lib.rs#L4584-L4651` | PASS | `execute_enter_plan_mode` function |
| `rust/crates/tools/src/lib.rs#L4653-L4721` | PASS | `execute_exit_plan_mode` function |
| `rust/crates/tools/src/lib.rs#L5075-L5077` | PASS | `plan_mode_state_file()` path function |
| `rust/crates/tools/src/lib.rs#L5083-L5110` | PASS | State read/write/clear helpers |
| `rust/crates/tools/src/lib.rs#L7986-L8105` | MINOR OFFSET | Claims "two test cases" in this range. The first test starts at L7965 (`enter_and_exit_plan_mode_round_trip_existing_local_override`) and the second starts at L8038 (`exit_plan_mode_clears_override_when_enter_created_it_from_empty_local_state`). The range L7986-L8105 captures the body of the first test and most of the second, but truncates before the second test ends (which ends before L8106 where `structured_output_echoes_input_payload` begins). The range is therefore **not the exact test range** but an approximate bounding box. Since no specific symbols are claimed to sit precisely on L8105, this is a MINOR formatting/accuracy issue rather than a broken anchor. |
| `rust/crates/runtime/src/config.rs#L851-L862` | PASS | `parse_permission_mode_label` with `"plan" => ReadOnly` |
| `rust/crates/runtime/src/permission_enforcer.rs#L74-L108` | PASS | `check_file_write` method |
| `rust/crates/runtime/src/permission_enforcer.rs#L111-L139` | PASS | `check_bash` method |
| `rust/crates/runtime/src/permissions.rs` | PASS | File exists; no line range claimed |

### MINOR — Source index line references

The "源码索引" table at the end lists:
- `rust/crates/tools/src/lib.rs#L2182-L2186` — `EnterPlanModeInput` / `ExitPlanModeInput` 结构

Actual lines:
- L2182: `struct EnterPlanModeInput {}`
- L2183: `` (blank)
- L2184: `#[derive(Debug, Default, Deserialize)]`
- L2185: `#[serde(default)]`
- L2186: `struct ExitPlanModeInput {}`

This range is correct.

- `rust/crates/tools/src/lib.rs#L2491-L2515` — `PlanModeState` / `PlanModeOutput` 结构

Actual:
- L2491-L2496: `PlanModeState`
- L2498-L2515: `PlanModeOutput`

This range is correct.

The only MINOR issue is the test range imprecision noted above.

---

## 58-external-dependencies.md

### PASS — All factual anchors and file references verified

| Anchor | Status | Notes |
|--------|--------|-------|
| `rust/Cargo.toml` (workspace config) | PASS | File exists; 22 lines |
| `rust/crates/api/Cargo.toml` | PASS | 17 lines; `reqwest` on L9, `serde` on L11, `tokio` on L14 |
| `rust/crates/runtime/Cargo.toml` | PASS | 20 lines; `sha2` L9, `glob` L10, `regex` L12, `tokio` L16, `walkdir` L17 |
| `rust/crates/rusty-claude-cli/Cargo.toml` | PASS | 34 lines; `crossterm` L16, `pulldown-cmark` L17, `rustyline` L18, `syntect` L21, `tokio` L24 |
| `rust/crates/tools/Cargo.toml:11` | PASS | `flate2 = "1"` |
| `rust/crates/api/src/http_client.rs:63-113` | PASS | `build_http_client` and `build_http_client_with` |
| `rust/crates/api/src/http_client.rs:3-21` | PASS | `ProxyConfig` struct definition |
| `rust/crates/api/src/http_client.rs:83-112` | PASS | `build_http_client_with` function body |
| `rust/crates/rusty-claude-cli/src/input.rs:6-13` | PASS | `rustyline` imports |
| `rust/crates/telemetry/src/lib.rs:9-10` | PASS | `serde` / `serde_json` imports |
| `rust/crates/runtime/src/lane_events.rs:2-3` | PASS | `serde` imports |
| `rust/crates/runtime/src/sandbox.rs:5` | PASS | `serde` import |
| `rust/crates/tools/src/pdf_extract.rs:363-365` | PASS | `flate2` usage (`ZlibEncoder`) |
| `rust/crates/mock-anthropic-service/src/main.rs:5` | PASS | `#[tokio::main(flavor = "multi_thread")]` |
| `rust/crates/mock-anthropic-service/src/lib.rs:8-10` | PASS | `tokio::net::TcpListener` etc. |
| `rust/Cargo.lock` (crate count) | PASS | `grep -c '^name = '` returns 230 |

### MINOR — Inline anchor style inconsistency

The report uses backtick-quoted inline paths with colon separators (e.g., `rust/crates/api/Cargo.toml:9`) rather than `#L` hyperlinks. This is acceptable for a report that does not use markdown link syntax, but it is **stylistically inconsistent** with the rest of the `.report/` corpus which almost exclusively uses `#Lxx`/`#Lxx-Lyy` in markdown links. Recommend aligning with the established `#L` convention for better cross-referencing consistency.

### MINOR — Dependency count precision

Line 31 states: "约 230 个 (包含传递依赖，`Cargo.lock` 中 `name = ` 条目计数)". Verified: exactly 230. The "约" is technically unnecessary since the count is exact, but not a factual error.

### MINOR — `reqwest` feature claim

Line 39 shows `reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }`. Verified exact match in `api/Cargo.toml`. However, `tools/Cargo.toml` shows `reqwest = { version = "0.12", default-features = false, features = ["blocking", "rustls-tls"] }` (no `"json"`). The report only mentions the `api` crate's feature set, which is fine, but a more complete analysis might note the `blocking` feature used in `tools`.

---

## 40-teammem.md

### PASS — Expected findings

This report contains **no `rust/crates/` source anchors**. As documented in its own preamble:

> **源码映射说明**：Teammem 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。

Verification:
- No `packages/ccb/` directory exists in the current repo.
- No `memory/team/` or `teamMemory*` files exist under `src/` or `rust/`.
- The report is a conceptual upstream analysis only.

### MINOR — Preamble formatting

Line 9 has a missing space before `## 代码库定位`:
```markdown
... 请区分上游与 Rust 实现的覆盖范围。## 代码库定位
```
This renders as a run-on sentence in some markdown viewers.

---

## Cross-Report Notes

No contradictions found across the reviewed unit or with other units.

- **43-bridge-mode** and **26-plan-mode** both reference `rust/crates/tools/src/lib.rs` but for entirely different tool families (MCP bridge vs. plan mode). Lines do not overlap and are consistent.
- **58-external-dependencies** references `rust/Cargo.lock` with ~230 crates. No other unit disputes this.
- **40-teammem** explicitly disclaims any Rust source mapping, so it does not contradict any other report.

---

## Severity Summary

| Severity | Count | Description |
|----------|-------|-------------|
| CRITICAL | 0 | No broken anchors, false claims, or contradictions |
| MAJOR | 0 | No significant factual inaccuracies or missing context |
| MINOR | 6 | Stylistic and precision issues (see above) |
| PASS | 4 | All files are fundamentally sound |
