# Permission Model 技术报告

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/safety/permission-model) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 一句话定义

`claw-code` 的权限模型是一个**从 ReadOnly 到 Allow 的五级引擎**，通过模式等级、规则列表（Allow/Ask/Deny）和 Hook 覆盖三层决策，控制 AI 在终端中对文件系统、Shell、网络等敏感能力的访问边界。权限判定发生在每次 `ToolUse` 被模型请求之后、实际执行之前。

---

## 权限模型全景

### 五级 PermissionMode

核心枚举定义在 [`runtime/src/permissions.rs#L9-L16`](/rust/crates/runtime/src/permissions.rs#L9-L16)：

```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,
    Allow,
}
```

| 模式 | 含义 | 典型场景 |
| --- | --- | --- |
| `ReadOnly` | 只允许读取操作 | 审查代码、搜索文件、查看日志 |
| `WorkspaceWrite` | 允许在当前工作区内写文件 | 自动修复代码、生成文件 |
| `DangerFullAccess` | 允许所有操作（包括外部路径、危险命令） | 系统级调试、全局安装 |
| `Prompt` | 每次敏感操作都向用户确认 | 默认安全模式 |
| `Allow` | 完全放行，不做任何确认 | CI/CD、信任环境 |

配置层面支持别名映射。参见 [`runtime/src/config.rs#L851-L860`](/rust/crates/runtime/src/config.rs#L851-L860)：

```rust
fn parse_permission_mode_label(mode: &str, context: &str) -> Result<ResolvedPermissionMode, ConfigError> {
    match mode {
        "default" | "plan" | "read-only" => Ok(ResolvedPermissionMode::ReadOnly),
        "acceptEdits" | "auto" | "workspace-write" => Ok(ResolvedPermissionMode::WorkspaceWrite),
        "dontAsk" | "danger-full-access" => Ok(ResolvedPermissionMode::DangerFullAccess),
        ...
    }
}
```

这意味着用户在配置文件中写 `permissionMode: "acceptEdits"` 时，实际生效的是 `WorkspaceWrite`。

---

## 权限判定流程：authorize_with_context

每一次工具调用都要经过 `PermissionPolicy::authorize_with_context` 方法判定。完整实现位于 [`runtime/src/permissions.rs#L175-L292`](/rust/crates/runtime/src/permissions.rs#L175-L292)。

### 判定优先级（从高到低）

```
1. Deny 规则（静态规则中最优先）
2. Hook Override（Deny / Ask / Allow）
3. Ask 规则
4. Allow 规则 / 模式匹配
5. 模式升级弹窗（Prompt 模式或跨级升级）
6. 默认 Deny
```

#### 1. Deny 规则短路

```rust
if let Some(rule) = Self::find_matching_rule(&self.deny_rules, tool_name, input) {
    return PermissionOutcome::Deny {
        reason: format!("Permission to use {tool_name} has been denied by rule '{}'", rule.raw),
    };
}
```

无论当前处于什么模式、Hook 给了什么覆盖，**Deny 规则永远最先检查**，一旦匹配立即拒绝。

#### 2. Hook Override

`PermissionContext` 可以携带 `PermissionOverride`（Deny / Ask / Allow），由 `PreToolUse` hook 注入。这是运行时动态策略的入口。参见 [`runtime/src/permissions.rs#L31-L36`](/rust/crates/runtime/src/permissions.rs#L31-L36)：

```rust
pub enum PermissionOverride {
    Allow,
    Deny,
    Ask,
}
```

在 `conversation.rs` 的 `run_turn` 中，每次工具调用前都会执行 `run_pre_tool_use_hook`，并将其输出转换为 `PermissionContext`：

```rust
let permission_context = PermissionContext::new(
    pre_hook_result.permission_override(),
    pre_hook_result.permission_reason().map(ToOwned::to_owned),
);
```

参见 [`runtime/src/conversation.rs#L393-L398`](/rust/crates/runtime/src/conversation.rs#L393-L398)。

Hook 的三种覆盖行为：

- **Deny**：直接返回 `PermissionOutcome::Deny`，reason 使用 hook 的消息。
- **Ask**：强制进入弹窗流程，即使当前模式已经允许该操作。
- **Allow**：跳过模式匹配，但仍需受 Ask 规则约束。如果存在匹配的 Ask 规则，还是会弹窗。

这体现了设计上的一个重要原则：**Allow 可以提升权限，但不能绕过 Ask 规则**。测试用例 [`hook_allow_still_respects_ask_rules`](/rust/crates/runtime/src/permissions.rs#L615-L642) 明确验证了这一点。

#### 3. Ask 规则

如果工具匹配了 Ask 规则，系统会调用 `prompt_or_deny`：

```rust
if let Some(rule) = ask_rule {
    let reason = format!("tool '{tool_name}' requires approval due to ask rule '{}'", rule.raw);
    return Self::prompt_or_deny(tool_name, input, current_mode, required_mode, Some(reason), prompter);
}
```

Ask 规则是一种**强制确认策略**，常用于：
- 某些高风险命令（如 `bash(rm -rf:*)`）即使在 `DangerFullAccess` 模式下也需要确认
- 对个人敏感目录的访问（如 `read_file(~/.ssh:*)`）

#### 4. Allow 规则 / 模式匹配

如果没有 Ask 规则命中，则检查以下任一条件：

- 匹配了 Allow 规则
- 当前模式是 `Allow`
- 当前模式等级 `>=` 工具所需模式等级

```rust
if allow_rule.is_some()
    || current_mode == PermissionMode::Allow
    || current_mode >= required_mode
{
    return PermissionOutcome::Allow;
}
```

注意 `PermissionMode` 实现了 `PartialOrd` 和 `Ord`，因此可以直接用 `>=` 比较。其排序由枚举定义顺序决定：`ReadOnly < WorkspaceWrite < DangerFullAccess < Prompt < Allow`。

#### 5. 模式升级弹窗

当当前模式不够，且用户配置了 `Prompt` 模式，或从 `WorkspaceWrite` 升级到 `DangerFullAccess` 时，系统会尝试弹窗请求确认：

```rust
if current_mode == PermissionMode::Prompt
    || (current_mode == PermissionMode::WorkspaceWrite
        && required_mode == PermissionMode::DangerFullAccess)
{
    let reason = Some(format!(
        "tool '{tool_name}' requires approval to escalate from {} to {}",
        current_mode.as_str(),
        required_mode.as_str()
    ));
    return Self::prompt_or_deny(...);
}
```

#### 6. 默认 Deny

以上条件都不满足时，返回明确的 Deny：

```rust
PermissionOutcome::Deny {
    reason: format!(
        "tool '{tool_name}' requires {} permission; current mode is {}",
        required_mode.as_str(),
        current_mode.as_str()
    ),
}
```

---

## 规则语法与匹配机制

### Allow / Deny / Ask 规则配置

规则存储在 `RuntimePermissionRuleConfig` 中，结构如下（[`runtime/src/config.rs#L87-L91`](/rust/crates/runtime/src/config.rs#L87-L91)）：

```rust
pub struct RuntimePermissionRuleConfig {
    allow: Vec<String>,
    deny: Vec<String>,
    ask: Vec<String>,
}
```

从配置文件解析的位置在 [`runtime/src/config.rs#L780-L795`](/rust/crates/runtime/src/config.rs#L780-L795)：

```json
{
  "permissions": {
    "defaultMode": "prompt",
    "allow": ["bash(git:*)"],
    "deny": ["bash(rm -rf:*)", "write_file(/etc:*)"],
    "ask": ["bash(curl:*)"]
  }
}
```

### 规则语法

规则解析器在 [`runtime/src/permissions.rs#L350-L401`](/rust/crates/runtime/src/permissions.rs#L350-L401)：

- `Bash` — 匹配工具名 `Bash`，对任意输入放行
- `bash(git:*)` — 匹配工具 `bash`，且输入中提取的 `command` 字段以 `git:` 开头
- `bash(rm -rf:*)` — 匹配 `bash` 工具，输入中以 `rm -rf:` 开头
- `write_file(/etc/passwd)` — 精确匹配 `path` 为 `/etc/passwd`

解析逻辑会提取括号内的内容，并支持三种匹配器：

```rust
enum PermissionRuleMatcher {
    Any,
    Exact(String),
    Prefix(String),
}
```

提取输入中的 subject 时，系统会尝试从 JSON 参数中读取以下字段（按顺序）：

```rust
["command", "path", "file_path", "filePath", "notebook_path", "notebookPath",
 "url", "pattern", "code", "message"]
```

参见 [`runtime/src/permissions.rs#L447-L466`](/rust/crates/runtime/src/permissions.rs#L447-L466)。如果 JSON 解析失败或没有匹配字段，则回退到整个输入字符串。

---

## PermissionEnforcer：执行层的权限门控

`PermissionPolicy` 负责抽象判定，而 `PermissionEnforcer` 负责将判定结果转换为执行层可消费的结构。定义在 [`runtime/src/permission_enforcer.rs#L27-L139`](/rust/crates/runtime/src/permission_enforcer.rs#L27-L139)。

### EnforcementResult

```rust
pub enum EnforcementResult {
    Allowed,
    Denied { tool: String, active_mode: String, required_mode: String, reason: String },
}
```

### check 方法

```rust
pub fn check(&self, tool_name: &str, input: &str) -> EnforcementResult {
    if self.policy.active_mode() == PermissionMode::Prompt {
        return EnforcementResult::Allowed;
    }
    let outcome = self.policy.authorize(tool_name, input, None);
    ...
}
```

注意：**Prompt 模式下 `check` 直接返回 `Allowed`**，因为交互式提示逻辑由 `ConversationRuntime` 在上层处理（通过 `PermissionPrompter` trait）。`PermissionEnforcer` 本身不持有 prompter，只做非交互式判定。

### 文件写边界检查 check_file_write

[`runtime/src/permission_enforcer.rs#L74-L108`](/rust/crates/runtime/src/permission_enforcer.rs#L74-L108)：

```rust
pub fn check_file_write(&self, path: &str, workspace_root: &str) -> EnforcementResult {
    match mode {
        PermissionMode::ReadOnly => EnforcementResult::Denied { ... },
        PermissionMode::WorkspaceWrite => {
            if is_within_workspace(path, workspace_root) {
                EnforcementResult::Allowed
            } else {
                EnforcementResult::Denied {
                    reason: format!("path '{}' is outside workspace root '{}'", path, workspace_root),
                    ...
                }
            }
        }
        PermissionMode::Allow | PermissionMode::DangerFullAccess => EnforcementResult::Allowed,
        PermissionMode::Prompt => EnforcementResult::Denied { reason: "file write requires confirmation in prompt mode".to_owned(), ... },
    }
}
```

工作区边界检查通过字符串前缀实现：

```rust
fn is_within_workspace(path: &str, workspace_root: &str) -> bool {
    let normalized = if path.starts_with('/') { path.to_owned() } else { format!("{workspace_root}/{path}") };
    let root = if workspace_root.ends_with('/') { workspace_root.to_owned() } else { format!("{workspace_root}/") };
    normalized.starts_with(&root) || normalized == workspace_root.trim_end_matches('/')
}
```

参见 [`runtime/src/permission_enforcer.rs#L143-L157`](/rust/crates/runtime/src/permission_enforcer.rs#L143-L157)。

### Bash 只读命令启发式检查

[`runtime/src/permission_enforcer.rs#L111-L139`](/rust/crates/runtime/src/permission_enforcer.rs#L111-L139)：

在 `ReadOnly` 模式下，Bash 命令并非一概拒绝。`check_bash` 使用 `is_read_only_command` 做启发式检查，允许白名单内的一组安全命令：

```rust
fn is_read_only_command(command: &str) -> bool {
    let first_token = command.split_whitespace().next()...;
    matches!(first_token, "cat" | "ls" | "grep" | "git" | "jq" | ...)
        && !command.contains("-i ")
        && !command.contains("--in-place")
        && !command.contains(" > ")
        && !command.contains(" >> ")
}
```

同时它会拦截重定向（`>`、`>>`）和原地编辑（`sed -i`、`--in-place`），防止只读白名单被绕过。

---

## Sandbox：权限模型之外的第二层隔离

除了权限模式，`claw-code` 还提供基于 Linux namespace 的运行时沙箱。核心代码在 [`runtime/src/sandbox.rs`](/rust/crates/runtime/src/sandbox.rs)。

### FilesystemIsolationMode

```rust
pub enum FilesystemIsolationMode {
    Off,
    WorkspaceOnly,
    AllowList,
}
```

参见 [`runtime/src/sandbox.rs#L9-L15`](/rust/crates/runtime/src/sandbox.rs#L9-L15)。

- **Off**：不限制文件系统访问
- **WorkspaceOnly**：只允许访问工作区目录
- **AllowList**：只允许访问配置中显式声明的挂载点

### Linux Sandbox 命令构建

当系统检测到运行在 Linux 上且 `unshare --user` 可用时，会构建如下命令：

```rust
LinuxSandboxCommand {
    program: "unshare".to_string(),
    args: vec!["--user", "--map-root-user", "--mount", "--ipc", "--pid", "--uts", "--fork", "--net", "sh", "-lc", command],
    env: vec![
        ("HOME".to_string(), sandbox_home),
        ("TMPDIR".to_string(), sandbox_tmp),
        ("CLAWD_SANDBOX_FILESYSTEM_MODE".to_string(), status.filesystem_mode.as_str().to_string()),
        ("CLAWD_SANDBOX_ALLOWED_MOUNTS".to_string(), status.allowed_mounts.join(":")),
    ],
}
```

参见 [`runtime/src/sandbox.rs#L211-L262`](/rust/crates/runtime/src/sandbox.rs#L211-L262)。

权限模型决定**是否允许调用某工具**，而沙箱决定**该工具在操作系统层面能访问哪些资源**。两者是互补关系：权限是策略层，沙箱是执行层隔离。

---

## 用户确认交互：PermissionPrompter

当判定流程要求用户确认时，系统通过 `PermissionPrompter` trait 发起交互。定义在 [`runtime/src/permissions.rs#L86-L88`](/rust/crates/runtime/src/permissions.rs#L86-L88)：

```rust
pub trait PermissionPrompter {
    fn decide(&mut self, request: &PermissionRequest) -> PermissionPromptDecision;
}
```

### PermissionRequest 结构

```rust
pub struct PermissionRequest {
    pub tool_name: String,
    pub input: String,
    pub current_mode: PermissionMode,
    pub required_mode: PermissionMode,
    pub reason: Option<String>,
}
```

### 在 conversation.rs 中的集成

`run_turn` 的每次 `ToolUse` 处理中，权限判定的完整链路如下：

1. 运行 `PreToolUse` hook，获取覆盖和可能的取消/失败状态
2. 构建 `PermissionContext`
3. 调用 `authorize_with_context`
4. 若需要 prompter，传入 `prompter.as_mut()`
5. 若结果为 `Allow`，执行工具；若为 `Deny`，将拒绝原因作为 `ToolResult` 返回给模型

代码参见 [`runtime/src/conversation.rs#L386-L417`](/rust/crates/runtime/src/conversation.rs#L386-L417)。

### prompt_or_deny 的具体行为

```rust
fn prompt_or_deny(...) -> PermissionOutcome {
    let request = PermissionRequest { tool_name: tool_name.to_string(), input: input.to_string(), current_mode, required_mode, reason: reason.clone() };
    match prompter.as_mut() {
        Some(prompter) => match prompter.decide(&request) {
            PermissionPromptDecision::Allow => PermissionOutcome::Allow,
            PermissionPromptDecision::Deny { reason } => PermissionOutcome::Deny { reason },
        },
        None => PermissionOutcome::Deny { reason: reason.unwrap_or_else(|| format!("tool '{tool_name}' requires approval to run while mode is {}", current_mode.as_str())) },
    }
}
```

参见 [`runtime/src/permissions.rs#L294-L324`](/rust/crates/runtime/src/permissions.rs#L294-L324)。

如果调用方没有提供 prompter（例如非交互式运行或某些测试场景），需要弹窗时会直接降级为 Deny。

---

## 测试覆盖

权限系统拥有非常完整的单元测试，覆盖了各条判定分支：

| 测试名 | 所在文件 | 验证点 |
| --- | --- | --- |
| `allows_tools_when_active_mode_meets_requirement` | permissions.rs | 模式匹配放行 |
| `denies_read_only_escalations_without_prompt` | permissions.rs | 低模式直接拒绝 |
| `prompts_for_workspace_write_to_danger_full_access_escalation` | permissions.rs | 升级弹窗 |
| `applies_rule_based_denials_and_allows` | permissions.rs | 规则生效 |
| `ask_rules_force_prompt_even_when_mode_allows` | permissions.rs | Ask 规则强制确认 |
| `hook_allow_still_respects_ask_rules` | permissions.rs | Allow 覆盖不绕过 Ask |
| `hook_deny_short_circuits_permission_flow` | permissions.rs | Hook Deny 立即拒绝 |
| `hook_ask_forces_prompt` | permissions.rs | Hook Ask 强制弹窗 |
| `allow_mode_permits_everything` | permission_enforcer.rs | Allow 模式全放行 |
| `read_only_denies_writes` | permission_enforcer.rs | ReadOnly 拒绝写操作 |
| `read_only_allows_read_commands` | permission_enforcer.rs | Bash 白名单放行 |
| `workspace_write_allows_within_workspace` | permission_enforcer.rs | 工作区边界内放行 |
| `workspace_write_denies_outside_workspace` | permission_enforcer.rs | 工作区边界外拒绝 |
| `prompt_mode_denies_without_prompter` | permission_enforcer.rs | 无 prompter 时拒绝 |
| `records_denied_tool_results_when_prompt_rejects` | conversation.rs | 弹窗拒绝后的 ToolResult |
| `denies_tool_use_when_pre_tool_hook_blocks` | conversation.rs | Hook 拦截工具执行 |

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode`、`PermissionPolicy`、`authorize_with_context`、规则解析 |
| [`/rust/crates/runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) | `PermissionEnforcer`、文件写边界检查、Bash 只读启发式 |
| [`/rust/crates/runtime/src/sandbox.rs`](/rust/crates/runtime/src/sandbox.rs) | `SandboxConfig`、`FilesystemIsolationMode`、Linux namespace 沙箱命令构建 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `run_turn` 中的权限判定与 `PermissionPrompter` 集成 |
| [`/rust/crates/runtime/src/config.rs`](/rust/crates/runtime/src/config.rs) | 权限模式与规则、沙箱配置的 JSON 解析和别名映射 |
