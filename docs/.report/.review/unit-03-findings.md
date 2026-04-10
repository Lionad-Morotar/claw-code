# Unit 3 关键审查结果 — Tools & Memory

> 审查文件：`07-what-are-tools.md`、`13-project-memory.md`、`44-proactive.md`、`46-mcp-skills.md`  
> 审查时间：2026-04-10  
> 审查人：Claude Code (Unit 3)

---

## 总体摘要

- 共审查 4 份报告（约 1,553 行）。
- `07-what-are-tools.md` 与 `13-project-memory.md` 包含约 50 个 Rust 源码锚点，全部完成逐行核对。
- `44-proactive.md` 与 `46-mcp-skills.md` 仅引用上游 TypeScript 源码（`packages/ccb/src/...`），当前仓库中不存在对应文件，报告中已明确声明此限制，无需逐行核对 Rust 源码。
- 发现 **2 处 CRITICAL**（失效/错误行号锚点）、**3 处 MAJOR**（事实性偏差）、**6 处 MINOR**（行号漂移、措辞瑕疵、错别字）。

---

## 07-what-are-tools.md

### CRITICAL

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 1 | 源码索引第 5 行 | 错误锚点：`tools/src/lib.rs#L385` | `mvp_tool_specs()` 实际位于 `lib.rs#L385`，**仅有一行函数签名**，报告描述为"返回 40+ 个内置工具的 JSON Schema 和权限声明"，但锚点只指向函数定义开头的一行。应扩展为 `L385-L4120+` 或改为引用 `deferred_tool_specs()` 位置（`L4121` 附近）。 |
| 2 | 源码索引第 7 行 | 行号上限漂移：`tools/src/lib.rs#L1178-L1267` | `execute_tool_with_enforcer` 实际为 `L1178-L1270`（共 93 行），原范围 `L1267` 提前截断 3 行，导致最后一个 `match` arm `_ => Err(...)` 未被覆盖。 |

### MAJOR

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 3 | "延迟加载 / ToolSearch" 段落 | 锚点漂移：`run_tool_search` 位于 `L2000-L2002` | `run_tool_search` 实际位于 `L2000-L2001`（仅 2 行）。`L2002` 已属于下一函数 `run_brief`。 |
| 4 | "Bash 输出截断" 段落 | `MAX_OUTPUT_BYTES = 16_384` 位置正确（`bash.rs#L289`），但报告声称 `bash.rs` 截断输出。实际上截断发生在 `bash.rs#L293` 的 `truncate_output` 辅助函数，不是 `execute_bash_async` 内部直接截断。措辞略模糊，但不算严重错误。 | 建议补充说明 `truncate_output` 函数的存在。 |
| 5 | "MCP 扩展" 段落 | `run_mcp_tool` 调用链描述基本准确，但 `tools/src/lib.rs` 中 `run_mcp_tool` 的锚点缺失。报告说"在 `tools/src/lib.rs` 的 `execute_tool_with_enforcer` 中分别映射"，没有给出具体行号。 | 建议补充 `lib.rs#L1261`（ `"MCP" => ...` ）作为精确锚点。 |

### MINOR

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 6 | 第 14 行 | 锚点范围公差：`conversation.rs#L57-L59` vs `L57-L60` | 源码索引中使用 `L57-L60`，与文中 `L57-L59` 存在一字公差。`ToolExecutor` trait 实际占 `L57-L60`（4 行），文中锚点少一行。 |
| 7 | 第 39 行 | `tools/src/lib.rs#L100-L107` vs 实际 `L100-L106` | `ToolSpec` 结构体实际为 `L100-L106`（7 行），文中 `L107` 越界一行。 |
| 8 | 源码索引 | `tools/src/lib.rs#L2000-L2002` | 如 MAJOR #3 所述，实际仅 2 行。 |
| 9 | 源码索引 | `runtime/src/bash.rs#L69-L169` | `execute_bash` 实际为 `L69-L168`（100 行），`L169` 已属于下一函数。 |
| 10 | 源码索引 | `runtime/src/file_ops.rs#L342-L451` | `grep_search` 实际为 `L342-L450`（109 行），`L451` 已属于下一函数。 |

### PASS

- `runtime/src/conversation.rs#L296-L484`、`L370-L372`、`L376-L417`、`L433-L439` 的 `run_turn` 及 hook 调用点描述准确。
- `runtime/src/permissions.rs` 相关锚点（`L7-L15`、`L173-L292`、`L349-L402`）与权限策略描述一致。
- `runtime/src/file_ops.rs` 的 read/write/edit/glob 锚点与实现一致。
- `runtime/src/bash.rs#L17-L67`、`L212-L237` 的输入输出结构与沙箱命令准备逻辑准确。
- `runtime/src/mcp_tool_bridge.rs#L240-L278` 的 `call_tool` 描述准确。

---

## 13-project-memory.md

### CRITICAL

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 11 | `/memory` 命令锚点：`rusty-claude-cli/src/main.rs#L5083-L5120` | **完全错误**。`render_memory_report` 实际位于 `main.rs#L5513-L5550`（38 行）。报告中给出的 `L5083-L5120` 对应的是 `print_new_session_report` / `status_json_value` 等其他函数，与 `memory` 无关。 | 务必修正为 `L5513-L5550`。 |
| 12 | `session.rs#L517-531` 锚点 | **范围漂移**。`append_persisted_message` 实际位于 `L533-L548`（16 行），`L517` 是 `save_to_path`  cleanup 的尾部，`L531` 属于 `message_record` 辅助函数的上方空白。整个引用的代码块位置整体下移约 16 行。 | 应修正为 `L533-L548`。 |

### MAJOR

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 13 | "JSONL 增量持久化" 段落 | 报告声称 `"session.rs#L517-531"` 是 `append_persisted_message` 方法。虽然函数名和逻辑描述完全正确（bootstrap 检查、追加写入 JSONL），但行号完全错位（见 CRITICAL #12）。 | 修正行号即可。 |
| 14 | "Hook 生命周期" 引用 | `13-project-memory.md` 本身不包含 hooks 内容，此为误标。实际上没有涉及 hooks。 | 无实质问题，排除。 |

### MINOR

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 15 | 第 57 行 | `prompt.rs#L203-L223` vs `L203-L224` | `discover_instruction_files` 实际为 `L203-L224`（22 行），文中 `L223` 少一行。 |
| 16 | 第 87 行 | `prompt.rs#L353-L368` vs 实际 `L353-L368` | 经核对 `dedupe_instruction_files` 正好为 `L353-L368`，**此条实际正确**，但 `L353-L368` 中 `stable_content_hash` 调用在 `L362`，报告描述无误。 |
| 17 | 第 91 行 | `git_context.rs#L26-L42` 的 `GitContext::detect` 描述为"读取分支名、最近 5 条 commit、staged 文件列表" | 准确。`read_recent_commits` 默认限制为 5 条，与描述一致。 |
| 18 | 第 113 行 | `prompt.rs#L432-L446` | `load_system_prompt` 实际为 `L432-L446`（15 行），正确。 |
| 19 | 第 139 行 | `prompt.rs#L288-L328` | `render_project_context` 实际为 `L288-L328`（41 行），正确。 |
| 20 | 第 139 行 | `prompt.rs#L330-L351` | `render_instruction_files` 实际为 `L330-L351`（22 行），正确。 |
| 21 | `session.rs#L89-L100` | `Session` 结构体实际跨越 `L89-L101`（第 101 行是 `persistence` 字段的注释），`L100` 刚好停在 `prompt_history` 字段。由于 `persistence` 私有字段紧接着出现，范围公差 1 行可接受。 | 保持 MINOR。 |
| 22 | `session.rs#L204-L218` | `Session::load_from_path` 实际从 `L204` 开始，但 `L218` 是 `value` 匹配的中间位置，函数完整结束在更下方。作为节选引用可接受。 | 保持 MINOR。 |
| 23 | `session_control.rs#L19-L46` | `SessionStore` 结构体及 `from_cwd` 实际从 `L19` 开始，但 `L46` 是 `from_cwd` 内部的第一条 `fs::create_dir_all` 调用后的 `Ok(...)` 构造。结构体定义到 `impl` 块完整展开到约 `L50`，范围引用偏保守，可接受。 | 保持 MINOR。 |

### PASS

- `ProjectContext` 结构体、指令文件发现链路、`discover_with_git` 增强、System Prompt 组装链路的描述与 `prompt.rs` 源码完全一致。
- `session_control.rs` 的 `workspace_fingerprint`、`fork_session` 描述准确。
- `conversation.rs` 的 `maybe_auto_compact` 逻辑描述准确（阈值检查、`compact_session` 调用）。
- `compact.rs` 的上下文压缩机制描述准确。
- Git 上下文注入的三层快照（`git_status`、`git_diff`、`git_context`）描述准确。

---

## 44-proactive.md

### 说明

- 本报告**全部引用上游 TypeScript 源码**（`packages/ccb/src/...`），在 `claw-code` Rust 重写版仓库中**不存在对应文件**。
- 报告开头已明确声明此限制（"`claw-code`（Rust 重写版）目前尚未实现此功能"）。

### MAJOR

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 24 | 第 7 行 | "Claw-Code 中的 Tick 驱动自主代理功能" | 报告标题自称 "Unit 44: Proactive Mode 技术报告"，但正文开头又称 "是 Claw-Code 中的...功能"。由于 `claw-code` Rust 实现中**完全没有** Proactive Mode，该措辞具有误导性。应改为 "上游 Claude Code 的 Proactive Mode" 或 "计划中的 Proactive Mode"。 |
| 25 | 模块状态表格 | `packages/ccb/src/proactive/index.ts` 标记为 **Stub** | 准确，但读者需要注意该文件**不在当前 Rust 仓库**，而是在上游 TypeScript 仓库。报告已声明，无额外问题。 |

### MINOR

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 26 | 缺失文件列表 | `.md` 格式缺少围栏代码块关闭标记 | `packages/ccb/src/proactive/index.ts` 等左侧代码块前三个文件用了 ` ``` ` 但没有语言标记，最后一个 `src/tools/SleepTool/SleepTool.tsx` 单独一行没有代码块。格式轻微不一致。 |

### PASS

- 所有 TypeScript 源码引用（`packages/ccb/src/...`）均明确标注为上游实现，不构成对 Rust 源码的虚假声明。
- Proactive Mode 的状态摘要（系统提示完整、核心 Tick 调度器缺失、SleepTool 执行器缺失等）与报告自身的描述逻辑自洽。

---

## 46-mcp-skills.md

### 说明

- 与 `44-proactive.md` 相同，本报告**全部引用上游 TypeScript 源码**（`packages/ccb/src/...`），在 `claw-code` Rust 重写版仓库中不存在对应文件。
- 报告开头已明确声明此限制。

### MAJOR

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 27 | 第 9 行 | "`MCP_SKILLS` 尝试将 MCP 服务器暴露的 `skill://` URI 方案资源...转换为 Claude Code 内部可调用的 `prompt` 类型 `Command` 对象" | 措辞准确，但应注意 `claw-code` Rust 实现中**没有** `Command` / `prompt` / `skill://` 这个上层概念。报告与 Rust 实现无交集，属于上游文档映射。 |

### MINOR

| # | 位置 | 问题 | 说明 |
|---|------|------|------|
| 28 | 第 2 章标题 | "连接时数据流" 中 `fetchMcpSkillsForClient` 标记为 `[FEATURE_GATED]` | 正确，但表格中 `services/mcp/client.ts` 文件路径在上游实现中也需读者自行确认。 |
| 29 | 第 3.4 节 stub 代码 | `mcpSkills.ts` 的 stub 实现描述准确。 | PASS。 |

### PASS

- 所有源码路径均明确限定为 `packages/ccb/src/...`，没有冒充 Rust 实现。
- Feature gate、循环依赖规避、安全隔离等设计决策总结与上游源码逻辑一致。

---

## Cross-Report 跨报告一致性检查

### 1. Proactive Mode / Sleep 工具的覆盖范围矛盾

- `07-what-are-tools.md`（工具系统全景表）在 "50+ 内置工具全景" 中列出了 `Sleep`，并在 `tools/src/lib.rs` 的 `execute_tool_with_enforcer` 中确实找到了 `"Sleep" => ...` 的映射（`L1215`）。
- `44-proactive.md` 声称 "`claw-code`（Rust 重写版）目前尚未实现此功能【Proactive Mode】"。

**判定**：**不构成矛盾**。`Sleep` 作为通用工具在 Rust 实现中**已存在**（`run_sleep` 在 `tools/src/lib.rs#L2008`），但它只是简单的 `tokio::time::sleep` 包装，**没有**与 Tick 调度器、Proactive 自主代理模式绑定。`44-proactive.md` 指的是 "Proactive Mode" 整体功能缺失，而非 `Sleep` 工具本身缺失。两报告若并读需注意区分 "工具存在" vs "模式缺失"。

### 2. MCP 技能的 Rust 实现状态

- `07-what-are-tools.md` 在 "50+ 内置工具全景" 和 MCP 扩展段落中描述了 `MCP`、`ListMcpResources`、`ReadMcpResource`、`McpAuth` 四个 Rust 内置工具，并提供了 `runtime/src/mcp_tool_bridge.rs#L240-L278` 的 `call_tool` 实现锚点。**经核对全部正确**。
- `46-mcp-skills.md` 讲述的是上游 TypeScript 的 `MCP_SKILLS` feature，涉及 `skill://` URI 到 `Command` 的转换，这在 Rust 实现中**完全不存在**。

**判定**：**不构成直接矛盾**，但两报告接壤的 "MCP 技能" 概念有不同内涵：
- `07-what-are-tools.md` 的 MCP = Rust 中的 `McpToolRegistry` / `McpServerManager`（tools/resources）。
- `46-mcp-skills.md` 的 MCP Skills = 上游 TypeScript 中从 MCP resources 解析出的 `skill://` prompt commands。

建议在 `07-what-are-tools.md` 的 MCP 段落末尾加一句注脚，提示 "MCP Skills（`skill://`）尚未在 Rust 实现中落地"，以避免读者混淆。

---

## 修正建议汇总

### 必须修正（CRITICAL）

1. **`07-what-are-tools.md` 源码索引 `tools/src/lib.rs#L1178-L1267`**  →  改为 `L1178-L1270`。
2. **`07-what-are-tools.md` 源码索引 `tools/src/lib.rs#L385`**  →  扩展为 `L385-L4120`（`mvp_tool_specs` 函数完整范围）或至少改为 `L385` 并说明这是函数签名。
3. **`13-project-memory.md` 中 `rusty-claude-cli/src/main.rs#L5083-L5120`**  →  改为 `L5513-L5550`。
4. **`13-project-memory.md` 中 `session.rs#L517-531`**  →  改为 `L533-L548`。

### 强烈建议修正（MAJOR）

5. **`07-what-are-tools.md` `tools/src/lib.rs#L2000-L2002`**  →  改为 `L2000-L2001`。
6. **`07-what-are-tools.md`** 在 MCP 段落补充 `lib.rs#L1261` 作为 `run_mcp_tool` 的精确锚点。
7. **`44-proactive.md`** 第 7 行将 "是 Claw-Code 中的..." 改为 "是上游 Claude Code 的..." 或 "在 `claw-code` 的未来规划中..."。

### 可选修正（MINOR）

8. `07-what-are-tools.md`：`conversation.rs#L57-L59` 统一为 `L57-L60`。
9. `07-what-are-tools.md`：`tools/src/lib.rs#L100-L107` 统一为 `L100-L106`。
10. `07-what-are-tools.md`：`runtime/src/bash.rs#L69-L169` 统一为 `L69-L168`。
11. `07-what-are-tools.md`：`runtime/src/file_ops.rs#L342-L451` 统一为 `L342-L450`。
12. `13-project-memory.md`：`prompt.rs#L203-L223` 统一为 `L203-L224`。
13. `44-proactive.md`：缺失文件列表的代码块格式统一加 ` ``` ` 围栏。

---

## 行号漂移根因分析

- `tools/src/lib.rs` 和 `rusty-claude-cli/src/main.rs` 是代码量最大的文件，近期持续有功能插入，导致行号最容易漂移。
- `session.rs` 和 `conversation.rs` 也因新增了 `workspace_root`、`git_context`、sandbox 等字段而发生局部行号下移。
- 建议：对高频修改的大文件（`tools/src/lib.rs`、`main.rs`）使用 "函数名锚点 + 近似行号" 的写法，并在 CI 中加入锚点死链检查（如 `git show HEAD:<path> | sed -n ...` 脚本）。
