# GrowthBook A/B 测试体系 - 运行时功能发布

> 本报告基于 [Claude Code Architecture 中文文档](https://ccb.agent-aura.top/docs/internals/growthbook-ab-testing) 的目录结构进一步深化，并映射到 `packages/ccb`（Claude Code 的 TypeScript 实现）的源码实现。
>
> **源码映射说明**：GrowthBook A/B 测试体系全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。
> 文末附 [源码索引](#源码索引)。

---

## 为什么需要运行时 A/B 测试

构建时 `feature()` 是"全有或全无"的——要么所有用户都有，要么所有用户都没有。但产品团队需要更精细的控制：

- 只对 5% 的用户灰度发布新功能
- 按订阅类型（Free / Pro / Team）差异化体验
- 对特定组织静默开启实验性能力
- 随时远程关闭出问题的功能，无需发版

这就是 GrowthBook 的用武之地——一个**运行时的、基于用户属性的功能门控和 A/B 测试系统**。

---

## 集成架构

GrowthBook 的完整实现位于 `packages/ccb/src/services/analytics/growthbook.ts`（约 1256 行），工作流程如下：

### 1. 启动时获取远程配置

CLI 启动时，GrowthBook SDK 通过 Anthropic API 端点获取当前的功能配置和实验分组规则。使用 `remoteEval: true` 模式——**在服务端计算分组，客户端只拿结果**。这种方式既保护了实验算法的商业机密，又减少了客户端的计算负担。

参见 `growthbook.ts#L601-L612`：

```typescript
const thisClient = new GrowthBook({
  apiHost: baseUrl,
  clientKey,
  attributes,
  remoteEval: !isAdapterMode,
  ...(!isAdapterMode
    ? { cacheKeyAttributes: ['id', 'organizationUUID'] }
    : {}),
  ...(authHeaders.error
    ? {}
    : { apiHostRequestHeaders: authHeaders.headers }),
})
```

### 2. 计算用户属性

SDK 收集当前用户的属性（设备 ID、订阅类型、组织 UUID 等），用于决定该用户属于哪些实验的哪个分组。详见 [用户定向属性](#用户定向属性) 一节。

### 3. 缓存到本地

计算结果缓存到 `~/.claude.json` 的 `cachedGrowthBookFeatures` 字段。刷新间隔：Anthropic 员工 20 分钟，外部用户 6 小时。

参见 `growthbook.ts#L718-L737`：

```typescript
function syncRemoteEvalToDisk(): void {
  const fresh = Object.fromEntries(remoteEvalFeatureValues)
  const config = getGlobalConfig()
  if (isEqual(config.cachedGrowthBookFeatures, fresh)) {
    return
  }
  saveGlobalConfig(current => ({
    ...current,
    cachedGrowthBookFeatures: fresh,
  }))
}
```

### 4. 代码中查询 flag

业务代码通过 `tengu_*` 前缀的 flag 名查询功能状态，GrowthBook SDK 返回当前用户的分组值。

---

## 用户定向属性

GrowthBook 根据以下用户属性决定实验分组，定义在 `growthbook.ts#L36-L55`：

```typescript
export type GrowthBookUserAttributes = {
  id: string
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

| 属性 | 类型 | 来源 | 用途 |
|------|------|------|------|
| `id` | string | 会话 ID | 按会话粒度分组 |
| `deviceID` | string | 持久化设备标识 | 跨会话一致性 |
| `sessionId` | string | 当前会话 ID | 会话级实验 |
| `platform` | enum | `process.platform` | 按操作系统差异化 |
| `organizationUUID` | string | API 认证信息 | 按组织灰度 |
| `accountUUID` | string | API 认证信息 | 按个人账户灰度 |
| `subscriptionType` | string | API 认证信息 | Free / Pro / Team 差异化 |
| `rateLimitTier` | string | API 认证信息 | 按速率限制层级 |
| `email` | string | API 认证信息 | 精确定向特定用户 |
| `appVersion` | string | `MACRO.VERSION` | 按版本号灰度 |
| `github` | object | GitHub Actions 元数据 | CI 环境特殊处理 |

属性组装逻辑在 `growthbook.ts#L523-L560` 的 `getUserAttributes()` 函数中：通过 `getUserForGrowthBook()` 获取基础用户信息，对员工额外尝试从 OAuth 配置读取 email，并附加 `apiBaseUrlHost` 以支持企业代理部署定向。

---

## 代号文化：tengu_* 的世界

所有运行时 flag 都以 `tengu_` 为前缀——"Tengu"（天狗）是 Claude Code 的内部项目代号。flag 名采用**动物/植物/矿物 + 形容词**的命名约定，刻意保持不透明。

### 典型运行时 flag 示例

全部定义在 `growthbook.ts#L434-L479` 的 `LOCAL_GATE_DEFAULTS` 常量中（本地兜底默认值）：

| Flag | 语义 | 本地默认值 |
|------|------|------------|
| `tengu_kairos` | Kairos 助手模式 | 见服务端配置 |
| `tengu_amber_stoat` | Explore / Plan Agent 开关 | `true` |
| `tengu_auto_background_agents` | 后台 Agent 自动化 | `true` |
| `tengu_onyx_plover` | Auto-Dream 后台记忆 | `{ enabled: true }` |
| `tengu_glacier_2xr` | 工具搜索行为变体 | 见服务端配置 |
| `tengu_birch_trellis` | Tree-sitter bash 安全分析 | `true` |
| `tengu_scratch` | 草稿本功能 | 见服务端配置 |
| `tengu_quartz_lantern` | Diff 计算策略 | 见服务端配置 |

### 双重门控与 A/B 测试示例

以 `tengu_amber_stoat`（Explore Agent）为例，构建时 `feature('BUILTIN_EXPLORE_PLAN_AGENTS')` 是第一道门，运行时 `tengu_amber_stoat` 是第二道门。参见 `builtInAgents.ts#L13-L20`：

```typescript
export function areExplorePlanAgentsEnabled(): boolean {
  if (feature('BUILTIN_EXPLORE_PLAN_AGENTS')) {
    return getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_stoat', true)
  }
  return false
}
```

`getBuiltInAgents()`（`builtInAgents.ts#L22-L72`）在构建时 flag 启用后，再查询 GrowthBook 的运行时 flag：若命中则把 `EXPLORE_AGENT` 和 `PLAN_AGENT` 注入可用 agent 列表。这种"构建时 + 运行时"双重门控确保新功能可以分阶段发布，也允许在不需要重新发版的情况下灰度或回滚。

`EXPLORE_AGENT` 的定义位于 `exploreAgent.ts#L64-L83`：

```typescript
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: EXPLORE_WHEN_TO_USE,
  disallowedTools: [
    AGENT_TOOL_NAME,
    EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  source: 'built-in',
  baseDir: 'built-in',
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

"amber stoat"（琥珀色白鼬）是随机生成的代号，与功能内容无关——这是为了防止通过 flag 名猜测功能。

---

## 取值 API 与缓存策略

`growthbook.ts` 对外暴露了几种不同语义特征的 flag 查询函数，以适配不同的性能/准确性需求：

### `getFeatureValue_CACHED_MAY_BE_STALE<T>`

**非阻塞**的纯读取函数，优先顺序：`env 覆盖` → `config 覆盖` → `内存缓存` → `磁盘缓存` → `LOCAL_GATE_DEFAULTS` → `调用方默认值`。参见 `growthbook.ts#L767-L809`。

这是**启动关键路径和同步上下文**的首选方法。内存中的 `remoteEvalFeatureValues` 在 `processRemoteEvalPayload` 成功后立即填充，因此热路径无需解析磁盘 JSON。

### `getFeatureValue_DEPRECATED<T>` / `getDynamicConfig_BLOCKS_ON_INIT<T>`

**阻塞**到 GrowthBook 初始化完成的函数。内部调用 `getFeatureValueInternal`，会先 `await initializeGrowthBook()`（最长等待 5 秒）。参见 `growthbook.ts#L740-L765`。

### `checkGate_CACHED_OR_BLOCKING`

**混合语义**：若磁盘缓存已经返回 `true`，则立即信任；若磁盘缓存为 `false` 或缺失，则走阻塞路径重新拉取。适合用户主动触发的功能（如 `/remote-control`），避免"缓存说没有就真没有"的不公平拒绝。参见 `growthbook.ts#L1001-L1028`。

### `checkSecurityRestrictionGate`

专门用于安全限制类 gate。若 GrowthBook 正在 re-initialization（如登录后），会**等待重初始化完成**再走缓存读取，确保拿到最新的用户身份对应的值。参见 `growthbook.ts#L965-L999`。

### `checkStatsigFeatureGate_CACHED_MAY_BE_STALE`

**迁移兼容层**：GrowthBook 正在从 Statsig 迁移而来，该函数先查 GrowthBook 缓存，再回退到 `cachedStatsigGates`。新代码不应使用。参见 `growthbook.ts#L812-L863`。

---

## 覆盖机制：Env + Config 两级覆盖

### 环境变量覆盖

```bash
# 仅在 USER_TYPE=ant 的构建中生效
CLAUDE_INTERNAL_FC_OVERRIDES='{"tengu_kairos": true}' claude
```

覆盖逻辑在 `growthbook.ts#L176-L193` 的 `getEnvOverrides()` 中：解析 `CLAUDE_INTERNAL_FC_OVERRIDES` JSON，按 feature key 直接覆盖远程求值和磁盘缓存的结果。所有取值 API 都会先检查 env override，因此评估套件可以保持确定性。

### Config 界面覆盖

在内部构建中，`/config` 命令的 Gates 标签页通过 `getAllGrowthBookFeatures()`（`growthbook.ts#L228-L234`）读取所有已知 feature，并允许用户通过 `setGrowthBookConfigOverride()` / `clearGrowthBookConfigOverrides()`（`growthbook.ts#L241-L280`）实时切换任意 GrowthBook flag。Config 覆盖的优先级仅次于 env 覆盖。

---

## 初始化与刷新机制

### 客户端初始化

`getGrowthBookClient`（`growthbook.ts#L568-L713`）被 `memoize` 包装，确保同进程内只创建一个 SDK 实例。创建后会调用 `thisClient.init({ timeout: 5000 })`，在回调中执行：

1. `processRemoteEvalPayload(thisClient)` — 转换 API 返回格式并写入内存缓存
2. `syncRemoteEvalToDisk()` — 将完整 feature map 写入 `~/.claude.json`
3. `refreshed.emit()` — 通知所有订阅者配置已更新
4. `setupPeriodicGrowthBookRefresh()` — 设置定时刷新（外部 6h / ant 20min）

### 轻量刷新

`refreshGrowthBookFeatures()`（`growthbook.ts#L1100-L1143`）调用 SDK 的 `refreshFeatures()` 重新拉取远程配置，并重建内存缓存和磁盘缓存，**不销毁客户端实例**。这是定时刷新使用的路径。

### 认证变更后的重初始化

`refreshGrowthBookAfterAuthChange()`（`growthbook.ts#L1031-L1064`）在登录/登出时被调用。由于 `apiHostRequestHeaders` 无法在客户端创建后更新，它会先 `resetGrowthBook()` 销毁旧实例，再重新 `initializeGrowthBook()`。`reinitializingPromise` 的存在让 `checkSecurityRestrictionGate` 可以等待这一过程完成。

### reset 与清理

`resetGrowthBook()`（`growthbook.ts#L1067-L1094`）用于测试和清理：移除 process 事件监听、销毁 SDK 实例、清空所有 Map/Set、清除 memoize 缓存、重置 env override 解析状态。

---

## 实验追踪

GrowthBook 集成了完整的实验曝光追踪：

- 每次查询 flag 时，若 feature 的 `source === 'experiment'`，在 `processRemoteEvalPayload`（`growthbook.ts#L312-L352`）中会把 `experimentId` + `variationId` 存入 `experimentDataByFeature` Map。
- 首次 `getFeatureValue*` 或 `checkGate*` 访问该 feature 时，调用 `logExposureForFeature()`（`growthbook.ts#L287-L300`），通过 `logGrowthBookExperimentTo1P()` 以 protobuf 格式上报 `GrowthbookExperimentEvent`。
- `variation_id` 语义：0 = 对照组，1+ = 实验组。
- `loggedExposures` 和 `pendingExposures` 两套机制分别实现**会话内去重**和**初始化前暂存**。

### 为什么需要两套曝光机制？

`pendingExposures` 解决"在 GrowthBook 初始化完成前就有代码查询 flag"的竞态问题。这些早期查询会被暂存，待初始化完成后再统一补报。`loggedExposures` 则是一个 Set，确保同一个 feature 在同一次会话中只上报一次曝光，避免重复计数。

---

## 源码索引

| 文件 | 核心内容 |
|------|----------|
| `packages/ccb/src/services/analytics/growthbook.ts` | GrowthBook SDK 封装、初始化、刷新、取值 API、覆盖机制、实验追踪 |
| `packages/ccb/src/tools/AgentTool/builtInAgents.ts` | 运行时 flag 控制内置 agent 列表（`tengu_amber_stoat` 等） |
| `packages/ccb/src/tools/AgentTool/built-in/exploreAgent.ts` | Explore Agent 定义，体现 A/B 测试在子 agent 层面的应用 |
| `packages/ccb/src/services/analytics/firstPartyEventLogger.ts` | 实验曝光事件的上报实现 |
