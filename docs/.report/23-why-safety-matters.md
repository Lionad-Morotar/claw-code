# AI 安全至关重要 — Claude Code 安全设计哲学

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/safety/why-safety-matters) 的安全主题，深入分析 `claw-code`（Claude Code 的 Rust 重写版）源码中的安全边界、危险操作检测、错误处理与审计日志机制。
> 文末附 [源码索引](#源码索引)。

---

## 一句话总结

当 AI 拥有完整的 shell 访问权和文件系统操作能力时，**纵深防御（Defense in Depth）** 是防止灾难性后果的唯一可靠策略。`claw-code` 通过 **五层安全门禁** 实现这一目标：**权限模型 → 沙箱隔离 → 命令验证 → Hook 拦截 → 审计追踪**。

---

## 为什么安全至关重要

Claude Code 不是运行在云端沙箱中的聊天机器人，而是**运行在本地终端中的 agentic coding system**。这意味着：

1. **完整的 Shell 访问权** — 可以执行任何用户在终端能执行的命令
2. **真实的文件系统操作** — 可以读/写/删除项目文件
3. **网络访问能力** — 可以发起 HTTP 请求、下载文件、连接远程服务
4. **进程管理能力** — 可以启动后台任务、终止进程

这种能力模型带来的安全威胁包括：

| 威胁类别 | 具体风险 | 潜在后果 |
| --- | --- | --- |
| **数据丢失** | `rm -rf` 误删关键文件 | 代码库、数据库、配置文件永久丢失 |
| **权限逃逸** | 符号链接攻击、路径遍历 | 访问工作区外敏感文件（如 `~/.ssh/id_rsa`） |
| **系统破坏** | `chmod -R 000`、`mkfs` | 系统权限混乱、磁盘数据销毁 |
| **网络攻击** | 未授权的外连请求 | 敏感数据泄露、内网渗透 |
| **资源耗尽** | Fork 炸弹、无限循环 | 系统崩溃、服务不可用 |

---

## 纵深防御架构

`claw-code` 的安全设计遵循**纵深防御**原则，不依赖单一安全措施，而是构建多层防护网。

```
┌──────────────────────────────────────────────────────────────┐
│ 第 5 层：审计追踪 (Audit Trail)                                 │
│        Session 持久化、ToolUse/ToolResult 配对追踪              │
├──────────────────────────────────────────────────────────────┤
│ 第 4 层：Hook 拦截 (Pre/Post ToolUse Hooks)                   │
│        外部策略检查、自定义拦截逻辑                            │
├──────────────────────────────────────────────────────────────┤
│ 第 3 层：命令验证 (Bash Validation Pipeline)                  │
│        只读验证、破坏性检测、路径检查、语义分类                 │
├──────────────────────────────────────────────────────────────┤
│ 第 2 层：沙箱隔离 (Sandbox Isolation)                         │
│        Linux unshare 命名空间隔离、文件系统限制                 │
├──────────────────────────────────────────────────────────────┤
│ 第 1 层：权限模型 (Permission Policy)                         │
│        五级权限模式、Allow/Deny/Ask 规则引擎                    │
└──────────────────────────────────────────────────────────────┘
```

---

## 第 1 层：权限模型（Permission Policy）

### 五级权限模式

[`runtime/src/permissions.rs#L9-L16`](/rust/crates/runtime/src/permissions.rs#L9-L15) 定义了从最严格到最宽松的五个权限级别：

```rust
pub enum PermissionMode {
    ReadOnly,        // 只读：仅允许查看、搜索、分析
    WorkspaceWrite,  // 工作区写入：允许修改项目文件，但禁止系统级操作
    DangerFullAccess, // 危险全权限：允许所有操作（包括系统级）
    Prompt,          // 提示模式：需要用户确认才能执行
    Allow,           // 允许模式：所有操作自动放行
}
```

### 权限决策流程

`PermissionPolicy::authorize_with_context`（[`permissions.rs#L175-L292`](/rust/crates/runtime/src/permissions.rs#L175-L292)）执行以下决策逻辑：

1. **Deny 规则优先** — 若匹配 deny 规则，立即拒绝（[`L182-L188`](/rust/crates/runtime/src/permissions.rs#L182-L188)）
2. **Hook 覆盖检查** — 若 Hook 返回 `Deny`，立即拒绝（[`L196-L204`](/rust/crates/runtime/src/permissions.rs#L196-L204)）
3. **Ask 规则强制** — 若匹配 ask 规则，必须弹窗确认（[`L240-L257`](/rust/crates/runtime/src/permissions.rs#L240-L257)）
4. **模式升级检查** — 若当前模式 < 所需模式，且处于 `Prompt` 或 `WorkspaceWrite` → `DangerFullAccess`，触发确认（[`L266-L283`](/rust/crates/runtime/src/permissions.rs#L266-L283)）

### 规则语法示例

```rust
// 配置示例：允许 git 查询，禁止 rm -rf，ask git 提交
let rules = RuntimePermissionRuleConfig::new(
    vec!["bash(git:*)".to_string()],      // allow: git 开头的命令
    vec!["bash(rm -rf:*)".to_string()],   // deny: rm -rf 开头的命令
    vec!["bash(git commit)".to_string()], // ask: git commit 需要确认
);
```

参见测试用例 [`permissions.rs#L568-L587`](/rust/crates/runtime/src/permissions.rs#L568-L587)。

---

## 第 2 层：沙箱隔离（Sandbox Isolation）

### Linux 命名空间隔离

[`runtime/src/sandbox.rs`](/rust/crates/runtime/src/sandbox.rs) 提供了基于 `unshare` 命令的 Linux 命名空间隔离能力：

```rust
// 构建 unshare 命令，隔离用户、挂载、IPC、PID、UTS 命名空间
let mut args = vec![
    "--user", "--map-root-user", "--mount", "--ipc", "--pid", "--uts", "--fork",
];
if network_isolation {
    args.push("--net");  // 网络隔离（可选）
}
```

参见 [`sandbox.rs#L210-L262`](/rust/crates/runtime/src/sandbox.rs#L210-L262)。

### 文件系统隔离模式

[`FilesystemIsolationMode`](/rust/crates/runtime/src/sandbox.rs#L9-L14) 定义了三种文件系统隔离级别：

```rust
pub enum FilesystemIsolationMode {
    Off,          // 无隔离
    WorkspaceOnly, // 仅允许访问工作区
    AllowList,    // 白名单模式（仅允许挂载指定目录）
}
```

### 容器环境检测

沙箱模块自动检测是否运行在容器中（[`sandbox.rs#L109-L153`](/rust/crates/runtime/src/sandbox.rs#L109-L153)），检测手段包括：

- `/.dockerenv` 文件存在性
- `/run/.containerenv` 文件存在性
- `/proc/1/cgroup` 中是否包含 docker/containerd/kubepods 关键字
- 环境变量 `CONTAINER`、`DOCKER`、`PODMAN`、`KUBERNETES_SERVICE_HOST`

---

## 第 3 层：命令验证（Bash Validation Pipeline）

[`runtime/src/bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs) 实现了完整的命令验证流水线，这是安全设计的核心。

### 3.1 只读验证（ReadOnly Validation）

[`validate_read_only`](/rust/crates/runtime/src/bash_validation.rs#L103-L160) 函数在 `ReadOnly` 模式下拦截写操作：

**黑名单命令：**
- 文件系统写操作：`cp`、`mv`、`rm`、`mkdir`、`rmdir`、`touch`、`chmod`、`chown`、`ln`、`tee`、`truncate`、`shred`、`dd`
- 系统状态修改：`apt`、`yum`、`pacman`、`brew`、`pip`、`npm`、`cargo`、`docker`、`systemctl`、`kill`、`reboot`

**Git 子命令分类：**
```rust
const GIT_READ_ONLY_SUBCOMMANDS: &[&str] = &[
    "status", "log", "diff", "show", "branch", "tag", "stash", "remote",
    "fetch", "ls-files", "cat-file", "rev-parse", "describe", "blame", "bisect",
];
```

参见 [`bash_validation.rs#L163-L183`](/rust/crates/runtime/src/bash_validation.rs#L163-L183)。

### 3.2 破坏性命令警告（Destructive Command Warning）

[`check_destructive`](/rust/crates/runtime/src/bash_validation.rs#L206-L248) 检测危险模式并返回警告：

```rust
const DESTRUCTIVE_PATTERNS: &[(&str, &str)] = &[
    (
        "rm -rf /",
        "Recursive forced deletion at root — this will destroy the system",
    ),
    ("rm -rf ~", "Recursive forced deletion of home directory"),
    (
        "rm -rf *",
        "Recursive forced deletion of all files in current directory",
    ),
    ("rm -rf .", "Recursive forced deletion of current directory"),
    (
        "mkfs",
        "Filesystem creation will destroy existing data on the device",
    ),
    (
        "dd if=",
        "Direct disk write — can overwrite partitions or devices",
    ),
    ("> /dev/sd", "Writing to raw disk device"),
    (
        "chmod -R 777",
        "Recursively setting world-writable permissions",
    ),
    ("chmod -R 000", "Recursively removing all permissions"),
    (":(){ :|:& };:", "Fork bomb — will crash the system"),
];

/// Commands that are always destructive regardless of arguments.
const ALWAYS_DESTRUCTIVE_COMMANDS: &[&str] = &["shred", "wipefs"];
```

参见 [`bash_validation.rs#L206-L248`](/rust/crates/runtime/src/bash_validation.rs#L206-L248)。

### 3.3 路径验证（Path Validation）

[`validate_paths`](/rust/crates/runtime/src/bash_validation.rs#L360-L384) 检测目录遍历攻击：

```rust
// 检测 "../" 目录遍历模式
if command.contains("../") {
    return ValidationResult::Warn {
        message: "Command contains directory traversal pattern '../' — verify the target path resolves within the workspace".to_string(),
    };
}

// 检测家目录引用
if command.contains("~/") || command.contains("$HOME") {
    return ValidationResult::Warn {
        message: "Command references home directory — verify it stays within the workspace scope".to_string(),
    };
}
```

### 3.4 命令语义分类（Command Semantics）

[`classify_command`](/rust/crates/runtime/src/bash_validation.rs#L533-L580) 将命令分类为不同意图：

```rust
pub enum CommandIntent {
    ReadOnly,        // ls, cat, grep, find
    Write,           // cp, mv, mkdir, touch
    Destructive,     // rm, shred, truncate
    Network,         // curl, wget, ssh
    ProcessManagement, // kill, pkill, ps
    PackageManagement, // apt, npm, cargo
    SystemAdmin,     // sudo, mount, systemctl
    Unknown,
}
```

### 3.5 完整验证流水线

[`validate_command`](/rust/crates/runtime/src/bash_validation.rs#L594-L619) 按顺序执行所有检查：

```rust
pub fn validate_command(command: &str, mode: PermissionMode, workspace: &Path) -> ValidationResult {
    // 1. 模式级验证（包含只读检查）
    let result = validate_mode(command, mode);
    if result != ValidationResult::Allow { return result; }

    // 2. sed 特定验证（阻止 -i 原地编辑）
    let result = validate_sed(command, mode);
    if result != ValidationResult::Allow { return result; }

    // 3. 破坏性命令警告
    let result = check_destructive(command);
    if result != ValidationResult::Allow { return result; }

    // 4. 路径验证
    validate_paths(command, workspace)
}
```

---

## 第 4 层：Hook 拦截（Pre/Post ToolUse Hooks）

### PreToolUse Hook

在工具执行前，`ConversationRuntime` 会调用 `run_pre_tool_use_hook`（[`conversation.rs#L378-L415`](/rust/crates/runtime/src/conversation.rs#L378-L415)），允许外部策略检查：

```rust
// Hook 可以返回三种决策
if pre_hook_result.is_denied() {
    return PermissionOutcome::Deny {
        reason: format_hook_message(&pre_hook_result, &format!("PreToolUse hook denied tool `{tool_name}`")),
    };
} else if pre_hook_result.is_ask() {
    // 强制进入用户确认流程
}
```

### PostToolUse Hook

工具执行后，`run_post_tool_use_hook` 或 `run_post_tool_use_failure_hook` 可以修改输出或标记错误（[`conversation.rs#L427-L450`](/rust/crates/runtime/src/conversation.rs#L427-L450)）：

```rust
let post_hook_result = if is_error {
    self.run_post_tool_use_failure_hook(&tool_name, &effective_input, &output)
} else {
    self.run_post_tool_use_hook(&tool_name, &effective_input, &output, false)
};

// Hook 可以覆盖执行结果
if post_hook_result.is_denied() || post_hook_result.is_failed() {
    is_error = true;
}
output = merge_hook_feedback(post_hook_result.messages(), output, ...);
```

---

## 第 5 层：审计追踪（Audit Trail）

### Session 持久化

所有工具调用和结果都记录在 `Session` 中，形成完整的审计日志（[`conversation.rs#L455-L467`](/rust/crates/runtime/src/conversation.rs#L455-L467)）：

```rust
let result_message = ConversationMessage::tool_result(
    tool_use_id, tool_name, output, is_error
);
self.session.push_message(result_message.clone())?;
```

### ToolUse/ToolResult 配对

`ContentBlock` 枚举中的 `ToolUse` 和 `ToolResult` 通过 `tool_use_id` 严格配对（参见 `session.rs`），确保每个工具调用都有对应的结果记录，便于事后追溯调用链。

### Turn 摘要

每个 `run_turn` 结束后生成 `TurnSummary`（[`conversation.rs#L474-L484`](/rust/crates/runtime/src/conversation.rs#L474-L484)），包含：

- `assistant_messages` — AI 返回的文本消息
- `tool_results` — 所有工具执行结果
- `prompt_cache_events` — Prompt 缓存事件
- `iterations` — 工具调用迭代次数
- `usage` — Token 用量统计
- `auto_compaction` — 自动压缩事件

---

## 权限执行层（Permission Enforcer）

[`runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) 提供了更细粒度的执行检查：

### 文件写边界检查

[`check_file_write`](/rust/crates/runtime/src/permission_enforcer.rs#L74-L108) 验证路径是否在工作区内：

```rust
pub fn check_file_write(&self, path: &str, workspace_root: &str) -> EnforcementResult {
    let mode = self.policy.active_mode();

    match mode {
        PermissionMode::ReadOnly => EnforcementResult::Denied {
            tool: "write_file".to_owned(),
            active_mode: mode.as_str().to_owned(),
            required_mode: PermissionMode::WorkspaceWrite.as_str().to_owned(),
            reason: format!("file writes are not allowed in '{}' mode", mode.as_str()),
        },
        PermissionMode::WorkspaceWrite => {
            if is_within_workspace(path, workspace_root) {
                EnforcementResult::Allowed
            } else {
                EnforcementResult::Denied {
                    tool: "write_file".to_owned(),
                    active_mode: mode.as_str().to_owned(),
                    required_mode: PermissionMode::DangerFullAccess.as_str().to_owned(),
                    reason: format!(
                        "path '{}' is outside workspace root '{}'",
                        path, workspace_root
                    ),
                }
            }
        }
        PermissionMode::Allow | PermissionMode::DangerFullAccess => EnforcementResult::Allowed,
        PermissionMode::Prompt => EnforcementResult::Denied {
            tool: "write_file".to_owned(),
            active_mode: mode.as_str().to_owned(),
            required_mode: PermissionMode::WorkspaceWrite.as_str().to_owned(),
            reason: "file write requires confirmation in prompt mode".to_owned(),
        },
    }
}
```

### Bash 命令只读启发式

[`is_read_only_command`](/rust/crates/runtime/src/permission_enforcer.rs#L160-L238) 通过白名单判断：

```rust
fn is_read_only_command(command: &str) -> bool {
    let first_token = command
        .split_whitespace()
        .next()
        .unwrap_or("")
        .rsplit('/')
        .next()
        .unwrap_or("");

    matches!(
        first_token,
        "cat"
            | "head"
            | "tail"
            | "less"
            | "more"
            | "wc"
            | "ls"
            | "find"
            | "grep"
            | "rg"
            | "awk"
            | "sed"
            | "echo"
            | "printf"
            | "which"
            | "where"
            | "whoami"
            | "pwd"
            | "env"
            | "printenv"
            | "date"
            | "cal"
            | "df"
            | "du"
            | "free"
            | "uptime"
            | "uname"
            | "file"
            | "stat"
            | "diff"
            | "sort"
            | "uniq"
            | "tr"
            | "cut"
            | "paste"
            | "tee"
            | "xargs"
            | "test"
            | "true"
            | "false"
            | "type"
            | "readlink"
            | "realpath"
            | "basename"
            | "dirname"
            | "sha256sum"
            | "md5sum"
            | "b3sum"
            | "xxd"
            | "hexdump"
            | "od"
            | "strings"
            | "tree"
            | "jq"
            | "yq"
            | "python3"
            | "python"
            | "node"
            | "ruby"
            | "cargo"
            | "rustc"
            | "git"
            | "gh"
    ) && !command.contains("-i ")
        && !command.contains("--in-place")
        && !command.contains(" > ")
        && !command.contains(" >>")
}
```

---

## Bash 工具的输出截断

[`bash.rs#L292-L304`](/rust/crates/runtime/src/bash.rs#L292-L304) 定义了输出截断机制，防止大输出淹没终端：

```rust
const MAX_OUTPUT_BYTES: usize = 16_384; // 16 KiB

fn truncate_output(s: &str) -> String {
    if s.len() <= MAX_OUTPUT_BYTES {
        return s.to_string();
    }
    // 找到最后一个有效的 UTF-8 边界
    let mut end = MAX_OUTPUT_BYTES;
    while end > 0 && !s.is_char_boundary(end) {
        end -= 1;
    }
    let mut truncated = s[..end].to_string();
    truncated.push_str("\n\n[output truncated — exceeded 16384 bytes]");
    truncated
}
```

测试用例验证了边界条件（[`bash.rs#L307-L336`](/rust/crates/runtime/src/bash.rs#L307-L336)）。

---

## 错误处理与超时

### 命令超时

[`bash.rs#L112-L133`](/rust/crates/runtime/src/bash.rs#L112-L133) 实现了命令超时控制：

```rust
let output_result = if let Some(timeout_ms) = input.timeout {
    match timeout(Duration::from_millis(timeout_ms), command.output()).await {
        Ok(result) => (result?, false),
        Err(_) => {
            return Ok(BashCommandOutput {
                stderr: format!("Command exceeded timeout of {timeout_ms} ms"),
                interrupted: true,
                return_code_interpretation: Some(String::from("timeout")),
                ...
            });
        }
    }
}
```

### 返回码解释

[`bash.rs#L143-L149`](/rust/crates/runtime/src/bash.rs#L143-L149) 将非零返回码转换为解释性字符串：

```rust
let return_code_interpretation = output.status.code().and_then(|code| {
    if code == 0 {
        None  // 成功无需解释
    } else {
        Some(format!("exit_code:{code}"))
    }
});
```

---

## 安全设计哲学总结

1. **默认拒绝（Deny by Default）** — 权限模型从最严格的 `ReadOnly` 开始，需要时才提升
2. **多层验证（Multi-layer Validation）** — 不依赖单一检查，而是通过多层独立验证
3. **最小权限（Least Privilege）** — 沙箱默认隔离网络和文件系统，只开放必要权限
4. **可追溯（Auditability）** — 所有工具调用和结果都记录在 Session 中
5. **用户确认（Human-in-the-loop）** — 危险操作强制弹窗确认

---

## 源码索引

| 文件 | 核心内容 | 行号参考 |
|------|----------|----------|
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | 五级权限模型、规则引擎、Hook 上下文 | L9-L292 |
| [`/rust/crates/runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) | 权限执行检查、文件边界、只读启发式 | L13-L416 |
| [`/rust/crates/runtime/src/sandbox.rs`](/rust/crates/runtime/src/sandbox.rs) | Linux unshare 沙箱、容器检测、文件系统隔离 | L7-L304 |
| [`/rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs) | Bash 工具执行、超时控制、输出截断 | L19-L336 |
| [`/rust/crates/runtime/src/bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs) | 命令验证流水线、破坏性检测、路径检查、语义分类 | L16-L1004 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | 工具执行循环、Hook 调用、Session 记录 | L378-L484 |
| [`/rust/crates/runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) | ContentBlock、ToolUse/ToolResult 配对 | L28-L43 |

---

## 与原文档的映射关系

| 原文档主题 | 本文源码映射 |
|------------|--------------|
| 威胁模型 | 见"为什么安全至关重要"章节 |
| 权限系统 | 见"第 1 层：权限模型" + `permissions.rs` |
| 沙箱机制 | 见"第 2 层：沙箱隔离" + `sandbox.rs` |
| 危险命令检测 | 见"第 3 层：命令验证" + `bash_validation.rs` |
| 审计日志 | 见"第 5 层：审计追踪" + `conversation.rs` |
