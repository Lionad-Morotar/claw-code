# Plan Mode 计划模式技术报告

> 本报告基于 [Claude Code 中文文档 - Plan Mode](https://ccb.agent-aura.top/docs/safety/plan-mode) 深化分析，并映射到 `claw-code`（Claude Code 的 Rust 重写版）的源码实现。
> 文末附 [源码索引](#源码索引)。

---

## 一句话定义

Plan Mode 是一个**先看后做的安全机制**——通过临时将权限模式切换为只读，强制 AI 在执行写操作前完成代码探索与方案规划，并在用户审批计划文件后才恢复完整执行权限。

---

## 为什么要引入 Plan Mode

当用户说"重构这个模块"时，传统模式下 AI 会立刻开始修改代码。此时用户还不清楚 AI 打算怎么改，等改了一半发现方向不对时，已经造成了破坏性变更。

Plan Mode 的核心设计 rationale 有三点：

1. **防止不可逆的过早写入**：将"探索"与"执行"拆分为两个独立阶段，探索阶段只能使用只读工具，从物理上阻断误写。
2. **强制书面化沟通**：AI 必须生成人类可读的计划文件，用户能在执行前理解、修改或拒绝方案。
3. **最小权限原则**：只读阶段的权限粒度 lowest，仅在用户显式批准后才提升到正常工作模式。

---

## Plan Mode 的核心机制

计划模式通过两个核心工具实现闭环：

| 步骤 | 工具 | 说明 | 是否需要用户审批 |
| --- | --- | --- | --- |
| 1 | `EnterPlanMode` | AI 自主判断或用户触发任务需要规划，调用此工具进入计划模式 | 是 |
| 2 | 探索阶段 | 权限模式切换为 `'plan'`，AI 只能使用 `isReadOnly()` 为 true 的工具 | 否 |
| 3 | `ExitPlanMode` | AI 完成探索后提交计划文件给用户审阅 | 是 |
| 4 | 恢复执行 | 用户批准后，权限模式恢复到进入前的状态，AI 按计划执行 | 否 |

**双重审批是有意为之的设计**：进入审批防止 AI 在用户不知情时突然"隐身"进入限制模式；退出审批则确保用户在恢复完整写入权限前，已经审阅并认可了执行方案。

---

## 源码深度解析

### EnterPlanMode 工具实现

在 `claw-code` 的 Rust 实现中，`EnterPlanMode` 工具的入口位于 [`rust/crates/tools/src/lib.rs#L670-L671`](/rust/crates/tools/src/lib.rs#L670-L671)：

```rust
// 工具注册
name: "EnterPlanMode",
description: "Enable a worktree-local planning mode override and remember the previous local setting for ExitPlanMode.",
```

工具调用分发逻辑在 [`lib.rs#L1218`](/rust/crates/tools/src/lib.rs#L1218)：

```rust
"EnterPlanMode" => from_value::<EnterPlanModeInput>(input).and_then(run_enter_plan_mode),
```

### execute_enter_plan_mode 核心逻辑

[`lib.rs#L4584-L4651`](/rust/crates/tools/src/lib.rs#L4584-L4651) 展示了进入计划模式的完整流程：

```rust
fn execute_enter_plan_mode(_input: EnterPlanModeInput) -> Result<PlanModeOutput, String> {
    let settings_path = config_file_for_scope(ConfigScope::Settings)?;
    let state_path = plan_mode_state_file()?;
    let mut document = read_json_object(&settings_path)?;
    let current_local_mode = get_nested_value(&document, PERMISSION_DEFAULT_MODE_PATH).cloned();
    let current_is_plan =
        matches!(current_local_mode.as_ref(), Some(Value::String(value)) if value == "plan");

    // 1. 检查是否已有计划模式状态
    if let Some(state) = read_plan_mode_state(&state_path)? {
        if current_is_plan {
            // 已在计划模式中，返回成功
            return Ok(PlanModeOutput { ... });
        }
        // 清除旧状态
        clear_plan_mode_state(&state_path)?;
    }

    // 2. 如果已经在 plan 模式但不是由 EnterPlanMode 管理的
    if current_is_plan {
        return Ok(PlanModeOutput {
            message: String::from(
                "Worktree-local plan mode is already enabled outside EnterPlanMode; leaving it unchanged.",
            ),
            ...
        });
    }

    // 3. 保存当前模式状态，用于退出时恢复
    let state = PlanModeState {
        had_local_override: current_local_mode.is_some(),
        previous_local_mode: current_local_mode.clone(),
    };
    write_plan_mode_state(&state_path, &state)?;

    // 4. 修改 settings.local.json，将 defaultMode 设置为 "plan"
    set_nested_value(
        &mut document,
        PERMISSION_DEFAULT_MODE_PATH,
        Value::String(String::from("plan")),
    );
    write_json_object(&settings_path, &document)?;

    Ok(PlanModeOutput {
        success: true,
        operation: String::from("enter"),
        changed: true,
        active: true,
        managed: true,
        message: String::from("Enabled worktree-local plan mode override."),
        ...
    })
}
```

### ExitPlanMode 工具实现

退出计划模式的工具注册在 [`lib.rs#L680-L681`](/rust/crates/tools/src/lib.rs#L680-L681)：

```rust
name: "ExitPlanMode",
description: "Restore or clear the worktree-local planning mode override created by EnterPlanMode.",
```

### execute_exit_plan_mode 核心逻辑

[`lib.rs#L4653-L4721`](/rust/crates/tools/src/lib.rs#L4653-L4721) 展示了退出计划模式的完整流程：

```rust
fn execute_exit_plan_mode(_input: ExitPlanModeInput) -> Result<PlanModeOutput, String> {
    let settings_path = config_file_for_scope(ConfigScope::Settings)?;
    let state_path = plan_mode_state_file()?;
    let mut document = read_json_object(&settings_path)?;
    let current_local_mode = get_nested_value(&document, PERMISSION_DEFAULT_MODE_PATH).cloned();
    let current_is_plan =
        matches!(current_local_mode.as_ref(), Some(Value::String(value)) if value == "plan");

    // 1. 检查是否存在计划模式状态
    let Some(state) = read_plan_mode_state(&state_path)? else {
        return Ok(PlanModeOutput {
            message: String::from("No EnterPlanMode override is active for this worktree."),
            ...
        });
    };

    // 2. 如果当前已不在 plan 模式，清除陈旧状态
    if !current_is_plan {
        clear_plan_mode_state(&state_path)?;
        return Ok(PlanModeOutput {
            message: String::from(
                "Cleared stale EnterPlanMode state because plan mode was already changed outside the tool.",
            ),
            ...
        });
    }

    // 3. 恢复之前的模式设置
    if state.had_local_override {
        if let Some(previous_local_mode) = state.previous_local_mode.clone() {
            set_nested_value(
                &mut document,
                PERMISSION_DEFAULT_MODE_PATH,
                previous_local_mode,
            );
        } else {
            remove_nested_value(&mut document, PERMISSION_DEFAULT_MODE_PATH);
        }
    } else {
        remove_nested_value(&mut document, PERMISSION_DEFAULT_MODE_PATH);
    }
    write_json_object(&settings_path, &document)?;
    
    // 4. 清除计划模式状态文件
    clear_plan_mode_state(&state_path)?;

    Ok(PlanModeOutput {
        success: true,
        operation: String::from("exit"),
        changed: true,
        active: false,
        managed: false,
        message: String::from("Restored the prior worktree-local plan mode setting."),
        ...
    })
}
```

### 计划模式状态持久化

计划模式的状态保存在 `settings.local.json` 同级目录下的 `tool-state/plan-mode.json` 文件中。

状态文件路径函数 [`lib.rs#L5075-L5077`](/rust/crates/tools/src/lib.rs#L5075-L5077)：

```rust
fn plan_mode_state_file() -> Result<PathBuf, String> {
    Ok(config_file_for_scope(ConfigScope::Settings)?
        .parent()
        .ok_or_else(|| String::from("settings.local.json has no parent directory"))?
        .join("tool-state")
        .join("plan-mode.json"))
}
```

状态数据结构 [`lib.rs#L2491-L2496`](/rust/crates/tools/src/lib.rs#L2491-L2496)：

```rust
struct PlanModeState {
    #[serde(rename = "hadLocalOverride")]
    had_local_override: bool,
    #[serde(rename = "previousLocalMode")]
    previous_local_mode: Option<Value>,
}
```

状态读写函数 [`lib.rs#L5083-L5110`](/rust/crates/tools/src/lib.rs#L5083-L5110)：

```rust
fn read_plan_mode_state(path: &Path) -> Result<Option<PlanModeState>, String> {
    match std::fs::read_to_string(path) {
        Ok(contents) => {
            if contents.trim().is_empty() {
                return Ok(None);
            }
            serde_json::from_str(&contents)
                .map(Some)
                .map_err(|error| error.to_string())
        }
        Err(error) if error.kind() == std::io::ErrorKind::NotFound => Ok(None),
        Err(error) => Err(error.to_string()),
    }
}

fn write_plan_mode_state(path: &Path, state: &PlanModeState) -> Result<(), String> {
    if let Some(parent) = path.parent() {
        std::fs::create_dir_all(parent).map_err(|error| error.to_string())?;
    }
    std::fs::write(
        path,
        serde_json::to_string_pretty(state).map_err(|error| error.to_string())?,
    )
    .map_err(|error| error.to_string())
}

fn clear_plan_mode_state(path: &Path) -> Result<(), String> {
    match std::fs::remove_file(path) {
        Ok(()) => Ok(()),
        Err(error) if error.kind() == std::io::ErrorKind::NotFound => Ok(()),
        Err(error) => Err(error.to_string()),
    }
}
```

---

## 权限模式解析

在 `claw-code` 的配置系统中，`"plan"` 模式被解析为 `ReadOnly` 权限级别。参见 [`runtime/src/config.rs#L851-L862`](/rust/crates/runtime/src/config.rs#L851-L862)：

```rust
fn parse_permission_mode_label(
    mode: &str,
    context: &str,
) -> Result<ResolvedPermissionMode, ConfigError> {
    match mode {
        "default" | "plan" | "read-only" => Ok(ResolvedPermissionMode::ReadOnly),
        "acceptEdits" | "auto" | "workspace-write" => Ok(ResolvedPermissionMode::WorkspaceWrite),
        "dontAsk" | "danger-full-access" => Ok(ResolvedPermissionMode::DangerFullAccess),
        other => Err(ConfigError::Parse(format!(
            "{context}: unsupported permission mode {other}"
        ))),
    }
}
```

这意味着当进入 Plan Mode 后，所有写操作都会被拒绝，只有只读工具（Read、Grep、Glob 等）可以执行。

---

## 权限执行器中的 Plan Mode 行为

在 [`runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) 中，`PermissionEnforcer` 根据当前模式执行不同的策略：

### 文件写入检查

[`permission_enforcer.rs#L74-L108`](/rust/crates/runtime/src/permission_enforcer.rs#L74-L108)：

```rust
pub fn check_file_write(&self, path: &str, workspace_root: &str) -> EnforcementResult {
    let mode = self.policy.active_mode();

    match mode {
        PermissionMode::ReadOnly => EnforcementResult::Denied {
            tool: "write_file".to_owned(),
            active_mode: mode.as_str().to_owned(),
            required_mode: PermissionMode::WorkspaceWrite.as_str().to_owned(),
            reason: format!("file writes are not allowed in '{}' mode", mode.as_str()),
        },
        PermissionMode::WorkspaceWrite => {
            if is_within_workspace(path, workspace_root) {
                EnforcementResult::Allowed
            } else {
                EnforcementResult::Denied { ... }
            }
        }
        PermissionMode::Allow | PermissionMode::DangerFullAccess => EnforcementResult::Allowed,
        PermissionMode::Prompt => EnforcementResult::Denied {
            tool: "write_file".to_owned(),
            active_mode: mode.as_str().to_owned(),
            required_mode: PermissionMode::WorkspaceWrite.as_str().to_owned(),
            reason: "file write requires confirmation in prompt mode".to_owned(),
        },
    }
}
```

### Bash 命令检查

[`permission_enforcer.rs#L111-L139`](/rust/crates/runtime/src/permission_enforcer.rs#L111-L139)：

```rust
pub fn check_bash(&self, command: &str) -> EnforcementResult {
    let mode = self.policy.active_mode();

    match mode {
        PermissionMode::ReadOnly => {
            if is_read_only_command(command) {
                EnforcementResult::Allowed
            } else {
                EnforcementResult::Denied {
                    tool: "bash".to_owned(),
                    active_mode: mode.as_str().to_owned(),
                    required_mode: PermissionMode::WorkspaceWrite.as_str().to_owned(),
                    reason: format!(
                        "command may modify state; not allowed in '{}' mode",
                        mode.as_str()
                    ),
                }
            }
        }
        PermissionMode::Prompt => EnforcementResult::Denied {
            tool: "bash".to_owned(),
            active_mode: mode.as_str().to_owned(),
            required_mode: PermissionMode::DangerFullAccess.as_str().to_owned(),
            reason: "bash requires confirmation in prompt mode".to_owned(),
        },
        // WorkspaceWrite, Allow, DangerFullAccess: permit bash
        _ => EnforcementResult::Allowed,
    }
}
```

---

## 完整生命周期

以下是 Plan Mode 从进入到退出的完整时序：

```
用户： "重构这个模块"
  ↓
AI 判断需要规划 → 调用 EnterPlanModeTool
  ↓ 用户审批（Ask 对话框）
execute_enter_plan_mode()
  • 保存当前 mode 到 plan-mode.json
  • 设置 settings.local.json 中 defaultMode 为 "plan"
  ↓
AI 使用 Read/Grep/Glob 探索代码库
  ↓（可能 10+ 轮只读工具调用）
AI 形成方案 → 调用 ExitPlanModeTool({
  plan: "1. 读取 X 文件\n2. 修改 Y 函数\n3. 运行测试"
})
  ↓ 用户审批计划（可编辑计划文件）
execute_exit_plan_mode()
  • 从 plan-mode.json 读取 previousLocalMode
  • 恢复 settings.local.json 中的原始设置
  • 删除 plan-mode.json
  ↓
AI 使用全部工具执行计划
```

### 边界情况

1. **已进入 plan 模式但无状态文件**：说明模式是用户手动设置的，工具会清除陈旧状态但**不会强制覆盖**，尊重用户选择。
2. **退出时发现已不在 plan 模式**：说明用户已通过其他方式（如手动编辑配置）切换了模式，工具仅清理残留状态文件。
3. **进程崩溃或异常退出**：由于状态保存在磁盘，重启后 `ExitPlanMode` 仍能正确读取并恢复之前的模式。

---

## 测试用例验证

在 [`lib.rs#L7986-L8105`](/rust/crates/tools/src/lib.rs#L7986-L8105) 中有两个测试用例验证 EnterPlanMode/ExitPlanMode 的往返行为：

```rust
#[test]
fn enter_and_exit_plan_mode_round_trip_existing_local_override() {
    // 测试场景：工作树原本已有本地覆盖设置
    // 验证进入和退出后能正确恢复原有设置
    let enter = execute_tool("EnterPlanMode", &json!({})).expect("enter plan mode");
    // 验证状态文件已创建
    let state = read_plan_mode_state(&state_path).expect("plan mode state");
    let exit = execute_tool("ExitPlanMode", &json!({})).expect("exit plan mode");
    // 验证设置已恢复
}

#[test]
fn exit_plan_mode_clears_override_when_enter_created_it_from_empty_local_state() {
    // 测试场景：工作树原本没有本地覆盖设置
    // 验证退出后设置被完全清除（而非保留空值）
    let enter = execute_tool("EnterPlanMode", &json!({})).expect("enter plan mode");
    let exit = execute_tool("ExitPlanMode", &json!({})).expect("exit plan mode");
    // 验证设置已被移除
}
```

---

## 源码索引

| 文件 | 核心内容 | 行号 |
|------|----------|------|
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `EnterPlanMode` / `ExitPlanMode` 工具定义 | L670-L681 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | 工具调用分发 | L1218-L1219 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `EnterPlanModeInput` / `ExitPlanModeInput` 结构 | L2182-L2186 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `PlanModeState` / `PlanModeOutput` 结构 | L2491-L2515 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `execute_enter_plan_mode` 实现 | L4584-L4651 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `execute_exit_plan_mode` 实现 | L4653-L4721 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | `plan_mode_state_file` 路径函数 | L5075-L5077 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | 状态读写清除函数 | L5083-L5110 |
| [`/rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) | 测试用例 | L7986-L8105 |
| [`/rust/crates/runtime/src/config.rs`](/rust/crates/runtime/src/config.rs) | `"plan"` 模式解析为 `ReadOnly` | L851-L862 |
| [`/rust/crates/runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) | 文件写入检查 | L74-L108 |
| [`/rust/crates/runtime/src/permission_enforcer.rs`](/rust/crates/runtime/src/permission_enforcer.rs) | Bash 命令检查 | L111-L139 |
| [`/rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs) | `PermissionMode` 枚举定义 | L9-L16 |

---

## 关键设计洞察

1. **状态持久化**：计划模式通过 `plan-mode.json` 文件保存进入前的状态，确保退出时能精确恢复，而不是简单重置为默认值。这种设计避免了"恢复后丢失用户原有的本地覆盖设置"的问题。

2. **工作树局部性**：所有设置都保存在工作树级别的 `settings.local.json` 中，这意味着不同工作树可以有不同的计划模式状态，彼此互不干扰。

3. **非侵入式**：当检测到计划模式已在外部被更改时（`current_is_plan` 但状态文件不存在），工具会清除陈旧状态并优雅退出，而不是强制覆盖用户的选择。这体现了对人工干预的尊重。

4. **双重审批**：进入计划模式需要审批（因为会改变权限行为），退出时提交计划文件同样需要审批（因为会恢复完整权限并开始实际修改）。两个审批节点分别 guarding 两个高杠杆决策点。
