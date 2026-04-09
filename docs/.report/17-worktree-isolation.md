# Worktree 隔离

> 本报告基于 [Claude Code 中文文档](https://ccb.agent-aura.top/docs/agent/worktree-isolation) 的章节，映射到 `claw-code`（Claude Code 的 Rust 重写版）及其上游 TypeScript 实现（`packages/ccb`）的源码。
> 文末附 [源码索引](#源码索引)。

---

## 为什么需要文件级隔离

多 Agent 并行工作时，共享同一工作目录会导致三类冲突：

1. **写入冲突**：两个 Agent 同时编辑 `config.ts`，后写的覆盖前写的
2. **状态干扰**：Agent A 的测试依赖某个环境状态，Agent B 的修改破坏了它
3. **不可区分**：半完成的修改混在一起，无法分辨哪些是哪个 Agent 的

Git worktree 是 git 原生的解决方案——在同一个仓库中创建多个独立工作目录，每个在自己的分支上。`claw-code` 的 Rust 实现目前尚未完整落地 worktree 隔离工具，但上游 `packages/ccb` 中已有完整的 `EnterWorktree` / `ExitWorktree` 实现。本报告以源码为主线，揭示其创建/销毁生命周期、路径命名规则、hook 机制和退出时的安全防护。

---

## 目录结构与命名规则

Worktree 文件统一存放在仓库根目录下的 `.claude/worktrees/`：

```
<repo-root>/
├── .claude/
│   └── worktrees/
│       ├── fix-auth-bug/          # worktree 工作目录
│       │   ├── .git               # 指向主仓库的链接文件
│       │   └── src/...            # 独立的文件系统视图
│       └── add-dark-mode/         # 另一个 worktree
│           └── ...
├── src/                           # 主工作目录（不受影响）
└── .git/                          # 主仓库
```

### Slug 校验与扁平化

slug 由 `validateWorktreeSlug()` 校验：每个 `/` 分隔的段只允许字母、数字、`.`、`_`、`-`，总长 ≤64 字符。为了规避 git refs 的 D/F 冲突以及子 worktree 被父级误删的风险，内部会把嵌套 slug 中的 `/` 替换为 `+`（`flattenSlug`），所以 `user/feature` 实际对应的目录是 `.claude/worktrees/user+feature/`，分支名为 `worktree-user+feature`。参见 [`packages/ccb/src/utils/worktree.ts#L66-L157`](/packages/ccb/src/utils/worktree.ts#L66-L157) 中的 `validateWorktreeSlug()` 以及 [`L217-L222`](/packages/ccb/src/utils/worktree.ts#L217-L222) 的 `flattenSlug()`。

---

## 创建流程：EnterWorktreeTool

`EnterWorktreeTool` 的核心实现位于 [`packages/ccb/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`](/packages/ccb/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts)。执行链路如下：

### 1. 使用门控

工具的 `prompt` 中明确限制了触发条件（[`prompt.ts#L3-L13`](/packages/ccb/src/tools/EnterWorktreeTool/prompt.ts#L3-L13)）：只有用户**显式提到 "worktree"** 时才能调用，避免在普通的分支切换或 bug 修复场景中误用。

### 2. 防嵌套与主仓库解析

```typescript
if (getCurrentWorktreeSession()) {
  throw new Error('Already in a worktree session')
}
const mainRepoRoot = findCanonicalGitRoot(getCwd())
if (mainRepoRoot && mainRepoRoot !== getCwd()) {
  process.chdir(mainRepoRoot)
  setCwd(mainRepoRoot)
}
```

参见 [`EnterWorktreeTool.ts#L77-L88`](/packages/ccb/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts#L77-L88)。如果当前已经在某个 worktree 内部，`findCanonicalGitRoot` 会穿透到真正的主仓库根目录，确保新 worktree 不嵌套。

### 3. Hook 优先的架构

`createWorktreeForSession()`（[`worktree.ts#L702-L779`](/packages/ccb/src/utils/worktree.ts#L702-L779)）首先检查 `hasWorktreeCreateHook()`——如果用户在 `settings.json` 中配置了 `WorktreeCreate` hook，系统**完全不调用 git**，而是执行 hook 命令并将返回的路径作为 worktree 路径。这允许非 git 版本控制系统（如 Pijul、Mercurial）通过 hook 接入。hook 路径与 git 路径互斥，保证行为一致。

### 4. 快速恢复路径

`getOrCreateWorktree()`（[`worktree.ts#L235-L503`](/packages/ccb/src/utils/worktree.ts#L235-L503)）有一个关键优化：如果目标路径已存在，直接读取 `.git` 指针文件获得 HEAD SHA（`readWorktreeHeadSha`，纯文件 I/O，无子进程），并立即返回 `{ existed: true, ... }`。这跳过了后续的 `git fetch` + `git worktree add` 流程。在 210K 文件的大仓库中，`fetch` 可能消耗 6-8 秒，而快速恢复路径将延迟降到接近 0。

### 5. Git 原生创建路径（未命中快速恢复时）

对于全新 worktree，流程为：

1. `mkdir .claude/worktrees/`（recursive）
2. 若本地已缓存 `origin/<default-branch>` 的 SHA，则跳过 `fetch`；否则执行 `git fetch origin <default-branch>`
3. `git worktree add -B worktree/<slug> <path> <base>`（使用 `-B` 重置孤儿分支，避免残留分支导致失败）
4. 若配置了 `sparsePaths`，应用 `--no-checkout` + `sparse-checkout set --cone -- <paths>` + `git checkout HEAD`

其中 sparse-checkout 的错误处理采用了"故障回退"（tearDown）：若 sparse-checkout 或 checkout 失败，立刻执行 `git worktree remove --force <path>`，防止下一次快速恢复误把一个空目录当作可用 worktree。参见 [`worktree.ts#L341-L368`](/packages/ccb/src/utils/worktree.ts#L341-L368)。

### 6. 创建后设置（performPostCreationSetup）

`performPostCreationSetup()`（[`worktree.ts#L510-L652`](/packages/ccb/src/utils/worktree.ts#L510-L652)）完成以下初始化：

- **传播本地配置**：将主仓库的 `settings.local.json` 复制到 worktree 中，保证 secrets 和本地覆盖生效
- **统一 git hooks**：检测 `.husky` 或 `.git/hooks`，通过 `git config core.hooksPath <absolute_path>` 让 worktree 复用主仓库的 hooks
- **目录符号链接**：根据 `settings.worktree.symlinkDirectories`，将 `node_modules` 等目录 symlink 到主仓库，避免磁盘膨胀
- **`.worktreeinclude` 文件复制**：将主仓库中被 gitignore 但列在 `.worktreeinclude` 中的文件复制到 worktree，支持单遍 `--directory` 扫描优化

### 7. 进程状态更新

创建成功后，[`EnterWorktreeTool.ts#L92-L103`](/packages/ccb/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts#L92-L103) 会：

- `process.chdir(worktreePath)` + `setCwd(worktreePath)` + `setOriginalCwd(worktreePath)`
- `saveWorktreeState(worktreeSession)` — 持久化到项目配置
- `clearSystemPromptSections()` + `clearMemoryFileCaches()` — 重新加载 worktree 中的 `CLAUDE.md` 和指令文件
- 清除 `getPlansDirectory` 的 memoized cache

这保证了后续工具调用时，cwd 和系统提示都反映 worktree 的上下文。

---

## 退出流程：ExitWorktreeTool

`ExitWorktreeTool` 的实现位于 [`packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts`](/packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts)。它支持两种策略：`keep` 和 `remove`。

### `keep`：保留 worktree

- `keepWorktree()`（[`worktree.ts#L780-L812`](/packages/ccb/src/utils/worktree.ts#L780-L812)）先 `process.chdir(originalCwd)`
- 清空 `currentWorktreeSession` 但不删除目录或分支
- `saveWorktreeState(null)` 将项目配置中的 `activeWorktreeSession` 置空
- `restoreSessionToOriginalCwd()` 恢复 `setCwd/setOriginalCwd/projectRoot` 和相关缓存

用户后续可以通过 `cd <worktreePath>` 继续在该 worktree 中工作，或手动发起 PR/合并。

### `remove`：删除 worktree（带安全防护）

#### 第一道防线：`validateInput`

在工具调用被模型确认之前，`validateInput` 就会执行严格检查（[`ExitWorktreeTool.ts#L174-L223`](/packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts#L174-L223)）：

1. **Scope guard**：`getCurrentWorktreeSession()` 为空则直接拒绝。这意味着只有**当前会话**通过 `EnterWorktree` 创建的 worktree 才能被此工具删除；手动 `git worktree add` 的或之前会话遗留的 worktree 不会被触碰。
2. **变更统计**：调用 `countWorktreeChanges()`（[`ExitWorktreeTool.ts#L79-L113`](/packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts#L79-L113)），分别执行：
   - `git status --porcelain` → 未提交文件数
   - `git rev-list --count <originalHeadCommit>..HEAD` → 新提交数
3. **fail-closed**：若上述任一 git 命令返回非零，或 `originalHeadCommit` 未定义（hook-based worktree），`countWorktreeChanges` 返回 `null`。此时 `validateInput` 直接拒绝删除，要求用户显式传入 `discard_changes: true`。
4. 若统计出 `changedFiles > 0` 或 `commits > 0`，同样拒绝并提示用户确认。

#### 第二道防线：`call` 执行时的重新校验

即使通过了 `validateInput`，`call()` 中仍会**再次**调用 `countWorktreeChanges()`（[`ExitWorktreeTool.ts#L256-L259`](/packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts#L256-L259)），以捕获验证与执行之间模型或用户可能新增的文件修改。

#### 实际删除流程

```typescript
if (tmuxSessionName) {
  await killTmuxSession(tmuxSessionName)
}
await cleanupWorktree()
restoreSessionToOriginalCwd(originalCwd, projectRootIsWorktree)
```

`cleanupWorktree()`（[`worktree.ts#L813-L901`](/packages/ccb/src/utils/worktree.ts#L813-L901)）的行为：

- **hook-based**：执行 `WorktreeRemove` hook，若未配置则日志告警但跳过
- **git-based**：从 `originalCwd`（主仓库根目录）执行 `git worktree remove --force <worktreePath>`，随后 `git branch -D <worktreeBranch>`

最后更新项目配置、清空缓存，恢复原始会话上下文。`restoreSessionToOriginalCwd()` 还会判断 `projectRootIsWorktree` 来决定是否恢复 `projectRoot` 和 hooks config snapshot，避免误触用户之前的 `cd` 状态（[`ExitWorktreeTool.ts#L122-L146`](/packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts#L122-L146)）。

---

## 与 Agent 工具的联动

Agent 工具（`AgentTool`）的 `isolation` 参数决定子 Agent 是否在 worktree 中运行：

- `isolation: "worktree"` → 调用 `createAgentWorktree(slug)`（[`worktree.ts#L902-L960`](/packages/ccb/src/utils/worktree.ts#L902-L960)）
- 无 `isolation` → 子 Agent 共享主工作目录

`createAgentWorktree` 与 `createWorktreeForSession` 的核心区别：**不触碰全局 session 状态**。它只创建/恢复 worktree 并返回路径，不会修改 `process.chdir` 或持久化 `activeWorktreeSession`。子 Agent 结束后，由父 Agent 调用 `removeAgentWorktree()`（[`worktree.ts#L961-L1057`](/packages/ccb/src/utils/worktree.ts#L961-L1057)）进行清理。

### 遗留 worktree 的自动清理

对于异常退出（Ctrl+C、ESC、crash）导致未清理的临时 Agent worktree，系统维护了一个 `cleanupStaleAgentWorktrees()` 扫描器（[`worktree.ts#L1058-L1143`](/packages/ccb/src/utils/worktree.ts#L1058-L1143)）。它只匹配若干严格的 ephemeral slug 正则（如 `agent-a[0-9a-f]{7}`、`wf_...`），并且：

- 跳过当前会话正在使用的 worktree
- 检测 `mtime` 是否超过 cutoff（默认 30 天）
- 使用 `git status --porcelain -uno` 和 `git rev-list HEAD --not --remotes` 做 fail-closed 检查：任何非零退出或输出非空都跳过
- 清理结束后执行 `git worktree prune`

---

## Session 状态持久化

`WorktreeSession` 对象通过 `saveCurrentProjectConfig()` 持久化到磁盘，数据结构如下：

```typescript
{
  originalCwd: string,
  worktreePath: string,
  worktreeName: string,
  worktreeBranch?: string,
  originalBranch?: string,
  originalHeadCommit?: string,
  sessionId: string,
  tmuxSessionName?: string,
  hookBased?: boolean,
  creationDurationMs?: number,
  usedSparsePaths?: boolean,
}
```

参见 [`worktree.ts#L140-L157`](/packages/ccb/src/utils/worktree.ts#L140-L157)。这使得 session 恢复（`--resume`）时能从项目配置中正确还原 worktree 上下文，即使进程已经重启。

---

## Tmux 集成：--worktree --tmux

`execIntoTmuxWorktree()`（[`worktree.ts#L1180-L1519`](/packages/ccb/src/utils/worktree.ts#L1180-L1519)）是 CLI 快速路径：在 `cli.tsx` 的早期参数解析阶段，若检测到 `--worktree` + `--tmux`，就直接创建工作树并 `exec`/`attach` 进 tmux。这支持：

- 经典 tmux 模式
- iTerm2 的 `-CC` control mode（默认启用，除非指定 `--tmux=classic`）
- 自动检测 tmux prefix 是否与 Claude 快捷键冲突并给出提示
- 为内部开发仓库（`claude-cli-internal`）自动创建 watch/start 的 dev panes

---

## claw-code Rust 实现现状

在 `claw-code` 的 Rust 重写代码（`rust/`）中，**目前尚未实现** `EnterWorktree` / `ExitWorktree` 工具。`PARITY.md` 的工具矩阵中仅列出了 `EnterPlanMode` / `ExitPlanMode`（worktree-local 的 plan mode 开关），并标记为 **good parity**：

- `rust/crates/tools/src/lib.rs#L4584-L4651` / `L4653-L4723` — `execute_enter_plan_mode` / `execute_exit_plan_mode` 的实现，读取和写入 `.claude/settings.json` 的局部覆盖，并维护一个 `plan_mode_state_file()` 进行状态回溯。

完整的 worktree 生命周期管理（git worktree 创建/销毁、Agent 隔离、tmux 集成、sparse checkout）目前仍是 `packages/ccb`（上游 TypeScript 实现）独有的能力，也是 Rust 端后续需要补齐的重要功能缺口。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| [`packages/ccb/docs/agent/worktree-isolation.mdx`](/packages/ccb/docs/agent/worktree-isolation.mdx) | 原始文档（本报告的信息源） |
| [`packages/ccb/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`](/packages/ccb/src/tools/EnterWorktreeTool/EnterWorktreeTool.ts) | `EnterWorktree` 工具定义、schema、call 实现 |
| [`packages/ccb/src/tools/EnterWorktreeTool/prompt.ts`](/packages/ccb/src/tools/EnterWorktreeTool/prompt.ts) | 工具触发条件与行为说明 |
| [`packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts`](/packages/ccb/src/tools/ExitWorktreeTool/ExitWorktreeTool.ts) | `ExitWorktree` 工具定义、validateInput、call、失败闭合设计 |
| [`packages/ccb/src/tools/ExitWorktreeTool/prompt.ts`](/packages/ccb/src/tools/ExitWorktreeTool/prompt.ts) | 工具退出策略说明 |
| [`packages/ccb/src/utils/worktree.ts#L66-L157`](/packages/ccb/src/utils/worktree.ts#L66-L157) | `validateWorktreeSlug` / 路径安全校验 |
| [`packages/ccb/src/utils/worktree.ts#L217-L222`](/packages/ccb/src/utils/worktree.ts#L217-L222) | `flattenSlug` / slug 扁平化 |
| [`packages/ccb/src/utils/worktree.ts#L235-L509`](/packages/ccb/src/utils/worktree.ts#L235-L509) | `getOrCreateWorktree`：快速恢复、fetch 优化、git worktree add |
| [`packages/ccb/src/utils/worktree.ts#L510-L652`](/packages/ccb/src/utils/worktree.ts#L510-L652) | `performPostCreationSetup`：配置传播、hooks、symlink、`.worktreeinclude` |
| [`packages/ccb/src/utils/worktree.ts#L702-L779`](/packages/ccb/src/utils/worktree.ts#L702-L779) | `createWorktreeForSession`：hook/git 分发、session 组装 |
| [`packages/ccb/src/utils/worktree.ts#L780-L812`](/packages/ccb/src/utils/worktree.ts#L780-L812) | `keepWorktree`：保留 worktree 并恢复目录 |
| [`packages/ccb/src/utils/worktree.ts#L813-L901`](/packages/ccb/src/utils/worktree.ts#L813-L901) | `cleanupWorktree`：git worktree remove --force + branch -D |
| [`packages/ccb/src/utils/worktree.ts#L902-L960`](/packages/ccb/src/utils/worktree.ts#L902-L960) | `createAgentWorktree`：为子 Agent 创建轻量 worktree |
| [`packages/ccb/src/utils/worktree.ts#L961-L1057`](/packages/ccb/src/utils/worktree.ts#L961-L1057) | `removeAgentWorktree`：清理子 Agent worktree |
| [`packages/ccb/src/utils/worktree.ts#L1058-L1143`](/packages/ccb/src/utils/worktree.ts#L1058-L1143) | `cleanupStaleAgentWorktrees`：过期 ephemeral worktree 自动清理 |
| [`packages/ccb/src/utils/worktree.ts#L1144-L1179`](/packages/ccb/src/utils/worktree.ts#L1144-L1179) | `hasWorktreeChanges`：变更检测（fail-closed） |
| [`packages/ccb/src/utils/worktree.ts#L1180-L1519`](/packages/ccb/src/utils/worktree.ts#L1180-L1519) | `execIntoTmuxWorktree`：`--worktree --tmux` CLI 快速路径 |
| [`rust/crates/tools/src/lib.rs#L4584-L4651`](/rust/crates/tools/src/lib.rs#L4584-L4651) / [`L4653-L4723`](/rust/crates/tools/src/lib.rs#L4653-L4722) | Rust 侧 `execute_enter_plan_mode` / `execute_exit_plan_mode` |
| [`rust/PARITY.md#L82-L83`](/rust/PARITY.md#L82-L83) | Rust 端 parity 状态：仅有 EnterPlanMode/ExitPlanMode |
