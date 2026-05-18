# 第三章：Query/Agent 执行内核

Claude Code 的 Query/Agent 执行内核是整个系统的核心引擎，负责处理用户输入、调用工具、执行 Agent 逻辑，并与 Claude API 进行流式交互。本章将深入剖析 Query 引擎的架构设计与实现细节。

## 1. Query 引擎概述

### 1.1 架构定位

Query 引擎在 Claude Code 架构中处于核心位置，连接上层 UI/REPL 与底层 API 服务：

```
┌─────────────────────────────────────────────────────────────────┐
│                         Entry Points                            │
│   (ask CLI, SDK, REPL, Co-work)                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      QueryEngine                                │
│   - 会话状态管理 (messages, usage, permissions)                  │
│   - 消息处理与流转                                              │
│   - 工具调用编排                                                │
│   - 中断与恢复                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       query.ts                                  │
│   - 核心查询循环                                                 │
│   - 流式响应处理                                                │
│   - 上下文压缩与恢复                                            │
│   - 错误处理与重试                                              │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     API Layer                                   │
│   (claude.ts - Model calling, Tool execution)                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心职责

Query 引擎承担以下核心职责：

1. **消息生命周期管理**：维护对话历史消息，包括 assistant、user、system 消息的添加、更新与持久化
2. **API 流式交互**：与 Claude API 进行流式通信，处理 content blocks、tool_use、streaming events
3. **工具编排执行**：协调内置工具与 MCP 工具的执行，处理权限管理与结果回传
4. **上下文压缩**：在 token 预算紧张时执行自动压缩（auto-compact），包括历史摘要与微压缩
5. **状态恢复**：支持会话中断后的恢复（resume），包括消息去重与状态重建
6. **错误恢复**：处理各类 API 错误（rate limit、prompt too long、max output tokens），实施多级恢复策略

## 2. query.ts 核心逻辑

`src/query.ts` 是 Query 引擎的核心实现（约 1730 行），包含查询循环、上下文管理、工具执行、错误恢复等核心逻辑。

### 2.1 消息处理流程

query.ts 中的消息处理遵循以下流程：

```typescript
// 查询入口点
export async function* query(
  params: QueryParams,
): AsyncGenerator<StreamEvent | Message | TombstoneMessage, Terminal> {
  const consumedCommandUuids: string[] = []
  const terminal = yield* queryLoop(params, consumedCommandUuids)
  // 命令生命周期通知
  for (const uuid of consumedCommandUuids) {
    notifyCommandLifecycle(uuid, 'completed')
  }
  return terminal
}
```

#### 2.1.1 查询循环架构

queryLoop 是核心循环，采用状态机模式管理多轮对话：

```typescript
async function* queryLoop(
  params: QueryParams,
  consumedCommandUuids: string[],
): AsyncGenerator<StreamEvent | Message | TombstoneMessage, Terminal> {
  // 不可变参数 — 查询循环期间永不重新赋值
  const { systemPrompt, userContext, systemContext, canUseTool, fallbackModel, querySource, maxTurns } = params
  const deps = params.deps ?? productionDeps()

  // 跨迭代可变状态
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    maxOutputTokensOverride: params.maxOutputTokensOverride,
    autoCompactTracking: undefined,
    stopHookActive: undefined,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    turnCount: 1,
    pendingToolUseSummary: undefined,
    transition: undefined,
  }

  // 内存预取 — 单次用户轮询
  using pendingMemoryPrefetch = startRelevantMemoryPrefetch(state.messages, state.toolUseContext)

  // 主循环
  while (true) {
    // 状态解构 — 每次迭代顶部读取
    let { toolUseContext } = state
    const { messages, autoCompactTracking, ... } = state

    // 技能发现预取
    const pendingSkillPrefetch = skillPrefetch?.startSkillDiscoveryPrefetch(...)

    yield { type: 'stream_request_start' }

    // ===== 上下文准备 =====
    let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]

    // Snip 压缩
    if (feature('HISTORY_SNIP')) {
      const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
      messagesForQuery = snipResult.messages
      snipTokensFreed = snipResult.tokensFreed
    }

    // 微压缩
    const microcompactResult = await deps.microcompact(messagesForQuery, toolUseContext, querySource)
    messagesForQuery = microcompactResult.messages

    // 上下文折叠
    if (feature('CONTEXT_COLLAPSE') && contextCollapse) {
      const collapseResult = await contextCollapse.applyCollapsesIfNeeded(messagesForQuery, toolUseContext, querySource)
      messagesForQuery = collapseResult.messages
    }

    // 自动压缩
    const { compactionResult, consecutiveFailures } = await deps.autocompact(
      messagesForQuery, toolUseContext,
      { systemPrompt, userContext, systemContext, toolUseContext, forkContextMessages: messagesForQuery },
      querySource, tracking, snipTokensFreed,
    )

    if (compactionResult) {
      // 处理压缩结果，生成后压缩消息
      const postCompactMessages = buildPostCompactMessages(compactionResult)
      for (const message of postCompactMessages) {
        yield message
      }
      messagesForQuery = postCompactMessages
    }

    // ===== API 调用循环 =====
    try {
      while (attemptWithFallback) {
        attemptWithFallback = false
        // 流式 API 调用
        for await (const message of deps.callModel({...})) {
          // 处理 assistant、stream_event、tool_use 等各类消息
          if (message.type === 'assistant') {
            assistantMessages.push(message)
            // 工具块提取
            const msgToolUseBlocks = message.message.content.filter(
              content => content.type === 'tool_use'
            ) as ToolUseBlock[]
            toolUseBlocks.push(...msgToolUseBlocks)
            needsFollowUp = true
          }
          // 流式工具执行
          if (streamingToolExecutor) {
            for (const result of streamingToolExecutor.getCompletedResults()) {
              yield result.message
            }
          }
        }
      }
    } catch (error) {
      // 错误处理与模型回退
      if (error instanceof FallbackTriggeredError && fallbackModel) {
        currentModel = fallbackModel
        attemptWithFallback = true
        continue
      }
    }

    // ===== 工具执行 =====
    if (needsFollowUp) {
      const toolUpdates = streamingToolExecutor
        ? streamingToolExecutor.getRemainingResults()
        : runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)

      for await (const update of toolUpdates) {
        if (update.message) {
          yield update.message
        }
      }
    }

    // ===== 附件处理 =====
    for await (const attachment of getAttachmentMessages(...)) {
      yield attachment
    }

    // ===== 轮次推进 =====
    state = {
      messages: [...messagesForQuery, ...assistantMessages, ...toolResults],
      turnCount: nextTurnCount,
      transition: { reason: 'next_turn' },
      ...
    }
  }
}
```

### 2.2 流式响应处理

query.ts 采用 AsyncGenerator 模式处理 Claude API 的流式响应，实现低延迟、高吞吐的消息处理。

#### 2.2.1 流式事件类型

```typescript
type StreamEvent =
  | { type: 'message_start'; message: { usage: Usage; ... } }
  | { type: 'message_delta'; delta: { stop_reason?: string; ... }; usage: Usage }
  | { type: 'message_stop' }
  | { type: 'content_block_start'; index: number; block: ContentBlock }
  | { type: 'content_block_delta'; index: number; delta: ContentBlockDelta }
  | { type: 'content_block_stop'; index: number }
```

#### 2.2.2 流式工具执行器

`StreamingToolExecutor` 允许在模型 streaming 时并行执行工具，显著降低延迟：

```typescript
// 流式工具执行初始化
const useStreamingToolExecution = config.gates.streamingToolExecution
let streamingToolExecutor = useStreamingToolExecution
  ? new StreamingToolExecutor(
      toolUseContext.options.tools,
      canUseTool,
      toolUseContext,
    )
  : null

// 流式循环中实时添加工具
for await (const message of deps.callModel({...})) {
  if (message.type === 'assistant') {
    assistantMessages.push(message)
    const msgToolUseBlocks = message.message.content.filter(
      content => content.type === 'tool_use'
    ) as ToolUseBlock[]
    for (const toolBlock of msgToolUseBlocks) {
      streamingToolExecutor.addTool(toolBlock, message)
    }
  }

  // 实时获取完成的工具结果
  if (streamingToolExecutor) {
    for (const result of streamingToolExecutor.getCompletedResults()) {
      yield result.message
      toolResults.push(...normalizeMessagesForAPI([result.message], ...))
    }
  }
}
```

### 2.3 错误处理机制

query.ts 实现了多层级错误恢复机制，确保查询的鲁棒性。

#### 2.3.1 错误分类与恢复策略

| 错误类型 | 恢复策略 | 实现位置 |
|---------|---------|---------|
| `invalid_request` (prompt too long) | 上下文折叠 → 响应式压缩 | query.ts:1062-1183 |
| `max_output_tokens` | 升级输出限制 → 恢复消息注入 | query.ts:1185-1256 |
| `rate_limit` | 模型回退 (fallback model) | query.ts:894-951 |
| 工具权限拒绝 | 跟踪拒绝，报告给 SDK | QueryEngine.ts:244-271 |
| 图片大小错误 | 用户友好消息，中断查询 | query.ts:969-978 |

#### 2.3.2 Prompt Too Long 恢复

```typescript
//  withheld 错误检测
const isWithheld413 =
  lastMessage?.type === 'assistant' &&
  lastMessage.isApiErrorMessage &&
  isPromptTooLongMessage(lastMessage)

// 阶段1：上下文折叠排水
if (feature('CONTEXT_COLLAPSE') && contextCollapse &&
    state.transition?.reason !== 'collapse_drain_retry') {
  const drained = contextCollapse.recoverFromOverflow(messagesForQuery, querySource)
  if (drained.committed > 0) {
    state = {
      ...state,
      messages: drained.messages,
      transition: { reason: 'collapse_drain_retry', committed: drained.committed },
    }
    continue
  }
}

// 阶段2：响应式压缩
if ((isWithheld413 || isWithheldMedia) && reactiveCompact) {
  const compacted = await reactiveCompact.tryReactiveCompact({...})
  if (compacted) {
    const postCompactMessages = buildPostCompactMessages(compacted)
    state = {
      messages: postCompactMessages,
      hasAttemptedReactiveCompact: true,
      transition: { reason: 'reactive_compact_retry' },
    }
    continue
  }
}
```

#### 2.3.3 Max Output Tokens 恢复

```typescript
// 升级策略：8k → 64k
if (capEnabled && maxOutputTokensOverride === undefined &&
    !process.env.CLAUDE_CODE_MAX_OUTPUT_TOKENS) {
  state = {
    ...state,
    maxOutputTokensOverride: ESCALATED_MAX_TOKENS,  // 64k
    transition: { reason: 'max_output_tokens_escalate' },
  }
  continue
}

// 恢复消息注入
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
  const recoveryMessage = createUserMessage({
    content: `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
             `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
    isMeta: true,
  })
  state = {
    messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
    maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
    transition: { reason: 'max_output_tokens_recovery', attempt: ... },
  }
  continue
}
```

## 3. QueryEngine.ts 架构

`src/QueryEngine.ts` (约 1300 行) 是查询引擎的高层封装，负责会话状态管理与 SDK 接口实现。

### 3.1 引擎初始化

QueryEngine 采用配置对象初始化：

```typescript
export type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  // ... 其他配置
}

export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private hasHandledOrphanedPermission = false
  private readFileState: FileStateCache
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()

  constructor(config: QueryEngineConfig) {
    this.config = config
    this.mutableMessages = config.initialMessages ?? []
    this.abortController = config.abortController ?? createAbortController()
    this.permissionDenials = []
    this.readFileState = config.readFileCache
    this.totalUsage = EMPTY_USAGE
  }
}
```

### 3.2 submitMessage 流程

`submitMessage` 是 QueryEngine 的主入口方法，返回 AsyncGenerator<SDKMessage>：

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {
  // 1. 上下文准备
  this.discoveredSkillNames.clear()
  setCwd(cwd)

  const {
    defaultSystemPrompt,
    userContext: baseUserContext,
    systemContext,
  } = await fetchSystemPromptParts({
    tools, mainLoopModel: initialMainLoopModel,
    additionalWorkingDirectories: Array.from(
      initialAppState.toolPermissionContext.additionalWorkingDirectories.keys()
    ),
    mcpClients, customSystemPrompt: customPrompt,
  })

  // 2. 系统提示构建
  const systemPrompt = asSystemPrompt([
    ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
    ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])

  // 3. 孤立权限处理
  if (orphanedPermission && !this.hasHandledOrphanedPermission) {
    this.hasHandledOrphanedPermission = true
    for await (const message of handleOrphanedPermission(...)) {
      yield message
    }
  }

  // 4. 用户输入处理
  const {
    messages: messagesFromUserInput,
    shouldQuery,
    allowedTools,
    model: modelFromUserInput,
    resultText,
  } = await processUserInput({...})

  // 5. 消息持久化到 transcript
  if (persistSession && messagesFromUserInput.length > 0) {
    const transcriptPromise = recordTranscript(messages)
    if (isBareMode()) {
      void transcriptPromise
    } else {
      await transcriptPromise
    }
  }

  // 6. 技能与插件加载
  const [skills, { enabled: enabledPlugins }] = await Promise.all([
    getSlashCommandToolSkills(getCwd()),
    loadAllPluginsCacheOnly(),
  ])

  // 7. 系统初始化消息
  yield buildSystemInitMessage({ tools, mcpClients, model: mainLoopModel, ... })

  // 8. 本地命令处理（非查询场景）
  if (!shouldQuery) {
    for (const msg of messagesFromUserInput) {
      if (msg.type === 'user' && typeof msg.message.content === 'string' && ...) {
        yield { type: 'user', ... } as SDKUserMessageReplay
      }
      // 本地命令输出作为合成 assistant 消息
      if (msg.type === 'system' && msg.subtype === 'local_command' && ...) {
        yield localCommandOutputToSDKAssistantMessage(msg.content, msg.uuid)
      }
    }
    yield { type: 'result', subtype: 'success', result: resultText ?? '', ... }
    return
  }

  // 9. 查询循环
  for await (const message of query({ messages, systemPrompt, userContext, systemContext, ... })) {
    // 消息类型路由
    switch (message.type) {
      case 'assistant':
        this.mutableMessages.push(message)
        yield* normalizeMessage(message)
        break
      case 'progress':
        this.mutableMessages.push(message)
        if (persistSession) {
          messages.push(message)
          void recordTranscript(messages)
        }
        yield* normalizeMessage(message)
        break
      case 'attachment':
        this.mutableMessages.push(message)
        // 提取结构化输出
        if (message.attachment.type === 'structured_output') {
          structuredOutputFromTool = message.attachment.data
        }
        // 最大轮次到达
        else if (message.attachment.type === 'max_turns_reached') {
          yield { type: 'result', subtype: 'error_max_turns', ... }
          return
        }
        break
      // ... 其他类型处理
    }

    // USD 预算检查
    if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
      yield { type: 'result', subtype: 'error_max_budget_usd', ... }
      return
    }
  }

  // 10. 结果生成
  const result = messages.findLast(m => m.type === 'assistant' || m.type === 'user')
  if (!isResultSuccessful(result, lastStopReason)) {
    yield { type: 'result', subtype: 'error_during_execution', ... }
    return
  }

  yield {
    type: 'result',
    subtype: 'success',
    result: textResult,
    usage: this.totalUsage,
    ...
  }
}
```

### 3.3 工具调用编排

QueryEngine 通过 `canUseTool` 包装器跟踪权限拒绝：

```typescript
const wrappedCanUseTool: CanUseToolFn = async (
  tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision,
) => {
  const result = await canUseTool(
    tool, input, toolUseContext, assistantMessage, toolUseID, forceDecision,
  )

  // 跟踪拒绝以报告给 SDK
  if (result.behavior !== 'allow') {
    this.permissionDenials.push({
      tool_name: sdkCompatToolName(tool.name),
      tool_use_id: toolUseID,
      tool_input: input,
    })
  }

  return result
}
```

### 3.4 状态管理

QueryEngine 维护以下状态：

| 状态字段 | 类型 | 说明 |
|---------|------|------|
| `mutableMessages` | Message[] | 对话历史，可变数组 |
| `permissionDenials` | SDKPermissionDenial[] | 被拒绝的工具调用 |
| `totalUsage` | NonNullableUsage | 累计 API 使用量 |
| `hasHandledOrphanedPermission` | boolean | 是否已处理孤立权限 |
| `readFileState` | FileStateCache | 文件读取缓存 |
| `discoveredSkillNames` | Set<string> | 本轮发现的技能 |
| `loadedNestedMemoryPaths` | Set<string> | 已加载的嵌套内存路径 |

## 4. 上下文管理

上下文管理是 Query 引擎的核心功能之一，负责构建、注入与压缩上下文。

### 4.1 上下文构建

#### 4.1.1 系统上下文 (System Context)

```typescript
export const getSystemContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    // Git 状态
    const gitStatus =
      isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) || !shouldIncludeGitInstructions()
        ? null
        : await getGitStatus()

    // 系统提示注入（仅 ant 用户，用于缓存破坏）
    const injection = feature('BREAK_CACHE_COMMAND')
      ? getSystemPromptInjection()
      : null

    return {
      ...(gitStatus && { gitStatus }),
      ...(feature('BREAK_CACHE_COMMAND') && injection
        ? { cacheBreaker: `[CACHE_BREAKER: ${injection}]` }
        : {}),
    }
  },
)
```

#### 4.1.2 用户上下文 (User Context)

```typescript
export const getUserContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    const shouldDisableClaudeMd =
      isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
      (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

    const claudeMd = shouldDisableClaudeMd
      ? null
      : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

    // 缓存供 yoloClassifier 使用
    setCachedClaudeMdContent(claudeMd || null)

    return {
      ...(claudeMd && { claudeMd }),
      currentDate: `Today's date is ${getLocalISODate()}.`,
    }
  },
)
```

#### 4.1.3 上下文注入

```typescript
const systemPrompt = asSystemPrompt([
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
])

const fullSystemPrompt = asSystemPrompt(
  appendSystemContext(systemPrompt, systemContext),
)
```

### 4.2 Token 预算管理

`src/query/tokenBudget.ts` 实现了基于 Token 预算的智能停止机制：

```typescript
export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // Agent 子进程跳过预算检查
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null }
  }

  const turnTokens = globalTurnTokens
  const pct = Math.round((turnTokens / budget) * 100)
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens

  // 递减回报检测：连续 3 次检查且增量小于 500 tokens
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD

  // 未达到完成阈值（90%）则继续
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    tracker.continuationCount++
    tracker.lastDeltaTokens = deltaSinceLastCheck
    tracker.lastGlobalTurnTokens = globalTurnTokens
    return {
      action: 'continue',
      nudgeMessage: getBudgetContinuationMessage(pct, turnTokens, budget),
      continuationCount: tracker.continuationCount,
      pct, turnTokens, budget,
    }
  }

  // 停止：已达阈值或递减回报
  if (isDiminishing || tracker.continuationCount > 0) {
    return {
      action: 'stop',
      completionEvent: {
        continuationCount: tracker.continuationCount,
        pct, turnTokens, budget,
        diminishingReturns: isDiminishing,
        durationMs: Date.now() - tracker.startedAt,
      },
    }
  }

  return { action: 'stop', completionEvent: null }
}
```

### 4.3 上下文压缩

Claude Code 实现了多级上下文压缩机制：

#### 4.3.1 历史截断 (History Snip)

```typescript
if (feature('HISTORY_SNIP')) {
  const snipResult = snipModule!.snipCompactIfNeeded(messagesForQuery)
  messagesForQuery = snipResult.messages
  snipTokensFreed = snipResult.tokensFreed
  if (snipResult.boundaryMessage) {
    yield snipResult.boundaryMessage
  }
}
```

#### 4.3.2 微压缩 (Microcompact)

微压缩针对工具结果进行优化，仅保留必要的工具 ID 信息：

```typescript
const microcompactResult = await deps.microcompact(
  messagesForQuery,
  toolUseContext,
  querySource,
)
messagesForQuery = microcompactResult.messages
```

#### 4.3.3 自动压缩 (Autocompact)

```typescript
const { compactionResult, consecutiveFailures } = await deps.autocompact(
  messagesForQuery,
  toolUseContext,
  {
    systemPrompt,
    userContext,
    systemContext,
    toolUseContext,
    forkContextMessages: messagesForQuery,
  },
  querySource,
  tracking,
  snipTokensFreed,
)

if (compactionResult) {
  logEvent('tengu_auto_compact_succeeded', {
    originalMessageCount: messages.length,
    compactedMessageCount: compactionResult.summaryMessages.length + ...,
    preCompactTokenCount,
    postCompactTokenCount,
    ...
  })

  const postCompactMessages = buildPostCompactMessages(compactionResult)
  for (const message of postCompactMessages) {
    yield message
  }
  messagesForQuery = postCompactMessages
}
```

## 5. Agent SDK 集成

QueryEngine 作为 Agent SDK 的核心实现，为 SDK 调用者提供统一的接口。

### 5.1 SDK 消息类型

```typescript
type SDKMessage =
  | SDKUserMessageReplay
  | SDKAssistantMessage
  | SDKSystemMessage
  | SDKStreamEvent
  | SDKResultMessage
  | SDKPermissionDenial
  | SDKCompactBoundaryMessage
  | SDKToolsUseSummary
  | SDKContentBlock
```

### 5.2 SDK 权限模式

```typescript
export type PermissionMode =
  | 'default'      // 标准权限模式
  | 'plan'         // 计划模式（允许编辑文件但需确认）
  | 'auto'         // 自动模式（自动允许某些操作）
  | 'bypass'       // 绕过模式（所有操作自动允许）

interface ToolPermissionContext {
  mode: PermissionMode
  alwaysAllowRules: {
    command: string[]  // 允许的命令白名单
    // ... 其他规则
  }
}
```

### 5.3 Agent 间通信

#### 5.3.1 子 Agent 启动

AgentTool 通过 `querySource: 'agent:${agentId}'` 标识子 Agent 调用：

```typescript
// 在 query() 中检测 Agent 调用
const queryTracking = toolUseContext.queryTracking
  ? {
      chainId: toolUseContext.queryTracking.chainId,
      depth: toolUseContext.queryTracking.depth + 1,
    }
  : {
      chainId: deps.uuid(),
      depth: 0,
    }
```

#### 5.3.2 链式追踪

```typescript
// 查询链 ID 用于分析
const queryChainIdForAnalytics = queryTracking.chainId as AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS

// 深度追踪
logEvent('tengu_query_started', {
  queryChainId: queryChainIdForAnalytics,
  queryDepth: queryTracking.depth,
})
```

## 6. 执行流程时序图

### 6.1 完整查询时序

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   SDK/REPL  │     │ QueryEngine │     │   query.ts │     │  claude.ts  │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │                   │
       │ submitMessage()   │                   │                   │
       │──────────────────>│                   │                   │
       │                   │                   │                   │
       │                   │ fetchSystemPromptParts()             │
       │                   │──────────────────>│                   │
       │                   │<──────────────────│                   │
       │                   │                   │                   │
       │                   │ processUserInput()│                   │
       │                   │──────────────────>│                   │
       │                   │<──────────────────│                   │
       │                   │                   │                   │
       │                   │ recordTranscript()│                   │
       │                   │───────────────────────────────────────>
       │                   │                   │                   │
       │  buildSystemInitMessage               │                   │
       │<──────────────────│                   │                   │
       │                   │                   │                   │
       │                   │ query()           │                   │
       │                   │──────────────────>│                   │
       │                   │                   │                   │
       │                   │                   │ microcompact()    │
       │                   │                   │───────┐           │
       │                   │                   │       │           │
       │                   │                   │<───────┘           │
       │                   │                   │                   │
       │                   │                   │ autocompact()     │
       │                   │                   │───────┐           │
       │                   │                   │       │           │
       │                   │                   │<───────┘           │
       │                   │                   │                   │
       │                   │                   │ callModel()       │
       │                   │                   │──────────────────>│
       │                   │                   │                   │
       │  yield stream_event                   │                   │
       │<──────────────────│<──────────────────│                   │
       │                   │                   │                   │
       │                   │                   │ 工具块流式到达    │
       │                   │                   │<──────────────────│
       │                   │                   │                   │
       │                   │                   │ streamingToolExec │
       │                   │                   │───────┐           │
       │                   │                   │       │           │
       │                   │                   │<───────┘           │
       │                   │                   │                   │
       │  yield message    │                   │                   │
       │<──────────────────│<──────────────────│                   │
       │                   │                   │                   │
       │  (tool_use blocks processed)         │                   │
       │                   │                   │                   │
       │                   │                   │ runTools()        │
       │                   │                   │───────┐           │
       │                   │                   │       │           │
       │                   │                   │<───────┘           │
       │                   │                   │                   │
       │  yield tool_result                   │                   │
       │<──────────────────│<──────────────────│                   │
       │                   │                   │                   │
       │                   │                   │ handleStopHooks() │
       │                   │                   │───────┐           │
       │                   │                   │       │           │
       │                   │                   │<───────┘           │
       │                   │                   │                   │
       │  yield hook messages                 │                   │
       │<──────────────────│<──────────────────│                   │
       │                   │                   │                   │
       │                   │                   │ [loop: next turn] │
       │                   │                   │                   │
       │  yield result (success/error)        │                   │
       │<──────────────────│<──────────────────│                   │
       │                   │                   │                   │
```

### 6.2 工具执行时序

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   query.ts  │     │  Streaming   │     │ ToolExecutor │     │  canUseTool │
│             │     │   ToolExec   │     │             │     │             │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │                   │
       │ addTool(toolBlock)│                   │                   │
       │──────────────────>│                   │                   │
       │                   │                   │                   │
       │ getCompletedResults()                │                   │
       │<──────────────────│                   │                   │
       │                   │                   │                   │
       │                   │ executeTool()     │                   │
       │                   │──────────────────>│                   │
       │                   │                   │                   │
       │                   │                   │ canUseTool()      │
       │                   │                   │─────────────────>│
       │                   │                   │                   │
       │                   │                   │    Permission    │
       │                   │                   │<─────────────────│
       │                   │                   │                   │
       │                   │                   │ tool execution   │
       │                   │                   │                   │
       │                   │ result.message   │                   │
       │                   │<──────────────────│                   │
       │                   │                   │                   │
       │  yield result    │                   │                   │
       │<──────────────────│                   │                   │
       │                   │                   │                   │
```

### 6.3 错误恢复时序

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   query.ts  │     │   Claude    │     │   context   │     │   reactive  │
│             │     │     API      │     │   Collapse  │     │   Compact   │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │                   │
       │ callModel()       │                   │                   │
       │──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │<──────────────────────────────────────│                   │
       │    413 (Prompt Too Long)              │                   │
       │                   │                   │                   │
       │ isWithheldPromptTooLong()             │                   │
       │──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │<──────────────────────────────────────│                   │
       │    withheld = true (不 yield 错误)    │                   │
       │                   │                   │                   │
       │ recoverFromOverflow()                 │                   │
       │──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │<──────────────────────────────────────│                   │
       │    drained (排水折叠)                 │                   │
       │                   │                   │                   │
       │ state = { messages: drained.messages, transition: ... } │
       │                   │                   │                   │
       │ [continue: 折叠排水重试]              │                   │
       │                   │                   │                   │
       │ callModel() (重试)                    │                   │
       │──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │<──────────────────────────────────────│                   │
       │    再次 413                           │                   │
       │                   │                   │                   │
       │ tryReactiveCompact()                  │                   │
       │──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │<──────────────────────────────────────│                   │
       │    compacted (压缩成功)               │                   │
       │                   │                   │                   │
       │ state = { messages: postCompactMessages, ... }           │
       │                   │                   │                   │
       │ [continue: 压缩重试]                  │                   │
       │                   │                   │                   │
       │ callModel() (再次重试)                │                   │
       │──────────────────────────────────────>│                   │
       │                   │                   │                   │
       │<──────────────────────────────────────│                   │
       │    Success!                          │                   │
       │                   │                   │                   │
```

## 7. 配置与依赖注入

### 7.1 QueryConfig

`src/query/config.ts` 定义了查询配置快照：

```typescript
export type QueryConfig = {
  sessionId: SessionId

  // 运行时门控（env/statsig）
  gates: {
    streamingToolExecution: boolean    // 流式工具执行
    emitToolUseSummaries: boolean     // 工具使用摘要
    isAnt: boolean                    // 是否 Ant 用户
    fastModeEnabled: boolean           // 快速模式
  }
}

export function buildQueryConfig(): QueryConfig {
  return {
    sessionId: getSessionId(),
    gates: {
      streamingToolExecution: checkStatsigFeatureGate_CACHED_MAY_BE_STALE(
        'tengu_streaming_tool_execution2',
      ),
      emitToolUseSummaries: isEnvTruthy(
        process.env.CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES,
      ),
      isAnt: process.env.USER_TYPE === 'ant',
      fastModeEnabled: !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FAST_MODE),
    },
  }
}
```

### 7.2 QueryDeps

`src/query/deps.ts` 实现依赖注入，便于测试：

```typescript
export type QueryDeps = {
  callModel: typeof queryModelWithStreaming   // 模型调用
  microcompact: typeof microcompactMessages     // 微压缩
  autocompact: typeof autoCompactIfNeeded       // 自动压缩
  uuid: () => string                             // UUID 生成
}

export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    microcompact: microcompactMessages,
    autocompact: autoCompactIfNeeded,
    uuid: randomUUID,
  }
}
```

### 7.3 StopHooks

`src/query/stopHooks.ts` 处理停止钩子与完成钩子：

```typescript
export async function* handleStopHooks(
  messagesForQuery: Message[],
  assistantMessages: AssistantMessage[],
  systemPrompt: SystemPrompt,
  userContext: { [k: string]: string },
  systemContext: { [k: string]: string },
  toolUseContext: ToolUseContext,
  querySource: QuerySource,
  stopHookActive?: boolean,
): AsyncGenerator<StreamEvent | Message | ..., StopHookResult> {
  // 执行停止钩子
  const generator = executeStopHooks(
    permissionMode,
    toolUseContext.abortController.signal,
    undefined,
    stopHookActive ?? false,
    toolUseContext.agentId,
    toolUseContext,
    [...messagesForQuery, ...assistantMessages],
    toolUseContext.agentType,
  )

  // 消费进度消息与阻塞错误
  for await (const result of generator) {
    if (result.message) {
      yield result.message
    }
    if (result.blockingError) {
      const userMessage = createUserMessage({
        content: getStopHookMessage(result.blockingError),
        isMeta: true,
      })
      blockingErrors.push(userMessage)
      yield userMessage
    }
    if (result.preventContinuation) {
      preventedContinuation = true
    }
  }

  // 队友空闲与任务完成钩子
  if (isTeammate()) {
    // 执行 TaskCompleted 钩子
    // 执行 TeammateIdle 钩子
  }

  return { blockingErrors, preventContinuation: preventedContinuation }
}
```

## 8. 关键技术细节

### 8.1 消息规范化

```typescript
// 规范化消息用于 API 传输
export function normalizeMessagesForAPI(
  messages: Message[],
  tools: Tools,
): Message[] {
  return messages.map(msg => {
    switch (msg.type) {
      case 'assistant':
        return {
          ...msg,
          message: {
            ...msg.message,
            content: msg.message.content.map(block => {
              if (block.type === 'tool_use') {
                // 确保工具输入可观察
                const tool = findToolByName(tools, block.name)
                if (tool?.backfillObservableInput) {
                  const inputCopy = { ...block.input }
                  tool.backfillObservableInput(inputCopy)
                  return { ...block, input: inputCopy }
                }
              }
              return block
            }),
          },
        }
      default:
        return msg
    }
  })
}
```

### 8.2 内容替换追踪

```typescript
// 工具结果预算应用
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements
    ? records => void recordContentReplacement(records, toolUseContext.agentId)
    : undefined,
  new Set(toolUseContext.options.tools
    .filter(t => !Number.isFinite(t.maxResultSizeChars))
    .map(t => t.name)),
)
```

### 8.3 Task Budget 追踪

```typescript
// task_budget.remaining 跨压缩边界追踪
if (params.taskBudget) {
  const preCompactContext = finalContextTokensFromLastResponse(messagesForQuery)
  taskBudgetRemaining = Math.max(
    0,
    (taskBudgetRemaining ?? params.taskBudget.total) - preCompactContext,
  )
}
```

## 9. 小结

Query/Agent 执行内核是 Claude Code 的核心引擎，通过以下机制实现高效、鲁棒的 Agent 执行：

1. **分层架构**：QueryEngine 负责会话状态与 SDK 接口，query.ts 负责核心查询循环
2. **流式处理**：AsyncGenerator 模式实现低延迟流式响应处理
3. **多级压缩**：Snip → Microcompact → Autocompact → Reactive Compact 的多级上下文压缩体系
4. **智能恢复**：针对不同错误类型（PTL、max_output_tokens、rate_limit）实施差异化恢复策略
5. **依赖注入**：QueryDeps 实现测试可测试性，QueryConfig 隔离环境相关配置

下一章将详细介绍工具系统的设计与实现，包括内置工具与 MCP 工具的编排机制。