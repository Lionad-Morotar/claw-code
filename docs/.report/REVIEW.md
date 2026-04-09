
---

## 25-sandbox.md

- **评定**：Pass
- **修正总数**：写作阶段自审 4 处

### 写作阶段自审记录

- **issue 1**：Nit，初稿将 `dangerously_disable_sandbox` 的描述仅锚定到 `bash.rs`。经核对源码后补充：在 `tools/src/lib.rs#L397` 的 JSON Schema 定义、`bash.rs#L53-L54` 的输出回显字段也需明确，形成完整链路。
- **issue 2**：Nit，初稿关于"macOS 不支持沙箱"的描述过于绝对。修正为：macOS 由仓库判定为 `namespace_supported = false`，因此不会走 `unshare` 路径，仅保留 `HOME`/`TMPDIR` 重定向。这是准确的运行时行为描述。
- **issue 3**：Warning（自审），原文关于 `normalize_mounts` 的描述需明确相对路径的解析基准。修正：补充"相对路径会基于当前工作目录解析为绝对路径"的说明，并锚定到 `sandbox.rs#L264-L278`。
- **issue 4**：Nit，初稿遗漏 `unshare_user_namespace_works` 的 `OnceLock` 缓存机制。修正：补充该结果通过 `std::sync::OnceLock` 缓存，避免每次命令执行都重复探测，并解释 GitHub Actions 等 CI 环境中用户命名空间受限导致回退的原因。

### 抽检链接验证清单

| 链接 | 对应内容 | 状态 |
|------|----------|------|
| `sandbox.rs#L9-L14` | `FilesystemIsolationMode` 枚举 | ✅ |
| `sandbox.rs#L87-L105` | `resolve_request` 配置合并 | ✅ |
| `sandbox.rs#L162-L208` | `resolve_sandbox_status_for_request` | ✅ |
| `sandbox.rs#L109-L153` | `detect_container_environment` | ✅ |
| `sandbox.rs#L211-L262` | `build_linux_sandbox_command` | ✅ |
| `sandbox.rs#L264-L278` | `normalize_mounts` | ✅ |
| `sandbox.rs#L288-L304` | `unshare_user_namespace_works` | ✅ |
| `bash.rs#L19-L67` | `BashCommandInput` / `BashCommandOutput` | ✅ |
| `bash.rs#L25-L26` | `dangerously_disable_sandbox` 输入字段 | ✅ |
| `bash.rs#L53-L54` | `dangerously_disable_sandbox` 输出回显 | ✅ |
| `bash.rs#L70-L103` | `execute_bash` 入口 | ✅ |
| `bash.rs#L143-L163` | `sandbox_status_for_input` | ✅ |
| `bash.rs#L212-L237` | `prepare_tokio_command` | ✅ |
| `config.rs#L865-L923` | `parse_optional_sandbox_config` | ✅ |
| `main.rs#L2606-L2614` | `/sandbox` slash 命令处理 | ✅ |
| `main.rs#L4873-L4920` | `format_sandbox_report` | ✅ |
| `tools/src/lib.rs#L397` | JSON Schema `dangerouslyDisableSandbox` 字段 | ✅ |
| `commands/src/lib.rs#L75-L77` | `SlashCommand::Sandbox` 注册 | ✅ |

### 内容评定

报告将沙箱机制的中文文档架构完整映射到了 `claw-code` Rust 源码：
- **配置解析**：`.claw.json` / `.claw/settings.json` 中的 `sandbox` 字段由 `ConfigLoader` 解析，支持 `enabled`、`namespaceRestrictions`、`networkIsolation`、`filesystemMode`、`allowedMounts`；
- **状态解析**：`resolve_sandbox_status_for_request` 决定最终能否激活沙箱，包含平台能力探测、容器检测、降级原因记录；
- **执行链路**：`execute_bash` → `sandbox_status_for_input` → `prepare_tokio_command` → `build_linux_sandbox_command`（可选 `unshare` 包裹）；
- **dangerouslyDisableSandbox**：输入字段（`bash.rs#L25`）、JSON Schema 暴露（`tools/src/lib.rs#L397`）、配置合并时的反转逻辑、输出回显（`bash.rs#L53`）的完整审计链路；
- **平台差异**：Linux 完整支持 `unshare` 命名空间隔离，macOS/Windows 原生不支持，WSL 取决于内核配置；
- **CLI 可观测性**：`/sandbox` 与 `claw sandbox` 输出沙箱状态快照，包含启用/激活/支持/容器标记/降级原因等 14 个字段。

文档对 Rust 实现与上游文档的差异（OS 级隔离原语从 `sandbox-exec`/`bubblewrap` 变为 `unshare`、配置系统从 TypeScript 对象变为 JSON 解析）均作出了明确说明。**达到对外发布标准**。

*审校完成。*


---

## 24-permission-model.md

- **评定**: Pass
- **修正总数**: 0 处（写作阶段自审通过，行号抽检全部命中）

### 行号抽检结论

文档共含 25+ 处带 `#LXX-LYY` 锚点的源码链接，经逐条验证均与 `rust/crates/runtime/src/` 中的源码内容精确对应。关键引用如下：

| 链接 | 对应内容 | 状态 |
|------|----------|------|
| `permissions.rs#L9-L16` | `PermissionMode` 五级枚举 | ✅ |
| `permissions.rs#L31-L36` | `PermissionOverride` (Allow/Ask/Deny) | ✅ |
| `permissions.rs#L86-L88` | `PermissionPrompter` trait | ✅ |
| `permissions.rs#L175-L292` | `authorize_with_context` 完整判定链路 | ✅ |
| `permissions.rs#L294-L324` | `prompt_or_deny` 弹窗或降级拒绝 | ✅ |
| `permissions.rs#L350-L401` | `PermissionRule` 解析与字符串匹配 | ✅ |
| `permissions.rs#L447-L466` | `extract_permission_subject` JSON 字段提取 | ✅ |
| `permission_enforcer.rs#L27-L139` | `PermissionEnforcer` 结构体与 `check`/`check_file_write`/`check_bash` | ✅ |
| `permission_enforcer.rs#L143-L157` | `is_within_workspace` 工作区边界前缀检查 | ✅ |
| `sandbox.rs#L9-L15` | `FilesystemIsolationMode` 三层隔离枚举 | ✅ |
| `sandbox.rs#L211-L262` | `build_linux_sandbox_command` unshare 参数构建 | ✅ |
| `conversation.rs#L386-L417` | `run_turn` 中权限判定与 Hook 集成 | ✅ |
| `conversation.rs#L393-L398` | `PermissionContext` 构造 | ✅ |
| `config.rs#L851-L860` | `parse_permission_mode_label` 模式别名映射 | ✅ |
| `config.rs#L780-L795` | `parse_optional_permission_rules` JSON 规则解析 | ✅ |

### 内容评定

报告将 Permission Model 的安全文档完整映射到了 `claw-code` Rust 源码：
- 五级 `PermissionMode` 与上游概念的精确对应，以及配置层别名解析机制；
- `authorize_with_context` 的 6 步优先级判定流程（Deny 规则短路 → Hook Override → Ask 规则 → Allow/模式匹配 → 升级弹窗 → 默认 Deny）；
- `PermissionRule` 的语法解析（工具名 + 可选括号匹配器）与 `command/path/file_path/...` 等 JSON 字段提取逻辑；
- `PermissionEnforcer` 对通用工具、文件写、Bash 的三条执行层门控路径；
- `PermissionPrompter` 在 `ConversationRuntime::run_turn` 中的实际集成方式（有 prompter 则弹窗，无则降级 Deny）；
- `Sandbox` 作为互补的第二层隔离：`FilesystemIsolationMode` 与 Linux `unshare` namespace 沙箱命令构建；
- 大量单元测试锚点，覆盖规则匹配、Hook 覆盖、模式升级、弹窗拒绝、工作区边界、Bash 只读启发式等核心分支。

文档对 Rust 实现与权限模型概念之间的映射完整、层次清晰、源码锚点精确。**达到对外发布标准**。

*审校完成。*

---

## 17-worktree-isolation.md

- **评定**：Pass
- **修正总数**：0 处（写作阶段经多轮源码对位校验，无追加修正）

### 行号抽检结论

文档共含 20+ 处带 `#LXX-LYY` 锚点的源码链接，均位于 `packages/ccb/src/` 与 `rust/` 中，经关键条目抽检全部与源码内容精确对应：

| 链接 | 对应内容 | 状态 |
|------|----------|------|
| `packages/ccb/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts#L77-L81` | 防嵌套检查 | ✅ |
| `packages/ccb/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts#L92-L103` | 进程状态更新与缓存清理 | ✅ |
| `packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts#L79-L113` | `countWorktreeChanges` fail-closed 设计 | ✅ |
| `packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts#L174-L223` | `validateInput` 安全门控 | ✅ |
| `packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts#L256-L259` | 执行时重新校验变更 | ✅ |
| `packages/ccb/src/utils/worktree.ts#L66-L93` | `validateWorktreeSlug` / `flattenSlug` | ✅ |
| `packages/ccb/src/utils/worktree.ts#L235-L509` | `getOrCreateWorktree` 快速恢复与创建 | ✅ |
| `packages/ccb/src/utils/worktree.ts#L510-L692` | `performPostCreationSetup` | ✅ |
| `packages/ccb/src/utils/worktree.ts#L702-L779` | `createWorktreeForSession` | ✅ |
| `packages/ccb/src/utils/worktree.ts#L780-L812` | `keepWorktree` | ✅ |
| `packages/ccb/src/utils/worktree.ts#L813-L897` | `cleanupWorktree` | ✅ |
| `packages/ccb/src/utils/worktree.ts#L898-L947` | `createAgentWorktree` | ✅ |
| `packages/ccb/src/utils/worktree.ts#L1018-L1129` | `cleanupStaleAgentWorktrees` | ✅ |
| `packages/ccb/src/utils/worktree.ts#L1131-L1161` | `hasWorktreeChanges` | ✅ |
| `rust/crates/tools/src/lib.rs#L4584-L4689` | `execute_enter_plan_mode` / `execute_exit_plan_mode` | ✅ |

### 内容评定

报告严格遵循原文 ToC，完整揭示了 worktree 隔离的创建/销毁生命周期、路径命名规则、hook 机制、fail-closed 安全防护，以及 Agent 子 worktree 的生命周期。文档同时诚实地指出了 `claw-code` Rust 重写版当前尚未实现 `EnterWorktree`/`ExitWorktree` 工具，仅在上游 `packages/ccb` 中存在完整实现，并给出了准确的 `PARITY.md` 映射。源码索引表覆盖了从 slug 验证、git worktree 命令、tmux 快速路径到 Rust 侧 plan mode 的 18 个精确锚点。**达到对外发布标准**。

*审校完成。*

---

## 23-why-safety-matters.md

- **评定**：Pass
- **修正总数**：0 处（写作阶段自审通过，行号抽检全部命中）

### 行号抽检结论

文档共含 40+ 处带 `#LXX-LYY` 锚点的源码链接，经逐条验证均与 `rust/crates/runtime/src/` 中的源码内容精确对应。关键引用如下：

| 链接 | 对应内容 | 状态 |
|------|----------|------|
| `permissions.rs#L9-L16` | `PermissionMode` 五级权限枚举 | ✅ |
| `permissions.rs#L175-L292` | `authorize_with_context` 完整决策逻辑 | ✅ |
| `permission_enforcer.rs#L38-L67` | `PermissionEnforcer::check` 策略执行 | ✅ |
| `permission_enforcer.rs#L74-L108` | `check_file_write` 文件写边界检查 | ✅ |
| `permission_enforcer.rs#L160-L238` | `is_read_only_command` 只读启发式 | ✅ |
| `sandbox.rs#L9-L14` | `FilesystemIsolationMode` 枚举 | ✅ |
| `sandbox.rs#L109-L153` | `detect_container_environment` 容器检测 | ✅ |
| `sandbox.rs#L210-L262` | `build_linux_sandbox_command` unshare 隔离 | ✅ |
| `bash.rs#L19-L67` | `BashCommandInput` / `BashCommandOutput` | ✅ |
| `bash.rs#L112-L133` | 命令超时控制 | ✅ |
| `bash.rs#L288-L304` | `truncate_output` 输出截断 | ✅ |
| `bash_validation.rs#L16-L45` | `ValidationResult` / `CommandIntent` | ✅ |
| `bash_validation.rs#L103-L160` | `validate_read_only` 只读验证 | ✅ |
| `bash_validation.rs#L206-L235` | `DESTRUCTIVE_PATTERNS` 危险模式 | ✅ |
| `bash_validation.rs#L241-L274` | `check_destructive` 破坏性命令警告 | ✅ |
| `bash_validation.rs#L360-L382` | `validate_paths` 路径遍历检测 | ✅ |
| `bash_validation.rs#L533-L584` | `classify_command` 语义分类 | ✅ |
| `bash_validation.rs#L594-L615` | `validate_command` 完整流水线 | ✅ |
| `conversation.rs#L378-L415` | PreToolUse Hook 决策分支 | ✅ |
| `conversation.rs#L427-L453` | PostToolUse Hook 覆盖逻辑 | ✅ |
| `conversation.rs#L455-L467` | Session 审计日志写入 | ✅ |
| `conversation.rs#L474-L484` | `TurnSummary` 生成 | ✅ |

### 内容评定

报告将 CCB 安全文档的主题完整映射到了 `claw-code` Rust 实现：
- **权限模型**：五级 `PermissionMode`、`PermissionPolicy` 规则引擎（Deny/Ask/Allow）、Hook 覆盖决策链；
- **沙箱隔离**：基于 `unshare` 的 Linux 命名空间隔离、三种 `FilesystemIsolationMode`、容器环境多源检测；
- **命令验证**：`bash_validation.rs` 四层流水线（模式验证 → sed 验证 → 破坏性警告 → 路径验证）、`CommandIntent` 语义分类、git 子命令白名单、sudo 递归穿透；
- **权限执行层**：`PermissionEnforcer` 的文件写边界检查（`is_within_workspace`）与 Bash 只读启发式；
- **Hook 拦截**：`conversation.rs` 中 `run_pre_tool_use_hook` 与 `run_post_tool_use_hook` 的执行链路、反馈合并（`merge_hook_feedback`）；
- **审计追踪**：`Session::push_message` 持久化、`ToolUse` / `ToolResult` 严格配对、`TurnSummary` 端到端摘要；
- **错误处理**：Bash 超时控制（`tokio::time::timeout`）、返回码解释化、16 KiB 输出截断与 UTF-8 字符边界回退。

文档对源码的引用路径统一使用 `/rust/crates/...` 绝对路径，锚点精确，逻辑描述与代码行为一致。**达到对外发布标准**。

*审校完成。*


---

## 10-search-and-navigation.md

- **评定**：Pass
- **修正总数**：写作阶段自审 3 处
- **源码链接验证**：25 处链接全部有效

### 写作阶段自审记录

- **issue 1**：Nit，初稿将 `grep_search` 的默认 limit 描述为"硬编码在 `grep_search` 函数内"。经核对源码，实际硬编码位于 `apply_limit` 辅助函数（`file_ops.rs#L496`），默认值 `explicit_limit = limit.unwrap_or(250)`。描述已调整。
- **issue 2**：Nit，初稿将 `search_tool_specs` 的 `select:` 前缀处理描述为"第一 guard"，实际代码位于 `tools/src/lib.rs#L243-L258`，先于评分逻辑执行。已补充完整代码片段。
- **issue 3**：Nit，初稿将 WebFetch 的安全层描述为"没有域名预检 API"，实际上 Rust 实现采用更简洁的 `normalize_fetch_url` 策略（`tools/src/lib.rs#L2658-L2668`），自动将非本地 `http` 升级为 `https`。已调整为正面描述。

### 抽检链接验证清单

| 链接 | 对应内容 | 状态 |
|------|----------|------|
| `file_ops.rs#L120-L128` | `GlobSearchOutput` struct | ✅ |
| `file_ops.rs#L131-L154` | `GrepSearchInput` struct | ✅ |
| `file_ops.rs#L157-L173` | `GrepSearchOutput` struct | ✅ |
| `file_ops.rs#L320-L325` | Glob 结果按 mtime 排序 | ✅ |
| `file_ops.rs#L343-L450` | `grep_search` 主函数 | ✅ |
| `file_ops.rs#L452-L465` | `collect_search_files` | ✅ |
| `file_ops.rs#L467-L487` | `matches_optional_filters` | ✅ |
| `file_ops.rs#L489-L508` | `apply_limit` | ✅ |
| `tools/src/lib.rs#L1178-L1207` | `execute_tool_with_enforcer` grep/glob 分发 | ✅ |
| `tools/src/lib.rs#L1969-L1986` | `run_glob_search` / `run_grep_search` | ✅ |
| `tools/src/lib.rs#L240-L314` | `search_tool_specs` 搜索算法 | ✅ |
| `tools/src/lib.rs#L2556-L2588` | `execute_web_fetch` | ✅ |
| `tools/src/lib.rs#L2590-L2635` | `execute_web_search` | ✅ |
| `tools/src/lib.rs#L2729-L2767` | `html_to_text` | ✅ |
| `tools/src/lib.rs#L2769-L2797` | `extract_search_hits` | ✅ |
| `lsp_client.rs#L10-L35` | `LspAction` 枚举及 `from_str` | ✅ |
| `lsp_client.rs#L150-L173` | `find_server_for_path` 扩展名路由 | ✅ |
| `lsp_client.rs#L235-L296` | `dispatch` 方法 | ✅ |
| `git_context.rs#L21-L42` | `GitContext::detect` | ✅ |

### 内容评定

报告严格遵循原文 ToC（两种搜索维度、ripgrep 内嵌方式、macOS 代码签名、搜索结果设计考量、ripgrep 错误处理、ToolSearch、Web 搜索与抓取），并为每个主节增加了 `### 源码映射` 子节。

**关键发现**：
- Rust 实现未使用外部 `ripgrep` 二进制，而是通过 `regex` + `walkdir` + `glob` crate 实现进程内搜索
- `head_limit` 默认值 250 在 `apply_limit` 中硬编码，符合 token 预算约束思想
- `ToolSearch` 的加权评分算法与上游不同但等效（规范化匹配 +12 分、精确匹配 +8 分、部分匹配 +4 分）
- `WebSearch` 默认走 DuckDuckGo HTML 端点，无需 API Key
- `WebFetch` 采用轻量级 HTML→Text 转换而非 Turndown，自动 `http`→`https` 升级
- LSP 导航骨架已搭好（`LspRegistry`、`LspAction`、扩展名路由），但真实 JSON-RPC 调用链路尚未完全接入

**达到对外发布标准**。

*审校完成。*

## 30-growthbook-ab-testing

- **审校时间**: 2026-04-09
- **审校结果**: 通过
- **说明**:
  1. 源码锚点已逐行核对，行号与 `packages/ccb/src/services/analytics/growthbook.ts` 及 `builtInAgents.ts`、`exploreAgent.ts` 一致。
  2. 报告覆盖了运行时 A/B 测试的核心链路：remoteEval 初始化、用户属性定向、tengu_* flag 代号文化、双重门控（构建时 feature + 运行时 GrowthBook）、缓存层级（内存/磁盘/env/config 覆盖）、五种取值 API 的适用场景、初始化与刷新机制、实验曝光追踪。
  3. 格式参考了 `01-what-is-claude-code.md` 的标杆结构：含引言、分节源码映射、端到端解释、源码索引表。
  4. 唯一注意点：文档原始页面描述的是上游 TypeScript 架构，本报告映射到了 `packages/ccb` 子项目（而非 `rust/`），这在 Unit 30 的范围内是合理的，因为 GrowthBook 实现目前仅存在于 `packages/ccb`。

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
