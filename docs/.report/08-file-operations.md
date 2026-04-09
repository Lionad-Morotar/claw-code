# 文件操作工具 - 三大工具的源码级解剖

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/tools/file-operations) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 三大工具的职责分化

Claude Code 将文件操作拆分为三个独立工具——这不是简单的功能划分，而是**风险分级**：

| 工具 | 权限级别 | 核心能力 | 关键限制 |
|------|----------|----------|----------|
| `read_file` | 只读（免审批） | 读文本 / 分页窗口 | `PermissionMode::ReadOnly` |
| `write_file` | 写入（需确认/升级） | 全量覆盖写入 | `PermissionMode::WorkspaceWrite` |
| `edit_file` | 写入（需确认/升级） | 精确字符串替换 | `PermissionMode::WorkspaceWrite` |

`read_file` 与 `glob_search`、`grep_search` 同属只读工具；`write_file` 和 `edit_file` 则需要更高的 `WorkspaceWrite` 或 `DangerFullAccess` 权限。权限判定由 `PermissionPolicy` 统一执行（参见 [`permissions.rs#L174-L292`](/rust/crates/runtime/src/permissions.rs#L174-L292)）。

在 `claw-code` Rust 实现中，这三大工具（加上 `glob_search`、`grep_search`）被定义在 [`tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) 的 `mvp_tool_specs()` 中，并通过 `execute_tool()` 分发给 [`runtime/src/file_ops.rs`](/rust/crates/runtime/src/file_ops.rs) 中的原生函数。工具声明的 `required_permission` 字段直接决定了用户在 `ReadOnly` 模式下能否调用该工具。

### 源码映射：Tools 层如何封装文件操作

`tools` crate 是文件操作工具的“外部接口层”。它负责三件事：

1. **定义 ToolSpec（JSON Schema + 权限级别）**
2. **反序列化模型输入**
3. **调用 `runtime` crate 执行实际 I/O**

以下是 `mvp_tool_specs()` 中对文件操作工具的声明（[`tools/src/lib.rs#L409-L475`](/rust/crates/tools/src/lib.rs#L409-L475)）：

```rust
ToolSpec {
    name: "read_file",
    description: "Read a text file from the workspace.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "path": { "type": "string" },
            "offset": { "type": "integer", "minimum": 0 },
            "limit": { "type": "integer", "minimum": 1 }
        },
        "required": ["path"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::ReadOnly,
},
ToolSpec {
    name: "write_file",
    description: "Write a text file in the workspace.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "path": { "type": "string" },
            "content": { "type": "string" }
        },
        "required": ["path", "content"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::WorkspaceWrite,
},
ToolSpec {
    name: "edit_file",
    description: "Replace text in a workspace file.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "path": { "type": "string" },
            "old_string": { "type": "string" },
            "new_string": { "type": "string" },
            "replace_all": { "type": "boolean" }
        },
        "required": ["path", "old_string", "new_string"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::WorkspaceWrite,
},
```

`execute_tool()` 的分发逻辑（[`tools/src/lib.rs#L1188-L1207`](/rust/crates/tools/src/lib.rs#L1188-L1207)）如下：

```rust
"read_file" => {
    maybe_enforce_permission_check(enforcer, name, input)?;
    from_value::<ReadFileInput>(input).and_then(run_read_file)
}
"write_file" => {
    maybe_enforce_permission_check(enforcer, name, input)?;
    from_value::<WriteFileInput>(input).and_then(run_write_file)
}
"edit_file" => {
    maybe_enforce_permission_check(enforcer, name, input)?;
    from_value::<EditFileInput>(input).and_then(run_edit_file)
}
```

注意 `maybe_enforce_permission_check` 会先对输入执行权限预检；若当前 `PermissionMode` 低于工具所需的 `required_permission`，则直接返回拒绝，不会进入任何 I/O 路径。这是“风险分级”在代码中的第一道闸门。

---

## FileRead：多模态文件读取引擎

`read_file` 是文件操作三大工具中权限最低的，也是与 AI 交互最频繁的工具。`claw-code` 的实现位于 [`runtime/src/file_ops.rs#L175-L221`](/rust/crates/runtime/src/file_ops.rs#L175-L221)。

### 读取去重与分页窗口

Rust 实现目前还没有引入 TypeScript 上游的 `readFileState` 去重缓存层，但提供了等效的分页读取能力：通过 `offset` 和 `limit` 参数让模型只读取文件的一个行窗口，避免单次请求将整个大文件塞进上下文。

```rust
pub fn read_file(
    path: &str,
    offset: Option<usize>,
    limit: Option<usize>,
) -> io::Result<ReadFileOutput> {
    let absolute_path = normalize_path(path)?;
    // 10 MB 上限检查
    let metadata = fs::metadata(&absolute_path)?;
    if metadata.len() > MAX_READ_SIZE { ... }
    // 二进制文件拒绝
    if is_binary_file(&absolute_path)? { ... }
    let content = fs::read_to_string(&absolute_path)?;
    let lines: Vec<&str> = content.lines().collect();
    let start_index = offset.unwrap_or(0).min(lines.len());
    let end_index = limit.map_or(lines.len(), |limit| {
        start_index.saturating_add(limit).min(lines.len())
    });
    let selected = lines[start_index..end_index].join("\n");
    Ok(ReadFileOutput { ... })
}
```

`ReadFileOutput` 返回的结构包含 `content`、`num_lines`、`start_line`、`total_lines`，模型可以在后续调用中通过调整 `offset` 滑动窗口，实现对超大源码文件的“流式阅读”。

### 安全防线：二进制检测与路径规范化

`read_file` 在进入内容读取之前设置了多层防线：

1. **路径规范化**：`normalize_path(path)` 调用 `canonicalize()`，将相对路径解析为绝对路径，同时消解 `..` 和符号链接。
2. **大小检查**：`MAX_READ_SIZE = 10 * 1024 * 1024`（10 MB），防止因读取日志或核心转储文件而阻塞。
3. **二进制检测**：[`is_binary_file`](/rust/crates/runtime/src/file_ops.rs#L20-L26) 读取前 8192 字节，检查是否包含 NUL 字节；若包含则拒绝读取，防止将二进制内容当作文本注入上下文。

```rust
fn is_binary_file(path: &Path) -> io::Result<bool> {
    let mut file = fs::File::open(path)?;
    let mut buffer = [0u8; 8192];
    let bytes_read = file.read(&mut buffer)?;
    Ok(buffer[..bytes_read].contains(&0))
}
```

这与上游的 `hasBinaryExtension`  whitelist 策略不同：Rust 实现采用**内容探测**而非扩展名白名单，这意味着即使文件没有 `.exe` 后缀，只要内部含二进制字节，同样会被拒绝。

### 工作区边界与符号链接逃逸检测

`file_ops.rs` 还提供了带边界检查的变体 [`read_file_in_workspace`](/rust/crates/runtime/src/file_ops.rs#L562-L574) 和符号链接逃逸检测 [`is_symlink_escape`](/rust/crates/runtime/src/file_ops.rs#L610-L620)：

```rust
pub fn read_file_in_workspace(
    path: &str,
    offset: Option<usize>,
    limit: Option<usize>,
    workspace_root: &Path,
) -> io::Result<ReadFileOutput> {
    let absolute_path = normalize_path(path)?;
    let canonical_root = workspace_root.canonicalize()
        .unwrap_or_else(|_| workspace_root.to_path_buf());
    validate_workspace_boundary(&absolute_path, &canonical_root)?;
    read_file(path, offset, limit)
}
```

[`validate_workspace_boundary`](/rust/crates/runtime/src/file_ops.rs#L32-L44) 的核心逻辑只有一行：

```rust
fn validate_workspace_boundary(resolved: &Path, workspace_root: &Path) -> io::Result<()> {
    if !resolved.starts_with(workspace_root) {
        return Err(io::Error::new(
            io::ErrorKind::PermissionDenied,
            format!("path {} escapes workspace boundary {}", ...),
        ));
    }
    Ok(())
}
```

结合 [`is_symlink_escape`](/rust/crates/runtime/src/file_ops.rs#L610-L620) 可以进一步检测通过符号链接指向工作区外部的路径：

```rust
pub fn is_symlink_escape(path: &Path, workspace_root: &Path) -> io::Result<bool> {
    let metadata = fs::symlink_metadata(path)?;
    if !metadata.is_symlink() {
        return Ok(false);
    }
    let resolved = path.canonicalize()?;
    let canonical_root = workspace_root.canonicalize()
        .unwrap_or_else(|_| workspace_root.to_path_buf());
    Ok(!resolved.starts_with(&canonical_root))
}
```

这在本地 agentic 系统中至关重要：模型可能通过 `read_file("../.ssh/id_rsa")` 或符号链接尝试读取用户主目录中的敏感文件。边界检查是“只读≠无害”的安全修正项。

---

## FileEdit：精确字符串替换引擎

`edit_file` 实现了“搜索-替换”式编辑，是 Claude Code 最常用的代码修改工具。它的设计哲学是：**AI 不直接写文件，而是声明性的描述变更**。Rust 实现在 [`runtime/src/file_ops.rs#L258-L296`](/rust/crates/runtime/src/file_ops.rs#L258-L296)。

### 精确匹配与 `replace_all`

```rust
pub fn edit_file(
    path: &str,
    old_string: &str,
    new_string: &str,
    replace_all: bool,
) -> io::Result<EditFileOutput> {
    let absolute_path = normalize_path(path)?;
    let original_file = fs::read_to_string(&absolute_path)?;
    if old_string == new_string {
        return Err(io::Error::new(
            io::ErrorKind::InvalidInput,
            "old_string and new_string must differ",
        ));
    }
    if !original_file.contains(old_string) {
        return Err(io::Error::new(
            io::ErrorKind::NotFound,
            "old_string not found in file",
        ));
    }

    let updated = if replace_all {
        original_file.replace(old_string, new_string)
    } else {
        original_file.replacen(old_string, new_string, 1)
    };
    fs::write(&absolute_path, &updated)?;
    ...
}
```

相比 TypeScript 上游的 `findActualString()` + `preserveQuoteStyle()` 引号标准化逻辑，`claw-code` 的 Rust 实现目前采用**直截了当的字符串包含检查**。这意味着如果源码使用了弯引号（`'` / `'` / `“` / `”`），而 AI 输出的是直引号，则 `contains(old_string)` 会返回 `false`，编辑请求会被拒绝并返回 `NotFound` 错误。

这既是当前实现的局限性，也是一种**保守的安全策略**：不匹配就拒绝，避免 AI 在无意识的情况下改错位置。但在实际使用中，开发者需要注意这一行为与上游的差异。

### 原子性读-改-写与 diff 输出

`edit_file` 的调用链是同步 blocking I/O：

1. `normalize_path(path)` → 解析绝对路径
2. `fs::read_to_string(&absolute_path)?` → 读取原始内容
3. `contains` / `replacen` → 内存中计算更新后文本
4. `fs::write(&absolute_path, &updated)?` → 一次性覆盖写回磁盘

虽然这里没有显式的文件锁，但整个操作在单线程同步上下文中完成，因此具备了**近似原子性**：在第 3 步和第 4 步之间不会插入其他进程写操作。

`EditFileOutput` 还附带了一个 `structured_patch` 字段，由 [`make_patch`](/rust/crates/runtime/src/file_ops.rs#L510-L526) 生成：

```rust
fn make_patch(original: &str, updated: &str) -> Vec<StructuredPatchHunk> {
    let mut lines = Vec::new();
    for line in original.lines() {
        lines.push(format!("-{line}"));
    }
    for line in updated.lines() {
        lines.push(format!("+{line}"));
    }
    vec![StructuredPatchHunk {
        old_start: 1,
        old_lines: original.lines().count(),
        new_start: 1,
        new_lines: updated.lines().count(),
        lines,
    }]
}
```

注意这不是真正的 line-oriented diff（如 Myers diff），而是一个**全文件范围的上下文 patch**：旧文件的每一行前缀 `-`，新文件的每一行前缀 `+`。其用途主要是让 UI 层快速展示“ before / after ”的概览，而不是生成精确的 hunks。

---

## FileWrite：全量写入与创建

`write_file` 对应上游的 `FileWriteTool`，用于创建新文件或完全覆盖已有文件。Rust 实现位于 [`runtime/src/file_ops.rs#L224-L255`](/rust/crates/runtime/src/file_ops.rs#L224-L255)。

### 大小检查与目录自动创建

```rust
pub fn write_file(path: &str, content: &str) -> io::Result<WriteFileOutput> {
    if content.len() > MAX_WRITE_SIZE {
        return Err(io::Error::new(
            io::ErrorKind::InvalidData,
            "content is too large",
        ));
    }

    let absolute_path = normalize_path_allow_missing(path)?;
    let original_file = fs::read_to_string(&absolute_path).ok();
    if let Some(parent) = absolute_path.parent() {
        fs::create_dir_all(parent)?;
    }
    fs::write(&absolute_path, content)?;

    Ok(WriteFileOutput {
        kind: if original_file.is_some() { "update" } else { "create" },
        file_path: absolute_path.to_string_lossy().into_owned(),
        content: content.to_owned(),
        structured_patch: make_patch(original_file.as_deref().unwrap_or(""), content),
        original_file,
        git_diff: None,
    })
}
```

关键设计点：

- `MAX_WRITE_SIZE` 同样为 10 MB，防止模型生成过大的单文件输出。
- `normalize_path_allow_missing` 允许目标文件尚不存在：它会尝试对父目录做 `canonicalize()`，然后拼接文件名。这样 AI 可以通过 `write_file("src/new_module.rs", "...")` 直接创建新文件，而无需预先调用 `mkdir`。
- `fs::create_dir_all(parent)?` 自动创建缺失的中间目录。
- 返回的 `kind` 字段区分 `"create"` 和 `"update"`，便于 CLI 渲染层展示不同的成功图标和颜色。

### 行尾处理

Rust 实现中没有显式的行尾采样或 CRLF→LF 转换逻辑。`fs::write` 会**原样写入**模型提供的 `content` 字符串。这意味着如果 AI 输出的是 CRLF（Windows 风格换行），文件将以 CRLF 保存；如果输出的是 LF（Unix 风格换行），文件将以 LF 保存。当前实现将行尾一致性的责任交给了模型和系统提示词，而不是在 I/O 层做强制统一。

---

## Glob 与 Grep：代码库精准定位

除了读写改，`claw-code` 还内置了 `glob_search` 和 `grep_search` 两个只读导航工具，分别对应文件路径匹配和内容正则搜索。

### glob_search

实现位于 [`runtime/src/file_ops.rs#L299-L340`](/rust/crates/runtime/src/file_ops.rs#L299-L340)。它使用 `glob` crate 解析模式，支持相对路径和绝对路径：

```rust
pub fn glob_search(pattern: &str, path: Option<&str>) -> io::Result<GlobSearchOutput> {
    let base_dir = path.map(normalize_path).transpose()?.unwrap_or(std::env::current_dir()?);
    let search_pattern = if Path::new(pattern).is_absolute() {
        pattern.to_owned()
    } else {
        base_dir.join(pattern).to_string_lossy().into_owned()
    };
    // ...只取前 100 个结果，按修改时间倒序...
}
```

返回结果按 `mtime` 倒序排列，并做 `truncated` 标记（超过 100 个文件时截断）。这与上游的 `GlobSearchTool` 行为一致，都是为了让模型快速获得“最近修改过哪些文件”的概览。

### grep_search

实现位于 [`runtime/src/file_ops.rs#L343-L450`](/rust/crates/runtime/src/file_ops.rs#L343-L450)。它基于 `regex` + `walkdir` 做全目录递归搜索，支持丰富的输出模式：

- `output_mode: "files_with_matches"`（默认）— 只返回匹配到的文件名列表
- `output_mode: "content"` — 返回带上下文行的匹配内容
- `output_mode: "count"` — 返回每个文件的匹配计数

参数设计几乎与 upstream 的 Grep 工具一一对应：`-i`（忽略大小写）、`-n`（行号）、`-B`/`-A`/`-C`（上下文行）、`head_limit` / `offset`（分页）、`multiline`（点号匹配换行）。

```rust
let regex = RegexBuilder::new(&input.pattern)
    .case_insensitive(input.case_insensitive.unwrap_or(false))
    .dot_matches_new_line(input.multiline.unwrap_or(false))
    .build()
    .map_err(|error| io::Error::new(io::ErrorKind::InvalidInput, error.to_string()))?;
```

搜索文件集由 [`collect_search_files`](/rust/crates/runtime/src/file_ops.rs#L452-L465) 收集，使用 `WalkDir::new(base_path)` 递归遍历目录树。随后通过 [`matches_optional_filters`](/rust/crates/runtime/src/file_ops.rs#L467-L487) 过滤 `glob` 模式和文件扩展名，最终输出受 [`apply_limit`](/rust/crates/runtime/src/file_ops.rs#L489-L508) 控制，默认 `head_limit = 250`。

这两个工具在 `tools/src/lib.rs` 的 `allowed_tools_for_subagent` 中被高频分配给各种子 Agent：

- `Explore` 子代理：`read_file`、`glob_search`、`grep_search`
- `Plan` 子代理：`read_file`、`glob_search`、`grep_search`
- `Verification` 子代理：`bash`、`read_file`、`glob_search`、`grep_search`

参见 [`tools/src/lib.rs#L3454-L3517`](/rust/crates/tools/src/lib.rs#L3454-L3517)。

---

## 权限模型与执行边界

文件操作工具虽然是“本地文件系统 I/O”，但它们的调用并非无约束。`claw-code` 通过五级权限模型 `PermissionMode` 和三层规则系统控制工具执行：

### 五级权限模型

定义于 [`runtime/src/permissions.rs#L9-L16`](/rust/crates/runtime/src/permissions.rs#L9-L16)：

```rust
pub enum PermissionMode {
    ReadOnly,
    WorkspaceWrite,
    DangerFullAccess,
    Prompt,
    Allow,
}
```

- `ReadOnly`：只能调用 `read_file`、`glob_search`、`grep_search` 等只读工具；`write_file` / `edit_file` 会被拒绝。
- `WorkspaceWrite`：可以读写工作区内的文件，但不能执行 `bash` 等危险命令。
- `DangerFullAccess`：完全放行（仍受 deny / ask 规则约束）。
- `Prompt`：每次权限升级都弹窗确认。
- `Allow`：完全允许（通常用于自动化场景）。

### 权限策略的判定流程

`PermissionPolicy::authorize_with_context`（[`permissions.rs#L174-L292`](/rust/crates/runtime/src/permissions.rs#L174-L292)）的判定顺序如下：

1. **deny 规则短接**：若输入匹配 `deny_rules`，直接拒绝。
2. **hook override**：若 hooks 注入了 `Deny` / `Ask` / `Allow`，按 override 处理。
3. **ask 规则短接**：若输入匹配 `ask_rules`，强制进入 prompt（即使当前模式已经是 `Allow`）。
4. **allow 规则放行**：若输入匹配 `allow_rules`，直接允许。
5. **模式比较**：`current_mode >= required_mode` 时允许；否则根据 `Prompt` / `WorkspaceWrite` 等模式决定是否弹窗。

测试用例 [`permissions.rs#L568-L587`](/rust/crates/runtime/src/permissions.rs#L568-L587) 展示了规则系统的实际效果：

```rust
let rules = RuntimePermissionRuleConfig::new(
    vec!["bash(git:*)".to_string()],
    vec!["bash(rm -rf:*)".to_string()],
    Vec::new(),
);
let policy = PermissionPolicy::new(PermissionMode::ReadOnly)
    .with_tool_requirement("bash", PermissionMode::DangerFullAccess)
    .with_permission_rules(&rules);

// allow 规则匹配 git status → 允许
policy.authorize("bash", r#"{"command":"git status"}"#, None)
// deny 规则匹配 rm -rf → 拒绝
policy.authorize("bash", r#"{"command":"rm -rf /tmp/x"}"#, None)
```

### 文件操作与 Bash 的沙箱边界

`bash.rs` 中的 `BashCommandInput`（[`runtime/src/bash.rs#L19-L35`](/rust/crates/runtime/src/bash.rs#L19-L35)）包含一组与文件系统隔离相关的字段：`filesystem_mode`、`allowed_mounts`、`dangerously_disable_sandbox`。当模型请求执行 bash 命令时，这些字段会决定命令是在沙箱容器中运行，还是直接运行在宿主系统上。

这与 `file_ops.rs` 的边界检查是**互补**的：

- `file_ops.rs` 防的是 **AI 通过文件工具读取/写出工作区外**
- `bash.rs` 防的是 **AI 通过 shell 命令逃逸到宿主系统**

两者共同构成了 claw-code 的本地安全边界。

---

## Bash 辅助中的文件相关边界

虽然 Bash 不是文件操作工具，但它在实际使用中大量执行 `cat`、`sed`、`git status` 等命令，因此 `bash.rs` 中有一些与文件 I/O 直接相关的 schema 设计：

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
```

当 `filesystem_mode` 被设为 `WorkspaceOnly` 时，Linux 环境下会触发 `build_linux_sandbox_command` 构建一个受限命名空间，将 `$HOME` 和 `$TMPDIR` 重映射到工作区子目录内（`.sandbox-home` 和 `.sandbox-tmp`）。参见 [`bash.rs#L185-L207`](/rust/crates/runtime/src/bash.rs#L185-L207)。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/file_ops.rs`](/rust/crates/runtime/src/file_ops.rs) | `read_file`、`write_file`、`edit_file`、`glob_search`、`grep_search` 的实现与工作区边界检查 |
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode`、`PermissionPolicy`、`authorize_with_context`、规则匹配 |
| [`/rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs) | `BashCommandInput` schema、沙箱命令构建、超时截断 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `ToolSpec` 定义、`execute_tool` 分发、子代理工具白名单 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、参数解析、`LiveCli` 与运行时组装 |
