# Unit 2 — Sub-agents & Infrastructure 关键审查结果

> 审查文件：`16-sub-agents.md`、`20-hooks.md`、`47-tree-sitter-bash.md`、`32-sentry-setup.md`
> 审查日期：2026-04-10
> 审查方式：逐条核对源码锚点（`git show HEAD:<path> | sed -n 'start,endp'`）

---

## 16-sub-agents.md — 子 Agent 机制

### CRITICAL

**C01 — 锚点行号严重错位：Conversation 后处理 Hook 位置**
- 原文：`conversation.rs#L371-L375` 声称包含 **PostToolUse / PostToolUseFailure** Hook 调用。
- 实际：L371-375 仅包含 `effective_input` / `permission_context` 的构造逻辑，**不含任何 PostHook 调用**。PostHook 调用实际位于 L429 附近（`run_post_tool_use_hook` / `run_post_tool_use_failure_hook`）。
- 影响：读者依据该锚点无法找到 PostHook 的实际触发位置，属于关键流程说明错误。

### MAJOR

无。

### MINOR

**M01 — `workspace_root` 锚点未精确指向字段**
- 原文：`session.rs#L90` — "Worktree 绑定关键字段"
- 实际：L90 为 `pub version: u32,`，`workspace_root` 字段实际在 **L95**。建议改为 `session.rs#L95`。

**M02 — `with_workspace_root` 行号范围偏宽**
- 原文：`session.rs#L176-L183`
- 实际：方法体起始于 L178（doc 注释结束后）。建议调整为 `L178-L186`。

**M03 — `run_agent_job` 行号范围偏宽**
- 原文：`lib.rs#L3399-L3404`
- 实际：`run_agent_job` 函数签名为 L3402、函数体到 L3405。建议调整为 `L3402-L3405`。

**M04 — `build_agent_runtime` 行号范围偏宽**
- 原文：`lib.rs#L3403-L3428`
- 实际：函数起始于 L3406，终止于 L3427。建议调整为 `L3406-L3427`。

**M05 — `build_agent_system_prompt` 行号范围偏宽**
- 原文：`lib.rs#L3436-L3447`
- 实际：函数终止于 L3442。建议调整为 `L3436-L3442`。

**M06 — `output_contents` 锚点行号偏后**
- 原文注释内引用 `L3320-3334`
- 实际：`let output_contents = format!(...)` 起始于 **L3318**。建议调整为 `L3318-L3332`。

**M07 — `write_agent_manifest` 锚点行号偏后**
- 原文注释内引用 `L3539-L3548`
- 实际：函数起始于 **L3540**。建议调整为 `L3540-L3549`。

**M08 — `fork` 行号范围偏宽**
- 原文：`session.rs#L251-L268`
- 实际：`fork` 方法起始于 L254。建议调整为 `L254-L269`。

---

## 20-hooks.md — Hooks 扩展机制

### CRITICAL

无（该文件未单独声明 C01 那样的独立 CRITICAL，但与 16-sub-agents.md 共享同一 `conversation.rs#L371-L375` 错误引用）。

### MAJOR

**H01 — PreHook 分支说明缺少 `Prompt` 模式条件**
- 原文 2.3 节声称 PreHook 拒绝后直接进入 `PermissionOutcome::Deny`。
- 实际 `conversation.rs#L379-L411` 中，如果 `is_cancelled/denied/failed` 均不成立，**还存在 `else if let Some(prompt) = prompter` 分支**进入弹窗确认（`authorize_with_context` 带 prompter）。说明中的"否则进入正常权限策略检查"准确，但未显式指出 Prompt 模式的存在，对于理解弹窗行为不够完整。

### MINOR

**H02 — `config.rs#L598-L605` 未直接对应 `extend_unique`**
- 原文引用 `config.rs#L598-L605` 说明"去重追加策略（`extend_unique`）"。
- 实际 L598-L605 展示的是 `RuntimeHookConfig::extend` 方法，内部调用了 `extend_unique`。描述基本可接受，但严格来说锚点未覆盖 `extend_unique` 函数定义本身。

**H03 — `hooks.rs#L695-L699` 锚点未完全覆盖 kill 逻辑**
- 原文引用 `L695-L699` 说明取消后 kill 子进程。
- 实际 kill 发生在 L696，`wait_with_output` 在 L697，return 在 L698；循环 sleep 在 L702。引用范围略宽但无伤大雅。

---

## 47-tree-sitter-bash.md — Bash AST 解析技术报告

### CRITICAL

无。

### MAJOR

无。

### MINOR

**T01 — `classify_command` 行号范围偏窄**
- 原文：`bash_validation.rs#L533-L584`
- 实际：`classify_command` 签名在 L533，但 `classify_by_first_command` 延伸至 **~L588**。建议调整为 `L533-L588`。

**T02 — `authorize_with_context` 起始行偏前**
- 原文：`permissions.rs#L175-L292`
- 实际：`authorize_with_context` 函数签名为 **L176**。建议调整为 `L176-L292`。

**T03 — `PermissionMode` 起始行偏后**
- 原文：`permissions.rs#L9-L14`
- 实际：`pub enum PermissionMode` 声明在 **L8**，变体在 L9-L14。建议调整为 `L8-L14` 或保留说明。

---

## 32-sentry-setup.md — 自定义 Sentry 错误上报配置

### CRITICAL

无。

### MAJOR

**S01 — 关于 `telemetry` crate 的功能声明错误**
- 原文："Rust 生态更倾向 `tracing` + OpenTelemetry（已由 [`telemetry` crate](/rust/crates/telemetry/src/lib.rs) 实现）"
- 实际：[`rust/crates/telemetry/src/lib.rs`](rust/crates/telemetry/src/lib.rs) 是一个**自定义内部遥测系统**（`TelemetrySink`、`MemoryTelemetrySink`、`JsonlTelemetrySink`、`SessionTracer` 等），**既未使用 OpenTelemetry 协议，也未实现 `tracing` 生态集成**。该声明属于事实错误，需修正为 "已由 `telemetry` crate 实现基础 JSONL/内存遥测，但未集成 OpenTelemetry"。

### MINOR

无。

---

## 跨报告一致性（Cross-Report）

### X01 — `conversation.rs#L371-L375` 被两处报告共同误用

- **16-sub-agents.md** 第 3.1 节与 **20-hooks.md** 第 2.2 节均把该范围描述为包含 **PostToolUse** 调用。
- 实际该范围仅与 PreHook 相关；**PostHook 调用远在 L429 之后**。
- 这是一个跨报告的共同锚点错误，建议两处同步修正。

### X02 — `config.rs` 合并策略描述一字之差

- **20-hooks.md** 称用户级与项目级配置"深度合并"，使用 `extend_unique`。
- 源码 `config.rs#L598-L605` 的方法名是 `extend`（内部调用 `extend_unique`）。两处报告用词一致，但与函数名不完全对应，建议统一为 "通过 `RuntimeHookConfig::extend` 合并"。

---

## 审查统计

| 文件 | 锚点总数 | PASS | MINOR | MAJOR | CRITICAL |
|------|----------|------|-------|-------|----------|
| 16-sub-agents.md | ~31 | ~21 | 8 | 0 | 1 |
| 20-hooks.md | ~22 | ~19 | 2 | 1 | 0* |
| 47-tree-sitter-bash.md | ~9 | ~5 | 3 | 0 | 0 |
| 32-sentry-setup.md | 1 | 0 | 0 | 1 | 0 |

> *20-hooks.md 的 CRITICAL 项与 16-sub-agents.md 共享同一 `conversation.rs#L371-L375` 错误，已在 Cross-Report 中记录。

---

## 建议优先修复顺序

1. **C01 / X01**：修正 `conversation.rs#L371-L375` 的 PostHook 位置说明（影响 16-sub-agents.md 与 20-hooks.md）。
2. **S01**：修正 `telemetry` crate 的功能描述（32-sentry-setup.md）。
3. **M01-M08、T01-T03**：批量修正偏移的源码行号锚点。
