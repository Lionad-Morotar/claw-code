# Unit 57: LSP Integration 技术报告

**Source URL**: https://ccb.agent-aura.top/docs/lsp-integration  
**Report Generated**: 2026-04-09  
**Unit**: 57-lsp-integration  

---

## 概述

claw-code 对 LSP 的支持目前停留在**骨架层**：`rust/crates/runtime/src/lsp_client.rs` 实现了注册表、分发接口和测试占位，`rust/crates/tools/src/lib.rs` 暴露了 `LSP` 工具入口，`rust/crates/runtime/src/mcp_stdio.rs` 提供了 LSP 兼容的 JSON-RPC 帧读写能力。实际的 language server 进程启停、文档同步、请求/响应翻译尚未实现。以下梳理现有源码结构与限制。

---

## 架构概览

```
┌─────────────────────────────────────────────────────────────────┐
│                      claw CLI (main.rs)                         │
│                         execute_tool()                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    tools/src/lib.rs                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   ToolSpec      │  │   run_lsp()     │  │   LspInput      │  │
│  │   "LSP"         │──│   #L1606-L1620  │──│   #L2303-L2312  │  │
│  └─────────────────┘  └────────┬────────┘  └─────────────────┘  │
└─────────────────────────────────┼────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                  runtime/src/lsp_client.rs                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  LspRegistry    │  │  LspAction      │  │  LspServerState │  │
│  │  #L110-L112     │  │  #L12-L20       │  │  #L101-L107     │  │
│  └────────┬────────┘  └─────────────────┘  └─────────────────┘  │
│           │                                                     │
│           ▼ dispatch() #L236-L296                               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  • Diagnostics: return cached results                    │    │
│  │  • Other actions: require connected server               │    │
│  │  • File extension → language mapping                     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ (future: actual LSP JSON-RPC)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              runtime/mcp_stdio.rs - LSP Framing                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  McpStdioProcess│  │  read_frame()   │  │  write_frame()  │  │
│  │  #L1143-L1148   │  │  #L1215-L1247   │  │  #L1209-L1213   │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                   │
│  LSP Frame Format: Content-Length header + JSON-RPC body          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 组件分析

### 1. LSP Registry (`rust/crates/runtime/src/lsp_client.rs`)

#### 核心数据结构

**LspAction** ([`lsp_client.rs#L12-L20`](rust/crates/runtime/src/lsp_client.rs#L12-L20))
```rust
pub enum LspAction {
    Diagnostics, Hover, Definition, References,
    Completion, Symbols, Format,
}
```

字符串解析与别名 ([`#L23-L35`](rust/crates/runtime/src/lsp_client.rs#L23-L35))：
- `definition` / `goto_definition` → `LspAction::Definition`
- `references` / `find_references` → `LspAction::References`
- `completion` / `completions` → `LspAction::Completion`
- `symbols` / `document_symbols` → `LspAction::Symbols`
- `format` / `formatting` → `LspAction::Format`

**LspServerState** ([`lsp_client.rs#L101-L107`](rust/crates/runtime/src/lsp_client.rs#L101-L107))
```rust
pub struct LspServerState {
    pub language: String,
    pub status: LspServerStatus,
    pub root_path: Option<String>,
    pub capabilities: Vec<String>,
    pub diagnostics: Vec<LspDiagnostic>,
}
```

**LspRegistry** ([`lsp_client.rs#L110-L112`](rust/crates/runtime/src/lsp_client.rs#L110-L112))
- 使用 `Arc<Mutex<RegistryInner>>` 的线程安全注册表
- 未实现多语言 server 的并发生命周期管理

#### 关键方法

**register()** ([`#L125-L143`](rust/crates/runtime/src/lsp_client.rs#L125-L143))
注册语言 server 的能力声明。

**find_server_for_path()** ([`#L151-L172`](rust/crates/runtime/src/lsp_client.rs#L151-L172))
按扩展名映射语言 server：
| 扩展名 | 语言 |
|--------|------|
| `.rs` | rust |
| `.ts`, `.tsx` | typescript |
| `.js`, `.jsx` | javascript |
| `.py` | python |
| `.go` | go |
| `.java` | java |
| `.c`, `.h` | c |
| `.cpp`, `.hpp`, `.cc` | cpp |
| `.rb` | ruby |
| `.lua` | lua |

**dispatch()** ([`#L236-L296`](rust/crates/runtime/src/lsp_client.rs#L236-L296))
核心分发逻辑，当前仅返回占位数据：

1. **Diagnostics 路径** ([`#L248-L270`](rust/crates/runtime/src/lsp_client.rs#L248-L270)): 直接返回缓存的 diagnostics，不要求 live server
2. **Action 路径** ([`#L272-L295`](rust/crates/runtime/src/lsp_client.rs#L272-L295)): 仅检查 server 存在且处于 `Connected` 状态，随后返回结构化占位

```rust
// Lines 285-295 - 结构化占位响应
Ok(serde_json::json!({
    "action": action,
    "path": path,
    "line": line,
    "character": character,
    "language": server.language,
    "status": "dispatched",
    "message": format!("LSP {} dispatched to {} server", action, server.language)
}))
```

**测试覆盖** ([`#L299-L747`](rust/crates/runtime/src/lsp_client.rs#L299-L747))
- 20+ 单元测试覆盖注册、分发、diagnostics 聚合与错误路径
- `dispatch_diagnostics_without_path_aggregates()` ([`#L486-L527`](rust/crates/runtime/src/lsp_client.rs#L486-L527)) 验证多 server 聚合

---

### 2. LSP Tool 接口 (`rust/crates/tools/src/lib.rs`)

#### Tool 定义 ([`lib.rs#L1052-L1071`](rust/crates/tools/src/lib.rs#L1052-L1071))

```rust
ToolSpec {
    name: "LSP",
    description: "Query Language Server Protocol for code intelligence (symbols, references, diagnostics).",
    input_schema: json!({
        "type": "object",
        "properties": {
            "action": { "type": "string", "enum": ["symbols", "references", "diagnostics", "definition", "hover"] },
            "path": { "type": "string" },
            "line": { "type": "integer", "minimum": 0 },
            "character": { "type": "integer", "minimum": 0 },
            "query": { "type": "string" }
        },
        "required": ["action"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::ReadOnly,
}
```

#### 输入结构 ([`lib.rs#L2303-L2312`](rust/crates/tools/src/lib.rs#L2303-L2312))

```rust
#[derive(Debug, Deserialize)]
struct LspInput {
    action: String,
    #[serde(default)]
    path: Option<String>,
    #[serde(default)]
    line: Option<u32>,
    #[serde(default)]
    character: Option<u32>,
    #[serde(default)]
    query: Option<String>,
}
```

#### 执行处理函数 ([`lib.rs#L1606-L1620`](rust/crates/tools/src/lib.rs#L1606-L1620))

```rust
fn run_lsp(input: LspInput) -> Result<String, String> {
    let registry = global_lsp_registry();  // [`lib.rs#L35-L42`](rust/crates/tools/src/lib.rs#L35-L42)
    let action = &input.action;
    let path = input.path.as_deref();
    let line = input.line;
    let character = input.character;
    let query = input.query.as_deref();

    match registry.dispatch(action, path, line, character, query) {
        Ok(result) => to_pretty_json(result),
        Err(e) => to_pretty_json(json!({
            "action": action,
            "error": e,
            "status": "error"
        })),
    }
}
```

#### Tool 调度集成 ([`lib.rs#L1254`](rust/crates/tools/src/lib.rs#L1254))

```rust
"LSP" => from_value::<LspInput>(input).and_then(run_lsp),
```

---

### 3. MCP stdio Transport (`rust/crates/runtime/src/mcp_stdio.rs`)

MCP stdio 实现复用了 LSP 的 `Content-Length` 帧格式，说明底层传输层已具备与 LSP server 对话的能力，但上层 LSP 生命周期管理尚未接入。

**帧结构** ([`mcp_stdio.rs#L1209-L1247`](rust/crates/runtime/src/mcp_stdio.rs#L1209-L1247))
```
Content-Length: <byte-count>\r\n
\r\n
<JSON-RPC payload>
```

**McpStdioProcess** ([`mcp_stdio.rs#L1143-L1148`](rust/crates/runtime/src/mcp_stdio.rs#L1143-L1148))
```rust
pub struct McpStdioProcess {
    child: Child,
    stdin: ChildStdin,
    stdout: BufReader<ChildStdout>,
}
```

**write_frame()** ([`mcp_stdio.rs#L1209-L1213`](rust/crates/runtime/src/mcp_stdio.rs#L1209-L1213))
```rust
pub async fn write_frame(&mut self, payload: &[u8]) -> io::Result<()> {
    let encoded = encode_frame(payload);
    self.write_all(&encoded).await?;
    self.flush().await
}
```

**read_frame()** ([`mcp_stdio.rs#L1215-L1247`](rust/crates/runtime/src/mcp_stdio.rs#L1215-L1247))
```rust
pub async fn read_frame(&mut self) -> io::Result<Vec<u8>> {
    let mut content_length = None;
    loop {
        let mut line = String::new();
        let bytes_read = self.stdout.read_line(&mut line).await?;
        if bytes_read == 0 {
            return Err(io::Error::new(
                io::ErrorKind::UnexpectedEof,
                "MCP stdio stream closed while reading headers",
            ));
        }
        if line == "\r\n" {
            break;
        }
        let header = line.trim_end_matches(['\r', '\n']);
        if let Some((name, value)) = header.split_once(':') {
            if name.trim().eq_ignore_ascii_case("Content-Length") {
                let parsed = value.trim().parse::<usize>()?;
                content_length = Some(parsed);
            }
        }
    }
    let content_length = content_length.ok_or_else(|| ...)?;
    let mut payload = vec![0_u8; content_length];
    self.stdout.read_exact(&mut payload).await?;
    Ok(payload)
}
```

**JSON-RPC Message 处理** ([`mcp_stdio.rs#L1249-L1267`](rust/crates/runtime/src/mcp_stdio.rs#L1249-L1267))
```rust
pub async fn write_jsonrpc_message<T: Serialize>(&mut self, message: &T) -> io::Result<()> {
    let body = serde_json::to_vec(message)?;
    self.write_frame(&body).await
}

pub async fn read_jsonrpc_message<T: DeserializeOwned>(&mut self) -> io::Result<T> {
    let payload = self.read_frame().await?;
    serde_json::from_slice(&payload).map_err(...)
}
```

**spawn_mcp_stdio_process()** ([`mcp_stdio.rs#L1371-L1385`](rust/crates/runtime/src/mcp_stdio.rs#L1371-L1385))
```rust
pub fn spawn_mcp_stdio_process(bootstrap: &McpClientBootstrap) -> io::Result<McpStdioProcess> {
    match &bootstrap.transport {
        McpClientTransport::Stdio(transport) => McpStdioProcess::spawn(transport),
        other => Err(io::Error::new(
            io::ErrorKind::InvalidInput,
            format!("expected stdio transport, got: {other:?}"),
        )),
    }
}
```

---

## 集成点

### Global Registry Pattern

**tools/src/lib.rs** ([`lib.rs#L35-L42`](rust/crates/tools/src/lib.rs#L35-L42)):
```rust
fn global_lsp_registry() -> &'static LspRegistry {
    use std::sync::OnceLock;
    static REGISTRY: OnceLock<LspRegistry> = OnceLock::new();
    REGISTRY.get_or_init(LspRegistry::new)
}
```

### 权限模型

`LSP` 工具要求 `PermissionMode::ReadOnly` ([`lib.rs#L1071`](rust/crates/tools/src/lib.rs#L1071))。

---

## 当前限制

1. **占位分发** ([`lsp_client.rs#L285-L295`](rust/crates/runtime/src/lsp_client.rs#L285-L295))：
   `dispatch()` 目前返回固定格式的 JSON 占位，未触发任何真实 LSP JSON-RPC 调用。

2. **无 live server 生命周期管理**：
   `lsp_client.rs` 尚未实现启动 `rust-analyzer`、`typescript-language-server` 等进程、维护连接、发送 `textDocument/didOpen` / `didChange`。

3. **Diagnostics 为手动缓存**：
   diagnostics 通过 `add_diagnostics()` ([`#L181-L193`](rust/crates/runtime/src/lsp_client.rs#L181-L193)) 写入，而非从 live server 拉取。

---

## 待实现路径

若要将 LSP 支持真正落地，需补齐：

1. **LSP Server Spawner**：基于 `McpStdioProcess` 启动 language server 进程
2. **LSP Session Manager**：追踪已打开文档，发送 didOpen/didChange
3. **Request Router**：映射 `LspAction` 到 JSON-RPC 方法
   - `hover` → `textDocument/hover`
   - `definition` → `textDocument/definition`
   - `references` → `textDocument/references`
   - `completion` → `textDocument/completion`
   - `symbols` → `textDocument/documentSymbol`
   - `format` → `textDocument/formatting`
4. **Response Parser**：将 LSP JSON 响应转为 Rust 结构化类型

---

## 源码索引

| 组件 | 文件 | 行号 |
|------|------|------|
| LspAction enum | [`lsp_client.rs`](rust/crates/runtime/src/lsp_client.rs) | [L12-L20](rust/crates/runtime/src/lsp_client.rs#L12-L20) |
| LspAction::from_str | [`lsp_client.rs`](rust/crates/runtime/src/lsp_client.rs) | [L23-L35](rust/crates/runtime/src/lsp_client.rs#L23-L35) |
| LspServerState | [`lsp_client.rs`](rust/crates/runtime/src/lsp_client.rs) | [L101-L107](rust/crates/runtime/src/lsp_client.rs#L101-L107) |
| LspRegistry | [`lsp_client.rs`](rust/crates/runtime/src/lsp_client.rs) | [L110-L112](rust/crates/runtime/src/lsp_client.rs#L110-L112) |
| LspRegistry::dispatch | [`lsp_client.rs`](rust/crates/runtime/src/lsp_client.rs) | [L236-L296](rust/crates/runtime/src/lsp_client.rs#L236-L296) |
| LspRegistry tests | [`lsp_client.rs`](rust/crates/runtime/src/lsp_client.rs) | [L299-L747](rust/crates/runtime/src/lsp_client.rs#L299-L747) |
| global_lsp_registry | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L35-L42](rust/crates/tools/src/lib.rs#L35-L42) |
| LSP ToolSpec | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L1052-L1071](rust/crates/tools/src/lib.rs#L1052-L1071) |
| LspInput struct | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L2303-L2312](rust/crates/tools/src/lib.rs#L2303-L2312) |
| run_lsp handler | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L1606-L1620](rust/crates/tools/src/lib.rs#L1606-L1620) |
| Tool dispatch | [`lib.rs`](rust/crates/tools/src/lib.rs) | [L1254](rust/crates/tools/src/lib.rs#L1254) |
| McpStdioProcess | [`mcp_stdio.rs`](rust/crates/runtime/src/mcp_stdio.rs) | [L1143-L1148](rust/crates/runtime/src/mcp_stdio.rs#L1143-L1148) |
| read_frame | [`mcp_stdio.rs`](rust/crates/runtime/src/mcp_stdio.rs) | [L1215-L1247](rust/crates/runtime/src/mcp_stdio.rs#L1215-L1247) |
| write_frame | [`mcp_stdio.rs`](rust/crates/runtime/src/mcp_stdio.rs) | [L1209-L1213](rust/crates/runtime/src/mcp_stdio.rs#L1209-L1213) |
| spawn_mcp_stdio_process | [`mcp_stdio.rs`](rust/crates/runtime/src/mcp_stdio.rs) | [L1371-L1385](rust/crates/runtime/src/mcp_stdio.rs#L1371-L1385) |

---

## Parity 文档参考

`rust/PARITY.md` ([`PARITY.md#L73`](rust/PARITY.md#L73)):

> | **LSP** | `runtime::lsp_client` + `tools` | registry + dispatch for diagnostics, hover, definition, references, completion, symbols, formatting — **good parity** |

该描述与实际代码一致：注册表与分发层已具备测试与线程安全结构，但 live LSP server 通信仍标记为未来工作。

---

## 结论

claw-code 的 LSP 相关代码提供了**结构清晰但功能未完整**的基础：
- **Registry 层** (`lsp_client.rs`)：server 状态管理与分发占位逻辑已就绪
- **Tool 层** (`tools/src/lib.rs`)：模型调用入口已暴露
- **Transport 层** (`mcp_stdio.rs`)：LSP 兼容的 JSON-RPC 帧读写已具备

距离真正支持 live LSP server connections，仍需实现 server spawner、session manager 和 JSON-RPC request router。

---

<div align="center">

**Unit 57 Done** — `docs/.report/57-lsp-integration.md`

</div>
