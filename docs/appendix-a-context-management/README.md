# 附录A：Context 上下文管理

## 1. Context 管理概述

Claude Code 的上下文管理是确保 Agent 在长期对话中保持高效运行的核心系统。Context（上下文）包含了 Agent 执行任务所需的所有信息和状态，包括系统提示、用户上下文、Git 状态、附件信息等。

### 1.1 Context 的组成结构

Claude Code 的上下文由以下几个主要部分组成：

```
┌─────────────────────────────────────────────────────────────┐
│                      Context Structure                       │
├─────────────────────────────────────────────────────────────┤
│  systemPrompt    │ 系统提示词（工具定义、行为规则）          │
│  systemContext   │ 系统上下文（Git 状态、缓存破坏标识）       │
│  userContext     │ 用户上下文（CLAUDE.md 内容、当前日期）     │
│  attachments     │ 附件（文件、IDE选择、任务等动态内容）      │
│  messages        │ 对话历史消息                              │
│  toolResults     │ 工具执行结果                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 核心文件职责

| 文件 | 职责 |
|------|------|
| `src/context.ts` | 提供 `getSystemContext()` 和 `getUserContext()` 两个核心 memoized 函数 |
| `src/utils/queryContext.ts` | 负责构建 API 缓存键前缀，管理查询上下文参数 |
| `src/utils/attachments.ts` | 处理各种类型的上下文附件生成 |

---

## 2. 上下文构建流程

### 2.1 系统上下文 (System Context)

`getSystemContext()` 函数提供了会话级别的系统信息，使用 `memoize` 缓存以避免重复计算：

```typescript
// src/context.ts
export const getSystemContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    // Git 状态（除非在 CCR 恢复流程中或 Git 说明被禁用）
    const gitStatus = isEnvTruthy(process.env.CLAUDE_CODE_REMOTE) ||
                      !shouldIncludeGitInstructions()
      ? null
      : await getGitStatus()

    // 缓存破坏注入（仅 ant 用户，用于临时调试）
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

**关键特性**：
- `memoize` 装饰器确保整个会话期间只计算一次
- Git 状态包含：分支信息、main 分支、状态摘要、最近 5 次提交、Git 用户名
- `MAX_STATUS_CHARS = 2000` 限制状态输出长度，超出部分截断并提示用户使用 `git status`
- 支持缓存破坏机制（`CACHE_BREAKER`），允许动态注入内容而不破坏工具 schema 缓存

### 2.2 用户上下文 (User Context)

`getUserContext()` 负责加载用户和项目级别的内存文件：

```typescript
// src/context.ts
export const getUserContext = memoize(
  async (): Promise<{ [k: string]: string }> => {
    // CLAUDE_CODE_DISABLE_CLAUDE_MDS: 硬关闭
    // --bare: 跳过自动发现，但显式 --add-dir 除外
    const shouldDisableClaudeMd = isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_CLAUDE_MDS) ||
      (isBareMode() && getAdditionalDirectoriesForClaudeMd().length === 0)

    const claudeMd = shouldDisableClaudeMd
      ? null
      : getClaudeMds(filterInjectedMemoryFiles(await getMemoryFiles()))

    return {
      ...(claudeMd && { claudeMd }),
      currentDate: `Today's date is ${getLocalISODate()}.`,
    }
  },
)
```

**关键特性**：
- 支持通过 `CLAUDE_CODE_DISABLE_CLAUDE_MDS` 环境变量完全禁用
- `--bare` 模式下，只有显式指定的目录会被加载
- 包含当前日期，用于时间敏感的任务处理

### 2.3 上下文获取流程

`fetchSystemPromptParts()` 在 `queryContext.ts` 中协调获取系统提示的各个部分：

```typescript
// src/utils/queryContext.ts
export async function fetchSystemPromptParts({
  tools,
  mainLoopModel,
  additionalWorkingDirectories,
  mcpClients,
  customSystemPrompt,
}: {
  tools: Tools
  mainLoopModel: string
  additionalWorkingDirectories: string[]
  mcpClients: MCPServerConnection[]
  customSystemPrompt: string | undefined
}): Promise<{
  defaultSystemPrompt: string[]
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
}> {
  const [defaultSystemPrompt, userContext, systemContext] = await Promise.all([
    customSystemPrompt !== undefined
      ? Promise.resolve([])
      : getSystemPrompt(tools, mainLoopModel, additionalWorkingDirectories, mcpClients),
    getUserContext(),
    customSystemPrompt !== undefined ? Promise.resolve({}) : getSystemContext(),
  ])
  return { defaultSystemPrompt, userContext, systemContext }
}
```

---

## 3. Token 预算管理

### 3.1 模型上下文窗口

Claude Code 根据不同模型动态计算有效的上下文窗口大小：

```typescript
// 来自 src/services/compact/autoCompact.js
getEffectiveContextWindowSize()
isAutoCompactEnabled()
```

不同模型的上下文窗口：
- **Haiku**: 较小上下文窗口，适合快速任务
- **Sonnet**: 中等上下文窗口
- **Opus**: 最大上下文窗口，适合复杂任务

### 3.2 Token 使用监控

`attachments.ts` 中包含 Token 使用情况的附件生成：

```typescript
// src/utils/attachments.ts
export type TokenUsageAttachment = {
  type: 'token_usage'
  used: number
  total: number
  remaining: number
}

export type OutputTokenUsageAttachment = {
  type: 'output_token_usage'
  turn: number
  session: number
  budget: number | null
}

export type BudgetUsdAttachment = {
  type: 'budget_usd'
  used: number
  total: number
  remaining: number
}
```

### 3.3 预算管理机制

Claude Code 通过以下方式追踪和管理 Token 预算：

```typescript
// src/utils/attachments.ts 中的附件收集
maybe('token_usage', async () =>
  Promise.resolve(
    getTokenUsageAttachment(messages ?? [], toolUseContext.options.mainLoopModel),
  ),
),
maybe('budget_usd', async () =>
  Promise.resolve(
    getMaxBudgetUsdAttachment(toolUseContext.options.maxBudgetUsd),
  ),
),
maybe('output_token_usage', async () =>
  Promise.resolve(getOutputTokenUsageAttachment()),
),
```

### 3.4 相关记忆预算配置

```typescript
// src/utils/attachments.ts
export const RELEVANT_MEMORIES_CONFIG = {
  // 每轮限制（5 × 4KB = 20KB）bounds a single injection,
  // 但在长会话中，选择器会持续展示不同的文件
  // 在生产环境中观察到约 26K tokens/session
  // 限制累积字节数：一旦达到，停止 prefetch
  // 预算约 3 次完整注入；之后最相关的记忆已在上下文中
  MAX_SESSION_BYTES: 60 * 1024,
} as const
```

---

## 4. 附件处理

### 4.1 附件类型体系

Claude Code 使用强类型的 `Attachment` 联合类型来统一管理各种上下文附件：

```typescript
// src/utils/attachments.ts
export type Attachment =
  | FileAttachment                    // 用户 @mention 的文件
  | CompactFileReferenceAttachment    // 压缩后的文件引用
  | PDFReferenceAttachment           // PDF 文件引用
  | AlreadyReadFileAttachment         // 已读取的文件
  | { type: 'edited_text_file', ... }    // 编辑过的文本文件
  | { type: 'edited_image_file', ... }   // 编辑过的图片文件
  | { type: 'directory', ... }           // 目录列表
  | { type: 'selected_lines_in_ide', ... }  // IDE 选中的行
  | { type: 'opened_file_in_ide', ... }    // IDE 打开的文件
  | { type: 'todo_reminder', ... }         // Todo 提醒
  | { type: 'task_reminder', ... }         // 任务提醒
  | { type: 'nested_memory', ... }         // 嵌套内存文件
  | { type: 'relevant_memories', ... }      // 相关记忆
  | { type: 'skill_discovery', ... }       // 技能发现
  | { type: 'diagnostics', ... }           // 诊断信息
  | { type: 'plan_mode', ... }             // 计划模式
  | { type: 'mcp_resource', ... }          // MCP 资源
  | { type: 'token_usage', ... }           // Token 使用情况
  | { type: 'hook_*', ... }               // Hook 相关附件
  // ... 更多类型
```

### 4.2 附件生成流程

`getAttachments()` 是核心附件生成函数，使用并发模式优化性能：

```typescript
// src/utils/attachments.ts
export async function getAttachments(
  input: string | null,
  toolUseContext: ToolUseContext,
  ideSelection: IDESelection | null,
  queuedCommands: QueuedCommand[],
  messages?: Message[],
  querySource?: QuerySource,
  options?: { skipSkillDiscovery?: boolean },
): Promise<Attachment[]> {
  // 1. 用户输入相关附件（@提及的文件、Agent 提及、技能发现）
  const userInputAttachments = input
    ? [
        maybe('at_mentioned_files', () => processAtMentionedFiles(input, context)),
        maybe('mcp_resources', () => processMcpResourceAttachments(input, context)),
        maybe('agent_mentions', () => processAgentMentions(input, ...)),
        // ... 技能发现等
      ]
    : []

  // 2. 线程安全附件（主线程和子 Agent 都会运行）
  const allThreadAttachments = [
    maybe('queued_commands', () => getQueuedCommandAttachments(queuedCommands)),
    maybe('date_change', () => getDateChangeAttachments(messages)),
    maybe('changed_files', () => getChangedFiles(context)),
    maybe('nested_memory', () => getNestedMemoryAttachments(context)),
    // ...
  ]

  // 3. 仅主线程附件（IDE 相关等）
  const mainThreadAttachments = isMainThread
    ? [
        maybe('ide_selection', () => getSelectedLinesFromIDE(ideSelection, toolUseContext)),
        maybe('diagnostics', () => getDiagnosticAttachments(toolUseContext)),
        maybe('token_usage', () => getTokenUsageAttachment(...)),
        // ...
      ]
    : []

  // 4. 并发执行
  const [userAttachmentResults, threadAttachmentResults, mainThreadAttachmentResults] =
    await Promise.all([
      Promise.all(userInputAttachments),
      Promise.all(allThreadAttachments),
      Promise.all(mainThreadAttachments),
    ])

  return [...userAttachmentResults.flat(), ...threadAttachmentResults.flat(), ...mainThreadAttachmentResults.flat()]
}
```

### 4.3 附件节流机制

为避免频繁的附件生成影响性能，Claude Code 实现了精细的节流机制：

```typescript
// Todo 提醒配置
export const TODO_REMINDER_CONFIG = {
  TURNS_SINCE_WRITE: 10,
  TURNS_BETWEEN_REMINDERS: 10,
} as const

// 计划模式附件配置
export const PLAN_MODE_ATTACHMENT_CONFIG = {
  TURNS_BETWEEN_ATTACHMENTS: 5,
  FULL_REMINDER_EVERY_N_ATTACHMENTS: 5,
} as const

// 自动模式附件配置
export const AUTO_MODE_ATTACHMENT_CONFIG = {
  TURNS_BETWEEN_ATTACHMENTS: 5,
  FULL_REMINDER_EVERY_N_ATTACHMENTS: 5,
} as const
```

### 4.4 嵌套内存文件处理

嵌套内存文件（CLAUDE.md 等）通过专门的算法处理：

```typescript
// src/utils/attachments.ts
function getDirectoriesToProcess(
  targetPath: string,
  originalCwd: string,
): { nestedDirs: string[]; cwdLevelDirs: string[] } {
  // 从目标目录向上遍历到原始 CWD
  // nestedDirs: CWD → 目标路径的目录
  // cwdLevelDirs: 根目录 → CWD 的目录（仅条件规则）
}

// 处理顺序（必须保持）：
// 1. Managed/User 条件规则匹配 targetPath
// 2. 嵌套目录（CWD → 目标）：CLAUDE.md + 无条件规则 + 条件规则
// 3. CWD 级别目录（根 → CWD）：仅条件规则
```

---

## 5. 上下文压缩策略

### 5.1 自动压缩 (Auto Compact)

Claude Code 实现了智能的自动压缩机制，当上下文接近 Token 限制时自动触发：

```typescript
// src/services/compact/autoCompact.js
getEffectiveContextWindowSize()
isAutoCompactEnabled()
```

### 5.2 压缩触发条件

压缩机制会考虑以下因素：
- 当前 Token 使用量与上下文窗口的比例
- 最近的工具调用频率
- 对话长度和历史消息的 Relevance

### 5.3 压缩后的文件引用

压缩后，系统会生成 `compact_file_reference` 附件来保留对重要文件的引用：

```typescript
// src/utils/attachments.ts
export type CompactFileReferenceAttachment = {
  type: 'compact_file_reference'
  filename: string
  displayPath: string
}
```

### 5.4 历史消息的 Relevance 评分

```typescript
// 相关记忆配置
export const RELEVANT_MEMORIES_CONFIG = {
  // 每轮限制：5 × 4KB = 20KB
  // 限制累积字节数：60KB 后停止 prefetch
  MAX_SESSION_BYTES: 60 * 1024,
}
```

### 5.5 压缩相关附件

```typescript
// 压缩提醒附件
{ type: 'compaction_reminder' }

// 上下文效率附件
{ type: 'context_efficiency' }
```

---

## 6. 高级特性

### 6.1 缓存破坏机制 (Cache Breaker)

用于在不破坏工具 Schema 缓存的情况下，注入临时的调试或特殊指令：

```typescript
// src/context.ts
let systemPromptInjection: string | null = null

export function setSystemPromptInjection(value: string | null): void {
  systemPromptInjection = value
  // 注入更改时立即清除上下文缓存
  getUserContext.cache.clear?.()
  getSystemContext.cache.clear?.()
}
```

### 6.2 日期变更处理

当本地日期变更（用户通宵编码）时，系统会生成 `date_change` 附件通知模型：

```typescript
// src/utils/attachments.ts
export function getDateChangeAttachments(
  messages: Message[] | undefined,
): Attachment[] {
  const currentDate = getLocalISODate()
  const lastDate = getLastEmittedDate()

  if (currentDate === lastDate) return []

  // 刷新昨天的 transcript 到每日文件
  if (feature('KAIROS')) {
    sessionTranscriptModule?.flushOnDateChange(messages, currentDate)
  }

  return [{ type: 'date_change', newDate: currentDate }]
}
```

### 6.3 MCP 资源处理

支持从 MCP 服务器获取资源作为上下文附件：

```typescript
export type MCPResourceAttachment = {
  type: 'mcp_resource'
  server: string
  uri: string
  name: string
  description?: string
  content: ReadResourceResult
}
```

### 6.4 Hook 系统附件

Hook 执行结果也会作为附件传递给模型：

```typescript
export type HookSuccessAttachment = {
  type: 'hook_success'
  content: string
  hookName: string
  toolUseID: string
  hookEvent: HookEvent
  stdout?: string
  stderr?: string
  exitCode?: number
  command?: string
  durationMs?: number
}

export type HookErrorDuringExecutionAttachment = {
  type: 'hook_error_during_execution'
  content: string
  hookName: string
  toolUseID: string
  hookEvent: HookEvent
  command?: string
  durationMs?: number
}
```

---

## 7. 性能优化

### 7.1 Memoization 缓存

系统上下文和用户上下文都使用 `memoize` 进行缓存：

```typescript
export const getSystemContext = memoize(async () => { ... })
export const getUserContext = memoize(async () => { ... })
```

### 7.2 附件计算超时

附件计算设置了 1 秒超时限制：

```typescript
const abortController = createAbortController()
const timeoutId = setTimeout(ac => ac.abort(), 1000, abortController)
const context = { ...toolUseContext, abortController }
```

### 7.3 采样日志

为减少日志量，仅对 5% 的附件计算事件进行详细日志记录：

```typescript
if (Math.random() < 0.05) {
  logEvent('tengu_attachment_compute_duration', {
    label,
    duration_ms: duration,
    attachment_size_bytes: attachmentSizeBytes,
    attachment_count: result.length,
  })
}
```

---

## 8. 总结

Claude Code 的上下文管理系统通过以下核心机制实现了高效、可扩展的对话上下文维护：

1. **分层缓存**：系统上下文和用户上下文使用 memoization 避免重复计算
2. **类型化附件系统**：统一的 `Attachment` 类型覆盖所有上下文内容
3. **智能节流**：基于轮次的节流机制防止附件过载
4. **自动压缩**：当接近 Token 限制时自动触发上下文压缩
5. **并发处理**：用户输入附件、线程附件、主线程附件并行处理
6. **Budget 追踪**：多维度的 Token 和 USD 预算监控

---

## 9. 附录：核心类型定义

### 9.1 Attachment 类型体系

```typescript
// 附件类型联合（30+ 变体）
type Attachment =
  | TextAttachment
  | ImageAttachment
  | ToolResultAttachment
  | ReasoningAttachment
  | CompactReminderAttachment
  | ContextEfficiencyAttachment
  | CodeDiffAttachment
  | TodoListAttachment
  | CliResponseAttachment
  | PlanModeAttachment
  | AutoModeAttachment
  | EmbeddingChunkAttachment
  | CompactFileReferenceAttachment
  | MemoryPromptAttachment
  | SemanticContextAttachment
  | RecentQueriesAttachment
  | CommandResultAttachment;
```

### 9.2 Context 构建器类型

```typescript
interface QueryContextParams {
  messages: Message[];
  attachments: Attachment[];
  systemPrompt: ContentBlockParam[];
  systemContext: Record<string, string>;
  userContext: Record<string, string>;
  compactBoundary?: string;
  querySource?: QuerySource;
}

interface BudgetTracking {
  inputTokens: number;
  outputTokens: number;
  totalTokens: number;
  inputUsd: number;
  outputUsd: number;
  totalUsd: number;
  remainingBudget: number;
}
```

### 9.3 缓存破坏机制

```typescript
// CACHE_BREAKER 注入到系统上下文，强制模型重新评估
const CACHE_BREAKER = ' ENABLE_RUNTIME_VALUES ';

// 使用场景：当需要模型忽略工具 schema 缓存时注入
const systemContext = {
  ...baseContext,
  ENABLE_RUNTIME_VALUES: isAntUser ? 'true' : undefined,
  // ... 其他字段
};
```