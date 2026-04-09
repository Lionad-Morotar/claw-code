# Unit 57: LSP Integration Technical Report

**Source URL**: https://ccb.agent-aura.top/docs/lsp-integration  
**Report Generated**: 2026-04-09  
**Unit**: 57-lsp-integration

---

## Executive Summary

The claw-code codebase implements a comprehensive Language Server Protocol (LSP) integration layer that provides code intelligence capabilities including diagnostics, hover, definition lookup, references, completion, symbols, and formatting. The implementation consists of three primary components:

1. **LSP Registry** (`lsp_client.rs`) - Server registration and dispatch
2. **LSP Tool** (`tools/src/lib.rs`) - Tool interface for model access
3. **MCP stdio Transport** (`mcp_stdio.rs`) - LSP-framed JSON-RPC communication

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      claw CLI (main.rs)                         │
│                         execute_tool()                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    tools/src/lib.rs                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │   ToolSpec      │  │   run_lsp()     │  │   LspInput      │ │
│  │   "LSP"         │──│   #L1606-L1622  │  │   #L2303-L2313  │ │
│  └─────────────────┘  └────────┬────────┘  └─────────────────┘ │
└─────────────────────────────────┼────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                  runtime/src/lsp_client.rs                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  LspRegistry    │  │  LspAction      │  │  LspServerState │ │
│  │  #L110-L112     │  │  #L12-L20       │  │  #L101-L107     │ │
│  └────────┬────────┘  └─────────────────┘  └─────────────────┘ │
│           │                                                     │
│           ▼ dispatch() #L236-L296                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  • Diagnostics: return cached results                    │   │
│  │  • Other actions: require connected server               │   │
│  │  • File extension → language mapping                     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ (future: actual LSP JSON-RPC)
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              runtime/mcp_stdio.rs - LSP Framing                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  McpStdioProcess│  │  read_frame()   │  │  write_frame()  │ │
│  │  #L1143-L1148   │  │  #L1215-L1247   │  │  #L1209-L1213   │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                 │
│  LSP Frame Format: Content-Length header + JSON-RPC body        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Component Analysis

### 1. LSP Registry (`rust/crates/runtime/src/lsp_client.rs`)

#### Core Data Structures

**LspAction** (#L12-L20)
```rust
pub enum LspAction {
    Diagnostics, Hover, Definition, References,
    Completion, Symbols, Format,
}
```

String parsing with aliases (#L23-L35):
- `definition` / `goto_definition` → `LspAction::Definition`
- `references` / `find_references` → `LspAction::References`
- `completion` / `completions` → `LspAction::Completion`
- `symbols` / `document_symbols` → `LspAction::Symbols`
- `format` / `formatting` → `LspAction::Format`

**LspServerState** (#L101-L107)
```rust
pub struct LspServerState {
    pub language: String,
    pub status: LspServerStatus,  // Connected, Disconnected, Starting, Error
    pub root_path: Option<String>,
    pub capabilities: Vec<String>,
    pub diagnostics: Vec<LspDiagnostic>,
}
```

**LspRegistry** (#L110-L112)
- Thread-safe registry using `Arc<Mutex<RegistryInner>>`
- Supports multiple language servers simultaneously

#### Key Methods

**register()** (#L125-L143)
Registers a language server with its capabilities.

**find_server_for_path()** (#L151-L172)
Maps file extensions to language servers:
| Extension | Language |
|-----------|----------|
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

**dispatch()** (#L236-L296)
Core dispatch logic with two paths:

1. **Diagnostics path** (#L248-L270): Returns cached diagnostics without requiring live LSP connection
2. **Action path** (#L272-L295): Requires connected server, returns structured placeholder

```rust
// Lines 285-295 - Placeholder response structure
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

**Test Coverage** (#L299-L747)
- 20+ unit tests covering registration, dispatch, diagnostics, and error cases
- Test `dispatch_diagnostics_without_path_aggregates()` (#L486-L527) validates multi-server aggregation

---

### 2. LSP Tool Interface (`rust/crates/tools/src/lib.rs`)

#### Tool Definition (#L1056-L1072)

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

#### Input Structure (#L2303-L2313)

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

#### Execution Handler (#L1606-L1622)

```rust
fn run_lsp(input: LspInput) -> Result<String, String> {
    let registry = global_lsp_registry();  // #L35-L39: static OnceLock<LspRegistry>
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

#### Tool Dispatch Integration (#L1254)

```rust
"LSP" => from_value::<LspInput>(input).and_then(run_lsp),
```

---

### 3. MCP stdio Transport (`rust/crates/runtime/src/mcp_stdio.rs`)

#### LSP Frame Format

The MCP stdio implementation uses LSP-compatible framing with `Content-Length` headers:

**Frame Structure** (#L1209-L1247):
```
Content-Length: <byte-count>\r\n
\r\n
<JSON-RPC payload>
```

#### McpStdioProcess (#L1143-L1148)

```rust
pub struct McpStdioProcess {
    child: Child,
    stdin: ChildStdin,
    stdout: BufReader<ChildStdout>,
}
```

**write_frame()** (#L1209-L1213)
```rust
pub async fn write_frame(&mut self, payload: &[u8]) -> io::Result<()> {
    let encoded = encode_frame(payload);
    self.write_all(&encoded).await?;
    self.flush().await
}
```

**read_frame()** (#L1215-L1247)
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
                let parsed = value.trim().parse::<usize>()...;
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

**JSON-RPC Message Handling** (#L1249-L1267)
```rust
pub async fn write_jsonrpc_message<T: Serialize>(&mut self, message: &T) -> io::Result<()> {
    let body = serde_json::to_vec(message)...;
    self.write_frame(&body).await
}

pub async fn read_jsonrpc_message<T: DeserializeOwned>(&mut self) -> io::Result<T> {
    let payload = self.read_frame().await?;
    serde_json::from_slice(&payload)...
}
```

#### spawn_mcp_stdio_process() (#L1371-L1385)

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

## Integration Points

### Global Registry Pattern

**tools/src/lib.rs** (#L35-L39):
```rust
fn global_lsp_registry() -> &'static LspRegistry {
    use std::sync::OnceLock;
    static REGISTRY: OnceLock<LspRegistry> = OnceLock::new();
    REGISTRY.get_or_init(LspRegistry::new)
}
```

### Permission Model

The LSP tool requires `PermissionMode::ReadOnly` (#L1071), ensuring LSP queries cannot modify filesystem state.

---

## Current Limitations

### 1. Placeholder Dispatch (#L285-#L295)

The current `dispatch()` implementation returns structured placeholders rather than actual LSP JSON-RPC calls:

```rust
// rust/crates/runtime/src/lsp_client.rs:285-295
// Return structured placeholder — actual LSP JSON-RPC calls would
// go through the real LSP process here.
```

### 2. No Live LSP Server Management

The `lsp_client.rs` registry does not:
- Spawn actual language server processes (rust-analyzer, typescript-language-server, etc.)
- Maintain live JSON-RPC connections
- Handle LSP protocol messages (textDocument/didOpen, textDocument/didChange, etc.)

### 3. Cached Diagnostics Only

Diagnostics are manually added via `add_diagnostics()` (#L181-L193) rather than pulled from live LSP servers.

---

## Future Implementation Path

To complete the LSP integration, the following components would need implementation:

1. **LSP Server Spawner**: Wrapper around `McpStdioProcess` for launching language servers
2. **LSP Session Manager**: Track open documents, send didOpen/didChange notifications
3. **Request Router**: Map `LspAction` to JSON-RPC methods:
   - `hover` → `textDocument/hover`
   - `definition` → `textDocument/definition`
   - `references` → `textDocument/references`
   - `completion` → `textDocument/completion`
   - `symbols` → `textDocument/documentSymbol`
   - `format` → `textDocument/formatting`
4. **Response Parser**: Convert LSP JSON responses to structured Rust types

---

## Source File Reference

| Component | File | Lines |
|-----------|------|-------|
| LspAction enum | `rust/crates/runtime/src/lsp_client.rs` | #L12-L20 |
| LspAction::from_str | `rust/crates/runtime/src/lsp_client.rs` | #L23-L35 |
| LspServerState | `rust/crates/runtime/src/lsp_client.rs` | #L101-L107 |
| LspRegistry | `rust/crates/runtime/src/lsp_client.rs` | #L110-L112 |
| LspRegistry::dispatch | `rust/crates/runtime/src/lsp_client.rs` | #L236-L296 |
| LspRegistry tests | `rust/crates/runtime/src/lsp_client.rs` | #L299-L747 |
| global_lsp_registry | `rust/crates/tools/src/lib.rs` | #L35-L39 |
| LSP ToolSpec | `rust/crates/tools/src/lib.rs` | #L1056-L1072 |
| LspInput struct | `rust/crates/tools/src/lib.rs` | #L2303-L2313 |
| run_lsp handler | `rust/crates/tools/src/lib.rs` | #L1606-L1622 |
| Tool dispatch | `rust/crates/tools/src/lib.rs` | #L1254 |
| McpStdioProcess | `rust/crates/runtime/src/mcp_stdio.rs` | #L1143-L1148 |
| read_frame | `rust/crates/runtime/src/mcp_stdio.rs` | #L1215-L1247 |
| write_frame | `rust/crates/runtime/src/mcp_stdio.rs` | #L1209-L1213 |
| spawn_mcp_stdio_process | `rust/crates/runtime/src/mcp_stdio.rs` | #L1371-L1385 |

---

## Parity Documentation Reference

From `rust/PARITY.md` (#L73):

> | **LSP** | `runtime::lsp_client` + `tools` | registry + dispatch for diagnostics, hover, definition, references, completion, symbols, formatting — **good parity** |

This indicates the current implementation achieves "good parity" for the registry and dispatch layer, with actual LSP protocol communication marked as future work.

---

## Conclusion

The LSP integration in claw-code provides a well-structured foundation for language server capabilities. The architecture cleanly separates:
- **Registry layer** (`lsp_client.rs`): Server state management and dispatch logic
- **Tool layer** (`tools/src/lib.rs`): Model-facing interface with permission controls
- **Transport layer** (`mcp_stdio.rs`): LSP-compatible JSON-RPC framing

The implementation is testable (20+ unit tests), thread-safe, and ready for extension to support live LSP server connections.

---

<div align="center">

**Unit 57 Done** — `docs/.report/57-lsp-integration.md`

</div>
