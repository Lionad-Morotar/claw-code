# 三层门禁系统：功能可见性控制架构

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/internals/three-tier-gating) 的原始内容，结合 `claw-code` 源码参考数据进行技术映射。文末附 [源码索引](#源码索引)。

---

## 冰山一角

你日常使用的 Claude Code，只是完整代码库的冰山一角。

逆向工程揭示了一个事实：大量功能被精心"藏"在三层独立的门禁系统之后。有些是正在 A/B 测试的实验性功能，有些是仅限 Anthropic 员工使用的内部工具，还有些是尚未对外发布的下一代能力。

在原始 TypeScript 代码库的 `src/` 目录中发现了：
- **88+** 个构建时 feature flags
- **500+** 个运行时 A/B 测试标记（`tengu_*` 前缀）
- 一整套身份门控机制（`USER_TYPE`）

---

## 三层门禁全景

| 维度 | 第一层：构建时 `feature()` | 第二层：运行时 GrowthBook | 第三层：身份 `USER_TYPE` |
| --- | --- | --- | --- |
| **控制方式** | `bun:bundle` 编译时宏 | GrowthBook SDK 远程求值 | 构建时 `--define` 常量 |
| **决策时机** | 打包时（代码直接被删除） | 启动时 + 定期刷新 | 打包时（常量折叠） |
| **粒度** | 全有或全无 | 按用户/设备/组织定向 | 按构建版本（`ant` / `external`） |
| **标记数量** | 88+ | 500+ (`tengu_*` 前缀) | 1（`ant` vs `external`） |
| **逆向可见性** | 代码残留，但永远走 `false` 分支 | 完整 SDK 代码可读 | 条件分支清晰可见 |

---

## 决策流程

当一个功能请求进入 Claude Code，它会依次经过三层门禁的检查：

```
功能请求
│
▼
┌─────────────────────────┐
│ 第一层：feature('X')    │
│ ──── 编译时已决定 ───→ false → 代码被 DCE 移除
│ (构建时 Feature Flag)   │
└─────────┬───────────────┘
          │ true (仅内部构建)
          ▼
┌─────────────────────────┐
│ 第二层：tengu_xxx       │
│ ──── 运行时按用户属性 ──→ 不在实验组 → 功能关闭
│ (GrowthBook A/B 测试)   │
└─────────┬───────────────┘
          │ 在实验组
          ▼
┌─────────────────────────┐
│ 第三层：USER_TYPE       │
│ ──── ant? external? ───→ external → 功能不可用
│ (身份门控)              │
└─────────┬───────────────┘
          │ ant
          ▼
功能可用 ✓
```

三层门禁相互独立，一个功能可能同时受多层控制。例如，**KAIROS 助手模式**同时需要 `feature('KAIROS')` 构建时开启和 `tengu_kairos` 运行时实验命中。

---

## 逆向工程揭示了什么

在反编译版本中：

| 门禁层 | 逆向可见状态 |
| --- | --- |
| **第一层** | 完全透明——`feature()` 被兜底为 `() => false`，所有 88+ 个 flag 的代码路径都可以阅读，只是永远不会执行 |
| **第二层** | 完整保留——GrowthBook SDK 的 1156 行代码完整可读，包括用户定向属性、缓存策略、覆盖机制 |
| **第三层** | 清晰可见——`process.env.USER_TYPE === 'ant'` 出现在 60+ 个位置，每一处都标记着"仅限内部"的功能边界 |

**这三层门禁不是安全机制——它们是产品发布策略**。目的是让 Anthropic 能够在不同用户群体中渐进式地测试和发布功能，而不是阻止逆向工程。

---

## claw-code 中的门禁实现映射

`claw-code` 作为 Rust 重写版本，虽然没有直接复制原始 TypeScript 代码库的三层门禁系统，但在工具系统和服务架构中保留了类似的影子。

### TodoWrite 工具的门禁设计

在 `claw-code` 的工具定义中，`TodoWrite` 工具的状态枚举体现了门禁控制的思想：

```rust
#[derive(Debug, Deserialize, Serialize, Clone, PartialEq, Eq)]
#[serde(rename_all = "snake_case")]
enum TodoStatus {
    Pending,      // pending: 任务未开始
    InProgress,   // in_progress: 正在进行
    Completed,    // completed: 任务已完成
}
```

这一定义位于 [`rust/crates/tools/src/lib.rs#L2094-L2107`](/rust/crates/tools/src/lib.rs#L2094-L2107)：

```rust
#[derive(Debug, Deserialize, Serialize, Clone, PartialEq, Eq)]
struct TodoItem {
    content: String,
    #[serde(rename = "activeForm")]
    active_form: String,
    status: TodoStatus,
}
```

工具规格定义在 [`rust/crates/tools/src/lib.rs#L530-L555`](/rust/crates/tools/src/lib.rs#L530-L555) 附近，输入 schema 明确定义了状态枚举：

```rust
ToolSpec {
    name: "TodoWrite",
    description: "Update the structured task list for the current session.",
    input_schema: json!({
        "type": "object",
        "properties": {
            "todos": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "content": { "type": "string" },
                        "activeForm": { "type": "string" },
                        "status": {
                            "type": "string",
                            "enum": ["pending", "in_progress", "completed"]
                        }
                    },
                    "required": ["content", "activeForm", "status"],
                }
            }
        },
        "required": ["todos"],
        "additionalProperties": false
    }),
    required_permission: PermissionMode::WorkspaceWrite,
}
```

执行实现位于 [`rust/crates/tools/src/lib.rs#L1987-L1989`](/rust/crates/tools/src/lib.rs#L1987-L1989)：

```rust
fn run_todo_write(input: TodoWriteInput) -> Result<String, String> {
    to_pretty_json(execute_todo_write(input)?)
}
```

### Proactive 模式标记

在工具的 schema 定义中，发现了与原始文档提到的 `PROACTIVE` 功能相关的标记。位于 [`rust/crates/tools/src/lib.rs#L645`](/rust/crates/tools/src/lib.rs#L645)：

```rust
"status": {
    "type": "string",
    "enum": ["normal", "proactive"]
}
```

这是一个 `normal` vs `proactive` 的模式切换，类似于原始代码库中的 `USER_TYPE` 门控——区分标准用户行为与主动/预测性模式。

### Buddy 系统

原始文档提到的 Buddy 宠物系统在 `claw-code` 中以参考数据的形式存档。位于 [`src/reference_data/subsystems/buddy.json`](/src/reference_data/subsystems/buddy.json)：

```json
{
  "archive_name": "buddy",
  "package_name": "buddy",
  "module_count": 6,
  "sample_files": [
    "buddy/CompanionSprite.tsx",
    "buddy/companion.ts",
    "buddy/prompt.ts",
    "buddy/sprites.ts",
    "buddy/types.ts",
    "buddy/useBuddyNotification.tsx"
  ]
}
```

这表明 Buddy 是原始 TypeScript 代码库中的功能模块，`claw-code` 将其作为参考数据存档，但未在当前 Rust 实现中落地。

### GrowthBook 集成点

在参考数据的服务列表中，明确提到了 GrowthBook 分析服务。位于 [`src/reference_data/subsystems/services.json#L18`](/src/reference_data/subsystems/services.json#L18)：

```json
"services/analytics/growthbook.ts"
```

同时在事件类型定义中发现了 GrowthBook 实验事件。位于 [`src/reference_data/subsystems/types.json#L9`](/src/reference_data/subsystems/types.json#L9)：

```json
"types/generated/events_mono/growthbook/v1/growthbook_experiment_event.ts"
```

这证明原始代码库中存在完整的事件埋点系统，用于追踪 A/B 测试实验的效果。

### Debug 模式命令

在命令快照中发现了 `debug-tool-call` 命令。位于 [`src/reference_data/commands_snapshot.json`](/src/reference_data/commands_snapshot.json)：

```json
{
  "name": "debug-tool-call",
  "source_hint": "commands/debug-tool-call/index.js",
  "responsibility": "Command module mirrored from archived TypeScript path commands/debug-tool-call/index.js"
}
```

这与原始文档提到的"Debug 模式"相呼应——这类诊断工具通常只在 `ant` (内部) 构建中可用。

---

## 与 claw-code 的对比分析

| 原始代码库门禁 | claw-code 映射 |
| --- | --- |
| **88+ 构建时 feature flags** | 未直接实现，Rust 使用条件编译 `#[cfg(feature = "...")]` |
| **500+ GrowthBook tengu_* 标记** | 参考数据中保留 `growthbook_experiment_event.ts` 类型定义 |
| **USER_TYPE (ant vs external)** | `proactive` vs `normal` 模式标记 |
| **Buddy 宠物系统** | 存档于 `src/reference_data/subsystems/buddy.json` |
| **Debug 模式命令** | 存档于 `src/reference_data/commands_snapshot.json` |

---

## 总结

三层门禁系统是 Claude Code 产品发布策略的核心架构：

1. **构建时门禁** (`feature()`)：通过编译时宏实现代码级开关，未启用的功能在打包阶段即被删除
2. **运行时门禁** (GrowthBook)：基于用户属性的远程配置系统，支持 A/B 测试和灰度发布
3. **身份门禁** (`USER_TYPE`)：区分内部员工与外部用户，控制敏感功能的可见性

在 `claw-code` Rust 重写版本中，这一架构并未被完全复制，而是以参考数据的形式保留了原始设计的痕迹。这种差异反映了两个项目的不同定位：
- 原始 Claude Code：面向 Anthropic 内部迭代与外部发布的双轨产品
- `claw-code`：面向社区的技术探索与学习实现

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`/rust/crates/tools/src/lib.rs#L530-L555`](/rust/crates/tools/src/lib.rs#L530-L555) | TodoWrite 工具规格与输入 schema |
| [`/rust/crates/tools/src/lib.rs#L2094-L2107`](/rust/crates/tools/src/lib.rs#L2094-L2107) | TodoItem 与 TodoStatus 枚举定义 |
| [`/rust/crates/tools/src/lib.rs#L645`](/rust/crates/tools/src/lib.rs#L645) | Proactive 模式标记 |
| [`/rust/crates/tools/src/lib.rs#L1987-L1989`](/rust/crates/tools/src/lib.rs#L1987-L1989) | TodoWrite 执行入口 |
| [`/src/reference_data/subsystems/buddy.json`](/src/reference_data/subsystems/buddy.json) | Buddy 系统存档数据 |
| [`/src/reference_data/subsystems/services.json`](/src/reference_data/subsystems/services.json#L18) | GrowthBook 服务引用 |
| [`/src/reference_data/subsystems/types.json`](/src/reference_data/subsystems/types.json#L9) | GrowthBook 实验事件类型 |
| [`/src/reference_data/commands_snapshot.json`](/src/reference_data/commands_snapshot.json) | debug-tool-call 命令存档 |

---

## 后续文档

本单元是"揭秘：隐藏功能与内部机制"系列的第 28 篇。后续文档将深入：
- `88 面旗帜` — 构建时 Feature Flags 的完整分类与解读
- `千面千人` — GrowthBook A/B 测试体系的运作机制
- `未公开功能巡礼` — KAIROS、PROACTIVE 等 8 大隐藏功能深度解析
- `Ant 的特权世界` — Anthropic 员工专属的工具、命令与 API
