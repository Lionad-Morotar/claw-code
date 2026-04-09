# Unit Review Log

## Unit 12 — project-memory

- **评定**: Pass
- **修正总数**: 0 处
- **自审问题**: 3 项（均不涉及源码锚点修正）
  1. `description` 属性中的 `"Based on ... "` 应为中文语境下的 `"基于 ..."`（文体问题，非技术错误）。
  2. `compact_session` 的 `max_estimated_tokens: 0` 需要额外说明——这是“尽可能压缩到最简”的意图，否则读者可能误会为 budget 设为 0。
  3. `SessionStore::from_cwd` 中对 `.claw/sessions/<fingerprint>` 的目录创建逻辑未在报告中显式展示，可补充一行以说明“首次访问即自动建目录”。

### 内容评定

报告准确还原了原始文档中 Claude Code 项目记忆系统的核心架构，并将 TypeScript 上游实现映射到 `claw-code` 的 Rust 实现：
- **记忆存储架构**: 对比了上游 `~/.claude/projects/<repo>/memory/` 目录布局与 Rust 当前以 `.claw/` 和 `CLAUDE.md` 为主的简化实现；
- **`ProjectContext`**: 映射到 `rust/crates/runtime/src/prompt.rs#L55-L62`，展示了指令文件发现、Git 感知增强和 System Prompt 注入链路；
- **指令文件发现**: `discover_instruction_files`（`prompt.rs#L203-L224`）向上遍历+reverse 的层级策略，`dedupe_instruction_files`（`L353-L368`）的 `stable_content_hash` 去重，以及 4KB/12KB 的体积硬上限；
- **Session 持久化**: `Session` JSONL 增量写入（`session.rs#L484-L498`），`SessionStore` 工作区隔离（`session_control.rs#L19-L63`）和 `workspace_fingerprint` FNV-1a 哈希（`L246-L254`）；
- **上下文压缩**: `maybe_auto_compact`（`conversation.rs#L548-L568`）触发 `compact_session`（`compact.rs#L96-L139`），将早期消息压缩为 System 摘要；
- **CLI 可见性**: `/memory` slash command 调用 `render_memory_report`（`rusty-claude-cli/src/main.rs#L5057-L5082`）。

文档技术准确、结构完整、源码锚点全部经过 `#LXX-LYY` 格式校验，**达到对外发布标准**。

*审校完成。*

---

## Unit 51 — token-budget-feature

- **Status**: Approved
- **原始文档**: https://ccb.agent-aura.top/docs/features/token-budget
- **输出报告**: `docs/.report/51-token-budget-feature.md`
- **审校结果**:
  - 源码锚点精确到文件路径 + `#LXX-LYY` 行号范围。
  - 解析层 (`src/utils/tokenBudget.ts`)、状态层 (`src/bootstrap/state.ts`)、决策层 (`src/query/tokenBudget.ts`)、主循环集成 (`src/query.ts`)、UI 层 (`PromptInput.tsx`, `Spinner.tsx`, `REPL.tsx`)、系统提示 (`src/constants/prompts.ts`)、API 附件 (`src/utils/attachments.ts`) 均已覆盖。
  - 明确区分了 `+500k` 自动续接 feature 与 API-side `task_budget` beta 机制，避免概念混淆。
  - 关键设计决策（90% 阈值、收益递减保护、子 agent 豁免、无条件缓存提示、用户取消清预算）均有对应源码引用。

---

## 36-buddy.md

- **审校时间**: 2026-04-09
- **审校结果**: 通过
- **说明**:
  1. 源码锚点已逐行核对，行号与 `packages/ccb/src/buddy/` 下 8 个文件及 `commands/buddy/buddy.ts` 一致。
  2. 报告覆盖了 Buddy 宠物系统的完整链路：Feature Flag 门控、命令处理、物种/稀有度/属性生成、确定性哈希算法、Mulberry32 PRNG、18 物种的 3 帧精灵图、爱心动画、语音气泡淡出、API 反应触发。
  3. 格式遵循既有报告结构：引言、分节源码映射、源码索引表、附录动画说明。
  4. 关键设计点：
     - 宠物 bones 基于用户 ID 哈希确定性生成，无法通过编辑 config 篡改稀有度
     - `roll()` 函数使用 `rollCache` 缓存，避免重复计算
     - `companionReact.ts` 调用 Anthropic OAuth API 生成宠物反应（需 accessToken）
     - 2026 年 4 月 1-7 日为 teaser window，仅在此期间显示彩虹色 `/buddy` 启动提示
  5. 源码索引表覆盖 36 处精确锚点，状态全部为 ✅。

---

## 28-three-tier-gating.md

- **评定**：Pass
- **修正总数**：1 处
- **issue 1**：Warning，源码索引 `lib.rs#L2090-L2106` 修正为 `L2095-L2108`。`struct TodoItem` 实际从第 2095 行开始，`enum TodoStatus` 实际位于第 2104 行，原引用范围起点与终点均存在偏差。

### 内容评定

报告准确还原了原始文档中三层门禁系统（构建时 `feature()`、运行时 GrowthBook、身份 `USER_TYPE`）的核心架构，并将原始 TypeScript 上游实现映射到 `claw-code` 的 Rust 参考数据与工具系统：
- **TodoWrite 状态枚举**：映射到 `tools/src/lib.rs#L2095-L2108`，展示了 `pending` / `in_progress` / `completed` 三门控状态；
- **Proactive 模式标记**：映射到 `tools/src/lib.rs#L645` 的 `"enum": ["normal", "proactive"]`，解释了 `normal` vs `proactive` 与原始 `USER_TYPE` 门控的对应关系；
- **Buddy 系统**：映射到 `src/reference_data/subsystems/buddy.json`，说明该功能在 Rust 实现中仅作为存档数据保留；
- **GrowthBook 集成点**：映射到 `services.json` 和 `types.json` 中的 `growthbook.ts` 与 `growthbook_experiment_event.ts` 引用；
- **Debug 模式命令**：映射到 `commands_snapshot.json` 中的 `debug-tool-call`，属于内部构建 (`ant`) 常用诊断工具。

文档技术准确、结构清晰、源码锚点丰富且可验证，**达到对外发布标准**。

*审校完成。*

---

## 27-auto-mode.md

- **评定**：Pass
- **修正总数**：写作阶段自审 2 处

### 写作阶段自审记录

- **issue 1**：Nit，初稿将 `PermissionMode` 枚举引用为 `L9-L16`。经核对源码，实际结束于第 15 行（`Allow` 在第 14 行，第 15 行为右大括号）。已修正为 `L9-L15`，确保引用精确覆盖枚举定义。
- **issue 2**：Nit，初稿将 `authorize_with_context` 引用为 `L175-L292`。经核对源码，函数实际结束于第 292 行（含右大括号），但起始行 175 包含了 `#[must_use]` 属性注解，函数签名实际从 176 开始。为保持一致性，已调整为 `L176-L292`，明确指向函数体。

### 抽检链接验证清单

| 链接 | 对应内容 | 状态 |
|------|----------|------|
| `permissions.rs#L9-L15` | `PermissionMode` 枚举定义 | ✅ |
| `permissions.rs#L176-L292` | `authorize_with_context` 完整判定逻辑 | ✅ |
| `permissions.rs#L259-L263` | `current_mode >= required_mode` 放行条件 | ✅ |
| `permissions.rs#L102-L104` | `allow_rules`/`deny_rules`/`ask_rules` 字段 | ✅ |
| `permissions.rs#L350-L402` | `PermissionRule::parse` 与匹配器解析 | ✅ |
| `permissions.rs#L447-L469` | `extract_permission_subject` JSON 字段提取 | ✅ |
| `permissions.rs#L569-L587` | allow/deny 规则测试用例 | ✅ |
| `permissions.rs#L614-L642` | hook Allow 被 ask 规则覆盖测试 | ✅ |
| `config.rs#L851-L863` | `parse_permission_mode_label` "auto"→WorkspaceWrite 映射 | ✅ |
| `config.rs#L857` | `"acceptEdits" \| "auto" \| "workspace-write"` 分支 | ✅ |
| `conversation.rs#L401-L410` | `run_turn` 中 `authorize_with_context` 调用 | ✅ |
| `conversation.rs#L392-L396` | `PermissionContext` 构造（hook override） | ✅ |
| `bash_validation.rs#L594-L615` | `validate_command` 五阶段验证管道 | ✅ |
| `bash_validation.rs#L27-L45` | `CommandIntent` 语义分类枚举 | ✅ |
| `bash_validation.rs#L533-L575` | `classify_command` 分类逻辑 | ✅ |
| `permission_enforcer.rs#L38-L44` | `Prompt` 模式特殊处理 | ✅ |
| `permission_enforcer.rs#L74-L108` | `check_file_write` 工作区边界检查 | ✅ |
| `permission_enforcer.rs#L111-L139` | `check_bash` 模式判定 | ✅ |
| `permission_enforcer.rs#L160-L238` | `is_read_only_command` 白名单启发式 | ✅ |

### 内容评定

报告严格遵循原文 ToC（Auto Mode 定义、权限模型、配置映射、授权判定流程、规则引擎、Bash 分类器、PermissionEnforcer、hook 角色、端到端示例），并为每个主节展开了 Rust 实现源码映射。

**关键发现**：
- `PermissionMode` 五级模型为偏序可比较类型，支持 `current_mode >= required_mode` 的直接比较
- `"auto"` 配置项在 `parse_permission_mode_label` 中被映射为 `ResolvedPermissionMode::WorkspaceWrite`，与 `"acceptEdits"`、`"workspace-write"` 等价
- `authorize_with_context` 采用规则优先于模式的判定顺序：deny → hook override → ask → allow/mode check → prompt
- Bash 命令验证采用五阶段管道（modeValidation → sedValidation → destructiveCommandWarning → pathValidation），并辅以 `CommandIntent` 语义分类器
- `PermissionEnforcer` 在 `Prompt` 模式下返回 `Allowed`，将确认职责交还给上层 `conversation.rs` 的 prompter 分支
- hook 可通过标准输出注入 `Allow`/`Ask`/`Deny` 策略，但 ask 规则始终优先于 hook Allow

文档对 Rust 实现与上游文档的差异（如 `"auto"` 别名映射、Bash 分类器的启发式策略）均作出了明确说明，无过度假设。**达到对外发布标准**。

*审校完成。*
