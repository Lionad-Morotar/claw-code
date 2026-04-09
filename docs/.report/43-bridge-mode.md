# Unit 43: Bridge Mode — 远程控制与 MCP Tool Bridge 实现

> 原始页面：https://ccb.agent-aura.top/docs/features/bridge-mode  
> 生成时间：2026-04-09

---

## 1. 功能概述

Bridge mode 将本地 CLI 注册为可被远程驱动的 "bridge 环境"，使本地终端成为远程控制面的执行者。Rust 运行时中对应的实现分两条线：

1. **Bootstrap 快速路径**：`BridgeFastPath` 识别 `remote-control` 子命令，走独立的启动阶段。
2. **MCP Tool Bridge**：`runtime::mcp_tool_bridge` 提供状态化的 MCP 服务器注册表，支撑/ListMcpResources、ReadMcpResource、McpAuth、MCP 四个工具面。

本报告聚焦**源码级实现**，所有行号均指向当前 `research` 分支。

---

## 2. Bootstrap 中的 BridgeFastPath

文件：`rust/crates/runtime/src/bootstrap.rs`

```rust
pub enum BootstrapPhase {
    CliEntry,
    FastPathVersion,
    StartupProfiler,
    SystemPromptFastPath,
    ChromeMcpFastPath,
    DaemonWorkerFastPath,
    BridgeFastPath,          // #L9
    DaemonFastPath,
    BackgroundSessionFastPath,
    TemplateFastPath,
    EnvironmentRunnerFastPath,
    MainRuntime,
}
```

默认启动计划 `claude_code_default()` 将 `BridgeFastPath` 放在 `DaemonWorkerFastPath` 之后、`DaemonFastPath` 之前：

```rust
BootstrapPlan::from_phases(vec![
    BootstrapPhase::CliEntry,
    BootstrapPhase::FastPathVersion,
    BootstrapPhase::StartupProfiler,
    BootstrapPhase::SystemPromptFastPath,
    BootstrapPhase::ChromeMcpFastPath,
    BootstrapPhase::DaemonWorkerFastPath,
    BootstrapPhase::BridgeFastPath,  // #L32
    BootstrapPhase::DaemonFastPath,
    // ...
])
```

compat-harness 通过源码字符串匹配来决定是否注入该阶段：

文件：`rust/crates/compat-harness/src/lib.rs`（#L202）

```rust
if source.contains("remote-control") {
    phases.push(BootstrapPhase::BridgeFastPath);
}
```

这确保了在 fork 出的老版本 JS 代码包含 `remote-control` 逻辑时，Rust bootstrap 能正确为其预热 bridge 相关的运行时资源。

---

## 3. MCP Tool Bridge — mcp_tool_bridge.rs

文件：`rust/crates/runtime/src/mcp_tool_bridge.rs`（共 921 行）

这是 Bridge mode 与 MCP 生态之间的**核心粘合层**。它暴露了一个线程安全的 `McpToolRegistry`，向上对 `tools` crate 提供同步调用面，向下委托给 `McpServerManager` 做异步_stdio MCP 通信。

### 3.1 连接状态模型

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum McpConnectionStatus {
    Disconnected,
    Connecting,
    Connected,
    AuthRequired,
    Error,
}
```

行号：#L25-33

每个服务器的状态被封装为 `McpServerState`：

```rust
pub struct McpServerState {
    pub server_name: String,
    pub status: McpConnectionStatus,
    pub tools: Vec<McpToolInfo>,
    pub resources: Vec<McpResourceInfo>,
    pub server_info: Option<String>,
    pub error_message: Option<String>,
}
```

行号：#L64-72

### 3.2 注册表结构

```rust
#[derive(Debug, Clone, Default)]
pub struct McpToolRegistry {
    inner: Arc<Mutex<HashMap<String, McpServerState>>>,
    manager: Arc<OnceLock<Arc<Mutex<McpServerManager>>>>,
}
```

行号：#L74-77

关键接口（节选）：

| 方法 | 行号 | 说明 |
|------|------|------|
| `set_manager` | #L85 | 将 `McpServerManager` 注入注册表， OnceLock 保证仅初始化一次 |
| `register_server` | #L92 | upsert 式注册服务器元数据 |
| `get_server` / `list_servers` | #L114 / #L119 | 查询 |
| `list_resources` / `read_resource` | #L124 / #L140 | 资源浏览与读取，要求状态为 `Connected` |
| `list_tools` | #L161 | 工具列表 |
| `call_tool` | #L240 | 同步调用 MCP 工具 |
| `set_auth_status` | #L281 | 动态更新认证状态 |
| `disconnect` | #L295 | 移除服务器 |

### 3.3 同步调用异步 MCP 的实现细节

`call_tool` 在运行时需要把 `&self` 的同步调用桥接到 `McpServerManager` 的 `async` 方法上，方案是在一个独立线程里启动一个** current_thread tokio runtime**：

```rust
fn spawn_tool_call(
    manager: Arc<Mutex<McpServerManager>>,
    qualified_tool_name: String,
    arguments: Option<serde_json::Value>,
) -> Result<serde_json::Value, String> {
    let join_handle = std::thread::Builder::new()
        .name(format!("mcp-tool-call-{qualified_tool_name}"))
        .spawn(move || {
            let runtime = tokio::runtime::Builder::new_current_thread()
                .enable_all()
                .build()
                .map_err(|error| format!("failed to create MCP tool runtime: {error}"))?;

            runtime.block_on(async move {
                let response = {
                    let mut manager = manager
                        .lock()
                        .map_err(|_| "mcp server manager lock poisoned".to_string())?;
                    manager.discover_tools().await.map_err(|error| error.to_string())?;
                    let response = manager
                        .call_tool(&qualified_tool_name, arguments)
                        .await
                        .map_err(|error| error.to_string());
                    let shutdown = manager.shutdown().await.map_err(|error| error.to_string());

                    match (response, shutdown) {
                        (Ok(response), Ok(())) => Ok(response),
                        (Err(error), Ok(())) | (Err(error), Err(_)) => Err(error),
                        (Ok(_), Err(error)) => Err(error),
                    }
                }?;

                if let Some(error) = response.error {
                    return Err(format!(
                        "MCP server returned JSON-RPC error for tools/call: {} ({})",
                        error.message, error.code
                    ));
                }

                let result = response.result.ok_or_else(|| {
                    "MCP server returned no result for tools/call".to_string()
                })?;

                serde_json::to_value(result)
                    .map_err(|error| format!("failed to serialize MCP tool result: {error}"))
            })
        })
        .map_err(|error| format!("failed to spawn MCP tool call thread: {error}"))?;

    join_handle.join().map_err(|panic_payload| {
        if let Some(message) = panic_payload.downcast_ref::<&str>() {
            format!("MCP tool call thread panicked: {message}")
        } else if let Some(message) = panic_payload.downcast_ref::<String>() {
            format!("MCP tool call thread panicked: {message}")
        } else {
            "MCP tool call thread panicked".to_string()
        }
    })?
}
```

行号：#L177-238

注意该线程每次调用都会执行完整的 `discover_tools -> call_tool -> shutdown` 生命周期。这意味着每个工具调用都会创建一个全新的 tokio `current_thread` runtime 并 spawn 一个 OS 线程，频繁调用时线程创建/销毁开销和进程重启开销可能成为瓶颈（_latencies 在毫秒级但不排除高并发时累积）。若需优化，应考虑复用 runtime 和 MCP stdio 长连接。每个工具调用结束后关闭进程，下次调用重新初始化，确保状态隔离。

### 3.4 工具名规范化

调用前，注册表通过 `mcp_tool_name` 将 `(server_name, tool_name)` 拼接为-qualified name：

```rust
Self::spawn_tool_call(
    manager,
    mcp_tool_name(server_name, tool_name),  // #L275
    (!arguments.is_null()).then(|| arguments.clone()),
)
```

`mcp_tool_name` 实现在 `rust/crates/runtime/src/mcp.rs` #L31：

```rust
pub fn mcp_tool_name(server_name: &str, tool_name: &str) -> String {
    format!("{}{}", mcp_tool_prefix(server_name), normalize_name_for_mcp(tool_name))
}
```

---

## 4. 下层 stdio MCP 通信 — mcp_stdio.rs

文件：`rust/crates/runtime/src/mcp_stdio.rs`

### 4.1 McpServerManager

```rust
pub struct McpServerManager {
    servers: BTreeMap<String, ManagedMcpServer>,
    unsupported_servers: Vec<UnsupportedMcpServer>,
    tool_index: BTreeMap<String, ToolRoute>,
    next_request_id: u64,
}
```

行号：#L480-486

- `from_servers` 仅支持 `McpTransport::Stdio`，其他 transport 进入 `unsupported_servers`。
- `discover_tools` 遍历服务器，填充 `tool_index`（qualified_name -> server_name + raw_name）。
- `call_tool` 从 `tool_index` 路由，找到原始 tool name，通过 JSON-RPC `tools/call` 发出请求。失败后若判定为可恢复错误会触发 `reset_server`。

### 4.2 进程模型

```rust
struct ManagedMcpServer {
    bootstrap: McpClientBootstrap,
    process: Option<McpStdioProcess>,
    initialized: bool,
}
```

行号：#L457-460（ToolRoute）、#L463-468（ManagedMcpServer）

`McpStdioProcess` 封装了 `tokio::process::Child`，通过 `ChildStdin` / `ChildStdout` 收发 JSON-RPC 消息。

---
## 5. 工具面暴露 — tools/src/lib.rs

文件：`rust/crates/tools/src/lib.rs`

### 5.1 全局单例

```rust
fn global_mcp_registry() -> &'static McpToolRegistry {
    use std::sync::OnceLock;
    static REGISTRY: OnceLock<McpToolRegistry> = OnceLock::new();
    REGISTRY.get_or_init(McpToolRegistry::new)
}
```

行号：#L41-46

### 5.2 四个 MCP 工具定义

| 工具名 | 行号 | 权限模式 |
|--------|------|----------|
| `ListMcpResources` | #L1074 | `ReadOnly` |
| `ReadMcpResource` | #L1086 | `ReadOnly` |
| `McpAuth` | #L1100 | `DangerFullAccess` |
| `MCP` | #L1129 | `ReadOnly` |

### 5.3 工具处理器

- `run_list_mcp_resources` — #L1625
- `run_read_mcp_resource` — #L1656
- `run_mcp_auth` — #L1677（当前仅返回服务器元数据，认证流未在此完成）
- `run_mcp_tool` — #L1760

所有处理器都通过 `global_mcp_registry()` 访问共享注册表，返回统一 JSON 结构 `{server, tool/uri, result/error, status}`。

---

## 6. Plugin Lifecycle 与 Degraded Mode

文件：`rust/crates/runtime/src/plugin_lifecycle.rs`

`McpToolInfo` / `McpResourceInfo` 被重导出为 `ToolInfo` / `ResourceInfo`：

```rust
pub type ToolInfo = McpToolInfo;
pub type ResourceInfo = McpResourceInfo;
```

行号：#L16-17

当部分服务器启动失败时，系统进入 `PluginState::Degraded`，并在工具响应中携带 `McpDegradedReport`。这会反映到前端：`tools/src/lib.rs` 在 `AgentState` 中通过 `pending_mcp_servers` 和 `mcp_degraded` 字段暴露降级信息（#L2438-2439）。

---

## 7. 测试覆盖

`mcp_tool_bridge.rs` 自带丰富的内联单元测试（#L314-920），涵盖：

- 服务器注册与查询（`registers_and_retrieves_server`）
- 资源列表/读取，含错误分支
- 无 manager 时的工具调用拒绝
- **真实 stdio MCP server 集成测试**：通过生成临时 Python bridge server 脚本，验证完整的 `initialize -> tools/list -> tools/call` 链路（#L572、#L815）
- auth 状态切换与 disconnect
- upsert 覆盖旧状态

该测试脚本展示了整个 MCP stdio 协议的最小子集（JSON-RPC + Content-Length framing）。

---

## 8. 架构要点总结

1. **Bridge = Bootstrap FastPath + MCP Tool Bridge**。前者用于兼容层识别远程控制入口，后者是实际能力暴露点。
2. **同步/异步桥接**：`McpToolRegistry` 是同步面（供 tools crate 调用），内部用独立线程 + tokio current_thread runtime 包裹 `McpServerManager` 的异步 I/O。
3. **生命周期简单直接**：每次 `call_tool` 做 discover -> call -> shutdown，适配无状态或短连接的 stdio MCP 进程。
4. **名字空间隔离**：通过 `mcp__{server}__{tool}` 的 qualified name 避免不同 MCP 服务器之间的工具名冲突。
5. **降级支持**：部分服务器失败时进入 `Degraded` 状态，工具列表和 AgentState 会将其显式传回前端。

---

## 9. 源码索引

| 文件 | 职责 |
|------|------|
| `rust/crates/runtime/src/bootstrap.rs` | `BridgeFastPath` 启动阶段定义 |
| `rust/crates/compat-harness/src/lib.rs` | 基于源码字符串匹配注入 `BridgeFastPath` |
| `rust/crates/runtime/src/mcp_tool_bridge.rs` | `McpToolRegistry`：连接状态、资源/工具列表、同步工具调用 |
| `rust/crates/runtime/src/mcp_stdio.rs` | `McpServerManager`：stdio 进程管理、JSON-RPC、路由 |
| `rust/crates/runtime/src/mcp.rs` | `mcp_tool_name`、名字规范化 |
| `rust/crates/runtime/src/plugin_lifecycle.rs` | `PluginState`、降级模式、生命周期事件 |
| `rust/crates/tools/src/lib.rs` | `ListMcpResources` / `ReadMcpResource` / `McpAuth` / `MCP` 工具面 |
