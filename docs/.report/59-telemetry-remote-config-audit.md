# Unit 59 — Telemetry Remote Config Audit

> Source page: https://ccb.agent-aura.top/docs/telemetry-remote-config-audit  
> Audit scope: Rust CLI (`rust/`) + CCB TypeScript source (`packages/ccb/src/`)  
> Generated: 2026-04-09

---

## 1. Datadog 日志

**CCB TypeScript**

- 文件: `packages/ccb/src/services/analytics/datadog.ts`
- 端点与环境变量 [#L19-L22](packages/ccb/src/services/analytics/datadog.ts#L19-L22):
  - `DATADOG_LOGS_ENDPOINT` — 默认空（禁用）
  - `DATADOG_API_KEY` — 默认空（禁用）
- 行为: 批量发送，flush 间隔 15s，单次上限 100 条；仅限 1P（直连 Anthropic API）用户。
- 事件白名单 [#L35-L69](packages/ccb/src/services/analytics/datadog.ts#L35-L69):
  `tengu_api_error`、`tengu_started`、`tengu_tool_use_success` 等约 35 种 `tengu_*` 事件。
- 基线数据: model / platform / arch / version / userBucket（用户 hash 到 30 个桶）。
- 生产环境限制: `NODE_ENV === 'production'`。

**Rust CLI**

Rust CLI 当前未实现独立的 Datadog 导出器；遥测统一由 CCB 宿主进程通过 `telemetry` crate 的 `TelemetryEvent::Analytics` 消费后再路由到 Datadog。

- 相关类型: `rust/crates/telemetry/src/lib.rs` [#L134-L157](rust/crates/telemetry/src/lib.rs#L134-L157) `AnalyticsEvent`
- 路由层: `packages/ccb/src/services/analytics/sink.ts` [#L48-L71](packages/ccb/src/services/analytics/sink.ts#L48-L71)

---

## 2. 1P 事件日志（BigQuery）

**CCB TypeScript**

- 文件:
  - `packages/ccb/src/services/analytics/firstPartyEventLogger.ts`
  - `packages/ccb/src/services/analytics/firstPartyEventLoggingExporter.ts`
- 端点: `https://api.anthropic.com/api/event_logging/batch` [#L148](packages/ccb/src/services/analytics/firstPartyEventLogger.ts#L148)
- 行为: 使用 OpenTelemetry SDK 的 `BatchLogRecordProcessor`，批量导出到 Anthropic 自有 BQ 管道。
- 弹性: 本地磁盘持久化失败事件（JSONL），退避重试，最多 8 次尝试。
- Proto schema: 事件序列化为 `ClaudeCodeInternalEvent` / `GrowthbookExperimentEvent` protobuf 格式。
- Auth fallback: 401 时自动去掉 auth header 重试。

**Rust CLI**

- Rust 通过 `telemetry` crate 生成结构化事件，由 CCB 侧完成批量导出。
- `rust/crates/telemetry/src/lib.rs` [#L205-L231](rust/crates/telemetry/src/lib.rs#L205-L231) `TelemetrySink` trait 与 `MemoryTelemetrySink`。

---

## 3. GrowthBook 远程 Feature Flags / 动态配置

**CCB TypeScript**

- 文件: `packages/ccb/src/services/analytics/growthbook.ts`
- 服务端: `https://api.anthropic.com/`（remote eval 模式）。
- 磁盘缓存 [#L410-L415](packages/ccb/src/services/analytics/growthbook.ts#L410-L415):
  feature values 写入 `~/.claude.json` 的 `cachedGrowthBookFeatures`。
- 刷新策略: 外部用户 6h / ant 20min。
- 典型用途（代码中显式引用）:
  - `tengu_log_datadog_events` — Datadog 开关
  - `tengu_event_sampling_config` — 事件采样率
  - `tengu_frond_boric` — sink killswitch
  - `tengu_1p_event_batch_config` — BQ batch 配置
  - `tengu_max_version_config` / 版本上限 kill switch
  - 远程管理设置的安全检查 gate
- 用户属性 [#L34-L38](packages/ccb/src/services/analytics/growthbook.ts#L34-L38):
  `deviceId`, `sessionId`, `organizationUUID`, `accountUUID`, `email`, `subscriptionType` 等。

**Rust CLI**

- Rust CLI 尚未实现独立的 GrowthBook 客户端。运行时特性通过 CCB 传递的配置注入。
- 注意: `rust/crates/runtime/src/config.rs` [#L1970-L1990](rust/crates/runtime/src/config.rs#L1970-L1990) 的验证测试显式拒绝未知顶层键 `"telemetry"`，说明 Rust 配置层目前不直接暴露 telemetry 开关，而是走 CCB 侧配置。

---

## 4. Remote Managed Settings（企业远程配置下发）

**CCB TypeScript**

- 文件: `packages/ccb/src/services/remoteManagedSettings/index.ts`
- 端点 [#L106](packages/ccb/src/services/remoteManagedSettings/index.ts#L106):
  `{BASE_API_URL}/api/claude_code/settings`
- 行为:
  - 支持 ETag/304 缓存 [#L273-L289](packages/ccb/src/services/remoteManagedSettings/index.ts#L273-L289)
  - 每小时后台轮询
  - 变更包含“危险设置”时弹窗让用户确认
  - Fail-open: 请求失败时使用本地缓存，无缓存则跳过
- 适用 [#L10-L11](packages/ccb/src/services/remoteManagedSettings/index.ts#L10-L11):
  - API key 用户全部可拉取
  - OAuth 用户仅 Enterprise / C4E / Team

---

## 5. Settings Sync（设置同步）

**CCB TypeScript**

- 文件: `packages/ccb/src/services/settingsSync/index.ts`
- 端点: `{BASE_API_URL}/api/claude_code/user_settings`
- 行为: CLI 上传本地设置/memory 到远程；CCR 模式从远程下载。
- 同步内容 [#L424-L564](packages/ccb/src/services/settingsSync/index.ts#L424-L564):
  `userSettings`、`userMemory`、`projectSettings`、`projectMemory`
- Feature gate [#L63](packages/ccb/src/services/settingsSync/index.ts#L63):
  `UPLOAD_USER_SETTINGS` / `DOWNLOAD_USER_SETTINGS`
- 大小限制: 500KB/文件（通过 `exceedsSizeLimit` 检查）。

---

## 6. OpenTelemetry 三方遥测

**CCB TypeScript**

- 文件: `packages/ccb/src/utils/telemetry/instrumentation.ts`
- 行为: 完整的 OTEL SDK 初始化，支持 metrics / logs / traces 三种信号。
- 协议: gRPC / http-json / http-protobuf（通过 `OTEL_EXPORTER_OTLP_PROTOCOL` 选择）。
- Exporter: console / otlp / prometheus。
- 触发:
  - `CLAUDE_CODE_ENABLE_TELEMETRY=1`（或等价的 ant 内部变量）
  - `feature('ENHANCED_TELEMETRY_BETA')` + GrowthBook gate `enhanced_telemetry_beta`。
- 环境变量透见 [#L90-L108](packages/ccb/src/utils/telemetry/instrumentation.ts#L90-L108):
  `ANT_OTEL_METRICS_EXPORTER`、`ANT_OTEL_LOGS_EXPORTER`、`ANT_OTEL_TRACES_EXPORTER`、`ANT_OTEL_EXPORTER_OTLP_PROTOCOL`、`ANT_OTEL_EXPORTER_OTLP_ENDPOINT`、`ANT_OTEL_EXPORTER_OTLP_HEADERS`。

**Rust CLI**

- Rust 当前未集成 OTEL SDK；遥测由 `telemetry` crate 以应用自定义事件协议输出，由宿主层转换为 OTEL。

---

## 7. BigQuery Metrics Exporter（内部指标）

**CCB TypeScript**

- 文件: `packages/ccb/src/utils/telemetry/bigqueryExporter.ts`
- 端点 [#L47](packages/ccb/src/utils/telemetry/bigqueryExporter.ts#L47):
  `https://api.anthropic.com/api/claude_code/metrics`
- 行为: 定期（默认 5min 间隔）导出 OTel metrics 到内部 BQ。
- 适用: API 客户、C4E/Team 订阅者。
- 组织级 opt-out 控制 [#L104-L106](packages/ccb/src/utils/telemetry/bigqueryExporter.ts#L104-L106):
  `checkMetricsEnabled()` 返回 false 时跳过导出。

---

## 8. 组织级 Metrics Opt-out 查询

**CCB TypeScript**

- 文件: `packages/ccb/src/services/api/metricsOptOut.ts`
- 端点 [#L45](packages/ccb/src/services/api/metricsOptOut.ts#L45):
  `https://api.anthropic.com/api/claude_code/organizations/metrics_enabled`
- 缓存策略 [#L121-L147](packages/ccb/src/services/api/metricsOptOut.ts#L121-L147):
  - 内存缓存 1h
  - 磁盘缓存 24h（写入 `~/.claude.json` 的 `metricsStatusCache`）
- 作用: 控制 BigQuery metrics exporter 是否导出。

---

## 9. Startup Profiling

**CCB TypeScript**

- 文件: `packages/ccb/src/utils/startupProfiler.ts`
- 行为: 采样启动性能数据。
- 采样率: 100% ant / 0.5% 外部用户。
- 上报事件: `tengu_startup_perf` [#L191](packages/ccb/src/utils/startupProfiler.ts#L191)
- 详细模式: `CLAUDE_CODE_PROFILE_STARTUP=1` 输出完整性能报告到文件。
- 采样决策 [#L26-L35](packages/ccb/src/utils/startupProfiler.ts#L26-L35):
  模块加载时一次性决定，未命中用户零开销。

---

## 10. Beta Session Tracing

**CCB TypeScript**

- 文件: `packages/ccb/src/utils/telemetry/betaSessionTracing.ts`
- 行为: 详细调试 trace，发送 system prompt、model output、tool schema 等。
- 触发条件 [#L74-L81](packages/ccb/src/utils/telemetry/betaSessionTracing.ts#L74-L81):
  - `ENABLE_BETA_TRACING_DETAILED=1` + `BETA_TRACING_ENDPOINT`
  - 外部用户交互模式需要 GrowthBook gate `tengu_trace_lantern`
  - SDK/headless 模式自动启用
- 去重策略 [#L42-L44](packages/ccb/src/utils/telemetry/betaSessionTracing.ts#L42-L44):
  按 hash 追踪，同一个 session 内 system prompt / tool schema 仅发送一次。

---

## 11. Bridge Poll Config（远程轮询间隔配置）

**CCB TypeScript**

- 文件: `packages/ccb/src/bridge/pollConfigDefaults.ts`
- 行为: 从 GrowthBook 拉取 `tengu_bridge_poll_interval_config`。
- 控制项 [#L45-L70](packages/ccb/src/bridge/pollConfigDefaults.ts#L45-L70):
  - 单会话: `poll_interval_ms_not_at_capacity`、`poll_interval_ms_at_capacity`
  - 多会话: `multisession_poll_interval_ms_not_at_capacity`、`multisession_poll_interval_ms_partial_capacity`、`multisession_poll_interval_ms_at_capacity`

---

## 12. Plugin / MCP 遥测

**CCB TypeScript**

- 文件: `packages/ccb/src/utils/plugins/fetchTelemetry.ts`
- 事件: `tengu_plugin_remote_fetch` [#L88](packages/ccb/src/utils/plugins/fetchTelemetry.ts#L88)
- 数据: host（已脱敏）、outcome、duration_ms。
- Host 脱敏规则 [#L31-L73](packages/ccb/src/utils/plugins/fetchTelemetry.ts#L31-L73):
  只允许白名单中的公共域名（如 `github.com`），其余映射为 `other`，本地路径映射为 `unknown`。

---

## 13. 全局禁用方式

**CCB TypeScript**

- 文件: `packages/ccb/src/utils/privacyLevel.ts`
- 三个隐私级别 [#L18-L27](packages/ccb/src/utils/privacyLevel.ts#L18-L27):
  1. `default` — 全部启用
  2. `no-telemetry` — `DISABLE_TELEMETRY=1`（禁用 Datadog + 1P 事件 + 调查问卷）
  3. `essential-traffic` — `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`（同时禁用自动更新、grove、release notes 等）
- 3P 提供商自动禁用: `CLAUDE_CODE_USE_BEDROCK=1`、`VERTEX`、`FOUNDRY`。

---

## 14. Rust CLI 遥测实现现状

### 14.1 telemetry crate

- 文件: `rust/crates/telemetry/src/lib.rs`
- 核心类型:
  - `TelemetryEvent` 枚举 [#L170-L203](rust/crates/telemetry/src/lib.rs#L170-L203):
    `HttpRequestStarted`、`HttpRequestSucceeded`、`HttpRequestFailed`、`Analytics`、`SessionTrace`
  - `TelemetrySink` trait [#L205-L207](rust/crates/telemetry/src/lib.rs#L205-L207)
  - `MemoryTelemetrySink` / `JsonlTelemetrySink` [#L209-L277](rust/crates/telemetry/src/lib.rs#L209-L277)
  - `SessionTracer` [#L279-L407](rust/crates/telemetry/src/lib.rs#L279-L407)
    - `record_http_request_started`
    - `record_http_request_succeeded`
    - `record_http_request_failed`
    - `record_analytics`

### 14.2 API 层的 HTTP 遥测

- 文件: `rust/crates/api/src/providers/anthropic.rs`
- `AnthropicClient` 包含可选的 `session_tracer: Option<SessionTracer>` [#L122](rust/crates/api/src/providers/anthropic.rs#L122)
- Builder: `with_session_tracer` [#L215-L220](rust/crates/api/src/providers/anthropic.rs#L215-L220)
- 请求生命周期打点:
  - started [#L410-L417](rust/crates/api/src/providers/anthropic.rs#L410-L417)
  - succeeded [#L421-L430](rust/crates/api/src/providers/anthropic.rs#L421-L430)
  - failed [#L545-L557](rust/crates/api/src/providers/anthropic.rs#L545-L557)
- `message_usage` analytics 在响应成功后记录 [#L314-L339](rust/crates/api/src/providers/anthropic.rs#L314-L339)

### 14.3 ConversationRuntime 的 Session Trace

- 文件: `rust/crates/runtime/src/conversation.rs`
- `ConversationRuntime` 包含可选 `session_tracer` [#L138](rust/crates/runtime/src/conversation.rs#L138)
- Builder: `with_session_tracer` [#L219-L224](rust/crates/runtime/src/conversation.rs#L219-L224)
- 记录的事件:
  - `turn_started` [#L547-L556](rust/crates/runtime/src/conversation.rs#L547-L556)
  - `assistant_iteration_completed` [#L565-L579](rust/crates/runtime/src/conversation.rs#L565-L579)
  - `tool_execution_started` [#L583-L593](rust/crates/runtime/src/conversation.rs#L583-L593)
  - `tool_execution_finished` [#L597-L617](rust/crates/runtime/src/conversation.rs#L597-L617)
  - `turn_completed` [#L618-L639](rust/crates/runtime/src/conversation.rs#L618-L639)
  - `turn_failed` [#L643-L654](rust/crates/runtime/src/conversation.rs#L643-L654)

### 14.4 集成测试中的遥测断言

- 文件: `rust/crates/api/tests/client_integration.rs`
- 测试 `send_message_applies_request_profile_and_records_telemetry` [#L143-L241](rust/crates/api/tests/client_integration.rs#L143-L241)
  断言了 `HttpRequestStarted`、`HttpRequestSucceeded`、`Analytics(message_usage)`、`SessionTrace` 的完整序列。

- 文件: `rust/crates/runtime/src/conversation.rs`（tests 模块）
- 测试 `records_runtime_session_trace_events` [#L935-L967](rust/crates/runtime/src/conversation.rs#L935-L967)
  验证了 `turn_started` -> `assistant_iteration_completed` -> `tool_execution_started` -> `tool_execution_finished` -> `turn_completed` 全链路。

---

## 15. 数据流架构

```
用户操作 / API 调用
    |
    v
logEvent() / SessionTracer::record_*()
    |
    v
+-------------------+
|   sink.ts (TS)    |   <-- 或 TelemetrySink (Rust)
+-------------------+
    |              |
    v              v
 trackDatadog    logEventTo1P
    |              |
    v              v
 Datadog         BQ (OTEL BatchLogRecordProcessor)
```

Rust CLI 侧的数据流:

```
AnthropicClient::send_with_retry()
    |
    v
SessionTracer::record_http_request_started/succeeded/failed
    |
    v
TelemetrySink (Memory / Jsonl)
    |
    v
CCB 宿主进程消费并路由到 Datadog / BQ / OTEL
```

---

## 16. 缺失与差异总结

| 能力 | CCB TypeScript | Rust CLI |
|------|----------------|----------|
| Datadog 日志 | 完整实现 | 未实现（由宿主层路由） |
| 1P BQ 事件 | 完整实现 | 未实现（由宿主层路由） |
| GrowthBook flags | 完整实现 | 未实现 |
| Remote Managed Settings | 完整实现 | 未实现 |
| Settings Sync | 完整实现 | 未实现 |
| OTEL SDK 初始化 | 完整实现 | 未实现 |
| BigQuery Metrics Exporter | 完整实现 | 未实现 |
| Metrics Opt-out | 完整实现 | 未实现 |
| Startup Profiling | 完整实现 | 未实现 |
| Beta Session Tracing | 完整实现 | 未实现 |
| Bridge Poll Config | 完整实现 | N/A（无 bridge） |
| Plugin/MCP 遥测 | 完整实现 | 未实现（无 plugin 市场） |
| Privacy Level 控制 | 完整实现 | 未实现 |
| 基础 SessionTracer | 消费端在 CCB | `telemetry` crate 提供 |

Rust CLI (`rust/`) 当前仅提供了基础遥测类型和内存/JSONL sink，尚未接入任何外部遥测后端。所有高级遥测、远程配置、动态 Feature Flags 均保留在 CCB TypeScript 侧实现。
