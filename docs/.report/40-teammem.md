# Unit 40 — Teammem 技术报告

## 原始来源

- 文档页面：https://ccb.agent-aura.top/docs/features/teammem
- 版本：2026-04-09


> **源码映射说明**：Teammem 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。

## 代码库定位

> 子项目：`packages/ccb`（Anthropic Claude Code CLI 的逆向工程/恢复版本）  
> 主仓库：`claw-code`

---

## 一、功能概述

Teammem（Team Memory）是基于 GitHub 仓库的团队共享记忆系统。`memory/team/` 目录中的文件会与 Anthropic 服务器进行双向同步，团队内所有通过 Anthropic OAuth 认证的成员均可共享项目知识。

核心特性：
- **增量同步**：仅上传内容哈希（sha256）发生变化的文件（delta upload）。
- **冲突解决**：基于 ETag 的乐观锁 + 412 冲突重试。
- **密钥扫描**：上传前通过 gitleaks 规则检测并跳过包含密钥的文件（PSR M22174）。
- **路径穿越防护**：所有写入路径验证在 `memory/team/` 边界内，且通过 `realpath` 解析符号链接防止逃逸（PSR M22186）。
- **分批上传**：将 PUT 请求体拆分为 ≤200KB 的批次，避免 API 网关（~256–512KB）拒绝。

Feature Flag：`FEATURE_TEAMMEM=1`

---

## 二、文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| `packages/ccb/src/services/teamMemorySync/index.ts` | 1256 | 核心同步逻辑（pull/push/sync、ETag 管理、delta 计算、413 容量学习） |
| `packages/ccb/src/services/teamMemorySync/watcher.ts` | 388 | 文件监视（`fs.watch` recursive）+ 自动 debounced push |
| `packages/ccb/src/services/teamMemorySync/secretScanner.ts` | 325 | 客户端密钥扫描（gitleaks 规则子集） |
| `packages/ccb/src/services/teamMemorySync/types.ts` | 157 | Zod schema + 类型定义 |
| `packages/ccb/src/services/teamMemorySync/teamMemSecretGuard.ts` | 44 | 密钥防护辅助（供 FileWriteTool/FileEditTool 调用） |
| `packages/ccb/src/memdir/teamMemPaths.ts` | 293 | 路径验证 + 目录管理 + 符号链接安全检查 |
| `packages/ccb/src/memdir/teamMemPrompts.ts` | 101 | 组合 memory prompt（private + team） |
| `packages/ccb/src/utils/teamMemoryOps.ts` | 62 | 工具层辅助：判断 tool use 是否针对 team memory |

---

## 三、源码详解

### 3.1 同步状态（SyncState）

文件：`packages/ccb/src/services/teamMemorySync/index.ts` #L100–#L127

```ts
export type SyncState = {
  lastKnownChecksum: string | null               // ETag 条件请求
  serverChecksums: Map<string, string>          // sha256:<hex> 逐文件哈希
  serverMaxEntries: number | null               // 从 413 学习的服务端容量
}

export function createSyncState(): SyncState {
  return {
    lastKnownChecksum: null,
    serverChecksums: new Map(),
    serverMaxEntries: null,
  }
}
```

- `serverChecksums` 在 pull 时由 `entryChecksums` 刷新（`#L822–#L834`）。
- `serverMaxEntries` 仅在收到结构化 413 时写入（`#L1044–#L1049`），策略是“不预设客户端上限”。

### 3.2 Pull 流程（Server → Local）

文件：`packages/ccb/src/services/teamMemorySync/index.ts` #L770–#L867

入口：`export async function pullTeamMemory(state: SyncState, ...)`

流程：
1. `#L779–#L787` 检查 OAuth（`isUsingOAuth()`）。
2. `#L789–#L797` 检查 GitHub remote（`getGithubRepo()`）。
3. `#L800` 调用 `fetchTeamMemory(state, repoSlug, etag)`（带重试，`#L387–#L424`）。
   - 304 Not Modified → 快速返回。
   - 404 → 清除 `serverChecksums`。
   - 200 → 解析 `TeamMemoryData`。
4. `#L822–#L834` 刷新 `serverChecksums`。
5. `#L844` 调用 `writeRemoteEntriesToLocal(entries)`（`#L689–#L760`）：
   - 每个 entry 先经过 `validateTeamMemKey(relPath)`（`#L694–#L703`）。
   - 超大文件（>250KB）跳过（`#L705–#L713`）。
   - 内容相同则跳过写入，避免无效 watcher 事件（`#L715–#L728`）。
   - 并行写入所有文件（`Promise.all`，`#L689`）。

关键设计：server-wins on pull。服务端内容无条件覆盖本地。

### 3.3 Push 流程（Local → Server）

文件：`packages/ccb/src/services/teamMemorySync/index.ts` #L889–#L1151

入口：`export async function pushTeamMemory(state: SyncState)`

流程：
1. `#L917–#L924` 读取本地文件（`readLocalTeamMemory`）。
2. `#L948–#L968` 扫描密钥；若检测到，跳过该文件并记录 `tengu_team_mem_secret_skipped` 事件。
3. `#L976–#L980` 预计算所有本地条目的 sha256。
4. `#L976–#L1151` 冲突重试循环（最多 `MAX_CONFLICT_RETRIES = 2` 次）：
   - `#L987–#L993` 计算 delta：本地哈希 ≠ `serverChecksums` 的 key 才上传。
   - `#L1006–#L1007` 调用 `batchDeltaByBytes(delta)`（`#L426–#L460`）拆分为 ≤200KB 的批次。
   - `#L1010–#L1020` 逐批 `uploadTeamMemory(...)`（`#L462–#L565`），每批使用 `If-Match` 乐观锁。
   - 若遇到 412 conflict：
     - `#L1111–#L1115` 调用 `fetchTeamMemoryHashes(state, repoSlug)` 仅获取 checksums。
     - `#L1115–#L1119` 刷新 `serverChecksums`，重算 delta 后重试。
   - 若遇到 413：
     - 解析结构化错误体，学习 `serverMaxEntries`（`#L1044–#L1049`）。

关键设计：local-wins on push。用户的本地编辑不会被静默丢弃。

#### batchDeltaByBytes（分批算法）

文件：`packages/ccb/src/services/teamMemorySync/index.ts` #L426–#L460

- 固定开销 `{"entries":{}}` = `EMPTY_BODY_BYTES`。
- 按键名字典序排序后贪心装箱，保证批次确定性（对 ETag 重试稳定）。
- 单文件超过上限时自成一批（因为单文件上限 250KB 已接近网关阈值）。

### 3.4 读取本地文件与密钥扫描

文件：`packages/ccb/src/services/teamMemorySync/index.ts` #L567–#L687

`readLocalTeamMemory(maxEntries)`：
- 递归扫描 `memory/team/`（`walkDir`，`#L575–#L650`）。
- 跳过超大文件（`stats.size > MAX_FILE_SIZE_BYTES`，`#L588–#L594`）。
- `#L599–#L614` **密钥扫描**：在加入上传集之前调用 `scanForSecrets(content)`。
- 若 `maxEntries !== null` 且本地文件数超过上限，按字典序截断并记录 `tengu_team_mem_entries_capped`（`#L653–#L681`）。

### 3.5 密钥扫描器

文件：`packages/ccb/src/services/teamMemorySync/secretScanner.ts`

- `#L39–#L225` 定义了精选的 gitleaks 高置信度规则（AWS、GCP、Azure、Anthropic、OpenAI、GitHub、Slack、Stripe、Private Key 等）。
- `#L229–#L238` 懒编译正则缓存。
- `#L277–#L296` `scanForSecrets(content)` 返回命中的 ruleId 列表（不返回匹配文本）。
- `#L312–#L325` `redactSecrets(content)` 用 `[REDACTED]` 替换捕获组。

Anthropic 密钥前缀采用运行时拼接（`#L46`），避免字面量在 bundle 中被静态检测到：
```ts
const ANT_KEY_PFX = ['sk', 'ant', 'api'].join('-')
```

### 3.6 文件监视器

文件：`packages/ccb/src/services/teamMemorySync/watcher.ts`

- `#L167–#L230` `startFileWatcher(teamDir)`：
  - 使用 Node.js `fs.watch({ recursive: true })`（macOS 上走 FSEvents，保持 O(1) fd）。
  - 目录不存在时自动 `mkdir(teamDir, { recursive: true })`。
- `#L35` debounce 时间 2000ms。
- `#L45–#L52` `pushSuppressedReason`：当遇到不可自愈错误（no_oauth / 4xx 非 409/429）时永久抑制重试，防止会话期间产生无限 push 循环（参考 BQ Mar 14-16：一台无 OAuth 设备在 2.5 天内产生了 167K push 事件）。
- `#L252–#L306` `startTeamMemoryWatcher()`： feature flag + OAuth + GitHub remote 检查通过后先 `pullTeamMemory`，再启动 watcher。
- `#L314–#L320` `notifyTeamMemoryWrite()`：供 `PostToolUse` hooks 调用，显式调度 push。
- `#L327–#L353` `stopTeamMemoryWatcher()`：清理 debounce timer、关闭 watcher、等待 in-flight push、flush pending changes（best-effort）。

### 3.7 路径安全

文件：`packages/ccb/src/memdir/teamMemPaths.ts`

- `#L84–#L87` `getTeamMemPath()`：返回 `<autoMem>/team/`（NFC 规范化）。
- `#L22–#L65` `sanitizePathKey(key)`：防御 null byte、URL-encoded traversal（`%2e%2e%2f`）、Unicode normalization attack（全角 `．．／` NFKC 后变为 `../`）、反斜杠、绝对路径。
- `#L109–#L172` `realpathDeepestExisting(absolutePath)`：对最深存在的祖先调用 `realpath()` 解析符号链接（PSR M22186）。检测 dangling symlink 与 symlink loop。
- `#L228–#L257` `validateTeamMemWritePath(filePath)`：
  - 第一遍 `path.resolve()` 做字符串级 containment 检查。
  - 第二遍 `realpathDeepestExisting()` 解析 symlink 后再确认仍在 teamDir 内。
- `#L265–#L285` `validateTeamMemKey(relativeKey)`：供 pull 时使用，对服务端下发的相对 key 做同样两阶段验证。

### 3.8 类型定义

文件：`packages/ccb/src/services/teamMemorySync/types.ts`

- `#L16–#L25` `TeamMemoryContentSchema`：`entries` + 可选 `entryChecksums`。
- `#L29–#L39` `TeamMemoryDataSchema`：完整 GET 响应（含 `organizationId`、`repo`、`version`、`lastModified`、`checksum`、`content`）。
- `#L47–#L58` `TeamMemoryTooManyEntriesSchema`：结构化 413 错误体（`error_code = 'team_memory_too_many_entries'`、`max_entries`、`received_entries`）。
- `#L77–#L88` `TeamMemorySyncFetchResult`：pull 结果。
- `#L107–#L125` `TeamMemorySyncPushResult`：push 结果（含 `skippedSecrets`）。
- `#L129–#L157` `TeamMemorySyncUploadResult`：单批次 PUT 结果（含 `serverErrorCode`、`serverMaxEntries`、`serverReceivedEntries`）。

### 3.9 写入时的密钥拦截

文件：`packages/ccb/src/services/teamMemorySync/teamMemSecretGuard.ts`

`checkTeamMemSecrets(filePath, content)`（`#L15–#L45`）：
- 被 `FileWriteTool` 和 `FileEditTool` 在 `validateInput` 阶段调用。
- 若内容包含密钥，返回错误信息阻止写入。

### 3.10 Prompt 集成

文件：`packages/ccb/src/memdir/teamMemPrompts.ts`

`buildCombinedMemoryPrompt()`（`#L22–#L101`）：
- 当 auto memory 与 team memory 同时启用时，生成组合 prompt。
- 定义了 four-type taxonomy（user / feedback / project / reference）及 `<scope>` 指导（private vs team）。
- 告知模型团队记忆存储在 `teamDir`，会跨会话同步。

### 3.11 工具层的 team memory  awareness

文件：`packages/ccb/src/utils/teamMemoryOps.ts`

- `isTeamMemorySearch(toolInput)`（`#L10–#L22`）
- `isTeamMemoryWriteOrEdit(toolName, toolInput)`（`#L26–#L38`）
- `appendTeamMemorySummaryParts(...)`（`#L39–#L63`）

供搜索/读写摘要渲染使用。

---

## 四、API 端点

- `GET  /api/claude_code/team_memory?repo={owner/repo}`             → 完整数据 + entryChecksums
- `GET  /api/claude_code/team_memory?repo={owner/repo}&view=hashes` → 仅 checksums（冲突解决用）
- `PUT  /api/claude_code/team_memory?repo={owner/repo}`             → 上传 entries（upsert 语义）

---

## 五、关键设计决策与源码锚点

| 决策 | 说明 | 源码锚点 |
|------|------|----------|
| Server-wins on pull | 服务端内容覆盖本地；pull 是只读合并 | `#L770–#L867` |
| Local-wins on push | 用户本地编辑优先，不会被并发 push 静默丢弃 | `#L889–#L1151` |
| Delta upload | 仅上传哈希不同的条目，通过 `serverChecksums` 比较 | `#L987–#L993` |
| 分批 PUT | `batchDeltaByBytes` 拆分为 ≤200KB 批次 | `#L426–#L460` |
| 密钥扫描在 upload 前 | `readLocalTeamMemory` 中调用 `scanForSecrets` | `#L599–#L614` |
| ETag 乐观锁 | `uploadTeamMemory` 使用 `If-Match` header | `#L478–#L483` |
| 412 轻量探针 | `fetchTeamMemoryHashes` 只取 checksums | `#L315–#L385` |
| 服务端容量动态学习 | 从 413 解析 `extra_details.max_entries` | `#L1044–#L1050` |
| 路径穿越 + symlink 防护 | `validateTeamMemKey` 两阶段验证（resolve + realpath） | `#L265–#L285` |
| Watcher debounce + 永久失败抑制 | `pushSuppressedReason` 防止 no_oauth 无限重试 | `#L45–#L52`, `#L103–#L117` |

---

## 六、外部依赖与缺失项

- **Anthropic OAuth**：first-party 认证（`isUsingOAuth`，`#L151–#L162`）。
- **GitHub Remote**：`getGithubRepo()` 获取 `owner/repo` 作为同步 scope。
- **Team Memory API**：`/api/claude_code/team_memory` 端点。
- **gitleaks**：规则模式来源（MIT license），仅选取高置信度前缀规则。

> **实现风险**：当前密钥扫描器仅覆盖 gitleaks 高置信度前缀规则子集，新型密钥格式（如自定义 JWT、短-lived 云令牌）可能漏检。迁移到完整 gitleaks 规则集或引入熵检测可提升覆盖率，但会增加误报。

---

## 七、使用方式

```bash
# 启用 feature
FEATURE_TEAMMEM=1 bun run dev

# 前提条件：
# 1. 已通过 Anthropic OAuth 登录
# 2. 项目有 GitHub remote（git remote -v 显示 origin）
# 3. memory/team/ 目录自动创建
```

---

## 八、相关事件（Analytics）

- `tengu_team_mem_sync_started`
- `tengu_team_mem_sync_pull`
- `tengu_team_mem_sync_push`
- `tengu_team_mem_secret_skipped`
- `tengu_team_mem_entries_capped`
- `tengu_team_mem_push_suppressed`
