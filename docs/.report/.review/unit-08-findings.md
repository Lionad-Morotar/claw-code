# Unit 8 — Agents & bash 关键核查报告

> 审查文件：
> - `docs/.report/22-custom-agents.md`
> - `docs/.report/48-bash-classifier.md`
> - `docs/.report/27-auto-mode.md`
> - `docs/.report/39-daemon.md`
> - 核查时间：2026-04-10
> - 代码基线：`HEAD` on `/Users/lionad/Github/Run/claw-code`

---

## 一、22-custom-agents.md

### 1.1 CRITICAL — `commands/src/lib.rs` 大量锚点行号失效

该报告中 `rust/crates/commands/src/lib.rs` 的所有行号锚点均严重偏离当前 HEAD，说明报告是基于旧版本源码撰写的，未随代码漂移更新。

| 报告中锚点 | 声称内容 | 实际 HEAD 位置 | 实际内容 |
|-----------|---------|---------------|----------|
| L1992-L2001 | `DefinitionSource` | **L2106** 起 | 枚举定义 |
| L2036-L2044 | `AgentSummary` | **L2151** 起 | struct 定义 |
| L2586-L2652 | `discover_definition_roots()` | **L2700** 起 | 函数定义 |
| L3042-L3083 | `load_agents_from_roots()` | **L3156** 起 | 函数定义 |
| L3231-L3271 | `render_agents_report()` | **L3345** 起 (text)，**L3387** 起 (json) | 函数定义 |
| L2216 / L2237 | `handle_agents_slash_command()` 调用点 | 实际在 **L2318 / L2339** | 函数定义而非调用点 |

**影响**：行号完全不可信，读者按锚点跳转将看到无关代码（如 `render_mcp_report`、skill frontmatter 解析等），严重破坏技术报告的可验证性。

### 1.2 MAJOR — `load_agents_from_roots()` 代码块摘录不完整/过时

报告中为 `load_agents_from_roots()` 分配的行号区间 L3042-L3083 与当前源码（L3156 起）相差 **114 行**，说明该报告对应的代码基线已显著过期。

### 1.3 MINOR — `render_agents_report` 行号指向 skill frontmatter 解析代码

L3231-L3271 在 HEAD 上实际显示的是 Markdown skill 的 frontmatter 解析与排序逻辑，与 `render_agents_report` 完全无关。

### 1.4 PASS — `tools/src/lib.rs` 及 `runtime/src/prompt.rs` 锚点基本正确

以下锚点在当前 HEAD 上内容匹配：
- `rust/crates/tools/src/lib.rs:L1212` — `"Agent" => from_value::<AgentInput>(input).and_then(run_agent)`
- `L2116-L2123` — `AgentInput`
- `L2391-L2400` — `AgentOutput`
- `L3290-L3336` — `execute_agent_with_spawn`
- `L3370-L3395` — `spawn_agent_job`
- `L3397-L3404` — `run_agent_job`
- `L3406-L3426` — `build_agent_runtime`
- `L3428-L3441` — `build_agent_system_prompt`
- `L3451-L3530` — `allowed_tools_for_subagent`
- `L3951-L3980` — `SubagentToolExecutor`
- `rust/crates/runtime/src/prompt.rs:L95-L167` — `SystemPromptBuilder`
- `L432-L446` — `load_system_prompt`

### 1.5 MINOR — 源码索引表中的行号同样过期

第 7 节“源码索引”中列出的 `commands/src/lib.rs` 所有行号均与当前 HEAD 不一致（差值约 100+ 行），需要整表刷新。

---

## 二、48-bash-classifier.md

### 2.1 CRITICAL — `authorize_with_context()` 锚点文件错误

报告在 7.1.2 节的表格中写道：

> `authorize_with_context()` | `[#L175](rust/crates/runtime/src/bash_validation.rs#L175)` | 带上下文的授权决策

**问题**：
- HEAD 上 `bash_validation.rs#L175` 的内容是 `"cat-file",`（某个数组内的字符串字面量），根本不存在 `authorize_with_context()` 函数。
- 该函数实际位于 `rust/crates/runtime/src/permissions.rs`（约 L175-L292），完全不在 `bash_validation.rs` 中。

这是一个**文件路径与符号双重错配**的致命错误，会导致读者在完全错误的文件中寻找关键函数。

### 2.2 PASS — `bash_validation.rs` 绝大多数锚点内容匹配

以下锚点经验证在当前 HEAD 上内容正确：
- `L103` — `pub fn validate_read_only(...)`
- `L241` — `pub fn check_destructive(...)`
- `L284` — `pub fn validate_mode(...)`
- `L336` — `pub fn validate_sed(...)`
- `L360` — `pub fn validate_paths(...)`
- `L533` — `pub fn classify_command(...)`
- `L594` — `pub fn validate_command(...)`
- `L622` — `fn extract_first_command(...)`
- `L206-L232` — `DESTRUCTIVE_PATTERNS`
- `L235` — `ALWAYS_DESTRUCTIVE_COMMANDS`
- `L389-L457` — `SEMANTIC_READ_ONLY_COMMANDS`
- `L460-L482` — `NETWORK_COMMANDS`
- `L485-L488` — `PROCESS_COMMANDS`
- `L491-L494` — `PACKAGE_COMMANDS`
- `L497-L527` — `SYSTEM_ADMIN_COMMANDS`
- `L52-L55` — `WRITE_COMMANDS`
- `L58-L94` — `STATE_MODIFYING_COMMANDS`
- `L533-L584`（行号范围）— `classify_command` 及其内部 `classify_by_first_command`
- `L594-L615` — `validate_command` 完整四阶段流水线

### 2.3 PASS — `permissions.rs`、`bash.rs`、`tools/src/lib.rs` 锚点匹配

- `permissions.rs` 中的 `PermissionMode`、`PermissionPolicy`、`PermissionOutcome` 位置与声明一致。
- `bash.rs` `L19-L39` — `BashCommandInput` / `BashCommandOutput`
- `bash.rs` `L70` — `execute_bash`
- `tools/src/lib.rs:L387-L407` — bash `ToolSpec`（含 `PermissionMode::DangerFullAccess`）
- `tools/src/lib.rs:L1183-L1186` — bash 工具调度
- `tools/src/lib.rs:L1791-L1794` — `run_bash` 执行入口

### 2.4 MINOR — 表格中 `permissions.rs` 行号缺少范围后缀

7.1.2 节表格里 `PermissionMode` 标记为 `L9`、`PermissionPolicy` 标记为 `L99`、`PermissionOutcome` 标记为 `L92`。虽然单点行号能够命中声明的起始行，但缺少范围后缀（如 `L9-L15`）不够严谨；不过这未造成事实性错误，列为 MINOR。

---

## 三、27-auto-mode.md

### 3.1 CRITICAL — `bash_validation.rs#L558-L580` 锚点范围过时

报告中有一处锚点：

> 代码在 `[bash_validation.rs#L558-L580](/rust/crates/runtime/src/bash_validation.rs#L558-L580)`。

**HEAD 实际内容**：`L558-L580` 落在了 `classify_by_first_command` 函数的中后段（`PROCESS_COMMANDS`、`PACKAGE_COMMANDS`、`SYSTEM_ADMIN_COMMANDS`、`git` 子命令判断），而 `CommandIntent` 枚举及其分类逻辑的起始在 `L533` 附近。该范围未能覆盖完整的分类器实现，导致读者只拿到尾部代码。

### 3.2 PASS — 其余 `bash_validation.rs` 锚点内容匹配

- `L27-L45` — `CommandIntent` 枚举及文档注释
- `L585-L619` — `validate_command` 四阶段流水线（含注释与函数签名）

### 3.3 PASS — `permissions.rs` 锚点全部命中

验证通过：
- `L9-L15` — `PermissionMode`
- `L102-L104` — `allow_rules/deny_rules/ask_rules`
- `L175-L292` — `authorize_with_context` 完整实现
- `L255-L264` — allow 规则 / 模式比较核心逻辑
- `L350-L402` — `PermissionRule::parse` 与 `parse_rule_matcher`
- `L447-L469` — `extract_permission_subject`
- `L560-L585` — 测试用例 `applies_rule_based_denials_and_allows`
- `L605-L644` — 测试用例 `hook_allow_still_respects_ask_rules`

### 3.4 PASS — `config.rs`、`conversation.rs`、`permission_enforcer.rs` 锚点命中

- `config.rs:L845-L862` — `parse_permission_mode_label`（含 `"auto" -> WorkspaceWrite`）
- `config.rs:L857` — `"acceptEdits" | "auto" | "workspace-write" => Ok(ResolvedPermissionMode::WorkspaceWrite)`
- `conversation.rs:L375-L378` — `PermissionContext::new(...)` 构造
- `conversation.rs:L400-L410` — `run_turn` 中权限判定分支
- `permission_enforcer.rs:L41-L52` — Prompt 模式返回 `EnforcementResult::Allowed`
- `permission_enforcer.rs:L74-L108` — `check_file_write`
- `permission_enforcer.rs:L111-L139` — `check_bash`
- `permission_enforcer.rs:L158-L210` — `is_read_only_command`

### 3.5 MINOR — 行号尾注写法不统一

报告中有的锚点写成 `L255-L265`（5 行差），而源码中实际 allow 规则比较只到 L264；`L375-L378` 与 `L400-L410` 等范围基本准确，但 `L255-L265` 多写了一行空行，属 MINOR。

---

## 四、39-daemon.md

### 4.1 PASS — 无 Rust 源码锚点需要核查

该报告在开头已明确声明：

> `claw-code`（Rust 重写版）目前尚未实现此功能。

报告中引用的所有源码路径均为上游 `packages/ccb/src/...`（TypeScript 实现），当前仓库中不存在对应文件。因此不存在 Rust 端锚点失效问题。

### 4.2 MINOR — 两处缺失标点

第 3 行开头：

> `## 一、功能概述`

紧跟在“源码映射说明”段落之后，缺少一个空行或换行分隔，格式上略显紧凑。属 MINOR。

---

## 五、跨报告（Cross-Report）注意事项

### 5.1 `commands/src/lib.rs` 行号基线不一致

`22-custom-agents.md` 中 `commands/src/lib.rs` 的行号明显是基于一个**更老**的代码基线（比当前 HEAD 小约 100-120 行），而 `48-bash-classifier.md` 和 `27-auto-mode.md` 中 `runtime/src/permissions.rs`、`bash_validation.rs` 等文件的锚点**基本与 HEAD 对齐**。这说明不同报告并非在同一时间切片上生成，建议统一刷新一次所有报告中的行号。

### 5.2 `authorize_with_context()` 的归属

`48-bash-classifier.md` 错误地将其归属到 `bash_validation.rs`，而 `27-auto-mode.md` 正确地将其归属到 `permissions.rs`。这是一个跨报告的事实矛盾，已在 2.1 中标记为 CRITICAL。

---

## 六、汇总统计

| 文件 | CRITICAL | MAJOR | MINOR | PASS |
|------|----------|-------|-------|------|
| 22-custom-agents.md | 6 | 1 | 2 | ~11 |
| 48-bash-classifier.md | 1 | 0 | 1 | ~24 |
| 27-auto-mode.md | 1 | 0 | 1 | ~16 |
| 39-daemon.md | 0 | 0 | 1 | 1 |
| **合计** | **8** | **1** | **5** | ~52 |

---

## 七、建议修复优先级

1. **最高**：修正 `22-custom-agents.md` 中 `commands/src/lib.rs` 的所有行号；该文件可能经历过一次大规模重构导致行号整体漂移。
2. **最高**：修正 `48-bash-classifier.md` 7.1.2 节中 `authorize_with_context()` 的文件路径（应指向 `permissions.rs` 而非 `bash_validation.rs`）。
3. **高**：修正 `27-auto-mode.md` 中 `bash_validation.rs#L558-L580` 的范围，使其覆盖完整的 `classify_command` 逻辑（建议改为 `L533-L580` 或 `L533-L584`）。
4. **中**：统一刷新 `22-custom-agents.md` 第 7 节“源码索引”表的行号。
5. **低**：处理 39-daemon.md 的格式 MINOR 问题。

---

*Review generated by Unit 8 — Agents & bash critical review.*
