# 项目记忆系统 — 文件级跨对话记忆架构

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/context/project-memory) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 一句话定义

Claude Code 的**项目记忆系统**是一套基于纯文件（Markdown）的跨对话持久化机制。它没有数据库、没有向量存储，只靠 `~/.claude/projects/<repo>/memory/` 下的目录结构、索引文件和约定文件，就能在多次对话之间传承用户偏好、项目上下文和反馈修正。

---

## 记忆系统的存储架构

### 目录布局

在 TypeScript 上游实现中，记忆目录的标准路径为：

```
~/.claude/projects/<sanitized-git-root>/memory/
├── MEMORY.md                    ← 入口索引
├── user_role.md                 ← 用户记忆
├── feedback_testing.md          ← 反馈记忆
├── project_mobile_release.md    ← 项目记忆
├── reference_linear_ingest.md   ← 参考记忆
└── logs/                        ← KAIROS 每日日志
    └── 2026/
        └── 04/
            └── 2026-04-01.md
```

`claw-code` 作为 Rust 重写版，当前版本的记忆系统尚未完整复刻上游的 `memdir` 模块（`memoryTypes.ts`、`findRelevantMemories.ts`、`memdir.ts` 等），但已经实现了**等价的核心入口**—— `ProjectContext` + 指令文件发现（instruction files），作为系统提示（System Prompt）中 "memory" 的第一层落地。

---

## `ProjectContext`：Rust 实现的记忆入口

在 `claw-code` 中，项目级上下文被封装为 `runtime/src/prompt.rs` 中的 `ProjectContext` 结构体（[`L55-L63`](/rust/crates/runtime/src/prompt.rs#L55-L63)）：

```rust
pub struct ProjectContext {
    pub cwd: PathBuf,
    pub current_date: String,
    pub git_status: Option<String>,
    pub git_diff: Option<String>,
    pub git_context: Option<GitContext>,
    pub instruction_files: Vec<ContextFile>,
}
```

`instruction_files` 就是当前 Rust 实现中**最接近** "项目记忆" 的载体：它负责在每次启动时自动收集工作区及其祖先目录中的约定文件，并注入到 System Prompt 中。

### 指令文件发现链路

`ProjectContext::discover` 调用 `discover_instruction_files`（[`L203-L224`](/rust/crates/runtime/src/prompt.rs#L203-L224)）：

```rust
fn discover_instruction_files(cwd: &Path) -> std::io::Result<Vec<ContextFile>> {
    let mut directories = Vec::new();
    let mut cursor = Some(cwd);
    while let Some(dir) = cursor {
        directories.push(dir.to_path_buf());
        cursor = dir.parent();
    }
    directories.reverse();

    let mut files = Vec::new();
    for dir in directories {
        for candidate in [
            dir.join("CLAUDE.md"),
            dir.join("CLAUDE.local.md"),
            dir.join(".claw").join("CLAUDE.md"),
            dir.join(".claw").join("instructions.md"),
        ] {
            push_context_file(&mut files, candidate)?;
        }
    }
    Ok(dedupe_instruction_files(files))
}
```

设计要点：

1. **向上遍历 + reverse**：从最顶层目录（workspace root）开始向下收集，保证父级规则先进入 prompt，子级规则可以后覆盖。
2. **去重策略**：`dedupe_instruction_files`（[`L353-L368`](/rust/crates/runtime/src/prompt.rs#L353-L368)）使用内容的 `stable_content_hash` 进行去重，避免同一内容在多个层级重复出现浪费 token。
3. **大小限制**：`MAX_INSTRUCTION_FILE_CHARS = 4_000`，`MAX_TOTAL_INSTRUCTION_CHARS = 12_000`（[`L43-L44`](/rust/crates/runtime/src/prompt.rs#L43-L44)）。单份文件和总体积都有硬上限，防止恶意或过大的约定文件撑爆上下文。

### 带 Git 感知的增强发现

`ProjectContext::discover_with_git`（[`L81-L91`](/rust/crates/runtime/src/prompt.rs#L81-L91)）在基础发现之上追加三层 git 快照：

```rust
pub fn discover_with_git(...) -> std::io::Result<Self> {
    let mut context = Self::discover(cwd, current_date)?;
    context.git_status = read_git_status(&context.cwd);
    context.git_diff = read_git_diff(&context.cwd);
    context.git_context = GitContext::detect(&context.cwd);
    Ok(context)
}
```

- `read_git_status`（[`L238-L254`](/rust/crates/runtime/src/prompt.rs#L238-L254)）：抓取 `git status --short --branch`
- `read_git_diff`（[`L256-L274`](/rust/crates/runtime/src/prompt.rs#L256-L274)）：抓取 staged + unstaged diff
- `GitContext::detect`（[`git_context.rs#L26-L42`](/rust/crates/runtime/src/git_context.rs#L26-L42)）：读取分支名、最近 5 条 commit、staged 文件列表

这意味着模型在收到用户第一句话之前，就已经知道了：项目约定文件、git 分支、 staged/unstaged 变更、最近提交摘要。

---

## 记忆注入 System Prompt 的链路

`load_system_prompt`（[`L432-L446`](/rust/crates/runtime/src/prompt.rs#L432-L446)）是记忆/上下文注入的入口：

```rust
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

调用链：

1. CLI `run()` → `LiveCli::new()` → `build_runtime()` → `load_system_prompt()`
2. `load_system_prompt` 组装 `ProjectContext` + `RuntimeConfig`
3. `SystemPromptBuilder::build()` 把 `ProjectContext` 渲染成多个 prompt section

具体渲染由 `render_project_context`（[`L288-L328`](/rust/crates/runtime/src/prompt.rs#L288-L328)）和 `render_instruction_files`（[`L330-L351`](/rust/crates/runtime/src/prompt.rs#L330-L351)）完成：

```rust
fn render_project_context(project_context: &ProjectContext) -> String {
    let mut lines = vec!["# Project context".to_string()];
    let mut bullets = vec![
        format!("Today's date is {}.", project_context.current_date),
        format!("Working directory: {}", project_context.cwd.display()),
    ];
    if !project_context.instruction_files.is_empty() {
        bullets.push(format!(
            "Claude instruction files discovered: {}.",
            project_context.instruction_files.len()
        ));
    }
    lines.extend(prepend_bullets(bullets));
    // ... git status / recent commits / git diff / git_context.render()
    lines.join("\n")
}
```

`render_instruction_files` 会把每个 `ContextFile` 按 "scope" 分组渲染，例如 `CLAUDE.md (scope: /tmp/project)`，让模型明确知道某条指令的生效范围。

---

## Session 持久化：跨对话的另一种"记忆"

虽然 `claw-code` 当前版本还没有完全实现上游 `memdir` 的记忆分类法（user / feedback / project / reference），但它已经在 `session.rs` 和 `session_control.rs` 中实现了**会话级别的持久化与恢复**，这构成了跨对话记忆的底层基础设施。

### JSONL 增量持久化

`Session` 结构体定义于 [`session.rs#L89-L107`](/rust/crates/runtime/src/session.rs#L89-L107)（节选）：

```rust
pub struct Session {
    pub version: u32,
    pub session_id: String,
    pub created_at_ms: u64,
    pub updated_at_ms: u64,
    pub messages: Vec<ConversationMessage>,
    pub compaction: Option<SessionCompaction>,
    pub fork: Option<SessionFork>,
    pub workspace_root: Option<PathBuf>,
    pub prompt_history: Vec<SessionPromptEntry>,
    persistence: Option<SessionPersistence>,
}
```

当配置了 `persistence_path` 后，每条消息会通过 `append_persisted_message`（[`session.rs#L484-498`](/rust/crates/runtime/src/session.rs#L484-498)）以 **JSONL** 格式追加写入磁盘：

```rust
fn append_persisted_message(&self, message: &ConversationMessage) -> Result<(), SessionError> {
    let Some(path) = self.persistence_path() else {
        return Ok(());
    };
    let needs_bootstrap = !path.exists() || fs::metadata(path)?.len() == 0;
    if needs_bootstrap {
        self.save_to_path(path)?;
        return Ok(());
    }
    let mut file = OpenOptions::new().append(true).open(path)?;
    writeln!(file, "{}", message_record(message).render())?;
    Ok(())
}
```

文件格式为 JSON Lines，支持三种 record type：

- `session_meta`：会话元数据（version、session_id、fork、workspace_root）
- `message`：对话消息（role、blocks、usage）
- `compaction`：压缩摘要
- `prompt_history`：用户 prompt 历史

这保证了即使进程崩溃，已写入的 message 也不会丢失。

### 工作区隔离的 Session Store

`session_control.rs` 中的 `SessionStore`（[`L19-L63`](/rust/crates/runtime/src/session_control.rs#L19-L63)）实现了按工作区指纹隔离的会话目录：

```rust
pub struct SessionStore {
    sessions_root: PathBuf,
    workspace_root: PathBuf,
}

impl SessionStore {
    pub fn from_cwd(cwd: impl AsRef<Path>) -> Result<Self, SessionControlError> {
        let sessions_root = cwd
            .join(".claw")
            .join("sessions")
            .join(workspace_fingerprint(cwd));
        fs::create_dir_all(&sessions_root)?;
        Ok(Self { sessions_root, ... })
    }
}
```

`workspace_fingerprint`（[`L246-L254`](/rust/crates/runtime/src/session_control.rs#L246-L254)）使用 FNV-1a 64-bit 哈希生成 16 字符十六进制摘要：

```rust
pub fn workspace_fingerprint(workspace_root: &Path) -> String {
    let input = workspace_root.to_string_lossy();
    let mut hash = 0xcbf2_9ce4_8422_2325_u64;
    for byte in input.as_bytes() {
        hash ^= u64::from(*byte);
        hash = hash.wrapping_mul(0x0100_0000_01b3);
    }
    format!("{hash:016x}")
}
```

同一目录总是生成相同的 fingerprint，不同目录天然隔离。这避免了多项目并行时 session 文件互相覆盖。

### 会话恢复与 Fork

- **恢复**：`SessionStore::load_session` → `Session::load_from_path`（[`session.rs#L151-L159`](/rust/crates/runtime/src/session.rs#L151-L159)）支持从 JSON 或 JSONL 恢复。
- **Fork**：`SessionStore::fork_session`（[`session_control.rs#L218-L238`](/rust/crates/runtime/src/session_control.rs#L218-L238)）基于原会话创建新的 `session_id`，保留完整消息历史，同时记录 `parent_session_id` 和可选 `branch_name`。这是 multi-agent / worktree 协作场景下的重要记忆分叉机制。

---

## 上下文压缩：记忆与 Token 预算的联动

当对话变长时，`ConversationRuntime` 会在 `run_turn` 结束后自动检查 token 阈值（默认 100k input tokens），触发 `maybe_auto_compact`（[`conversation.rs#L548-L568`](/rust/crates/runtime/src/conversation.rs#L548-L568)）：

```rust
fn maybe_auto_compact(&mut self) -> Option<AutoCompactionEvent> {
    if self.usage_tracker.cumulative_usage().input_tokens
        < self.auto_compaction_input_tokens_threshold
    {
        return None;
    }
    let result = compact_session(
        &self.session,
        CompactionConfig { max_estimated_tokens: 0, ..CompactionConfig::default() },
    );
    if result.removed_message_count == 0 {
        return None;
    }
    self.session = result.compacted_session;
    Some(AutoCompactionEvent { removed_message_count: result.removed_message_count })
}
```

`compact_session`（[`compact.rs#L96-L139`](/rust/crates/runtime/src/compact.rs#L96-L139)）将较早的消息压缩成一条 `MessageRole::System` 的摘要消息，保留最近 N 条消息（默认 4 条）。这相当于给 Session 做了一层"记忆摘要"，既保留了关键上下文，又腾出了 token 预算。

---

## CLI 层的记忆可见性

在 `rusty-claude-cli/src/main.rs` 中，`/memory` slash command 会调用 `render_memory_report`（[`L5057-L5082`](/rust/crates/rusty-claude-cli/src/main.rs#L5057-L5082)），将当前 `ProjectContext` 中发现的指令文件以报告形式展示给用户：

```rust
fn render_memory_report() -> Result<String, Box<dyn std::error::Error>> {
    let cwd = env::current_dir()?;
    let project_context = ProjectContext::discover(&cwd, DEFAULT_DATE)?;
    let mut lines = vec![format!(
        "Memory\n  Working directory {}\n  Instruction files {}",
        cwd.display(),
        project_context.instruction_files.len()
    )];
    // ... 列出每个文件的路径和首行预览
    Ok(lines.join("\n"))
}
```

这提供了人机双可见的“记忆”——模型通过 prompt 读取，用户通过 `/memory` 命令查看。

---

## Rust 实现与上游 TypeScript 的差异

| 特性 | TypeScript 上游 | `claw-code` Rust 实现 |
|------|----------------|----------------------|
| 记忆目录 | `~/.claude/projects/<repo>/memory/` | 尚未完整实现；目前以 `.claw/` 下的 `sessions/` 和指令文件为主 |
| MEMORY.md | 每次对话加载，上限 200 行 / 25KB | 未引入；以 `ProjectContext::instruction_files` 为等价入口 |
| 四类型分类法 | user / feedback / project / reference | 未引入；当前所有上下文统一走 `CLAUDE.md` / `.claw/instructions.md` |
| Sonnet 侧查询召回 | `findRelevantMemories.ts` 的 `selectRelevantMemories()` | 未引入；所有指令文件在启动时全量加载 |
| KAIROS 日志 | `logs/YYYY/MM/YYYY-MM-DD.md` | 未引入 |
| Session 持久化 | 有 | **已实现**：JSONL 增量写入 + `SessionStore` 工作区隔离 |
| 自动压缩 | 有 | **已实现**：`compact_session` + `maybe_auto_compact` |
| 会话 Fork | 有 (worktree) | **已实现**：`Session::fork` + `SessionStore::fork_session` |

当前 `claw-code` 的记忆系统可以理解为：

- **已完成**：基于文件的 instruction context 注入、session 持久化/恢复/fork、工作区隔离、token 压缩。
- **尚未完成**：跨仓库的 `~/.claude/projects/` 目录结构、`MEMORY.md` 索引、四类型分类、Sonnet 智能召回、KAIROS 日志模式。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/prompt.rs`](/rust/crates/runtime/src/prompt.rs) | `ProjectContext`、`SystemPromptBuilder`、指令文件发现、System Prompt 组装 |
| [`/rust/crates/runtime/src/git_context.rs`](/rust/crates/runtime/src/git_context.rs) | `GitContext::detect`、分支/提交/staged 文件读取 |
| [`/rust/crates/runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) | `Session` 结构体、JSONL 持久化、消息追加、fork、恢复 |
| [`/rust/crates/runtime/src/session_control.rs`](/rust/crates/runtime/src/session_control.rs) | `SessionStore`、工作区指纹隔离、会话列表/恢复/fork |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime`、自动压缩触发 `maybe_auto_compact` |
| [`/rust/crates/runtime/src/compact.rs`](/rust/crates/runtime/src/compact.rs) | `compact_session`、摘要生成、上下文压缩 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、`/memory` 命令报告、`ProjectContext` 消费 |
