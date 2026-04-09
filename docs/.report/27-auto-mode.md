# Auto Mode — AI 分类器驱动的自主执行模式

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/safety/auto-mode) 对 Auto Mode 的介绍，结合 `claw-code`（Claude Code Rust 重写版）的源码实现，深入解析权限分类器、自动判定逻辑与执行边界。
> 文末附 [源码索引](#源码索引)。

---

## 一句话定义

**Auto Mode 是一种让用户预先设定信任边界、从而将低危操作交由 AI 自主执行的权限模式。** 它不是"全自动无限制"，而是通过层级化的 `PermissionMode` 与细粒度规则（allow/deny/ask），在不需要逐次弹窗确认的前提下完成大部分编码任务。

---

## 权限模型：五级 `PermissionMode`

`claw-code` 的权限核心定义在 [`runtime/src/permissions.rs#L9-L15`](/rust/crates/runtime/src/permissions.rs#L9-L15)：

```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,
    Allow,
}
```

| 模式 | 含义 | 典型行为 |
| --- | --- | --- |
| **ReadOnly** | 只读 | 允许 `read_file`、`grep`、`ls`、`cat` 等纯读取操作 |
| **WorkspaceWrite** | 工作区可写 | 允许在工作区内修改文件、执行常规 bash（受限） |
| **DangerFullAccess** | 危险操作开放 | 允许跨工作区文件写入、任意 bash（仍受规则约束） |
| **Prompt** | 每次确认 | 任何需要升级权限的操作都会触发交互式确认 |
| **Allow** | 完全放行 | 所有已知工具默认允许（仍受 deny/ask 规则约束） |

从排列顺序可以看出，这五级是**偏序可比较**的：`Allow > DangerFullAccess > WorkspaceWrite > ReadOnly`。这正是 [`authorize_with_context`](/rust/crates/runtime/src/permissions.rs#L175-L292) 中进行模式比较的基础：

```rust
if allow_rule.is_some()
    || current_mode == PermissionMode::Allow
    || current_mode >= required_mode
{
    return PermissionOutcome::Allow;
}
```

参见 [`permissions.rs#L259-L263`](/rust/crates/runtime/src/permissions.rs#L259-L263)。

---

## Auto Mode 在配置层如何映射

用户在 `.claw.json` 或 `.claw/settings.json` 中写 `"permissions.defaultMode": "auto"` 时，配置解析器会将其映射为 [`ResolvedPermissionMode::WorkspaceWrite`](/rust/crates/runtime/src/config.rs#L857)。

配置解码逻辑在 [`runtime/src/config.rs#L851-L863`](/rust/crates/runtime/src/config.rs#L851-L863)：

```rust
fn parse_permission_mode_label(mode: &str, context: &str) -> Result<ResolvedPermissionMode, ConfigError> {
    match mode {
        "default" | "plan" | "read-only" => Ok(ResolvedPermissionMode::ReadOnly),
        "acceptEdits" | "auto" | "workspace-write" => Ok(ResolvedPermissionMode::WorkspaceWrite),
        "dontAsk" | "danger-full-access" => Ok(ResolvedPermissionMode::DangerFullAccess),
        other => Err(ConfigError::Parse(format!(...))),
    }
}
```

注意上游文档的 `"acceptEdits"` 与 `"auto"` 在 `claw-code` 中等价——都落入 `WorkspaceWrite`。这说明 **Auto Mode 的本质并不是"全部自动"，而是"在信任层级内自动"**：只读操作和常规工作区修改无需确认，但 bash、跨区写入、破坏性命令仍受策略和显式规则约束。

---

## 授权判定流程：`authorize_with_context`

当模型在一次 turn 中发起 `ToolUse` 时，[`conversation.rs`](/rust/crates/runtime/src/conversation.rs) 的 `run_turn` 会依次经过 hook 处理、权限判定、工具执行三阶段。权限判定的入口在 [`conversation.rs#L401-L410`](/rust/crates/runtime/src/conversation.rs#L401-L410) 附近：

```rust
let permission_outcome = if pre_hook_result.is_cancelled() { ... }
else if let Some(prompt) = prompter.as_mut() {
    self.permission_policy.authorize_with_context(
        &tool_name,
        &effective_input,
        &permission_context,
        Some(*prompt),
    )
} else {
    self.permission_policy.authorize_with_context(
        &tool_name,
        &effective_input,
        &permission_context,
        None,
    )
};
```

`authorize_with_context` 的判定顺序是**规则优先于模式**：

1. **deny 规则** —— 一旦匹配，立即拒绝（不可_override）。
2. **hook override** —— `PermissionContext` 中的 `Allow`/`Ask`/`Deny` 来自 `pre_tool_use` hook。
3. **ask 规则** —— 无论当前模式是什么，匹配即强制进入确认流程。
4. **allow 规则 / 模式检查** —— `allow_rule` 存在、模式为 `Allow`、或 `current_mode >= required_mode` 时允许。
5. **Prompt 模式或权限升级** —— 若当前模式不足且存在 prompter，则交互确认；否则直接拒绝。

完整实现参见 [`permissions.rs#L182-L292`](/rust/crates/runtime/src/permissions.rs#L182-L292)。

---

## 规则引擎：allow / deny / ask 的语法

`PermissionPolicy` 维护三个规则列表：

```rust
allow_rules: Vec<PermissionRule>,
deny_rules: Vec<PermissionRule>,
ask_rules: Vec<PermissionRule>,
```

参见 [`permissions.rs#L102-L104`](/rust/crates/runtime/src/permissions.rs#L102-L104)。

一条规则的基本语法是 `toolName(匹配器)`，例如：

- `bash(git:*)` —— 只允许/拒绝/要求确认以 `git` 开头的 bash 命令
- `bash(rm -rf:*)` —— 精确匹配 `rm -rf` 前缀
- `read_file(*)` —— 针对 `read_file` 的所有输入

匹配器解析在 [`permissions.rs#L350-L402`](/rust/crates/runtime/src/permissions.rs#L350-L402)。匹配时会先从 tool input 中提取**权限主体（subject）**：优先取 JSON 中的 `command`、`path`、`file_path`、`url`、`pattern` 等字段；若 input 不是 JSON 或不含这些键，则退化为整个 input 字符串。参见 [`permissions.rs#L447-L469`](/rust/crates/runtime/src/permissions.rs#L447-L469)。

测试用例 [`permissions.rs#L569-L587`](/rust/crates/runtime/src/permissions.rs#L569-L587) 展示了 `allow/deny` 规则的实际效果：

```rust
let rules = RuntimePermissionRuleConfig::new(
    vec!["bash(git:*)".to_string()],
    vec!["bash(rm -rf:*)".to_string()],
    Vec::new(),
);
let policy = PermissionPolicy::new(PermissionMode::ReadOnly)
    .with_tool_requirement("bash", PermissionMode::DangerFullAccess)
    .with_permission_rules(&rules);

assert_eq!(policy.authorize("bash", r#"{"command":"git status"}"#, None), PermissionOutcome::Allow);
// rm -rf 被 deny 规则拦截
```

---

## Bash 命令的二次分类与验证

Auto Mode 下 Bash 不是无条件放行。`bash_validation.rs` 实现了一套与上游 `BashTool` 对齐的验证管道，包含五个阶段：

| 阶段 | 函数 | 说明 |
| --- | --- | --- |
| `readOnlyValidation` | `validate_read_only` | ReadOnly 模式下阻止写入命令 |
| `modeValidation` | `validate_mode` | WorkspaceWrite 下警告系统路径目标 |
| `sedValidation` | `validate_sed` | 阻止 `sed -i` 在 ReadOnly 模式下运行 |
| `destructiveCommandWarning` | `check_destructive` | 对 `rm -rf /`、`mkfs` 等发出警告 |
| `pathValidation` | `validate_paths` | 检测 `../`、`~/` 等可疑路径 |

整套管道的入口是 [`validate_command`](/rust/crates/runtime/src/bash_validation.rs#L594-L615)：

```rust
pub fn validate_command(command: &str, mode: PermissionMode, workspace: &Path) -> ValidationResult {
    let result = validate_mode(command, mode);
    if result != ValidationResult::Allow { return result; }

    let result = validate_sed(command, mode);
    if result != ValidationResult::Allow { return result; }

    let result = check_destructive(command);
    if result != ValidationResult::Allow { return result; }

    validate_paths(command, workspace)
}
```

### 命令语义分类器 `classify_command`

`bash_validation.rs` 还包含一个轻量命令语义分类器，用于快速识别命令意图：

```rust
pub enum CommandIntent {
    ReadOnly,
    Write,
    Destructive,
    Network,
    ProcessManagement,
    PackageManagement,
    SystemAdmin,
    Unknown,
}
```

参见 [`bash_validation.rs#L27-L45`](/rust/crates/runtime/src/bash_validation.rs#L27-L45)。

分类逻辑通过提取首词（忽略环境变量和 `sudo`）与预定义白名单/黑名单比对来实现。例如：

- `ls`、`cat`、`grep` → `ReadOnly`
- `cp`、`mv`、`mkdir` → `Write`
- `rm`、`shred` → `Destructive`
- `curl`、`wget`、`ssh` → `Network`

代码在 [`bash_validation.rs#L533-L575`](/rust/crates/runtime/src/bash_validation.rs#L533-L575)。配合 `is_read_only_command` 的启发式判断（[`permission_enforcer.rs#L160-L238`](/rust/crates/runtime/src/permission_enforcer.rs#L160-L238)），ReadOnly 模式下也能放行 `cargo`、`git`、`python3` 等开发常用命令——只要它们不包含重定向或 `-i`/`--in-place` 标志。

---

## `PermissionEnforcer`：策略到执行 gate 的桥接

`permission_enforcer.rs` 在 `PermissionPolicy` 之上增加了一层运行时的执行门控。它在 `Prompt` 模式下有特殊处理：由于 enforcer 本身不带交互式 prompter，它会直接返回 `Allowed`，将确认职责交给上层（即 `conversation.rs` 中的 prompter 分支）。参见 [`permission_enforcer.rs#L38-L44`](/rust/crates/runtime/src/permission_enforcer.rs#L38-L44)：

```rust
pub fn check(&self, tool_name: &str, input: &str) -> EnforcementResult {
    if self.policy.active_mode() == PermissionMode::Prompt {
        return EnforcementResult::Allowed;
    }
    // ...
}
```

此外，`PermissionEnforcer` 还提供了针对具体能力的快捷检查：

- `check_file_write(path, workspace_root)` —— 检查文件写入是否越界（[`L74-L108`](/rust/crates/runtime/src/permission_enforcer.rs#L74-L108)）
- `check_bash(command)` —— 检查 bash 在当前模式下是否被允许（[`L111-L139`](/rust/crates/runtime/src/permission_enforcer.rs#L111-L139)）

`check_bash` 在 `ReadOnly` 模式下会调用 `is_read_only_command` 进行保守白名单过滤；在 `WorkspaceWrite` 及以上则直接放行。

---

## hook 在 Auto Mode 中的角色

`pre_tool_use` hook 为 Auto Mode 提供了**动态策略注入**通道。hook 可以通过标准输出返回 `CLAW_PERMISSION=allow`、`CLAW_PERMISSION=deny` 或 `CLAW_PERMISSION=ask`，从而影响 `authorize_with_context` 的决策。

`PermissionContext` 的构造逻辑在 [`conversation.rs#L392-L396`](/rust/crates/runtime/src/conversation.rs#L392-L396) 附近：

```rust
let permission_context = PermissionContext::new(
    pre_hook_result.permission_override(),
    pre_hook_result.permission_reason().map(ToOwned::to_owned),
);
```

测试用例 [`permissions.rs#L614-L642`](/rust/crates/runtime/src/permissions.rs#L614-L642) 验证了 hook `Allow` 仍会被 ask 规则覆盖：

```rust
let context = PermissionContext::new(
    Some(PermissionOverride::Allow),
    Some("hook approved".to_string()),
);
// 但 ask 规则存在时，仍要求提示确认
```

这说明 Auto Mode 的自动执行不是单点决策，而是**hook + 规则 + 模式 + 分类器**的多层防御网。

---

## 端到端示例：Auto Mode 下一次典型工具调用链

假设用户处于 `"auto"`（即 `WorkspaceWrite`），让 AI 修复一个 TypeScript 报错：

| Turn | AI 决策 | 工具/命令 | 权限判定过程 | 结果 |
| --- | --- | --- | --- | --- |
| 1 | 读取报错 | `Bash("bun run dev 2>&1 \\| head -20")` | `check_bash` → `WorkspaceWrite` 放行；无 deny/ask 规则 | 自动执行 |
| 2 | 定位文件 | `Read("src/utils/foo.ts")` | `required_mode = ReadOnly`，`current_mode >= required_mode` | 自动执行 |
| 3 | 搜索类型定义 | `Grep("interface Foo", "src/")` | 同理 | 自动执行 |
| 4 | 写入修复 | `FileEdit(...)` | `check_file_write` 确认路径在工作区内 | 自动执行 |
| 5 | 验证 | `Bash("bun run dev 2>&1 \\| head -10")` | `check_bash` 放行 | 自动执行 |

但如果第 4 步 AI 突然尝试写入 `/etc/passwd`：

- `check_file_write` 发现路径在工作区外
- `current_mode = WorkspaceWrite` < `DangerFullAccess`
- 若存在 prompter，则触发确认；否则直接拒绝

这就是 Auto Mode 的边界：它**自动的是劳动密集的低危操作**，而非所有操作。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode` 五级模型、`PermissionPolicy`、`authorize_with_context`、规则引擎 |
| [`/rust/crates/runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) | `PermissionEnforcer`、`check_bash`、`check_file_write`、运行态门控 |
| [`/rust/crates/runtime/src/bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs) | Bash 命令五阶段验证管道、`CommandIntent` 语义分类器 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `run_turn` 中的权限判定与工具执行编排 |
| [`/rust/crates/runtime/src/config.rs`](/rust/crates/runtime/src/config.rs) | 配置解析，`"auto"` → `ResolvedPermissionMode::WorkspaceWrite` 的映射 |
