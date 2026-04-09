# Hooks 扩展机制

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/extensibility/hooks) 的目录结构进一步深化，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 一句话定义

Hooks 是 Claude Code 的**可插拔拦截机制**。它允许用户在关键环节注入自定义脚本，实现对工具调用、权限决策和输入参数的拦截与修改。

### 支持的 Hook 阶段

| Hook 事件 | 触发时机 | 典型用途 |
| --- | --- | --- |
| `PreToolUse` | 工具执行前 | 拦截、修改工具输入、覆盖权限决策 |
| `PostToolUse` | 工具成功执行后 | 后置处理、审计、结果增强 |
| `PostToolUseFailure` | 工具执行失败后 | 错误恢复、告警、日志上报 |

---

## Hook 生命周期在源码中的位置

Hook 的核心实现位于 [`runtime/src/hooks.rs`](/rust/crates/runtime/src/hooks.rs)。它的职责是从配置中读取命令列表，在指定阶段以子进程方式执行，并解析子进程的标准输出作为拦截结果。

### HookEvent 枚举

[`hooks.rs#L18-L34`](/rust/crates/runtime/src/hooks.rs#L18-L34) 定义了三个生命周期事件：

```rust
pub enum HookEvent {
    PreToolUse,
    PostToolUse,
    PostToolUseFailure,
}
```

每个事件都对应 `HookRunner` 上的一个公开方法：

- `run_pre_tool_use` / `run_pre_tool_use_with_context`
- `run_post_tool_use` / `run_post_tool_use_with_context`
- `run_post_tool_use_failure` / `run_post_tool_use_failure_with_context`

这些方法本质上都会进入同一个私有方法 [`run_commands`](/rust/crates/runtime/src/hooks.rs#L310-L411)，依次执行配置在当前阶段的所有命令。

---

## Hook 在 Agentic Loop 中的调用位置

真正决定 Hook 何时生效的代码在 [`runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) 的 `run_turn` 主循环中。对于模型发起的每一次 `ToolUse`，执行顺序如下：

1. **PreToolUse Hook**（[`conversation.rs#L370-L371`](/rust/crates/runtime/src/conversation.rs#L370-L371)）
2. 权限检查（`permission_policy.authorize_with_context`）
3. 工具实际执行（`tool_executor.execute`）
4. **PostToolUse Hook 或 PostToolUseFailure Hook**（[`conversation.rs#L434-L438`](/rust/crates/runtime/src/conversation.rs#L434-L438)）

### 详细调用链

```rust
let pre_hook_result = self.run_pre_tool_use_hook(&tool_name, &input);
let effective_input = pre_hook_result
    .updated_input()
    .map_or_else(|| input.clone(), ToOwned::to_owned);
let permission_context = PermissionContext::new(
    pre_hook_result.permission_override(),
    pre_hook_result.permission_reason().map(ToOwned::to_owned),
);
```

参见 [`conversation.rs#L370-L377`](/rust/crates/runtime/src/conversation.rs#L370-L377)。

`PreToolUse` 是唯一能够在**权限检查之前**影响后续流程的阶段。它通过 `updated_input` 修改原始工具输入，也可以通过 `permission_override` 覆盖默认的权限决策（Allow / Deny / Ask）。

### PreHook 的决策分支

在 [`conversation.rs#L379-L411`](/rust/crates/runtime/src/conversation.rs#L379-L411) 中，系统按以下优先级处理 PreHook 的结果：

1. `is_cancelled()` → `PermissionOutcome::Deny`
2. `is_failed()` → `PermissionOutcome::Deny`
3. `is_denied()` → `PermissionOutcome::Deny`
4. 否则进入正常权限策略检查（Prompt 模式会弹窗确认）

这意味着：即使把权限模式设为 `Allow`，一个返回 `deny` 的 PreHook 仍然可以**强制拦截**工具调用。

---

## Hook 与权限系统的交互

Hook 脚本可以通过标准输出返回一个 JSON 对象，向运行时传递更精细的指令。解析逻辑在 [`hooks.rs#L535-L589`](/rust/crates/runtime/src/hooks.rs#L535-L589) 的 `parse_hook_output` 函数中。

### Hook 输出 Schema（JSON）

```json
{
  "systemMessage": "用户可见的提示信息",
  "reason": "额外原因说明",
  "continue": false,
  "decision": "block",
  "hookSpecificOutput": {
    "permissionDecision": "allow" | "deny" | "ask",
    "permissionDecisionReason": "决策原因",
    "updatedInput": { /* 新的工具参数 */ }
  }
}
```

关键字段映射到源码：

| 字段 | 源码处理位置 | 作用 |
|------|--------------|------|
| `systemMessage` / `reason` | [`hooks.rs#L549-L553`](/rust/crates/runtime/src/hooks.rs#L549-L553) | 合并为运行时消息反馈给模型 |
| `continue: false` 或 `decision: block` | [`hooks.rs#L555-L558`](/rust/crates/runtime/src/hooks.rs#L555-L558) | 标记 `deny = true`，拦截工具执行 |
| `permissionDecision` | [`hooks.rs#L565-L571`](/rust/crates/runtime/src/hooks.rs#L565-L571) | 转换为 `PermissionOverride::Allow/Deny/Ask` |
| `updatedInput` | [`hooks.rs#L579-L580`](/rust/crates/runtime/src/hooks.rs#L579-L580) | 覆盖原工具输入的 JSON 值 |

### 退出码语义

[`hooks.rs#L442-L470`](/rust/crates/runtime/src/hooks.rs#L442-L470) 定义了命令退出码的处理规则：

| 退出码 | 含义 |
|--------|------|
| `0` | 允许继续执行；若 JSON 中 `deny = true` 则仍拦截 |
| `2` | **显式拒绝**（Deny），无论 JSON 内容如何 |
| 其他非零 | **失败**（Failed），以错误形式中断该工具调用 |
| 信号终止 | **失败**（Failed） |

---

## Hook 环境变量与输入协议

Hook 子进程通过环境变量和标准输入接收上下文。

### 环境变量

在 [`hooks.rs#L428-L434`](/rust/crates/runtime/src/hooks.rs#L428-L434) 中，运行时设置了以下变量：

```rust
child.env("HOOK_EVENT", event.as_str());
child.env("HOOK_TOOL_NAME", tool_name);
child.env("HOOK_TOOL_INPUT", tool_input);
child.env("HOOK_TOOL_IS_ERROR", if is_error { "1" } else { "0" });
if let Some(tool_output) = tool_output {
    child.env("HOOK_TOOL_OUTPUT", tool_output);
}
```

### 标准输入（JSON Payload）

[`hooks.rs#L591-L616`](/rust/crates/runtime/src/hooks.rs#L591-L616) 的 `hook_payload` 函数描述了写入子进程 stdin 的 JSON 结构：

```rust
json!({
    "hook_event_name": "PreToolUse",
    "tool_name": tool_name,
    "tool_input": parse_tool_input(tool_input),      // 解析后的 JSON 对象
    "tool_input_json": tool_input,                   // 原始字符串
    "tool_output": tool_output,
    "tool_result_is_error": is_error,
})
```

注意 `PostToolUseFailure` 的 payload 稍有不同：它使用 `tool_error` 字段代替 `tool_output`。

---

## Hook 的并发与可取消性

### HookAbortSignal

[`hooks.rs#L59-L78`](/rust/crates/runtime/src/hooks.rs#L59-L78) 提供了一个基于 `AtomicBool` 的取消信号：

```rust
pub struct HookAbortSignal {
    aborted: Arc<AtomicBool>,
}
```

调用方可以在外部调用 `abort()`，主循环中以 20ms 的轮询间隔检查子进程是否完成，若检测到取消则 `kill` 子进程（[`hooks.rs#L695-L699`](/rust/crates/runtime/src/hooks.rs#L695-L699)）。

### 顺序执行与短路

每个阶段的多个 Hook 命令是**串行执行**的。如果前一个命令返回 `Deny`、`Failed` 或 `Cancelled`，后续命令会被**短路跳过**。参见 [`hooks.rs#L361-L407`](/rust/crates/runtime/src/hooks.rs#L361-L407) 的 `match HookCommandOutcome` 逻辑。

---

## Hook 配置格式

Hook 配置通过 `.claw.json` 或 `.claw/settings.json` 定义。解析代码在 [`runtime/src/config.rs`](/rust/crates/runtime/src/config.rs)。

### 配置 Schema

```json
{
  "hooks": {
    "PreToolUse": ["node /path/to/pre-hook.js"],
    "PostToolUse": ["cat"],
    "PostToolUseFailure": ["python /path/to/failure-hook.py"]
  }
}
```

### 配置解析入口

[`config.rs#L750-L770`](/rust/crates/runtime/src/config.rs#L750-L770)：

```rust
fn parse_optional_hooks_config_object(
    object: &BTreeMap<String, JsonValue>,
    context: &str,
) -> Result<RuntimeHookConfig, ConfigError> {
    let Some(hooks_value) = object.get("hooks") else {
        return Ok(RuntimeHookConfig::default());
    };
    let hooks = expect_object(hooks_value, context)?;
    Ok(RuntimeHookConfig {
        pre_tool_use: optional_string_array(hooks, "PreToolUse", context)?.unwrap_or_default(),
        post_tool_use: optional_string_array(hooks, "PostToolUse", context)?.unwrap_or_default(),
        post_tool_use_failure: optional_string_array(hooks, "PostToolUseFailure", context)?
            .unwrap_or_default(),
    })
}
```

用户级配置与项目级配置会**深度合并**，同一阶段的命令列表采用去重追加策略（`extend_unique`），见 [`config.rs#L598-L605`](/rust/crates/runtime/src/config.rs#L598-L605)。

---

## PostToolUse 与 PostToolUseFailure 的差异

在 `run_turn` 中，工具执行成功后运行 `PostToolUse`，执行失败后运行 `PostToolUseFailure`，两者**不会同时触发**。参见 [`conversation.rs#L427-L438`](/rust/crates/runtime/src/conversation.rs#L427-L438)：

```rust
let post_hook_result = if is_error {
    self.run_post_tool_use_failure_hook(&tool_name, &effective_input, &output)
} else {
    self.run_post_tool_use_hook(&tool_name, &effective_input, &output, false)
};
```

如果 PostHook 返回 `denied`、`failed` 或 `cancelled`，运行时会将工具结果标记为 `is_error = true`，并把 Hook 反馈追加到输出中（[`conversation.rs#L439-L447`](/rust/crates/runtime/src/conversation.rs#L439-L447)）。

反馈合并逻辑在 [`conversation.rs#L744-L755`](/rust/crates/runtime/src/conversation.rs#L744-L755)：

```rust
fn merge_hook_feedback(messages: &[String], output: String, is_error: bool) -> String {
    let label = if is_error { "Hook feedback (error)" } else { "Hook feedback" };
    sections.push(format!("{label}:\n{}", messages.join("\n")));
    sections.join("\n\n")
}
```

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/runtime/src/hooks.rs`](/rust/crates/runtime/src/hooks.rs) | `HookEvent`、`HookRunner`、`HookRunResult`、输出解析、子进程执行 |
| [`/rust/crates/runtime/src/conversation.rs`](/rust/crates/runtime/src/conversation.rs) | `run_turn` 中的 Hook 调用链与反馈合并 |
| [`/rust/crates/runtime/src/config.rs`](/rust/crates/runtime/src/config.rs) | `RuntimeHookConfig`、配置解析与合并策略 |
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionOverride`、`PermissionContext`、`PermissionOutcome` |
| [`/rust/crates/runtime/src/plugin_lifecycle.rs`](/rust/crates/runtime/src/plugin_lifecycle.rs) | Plugin 生命周期状态机（与 Hook 同属扩展性体系） |
