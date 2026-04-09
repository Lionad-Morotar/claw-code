# claw-code 隐藏功能技术报告

> 本报告基于 ccb.agent-aura.top/docs/internals/hidden-features 目录结构，映射到 `claw-code`（Claude Code Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 摘要

claw-code 实现了 **8 个隐藏功能**（Hidden Features），这些功能不通过常规 REPL 帮助菜单的高频区展示，但均已在 `SLASH_COMMAND_SPECS` 静态注册，可通过特定 slash command 直接调用。其中 Bughunter、Ultraplan、Teleport、DebugToolCall 在 REPL 层直接返回格式化报告；Team/Cron 通过 `DangerFullAccess` 级别的 MCP 工具执行；Sandbox 和 Doctor 则提供系统状态诊断。

| 功能 | Slash Command | 实际执行位置 | 权限要求 |
|------|---------------|--------------|----------|
| **Bughunter** | `/bughunter [scope]` | `main.rs#L4336` | 无额外限制 |
| **Ultraplan** | `/ultraplan [task]` | `main.rs#L4341` | 无额外限制 |
| **Teleport** | `/teleport <path>` | `main.rs#L4346` | 无额外限制 |
| **DebugToolCall** | `/debug-tool-call` | `main.rs#L4356` | 无额外限制 |
| **Team** | `/team [action]` | `tools.rs#L1521` | `DangerFullAccess` |
| **Cron** | `/cron [list\|add\|remove]` | `tools.rs#L1556` | `DangerFullAccess` |
| **Sandbox** | `/sandbox` | `main.rs#L3827` | 无额外限制 |
| **Doctor** | `/doctor` | `main.rs#L1351` | 无额外限制 |

---

## 1. Bughunter — 代码缺陷检测器

### 功能描述

Bughunter 是一个内部代码审查工具，可扫描指定作用域内的代码并识别潜在缺陷。它输出包含文件路径、严重性等级和建议修复方案的报告。

### 源码锚点

**命令定义** [`commands/src/lib.rs#L166-L172`](/rust/crates/commands/src/lib.rs#L166-L172)：

```rust
SlashCommandSpec {
    name: "bughunter",
    aliases: &[],
    summary: "Inspect code for likely bugs and correctness issues",
    argument_hint: Some("[scope]"),
    resume_supported: false,
}
```

**命令解析** [`commands/src/lib.rs#L1263`](/rust/crates/commands/src/lib.rs#L1263)：

```rust
"bughunter" => SlashCommand::Bughunter { scope: remainder },
```

**执行入口** [`rusty-claude-cli/src/main.rs#L4336-L4340`](/rust/crates/rusty-claude-cli/src/main.rs#L4336-L4339)：

```rust
fn run_bughunter(&self, scope: Option<&str>) -> Result<(), Box<dyn std::error::Error>> {
    println!("{}", format_bughunter_report(scope));
    Ok(())
}
```

**报告格式化** [`rusty-claude-cli/src/main.rs#L5321-L5331`](/rust/crates/rusty-claude-cli/src/main.rs#L5321-L5329)：

```rust
fn format_bughunter_report(scope: Option<&str>) -> String {
    format!(
        "Bughunter
  Scope            {}
  Action           inspect the selected code for likely bugs and correctness issues
  Output           findings should include file paths, severity, and suggested fixes",
        scope.unwrap_or("the current repository")
    )
}
```

### 调用示例

```bash
# 扫描整个仓库
/bughunter

# 扫描指定模块
/bughunter runtime
```

---

## 2. Ultraplan — 多步规划引擎

### 功能描述

Ultraplan 是一个深度规划工具，可将复杂任务分解为多步执行计划。它支持进度跟踪和阶段性输出，适用于需要多轮工具调用的大型任务。

### 源码锚点

**命令定义** [`commands/src/lib.rs#L194-L200`](/rust/crates/commands/src/lib.rs#L194-L200)：

```rust
SlashCommandSpec {
    name: "ultraplan",
    aliases: &[],
    summary: "Run a deep planning prompt with multi-step reasoning",
    argument_hint: Some("[task]"),
    resume_supported: false,
}
```

**命令解析** [`commands/src/lib.rs#L1270`](/rust/crates/commands/src/lib.rs#L1270)：

```rust
"ultraplan" => SlashCommand::Ultraplan { task: remainder },
```

**执行入口** [`rusty-claude-cli/src/main.rs#L4341-L4344`](/rust/crates/rusty-claude-cli/src/main.rs#L4341-L4344)：

```rust
fn run_ultraplan(&self, task: Option<&str>) -> Result<(), Box<dyn std::error::Error>> {
    println!("{}", format_ultraplan_report(task));
    Ok(())
}
```

**进度追踪器** [`rusty-claude-cli/src/main.rs#L5944-L5980`](/rust/crates/rusty-claude-cli/src/main.rs#L5944-L5980)：

```rust
impl InternalPromptProgressReporter {
    fn ultraplan(task: &str) -> Self {
        Self {
            shared: Arc::new(InternalPromptProgressShared {
                state: Mutex::new(InternalPromptProgressState {
                    command_label: "Ultraplan",
                    task_label: task.to_string(),
                    step: 0,
                    phase: "planning started".to_string(),
                    detail: Some(format!("task: {task}")),
                    saw_final_text: false,
                }),
                output_lock: Mutex::new(()),
                started_at: Instant::now(),
            }),
        }
    }
}
```

**运行周期** [`rusty-claude-cli/src/main.rs#L6088-L6110`](/rust/crates/rusty-claude-cli/src/main.rs#L6088-L6110)：

```rust
impl InternalPromptProgressRun {
    fn start_ultraplan(task: &str) -> Self {
        let reporter = InternalPromptProgressReporter::ultraplan(task);
        reporter.emit(InternalPromptProgressEvent::Started, None);

        let (heartbeat_stop, heartbeat_rx) = mpsc::channel();
        let heartbeat_reporter = reporter.clone();
        let heartbeat_handle = thread::spawn(move || loop {
            match heartbeat_rx.recv_timeout(INTERNAL_PROGRESS_HEARTBEAT_INTERVAL) {
                Ok(()) | Err(RecvTimeoutError::Disconnected) => break,
                Err(RecvTimeoutError::Timeout) => heartbeat_reporter.emit_heartbeat(),
            }
        });

        Self {
            reporter,
            heartbeat_stop: Some(heartbeat_stop),
            heartbeat_handle: Some(heartbeat_handle),
        }
    }
}
```

### 调用示例

```bash
# 规划发布流程
/ultraplan ship the release

# 无参数调用（默认当前仓库工作）
/ultraplan
```

---

## 3. Teleport — 工作区跳转器

### 功能描述

Teleport 允许用户通过搜索工作区快速跳转到指定文件或符号。它支持路径和符号两种目标格式。

### 源码锚点

**命令定义** [`commands/src/lib.rs#L201-L208`](/rust/crates/commands/src/lib.rs#L201-L208)：

```rust
SlashCommandSpec {
    name: "teleport",
    aliases: &[],
    summary: "Jump to a file or symbol by searching the workspace",
    argument_hint: Some("<symbol-or-path>"),
    resume_supported: false,
}
```

**命令解析** [`commands/src/lib.rs#L1271-L1274`](/rust/crates/commands/src/lib.rs#L1271-L1274)：

```rust
"teleport" => SlashCommand::Teleport {
    target: Some(require_remainder(command, remainder, "<symbol-or-path>")?),
},
```

**执行入口** [`rusty-claude-cli/src/main.rs#L4346-L4354`](/rust/crates/rusty-claude-cli/src/main.rs#L4346-L4354)：

```rust
fn run_teleport(target: Option<&str>) -> Result<(), Box<dyn std::error::Error>> {
    let Some(target) = target.map(str::trim).filter(|value| !value.is_empty()) else {
        println!("Usage: /teleport <symbol-or-path>");
        return Ok(());
    };

    println!("{}", render_teleport_report(target)?);
    Ok(())
}
```

### 调用示例

```bash
# 跳转到符号
/teleport ConversationRuntime

# 跳转到文件
/teleport src/main.rs
```

---

## 4. DebugToolCall — 工具调用调试器

### 功能描述

DebugToolCall 用于回放最后一次工具调用并显示调试详情，包括输入参数、输出结果和执行时间。

### 源码锚点

**命令定义** [`commands/src/lib.rs#L208-L215`](/rust/crates/commands/src/lib.rs#L208-L215)：

```rust
SlashCommandSpec {
    name: "debug-tool-call",
    aliases: &[],
    summary: "Replay the last tool call with debug details",
    argument_hint: None,
    resume_supported: false,
}
```

**命令解析** [`commands/src/lib.rs#L1274-L1277`](/rust/crates/commands/src/lib.rs#L1274-L1277)：

```rust
"debug-tool-call" => {
    validate_no_args(command, &args)?;
    SlashCommand::DebugToolCall
}
```

**执行入口** [`rusty-claude-cli/src/main.rs#L4356-L4360`](/rust/crates/rusty-claude-cli/src/main.rs#L4356-L4360)：

```rust
fn run_debug_tool_call(&self, args: Option<&str>) -> Result<(), Box<dyn std::error::Error>> {
    validate_no_args("/debug-tool-call", args)?;
    println!("{}", render_last_tool_debug_report(self.runtime.session())?);
    Ok(())
}
```

### 调用示例

```bash
# 回放最后一次工具调用
/debug-tool-call
```

---

## 5. Team — 多代理团队管理

### 功能描述

Team 功能允许创建和管理多个子代理组成的团队，用于并行任务执行。支持团队的创建、删除和状态查询。

### 源码锚点

**命令定义** [`commands/src/lib.rs#L814-L821`](/rust/crates/commands/src/lib.rs#L814-L821)：

```rust
SlashCommandSpec {
    name: "team",
    aliases: &[],
    summary: "Manage agent teams",
    argument_hint: Some("[list|create|delete]"),
    resume_supported: true,
}
```

**工具定义** [`tools/src/lib.rs#L982-L1055`](/rust/crates/tools/src/lib.rs#L982-L1055)：

```rust
ToolSpec {
    name: "TeamCreate",
    description: "Create a team of sub-agents for parallel task execution.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "name": { "type": "string" },
            "tasks": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "prompt": { "type": "string" },
                        "description": { "type": "string" }
                    },
                    "required": ["prompt"]
                }
            }
        },
        "required": ["name", "tasks"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::DangerFullAccess,
},
ToolSpec {
    name: "TeamDelete",
    description: "Delete a team and stop all its running tasks.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "team_id": { "type": "string" }
        },
        "required": ["team_id"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::DangerFullAccess,
},
```

**执行入口** [`tools/src/lib.rs#L1521-L1543`](/rust/crates/tools/src/lib.rs#L1521-L1542)：

```rust
fn run_team_create(input: TeamCreateInput) -> Result<String, String> {
    let task_ids: Vec<String> = input
        .tasks
        .iter()
        .filter_map(|t| t.get("task_id").and_then(|v| v.as_str()).map(str::to_owned))
        .collect();
    let team = global_team_registry().create(&input.name, task_ids);
    // Register team assignment on each task
    for task_id in &team.task_ids {
        let _ = global_task_registry().assign_team(task_id, &team.team_id);
    }

    to_pretty_json(json!({
        "team_id": team.team_id,
        "name": team.name,
        "task_count": team.task_ids.len(),
        "task_ids": team.task_ids,
        "status": team.status,
        "created_at": team.created_at
    }))
}
```

**删除实现** [`tools/src/lib.rs#L1543-L1555`](/rust/crates/tools/src/lib.rs#L1543-L1555)：

```rust
fn run_team_delete(input: TeamDeleteInput) -> Result<String, String> {
    match global_team_registry().delete(&input.team_id) {
        Ok(team) => to_pretty_json(json!({
            "team_id": team.team_id,
            "name": team.name,
            "status": team.status,
            "message": "Team deleted"
        })),
        Err(e) => Err(e),
    }
}
```

**团队注册表** [`runtime/src/team_cron_registry.rs#L21-L90`](/rust/crates/runtime/src/team_cron_registry.rs#L21-L90)：

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Team {
    pub team_id: String,
    pub name: String,
    pub task_ids: Vec<String>,
    pub status: TeamStatus,
    pub created_at: u64,
    pub updated_at: u64,
}

impl TeamRegistry {
    pub fn create(&self, name: &str, task_ids: Vec<String>) -> Team {
        let mut inner = self.inner.lock().expect("team registry lock poisoned");
        inner.counter += 1;
        let ts = now_secs();
        let team_id = format!("team_{:08x}_{}", ts, inner.counter);
        let team = Team {
            team_id: team_id.clone(),
            name: name.to_owned(),
            task_ids,
            status: TeamStatus::Created,
            created_at: ts,
            updated_at: ts,
        };
        inner.teams.insert(team_id, team.clone());
        team
    }
}
```

### 调用示例

```bash
# 列出所有团队
/team list

# 创建新团队（通过 MCP 工具）
# TeamCreate { name: "Alpha", tasks: [...] }
```

---

## 6. Cron — 定时任务调度器

### 功能描述

Cron 功能提供定时任务的创建、删除和列表查询，支持周期性自动执行指定任务。

### 源码锚点

**命令定义** [`commands/src/lib.rs#L805-L812`](/rust/crates/commands/src/lib.rs#L805-L812)：

```rust
SlashCommandSpec {
    name: "cron",
    aliases: &[],
    summary: "Manage scheduled tasks",
    argument_hint: Some("[list|add|remove]"),
    resume_supported: true,
}
```

**工具定义** [`tools/src/lib.rs#L1019-L1070`](/rust/crates/tools/src/lib.rs#L1019-L1070)：

```rust
ToolSpec {
    name: "CronCreate",
    description: "Create a scheduled recurring task.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "schedule": { "type": "string" },
            "prompt": { "type": "string" },
            "description": { "type": "string" }
        },
        "required": ["schedule", "prompt"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::DangerFullAccess,
},
ToolSpec {
    name: "CronDelete",
    description: "Delete a scheduled recurring task by ID.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "cron_id": { "type": "string" }
        },
        "required": ["cron_id"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::DangerFullAccess,
},
ToolSpec {
    name: "CronList",
    description: "List all scheduled recurring tasks.",
    input_schema: json!({
        "type": "object",
        "properties": {},
        "additionalProperties": false
    }),
    required_permission: PermissionMode::ReadOnly,
},
```

**注册表实现在** `runtime/src/team_cron_registry.rs`（与 Team 共享同一文件）。

---

## 7. Sandbox — 沙箱状态检查器

### 功能描述

Sandbox 命令显示当前会话的沙箱隔离状态，包括是否启用、保护级别和网络隔离情况。

### 源码锚点

**命令定义** [`commands/src/lib.rs#L75-L81`](/rust/crates/commands/src/lib.rs#L75-L81)：

```rust
SlashCommandSpec {
    name: "sandbox",
    aliases: &[],
    summary: "Show sandbox isolation status",
    argument_hint: None,
    resume_supported: true,
}
```

**CLI Action 定义** [`rusty-claude-cli/src/main.rs#L229-L232`](/rust/crates/rusty-claude-cli/src/main.rs#L229-L232)：

```rust
Sandbox {
    output_format: CliOutputFormat,
},
```

**Resume 分发** [`rusty-claude-cli/src/main.rs#L2606-L2614`](/rust/crates/rusty-claude-cli/src/main.rs#L2606-L2614) 与 **Slash 分发** [`rusty-claude-cli/src/main.rs#L3621-L3625`](/rust/crates/rusty-claude-cli/src/main.rs#L3621-L3625)：

```rust
SlashCommand::Sandbox => {
    let cwd = env::current_dir()?;
    let loader = ConfigLoader::default_for(&cwd);
    let runtime_config = loader.load()?;
    let status = resolve_sandbox_status(runtime_config.sandbox(), &cwd);
    println!("{}", format_sandbox_report(&status));
    false
}
```

**独立状态打印** [`rusty-claude-cli/src/main.rs#L3819-L3829`](/rust/crates/rusty-claude-cli/src/main.rs#L3819-L3829)：

```rust
fn print_sandbox_status() {
    let cwd = env::current_dir().expect("current dir");
    let loader = ConfigLoader::default_for(&cwd);
    let runtime_config = loader
        .load()
        .unwrap_or_else(|_| runtime::RuntimeConfig::empty());
    println!(
        "{}",
        format_sandbox_report(&resolve_sandbox_status(runtime_config.sandbox(), &cwd))
    );
}
```

**Doctor 诊断集成** [`rusty-claude-cli/src/main.rs#L1726-L1760`](/rust/crates/rusty-claude-cli/src/main.rs#L1726-L1760)：

```rust
fn check_sandbox_health(status: &runtime::SandboxStatus) -> DiagnosticCheck {
    let degraded = status.enabled && !status.active;
    let mut details = vec![
        format!("Enabled          {}", status.enabled),
        format!("Active           {}", status.active),
        format!("Supported        {}", status.supported),
        format!("Filesystem mode  {}", status.filesystem_mode.as_str()),
        format!("Filesystem live  {}", status.filesystem_active),
    ];
    if let Some(reason) = &status.fallback_reason {
        details.push(format!("Fallback reason  {reason}"));
    }
    DiagnosticCheck::new(
        "Sandbox",
        if degraded {
            DiagnosticLevel::Warn
        } else {
            DiagnosticLevel::Ok
        },
        if degraded {
            "sandbox was requested but is not currently active"
        } else if status.active {
            "sandbox protections are active"
        } else {
            "sandbox is not active for this session"
        },
    )
    .with_details(details)
    .with_data(Map::from_iter([
        ("enabled".to_string(), json!(status.enabled)),
        ("active".to_string(), json!(status.active)),
    ]))
}
```

### 调用示例

```bash
# 查看沙箱状态
/sandbox

# 在诊断报告中查看
/doctor
```

---

## 8. Doctor — 系统健康诊断器

### 功能描述

Doctor 是一个综合性的系统诊断工具，检查认证、配置、工作区、沙箱和系统健康状态，输出结构化的诊断报告。

### 源码锚点

**命令定义** [`commands/src/lib.rs#L254-L261`](/rust/crates/commands/src/lib.rs#L254-L261)：

```rust
SlashCommandSpec {
    name: "doctor",
    aliases: &[],
    summary: "Run diagnostic checks on your Claude Code setup",
    argument_hint: None,
    resume_supported: true,
}
```

**CLI Action 定义** [`rusty-claude-cli/src/main.rs#L232`](/rust/crates/rusty-claude-cli/src/main.rs#L232)：

```rust
CliAction::Doctor { output_format } => run_doctor(output_format)?,
```

**执行入口** [`rusty-claude-cli/src/main.rs#L1351-L1365`](/rust/crates/rusty-claude-cli/src/main.rs#L1351-L1364)：

```rust
fn run_doctor(output_format: CliOutputFormat) -> Result<(), Box<dyn std::error::Error>> {
    let report = render_doctor_report()?;
    let message = report.render();
    match output_format {
        CliOutputFormat::Text => println!("{message}"),
        CliOutputFormat::Json => {
            println!("{}", serde_json::to_string_pretty(&report.json_value())?);
        }
    }
    if report.has_failures() {
        std::process::exit(1);
    }
    Ok(())
}
```

**诊断报告渲染** [`rusty-claude-cli/src/main.rs#L1315-L1363`](/rust/crates/rusty-claude-cli/src/main.rs#L1315-L1363)：

```rust
fn render_doctor_report() -> Result<DoctorReport, Box<dyn std::error::Error>> {
    let cwd = env::current_dir()?;
    let config_loader = ConfigLoader::default_for(&cwd);
    let config = config_loader.load();
    let discovered_config = config_loader.discover()?;
    let project_context = ProjectContext::discover_with_git(&cwd, DEFAULT_DATE)?;
    let (project_root, git_branch) =
        parse_git_status_metadata(project_context.git_status.as_deref());
    let git_summary = parse_git_workspace_summary(project_context.git_status.as_deref());
    let empty_config = runtime::RuntimeConfig::empty();
    let sandbox_config = config.as_ref().ok().unwrap_or(&empty_config);
    let context = StatusContext {
        cwd: cwd.clone(),
        session_path: None,
        loaded_config_files: config
            .as_ref()
            .ok()
            .map_or(0, |runtime_config| runtime_config.loaded_entries().len()),
        discovered_config_files: discovered_config.len(),
        memory_file_count: project_context.instruction_files.len(),
        project_root,
        git_branch,
        git_summary,
        sandbox_status: resolve_sandbox_status(sandbox_config.sandbox(), &cwd),
    };
    Ok(DoctorReport {
        checks: vec![
            check_auth_health(),
            check_config_health(&config_loader, config.as_ref()),
            check_workspace_health(&context),
            check_sandbox_health(&context.sandbox_status),
            check_system_health(&cwd, config.as_ref().ok()),
        ],
    }))
}
```

### 调用示例

```bash
# 运行诊断
/doctor

# 从 CLI 启动
claw doctor
```

---

## 附加隐藏入口

### Daemon 相关

在 compat-harness 中检测到 daemon 相关快速路径：

**Bootstrap Phase 定义** [`runtime/src/bootstrap.rs#L2-L14`](/rust/crates/runtime/src/bootstrap.rs#L2-L14)：

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum BootstrapPhase {
    CliEntry,
    FastPathVersion,
    StartupProfiler,
    SystemPromptFastPath,
    ChromeMcpFastPath,
    DaemonWorkerFastPath,
    BridgeFastPath,
    DaemonFastPath,
    BackgroundSessionFastPath,
    TemplateFastPath,
    EnvironmentRunnerFastPath,
    MainRuntime,
}
```

**检测逻辑** [`compat-harness/src/lib.rs#L198-L210`](/rust/crates/compat-harness/src/lib.rs#L198-L210)：

```rust
if source.contains("--daemon-worker") {
    phases.push(BootstrapPhase::DaemonWorkerFastPath);
}
if source.contains("args[0] === 'daemon'") {
    phases.push(BootstrapPhase::DaemonFastPath);
}
```

---

## 权限边界说明

在当前 Rust 实现中，**斜杠命令（slash command）本身不附带权限检查** —— 任何在 REPL 中输入 `/bughunter`、`/ultraplan`、`/sandbox`、`/doctor` 等命令的用户都会直接触发对应 handler。真正的权限门控发生在 **MCP 工具层**：

| 层级 | 受控对象 | 权限机制 |
|------|----------|----------|
| **Slash Commands** | Bughunter、Ultraplan、Teleport、DebugToolCall、Sandbox、Doctor | 无额外限制，REPL 中任意调用 |
| **MCP 工具** | TeamCreate / TeamDelete、CronCreate / CronDelete | `required_permission: PermissionMode::DangerFullAccess` |
| **MCP 工具** | CronList | `required_permission: PermissionMode::ReadOnly` |

这意味着 `Team` 和 `Cron` 的实际危险操作（创建团队、调度后台任务）受到 `DangerFullAccess` 约束；而仅输出格式化报告的隐藏命令则对 REPL 用户完全开放。若未来需要为 Bughunter / Ultraplan 增加自动写入能力，可以在对应的 `SlashCommandSpec` 或工具调用路径前补充权限校验，而不是现在就在文档中预设一个不存在的权限边界。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`rust/crates/commands/src/lib.rs`](/rust/crates/commands/src/lib.rs) | `SlashCommand` 枚举、解析逻辑、`handle_slash_command` |
| [`rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、隐藏命令执行器、进度追踪器 |
| [`rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `TeamCreate/Delete`、`CronCreate/Delete/List` 工具实现 |
| [`rust/crates/runtime/src/team_cron_registry.rs`](/rust/crates/runtime/src/team_cron_registry.rs) | `TeamRegistry`、`CronRegistry` 注册表 |
| [`rust/crates/runtime/src/task_registry.rs`](/rust/crates/runtime/src/task_registry.rs) | `TaskRegistry`、团队任务分配 |
| [`rust/crates/runtime/src/bootstrap.rs`](/rust/crates/runtime/src/bootstrap.rs) | `BootstrapPhase`、Daemon 快速路径 |
| [`rust/crates/compat-harness/src/lib.rs`](/rust/crates/compat-harness/src/lib.rs) | Daemon 检测逻辑 |

---

## 测试覆盖

关键测试用例：

**Ultraplan 进度测试** [`rusty-claude-cli/src/main.rs#L10533-L10572`](/rust/crates/rusty-claude-cli/src/main.rs#L10533-L10572)：

```rust
#[test]
fn ultraplan_progress_lines_include_phase_step_and_elapsed_status() {
    let snapshot = InternalPromptProgressState {
        command_label: "Ultraplan",
        task_label: "ship plugin progress".to_string(),
        step: 3,
        phase: "running read_file".to_string(),
        detail: Some("reading rust/crates/rusty-claude-cli/src/main.rs".to_string()),
        saw_final_text: false,
    };
    // ... 测试 Started/Heartbeat/Complete/Failed 事件
}
```

**命令解析测试** [`commands/src/lib.rs#L4089-L4128`](/rust/crates/commands/src/lib.rs#L4089-L4128)：

```rust
#[test]
fn parses_supported_slash_commands() {
    assert_eq!(SlashCommand::parse("/help"), Ok(Some(SlashCommand::Help)));
    assert_eq!(
        SlashCommand::parse(" /status "),
        Ok(Some(SlashCommand::Status))
    );
    assert_eq!(
        SlashCommand::parse("/sandbox"),
        Ok(Some(SlashCommand::Sandbox))
    );
    assert_eq!(
        SlashCommand::parse("/bughunter runtime"),
        Ok(Some(SlashCommand::Bughunter {
            scope: Some("runtime".to_string())
        }))
    );
    assert_eq!(
        SlashCommand::parse("/ultraplan ship both features"),
        Ok(Some(SlashCommand::Ultraplan {
            task: Some("ship both features".to_string())
        }))
    );
    // ... 更多测试
}
```

**团队注册表测试** [`runtime/src/team_cron_registry.rs#L251-L283`](/rust/crates/runtime/src/team_cron_registry.rs#L251-L283)：

```rust
#[test]
fn lists_and_deletes_teams() {
    let registry = TeamRegistry::new();
    let t1 = registry.create("Team A", vec![]);
    let t2 = registry.create("Team B", vec![]);
    let all = registry.list();
    assert_eq!(all.len(), 2);
    let deleted = registry.delete(&t1.team_id).expect("delete should succeed");
    assert_eq!(deleted.status, TeamStatus::Deleted);
    let still_there = registry.get(&t1.team_id).unwrap();
    assert_eq!(still_there.status, TeamStatus::Deleted);
    registry.remove(&t2.team_id);
    assert_eq!(registry.len(), 1);
}
```

---

## 设计总结

claw-code 的隐藏功能设计遵循以下原则：

1. **内部优先**：这些功能主要面向开发者和高级用户，不通过常规帮助菜单展示。
2. **权限分离**：仅对具有副作用的 MCP 工具（Team、Cron）施加 `DangerFullAccess` 约束；纯输出型 slash command 在 REPL 层无额外限制，降低使用门槛。
3. **可测试性**：所有隐藏功能均有对应的单元测试覆盖，包括命令解析、报告渲染和进度追踪。
4. **结构化输出**：支持 Text 和 JSON 两种输出格式（通过 `CliOutputFormat`）。
5. **进度追踪**：复杂操作（如 Ultraplan）提供实时进度反馈，包括心跳事件和阶段切换。
