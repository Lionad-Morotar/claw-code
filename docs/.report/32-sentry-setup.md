# Unit 32 — 自定义 Sentry 错误上报配置

> **原始页面**：https://ccb.agent-aura.top/docs/internals/sentry-setup

---

## 摘要

原始文档声称 Claude Code 支持通过环境变量 `SENTRY_DSN` 连接自托管或 SaaS Sentry，实现 CLI 运行时的错误捕获与上报。但**经对当前代码库全面检索，Sentry 相关实现不存在于任何源码路径中**。本文档同时记录原始文档描述的功能架构，并明确指出其与当前代码库的差异，方便后续维护者判断是“尚未移植”还是“已废弃”。

---

## 1. 原始文档概述

根据原始页面，Sentry 集成遵循如下设计原则：

- **单一开关**：仅需环境变量 `SENTRY_DSN` 即可启用。
- **默认静默**：未配置时，所有 Sentry 调用均为 no-op，零运行时开销。
- **两种部署形态**：支持 Sentry Cloud (SaaS) 与自托管实例。

### 1.1 环境变量

| 变量 | 必填 | 说明 |
|------|------|------|
| `SENTRY_DSN` | 是 | 项目 DSN，例如 `https://public_key@o123456.ingest.sentry.io/789` |

### 1.2 使用方式（原始文档示例）

```bash
# 自托管 Sentry
SENTRY_DSN=https://public_key@your-sentry.example.com/123 \
  bun run dev

# Sentry Cloud (SaaS)
SENTRY_DSN=https://public_key@o123456.ingest.sentry.io/789 \
  bun run dev

# 不使用 Sentry（默认行为）
bun run dev
```

> 注意：示例中使用 `bun run dev`，暗示原始文档可能针对 Node.js/TypeScript 运行时。

---

## 2. 代码库现状：Sentry 实现缺失

### 2.1 检索范围与方法

子代理执行了以下全覆盖检索：

1. **全局文件名匹配**：搜索任何包含 `sentry` 的文件名（Rust、TS、JS）。
2. **全局内容匹配**：在 `rust/`、`src/`、`tests/` 目录下搜索关键词 `sentry`、`Sentry`、`DSN`、`dsn`、`captureException`、`beforeSend`、`errorLogSink`、`gracefulShutdown`、`SentryErrorBoundary`。
3. **依赖项检查**：检查 `rust/Cargo.toml`、`rust/crates/*/Cargo.toml` 与 `package.json`，未发现 `sentry` 相关依赖。
4. **Rust CLI 主入口扫描**：对 `rust/crates/rusty-claude-cli/src/main.rs` 进行关键词搜索，无 Sentry 引用。

### 2.2 结果

**当前代码库中不存在 Sentry 错误上报的任何实现。** 下表列出原始文档声称的实现文件与实际仓库状态的对比：

| 原始文档声称文件 | 当前仓库状态 | 说明 |
|------------------|--------------|------|
| `src/utils/sentry.ts` | **不存在** | 核心 SDK 初始化与封装 |
| `src/components/SentryErrorBoundary.ts` | **不存在** | React Error Boundary 组件 |
| `src/utils/errorLogSink.ts` | **不存在** | 错误日志 sink，集成 `captureException` |
| `src/utils/gracefulShutdown.ts` | **不存在** | 优雅退出，调用 `closeSentry()` |
| `src/entrypoints/init.ts` | **不存在** | 启动时调用 `initSentry()` |

---

## 3. 原始文档声称的功能架构

虽然当前仓库未实现，但为保持单元文档完整性，以下记录原始文档的功能设计，供后续移植参考。

### 3.1 错误捕获机制

| 机制 | 描述 |
|------|------|
| 自动捕获 | `SentryErrorBoundary` 包裹关键 React 组件，捕获渲染错误 |
| 手动上报 | `errorLogSink` 在写入错误日志时同步调用 `captureException` |
| 优雅关闭 | 进程退出时调用 `closeSentry()`，2 秒超时确保事件发送完毕 |

### 3.2 安全过滤

`beforeSend` 钩子声称会自动剥离以下敏感 Header：

- `authorization`
- `x-api-key`
- `cookie`
- `set-cookie`

### 3.3 忽略的错误类型

以下错误模式被声明为不上报：

| 错误模式 | 原因 |
|----------|------|
| `ECONNREFUSED` / `ECONNRESET` / `ENOTFOUND` / `ETIMEDOUT` | 网络不可达，不可操作 |
| `AbortError` / “The user aborted a request” | 用户主动取消 |
| `CancelError` | 交互式取消信号 |

### 3.4 其他配置参数

- **采样率**：`sampleRate: 1.0`（捕获全部错误事件）
- **面包屑上限**：`maxBreadcrumbs: 20`
- **性能事务**：已关闭（`beforeSendTransaction` 返回 `null`），仅上报错误事件

### 3.5  claimed API 列表

| 函数 | 说明 |
|------|------|
| `initSentry()` | 初始化 SDK，在 `src/entrypoints/init.ts` 中自动调用 |
| `captureException(error, context?)` | 手动上报异常，可附加额外上下文 |
| `setTag(key, value)` | 设置标签，用于 Sentry 面板分组过滤 |
| `setUser({ id, email, username })` | 设置用户上下文，用于错误归因 |
| `closeSentry(timeoutMs?)` | 刷出队列并关闭客户端，进程退出时调用 |
| `isSentryInitialized()` | 检查是否已初始化 |

---

## 4. 差异判定与建议

### 4.1 差异判定

- **运行时不一致**：原始文档示例使用 `bun run dev`（Node.js 生态），而当前仓库主运行时为 **Rust**（`cargo run -p rusty-claude-cli`）。
- **实现缺失**：无论是 TypeScript 还是 Rust 侧，当前代码库均**没有** Sentry SDK 初始化、错误边界、日志 sink 或优雅退出的对应代码。
- **依赖缺失**：`Cargo.toml` 中未引入任何 Rust Sentry crate（如 `sentry`）；不存在 `package.json`，说明 TypeScript 前端层也未保留。

### 4.2 后续行动建议

1. **功能废弃**：如果项目已放弃 Sentry 支持，应同步在公开文档中移除该章节，避免误导。
2. **Rust 侧移植**：如果仍需错误上报，可在 `rust/crates/rusty-claude-cli` 或 `rust/crates/runtime` 中引入 `sentry` crate，并重新设计：
   - 在 `main.rs` 的 `main()` 入口处根据 `SENTRY_DSN` env 初始化；
   - 在 panic hook 中调用 `sentry::capture_panic`；
   - 在 CLI 退出前调用 `sentry::close(timeout)`。
3. **文档拆分**：若未来 Rust 实现落地，应新建对应于 Rust 代码路径的单页报告，替换本文档中基于 TypeScript 路径的引用。

---

## 5. 结论

原始页面描述的 Sentry 错误上报配置在当前 claw-code 代码库中**没有对应实现**。原始文档中提到的所有文件路径（`src/utils/sentry.ts`、`src/components/SentryErrorBoundary.ts` 等）均不存在。本单元报告如实记录了该差异，并提供了后续决策建议。
