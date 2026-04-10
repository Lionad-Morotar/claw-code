# BASH_CLASSIFIER — Bash 命令分类器

**Feature Flag**: `FEATURE_BASH_CLASSIFIER=1`
**实现状态**: `bashClassifier.ts` 全部 Stub，`yoloClassifier.ts` 完整实现可参考
**引用数**: 45

> **源码映射说明**：本报告同时覆盖上游 TypeScript 实现（`packages/ccb/src/utils/permissions/bashClassifier.ts` 为 stub，`yoloClassifier.ts` 为完整参考实现）和 `claw-code` Rust 实现（[`bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs) 规则驱动分类器）。报告第七节专门描述 Rust 端实现。上游 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。

---

## 一、功能概述

BASH_CLASSIFIER 使用 LLM 对 bash 命令进行意图分类（允许/拒绝/询问），实现自动权限决策。用户不需要逐个审批 bash 命令，分类器根据命令内容和上下文自动判断安全性。

### 核心特性

- **LLM 驱动分类**: 使用 Opus 模型评估命令安全性
- **两阶段分类**: 快速阻止/允许 → 深度思考链
- **自动审批**: 分类器判定安全的命令自动通过
- **UI 集成**: 权限对话框显示分类器状态和审核选项

---

## 二、实现架构

### 2.1 模块状态

| 模块 | 文件 | 状态 | 说明 |
|------|------|------|------|
| Bash 分类器 | `packages/ccb/src/utils/permissions/bashClassifier.ts` | Stub | 所有函数返回空操作/默认值。注释："ANT-ONLY" |
| YOLO 分类器 | `packages/ccb/src/utils/permissions/yoloClassifier.ts` | 完整 | 1495 行，两阶段 XML 分类器 |
| 审批信号 | `packages/ccb/src/utils/classifierApprovals.ts` | 完整 | Map + 信号管理分类器决策 |
| 权限 UI | `packages/ccb/src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx` | 布线 | 分类器状态显示、审核选项 |
| 权限管道 | `src/hooks/toolPermission/handlers/*.ts` | 布线 | 分类器结果路由到决策 |
| API beta 标头 | `src/services/api/withRetry.ts` | 布线 | 启用时发送 bash_classifier beta |
| 规则驱动分类器（Rust） | [`bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs) | 完整 | `CommandIntent` 语义分类 + 四阶段验证管道 |

### 2.2 参考实现：yoloClassifier.ts

**文件**: `packages/ccb/src/utils/permissions/yoloClassifier.ts`（1495 行）  
这是已实现的完整分类器，可作为 `bashClassifier.ts` 的参考：

**两阶段分类**:
1. **快速阶段**: 构建对话记录 → 调用 sideQuery（Opus）→ 快速阻止/允许
2. **深度阶段**: 思考链分析 → 最终决策

**特性**:
- 构建完整对话记录上下文
- 调用安全系统提示的 sideQuery
- GrowthBook 配置和指标
- 错误处理和降级

### 2.3 分类器在权限管道中的位置与实现风险

> **风险提醒**：`bashClassifier.ts` 当前为全 stub（ANT-ONLY 标记）。这意味着在当前 TypeScript 构建中开启 `FEATURE_BASH_CLASSIFIER=1` 后，分类器不会调用任何 LLM，而是直接返回 `matches: false` 的默认结果；权限决策将回退到传统规则匹配或完全缺失，可能产生安全真空。实现一个生产级 LLM 分类器需要解决延迟（每次 Bash 调用增加一次 sideQuery）、错误降级（LLM 幻觉导致的自动 allow）、上下文窗口成本（注入完整对话记录）以及可解释性（用户可审计分类理由）等问题，其工作量应评估为「大」而非「中」。claw-code 的 Rust 端仅提供规则驱动分类（`bash_validation.rs`），不含 LLM 分类器。


```
bash 命令到达
      │
      ▼
bashPermissions.ts 权限检查
      │
      ├── 传统规则匹配（字符串级别）
      │
      └── [BASH_CLASSIFIER] LLM 分类
            │
            ├── allow → 自动通过
            ├── deny → 自动拒绝
            └── ask → 显示权限对话框
                  │
                  ├── 分类器自动审批标记
                  └── 审核选项（用户可覆盖）
```

---

## 三、需要补全的内容

| 函数 | 需要实现 | 说明 |
|------|----------|------|
| `classifyBashCommand()` | LLM 调用评估安全性 | 参考 `yoloClassifier.ts` 的两阶段模式 |
| `isClassifierPermissionsEnabled()` | GrowthBook/配置检查 | 控制分类器是否激活 |
| `getBashPromptDenyDescriptions()` | 返回基于提示的拒绝规则 | 权限设置描述 |
| `getBashPromptAskDescriptions()` | 返回询问规则 | 需要用户确认的命令 |
| `getBashPromptAllowDescriptions()` | 返回允许规则 | 自动通过的命令 |
| `generateGenericDescription()` | LLM 生成命令描述 | 为权限对话框提供说明 |
| `extractPromptDescription()` | 解析规则内容 | 从规则中提取描述 |

---

## 四、关键设计决策

- **ANT-ONLY 标记**: `bashClassifier.ts` 标注为 "ANT-ONLY"，可能是 Anthropic 内部服务端分类器的客户端适配
- **两阶段分类**: 快速阶段处理明确情况（减少延迟），深度阶段处理模糊情况
- **分类器结果可审核**: 权限 UI 显示分类器决策，用户可覆盖
- **YOLO 分类器参考**: `yoloClassifier.ts` 提供完整的分类器实现模式，可直接参考

---

## 五、使用方式

```bash
# 启用 feature
FEATURE_BASH_CLASSIFIER=1 bun run dev

# 配合 TREE_SITTER_BASH 使用（AST + LLM 双重安全）
FEATURE_BASH_CLASSIFIER=1 FEATURE_TREE_SITTER_BASH=1 bun run dev
```

---

## 六、文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/ccb/src/utils/permissions/bashClassifier.ts` | — | Bash 分类器（stub，ANT-ONLY） |
| `packages/ccb/src/utils/permissions/yoloClassifier.ts` | 1495 | YOLO 分类器（完整参考实现） |
| `packages/ccb/src/utils/classifierApprovals.ts` | — | 分类器审批信号管理 |
| `packages/ccb/src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx` | — | 分类器 UI |
| `packages/ccb/src/hooks/toolPermission/handlers/interactiveHandler.ts` | — | 交互式权限处理 |
| `packages/ccb/src/services/api/withRetry.ts:81` | — | API beta 标头 |

---

## 七、源码映射（claw-code 实现）

在 claw-code 项目中，bash 命令分类器的核心实现在 Rust 端完成。

### 7.1 核心模块

#### [`bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs)

**位置**: [`rust/crates/runtime/src/bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs)

这是 bash 命令验证和分类的核心模块，包含以下功能：

| 函数 | 行号 | 功能 |
|------|------|------|
| `validate_read_only()` | [`#L103`](rust/crates/runtime/src/bash_validation.rs#L103) | 只读模式下的命令验证 |
| `check_destructive()` | [`#L241`](rust/crates/runtime/src/bash_validation.rs#L241) | 破坏性命令检查 |
| `validate_mode()` | [`#L284`](rust/crates/runtime/src/bash_validation.rs#L284) | 权限模式验证 |
| `validate_sed()` | [`#L336`](rust/crates/runtime/src/bash_validation.rs#L336) | sed 命令特殊验证 |
| `validate_paths()` | [`#L360`](rust/crates/runtime/src/bash_validation.rs#L360) | 路径安全检查 |
| `classify_command()` | [`#L533`](rust/crates/runtime/src/bash_validation.rs#L533) | 命令意图分类 |
| `validate_command()` | [`#L594`](rust/crates/runtime/src/bash_validation.rs#L594) | 完整验证流水线 |
| `extract_first_command()` | [`#L622`](rust/crates/runtime/src/bash_validation.rs#L622) | 提取命令首个词 |

#### [`permissions.rs`](/rust/crates/runtime/src/permissions.rs)

**位置**: [`rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs)

权限模式定义和策略评估：

| 类型/函数 | 行号 | 功能 |
|-----------|------|------|
| `PermissionMode` | L9 | 权限模式枚举（ReadOnly, WorkspaceWrite, DangerFullAccess, Prompt, Allow） |
| `PermissionPolicy` | L99 | 权限策略评估器 |
| `PermissionOutcome` | L92 | 授权结果（Allow/Deny） |
| `authorize_with_context()` | [`#L175`](rust/crates/runtime/src/bash_validation.rs#L175) | 带上下文的授权决策 |

#### [`bash.rs`](/rust/crates/runtime/src/bash.rs)

**位置**: [`rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs)

Bash 命令执行和沙箱配置：

| 类型/函数 | 行号 | 功能 |
|-----------|------|------|
| `BashCommandInput` | L19 | 命令输入结构 |
| `BashCommandOutput` | L39 | 命令输出结构 |
| `execute_bash()` | L70 | 命令执行入口 |

### 7.2 命令意图分类器

[`classify_command()`](/rust/crates/runtime/src/bash_validation.rs#L533-L584) 函数将 bash 命令分类为以下意图：

```rust
pub enum CommandIntent {
    ReadOnly,        // 只读操作：ls, cat, grep, find 等
    Write,           // 文件系统写入：cp, mv, mkdir, touch, tee 等
    Destructive,     // 破坏性操作：rm, shred, truncate 等
    Network,         // 网络操作：curl, wget, ssh 等
    ProcessManagement, // 进程管理：kill, pkill 等
    PackageManagement, // 包管理：apt, brew, pip, npm 等
    SystemAdmin,     // 系统管理：sudo, chmod, chown, mount 等
    Unknown,         // 未知或无法分类的命令
}
```

**实现位置**: [`bash_validation.rs:533-584`](/rust/crates/runtime/src/bash_validation.rs#L533-L584)

### 7.3 验证流水线

[`validate_command()`](/rust/crates/runtime/src/bash_validation.rs#L594-L615) 执行四层验证：

```
bash 命令
    │
    ▼
1. validate_mode() — 模式级验证（含只读检查）
    │
    ▼
2. validate_sed() — sed 特殊验证
    │
    ▼
3. check_destructive() — 破坏性命令警告
    │
    ▼
4. validate_paths() — 路径验证
```

**实现位置**: [`bash_validation.rs:594-615`](/rust/crates/runtime/src/bash_validation.rs#L594-L615)

### 7.4 命令语义分类表

| 分类 | 命令示例 | 实现位置 |
|------|----------|----------|
| **只读命令** | `ls`, `cat`, `head`, `tail`, `grep`, `find`, `jq` | [`bash_validation.rs:389-457`](/rust/crates/runtime/src/bash_validation.rs#L389-L457) |
| **写入命令** | `cp`, `mv`, `rm`, `mkdir`, `touch`, `chmod`, `tee`, `dd` 等 17 个 | [`bash_validation.rs:52-55`](/rust/crates/runtime/src/bash_validation.rs#L52-L55) |
| **状态修改命令** | `apt`, `brew`, `docker`, `systemctl`, `reboot`, `shutdown` 等 34 个 | [`bash_validation.rs:58-94`](/rust/crates/runtime/src/bash_validation.rs#L58-L94) |
| **破坏性模式** | `rm -rf /`, `rm -rf ~`, `mkfs`, `dd if=`, `fork bomb` 等 10 个模式 | [`bash_validation.rs:206-232`](/rust/crates/runtime/src/bash_validation.rs#L206-L232) |
| **始终破坏性** | `shred`, `wipefs` | [`bash_validation.rs:235`](/rust/crates/runtime/src/bash_validation.rs#L235) |
| **网络命令** | `curl`, `wget`, `ssh`, `scp`, `ping`, `nmap` 等 22 个 | [`bash_validation.rs:460-482`](/rust/crates/runtime/src/bash_validation.rs#L460-L482) |
| **进程命令** | `kill`, `pkill`, `ps`, `top`, `htop` 等 14 个 | [`bash_validation.rs:485-488`](/rust/crates/runtime/src/bash_validation.rs#L485-L488) |
| **包管理** | `apt`, `brew`, `pip`, `npm`, `cargo`, `yarn` 等 19 个 | [`bash_validation.rs:491-494`](/rust/crates/runtime/src/bash_validation.rs#L491-L494) |
| **系统管理** | `sudo`, `mount`, `systemctl`, `crontab` 等 30 个 | [`bash_validation.rs:497-527`](/rust/crates/runtime/src/bash_validation.rs#L497-L527) |

### 7.5 工具集成

在 [`tools/src/lib.rs`](/rust/crates/tools/src/lib.rs) 中，bash 工具被注册为需要 `DangerFullAccess` 权限：

```rust
ToolSpec {
    name: "bash",
    description: "Execute a shell command in the current workspace.",
    input_schema: json!({ ... }),
    required_permission: PermissionMode::DangerFullAccess,
}
```

**实现位置**: [`tools/src/lib.rs:387-407`](/rust/crates/tools/src/lib.rs#L387-L407)

### 7.6 权限执行流程

```
execute_tool("bash", input)
    │
    ▼
maybe_enforce_permission_check(enforcer, name, input)
    │
    ▼
PermissionPolicy::authorize(tool_name, input, prompter)
    │
    ├── Allow → run_bash() → execute_bash()
    ├── Deny → 返回错误
    └── Prompt → 等待用户决策
```

**实现位置**:
- [`tools/src/lib.rs:1183-1186`](/rust/crates/tools/src/lib.rs#L1183-L1186) — bash 工具调度
- [`tools/src/lib.rs:1791-1794`](/rust/crates/tools/src/lib.rs#L1791-L1794) — run_bash 执行

---

## 八、对比：Claude Code vs claw-code

| 特性 | Claude Code (上游) | claw-code (本项目) |
|------|-------------------|-------------------|
| **分类器类型** | LLM 驱动（Opus） | 规则驱动 + 语义分类 |
| **实现语言** | TypeScript | Rust |
| **分类决策** | allow/deny/ask | CommandIntent 枚举 |
| **权限模式** | 三级（Allow/Ask/Deny） | 五级（ReadOnly/WorkspaceWrite/DangerFullAccess/Prompt/Allow） |
| **验证层** | 两阶段（快速 + 深度） | 四层验证流水线 |
| **可审核性** | UI 显示分类器决策 | 权限对话框 + Hook 覆盖 |

---

## 参考资料

- 原始文档：<https://ccb.agent-aura.top/docs/features/bash-classifier>
- 上游实现参考：`yoloClassifier.ts`（1495 行完整分类器）
- claw-code 实现：
  - [`rust/crates/runtime/src/bash_validation.rs`](/rust/crates/runtime/src/bash_validation.rs)
  - [`rust/crates/runtime/src/permissions.rs`](/rust/crates/runtime/src/permissions.rs)
  - [`rust/crates/runtime/src/bash.rs`](/rust/crates/runtime/src/bash.rs)
  - [`rust/crates/tools/src/lib.rs`](/rust/crates/tools/src/lib.rs)
