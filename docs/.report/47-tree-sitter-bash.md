# Unit 47: TREE_SITTER_BASH — Bash AST 解析技术报告

**原始页面**: https://ccb.agent-aura.top/docs/features/tree-sitter-bash  
**输出文件**: `docs/.report/47-tree-sitter-bash.md`  
**生成日期**: 2026-04-09


> **源码映射说明**：Tree-Sitter Bash 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。---

## 一、功能概述

`TREE_SITTER_BASH` 启用一个完整的 Bash AST 解析器，用于安全验证 Bash 命令。它用完整的树遍历安全分析器取代了旧的基于正则表达式的 shell-quote 解析器。关键属性是 **fail-closed**：任何无法识别的内容都被归类为 `too-complex` 并需要用户批准。

### 关联 Feature

| Feature | 说明 |
|---------|------|
| `TREE_SITTER_BASH` | 激活用于权限检查的 AST 解析器 |
| `TREE_SITTER_BASH_SHADOW` | Shadow/观测模式：运行解析器但丢弃结果，仅记录遥测 |

---

## 二、安全架构

### 2.1 Fail-Closed 设计

核心设计使用 allowlist 遍历模式：
- `walkArgument()` 只处理已知安全的节点类型（`word`、`number`、`raw_string`、`string`、`concatenation`、`arithmetic_expansion`、`simple_expansion`）
- 任何未知节点类型 → `tooComplex()` → 需要用户批准
- 解析器加载但失败（超时/节点预算/panic）→ 返回 `PARSE_ABORTED` 符号（区别于"模块未加载"）

### 2.2 解析结果

`parseForSecurity(cmd)` 返回：
```typescript
{ kind: 'simple', commands: SimpleCommand[] }     // 可静态分析
{ kind: 'too-complex', reason, nodeType }         // 需要用户批准
{ kind: 'parse-unavailable' }                      // 解析器未加载
```

### 2.3 安全检查层次

```
parseForSecurity(cmd)
  │
  ▼
parseCommandRaw(cmd) → AST root node
  │
  ▼
预检查：控制字符、Unicode 空白、反斜杠 + 空白、zsh ~[ ] 语法、zsh =cmd 展开、大括号 + 引号混淆
  │
  ▼
walkProgram(root) → collectCommands(root, commands, varScope)
  ├── 'command' → walkCommand()
  ├── 'pipeline'/'list' → 结构性，递归子节点
  ├── 'for_statement' → 跟踪循环变量为 VAR_PLACEHOLDER
  ├── 'if/while' → 作用域隔离的分支
  ├── 'subshell' → 作用域复制
  ├── 'variable_assignment' → walkVariableAssignment()
  ├── 'declaration_command' → 验证 declare/export flags
  ├── 'test_command' → walk test expressions
  └── 其他 → tooComplex()
  │
  ▼
checkSemantics(commands)
  ├── EVAL_LIKE_BUILTINS（eval, source, exec, trap...）
  ├── ZSH_DANGEROUS_BUILTINS（zmodload, emulate...）
  ├── SUBSCRIPT_EVAL_FLAGS（test -v, printf -v, read -a）
  ├── Shell keywords as argv[0]（误解析检测）
  ├── /proc/*/environ 访问
  ├── jq system() 和危险 flags
  └── 包装器剥离（time, nohup, timeout, nice, env, stdbuf）
```

---

## 三、实现架构

### 3.1 核心模块

| 模块 | 文件 | 行数 | 职责 |
|------|------|------|------|
| 门控入口 | `packages/ccb/src/utils/bash/parser.ts` | ~110 | `parseCommand()`、`parseCommandRaw()`、`ensureInitialized()` |
| Bash 解析器 | `packages/ccb/src/utils/bash/bashParser.ts` | 4437 | 纯 TS 词法分析 + 递归下降解析器 |
| 安全分析器 | `packages/ccb/src/utils/bash/ast.ts` | 2680 | 树遍历安全分析 + `parseForSecurity()` |
| AST 分析辅助 | `packages/ccb/src/utils/bash/treeSitterAnalysis.ts` | 507 | 引号上下文、复合结构、危险模式提取 |
| 权限检查入口 | `packages/ccb/src/tools/BashTool/bashPermissions.ts` | ~140 | 集成 AST 结果到权限决策 |

### claw-code Rust 实现

在 `rust/crates/runtime/src/` 中，Bash 验证由 `bash_validation.rs`（1004 行）和 `permissions.rs`（683 行）实现。

### 3.2 Bash 解析器

**文件**: `packages/ccb/src/utils/bash/bashParser.ts`（4436 行）

> **实现风险**：该解析器为纯 TypeScript 手写实现，与上游 `tree-sitter-bash` 语法不同步。Bash 语法持续演进（如 `assoc+=()`、`coproc` 变体），任何新增语法都需要手动更新 parser 与 `walkArgument` allowlist，否则这些构造会被 fail-closed 归类为 `too-complex`，导致用户体验频繁中断。Parser 维护成本应被纳入长期技术债务评估。

- 纯 TypeScript 实现（无原生依赖）
- 生成与 tree-sitter-bash 兼容的 AST
- 关键类型：`TsNode`（type、text、startIndex、endIndex、children）
- 安全限制：`PARSE_TIMEOUT_MS = 50`、`MAX_NODES = 50_000` — 防止对抗性输入导致 OOM

### 3.3 安全分析器

**文件**: `packages/ccb/src/utils/bash/ast.ts`（2679 行）

核心函数：

| 函数 | 职责 |
|------|------|
| `parseForSecurity(cmd)` | 顶层入口，返回 `simple`/`too-complex`/`parse-unavailable` |
| `parseForSecurityFromAst(cmd, root)` | 接受预解析 AST |
| `checkSemantics(commands)` | 后解析语义检查 |
| `walkCommand()` | 提取 argv、envVars、redirects |
| `walkArgument()` | Allowlist 参数遍历 |
| `collectCommands()` | 递归收集所有命令 |

### 3.4 AST 分析辅助

**文件**: `packages/ccb/src/utils/bash/treeSitterAnalysis.ts`（507 行）

| 函数 | 职责 |
|------|------|
| `extractQuoteContext()` | 识别单引号、双引号、ANSI-C 字符串、heredoc |
| `extractCompoundStructure()` | 检测管道、子 shell、命令组 |
| `hasActualOperatorNodes()` | 区分真实 `;`/`&&`/`` ` `` 与转义形式 |
| `extractDangerousPatterns()` | 检测命令替换、参数展开、heredocs |
| `analyzeCommand()` | 单次遍历提取 |

### 3.5 Shadow 模式

`TREE_SITTER_BASH_SHADOW` 运行解析器但 **从不影响权限决策**：

```typescript
// Shadow 模式：记录遥测，然后强制使用旧版路径
astResult = { kind: 'parse-unavailable' }
astRoot = null
// 记录：available, astTooComplex, astSemanticFail, subsDiffer, ...
```

记录 `tengu_tree_sitter_shadow` 事件，包含与旧版 `splitCommand()` 的对比数据。用于在不影响行为的情况下收集遥测。

---

## 四、关键设计决策

1. **Allowlist 遍历**: 只处理已知安全的节点类型，未知类型直接 `tooComplex()`
2. **PARSE_ABORTED 符号**: 区分"解析器未加载"和"解析器加载但失败"。后者阻止回退旧版（旧版缺少 `EVAL_LIKE_BUILTINS` 检查）
3. **变量作用域跟踪**: `VAR=value && cmd $VAR` 模式。静态值解析为真实字符串，`$()` 输出使用 `VAR_PLACEHOLDER`
4. **PS4/IFS Allowlist**: PS4 赋值使用严格白名单验证

---

## 五、使用方式

```bash
# 激活 AST 解析用于权限检查
FEATURE_TREE_SITTER_BASH=1 bun run dev

# Shadow 模式（仅遥测）
FEATURE_TREE_SITTER_BASH_SHADOW=1 bun run dev
```

---

## 六、文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/ccb/src/utils/bash/parser.ts` | ~110 | 门控入口点 |
| `packages/ccb/src/utils/bash/bashParser.ts` | 4437 | 纯 TS bash 解析器 |
| `packages/ccb/src/utils/bash/ast.ts` | 2680 | 安全分析器（核心） |
| `packages/ccb/src/utils/bash/treeSitterAnalysis.ts` | 507 | AST 分析辅助 |
| `packages/ccb/src/tools/BashTool/bashPermissions.ts:1670-1810` | ~140 | 权限集成 + Shadow 遥测 |

---

## 七、claw-code Rust 实现对照

在 claw-code 项目中，Bash 命令验证采用 Rust 实现，位于 `rust/crates/runtime/src/bash_validation.rs`（1004 行）。

### 7.1 核心模块结构

```
rust/crates/runtime/src/
├── bash.rs              # Bash 命令执行入口 (336 行)
├── bash_validation.rs   # 命令验证核心逻辑 (1004 行)
├── permissions.rs       # 权限策略引擎 (683 行)
└── lib.rs               # 模块导出 (#L8 pub mod bash_validation)
```

### 7.2 验证函数映射

| 原文 TypeScript | claw-code Rust | 行号 | 职责 |
|----------------|----------------|------|------|
| `validate_read_only` | `validate_read_only()` | #L102-L160 | 只读模式验证 |
| `check_destructive` | `check_destructive()` | #L241-L274 | 危险命令检测 |
| `validate_mode` | `validate_mode()` | #L284-L303 | 权限模式验证 |
| `validate_sed` | `validate_sed()` | #L336-L350 | sed 表达式验证 |
| `validate_paths` | `validate_paths()` | #L360-L382 | 路径遍历检测 |
| `classify_command` | `classify_command()` | #L533-L584 | 命令语义分类 |
| `validate_command` | `validate_command()` | #L594-L615 | 完整验证管道 |

### 7.3 命令意图分类

Rust 实现定义了 8 种命令意图类型（#L28-L45）：

```rust
pub enum CommandIntent {
    ReadOnly,         // ls, cat, grep, find 等
    Write,            // cp, mv, mkdir, touch 等
    Destructive,      // rm, shred, truncate 等
    Network,          // curl, wget, ssh 等
    ProcessManagement,// kill, pkill 等
    PackageManagement,// apt, brew, pip, npm 等
    SystemAdmin,      // sudo, chmod, chown, mount 等
    Unknown,          // 未知命令
}
```

### 7.4 验证管道

完整验证管道 `validate_command()`（#L594-L615）执行 4 层检查：

```rust
pub fn validate_command(command: &str, mode: PermissionMode, workspace: &Path) -> ValidationResult {
    // 1. 模式级验证（包含只读检查）
    let result = validate_mode(command, mode);
    if result != ValidationResult::Allow { return result; }

    // 2. sed 专用验证
    let result = validate_sed(command, mode);
    if result != ValidationResult::Allow { return result; }

    // 3. 危险命令警告
    let result = check_destructive(command);
    if result != ValidationResult::Allow { return result; }

    // 4. 路径验证
    validate_paths(command, workspace)
}
```

### 7.5 权限集成

权限策略引擎 `permissions.rs` 集成 Bash 验证：

- **PermissionMode** 枚举（#L8-L14）：`ReadOnly`、`WorkspaceWrite`、`DangerFullAccess`、`Prompt`、`Allow`
- **PermissionPolicy::authorize_with_context()**（#L173-L292）：执行规则匹配与权限升级判定
- 工具需求映射（#L119-L162）：`with_tool_requirement` 注册各工具所需权限级别

示例（#L576-L584）：
```rust
let policy = PermissionPolicy::new(PermissionMode::DangerFullAccess)
    .with_tool_requirement("bash", PermissionMode::DangerFullAccess)
    .with_permission_rules(&rules);

assert_eq!(
    policy.authorize("bash", r#"{"command":"git status"}"#, None),
    PermissionOutcome::Allow
);
```

### 7.6 关键设计对照

| 设计决策 | TypeScript 原文 | claw-code Rust |
|---------|----------------|----------------|
| Fail-Closed | 未知节点 → tooComplex | 未知命令 → Unknown (#L44) |
| Allowlist | `SEMANTIC_READ_ONLY_COMMANDS` | `SEMANTIC_READ_ONLY_COMMANDS` (#L389-L457) |
| 危险模式 | `DESTRUCTIVE_PATTERNS` | `DESTRUCTIVE_PATTERNS` (#L206-L235) |
| 写操作检测 | 写重定向 `>`, `>>` | `WRITE_REDIRECTIONS` (#L96) |
| Git 子命令 | 白名单验证 | `GIT_READ_ONLY_SUBCOMMANDS` (#L163-L183) |

---

## 八、总结

---

**源码锚点汇总**：
- `rust/crates/runtime/src/lib.rs:#L8` — 模块声明
- `rust/crates/runtime/src/bash.rs:#L70-L103` — Bash 执行入口
- `rust/crates/runtime/src/bash_validation.rs:#L103-L160` — 只读验证
- `rust/crates/runtime/src/bash_validation.rs:#L241-L274` — 危险命令检测
- `rust/crates/runtime/src/bash_validation.rs:#L533-L584` — 命令分类
- `rust/crates/runtime/src/bash_validation.rs:#L594-L615` — 完整验证管道
- `rust/crates/runtime/src/permissions.rs:#L9-L14` — 权限模式定义
- `rust/crates/runtime/src/permissions.rs:#L175-L292` — 权限授权逻辑
