# Unit 7 关键审查结果 — Architecture + Coordination

> 审查范围：
> - `03-architecture-overview.md`
> - `37-coordinator-mode.md`
> - `18-coordinator-and-swarm.md`
> - `28-three-tier-gating.md`
>
> 审查方法：提取全部 Rust 源码锚点，使用 `git show HEAD:<path> | sed -n '<range>p'` 逐条验证；对事实性声明进行源码交叉核对。

---

## 1. `03-architecture-overview.md` 审查结果

### 1.1 锚点总体状态

全部 37 个 Rust 源码锚点所在文件均存在。但有 **5 处锚点行号严重偏离**当前 HEAD，属于 CRITICAL。

### 1.2 CRITICAL：行号偏移

| 报告中的锚点 | 报告中声称的范围 | 实际 HEAD 位置 | 偏差 |
|-------------|-----------------|----------------|------|
| `rusty-claude-cli/src/main.rs` `run_repl` | L2789-L2838 | **L3143** 起 | 严重偏离 |
| `rusty-claude-cli/src/main.rs` `prepare_turn_runtime` | L3442-L3459 | **L3819** 起 | 严重偏离 |
| `rusty-claude-cli/src/main.rs` `build_runtime` | L6235-L6309 | `build_runtime_plugin_state` 在 L6321，`build_runtime` 在 **L6726** 起 | 严重偏离 |
| `rusty-claude-cli/src/main.rs` `consume_stream` | L6531-L6695 | **L7059** 起 | 严重偏离 |
| `rusty-claude-cli/src/main.rs` `parse_args` | L371-L520 | **L396** 起 | 中度偏离 |

> 说明：`main.rs` 在近期提交中经历了大规模内容插入（新增命令行参数、内部进度报告、reasoning effort 等），导致几乎所有基于旧版本的行号锚点均向下大幅漂移。建议统一将 `main.rs` 的锚点替换为符号搜索（`grep -n 'fn run_repl'`）所得的当前行号，或改用不带行号的文件级链接。

### 1.3 MAJOR：事实准确性

- **工具数量**：报告声称 `execute_tool` "直接映射 40+ 个内置工具"。当前 HEAD 的 `execute_tool_with_enforcer`（`tools/src/lib.rs#L1174` 附近）实际包含的 `match` 分支约为 35 个显式工具名 + `_ => Err(...)`。"40+" 的说法在精确计数意义上略有夸大，但考虑到还有 MCP 动态工具，可视作文案修辞。如追求严谨，建议改为 "30+ 个显式内置工具 + MCP 动态扩展"。

### 1.4 MINOR

- **代码块示例与源码不一致**：第 42-56 行展示的 `run()` 函数代码块是高度简化的伪代码，与实际 `main.rs#L180` 起的 `run()` 函数差异较大（缺少大量 `CliAction` 分支）。由于明确标注了 `// ... 单次非交互调用` 等省略注释，定性为 MINOR 格式化/示例简化问题。
- **表格链接格式**：源码索引表格中存在不带行号的文件链接（如 `/rust/crates/api/src/client.rs`），与报告正文带行号的链接风格不统一，属于 MINOR 格式问题。

---

## 2. `37-coordinator-mode.md` 审查结果

### 2.1 锚点总体状态

全部 24 个 Rust 源码锚点均有效，符号与行号匹配良好。

### 2.2 CRITICAL

- **无**。锚点全部命中，声明的"上游未实现"也明确无误。

### 2.3 MAJOR：事实交叉核对

- **TaskStop / TaskUpdate 与 SendMessage 的映射**：文档指出这两个工具对应原始文档中的 `SendMessage`/`TaskStop`。源码中确实只存在 `TaskStop` 与 `TaskUpdate`，不存在名为 `SendMessage` 的工具。此映射在概念层面成立，但需要读者自行理解——建议可接受，无错误。
- **`Plan` subagent 的 `StructuredOutput` 工具**：`allowed_tools_for_subagent` 中 `Plan` 确实包含 `StructuredOutput`（L13-L24），报告中表格未列出，但未声称其不存在，不构成错误。

### 2.4 MINOR

- **文档引言 clarity**："Coordinator Mode 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。" 这段声明非常清晰，但后续的 Rust 源码锚点实际上展示的是"如何用现有 Rust 子 Agent 机制模拟 Coordinator Mode"，这种"上游未实现 / 但 Rust 侧有等价物"的双重叙事在快速阅读时可能产生混淆。建议增加一句过渡："以下 Rust 代码为现有 Subagent 机制，可被用于实现 Coordinator 角色的等价行为"。

---

## 3. `18-coordinator-and-swarm.md` 审查结果

### 3.1 锚点总体状态

18 个 Rust 源码锚点全部有效，符号匹配正确。

### 3.2 CRITICAL

- **无**。

### 3.3 MAJOR

- **`WorkerRegistry` 范围 end**：报告中将 `runtime/src/worker_boot.rs#L144-L531` 标记为 `WorkerRegistry` 范围。HEAD 中 `WorkerRegistry` 的 `impl` 块从 L12 开始，到 L144 左右结束主要方法；L531 实际上超出了 `WorkerRegistry` 的核心实现，进入了文件末尾的辅助函数区（如 `detect_prompt_misdelivery` 从 L640+ 开始）。严格来说 L144-L531 包含大量 `WorkerRegistry` 无关的私有函数。建议拆分为：
  - `L12-L144`：`WorkerRegistry` 结构体与核心方法
  - `L640-L700`：`detect_prompt_misdelivery`
- **TaskStatus 枚举值**：报告中列出了 `Created, Running, Completed, Failed, Stopped`。源码（`task_registry.rs#L14-L20`）中只有 `Created, Running, Completed, Failed`——**不存在 `Stopped`**。这是一个事实性错误。尽管 `TaskRegistry::stop()` 方法存在（L231 附近），但状态流转目标写的是 `TaskStatus::Failed`，而非 `Stopped`。此错误为 MAJOR。

### 3.4 MINOR

- **工具名称大小写**：报告中第 271-279 行表格将 `TaskGet` 等工具名保持首字母大写，与源码一致，很好；但正文中偶尔出现小写（如 `task-notification` XML 形式），这是原文档引用，不构成错误。

---

## 4. `28-three-tier-gating.md` 审查结果

### 4.1 锚点总体状态

6 个 Rust 源码锚点全部有效。

### 4.2 CRITICAL

- **无**。

### 4.3 MAJOR

- **对比分析表格中的 `proactive` vs `normal` 映射**：报告声称 "`USER_TYPE (ant vs external)` → `proactive` vs `normal` 模式标记"。但 `proactive` / `normal` 在 `tools/src/lib.rs#L645` 附近只是 `SendUserMessage` 工具（或类似 brief 模式）的枚举值，用于控制消息发送风格，与上游 `USER_TYPE` 的身份门控在语义上完全不同。这种映射过于牵强，有误导读者之嫌。建议删除或改为 "无直接等价映射"，并说明 `proactive` 只是工具参数值。

### 4.4 MINOR

- **第 168 行**：锚点 `rust/crates/tools/src/lib.rs#L1987-L1989` 在 Markdown 链接语法中末尾有额外的反引号 / 括号闭合问题：`L1987-L1989`](/rust/crates/tools/src/lib.rs#L1987-L1989`，渲染可能正常但源码中存在 `)` 与反引号嵌套混乱。属于 MINOR 格式问题。
- **后续文档预告**："`88 面旗帜`"、"`千面千人`" 等标题使用了反引号包裹，看起来像代码引用，但实际是文档系列标题。风格选择，可接受。

---

## 5. 跨报告矛盾（Cross-Report）

| 矛盾点 | 涉及报告 | 详情 |
|--------|---------|------|
| **Coordinator 实现状态** | `37-coordinator-mode.md` vs `18-coordinator-and-swarm.md` | `37` 报告明确说 "Coordinator Mode ... 目前尚未实现此功能"；`18` 报告说 "Coordinator 的核心逻辑目前更多存在于 TypeScript 上游 ... 而 Swarm 所需 ... 已经在 Rust 侧有了完整的运行时支撑"。两者表述在语义上一致，但 `18` 的主标题及目录结构把 Coordinator 和 Swarm 并列，易让读者误以为 Rust 中有 Coordinator 模块。建议 `18` 在开头再强调一次：Rust 侧无显式 Coordinator Mode，仅有 Agent Tool 作为底层支撑。|
| **TaskStatus 枚举值** | `18-coordinator-and-swarm.md` 内部 | 报告自身在文字中列出 `Stopped`，但源码无此枚举值，属于自相矛盾（已标记为 MAJOR）。|
| **三层门禁的 Rust 映射** | `28-three-tier-gating.md` vs `03-architecture-overview.md` | `28` 将 `proactive` 映射为 `USER_TYPE` 的等价物；`03` 完全没有提及这种映射，`03` 对 `PermissionMode` 的描述（五级模型）更为准确。`28` 中的 `proactive` 映射与 `03` 的架构描述不相容。|

---

## 6. 汇总

| 文件 | CRITICAL | MAJOR | MINOR |
|------|----------|-------|-------|
| `03-architecture-overview.md` | 5 处行号严重偏离 | 1 处（40+ 工具计数） | 2 处 |
| `37-coordinator-mode.md` | 0 | 0 | 1 处 |
| `18-coordinator-and-swarm.md` | 0 | 2 处（WorkerRegistry 范围 + TaskStatus 枚举） | 1 处 |
| `28-three-tier-gating.md` | 0 | 1 处（`proactive` 映射） | 1 处 |
| **总计** | **5** | **4** | **5** |

### 修复优先级建议

1. **最高优先级**：修正 `03-architecture-overview.md` 中 `main.rs` 的全部过时行号锚点（CRITICAL × 5）。
2. **高优先级**：修正 `18-coordinator-and-swarm.md` 中 `TaskStatus` 枚举描述（去掉 `Stopped`）。
3. **中优先级**：修正 `18-coordinator-and-swarm.md` 中 `WorkerRegistry` 的过大行号范围。
4. **中优先级**：修正 `28-three-tier-gating.md` 中 `proactive` 与 `USER_TYPE` 的不当映射。
5. 其余 MINOR 问题可在统一润色时一并处理。
