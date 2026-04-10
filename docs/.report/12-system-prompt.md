# System Prompt 动态组装

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/context/system-prompt) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 从数组到 API 调用：System Prompt 的完整链路

在 `claw-code` 中，System Prompt 同样不是一个写死的字符串，而是一个 `Vec<String>`。它经历了“组装 → 注入运行时 → API 序列化”的完整链路。

```
load_system_prompt()          →  Vec<String>    （prompt.rs 组装内容）
↓
LiveCli::new() / build_runtime() →  注入 ConversationRuntime
↓
ConversationRuntime::run_turn()  →  ApiRequest { system_prompt, messages }
↓
AnthropicRuntimeClient::stream() →  MessageRequest { system: Option<String>, ... }
```

### 源码映射

- [`prompt.rs#L432-L446`](/rust/crates/runtime/src/prompt.rs#L432-L446) 的 `load_system_prompt` 负责收集静态段、项目上下文、配置信息，返回一个 `Vec<String>`。
- [`conversation.rs#L23-L26`](/rust/crates/runtime/src/conversation.rs#L23-L26) 定义了 `ApiRequest`，其 `system_prompt` 字段类型正是 `Vec<String>`。
- [`main.rs#L6312-L6319`](/rust/crates/rusty-claude-cli/src/main.rs#L6312-L6319) 的 `build_system_prompt` 包装了 `load_system_prompt`，供 CLI 启动时调用。
- [`types.rs#L1-L28`](/rust/crates/api/src/types.rs#L1-L28) 的 `MessageRequest` 将 `system_prompt` 拼接为单个 `String` 放入 `system` 字段，最终发向上游 API。

---

## 三阶段管道

在 Rust 实现中，System Prompt 的三阶段管道被显式拆分为三个 crate 的职责边界：

| 阶段 | 负责 crate / 函数 | 输出类型 | 核心行为 |
|------|------------------|---------|---------|
| 1. 组装 | `runtime::prompt::load_system_prompt` | `Vec<String>` | 收集静态段 + 动态段 + 配置 |
| 2. 注入 | `rusty-claude-cli::main::build_runtime` | 绑定到 `ConversationRuntime` | 将 `system_prompt` 传入 runtime |
| 3. API 消费 | `api::client::ProviderClient` | `MessageRequest` | 按 provider 序列化为 Anthropic / OpenAI 格式 |

### 源码映射

- `load_system_prompt` 在 [`prompt.rs#L432-L446`](/rust/crates/runtime/src/prompt.rs#L432-L446) 组装完整 prompt。
- `LiveCli::new()` 在 [`main.rs#L3725-L3751`](/rust/crates/rusty-claude-cli/src/main.rs#L3725-L3751) 调用 `build_system_prompt()` 并将结果存入 `LiveCli` 和 `ConversationRuntime`。
- `ConversationRuntime::run_turn()` 在 [`conversation.rs#L321-L324`](/rust/crates/runtime/src/conversation.rs#L321-L324) 将 `self.system_prompt.clone()` 写入 `ApiRequest`。
- `ProviderClient` 在 [`client.rs#L10-L48`](/rust/crates/api/src/client.rs#L10-L47) 根据模型名选择 `AnthropicClient` 或 `OpenAiCompatClient`，两者都消费 `MessageRequest.system`。

---

## SystemPrompt 类型设计

上游 TypeScript 使用 branded type `SystemPrompt = readonly string[] & { readonly __brand: 'SystemPrompt' }` 防止普通数组被误传。在 Rust 中，没有运行时 brand，但类型系统通过以下方式实现了等价的防御：

- `load_system_prompt` 返回 `Vec<String>`，但调用链很短且完全显式——从 `prompt.rs` → `main.rs` → `conversation.rs` → `api` crate，每一步的类型签名都强制要求 `Vec<String>` 或 `ApiRequest`。
- `ApiRequest` struct 明确区分了 `system_prompt: Vec<String>` 和 `messages: Vec<ConversationMessage>`，在编译期杜绝了消息与系统提示词混用。

### 源码映射

- `ApiRequest` 定义在 [`conversation.rs#L23-L26`](/rust/crates/runtime/src/conversation.rs#L23-L26)。
- `MessageRequest` 定义在 [`types.rs#L1-L28`](/rust/crates/api/src/types.rs#L1-L28)，`system` 字段为 `Option<String>`，与 `messages` 完全分离。

---

## `load_system_prompt()`：内容组装的全景

`claw-code` 的 `load_system_prompt()` 是 System Prompt 的核心工厂函数。它会返回一个有序数组，包含以下内容：

1. **Intro Section** —— 角色声明（`get_simple_intro_section`）
2. **Output Style**（可选）—— 输出风格定制
3. **System Rules** —— 系统级行为约束（`get_simple_system_section`）
4. **Doing Tasks** —— 任务执行规范（`get_simple_doing_tasks_section`）
5. **Actions** —— 操作建议（`get_actions_section`）
6. **DYNAMIC BOUNDARY** —— `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 分界标记
7. **Environment Section** —— 模型、工作目录、日期、平台
8. **Project Context** —— git 状态、CLAUDE.md 内容、diff 快照
9. **Runtime Config** —— `.claw.json` / `.claw/settings.json` 加载记录

### 源码映射

- `SystemPromptBuilder::build()` 在 [`prompt.rs#L144-L166`](/rust/crates/runtime/src/prompt.rs#L144-L166) 按上述顺序拼接所有 section。
- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 常量定义在 [`prompt.rs#L40`](/rust/crates/runtime/src/prompt.rs#L40)。
- 各静态 section 的生成函数：
  - `get_simple_intro_section`：[`prompt.rs#L469-L478`](/rust/crates/runtime/src/prompt.rs#L469-L478)
  - `get_simple_system_section`：[`prompt.rs#L480-L494`](/rust/crates/runtime/src/prompt.rs#L480-L494)
  - `get_simple_doing_tasks_section`：[`prompt.rs#L496-L510`](/rust/crates/runtime/src/prompt.rs#L496-L510)
  - `get_actions_section`：[`prompt.rs#L512-L518`](/rust/crates/runtime/src/prompt.rs#L512-L518)
- `environment_section()` 在 [`prompt.rs#L173-L195`](/rust/crates/runtime/src/prompt.rs#L173-L195) 组装环境信息。

---

## 动态区的 Section 注册表

与上游 TypeScript 的 `systemPromptSection()` / `DANGEROUS_uncachedSystemPromptSection()` 注册表不同，`claw-code` 采用了一种更精简的 Builder 模式：所有动态内容在 `SystemPromptBuilder` 初始化时一次性计算，没有 per-turn 的 section 缓存/重新解析逻辑。`ProjectContext` 和 `RuntimeConfig` 在 `load_system_prompt` 时被完整收集，随后保持不变。

这意味着 Rust 实现中没有显式的“危险 uncached section”概念——因为整个 `system_prompt` 是构建完成后以 `Vec<String>` 形式绑定到 `ConversationRuntime` 的。如果在会话期间配置或 MCP 发生变化，当前实现需要重建整个 `ConversationRuntime` 才能生效——这意味着 `system_prompt` 在会话生命周期内基本是静态的。

### 源码映射

- `ProjectContext::discover_with_git()` 在 [`prompt.rs#L81-L91`](/rust/crates/runtime/src/prompt.rs#L81-L91) 一次性收集指令文件和 git 上下文。
- `ConfigLoader::load()` 在 [`config.rs#L271-L340`](/rust/crates/runtime/src/config.rs#L271-L340) 读取并合并所有配置文件。
- `SystemPromptBuilder` 在 [`prompt.rs#L95-L107`](/rust/crates/runtime/src/prompt.rs#L95-L107) 定义，Builder 方法如 `with_project_context`、`with_runtime_config` 在 L125-L135。

---

## `CLAUDE_CODE_SIMPLE` 快速路径

当 `CLAUDE_CODE_SIMPLE` 环境变量生效时，System Prompt 会被大幅简化。在 `claw-code` 中，这个快速路径没有单独的条件分支函数，而是被上游实现为极简 prompt。Rust 版本中虽然未显式看到 `CLAUDE_CODE_SIMPLE` 的处理分支，但 `SystemPromptBuilder` 的设计天然支持类似简化：如果 `output_style_name` 为空，`get_simple_intro_section` 会生成更精简的 intro。

实际上更精简的路径在 Rust 实现中尚未完全对齐上游，但 `SystemPromptBuilder` 的模块化 section 设计使得未来增加 `simple` 模式只需跳过部分 section 即可。

### 审视角：静态 Prompt 的上下文漂移

因为 `system_prompt` 在 `ConversationRuntime` 初始化后保持不变，它无法反映会话期间发生的项目状态变化。例如，用户在第 3 轮执行了一次 `git commit`，但 system prompt 中注入的 `git status` 和 `recent_commits` 仍然基于启动时的快照。这意味着模型在后续轮次中可能对代码库状态产生"时间冻结"的误解。上游 TypeScript 版本虽然没有在每次 turn 都重建 prompt，但某些 section（如 git context）会被标记为 uncached 并在特定时机刷新。Rust 实现目前缺乏这种增量刷新机制，是准确性与性能之间的一个保守折中。

---

## 五级优先级与有效 System Prompt

上游 TypeScript 的 `buildEffectiveSystemPrompt()` 实现了 Override > Coordinator > Agent > Custom > Default 五级优先级。`claw-code` 的 Rust 实现目前将优先级收敛到了 CLI 层：

- `--system-prompt` 参数（Custom）直接由 CLI 解析并传入 `build_system_prompt` 的包装逻辑。
- `livecli` / `agent` 模式在 `main.rs` 中各自调用自己的 prompt 构建路径。

换言之，Rust 实现中没有单一函数来裁决五级优先级，而是将优先级拆解到了 CLI 入口的不同 match arm 中。这种设计更直接，但少了上游的集中式策略注册表。

### 源码映射

- `main.rs` 中 `Prompt` 和 `Repl` 的 match 分发在 [`main.rs#L180-L240`](/rust/crates/rusty-claude-cli/src/main.rs#L180-L240) 附近。
- `build_system_prompt()` 在 [`main.rs#L6312-L6319`](/rust/crates/rusty-claude-cli/src/main.rs#L6312-L6319)。

---

## Provider 系统概述

`claw-code` 的 `api` crate 根据模型名称自动选择 provider，支持 Anthropic（1P）和 OpenAI 兼容层（3P，覆盖 xAI、DashScope 等）。Provider 选择直接决定了：

- **beta headers 可用性**：Anthropic 原生 API 独享部分实验性功能。
- **缓存策略**：Anthropic 支持 Prompt Cache；OpenAI 兼容层目前仅有 completion 缓存（`prompt_cache.rs` 的本地 completion cache）。
- **token 计数方式**：Anthropic 从流式响应中回传精确 token；OpenAI 兼容层使用估算。

### 源码映射

- `detect_provider_kind` 与 `resolve_model_alias` 在 `api/src/providers/mod.rs`（provider 分发核心）。
- `ProviderClient::from_model()` 在 [`client.rs#L17-L19`](/rust/crates/api/src/client.rs#L17-L19)。
- `ProviderClient::from_model_with_anthropic_auth()` 的完整 match 分支在 [`client.rs#L21-L48`](/rust/crates/api/src/client.rs#L21-L47)。

---

## 缓存策略：分块、标记、命中

`claw-code` 的缓存策略比上游更收敛。当前实现并未接入 Anthropic 服务器端的 Prompt Cache（`cache_control`），而是完全依赖一层**本地 completion cache**（`prompt_cache.rs`），用于缓存完全相同请求（system + messages + tools）的模型响应，避免重复支付 API 费用。

### 本地 Prompt Cache 机制

`api/src/prompt_cache.rs` 实现了一个基于文件系统的 completion cache：

- 以请求哈希（FNV-1a）为键，缓存模型响应到 `~/.claude/cache/prompt-cache/<session>/completions/`。
- 默认 completion TTL 为 30 秒，prompt TTL 为 5 分钟。
- 通过对比请求的 `model_hash`、`system_hash`、`tools_hash`、`messages_hash` 检测缓存破坏（cache break）。

### 源码映射

- `PromptCache` struct 定义在 [`prompt_cache.rs#L109-L134`](/rust/crates/api/src/prompt_cache.rs#L109-L134)。
- `detect_cache_break()` 在 [`prompt_cache.rs#L314-L384`](/rust/crates/api/src/prompt_cache.rs#L314-L384)，是缓存失效分析的核心。
- `TrackedPromptState` 在 [`prompt_cache.rs#L269-L294`](/rust/crates/api/src/prompt_cache.rs#L269-L294) 记录每次请求的指纹状态。
- `RequestFingerprints::from_request()` 在 [`prompt_cache.rs#L304-L313`](/rust/crates/api/src/prompt_cache.rs#L304-L313) 计算请求 fingerprint。

---

## `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 的放置

`claw-code` 的 `SystemPromptBuilder::build()` 会在静态 section 与动态 section 之间插入 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 标记。这个标记在 Rust 实现中有两个作用：

1. **语义分界**：让调试者（和人）能一眼看出哪些是恒定的 scaffolding，哪些是随项目/会话变化的运行时上下文。
2. **未来扩展**：为未来接入 Anthropic 的 `cache_control` 分块策略预留了明确的切分点。当前实现尚未根据 Boundary 做自动 cache scope 标记，但数据结构已经 readiness。

### 源码映射

- `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 常量定义在 [`prompt.rs#L40`](/rust/crates/runtime/src/prompt.rs#L40)。
- 插入位置在 `SystemPromptBuilder::build()` 的 [`prompt.rs#L153`](/rust/crates/runtime/src/prompt.rs#L153)。

---

## 上下文注入：Project Context 与 Runtime Config

System Prompt 数组本身不包含 git 状态或 CLAUDE.md 内容。这些上下文通过 `ProjectContext` 和 `RuntimeConfig` 两个结构注入。

### Project Context

`ProjectContext` 在 `prompt.rs#L55-L63` 定义，包含：

- `cwd`：当前工作目录
- `current_date`：日期字符串
- `git_status`：`git status --short --branch` 的快照
- `git_diff`：staged + unstaged diff
- `git_context`：分支名、最近 5 条 commit、staged files
- `instruction_files`：发现的 CLAUDE.md / `.claw/instructions.md` 等指令文件

### 源码映射

- `ProjectContext` 定义在 [`prompt.rs#L55-L63`](/rust/crates/runtime/src/prompt.rs#L55-L63)。
- `ProjectContext::discover_with_git()` 在 [`prompt.rs#L82-L91`](/rust/crates/runtime/src/prompt.rs#L82-L91) 完成收集。
- `render_project_context()` 在 [`prompt.rs#L288-L328`](/rust/crates/runtime/src/prompt.rs#L288-L328) 将其格式化为 `# Project context` section。
- `GitContext` 定义在 [`git_context.rs#L12-L18`](/rust/crates/runtime/src/git_context.rs#L12-L17)。
- `GitContext::detect()` 在 [`git_context.rs#L26-L42`](/rust/crates/runtime/src/git_context.rs#L26-L42) 通过 `git rev-parse --is-inside-work-tree` 作为 quick gate，随后读取分支、commit、staged files。

### Runtime Config

`RuntimeConfig` 由 `ConfigLoader` 从以下路径加载并深度合并：

- `~/.claw.json`（User 级旧配置）
- `~/.claw/settings.json`（User 级配置）
- `./.claw.json`（Project 级旧配置）
- `./.claw/settings.json`（Project 级配置）
- `./.claw/settings.local.json`（Local 级覆盖）

### 源码映射

- `ConfigLoader::discover()` 在 [`config.rs#L242-L268`](/rust/crates/runtime/src/config.rs#L242-L268) 返回上述 5 个候选路径。
- `ConfigLoader::load()` 在 [`config.rs#L271-L340`](/rust/crates/runtime/src/config.rs#L271-L340) 读取、验证、合并配置。
- `render_config_section()` 在 [`prompt.rs#L448-L467`](/rust/crates/runtime/src/prompt.rs#L448-L467) 将加载结果渲染为 `# Runtime config` section。

---

## `CLAUDE.md`：项目级知识注入

`discover_instruction_files()` 会从 `cwd` 向上遍历目录树，收集每一段目录层级上的 `CLAUDE.md`、`CLAUDE.local.md`、`.claw/CLAUDE.md`、`.claw/instructions.md`。收集完成后进行去重（基于内容哈希），并按目录深度由浅到深排序。

### 源码映射

- `discover_instruction_files()` 在 [`prompt.rs#L203-L223`](/rust/crates/runtime/src/prompt.rs#L203-L223) 实现向上遍历与候选文件收集。
- `push_context_file()` 在 [`prompt.rs#L226-L236`](/rust/crates/runtime/src/prompt.rs#L226-L236) 安全读取文件（忽略 `NotFound`）。
- `dedupe_instruction_files()` 在 [`prompt.rs#L353-L368`](/rust/crates/runtime/src/prompt.rs#L353-L368) 基于 `stable_content_hash` 去重。
- `render_instruction_files()` 在 [`prompt.rs#L330-L351`](/rust/crates/runtime/src/prompt.rs#L330-L351) 渲染为 `# Claude instructions` section，并实施 `MAX_TOTAL_INSTRUCTION_CHARS = 12_000` 的总预算截断。
- `truncate_instruction_content()` 在 [`prompt.rs#L393-L403`](/rust/crates/runtime/src/prompt.rs#L393-L403) 实施单文件 `MAX_INSTRUCTION_FILE_CHARS = 4_000` 的硬限制。

---

## 设计洞察：为什么是 `Vec<String>` 而非单个 `String`

`claw-code` 将 System Prompt 设计为 `Vec<String>` 而非单一 `String`，直接继承了上游的分块哲学：

- **调试可读性**：每个 section 独立成一个字符串，打印 `system-prompt` 子命令时可以直接按 section 输出 JSON 数组。
- **未来缓存分块**：虽然目前 Rust 实现未按 section 附加 `cache_control`，但数组结构天然适合未来将“静态前缀”和“动态后缀”拆分为不同的 Anthropic `TextBlock`，从而复用 Prompt Cache。
- **增量更新**：如果未来要在 turn 之间只替换 `project_context` section，数组结构让替换成本远低于重新拼接整个字符串。

### 源码映射

- `SystemPromptBuilder::build()` 返回 `Vec<String>`：[`prompt.rs#L144`](/rust/crates/runtime/src/prompt.rs#L144)。
- `SystemPromptBuilder::render()` 用 `"\n\n".join(...)` 拼接：[`prompt.rs#L169-L171`](/rust/crates/runtime/src/prompt.rs#L169-L171)。
- `print_system_prompt()` 输出既支持纯文本拼接，也支持 JSON section 数组：[`main.rs#L2172-L2195`](/rust/crates/rusty-claude-cli/src/main.rs#L2172-L2195)。

---

## 兼容层：OpenAI 与 Anthropic

`claw-code` 的 `api` crate 通过 `ProviderClient` 封装了多后端支持。

### Anthropic 原生路径

`AnthropicClient` 将 `MessageRequest.system`（由 `Vec<String>` 拼接而成的单个 `String`）直接放入 Anthropic Messages API 的 `system` 字段。

### OpenAI 兼容路径

`OpenAiCompatClient`（覆盖 xAI、OpenAI、DashScope 等）将相同的消息结构转换为 OpenAI Chat Completions 格式。System Prompt 被放入 messages 数组首条的 `role: "system"` 中。由于 OpenAI 格式不支持 `cache_control`，Prompt Cache 的细粒度分块不在此路径生效，但本地 `PromptCache` completion cache 仍然工作。

### 源码映射

- `MessageRequest` 的 `system: Option<String>` 定义在 [`types.rs#L11`](/rust/crates/api/src/types.rs#L11)。
- `ProviderClient` 分发逻辑在 [`client.rs#L21-L48`](/rust/crates/api/src/client.rs#L21-L47)。
- Anthropic provider 实现位于 `api/src/providers/anthropic.rs`。
- OpenAI 兼容 provider 实现位于 `api/src/providers/openai_compat.rs`。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/prompt.rs`](/rust/crates/runtime/src/prompt.rs) | `load_system_prompt`、`SystemPromptBuilder`、`ProjectContext`、指令文件发现与渲染 |
| [`/rust/crates/runtime/src/git_context.rs`](/rust/crates/runtime/src/git_context.rs) | `GitContext`、分支/commit/staged files 检测 |
| [`/rust/crates/runtime/src/config.rs`](/rust/crates/runtime/src/config.rs) | `ConfigLoader`、`RuntimeConfig`、配置合并与验证 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `ConversationRuntime`、`ApiRequest`、`run_turn` 中 system prompt 的使用 |
| [`/rust/crates/runtime/src/session.rs`](/rust/crates/runtime/src/session.rs) | `Session`、`ConversationMessage`、`ContentBlock` 消息模型 |
| [`/rust/crates/api/src/client.rs`](/rust/crates/api/src/client.rs) | `ProviderClient`、provider 分发 |
| [`/rust/crates/api/src/types.rs`](/rust/crates/api/src/types.rs) | `MessageRequest`、`Usage`、API 请求/响应类型 |
| [`/rust/crates/api/src/prompt_cache.rs`](/rust/crates/api/src/prompt_cache.rs) | 本地 Prompt Cache、缓存命中与失效检测 |
| [`/rust/crates/rusty-claude-cli/src/main.rs`](/rust/crates/rusty-claude-cli/src/main.rs) | CLI 入口、`build_system_prompt`、`print_system_prompt` |
