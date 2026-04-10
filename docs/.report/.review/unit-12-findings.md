# Unit 12 Critical Review Findings

**Files reviewed**: 42-voice-mode.md, 21-skills.md, 29-feature-flags.md, 30-growthbook-ab-testing.md  
**Review date**: 2026-04-10  
**Reviewer**: Unit 12 (Voice + skills + flags)

---

## 42-voice-mode.md

**Verdict**: PASS (with known upstream-only scope)

- This report exclusively references `packages/ccb/src/...` (upstream TypeScript implementation).
- The preamble clearly states that `claw-code` (Rust rewrite) has **not yet implemented** Voice Mode and that these source files do not exist in the current repository.
- No `rust/crates/...` anchors are present to verify.
- Factual claims about behavior, constants (e.g., `RELEASE_TIMEOUT_MS = 200`, `HOLD_THRESHOLD = 5`), and file responsibilities are consistent with the upstream code as described.

**Findings**: None. The report is self-consistent and its upstream-only nature is explicitly disclosed.

---

## 21-skills.md

**Verdict**: CRITICAL line-number drift on 6 anchors; 1 minor formatting issue

### CRITICAL — Broken / Stale Line Ranges

| Claimed anchor | Expected symbol | Actual content at claimed range | Correct range (HEAD) |
|----------------|-----------------|--------------------------------|----------------------|
| `rust/crates/commands/src/lib.rs#L2262-L2291` | `handle_skills_slash_command()` | **Plugins command handler** (`/plugins install/uninstall/update`) | `L2376-L2438` |
| `rust/crates/commands/src/lib.rs#L2325-L2343` | `classify_skills_slash_command()` | **Agents slash command handler** (`handle_agents_slash_command`) | `L2439-L2470` |
| `rust/crates/commands/src/lib.rs#L3085-L3160` | `load_skills_from_roots()` | `copy_directory_contents()`, `SkillInstallSource` impls, `push_unique_root()`, `load_agents_from_roots()` | `L3199-L3299` |
| `rust/crates/commands/src/lib.rs#L3186-L3215` | `parse_skill_frontmatter()` | Tail of `load_agents_from_roots()` and start of `load_skills_from_roots()` | `L3300-L3329` |
| `rust/crates/rusty-claude-cli/src/main.rs#L178-L181` | `CliAction::Skills` CLI entry | `}` (end of previous block) and `fn run()` declaration | `L193-L196` |
| `rust/crates/rusty-claude-cli/src/main.rs#L3678-L3684` | REPL `SlashCommand::Skills` handling | `runtime` builder code (`enable_all().build()`), completely unrelated | `L2923-L2936` (resume flow) or `L4052-L4065` (live REPL) |

The report quotes code snippets that do **not** exist at the claimed ranges; following those anchors leads to unrelated symbols. This is a significant regression for a documentation-audit task where anchor accuracy is the primary quality gate.

### MAJOR — Symbol drift on 1 additional anchor

- `rust/crates/commands/src/lib.rs#L1809-L1895` (claimed as `resume_supported_slash_commands()`): This range is extremely broad. The function `resume_supported_slash_commands()` actually starts at **L1887**, which *is* inside the claimed range, but the range also covers `command_error`, `remainder_after_command`, `find_slash_command_spec`, `command_root_name`, `slash_command_usage`, `slash_command_detail_lines`, etc. A tighter range like `L1887-L1893` would be more useful.

### PASS — Accurate anchors

The following anchors are accurate and contain the claimed symbols:

- `rust/crates/commands/src/lib.rs#L54-L60` — `SkillSlashDispatch`
- `rust/crates/tools/src/lib.rs#L558-L569` — `Skill` ToolSpec
- `rust/crates/tools/src/lib.rs#L2985-L2997` — `execute_skill()`
- `rust/crates/tools/src/lib.rs#L3021-L3044` — `resolve_skill_path()`
- `rust/crates/tools/src/lib.rs#L3029-L3044` — `resolve_skill_path_from_compat_roots()`
- `rust/crates/tools/src/lib.rs#L3167-L3196` — `resolve_skill_path_in_skills_dir()`
- `rust/crates/tools/src/lib.rs#L6511-L6567` — `skill_loads_local_skill_prompt()` test
- `rust/crates/tools/src/lib.rs#L6568-L6611` — `skill_resolves_project_local_skills_and_legacy_commands()` test
- `rust/crates/rusty-claude-cli/src/main.rs#L4052-L4065` — `print_skills()` / REPL `SlashCommand::Skills`

### MINOR — Markdown formatting

- The source-index table at the end lists `/rust/crates/commands/src/lib.rs` → `discover_skill_roots()` as `L2654-L2817`. The report text earlier links to `L2654-L2816`. In practice `discover_skill_roots()` starts at `L2768`, so even the broader table entry is off, but this is less severe than the code-quote anchors above.
- The “Tool vs Skill 的本质差异” table cell for registration location says `resume_supported_slash_commands()` is at `L1809-L1895`. As noted, the function begins at `L1887`.

---

## 29-feature-flags.md

**Verdict**: PASS (with known upstream-only scope)

- This report exclusively references `packages/ccb/src/...` (upstream TypeScript `bun:bundle` feature-flag system).
- The preamble clearly states that `claw-code` (Rust rewrite) does **not** use `feature()` or build-time feature flags, instead relying on Cargo features and runtime config.
- No `rust/crates/...` anchors are present.
- The flag taxonomy, usage patterns, and `build.ts`/`dev.ts` lists are internally consistent with the upstream implementation description.

**Findings**: None. Scope limitations are explicitly disclosed and the report is self-consistent.

---

## 30-growthbook-ab-testing.md

**Verdict**: PASS (with known upstream-only scope)

- This report exclusively references `packages/ccb/src/services/analytics/growthbook.ts` (upstream TypeScript implementation).
- The preamble clearly states that `claw-code` (Rust rewrite) has **not yet implemented** GrowthBook A/B testing.
- No `rust/crates/...` anchors are present.
- Factual claims about caching strategy (`~/.claude.json`, 20 min for employees / 6h for external users), `tengu_` naming conventions, and the dual-gate example (`tengu_amber_stoat`) are consistent with the upstream code as described.

**Findings**: None. The report is self-consistent and its upstream-only nature is explicitly disclosed.

---

## Cross-Report Observations

No direct contradictions with other units were observed in the four assigned files. The upstream-only reports (42, 29, 30) consistently disclose their scope and do not make false claims about Rust implementation coverage.

---

## Summary by Severity

| Severity | Count | Description |
|----------|-------|-------------|
| CRITICAL | 6 | Broken line ranges in `21-skills.md` that point to unrelated symbols |
| MAJOR | 1 | Overly broad range for `resume_supported_slash_commands()` |
| MINOR | 2 | Table/source-index formatting inconsistencies in `21-skills.md` |
| PASS | 3 | `42-voice-mode.md`, `29-feature-flags.md`, `30-growthbook-ab-testing.md` |

**Recommended fix**: Update the 6 CRITICAL anchors in `21-skills.md` to the correct HEAD ranges listed above, and tighten the `L1809-L1895` anchor to `L1887-L1893` (or similar) to avoid capturing unrelated helper functions.
