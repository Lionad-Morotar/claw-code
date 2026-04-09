# 协调者与蜂群模式 - 多 Agent 高级编排

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/agent/coordinator-and-swarm) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 两种协作模式的架构差异

文档将多 Agent 协作抽象为两种模式：

| 维度 | Coordinator Mode | Agent Swarms |
|------|------------------|--------------|
| 门控 | `feature('COORDINATOR_MODE')` + `CLAUDE_CODE_COORDINATOR_MODE=1` | 任务系统 V2（默认启用） |
| 拓扑 | 星型：Coordinator 居中，Worker 外围 | 网状：对等 Agent 共享任务列表 |
| 角色 | 明确分工：Coordinator 编排、Worker 执行 | 模糊：每个 Agent 自主认领任务 |
| 通信 | SendMessage 定向通信 + `<task-notification>` | 任务文件系统 + 邮箱广播 |
| 适用 | 需要集中决策的复杂任务 | 并行度高的独立子任务 |

在 `claw-code` 的 Rust 实现中，这两种能力分别以不同的源码层面落地：Coordinator 的核心逻辑目前更多存在于 TypeScript 上游的 `coordinatorMode.ts` 中，而 **Swarm 所需的任务系统、Team 管理、Worker 生命周期** 已经在 Rust 侧有了完整的运行时支撑。

---

## Coordinator 的 Rust 侧基础设施：Agent Tool

虽然 `claw-code` 目前还没有完整复刻上游的 Coordinator Mode，但它已经具备 Coordinator 派发 Worker 的核心能力——`Agent` 工具。该工具定义在 [`rust/crates/tools/src/lib.rs#L572-L583`](/rust/crates/tools/src/lib.rs#L572-L583)：

```rust
ToolSpec {
    name: "Agent",
    description: "Launch a specialized agent task and persist its handoff metadata.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "description": { "type": "string" },
            "prompt": { "type": "string" },
            "subagent_type": { "type": "string" },
            "name": { "type": "string" },
            "model": { "type": "string" }
        },
        "required": ["description", "prompt"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::DangerFullAccess,
}
```

`Agent` 被标记为 `DangerFullAccess`，这意味着只有高权限模式下才能启动子 Agent——这与 Coordinator 作为编排者的安全模型一致。

### 子 Agent 的启动链路

`Agent` 的执行入口在 [`rust/crates/tools/src/lib.rs#L1212`](/rust/crates/tools/src/lib.rs#L1212) 处分发到 `run_agent`，最终进入 `execute_agent_with_spawn`（[`tools/src/lib.rs#L3290-L3336`](/rust/crates/tools/src/lib.rs#L3290-L3336)）。它的启动流程如下：

1. **生成 agent_id** 并创建持久化目录 `.claw/agents/`
2. **解析 subagent_type**，决定可用工具集
3. **构建 system prompt**（基于当前项目上下文 + subagent 类型声明）
4. **写入输出文件**（`.md`）和 manifest（`.json`）
5. **在新线程中启动** `ConversationRuntime`，执行 `run_turn`

```rust
fn execute_agent_with_spawn<F>(input: AgentInput, spawn_fn: F) -> Result<AgentOutput, String>
where
    F: FnOnce(AgentJob) -> Result<(), String>,
{
    // ... 生成 agent_id、准备输出目录、构建 manifest ...
    let job = AgentJob {
        manifest: manifest_for_spawn,
        prompt: input.prompt,
        system_prompt,
        allowed_tools,
    };
    spawn_fn(job)?;
    Ok(manifest)
}
```

子 Agent 的线程名固定为 `clawd-agent-{agent_id}`，便于在系统层面追踪（[`tools/src/lib.rs#L3370-L3374`](/rust/crates/tools/src/lib.rs#L3370-L3374)）。

### 子 Agent 的工具权限隔离

`allowed_tools_for_subagent` 函数（[`tools/src/lib.rs#L3411-L3455`](/rust/crates/tools/src/lib.rs#L3411-L3455)）根据 `subagent_type` 严格限制子 Agent 的工具集：

| subagent_type | 可用工具 |
|---------------|----------|
| `Explore` | `read_file`, `glob_search`, `grep_search`, `WebFetch`, `WebSearch`, `ToolSearch`, `Skill`, `StructuredOutput` |
| `Plan` | 以上 + `TodoWrite`, `SendUserMessage` |
| `Verification` | 以上 + `bash`, `PowerShell` |
| 默认 | 包含 `bash`, `write_file`, `edit_file` 等完整工具集 |

这种限制对应了文档中 "Worker 不能嵌套创建团队或发送消息" 的设计精神——Rust 实现通过 `BTreeSet<String>` 显式工具白名单来实现。

---

## Swarm 蜂群模式：任务与团队的 Rust 实现

Agent Swarm 的核心是**共享任务列表 + 竞争认领**。`claw-code` 在 Rust 侧为此构建了三个内存级注册表：

| 注册表 | 文件 | 用途 |
|--------|------|------|
| `TaskRegistry` | `runtime/src/task_registry.rs` | 任务创建、状态流转、消息追加、团队绑定 |
| `TeamRegistry` | `runtime/src/team_cron_registry.rs` | 团队创建、软删除、任务列表关联 |
| `WorkerRegistry` | `runtime/src/worker_boot.rs` | Worker 生命周期、信任门控、prompt 投递 |

### TaskRegistry：任务的完整状态机

`TaskRegistry`（[`runtime/src/task_registry.rs#L55-L231`](/rust/crates/runtime/src/task_registry.rs#L55-L231)）使用 `Arc<Mutex<RegistryInner>>` 实现了线程安全的内存级任务管理。任务状态枚举为（[`L13-L20`](/rust/crates/runtime/src/task_registry.rs#L13-L20)）：

```rust
pub enum TaskStatus {
    Created,
    Running,
    Completed,
    Failed,
    Stopped,
}
```

`Task` 结构体（[`L35-L46`](/rust/crates/runtime/src/task_registry.rs#L35-L46)）包含：

- `task_id`：格式为 `task_{timestamp}_{counter}`
- `prompt` / `description`：任务的原始指令
- `task_packet`：可选的结构化任务包（见下文）
- `status`：当前状态
- `messages`：运行时追加的消息历史
- `output`：累积输出内容
- `team_id`：所属团队 ID

关键 API：

- `create()` / `create_from_packet()`：创建任务
- `set_status()`：状态流转
- `update()`：追加消息
- `append_output()` / `output()`：读写输出
- `assign_team()`：绑定到团队
- `stop()`：终止运行中的任务（拒绝已完成/已失败的任务）

这与文档中描述的 `claimTask() → TaskUpdate()` 竞争机制形成了数据层基础。虽然当前 Rust 实现还没有完整的文件锁 + 高水位标记的并发认领协议，但 `TaskRegistry` 提供的原子 `Mutex` 操作已经可以在同进程内实现基础的任务竞争。

### TeamRegistry：团队的创建与软删除

`TeamRegistry`（[`runtime/src/team_cron_registry.rs#L50-L120`](/rust/crates/runtime/src/team_cron_registry.rs#L50-L120)）管理 `Team` 结构体：

```rust
pub struct Team {
    pub team_id: String,
    pub name: String,
    pub task_ids: Vec<String>,
    pub status: TeamStatus,
    pub created_at: u64,
    pub updated_at: u64,
}
```

`TeamCreate` 工具的实现（[`tools/src/lib.rs#L1521-L1540`](/rust/crates/tools/src/lib.rs#L1521-L1540)）展现了团队-任务绑定的完整链路：

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
    // ...
}
```

这精确对应了文档中的"团队初始化"流程：Leader 创建团队 → 所有 teammate 自动获得相同的 task list 上下文。

### TaskPacket：结构化任务的契约

`RunTaskPacket` 工具支持从结构化数据创建任务，对应的 `TaskPacket` 定义在 [`runtime/src/task_packet.rs#L4-L14`](/rust/crates/runtime/src/task_packet.rs#L4-L14)：

```rust
pub struct TaskPacket {
    pub objective: String,
    pub scope: String,
    pub repo: String,
    pub branch_policy: String,
    pub acceptance_tests: Vec<String>,
    pub commit_policy: String,
    pub reporting_contract: String,
    pub escalation_policy: String,
}
```

`validate_packet()` 函数（[`L56-L84`](/rust/crates/runtime/src/task_packet.rs#L56-L84)）对必填字段和 `acceptance_tests` 的空值进行校验。这套结构化契约非常适合 Swarm 中的任务分发——每个 Worker 拿到的是一个边界清晰、验收标准明确的任务包，而不是模糊的文本 prompt。

---

## Worker Boot：子 Agent 的终端生命周期管理

`WorkerRegistry`（[`runtime/src/worker_boot.rs#L125-L535`](/rust/crates/runtime/src/worker_boot.rs#L125-L535)）为子 Agent 提供了更细粒度的终端控制状态机。它解决的核心问题是：**如何可靠地把 prompt 投递到一个外部终端进程，并观测它的执行状态。**

### Worker 状态机

```rust
pub enum WorkerStatus {
    Spawning,        // 刚创建
    TrustRequired,   // 等待用户信任确认
    ReadyForPrompt,  // 可以投递 prompt
    Running,         // 正在执行
    Finished,        // 正常结束
    Failed,          // 失败
}
```

### 信任门控（Trust Gate）

Worker 启动后可能遇到 "Do you trust the files in this folder?" 之类的交互提示。`observe()` 方法（[`L170-L230`](/rust/crates/runtime/src/worker_boot.rs#L170-L230)）会通过屏幕文本检测信任提示：

```rust
if !worker.trust_gate_cleared && detect_trust_prompt(&lowered) {
    worker.status = WorkerStatus::TrustRequired;
    // ... 记录 TrustRequired 事件
    if worker.trust_auto_resolve {
        worker.trust_gate_cleared = true;
        // 自动放行（allowlist 匹配时）
    }
}
```

如果 `cwd` 在 `trusted_roots` allowlist 中，`trust_auto_resolve` 为 true，会自动跳过信任门控——这与 Coordinator 自动化派发 Worker 的场景高度相关。

### Prompt 误投递检测与恢复

这是 Worker Boot 最精密的机制。`detect_prompt_misdelivery()`（[`runtime/src/worker_boot.rs#L640-L700`](/rust/crates/runtime/src/worker_boot.rs#L640-L700) 附近）检测以下两类错误：

1. **Shell 误投递**：prompt 被当成 shell 命令执行（出现 `command not found`）
2. **错误目标投递**：prompt 出现在非预期的终端窗口（通过 CWD 路径匹配判断）

检测到误投递后，若 `auto_recover_prompt_misdelivery` 为 true，Worker 会：

- 将 `replay_prompt` 设置为上一次 prompt
- 状态回退到 `ReadyForPrompt`
- 在下一次 `send_prompt(None)` 时自动重发

这对应文档中 Worker 异常退出后的续传能力。

### 文件级可观测性

`emit_state_file()`（[`runtime/src/worker_boot.rs#L560-L600`](/rust/crates/runtime/src/worker_boot.rs#L560-L600)）将 Worker 状态写入 `.claw/worker-state.json`：

```rust
struct StateSnapshot<'a> {
    worker_id: &'a str,
    status: WorkerStatus,
    is_ready: bool,
    trust_gate_cleared: bool,
    prompt_in_flight: bool,
    last_event: Option<&'a WorkerEvent>,
    updated_at: u64,
    seconds_since_update: u64,
}
```

外部编排器（如 Coordinator 或 clawhip）可以轮询这个文件来判断 Worker 是否就绪、是否阻塞、是否需要干预。这是一种**无 HTTP 路由、纯文件系统**的进程间通信设计，非常符合 Terminal-native 的架构哲学。

---

## 工具层面的多 Agent 接口

在 [`rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) 中，与多 Agent 协作直接相关的工具包括：

| 工具 | 行号 | 功能 |
|------|------|------|
| `Agent` | L572 | 启动子 Agent |
| `TaskCreate` | L747 | 从 prompt 创建后台任务 |
| `RunTaskPacket` | L761 | 从结构化包创建后台任务 |
| `TaskGet` | L793 | 查询任务详情 |
| `TaskList` | L806 | 列出所有任务 |
| `TaskStop` | L816 | 停止任务 |
| `TaskUpdate` | L829 | 向任务发送消息/更新 |
| `TeamCreate` | L982 | 创建团队并绑定任务 |
| `TeamDelete` | L1006 | 软删除团队 |

`TaskStop` 的权限级别也是 `DangerFullAccess`（[`tools/src/lib.rs#L816-L828`](/rust/crates/tools/src/lib.rs#L816-L828)），与 `Agent` 一致。这反映了"控制其他 Agent 生命周期"属于高敏感操作的设计原则。

---

## Coordinator vs Swarm 在 Rust 源码中的映射

文档给出了一张选择对照表。在 `claw-code` 当前实现中：

- **Coordinator 模式**：主要通过 `Agent` 工具 + `WorkerRegistry` 实现。Coordinator（主会话）理解任务后，调用 `Agent` 启动子 Agent（Worker），子 Agent 在独立线程中运行，结果写回文件系统，Coordinator 再读取综合。
- **Swarm 模式**：主要通过 `TaskRegistry` + `TeamRegistry` + `TaskPacket` 实现。多个子 Agent 共享同一个 `team_id` 的任务列表，通过 `TaskList`/`TaskUpdate` 进行竞争和协同。

`WorkerRegistry` 和 `TaskRegistry` 是两个可以组合使用的原语：

- 如果你需要**终端级精细控制**（信任门控、prompt 投递、状态观测），用 `WorkerRegistry`
- 如果你需要**任务级结构化编排**（状态流转、团队绑定、消息历史），用 `TaskRegistry`
- 如果你需要**两者兼得**，可以在 `Agent` 启动时将其同时注册为 `Task` 和 `Worker`

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `Agent`、`TaskCreate`、`RunTaskPacket`、`TaskGet`、`TaskList`、`TaskStop`、`TaskUpdate`、`TeamCreate`、`TeamDelete` 工具定义与执行实现 |
| [`/rust/crates/runtime/src/task_registry.rs`](/rust/crates/runtime/src/task_registry.rs) | `TaskRegistry`、`Task`、`TaskStatus`、`TaskMessage` |
| [`/rust/crates/runtime/src/task_packet.rs`](/rust/crates/runtime/src/task_packet.rs) | `TaskPacket`、`TaskPacketValidationError`、`validate_packet()` |
| [`/rust/crates/runtime/src/team_cron_registry.rs`](/rust/crates/runtime/src/team_cron_registry.rs) | `TeamRegistry`、`Team`、`TeamStatus`、`CronRegistry` |
| [`/rust/crates/runtime/src/worker_boot.rs`](/rust/crates/runtime/src/worker_boot.rs) | `WorkerRegistry`、`Worker`、`WorkerStatus`、`WorkerEvent`、信任门控、prompt 误投递恢复、状态文件写入 |
| [`/rust/crates/runtime/src/lib.rs`](/rust/crates/runtime/src/lib.rs) | runtime crate 的公开模块导出，包含 `worker_boot`、`task_registry`、`task_packet`、`team_cron_registry` |
