# MCP 协议：连接管理、工具发现与执行链路

> 本报告基于 [Claude Code 中文文档 — MCP 协议](https://ccb.agent-aura.top/docs/extensibility/mcp-protocol) 的架构描述，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 架构总览：从配置到可用工具

```
.claw.json / settings.json: { mcpServers: { "my-db": { command: "uvx", args: [...] } } }
  ↓
RuntimeConfig::mcp().servers()        ← 合并 user/project/local 三级配置
  ↓
RuntimeMcpState::new()                ← CLI 初始化时构建 MCP 状态
  ↓
McpServerManager::from_runtime_config()
  ├── 过滤出 stdio 类型服务器
  └── 非 stdio 服务器记入 unsupported_servers
  ↓
manager.discover_tools_best_effort()  ← 逐个初始化、握手、tools/list
  ↓
每个工具包装为 RuntimeToolDefinition ← 统一 Tool 接口注入 CLI
  ↓
工具名格式: mcp__<serverName>__<toolName>  ← mcp_tool_name()
```

与上游 TypeScript 实现支持 7 种传输层不同，`claw-code` 当前 Rust 实现中的 `McpServerManager` **仅完整支持 `stdio` 传输**；`sse`、`http`、`ws`、`sdk`、`claudeai-proxy` 等类型会被记录到 `unsupported_servers` 而不会进入连接生命周期（[`mcp_stdio.rs#L531-L548`](/rust/crates/runtime/src/mcp_stdio.rs#L531-L548)）。这是当前实现的一个重要边界条件。

---

## 传输层实现：stdio 子进程的一生

### McpStdioProcess：LSP 帧格式的 JSON-RPC

`McpStdioProcess` 是对 MCP stdio 子进程的标准 I/O 封装。进程间通信采用**与 LSP 相同的帧格式**：`Content-Length: <len>\r\n\r\n<payload>`。

参见 [`mcp_stdio.rs#L1139-L1225`](/rust/crates/runtime/src/mcp_stdio.rs#L1139-L1225) 的核心方法：

```rust
pub async fn write_frame(&mut self, payload: &[u8]) -> io::Result<()> {
    let encoded = encode_frame(payload);
    self.write_all(&encoded).await?;
    self.flush().await
}

pub async fn read_frame(&mut self) -> io::Result<Vec<u8>> {
    let mut content_length = None;
    loop {
        let mut line = String::new();
        let bytes_read = self.stdout.read_line(&mut line).await?;
        // ... 解析 Content-Length 头部
    }
    // ... 按长度读取 payload
}
```

`encode_frame` 实现位于 [`mcp_stdio.rs#L1244-L1249`](/rust/crates/runtime/src/mcp_stdio.rs#L1244-L1249)。它确保每个 JSON-RPC 消息都带上精确的字节长度头，避免换行符截断问题。

### JSON-RPC 请求/响应类型

协议层定义了与 MCP 规范一一对应的 JSON-RPC 结构（[`mcp_stdio.rs#L34-L181`](/rust/crates/runtime/src/mcp_stdio.rs#L34-L181)）：

```rust
pub struct JsonRpcRequest<T = JsonValue> {
    pub jsonrpc: String,
    pub id: JsonRpcId,
    pub method: String,
    pub params: Option<T>,
}

pub struct JsonRpcResponse<T = JsonValue> {
    pub jsonrpc: String,
    pub id: JsonRpcId,
    pub result: Option<T>,
    pub error: Option<JsonRpcError>,
}
```

请求的 `id` 可以是数字、字符串或 Null；`McpServerManager` 内部使用自增的 `u64` 作为 `JsonRpcId::Number`（[`mcp_stdio.rs#L963-L967`](/rust/crates/runtime/src/mcp_stdio.rs#L963-L967)）。

---

## 连接生命周期：初始化、发现、调用、关闭

### McpServerManager 的状态模型

`McpServerManager` 为每个服务器维护一个 `ManagedMcpServer`：

```rust
struct ManagedMcpServer {
    bootstrap: McpClientBootstrap,
    process: Option<McpStdioProcess>,
    initialized: bool,
}
```

参见 [`mcp_stdio.rs#L463-529`](/rust/crates/runtime/src/mcp_stdio.rs#L463-529)。状态机非常简单：`process` 存在且 `initialized == true` 时，该服务器才被认为可用。

### ensure_server_ready：带重试的初始化握手

任何工具发现或调用之前，都必须经过 `ensure_server_ready`（[`mcp_stdio.rs#L1052-L1120`](/rust/crates/runtime/src/mcp_stdio.rs#L1052-L1120)）。它的逻辑是：

1. 若进程已退出，调用 `reset_server` 清除状态
2. 若 `process` 为 None，通过 `spawn_mcp_stdio_process` 启动子进程
3. 若 `initialized == false`，发送 `initialize` 请求（协议版本 `"2025-03-26"`）
4. 收到正确响应后设置 `initialized = true`
5. 若首次初始化失败且错误可重试（Transport / Timeout），会重试一次

初始化超时默认为 **10 秒**（`MCP_INITIALIZE_TIMEOUT_MS`，[`mcp_stdio.rs#L20-L24`](/rust/crates/runtime/src/mcp_stdio.rs#L20-L24)），测试环境下缩短为 200ms。

### 工具发现：discover_tools 与 discover_tools_best_effort

- `discover_tools`：严格模式，任一服务器失败即返回 `Err`
- `discover_tools_best_effort`：尽力模式，失败的服务器被记录到 `McpDiscoveryFailure`，成功的服务器继续可用

工具发现逻辑位于 [`mcp_stdio.rs#L806-L875`](/rust/crates/runtime/src/mcp_stdio.rs#L806-L875)。它处理 `tools/list` 请求，支持分页（`cursor`/`next_cursor`），并将每个原始工具名通过 `mcp_tool_name(server_name, tool.name)` 映射为**完全限定名**（qualified name）。

### 工具执行：call_tool

`McpServerManager::call_tool`（[`mcp_stdio.rs#L624-L773`](/rust/crates/runtime/src/mcp_stdio.rs#L624-L773)）的执行链路如下：

1. 在 `tool_index` 中根据 qualified name 查找 `ToolRoute`（含 `server_name` + `raw_name`）
2. 调用 `tool_call_timeout_ms` 获取该服务器配置的超时时间（默认 60 秒）
3. `ensure_server_ready` 确保进程和初始化就绪
4. 通过 `process.call_tool()` 发送 JSON-RPC `tools/call`
5. 若返回 Transport / Timeout / InvalidResponse 错误，触发 `reset_server` 以便下次调用自动重建连接

这个**调用后自动重置**的策略是容错的关键：stdio 子进程可能在一次调用后崩溃，manager 会清除旧进程，下一次 `ensure_server_ready` 会重新 `spawn` 并 `initialize`。

### 关闭：shutdown

`McpServerManager::shutdown`（[`mcp_stdio.rs#L723-L734`](/rust/crates/runtime/src/mcp_stdio.rs#L723-L734)）遍历所有服务器，对存活子进程调用 `process.shutdown()`。`shutdown` 的实现是：`try_wait` 检查进程是否仍在运行，若存活则 `child.kill().await`，随后 `child.wait().await` 回收僵尸进程（[`mcp_stdio.rs#L1231-L1241`](/rust/crates/runtime/src/mcp_stdio.rs#L1231-L1241)）。

---

## 错误处理与生命周期硬化

### McpServerManagerError 枚举

所有 MCP 相关错误都被封装为 `McpServerManagerError`（[`mcp_stdio.rs#L254-L282`](/rust/crates/runtime/src/mcp_stdio.rs#L254-L282)）：

```rust
pub enum McpServerManagerError {
    Io(io::Error),
    Transport { server_name: String, method: &'static str, source: io::Error },
    JsonRpc { server_name: String, method: &'static str, error: JsonRpcError },
    InvalidResponse { server_name: String, method: &'static str, details: String },
    Timeout { server_name: String, method: &'static str, timeout_ms: u64 },
    UnknownTool { qualified_name: String },
    UnknownServer { server_name: String },
}
```

每种错误都能自动映射到一个 `McpLifecyclePhase`（[`mcp_stdio.rs#L267-L282`](/rust/crates/runtime/src/mcp_stdio.rs#L267-L282)），如 `initialize` 对应 `InitializeHandshake`，`tools/list` 对应 `ToolDiscovery`，`tools/call` 对应 `Invocation`。

### 生命周期阶段与降级报告

`mcp_lifecycle_hardened.rs` 定义了 11 个生命周期阶段（[`mcp_lifecycle_hardened.rs#L16-L34`](/rust/crates/runtime/src/mcp_lifecycle_hardened.rs#L16-L34)）：

```rust
pub enum McpLifecyclePhase {
    ConfigLoad, ServerRegistration, SpawnConnect, InitializeHandshake,
    ToolDiscovery, ResourceDiscovery, Ready, Invocation, ErrorSurfacing, Shutdown, Cleanup,
}
```

当 `discover_tools_best_effort` 遇到部分成功、部分失败时，`McpServerManager` 会生成 `McpDegradedReport`（[`mcp_stdio.rs#L585-L620`](/rust/crates/runtime/src/mcp_stdio.rs#L585-L620)），记录哪些服务器可用、哪些失败、哪些工具因此缺失。这为 CLI 启动时的用户提示提供了结构化数据。

### 重试策略

以下错误会被认为是**可重试的**（`is_retryable_error`，[`mcp_stdio.rs#L878-L884`](/rust/crates/runtime/src/mcp_stdio.rs#L878-L884)）：

- `McpServerManagerError::Transport { .. }`
- `McpServerManagerError::Timeout { .. }`

首次失败后，manager 会调用 `reset_server` 并再尝试一次。测试用例 [`given_initialize_hangs_once_when_discover_tools_then_manager_retries_and_succeeds`](/rust/crates/runtime/src/mcp_stdio.rs#L2553-L2602) 和 [`given_tool_call_disconnects_once_when_calling_twice_then_manager_resets_and_next_call_succeeds`](/rust/crates/runtime/src/mcp_stdio.rs#L2603-L2663) 验证了这一行为。

---

## 工具名规范与签名匹配

### normalize_name_for_mcp

MCP 工具名和服务器名需要被规范化，以符合统一的命名空间。`normalize_name_for_mcp`（[`mcp.rs#L7-L23`](/rust/crates/runtime/src/mcp.rs#L7-L23)）将非字母数字、下划线、横杠的字符替换为 `_`：

```rust
pub fn normalize_name_for_mcp(name: &str) -> String {
    let mut normalized = name
        .chars()
        .map(|ch| match ch {
            'a'..='z' | 'A'..='Z' | '0'..='9' | '_' | '-' => ch,
            _ => '_',
        })
        .collect::<String>();
    // ... 对 claude.ai 前缀做额外下划线折叠
    normalized
}
```

### mcp_tool_name

完全限定工具名由 `mcp__<normalized_server>__<normalized_tool>` 构成（[`mcp.rs#L31-L37`](/rust/crates/runtime/src/mcp.rs#L31-L37)）。例如服务器 `"my-db"`、工具 `"query"` 会生成 `"mcp__my-db__query"`。

### scoped_mcp_config_hash

为了检测配置变更，`scoped_mcp_config_hash`（[`mcp.rs#L84-L121`](/rust/crates/runtime/src/mcp.rs#L84-L121)）对每种传输类型生成稳定哈希：

- `stdio`：command + args + env + timeout
- `http`/`sse`：url + headers + headers_helper + oauth
- `ws`：url + headers + headers_helper
- `sdk`：name
- `claudeai-proxy`：url + id

这个哈希函数是**纯 Rust 实现**的稳定哈希（[`mcp.rs#L152-L159`](/rust/crates/runtime/src/mcp.rs#L152-L159)），不依赖 `std::hash::DefaultHasher`，确保跨版本一致性。

---

## CLI 集成：从 RuntimeMcpState 到可调用工具

### RuntimeMcpState 的构建

在 [`rusty-claude-cli/src/main.rs#L2995-L3008`](/rust/crates/rusty-claude-cli/src/main.rs#L2995-L3008) 中，CLI 启动时会调用 `RuntimeMcpState::new`：

```rust
let mut manager = McpServerManager::from_runtime_config(runtime_config);
let runtime = tokio::runtime::Runtime::new()?;
let discovery = runtime.block_on(manager.discover_tools_best_effort());
```

若没有任何 MCP 服务器配置，函数返回 `None`，不会创建额外的 Tokio runtime。

###  discovered 工具注册到 CLI

`build_runtime_mcp_state`（[`main.rs#L3143-L3165`](/rust/crates/rusty-claude-cli/src/main.rs#L3143-L3165)）将每个 `ManagedMcpTool` 转换为 `RuntimeToolDefinition`，随后这些定义被注入到 `CliToolExecutor` 的工具注册表中。这意味着 MCP 工具和内置工具（如 `Bash`、`ReadFile`）在调用链路上完全等价。

### MCP 权限映射

MCP 工具的权限级别由 `tool.annotations` 决定（[`main.rs#L3647-L3665`](/rust/crates/rusty-claude-cli/src/main.rs#L3647-L3665)）：

```rust
fn permission_mode_for_mcp_tool(tool: &McpTool) -> PermissionMode {
    let read_only = mcp_annotation_flag(tool, "readOnlyHint");
    let destructive = mcp_annotation_flag(tool, "destructiveHint");
    let open_world = mcp_annotation_flag(tool, "openWorldHint");

    if read_only && !destructive && !open_world {
        PermissionMode::ReadOnly
    } else if destructive || open_world {
        PermissionMode::DangerFullAccess
    } else {
        PermissionMode::WorkspaceWrite
    }
}
```

这与上游 TypeScript 实现的注解映射一致：`readOnlyHint` 映射为只读权限，`destructiveHint` / `openWorldHint` 映射为最高危险权限。

### 包装工具

即使某个 MCP 工具因服务器暂时不可用未被发现，CLI 仍注册了 3 个通用包装工具（[`main.rs#L3171-L3201`](/rust/crates/rusty-claude-cli/src/main.rs#L3171-L3200)）：

- `MCPTool` —— 通过 qualified name 调用任意 MCP 工具（`DangerFullAccess`）
- `ListMcpResourcesTool` —— 列出 MCP 资源（`ReadOnly`）
- `ReadMcpResourceTool` —— 读取指定 MCP 资源（`ReadOnly`）

这三个工具的权限是固定的，不依赖具体服务器配置。

---

## MCP Server：claw 也可以作为 MCP 服务器运行

`claw-code` 不仅是 MCP 客户端，也能作为**最小化的 MCP stdio 服务器**运行。`McpServer` 定义于 [`mcp_server.rs#L66-L70`](/rust/crates/runtime/src/mcp_server.rs#L66-L70)：

```rust
pub struct McpServer {
    spec: McpServerSpec,
    stdin: BufReader<Stdin>,
    stdout: Stdout,
}
```

它实现了一个阻塞式的 read/dispatch/write 循环（[`mcp_server.rs#L86-L141`](/rust/crates/runtime/src/mcp_server.rs#L86-L141)），支持三个核心方法：

- `initialize` —— 返回协议版本 `"2025-03-26"` 和服务器信息（[`mcp_server.rs#L162-L177`](/rust/crates/runtime/src/mcp_server.rs#L162-L177)）
- `tools/list` —— 返回预注册的 `McpTool` 描述符列表（[`mcp_server.rs#L178-L188`](/rust/crates/runtime/src/mcp_server.rs#L178-L188)）
- `tools/call` —— 调用用户提供的 `ToolCallHandler`（[`mcp_server.rs#L190-L231`](/rust/crates/runtime/src/mcp_server.rs#L190-L230)）

`ToolCallHandler` 是一个同步闭包类型（[`mcp_server.rs#L41-L42`](/rust/crates/runtime/src/mcp_server.rs#L41-L42)）：

```rust
pub type ToolCallHandler =
    Box<dyn Fn(&str, &JsonValue) -> Result<String, String> + Send + Sync + 'static>;
```

OK 返回 `text` 内容块（`isError: false`），Err 返回 `text` 内容块（`isError: true`）。这个服务器可以被外部 MCP 客户端（如 Claude Desktop）驱动，也可以被 `claw` 自身的 `McpServerManager` 使用，因为它的帧格式与 `McpStdioProcess` 完全一致。

---

## 当前实现的边界与差异

### 仅支持 stdio 传输

`McpServerManager::from_servers` 显式过滤了所有非 `stdio` 服务器：

```rust
pub fn from_servers(servers: &BTreeMap<String, ScopedMcpServerConfig>) -> Self {
    let mut managed_servers = BTreeMap::new();
    let mut unsupported_servers = Vec::new();

    for (server_name, server_config) in servers {
        if server_config.transport() == McpTransport::Stdio {
            let bootstrap = McpClientBootstrap::from_scoped_config(server_name, server_config);
            managed_servers.insert(server_name.clone(), ManagedMcpServer::new(bootstrap));
        } else {
            unsupported_servers.push(UnsupportedMcpServer {
                server_name: server_name.clone(),
                transport: server_config.transport(),
                reason: format!(
                    "transport {:?} is not supported by McpServerManager",
                    server_config.transport()
                ),
            });
        }
    }
    // ...
}
```

参见 [`mcp_stdio.rs#L494-L512`](/rust/crates/runtime/src/mcp_stdio.rs#L494-L512)。这意味着配置中的 `sse`、`http`、`ws`、`sdk`、`claudeai-proxy` 服务器目前**不会建立连接**，只会生成降级报告。

### 无 LRU 缓存

上游 TypeScript 实现对 `fetchToolsForClient` 使用了 LRU(20) 缓存；当前 Rust 实现中没有工具缓存层。每次 `discover_tools` 都会重新走完 `initialize` + `tools/list`。作为补偿，`McpServerManager` 会复用已初始化的子进程，同一生命周期内的多次调用不会重复 spawn。

### 无信号升级策略

上游为 stdio 子进程设计了 `SIGINT → SIGTERM → SIGKILL` 的优雅关闭策略；Rust 实现目前直接使用 `child.kill().await`（发送 SIGKILL），没有中间信号降级过程。

### 无远程 OAuth 认证状态机

由于远程传输尚未支持，`ClaudeAuthProvider`、OAuth 缓存、`needs-auth` 状态机等均不在当前 `McpServerManager` 的路径中。`McpClientAuth::OAuth` 仅在 `McpClientBootstrap` 层被解析和保存，未被 stdio manager 消费。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/mcp_stdio.rs`](/rust/crates/runtime/src/mcp_stdio.rs) | `McpServerManager`、`McpStdioProcess`、JSON-RPC 帧协议、工具发现/调用/资源读写 |
| [`/rust/crates/runtime/src/mcp_client.rs`](/rust/crates/runtime/src/mcp_client.rs) | `McpClientTransport`、`McpClientBootstrap`、`McpStdioTransport`、超时配置 |
| [`/rust/crates/runtime/src/mcp.rs`](/rust/crates/runtime/src/mcp.rs) | `normalize_name_for_mcp`、`mcp_tool_name`、`mcp_server_signature`、`scoped_mcp_config_hash` |
| [`/rust/crates/runtime/src/mcp_server.rs`](/rust/crates/runtime/src/mcp_server.rs) | `McpServer`、`McpServerSpec`、`ToolCallHandler`：作为 MCP 服务器被外部驱动 |
| [`/rust/crates/runtime/src/mcp_tool_bridge.rs`](/rust/crates/runtime/src/mcp_tool_bridge.rs) | `McpToolRegistry`：状态ful 注册表，桥接工具表层与 `McpServerManager` |
| [`/rust/crates/runtime/src/mcp_lifecycle_hardened.rs`](/rust/crates/runtime/src/mcp_lifecycle_hardened.rs) | `McpLifecyclePhase`、`McpErrorSurface`、`McpDegradedReport` 生命周期硬化 |
| [`/rust/crates/runtime/src/config.rs`](/rust/crates/runtime/src/config.rs) | `McpServerConfig` 枚举、`McpStdioServerConfig`、`McpRemoteServerConfig`、配置解析 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | `RuntimeMcpState`、MCP 工具包装定义、权限映射、CLI 启动时 MCP 初始化 |
| [`/rust/crates/runtime/src/lib.rs`](/rust/crates/runtime/src/lib.rs) | runtime crate 的 MCP 模块与类型重导出 |
