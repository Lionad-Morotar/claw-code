# Unit 1 Critical Review Findings

**Reviewer**: Unit 1 — Large mixed bag  
**Date**: 2026-04-10  
**Files reviewed**:
1. `docs/.report/33-hidden-features.md`
2. `docs/.report/14-compaction.md`
3. `docs/.report/31-growthbook-adapter.md`

**Method**: Static anchor verification against `HEAD` in `/Users/lionad/Github/Run/claw-code` using `git show HEAD:<path> | sed -n '<start>,<end>p'`.

---

## Executive Summary

| File | CRITICAL | MAJOR | MINOR |
|------|----------|-------|-------|
| `33-hidden-features.md` | 19 | 1 | 2 |
| `14-compaction.md` | 9 | 1 | 1 |
| `31-growthbook-adapter.md` | 0 | 4 | 1 |
| **Total** | **28** | **6** | **4** |

The majority of issues are **stale source-code anchors** caused by line-number drift in `main.rs`, `commands/src/lib.rs`, `compact.rs`, and `session.rs`. Additionally, `31-growthbook-adapter.md` contains a false disclaimer that the referenced upstream TypeScript files do not exist in the current repository, when in fact `packages/ccb/` is present and verifiable.

---

## 1. `33-hidden-features.md`

### CRITICAL — Stale / Broken Anchors

| # | Claimed anchor | Actual location at HEAD | Issue |
|---|----------------|-------------------------|-------|
| 1 | `commands/src/lib.rs#L1263` | ~`L1341` | Claimed line is `"Fast" => "/fast"`; `bughunter` parsing is ~10 lines later. |
| 2 | `commands/src/lib.rs#L1270` | ~`L1348` | Claimed line is `"Insights" => "/insights"`; `ultraplan` parsing follows. |
| 3 | `commands/src/lib.rs#L1271-L1274` | ~`L1349-L1352` | Teleport parse block shifted by ~80 lines. |
| 4 | `commands/src/lib.rs#L1274-L1277` | ~`L1352-L1355` | `debug-tool-call` parse block shifted. |
| 5 | `commands/src/lib.rs#L4089-L4128` | ~`L4207-L4280` | Command-parsing tests and `SlashCommand` assertions shifted downward. |
| 6 | `rusty-claude-cli/src/main.rs#L229-L232` | `L218` (relevant `CliAction::Sandbox` match arm) | Claimed range is inside a prior match arm (`} => {`). |
| 7 | `rusty-claude-cli/src/main.rs#L2606-L2614` | ~`L2817-L2824` | Resume dispatch for `SlashCommand::Sandbox` shifted by ~210 lines. |
| 8 | `rusty-claude-cli/src/main.rs#L3621-L3625` | ~`L3990-L3998` | Slash dispatch for `SlashCommand::Sandbox` shifted. |
| 9 | `rusty-claude-cli/src/main.rs#L3819-L3829` | ~`L4201-L4210` | `print_sandbox_status()` shifted by ~380 lines. |
| 10 | `rusty-claude-cli/src/main.rs#L4336-L4339` | ~`L4718-L4721` | `run_bughunter()` shifted by ~380 lines. |
| 11 | `rusty-claude-cli/src/main.rs#L4341-L4344` | ~`L4723-L4726` | `run_ultraplan()` shifted. |
| 12 | `rusty-claude-cli/src/main.rs#L4346-L4354` | ~`L4728-L4737` | `run_teleport()` shifted. |
| 13 | `rusty-claude-cli/src/main.rs#L4356-L4360` | ~`L4738-L4742` | `run_debug_tool_call()` shifted. |
| 14 | `rusty-claude-cli/src/main.rs#L5321-L5329` | ~`L5812-L5821` | `format_bughunter_report()` shifted by ~490 lines. |
| 15 | `rusty-claude-cli/src/main.rs#L5944-L5980` | ~`L6435-L6453`+ | `InternalPromptProgressReporter::ultraplan` shifted by ~490 lines. |
| 16 | `rusty-claude-cli/src/main.rs#L6088-L6110` | ~`L6567-L6589` | `InternalPromptProgressRun::start_ultraplan` shifted. |
| 17 | `rusty-claude-cli/src/main.rs#L10533-L10572` | ~`L11162-L11195` | `ultraplan_progress_lines_include_phase_step_and_elapsed_status` test shifted by ~630 lines. |
| 18 | `rusty-claude-cli/src/main.rs#L1315-L1363` | ~`L1407-L1441` | Claimed range is `DoctorReport` struct/`render`, not `render_doctor_report()`. |
| 19 | `rusty-claude-cli/src/main.rs#L1351-L1364` | ~`L1443-L1452` | Claimed range is inside `DoctorReport` impl; `run_doctor()` lives much later. |

### MAJOR — Factual Inaccuracy

| # | Claim | Source evidence | Issue |
|---|-------|-----------------|-------|
| 1 | Report quotes `bughunter` spec summary as `"Inspect code for likely bugs and correctness issues"`. | Actual `SlashCommandSpec` at `commands/src/lib.rs#L166-L172` reads `"Inspect the codebase for likely bugs"`. The string quoted in the report actually matches `format_bughunter_report` (lowercase in the formatter), conflating two distinct source strings. | Minor symbol/quote mismatch that could mislead a reader grepping for the spec text. |

### MINOR — Formatting / Typos

| # | Issue |
|---|-------|
| 1 | In the abstract command table, `main.rs#L4336` appears twice (Bughunter and Ultraplan) with no visual confusion, but the line ranges in the anchor body later contradict this. |
| 2 | The `Doctor` execution entry cross-reference table claims `main.rs#L1351` as entry point, which is structurally a formatting error if the anchor is off by ~90 lines. |

---

## 2. `14-compaction.md`

### CRITICAL — Stale / Broken Anchors

| # | Claimed anchor | Actual location at HEAD | Issue |
|---|----------------|-------------------------|-------|
| 1 | `conversation.rs#L121-L123` | `L18` | Claimed range is `AutoCompactionEvent` struct; default threshold constant `DEFAULT_AUTO_COMPACTION_INPUT_TOKENS_THRESHOLD` is at `L18`. |
| 2 | `compact.rs#L141-L149` | ~`L186-L193` | Claimed range is inside the tool-use boundary guard loop of `compact_session`; `compacted_summary_prefix_len()` starts at `L186`. |
| 3 | `compact.rs#L238-L271` | ~`L283-L311` | Claimed range is mid `summarize_messages`; `merge_compact_summaries()` starts at `L283`. |
| 4 | `compact.rs#L273-L288` | ~`L318-L333` | Claimed range ends inside `summarize_messages`; `summarize_block()` starts at `L318`. |
| 5 | `compact.rs#L308-L327` | ~`L353-L375` | Claimed range is inside `summarize_messages`/merge logic; `infer_pending_work()` starts at `L353`. |
| 6 | `compact.rs#L363-L388` | ~`L376-L400`+ | `collect_key_files()` starts at `L376`; extension whitelist `has_interesting_extension()` starts at `L394`. The anchor misses the actual whitelist logic. |
| 7 | `session.rs#L297-L298` | ~`L302-L303` | JSON serialization of `compaction` shifted by ~5 lines (inside `to_json`). |
| 8 | `session.rs#L499-L500` | ~`L515-L516` | JSONL compaction record shifted into `to_jsonl_records()`. |
| 9 | `session.rs#L1208-L1224` | ~`L1230-L1244` | Claimed range covers `appends_messages_to_persisted_jsonl_session`; the compaction-persistence test `persists_compaction_metadata` begins at `L1230`. |

### MAJOR — Factual Inaccuracy / Missing Context

| # | Claim | Source evidence | Issue |
|---|-------|-----------------|-------|
| 1 | Section **工具对完整性保护** states that tool-result completeness is guaranteed solely by preserving the tail window (`preserve_recent_messages`), implying the middle segment is fully summarized without raw blocks, which "彻底避免了 'tool_result 缺少对应 tool_use' 的 API 错误". | `compact.rs#L96-L139` contains an explicit **tool-use/tool-result boundary guard** that walks `keep_from` backward if the first preserved message would be an orphaned `ToolResult`. The code comment literally says: *"Ensure we do not split a tool-use / tool-result pair at the compaction boundary."* | The report ignores a critical runtime fix that exists specifically to prevent the API error it claims is avoided purely by the tail-window design. This is a significant omission for readers auditing compaction safety. |

### MINOR — Formatting

| # | Issue |
|---|-------|
| 1 | `compact.rs#L96-L139` is cited as the core `compact_session` function, but the function body continues to `L184`. The excerpt is acceptable since the anchor covers the excerpted code block, but a note that the function extends beyond would improve precision. |

---

## 3. `31-growthbook-adapter.md`

### MAJOR — Factual Inaccuracies

| # | Claim | Source evidence | Issue |
|---|-------|-----------------|-------|
| 1 | Preamble states: *"`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**"*. | `packages/ccb/src/constants/keys.ts` and `packages/ccb/src/services/analytics/growthbook.ts` **do exist** in HEAD under `/Users/lionad/Github/Run/claw-code/packages/ccb/`. All TS anchors verified successfully against these files. | The disclaimer is factually false and actively discourages readers from verifying the anchors. |
| 2 | Section 6.1 table lists `tengu_hive_evidence` default as `false` (用途: 任务证据系统). | `packages/ccb/src/services/analytics/growthbook.ts#L434-L472` (`LOCAL_GATE_DEFAULTS`) shows `tengu_hive_evidence: true` (comment: "Verification agent"). | Incorrect default value. |
| 3 | Section 6.1 table lists `tengu_auto_background_agents` default as `false`. | `LOCAL_GATE_DEFAULTS` shows `tengu_auto_background_agents: true` (comment: "Auto-background agents after 120s"). | Incorrect default value; also contradicted by Section 7 P0 which lists it as `true`. |
| 4 | Section 6.5 table lists Gate `tengu_chair_sermon` default as `false`. | `LOCAL_GATE_DEFAULTS` shows `tengu_chair_sermon: true` (comment: "Message smooshing"). | Incorrect default value. |

### MINOR — Formatting / Typos

| # | Issue |
|---|-------|
| 1 | Section 10 index row `packages/ccb/scripts/verify-gates.ts` has no line-range info; acceptable because the report treats it as an index, but consistency could be improved. |

### PASS — Verified TS Anchors

All of the following anchors were verified against actual file contents at HEAD and are correct:

- `packages/ccb/src/constants/keys.ts#L5-L16`
- `packages/ccb/src/services/analytics/growthbook.ts#L488-L496`
- `packages/ccb/src/services/analytics/growthbook.ts#L560-L694`
- `packages/ccb/src/services/analytics/growthbook.ts#L79-L81`
- `packages/ccb/src/services/analytics/growthbook.ts#L407-L417`
- `packages/ccb/src/services/analytics/growthbook.ts#L1113-L1117`
- `packages/ccb/src/services/analytics/growthbook.ts#L434-L472`
- `packages/ccb/src/services/analytics/growthbook.ts#L814-L867`

---

## Cross-Report Findings

No direct contradictions with files outside this unit were detected during the review. However, the following **internal contradictions** are noted:

1. **Within `31-growthbook-adapter.md`**:
   - `tengu_auto_background_agents` is `false` in Section 6.1 but `true` in Section 7 (P0).
   - `tengu_hive_evidence` is `false` in Section 6.1 but `true` in `LOCAL_GATE_DEFAULTS` source (Section 7 P1 omits it, but the source default is `true`).

2. **Within `33-hidden-features.md`**:
   - The quoted `bughunter` `summary` string in the anchor prose does not match the `SlashCommandSpec` definition, though it does match the lowercased report-formatter string.

---

## Recommendations

### Immediate (before next publication)
1. **Re-anchor all stale line numbers** in `33-hidden-features.md` and `14-compaction.md` by re-grepping current HEAD.
2. **Correct the `packages/ccb` existence disclaimer** in `31-growthbook-adapter.md` to state that the upstream TypeScript code *is* present in the repository under `packages/ccb/` and the anchors are verifiable.
3. **Fix the three incorrect feature-flag defaults** (`tengu_hive_evidence`, `tengu_auto_background_agents`, `tengu_chair_sermon`) in `31-growthbook-adapter.md` using the actual `LOCAL_GATE_DEFAULTS` source.
4. **Update the "工具对完整性保护" paragraph** in `14-compaction.md` to explicitly mention the tool-use/tool-result boundary guard introduced in `compact_session`.

### Process
- Add a CI check (or pre-commit hook) that verifies `rust/crates/...#Lxx-Lyy` and `packages/ccb/...#Lxx-Lyy` anchors against the current working tree to prevent future drift.
