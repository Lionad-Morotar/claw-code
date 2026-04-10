# Unit 13 — Critical Review Findings

**Review scope**: `23-why-safety-matters.md`, `08-file-operations.md`, `54-auto-dream.md`, `53-workflow-scripts.md`
**Commit**: current HEAD (`main`)
**Date**: 2026-04-10

---

## Executive Summary

- **Total anchors reviewed**: 47 unique `rust/crates/...` anchors (all valid against HEAD)
- **Critical issues**: 1 (false factual claim about TOCTOU)
- **Major issues**: 2 (stale code snippets, incorrect whitelist)
- **Minor issues**: 3 (line-range drift, oversimplification, formatting)
- **Cross-report observations**: 2

---

## 23-why-safety-matters.md

### CRITICAL

_None._

### MAJOR

1. **`permission_enforcer.rs` — stale `is_read_only_command` whitelist**
   - **Anchor**: `rust/crates/runtime/src/permission_enforcer.rs#L160-L238`
   - **Issue**: The report reproduces a long whitelist that does **not match** current HEAD. The actual source (as of HEAD) contains commands such as `rg`, `printf`, `where`, `python3`, `node`, `ruby`, `cargo`, `rustc`, `cal`, `uptime`, `sha256sum`, `md5sum`, `b3sum`, `tree`, `yq`, etc. It omits many entries shown in the report (e.g., `pgrep`, `zcat`, `bzcat`, `xzcat`, `lsof`, `top`, `htop`, `vmstat`, `ss`, `netstat`, `ip`, `ifconfig`, `route`, `ping`, `curl`, `wget`, `dig`, `nmap`, `telnet`, `ssh`, `scp`, `rsync`, `svn`, `hg`, `cvs`, `bzr`, `fossil`, `p4`, `tf`, `glab`, `hub`).
   - **Impact**: Readers relying on the quoted whitelist to understand security boundaries will get a materially incorrect picture.

2. **`bash_validation.rs` — incomplete `DESTRUCTIVE_PATTERNS` list**
   - **Anchor**: `rust/crates/runtime/src/bash_validation.rs#L206-L235`
   - **Issue**: The quoted `DESTRUCTIVE_PATTERNS` array omits several entries present in HEAD: `("rm -rf *", ...)`, `("rm -rf .", ...)`, `("chmod -R 000", ...)` and the `ALWAYS_DESTRUCTIVE_COMMANDS` (`shred`, `wipefs`).
   - **Impact**: Underestimates the actual destructive-command surface that the runtime guards against.

### MINOR

3. **`permissions.rs` test-case anchor — line drift**
   - **Anchor**: `rust/crates/runtime/src/permissions.rs#L569-L586`
   - **Issue**: The test `applies_rule_based_denials_and_allows()` actually sits at `L568-L587` in HEAD. The off-by-one start/end is benign but indicates drift from a prior commit.

4. **Git read-only subcommands list is truncated**
   - **Anchor**: `rust/crates/runtime/src/bash_validation.rs#L163-L183`
   - **Issue**: `GIT_READ_ONLY_SUBCOMMANDS` in HEAD includes `ls-tree`, `shortlog`, `reflog`, and `config`, which are absent from the report snippet.

---

## 08-file-operations.md

### CRITICAL

5. **TOCTOU claim about `read_file_in_workspace` is factually false**
   - **Anchor**: `rust/crates/runtime/src/file_ops.rs#L562-L574`
   - **Issue**: The report claims: *"如果传入的路径本身在工作区内，即使它是指向外部的符号链接，当前实现也可能放行，因为检查的是传入路径而非 canonicalize 后的路径"*. This is incorrect. `read_file_in_workspace` calls `normalize_path(path)?` first, and `normalize_path` invokes `candidate.canonicalize()` whenever possible. The boundary check is therefore performed on the **canonicalized** target path, not the raw input path.
   - **Impact**: A false security concern is raised; the documented "blind spot" does not exist for the stated reason. (A true TOCTOU would require a symlink race between `normalize_path` and the subsequent `read_file`, but the report does not describe that race.)

### MAJOR

_None._

### MINOR

6. **`allowed_tools_for_subagent` range ends mid-vector**
   - **Anchor**: `rust/crates/tools/src/lib.rs#L3454-L3517`
   - **Issue**: The function `allowed_tools_for_subagent` continues to `L3531`. `L3517` cuts off the default `_ => vec![...]` case before its closing bracket. The anchor is not broken, but it is imprecise.

7. **`permissions.rs` test-case anchor — line drift (same as #3)**
   - **Anchor**: `rust/crates/runtime/src/permissions.rs#L568-L587`
   - **Issue**: Matches actual HEAD, which differs from the `L569-L586` reference in `23-why-safety-matters.md`. Both should be reconciled to the same range.

8. **Oversimplified `Prompt` mode description**
   - **Text**: "`Prompt`：每次权限升级都弹窗确认。"
   - **Issue**: In `permissions.rs`, `Prompt` means *any* non-allowable action triggers a prompt, not strictly "escalations". For example, a `ReadOnly`-required tool in `Prompt` mode would also prompt (or deny if no prompter is available). This nuance is lost.

---

## 54-auto-dream.md

### Notes
- **No `rust/crates/` anchors exist** in this report. The document is a mapping of upstream `packages/ccb` TypeScript code and explicitly states that the Rust rewrite has **not yet implemented** Auto Dream.
- No factual claims about the Rust codebase are made.

### Verdict
- **PASS** (for Rust codebase alignment). The disclaimer is prominent and accurate.

---

## 53-workflow-scripts.md

### Notes
- **No `rust/crates/` anchors exist** in this report. Like `54-auto-dream.md`, it documents upstream `packages/ccb` TypeScript stubs and carries the same explicit disclaimer.
- No factual claims about the Rust codebase are made.

### Verdict
- **PASS** (for Rust codebase alignment). The disclaimer is prominent and accurate.

---

## Cross-Report Observations

| # | Topic | Observation | Reports |
|---|-------|-------------|---------|
| C-1 | `permissions.rs` test case | `23-why-safety-matters.md` cites `L569-L586`; `08-file-operations.md` cites `L568-L587`. HEAD has `L568-L587`. The former should be updated. | `23-why-safety-matters.md`, `08-file-operations.md` |
| C-2 | `PermissionMode::Prompt` semantics | `08-file-operations.md` simplifies `Prompt` to "每次权限升级都弹窗确认", while `24-permission-model.md` (outside this unit) covers the full semantics more accurately. A cross-link or brief clarification would help. | `08-file-operations.md` vs. `24-permission-model.md` |

---

## Verified Anchor Inventory

All 47 unique `rust/crates/...#Lxx-Lyy` anchors from the two Rust-facing reports were verified against HEAD using `git show HEAD:<path> | sed -n '<start>,<end>p'`. Every anchor resolved successfully and pointed to valid source code.

### `23-why-safety-matters.md` (25 unique anchors)
- `bash.rs`: L112-L133, L143-L149, L292-L304, L307-L336
- `bash_validation.rs`: L103-L160, L163-L183, L206-L235, L206-L248, L360-L384, L533-L580, L594-L619
- `conversation.rs`: L378-L415, L427-L450, L455-L467, L474-L484
- `permission_enforcer.rs`: L74-L108, L160-L238
- `permissions.rs`: L9-L15, L175-L292, L182-L188, L196-L204, L240-L257, L266-L283, L569-L586
- `sandbox.rs`: L9-L14, L109-L153, L210-L262
- `session.rs`: (file-level reference only)

### `08-file-operations.md` (22 unique anchors)
- `bash.rs`: L19-L35, L185-L207
- `file_ops.rs`: L20-L26, L32-L44, L175-L221, L224-L255, L258-L296, L299-L340, L343-L450, L452-L465, L467-L487, L489-L508, L510-L526, L562-L574, L610-L620
- `permission_enforcer.rs`: L74-L108 (reused)
- `permissions.rs`: L9-L15, L174-L292, L568-L587 (reused)
- `rusty-claude-cli/src/main.rs`: (file-level reference only)
- `tools/src/lib.rs`: L409-L475, L1188-L1207, L3454-L3517

---

*End of findings.*
