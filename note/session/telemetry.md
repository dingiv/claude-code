# 遥测与可观测性系统

## 一、初始化流水线

遥测初始化分为 5 个阶段，与信任对话框紧密耦合：

### Phase 1: 早期初始化（信任前）

`init()`（`src/entrypoints/init.ts`）在启动时运行：

- `initSentry()` — 仅在 `SENTRY_DSN` 环境变量存在时激活，否则 no-op
- `initLangfuse()` — 仅在 `LANGFUSE_PUBLIC_KEY` + `LANGFUSE_SECRET_KEY` 存在时激活
- 动态导入 `firstPartyEventLogger.js` 启动 1P 事件日志

**注意：OpenTelemetry 此时不会初始化**，延迟到信任确认后。

### Phase 2: 信任后初始化

`showSetupScreens()`（`src/interactiveHelpers.tsx`）在用户信任目录后：

1. `resetGrowthBook()` + `initializeGrowthBook()` — 用新的认证 headers 重新初始化
2. `applyConfigEnvironmentVariables()` — 应用全部环境变量（含远程管理的 OTEL 设置）
3. `setImmediate(() => initializeTelemetryAfterTrust())` — 延迟到下一个 tick，不阻塞渲染

### Phase 3: OpenTelemetry 启动

`initializeTelemetryAfterTrust()`：
- 如果远程托管设置可用，先等待加载完成再重新应用环境变量
- 懒加载 `instrumentation.ts` 并调用 `initializeTelemetry()`
- `telemetryInitialized` 标志防止重复初始化

### Phase 4: Analytics Sink 挂载

`initializeAnalyticsSink()` 挂载 sink 实现，路由事件到 Datadog 和 1P 日志。

### Phase 5: GrowthBook 初始化

`initializeGrowthBook()` 创建客户端，从 `api.anthropic.com` 远程拉取 feature flags：
- 内部 "ant" 用户：每 20 分钟刷新
- 外部用户：每 6 小时刷新
- 缓存到磁盘 `~/.claude.json` 的 `cachedGrowthBookFeatures`

---

## 二、事件体系

代码库中共有 **697 个唯一事件名**，全部以 `tengu_` 为前缀（除 `analytics_sink_attached`）。

### 事件分类

| 类别 | 示例事件 |
|------|---------|
| API/Model | `tengu_api_query`, `tengu_api_error`, `tengu_api_success`, `tengu_api_retry` |
| Tool Use | `tengu_tool_use_success`, `tengu_tool_use_error`, `tengu_tool_use_granted_by_permission_hook` |
| Session Lifecycle | `tengu_started`, `tengu_exit`, `tengu_cancel`, `tengu_init`, `tengu_session_resumed` |
| OAuth/Auth | `tengu_oauth_success`, `tengu_oauth_error`, `tengu_oauth_token_refresh_success` |
| Bridge/Remote | `tengu_bridge_started`, `tengu_bridge_repl_started`, `tengu_bridge_session_done` |
| MCP | `tengu_mcp_add`, `tengu_mcp_server_connection_failed/succeeded` |
| Memory/Context | `tengu_session_memory_loaded`, `tengu_extract_memories_extraction` |
| Permissions | `tengu_auto_mode_decision`, `tengu_auto_mode_outcome` |
| Voice | `tengu_voice_recording_started`, `tengu_voice_toggled` |
| Plugins | `tengu_plugins_loaded`, `tengu_plugin_installed` |
| Compact | `tengu_compact`, `tengu_auto_compact_succeeded` |
| Files | `tengu_file_operation`, `tengu_file_changed` |

### 事件提交流水线

```
logEvent(eventName, metadata)
    │
    ├─ Sink 未挂载？ → 加入 eventQueue[] 等待
    │
    ▼
sink.logEvent()
    │
    ├─ shouldSampleEvent() — 采样/丢弃
    │
    ├─ Datadog 路径（tengu_log_datadog_events gate 开启时）
    │   → trackDatadogEvent() → HTTPS POST 到 DATADOG_LOGS_ENDPOINT
    │   → 仅 ~30 个白名单事件
    │
    └─ 1P 路径（始终启用，除非 killswitched）
        → OTel LoggerProvider
        → BatchLogRecordProcessor
        → FirstPartyEventLoggingExporter
        → HTTPS POST 到 api.anthropic.com/api/event_logging/batch
        → 失败时写入 ~/.claude/telemetry/1p_failed_events.{sessionId}.{batchUUID}.json
        → 二次退避重试（base * attempts²），最多 8 次
```

---

## 三、Token 计数

两套并行系统：

### OTel Counters（BigQuery 指标）

`src/bootstrap/state.ts` 定义：

| Counter | Attributes |
|---------|-----------|
| `claude_code.token.usage` | model, type=[input\|output\|cacheRead\|cacheCreation] |
| `claude_code.cost.usage` | model, unit: USD |
| `claude_code.session.count` | — |
| `claude_code.lines_of_code.count` | — |
| `claude_code.commit.count` | — |
| `claude_code.pull_request.count` | — |
| `claude_code.active_time.total` | — |

### Cost Tracker（会话级显示）

`src/cost-tracker.ts` 的 `addToTotalSessionCost()`：

- 按模型追踪 `ModelUsage`：inputTokens、outputTokens、cacheReadInputTokens、cacheCreationInputTokens、webSearchRequests、costUSD
- 通过 `calculateUSDCost()`（`src/utils/modelCost.ts` 的 `MODEL_COSTS`）计算美元成本
- 会话成本持久化到项目配置（用于 `--continue` 恢复）

### Langfuse Token 追踪

`src/services/langfuse/tracing.ts` 的 `recordLLMObservation()`：
- 分别记录 `input_tokens`、`cache_creation_input_tokens`、`cache_read_input_tokens`、`output_tokens`

---

## 四、会话统计

| 维度 | 实现方式 |
|------|---------|
| API 耗时 | `getTotalAPIDuration()` / `getTotalAPIDurationWithoutRetries()` |
| 总耗时 | `getTotalDuration()`（挂钟时间） |
| 工具耗时 | `getTotalToolDuration()` |
| 工具使用 | `tengu_tool_use_success/error` 事件 + `toolName` 属性 |
| 成本 | `getTotalCostUSD()` + 按模型 `getModelUsage()` |
| 代码变更 | `getTotalLinesAdded()` / `getTotalLinesRemoved()` |

成本持久化：保存到项目配置的 `lastCost`、`lastAPIDuration`、`lastModelUsage`、`lastSessionId`。

---

## 五、四大数据出口

| Sink | 目标 | 控制方式 | 数据格式 |
|------|------|---------|---------|
| **Datadog** | `DATADOG_LOGS_ENDPOINT` | GrowthBook gate + `DATADOG_API_KEY` | JSON via HTTPS |
| **1P Event Logging** | `api.anthropic.com/api/event_logging/batch` | 始终启用（除非 killswitched） | Protobuf JSON |
| **BigQuery Metrics** | `api.anthropic.com/api/claude_code/metrics` | 1P API 客户 + C4E/Teams | OTel 自定义 JSON |
| **Customer OTLP** | `OTEL_EXPORTER_OTLP_ENDPOINT` | `CLAUDE_CODE_ENABLE_TELEMETRY` | 标准 OTLP |

**Sink Killswitch**（`tengu_frond_boric` 配置）：
```json
{ "datadog": true, "firstParty": true }
```
值为 `true` 停止向该 sink 派发。

**事件采样**（`tengu_event_sampling_config`）：
- 每事件独立采样率（0-1），通过 GrowthBook 动态配置
- 未配置的事件 100% 记录

---

## 六、PII 保护（12 层纵深防御）

| 层级 | 机制 | 说明 |
|------|------|------|
| 1 | 类型系统强制 | `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`（`never` 类型） |
| 2 | 元数据类型限制 | `logEvent()` metadata 类型为 `{ [key: string]: boolean \| number \| undefined }`，排除 string |
| 3 | `_PROTO_` 字段分离 | Datadog 导出前 `stripProtoFields()` 移除所有 `_PROTO_*` 键；仅 1P 导出器可见 |
| 4 | 工具名脱敏 | MCP 工具名 `mcp__<server>__<tool>` → `mcp_tool`；内置工具名保留 |
| 5 | 文件扩展名提取 | 仅提取扩展名（如 `.ts`），超 10 字符替换为 `other` |
| 6 | Bash 命令脱敏 | 仅从白名单命令（rm, mv, cp 等）提取扩展名 |
| 7 | 工具输入截断 | 字符串截断 512 字符，JSON 限制 4KB，嵌套深度 2，集合最多 20 项 |
| 8 | Datadog 基数降低 | 模型名规范化，Dev 版本截断为 base+date，用户 ID 哈希为 30 桶 |
| 9 | 隐私级别 | `default` / `no-telemetry` / `essential-traffic` 三级 |
| 10 | 分析禁用条件 | 第三方 provider（Bedrock/Vertex/Foundry）、test 环境、隐私级别 |
| 11 | Langfuse 脱敏 | 主目录 → `~`，敏感 key → `[REDACTED]`，文件工具输出完全脱敏 |
| 12 | Sentry 过滤 | 剥离 auth headers，不发送性能事务 |

### 分析完全禁用的条件

```typescript
// src/services/analytics/config.ts — isAnalyticsDisabled()
NODE_ENV === 'test'
|| CLAUDE_CODE_USE_BEDROCK
|| CLAUDE_CODE_USE_VERTEX
|| CLAUDE_CODE_USE_FOUNDRY
|| isTelemetryDisabled()  // privacy level === 'no-telemetry' || 'essential-traffic'
```

---

## 七、Sentry 错误追踪

`src/utils/sentry.ts`：

- **仅 `SENTRY_DSN` 环境变量存在时激活**
- `sampleRate: 1.0`（捕获所有错误）
- `maxBreadcrumbs: 20`
- `beforeSend` 过滤 auth headers（authorization, x-api-key, cookie, set-cookie）
- `beforeSendTransaction` 返回 `null` — **不发送性能事务**（仅错误）
- 忽略：ECONNREFUSED、ECONNRESET、ENOTFOUND、ETIMEDOUT、AbortError、CancelError

---

## 八、Langfuse 可观测性

`src/services/langfuse/` 目录，基于 `@langfuse/otel` 集成。

**激活条件**：`LANGFUSE_PUBLIC_KEY` + `LANGFUSE_SECRET_KEY` 环境变量。

**捕获内容**：

| 类型 | 函数 | 说明 |
|------|------|------|
| Root Trace | `createTrace()` | Agent 运行，命名 `agent-run` 或 `agent-run:{querySource}` |
| LLM Generation | `recordLLMObservation()` | 每次 API 调用：model、input/output、token、TTFT |
| Tool Observation | `recordToolObservation()` | 单个工具调用（脱敏后的输入/输出） |
| Tool Batch Span | `createToolBatchSpan()` | 并发工具调用组 |
| Sub-agent Trace | `createSubagentTrace()` | 嵌套 agent 运行 |
| Child Span | `createChildSpan()` | 主 trace 下的子查询 |

---

## 九、GrowthBook Feature Flags

`src/services/analytics/growthbook.ts`：

**Gate 检查函数**：

| 函数 | 行为 |
|------|------|
| `getFeatureValue_CACHED_MAY_BE_STALE()` | 即时返回，从内存/磁盘缓存 |
| `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` | Statsig 迁移兼容 |
| `checkSecurityRestrictionGate()` | 初始化中则等待 |
| `checkGate_CACHED_OR_BLOCKING()` | 缓存=true 快速路径，否则阻塞等待初始化 |

**LOCAL_GATE_DEFAULTS**：当 GrowthBook 未连接时（如无 1P 事件日志），使用硬编码默认值。约 40+ 个 feature flag 默认 `true`，按优先级分 P0（纯本地功能）、P1（API 依赖功能）、KS（Kill Switch）。

**动态配置**（GrowthBook features 带 object 值）：
- `tengu_1p_event_batch_config` — 批量大小、延迟、端点
- `tengu_event_sampling_config` — 每事件采样率
- `tengu_frond_boric` — 每 sink killswitch 配置

---

## 十、穷鬼模式（Budget Mode）对遥测的影响

`/poor` 命令不直接禁用遥测，而是跳过消耗 token 的操作（间接减少 API 调用）：

| 跳过的操作 | 位置 |
|-----------|------|
| 记忆提取（extract_memories） | `sessionMemory.ts:287` |
| 自动记忆（auto memory） | `query/stopHooks.ts` |
| 权限解释器用廉价模型 | `permissionExplainer.ts:178` |
| YOLO 分类器用廉价模型 | `yoloClassifier.ts:1358` |
| Agent 摘要 | `agentSummary.ts:72` |
| 附件处理缩减 | `attachments.ts:2405` |
| 搜索系统提示段落省略 | `prompts.ts:484` |

---

## 十一、关键文件索引

| 用途 | 文件路径 |
|------|---------|
| Analytics 公共 API | `src/services/analytics/index.ts` |
| Sink 路由（Datadog + 1P） | `src/services/analytics/sink.ts` |
| Analytics 配置/禁用检查 | `src/services/analytics/config.ts` |
| Datadog sink | `src/services/analytics/datadog.ts` |
| 1P 事件日志 | `src/services/analytics/firstPartyEventLogger.ts` |
| 1P 导出器 | `src/services/analytics/firstPartyEventLoggingExporter.ts` |
| GrowthBook | `src/services/analytics/growthbook.ts` |
| Sink Killswitch | `src/services/analytics/sinkKillswitch.ts` |
| 事件元数据 | `src/services/analytics/metadata.ts` |
| 隐私级别 | `src/utils/privacyLevel.ts` |
| Sentry | `src/utils/sentry.ts` |
| OTel Instrumentation | `src/utils/telemetry/instrumentation.ts` |
| 会话 Tracing | `src/utils/telemetry/sessionTracing.ts` |
| BigQuery 导出器 | `src/utils/telemetry/bigqueryExporter.ts` |
| Langfuse Client | `src/services/langfuse/client.ts` |
| Langfuse Tracing | `src/services/langfuse/tracing.ts` |
| Langfuse 脱敏 | `src/services/langfuse/sanitize.ts` |
| Cost Tracker | `src/cost-tracker.ts` |
| Poor Mode | `src/commands/poor/poorMode.ts` |

## 十二、核心设计思想

1. **信任前沉默** — OpenTelemetry 和 GrowthBook 都在用户信任目录后才初始化，避免未授权的遥测
2. **纵深 PII 防护** — 12 层保护从类型系统到运行时过滤，确保代码路径、文件路径、用户数据不泄露
3. **多出口并行** — Datadog（运维监控）、1P（产品分析）、BigQuery（指标）、OTLP（客户自定义）四路并行，各自独立控制
3. **Graceful Degradation** — 网络失败时写入本地文件、第三方 provider 自动禁用分析、缓存允许离线运行
4. **采样而非丢弃** — 高频事件通过 GrowthBook 动态配置采样率，而非简单丢弃
5. **穷鬼模式省的是 token 而非遥测** — `/poor` 减少的是消耗 token 的辅助操作，核心遥测管道不受影响
