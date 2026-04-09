# 自定义 Agent - 从 Markdown 到运行时的完整链路

> 技术报告 · Unit 22 — custom-agents  
> 原文：https://ccb.agent-aura.top/docs/extensibility/custom-agents  
> 代码库：`rust/crates/*`

---

## 执行摘要

本报告追踪 claw-code 中自定义 Agent 的完整技术链路：从定义文件的发现与加载、到运行时 `AgentTool` 的执行调度、再到子 Agent 的工具过滤与隔离架构。与原文描述的 `.claude/agents/*.md` Markdown 格式不同，**claw-code 使用 TOML 格式的 Agent 定义文件**，存储于 `.claw/agents/`、`.codex/agents/`、`.claude/agents/` 等多级目录中。

---

## 1. Agent 定义的来源与发现机制

### 1.1 三级发现路径

Agent 定义文件的发现由 `discover_definition_roots()` 函数执行，按优先级扫描以下路径：

```rust
// rust/crates/commands/src/lib.rs:L2586-L2652
fn discover_definition_roots(cwd: &Path, leaf: &str) -> Vec<(DefinitionSource, PathBuf)> {
    let mut roots = Vec::new();

    // 1. 项目级 (从 cwd 向上遍历祖先目录)
    for ancestor in cwd.ancestors() {
        push_unique_root(&mut roots, DefinitionSource::ProjectClaw,
            ancestor.join(".claw").join(leaf));
        push_unique_root(&mut roots, DefinitionSource::ProjectCodex,
            ancestor.join(".codex").join(leaf));
        push_unique_root(&mut roots, DefinitionSource::ProjectClaude,
            ancestor.join(".claude").join(leaf));
    }

    // 2. 用户配置级 (环境变量)
    if let Ok(claw_config_home) = env::var("CLAW_CONFIG_HOME") {
        push_unique_root(&mut roots, DefinitionSource::UserClawConfigHome,
            PathBuf::from(claw_config_home).join(leaf));
    }

    // 3. 用户家目录级
    if let Some(home) = env::var_os("HOME") {
        let home = PathBuf::from(home);
        push_unique_root(&mut roots, DefinitionSource::UserClaw,
            home.join(".claw").join(leaf));
        push_unique_root(&mut roots, DefinitionSource::UserCodex,
            home.join(".codex").join(leaf));
        push_unique_root(&mut roots, DefinitionSource::UserClaude,
            home.join(".claude").join(leaf));
    }

    roots
}
```

**调用点**：`handle_agents_slash_command()` → `L2216` / `L2237`

### 1.2 定义文件格式：TOML 而非 Markdown

与原文描述的 Markdown + YAML frontmatter 格式不同，claw-code 使用 **TOML** 格式：

```toml
# ~/.claude/agents/reviewer.toml
name = "reviewer"
description = "Code review specialist, read-only analysis"
model = "haiku"
model_reasoning_effort = "high"
```

加载逻辑见 `load_agents_from_roots()`：

```rust
// rust/crates/commands/src/lib.rs:L3042-L3083
fn load_agents_from_roots(
    roots: &[(DefinitionSource, PathBuf)],
) -> std::io::Result<Vec<AgentSummary>> {
    let mut agents = Vec::new();
    let mut active_sources = BTreeMap::<String, DefinitionSource>::new();

    for (source, root) in roots {
        let mut root_agents = Vec::new();
        for entry in fs::read_dir(root)? {
            let entry = entry?;
            // 仅处理 .toml 文件
            if entry.path().extension().is_none_or(|ext| ext != "toml") {
                continue;
            }
            let contents = fs::read_to_string(entry.path())?;
            let fallback_name = entry.path().file_stem().map_or_else(
                || entry.file_name().to_string_lossy().to_string(),
                |stem| stem.to_string_lossy().to_string(),
            );
            root_agents.push(AgentSummary {
                name: parse_toml_string(&contents, "name").unwrap_or(fallback_name),
                description: parse_toml_string(&contents, "description"),
                model: parse_toml_string(&contents, "model"),
                reasoning_effort: parse_toml_string(&contents, "model_reasoning_effort"),
                source: *source,
                shadowed_by: None,
            });
        }
        root_agents.sort_by(|left, right| left.name.cmp(&right.name));

        // 按名称去重，后者覆盖前者 (shadowing)
        for mut agent in root_agents {
            let key = agent.name.to_ascii_lowercase();
            if let Some(existing) = active_sources.get(&key) {
                agent.shadowed_by = Some(*existing);
            } else {
                active_sources.insert(key, agent.source);
            }
            agents.push(agent);
        }
    }

    Ok(agents)
}
```

### 1.3 AgentSummary 数据结构

```rust
// rust/crates/commands/src/lib.rs:L2036-L2044
#[derive(Debug, Clone, PartialEq, Eq)]
struct AgentSummary {
    name: String,
    description: Option<String>,
    model: Option<String>,
    reasoning_effort: Option<String>,
    source: DefinitionSource,
    shadowed_by: Option<DefinitionSource>,  // 被哪个更高优先级的源覆盖
}
```

### 1.4 DefinitionSource 枚举

```rust
// rust/crates/commands/src/lib.rs:L1992-L2001
enum DefinitionSource {
    ProjectClaw,      // .claw/
    ProjectCodex,     // .codex/
    ProjectClaude,    // .claude/
    UserClawConfigHome,
    UserCodexHome,
    UserClaw,
    UserCodex,
    UserClaude,
}
```

---

## 2. AgentTool 执行链路

### 2.1 工具注册与调用入口

`Agent` 工具在 `execute_tool()` 中注册：

```rust
// rust/crates/tools/src/lib.rs:L1212
"Agent" => from_value::<AgentInput>(input).and_then(run_agent),
```

### 2.2 AgentInput 与 AgentOutput

```rust
// rust/crates/tools/src/lib.rs:L2116-L2123
#[derive(Debug, Deserialize)]
struct AgentInput {
    description: String,
    prompt: String,
    subagent_type: Option<String>,  // 如 "Explore", "Plan", "Verification"
    name: Option<String>,
    model: Option<String>,
}

// rust/crates/tools/src/lib.rs:L2391-L2400
#[derive(Debug, Clone, Serialize, Deserialize)]
struct AgentOutput {
    #[serde(rename = "agentId")]
    agent_id: String,
    name: String,
    description: String,
    #[serde(rename = "subagentType")]
    subagent_type: Option<String>,
    model: Option<String>,
    status: String,
    output_file: String,
    manifest_file: String,
    created_at: String,
    started_at: Option<String>,
    completed_at: Option<String>,
    lane_events: Vec<LaneEvent>,
    current_blocker: Option<String>,
    derived_state: String,
    error: Option<String>,
}
```

### 2.3 执行流程：execute_agent

```rust
// rust/crates/tools/src/lib.rs:L3290-L3336
fn execute_agent(input: AgentInput) -> Result<AgentOutput, String> {
    execute_agent_with_spawn(input, spawn_agent_job)
}

fn execute_agent_with_spawn<F>(input: AgentInput, spawn_fn: F) -> Result<AgentOutput, String>
where
    F: FnOnce(AgentJob) -> Result<(), String>,
{
    // 1. 参数校验
    if input.description.trim().is_empty() {
        return Err(String::from("description must not be empty"));
    }
    if input.prompt.trim().is_empty() {
        return Err(String::from("prompt must not be empty"));
    }

    // 2. 生成唯一 ID 与输出路径
    let agent_id = make_agent_id();  // L4251: "agent-{nanos}"
    let output_dir = agent_store_dir()?;  // L4240: .clawd-agents/
    std::fs::create_dir_all(&output_dir).map_err(|error| error.to_string())?;
    let output_file = output_dir.join(format!("{agent_id}.md"));
    let manifest_file = output_dir.join(format!("{agent_id}.json"));

    // 3. 规范化 subagent_type
    let normalized_subagent_type = normalize_subagent_type(input.subagent_type.as_deref());
    
    // 4. 解析模型
    let model = resolve_agent_model(input.model.as_deref());
    
    // 5. 生成名称
    let agent_name = input.name.as_deref()
        .map(slugify_agent_name)
        .filter(|name| !name.is_empty())
        .unwrap_or_else(|| slugify_agent_name(&input.description));

    // 6. 构建 system prompt
    let system_prompt = build_agent_system_prompt(&normalized_subagent_type)?;
    
    // 7. 根据 subagent_type 过滤可用工具
    let allowed_tools = allowed_tools_for_subagent(&normalized_subagent_type);

    // 8. 写入 .md 输出文件
    let output_contents = format!(
        "# Agent Task\n\n- id: {}\n- name: {}\n- description: {}\n- subagent_type: {}\n...",
        agent_id, agent_name, input.description, normalized_subagent_type, ...
    );
    std::fs::write(&output_file, output_contents).map_err(|error| error.to_string())?;

    // 9. 构建 manifest
    let manifest = AgentOutput { agent_id, name: agent_name, ... };
    write_agent_manifest(&manifest)?;

    // 10. 派生子任务
    let job = AgentJob {
        manifest: manifest.clone(),
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

### 2.4 子 Agent 类型与工具映射

```rust
// rust/crates/tools/src/lib.rs:L3451-L3530
fn allowed_tools_for_subagent(subagent_type: &str) -> BTreeSet<String> {
    let tools = match subagent_type {
        "Explore" => vec![
            "read_file", "glob_search", "grep_search", "WebFetch", "WebSearch",
            "ToolSearch", "Skill", "StructuredOutput",
        ],
        "Plan" => vec![
            "read_file", "glob_search", "grep_search", "WebFetch", "WebSearch",
            "ToolSearch", "Skill", "TodoWrite", "StructuredOutput", "SendUserMessage",
        ],
        "Verification" => vec![
            "bash", "read_file", "glob_search", "grep_search", "WebFetch", "WebSearch",
            "ToolSearch", "TodoWrite", "StructuredOutput", "SendUserMessage", "PowerShell",
        ],
        "claw-guide" => vec![...],
        "statusline-setup" => vec![...],
        _ => vec![
            "bash", "read_file", "write_file", "edit_file", "glob_search", "grep_search",
            "WebFetch", "WebSearch", "TodoWrite", "Skill", "ToolSearch", "NotebookEdit",
            "Sleep", "SendUserMessage", "Config", "StructuredOutput", "REPL", "PowerShell",
        ],
    };
    tools.into_iter().map(str::to_string).collect()
}
```

### 2.5 线程派生与运行时构建

```rust
// rust/crates/tools/src/lib.rs:L3370-L3395
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
                Ok(Err(error)) => persist_agent_terminal_state(&job.manifest, "failed", ...),
                Err(_) => persist_agent_terminal_state(&job.manifest, "failed", ..., "sub-agent thread panicked"),
            }
        })
        ...
}

// rust/crates/tools/src/lib.rs:L3397-L3404
fn run_agent_job(job: &AgentJob) -> Result<(), String> {
    let mut runtime = build_agent_runtime(job)?
        .with_max_iterations(DEFAULT_AGENT_MAX_ITERATIONS);  // 32
    let summary = runtime.run_turn(job.prompt.clone(), None)
        .map_err(|error| error.to_string())?;
    let final_text = final_assistant_text(&summary);
    persist_agent_terminal_state(&job.manifest, "completed", Some(final_text.as_str()), None)
}
```

### 2.6 运行时构建与 SubagentToolExecutor

```rust
// rust/crates/tools/src/lib.rs:L3406-L3426
fn build_agent_runtime(
    job: &AgentJob,
) -> Result<ConversationRuntime<ProviderRuntimeClient, SubagentToolExecutor>, String> {
    let model = job.manifest.model.clone()
        .unwrap_or_else(|| DEFAULT_AGENT_MODEL.to_string());  // "claude-opus-4-6"
    let allowed_tools = job.allowed_tools.clone();
    let api_client = ProviderRuntimeClient::new(model, allowed_tools.clone())?;
    let permission_policy = agent_permission_policy();
    let tool_executor = SubagentToolExecutor::new(allowed_tools)
        .with_enforcer(PermissionEnforcer::new(permission_policy.clone()));
    Ok(ConversationRuntime::new(
        Session::new(), api_client, tool_executor, permission_policy, job.system_prompt.clone()
    ))
}
```

### 2.7 SubagentToolExecutor 实现

```rust
// rust/crates/tools/src/lib.rs:L3951-L3980
struct SubagentToolExecutor {
    allowed_tools: BTreeSet<String>,
    enforcer: Option<PermissionEnforcer>,
}

impl SubagentToolExecutor {
    fn new(allowed_tools: BTreeSet<String>) -> Self {
        Self { allowed_tools, enforcer: None }
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

---

## 3. System Prompt 构建链路

### 3.1 build_agent_system_prompt

```rust
// rust/crates/tools/src/lib.rs:L3428-L3441
fn build_agent_system_prompt(subagent_type: &str) -> Result<Vec<String>, String> {
    let cwd = std::env::current_dir().map_err(|error| error.to_string())?;
    let mut prompt = load_system_prompt(
        cwd,
        DEFAULT_AGENT_SYSTEM_DATE.to_string(),  // "2026-03-31"
        std::env::consts::OS,
        "unknown",
    ).map_err(|error| error.to_string())?;
    
    // 追加 sub-agent 身份指令
    prompt.push(format!(
        "You are a background sub-agent of type `{subagent_type}`. Work only on the delegated task, \
         use only the tools available to you, do not ask the user questions, and finish with a \
         concise result."
    ));
    Ok(prompt)
}
```

### 3.2 load_system_prompt 与 ProjectContext

```rust
// rust/crates/runtime/src/prompt.rs:L432-L446
pub fn load_system_prompt(
    cwd: impl Into<PathBuf>,
    current_date: impl Into<String>,
    os_name: impl Into<String>,
    os_version: impl Into<String>,
) -> Result<Vec<String>, PromptBuildError> {
    let cwd = cwd.into();
    let project_context = ProjectContext::discover_with_git(&cwd, current_date.into())?;
    let config = ConfigLoader::default_for(&cwd).load()?;
    Ok(SystemPromptBuilder::new()
        .with_os(os_name, os_version)
        .with_project_context(project_context)
        .with_runtime_config(config)
        .build())
}
```

### 3.3 SystemPromptBuilder

```rust
// rust/crates/runtime/src/prompt.rs:L95-L167
#[derive(Debug, Clone, Default, PartialEq, Eq)]
pub struct SystemPromptBuilder {
    output_style_name: Option<String>,
    output_style_prompt: Option<String>,
    os_name: Option<String>,
    os_version: Option<String>,
    append_sections: Vec<String>,
    project_context: Option<ProjectContext>,
    config: Option<RuntimeConfig>,
}

impl SystemPromptBuilder {
    pub fn build(&self) -> Vec<String> {
        let mut sections = Vec::new();
        sections.push(get_simple_intro_section(self.output_style_name.is_some()));
        sections.push(get_simple_system_section());
        sections.push(get_simple_doing_tasks_section());
        sections.push(get_actions_section());
        sections.push(SYSTEM_PROMPT_DYNAMIC_BOUNDARY.to_string());
        sections.push(self.environment_section());
        if let Some(project_context) = &self.project_context {
            sections.push(render_project_context(project_context));
            if !project_context.instruction_files.is_empty() {
                sections.push(render_instruction_files(&project_context.instruction_files));
            }
        }
        if let Some(config) = &self.config {
            sections.push(render_config_section(config));
        }
        sections.extend(self.append_sections.iter().cloned());
        sections
    }
}
```

---

## 4. 关键差异：claw-code vs 原文

| 特性 | 原文描述 (ccb-docs) | claw-code 实现 |
|------|---------------------|----------------|
| 定义格式 | Markdown + YAML frontmatter | TOML |
| 定义路径 | `.claude/agents/*.md` | `.claw/agents/*.toml`、`.codex/agents/*.toml`、`.claude/agents/*.toml` |
| 字段名 | `tools`, `disallowedTools`, `maxTurns`, `memory` | `name`, `description`, `model`, `model_reasoning_effort` |
| 子 Agent 类型 | `general-purpose`, `Explore`, `Plan`, `Verification`, `claw-code-guide`, `statusline-setup` | 同左 |
| 工具过滤 | `tools` / `disallowedTools` 白黑名单 | `allowed_tools_for_subagent()` 硬编码映射 |
| Memory | `memory: local/project/user` | 未发现相关实现 |
| isolation | `worktree` / `remote` | 未发现相关实现 |
| hooks | `hooks: PreToolUse: [...]` | 全局 Hook 机制已实现，但未在 Agent 定义中配置 |
| skills | `skills: "code-review,security-review"` | 有 `/skills` 命令，但未在 Agent 定义中引用 |

---

## 5. Explore 代理分析

### 5.1 Explore 工具集

根据 `allowed_tools_for_subagent("Explore")`：

| 工具 | 用途 |
|------|------|
| `read_file` | 读取文件内容 |
| `glob_search` | 文件路径匹配搜索 |
| `grep_search` | 代码内容正则搜索 |
| `WebFetch` | 抓取 URL 内容 |
| `WebSearch` | 网络搜索 |
| `ToolSearch` | 工具 registry 搜索 |
| `Skill` | 调用预定义技能 |
| `StructuredOutput` | 结构化输出 |

**特征**：只读工具集，无 `bash` / `write_file` / `edit_file` 等写操作权限。

### 5.2 内置 Agent 对照

claw-code 未发现类似 `exploreAgent.ts` 的内置 Agent TypeScript 实现，所有 Agent 行为均由 Rust 代码中的 `allowed_tools_for_subagent()` 控制。

---

## 6. CLI 命令支持

```bash
# 列出已配置 Agent
claw agents
claw agents list
claw agents help

# 斜杠命令
/agents
/agents list
```

**输出格式支持**：`--format text` (默认) / `--format json`

---

## 7. 源码索引

| 组件 | 文件 | 行号 |
|------|------|------|
| `discover_definition_roots()` | `rust/crates/commands/src/lib.rs` | L2586-L2652 |
| `load_agents_from_roots()` | `rust/crates/commands/src/lib.rs` | L3042-L3083 |
| `AgentSummary` | `rust/crates/commands/src/lib.rs` | L2036-L2044 |
| `DefinitionSource` | `rust/crates/commands/src/lib.rs` | L1992-L2001 |
| `render_agents_report()` | `rust/crates/commands/src/lib.rs` | L3231-L3271 |
| `execute_agent()` | `rust/crates/tools/src/lib.rs` | L3290-L3336 |
| `spawn_agent_job()` | `rust/crates/tools/src/lib.rs` | L3370-L3395 |
| `allowed_tools_for_subagent()` | `rust/crates/tools/src/lib.rs` | L3451-L3530 |
| `SubagentToolExecutor` | `rust/crates/tools/src/lib.rs` | L3951-L3980 |
| `build_agent_system_prompt()` | `rust/crates/tools/src/lib.rs` | L3428-L3441 |
| `load_system_prompt()` | `rust/crates/runtime/src/prompt.rs` | L432-L446 |
| `SystemPromptBuilder` | `rust/crates/runtime/src/prompt.rs` | L95-L167 |
| `agent_store_dir()` | `rust/crates/tools/src/lib.rs` | L4240-L4249 |
| `make_agent_id()` | `rust/crates/tools/src/lib.rs` | L4251-L4258 |
| `normalize_subagent_type()` | `rust/crates/tools/src/lib.rs` | L4276-L4294 |

---

## 8. 结论

claw-code 的自定义 Agent 系统采用 **TOML 定义 + Rust 运行时** 架构，相比原文描述的 Markdown 格式更为简化：

1. **定义发现**：支持三级路径发现与 shadowing 机制
2. **执行模型**：通过 `AgentTool` 派生独立线程，使用 `SubagentToolExecutor` 进行工具白名单过滤
3. **System Prompt**：动态组装项目上下文、配置与 sub-agent 身份指令
4. **差异点**：未实现原文所述的 `memory`、`isolation`、`hooks` 等高级特性

---

*Generated by Unit 22 — custom-agents*
