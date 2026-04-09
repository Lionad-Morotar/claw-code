# Unit 49: Web Browser Tool 技术报告

## 摘要

本文档分析了 claw-code 项目中与 Web 浏览器工具相关的实现。经过源码探索，发现项目实现了 **WebFetch** 和 **WebSearch** 两个核心工具，但未直接实现 Safari MCP 或 Playwright 浏览器自动化工具。

---

## 1. 原始文档参考

**预期来源**: https://ccb.agent-aura.top/docs/features/web-browser-tool

> ⚠️ **注意**: 由于网络访问权限限制，无法直接抓取原始文档内容。本报告基于源码逆向分析完成。

---

## 2. 核心工具实现

### 2.1 WebFetch 工具

**功能**: 抓取指定 URL 内容，转换为可读文本，并根据提示词回答问题。

**源码位置**: `rust/crates/tools/src/lib.rs`

#### 2.1.1 工具定义 (L493-L507)

```rust
ToolSpec {
    name: "WebFetch",
    description: "Fetch a URL, convert it into readable text, and answer a prompt about it.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "url": { "type": "string", "format": "uri" },
            "prompt": { "type": "string" }
        },
        "required": ["url", "prompt"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::ReadOnly,
}
```

#### 2.1.2 输入结构 (L2076-L2080)

```rust
#[derive(Debug, Deserialize)]
struct WebFetchInput {
    url: String,
    prompt: String,
}
```

#### 2.1.3 执行入口 (L1979-L1981)

```rust
fn run_web_fetch(input: WebFetchInput) -> Result<String, String> {
    to_pretty_json(execute_web_fetch(&input)?)
}
```

#### 2.1.4 核心执行逻辑 (L2556-L2588)

```rust
fn execute_web_fetch(input: &WebFetchInput) -> Result<WebFetchOutput, String> {
    let started = Instant::now();
    let client = build_http_client()?;
    let request_url = normalize_fetch_url(&input.url)?;
    let response = client
        .get(request_url.clone())
        .send()
        .map_err(|error| error.to_string())?;

    let status = response.status();
    let final_url = response.url().to_string();
    let code = status.as_u16();
    let code_text = status.canonical_reason().unwrap_or("Unknown").to_string();
    let content_type = response
        .headers()
        .get(reqwest::header::CONTENT_TYPE)
        .and_then(|value| value.to_str().ok())
        .unwrap_or_default()
        .to_string();
    let body = response.text().map_err(|error| error.to_string())?;
    let bytes = body.len();
    let normalized = normalize_fetched_content(&body, &content_type);
    let result = summarize_web_fetch(&final_url, &input.prompt, &normalized, &body, &content_type);

    Ok(WebFetchOutput {
        bytes,
        code,
        code_text,
        result,
        duration_ms: started.elapsed().as_millis(),
        url: final_url,
    })
}
```

#### 2.1.5 URL 规范化 (L2653-L2666)

```rust
fn normalize_fetch_url(url: &str) -> Result<String, String> {
    let parsed = reqwest::Url::parse(url).map_err(|error| error.to_string())?;
    if parsed.scheme() == "http" {
        let host = parsed.host_str().unwrap_or_default();
        if host != "localhost" && host != "127.0.0.1" && host != "::1" {
            let mut upgraded = parsed;
            upgraded
                .set_scheme("https")
                .map_err(|()| String::from("failed to upgrade URL to https"))?;
            return Ok(upgraded.to_string());
        }
    }
    Ok(parsed.to_string())
}
```

**特性**:
- 自动将 HTTP 升级为 HTTPS (localhost 除外)
- 支持重定向跟踪
- 20 秒超时

#### 2.1.6 内容处理 (L2682-L2802)

```rust
fn normalize_fetched_content(body: &str, content_type: &str) -> String {
    if content_type.contains("html") {
        html_to_text(body)
    } else {
        body.trim().to_string()
    }
}

fn html_to_text(html: &str) -> String {
    let mut text = String::with_capacity(html.len());
    let mut in_tag = false;
    let mut previous_was_space = false;

    for ch in html.chars() {
        match ch {
            '<' => in_tag = true,
            '>' => in_tag = false,
            _ if in_tag => {}
            '&' => {
                text.push('&');
                previous_was_space = false;
            }
            ch if ch.is_whitespace() => {
                if !previous_was_space {
                    text.push(' ');
                    previous_was_space = true;
                }
            }
            _ => {
                text.push(ch);
                previous_was_space = false;
            }
        }
    }

    collapse_whitespace(&decode_html_entities(&text))
}
```

**特性**:
- HTML 标签剥离
- HTML 实体解码
- 空白字符折叠

#### 2.1.7 智能摘要 (L2695-L2716)

```rust
fn summarize_web_fetch(
    url: &str,
    prompt: &str,
    content: &str,
    raw_body: &str,
    content_type: &str,
) -> String {
    let lower_prompt = prompt.to_lowercase();
    let compact = collapse_whitespace(content);

    let detail = if lower_prompt.contains("title") {
        extract_title(content, raw_body, content_type).map_or_else(
            || preview_text(&compact, 600),
            |title| format!("Title: {title}"),
        )
    } else if lower_prompt.contains("summary") || lower_prompt.contains("summarize") {
        preview_text(&compact, 900)
    } else {
        let preview = preview_text(&compact, 900);
        format!("Prompt: {prompt}\nContent preview:\n{preview}")
    };

    format!("Fetched {url}\n{detail}")
}
```

**特性**:
- 根据提示词自动识别需求 (title/summary/通用)
- 智能标题提取
- 可配置长度的内容预览

---

### 2.2 WebSearch 工具

**功能**: 使用搜索引擎查询网络信息，返回带引用的结果列表。

**源码位置**: `rust/crates/tools/src/lib.rs`

#### 2.2.1 工具定义 (L508-L528)

```rust
ToolSpec {
    name: "WebSearch",
    description: "Search the web for current information and return cited results.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "query": { "type": "string", "minLength": 2 },
            "allowed_domains": {
                "type": "array",
                "items": { "type": "string" }
            },
            "blocked_domains": {
                "type": "array",
                "items": { "type": "string" }
            }
        },
        "required": ["query"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::ReadOnly,
}
```

#### 2.2.2 输入结构 (L2082-L2087)

```rust
#[derive(Debug, Deserialize)]
struct WebSearchInput {
    query: String,
    allowed_domains: Option<Vec<String>>,
    blocked_domains: Option<Vec<String>>,
}
```

#### 2.2.3 执行入口 (L1984-L1986)

```rust
fn run_web_search(input: WebSearchInput) -> Result<String, String> {
    to_pretty_json(execute_web_search(&input)?)
}
```

#### 2.2.4 核心执行逻辑 (L2590-L2642)

```rust
fn execute_web_search(input: &WebSearchInput) -> Result<WebSearchOutput, String> {
    let started = Instant::now();
    let client = build_http_client()?;
    let search_url = build_search_url(&input.query)?;
    let response = client
        .get(search_url)
        .send()
        .map_err(|error| error.to_string())?;

    let final_url = response.url().clone();
    let html = response.text().map_err(|error| error.to_string())?;
    let mut hits = extract_search_hits(&html);

    if hits.is_empty() && final_url.host_str().is_some() {
        hits = extract_search_hits_from_generic_links(&html);
    }

    if let Some(allowed) = input.allowed_domains.as_ref() {
        hits.retain(|hit| host_matches_list(&hit.url, allowed));
    }
    if let Some(blocked) = input.blocked_domains.as_ref() {
        hits.retain(|hit| !host_matches_list(&hit.url, blocked));
    }

    dedupe_hits(&mut hits);
    hits.truncate(8);

    let summary = if hits.is_empty() {
        format!("No web search results matched the query {:?}.", input.query)
    } else {
        let rendered_hits = hits
            .iter()
            .map(|hit| format!("- [{}]({})", hit.title, hit.url))
            .collect::<Vec<_>>()
            .join("\n");
        format!(
            "Search results for {:?}. Include a Sources section in the final answer.\n{}",
            input.query, rendered_hits
        )
    };

    Ok(WebSearchOutput {
        query: input.query.clone(),
        results: vec![
            WebSearchResultItem::Commentary(summary),
            WebSearchResultItem::SearchResult {
                tool_use_id: String::from("web_search_1"),
                content: hits,
            },
        ],
        duration_seconds: started.elapsed().as_secs_f64(),
    })
}
```

**特性**:
- 默认使用 DuckDuckGo HTML 接口
- 支持 `CLAWD_WEB_SEARCH_BASE_URL` 环境变量自定义搜索引擎
- 域名白名单/黑名单过滤
- 自动去重
- 最多返回 8 条结果

#### 2.2.5 搜索 URL 构建 (L2668-L2681)

```rust
fn build_search_url(query: &str) -> Result<reqwest::Url, String> {
    if let Ok(base) = std::env::var("CLAWD_WEB_SEARCH_BASE_URL") {
        let mut url = reqwest::Url::parse(&base).map_err(|error| error.to_string())?;
        url.query_pairs_mut().append_pair("q", query);
        return Ok(url);
    }

    let mut url = reqwest::Url::parse("https://html.duckduckgo.com/html/")
        .map_err(|error| error.to_string())?;
    url.query_pairs_mut().append_pair("q", query);
    Ok(url)
}
```

#### 2.2.6 DuckDuckGo 结果解析 (L2797-L2834)

```rust
fn extract_search_hits(html: &str) -> Vec<SearchHit> {
    let mut hits = Vec::new();
    let mut remaining = html;

    while let Some(anchor_start) = remaining.find("result__a") {
        let after_class = &remaining[anchor_start..];
        let Some(href_idx) = after_class.find("href=") else {
            remaining = &after_class[1..];
            continue;
        };
        let href_slice = &after_class[href_idx + 5..];
        let Some((url, rest)) = extract_quoted_value(href_slice) else {
            remaining = &after_class[1..];
            continue;
        };
        // ... 解析标题和 URL
        if let Some(decoded_url) = decode_duckduckgo_redirect(&url) {
            hits.push(SearchHit {
                title: title.trim().to_string(),
                url: decoded_url,
            });
        }
        remaining = &after_tag[end_anchor_idx + 4..];
    }

    hits
}
```

#### 2.2.7 重定向 URL 解码 (L2878-L2905)

```rust
fn decode_duckduckgo_redirect(url: &str) -> Option<String> {
    if url.starts_with("http://") || url.starts_with("https://") {
        return Some(html_entity_decode_url(url));
    }

    let joined = if url.starts_with("//") {
        format!("https:{url}")
    } else if url.starts_with('/') {
        format!("https://duckduckgo.com{url}")
    } else {
        return None;
    };

    let parsed = reqwest::Url::parse(&joined).ok()?;
    if parsed.path() == "/l/" || parsed.path() == "/l" {
        for (key, value) in parsed.query_pairs() {
            if key == "uddg" {
                return Some(html_entity_decode_url(value.as_ref()));
            }
        }
    }
    Some(joined)
}
```

---

## 3. HTTP 客户端配置

**源码位置**: `rust/crates/tools/src/lib.rs` (L2644-L2651)

```rust
fn build_http_client() -> Result<Client, String> {
    Client::builder()
        .timeout(Duration::from_secs(20))
        .redirect(reqwest::redirect::Policy::limited(10))
        .user_agent("clawd-rust-tools/0.1")
        .build()
        .map_err(|error| error.to_string())
}
```

**配置参数**:
| 参数 | 值 | 说明 |
|------|-----|------|
| timeout | 20 秒 | 请求超时 |
| redirect | limited(10) | 最多 10 次重定向 |
| user_agent | clawd-rust-tools/0.1 | 自定义 UA |

---

## 4. 工具调度集成

**源码位置**: `rust/crates/tools/src/lib.rs` (L1208-L1209)

```rust
"WebFetch" => from_value::<WebFetchInput>(input).and_then(run_web_fetch),
"WebSearch" => from_value::<WebSearchInput>(input).and_then(run_web_search),
```

**工具注册**: 参见 `mvp_tool_specs()` 函数 (L385+) 中的完整工具列表。

---

## 5. 未发现的功能

### 5.1 Safari MCP

**搜索结果**: 未在源码中找到 Safari 浏览器自动化相关的 MCP 工具实现。

**相关搜索关键词**:
- `safari`
- `mcp__safari`
- `safari_snapshot`
- `safari_click`
- `safari_navigate`

### 5.2 Playwright

**搜索结果**: 未在源码中找到 Playwright 浏览器自动化相关的工具实现。

**相关搜索关键词**:
- `playwright`
- `mcp__plugin_playwright`
- `browser_automation`

### 5.3 对比分析

根据用户提供的技能列表中提到的 MCP 工具，claw-code 项目当前的 Web 能力主要集中在 **HTTP 请求级别**的 Fetch 和 Search，而非**浏览器自动化级别**的：
- 页面截图
- DOM 操作
- JavaScript 执行
- 表单填写
- 键盘/鼠标事件模拟

---

## 6. 输出结构

### 6.1 WebFetchOutput (L2354-L2364)

```rust
#[derive(Debug, Serialize)]
struct WebFetchOutput {
    bytes: usize,
    code: u16,
    code_text: String,
    result: String,
    duration_ms: u128,
    url: String,
}
```

### 6.2 WebSearchOutput (L2366-L2374)

```rust
#[derive(Debug, Serialize)]
struct WebSearchOutput {
    query: String,
    results: Vec<WebSearchResultItem>,
    duration_seconds: f64,
}
```

---

## 7. 测试验证

**源码位置**: `rust/crates/tools/src/lib.rs` (L6177-L6352)

项目包含完整的 WebFetch 和 WebSearch 工具集成测试：

```rust
// WebFetch 测试
let result = execute_tool("WebFetch", &json!({
    "url": "https://example.com",
    "prompt": "Extract the title"
})).expect("WebFetch should succeed");

// WebSearch 测试
let result = execute_tool("WebSearch", &json!({
    "query": "rust programming"
})).expect("WebSearch should succeed");
```

---

## 8. 结论

### 已实现功能

| 工具 | 状态 | 源码位置 |
|------|------|----------|
| WebFetch | ✅ 完整实现 | `rust/crates/tools/src/lib.rs:L493-L507`, `L2556-L2588` |
| WebSearch | ✅ 完整实现 | `rust/crates/tools/src/lib.rs:L508-L528`, `L2590-L2642` |

### 未实现功能

| 功能 | 状态 | 备注 |
|------|------|------|
| Safari MCP | ❌ 未发现 | 可能需要作为外部 MCP 服务器集成 |
| Playwright | ❌ 未发现 | 可能需要作为插件或外部工具集成 |
| 浏览器截图 | ❌ 未发现 | commands 中有 screenshot 命令定义，但未找到浏览器集成 |
| DOM 操作 | ❌ 未发现 | - |

### 建议

1. **若需浏览器自动化**: 考虑通过 MCP 协议集成外部浏览器工具 (如 `mcp__safari` 或 `mcp__plugin_playwright`)
2. **扩展现有能力**: 当前 WebFetch 和 WebSearch 已满足基本的 Web 信息获取需求
3. **文档补充**: 建议在官方文档中明确区分 HTTP 级别工具与浏览器自动化工具的边界

---

## 附录：源码文件清单

| 文件 | 行数 | 说明 |
|------|------|------|
| `rust/crates/tools/src/lib.rs` | 8600 | 主工具实现 |
| `rust/crates/runtime/src/mcp_server.rs` | 200+ | MCP 服务器框架 |
| `rust/crates/runtime/src/mcp_stdio.rs` | - | MCP 客户端实现 |

---

*报告生成时间：2026-04-09*
*分析范围：claw-code rust workspace*
