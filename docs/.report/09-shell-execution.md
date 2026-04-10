# 命令执行工具 - BashTool 安全设计与实现

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/tools/shell-execution) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 执行链路总览

一条 Bash 命令从 AI 决策到实际执行的完整路径：

```
AI 生成 tool_use: { command: "npm test" }
  ↓
Bash 工具 schema 校验（JSON → BashCommandInput）
  ↓
权限检查（PermissionEnforcer / PermissionPolicy）
  ├── ReadOnly 模式 → is_read_only_command() 判定
  ├── Prompt 模式 → 必须弹窗确认
  └── DangerFullAccess / Allow → 直接放行
  ↓
tools::execute_tool("bash", input)
  ↓
run_bash() → runtime::execute_bash()
  ↓
sandbox_status_for_input() 决定沙箱策略
  ↓
tokio::process::Command 子进程执行
  ↓
输出截断 → 返回 BashCommandOutput
```

### 源码映射：工具注册与入口分发

在 `claw-code` 中，Bash 不是特权通道，而是与普通工具一样注册在 `tools` crate 中。

工具规格定义于 [`tools/src/lib.rs#L385-L404`](/rust/crates/tools/src/lib.rs#L385-L404)：

```rust
ToolSpec {
    name: "bash",
    description: "Execute a shell command in the current workspace.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "command": { "type": "string" },
            "timeout": { "type": "integer", "minimum": 1 },
            "description": { "type": "string" },
            "run_in_background": { "type": "boolean" },
            "dangerouslyDisableSandbox": { "type": "boolean" },
            "namespaceRestrictions": { "type": "boolean" },
            "isolateNetwork": { "type": "boolean" },
            "filesystemMode": { "type": "string", "enum": ["off", "workspace-only", "allow-list"] },
            "allowedMounts": { "type": "array", "items": { "type": "string" } }
        },
        "required": ["command"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::DangerFullAccess,
}
```

执行分发在 [`tools/src/lib.rs#L1178-L1185`](/rust/crates/tools/src/lib.rs#L1178-L1185)：

```rust
"bash" => {
    maybe_enforce_permission_check(enforcer, name, input)?;
    from_value::<BashCommandInput>(input).and_then(run_bash)
}
```

注意 `maybe_enforce_permission_check` 的存在：即使在模型层“已决定调用 Bash”，运行时仍需通过 `PermissionEnforcer` 的二次闸门。这是权限系统与工具系统解耦的关键设计。

---

## 只读命令的判定：为什么 Read 免审批而 Bash 不一定

文档指出 BashTool 的 `isReadOnly()` 决定命令是否免审批。在 `claw-code` 中，存在**两层**只读判定：

1. **快速启发式**：`permission_enforcer.rs` 中的 `is_read_only_command()`，用于 `PermissionEnforcer::check_bash()` 的第一道防线；
2. **完整验证管线**：`bash_validation.rs` 中的 `validate_read_only()`，提供更细粒度的命令分类与拦截。

### 快速启发式：PermissionEnforcer

[`runtime/src/permission_enforcer.rs#L159-L222`](/rust/crates/runtime/src/permission_enforcer.rs#L159-L222)：

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
        "cat" | "head" | "tail" | "less" | "more" | "wc"
            | "ls" | "find" | "grep" | "rg" | "awk" | "sed"
            | "echo" | "printf" | "which" | "pwd" | "env"
            | "git" | "cargo" | "jq" | "node" | "python3"
            | ...
    ) && !command.contains("-i ")
        && !command.contains("--in-place")
        && !command.contains(" > ")
        && !command.contains(" >> ")
}
```

该函数不仅检查首命令 token，还会主动拦截含有 `-i`（交互式 Python）、`--in-place`（sed 就地编辑）以及重定向符 `>` / `>>` 的命令。这对应了原文中“重定向即写操作”的安全直觉。

测试用例在 [`permission_enforcer.rs#L339-L341`](/rust/crates/runtime/src/permission_enforcer.rs#L339-L341) 明确验证：

```rust
assert!(!is_read_only_command("rm file.txt"));
assert!(!is_read_only_command("echo test > file.txt"));
assert!(!is_read_only_command("sed -i 's/a/b/' file"));
```

### 完整验证管线：bash_validation

[`runtime/src/bash_validation.rs#L99-L131`](/rust/crates/runtime/src/bash_validation.rs#L99-L130) 的 `validate_read_only()` 实现了更严格的策略：

- 显式列出 `WRITE_COMMANDS`：`cp`, `mv`, `rm`, `mkdir`, `touch`, `tee`, `dd` ...
- 显式列出 `STATE_MODIFYING_COMMANDS`：`apt`, `brew`, `npm`, `cargo`, `docker`, `systemctl`, `kill`, `reboot` ...
- 对 `sudo` 包装递归检查内部命令
- 对 `git` 子命令白名单校验：`status`, `log`, `diff`, `fetch` 等只读子命令放行；`push`, `commit`, `reset` 拦截

例如，[`bash_validation.rs#L185-L199`](/rust/crates/runtime/src/bash_validation.rs#L185-L199)：

```rust
fn validate_git_read_only(command: &str) -> ValidationResult {
    let parts: Vec<&str> = command.split_whitespace().collect();
    let subcommand = parts.iter().skip(1).find(|p| !p.starts_with('-'));
    match subcommand {
        Some(&sub) if GIT_READ_ONLY_SUBCOMMANDS.contains(&sub) => ValidationResult::Allow,
        Some(&sub) => ValidationResult::Block {
            reason: format!(
                "Git subcommand '{sub}' modifies repository state and is not allowed in read-only mode"
            ),
        },
        None => ValidationResult::Allow,
    }
}
```

这意味着 `git status` 在 ReadOnly 模式下可直接通过，而 `git push` 会被明确拦截并返回 `ValidationResult::Block`。

---

## AST 安全解析：命令语义分类

原文提到的 `preparePermissionMatcher()` 使用 `parseForSecurity()` 对命令做 AST 级别解析。由于 `claw-code` 的 Rust 实现目前尚未引入 tree-sitter bash 绑定，它采用了一套**纯手写的语义分类器**作为等效替代。

### CommandIntent 语义分类

[`runtime/src/bash_validation.rs#L529-L570`](/rust/crates/runtime/src/bash_validation.rs#L529-L570) 定义了 `classify_command()`：

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

分类逻辑：

```rust
fn classify_by_first_command(first: &str, command: &str) -> CommandIntent {
    if SEMANTIC_READ_ONLY_COMMANDS.contains(&first) {
        if first == "sed" && command.contains(" -i") {
            return CommandIntent::Write;
        }
        return CommandIntent::ReadOnly;
    }
    if ALWAYS_DESTRUCTIVE_COMMANDS.contains(&first) || first == "rm" {
        return CommandIntent::Destructive;
    }
    if WRITE_COMMANDS.contains(&first) { return CommandIntent::Write; }
    if NETWORK_COMMANDS.contains(&first) { return CommandIntent::Network; }
    if PROCESS_COMMANDS.contains(&first) { return CommandIntent::ProcessManagement; }
    if PACKAGE_COMMANDS.contains(&first) { return CommandIntent::PackageManagement; }
    if SYSTEM_ADMIN_COMMANDS.contains(&first) { return CommandIntent::SystemAdmin; }
    if first == "git" { return classify_git_command(command); }
    CommandIntent::Unknown
}
```

这虽然是基于字符串匹配的简化实现，但测试覆盖极为完整（近 100 个测试用例），并且与权限模式检查结合使用，构成了 fail-safe 的防御体系。

### 危险命令警告

[`bash_validation.rs#L240-L274`](/rust/crates/runtime/src/bash_validation.rs#L240-L274) 的 `check_destructive()` 会发出预警：

```rust
const DESTRUCTIVE_PATTERNS: &[(&str, &str)] = &[
    ("rm -rf /", "Recursive forced deletion at root — this will destroy the system"),
    ("rm -rf ~", "Recursive forced deletion of home directory"),
    ("rm -rf *", "Recursive forced deletion of all files in current directory"),
    ("> /dev/sd", "Writing to raw disk device"),
    ("chmod -R 777", "Recursively setting world-writable permissions"),
    (":(){ :|:& };:", "Fork bomb — will crash the system"),
];
```

匹配到这些模式时，不会直接拒绝，而是返回 `ValidationResult::Warn`，由更高层的权限策略决定是否继续执行。这与原文中“弹窗确认”的行为一致。

---

## 超时控制：分级策略

Bash 命令的超时由 `BashCommandInput.timeout` 显式指定， otherwise 走默认逻辑。

### 源码映射：tokio::time::timeout

[`runtime/src/bash.rs#L105-L127`](/rust/crates/runtime/src/bash.rs#L105-L127)：

```rust
async fn execute_bash_async(
    input: BashCommandInput,
    sandbox_status: SandboxStatus,
    cwd: std::path::PathBuf,
) -> io::Result<BashCommandOutput> {
    let mut command = prepare_tokio_command(&input.command, &cwd, &sandbox_status, true);

    let output_result = if let Some(timeout_ms) = input.timeout {
        match timeout(Duration::from_millis(timeout_ms), command.output()).await {
            Ok(result) => (result?, false),
            Err(_) => {
                return Ok(BashCommandOutput {
                    stdout: String::new(),
                    stderr: format!("Command exceeded timeout of {timeout_ms} ms"),
                    interrupted: true,
                    return_code_interpretation: Some(String::from("timeout")),
                    ...
                });
            }
        }
    } else {
        (command.output().await?, false)
    };
    ...
}
```

超时后不会直接杀进程：`timeout()` 超时会导致 `command.output()` 的 Future 被取消，但底层子进程在 tokio 的实现中可能仍然存活。不过 `claw-code` 的当前实现标记 `interrupted: true` 并将 `return_code_interpretation` 设为 `"timeout"`，告知调用方该命令未正常完成。

### 默认值与上限

`claw-code` 的 Bash timeout 没有内置的硬编码默认值（与上游 TypeScript 的 120,000ms 不同），完全依赖调用方传入。这意味着如果模型或用户未显式指定 timeout，Bash 命令将无限期阻塞，直到子进程自行结束。这意味着如果模型或用户未显式指定 timeout，Bash 命令将无限期阻塞，直到子进程自行结束。但在 MCP 工具层面有默认 `tool_call_timeout_ms`（15,000ms）用于 MCP 服务器调用，对原生 Bash 工具不生效。CLI 侧尚未暴露 `--timeout` 参数。

---

## 自动后台化

文档描述了自动后台化机制：当命令超过 15 秒阻塞预算时自动转为后台任务。`claw-code` 的 Rust 实现**目前尚未实现这一自动后台化逻辑**，但已经保留了相关数据结构字段。

### 已预留的字段

[`runtime/src/bash.rs#L19-L63`](/rust/crates/runtime/src/bash.rs#L19-L63)：

```rust
pub struct BashCommandInput {
    pub command: String,
    pub timeout: Option<u64>,
    pub description: Option<String>,
    #[serde(rename = "run_in_background")]
    pub run_in_background: Option<bool>,
    ...
}

pub struct BashCommandOutput {
    ...
    #[serde(rename = "backgroundTaskId")]
    pub background_task_id: Option<String>,
    #[serde(rename = "backgroundedByUser")]
    pub backgrounded_by_user: Option<bool>,
    #[serde(rename = "assistantAutoBackgrounded")]
    pub assistant_auto_backgrounded: Option<bool>,
    ...
}
```

### 手动后台化实现

[`runtime/src/bash.rs#L74-L93`](/rust/crates/runtime/src/bash.rs#L74-L93)：

```rust
if input.run_in_background.unwrap_or(false) {
    let mut child = prepare_command(&input.command, &cwd, &sandbox_status, false);
    let child = child
        .stdin(Stdio::null())
        .stdout(Stdio::null())
        .stderr(Stdio::null())
        .spawn()?;

    return Ok(BashCommandOutput {
        stdout: String::new(),
        stderr: String::new(),
        background_task_id: Some(child.id().to_string()),
        backgrounded_by_user: Some(false),
        assistant_auto_backgrounded: Some(false),
        ...
    });
}
```

当 AI 显式传入 `run_in_background: true` 时，子进程会脱离标准输出/错误，立即返回一个 `background_task_id`（即 OS PID）。但正如代码所显式，`.stdout(Stdio::null())` 丢弃了输出——这意味着手动后台化目前无法事后获取输出，与上游的 Task 后台化系统有功能差距。这是需要开发者注意的一个实现缺口。

---

## 输出截断策略

Bash 命令的输出可能非常巨大（如 `npm test` 的完整日志）。直接塞入上下文窗口会导致 token 爆炸。`claw-code` 在 `bash.rs` 中实现了硬截断。

### 源码映射：16 KiB 截断线

[`runtime/src/bash.rs#L288-L304`](/rust/crates/runtime/src/bash.rs#L288-L304)：

```rust
const MAX_OUTPUT_BYTES: usize = 16_384;

fn truncate_output(s: &str) -> String {
    if s.len() <= MAX_OUTPUT_BYTES {
        return s.to_string();
    }
    let mut end = MAX_OUTPUT_BYTES;
    while end > 0 && !s.is_char_boundary(end) {
        end -= 1;
    }
    let mut truncated = s[..end].to_string();
    truncated.push_str("\n\n[output truncated — exceeded 16384 bytes]");
    truncated
}
```

截断发生在字节层面（而非字符层面），但会正确回退到上一个 UTF-8 字符边界，避免产生非法 UTF-8。截断标记`[output truncated — exceeded 16384 bytes]` 与上游文案完全一致，便于模型识别并决定下一步行动。

CLI 渲染层在展示时还会做第二次截断（按显示行数），但不会影响实际返回给模型的内容。参见 [`rusty-claude-cli/src/main.rs#L7142-L7157`](/rust/crates/rusty-claude-cli/src/main.rs#L7142-L7157)。

---

## 沙箱机制：权限系统之外的第二道防线

即使权限系统放行，Bash 命令仍可能触及系统边界。`claw-code` 在 Linux 上通过 `unshare` 实现了 Namespace 沙箱。

### 沙箱配置层级

[`runtime/src/sandbox.rs#L27-L68`](/rust/crates/runtime/src/sandbox.rs#L27-L68)：

```rust
pub struct SandboxConfig {
    pub enabled: Option<bool>,
    pub namespace_restrictions: Option<bool>,
    pub network_isolation: Option<bool>,
    pub filesystem_mode: Option<FilesystemIsolationMode>,
    pub allowed_mounts: Vec<String>,
}

pub enum FilesystemIsolationMode {
    Off,
    WorkspaceOnly,
    AllowList,
}
```

模型可以显式覆盖沙箱行为：`dangerously_disable_sandbox` 关闭沙箱，`isolate_network` 隔离网络，`filesystem_mode` 控制文件系统可见范围。

### 运行时沙箱状态解析

[`runtime/src/sandbox.rs#L162-L203`](/rust/crates/runtime/src/sandbox.rs#L162-L203) 的 `resolve_sandbox_status_for_request()`：

```rust
pub fn resolve_sandbox_status_for_request(request: &SandboxRequest, cwd: &Path) -> SandboxStatus {
    let container = detect_container_environment();
    let namespace_supported = cfg!(target_os = "linux") && unshare_user_namespace_works();
    let network_supported = namespace_supported;
    ...
    let active = request.enabled
        && (!request.namespace_restrictions || namespace_supported)
        && (!request.network_isolation || network_supported);
    ...
}
```

注意 `unshare_user_namespace_works()` 会实际执行 `unshare --user --map-root-user true` 来探测当前环境是否支持用户命名空间（GitHub Actions 等 CI 环境中可能存在限制）。若不支持，则 `active = false`，沙箱回退为普通进程——这是 fail-safe 的降级策略。

### Linux 沙箱命令构建

[`runtime/src/sandbox.rs#L211-L255`](/rust/crates/runtime/src/sandbox.rs#L211-L255)：

```rust
pub fn build_linux_sandbox_command(command: &str, cwd: &Path, status: &SandboxStatus) -> Option<LinuxSandboxCommand> {
    if !cfg!(target_os = "linux") || !status.enabled || (!status.namespace_active && !status.network_active) {
        return None;
    }

    let mut args = vec![
        "--user".to_string(),
        "--map-root-user".to_string(),
        "--mount".to_string(),
        "--ipc".to_string(),
        "--pid".to_string(),
        "--uts".to_string(),
        "--fork".to_string(),
    ];
    if status.network_active {
        args.push("--net".to_string());
    }
    args.push("sh".to_string());
    args.push("-lc".to_string());
    args.push(command.to_string());

    Some(LinuxSandboxCommand {
        program: "unshare".to_string(),
        args,
        env: vec![
            ("HOME".to_string(), sandbox_home.display().to_string()),
            ("TMPDIR".to_string(), sandbox_tmp.display().to_string()),
            ("CLAWD_SANDBOX_FILESYSTEM_MODE".to_string(), status.filesystem_mode.as_str().to_string()),
            ("CLAWD_SANDBOX_ALLOWED_MOUNTS".to_string(), status.allowed_mounts.join(":")),
        ],
    })
}
```

在非 Linux 平台（如 macOS）上，`build_linux_sandbox_command` 返回 `None`，Bash 命令回退到直接 `sh -lc` 执行，但通过重写 `HOME` 和 `TMPDIR` 到工作区子目录（`.sandbox-home` / `.sandbox-tmp`）提供最低限度的隔离。参见 [`bash.rs#L203-L209`](/rust/crates/runtime/src/bash.rs#L203-L209)。

---

## 为什么用专用工具而不是直接调 shell

文档对比了专用工具（Read / Grep / Glob）与 Bash 命令的架构差异。在 `claw-code` 中，这一设计差异体现得非常直接：工具通过 `PermissionEnforcer` 的 `check()` 路径，Bash 通过 `check_bash()` 的独立路径。

### 对比表

| 维度 | 专用工具（Read / Grep / Edit） | Bash 命令 |
|------|------------------------------|-----------|
| 权限粒度 | `read_file` = `ReadOnly` 直接放行 | Bash 需 `DangerFullAccess` 或弹窗确认 |
| 输出结构化 | `ReadFileOutput` 精确到行号范围 | `BashCommandOutput` 只有 stdout/stderr 字符串 |
| 并发安全 | 工具自身无状态，可安全并发 | Bash 可能修改文件系统，串行执行 |
| 安全审计 | 规则匹配 `read_file(/path)` 精确 | 需提取 `command` 字段做字符串匹配 |
| 边界检查 | `file_ops.rs` 显式校验工作区逃逸 | 依赖沙箱和权限两道防线 |

在 [`permission_enforcer.rs#L38-L56`](/rust/crates/runtime/src/permission_enforcer.rs#L38-L56) 中，`check()` 的核心逻辑：

```rust
pub fn check(&self, tool_name: &str, input: &str) -> EnforcementResult {
    if self.policy.active_mode() == PermissionMode::Prompt {
        return EnforcementResult::Allowed; // Prompt 模式交给外部交互层处理
    }
    let outcome = self.policy.authorize(tool_name, input, None);
    ...
}
```

而 `check_bash()` 在 [`permission_enforcer.rs#L111-L139`](/rust/crates/runtime/src/permission_enforcer.rs#L111-L139) 中显式拒绝 Prompt 模式：

```rust
pub fn check_bash(&self, command: &str) -> EnforcementResult {
    let mode = self.policy.active_mode();

    match mode {
        PermissionMode::ReadOnly => {
            if is_read_only_command(command) {
                EnforcementResult::Allowed
            } else {
                EnforcementResult::Denied {
                    tool: "bash".to_owned(),
                    active_mode: mode.as_str().to_owned(),
                    required_mode: PermissionMode::WorkspaceWrite.as_str().to_owned(),
                    reason: format!(
                        "command may modify state; not allowed in '{}' mode",
                        mode.as_str()
                    ),
                }
            }
        }
        PermissionMode::Prompt => EnforcementResult::Denied {
            tool: "bash".to_owned(),
            active_mode: mode.as_str().to_owned(),
            required_mode: PermissionMode::DangerFullAccess.as_str().to_owned(),
            reason: "bash requires confirmation in prompt mode".to_owned(),
        },
        // WorkspaceWrite, Allow, DangerFullAccess: permit bash
        _ => EnforcementResult::Allowed,
    }
}
```

这解释了为什么 Read 工具在 ReadOnly 模式下畅通无阻，而 Bash 即使在“看起来只读”时也需要额外校验：Bash 的表达能力太强，字符串级别的 `is_read_only_command` 永远存在绕过空间。

---

## 进度反馈的流式设计

原文提到 `onProgress` 回调逐行推送输出。`claw-code` 目前的 `execute_bash()` 实现是**同步阻塞返回完整输出**，尚未实现 generator yield 的流式进度。CLI 渲染层虽然为上层的流式调用预留了接口，但 Bash 工具本身的事件粒度仍然是以命令退出为边界。这意味着长时间运行的构建任务（如 `cargo build`）在执行期间，终端只能看到一个静态的 spinner，用户无法通过流式输出来判断编译进展到了哪个 crate。

### CLI 渲染层的进度适配

[`rusty-claude-cli/src/main.rs#L6169-L6184`](/rust/crates/rusty-claude-cli/src/main.rs#L6169-L6184) 的 `describe_tool_progress`：

```rust
fn describe_tool_progress(name: &str, input: &str) -> String {
    match name {
        "bash" | "Bash" => {
            let command = parsed.get("command").and_then(|v| v.as_str()).unwrap_or_default();
            if command.is_empty() {
                "running shell command".to_string()
            } else {
                format!("command {}", truncate_for_summary(command.trim(), 100))
            }
        }
        ...
    }
}
```

TUI 中 Bash 工具调用的摘要会显示正在执行的命令文本，让用户感知到 shell 正在运行。未来的流式执行可以复用该摘要机制。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | 工具注册表 `GlobalToolRegistry`、`mvp_tool_specs()`、`execute_tool()` 分发 |
| [`/rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs) | `BashCommandInput` / `BashCommandOutput` schema、同步执行、超时、输出截断、后台化 |
| [`/rust/crates/runtime/src/bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs) | 只读验证、危险命令警告、sed / path / git 校验、语义分类 `CommandIntent` |
| [`/rust/crates/runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) | `PermissionEnforcer`、`check_bash()`、快速启发式 `is_read_only_command()` |
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode`、`PermissionPolicy`、`authorize_with_context()`、allow/ask/deny 规则解析 |
| [`/rust/crates/runtime/src/sandbox.rs`](/rust/crates/runtime/src/sandbox.rs) | `SandboxConfig`、`SandboxStatus`、`build_linux_sandbox_command()`、`unshare` 命名空间隔离 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime`、工具执行循环、`PermissionOutcome::Allow` 后的 `tool_executor.execute()` |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、`describe_tool_progress`、Bash 结果格式化、权限模式命令行参数 |
