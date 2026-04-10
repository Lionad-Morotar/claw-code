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

## claw-code 中的能力管控映射

`claw-code` 作为 Rust 重写版本，没有直接复制 TypeScript 代码库的三层门禁系统。它的设计哲学是**"单一构建产物，通过运行时权限策略和局部能力白名单实现精细化控制"**，而不是通过构建时宏或身份常量拆分版本。

### 1. 权限策略 = 运行时的功能边界

Rust 实现中不存在 `feature()` 宏或 `USER_TYPE` 门控。功能开放与否由 `PermissionMode` + `PermissionPolicy` + `PermissionEnforcer` 三层运行时机制决定：

- **ReadOnly**：仅允许 `isReadOnly()` 工具（`read_file`、`grep`、`glob` 等）。
- **WorkspaceWrite**：允许工作区内的文件写入和一般 bash。
- **DangerFullAccess**：允许跨工作区写入和无限制 bash。
- **Prompt**：每次权限升级都需要用户确认。
- **Allow**：默认放行所有已知工具（但仍受 deny/ask 规则约束）。

这套五级模型位于 [`runtime/src/permissions.rs#L9-L15`](/rust/crates/runtime/src/permissions.rs#L9-L15)。它替代了原始代码库中"构建时删代码"的思路，改为**运行时拒绝**，代码始终存在，但权限不足时会被拦截。

### 2. 子代理能力白名单 = 细粒度工具门控

虽然 Rust 代码没有 `feature('KAIROS')` 或 `feature('BUDDY')` 这样的编译时开关，但它通过 `allowed_tools_for_subagent` 实现了**按代理类型裁剪工具集**的动态门控。参见 [`tools/src/lib.rs#L3451-L3520`](/rust/crates/tools/src/lib.rs#L3451-L3520)：

| subagent_type | 允许的工具 |
|---|---|
| `Explore` | `read_file`、`grep_search`、`WebFetch`、`WebSearch` 等（只读型） |
| `Plan` | 在 Explore 基础上增加 `TodoWrite`、`SendUserMessage`（规划型） |
| `Verification` | 在 Plan 基础上增加 `bash`、`PowerShell`（可执行验证） |
| `general-purpose` | 全部标准工具（默认 worker） |

`SubagentToolExecutor` 在运行时严格校验白名单，若工具不在列表中则直接拒绝。这实际上是原始代码库第二层（GrowthBook 运行时门控）和第三层（身份门控）的**功能等价物**——不是按用户身份删代码，而是按会话角色限制可用工具。

### 3. 参考数据中的原始系统痕迹

`claw-code` 在 `src/reference_data/` 目录中保留了原始 TypeScript 代码库的结构快照，包括：

- **Buddy 系统**：[`src/reference_data/subsystems/buddy.json`](/src/reference_data/subsystems/buddy.json)
- **GrowthBook 服务**：[`src/reference_data/subsystems/services.json#L18`](/src/reference_data/subsystems/services.json#L18)
- **GrowthBook 实验事件类型**：[`src/reference_data/subsystems/types.json#L9`](/src/reference_data/subsystems/types.json#L9)
- **Debug 模式命令**：[`src/reference_data/commands_snapshot.json`](/src/reference_data/commands_snapshot.json)

这些 JSON 文件说明原始代码库中存在完整的三层门禁生态，但 `claw-code` 目前仅将其作为历史存档，并未在 Rust 运行时中复现对应的门控逻辑。

### 为什么 Rust 实现选择了不同的路径？

| 设计目标 | 上游 TypeScript | claw-code Rust |
|---|---|---|
| 构建产物 | `ant` / `external` 双轨 | 单一通用二进制 |
| 功能开关 | 编译时 DCE | 运行时权限策略 |
| 实验发布 | GrowthBook 远程求值 | 尚不存在等价机制 |
| 内部功能 | `USER_TYPE` 常量折叠 | 未引入身份分层 |

`claw-code` 目前的定位是**技术探索与学习实现**。它用运行时权限系统替代了编译时门禁，用子代理白名单替代了按身份删代码。这种差异不是遗漏，而是**设计价值观的不同**——倾向于代码透明和统一构建，而非严格的版本分层。

---

## 与 claw-code 的对比分析

| 原始代码库门禁 | claw-code 映射 |
| --- | --- |
| **88+ 构建时 feature flags** | 未直接实现，Rust 使用条件编译 `#[cfg(feature = "...")]` |
| **500+ GrowthBook tengu_* 标记** | 参考数据中保留 `growthbook_experiment_event.ts` 类型定义 |
| **USER_TYPE (ant vs external)** | 无直接等价映射；`proactive` / `normal` 仅为 `SendUserMessage` 工具参数，控制消息发送风格 |
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
