# Unit 10 — System + search + planning

**Files reviewed**
- `docs/.report/12-system-prompt.md`
- `docs/.report/10-search-and-navigation.md`
- `docs/.report/45-ultraplan.md`
- `docs/.report/17-worktree-isolation.md`

**Method**: extract all `rust/crates/…#Lxx-Lyy` anchors, verify with `git show HEAD:<path> | sed -n 'start,endp'`, and spot-check prose claims against source.

---

## Executive Summary

- **~91 anchors** were extracted across the four reports.
- **3 CRITICAL** stale anchors in `12-system-prompt.md` point to completely unrelated functions in `main.rs`.
- **10+ MAJOR** stale anchors in `10-search-and-navigation.md` miss the claimed symbols or cover adjacent code.
- **1 MAJOR factual inaccuracy**: the report claims Anthropic server-side Prompt Cache (`cache_control`) is implemented, but the codebase only provides a **local** completion cache (`prompt_cache.rs`).
- **45-ultraplan.md** and **17-worktree-isolation.md** correctly scope themselves to upstream TypeScript; their limited Rust anchors are valid.
- No cross-report contradictions were identified.

---

## 12-system-prompt.md

### CRITICAL

1. **`main.rs#L2048-L2066` — completely wrong target**
   - *Claimed*: `print_system_prompt()` implementation.
   - *Actual*: OAuth token-exchange code (`AuthSource`, `exchange_oauth_code`, etc.).
   - *Current location of symbol*: `print_system_prompt` is at **L2172**.

2. **`main.rs#L3348-L3379` — completely wrong target**
   - *Claimed*: `LiveCli::new()` calling `build_system_prompt()`.
   - *Actual*: MCP request structs (`McpToolRequest`, `RuntimeMcpState::new`).
   - *Current location of symbol*: `LiveCli::new()` is at **L3725**.

3. **`main.rs#L5821-L5827` — completely wrong target**
   - *Claimed*: `build_system_prompt()` wrapper.
   - *Actual*: `format_ultraplan_report()`.
   - *Current location of symbol*: `build_system_prompt()` is at **L6312**.

### MAJOR

4. **Factual inaccuracy — Anthropic Prompt Cache**
   - *Claim*: "Anthropic 原生 API 支持服务器端的 Prompt Cache（通过 `cache_control`）".
   - *Fact*: Searching the `api` crate and `providers/anthropic.rs` shows **no usage** of `cache_control`. The `prompt_cache.rs` module is a **local file-system completion cache** (FNV-1a keyed, 30-second TTL) attached to the Anthropic client, not an upstream server-side prompt cache. The claim conflates upstream TypeScript capabilities with the Rust implementation.

### MINOR

5. **`main.rs#L168-L240` — slight drift**
   - The range begins inside `append_to_prompt()`; the `CliAction::Prompt` match arm starts around **L180**. Because the prose says "附近" (nearby), the content is still visible, but the anchor start is imprecise.

6. **`prompt.rs` one-line drifts**
   - `L54-L62`: `ProjectContext` struct starts at L55.
   - `L81-L90`: `discover_with_git` starts at L82.
   - `L93-L103`: `SystemPromptBuilder` struct starts at L95.
   - `L144-L164`: `build()` method ends at L166; the range cuts off the last two lines.
   - `L173-L193`: `environment_section` ends at L194; cuts off closing `}`.
   - `L353-L364`: `dedupe_instruction_files` ends a few lines past L364.

7. **`conversation.rs#L20-L26` — slight drift**
   - `ApiRequest` struct starts at L23; L20 is empty.

8. **`api/src/prompt_cache.rs` — consistent 1-line early starts**
   - `L108-L132`, `L268-L293`, `L303-L312`, `L313-L382` all start one line before the claimed struct/function.

### PASS

- `client.rs#L10-L47`, `L17-L19`, `L21-L47` — exact.
- `types.rs#L1-L28`, `L11` — exact.
- `config.rs#L242-L268`, `L271-L340` — exact.
- `conversation.rs#L321-L324` — exact.
- `git_context.rs#L12-L17`, `L26-L42` — exact enough.
- `prompt.rs#L40`, `L144`, `L153`, `L169-L171`, `L203-L223`, `L226-L236`, `L288-L328`, `L330-L351`, `L393-L403`, `L432-L446`, `L448-L467`, `L469-L478`, `L480-L494`, `L496-L510`, `L512-L518` — exact.

---

## 10-search-and-navigation.md

### MAJOR

1. **`runtime/src/file_ops.rs#L320-L325` — misses the claimed code**
   - *Claim*: `glob_search` sort-by-mtime block.
   - *Actual*: The `matches.sort_by_key(...)` block is at **L328-L333**; the anchor points to code just before the sort.

2. **`runtime/src/file_ops.rs#L351-L355` — misses the claimed code**
   - *Claim*: `RegexBuilder::build()` error handling.
   - *Actual*: The `RegexBuilder::build()` call occurs around **L359-360**; L351-L355 is still setting up `base_path`.

3. **`runtime/src/file_ops.rs#L379-L381` — misses the claimed code**
   - *Claim*: Silent skip of `fs::read_to_string` failures.
   - *Actual*: The `let Ok(file_contents) = fs::read_to_string(...)` guard is at **L387**.

4. **`runtime/src/file_ops.rs#L343-L431` — starts 8 lines too early**
   - `grep_search` begins at L351. The range begins at L343, capturing the end of the preceding `glob_search` function.

5. **`runtime/src/file_ops.rs#L452-L462` — starts 8 lines too early**
   - `collect_search_files` begins at L460.

6. **`runtime/src/file_ops.rs#L467-L487` — starts 8 lines too early**
   - `matches_optional_filters` begins at L475.

7. **`runtime/src/file_ops.rs#L489-L507` — starts 8 lines too early**
   - `apply_limit` begins at L497.

8. **`tools/src/lib.rs#L2556-L2587` — ends before function closes**
   - `execute_web_fetch` starts at L2556 but ends at **L2590**; the anchor cuts off the `Ok(WebFetchOutput { ... })` return.

9. **`tools/src/lib.rs#L2588-L2636` — starts early and ends before close**
   - `execute_web_search` starts at **L2590**; the anchor begins at L2588 (blank/`}`). The function ends around L2642, so the anchor truncates the tail.

10. **`tools/src/lib.rs#L2637-L2645` — covers only the signature + first line of body**
    - `build_http_client` starts at **L2644**; L2637-L2645 captures the blank line, signature, and `Client::builder()` but misses `timeout`, `redirect`, `user_agent`, and `build`.

11. **`tools/src/lib.rs#L2743-L2766` — cuts off function signature**
    - `html_to_text` starts at **L2738**; L2743 begins inside the function body.

12. **`tools/src/lib.rs#L2769-L2805` — covers the wrong functions**
    - `extract_search_hits` starts at **L2790**. L2769-L2805 instead covers `decode_html_entities`, `collapse_whitespace`, `preview_text`, and only the beginning of `extract_search_hits`.

13. **`tools/src/lib.rs#L2807-L2838` — mostly covers the tail end of `extract_search_hits`**
    - `extract_search_hits_from_generic_links` starts at **L2827**. L2807-L2838 is dominated by the end of the preceding function.

14. **`tools/src/lib.rs#L2883-L2901` — starts inside the function**
    - `decode_duckduckgo_redirect` starts at **L2879**; L2883 is a few lines into the body.

### MINOR

15. **`runtime/src/file_ops.rs` struct drifts (1 line early)**
    - `L120-L127` (`GlobSearchOutput` starts at L121).
    - `L131-L154` (`GrepSearchInput` starts at L132).
    - `L157-L172` (`GrepSearchOutput` starts at L158).

16. **`runtime/src/lsp_client.rs` — 1-2 line early starts**
    - `L10-L35` (`LspAction` starts at L12).
    - `L109-L233` (`LspRegistry` starts at L110).
    - `L150-L172` (`find_server_for_path` starts at L151).
    - `L235-L348` (`dispatch` starts at L236).

### PASS

- `runtime/src/git_context.rs#L24-L42` — exact.
- `tools/src/lib.rs#L1178-L1207` — exact enough for claimed dispatch logic.
- `tools/src/lib.rs#L1969-L1986` — exact.
- `tools/src/lib.rs#L4121-L4131` — exact.
- `tools/src/lib.rs#L4133-L4205` — exact.
- All prose claims about `regex`/`walkdir` in-process search, LSP placeholder dispatch, `ReadOnly` permission mode, WebSearch 8-result truncation, 20-second HTTP timeout, and DuckDuckGo redirect decoding were verified as accurate.

---

## 45-ultraplan.md

### PASS

- The report explicitly states that **ULTRAPLAN is not implemented in the Rust codebase** and that all referenced paths are upstream (`packages/ccb/…`).
- No `rust/crates/…` anchors exist in this file, so no Rust source verification was required.
- No factual inaccuracies regarding the declared upstream scope.

---

## 17-worktree-isolation.md

### PASS

- The report explicitly disclaims complete worktree support in the Rust rewrite and limits upstream Rust anchors to:
  - `tools/src/lib.rs#L4584-L4651` (`execute_enter_plan_mode` at **L4584**) — verified exact.
  - `tools/src/lib.rs#L4653-L4722` (`execute_exit_plan_mode` at **L4653**) — verified exact.
  - `rust/PARITY.md#L82-L83` — verified exact (lists **EnterPlanMode** and **ExitPlanMode** as good parity).
- Prose accurately describes the gap between upstream TypeScript worktree tooling and current Rust capabilities.

---

## Cross-Report Observations

No contradictions were found **within Unit 10**.

No contradictions with **files outside Unit 10** were identified during this review (no external reports were inspected; this finding is based solely on the assigned files).

---

## Recommendations

1. **Refresh all `main.rs` anchors** in `12-system-prompt.md`; the file has grown significantly and several reported functions have shifted by hundreds of lines.
2. **Refresh `file_ops.rs` and `tools/src/lib.rs` anchors** in `10-search-and-navigation.md`; a consistent ~8-line drift exists across many ranges, and a few ranges cover the wrong symbols entirely.
3. **Correct or qualify the Prompt Cache claim** in `12-system-prompt.md` to clarify that the Rust implementation currently provides only a **local completion cache**, not Anthropic server-side `cache_control` caching.
4. **Re-run the anchor extraction script** after the next upstream merge to prevent further bit-rot.
