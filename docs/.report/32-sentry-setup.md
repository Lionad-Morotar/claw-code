# Unit 32 — 自定义 Sentry 错误上报配置

> **原始页面**：https://ccb.agent-aura.top/docs/internals/sentry-setup
> **源码映射**：完整实现存在于 `packages/ccb/src/`（TypeScript 上游子模块）。Rust CLI（`rust/`）未移植此功能。
> **生成日期**：2026-04-10（重写版，修正原版错误声明）

---

## 1. 功能概述

Claude Code 支持通过 Sentry 捕获运行时异常并上报到自定义 Sentry 实例。

- **配置了 `SENTRY_DSN`**：自动初始化 Sentry SDK，捕获未处理异常和关键错误
- **未配置**：所有 Sentry 调用均为 no-op，零运行时开销

---

## 2. 环境变量

| 变量 | 必填 | 说明 |
|------|------|------|
| `SENTRY_DSN` | 是 | Sentry 项目 DSN，如 `https://xxx@o123456.ingest.sentry.io/789` |

---

## 3. 使用方式

```bash
# 自托管 Sentry
SENTRY_DSN=https://public_key@your-sentry.example.com/123 bun run dev

# Sentry Cloud (SaaS)
SENTRY_DSN=https://public_key@o123456.ingest.sentry.io/789 bun run dev

# 不使用 Sentry（默认行为）— SENTRY_DSN 未设置时所有函数为 no-op
bun run dev
```

---

## 4. 源码实现

### 4.1 核心 SDK 初始化

**文件**: [`packages/ccb/src/utils/sentry.ts`](/packages/ccb/src/utils/sentry.ts)（160 行）

[`initSentry()`](/packages/ccb/src/utils/sentry.ts#L17-L83) 函数在进程启动时调用：

```typescript
// sentry.ts#L17-L83
export function initSentry(): void {
  if (initialized) return;
  const dsn = process.env.SENTRY_DSN;
  if (!dsn) return;

  Sentry.init({
    dsn,
    release: MACRO.VERSION,
    environment: BUILD_ENV ?? process.env.NODE_ENV ?? 'development',
    maxBreadcrumbs: 20,
    sampleRate: 1.0,

    beforeSend(event) {
      // 剥离敏感 header: authorization, x-api-key, cookie, set-cookie
      // ...
      return event;
    },

    ignoreErrors: [
      'ECONNREFUSED', 'ECONNRESET', 'ENOTFOUND', 'ETIMEDOUT',
      'AbortError', 'The user aborted a request', 'CancelError',
    ],

    beforeSendTransaction(event) {
      return null; // 关闭性能事务，仅上报错误
    },
  });
  initialized = true;
}
```

### 4.2 导出 API

| 函数 | 行号 | 说明 |
|------|------|------|
| [`initSentry()`](/packages/ccb/src/utils/sentry.ts#L17) | L17-83 | 初始化 SDK，幂等调用 |
| [`captureException(error, context?)`](/packages/ccb/src/utils/sentry.ts#L89) | L89-104 | 手动上报异常，可选附加上下文 |
| [`setTag(key, value)`](/packages/ccb/src/utils/sentry.ts#L110) | L110-120 | 设置标签，面板分组用 |
| [`setUser({ id, email, username })`](/packages/ccb/src/utils/sentry.ts#L126) | L126-136 | 设置用户上下文 |
| [`closeSentry(timeoutMs?)`](/packages/ccb/src/utils/sentry.ts#L142) | L142-153 | 刷出队列并关闭，默认 2s 超时 |
| [`isSentryInitialized()`](/packages/ccb/src/utils/sentry.ts#L158) | L158-160 | 检查初始化状态 |

所有函数在 `initialized === false` 时立即返回（no-op），不会产生任何 Sentry SDK 调用。

### 4.3 错误捕获机制

| 机制 | 文件 | 说明 |
|------|------|------|
| React Error Boundary | [`SentryErrorBoundary.ts`](/packages/ccb/src/components/SentryErrorBoundary.ts) | 包裹关键组件，捕获渲染错误 |
| 错误日志 sink | [`errorLogSink.ts`](/packages/ccb/src/utils/errorLogSink.ts) | 写入错误日志时同步调用 `captureException` |
| 优雅关闭 | [`gracefulShutdown.ts`](/packages/ccb/src/utils/gracefulShutdown.ts) | 进程退出时调用 `closeSentry()` |
| 启动初始化 | [`init.ts`](/packages/ccb/src/entrypoints/init.ts) | 启动流程中调用 `initSentry()` |

### 4.4 安全过滤

`beforeSend` 钩子（[sentry.ts#L41-L58](/packages/ccb/src/utils/sentry.ts#L41-L58)）自动剥离 4 种敏感 header：

- `authorization`
- `x-api-key`
- `cookie`
- `set-cookie`

### 4.5 忽略的错误类型

[ignoreErrors 配置](/packages/ccb/src/utils/sentry.ts#L62-L73)：

| 错误模式 | 原因 |
|----------|------|
| `ECONNREFUSED` / `ECONNRESET` / `ENOTFOUND` / `ETIMEDOUT` | 网络不可达，不可操作 |
| `AbortError` / "The user aborted a request" | 用户主动取消 |
| `CancelError` | 交互式取消信号 |

### 4.6 配置参数

| 参数 | 值 | 说明 |
|------|------|------|
| `sampleRate` | `1.0` | 捕获全部错误事件 |
| `maxBreadcrumbs` | `20` | 面包屑上限，控制 payload 体积 |
| `beforeSendTransaction` | `() => null` | 关闭性能事务，仅上报错误 |

---

## 5. Rust CLI 实现现状

`claw-code` Rust CLI（`rust/`）**未移植 Sentry 集成**：

- `rust/Cargo.toml` 及所有子 crate 未引入 `sentry` 依赖
- Rust 运行时无 `SENTRY_DSN` 环境变量读取
- 错误通过 `Result<T, E>` 链传播至顶层渲染，无外部上报管道
- Rust 生态更倾向 `tracing` + OpenTelemetry（`telemetry` crate 已实现基础 JSONL/内存遥测，但未集成 OpenTelemetry）

若未来需要在 Rust CLI 中支持 Sentry，建议：
1. 在 `main.rs` 入口读取 `SENTRY_DSN` 并初始化 `sentry` crate
2. 注册 panic hook 调用 `sentry::capture_panic`
3. 退出前调用 `sentry::close(timeout)` 刷出队列

---

## 6. 文件索引

| 文件 | 行数 | 职责 |
|------|------|------|
| [`packages/ccb/src/utils/sentry.ts`](/packages/ccb/src/utils/sentry.ts) | 160 | 核心 SDK 初始化与 6 个导出函数 |
| [`packages/ccb/src/components/SentryErrorBoundary.ts`](/packages/ccb/src/components/SentryErrorBoundary.ts) | — | React Error Boundary 组件 |
| [`packages/ccb/src/utils/errorLogSink.ts`](/packages/ccb/src/utils/errorLogSink.ts) | — | 错误日志 sink，集成 `captureException` |
| [`packages/ccb/src/utils/gracefulShutdown.ts`](/packages/ccb/src/utils/gracefulShutdown.ts) | — | 优雅退出，调用 `closeSentry()` |
| [`packages/ccb/src/entrypoints/init.ts`](/packages/ccb/src/entrypoints/init.ts) | — | 启动流程中调用 `initSentry()` |
