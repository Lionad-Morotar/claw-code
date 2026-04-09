# Unit 54: Auto Dream — 自动记忆整理技术报告

**原始文档**: https://ccb.agent-aura.top/docs/features/auto-dream  
**报告生成日期**: 2026-04-09  
**执行者**: Claude Code (claw-code 仓库)

---

## 1. 概述

Auto Dream 是 Claude Code 的后台记忆整合机制。它在会话间自动审查、组织和修剪持久化记忆文件，确保未来会话能快速获得准确的上下文。

### 1.1 核心功能

- **自动触发**: 满足时间/会话门控条件后，以 forked agent 方式在后台运行
- **手动触发**: 通过 `/dream` 命令无条件执行
- **四阶段整理**: Orient → Gather → Consolidate → Prune
- **锁文件机制**: 防止并发执行，支持死进程检测

---

## 2. 架构

### 2.1 核心模块

| 模块 | 路径 | 职责 | 行号 |
|------|------|------|------|
| 调度器 | `src/services/autoDream/autoDream.ts` | 时间/会话/锁三重门控，触发 forked agent | L1-L327 |
| 配置 | `src/services/autoDream/config.ts` | 读取 `isAutoDreamEnabled()` 开关 | L1-L22 |
| 提示词 | `src/services/autoDream/consolidationPrompt.ts` | 构建 4 阶段整理提示词 | L1-L66 |
| 锁文件 | `src/services/autoDream/consolidationLock.ts` | PID 锁 + mtime 作为 `lastConsolidatedAt` | L1-L141 |
| 任务 UI | `src/tasks/DreamTask/DreamTask.ts` | 后台任务注册，footer pill + Shift+Down 可见 | L1-L158 |
| 手动入口 | `src/skills/bundled/dream.ts` | `/dream` 命令，无条件可用 | L1-L45 |
| 路径解析 | `src/memdir/paths.ts` | 记忆目录路径解析 | L1-L279 |
| 进度对话框 | `src/components/tasks/DreamDetailDialog.tsx` | UI 详情展示 | L1-L134 |

### 2.2 记忆路径解析 (`src/memdir/paths.ts`)

优先级顺序：

1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量（完整路径覆盖）— L161-L166
2. `autoMemoryDirectory` 设置项（`settings.json`，支持 `~/` 展开）— L179-L186
3. 默认：`<memoryBase>/projects/<sanitized-git-root>/memory/` — L223-L235

其中 `memoryBase` = `CLAUDE_CODE_REMOTE_MEMORY_DIR` 或 `~/.claude`。

---

## 3. 触发机制

### 3.1 自动触发（Auto Dream）

每个对话轮次结束后，`executeAutoDream()` 按顺序检查三重门控：

```typescript
// src/services/autoDream/autoDream.ts L96-L101
function isGateOpen(): boolean {
  if (getKairosActive()) return false // KAIROS 模式 uses disk-skill dream
  if (getIsRemoteMode()) return false
  if (!isAutoMemoryEnabled()) return false
  return isAutoDreamEnabled()
}
```

#### 三重门控流程

```
┌─────────────────────────────────────────────────────┐
│  Gate 1: 全局开关                                     │
│  isAutoMemoryEnabled() && isAutoDreamEnabled()       │
│  排除：KAIROS 模式 / Remote 模式                      │
├─────────────────────────────────────────────────────┤
│  Gate 2: 时间门控                                     │
│  hoursSince(lastConsolidatedAt) >= minHours          │
│  默认：24 小时                                        │
├─────────────────────────────────────────────────────┤
│  Gate 3: 会话门控                                     │
│  sessionsTouchedSince(lastConsolidatedAt) >= minSessions │
│  默认：5 个会话（排除当前会话）                         │
├─────────────────────────────────────────────────────┤
│  Lock: PID 锁文件                                     │
│  .consolidate-lock (mtime = lastConsolidatedAt)      │
│  死进程检测 + 1 小时过期                               │
└─────────────────────────────────────────────────────┘
```

**代码实现**:
- 时间门控: `L131-L142` — 读取 `lastConsolidatedAt` 并计算经过小时数
- 会话门控: `L154-L172` — 扫描 transcripts 目录，过滤出满足条件的会话
- 锁获取: `L174-L191` — 调用 `tryAcquireConsolidationLock()`

#### Forked Agent 执行

全部通过后，以 `forked agent`（受限子代理）方式运行整理任务：

```typescript
// src/services/autoDream/autoDream.ts L225-L234
const result = await runForkedAgent({
  promptMessages: [createUserMessage({ content: prompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool: createAutoMemCanUseTool(memoryRoot),
  querySource: 'auto_dream',
  forkLabel: 'auto_dream',
  skipTranscript: true,
  overrides: { abortController },
  onMessage: makeDreamProgressWatcher(taskId, setAppState),
})
```

**约束**:
- Bash 工具限制为只读命令（`ls`、`grep`、`cat` 等）— `L219`
- 只能读写记忆目录内的文件 — `createAutoMemCanUseTool()` at `src/services/extractMemories/extractMemories.ts#L171-L222`
- 用户可在 Shift+Down 后台任务面板中查看进度或终止

### 3.2 手动触发（`/dream` 命令）

通过 `/dream` 命令随时触发，无门控限制：

```typescript
// src/skills/bundled/dream.ts L27-L44
async getPromptForCommand(args) {
  const memoryRoot = getAutoMemPath()
  const transcriptDir = getProjectDir(getOriginalCwd())

  // Stamp the consolidation lock optimistically
  await recordConsolidation()

  const basePrompt = buildConsolidationPrompt(memoryRoot, transcriptDir, '')
  let prompt = DREAM_PROMPT_PREFIX + basePrompt

  if (args) {
    prompt += `\n\n## Additional context from user\n\n${args}`
  }

  return [{ type: 'text', text: prompt }]
}
```

**特点**:
- 在主循环中运行（非 forked agent），拥有完整工具权限
- 用户可实时观察操作过程
- 执行前自动更新锁文件 mtime — `recordConsolidation()` at `src/services/autoDream/consolidationLock.ts#L130-L140`

### 3.3 配置开关

| 开关 | 位置 | 作用 |
|------|------|------|
| `autoDreamEnabled` | `settings.json` | `true`/`false` 显式开关 |
| `autoMemoryEnabled` | `settings.json` | 总开关，关闭后所有记忆功能禁用 |
| `CLAUDE_CODE_DISABLE_AUTO_MEMORY` | 环境变量 | `1`/`true` 关闭所有记忆功能 |
| `tengu_onyx_plover` | GrowthBook | 官方远程配置，控制 `enabled`/`minHours`/`minSessions` |

默认值（无 GrowthBook 连接时）：
```typescript
// src/services/autoDream/autoDream.ts L64-L67
const DEFAULTS: AutoDreamConfig = {
  minHours: 24,
  minSessions: 5,
}
```

---

## 4. 整理流程（4 阶段）

Dream agent 执行的提示词包含 4 个阶段：

### Phase 1 — 定位（Orient）

```typescript
// src/services/autoDream/consolidationPrompt.ts L26-L31
## Phase 1 — Orient

- `ls` the memory directory to see what already exists
- Read `MEMORY.md` to understand the current index
- Skim existing topic files so you improve them rather than creating duplicates
- If `logs/` or `sessions/` subdirectories exist, review recent entries there
```

### Phase 2 — 采集信号（Gather）

```typescript
// src/services/autoDream/consolidationPrompt.ts L33-L42
## Phase 2 — Gather recent signal

Look for new information worth persisting. Sources in rough priority order:

1. **Daily logs** (`logs/YYYY/MM/YYYY-MM-DD.md`) if present
2. **Existing memories that drifted** — facts that contradict something you see in the codebase now
3. **Transcript search** — grep the JSONL transcripts for narrow terms
```

### Phase 3 — 整合（Consolidate）

```typescript
// src/services/autoDream/consolidationPrompt.ts L44-L52
## Phase 3 — Consolidate

For each thing worth remembering, write or update a memory file at the top level of the memory directory.

Focus on:
- Merging new signal into existing topic files rather than creating near-duplicates
- Converting relative dates ("yesterday", "last week") to absolute dates
- Deleting contradicted facts — if today's investigation disproves an old memory, fix it at the source
```

### Phase 4 — 修剪与索引（Prune）

```typescript
// src/services/autoDream/consolidationPrompt.ts L54-L61
## Phase 4 — Prune and index

Update `MEMORY.md` so it stays under 200 lines AND under ~25KB.

- Remove pointers to memories that are now stale, wrong, or superseded
- Demote verbose entries
- Add pointers to newly important memories
- Resolve contradictions — if two files disagree, fix the wrong one
```

---

## 5. 记忆类型

记忆系统使用 4 种类型（`src/memdir/memoryTypes.ts`）：

| 类型 | 用途 | 示例 |
|------|------|------|
| `user` | 用户角色、偏好、知识 | 用户是高级后端工程师，偏好中文交流 |
| `feedback` | 工作方式指导 | 不要 mock 数据库测试；代码审查用 bundled PR |
| `project` | 项目上下文（非代码可推导的） | 合并冻结从 3 月 5 日开始；认证重写是合规需求 |
| `reference` | 外部系统指针 | Linear INGEST 项目跟踪 pipeline bugs |

**不保存的内容** — `src/memdir/memoryTypes.ts#L183-L195`:
- 代码模式、架构、文件路径（可从代码推导）
- Git 历史（`git log` 权威）
- 调试方案（代码中已有）

---

## 6. 锁文件机制

`.consolidate-lock` 文件位于记忆目录内：

### 文件结构

```typescript
// src/services/autoDream/consolidationLock.ts
const LOCK_FILE = '.consolidate-lock'
const HOLDER_STALE_MS = 60 * 60 * 1000 // 1 小时过期
```

- **文件内容**: 持有者 PID — `L72`
- **mtime**: 即 `lastConsolidatedAt` 时间戳 — `L29-L36`
- **过期**: 1 小时（防 PID 复用）— `L60-L68`

### 竞态处理

```typescript
// src/services/autoDream/consolidationLock.ts L46-L84
export async function tryAcquireConsolidationLock(): Promise<number | null> {
  // 1. Read existing lock file mtime and PID
  // 2. Check if stale (>1h) or PID dead → reclaim
  // 3. Write new PID
  // 4. Verify PID (last writer wins, loser bails)
}
```

### 回滚机制

```typescript
// src/services/autoDream/consolidationLock.ts L91-L108
export async function rollbackConsolidationLock(priorMtime: number): Promise<void> {
  // Rewind mtime to pre-acquire after a failed fork
  // Clear the PID body
}
```

---

## 7. 任务系统整合

### DreamTask 注册

```typescript
// src/tasks/DreamTask/DreamTask.ts L52-L74
export function registerDreamTask(
  setAppState: SetAppState,
  opts: {
    sessionsReviewing: number
    priorMtime: number
    abortController: AbortController
  },
): string {
  const id = generateTaskId('dream')
  const task: DreamTaskState = {
    ...createTaskStateBase(id, 'dream', 'dreaming'),
    type: 'dream',
    status: 'running',
    phase: 'starting',
    sessionsReviewing: opts.sessionsReviewing,
    filesTouched: [],
    turns: [],
    abortController: opts.abortController,
    priorMtime: opts.priorMtime,
  }
  registerTask(task, setAppState)
  return id
}
```

### 进度监控

```typescript
// src/services/autoDream/autoDream.ts L282-L315
function makeDreamProgressWatcher(
  taskId: string,
  setAppState: import('../../Task.js').SetAppState,
): (msg: Message) => void {
  return msg => {
    if (msg.type !== 'assistant') return
    let text = ''
    let toolUseCount = 0
    const touchedPaths: string[] = []
    for (const block of contentBlocks) {
      if (block.type === 'text') {
        text += block.text
      } else if (block.type === 'tool_use') {
        toolUseCount++
        if (block.name === FILE_EDIT_TOOL_NAME || block.name === FILE_WRITE_TOOL_NAME) {
          const input = block.input as { file_path?: unknown }
          if (typeof input.file_path === 'string') {
            touchedPaths.push(input.file_path)
          }
        }
      }
    }
    addDreamTurn(taskId, { text, toolUseCount }, touchedPaths, setAppState)
  }
}
```

### UI 展示

```tsx
// src/components/tasks/DreamDetailDialog.tsx L20-L133
export function DreamDetailDialog({ task, onDone, onBack, onKill }: Props) {
  // Shows elapsed time, sessions reviewing, files touched
  // Displays last 6 turns with text output
  // Tool-only turns collapse to a count
}
```

---

## 8. 使用场景

### 场景 1：日常开发中的自动整理

开发者连续多天使用 Claude Code 处理不同任务。Auto Dream 在积累 5+ 个会话且距上次整理 24 小时后自动触发，整合分散在多次会话中的用户偏好和项目决策。

### 场景 2：手动整理记忆

用户发现 Claude 重复犯相同错误或遗忘之前的决策。输入 `/dream` 立即触发整理，无需等待自动触发周期。

### 场景 3：新会话快速上下文

新会话启动时，`MEMORY.md` 被加载到上下文中。经过 Dream 整理的记忆文件结构清晰、信息准确，让 Claude 快速了解用户和项目。

### 场景 4：KAIROS 模式下的日志蒸馏

KAIROS（长驻助手模式）中，agent 以追加方式写入日期日志文件。Dream 负责将这些日志蒸馏为主题文件和 `MEMORY.md` 索引。

---

## 9. 与其他系统的关系

```
┌─────────────┐     ┌──────────────┐     ┌───────────────┐
│ 会话交互     │────▶│ 记忆写入      │────▶│ MEMORY.md     │
│ (主 agent)  │     │ (即时保存)    │     │ + 主题文件     │
└─────────────┘     └──────────────┘     └───────┬───────┘
                                                  │
┌─────────────────────────────────────────────────┘
▼
┌──────────────┐     ┌──────────────┐
│ Auto Dream   │────▶│ 整理/修剪    │
│ (后台触发)   │     │ 去重/纠错    │
└──────────────┘     └──────────────┘
      ▲
      │
┌──────────────┐
│ /dream 命令  │
│ (手动触发)   │
└──────────────┘
```

**相关系统**:
- `extractMemories` (`src/services/extractMemories/`) — 每轮次结束时从对话中提取新记忆并写入。Dream 不负责提取，只负责整理。
- `CLAUDE.md` — 项目级指令文件，加载到上下文中但不属于记忆系统。
- `Team Memory` (`TEAMMEM` feature) — 团队共享记忆目录，与个人记忆使用相同的 Dream 机制。

---

## 10. 关键代码锚点汇总

| 功能 | 文件 | 行号 |
|------|------|------|
| 入口函数 `executeAutoDream()` | `src/services/autoDream/autoDream.ts` | L321-L326 |
| 初始化 `initAutoDream()` | `src/services/autoDream/autoDream.ts` | L123-L174 |
| 门控检查 `isGateOpen()` | `src/services/autoDream/autoDream.ts` | L96-L101 |
| 配置读取 `getConfig()` | `src/services/autoDream/autoDream.ts` | L74-L94 |
| 启用检查 `isAutoDreamEnabled()` | `src/services/autoDream/config.ts` | L13-L21 |
| 提示词构建 `buildConsolidationPrompt()` | `src/services/autoDream/consolidationPrompt.ts` | L10-L65 |
| 读锁 `readLastConsolidatedAt()` | `src/services/autoDream/consolidationLock.ts` | L29-L36 |
| 获取锁 `tryAcquireConsolidationLock()` | `src/services/autoDream/consolidationLock.ts` | L46-L84 |
| 回滚锁 `rollbackConsolidationLock()` | `src/services/autoDream/consolidationLock.ts` | L91-L108 |
| 会话扫描 `listSessionsTouchedSince()` | `src/services/autoDream/consolidationLock.ts` | L118-L124 |
| 手动记录 `recordConsolidation()` | `src/services/autoDream/consolidationLock.ts` | L130-L140 |
| 任务注册 `registerDreamTask()` | `src/tasks/DreamTask/DreamTask.ts` | L52-L74 |
| 进度更新 `addDreamTurn()` | `src/tasks/DreamTask/DreamTask.ts` | L76-L104 |
| 任务完成 `completeDreamTask()` | `src/tasks/DreamTask/DreamTask.ts` | L106-L120 |
| 任务失败 `failDreamTask()` | `src/tasks/DreamTask/DreamTask.ts` | L122-L130 |
| 任务终止 `DreamTask.kill()` | `src/tasks/DreamTask/DreamTask.ts` | L136-L156 |
| 技能注册 `registerDreamSkill()` | `src/skills/bundled/dream.ts` | L18-L44 |
| 记忆路径 `getAutoMemPath()` | `src/memdir/paths.ts` | L223-L235 |
| 记忆类型定义 | `src/memdir/memoryTypes.ts` | L14-L21 |
| 工具权限 `createAutoMemCanUseTool()` | `src/services/extractMemories/extractMemories.ts` | L171-L222 |
| Forked Agent `runForkedAgent()` | `src/utils/forkedAgent.ts` | L489-L626 |
| UI 对话框 `DreamDetailDialog` | `src/components/tasks/DreamDetailDialog.tsx` | L20-L133 |

---

## 11. 测试与验证

当前仓库测试状态：
- 单元测试位于 `src/**/__tests__/`
- 集成测试位于 `tests/integration/`
- 测试运行：`bun test`

---

## 12. 参考资料

- 原始文档：https://ccb.agent-aura.top/docs/features/auto-dream
- 子项目路径：`packages/ccb/src/services/autoDream/`
- 任务系统：`packages/ccb/src/tasks/DreamTask/`
- 记忆系统：`packages/ccb/src/memdir/`

---

*报告生成完毕 — 所有源码锚点已验证于 `packages/ccb/` 子项目*
