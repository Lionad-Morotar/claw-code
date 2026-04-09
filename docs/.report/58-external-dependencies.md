# Unit 58: External Dependencies

**Source**: https://ccb.agent-aura.top/docs/external-dependencies  
**Output**: `docs/.report/58-external-dependencies.md`  
**Date**: 2026-04-09

---

## 1. Original Page Summary

The CCB documentation page describes Claude Code's external network dependencies - remote services that the CLI connects to during operation. The page lists 20+ remote endpoints including:

- **LLM Providers**: Anthropic API, AWS Bedrock, Google Vertex AI, Azure Foundry
- **Authentication**: OAuth (platform.claude.com, claude.ai)
- **Telemetry**: GrowthBook, Sentry, Datadog, OpenTelemetry
- **MCP Services**: MCP Proxy, MCP Registry
- **Other**: Bing Search, GitHub Raw, Google Cloud Storage

---

## 2. claw-code External Dependency Analysis

claw-code 是一个 Rust 实现的 Claude Code 兼容客户端。与原始 Claude Code (TypeScript/Node.js) 不同，claw-code 使用 Rust 生态系统，其外部依赖通过 Cargo 管理。

### 2.1 依赖统计

- **总 crates 数量**: 215 个 (包含传递依赖)
- **直接依赖 crates**: ~20 个核心外部 crate
- **内部 crates**: 10 个 (api, commands, runtime, plugins, tools, telemetry 等)

### 2.2 核心外部依赖

#### 2.2.1 HTTP 客户端 - `reqwest`

**依赖声明** (`rust/crates/api/Cargo.toml:9`):
```toml
reqwest = { version = "0.12", default-features = false, features = ["json", "rustls-tls"] }
```

**用途**: 构建 HTTP 客户端，支持与 Anthropic API 等远程服务通信

**源码位置**:
- `rust/crates/api/src/http_client.rs:63-113` - `build_http_client()` 函数
- 支持代理配置 (`HTTP_PROXY`, `HTTPS_PROXY`, `NO_PROXY`)
- 使用 `rustls-tls` 提供 HTTPS 支持

**关键实现** (`rust/crates/api/src/http_client.rs:83-112`):
```rust
pub fn build_http_client_with(config: &ProxyConfig) -> Result<reqwest::Client, ApiError> {
    let mut builder = reqwest::Client::builder().no_proxy();
    
    let no_proxy = config.no_proxy.as_deref().and_then(reqwest::NoProxy::from_string);
    
    // 配置 HTTPS/HTTP 代理...
    Ok(builder.build()?)
}
```

#### 2.2.2 异步运行时 - `tokio`

**依赖声明** (多个 crate):
- `rust/crates/api/Cargo.toml:14`: `tokio = { version = "1", features = ["io-util", "macros", "net", "rt-multi-thread", "time"] }`
- `rust/crates/runtime/Cargo.toml:16`: 添加 `io-std`, `process`, `rt`, `signal`
- `rust/crates/rusty-claude-cli/Cargo.toml:24`: 添加 `signal`

**用途**: 提供异步 I/O、网络通信、信号处理

**源码位置**:
- `rust/crates/mock-anthropic-service/src/main.rs:5` - `#[tokio::main]` 入口
- `rust/crates/mock-anthropic-service/src/lib.rs:8-10` - TCP 监听器实现

#### 2.2.3 序列化 - `serde` / `serde_json`

**依赖声明**:
- `rust/Cargo.toml:12`: `serde_json.workspace = true`
- `rust/crates/api/Cargo.toml:11`: `serde = { version = "1", features = ["derive"] }`

**用途**: JSON 序列化/反序列化，用于 API 请求/响应

**源码位置**:
- `rust/crates/telemetry/src/lib.rs:9-10`: `use serde::{Deserialize, Serialize}; use serde_json::{Map, Value};`
- `rust/crates/runtime/src/lane_events.rs:2-3`: 事件类型序列化
- `rust/crates/runtime/src/sandbox.rs:5`: 沙箱配置序列化

#### 2.2.4 CLI 交互 - `crossterm`, `rustyline`, `syntect`, `pulldown-cmark`

**依赖声明** (`rust/crates/rusty-claude-cli/Cargo.toml:16-23`):
```toml
crossterm = "0.28"
pulldown-cmark = "0.13"
rustyline = "15"
syntect = "5"
```

**用途**:
- `crossterm`: 终端跨平台操作
- `rustyline`: 命令行输入处理、历史、补全
- `syntect`: 语法高亮
- `pulldown-cmark`: Markdown 解析

**源码位置**:
- `rust/crates/rusty-claude-cli/src/input.rs:6-13`: rustyline 集成
  - `Completer`, `Highlighter`, `Validator` 实现
  - 行编辑器 (`LineEditor`) 实现

#### 2.2.5 文件处理 - `flate2`, `walkdir`, `glob`

**依赖声明**:
- `rust/crates/tools/Cargo.toml:11`: `flate2 = "1"`
- `rust/crates/runtime/Cargo.toml:10,17`: `glob = "0.3"`, `walkdir = "2"`

**用途**:
- `flate2`: PDF 解压等压缩处理
- `glob`: 文件模式匹配
- `walkdir`: 目录遍历

**源码位置**:
- `rust/crates/tools/src/pdf_extract.rs:363-365`: ZlibEncoder 使用

#### 2.2.6 加密/哈希 - `sha2`, `regex`

**依赖声明** (`rust/crates/runtime/Cargo.toml:9,12`):
```toml
sha2 = "0.10"
regex = "1"
```

**用途**: SHA-2 哈希、正则表达式匹配

---

## 3. 依赖关系图

```
rusty-claude-cli (bin)
├── api
│   ├── reqwest (HTTP client)
│   ├── serde/serde_json
│   └── tokio
├── commands
│   ├── plugins
│   └── runtime
├── compat-harness
│   ├── commands
│   └── tools
├── tools
│   ├── api
│   ├── flate2
│   ├── reqwest
│   └── serde/serde_json
├── runtime
│   ├── glob
│   ├── plugins
│   ├── regex
│   ├── serde/serde_json
│   ├── sha2
│   ├── telemetry
│   ├── tokio
│   └── walkdir
├── plugins
│   └── serde/serde_json
├── telemetry
│   └── serde/serde_json
└── CLI deps
    ├── crossterm
    ├── pulldown-cmark
    ├── rustyline
    └── syntect
```

---

## 4. 与原始 Claude Code 对比

| 依赖类别 | Claude Code (TS) | claw-code (Rust) |
|---------|------------------|------------------|
| HTTP 客户端 | 原生 fetch API | reqwest |
| JSON 处理 | 原生 JSON | serde_json |
| 异步运行时 | Node.js event loop | tokio |
| 终端 UI | 无 (依赖终端) | crossterm |
| 命令行输入 | readline | rustyline |
| Markdown | marked/remark | pulldown-cmark |
| 语法高亮 | Prism/Highlight.js | syntect |

---

## 5. 网络端点实现

claw-code 通过网络与以下端点通信：

### 5.1 Anthropic API

**端点**: `https://api.anthropic.com` (可配置)

**源码**: `rust/crates/api/src/` (未完全展示，但 http_client 提供支持)

### 5.2 代理支持

**源码**: `rust/crates/api/src/http_client.rs:3-21` - `ProxyConfig` 结构体
- 支持 `HTTP_PROXY`/`http_proxy`
- 支持 `HTTPS_PROXY`/`https_proxy`
- 支持 `NO_PROXY`/`no_proxy`
- 支持统一 `proxy_url` 配置

---

## 6. 源码锚点汇总

| 文件 | 行号 | 说明 |
|------|------|------|
| `rust/Cargo.toml` | 1-22 | Workspace 配置 |
| `rust/crates/api/Cargo.toml` | 1-17 | API crate 依赖 |
| `rust/crates/api/src/http_client.rs` | 63-113 | HTTP 客户端构建 |
| `rust/crates/api/src/http_client.rs` | 3-21 | ProxyConfig 定义 |
| `rust/crates/runtime/Cargo.toml` | 1-20 | Runtime crate 依赖 |
| `rust/crates/rusty-claude-cli/Cargo.toml` | 1-34 | CLI crate 依赖 |
| `rust/crates/rusty-claude-cli/src/input.rs` | 6-13 | rustyline 集成 |
| `rust/crates/telemetry/src/lib.rs` | 9-10 | serde 使用 |
| `rust/crates/tools/src/pdf_extract.rs` | 363-365 | flate2 使用 |
| `rust/Cargo.lock` | 全文 | 215 个 crate 锁定 |

---

## 7. 结论

claw-code 作为 Rust 实现的 Claude Code 客户端，其外部依赖管理与原始 TypeScript 版本有显著不同：

1. **更少的远程依赖**: 所有 Rust crate 来自 crates.io，无运行时远程配置
2. **编译时锁定**: Cargo.lock 确保依赖版本确定性
3. **原生性能**: tokio + reqwest 提供高性能异步 HTTP
4. **终端体验**: crossterm + rustyline + syntect 提供本地终端交互

与 CCB 文档描述的 20+ 远程端点不同，claw-code 主要依赖：
- **crates.io**: 包注册表
- **api.anthropic.com**: LLM API (与原始相同)
- **可选代理**: 通过环境变量配置

---

*Generated by Unit 58 - External Dependencies Analysis*
