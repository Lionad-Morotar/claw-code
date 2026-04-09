# Unit 46 — MCP_SKILLS 技术报告

## 1. 功能概述

**Feature Flag**: `FEATURE_MCP_SKILLS=1`  
**状态**: 配置门控完整，核心 fetcher 为 stub（`packages/ccb/src/skills/mcpSkills.ts`），其余集成链路已贯通。

`MCP_SKILLS` 尝试将 MCP 服务器暴露的 `skill://` URI 方案资源发现并转换为 Claude Code 内部可调用的 `prompt` 类型 `Command` 对象。 MCP 服务器通常同时暴露 tools、prompts 和 resources；当启用此特性后，符合 `skill://` 方案的资源被识别为技能，并进入模型的 SkillTool 可用列表。

## 2. 数据流与架构

### 2.1 连接时数据流

```
MCP Server 连接
      │
      ▼
services/mcp/client.ts
  ├── fetchToolsForClient(client)      → tools
  ├── fetchCommandsForClient(client)   → prompts (MCP prompt → Command)
  ├── fetchMcpSkillsForClient(client)  → skills  (MCP skill:// resource → Command) [FEATURE_GATED]
  └── fetchResourcesForClient(client)  → resources
      │
      ▼
commands = [...mcpCommands, ...mcpSkills]
      │
      ▼
AppState.mcp.commands 更新
      │
      ▼
getMcpSkillCommands(mcpCommands) 过滤
      │
      ▼
SkillTool 调用 (通过 getAllCommands 合并本地 + MCP 技能)
```

### 2.2 实时刷新数据流（运行期 prompts/resources 变化）

```
prompts/list_changed 通知
      │
      ▼
useManageMCPConnections.ts
  ├── fetchCommandsForClient.cache.delete(name)
  ├── fetchMcpSkillsForClient!(client)   [if FEATURE_MCP_SKILLS]
  └── updateServer({ commands: [...mcpPrompts, ...mcpSkills] })

resources/list_changed 通知
      │
      ▼
useManageMCPConnections.ts
  ├── fetchResourcesForClient.cache.delete(name)
  ├── fetchMcpSkillsForClient!.cache.delete(name)   [因为 skills 来自 resources]
  ├── fetchCommandsForClient.cache.delete(name)
  └── updateServer({ resources, commands: [...mcpPrompts, ...mcpSkills] })
```

## 3. 源码锚点逐层解析

### 3.1 技能命令过滤 — `packages/ccb/src/commands.ts`

#### `getMcpSkillCommands()` — `#L549-561`

```ts
// src/commands.ts
export function getMcpSkillCommands(
  mcpCommands: readonly Command[],
): readonly Command[] {
  if (feature('MCP_SKILLS')) {
    return mcpCommands.filter(
      cmd =>
        cmd.type === 'prompt' &&
        cmd.loadedFrom === 'mcp' &&
        !cmd.disableModelInvocation,
    )
  }
  return []
}
```

该函数是 AppState.mcp.commands 到“可供模型调用的 MCP 技能集合”的最后一道闸门。所有通过 `fetchMcpSkillsForClient` 发现的技能最终都必须满足 `prompt` 类型、`loadedFrom === 'mcp'` 且 `disableModelInvocation === false`。

#### `getSkillToolCommands()` / `getSlashCommandToolSkills()` — `#L565-610`

`getSkillToolCommands` 返回本地及 MCP 所有可展示给模型的 prompt 命令。在 `SkillTool.ts` 中，通过 `getAllCommands` 显式把 `AppState.mcp.commands.filter(cmd => cmd.type === 'prompt' && cmd.loadedFrom === 'mcp')` 与 `getCommands(getProjectRoot())` 做 `uniqBy` 合并（见 `#L81-94`）。这意味着 MCP skills 不是通过 `getCommands()` 路径混入，而是在 SkillTool 调用侧被显式拼接。

### 3.2 MCP Client 连接层 — `packages/ccb/src/services/mcp/client.ts`

#### 条件 require — `#L117-121`

```ts
const fetchMcpSkillsForClient = feature('MCP_SKILLS')
  ? (
      require('../../skills/mcpSkills.js') as typeof import('../../skills/mcpSkills.js')
    ).fetchMcpSkillsForClient
  : null
```

当 feature 关闭时，模块不会被加载，运行时无额外开销。

#### 连接建立时获取 skills — `#L2177-2190`

```ts
const supportsResources = !!client.capabilities?.resources

const [tools, mcpCommands, mcpSkills, resources] = await Promise.all([
  fetchToolsForClient(client),
  fetchCommandsForClient(client),
  feature('MCP_SKILLS') && supportsResources
    ? fetchMcpSkillsForClient!(client)
    : Promise.resolve([]),
  supportsResources
    ? fetchResourcesForClient(client)
    : Promise.resolve([]),
])
const commands = [...mcpCommands, ...mcpSkills]
```

服务器必须声明 `resources` 能力，才会触发 skills 获取。获取结果与 `mcpCommands` 合并写入连接上下文。

#### 重连/断连时缓存清除 — `#L1390-1396` / `#L1668-1673`

```ts
fetchToolsForClient.cache.delete(name)
fetchResourcesForClient.cache.delete(name)
fetchCommandsForClient.cache.delete(name)
if (feature('MCP_SKILLS')) {
  fetchMcpSkillsForClient!.cache.delete(name)
}
```

两处均执行：一处为连接 `onClose` 回调中的清理，一处为 `disconnectMcpServer(name)`。保证重新连接时不会读到旧会话的 stale 数据。

#### 刷新 prompts 时并行获取 skills — `#L2351-2358`

与连接建立时逻辑相同，在 reconnect pipeline 中再次并联获取 skills，确保重连后 commands 列表完整。

### 3.3 实时刷新 Hook — `packages/ccb/src/services/mcp/useManageMCPConnections.ts`

#### 条件 require — `#L22-26`

```ts
const fetchMcpSkillsForClient = feature('MCP_SKILLS')
  ? (
      require('../../skills/mcpSkills.js') as typeof import('../../skills/mcpSkills.js')
    ).fetchMcpSkillsForClient
  : null
```

与 `client.ts` 中的写法一致，保证 feature 未开启时代码零加载。

#### `prompts/list_changed` 处理 — `#L682-694`

```ts
const [mcpPrompts, mcpSkills] = await Promise.all([
  fetchCommandsForClient(client),
  feature('MCP_SKILLS')
    ? fetchMcpSkillsForClient!(client)
    : Promise.resolve([]),
])
updateServer({
  ...client,
  commands: [...mcpPrompts, ...mcpSkills],
})
clearSkillIndexCache?.()
```

注释明确说明：skills 来自 resources，但 prompts 变化时也会顺手刷新 skills（skills 自身有 `.cache`，不会重复请求）。最后调用 `clearSkillIndexCache` 使 `EXPERIMENTAL_SKILL_SEARCH` 的索引在下一次搜索时重建。

#### `resources/list_changed` 处理 — `#L718-738`

```ts
if (feature('MCP_SKILLS')) {
  fetchMcpSkillsForClient!.cache.delete(client.name)
  fetchCommandsForClient.cache.delete(client.name)
  const [newResources, mcpPrompts, mcpSkills] = await Promise.all([
    fetchResourcesForClient(client),
    fetchCommandsForClient(client),
    fetchMcpSkillsForClient!(client),
  ])
  updateServer({
    ...client,
    resources: newResources,
    commands: [...mcpPrompts, ...mcpSkills],
  })
  clearSkillIndexCache?.()
}
```

resources 变化时skills必须强制刷新（因为 skill:// 资源可能增删）。同时刷新 prompts cache 以避免并发 `prompts/list_changed` 通知导致的数据覆盖问题。

### 3.4 Stub 实现 — `packages/ccb/src/skills/mcpSkills.ts`

```ts
// src/skills/mcpSkills.ts
export const fetchMcpSkillsForClient: ((...args: unknown[]) => Promise<Command[]>) & { cache: Map<string, unknown> } = Object.assign(
  (..._args: unknown[]) => Promise.resolve([] as Command[]),
  { cache: new Map<string, unknown>() }
)
```

当前该文件为 auto-generated stub。函数签名需要满足：
1. 接收一个 MCP client 参数；
2. 返回 `Promise<Command[]>`；
3. 挂载一个 `.cache: Map<string, unknown>` 属性供外部清理。

> **实现风险**：`fetchMcpSkillsForClient` 仅返回空数组，意味着 `skill://` URI 到 `Command` 的完整转换链路尚未实现。开启 `FEATURE_MCP_SKILLS=1` 后，模型不会从 MCP 服务器发现任何技能，因为所有 plumbing 都通但核心转换器为空。实现者需要补充：resource 遍历过滤、`resource.read()` 获取 SKILL.md 内容、frontmatter 解析、`createSkillCommand` 调用并设置 `loadedFrom === 'mcp'`。

### 3.5 循环依赖规避 — `packages/ccb/src/skills/mcpSkillBuilders.ts`

```ts
// src/skills/mcpSkillBuilders.ts
export type MCPSkillBuilders = {
  createSkillCommand: typeof createSkillCommand
  parseSkillFrontmatterFields: typeof parseSkillFrontmatterFields
}

let builders: MCPSkillBuilders | null = null

export function registerMCPSkillBuilders(b: MCPSkillBuilders): void {
  builders = b
}

export function getMCPSkillBuilders(): MCPSkillBuilders {
  if (!builders) {
    throw new Error('MCP skill builders not registered — loadSkillsDir.ts has not been evaluated yet')
  }
  return builders
}
```

由于依赖关系为 `client.ts → mcpSkills.ts → loadSkillsDir.ts → … → client.ts`，直接引入会形成循环依赖。该模块作为依赖图叶节点，仅导类型，运行时由 `loadSkillsDir.ts` 在模块初始化时注册实现（`#L1083-1086`）：

```ts
// src/skills/loadSkillsDir.ts
registerMCPSkillBuilders({
  createSkillCommand,
  parseSkillFrontmatterFields,
})
```

注释详细说明了为什么不能使用动态 import(variable)（Bun bundling 后会在 `/$bunfs/root/…` 路径下解析失败），以及为什么不能用字面量动态 import（`dependency-cruiser` 会追踪并导致大量 cycle violation）。

### 3.6 技能构建原语 — `packages/ccb/src/skills/loadSkillsDir.ts`

#### `parseSkillFrontmatterFields()` — `#L183-259`

```ts
export function parseSkillFrontmatterFields(
  frontmatter: FrontmatterData,
  markdownContent: string,
  resolvedName: string,
  descriptionFallbackLabel: 'Skill' | 'Custom command' = 'Skill',
): {
  displayName: string | undefined
  description: string
  hasUserSpecifiedDescription: boolean
  allowedTools: string[]
  // ...
}
```

该函数是 MCP skill 和本地 file-based skill 共享的 frontmatter 解析入口。它将 `description`、`allowed-tools`、`when_to_use`、`disable-model-invocation` 等 frontmatter 字段统一归一化为 `Command` 所需的结构化字段。

#### `createSkillCommand()` — `#L270-404`

```ts
export function createSkillCommand({ skillName, displayName, description, ... }): Command {
  return {
    type: 'prompt',
    name: skillName,
    description,
    // ...
    loadedFrom,
    async getPromptForCommand(args, toolUseContext) {
      // ...
      if (loadedFrom !== 'mcp') {
        finalContent = await executeShellCommandsInPrompt(...)
      }
      return [{ type: 'text', text: finalContent }]
    },
  } satisfies Command
}
```

注意 `loadedFrom !== 'mcp'` 的安全隔离：MCP skills 是远程/不可信的，因此不能执行 body 中的 inline shell 命令（`!\`…\`` / ```! … ```）。这是未来的 stub 实现者必须继承的行为约束。

## 4. 使用方式

```bash
# 启用 MCP_SKILLS
FEATURE_MCP_SKILLS=1 bun run dev
```

前置条件：
1. 配置了支持 `skill://` URI 资源的 MCP 服务器；
2. 该服务器在 `capabilities` 中声明了 `resources`。

## 5. 待补全实现

| 文件 | 当前状态 | 需要实现 |
|------|---------|----------|
| `packages/ccb/src/skills/mcpSkills.ts` | Stub | `fetchMcpSkillsForClient(client)`：从 MCP resources 列表筛选 `skill://` URI，读取内容，调用 `getMCPSkillBuilders()` 构建 `Command[]` |
| `packages/ccb/src/skills/mcpSkillBuilders.ts` | 框架完整 | 无需修改，已提供 `createSkillCommand` / `parseSkillFrontmatterFields` 的运行时注册机制 |

## 6. 关键设计决策总结

1. **Feature gate 隔离**：所有 `fetchMcpSkillsForClient` 的调用点均包裹 `feature('MCP_SKILLS')`，且通过条件 `require()` 加载模块。关闭 flag 时零运行时开销。
2. **能力检查前置**：skills 获取不仅依赖 feature flag，还依赖 `client.capabilities?.resources`。无 resources 能力的 MCP 服务器不会触发 skills 拉取。
3. **缓存一致性**：`.cache` Map 按 server name 索引，在断连、重连、`resources/list_changed`、`prompts/list_changed` 时清理，保证 AppState 与现实一致。
4. **循环依赖规避**：`mcpSkillBuilders.ts` 作为 write-once registry，避免 `client.ts ↔ mcpSkills.ts ↔ loadSkillsDir.ts` 循环。
5. **安全隔离**：`createSkillCommand` 在 `loadedFrom === 'mcp'` 时跳过 `executeShellCommandsInPrompt`，防止远程 skill 执行本地 shell。

## 7. 文件索引

| 文件 | 职责 | 关键行号 |
|------|------|----------|
| `packages/ccb/src/commands.ts` | 技能命令过滤 | `#L549-561`, `#L565-610` |
| `packages/ccb/src/tools/SkillTool/SkillTool.ts` | SkillTool 侧合并本地 + MCP 技能 | `#L81-94` |
| `packages/ccb/src/services/mcp/client.ts` | 连接建立、重连、缓存清除、skills 获取 | `#L117-121`, `#L1390-1396`, `#L1668-1673`, `#L2177-2190`, `#L2351-2358` |
| `packages/ccb/src/services/mcp/useManageMCPConnections.ts` | 实时刷新（list_changed 通知） | `#L22-26`, `#L682-694`, `#L718-738` |
| `packages/ccb/src/skills/mcpSkills.ts` | 核心转换逻辑（stub） | `#L1-7` |
| `packages/ccb/src/skills/mcpSkillBuilders.ts` | 循环依赖规避与技能构建器注册 | `#L1-44` |
| `packages/ccb/src/skills/loadSkillsDir.ts` | `createSkillCommand` / `parseSkillFrontmatterFields` 定义与注册 | `#L183-259`, `#L270-404`, `#L1083-1086` |

---

