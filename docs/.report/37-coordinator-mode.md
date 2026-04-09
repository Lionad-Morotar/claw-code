# Unit 37: Coordinator Mode 技术报告

> 原始页面：https://ccb.agent-aura.top/docs/features/coordinator-mode  
> 生成日期：2026-04-09

---

## 一、功能定位

COORDINATOR_MODE 将 CLI 变为"编排者"（Orchestrator）角色。编排者不直接操作文件系统，而是通过 `AgentTool` 向多个 worker subagent 派发任务，由 worker 并行执行。适用于大型任务拆分、并行调研、实现+验证分离等场景。

核心约束：
- 编排者只能使用 `AgentTool`、`SendMessage`、`TaskStop`（任务管理）
- Worker 拥有完整标准工具（`Bash`、`Read`、`Edit` 等）+ MCP 工具 + Skill 工具
- Worker 结果以 `<task-notification>` XML 形式回流到编排者

---

## 二、源码实现追踪

### 2.1 模式检测与会话恢复（Python 侧参考实现）

原始文档提到 `isCoordinatorMode()` 和 `matchSessionMode()` 负责模式检测与会话恢复，但当前 Rust 代码库中**不存在** `COORDINATOR_MODE` 或 `CLAUDE_CODE_COORDINATOR_MODE` 的显式 feature flag 检测。说明 coordinator mode 的启用逻辑在 Rust 重构过程中被以下机制隐式替代：

- 通过 `allowed_tools` 白名单机制控制编排者可用工具
- 通过 `SubagentToolExecutor` 的工具拦截能力限制编排者只能调用被授权的工具

### 2.2 Worker 创建与生命周期 — Rust 实现

**文件**：`rust/crates/tools/src/lib.rs`

#### 2.2.1 Agent 派发入口 `run_agent_tool` / `spawn_agent_job`

- `#L3306-L3360`：准备 agent manifest、输出文件，生成 agent_id
- `#L3364-L3388`：写入 `.md` 报告文件和 `.json` manifest，随后 `spawn_agent_job(job)` 启动子线程
- `#L3394-L3420`：`spawn_agent_job` 在独立线程 `clawd-agent-{agent_id}` 中运行 `run_agent_job`
- `#L3408-L3420`：`build_agent_runtime` 构造 `ConversationRuntime<ProviderRuntimeClient, SubagentToolExecutor>`

```rust
fn build_agent_runtime(
    job: &AgentJob,
) -> Result<ConversationRuntime<ProviderRuntimeClient, SubagentToolExecutor>, String> {
    let model = job.manifest.model.clone().unwrap_or_else(|| DEFAULT_AGENT_MODEL.to_string());
    let allowed_tools = job.allowed_tools.clone();
    let api_client = ProviderRuntimeClient::new(model, allowed_tools.clone())?;
    let permission_policy = agent_permission_policy();
    let tool_executor = SubagentToolExecutor::new(allowed_tools)
        .with_enforcer(PermissionEnforcer::new(permission_policy.clone()));
    Ok(ConversationRuntime::new(Session::new(), …))
}
```

#### 2.2.2 工具白名单 `allowed_tools_for_subagent`

- `#L3451-L3520`：按 `subagent_type` 返回不同的允许工具集合

| subagent_type | 关键允许工具 |
|---------------|--------------|
| `Explore` | `read_file`, `glob_search`, `grep_search`, `WebFetch`, `WebSearch`, `ToolSearch`, `Skill`, `StructuredOutput` |
| `Plan` | 同上 + `TodoWrite`, `SendUserMessage` |
| `Verification` | 同上 + `bash`, `PowerShell` |
| `statusline-setup` | `bash`, `read_file`, `write_file`, `edit_file`, `glob_search`, `grep_search`, `ToolSearch` |
| `general-purpose`（默认） | 全部标准工具 |

**重要**：当前 Rust 实现中**不存在**独立的 `"coordinator"` subagent type。编排者角色由上层 REPL/Runtime 通过直接限制 `allowed_tools` 实现，而不是通过 `allowed_tools_for_subagent` 函数中的某个类型。

#### 2.2.3 子代理类型规范化 `normalize_subagent_type`

- `#L4276-L4305`

```rust
fn normalize_subagent_type(subagent_type: Option<&str>) -> String {
    let trimmed = subagent_type.map(str::trim).unwrap_or_default();
    if trimmed.is_empty() { return String::from("general-purpose"); }
    match canonical_tool_token(trimmed).as_str() {
        "explore" | "explorer" | "exploreagent" => String::from("Explore"),
        "plan" | "planagent" => String::from("Plan"),
        "verification" | "verificationagent" | "verify" | "verifier" => String::from("Verification"),
        "clawguide" | "clawguideagent" | "guide" => String::from("claw-guide"),
        "statusline" | "statuslinesetup" => String::from("statusline-setup"),
        _ => trimmed.to_string(),
    }
}
```

#### 2.2.4 Subagent 系统提示 `build_agent_system_prompt`

- `#L3428-L3442`

加载默认系统提示后追加：

```rust
prompt.push(format!(
    "You are a background sub-agent of type `{subagent_type}`. Work only on the delegated task, use only the tools available to you, do not ask the user questions, and finish with a concise result."
));
```

### 2.3 工具执行拦截 — `SubagentToolExecutor`

- `#L3951-L3982`

```rust
struct SubagentToolExecutor {
    allowed_tools: BTreeSet<String>,
    enforcer: Option<PermissionEnforcer>,
}

impl ToolExecutor for SubagentToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError> {
        if !self.allowed_tools.contains(tool_name) {
            return Err(ToolError::new(format!(
                "tool `{tool_name}` is not enabled for this sub-agent"
            )));
        }
        // … JSON parse + execute_tool_with_enforcer
    }
}
```

这是 Coordinator Mode 的**底层安全网**：如果上层 REPL 将编排者的 `allowed_tools` 设置为仅包含 `AgentTool`、`TaskStop`、`SendMessage`（以及 `TaskUpdate`），则编排者调用任何其他工具都会被 `SubagentToolExecutor` 直接拒绝。

### 2.4 Task 管理工具

#### `TaskStop`

- `#L816-L825`：工具定义
- `#L1232`：路由入口
- `#L1406-L1417`：实现调用 `global_task_registry().stop(&input.task_id)`

#### `TaskUpdate`

- `#L829-L838`：工具定义（向运行中的 task 发送消息）
- `#L1233`：路由入口
- `#L1419-L1430`：实现调用 `global_task_registry().update(&task_id, &message)`

这两个工具对应原始文档中的 `SendMessage`/`TaskStop`，用于编排者对 worker 进行消息投递和强制终止。

### 2.5 Worker Boot 机制（任务级 worker 监督）

**文件**：`rust/crates/runtime/src/worker_boot.rs`

- `#L268-L276`：记录 worker prompt 投递失败或 shell 误投的日志
- 该文件主要用于外部 terminal/worker 的启动握手（`clawhip`、`orchestrator` 外部观察）

文件中的注释提到：

```rust
/// This is the file-based observability surface: external observers (clawhip, orchestrators)
```

说明 worker boot 设计时就考虑了编排者的可观测性需求。

### 2.6 Simple Mode 工具限制

**文件**：`src/tools.py` `#L63-L80`

```python
def get_tools(
    simple_mode: bool = False,
    include_mcp: bool = True,
    permission_context: ToolPermissionContext | None = None,
) -> tuple[PortingModule, ...]:
    tools = list(PORTED_TOOLS)
    if simple_mode:
        tools = [module for module in tools if module.name in {'BashTool', 'FileReadTool', 'FileEditTool'}]
```

`simple_mode` 在 Python CLI 层被 `CLAUDE_CODE_SIMPLE=1` 触发，此处 `BashTool` / `FileReadTool` / `FileEditTool` 的命名与 Rust 侧 `bash` / `read_file` / `write_file` / `edit_file` 对应。文档中提到的 Simple 模式 worker 仅有 `Bash`、`Read`、`Edit` 即来源于此。

### 2.7 测试覆盖

**文件**：`rust/crates/tools/src/lib.rs`

- `#L7286-L7303`：`agent_tool_subset_mapping_is_expected()` 测试
  - 验证 `general-purpose`、`Explore`、`Plan`、`Verification` 各 subagent type 的工具映射是否符合预期
  - 关键断言：`Explore` 无 `bash`；`Plan` 无 `Agent`；`Verification` 有 `bash` 和 `PowerShell`

- `#L6075-L6088`：`subagent_tool_executor_denies_blocked_tool_before_dispatch()` 测试
  - 验证 `SubagentToolExecutor` 会拒绝不在白名单中的工具调用

---

## 三、数据流

```
[User Message]
    ↓
[Coordinator REPL] (limited allowed_tools: AgentTool, TaskStop, TaskUpdate, …)
    ├──→ AgentTool({ subagent_type: "Explore"/"general-purpose", prompt: "..." })
    │            ↓
    │    [Subagent Thread: clawd-agent-{id}]
    │            ↓
    │    [SubagentToolExecutor] — enforces allowed_tools_for_subagent()
    │            ↓
    │    Worker runs Bash/Read/Edit/WebSearch/Skill/MCP…
    │            ↓
    │    Result written to {agent_id}.md + manifest {agent_id}.json
    │            ↓
    │    ←── task-notification returns to Coordinator
    │
    ├──→ TaskUpdate({ task_id: "agent-id", message: "..." })
    │    Continue existing worker
    │
    └──→ TaskStop({ task_id: "agent-id" })
         Terminate running worker
```

---

## 四、与原始 TypeScript 实现的差异

| 原始 TS 文档 | 当前 Rust 实现 |
|-------------|---------------|
| `src/coordinator/coordinatorMode.ts` 显式 `isCoordinatorMode()` | **不存在**独立 coordinator 模块，通过 `allowed_tools` 白名单隐式实现 |
| `src/coordinator/workerAgent.ts` stub | Worker 直接复用通用 `AgentTool`，`subagent_type` 取 `"general-purpose"`、`"Explore"` 等 |
| `FEATURE_COORDINATOR_MODE=1` + `CLAUDE_CODE_COORDINATOR_MODE=1` | Rust 侧无相关 feature flag 代码 |
| `getCoordinatorUserContext()` | 功能被 `allowed_tools_for_subagent()` + `SubagentToolExecutor` 替代 |
| `CLAUDE_CODE_SIMPLE=1` 限制为 `Bash`、`Read`、`Edit` | Python 侧 `get_tools(simple_mode=True)` 实现，`src/tools.py:63-70` |

---

## 五、关键文件索引

| 文件 | 行号范围 | 职责 |
|------|---------|------|
| `rust/crates/tools/src/lib.rs` | `#L3306-L3360` | Agent manifest 生成与任务派发 |
| `rust/crates/tools/src/lib.rs` | `#L3394-L3420` | `spawn_agent_job` / `run_agent_job` 子线程模型 |
| `rust/crates/tools/src/lib.rs` | `#L3408-L3420` | `build_agent_runtime` + `SubagentToolExecutor` 构造 |
| `rust/crates/tools/src/lib.rs` | `#L3428-L3442` | `build_agent_system_prompt` 子代理系统提示 |
| `rust/crates/tools/src/lib.rs` | `#L3451-L3520` | `allowed_tools_for_subagent` 工具白名单 |
| `rust/crates/tools/src/lib.rs` | `#L3951-L3982` | `SubagentToolExecutor` 工具执行拦截器 |
| `rust/crates/tools/src/lib.rs` | `#L4276-L4305` | `normalize_subagent_type` 子代理类型规范化 |
| `rust/crates/tools/src/lib.rs` | `#L816-L838` | `TaskStop`、`TaskUpdate` 工具定义 |
| `rust/crates/tools/src/lib.rs` | `#L1406-L1430` | `run_task_stop`、`run_task_update` 实现 |
| `rust/crates/runtime/src/worker_boot.rs` | `#L268-L576` | Worker 启动监督与可观测性表面 |
| `src/tools.py` | `#L63-L80` | Simple mode 工具过滤 |
| `src/reference_data/subsystems/coordinator.json` | 全部 | Coordinator 子系统归档元数据（指向已归档的 `coordinatorMode.ts`） |

---

## 六、结论

当前仓库中的 Coordinator Mode 并非以一个显式模块存在，而是被**拆分消解**到以下 Rust 基础设施中：

1. **`AgentTool` / `allowed_tools_for_subagent`**：负责 worker 能力模型
2. **`SubagentToolExecutor`**：负责运行时工具拦截
3. **`TaskStop` / `TaskUpdate`**：负责编排者对 worker 的生命周期控制
4. **Python 侧 `get_tools(simple_mode=True)`**：负责 Simple 模式工具集裁剪

若需完整复现文档中描述的 Coordinator Mode，需要在 REPL/Conversation 层显式注入一个"coordinator" 角色，将其 `allowed_tools` 限制为 `{AgentTool, TaskStop, TaskUpdate}`，并在系统提示中追加 coordinator 专用指令。当前 Rust 代码已提供全部底层能力，只差一层编排者角色封装。
