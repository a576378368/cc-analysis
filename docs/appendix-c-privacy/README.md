# 附录C：隐私保护与数据处理

本附录详细说明 Claude Code Agent 设计中的隐私保护措施、数据收集范围、遥测系统设计、数据保留策略以及用户控制选项。文档基于 `src/services/analytics/` 和 `src/utils/privacyLevel.ts` 等源码的技术分析。

---

## 1. 数据收集范围

### 1.1 分析事件类型

Claude Code 通过多层次遥测系统收集分析数据。根据 `src/services/analytics/datadog.ts` 中 `DATADOG_ALLOWED_EVENTS` 的定义，主要事件类型包括：

**连接与通信事件**
- `chrome_bridge_connection_succeeded` / `chrome_bridge_connection_failed` — MCP 桥接连接状态
- `chrome_bridge_disconnected` — 桥接断开事件
- `chrome_bridge_tool_call_started` / `chrome_bridge_tool_call_completed` / `chrome_bridge_tool_call_timeout` — 工具调用生命周期

**API 调用事件**
- `tengu_api_success` / `tengu_api_error` — API 请求成功与错误
- `tengu_model_fallback_triggered` — 模型降级触发
- `tengu_oauth_*` — OAuth 认证相关事件（刷新、令牌获取等）

**用户交互事件**
- `tengu_brief_mode_enabled` / `tengu_brief_mode_toggled` — 简洁模式状态
- `tengu_flicker` — 界面闪烁事件
- `tengu_voice_recording_started` / `tengu_voice_toggled` — 语音功能

**工具使用事件**
- `tengu_tool_use_success` / `tengu_tool_use_error` — 工具执行结果
- `tengu_tool_use_granted_in_prompt_*` — 工具授权状态（永久/临时）
- `tengu_tool_use_rejected_in_prompt` — 工具拒绝

**团队协作事件**
- `tengu_team_mem_sync_*` — 团队记忆同步（pull/push/started/capped）

### 1.2 收集的环境上下文

根据 `src/services/analytics/metadata.ts` 中 `getEventMetadata()` 函数的实现，收集的环境上下文包括：

```typescript
type EnvContext = {
  platform: string           // 操作系统平台
  platformRaw: string        // 原始平台标识
  arch: string              // CPU 架构
  nodeVersion: string        // Node.js 版本
  terminal: string | null    // 终端类型
  packageManagers: string    // 已安装的包管理器
  runtimes: string          // 已安装的运行时
  isRunningWithBun: boolean // 是否使用 Bun 运行
  isCi: boolean             // 是否在 CI 环境
  isClaubbit: boolean       // 是否为 Claude Bit
  isClaudeCodeRemote: boolean   // 是否为远程会话
  isLocalAgentMode: boolean // 是否为本地 Agent 模式
  isConductor: boolean      // 是否为 Conductor
  isGithubAction: boolean   // 是否在 GitHub Actions
  isClaudeAiAuth: boolean  // 是否使用 Claude.ai 认证
  version: string           // Claude Code 版本
  versionBase?: string      // 版本基础字符串
  buildTime: string          // 构建时间戳
  deploymentEnvironment: string // 部署环境
  wslVersion?: string       // WSL 版本
  vcs?: string              // 版本控制系统
}
```

### 1.3 GrowthBook 实验属性

根据 `src/services/analytics/growthbook.ts` 的 `GrowthBookUserAttributes` 类型，还可能收集：

```typescript
type GrowthBookUserAttributes = {
  id: string              // 设备 ID
  sessionId: string       // 会话 ID
  deviceID: string        // 设备 ID（重复）
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string // API 基础 URL 主机名
  organizationUUID?: string
  accountUUID?: string
  userType?: string
  subscriptionType?: string // 订阅类型
  rateLimitTier?: string   // 速率限制等级
  firstTokenTime?: number   // 首次令牌时间戳
  email?: string
  appVersion?: string
  github?: GitHubActionsMetadata
}
```

---

## 2. 隐私保护措施

### 2.1 PII 防护机制

Claude Code 实现了多层次的 PII（个人身份信息）防护机制。

#### 2.1.1 工具名称脱敏

`src/services/analytics/metadata.ts` 中定义了 `sanitizeToolNameForAnalytics()` 函数：

```typescript
export function sanitizeToolNameForAnalytics(toolName: string): AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS {
  if (toolName.startsWith('mcp__')) {
    return 'mcp_tool' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
  }
  return toolName as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
}
```

MCP 工具名称格式为 `mcp__<server>__<tool>`，可能泄露用户配置的服务器名称，因此统一脱敏为 `mcp_tool`。

#### 2.1.2 MCP 工具详细日志控制

`isAnalyticsToolDetailsLoggingEnabled()` 函数实现了精细的 MCP 工具名称日志控制：

```typescript
export function isAnalyticsToolDetailsLoggingEnabled(
  mcpServerType: string | undefined,
  mcpServerBaseUrl: string | undefined,
): boolean {
  // Cowork 模式下记录所有 MCP 工具名称
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'local-agent') {
    return true
  }
  // Claude AI 代理连接器始终记录
  if (mcpServerType === 'claudeai-proxy') {
    return true
  }
  // 官方 MCP 注册表中的服务器记录
  if (mcpServerBaseUrl && isOfficialMcpUrl(mcpServerBaseUrl)) {
    return true
  }
  return false
}
```

#### 2.1.3 工具输入截断

`extractToolInputForTelemetry()` 函数对工具输入进行截断和限制：

```typescript
const TOOL_INPUT_STRING_TRUNCATE_AT = 512
const TOOL_INPUT_STRING_TRUNCATE_TO = 128
const TOOL_INPUT_MAX_JSON_CHARS = 4 * 1024
const TOOL_INPUT_MAX_COLLECTION_ITEMS = 20
const TOOL_INPUT_MAX_DEPTH = 2
```

#### 2.1.4 文件扩展名脱敏

`getFileExtensionForAnalytics()` 函数对文件扩展名进行脱敏处理：

```typescript
const MAX_FILE_EXTENSION_LENGTH = 10 // 超过 10 字符的扩展名视为敏感

export function getFileExtensionForAnalytics(filePath: string): AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS | undefined {
  const ext = extname(filePath).toLowerCase()
  if (!ext || ext === '.') {
    return undefined
  }
  const extension = ext.slice(1)
  if (extension.length > MAX_FILE_EXTENSION_LENGTH) {
    return 'other' as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
  }
  return extension as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS
}
```

#### 2.1.5 Proto 字段过滤

`src/services/analytics/index.ts` 中的 `stripProtoFields()` 函数移除敏感字段：

```typescript
export function stripProtoFields<V>(metadata: Record<string, V>): Record<string, V> {
  let result: Record<string, V> | undefined
  for (const key in metadata) {
    if (key.startsWith('_PROTO_')) {
      if (result === undefined) {
        result = { ...metadata }
      }
      delete result[key]
    }
  }
  return result ?? metadata
}
```

带有 `_PROTO_` 前缀的字段包含 PII 标记数据，仅限特权 BigQuery 列使用。

### 2.2 用户 ID 哈希分桶

`src/services/analytics/datadog.ts` 中的 `getUserBucket()` 函数实现用户 ID 哈希分桶：

```typescript
const NUM_USER_BUCKETS = 30

const getUserBucket = memoize((): number => {
  const userId = getOrCreateUserID()
  const hash = createHash('sha256').update(userId).digest('hex')
  return parseInt(hash.slice(0, 8), 16) % NUM_USER_BUCKETS
})
```

此设计通过将用户 ID 哈希到 30 个桶中，在保护用户隐私的同时允许聚合统计分析。

### 2.3 版本号截断

`trackDatadogEvent()` 函数对开发版本号进行截断处理：

```typescript
// 将 "2.0.53-dev.20251124.t173302.sha526cc6a" 截断为 "2.0.53-dev.20251124"
if (typeof allData.version === 'string') {
  allData.version = allData.version.replace(
    /^(\d+\.\d+\.\d+-dev\.\d{8})\.t\d+\.sha[a-f0-9]+$/,
    '$1',
  )
}
```

### 2.4 模型名称规范化

对外部用户（非 ant）的模型名称进行规范化以降低基数：

```typescript
if (process.env.USER_TYPE !== 'ant' && typeof allData.model === 'string') {
  const shortName = getCanonicalName(allData.model.replace(/\[1m]$/i, ''))
  allData.model = shortName in MODEL_COSTS ? shortName : 'other'
}
```

---

## 3. Telemetry 遥测系统

### 3.1 多后端架构

Claude Code 采用多后端遥测架构，将不同类型的数据路由到不同的后端服务。

#### 3.1.1 Datadog 遥测

`src/services/analytics/datadog.ts` 实现 Datadog 日志收集：

- **端点**: `https://http-intake.logs.us5.datadoghq.com/api/v2/logs`
- **刷新间隔**: 15 秒（可配置 `CLAUDE_CODE_DATADOG_FLUSH_INTERVAL_MS`）
- **批处理大小**: 最大 100 条日志
- **超时时间**: 5 秒
- **传输协议**: HTTP/JSON

#### 3.1.2 第一方事件日志

`src/services/analytics/firstPartyEventLogger.ts` 实现第一方事件日志：

- **目的地**: `/api/event_logging/batch`
- **使用 OpenTelemetry Logger SDK**
- **批处理配置**:
  - 调度延迟: 10 秒（默认）
  - 最大导出批次: 200
  - 最大队列: 8192

#### 3.1.3 OpenTelemetry 集成

`src/utils/telemetry/instrumentation.ts` 实现完整的 OpenTelemetry 支持：

- **指标 (Metrics)**: 通过 `MeterProvider` 和 `PeriodicExportingMetricReader`
- **日志 (Logs)**: 通过 `LoggerProvider` 和 `BatchLogRecordProcessor`
- **追踪 (Traces)**: 通过 `BasicTracerProvider` 和 `BatchSpanProcessor`
- **导出协议**: 支持 grpc、http/json、http/protobuf

### 3.2 事件采样机制

`src/services/analytics/firstPartyEventLogger.ts` 实现了事件采样机制：

```typescript
export function shouldSampleEvent(eventName: string): number | null {
  const config = getEventSamplingConfig()
  const eventConfig = config[eventName]

  if (!eventConfig) {
    return null // 无配置，全量记录
  }

  const sampleRate = eventConfig.sample_rate

  if (typeof sampleRate !== 'number' || sampleRate < 0 || sampleRate > 1) {
    return null
  }

  if (sampleRate >= 1) {
    return null // 全部记录
  }

  if (sampleRate <= 0) {
    return 0 // 全部丢弃
  }

  return Math.random() < sampleRate ? sampleRate : 0
}
```

采样配置通过 GrowthBook 动态获取（`tengu_event_sampling_config`），支持按事件类型配置不同的采样率。

### 3.3 数据流架构

```
┌─────────────────┐
│  logEvent()     │  ← 应用层调用入口
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────────────┐
│         Analytics Sink (sink.ts)            │
│  ┌──────────────┐    ┌──────────────────┐  │
│  │  Datadog     │    │  FirstParty      │  │
│  │  (datadog.ts)│    │  (firstParty...) │  │
│  └──────┬───────┘    └────────┬─────────┘  │
│         │                      │            │
└─────────┼──────────────────────┼────────────┘
          │                      │
          ▼                      ▼
   Datadog Logs API      /api/event_logging/batch
   (通用访问后端)        (特权 BigQuery 列)
```

### 3.4 第一方事件导出器可靠性

`src/services/analytics/firstPartyEventLoggingExporter.ts` 实现了高可靠性的事件导出机制：

**本地持久化**: 失败事件写入 `$CLAUDE_CONFIG_DIR/telemetry/` 目录的 JSONL 文件

**指数退避重试**:
- 基础延迟: 500ms
- 最大延迟: 30s
- 最大尝试次数: 8 次
- 重试公式: `baseBackoffDelayMs * attempts²`

**认证降级**: 401 响应后自动尝试无认证重发

---

## 4. 数据保留策略

### 4.1 隐私级别定义

`src/utils/privacyLevel.ts` 定义了三层隐私级别：

```typescript
type PrivacyLevel = 'default' | 'no-telemetry' | 'essential-traffic'

export function getPrivacyLevel(): PrivacyLevel {
  if (process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC) {
    return 'essential-traffic'
  }
  if (process.env.DISABLE_TELEMETRY) {
    return 'no-telemetry'
  }
  return 'default'
}
```

**级别说明**:

| 级别 | 描述 | 影响范围 |
|------|------|----------|
| `default` | 全部功能启用 | 所有遥测和可选网络功能 |
| `no-telemetry` | 禁用分析/遥测 | Datadog 事件、1P 事件、反馈调查 |
| `essential-traffic` | 禁用所有非必要流量 | 遥测 + 自动更新 + Grove + 发布说明 + 模型能力等 |

### 4.2 分析禁用条件

`src/services/analytics/config.ts` 中的 `isAnalyticsDisabled()` 函数：

```typescript
export function isAnalyticsDisabled(): boolean {
  return (
    process.env.NODE_ENV === 'test' ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_BEDROCK) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_VERTEX) ||
    isEnvTruthy(process.env.CLAUDE_CODE_USE_FOUNDRY) ||
    isTelemetryDisabled()
  )
}
```

**禁用条件**:
1. 测试环境 (`NODE_ENV === 'test'`)
2. 使用第三方云服务（Bedrock/Vertex/Foundry）
3. 隐私级别为 `no-telemetry` 或 `essential-traffic`

### 4.3 GrowthBook 配置缓存

`src/services/analytics/growthbook.ts` 实现特征标志缓存策略：

- **缓存位置**: `~/.claude.json` 中的 `cachedGrowthBookFeatures`
- **刷新间隔**:
  - 外部用户: 6 小时
  - 内部用户 (ant): 20 分钟
- **磁盘持久化**: 确保进程重启后立即可用

### 4.4 第一方事件文件保留

失败事件文件存储在 `$CLAUDE_CONFIG_DIR/telemetry/1p_failed_events.*.json`

- **文件命名格式**: `1p_failed_events.<sessionId>.<batchUUID>.json`
- **会话隔离**: 不同会话的事件文件分离
- **启动时重试**: 进程启动时自动重试之前失败的事件

---

## 5. 用户控制选项

### 5.1 环境变量控制

Claude Code 提供多个环境变量用于控制数据收集：

#### 5.1.1 完全禁用遥测

```bash
# 禁用所有遥测（等同于 no-telemetry 级别）
export DISABLE_TELEMETRY=1

# 禁用所有非必要网络流量（等同于 essential-traffic 级别）
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

#### 5.1.2 OpenTelemetry 配置

```bash
# 启用客户 OTLP 遥测
export CLAUDE_CODE_ENABLE_TELEMETRY=1

# 配置 OTLP 导出器
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_TRACES_EXPORTER=otlp

# 配置 OTLP 端点
export OTEL_EXPORTER_OTLP_ENDPOINT=https://your-collector:4317

# 配置 OTLP 协议
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc  # 或 http/protobuf, http/json

# 配置导出间隔（毫秒）
export OTEL_METRIC_EXPORT_INTERVAL=60000
```

#### 5.1.3 指标基数控制

```bash
# 控制指标中是否包含会话 ID
export OTEL_METRICS_INCLUDE_SESSION_ID=false

# 控制指标中是否包含版本
export OTEL_METRICS_INCLUDE_VERSION=true

# 控制指标中是否包含账户 UUID
export OTEL_METRICS_INCLUDE_ACCOUNT_UUID=true
```

#### 5.1.4 工具详情日志

```bash
# 启用工具输入详细日志（可能包含敏感信息）
export OTEL_LOG_TOOL_DETAILS=1
```

### 5.2 隐私设置命令

`src/commands/privacy-settings/privacy-settings.tsx` 实现了交互式隐私设置界面：

```typescript
// 显示 Grove 隐私设置对话框
export async function call(onDone: LocalJSXCommandOnDone): Promise<React.ReactNode | null> {
  const qualified = await isQualifiedForGrove()
  if (!qualified) {
    onDone(FALLBACK_MESSAGE)
    return null
  }
  // ...
}
```

通过 `/privacy-settings` 命令，用户可以：
- 查看当前隐私设置状态
- 启用/禁用 Grove 数据收集
- 查看数据共享政策

### 5.3 Grove 设置

`src/services/api/grove.ts` 提供 Grove 隐私控制：

```typescript
type GroveSettings = {
  grove_enabled: boolean | null  // null 表示未设置
  domain_excluded?: boolean       // 域名是否被排除
}
```

用户可以通过 `getGroveSettings()` 获取当前设置，通过 `setGroveSettings()` 更新设置。

### 5.4 服务级别killswitch

`src/services/analytics/sinkKillswitch.ts` 实现了服务端控制的分析接收器开关：

```typescript
const SINK_KILLSWITCH_CONFIG_NAME = 'tengu_frond_boric'

export type SinkName = 'datadog' | 'firstParty'

export function isSinkKilled(sink: SinkName): boolean {
  const config = getDynamicConfig_CACHED_MAY_BE_STALE<Partial<Record<SinkName, boolean>>>(SINK_KILLSWITCH_CONFIG_NAME, {})
  return config?.[sink] === true
}
```

可通过 GrowthBook 动态配置禁用特定分析后端。

---

## 6. 数据安全机制

### 6.1 认证与信任检查

`src/services/analytics/firstPartyEventLoggingExporter.ts` 实现了多层认证检查：

```typescript
const hasTrust = checkHasTrustDialogAccepted() || getIsNonInteractiveSession()

let shouldSkipAuth = this.skipAuth || !hasTrust
if (!shouldSkipAuth && isClaudeAISubscriber()) {
  const tokens = getClaudeAIOAuthTokens()
  if (!hasProfileScope()) {
    shouldSkipAuth = true
  } else if (tokens && isOAuthTokenExpired(tokens.expiresAt)) {
    shouldSkipAuth = true
  }
}
```

**信任建立条件**:
- 用户接受了信任对话框
- 会话信任已被接受
- 非交互式会话（如脚本）

### 6.2 mTLS 和代理支持

`src/utils/telemetry/instrumentation.ts` 的 `getOTLPExporterConfig()` 函数：

```typescript
function getOTLPExporterConfig() {
  const proxyUrl = getProxyUrl()
  const mtlsConfig = getMTLSConfig()
  const caCerts = getCACertificates()

  if (!proxyUrl || shouldBypassProxy(otelEndpoint)) {
    config.httpAgentOptions = {
      ...mtlsConfig,
      ...(caCerts && { ca: caCerts }),
    }
  } else {
    // 使用 HttpsProxyAgent
    config.httpAgentOptions = agentFactory
  }
}
```

### 6.3 关机时的数据刷新

`src/utils/telemetry/instrumentation.ts` 实现了优雅关闭机制：

```typescript
const shutdownTelemetry = async () => {
  const timeoutMs = parseInt(process.env.CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS || '2000')

  try {
    endInteractionSpan()
    const shutdownPromises = [meterProvider.shutdown()]
    // ... 强制刷新和关闭所有 provider
    await Promise.race([
      Promise.all(chains),
      telemetryTimeout(timeoutMs, 'OpenTelemetry shutdown timeout'),
    ])
  }
}

registerCleanup(shutdownTelemetry)
```

---

## 7. 总结

Claude Code 的隐私保护系统体现了多层防御的设计理念：

1. **数据最小化**: 工具输入截断、MCP 工具名称脱敏、文件扩展名处理
2. **隐私增强技术**: 用户 ID 哈希分桶、版本号截断、模型名称规范化
3. **细粒度控制**: 三层隐私级别、环境变量控制、服务端 killswitch
4. **可靠性保障**: 本地持久化、指数退避重试、优雅关闭
5. **透明性**: 详细的遥测配置日志、用户可访问的隐私设置界面

通过这些机制，Claude Code 在提供有用遥测数据的同时，最大程度地保护用户隐私。