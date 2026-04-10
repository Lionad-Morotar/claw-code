# Unit 9 — Critical Review Findings

**Scope**: 49-web-browser-tool.md, 24-permission-model.md, 36-buddy.md, 56-auto-updater.md  
**Date**: 2026-04-10  
**Reviewer**: Unit 9

---

## 1. docs/.report/49-web-browser-tool.md

### 1.1 Anchor Verification

All 25 unique `rust/crates/...` anchors were verified with `git show HEAD:<path> | sed -n '<start>,<end>p'`.

- **PASS** — All anchor ranges exist and the claimed symbols (`fn`, `struct`, `enum`) are present at/near the start line.
- **Note**: The source file `rust/crates/tools/src/lib.rs` is 8607 lines (report says ~8600), which is acceptable rounding.

### 1.2 Factual Claims vs. Source

- **PASS** — WebFetch/WebSearch tool specs, input structures, execution logic, HTTP client config (20s timeout, limited(10) redirect, UA), and test ranges all align with source.
- **PASS** — Claims about missing Safari MCP / Playwright are consistent with codebase (no `mcp__safari` or `playwright` references found in Rust workspace).

### 1.3 Issues Found

#### MINOR: Display text vs. anchor range mismatch in table
- **Location**: "已实现功能" table, WebFetch row  
- **Text**: `lib.rs#L2556-L2580`  
- **Issue**: The display text says 2580 but the href target earlier in the report for the same function is `L2556-L2590`. The table truncates 10 lines short of the actual function end. WebSearch row has the same inconsistency (`L2590-L2631` vs `L2590-L2642`). This does not break the link but is a stale truncation.

#### MINOR: Typo in "建议与实现风险"
- **Location**: Line 505-506  
- **Text**: `扩展现有能力***` and `文档补充***`  
- **Issue**: Rogue trailing `***` asterisks appear on items 3 and 4, inconsistent with items 1 and 2.

#### MINOR: Source index row
- **Location**: Appendix  
- **Text**: `rust/crates/runtime/src/mcp_stdio.rs` | `-` | MCP 客户端实现  
- **Issue**: While the file exists (2928 lines), claiming it is the "MCP 客户端实现" without line count or anchor makes the row low-value. Not a factual error, just weak precision.

---

## 2. docs/.report/24-permission-model.md

### 2.1 Anchor Verification

All 25 unique `rust/crates/...` anchors were verified.

- **PASS** — All anchor ranges exist and cover the claimed symbols.

### 2.2 Factual Claims vs. Source

- **PASS** — PermissionMode enum definition, alias parsing, `authorize_with_context` flow, rule parsing, `PermissionEnforcer`, sandbox enum, and `build_linux_sandbox_command` all match source.
- **PASS** — The "Allow 可以提升权限，但不能绕过 Ask 规则" claim is directly validated by test `hook_allow_still_respects_ask_rules` at L615-L642.

### 2.3 Issues Found

#### MINOR: Anchor display text / link range mismatch for PermissionMode
- **Location**: Line 18  
- **Text**: `runtime/src/permissions.rs#L9-L16`  
- **Issue**: The link text says L9-L16 but the actual anchor target is `#L9-L15`. The enum ends at line 15 (`Allow,` + closing `}` on line 16 but line 15 is the last value). Minor skew; the target range is valid.

#### MINOR: Anchor display text / link range mismatch for FilesystemIsolationMode
- **Location**: Line 353  
- **Text**: `runtime/src/sandbox.rs#L9-L15`  
- **Issue**: Link text says L9-L15 but the anchor target is `#L9-L14`. The enum ends at line 14. Minor skew.

#### MINOR: Imprecise `conversation.rs` reference
- **Location**: Line 103  
- **Text**: `runtime/src/conversation.rs#L393-L398`  
- **Issue**: The range L393-L398 falls inside a chain of `else if` branches handling hook outcomes (`is_failed`, `is_denied`). The actual `PermissionContext::new(...)` construction is at **L375-L379**. The cited range does not show the construction context claimed in prose ("构建 `PermissionContext`"). A more useful anchor would be L370-L381.

#### MINOR: `check` method prose slightly omits parameter details
- **Location**: Line 267  
- **Text**: "参见 ... L74-L108" for `check_file_write`
- **Issue**: None factual; just noting the prose says `check_file_write` checks workspace boundaries, which is correct per source.

---

## 3. docs/.report/36-buddy.md

### 3.1 Anchor Verification

This report references upstream TypeScript sources under `packages/ccb/src/...`. All 37 anchors were verified against HEAD.

- **PASS** — Every referenced file exists.
- **PASS** — Every line range is within bounds and contains the described code.

### 3.2 Factual Claims vs. Source

- **PASS** — Species list (18), rarity weights, stat names, `rollRarity`, `rollStats`, Shiny 1%, `SPECIES_NAMES`, command handlers (`off`, `on`, `pet`, default), `isBuddyTeaserWindow` logic, and sprite/animation constants all align with `packages/ccb/src/buddy/`.

### 3.3 Issues Found

None. The report prominently disclaims that Buddy is upstream-only and not present in the Rust rewrite, which is accurate.

---

## 4. docs/.report/56-auto-updater.md

### 4.1 Anchor Verification

All 42 anchors referencing `packages/ccb/src/...` were verified.

- **PASS** — Every file exists and every line range is valid.
- Representative spot-checks:
  - `download.ts#L74-L111` → `getLatestVersionFromBinaryRepo` starts at L74. ✅
  - `download.ts#L30-L73` → `getLatestVersionFromArtifactory` starts at L30. ✅
  - `download.ts#L382-L486` → `downloadVersionFromBinaryRepo` starts at L382. ✅
  - `download.ts#L293-L381` → `downloadAndVerifyBinary` starts at L293. ✅
  - `installer.ts#L115-L152` → `getBaseDirectories` starts at L115. ✅
  - `installer.ts#L639-L799` → `updateSymlink` starts at L639. ✅

### 4.2 Factual Claims vs. Source

- **PASS** — Installation-type routing, max-version gating, stall timeout (60s / 3 retries), file-lock path (`~/.claude/.update.lock`), 30-minute polling interval, `assertMinVersion` logic, semver wrapper, and `claude update` routing all match source.

### 4.3 Issues Found

None critical/major/minor. The report correctly scopes itself to the upstream `packages/ccb` implementation and does not falsely claim Rust parity.

---

## 5. Cross-Report Observations

No contradictions were found between the Unit 9 reports and any other reports in `docs/.report/`. Each report consistently distinguishes "upstream TypeScript (`packages/ccb`)" from "Rust rewrite (`rust/crates/...`)" where applicable.

---

## 6. Summary

| Severity | Count | Notes |
|----------|-------|-------|
| CRITICAL | 0 | No broken anchors, no false claims |
| MAJOR | 0 | No factual inaccuracies in behavior or numbers |
| MINOR | 5 | Display-text/anchor-range mismatches (L9-L16 vs L9-L15, L9-L15 vs L9-L14, L393-L398 vs actual L375-L379), truncated table ranges (2556-2580/2590-2631), and stray `***` formatting in 49-web-browser-tool.md |
| PASS | 4 | All four reports are structurally sound and source-aligned |
