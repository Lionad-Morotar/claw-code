# GrowthBook 适配器技术报告

**单元**: Unit 31  
**主题**: GrowthBook 适配器 - 自定义 Feature Flag 服务器接入  
**原文**: https://ccb.agent-aura.top/docs/internals/growthbook-adapter  
**生成日期**: 2026-04-09

---

## 1. 概述

GrowthBook 适配器是 Claude Code 中用于连接自定义 GrowthBook 服务器的机制，允许用户通过环境变量指定自己的 GrowthBook 实例，实现远程 feature flag 控制。

### 核心设计 rationale

原始 Claude Code 的 GrowthBook 是硬连线到 Anthropic 内部 API 的（`remoteEval: true`，需要 auth headers）。这对社区用户和自托管场景不友好。适配器的设计目标有三点：

1. **零配置降级**：未设置适配器变量时，所有 feature 读取直接返回代码中的默认值，不产生任何网络请求。这避免了原始代码中"GrowthBook 强依赖 1P 认证"导致的启动阻塞问题。
2. **向后兼容**：130+ 个调用方文件无需修改，统一的 `getFeatureValue_CACHED_MAY_BE_STALE` API 透明处理适配器模式与内部模式。
3. **最小侵入**：仅在 SDK 初始化层做分支（`isAdapterMode`），上层业务代码完全无感知。

### 核心行为

| 配置状态 | 行为 |
|---------|------|
| 已配置 `CLAUDE_GB_ADAPTER_URL` + `CLAUDE_GB_ADAPTER_KEY` | 连接自定义 GrowthBook 实例，拉取并缓存 feature 值 |
| 未配置 | 所有 feature 读取直接返回代码中的默认值 |

---

## 2. 环境变量

### 2.1 必需变量

```bash
CLAUDE_GB_ADAPTER_URL=<GrowthBook API 地址>   # 如 https://gb.example.com/
CLAUDE_GB_ADAPTER_KEY=<SDK Client Key>        # 如 sdk-abc123
```

两个变量同时设置时启用适配器模式，否则完全跳过 GrowthBook。

### 2.2 源码位置

**`packages/ccb/src/constants/keys.ts#L5-L16`** - Client Key 获取逻辑：

```typescript
export function getGrowthBookClientKey(): string {
  // 适配器优先：自定义 GrowthBook 服务器
  const adapterKey = process.env.CLAUDE_GB_ADAPTER_KEY
  if (adapterKey) return adapterKey

  return process.env.USER_TYPE === 'ant'
    ? isEnvTruthy(process.env.ENABLE_GROWTHBOOK_DEV)
      ? 'sdk-yZQvlplybuXjYh6L'
      : 'sdk-xRVcrliHIlrg4og4'
    : 'sdk-zAZezfDKGoZuXXKe'
}
```

**设计意图**：`CLAUDE_GB_ADAPTER_KEY` 优先级最高，覆盖原有的 `ant` / `external` 硬编码 SDK key。这使得即使是内部构建，也可以临时指向自定义 GrowthBook 服务器进行测试。

---

## 3. 适配器模式核心实现

### 3.1 启用检测

**`packages/ccb/src/services/analytics/growthbook.ts#L488-L496`**:

```typescript
function isGrowthBookEnabled(): boolean {
  // 适配器模式：有自定义服务器配置时直接启用
  if (process.env.CLAUDE_GB_ADAPTER_URL && process.env.CLAUDE_GB_ADAPTER_KEY) {
    return true
  }
  // GrowthBook depends on 1P event logging.
  return is1PEventLoggingEnabled()
}
```

### 3.2 GrowthBook 客户端配置

**`packages/ccb/src/services/analytics/growthbook.ts#L560-L694`** - `getGrowthBookClient()` 关键配置：

```typescript
const getGrowthBookClient = memoize(
  (): { client: GrowthBook; initialized: Promise<void> } | null => {
    // ...
    const clientKey = getGrowthBookClientKey()
    const baseUrl =
      process.env.CLAUDE_GB_ADAPTER_URL ||
      (process.env.USER_TYPE === 'ant'
        ? process.env.CLAUDE_CODE_GB_BASE_URL || 'https://api.anthropic.com/'
        : 'https://api.anthropic.com/')
    const isAdapterMode = !!(
      process.env.CLAUDE_GB_ADAPTER_URL && process.env.CLAUDE_GB_ADAPTER_KEY
    )

    // 适配器模式下不需要 auth，GrowthBook Cloud 用 clientKey 即可
    const hasAuth = isAdapterMode || !authHeaders.error
    clientCreatedWithAuth = hasAuth

    const thisClient = new GrowthBook({
      apiHost: baseUrl,
      clientKey,
      attributes,
      // remoteEval only works with Anthropic internal API, GrowthBook Cloud doesn't support it
      remoteEval: !isAdapterMode,
      // cacheKeyAttributes only valid with remoteEval
      ...(!isAdapterMode
        ? { cacheKeyAttributes: ['id', 'organizationUUID'] }
        : {}),
      // Add auth headers if available
      ...(authHeaders.error
        ? {}
        : { apiHostRequestHeaders: authHeaders.headers }),
      // Debug logging for Ants
      ...(process.env.USER_TYPE === 'ant'
        ? {
            log: (msg: string, ctx: Record<string, unknown>) => {
              logForDebugging(`GrowthBook: ${msg} ${jsonStringify(ctx)}`)
            },
          }
        : {}),
    })
    // ...
  }
)
```

### 3.3 适配器模式关键差异

| 特性 | 适配器模式 | Anthropic 内部模式 |
|------|-----------|------------------|
| `remoteEval` | `false` | `true` |
| `cacheKeyAttributes` | 不设置 | `['id', 'organizationUUID']` |
| Auth Headers | 不需要 | 需要 `getAuthHeaders()` |
| Base URL | `CLAUDE_GB_ADAPTER_URL` | `api.anthropic.com` |

**为什么适配器模式下 `remoteEval` 必须关闭？**

`remoteEval` 是 Anthropic 内部 GrowthBook 部署的定制功能：服务端根据用户属性直接计算分组结果，客户端只接收最终结果。标准的 GrowthBook Cloud 和自托管版本不支持这一模式。因此适配器模式下必须关闭 `remoteEval`，让 SDK 走传统的本地求值路径。

---

## 4. Feature Flag 读取优先级链

**`packages/ccb/src/services/analytics/growthbook.ts#L814-L867`** - `getFeatureValue_CACHED_MAY_BE_STALE()`:

```
1. CLAUDE_INTERNAL_FC_OVERRIDES 环境变量（JSON 对象覆盖，仅 ant 构建）
   ↓ 未命中
2. growthBookOverrides 配置（~/.claude.json，仅 ant 构建）
   ↓ 未命中
3. 内存缓存（remoteEvalFeatureValues，本次进程从服务器拉取）
   ↓ 未命中
4. 磁盘缓存（~/.claude.json 的 cachedGrowthBookFeatures）
   ↓ 未命中
5. 代码中的 defaultValue 参数（LOCAL_GATE_DEFAULTS）
```

**`packages/ccb/src/constants/keys.ts#L5-L16`** - 适配器 Key 优先级最高。

这种优先级设计确保了：即使在网络断开或服务器不可用时，已缓存的值仍然可用；如果连缓存都没有，代码中的硬编码默认值能保证功能不崩溃。

---

## 5. 缓存与刷新机制

### 5.1 内存缓存

**`packages/ccb/src/services/analytics/growthbook.ts#L79-L81`**:

```typescript
// Cache for remote eval feature values - workaround for SDK not respecting remoteEval response
const remoteEvalFeatureValues = new Map<string, unknown>()
```

注意：虽然变量名叫 `remoteEvalFeatureValues`，但在适配器模式下它同样被用来缓存从 GrowthBook 服务器拉取的 feature 值。

### 5.2 磁盘缓存

**`packages/ccb/src/services/analytics/growthbook.ts#L407-L417`** - `syncRemoteEvalToDisk()`:

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

### 5.3 周期性刷新

**`packages/ccb/src/services/analytics/growthbook.ts#L1113-L1117`** - 刷新间隔：

```typescript
// Periodic refresh interval (matches Statsig's 6-hour interval)
const GROWTHBOOK_REFRESH_INTERVAL_MS =
  process.env.USER_TYPE !== 'ant'
    ? 6 * 60 * 60 * 1000 // 6 hours
    : 20 * 60 * 1000 // 20 min (for ants)
```

### 5.4 初始化超时

首次连接超时 5 秒，超时后使用磁盘缓存或默认值。

---

## 6. Feature Key 列表

### 6.1 高频使用

| Feature Key | 类型 | 代码默认值 | 用途 |
|-------------|------|-----------|------|
| `tengu_hive_evidence` | boolean | false | 任务证据系统 |
| `tengu_quartz_lantern` | boolean | false | 文件写入/编辑保护 |
| `tengu_auto_background_agents` | boolean | false | 自动后台 Agent |
| `tengu_agent_list_attach` | boolean | false | Agent 列表附件 |
| `tengu_amber_stoat` | boolean | true | 内置 Agents |
| `tengu_slim_subagent_claudemd` | boolean | true | 子 Agent CLAUDE.md |
| `tengu_attribution_header` | boolean | true | API 归因 Header |
| `tengu_cobalt_harbor` | boolean | false | Bridge 模式 |
| `tengu_ccr_bridge` | boolean | false | CCR Bridge |
| `tengu_cicada_nap_ms` | number | 0 | 后台刷新节流（毫秒） |
| `tengu_miraculo_the_bard` | boolean | false | 启动欢迎信息 |

### 6.2 Agent / 工具控制

| Feature Key | 类型 | 代码默认值 | 用途 |
|-------------|------|-----------|------|
| `tengu_surreal_dali` | boolean | false | 远程触发工具 |
| `tengu_glacier_2xr` | boolean | false | 工具搜索增强 |
| `tengu_plum_vx3` | boolean | false | Web Search 使用 Haiku |
| `tengu_destructive_command_warning` | boolean | false | 危险命令警告 |
| `tengu_birch_trellis` | boolean | true | Bash 权限控制 |
| `tengu_harbor_permissions` | boolean | false | Harbor 权限模式 |

### 6.3 Bridge / 远程连接

| Feature Key | 类型 | 代码默认值 | 用途 |
|-------------|------|-----------|------|
| `tengu_bridge_repl_v2` | boolean | false | Bridge REPL v2 |
| `tengu_copper_bridge` | boolean | false | Copper Bridge |
| `tengu_ccr_mirror` | boolean | false | CCR Mirror |

### 6.4 配置对象 (动态配置)

| Feature Key | 类型 | 代码默认值 | 用途 |
|-------------|------|-----------|------|
| `tengu_file_read_limits` | object | null | 文件读取限制配置 |
| `tengu_cobalt_raccoon` | object | null | Cobalt 配置 |
| `tengu_cobalt_lantern` | object | null | Lantern 配置 |
| `tengu_desktop_upsell` | object | null | 桌面版引导 |
| `tengu_marble_sandcastle` | object | null | Marble 配置 |
| `tengu_marble_fox` | object | null | Marble Fox 配置 |
| `tengu_ultraplan_model` | string | null | Ultraplan 模型名 |

### 6.5 Gate (布尔门控)

| Gate Key | 代码默认值 | 用途 |
|----------|-----------|------|
| `tengu_chair_sermon` | false | 功能门控 |
| `tengu_scratch` | false | Scratch 功能 |
| `tengu_thinkback` | false | Thinkback 功能 |
| `tengu_tool_pear` | false | Tool Pear 功能 |

---

## 7. 本地默认配置 (LOCAL_GATE_DEFAULTS)

**`packages/ccb/src/services/analytics/growthbook.ts#L434-L472`** 定义了 40+ 个本地 gate 默认值，分为三类：

### P0 - 纯本地功能 (无外部依赖)
- `tengu_keybinding_customization_release`: true
- `tengu_streaming_tool_execution2`: true
- `tengu_kairos_cron`: true
- `tengu_amber_json_tools`: true
- `tengu_immediate_model_command`: true
- `tengu_basalt_3kr`: true
- `tengu_pebble_leaf_prune`: true
- `tengu_chair_sermon`: true
- `tengu_lodestone_enabled`: true
- `tengu_auto_background_agents`: true
- `tengu_fgts`: true

### P1 - 需要 Claude API
- `tengu_session_memory`: true
- `tengu_passport_quail`: true
- `tengu_moth_copse`: true
- `tengu_coral_fern`: true
- `tengu_chomp_inflection`: true
- `tengu_hive_evidence`: true
- `tengu_kairos_brief`: true
- `tengu_kairos_brief_config`: { enable_slash_command: true }
- `tengu_sedge_lantern`: true
- `tengu_onyx_plover`: { enabled: true }
- `tengu_willow_mode`: 'dialog'

### KS - 杀死开关 (默认 true)
- `tengu_turtle_carbon`: true
- `tengu_amber_stoat`: true
- `tengu_amber_flint`: true
- `tengu_slim_subagent_claudemd`: true
- `tengu_birch_trellis`: true
- `tengu_collage_kaleidoscope`: true
- `tengu_compact_cache_prefix`: true
- `tengu_kairos_cron_durable`: true
- `tengu_attribution_header`: true
- `tengu_slate_prism`: true

---

## 8. 实现变更摘要

适配器机制共修改了 3 个关键位置：

1. **`packages/ccb/src/constants/keys.ts#L5-L16`** - `getGrowthBookClientKey()` 优先读取 `CLAUDE_GB_ADAPTER_KEY`
2. **`packages/ccb/src/services/analytics/growthbook.ts#L488-L496`** - `isGrowthBookEnabled()` 适配器模式下直接启用
3. **`packages/ccb/src/services/analytics/growthbook.ts#L568-L694`** - `getGrowthBookClient()` 中 base URL 优先使用 `CLAUDE_GB_ADAPTER_URL`，并关闭 `remoteEval`

所有 130+ 个调用方文件无需修改。

---

## 9. 使用方式

### 9.1 启用适配器

```bash
CLAUDE_GB_ADAPTER_URL=https://gb.example.com/ \
CLAUDE_GB_ADAPTER_KEY=sdk-abc123 \
bun run dev
```

### 9.2 不使用 GrowthBook (默认行为)

```bash
bun run dev
# 所有 getFeatureValue_CACHED_MAY_BE_STALE("xxx", defaultValue) 直接返回 defaultValue
```

### 9.3 GrowthBook 服务端配置步骤

1. 部署 GrowthBook 服务端（Docker 自托管或 Cloud 版）
2. 创建 Environment（如 `production`）
3. 创建 SDK Connection，获得 SDK Key（即 `CLAUDE_GB_ADAPTER_KEY`）
4. 按需添加 Feature，key 和类型见上方列表

### 9.4 核心原则

- 不配置任何 feature 也能正常运行——代码中每个调用都提供了默认值
- 只创建你想远程控制的 feature，其余走代码默认
- GrowthBook 上配了某个 feature 后，其值会覆盖代码中的默认值

---

## 10. 相关文件索引

| 文件路径 | 说明 | 关键行 |
|---------|------|--------|
| `packages/ccb/src/constants/keys.ts` | Client Key 获取 | L5-L16 |
| `packages/ccb/src/services/analytics/growthbook.ts` | 主实现文件 | 全文 ~1256 行 |
| `packages/ccb/src/types/generated/events_mono/growthbook/v1/growthbook_experiment_event.ts` | 实验事件类型定义 | - |
| `packages/ccb/scripts/verify-gates.ts` | Gate 验证脚本 | - |

---

## 11. 调用方统计

通过 `grep` 统计，`getFeatureValue_CACHED_MAY_BE_STALE` 的调用方超过 60 个文件，分布在：
- `src/hooks/` - React hooks
- `src/services/` - 各种服务层
- `src/components/` - UI 组件
- `src/commands/` - CLI 命令
- `src/bridge/` - Bridge 模式相关

---

**报告完成**。本文档基于原始页面内容和源代码分析生成，所有源码锚点均指向 `packages/ccb/` 子项目中的实际文件。
