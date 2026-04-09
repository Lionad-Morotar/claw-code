# Buddy 宠物系统

> **源码映射说明**：Buddy 是 Claude Code 上游（TypeScript 版）的趣味功能。`claw-code`（Rust 重写版）目前尚未实现该功能，本报告引用的 `packages/ccb/src/buddy/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。

Buddy 是 CLI 中的虚拟宠物伴侣，通过 /buddy 命令孵化、互动，会出现在输入框旁边陪伴你写代码。

**Feature Flag**: `FEATURE_BUDDY=1`

---

## 概述

Buddy 是 Claude Code 内置的虚拟宠物系统。在 REPL 中通过 `/buddy` 命令可以孵化一只随机生成的宠物伴侣，它会出现在输入框旁边，陪伴你的编码过程。

### 设计意图：为什么 CLI 需要虚拟宠物？

在长时间的编码会话中，开发者容易感到孤独和疲劳。Buddy 的设计目标是通过轻量级的情感反馈提升用户体验：

1. **降低焦虑感**：等待编译或测试时，宠物会执行空闲动画，分散注意力。
2. **增强归属感**：宠物的确定性生成与用户 ID 绑定，形成一种"这是属于我的"个性化体验。
3. **彩蛋式惊喜**：稀有度、闪光变体和随机属性增加了探索乐趣，同时不影响核心功能。

### 源码映射

Buddy 系统的核心实现在 `packages/ccb/src/buddy/` 目录下，共 8 个文件：

| 文件 | 说明 |
|------|------|
| [`companion.ts`](/packages/ccb/src/buddy/companion.ts#L1-L136) | 宠物生成与加载逻辑 |
| [`types.ts`](/packages/ccb/src/buddy/types.ts#L1-L149) | 类型定义（物种、稀有度、属性） |
| [`sprites.ts`](/packages/ccb/src/buddy/sprites.ts#L1-L514) | 终端像素画渲染 |
| [`CompanionSprite.tsx`](/packages/ccb/src/buddy/CompanionSprite.tsx#L1-L347) | React 组件（输入框旁显示） |
| [`CompanionCard.tsx`](/packages/ccb/src/buddy/CompanionCard.tsx#L1-L110) | `/buddy` 命令显示卡片 |
| [`useBuddyNotification.tsx`](/packages/ccb/src/buddy/useBuddyNotification.tsx#L1-L67) | 启动提示通知 |
| [`companionReact.ts`](/packages/ccb/src/buddy/companionReact.ts#L1-L160) | 宠物反应触发 |
| [`prompt.ts`](/packages/ccb/src/buddy/prompt.ts#L1-L36) | 宠物相关 prompt 模板 |

---

## 启用方式

```bash
FEATURE_BUDDY=1 bun run dev
```

孵化窗口：2026 年 4 月 1-7 日期间启动时，会在 REPL 顶部显示彩虹色的 `/buddy` 提示。4 月 7 日之后命令仍然可用，但不再自动提示。

### 源码映射

Feature Flag 通过 `import { feature } from 'bun:bundle'` 导入，在所有 Buddy 相关模块中使用：

```typescript
// packages/ccb/src/buddy/CompanionSprite.tsx#L1
import { feature } from 'bun:bundle'
```

**门控检查点**：

- [`CompanionSprite.tsx#L122`](/packages/ccb/src/buddy/CompanionSprite.tsx#L122) - `companionReservedColumns()` 函数入口检查
- [`CompanionSprite.tsx#L175`](/packages/ccb/src/buddy/CompanionSprite.tsx#L175) - `CompanionSprite` 组件渲染检查
- [`CompanionSprite.tsx#L335`](/packages/ccb/src/buddy/CompanionSprite.tsx#L335) - `CompanionFloatingBubble` 组件检查
- [`useBuddyNotification.tsx#L43`](/packages/ccb/src/buddy/useBuddyNotification.tsx#L43) - 通知 hook 检查
- [`useBuddyNotification.tsx#L59`](/packages/ccb/src/buddy/useBuddyNotification.tsx#L59) - `/buddy` 触发器检测检查
- [`prompt.ts#L18`](/packages/ccb/src/buddy/prompt.ts#L18) - Prompt 附件生成检查

**启动提示窗口逻辑**（[`useBuddyNotification.tsx#L11-L23`](/packages/ccb/src/buddy/useBuddyNotification.tsx#L11-L23)）：

```typescript
// 4 月 1-7 日为 teaser window，仅当用户尚无宠物时显示彩虹色 /buddy 提示
export function isBuddyTeaserWindow(): boolean {
  if (process.env.USER_TYPE === 'ant') return true
  const d = new Date()
  return d.getFullYear() === 2026 && d.getMonth() === 3 && d.getDate() <= 7
}
```

---

## 命令

| 命令 | 说明 |
|------|------|
| `/buddy` | 查看当前宠物信息；若尚未拥有宠物则自动孵化 |
| `/buddy pet` | 撸宠物，触发爱心动画 |
| `/buddy off` | 静音宠物（隐藏） |
| `/buddy on` | 取消静音 |

> 说明：源码中实际子命令为 `off` / `on` / `pet`，以及无参数时的默认展示/孵化逻辑，并未提供独立的 `rehatch` 子命令。若需要替换宠物，目前的实现路径是通过编辑或删除 `~/.claude.json` 中的 `companion` 字段后再次运行 `/buddy`。

### 源码映射

命令处理实现在 [`commands/buddy/buddy.ts`](/packages/ccb/src/commands/buddy/buddy.ts#L1-L169)：

**`/buddy off` — 静音宠物**（[`buddy.ts#L79-L83`](/packages/ccb/src/commands/buddy/buddy.ts#L79-L83)）：
```typescript
if (sub === 'off') {
  saveGlobalConfig(cfg => ({ ...cfg, companionMuted: true }))
  onDone('companion muted', { display: 'system' })
  return null
}
```

**`/buddy on` — 取消静音**（[`buddy.ts#L86-L90`](/packages/ccb/src/commands/buddy/buddy.ts#L86-L90)）：
```typescript
if (sub === 'on') {
  saveGlobalConfig(cfg => ({ ...cfg, companionMuted: false }))
  onDone('companion unmuted', { display: 'system' })
  return null
}
```

**`/buddy pet` — 触发爱心动画**（[`buddy.ts#L93-L115`](/packages/ccb/src/commands/buddy/buddy.ts#L93-L115)）：
```typescript
if (sub === 'pet') {
  const companion = getCompanion()
  if (!companion) {
    onDone('no companion yet · run /buddy first', { display: 'system' })
    return null
  }

  // Auto-unmute on pet + trigger heart animation
  saveGlobalConfig(cfg => ({ ...cfg, companionMuted: false }))
  setState?.(prev => ({ ...prev, companionPetAt: Date.now() }))

  // Trigger a post-pet reaction
  triggerCompanionReaction(context.messages ?? [], reaction =>
    setState?.(prev =>
      prev.companionReaction === reaction
        ? prev
        : { ...prev, companionReaction: reaction },
    ),
  )

  onDone(`petted ${companion.name}`, { display: 'system' })
  return null
}
```

**`/buddy`（无参数）— 显示或孵化**（[`buddy.ts#L117-L169`](/packages/ccb/src/commands/buddy/buddy.ts#L117-L169)）：
- 若已有宠物，返回 [`CompanionCard`](/packages/ccb/src/buddy/CompanionCard.tsx#L25) JSX 组件
- 若无宠物，自动生成种子并孵化

---

## 宠物属性

### 物种（18 种）

| | | | |
|---|---|---|---|
| Duck | Goose | Blob | Cat |
| Dragon | Octopus | Owl | Penguin |
| Turtle | Snail | Ghost | Axolotl |
| Capybara | Cactus | Robot | Rabbit |
| Mushroom | Chonk | | |

### 源码映射

物种定义在 [`types.ts#L17-L73`](/packages/ccb/src/buddy/types.ts#L17-L73)，使用 `String.fromCharCode` 构造字符串以避免静态分析工具误报：

```typescript
// 通过 charCode 构造字符串，避免静态分析工具误报
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
export const goose = c(0x67, 0x6f, 0x6f, 0x73, 0x65) as 'goose'
export const blob = c(0x62, 0x6c, 0x6f, 0x62) as 'blob'
export const cat = c(0x63, 0x61, 0x74) as 'cat'
// ... 共 18 个物种
```

**物种名称映射**（[`buddy.ts#L19-L38`](/packages/ccb/src/commands/buddy/buddy.ts#L19-L38)）：
```typescript
const SPECIES_NAMES: Record<string, string> = {
  duck: 'Waddles',
  goose: 'Goosberry',
  blob: 'Gooey',
  cat: 'Whiskers',
  dragon: 'Ember',
  octopus: 'Inky',
  owl: 'Hoots',
  penguin: 'Waddleford',
  turtle: 'Shelly',
  snail: 'Trailblazer',
  ghost: 'Casper',
  axolotl: 'Axie',
  capybara: 'Chill',
  cactus: 'Spike',
  robot: 'Byte',
  rabbit: 'Flops',
  mushroom: 'Spore',
  chonk: 'Chonk',
}
```

### 稀有度

| 稀有度 | 星级 | 权重 |
|--------|------|------|
| Common | ★ | 60% |
| Uncommon | ★★ | 25% |
| Rare | ★★★ | 10% |
| Epic | ★★★★ | 4% |
| Legendary | ★★★★★ | 1% |

孵化时基于种子随机决定，存在极低概率出现 Shiny（闪光）变体。

### 源码映射

稀有度权重定义在 [`types.ts#L127-L149`](/packages/ccb/src/buddy/types.ts#L127-L149)：

```typescript
export const RARITY_WEIGHTS = {
  common: 60,
  uncommon: 25,
  rare: 10,
  epic: 4,
  legendary: 1,
} as const satisfies Record<Rarity, number>

export const RARITY_STARS = {
  common: '★',
  uncommon: '★★',
  rare: '★★★',
  epic: '★★★★',
  legendary: '★★★★★',
} as const satisfies Record<Rarity, string>
```

**稀有度掉落逻辑**（[`companion.ts#L43-L51`](/packages/ccb/src/buddy/companion.ts#L43-L51)）：
```typescript
function rollRarity(rng: () => number): Rarity {
  const total = Object.values(RARITY_WEIGHTS).reduce((a, b) => a + b, 0)
  let roll = rng() * total
  for (const rarity of RARITIES) {
    roll -= RARITY_WEIGHTS[rarity]
    if (roll < 0) return rarity
  }
  return 'common'
}
```

**Shiny 概率**（[`companion.ts#L98`](/packages/ccb/src/buddy/companion.ts#L98)）：
```typescript
shiny: rng() < 0.01,  // 1% 闪光概率
```

### 属性值

每只宠物拥有 5 项属性（0-100）：

- **DEBUGGING** — 调试能力
- **PATIENCE** — 耐心程度
- **CHAOS** — 混乱指数
- **WISDOM** — 智慧值
- **SNARK** — 毒舌度

### 源码映射

属性名定义在 [`types.ts#L91-L98`](/packages/ccb/src/buddy/types.ts#L91-L98)：

```typescript
export const STAT_NAMES = [
  'DEBUGGING',
  'PATIENCE',
  'CHAOS',
  'WISDOM',
  'SNARK',
] as const
```

**属性滚动逻辑**（[`companion.ts#L53-L82`](/packages/ccb/src/buddy/companion.ts#L53-L82)）：
```typescript
const RARITY_FLOOR: Record<Rarity, number> = {
  common: 5,
  uncommon: 15,
  rare: 25,
  epic: 35,
  legendary: 50,
}

function rollStats(rng: () => number, rarity: Rarity): Record<StatName, number> {
  const floor = RARITY_FLOOR[rarity]
  const peak = pick(rng, STAT_NAMES)
  let dump = pick(rng, STAT_NAMES)
  while (dump === peak) dump = pick(rng, STAT_NAMES)

  const stats = {} as Record<StatName, number>
  for (const name of STAT_NAMES) {
    if (name === peak) {
      stats[name] = Math.min(100, floor + 50 + Math.floor(rng() * 30))
    } else if (name === dump) {
      stats[name] = Math.max(1, floor - 10 + Math.floor(rng() * 15))
    } else {
      stats[name] = floor + Math.floor(rng() * 40)
    }
  }
  return stats
}
```

### 外观

每只宠物还有随机的外观配件：

- **眼睛**: `·` `✦` `×` `◉` `@` `°`
- **帽子**: none, crown, tophat, propeller, halo, wizard, beanie, tinyduck

### 源码映射

眼睛和帽子定义在 [`types.ts#L76-L89`](/packages/ccb/src/buddy/types.ts#L76-L89)：

```typescript
export const EYES = ['·', '✦', '×', '◉', '@', '°'] as const
export const HATS = [
  'none',
  'crown',
  'tophat',
  'propeller',
  'halo',
  'wizard',
  'beanie',
  'tinyduck',
] as const
```

**帽子渲染映射**（[`sprites.ts#L443-L452`](/packages/ccb/src/buddy/sprites.ts#L443-L452)）：
```typescript
const HAT_LINES: Record<Hat, string> = {
  none: '',
  crown: '   \^^^/    ',
  tophat: '   [___]    ',
  propeller: '    -+-     ',
  halo: '   (   )    ',
  wizard: '    /^\     ',
  beanie: '   (___)    ',
  tinyduck: '    ,>      ',
}
```

---

## 数据存储

宠物信息存储在 `~/.claude.json` 的 `companion` 字段中。宠物的外观属性（物种、稀有度、属性值等）基于用户 ID 的哈希确定性生成，不可通过编辑配置文件来篡改稀有度。

### 为什么用用户 ID 哈希而非随机数？

使用确定性种子有两个好处：
1. **同一用户始终获得同一只宠物**，跨设备登录时体验一致。
2. **防止外部修改**，即使玩家手动编辑 `~/.claude.json`，只要 `seed` 不变，重新计算后的属性会覆盖掉任何篡改。

### 源码映射

**配置存储结构**（[`types.ts#L122-L125`](/packages/ccb/src/buddy/types.ts#L122-L125)）：
```typescript
// What actually persists in config
export type StoredCompanion = CompanionSoul & { hatchedAt: number }
```

**用户 ID 哈希生成**（[`companion.ts#L16-L37`](/packages/ccb/src/buddy/companion.ts#L16-L37) 及 [`companion.ts#L84-L113`](/packages/ccb/src/buddy/companion.ts#L84-L113)）：
```typescript
const SALT = 'friend-2026-401'

// Mulberry32 — tiny seeded PRNG
function mulberry32(seed: number): () => number {
  let a = seed >>> 0
  return function () {
    a |= 0
    a = (a + 0x6d2b79f5) | 0
    let t = Math.imul(a ^ (a >>> 15), 1 | a)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}

function hashString(s: string): number {
  if (typeof Bun !== 'undefined') {
    return Number(BigInt(Bun.hash(s)) & 0xffffffffn)
  }
  let h = 2166136261
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i)
    h = Math.imul(h, 16777619)
  }
  return h >>> 0
}

// 缓存机制：同一 userId 总是返回相同结果
let rollCache: { key: string; value: Roll } | undefined
export function roll(userId: string): Roll {
  const key = userId + SALT
  if (rollCache?.key === key) return rollCache.value
  const value = rollFrom(mulberry32(hashString(key)))
  rollCache = { key, value }
  return value
}
```

**宠物读取逻辑**（[`companion.ts#L129-L136`](/packages/ccb/src/buddy/companion.ts#L129-L136)）：
```typescript
export function getCompanion(): Companion | undefined {
  const stored = getGlobalConfig().companion
  if (!stored) return undefined
  const seed = stored.seed ?? companionUserId()
  const { bones } = rollWithSeed(seed)
  // bones 最后合并，确保旧格式配置中的 stale bones 字段被覆盖
  return { ...stored, ...bones }
}
```

**用户 ID 获取**（[`companion.ts#L123-L126`](/packages/ccb/src/buddy/companion.ts#L123-L126)）：
```typescript
export function companionUserId(): string {
  const config = getGlobalConfig()
  return config.oauthAccount?.accountUuid ?? config.userID ?? 'anon'
}
```

---

## 相关源码

| 文件 | 说明 |
|------|------|
| [`commands/buddy/buddy.ts`](/packages/ccb/src/commands/buddy/buddy.ts#L1-L169) | `/buddy` 命令处理 |
| [`buddy/companion.ts`](/packages/ccb/src/buddy/companion.ts#L1-L136) | 宠物生成与加载 |
| [`buddy/types.ts`](/packages/ccb/src/buddy/types.ts#L1-L149) | 类型定义（物种、稀有度、属性） |
| [`buddy/sprites.ts`](/packages/ccb/src/buddy/sprites.ts#L1-L514) | 终端像素画渲染 |
| [`buddy/CompanionSprite.tsx`](/packages/ccb/src/buddy/CompanionSprite.tsx#L1-L347) | React 组件（输入框旁显示） |
| [`buddy/CompanionCard.tsx`](/packages/ccb/src/buddy/CompanionCard.tsx#L1-L110) | `/buddy` 显示卡片 |
| [`buddy/useBuddyNotification.tsx`](/packages/ccb/src/buddy/useBuddyNotification.tsx#L1-L67) | 启动提示通知 |
| [`buddy/companionReact.ts`](/packages/ccb/src/buddy/companionReact.ts#L1-L160) | 宠物反应触发 |
| [`buddy/prompt.ts`](/packages/ccb/src/buddy/prompt.ts#L1-L36) | 宠物相关 prompt 模板 |

### 源码索引表

| 链接 | 对应内容 | 状态 |
|------|----------|------|
| `types.ts#L1-L8` | `RARITIES` 稀有度枚举 | ✅ |
| `types.ts#L17-L73` | 18 个物种定义 | ✅ |
| `types.ts#L76-L89` | `EYES` / `HATS` 外观配件 | ✅ |
| `types.ts#L91-L98` | `STAT_NAMES` 五项属性 | ✅ |
| `types.ts#L101-L108` | `CompanionBones` 类型 | ✅ |
| `types.ts#L111-L125` | `CompanionSoul` / `StoredCompanion` 类型 | ✅ |
| `types.ts#L127-L149` | `RARITY_WEIGHTS` / `RARITY_STARS` / `RARITY_COLORS` | ✅ |
| `companion.ts#L16-L37` | Mulberry32 PRNG + `hashString` | ✅ |
| `companion.ts#L43-L51` | `rollRarity` 稀有度滚动 | ✅ |
| `companion.ts#L53-L82` | `rollStats` 属性值生成 | ✅ |
| `companion.ts#L84-L113` | `roll()` 确定性生成 + 缓存 | ✅ |
| `companion.ts#L129-L136` | `getCompanion()` 读取合并 | ✅ |
| `sprites.ts#L26-L441` | 18 物种的 3 帧动画精灵图 | ✅ |
| `sprites.ts#L443-L452` | `HAT_LINES` 帽子渲染 | ✅ |
| `sprites.ts#L454-L473` | `renderSprite()` / `spriteFrameCount()` | ✅ |
| `sprites.ts#L475-L514` | `renderFace()` 脸部渲染 | ✅ |
| `CompanionSprite.tsx#L34-L48` | `wrap()` 文本换行 | ✅ |
| `CompanionSprite.tsx#L50-L100` | `SpeechBubble` 组件 | ✅ |
| `CompanionSprite.tsx#L118-L129` | `companionReservedColumns()` 保留列计算 | ✅ |
| `CompanionSprite.tsx#L131-L304` | `CompanionSprite` 主组件 | ✅ |
| `CompanionSprite.tsx#L310-L347` | `CompanionFloatingBubble()` 悬浮气泡 | ✅ |
| `CompanionCard.tsx#L25-L110` | `CompanionCard` 卡片组件 | ✅ |
| `useBuddyNotification.tsx#L11-L23` | `isBuddyTeaserWindow()` / `isBuddyLive()` | ✅ |
| `useBuddyNotification.tsx#L39-L54` | `useBuddyNotification()` hook | ✅ |
| `useBuddyNotification.tsx#L56-L67` | `findBuddyTriggerPositions()` | ✅ |
| `companionReact.ts#L38-L63` | `triggerCompanionReaction()` 触发逻辑 | ✅ |
| `companionReact.ts#L67-L107` | `isAddressed()` / `buildTranscript()` | ✅ |
| `companionReact.ts#L111-L160` | `callBuddyReactAPI()` API 调用 | ✅ |
| `prompt.ts#L7-L13` | `companionIntroText()` intro 文本 | ✅ |
| `prompt.ts#L15-L36` | `getCompanionIntroAttachment()` 附件生成 | ✅ |
| `buddy.ts#L19-L64` | `SPECIES_NAMES` / `SPECIES_PERSONALITY` | ✅ |
| `buddy.ts#L70-L169` | `call()` 命令分发 | ✅ |

---

## 附录：动画与交互

### 空闲动画序列

宠物在空闲时执行 15 帧循环动画（[`CompanionSprite.tsx#L22`](/packages/ccb/src/buddy/CompanionSprite.tsx#L22)）：

```typescript
// 大部分时间静止（frame 0），偶尔抖动（frames 1-2），罕见眨眼
const IDLE_SEQUENCE = [0, 0, 0, 0, 1, 0, 0, 0, -1, 0, 0, 2, 0, 0, 0]
```

### 爱心动画

`/buddy pet` 后触发爱心动画，持续 5 帧约 2.5 秒（[`CompanionSprite.tsx#L24-L32`](/packages/ccb/src/buddy/CompanionSprite.tsx#L24-L32)）：

```typescript
const H = figures.heart
const PET_HEARTS = [
  `   ${H}    ${H}   `,
  `  ${H}  ${H}   ${H}  `,
  ` ${H}   ${H}  ${H}   `,
  `${H}  ${H}      ${H} `,
  '·    ·   ·  ',
]
```

### 语音气泡淡出

气泡显示约 10 秒后开始淡出（[`CompanionSprite.tsx#L16-L17`](/packages/ccb/src/buddy/CompanionSprite.tsx#L16-L17)）：

```typescript
const BUBBLE_SHOW = 20      // 20 ticks × 500ms = 10 秒
const FADE_WINDOW = 6       // 最后 6 ticks ≈ 3 秒淡出
```

---

*Unit 36 完成。*
