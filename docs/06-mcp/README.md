# 第六章：MCP 协议实现

## 目录

1. [MCP 协议概述](#1-mcp-协议概述)
2. [MCP 客户端架构](#2-mcp-客户端架构)
3. [传输层实现](#3-传输层实现)
4. [OAuth 认证流程](#4-oauth-认证流程)
5. [工具与资源管理](#5-工具与资源管理)
6. [MCP 服务器生命周期](#6-mcp-服务器生命周期)

---

## 1. MCP 协议概述

### 1.1 什么是 Model Context Protocol

Model Context Protocol (MCP) 是一种标准化协议，用于在大型语言模型（LLM）和外部数据源、工具之间建立连接。MCP 采用客户端-服务器架构，其中 Claude Code 作为客户端，通过各种传输机制连接到 MCP 服务器。

### 1.2 Claude Code 中的 MCP 集成

Claude Code 通过 `@modelcontextprotocol/sdk` 实现 MCP 客户端功能，支持多种服务器类型和传输协议。

#### 核心组件

```
src/services/mcp/
├── client.ts              # MCP 客户端核心实现
├── auth.ts                # OAuth 认证流程
├── config.ts              # 配置管理
├── types.ts               # 类型定义
├── utils.ts               # 工具函数
├── xaa.ts                 # Cross-App Access (SEP-990)
├── channelNotification.ts  # 渠道通知处理
├── channelPermissions.ts   # 渠道权限管理
├── claudeai.ts             # Claude.ai 代理集成
├── elicitationHandler.ts   # 用户交互处理
├── envExpansion.ts         # 环境变量展开
├── headersHelper.ts        # 动态请求头
└── SdkControlTransport.ts  # SDK 控制传输
```

#### 支持的服务器类型

| 类型 | 描述 | 传输协议 |
|------|------|----------|
| `stdio` | 标准输入/输出本地进程 | StdioClientTransport |
| `sse` | Server-Sent Events 远程服务器 | SSEClientTransport |
| `http` | Streamable HTTP 远程服务器 | StreamableHTTPClientTransport |
| `ws` | WebSocket 服务器 | WebSocketTransport |
| `sse-ide` | IDE 扩展 SSE 连接 | SSEClientTransport |
| `ws-ide` | IDE 扩展 WebSocket | WebSocketTransport |
| `sdk` | SDK 管理的传输 | SdkControlClientTransport |
| `claudeai-proxy` | Claude.ai 代理服务器 | StreamableHTTPClientTransport |

---

## 2. MCP 客户端架构

### 2.1 连接管理

Claude Code 使用 `connectToServer` 函数建立与 MCP 服务器的连接，采用记忆化 (memoization) 优化重复连接。

```typescript
// 核心连接函数签名
export const connectToServer = memoize(
  async (
    name: string,
    serverRef: ScopedMcpServerConfig,
    serverStats?: ServerStats,
  ): Promise<MCPServerConnection>
)
```

#### 服务器连接状态

```typescript
type MCPServerConnection =
  | ConnectedMCPServer    // 已连接
  | FailedMCPServer      // 连接失败
  | NeedsAuthMCPServer   // 需要认证
  | PendingMCPServer     // 连接中/重连中
  | DisabledMCPServer   // 已禁用
```

### 2.2 服务发现

服务发现通过多层级配置系统实现，支持配置优先级和继承：

```
配置优先级（高 → 低）:
enterprise > local > project > user > plugin
```

#### 配置作用域

```typescript
export const ConfigScopeSchema = z.enum([
  'local',      // 项目本地配置
  'user',       // 用户全局配置
  'project',    // 项目共享配置 (.mcp.json)
  'dynamic',    // 动态配置（命令行）
  'enterprise', // 企业托管配置
  'claudeai',   // Claude.ai 组织配置
  'managed',    // 托管配置
])
```

### 2.3 客户端初始化

```typescript
const client = new Client(
  {
    name: 'claude-code',
    title: 'Claude Code',
    version: MACRO.VERSION,
    description: "Anthropic's agentic coding tool",
    websiteUrl: PRODUCT_URL,
  },
  {
    capabilities: {
      roots: {},          // 根目录声明
      elicitation: {},   // 用户交互能力
    },
  },
)
```

---

## 3. 传输层实现

### 3.1 传输层架构

Claude Code 支持四种主要传输协议，每种针对不同的使用场景优化：

```
┌─────────────────────────────────────────────────────────┐
│                    Claude Code MCP Client                │
├─────────────────────────────────────────────────────────┤
│  StdioClientTransport  │  用于本地子进程通信              │
│  SSEClientTransport    │  用于 HTTP + SSE 远程服务器      │
│  StreamableHTTPClientTransport │ 用于 HTTP Streamable   │
│  WebSocketTransport    │  用于 WebSocket 连接            │
└─────────────────────────────────────────────────────────┘
           │                    │                    │
           ▼                    ▼                    ▼
    ┌──────────┐          ┌──────────┐          ┌──────────┐
    │  stdio   │          │   SSE    │          │   HTTP   │
    │  子进程   │          │ HTTP+Events │        │ Streamable│
    └──────────┘          └──────────┘          └──────────┘
```

### 3.2 Stdio 传输

Stdio 传输用于本地 MCP 服务器，通过子进程stdin/stdout通信：

```typescript
transport = new StdioClientTransport({
  command: finalCommand,
  args: finalArgs,
  env: {
    ...subprocessEnv(),
    ...serverRef.env,
  } as Record<string, string>,
  stderr: 'pipe',  // 捕获错误输出用于调试
})
```

**特点：**
- 低延迟，适合本地服务
- 自动环境变量传递
- stderr 管道用于日志记录
- 支持 shell 前缀包装

### 3.3 SSE 传输

Server-Sent Events (SSE) 传输用于需要长连接的远程服务器：

```typescript
const transportOptions: SSEClientTransportOptions = {
  authProvider,
  fetch: wrapFetchWithTimeout(
    wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
  ),
  requestInit: {
    headers: {
      'User-Agent': getMCPUserAgent(),
      ...combinedHeaders,
    },
  },
}
```

**EventSource 配置：**

```typescript
transportOptions.eventSourceInit = {
  fetch: async (url, init) => {
    const authHeaders: Record<string, string> = {}
    const tokens = await authProvider.tokens()
    if (tokens) {
      authHeaders.Authorization = `Bearer ${tokens.access_token}`
    }
    return fetch(url, {
      ...init,
      headers: {
        'User-Agent': getMCPUserAgent(),
        ...authHeaders,
        ...combinedHeaders,
        Accept: 'text/event-stream',
      },
    })
  },
}
```

**重要：** EventSource 连接是长连接的，不使用超时包装，避免 60 秒后自动断开。

### 3.4 StreamableHTTP 传输

HTTP Streamable 是 MCP 规范推荐的远程传输协议：

```typescript
const transportOptions: StreamableHTTPClientTransportOptions = {
  authProvider,
  fetch: wrapFetchWithTimeout(
    wrapFetchWithStepUpDetection(createFetchWithInit(), authProvider),
  ),
  requestInit: {
    headers: {
      'User-Agent': getMCPUserAgent(),
      ...combinedHeaders,
    },
  },
}
```

**协议要求：** 客户端必须在 POST 请求中发送 `Accept: application/json, text/event-stream` 头。

### 3.5 WebSocket 传输

Claude Code 实现了自定义 `WebSocketTransport`，兼容 Bun 原生 WebSocket 和 `ws` 包：

```typescript
export class WebSocketTransport implements Transport {
  private started = false
  private opened: Promise<void>

  constructor(private ws: WebSocketLike) {
    this.opened = new Promise((resolve, reject) => {
      if (this.ws.readyState === WS_OPEN) {
        resolve()
      } else if (this.isBun) {
        // Bun 原生 WebSocket 事件处理
        const nws = this.ws as unknown as globalThis.WebSocket
        nws.addEventListener('open', onOpen)
        nws.addEventListener('error', onError)
      } else {
        // ws 包事件处理
        const nws = this.ws as unknown as WsWebSocket
        nws.on('open', () => resolve())
        nws.on('error', error => reject(error))
      }
    })
  }
}
```

**兼容性处理：**

```typescript
// 检测运行时环境
private isBun = typeof Bun !== 'undefined'

// Bun 原生 WebSocket
if (this.isBun) {
  nws.addEventListener('message', this.onBunMessage)
} else {
  // ws 包
  nws.on('message', this.onNodeMessage)
}
```

---

## 4. OAuth 认证流程

### 4.1 认证架构

Claude Code 实现了完整的 OAuth 2.0 认证流程，通过 `ClaudeAuthProvider` 类管理：

```typescript
export class ClaudeAuthProvider implements OAuthClientProvider {
  private serverName: string
  private serverConfig: McpSSEServerConfig | McpHTTPServerConfig
  private redirectUri: string
  private _codeVerifier?: string
  private _authorizationUrl?: string
  private _state?: string
  private _scopes?: string
  private _metadata?
  private _refreshInProgress?: Promise<OAuthTokens | undefined>
  private _pendingStepUpScope?: string
}
```

### 4.2 认证流程详解

```
┌─────────┐    1. 初始化    ┌──────────────────┐
│  用户   │ ──────────────→ │ ClaudeAuthProvider│
└─────────┘                 └────────┬─────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │ 发现 OAuth 元数据 │
                           │ (RFC 8414/9728)  │
                           └────────┬─────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │ 启动 PKCE 流程   │
                           │ 生成 code_verifier│
                           └────────┬─────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │ 重定向到授权服务器│
                           └────────┬─────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │ 接收授权码       │
                           │ /callback        │
                           └────────┬─────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │ 交换令牌        │
                           │ authCode → tokens│
                           └────────┬─────────┘
                                      │
                                      ▼
                           ┌──────────────────┐
                           │ 保存令牌到存储   │
                           │ (Keychain/文件)  │
                           └──────────────────┘
```

### 4.3 令牌管理

#### 令牌存储结构

```typescript
type SecureStorageData = {
  mcpOAuth: {
    [serverKey: string]: {
      serverName: string
      serverUrl: string
      accessToken: string
      refreshToken?: string
      expiresAt: number
      scope?: string
      clientId?: string
      clientSecret?: string
      stepUpScope?: string
      discoveryState?: {
        authorizationServerUrl: string
        resourceMetadataUrl?: string
      }
    }
  }
}
```

#### 服务器密钥生成

```typescript
export function getServerKey(
  serverName: string,
  serverConfig: McpSSEServerConfig | McpHTTPServerConfig,
): string {
  const configJson = jsonStringify({
    type: serverConfig.type,
    url: serverConfig.url,
    headers: serverConfig.headers || {},
  })
  const hash = createHash('sha256')
    .update(configJson)
    .digest('hex')
    .substring(0, 16)
  return `${serverName}|${hash}`
}
```

### 4.4 令牌刷新

令牌刷新采用进程间锁机制，防止多个 Claude Code 实例同时刷新：

```typescript
async refreshAuthorization(refreshToken: string): Promise<OAuthTokens | undefined> {
  const lockfilePath = join(claudeDir, `mcp-refresh-${sanitizedKey}.lock`)

  // 获取锁（最多重试 5 次）
  for (let retry = 0; retry < MAX_LOCK_RETRIES; retry++) {
    try {
      release = await lockfile.lock(lockfilePath, {
        realpath: false,
        onCompromised: () => { /* 处理锁竞争 */ },
      })
      break
    } catch (e) {
      if (code === 'ELOCKED') {
        await sleep(1000 + Math.random() * 1000)  // 指数退避
        continue
      }
    }
  }
}
```

### 4.5 Step-up 认证

当服务器返回 403 insufficient_scope 时，触发 step-up 认证：

```typescript
export function wrapFetchWithStepUpDetection(
  baseFetch: FetchLike,
  provider: ClaudeAuthProvider,
): FetchLike {
  return async (url, init) => {
    const response = await baseFetch(url, init)
    if (response.status === 403) {
      const wwwAuth = response.headers.get('WWW-Authenticate')
      if (wwwAuth?.includes('insufficient_scope')) {
        const match = wwwAuth.match(/scope=(?:"([^"]+)"|([^\s,]+))/)
        const scope = match?.[1] ?? match?.[2]
        if (scope) {
          provider.markStepUpPending(scope)  // 标记需要 step-up
        }
      }
    }
    return response
  }
}
```

### 4.6 XAA (Cross-App Access)

XAA 是企业级跨应用访问方案，基于 RFC 8693 和 RFC 7523：

```
┌─────────┐                    ┌─────────┐                    ┌─────────┐
│ Claude  │                    │   IdP   │                    │  Auth   │
│  Code   │                    │ (OIDC)  │                    │ Server  │
└────┬────┘                    └────┬────┘                    └────┬────┘
     │                              │                              │
     │ 1. 获取 id_token             │                              │
     │─────────────────────────────→│                              │
     │                              │                              │
     │ 2. RFC 8693 Token Exchange    │                              │
     │ id_token → ID-JAG            │                              │
     │─────────────────────────────→│                              │
     │                              │                              │
     │ 3. ID-JAG                    │                              │
     │←─────────────────────────────│                              │
     │                              │                              │
     │ 4. RFC 7523 JWT Bearer       │                              │
     │ ID-JAG → access_token       │                              │
     │─────────────────────────────────────────────────────────────→│
     │                              │                              │
     │ 5. access_token              │                              │
     │←─────────────────────────────────────────────────────────────│
```

---

## 5. 工具与资源管理

### 5.1 工具命名规范

MCP 工具名称格式：`mcp__<serverName>__<toolName>`

```typescript
export function filterToolsByServer(tools: Tool[], serverName: string): Tool[] {
  const prefix = `mcp__${normalizeNameForMCP(serverName)}__`
  return tools.filter(tool => tool.name?.startsWith(prefix))
}
```

#### 名称规范化

```typescript
export function normalizeNameForMCP(name: string): string {
  return name
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '_')
    .replace(/^_|_$/g, '')
}
```

### 5.2 资源访问

MCP 服务器可以提供资源，格式为 `mcp://<server>/<uri>`：

```typescript
export type ServerResource = Resource & { server: string }

export function filterResourcesByServer(
  resources: ServerResource[],
  serverName: string,
): ServerResource[] {
  return resources.filter(resource => resource.server === serverName)
}
```

### 5.3 权限控制

#### 项目级别权限

```typescript
export function getProjectMcpServerStatus(
  serverName: string,
): 'approved' | 'rejected' | 'pending' {
  const settings = getSettings_DEPRECATED()
  const normalizedName = normalizeNameForMCP(serverName)

  // 检查禁用列表
  if (settings?.disabledMcpjsonServers?.some(
    name => normalizeNameForMCP(name) === normalizedName,
  )) {
    return 'rejected'
  }

  // 检查启用列表
  if (
    settings?.enabledMcpjsonServers?.some(
      name => normalizeNameForMCP(name) === normalizedName,
    ) || settings?.enableAllProjectMcpServers
  ) {
    return 'approved'
  }

  // 危险模式跳过检查
  if (
    hasSkipDangerousModePermissionPrompt() &&
    isSettingSourceEnabled('projectSettings')
  ) {
    return 'approved'
  }

  return 'pending'
}
```

#### 企业策略过滤

```typescript
export function filterMcpServersByPolicy<T>(configs: Record<string, T>): {
  allowed: Record<string, T>
  blocked: string[]
} {
  const allowed: Record<string, T> = {}
  const blocked: string[] = []

  for (const [name, config] of Object.entries(configs)) {
    if (
      config.type === 'sdk' ||
      isMcpServerAllowedByPolicy(name, config as McpServerConfig)
    ) {
      allowed[name] = config
    } else {
      blocked.push(name)
    }
  }

  return { allowed, blocked }
}
```

### 5.4 动态请求头

支持通过外部脚本生成动态请求头：

```typescript
export async function getMcpHeadersFromHelper(
  serverName: string,
  config: McpServerConfig,
): Promise<Record<string, string> | null> {
  if (!config.headersHelper) {
    return null
  }

  // 安全检查：项目配置需确认信任
  if (isMcpServerFromProjectOrLocalSettings(config)) {
    const hasTrust = checkHasTrustDialogAccepted()
    if (!hasTrust) {
      return null
    }
  }

  const execResult = await execFileNoThrowWithCwd(
    config.headersHelper,
    [],
    {
      shell: true,
      timeout: 10000,
      env: {
        ...process.env,
        CLAUDE_CODE_MCP_SERVER_NAME: serverName,
        CLAUDE_CODE_MCP_SERVER_URL: config.url,
      },
    },
  )

  return jsonParse(execResult.stdout.trim())
}
```

---

## 6. MCP 服务器生命周期

### 6.1 连接建立

```
┌─────────────────────────────────────────────────────────────┐
│                     连接建立流程                               │
├─────────────────────────────────────────────────────────────┤
│  1. 配置加载                                                 │
│     ├─ 加载企业配置 (优先)                                    │
│     ├─ 加载项目配置                                           │
│     ├─ 加载用户配置                                           │
│     └─ 加载动态配置                                           │
│                                                             │
│  2. 策略过滤                                                  │
│     ├─ 检查允许列表                                           │
│     ├─ 检查拒绝列表                                           │
│     └─ 分离重复服务器                                         │
│                                                             │
│  3. 连接建立                                                  │
│     ├─ 选择传输层                                             │
│     ├─ 创建传输实例                                           │
│     ├─ 初始化认证 (如需要)                                    │
│     └─ 建立 MCP 会话                                          │
│                                                             │
│  4. 能力协商                                                  │
│     ├─ 交换服务器能力                                         │
│     ├─ 注册处理器                                             │
│     └─ 订阅通知                                               │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 心跳保持

MCP 协议本身不包含心跳机制，但通过以下方式保持连接活跃：

| 传输类型 | 保活机制 |
|----------|----------|
| Stdio | 子进程监督，自动重启 |
| SSE | EventSource 长连接 |
| HTTP | Streamable HTTP keep-alive |
| WebSocket | WebSocket ping/pong |

### 6.3 异常断开与重连

#### 连接状态转换

```
                  ┌─────────────┐
                  │   pending   │
                  └──────┬──────┘
                         │
            ┌────────────┼────────────┐
            │            │            │
            ▼            ▼            ▼
     ┌───────────┐ ┌───────────┐ ┌───────────┐
     │ connected │ │   failed  │ │ needs-auth│
     └─────┬─────┘ └───────────┘ └───────────┘
           │
           │ disconnect/error
           ▼
     ┌───────────┐      ┌───────────┐
     │  pending  │ ───→ │  retry    │
     └───────────┘      └─────┬─────┘
                              │
                    max_attempts?
                    /    \
                   /      \
                  ▼        ▼
           ┌──────────┐  ┌──────────┐
           │  failed  │  │ pending  │
           └──────────┘  └──────────┘
```

#### 会话过期处理

```typescript
export function isMcpSessionExpiredError(error: Error): boolean {
  const httpStatus = 'code' in error
    ? (error as Error & { code?: number }).code
    : undefined

  if (httpStatus !== 404) {
    return false
  }

  // 检测 JSON-RPC 错误码 -32001 (Session not found)
  return (
    error.message.includes('"code":-32001') ||
    error.message.includes('"code": -32001')
  )
}
```

### 6.4 渠道通知

MCP 服务器可以向 Claude Code 推送用户消息（渠道功能）：

```typescript
// 通知格式
{
  method: 'notifications/claude/channel',
  params: {
    content: string,           // 消息内容
    meta?: Record<string, string>  // 元数据
  }
}
```

#### 消息包装

```typescript
export function wrapChannelMessage(
  serverName: string,
  content: string,
  meta?: Record<string, string>,
): string {
  const attrs = Object.entries(meta ?? {})
    .filter(([k]) => SAFE_META_KEY.test(k))
    .map(([k, v]) => ` ${k}="${escapeXmlAttr(v)}"`)
    .join('')
  return `<channel source="${escapeXmlAttr(serverName)}"${attrs}>\n${content}\n</channel>`
}
```

### 6.5 权限提升请求

服务器可以请求用户授权敏感操作：

```typescript
// 权限请求通知
{
  method: 'notifications/claude/channel/permission_request',
  params: {
    request_id: string,      // 用于匹配的请求 ID
    tool_name: string,       // 工具名称
    description: string,     // 描述
    input_preview: string,   // 输入预览（截断至 200 字符）
  }
}

// 权限回复通知
{
  method: 'notifications/claude/channel/permission',
  params: {
    request_id: string,
    behavior: 'allow' | 'deny'
  }
}
```

---

## 附录：关键数据结构

### A.1 服务器配置

```typescript
// stdio 服务器配置
type McpStdioServerConfig = {
  type?: 'stdio'
  command: string
  args: string[]
  env?: Record<string, string>
}

// SSE 服务器配置
type McpSSEServerConfig = {
  type: 'sse'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
  oauth?: McpOAuthConfig
}

// HTTP 服务器配置
type McpHTTPServerConfig = {
  type: 'http'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
  oauth?: McpOAuthConfig
}

// WebSocket 服务器配置
type McpWebSocketServerConfig = {
  type: 'ws'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
}
```

### A.2 OAuth 配置

```typescript
type McpOAuthConfig = {
  clientId?: string
  callbackPort?: number
  authServerMetadataUrl?: string  // 必须使用 HTTPS
  xaa?: boolean  // Cross-App Access
}
```

---

## 参考

- [MCP 规范](https://modelcontextprotocol.io/specification)
- [MCP SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [RFC 8414 - OAuth 2.0 Authorization Server Metadata](https://tools.ietf.org/html/rfc8414)
- [RFC 7523 - JWT Bearer Grant](https://tools.ietf.org/html/rfc7523)
- [RFC 8693 - Token Exchange](https://tools.ietf.org/html/rfc8693)
- [RFC 9728 - OAuth Protected Resource Metadata](https://datatracker.ietf.org/doc/html/rfc9728)
