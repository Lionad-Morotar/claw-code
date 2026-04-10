# 搜索与导航工具 - 代码库精准定位

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/tools/search-and-navigation) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 两种搜索维度

在大型代码库中，AI 需要快速完成两类定位任务：

| 维度 | 工具 | 底层实现 | 适用场景 |
| --- | --- | --- | --- |
| 按名称找文件 | Glob | `glob` crate 模式匹配 | "找到所有测试文件"、"找 `config` 开头的文件" |
| 按内容找代码 | Grep | `regex` + `walkdir` 遍历 | "哪里定义了这个函数"、"谁在调用这个 API" |

在 `claw-code` 的 Rust 实现中，这两个工具不是外部进程的简单透传，而是内建在同一套文件系统安全边界内的原生函数调用。

### 源码映射：Glob 与 Grep 的 Schema 定义

`Glob` 和 `Grep` 的输入输出结构定义在 `runtime/src/file_ops.rs`：

- `GrepSearchInput`（[`L132-L154`](/rust/crates/runtime/src/file_ops.rs#L132-L154)）:

```rust
pub struct GrepSearchInput {
    pub pattern: String,
    pub path: Option<String>,
    pub glob: Option<String>,
    pub output_mode: Option<String>,
    pub before: Option<usize>,
    pub after: Option<usize>,
    pub context_short: Option<usize>,
    pub context: Option<usize>,
    pub line_numbers: Option<bool>,
    pub case_insensitive: Option<bool>,
    pub file_type: Option<String>,
    pub head_limit: Option<usize>,
    pub offset: Option<usize>,
    pub multiline: Option<bool>,
}
```

- `GlobSearchOutput`（[`L120-L127`](/rust/crates/runtime/src/file_ops.rs#L120-L127)）和 `GrepSearchOutput`（[`L158-L172`](/rust/crates/runtime/src/file_ops.rs#L158-L172)）返回搜索的元数据、匹配文件列表以及可选内容。

注意 `GrepSearchInput` 中的 `head_limit` 和 `offset` —— 这是 token 预算控制的关键阀门。`output_mode` 支持 `files_with_matches`（仅文件名）、`content`（含上下文行）、`count`（仅计数）三种模式，与 `ripgrep` 的参数语义完全一致。

### 源码映射：工具分发与调度执行

当模型请求 `glob_search` 或 `grep_search` 时，`tools/src/lib.rs` 的 `execute_tool_with_enforcer` 函数（[`L1178-L1207`](/rust/crates/tools/src/lib.rs#L1178-L1207)）会按名分发：

```rust
"glob_search" => {
    maybe_enforce_permission_check(enforcer, name, input)?;
    from_value::<GlobSearchInputValue>(input).and_then(run_glob_search)
}
"grep_search" => {
    maybe_enforce_permission_check(enforcer, name, input)?;
    from_value::<GrepSearchInput>(input).and_then(run_grep_search)
}
```

这些工具默认所需的权限级别是 `ReadOnly`，所以在大多数模式下无需弹窗确认即可执行。`run_glob_search` 和 `run_grep_search` 只是薄薄的一层 JSON 反序列化和结果序列化包装（[`L1969-L1986`](/rust/crates/tools/src/lib.rs#L1969-L1986)），真正业务逻辑在 `runtime/src/file_ops.rs` 中。

---

## ripgrep 的内嵌方式

TypeScript 上游文档强调 Claude Code 内嵌了 `ripgrep` 并实现了三级降级策略（系统 rg / bundled / vendor）。在 `claw-code` 的 Rust 实现中，这一层被替换为原生的 Rust 生态依赖 —— 不调用外部 `rg` 二进制，而是直接在进程内使用 `regex`、`walkdir` 和 `glob` 三个 crate 完成等效逻辑。

### 源码映射：Grep 引擎实现

`runtime/src/file_ops.rs` 的 `grep_search` 函数（[`L351-L452`](/rust/crates/runtime/src/file_ops.rs#L351-L452)）就是进程内正则搜索引擎：

```rust
pub fn grep_search(input: &GrepSearchInput) -> io::Result<GrepSearchOutput> {
    let base_path = input
        .path
        .as_deref()
        .map(normalize_path)
        .transpose()?
        .unwrap_or(std::env::current_dir()?);

    let regex = RegexBuilder::new(&input.pattern)
        .case_insensitive(input.case_insensitive.unwrap_or(false))
        .dot_matches_new_line(input.multiline.unwrap_or(false))
        .build()
        .map_err(|error| io::Error::new(io::ErrorKind::InvalidInput, error.to_string()))?;
    // ... 遍历文件、匹配、输出
}
```

没有 fork/exec 开销，也不需要处理 macOS 代码签名或 Windows `.exe` 分发问题。`regex` crate 的性能在大多数工作区规模下足够高效，但 `ripgrep` 拥有更成熟的并行目录遍历和基于 SIMD 的字节码预过滤，在超大规模代码库中仍可能显著更快。

### 源码映射：文件收集与过滤

`collect_search_files`（[`L460-L474`](/rust/crates/runtime/src/file_ops.rs#L460-L474)）使用 `walkdir` 递归遍历目录：

```rust
fn collect_search_files(base_path: &Path) -> io::Result<Vec<PathBuf>> {
    if base_path.is_file() {
        return Ok(vec![base_path.to_path_buf()]);
    }
    let mut files = Vec::new();
    for entry in WalkDir::new(base_path) {
        let entry = entry.map_err(|error| io::Error::other(error.to_string()))?;
        if entry.file_type().is_file() {
            files.push(entry.path().to_path_buf());
        }
    }
    Ok(files)
}
```

 glob 过滤由 `matches_optional_filters`（[`L475-L495`](/rust/crates/runtime/src/file_ops.rs#L475-L495)）完成，支持 `glob` 模式和 `type`（扩展名）两类过滤：

```rust
fn matches_optional_filters(
    path: &Path,
    glob_filter: Option<&Pattern>,
    file_type: Option<&str>,
) -> bool {
    // ...
}
```

---

## macOS 代码签名

Rust 实现中不再需要担心 vendor 模式下 `rg` 二进制的 Gatekeeper / ad-hoc 签名问题，因为 `regex` 和 `walkdir` 都通过 Cargo 编译为静态链接的本机代码。所有搜索逻辑都在 `claw` 主二进制内部执行，天然享有与主程序相同的签名上下文（如有）。

---

## 搜索结果的设计考量

### head_limit 与 Token 预算

`grep_search` 对文件名和匹配行都会调用 `apply_limit`（[`L497-L516`](/rust/crates/runtime/src/file_ops.rs#L497-L516)）：

```rust
fn apply_limit<T>(
    items: Vec<T>,
    limit: Option<usize>,
    offset: Option<usize>,
) -> (Vec<T>, Option<usize>, Option<usize>) {
    let offset_value = offset.unwrap_or(0);
    let mut items = items.into_iter().skip(offset_value).collect::<Vec<_>>();
    let explicit_limit = limit.unwrap_or(250);
    // ...
}
```

默认 `limit = 250`。这与上游文档中"250 条 ≈ 12,500-25,000 token"的预算控制思想完全一致。在 Rust 实现中，这个默认值被硬编码在 `apply_limit` 中，而非依赖模型参数，确保任何未指定 `head_limit` 的搜索都不会一次性淹没上下文。

### 按修改时间排序

`glob_search` 函数在收集结果后，会对文件按 `mtime`（修改时间）降序排列（[`L328-L333`](/rust/crates/runtime/src/file_ops.rs#L328-L333)）：

```rust
matches.sort_by_key(|path| {
    fs::metadata(path)
        .and_then(|metadata| metadata.modified())
        .ok()
        .map(Reverse)
});
```

设计假设完全复刻上游：最近修改的文件最可能与当前任务相关。Glob 返回的 100 条结果上限内，AI 优先看到的是"活"的代码。

---

## ripgrep 的错误处理

Rust 实现将错误处理收敛到两个层面：

1. **Schema 级错误**：`RegexBuilder::build()` 失败时返回 `io::ErrorKind::InvalidInput`（[`L359-L360`](/rust/crates/runtime/src/file_ops.rs#L359-L360)）。
2. **IO 级错误**：文件读取失败（如权限不足、二进制内容）时，`grep_search` 通过 `let Ok(file_contents) = fs::read_to_string(...)` 静默跳过该文件（[`L387-L389`](/rust/crates/runtime/src/file_ops.rs#L387-L389)），而不是让整个搜索崩溃。

这种"部分成功"策略与上游文档中"超时返回已有部分结果"、"EAGAIN 降级重试"的思路一致：搜索是一个不需要原子成功的大范围扫描，允许单点失败。

---

## ToolSearch：在 50+ 工具中发现目标

当 MCP 服务器和运行时插件加载后，可用工具数量可能超过 50 个。人类开发者未必记得准确工具名，AI 也一样。`ToolSearch` 提供了基于关键词的工具发现机制。

### 源码映射：工具搜索算法

`tools/src/lib.rs` 的核心搜索逻辑在 `search_tool_specs`（[`L4133-L4205`](/rust/crates/tools/src/lib.rs#L4133-L4205)）：

```rust
fn search_tool_specs(query: &str, max_results: usize, specs: &[SearchableToolSpec]) -> Vec<String> {
    let lowered = query.to_lowercase();
    if let Some(selection) = lowered.strip_prefix("select:") {
        return selection
            .split(',')
            .map(str::trim)
            .filter_map(|wanted| {
                // ...
            })
            .take(max_results)
            .collect();
    }
    // ... 精确匹配、前缀匹配、加权评分 ...
}
```

查询如果以 `select:` 开头，走快速精确选择路径；否则执行关键词评分。评分规则与上游不同但等效：

- `name == term`：+8 分
- `canonical_name == canonical_term`：+12 分（去后缀规范化后的精确匹配）
- `name.contains(term)`：+4 分
- 描述匹配：+3 分
- 普通 haystack 包含：+2 分

同时支持 `+required` 前缀进行必选词过滤——如果某个必选词不在工具名或描述中，该工具直接淘汰。这与上游的"必选词过滤"设计一致。

### select: 直接选择

`select:` 前缀在 `search_tool_specs` 中作为第一guard处理。它比完整评分更快，且支持逗号分隔的批量选择，比如 `select:bash,grep_search,read_file`。

### 延迟加载（Deferred Tools）

`deferred_tool_specs`（[`L4121-L4131`](/rust/crates/tools/src/lib.rs#L4121-L4131)）只扫描**延迟工具**（deferred tools）：

```rust
fn deferred_tool_specs() -> Vec<ToolSpec> {
    mvp_tool_specs()
        .into_iter()
        .filter(|spec| {
            !matches!(
                spec.name,
                "bash" | "read_file" | "write_file" | "edit_file" | "glob_search" | "grep_search"
            )
        })
        .collect()
}
```

高频核心工具（Bash、FileOps、Glob/Grep）被排除在延迟集合之外，常驻提示词；低频和 MCP 工具只在 ToolSearch 命中后才被注入到系统提示中。这是控制 token 开销的关键策略。

---

## Web 搜索与抓取

AI 的信息获取不应局限于本地代码。`claw-code` 内置了两个网络工具：`WebSearch` 和 `WebFetch`，分别对应搜索引擎抓取和特定 URL 内容获取。

### 源码映射：WebSearch 实现机制

`WebSearch` 的实现位于 `tools/src/lib.rs` 的 `execute_web_search`（[`L2590-L2642`](/rust/crates/tools/src/lib.rs#L2590-L2642)）。与 TypeScript 上游的适配器模式不同，Rust 实现采用了一个轻量级的统一后端：

1. 默认使用 DuckDuckGo HTML 端点 (`https://html.duckduckgo.com/html/`)
2. 如果环境变量 `CLAWD_WEB_SEARCH_BASE_URL` 存在，则将其作为自定义搜索基地址
3. 使用 `reqwest` 发送 GET 请求，获取 HTML
4. 通过 `extract_search_hits` 解析 DuckDuckGo 的 redirect 链接和标题
5. 应用 `allowed_domains` / `blocked_domains` 过滤
6. 截断到最多 8 条结果
7. 将结果渲染为 Markdown 超链接列表返回

```rust
fn execute_web_search(input: &WebSearchInput) -> Result<WebFetchOutput, String> {
    let started = Instant::now();
    let client = build_http_client()?;
    let search_url = build_search_url(&input.query)?;
    // ... 获取 HTML、提取 hits、域过滤、去重、截断 ...
}
```

注意当前实现**没有**上游的 `ApiSearchAdapter`/`BingSearchAdapter` 双后端切换，也没有 Anthropic API 的 `web_search_20250305` server tool。Rust 版本目前走的是"零 API Key"路线：直接抓取 DuckDuckGo 的公开 HTML 结果页。

### 源码映射：WebFetch 实现机制

`WebFetch` 的实现位于 `execute_web_fetch`（[`L2556-L2590`](/rust/crates/tools/src/lib.rs#L2556-L2590)）:

```rust
fn execute_web_fetch(input: &WebFetchInput) -> Result<WebFetchOutput, String> {
    let started = Instant::now();
    let client = build_http_client()?;
    let request_url = normalize_fetch_url(&input.url)?;
    let response = client.get(request_url.clone()).send()?;
    // ... 读取 body、按 content_type 做 HTML→Text 转换、调用 summarize_web_fetch
}
```

HTTP 客户端配置在 `build_http_client`（[`L2644-L2650`](/rust/crates/tools/src/lib.rs#L2644-L2650)）中：

```rust
fn build_http_client() -> Result<Client, String> {
    Client::builder()
        .timeout(Duration::from_secs(20))
        .redirect(reqwest::redirect::Policy::limited(10))
        .user_agent("clawd-rust-tools/0.1")
        .build()
        // ...
}
```

与上游相比，Rust 实现的安全层更为简洁：

- 超时就地硬编码为 20 秒（而非 60 秒）
- 重定向策略由 `reqwest` 内建支持，上限 10 次
- URL 规范化会自动将非本地 `http` 升级为 `https`
- 没有上游的域名预检 API、也没有预批准域名列表和 Haiku 摘要步骤
- HTML 到纯文本的转换是一个自研的轻量级 parser（`html_to_text`，[`L2738-L2767`](/rust/crates/tools/src/lib.rs#L2738-L2767)），而非 Turndown

### 搜索结果解析

`extract_search_hits`（[`L2790-L2826`](/rust/crates/tools/src/lib.rs#L2790-L2826)）通过字符串扫描提取 DuckDuckGo 结果页中的 `result__a` 锚点。如果主模式找不到任何结果，会退化为 `extract_search_hits_from_generic_links`（[`L2827-L2868`](/rust/crates/tools/src/lib.rs#L2827-L2868)）——一个通用的 `<a href="...">` 抓取 fallback。

```rust
fn extract_search_hits(html: &str) -> Vec<SearchHit> {
    let mut hits = Vec::new();
    let mut remaining = html;
    while let Some(anchor_start) = remaining.find("result__a") {
        // ... 提取 href、解码重定向、收集 title ...
    }
    hits
}
```

DuckDuckGo 的重定向解码在 `decode_duckduckgo_redirect`（[`L2879-L2900`](/rust/crates/tools/src/lib.rs#L2879-L2900)）中处理：支持直接的 `http/https` URL、`//` 协议相对 URL、`/l/` 短链（从 `uddg` 参数提取真实目标）。

---

## LSP 集成导航

除了文件/Grep 级别的"文本搜索"，代码库导航的更高阶形态是"语义导航"——跳转到定义、查找引用、获取悬停提示。`claw-code` 在 `runtime/src/lsp_client.rs` 中实现了一个 LSP 客户端注册表，用于这类导航需求。

### 源码映射：LSP Registry 与 Action 分发

`lsp_client.rs` 定义了 `LspRegistry`（[`L110-L233`](/rust/crates/runtime/src/lsp_client.rs#L110-L233)）和 `LspAction` 枚举（[`L12-L35`](/rust/crates/runtime/src/lsp_client.rs#L12-L35)）：

```rust
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum LspAction {
    Diagnostics,
    Hover,
    Definition,
    References,
    Completion,
    Symbols,
    Format,
}

impl LspAction {
    pub fn from_str(s: &str) -> Option<Self> {
        match s {
            "diagnostics" => Some(Self::Diagnostics),
            "hover" => Some(Self::Hover),
            "definition" | "goto_definition" => Some(Self::Definition),
            "references" | "find_references" => Some(Self::References),
            // ...
        }
    }
}
```

`LspRegistry::find_server_for_path`（[`L151-L172`](/rust/crates/runtime/src/lsp_client.rs#L151-L172)）根据文件扩展名匹配语言服务器：

```rust
pub fn find_server_for_path(&self, path: &str) -> Option<LspServerState> {
    let ext = std::path::Path::new(path)
        .extension()
        .and_then(|e| e.to_str())
        .unwrap_or("");
    let language = match ext {
        "rs" => "rust",
        "ts" | "tsx" => "typescript",
        "js" | "jsx" => "javascript",
        "py" => "python",
        // ...
    };
    self.get(language)
}
```

`dispatch` 方法（[`L236-L348`](/rust/crates/runtime/src/lsp_client.rs#L236-L348)）目前返回结构化的占位结果或基于内部调试接口的简化响应，真实的 LSP JSON-RPC 调用链路尚未完全接入。这意味着 Rust 实现中 LSP 导航能力的骨架已经搭好（状态追踪、扩展名路由、Action 解析），但完整走到 `rust-analyzer` / `typescript-language-server` 还需要后续工作。当前状态更接近"概念验证"而非"生产可用"的 LSP 客户端。

---

## Git 上下文：代码库导航的罗盘

在启动系统提示词组装时，`GitContext` 提供了当前分支、最近提交和已暂存文件的快照。这些信息虽然不直接执行"搜索"，却是 AI 在代码库中导航时的方向感来源。

### 源码映射：GitContext 检测

`runtime/src/git_context.rs`（[`L24-L42`](/rust/crates/runtime/src/git_context.rs#L24-L42)）的 `GitContext::detect`：

```rust
impl GitContext {
    pub fn detect(cwd: &Path) -> Option<Self> {
        let rev_parse = Command::new("git")
            .args(["rev-parse", "--is-inside-work-tree"])
            .current_dir(cwd)
            .output()
            .ok()?;
        if !rev_parse.status.success() {
            return None;
        }
        Some(Self {
            branch: read_branch(cwd),
            recent_commits: read_recent_commits(cwd),
            staged_files: read_staged_files(cwd),
        })
    }
}
```

- `read_branch`：获取当前分支名
- `read_recent_commits`：获取最近 5 条提交摘要
- `read_staged_files`：获取已暂存的文件列表

`render` 方法将这些信息格式化为人类可读的文本，注入到 system prompt 中，帮助 AI 理解"当前工作在哪个分支上"、"最近动了哪些文件"。

### 审视角：进程内搜索的性能天花板

`grep_search` 使用 `regex` + `walkdir` 在单线程上下文中逐文件匹配。虽然省去了 fork/exec 的开销，但在超大规模代码库（数十万文件）中，它无法利用 `ripgrep` 的并行目录遍历、基于 `.gitignore` 的智能跳过和 SIMD 字节码预过滤。这意味着 `claw-code` 的搜索工具在大多数日常项目中足够快，但在极端规模下可能成为交互瓶颈。如果将来需要支撑企业级 monorepo，直接调用系统 `rg` 或引入 ` ignore` crate 的并行扫描可能是必要的性能投资。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/file_ops.rs`](/rust/crates/runtime/src/file_ops.rs) | `read_file`、`write_file`、`edit_file`、`glob_search`、`grep_search` 实现及安全边界 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ToolExecutor` trait、`ConversationRuntime` 工具执行循环 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `execute_tool_with_enforcer`、`GlobalToolRegistry::search`、WebSearch/WebFetch 实现 |
| [`/rust/crates/runtime/src/lsp_client.rs`](/rust/crates/runtime/src/lsp_client.rs) | `LspRegistry`、`LspAction`、语言服务器路由与 Action 分发 |
| [`/rust/crates/runtime/src/git_context.rs`](/rust/crates/runtime/src/git_context.rs) | `GitContext` 检测、分支/最近提交/暂存文件注入系统提示 |
