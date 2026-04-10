# Unit 56 — Auto updater

> **原始页面**: https://ccb.agent-aura.top/docs/auto-updater  
> **源码主项目**: `packages/ccb`  
> **生成时间**: 2026-04-09


> **源码映射说明**：Auto Updater 全部基于 `packages/ccb`（Claude Code 上游 TypeScript 实现）。`claw-code`（Rust 重写版）目前尚未实现此功能。本报告引用的 `packages/ccb/src/...` 路径在上游实现中存在，但在当前仓库中**不存在对应源码文件**。阅读时请注意区分上游与 Rust 实现的覆盖范围。---

## 1. 自动更新机制概述

Claude Code 的自动更新系统区分三种安装方式处理：`native`（二进制原生安装）、`npm-global/local`（JS/npm 安装）和 `package-manager`（包管理器安装）。

- `native` 与 `npm-*` 安装支持后台自动更新（30 分钟轮询）。
- `package-manager` 安装**仅展示升级命令**，不会自动执行安装，因为包管理器版本需要用户侧权限与一致性保证。

---

## 2. 安装类型检测

安装类型检测入口在 `getCurrentInstallationType()`（`src/utils/doctorDiagnostic.ts#L86-L148`）：

- `development`：`NODE_ENV === 'development'` 时直接返回
- `package-manager`：通过 `detectHomebrew()`、`detectWinget()`、`detectPacman()`、`detectDeb()`、`detectRpm()`、`detectApk()` 检测
- `npm-local`：`isRunningFromLocalInstallation()` 判断
- `npm-global`：通过路径前缀（`/usr/local/lib/node_modules`、nvm 路径等）或 `npm config get prefix` 判断
- `native`：`isInBundledMode()` 为真且未命中包管理器检测

`AutoUpdaterWrapper.tsx`（`src/components/AutoUpdaterWrapper.tsx#L35-L58`）根据检测结果挂载不同的更新器组件：

```tsx
const installationType = await getCurrentInstallationType();
setUseNativeInstaller(installationType === 'native');
setIsPackageManager(installationType === 'package-manager');
```

---

## 3. 前置检查门控

### 3.1 自动更新是否被禁用

`getAutoUpdaterDisabledReason()`（`src/utils/config.ts#L1735-L1761`）按以下优先级检查：

- `NODE_ENV === 'development'`
- `DISABLE_AUTOUPDATER` 环境变量
- 仅限必要流量模式（essential traffic only）
- `config.autoUpdates === false`（原生安装的保护模式 `autoUpdatesProtectedForNative` 除外）

### 3.2 最大版本上限

`getMaxVersion()`（`src/utils/autoUpdater.ts#L108-L114`）从 GrowthBook 读取 `tengu_max_version_config`，按用户类型（`ant` / `external`）返回服务端熔断版本。用于在 incidents 期间暂停自动更新。

### 3.3 跳过版本检查

`shouldSkipVersion()`（`src/utils/autoUpdater.ts#L145-L158`）读取用户 `settings.minimumVersion`，通过 `semver.gte()` 判断目标版本是否低于用户设定的最低版本，防止切换频道时意外降级。

---

## 4. 原生自动更新器

### 4.1 NativeAutoUpdater 组件

`src/components/NativeAutoUpdater.tsx#L57-L231` 是原生安装的后台更新器，核心流程：

```tsx
const checkForUpdates = React.useCallback(async () => {
  // 100ms 间隔轮询使用 isUpdatingRef 防止并发
  if (isUpdatingRef.current) return;
  onChangeIsUpdating(true);
  logEvent('tengu_native_auto_updater_start', {});

  const maxVersion = await getMaxVersion();
  if (maxVersion && gt(MACRO.VERSION, maxVersion)) {
    setMaxVersionIssue(msg ?? 'affects your version');
  }

  const result = await installLatest(channel);
  // ... 成功 / 失败 / 锁竞争处理
}, [onAutoUpdaterResult, channel]);

useInterval(checkForUpdates, 30 * 60 * 1000); // 每 30 分钟
```

错误分类函数 `getErrorType()`（`L19-L46`）将错误映射为：
`timeout`、`checksum_mismatch`、`not_found`、`permission_denied`、`disk_full`、`npm_error`、`network_error`、`unknown`。

### 4.2 原生安装器

`src/utils/nativeInstaller/installer.ts` 是核心安装逻辑：

| 函数 | 行号 | 职责 |
|------|------|------|
| `installLatest()` | `#L956-L969` | Singleflight 包装，防止重复安装 |
| `installLatestImpl()` | `#L976-L990` | 调用 `updateLatest()` |
| `updateLatest()` | `#L495-L622` | 核心更新流程：max version 检查 → 版本比对 → 文件锁 → 下载 → 安装 → 更新符号链接 |
| `performVersionUpdate()` | `#L441-L489` | 实际执行下载与安装到 `versions/` 目录 |
| `versionIsAvailable()` | `#L490-L494` | 检查版本是否已存在于本地 |
| `cleanupOldVersions()` | `#L1184-L1276` | 清理过期版本与暂存区，保留最近 2 个版本 |
| `lockCurrentVersion()` | `#L1048-L1117` | 进程生命周期锁，保护当前运行版本不被删除 |

二进制文件存储布局（`getBaseDirectories()` 相关逻辑 `#L115-L152`）：

```
~/.local/share/claude-code/
├── versions/          # claude-1.0.3, claude-1.0.4, ...
├── staging/           # 临时下载区
└── locks/             # 版本级文件锁
~/.local/bin/claude    # 指向当前版本的符号链接
```

Windows 使用文件复制而非符号链接（`updateSymlink()` `#L639-L799`）。

### 4.3 下载与校验

`src/utils/nativeInstaller/download.ts`：

- `getLatestVersionFromBinaryRepo()`（`#L74-L111`）从 GCS 存储桶拉取版本指针
- `getLatestVersionFromArtifactory()`（`#L30-L73`）内部用户通过 npm view 获取
- `downloadVersionFromBinaryRepo()`（`#L382-L486`）下载二进制并验证 SHA256
- `downloadAndVerifyBinary()`（`#L293-L381`）实现 60 秒卡顿检测（Stall Timeout）、3 次自动重试、SHA256 校验和验证

```ts
const DEFAULT_STALL_TIMEOUT_MS = 60000;
const MAX_DOWNLOAD_RETRIES = 3;
```

---

## 5. JS/npm 自动更新器

### 5.1 AutoUpdater 组件

`src/components/AutoUpdater.tsx#L38-L264` 处理 `npm-global` 与 `npm-local` 安装：

```tsx
const checkForUpdates = React.useCallback(async () => {
  const latestVersion = await getLatestVersion(channel);
  const maxVersion = await getMaxVersion();
  // ... 门控逻辑

  const installationType = await getCurrentInstallationType();
  if (installationType === 'npm-local') {
    installStatus = await installOrUpdateClaudePackage(channel);
  } else if (installationType === 'npm-global') {
    installStatus = await installGlobalPackage();
  }
  // ... 事件上报与结果回调
}, [onAutoUpdaterResult]);
```

### 5.2 npm 安装工具函数

`src/utils/autoUpdater.ts`：

| 函数 | 行号 | 职责 |
|------|------|------|
| `getLatestVersion()` | `#L319-L345` | `npm view @anthropic-ai/claude-code@<tag> version` |
| `getNpmDistTags()` | `#L355-L383` | 获取 `latest` 和 `stable` dist-tags |
| `installGlobalPackage()` | `#L456-L503` | 全局 `npm install -g` / `bun install -g`，含权限检查和文件锁 |
| `checkGlobalInstallPermissions()` | `#L292-L318` | 调用 `npm -g config get prefix` 并检查写权限 |

### 5.3 本地安装器

`src/utils/localInstaller.ts` 处理 `~/.claude/local/` 目录下的本地 npm 安装，提供 `installOrUpdateClaudePackage()`。

---

## 6. 包管理器通知器

`src/components/PackageManagerAutoUpdater.tsx#L29-L119` 不会自动安装，仅展示对应操作系统的升级命令：

```tsx
const updateCommand =
  packageManager === 'homebrew' ? 'brew upgrade claude-code'
  : packageManager === 'winget' ? 'winget upgrade Anthropic.ClaudeCode'
  : packageManager === 'apk' ? 'apk upgrade claude-code'
  : 'your package manager update command';
```

版本检查通过 `getLatestVersionFromGcs()`（`src/utils/autoUpdater.ts#L384-L402`）获取 GCS 指针，而非 npm registry。

---

## 7. 启动版本门控

`assertMinVersion()`（`src/utils/autoUpdater.ts#L70-L98`）定义了启动时最低版本检查逻辑：

```ts
export async function assertMinVersion(): Promise<void> {
  const versionConfig = await getDynamicConfig_BLOCKS_ON_INIT('tengu_version_config', { minVersion: '0.0.0' });
  if (versionConfig.minVersion && lt(MACRO.VERSION, versionConfig.minVersion)) {
    console.error('...needs an update...');
    gracefulShutdownSync(1);
  }
}
```

> **注**：`assertMinVersion()` 保留了完整的硬性门控逻辑。当前实际调用点是否活跃，需结合具体构建版本确认。

---

## 8. 手动 CLI 命令

### 8.1 `claude update`

`src/cli/update.ts#L30` 执行 `update()` 函数，完整流程：

1. `getDoctorDiagnostic()` 检查系统健康状态（`L41`）
2. 检测多安装冲突并输出警告（`L48-L58`）
3. 根据安装类型路由：
   - `development` → 报错退出（`L109-L115`）
   - `package-manager` → 打印对应包管理器命令（`L118-L167`）
   - `native` → `installLatestNative(channelOrVersion)`（`L170-L190`）
   - `npm-local` → `installOrUpdateClaudePackage(channel)`（`L193-L210`）
   - `npm-global` → `installGlobalPackage(specificVersion)`（`L213-L230`）

### 8.2 安装/回滚命令

- `claude install [target]` → `src/commands/install.tsx`
- `claude rollback [target]` → 仅限内部用户，支持 `--list`、`--dry-run`、`--safe`
- `claude doctor` → `src/screens/Doctor.tsx#L1-L516` 展示自动更新器健康状态

---

## 9. 文件锁机制

`acquireLock()` / `releaseLock()`（`src/utils/autoUpdater.ts#L176-L267`）用于防止并发更新：

- 锁文件路径：`~/.claude/.update.lock`
- 5 分钟超时：超过 5 分钟的锁视为过期，强制获取
- 原子创建：使用 `O_EXCL`（`flag: 'wx'`）避免竞态
- TOCTOU 防护：在删除过期锁前重新检查 mtime

原生安装器还使用版本级锁 `tryWithVersionLock()`（`src/utils/nativeInstaller/installer.ts#L181-L225`）和进程生命周期锁 `lockCurrentVersion()`（`#L1048-L1117`）。

---

## 10. 语义版本比较

`src/utils/semver.ts#L1-L60` 提供封装：

- Bun 环境下使用 `Bun.semver.order()`（快约 20 倍）
- Node.js 环境下回退到 npm `semver` 包，默认 `{ loose: true }`

导出函数：`gt()`、`gte()`、`lt()`、`lte()`、`satisfies()`、`order()`。

---

## 11. 更新通知去重

`src/hooks/useUpdateNotification.ts#L1-L35` 按 semver（`major.minor.patch`）去重，确保同一 patch 版本只通知一次：

```ts
export function useUpdateNotification(updatedVersion: string | null): string | null {
  const [lastNotifiedSemver, setLastNotifiedSemver] = useState(() => getSemverPart(MACRO.VERSION));
  const updatedSemver = getSemverPart(updatedVersion);
  if (updatedSemver !== lastNotifiedSemver) {
    setLastNotifiedSemver(updatedSemver);
    return updatedSemver;
  }
  return null;
}
```

---

## 12. 更新日志

`src/utils/releaseNotes.ts`：

- `CHANGELOG_URL`（`L28`）和 `RAW_CHANGELOG_URL`（`L30-L31`）指向 GitHub 仓库
- `fetchAndStoreChangelog()`（`L82-L109`）从 GitHub 获取并缓存到 `~/.claude/cache/changelog.md`
- `migrateChangelogFromConfig()`（`L55-L76`）一次性将旧配置存储迁移到文件缓存
- 每次启动时调用（`src/setup.ts`），通过 `semver.gt()` 判断哪些版本的日志比 `lastReleaseNotesSeen` 更新

---

## 13. 配置与迁移

### 13.1 全局配置字段

`src/utils/config.ts#L191-L193`：

```ts
autoUpdates?: boolean
autoUpdatesProtectedForNative?: boolean
```

### 13.2 Settings 类型

`src/utils/settings/types.ts` 中定义：

- `autoUpdatesChannel: 'latest' | 'stable'`
- `minimumVersion: string`

### 13.3 配置迁移

`migrateConfigFields()`（`src/utils/config.ts#L912-L960`）将旧版 `autoUpdaterStatus` 字段一次性迁移为 `installMethod` + `autoUpdates`。

---

## 14. 关键文件索引

| 文件 | 主要职责 |
|------|----------|
| `packages/ccb/src/utils/autoUpdater.ts` | 自动更新核心：版本检查、npm 安装、文件锁、门控 |
| `packages/ccb/src/utils/nativeInstaller/installer.ts` | 原生二进制安装、符号链接、版本清理 |
| `packages/ccb/src/utils/nativeInstaller/download.ts` | GCS/Artifactory 下载、SHA256 校验、卡顿检测 |
| `packages/ccb/src/components/AutoUpdaterWrapper.tsx` | 安装类型检测与策略路由 |
| `packages/ccb/src/components/AutoUpdater.tsx` | JS/npm 后台自动更新器（30 分钟轮询） |
| `packages/ccb/src/components/NativeAutoUpdater.tsx` | 原生二进制后台自动更新器 |
| `packages/ccb/src/components/PackageManagerAutoUpdater.tsx` | 包管理器通知（显示升级命令） |
| `packages/ccb/src/cli/update.ts` | `claude update` 命令处理 |
| `packages/ccb/src/hooks/useUpdateNotification.ts` | semver 级去重通知 |
| `packages/ccb/src/utils/releaseNotes.ts` | Changelog 获取、缓存与展示 |
| `packages/ccb/src/utils/semver.ts` | Semver 比较（Bun 原生 + npm 回退） |
| `packages/ccb/src/utils/doctorDiagnostic.ts` | 安装类型检测与健康诊断 |
| `packages/ccb/src/utils/config.ts` | 配置读取与迁移 |
| `packages/ccb/src/screens/Doctor.tsx` | Doctor 命令 UI |
