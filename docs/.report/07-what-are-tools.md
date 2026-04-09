# 工具系统设计 - AI 如何从说到做

> 本报告基于 [Claude Code 中文文档 - 工具：AI 的双手](https://ccb.agent-aura.top/docs/tools/what-are-tools) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## AI 为什么需要工具

大语言模型本质上只能做一件事：根据输入文本，生成输出文本。它不能读文件、不能执行命令、不能搜索代码。要让 AI 真正"动手"，需要一个桥梁——这就是 **Tool（工具）**。工具是 AI 的双手。AI 说"我想读这个文件"，工具系统替它真正去读；AI 说"我想执行这条命令"，工具系统替它真正去跑。

### Rust 实现中的核心抽象

在 `claw-code` 中，"AI 的双手"被抽象为一个极简的 `ToolExecutor` trait，定义在 [`runtime/src/conversation.rs#L57-L59`](/rust/crates/runtime/src/conversation.rs#L57-L59)：

```rust
pub trait ToolExecutor {
    fn execute(&mut self, tool_name: &str, input: &str) -> Result<String, ToolError>;
}
```

这个接口只有两个参数：工具名字（`tool_name`）和输入 JSON（`input`），返回一个字符串结果。整个 agentic loop 不需要知道工具内部是怎么工作的——它只负责把模型请求的工具调用交给 `ToolExecutor`，再把结果塞回对话历史。

---

## Tool 类型：统一接口与权限声明

原文描述 TypeScript 实现中所有工具都实现一个包含 35+ 字段的 `Tool<Input, Output, Progress>` 结构化类型。在 `claw-code` 的 Rust 实现中，这种"统一接口"被拆成了两个层次：

1. **调用层**：`ToolExecutor` trait（上面已述）负责执行时解耦。
2. **元数据层**：`ToolSpec` 结构体负责向模型暴露工具的 JSON Schema 和权限要求。

### 核心四要素（Rust 映射）

`ToolSpec` 定义在 [`tools/src/lib.rs#L100-L107`](/rust/crates/tools/src/lib.rs#L100-L107) 附近：

```rust
pub struct ToolSpec {
    pub name: &'static str,
    pub description: &'static str,
    pub input_schema: Value,
    pub required_permission: PermissionMode,
}
```

| 要素 | Rust 字段 | 说明 |
| --- | --- | --- |
| **name** | `name` | 唯一标识（如 `bash`、`read_file`、`grep_search`） |
| **description** | `description` | 静态描述字符串 |
| **input schema** | `input_schema` | JSON Schema（`serde_json::Value`），定义参数类型 |
| **call / execute** | `execute_tool(name, input)` | 巨大的 match 分发器 |

与 TypeScript 版本不同，Rust 实现没有采用"每个工具一个对象"的 OO 风格，而是把所有内置工具集中在 [`tools/src/lib.rs#L1178-L1267`](/rust/crates/tools/src/lib.rs#L1178-L1267) 的 `execute_tool_with_enforcer` 函数里，通过模式匹配（`match name`）统一分发。这种设计避免了 trait object 的虚函数开销，也更适合 Rust 的所有权模型。

### 注册与发现

Rust 实现中没有 `aliases`、`searchHint`、`shouldDefer` 等单独的字段，但实现了等价的机制：

- **别名**：`GlobalToolRegistry::normalize_allowed_tools`（[`tools/src/lib.rs#L192-L244`](/rust/crates/tools/src/lib.rs#L192-L244)）内置了别名映射：
  ```rust
  for (alias, canonical) in [
      ("read", "read_file"),
      ("write", "write_file"),
      ("edit", "edit_file"),
      ("glob", "glob_search"),
      ("grep", "grep_search"),
  ]
  ```
- **延迟加载 / ToolSearch**：`deferred_tool_specs()` 提供可搜索但默认不注册的工具规格，通过 `run_tool_search` 动态发现（[`tools/src/lib.rs#L2000-L2002`](/rust/crates/tools/src/lib.rs#L2000-L2002)）。
- **运行时开关**：某些工具通过在 `execute_tool_with_enforcer` 中直接返回错误来实现平台限制（如 PowerShell 在 Unix 上可用但语义简化）。

### 安全与权限

Rust 中每个 `ToolSpec` 都绑定了一个 `required_permission: PermissionMode`，这与 TypeScript 中的 `isReadOnly()`、`isDestructive()` 等字段等价，但粒度更粗、更明确。`PermissionMode` 定义在 [`runtime/src/permissions.rs#L7-L15`](/rust/crates/runtime/src/permissions.rs#L7-L15)：

```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,
    Allow,
}
```

- `ReadOnly`：只读操作（`read_file`、`glob_search`、`grep_search`）。
- `WorkspaceWrite`：可修改工作区内文件（`write_file`、`edit_file`、`TodoWrite`）。
- `DangerFullAccess`：可执行任意命令（`bash`、`Agent`、`REPL`）。

在执行前，`ConversationRuntime` 会调用 `permission_policy.authorize_with_context(...)`，其逻辑见 [`runtime/src/permissions.rs#L173-L292`](/rust/crates/runtime/src/permissions.rs#L173-L292)。如果当前会话模式低于工具所需权限，会拒绝或提示用户确认。

### 输出与结果

TypeScript 版本有 `maxResultSizeChars` 和结果持久化机制。Rust 实现中：

- **Bash 输出截断**：`bash.rs` 中设置了 `MAX_OUTPUT_BYTES = 16_384`（[`runtime/src/bash.rs#L289`](/rust/crates/runtime/src/bash.rs#L289)），超长输出会被截断并附加标记。
- **WebFetch 截断**：`tools/src/lib.rs` 的 `run_remote_trigger` 对 HTTP 响应体超过 8192 字节时截断。
- **文件读取**：`file_ops.rs` 中设置了 `MAX_READ_SIZE = 10 * 1024 * 1024`（10 MB）和 `MAX_WRITE_SIZE = 10 * 1024 * 1024`（[`runtime/src/file_ops.rs#L12-L16`](/rust/crates/runtime/src/file_ops.rs#L12-L16)），超限直接返回错误而非持久化到磁盘。

Rust 实现目前**没有**单独的结果持久化文件（如 `toolResultStorage.ts`），而是通过直接截断或报错来控制输出预算。

---

## 工具注册：getTools() 的分层组装

原文描述 TypeScript 的 `getAllBaseTools()` 按固定工具、条件工具、Feature-flag 工具、Ant-only 工具分层组装。Rust 实现的分层在 [`tools/src/lib.rs#L133-L184`](/rust/crates/tools/src/lib.rs#L133-L184) 的 `GlobalToolRegistry` 构造函数和 [`tools/src/lib.rs#L125-L131`](/rust/crates/tools/src/lib.rs#L125-L131) 的 `mvp_tool_specs()` 中体现：

### 内置固定工具

`mvp_tool_specs()` 返回 40+ 个内置工具的 JSON Schema 和权限声明，包括：

- 文件操作：`read_file`、`write_file`、`edit_file`、`glob_search`、`grep_search`、`NotebookEdit`
- 命令执行：`bash`、`PowerShell`、`REPL`
- 任务与对话：`TodoWrite`、`Agent`、`Skill`、`SendUserMessage`、`AskUserQuestion`
- Web 能力：`WebFetch`、`WebSearch`
- 工作流：`EnterPlanMode`、`ExitPlanMode`、`StructuredOutput`、`Sleep`
- 后台任务：`TaskCreate`、`TaskGet`、`TaskList`、`TaskStop`、`TaskUpdate`、`TaskOutput`、`RunTaskPacket`
- Worker 系统：`WorkerCreate`、`WorkerGet`、`WorkerObserve`、`WorkerSendPrompt`、`WorkerTerminate` 等
- 团队与调度：`TeamCreate`、`TeamDelete`、`CronCreate`、`CronDelete`、`CronList`
- MCP 与 LSP：`MCP`、`ListMcpResources`、`ReadMcpResource`、`McpAuth`、`LSP`
- 远程触发：`RemoteTrigger`

### 插件与 MCP 扩展工具

`GlobalToolRegistry::with_plugin_tools` 和 `with_runtime_tools` 支持加载外部插件工具和运行时工具：

```rust
pub fn with_plugin_tools(plugin_tools: Vec<PluginTool>) -> Result<Self, String> { ... }
pub fn with_runtime_tools(
    mut self,
    runtime_tools: Vec<RuntimeToolDefinition>,
) -> Result<Self, String> { ... }
```

这两个方法都会检查名称冲突：插件工具不能与内置工具重名，运行时工具也不能与已有工具重名（[`tools/src/lib.rs#L133-L184`](/rust/crates/tools/src/lib.rs#L133-L184)）。

### 权限过滤

`GlobalToolRegistry::definitions` 方法（[`tools/src/lib.rs#L247-L278`](/rust/crates/tools/src/lib.rs#L247-L278)）在返回工具定义列表时，会根据 `--allowedTools`（或配置中的允许列表）过滤：

```rust
pub fn definitions(&self, allowed_tools: Option<&BTreeSet<String>>) -> Vec<ToolDefinition> {
    let builtin = mvp_tool_specs()
        .into_iter()
        .filter(|spec| allowed_tools.is_none_or(|allowed| allowed.contains(spec.name)))
        ...
}
```

这与 TypeScript 中的 `filterToolsByDenyRules(base, permissionContext)` 等价——都是在向模型发送 tool definitions 之前做最后一道过滤。

---

## 工具调用的完整链路

原文描述了 TypeScript 中从 AI 发出 `tool_use` 到结果回传的 10 步链路。Rust 中的对应链路在 `ConversationRuntime::run_turn` 中完全展开，核心逻辑位于 [`runtime/src/conversation.rs#L296-L484`](/rust/crates/runtime/src/conversation.rs#L296-L484)。

### 源码追踪

```rust
pub fn run_turn(
    &mut self,
    user_input: impl Into<String>,
    mut prompter: Option<&mut dyn PermissionPrompter>,
) -> Result<TurnSummary, RuntimeError> {
    // 1. 接收用户输入，写入 Session
    self.session.push_user_text(user_input)?;

    loop {
        // 2. 组装 ApiRequest（system prompt + messages）
        let request = ApiRequest { system_prompt: ..., messages: ... };

        // 3. 调用 api_client.stream(request) 获取流式事件
        let events = self.api_client.stream(request)?;

        // 4. 解析 assistant_message，提取 pending_tool_uses
        let pending_tool_uses = assistant_message.blocks.iter().filter_map(|block| match block {
            ContentBlock::ToolUse { id, name, input } => Some((id.clone(), name.clone(), input.clone())),
            _ => None,
        }).collect::<Vec<_>>();

        // 5. 若无 tool_use，则本轮结束
        if pending_tool_uses.is_empty() { break; }

        // 6. 遍历每个 tool_use，执行权限检查 + 调用
        for (tool_use_id, tool_name, input) in pending_tool_uses {
            // 6a. 运行 PreToolUse hook
            let pre_hook_result = self.run_pre_tool_use_hook(&tool_name, &input);

            // 6b. 权限判定
            let permission_outcome = ... self.permission_policy.authorize_with_context(...);

            // 6c. 执行工具（或返回拒绝）
            let result_message = match permission_outcome {
                PermissionOutcome::Allow => {
                    let (output, is_error) = match self.tool_executor.execute(&tool_name, &effective_input) { ... };
                    // 6d. 运行 PostToolUse hook
                    let post_hook_result = self.run_post_tool_use_hook(...);
                    ConversationMessage::tool_result(tool_use_id, tool_name, output, is_error)
                }
                PermissionOutcome::Deny { reason } => {
                    ConversationMessage::tool_result(tool_use_id, tool_name, reason, true)
                }
            };

            // 7. 将 tool_result 回写到 Session
            self.session.push_message(result_message.clone())?;
            tool_results.push(result_message);
        }
    }
}
```

### 关键差异

与 TypeScript 版本相比，Rust 实现把 **hooks** 嵌入了链路更深层的位置：

- **PreToolUse**：在权限检查之前运行，hook 可以修改输入、覆盖权限决策，甚至直接取消执行（[`runtime/src/conversation.rs#L370-L372`](/rust/crates/runtime/src/conversation.rs#L370-L372)）。
- **PostToolUse / PostToolUseFailure**：在工具执行之后运行，结果会拼接到最终输出中（[`runtime/src/conversation.rs#L433-L439`](/rust/crates/runtime/src/conversation.rs#L433-L439)）。

Hook 的具体实现见 [`runtime/src/hooks.rs`](/rust/crates/runtime/src/hooks.rs)。HookRunner 支持通过 shell 脚本拦截工具调用，返回 JSON 结构来决定 `Allow`/`Deny`/`Ask`，甚至修改 `updated_input`。

---

## 具体工具的源码映射

### Bash 命令执行工具

Bash 的输入输出结构定义在 [`runtime/src/bash.rs#L17-L67`](/rust/crates/runtime/src/bash.rs#L17-L67)：

```rust
pub struct BashCommandInput {
    pub command: String,
    pub timeout: Option<u64>,
    pub description: Option<String>,
    pub run_in_background: Option<bool>,
    pub dangerously_disable_sandbox: Option<bool>,
    pub namespace_restrictions: Option<bool>,
    pub isolate_network: Option<bool>,
    pub filesystem_mode: Option<FilesystemIsolationMode>,
    pub allowed_mounts: Option<Vec<String>>,
}

pub struct BashCommandOutput {
    pub stdout: String,
    pub stderr: String,
    pub raw_output_path: Option<String>,
    pub interrupted: bool,
    pub is_image: Option<bool>,
    pub background_task_id: Option<String>,
    pub sandbox_status: Option<SandboxStatus>,
    ...
}
```

执行入口 `execute_bash`（[`runtime/src/bash.rs#L69-L103`](/rust/crates/runtime/src/bash.rs#L69-L103)）会根据 `run_in_background` 决定是后台 spawn 还是同步等待；同步路径会走到 `execute_bash_async`，使用 `tokio::process::Command` 执行，并支持基于毫秒的超时控制。

沙箱方面，`prepare_tokio_command`（[`runtime/src/bash.rs#L212-L237`](/rust/crates/runtime/src/bash.rs#L212-L237)）会先检查 Linux 下的沙箱启动器（`build_linux_sandbox_command`），如果没有则降级到普通 `sh -lc`，但会根据 `SandboxStatus` 重定向 `HOME` 和 `TMPDIR` 到工作区下的隔离目录。

### 文件操作工具（Read / Write / Edit / Glob / Grep）

全部实现在 `runtime/src/file_ops.rs` 中。

#### Read

`read_file` 函数（[`runtime/src/file_ops.rs#L174-L221`](/rust/crates/runtime/src/file_ops.rs#L174-L221)）：

- 先将路径规范化为绝对路径。
- 检查文件大小是否超过 `MAX_READ_SIZE`（10MB），超限则返回 `InvalidData` 错误。
- 检测二进制文件（首 8192 字节检查 NUL 字节），拒绝读取。
- 支持 `offset` / `limit` 参数实现行窗口读取；返回 `num_lines`、`start_line`、`total_lines` 等元数据。

```rust
pub fn read_file(path: &str, offset: Option<usize>, limit: Option<usize>) -> io::Result<ReadFileOutput>
```

#### Write

`write_file`（[`runtime/src/file_ops.rs#L223-L255`](/rust/crates/runtime/src/file_ops.rs#L223-L255)）：

- 内容大小检查（`MAX_WRITE_SIZE` = 10MB）。
- 自动创建父目录。
- 返回 `WriteFileOutput`，包含 `kind`（`create` 或 `update`）、`structured_patch`、`original_file`。

#### Edit

`edit_file`（[`runtime/src/file_ops.rs#L257-L296`](/rust/crates/runtime/src/file_ops.rs#L257-L296)）：

- 使用字符串级别的 `replacen`（默认替换 1 次）或 `replace`（`replace_all = true`）。
- 若 `old_string` 在文件中不存在，返回 `NotFound` 错误。
- 同样返回带 patch 元数据的 `EditFileOutput`。

#### Glob

`glob_search`（[`runtime/src/file_ops.rs#L298-L340`](/rust/crates/runtime/src/file_ops.rs#L298-L340)）使用 `glob::glob` 解析模式，按修改时间降序排列，最多返回 100 个文件，超长的结果会标记 `truncated: true`。

#### Grep

`grep_search`（[`runtime/src/file_ops.rs#L342-L450`](/rust/crates/runtime/src/file_ops.rs#L342-L450)）使用 `regex::RegexBuilder` 编译模式，配合 `walkdir::WalkDir` 遍历文件。支持：

- `output_mode`：`files_with_matches`（默认）、`content`、`count`
- `glob` 过滤器、`file_type` 后缀过滤
- `-B` / `-A` / `-C` 上下文行
- `-n` 行号前缀
- `-i` 忽略大小写
- `multiline` 多行匹配（通过 `dot_matches_new_line`）
- `head_limit` / `offset` 分页（默认 limit = 250）

### MCP 工具的扩展

MCP（Model Context Protocol）在 `claw-code` 中由 `runtime/src/mcp_tool_bridge.rs` 提供桥接。`McpToolRegistry` 维护了一个 `HashMap<String, McpServerState>`，记录了每个 MCP server 的 tools 和 resources。

MCP 相关的内置工具有四个：

- `ListMcpResources`
- `ReadMcpResource`
- `McpAuth`
- `MCP`（调用 server 上的具体 tool）

它们在 `tools/src/lib.rs` 的 `execute_tool_with_enforcer` 中分别映射到 `run_list_mcp_resources`、`run_read_mcp_resource`、`run_mcp_auth`、`run_mcp_tool`。

`run_mcp_tool` 最终调用 `global_mcp_registry().call_tool(&input.server, &input.tool, &args)`，后者在 [`runtime/src/mcp_tool_bridge.rs#L240-L278`](/rust/crates/runtime/src/mcp_tool_bridge.rs#L240-L278) 中会：

1. 验证 server 是否存在且状态为 `Connected`。
2. 验证 tool 是否在该 server 的 tools 列表中。
3. 通过 `spawn_tool_call` 在独立线程中创建 tokio runtime，向 `McpServerManager` 发起 `tools/call` JSON-RPC 请求。
4. 返回结果或错误。

这与 TypeScript 中 `mcpInfo` 标记来源、直接使用 JSON Schema 作为 inputJSONSchema 的设计完全对应。

---

## 50+ 内置工具全景

Rust 实现中，`mvp_tool_specs()` 返回的内置工具规格覆盖了原文所述的大部分领域：

| 领域 | 工具名 |
| --- | --- |
| **文件操作** | `read_file`、`write_file`、`edit_file`、`glob_search`、`grep_search`、`NotebookEdit` |
| **命令执行** | `bash`、`PowerShell`、`REPL` |
| **对话管理** | `Agent`、`SendUserMessage`、`AskUserQuestion` |
| **任务追踪** | `TodoWrite`、`TaskCreate`、`TaskGet`、`TaskList`、`TaskStop`、`TaskUpdate`、`TaskOutput`、`RunTaskPacket` |
| **Web 能力** | `WebFetch`、`WebSearch` |
| **规划与版本** | `EnterPlanMode`、`ExitPlanMode`、`Sleep`、`StructuredOutput` |
| **Worker 系统** | `WorkerCreate`、`WorkerGet`、`WorkerObserve`、`WorkerResolveTrust`、`WorkerAwaitReady`、`WorkerSendPrompt`、`WorkerRestart`、`WorkerTerminate`、`WorkerObserveCompletion` |
| **团队与调度** | `TeamCreate`、`TeamDelete`、`CronCreate`、`CronDelete`、`CronList` |
| **代码智能** | `LSP` |
| **MCP 扩展** | `MCP`、`ListMcpResources`、`ReadMcpResource`、`McpAuth` |
| **远程/调试** | `RemoteTrigger`、`ToolSearch`、`Config`、`Skill`、`Brief` |

---

## 工具与权限、Hook 的交互

### 权限策略的执行点

权限执行不在 `tools` crate 内部，而在 `runtime` crate 的 `ConversationRuntime::run_turn` 中：

```rust
let permission_outcome = if pre_hook_result.is_cancelled() { ... }
else if pre_hook_result.is_denied() { ... }
else if let Some(prompt) = prompter.as_mut() {
    self.permission_policy.authorize_with_context(&tool_name, &effective_input, &permission_context, Some(*prompt))
} else {
    self.permission_policy.authorize_with_context(&tool_name, &effective_input, &permission_context, None)
};
```

见 [`runtime/src/conversation.rs#L338-L365`](/rust/crates/runtime/src/conversation.rs#L338-L365)。

### 规则匹配引擎

`PermissionPolicy::authorize_with_context` 内部实现了 **deny → ask → allow** 的三级规则匹配（[`runtime/src/permissions.rs#L182-L292`](/rust/crates/runtime/src/permissions.rs#L182-L292)）：

1. 先查 `deny_rules`，命中直接拒绝。
2. 再查 hook 的 `PermissionOverride`。
3. 再查 `ask_rules`，命中则强制进入 prompt（即使当前模式足够高）。
4. 最后比较 `current_mode >= required_mode`，满足则允许。

规则语法支持 `tool_name(pattern)` 形式，例如 `bash(git:*)` 表示匹配 Bash 工具中 `command` 或 `path` 以 `git` 为前缀的输入。规则解析器在 [`runtime/src/permissions.rs#L349-L402`](/rust/crates/runtime/src/permissions.rs#L349-L402) 中实现。

### Hook 生命周期

`HookRunner` 定义了三个事件（[`runtime/src/hooks.rs#L18-L23`](/rust/crates/runtime/src/hooks.rs#L18-L23)）：

```rust
pub enum HookEvent {
    PreToolUse,
    PostToolUse,
    PostToolUseFailure,
}
```

执行时，HookRunner 会设置环境变量（`HOOK_EVENT`、`HOOK_TOOL_NAME`、`HOOK_TOOL_INPUT`、`HOOK_TOOL_OUTPUT`、`HOOK_TOOL_IS_ERROR`），并通过 `HOOK_TOOL_INPUT` 把 tool input 作为 JSON payload 写到 stdin。hook 脚本通过 stdout 返回 JSON 来传递决策或消息。

Hook 的三种退出码语义（[`runtime/src/hooks.rs#L440-L455`](/rust/crates/runtime/src/hooks.rs#L440-L455)）：

- `0`：Allow（如果 JSON 里有 `"deny": true` 或 `"decision": "block"` 则转为 Deny）
- `2`：Deny
- 其他非零：Failed（工具执行链路继续，但标记为 error）

---

## 源码索引

| 源码位置 | 描述 |
| --- | --- |
| [`runtime/src/conversation.rs#L57-L59`](/rust/crates/runtime/src/conversation.rs#L57-L59) | `ToolExecutor` trait 定义 |
| [`runtime/src/conversation.rs#L296-L484`](/rust/crates/runtime/src/conversation.rs#L296-L484) | `ConversationRuntime::run_turn`，完整的 tool use → tool result 循环 |
| [`runtime/src/conversation.rs#L370-L372`](/rust/crates/runtime/src/conversation.rs#L370-L372) | PreToolUse hook 调用点 |
| [`runtime/src/conversation.rs#L433-L439`](/rust/crates/runtime/src/conversation.rs#L433-L439) | PostToolUse / PostToolUseFailure hook 调用点 |
| [`tools/src/lib.rs#L100-L107`](/rust/crates/tools/src/lib.rs#L100-L107) | `ToolSpec` 结构体（name / description / input_schema / required_permission） |
| [`tools/src/lib.rs#L125-L131`](/rust/crates/tools/src/lib.rs#L125-L131) | `GlobalToolRegistry::builtin()` 构造器 |
| [`tools/src/lib.rs#L133-L184`](/rust/crates/tools/src/lib.rs#L133-L184) | `GlobalToolRegistry::with_plugin_tools` / `with_runtime_tools` 注册与去重 |
| [`tools/src/lib.rs#L192-L244`](/rust/crates/tools/src/lib.rs#L192-L244) | `normalize_allowed_tools`，包含 read/write/edit/glob/grep 别名映射 |
| [`tools/src/lib.rs#L247-L278`](/rust/crates/tools/src/lib.rs#L247-L278) | `definitions()` 方法：按 allowed_tools 过滤后返回 `ToolDefinition` 列表 |
| [`tools/src/lib.rs#L385-...`](/rust/crates/tools/src/lib.rs#L385) | `mvp_tool_specs()` 的开头部分：bash / read_file / write_file / edit_file / glob_search / grep_search / WebFetch / WebSearch / TodoWrite 等 |
| [`tools/src/lib.rs#L1178-L1267`](/rust/crates/tools/src/lib.rs#L1178-L1267) | `execute_tool_with_enforcer`：40+ 工具的统一 match 分发器 |
| [`tools/src/lib.rs#L2000-L2002`](/rust/crates/tools/src/lib.rs#L2000-L2002) | `run_tool_search` 入口 |
| [`runtime/src/bash.rs#L17-L67`](/rust/crates/runtime/src/bash.rs#L17-L67) | `BashCommandInput` / `BashCommandOutput` |
| [`runtime/src/bash.rs#L69-L103`](/rust/crates/runtime/src/bash.rs#L69-L103) | `execute_bash` 同步入口 |
| [`runtime/src/bash.rs#L212-L237`](/rust/crates/runtime/src/bash.rs#L212-L237) | `prepare_tokio_command`：沙箱启动器检查与普通 shell 回退 |
| [`runtime/src/bash.rs#L289`](/rust/crates/runtime/src/bash.rs#L289) | `MAX_OUTPUT_BYTES = 16_384`（bash 输出截断阈值） |
| [`runtime/src/file_ops.rs#L12-L16`](/rust/crates/runtime/src/file_ops.rs#L12-L16) | `MAX_READ_SIZE` / `MAX_WRITE_SIZE`（10MB） |
| [`runtime/src/file_ops.rs#L174-L221`](/rust/crates/runtime/src/file_ops.rs#L174-L221) | `read_file` 实现 |
| [`runtime/src/file_ops.rs#L223-L255`](/rust/crates/runtime/src/file_ops.rs#L223-L255) | `write_file` 实现 |
| [`runtime/src/file_ops.rs#L257-L296`](/rust/crates/runtime/src/file_ops.rs#L257-L296) | `edit_file` 实现 |
| [`runtime/src/file_ops.rs#L298-L340`](/rust/crates/runtime/src/file_ops.rs#L298-L340) | `glob_search` 实现 |
| [`runtime/src/file_ops.rs#L342-L450`](/rust/crates/runtime/src/file_ops.rs#L342-L450) | `grep_search` 实现 |
| [`runtime/src/permissions.rs#L7-L15`](/rust/crates/runtime/src/permissions.rs#L7-L15) | `PermissionMode` 五级权限枚举 |
| [`runtime/src/permissions.rs#L173-L292`](/rust/crates/runtime/src/permissions.rs#L173-L292) | `PermissionPolicy::authorize_with_context` |
| [`runtime/src/permissions.rs#L349-L402`](/rust/crates/runtime/src/permissions.rs#L349-L402) | 权限规则解析器 `PermissionRule::parse` |
| [`runtime/src/hooks.rs#L18-L23`](/rust/crates/runtime/src/hooks.rs#L18-L23) | `HookEvent` 枚举 |
| [`runtime/src/hooks.rs#L440-L455`](/rust/crates/runtime/src/hooks.rs#L440-L455) | hook 脚本退出码语义：0 = Allow, 2 = Deny, 其他 = Failed |
| [`runtime/src/mcp_tool_bridge.rs#L240-L278`](/rust/crates/runtime/src/mcp_tool_bridge.rs#L240-L278) | `McpToolRegistry::call_tool`：跨线程 MCP JSON-RPC 调用 |
| [`rusty-claude-cli/src/main.rs#L168-L240`](/rust/crates/rusty-claude-cli/src/main.rs#L168-L240) | CLI 入口 `run()`，按 `CliAction` 分发 Prompt / Repl 等模式 |
| [`rusty-claude-cli/src/main.rs#L2789-L2838`](/rust/crates/rusty-claude-cli/src/main.rs#L2789-L2838) | `run_repl`：REPL 启动，创建 `LiveCli` 并进入 `run_turn` 循环 |
