# 沙箱机制 - 权限系统之外的第二道防线

> 本报告基于 [Claude Code 中文文档 - 沙箱机制](https://ccb.agent-aura.top/docs/safety/sandbox) 的架构描述，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。文末附 [源码索引](#源码索引)。

---

## 一句话定义

沙箱不是用来替代权限系统，而是给 **shell 命令** 再套一层 **OS 级能力边界**：

- **权限系统**决定：这次工具调用要不要执行
- **沙箱**决定：就算执行了，这个子进程最多能碰到哪些文件、哪些网络目标

两者组合起来，才构成真正的 Defense-in-Depth。

---

## 实现分层：配置与适配在上层，OS 级隔离在下层

`claw-code` 的沙箱分成两层：

- **本仓库负责**：策略、配置转换、启停判断、命令包裹、权限联动
- **OS 级隔离**：在 Linux 上通过 `unshare` 实现用户/网络/挂载命名空间隔离

这意味着仓库本身不重新发明沙箱 runtime，而是决定“该不该启、该怎么配、该怎么接进工具链”，再由底层 OS 原语落地为真正的进程级约束。

---

## 它到底解决什么问题

如果只有应用层权限系统，所有判断都是“执行前推断”。但 shell 命令的真实副作用经常取决于运行时行为：

```bash
bash script.sh
python -c "..."
make
npm install
```

沙箱的作用，就是把运行时行为的能力范围压缩到一个明确边界内。即使应用层检查漏了，命令也不能随意写系统目录或访问不允许的网络目标。

### 为什么“拦住它”本身就是价值

`/etc/hosts` 被默认拦住，说明边界在生效。沙箱至少补上了 4 件权限系统单独做不好的事：

1. **给 shell 一个 OS 级兜底**：运行时分析再聪明，也无法覆盖所有动态展开的风险面
2. **让安全边界内的命令少弹窗**：工作区内的开发命令已被 OS 级边界收紧，因此许多低风险 Bash 命令可以自动放行
3. **把“出错”的后果从系统级破坏降成受限失败**：模型偶尔出错时，命令因权限不足而失败，而非真正修改系统
4. **拦截运行时绕过和逃逸路径**：限制对配置文件、技能目录等高价值目标的写入

---

## 设计边界：保护什么，不保护什么

### 保护对象

- Bash / shell 命令执行
- shell 子进程的文件系统写入范围
- shell 子进程的网络访问范围
- 已配置允许挂载路径的边界

### 不直接保护的对象

- `FileEditTool` / `FileWriteTool` 这类直接文件工具（它们走应用层权限系统）
- 纯应用层的权限弹窗和规则匹配
- Bash AST 解析本身

---

## 哪些场景会走沙箱

1. **BashTool 默认会走**：只要平台支持、沙箱开启、命令未被显式排除，就会进入沙箱执行
2. **未被显式绕过的调用**：如果本次调用没有设置 `dangerouslyDisableSandbox: true`，且策略未将其列入排除列表
3. **Linux 是唯一受完整支持的沙箱平台**：通过 `unshare` 的用户/挂载/网络命名空间实现隔离；macOS 和 Windows 原生不具备同等 OS 级隔离路径

### 启动期判断

沙箱不是等到第一条命令执行时才临时判断的。REPL / CLI 启动时，会通过 `resolve_sandbox_status()` 先检查当前环境是否真的具备沙箱条件。对应的 `/sandbox` 和 `claw sandbox` 命令也能直接输出当前解析后的沙箱状态快照。

---

## 完整执行链路

### 1. 配置解析

用户的 `.claw.json` 或 `.claw/settings.json` 中的 `sandbox` 字段由 `ConfigLoader` 解析。参见 [`runtime/src/config.rs#L865-L923`](/rust/crates/runtime/src/config.rs#L865-L923)：

```rust
fn parse_optional_sandbox_config(root: &JsonValue) -> Result<SandboxConfig, ConfigError> {
    // ... 从 JSON 中提取 enabled、namespaceRestrictions、
    //     networkIsolation、filesystemMode、allowedMounts
}
```

支持的 `filesystemMode` 有三个枚举值（[`runtime/src/sandbox.rs#L9-L14`](/rust/crates/runtime/src/sandbox.rs#L9-L14)）：

```rust
pub enum FilesystemIsolationMode {
    Off,
    WorkspaceOnly,
    AllowList,
}
```

### 2. 请求解析与默认值填充

`SandboxConfig` 通过 `resolve_request()` 把配置和单次调用的覆盖值合并为一次具体的 `SandboxRequest`。参见 [`runtime/src/sandbox.rs#L87-L105`](/rust/crates/runtime/src/sandbox.rs#L87-L105)：

```rust
pub fn resolve_request(
    &self,
    enabled_override: Option<bool>,
    namespace_override: Option<bool>,
    network_override: Option<bool>,
    filesystem_mode_override: Option<FilesystemIsolationMode>,
    allowed_mounts_override: Option<Vec<String>>,
) -> SandboxRequest { ... }
```

这里的 `enabled_override` 正是 Bash 输入中 `dangerously_disable_sandbox` 的反面——如果模型显式传了 `dangerouslyDisableSandbox: true`，就会在这里把 `enabled` 设为 `false`。

### 3. 状态解析

`resolve_sandbox_status_for_request()` 决定这次请求最终能否激活沙箱。核心逻辑在 [`runtime/src/sandbox.rs#L162-L208`](/rust/crates/runtime/src/sandbox.rs#L162-L208)：

- `namespace_supported`：仅 Linux 且 `unshare --user` 真实可用时为 `true`
- `network_supported`：与 `namespace_supported` 同值
- `active`：请求开启且依赖的条件都满足时才为 `true`
- 如果不满足条件，`fallback_reason` 会记录具体原因（例如 `"namespace isolation unavailable (requires Linux with `unshare`)"`）

容器检测也在这里发生。`detect_container_environment()`（[`L109-L153`](/rust/crates/runtime/src/sandbox.rs#L109-L153)）会检查：

- `/.dockerenv` 是否存在
- `/run/.containerenv` 是否存在
- 环境变量中是否有 `CONTAINER`、`DOCKER`、`PODMAN`、`KUBERNETES_SERVICE_HOST`
- `/proc/1/cgroup` 中是否包含 `docker`、`containerd`、`kubepods`、`podman`、`libpod`

### 4. Bash 执行体中的沙箱集成

真正执行 Bash 命令的是 `runtime/src/bash.rs` 中的 `execute_bash()`（[`L70-L103`](/rust/crates/runtime/src/bash.rs#L70-L103)）。它首先调用 `sandbox_status_for_input()`（[`L170-L183`](/rust/crates/runtime/src/bash.rs#L170-L183)）：

```rust
fn sandbox_status_for_input(input: &BashCommandInput, cwd: &std::path::Path) -> SandboxStatus {
    let config = ConfigLoader::default_for(cwd).load().map_or_else(
        |_| SandboxConfig::default(),
        |runtime_config| runtime_config.sandbox().clone(),
    );
    let request = config.resolve_request(
        input.dangerously_disable_sandbox.map(|disabled| !disabled),
        input.namespace_restrictions,
        input.isolate_network,
        input.filesystem_mode,
        input.allowed_mounts.clone(),
    );
    resolve_sandbox_status_for_request(&request, cwd)
}
```

这段代码精确展示了 `dangerously_disable_sandbox` 的处理逻辑：

- 如果传入 `Some(true)`，`enabled_override` 会变成 `Some(false)`，即**禁用沙箱**
- 如果传入 `Some(false)` 或未传入，则走配置默认值（默认启用）

随后 `prepare_tokio_command()`（[`L212-L237`](/rust/crates/runtime/src/bash.rs#L212-L237)）会决定如何构造命令：

1. 先创建 `.sandbox-home` 和 `.sandbox-tmp` 目录
2. 如果 `build_linux_sandbox_command()` 返回了 `Some(launcher)`，则用 `unshare` 包裹原始命令
3. 否则退化为普通 `sh -lc`，但如果 `filesystem_active` 为 `true`，仍然会把 `HOME` 和 `TMPDIR` 重定向到工作区内的临时目录

### 5. unshare 命令的构造

`build_linux_sandbox_command()`（[`runtime/src/sandbox.rs#L211-L262`](/rust/crates/runtime/src/sandbox.rs#L211-L262)）在 Linux 且沙箱激活时，会生成如下结构的命令：

```rust
LinuxSandboxCommand {
    program: "unshare".to_string(),
    args: vec![
        "--user", "--map-root-user", "--mount", "--ipc",
        "--pid", "--uts", "--fork",
        // 若 network_active 为 true，再追加 "--net"
        "sh", "-lc", command,
    ],
    env: vec![
        ("HOME", cwd/.sandbox-home),
        ("TMPDIR", cwd/.sandbox-tmp),
        ("CLAWD_SANDBOX_FILESYSTEM_MODE", ...),
        ("CLAWD_SANDBOX_ALLOWED_MOUNTS", ...),
        ("PATH", ...),
    ],
}
```

这意味着：

- **用户命名空间**：`--user --map-root-user` 让进程以为自己有 root 权限，但实际上只是映射到当前 UID
- **挂载命名空间**：`--mount` 隔离文件系统挂载视图
- **IPC / PID / UTS 命名空间**：进一步减少进程对外部系统的可见性
- **网络命名空间**：`--net` 在网络隔离开启时添加，子进程只能看到 loopback，无法访问外部网络

### 6. 挂载路径规范化

`allowed_mounts` 在生效前会经过 `normalize_mounts()`（[`L264-L278`](/rust/crates/runtime/src/sandbox.rs#L264-L278)）：

```rust
fn normalize_mounts(mounts: &[String], cwd: &Path) -> Vec<String> {
    mounts
        .iter()
        .map(|mount| {
            let path = PathBuf::from(mount);
            if path.is_absolute() { path } else { cwd.join(path) }
        })
        .map(|path| path.display().to_string())
        .collect()
}
```

相对路径会基于当前工作目录解析为绝对路径，确保后续沙箱层不会误判。

### 7. 能力探测缓存

`unshare` 是否真实可用，会由 `unshare_user_namespace_works()` 判断（[`L288-L304`](/rust/crates/runtime/src/sandbox.rs#L288-L304)）。该结果通过 `std::sync::OnceLock` 缓存，避免每次命令执行都重复探测：

```rust
fn unshare_user_namespace_works() -> bool {
    static RESULT: OnceLock<bool> = OnceLock::new();
    *RESULT.get_or_init(|| {
        if !command_exists("unshare") { return false; }
        std::process::Command::new("unshare")
            .args(["--user", "--map-root-user", "true"])
            // ... 静默执行并检查退出码
            .status()
            .map(|s| s.success())
            .unwrap_or(false)
    })
}
```

这解释了为什么某些 CI 环境（如 GitHub Actions）即使 `unshare` 二进制存在，也会被判定为不支持——因为内核禁用了用户命名空间。

---

## 默认沙箱到底限制了什么

### 文件系统

- `FilesystemIsolationMode::Off`：不限制
- `FilesystemIsolationMode::WorkspaceOnly`：默认限制在工作目录内
- `FilesystemIsolationMode::AllowList`：额外限制在配置的 `allowed_mounts` 列表内

### 网络

- `network_isolation` 为 `true` 时，子进程会被 `--net` 隔离，无法访问外部网络
- 默认值为 `false`，即网络不隔离（参见 `resolve_request` 的默认值）

### 沙箱内环境变量

即使命令最终没有走到 `unshare`，只要 `filesystem_active` 为 `true`，`HOME` 和 `TMPDIR` 也会被重写到工作区内的 `.sandbox-home` 和 `.sandbox-tmp`，避免向用户真实 home 目录落文件。

---

## dangerouslyDisableSandbox 处理逻辑

`dangerouslyDisableSandbox` 被设计成一条显眼的安全逃生通道：

- 在 `BashCommandInput` 中被序列化为 `dangerouslyDisableSandbox`（[`runtime/src/bash.rs#L25-L26`](/rust/crates/runtime/src/bash.rs#L25-L26)）
- 在工具 JSON Schema 中同样暴露为 `dangerouslyDisableSandbox: { type: "boolean" }`（[`tools/src/lib.rs#L397`](/rust/crates/tools/src/lib.rs#L397)）
- 在 `sandbox_status_for_input()` 中被反转后传入 `resolve_request(..., enabled_override, ...)`
- 输出 schema 会把原值回显在 `dangerously_disable_sandbox` 字段中（[`runtime/src/bash.rs#L57-L58`](/rust/crates/runtime/src/bash.rs#L57-L58)），便于审计

保留这个入口的原因是：有些真实开发任务确实需要越过默认边界（例如访问未加入白名单的工具链目录、调试系统级环境）。但命名故意很长很重，提醒这是例外路径。

---

## 平台差异

| 平台 | 沙箱支持情况 |
|------|-------------|
| Linux | 完整支持，通过 `unshare` 实现用户/挂载/网络命名空间隔离 |
| macOS | 由仓库判定为 `namespace_supported = false`，因此不会走 `unshare` 路径，仅保留 `HOME`/`TMPDIR` 重定向 |
| Windows 原生 | 不支持，`namespace_supported = false`，沙箱退化为普通 `sh` 执行 |
| WSL | 若内核启用了用户命名空间，则与 Linux 行为一致 |

---

## 用户能看到什么：/sandbox 命令与 CLI 输出

在 REPL 中输入 `/sandbox`，或在 shell 中运行 `claw sandbox`，都会输出当前目录下的沙箱解析状态。参见 [`rusty-claude-cli/src/main.rs#L2817-L2829`](/rust/crates/rusty-claude-cli/src/main.rs#L2817-L2829)：

```rust
SlashCommand::Sandbox => {
    let cwd = env::current_dir()?;
    let loader = ConfigLoader::default_for(&cwd);
    let runtime_config = loader.load()?;
    let status = resolve_sandbox_status(runtime_config.sandbox(), &cwd);
    Ok(ResumeCommandOutcome {
        session: session.clone(),
        message: Some(format_sandbox_report(&status)),
        json: Some(sandbox_json_value(&status)),
    })
}
```

输出包含：是否启用、是否激活、平台是否支持、是否在容器内、命名空间和网络的请求值/实际值、文件系统模式、允许挂载列表、容器标记、降级原因等。

---

## 常见误区

### 误区 1：沙箱会保护所有文件修改

**不是。** `FileEditTool` / `FileWriteTool` 走的是直接文件 I/O，不经过 Bash 执行链路，因此不受沙箱保护。它们依赖应用层的权限系统和工作区边界检查。

### 误区 2：只要启用了沙箱，就不再需要权限系统

**不是。** 沙箱只限制进程能力，不负责解释用户意图、路径安全语义、工具模式、审批体验。`allow / ask / deny` 体系仍然是必须的。

### 误区 3：如果某个危险操作被沙箱拦住，就说明应用层检查没价值

**不是。** 应用层检查的价值在于更早提示、更好用户体验、更细的语义判断；沙箱负责的是最终兜底。两者是 Defense-in-Depth 的不同层级。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/sandbox.rs`](/rust/crates/runtime/src/sandbox.rs) | `SandboxConfig`、`SandboxRequest`、`SandboxStatus`、`FilesystemIsolationMode`、`build_linux_sandbox_command`、`detect_container_environment`、`resolve_sandbox_status_for_request` |
| [`/rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs) | `BashCommandInput`、`execute_bash`、`sandbox_status_for_input`、`prepare_tokio_command`、`dangerously_disable_sandbox` 处理 |
| [`/rust/crates/runtime/src/config.rs`](/rust/crates/runtime/src/config.rs) | `parse_optional_sandbox_config`、配置文件中 `sandbox` 字段的解析 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | `CliAction::Sandbox`、`/sandbox` slash 命令、`format_sandbox_report`、`print_sandbox_status_snapshot` |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | Bash 工具的 JSON Schema 定义，包括 `dangerouslyDisableSandbox` 字段 |
| [`/rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `SlashCommand::Sandbox` 的注册与解析 |
