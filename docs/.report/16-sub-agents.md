# 子 Agent 机制 - AgentTool 的执行链路与隔离架构

> 从源码角度解析 claw-code 子 Agent 机制：AgentTool 的完整执行链路、线程级进程隔离、Worktree 会话隔离、工具池独立组装、以及结果回传的数据格式。

---

## 1. 概述

claw-code 的子 Agent 机制通过 `Agent` 工具实现，允许主 Agent 派生专用子任务给独立运行的子 Agent 处理。子 Agent 在独立线程中运行，拥有隔离的会话状态、受限的工具池和独立的输出存储。

**核心特性**：
- **线程级隔离**：每个子 Agent 在独立的 `std::thread` 中运行，与主进程并发执行
- **会话绑定 Worktree**：通过 `Session.workspace_root` 绑定到创建工作树的目录，防止并行实例竞争
- **工具池限制**：根据子 Agent 类型（Explore/Plan/Verification 等）提供不同的工具子集
- **状态持久化**：运行状态和结果通过 `.md` 输出文件和 `.json` 清单文件持久化

---

## 2. Agent 工具定义

### 2.1 工具规范（ToolSpec）

`Agent` 工具在 `rust/crates/tools/src/lib.rs` 中定义：

```rust
// L572-587: Agent ToolSpec 定义
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

### 2.2 输入结构（AgentInput）

```rust
// L2117-2123: AgentInput 结构
struct AgentInput {
    description: String,
    prompt: String,
    subagent_type: Option<String>,
    name: Option<String>,
    model: Option<String>,
}
```

### 2.3 输出结构（AgentOutput）

```rust
// L2392-2415: AgentOutput 结构
struct AgentOutput {
    agent_id: String,              // 唯一标识符，格式：agent-{nanos}
    name: String,                  // slugify 后的名称
    description: String,           // 任务描述
    subagent_type: Option<String>, // 子 Agent 类型
    model: Option<String>,         // 使用的模型
    status: String,                // running/completed/failed
    output_file: String,           // Markdown 输出文件路径
    manifest_file: String,         // JSON 清单文件路径
    created_at: String,            // 创建时间戳
    started_at: Option<String>,    // 开始时间戳
    completed_at: Option<String>,  // 完成时间戳
    lane_events: Vec<LaneEvent>,   // 事件时间线
    current_blocker: Option<LaneEventBlocker>,
    derived_state: String,         // 派生状态：working/finished_cleanable 等
    error: Option<String>,         // 错误信息
}
```

---

## 3. 执行链路

### 3.1 工具分发入口

`execute_tool` 函数在 [`lib.rs#L1212`](rust/crates/tools/src/lib.rs#L1212) 处分发 `Agent` 工具调用：

```rust
// L1212: Agent 工具分发
"Agent" => from_value::<AgentInput>(input).and_then(run_agent),
```

### 3.2 run_agent 执行流程

```rust
// L1996-1998: run_agent 入口
fn run_agent(input: AgentInput) -> Result<String, String> {
    to_pretty_json(execute_agent(input)?)
}
```

### 3.3 execute_agent 核心逻辑

```rust
// L3286-3367: execute_agent 实现
fn execute_agent(input: AgentInput) -> Result<AgentOutput, String> {
    execute_agent_with_spawn(input, spawn_agent_job)
}

fn execute_agent_with_spawn<F>(input: AgentInput, spawn_fn: F) -> Result<AgentOutput, String>
where
    F: FnOnce(AgentJob) -> Result<(), String>,
{
    // 1. 输入验证
    if input.description.trim().is_empty() {
        return Err(String::from("description must not be empty"));
    }
    if input.prompt.trim().is_empty() {
        return Err(String::from("prompt must not be empty"));
    }

    // 2. 生成 Agent ID 和文件路径
    let agent_id = make_agent_id();                                    // L4251-4257
    let output_dir = agent_store_dir()?;                               // L4240-4249
    let output_file = output_dir.join(format!("{agent_id}.md"));
    let manifest_file = output_dir.join(format!("{agent_id}.json"));

    // 3. 解析子 Agent 类型和模型
    let normalized_subagent_type = normalize_subagent_type(input.subagent_type.as_deref()); // L4276-4293
    let model = resolve_agent_model(input.model.as_deref());           // L3449-3456
    let agent_name = slugify_agent_name(&input.description);           // L4259-4274

    // 4. 构建系统提示和工具池
    let system_prompt = build_agent_system_prompt(&normalized_subagent_type)?; // L3436-3447
    let allowed_tools = allowed_tools_for_subagent(&normalized_subagent_type); // L3458-3530

    // 5. 写入初始输出文件
    std::fs::write(&output_file, output_contents)?;

    // 6. 创建并持久化 manifest
    let manifest = AgentOutput {
        agent_id,
        name: agent_name,
        description: input.description,
        subagent_type: normalized_subagent_type,
        model,
        status: "running".to_string(),
        output_file: output_file.to_string_lossy().to_string(),
        manifest_file: manifest_file.to_string_lossy().to_string(),
        created_at: iso8601_now(),
        started_at: None,
        completed_at: None,
        lane_events: vec![],
        current_blocker: None,
        derived_state: "working".to_string(),
        error: None,
    };
    write_agent_manifest(&manifest)?;

    // 7.  spawn 子线程执行
    let job = AgentJob {
        manifest,
        prompt: input.prompt,
        system_prompt,
        allowed_tools,
    };
    if let Err(error) = spawn_fn(job) {
        persist_agent_terminal_state(&manifest, "failed", None, Some(error))?;
        return Err(error);
    }

    Ok(manifest)
}
```

---

## 4. 线程级隔离

### 4.1 spawn_agent_job 实现

```rust
// L3370-3397: 子 Agent 线程创建
fn spawn_agent_job(job: AgentJob) -> Result<(), String> {
    let thread_name = format!("clawd-agent-{}", job.manifest.agent_id);
    std::thread::Builder::new()
        .name(thread_name)
        .spawn(move || {
            let result = std::panic::catch_unwind(
                std::panic::AssertUnwindSafe(|| run_agent_job(&job))
            );
            match result {
                Ok(Ok(())) => {}
                Ok(Err(error)) => {
                    let _ = persist_agent_terminal_state(&job.manifest, "failed", None, Some(error));
                }
                Err(_) => {
                    let _ = persist_agent_terminal_state(
                        &job.manifest, "failed", None,
                        Some(String::from("sub-agent thread panicked")),
                    );
                }
            }
        })
        .map(|_| ())
        .map_err(|error| error.to_string())
}
```

### 4.2 子 Agent 运行时

```rust
// L3399-3404: 子 Agent 执行
fn run_agent_job(job: &AgentJob) -> Result<(), String> {
    let mut runtime = build_agent_runtime(job)?
        .with_max_iterations(DEFAULT_AGENT_MAX_ITERATIONS);
    let summary = runtime
        .run_turn(job.prompt.clone(), None)
        .map_err(|error| error.to_string())?;
    let final_text = final_assistant_text(&summary);
    persist_agent_terminal_state(&job.manifest, "completed", Some(final_text.as_str()), None)
}
```

---

## 5. 工具池隔离

### 5.1 SubagentToolExecutor

```rust
// L3951-3980: SubagentToolExecutor 定义与实现
struct SubagentToolExecutor {
    allowed_tools: BTreeSet<String>,
    enforcer: Option<PermissionEnforcer>,
}

impl SubagentToolExecutor {
    fn new(allowed_tools: BTreeSet<String>) -> Self {
        Self {
            allowed_tools,
            enforcer: None,
        }
    }

    fn with_enforcer(mut self, enforcer: PermissionEnforcer) -> Self {
        self.enforcer = Some(enforcer);
        self
    }
}

impl ToolExecutor for SubagentToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError> {
        // 工具白名单检查
        if !self.allowed_tools.contains(tool_name) {
            return Err(ToolError::new(format!(
                "tool `{tool_name}` is not enabled for this sub-agent"
            )));
        }
        let value = serde_json::from_str(input)
            .map_err(|error| ToolError::new(format!("invalid tool input JSON: {error}")))?;
        execute_tool_with_enforcer(self.enforcer.as_ref(), tool_name, &value)
            .map_err(ToolError::new)
    }
}
```

### 5.2 按类型的工具子集

```rust
// L3451-3530: allowed_tools_for_subagent 实现
fn allowed_tools_for_subagent(subagent_type: &str) -> BTreeSet<String> {
    let tools = match subagent_type {
        // Explore 类型：只读 + 搜索工具
        "Explore" => vec![
            "read_file", "glob_search", "grep_search",
            "WebFetch", "WebSearch", "ToolSearch", "Skill", "StructuredOutput",
        ],
        // Plan 类型：+ TodoWrite, SendUserMessage
        "Plan" => vec![
            "read_file", "glob_search", "grep_search",
            "WebFetch", "WebSearch", "ToolSearch", "Skill",
            "TodoWrite", "StructuredOutput", "SendUserMessage",
        ],
        // Verification 类型：+ bash, PowerShell
        "Verification" => vec![
            "bash", "read_file", "glob_search", "grep_search",
            "WebFetch", "WebSearch", "ToolSearch",
            "TodoWrite", "StructuredOutput", "SendUserMessage", "PowerShell",
        ],
        // 通用类型：完整工具集
        _ => vec![
            "bash", "read_file", "write_file", "edit_file",
            "glob_search", "grep_search", "WebFetch", "WebSearch",
            "TodoWrite", "Skill", "ToolSearch", "NotebookEdit", "Sleep",
            "SendUserMessage", "Config", "StructuredOutput", "REPL", "PowerShell",
        ],
    };
    tools.into_iter().map(str::to_string()).collect()
}
```

### 5.3 工具规范过滤

```rust
// L3984-3989: 根据允许的工具过滤 ToolSpec
fn tool_specs_for_allowed_tools(allowed_tools: Option<&BTreeSet<String>>) -> Vec<ToolSpec> {
    mvp_tool_specs()
        .into_iter()
        .filter(|spec| allowed_tools.is_none_or(|allowed| allowed.contains(spec.name)))
        .collect()
}
```

---

## 6. Worktree 会话隔离

### 6.1 Session 结构绑定 Worktree

```rust
// rust/crates/runtime/src/session.rs L89-100: Session 结构
#[derive(Debug, Clone)]
pub struct Session {
    pub version: u32,
    pub session_id: String,
    pub created_at_ms: u64,
    pub updated_at_ms: u64,
    pub messages: Vec<ConversationMessage>,
    pub compaction: Option<SessionCompaction>,
    pub fork: Option<SessionFork>,
    pub workspace_root: Option<PathBuf>,  // L90: Worktree 绑定关键字段
    pub prompt_history: Vec<SessionPromptEntry>,
    persistence: Option<SessionPersistence>,
}
```

**设计动机**（`session.rs` L82–87）：

> `workspace_root` 将会话绑定到其创建时所在的 worktree。全局 session store 在所有 `opencode serve` 实例间共享，因此若没有显式的 workspace root，并行 lane 可能产生竞态，导致写入落在错误的 CWD 中。

### 6.2 Session::with_workspace_root

```rust
// rust/crates/runtime/src/session.rs L180-183
/// Bind this session to the workspace root it was created in.
///
/// This is the per-worktree counterpart to the global session store and
/// lets downstream tooling reject writes that drift to the wrong CWD when
/// multiple `opencode serve` instances share `~/.local/share/opencode`.
#[must_use]
pub fn with_workspace_root(mut self, workspace_root: impl Into<PathBuf>) -> Self {
    self.workspace_root = Some(workspace_root.into());
    self
}
```

### 6.3 SessionFork 继承 Worktree

```rust
// rust/crates/runtime/src/session.rs L251-268
pub fn fork(&self, branch_name: Option<String>) -> Self {
    let now = current_time_millis();
    Self {
        version: self.version,
        session_id: generate_session_id(),
        created_at_ms: now,
        updated_at_ms: now,
        messages: self.messages.clone(),
        compaction: self.compaction.clone(),
        fork: Some(SessionFork {
            parent_session_id: self.session_id.clone(),
            branch_name: normalize_optional_string(branch_name),
        }),
        workspace_root: self.workspace_root.clone(),  // L264: 继承 workspace_root
        prompt_history: self.prompt_history.clone(),
        persistence: None,
    }
}
```

### 6.4 SessionStore 按 Worktree 隔离

```rust
// rust/crates/runtime/src/session_control.rs L10-27
/// Per-worktree session store that namespaces on-disk session files by
/// workspace fingerprint so that parallel `opencode serve` instances never
/// collide.
///
/// Create via [`SessionStore::from_cwd`] (derives the store path from the
/// server's working directory) or [`SessionStore::from_data_dir`] (honours an
/// explicit `--data-dir` flag).  Both constructors produce a directory layout
/// of `<data_dir>/sessions/<workspace_hash>/` where `<workspace_hash>` is a
/// stable hex digest of the canonical workspace root.
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct SessionStore {
    sessions_root: PathBuf,      // e.g. /home/user/project/.claw/sessions/a1b2c3d4e5f60718/
    workspace_root: PathBuf,     // 绑定的 workspace 根目录
}
```

### 6.5 按 Workspace Fingerprint 隔离

```rust
// rust/crates/runtime/src/session_control.rs L49-64
pub fn from_data_dir(
    data_dir: impl AsRef<Path>,
    workspace_root: impl AsRef<Path>,
) -> Result<Self, SessionControlError> {
    let workspace_root = workspace_root.as_ref();
    let sessions_root = data_dir
        .as_ref()
        .join("sessions")
        .join(workspace_fingerprint(workspace_root));  // L57: Worktree 指纹
    fs::create_dir_all(&sessions_root)?;
    Ok(Self {
        sessions_root,
        workspace_root: workspace_root.to_path_buf(),
    })
}
```

### 6.6 测试验证：不同 Worktree 产生不同 Session 目录

```rust
// rust/crates/runtime/src/session.rs L1473-1495
#[test]
fn workspace_sessions_dir_differs_for_different_cwds() {
    let tmp_a = std::env::temp_dir().join("claw-session-dir-a");
    let tmp_b = std::env::temp_dir().join("claw-session-dir-b");
    fs::create_dir_all(&tmp_a).expect("create dir a");
    fs::create_dir_all(&tmp_b).expect("create dir b");

    let dir_a = workspace_sessions_dir(&tmp_a).expect("dir a");
    let dir_b = workspace_sessions_dir(&tmp_b).expect("dir b");
    assert_ne!(
        dir_a, dir_b,
        "different CWDs must produce different session dirs"
    );
}
```

---

## 7. API 通信链路

### 7.1 ProviderRuntimeClient 构建

```rust
// rust/crates/tools/src/lib.rs L3406-3428
fn build_agent_runtime(
    job: &AgentJob,
) -> Result<ConversationRuntime<ProviderRuntimeClient, SubagentToolExecutor>, String> {
    let model = job.manifest.model.clone().unwrap_or_else(|| DEFAULT_AGENT_MODEL.to_string());
    let allowed_tools = job.allowed_tools.clone();

    // 构建 API 客户端
    let api_client = ProviderRuntimeClient::new(model, allowed_tools.clone())?;

    // 构建权限策略和工具执行器
    let permission_policy = agent_permission_policy();
    let tool_executor = SubagentToolExecutor::new(allowed_tools)
        .with_enforcer(PermissionEnforcer::new(permission_policy.clone()));

    // 创建运行时
    Ok(ConversationRuntime::new(
        Session::new(),
        api_client,
        tool_executor,
        permission_policy,
        job.system_prompt.clone(),
    ))
}
```

### 7.2 ProviderClient 模型路由

```rust
// rust/crates/api/src/client.rs L17-49
impl ProviderClient {
    pub fn from_model(model: &str) -> Result<Self, ApiError> {
        Self::from_model_with_anthropic_auth(model, None)
    }

    pub fn from_model_with_anthropic_auth(
        model: &str,
        anthropic_auth: Option<AuthSource>,
    ) -> Result<Self, ApiError> {
        let resolved_model = providers::resolve_model_alias(model);
        match providers::detect_provider_kind(&resolved_model) {
            ProviderKind::Anthropic => Ok(Self::Anthropic(...)),
            ProviderKind::Xai => Ok(Self::Xai(OpenAiCompatClient::from_env(...)?)),
            ProviderKind::OpenAi => Ok(Self::OpenAi(OpenAiCompatClient::from_env(...)?)),
        }
    }
}
```

### 7.3 消息发送接口

```rust
// rust/crates/api/src/client.rs L80-98
pub async fn send_message(
    &self,
    request: &MessageRequest,
) -> Result<MessageResponse, ApiError> {
    match self {
        Self::Anthropic(client) => client.send_message(request).await,
        Self::Xai(client) | Self::OpenAi(client) => client.send_message(request).await,
    }
}
```

---

## 8. 系统提示构建

### 8.1 build_agent_system_prompt

```rust
// rust/crates/tools/src/lib.rs L3428-3441
fn build_agent_system_prompt(subagent_type: &str) -> Result<Vec<String>, String> {
    let cwd = std::env::current_dir().map_err(|error| error.to_string())?;
    let mut prompt = load_system_prompt(
        cwd,
        DEFAULT_AGENT_SYSTEM_DATE.to_string(),
        std::env::consts::OS,
        "unknown",
    )
    .map_err(|error| error.to_string())?;

    // 注入子 Agent 类型提示
    prompt.push(format!(
        "You are a background sub-agent of type `{subagent_type}`. Work only on the delegated task, use only the tools available to you, do not ask the user questions, and finish with a concise result."
    ));
    Ok(prompt)
}
```

---

## 9. 状态持久化

### 9.1 输出文件结构

```rust
// rust/crates/tools/src/lib.rs L3320-3334
let output_contents = format!(
    "# Agent Task

- id: {}
- name: {}
- description: {}
- subagent_type: {}
- created_at: {}

## Prompt

{}
",
    agent_id, agent_name, input.description, normalized_subagent_type, created_at, input.prompt
);
std::fs::write(&output_file, output_contents).map_err(|error| error.to_string())?;
```

### 9.2 Manifest JSON 结构

```rust
// rust/crates/tools/src/lib.rs L3539-3548
fn write_agent_manifest(manifest: &AgentOutput) -> Result<(), String> {
    let mut normalized = manifest.clone();
    normalized.lane_events = dedupe_superseded_commit_events(&normalized.lane_events);
    std::fs::write(
        &normalized.manifest_file,
        serde_json::to_string_pretty(&normalized).map_err(|error| error.to_string())?,
    )
    .map_err(|error| error.to_string())
}
```

### 9.3 终端状态持久化

```rust
// rust/crates/tools/src/lib.rs L3555-3604
fn persist_agent_terminal_state(
    manifest: &AgentOutput,
    status: &str,
    result: Option<&str>,
    error: Option<String>,
) -> Result<(), String> {
    let blocker = error.as_deref().map(classify_lane_blocker);

    // 追加输出文件
    append_agent_output(
        &manifest.output_file,
        &format_agent_terminal_output(status, result, blocker.as_ref(), error.as_deref()),
    )?;

    // 更新 manifest
    let mut next_manifest = manifest.clone();
    next_manifest.status = status.to_string();
    next_manifest.completed_at = Some(iso8601_now());
    next_manifest.current_blocker.clone_from(&blocker);
    next_manifest.derived_state =
        derive_agent_state(status, result, error.as_deref(), blocker.as_ref()).to_string();
    next_manifest.error = error;

    // 写入 LaneEvent
    if let Some(blocker) = blocker {
        next_manifest.lane_events.push(LaneEvent::blocked(iso8601_now(), &blocker));
        next_manifest.lane_events.push(LaneEvent::failed(iso8601_now(), &blocker));
    } else {
        next_manifest.current_blocker = None;
        let compressed_detail = result
            .filter(|value| !value.trim().is_empty())
            .map(|value| compress_summary_text(value.trim()));
        next_manifest.lane_events.push(LaneEvent::finished(iso8601_now(), compressed_detail));
        if let Some(provenance) = maybe_commit_provenance(result) {
            next_manifest.lane_events.push(LaneEvent::commit_created(
                iso8601_now(),
                Some(format!("commit {}", provenance.commit)),
                provenance,
            ));
        }
    }
    write_agent_manifest(&next_manifest)
}
```

### 9.4 Agent 存储目录

```rust
// rust/crates/tools/src/lib.rs L4240-4249
fn agent_store_dir() -> Result<std::path::PathBuf, String> {
    if let Ok(path) = std::env::var("CLAWD_AGENT_STORE") {
        return Ok(std::path::PathBuf::from(path));
    }
    let cwd = std::env::current_dir().map_err(|error| error.to_string())?;
    if let Some(workspace_root) = cwd.ancestors().nth(2) {
        return Ok(workspace_root.join(".clawd-agents"));
    }
    Ok(cwd.join(".clawd-agents"))
}
```

---

## 10. 子 Agent 类型与工具映射表

| 子 Agent 类型 | 工具集 | 典型用途 |
|-------------|--------|----------|
| `Explore` | `read_file`, `glob_search`, `grep_search`, `WebFetch`, `WebSearch`, `ToolSearch`, `Skill`, `StructuredOutput` | 代码探索、文档调研 |
| `Plan` | Explore 工具集 + `TodoWrite`, `StructuredOutput`, `SendUserMessage` | 任务规划、进度跟踪 |
| `Verification` | `bash`, `read_file`, `glob_search`, `grep_search`, `WebFetch`, `WebSearch`, `ToolSearch`, `TodoWrite`, `StructuredOutput`, `SendUserMessage`, `PowerShell` | 构建验证、测试执行 |
| `claw-guide` | 同 Explore + `SendUserMessage` | claw-code 指南生成 |
| `statusline-setup` | `bash`, `read_file`, `write_file`, `edit_file`, `glob_search`, `grep_search`, `ToolSearch` | 状态行配置 |
| `general-purpose` | 完整工具集 | 通用任务 |

---

## 11. 关键源码锚点汇总

| 组件 | 文件 | 行号范围 | 说明 |
|------|------|----------|------|
| Agent ToolSpec | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L572-L587](rust/crates/tools/src/lib.rs#L572-L587) | 工具定义与输入 Schema |
| AgentInput | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L2117-L2123](rust/crates/tools/src/lib.rs#L2117-L2123) | 输入结构 |
| AgentOutput | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L2392-L2415](rust/crates/tools/src/lib.rs#L2392-L2415) | 输出结构 |
| AgentJob | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L2422](rust/crates/tools/src/lib.rs#L2422) | 内部任务结构 |
| run_agent | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L1996-L1998](rust/crates/tools/src/lib.rs#L1996-L1998) | 工具执行入口 |
| execute_agent | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L3286-L3367](rust/crates/tools/src/lib.rs#L3286-L3367) | 核心执行逻辑 |
| spawn_agent_job | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L3370-L3397](rust/crates/tools/src/lib.rs#L3370-L3397) | 线程创建 |
| run_agent_job | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L3399-L3404](rust/crates/tools/src/lib.rs#L3399-L3404) | 子 Agent 运行 |
| SubagentToolExecutor | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L3951-L3980](rust/crates/tools/src/lib.rs#L3951-L3980) | 工具执行器 |
| allowed_tools_for_subagent | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L3458-L3530](rust/crates/tools/src/lib.rs#L3458-L3530) | 工具子集映射 |
| build_agent_runtime | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L3403-L3428](rust/crates/tools/src/lib.rs#L3403-L3428) | 运行时构建 |
| build_agent_system_prompt | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L3436-L3447](rust/crates/tools/src/lib.rs#L3436-L3447) | 系统提示构建 |
| Session.workspace_root | [`session.rs`](rust/crates/runtime/src/session.rs) | [L90](rust/crates/runtime/src/session.rs#L90) | Worktree 绑定字段 |
| Session::with_workspace_root | [`session.rs`](rust/crates/runtime/src/session.rs) | [L176-L183](rust/crates/runtime/src/session.rs#L176-L183) | Worktree 绑定方法 |
| Session::fork | [`session.rs`](rust/crates/runtime/src/session.rs) | [L251-L268](rust/crates/runtime/src/session.rs#L251-L268) | 会话 Fork 继承 Worktree |
| SessionStore | [`session_control.rs`](rust/crates/runtime/src/session_control.rs) | [L20-L27](rust/crates/runtime/src/session_control.rs#L20-L27) | 会话存储结构 |
| SessionStore::from_data_dir | [`session_control.rs`](rust/crates/runtime/src/session_control.rs) | [L49-L64](rust/crates/runtime/src/session_control.rs#L49-L64) | 按 Worktree 隔离 |
| ProviderClient::from_model | [`client.rs`](rust/crates/api/src/client.rs) | [L17-L49](rust/crates/api/src/client.rs#L17-L49) | 模型路由 |
| persist_agent_terminal_state | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L3555-L3604](rust/crates/tools/src/lib.rs#L3555-L3604) | 状态持久化 |
| agent_store_dir | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L4240-L4249](rust/crates/tools/src/lib.rs#L4240-L4249) | 存储目录解析 |

---

## 12. 设计要点总结

1. **线程级并发**：子 Agent 在独立线程中运行，主进程不阻塞等待，支持多子 Agent 并行
2. **工具白名单**：`SubagentToolExecutor` 在工具调用前进行白名单检查，防止子 Agent 越权
3. **Worktree 绑定**：`Session.workspace_root` 和 `SessionStore` 的指纹隔离确保并行实例不竞争
4. **状态可追溯**：通过 `LaneEvent` 时间线和 `manifest.json` 提供完整的执行审计轨迹
5. **类型化提示**：根据子 Agent 类型注入不同的系统提示和工具集，实现专业化分工
