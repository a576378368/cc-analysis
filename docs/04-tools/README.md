# 第四章：工具系统设计与实现

Claude Code 的工具系统是其核心能力之一，它允许 AI Agent 通过工具与文件系统、Shell 命令、第三方服务进行交互。本章将深入分析 Claude Code 的工具系统架构、工具基类设计、权限管理机制以及各种内置工具的实现原理。

## 4.1 工具系统架构

### 4.1.1 工具基类设计

Claude Code 的工具系统基于类型安全的 `Tool` 接口构建，所有工具都通过 `buildTool` 工厂函数创建。核心类型定义位于 `src/Tool.ts` 中。

```typescript
// Tool 接口的核心结构 (src/Tool.ts)
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 工具名称和别名
  name: string
  aliases?: string[]

  // 核心方法：执行工具
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>

  // 工具描述（供模型理解用途）
  description(input: z.infer<Input>, options: {...}): Promise<string>

  // 输入输出模式
  readonly inputSchema: Input
  outputSchema?: z.ZodType<unknown>

  // 状态方法
  isEnabled(): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isConcurrencySafe(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean

  // 权限验证
  checkPermissions(
    input: z.infer<Input>,
    context: ToolUseContext,
  ): Promise<PermissionResult>

  // 输入验证
  validateInput?(
    input: z.infer<Input>,
    context: ToolUseContext,
  ): Promise<ValidationResult>

  // 安全分类器输入
  toAutoClassifierInput(input: z.infer<Input>): unknown
}
```

**关键设计特点**：

1. **泛型类型安全**：工具使用泛型定义输入(`Input`)、输出(`Output`)和进度数据类型(`P`)，通过 Zod 进行运行时验证

2. **默认实现填充**：`buildTool` 工厂函数为常用方法提供默认实现，工具开发者只需覆盖特定方法

```typescript
// TOOL_DEFAULTS (src/Tool.ts)
const TOOL_DEFAULTS = {
  isEnabled: () => true,
  isConcurrencySafe: (_input?: unknown) => false,
  isReadOnly: (_input?: unknown) => false,
  isDestructive: (_input?: unknown) => false,
  checkPermissions: (...): Promise<PermissionResult> =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  toAutoClassifierInput: (_input?: unknown) => '',
  userFacingName: (_input?: unknown) => '',
}
```

3. **上下文传递**：工具通过 `ToolUseContext` 访问应用状态、文件系统、能力检查回调等

```typescript
// ToolUseContext 关键字段 (src/Tool.ts)
export type ToolUseContext = {
  options: {
    commands: Command[]
    debug: boolean
    mainLoopModel: string
    tools: Tools
    mcpClients: MCPServerConnection[]
  }
  abortController: AbortController
  readFileState: FileStateCache
  getAppState(): AppState
  setAppState(f: (prev: AppState) => AppState): void
  nestedMemoryAttachmentTriggers?: Set<string>
  fileReadingLimits?: { maxTokens?: number; maxSizeBytes?: number }
  globLimits?: { maxResults?: number }
  toolDecisions?: Map<string, { source: string; decision: 'accept' | 'reject' }>
}
```

### 4.1.2 工具注册机制

工具的注册和汇总在 `src/tools.ts` 中完成。Claude Code 采用**条件注册+死码消除**策略来优化最终构建体积。

```typescript
// 工具注册表 (src/tools.ts)
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // 当存在嵌入式搜索工具时跳过独立 Grep/Glob
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    // 仅 ANT 构建包含的工具
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    // 功能开关控制的工具
    ...(isTodoV2Enabled()
      ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool]
      : []),
    ...(isWorktreeModeEnabled() ? [EnterWorktreeTool, ExitWorktreeTool] : []),
    // 团队协作功能
    ...(isAgentSwarmsEnabled()
      ? [getTeamCreateTool(), getTeamDeleteTool()]
      : []),
    // ... 更多条件工具
  ]
}
```

**死码消除机制**：

Claude Code 使用 `feature()` 函数和条件导入实现构建时死码消除（Dead Code Elimination）。例如：

```typescript
// 通过环境变量条件导入实现 DCE
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool
  : null

const cronTools = feature('AGENT_TRIGGERS')
  ? [
      require('./tools/ScheduleCronTool/CronCreateTool.js').CronCreateTool,
      require('./tools/ScheduleCronTool/CronDeleteTool.js').CronDeleteTool,
      require('./tools/ScheduleCronTool/CronListTool.js').CronListTool,
    ]
  : []
```

### 4.1.3 工具调度流程

工具的执行遵循统一的生命周期：

```
┌─────────────────────────────────────────────────────────────────┐
│                     工具选择与调度流程                            │
├─────────────────────────────────────────────────────────────────┤
│  1. 模型推理                                                      │
│     ├─ 决定调用哪个工具 + 参数                                      │
│     └─ 生成 tool_use block                                        │
│                                                                  │
│  2. 工具查找 (findToolByName)                                     │
│     ├─ 根据 name 或 aliases 匹配工具                                │
│     └─ 返回 Tool 实例                                              │
│                                                                  │
│  3. 输入验证 (validateInput)                                      │
│     ├─ 路径验证/参数检查                                           │
│     ├─ 危险模式检测                                                │
│     └─ 返回 ValidationResult                                      │
│                                                                  │
│  4. 权限检查 (checkPermissions)                                   │
│     ├─ 规则匹配 (alwaysAllow/alwaysDeny/alwaysAsk)                 │
│     ├─ 分类器决策 (可选)                                           │
│     └─ 返回 PermissionResult                                       │
│                                                                  │
│  5. 工具执行 (call)                                               │
│     ├─ 进度回调 (onProgress)                                      │
│     ├─ 结果包装 (ToolResult)                                       │
│     └─ 新消息生成 (newMessages)                                   │
│                                                                  │
│  6. 结果渲染                                                      │
│     ├─ renderToolResultMessage (React UI)                         │
│     ├─ mapToolResultToToolResultBlockParam (API)                  │
│     └─ extractSearchText (转录本索引)                               │
└─────────────────────────────────────────────────────────────────┘
```

## 4.2 工具类型详解

### 4.2.1 BashTool：命令执行

BashTool 是 Claude Code 中最复杂也是最强大的工具，允许 Agent 执行 Shell 命令。相关代码位于 `src/tools/BashTool/`。

**核心输入模式**：

```typescript
// src/tools/BashTool/bashTool.tsx
const inputSchema = lazySchema(() => z.object({
  command: z.string().describe('The bash command to execute'),
  timeout: semanticNumber(z.number().optional()).describe('Timeout in milliseconds'),
  working_directory: z.string().optional(),
  environment: z.record(z.string()).optional(),
}))
```

**安全架构**：

BashTool 的安全架构由多个层次组成：

1. **bashSecurity.ts** - Shell 语法安全分析
   - 检测危险的命令替换模式：`$()`, `$(...)`, `` `...` ``
   - 阻止进程替换：`<()`, `>()`
   - 识别 Zsh 危险命令：`zmodload`, `emulate`, `sysopen`, `zpty` 等

```typescript
// 危险模式检测 (src/tools/BashTool/bashSecurity.ts)
const COMMAND_SUBSTITUTION_PATTERNS = [
  { pattern: /<\(/, message: 'process substitution <()' },
  { pattern: />\(/, message: 'process substitution >()' },
  { pattern: /\$\(/, message: '$() command substitution' },
  { pattern: /\$\{/, message: '${} parameter substitution' },
  // ...
]
```

2. **bashPermissions.ts** - 权限检查
   - 基于规则的权限匹配
   - Bash 分类器集成（机器学习增强的决策）
   - 路径约束验证

```typescript
// 权限检查入口 (src/tools/BashTool/bashPermissions.ts)
export async function bashToolHasPermission(
  command: string,
  context: ToolPermissionContext,
): Promise<PermissionResult> {
  // 解析命令
  const parsed = parseCommandRaw(command)

  // 检查命令操作符权限 (&&, ||, |, >, etc.)
  const operatorCheck = await checkCommandOperatorPermissions(parsed)

  // 路径约束检查
  const pathCheck = await checkPathConstraints(command, context)

  // 分类器决策（如果启用）
  if (isClassifierPermissionsEnabled()) {
    const classifierResult = await classifyBashCommand(command)
    // 根据分类器结果决定行为
  }

  return result
}
```

3. **pathValidation.ts** - 路径安全验证
   - 检测路径遍历攻击：`../`, `..\\`
   - 阻止危险路径访问：`/proc`, `/sys`, `/dev`（特定设备）
   - 验证 UNC 路径（防止 NTLM 凭证泄露）

4. **readOnlyValidation.ts** - 只读命令识别
   - 自动识别只读命令（如 `ls`, `cat --help`, `git log`）
   - 只读命令绕过写权限检查

**进度报告**：

BashTool 通过 `BashProgress` 类型支持实时进度报告：

```typescript
// 进度类型 (src/types/tools.js)
export type BashProgress = {
  type: 'bash_progress'
  data: {
    running: boolean
    restart?: boolean
    shell_pid?: number
    timeout_seconds?: number
    truncated?: boolean
    partial_line?: string
  }
}
```

### 4.2.2 FileReadTool：文件读取

FileReadTool 负责读取文件内容，支持多种文件类型。核心逻辑位于 `src/tools/FileReadTool/FileReadTool.ts`。

**支持的输出类型**：

```typescript
// src/tools/FileReadTool/FileReadTool.ts
const outputSchema = lazySchema(() =>
  z.discriminatedUnion('type', [
    z.object({ type: z.literal('text'), file: {...} }),
    z.object({ type: z.literal('image'), file: {...} }),
    z.object({ type: z.literal('notebook'), file: {...} }),
    z.object({ type: z.literal('pdf'), file: {...} }),
    z.object({ type: z.literal('parts'), file: {...} }),
    z.object({ type: z.literal('file_unchanged'), file: {...} }),
  ])
)
```

**关键安全特性**：

1. **设备文件阻止**：阻止 `/dev/zero`, `/dev/random`, `/dev/stdin` 等会阻塞或无限输出的设备文件

```typescript
// 阻止的设备路径 (src/tools/FileReadTool/FileReadTool.ts)
const BLOCKED_DEVICE_PATHS = new Set([
  '/dev/zero', '/dev/random', '/dev/urandom', '/dev/full',
  '/dev/stdin', '/dev/tty', '/dev/console',
  '/dev/stdout', '/dev/stderr',
])
```

2. **二进制文件阻止**：自动检测并阻止二进制文件（图片、PDF、Jupyter 除外）

3. **内容去重**：通过 `readFileState` 缓存实现同一文件同一范围的去重读取

```typescript
// 去重逻辑 (src/tools/FileReadTool/FileReadTool.ts)
const existingState = dedupKillswitch ? undefined : readFileState.get(fullFilePath)
if (existingState && !existingState.isPartialView && existingState.offset !== undefined) {
  const rangeMatch = existingState.offset === offset && existingState.limit === limit
  if (rangeMatch) {
    const mtimeMs = await getFileModificationTimeAsync(fullFilePath)
    if (mtimeMs === existingState.timestamp) {
      return { data: { type: 'file_unchanged', file: { filePath: file_path } } }
    }
  }
}
```

4. **Token 预算控制**：通过 `fileReadingLimits` 限制文件读取的 token 数量

```typescript
async function validateContentTokens(content: string, ext: string, maxTokens?: number) {
  const tokenEstimate = roughTokenCountEstimationForFileType(content, ext)
  if (effectiveCount > effectiveMaxTokens) {
    throw new MaxFileReadTokenExceededError(effectiveCount, effectiveMaxTokens)
  }
}
```

### 4.2.3 FileEditTool：文件编辑

FileEditTool 提供了精确的文件编辑能力，是 Claude Code 最复杂的工具之一。

**编辑流程序列图**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    FileEditTool 执行流程                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 输入验证 validateInput()                                       │
│     ├─ 路径展开 (expandPath)                                      │
│     ├─ 危险文件检查 (checkTeamMemSecrets)                          │
│     ├─ 文件存在性检查                                              │
│     ├─ 修改时间检查 (防止外部修改)                                  │
│     └─ 字符串匹配检查                                              │
│                                                                  │
│  2. 权限检查 checkPermissions()                                    │
│     ├─ 获取应用状态                                                │
│     └─ checkWritePermissionForTool()                               │
│                                                                  │
│  3. 执行 call()                                                   │
│     ├─ 技能发现 (discoverSkillDirsForPaths)                       │
│     ├─ 诊断跟踪 (diagnosticTracker.beforeFileEdited)             │
│     ├─ 原子读-改-写                                                │
│     │   ├─ 读取当前内容                                            │
│     │   ├─ 生成补丁 (getPatchForEdit)                             │
│     │   └─ 写入磁盘                                                │
│     ├─ LSP 通知 (lspManager.changeFile/saveFile)                   │
│     ├─ 更新 readFileState                                          │
│     └─ 记录文件历史 (fileHistoryTrackEdit)                         │
│                                                                  │
│  4. 结果渲染 renderToolResultMessage()                            │
│     └─ 返回成功消息                                                │
└─────────────────────────────────────────────────────────────────┘
```

**核心特性**：

1. **字符串精确匹配**：通过 `findActualString` 处理引号规范化差异

```typescript
// 引用样式保持 (src/tools/FileEditTool/utils.ts)
function preserveQuoteStyle(oldString, actualOldString, newString) {
  // 检测旧字符串实际使用的引号类型
  // 保持新字符串使用相同引号风格
}
```

2. **多匹配处理**：`replace_all` 参数控制是否替换所有匹配项

```typescript
const matches = file.split(actualOldString).length - 1
if (matches > 1 && !replace_all) {
  return {
    result: false,
    behavior: 'ask',
    message: `Found ${matches} matches...`,
  }
}
```

3. **修改时间验证**：防止外部工具修改导致的内容不一致

```typescript
const lastWriteTime = getFileModificationTime(fullFilePath)
if (lastWriteTime > readTimestamp.timestamp) {
  // 时间戳变化，但内容可能未变（Windows 云同步）
  if (isFullRead && fileContent === readTimestamp.content) {
    // 内容相同，安全继续
  } else {
    throw new Error(FILE_UNEXPECTEDLY_MODIFIED_ERROR)
  }
}
```

### 4.2.4 GrepTool/GlobTool：搜索工具

搜索工具用于在代码库中查找文件和内容。

**GrepTool 特性**：

```typescript
// src/tools/GrepTool/GrepTool.ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string(),           // 正则表达式模式
    path: z.string().optional(),   // 搜索路径
    glob: z.string().optional(),   // glob 过滤
    output_mode: z.enum(['content', 'files_with_matches', 'count']),
    '-B': semanticNumber(...),      // 前文行数
    '-A': semanticNumber(...),     // 后文行数
    '-C': semanticNumber(...),     // 上下文行数
    '-n': semanticBoolean(...),    // 显示行号
    '-i': semanticBoolean(...),    // 大小写不敏感
    type: z.string().optional(),   // 文件类型
    head_limit: semanticNumber(...),
    multiline: semanticBoolean(...),
  })
)
```

**输出模式**：

```typescript
// GrepTool 三种输出模式
const outputSchema = lazySchema(() =>
  z.object({
    mode: z.enum(['content', 'files_with_matches', 'count']),
    numFiles: z.number(),
    filenames: z.array(z.string()),
    content: z.string().optional(),  // content 模式
    numLines: z.number().optional(),
    numMatches: z.number().optional(), // count 模式
    appliedLimit: z.number().optional(),
    appliedOffset: z.number().optional(),
  })
)
```

**自动结果限制**：

```typescript
// 默认限制 250 条结果，防止上下文膨胀
const DEFAULT_HEAD_LIMIT = 250

function applyHeadLimit<T>(items: T[], limit: number | undefined, offset = 0) {
  if (limit === 0) {
    return { items: items.slice(offset), appliedLimit: undefined }
  }
  const effectiveLimit = limit ?? DEFAULT_HEAD_LIMIT
  const sliced = items.slice(offset, offset + effectiveLimit)
  return {
    items: sliced,
    appliedLimit: wasTruncated ? effectiveLimit : undefined,
  }
}
```

**GlobTool 特性**：

```typescript
// src/tools/GlobTool/GlobTool.ts
const inputSchema = lazySchema(() =>
  z.strictObject({
    pattern: z.string().describe('The glob pattern to match files against'),
    path: z.string().optional().describe('The directory to search in'),
  })
)
```

- 使用内置 `glob()` 函数（在 RipperFS 可用时使用嵌入式搜索）
- 默认限制 100 个结果
- 支持 `globLimits.maxResults` 配置

### 4.2.5 AgentTool：子 Agent 创建

AgentTool 允许 Claude Code 创建和管理子 Agent，是多智能体协作的基础。

**输入模式**：

```typescript
// src/tools/AgentTool/AgentTool.tsx
const baseInputSchema = lazySchema(() => z.object({
  description: z.string(),      // 简短描述
  prompt: z.string(),           // 任务指令
  subagent_type: z.string().optional(), // 专用代理类型
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
}))

// 多智能体参数
const multiAgentInputSchema = z.object({
  name: z.string().optional(),        // 队友名称
  team_name: z.string().optional(),   // 团队名称
  mode: permissionModeSchema().optional(), // 权限模式
  isolation: z.enum(['worktree']).optional(), // 隔离模式
  cwd: z.string().optional(),         // 工作目录
})
```

**Agent 类型系统**：

Claude Code 支持两种 Agent 类型：

1. **内置 Agent**：预定义的专用 Agent（如 `general-purpose`）
2. **自定义 Agent**：从 `~/.claude/agents/` 目录加载

```typescript
// src/tools/AgentTool/loadAgentsDir.ts
export type AgentDefinition = {
  agentType: string
  name: string
  description: string
  model?: string
  tools?: string[]
  // ...
}
```

**执行流程**：

```
┌─────────────────────────────────────────────────────────────────┐
│                      AgentTool 执行流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 参数解析                                                      │
│     ├─ 解析输入 (description, prompt, subagent_type, etc.)      │
│     └─ 获取权限模式                                               │
│                                                                  │
│  2. 分类器决策（可选）                                            │
│     ├─ 检查 agent 类型是否允许                                    │
│     └─ MCP 服务要求检查                                            │
│                                                                  │
│  3. 多智能体 vs 子代理分流                                        │
│     ├─ 团队 + name → spawnTeammate() (tmux/进程)                  │
│     ├─ 工作树隔离 → 创建 git worktree                            │
│     └─ 普通子代理 → runAgent()                                    │
│                                                                  │
│  4. 结果返回                                                      │
│     ├─ 同步完成：status: 'completed'                              │
│     ├─ 后台启动：status: 'async_launched'                        │
│     └─ 队友启动：status: 'teammate_spawned'                       │
└─────────────────────────────────────────────────────────────────┘
```

**队友 Spawn 机制**：

```typescript
// src/tools/shared/spawnMultiAgent.ts
export type SpawnTeammateConfig = {
  name: string
  prompt: string
  team_name?: string
  cwd?: string
  use_splitpane?: boolean
  plan_mode_required?: boolean
  model?: string
  agent_type?: string
}

// 支持的 Backend 类型
export type BackendType = 'tmux' | 'it2' | 'in-process'
```

### 4.2.6 MCPTool：MCP 协议集成

MCPTool 是 Claude Code 与 Model Context Protocol (MCP) 服务器集成的桥梁。

**核心结构**：

```typescript
// src/tools/MCPTool/MCPTool.ts
export const MCPTool = buildTool({
  isMcp: true,
  name: 'mcp',  // 运行时由 mcpClient.ts 覆盖
  maxResultSizeChars: 100_000,

  async call() {
    // 实际执行由 mcpClient.ts 注入
    return { data: '' }
  },

  async checkPermissions(): Promise<PermissionResult> {
    return { behavior: 'passthrough' }  // MCP 工具有自己的权限系统
  },

  renderToolUseProgressMessage,  // MCP 进度渲染
  renderToolResultMessage,
})
```

**MCP 工具特点**：

1. **动态名称**：工具名称格式为 `mcp__serverName__toolName`
2. **独立权限系统**：MCP 工具有自己的权限检查流程
3. **进度支持**：通过 `MCPProgress` 类型报告进度

## 4.3 工具权限系统

### 4.3.1 权限模式与规则

Claude Code 实现了分层的权限管理系统：

```typescript
// src/types/permissions.js
export type ToolPermissionContext = DeepImmutable<{
  mode: PermissionMode  // 'default' | 'auto' | 'bypassPermissions' | 'acceptEdits'
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  shouldAvoidPermissionPrompts?: boolean
  // ...
}>
```

**权限模式**：

| 模式 | 行为 |
|------|------|
| `default` | 首次使用时询问用户 |
| `auto` | 自动批准分类器认为安全的操作 |
| `bypassPermissions` | 跳过所有权限检查（需用户明确启用） |
| `acceptEdits` | 跳过编辑类工具的权限确认 |

### 4.3.2 权限检查点

每个文件系统工具都会在执行前进行权限检查：

```typescript
// src/utils/permissions/filesystem.ts
export async function checkReadPermissionForTool(
  tool: Tool,
  input: any,
  permissionContext: ToolPermissionContext,
): Promise<PermissionDecision> {
  // 1. 检查 alwaysDenyRules
  const denyRule = matchingRuleForInput(filePath, context, 'read', 'deny')
  if (denyRule !== null) {
    return { behavior: 'deny', reason: 'Path is in denied directory' }
  }

  // 2. 检查 alwaysAllowRules
  const allowRule = matchingRuleForInput(filePath, context, 'read', 'allow')
  if (allowRule !== null) {
    return { behavior: 'allow' }
  }

  // 3. 执行分类器检查（如果启用）
  // ...

  return { behavior: 'ask' }  // 默认询问
}
```

**检查点层级**：

```
┌─────────────────────────────────────────────────────────────────┐
│                      权限检查层级                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  规则匹配层                                                      │
│  ├─ alwaysDenyRules  → 立即拒绝                                  │
│  ├─ alwaysAllowRules → 立即允许                                   │
│  └─ alwaysAskRules   → 标记为需要询问                             │
│                                                                  │
│  危险路径层                                                      │
│  ├─ .git 目录                                                    │
│  ├─ .claude 目录                                                 │
│  ├─ 危险文件 (.gitconfig, .mcp.json, etc.)                       │
│  └─ UNC 路径 (防止 NTLM 泄露)                                     │
│                                                                  │
│  分类器层 (可选)                                                  │
│  ├─ BashClassifier - Shell 命令分类                              │
│  └─ 文件操作分类器                                               │
│                                                                  │
│  最终决策                                                      │
│  ├─ allow  → 执行                                               │
│  ├─ deny    → 拒绝并返回原因                                     │
│  └─ ask     → 显示权限对话框                                     │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3.3 路径验证

路径验证是权限系统的核心组成部分：

```typescript
// src/utils/permissions/filesystem.ts
export const DANGEROUS_DIRECTORIES = [
  '.git', '.vscode', '.idea', '.claude',
]

export function matchingRuleForInput(
  filePath: string,
  permissionContext: ToolPermissionContext,
  operation: 'read' | 'edit' | 'write' | 'bash',
  ruleType: 'allow' | 'deny' | 'ask',
): PermissionRule | null {
  // 遍历所有规则，返回匹配项
  // 支持通配符模式匹配
}
```

**通配符匹配**：

```typescript
// src/utils/permissions/shellRuleMatching.ts
export function matchWildcardPattern(
  pattern: string,  // e.g., "src/**/*.ts"
  target: string,   // e.g., "src/components/Button.ts"
): boolean {
  // 将 glob 模式转换为正则表达式
  // 处理 **, *, ?, [] 等通配符
}
```

### 4.3.4 安全策略

**危险文件保护**：

```typescript
// src/utils/permissions/filesystem.ts
export const DANGEROUS_FILES = [
  '.gitconfig', '.gitmodules', '.bashrc', '.bash_profile',
  '.zshrc', '.zprofile', '.profile', '.ripgreprc',
  '.mcp.json', '.claude.json',
]
```

**Shell 命令安全**：

BashTool 实现了多层次的 Shell 安全检查：

1. **语法层面**：解析命令 AST，识别危险模式
2. **语义层面**：识别危险命令（`rm -rf`, `chmod 777` 等）
3. **路径层面**：阻止访问危险路径
4. **分类器层面**：机器学习模型辅助决策

```typescript
// src/tools/BashTool/bashSecurity.ts
const ZSH_DANGEROUS_COMMANDS = new Set([
  'zmodload', 'emulate', 'sysopen', 'sysread', 'syswrite',
  'sysseek', 'zpty', 'ztcp', 'zsocket', 'mapfile',
  'zf_rm', 'zf_mv', 'zf_ln', 'zf_chmod', 'zf_chown',
])
```

## 4.4 工具执行流程详解

### 4.4.1 工具执行序列图

以下序列图展示了工具从选择到结果返回的完整流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                    工具执行完整序列图                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Claude (LLM)                                                    │
│       │                                                          │
│       │ 1. tool_use { name: "Bash", arguments: { command: "ls" }  │
│       ▼                                                          │
│  ToolRegistry                                                   │
│       │                                                          │
│       │ 2. findToolByName(tools, "Bash")                         │
│       ▼                                                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                   工具生命周期                               │  │
│  │                                                            │  │
│  │  3. validateInput() ──────────────────────────────────────┐ │  │
│  │  │  ├─ 参数验证                                          │ │  │
│  │  │  ├─ 路径验证                                          │ │  │
│  │  │  └─ 返回 ValidationResult                            │ │  │
│  │  └───────────────────────────────────────────────────────┘ │  │
│  │                         │                                  │  │
│  │                         ▼                                  │  │
│  │  4. checkPermissions() ────────────────────────────────┐   │  │
│  │  │  ├─ 规则匹配                                        │   │  │
│  │  │  ├─ 分类器决策（可选）                              │   │  │
│  │  │  └─ 返回 PermissionResult                           │   │  │
│  │  └───────────────────────────────────────────────────────┘   │  │
│  │                         │                                  │  │
│  │                         ▼                                  │  │
│  │  5. canUseTool() ────────────────────────────────────────┐   │  │
│  │  │  ├─ PreToolUse Hook                                 │   │  │
│  │  │  ├─ 权限确认（如果需要）                            │   │  │
│  │  │  └─ 返回 boolean                                    │   │  │
│  │  └───────────────────────────────────────────────────────┘   │  │
│  │                         │                                  │  │
│  │                         ▼                                  │  │
│  │  6. call() ────────────────────────────────────────────┐   │  │
│  │  │  ├─ 执行工具逻辑                                    │   │  │
│  │  │  ├─ 进度回调 (onProgress)                          │   │  │
│  │  │  └─ 返回 ToolResult                                 │   │  │
│  │  └───────────────────────────────────────────────────────┘   │  │
│  │                         │                                  │  │
│  │                         ▼                                  │  │
│  │  7. PostToolUse Hook ─────────────────────────────────┐   │  │
│  │  │  └─ 后置处理                                        │   │  │
│  │  └───────────────────────────────────────────────────────┘   │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
│       │                                                          │
│       ▼                                                          │
│  ResultRenderer                                                  │
│       │                                                          │
│       │ 8. renderToolResultMessage()                             │
│       │ 9. mapToolResultToToolResultBlockParam()                 │
│       ▼                                                          │
│  API Response                                                   │
│       │                                                          │
│       │ tool_result { content: "...", type: "tool_result" }      │
│       ▼                                                          │
│  Claude (LLM)                                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4.2 输入验证流程

输入验证是工具执行的第一道防线：

```typescript
// FileEditTool.validateInput 流程 (src/tools/FileEditTool/FileEditTool.ts)
async validateInput(input: FileEditInput, toolUseContext: ToolUseContext) {
  const { file_path, old_string, new_string, replace_all = false } = input

  // 1. 展开路径并规范化
  const fullFilePath = expandPath(file_path)

  // 2. 危险内容检查
  const secretError = checkTeamMemSecrets(fullFilePath, new_string)
  if (secretError) return { result: false, ... }

  // 3. 空操作检查
  if (old_string === new_string) return { result: false, ... }

  // 4. 拒绝规则检查
  const denyRule = matchingRuleForInput(fullFilePath, context, 'edit', 'deny')
  if (denyRule !== null) return { result: false, ... }

  // 5. UNC 路径安全检查
  if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//')) {
    return { result: true }  // defer to permission check
  }

  // 6. 文件大小检查
  const { size } = await fs.stat(fullFilePath)
  if (size > MAX_EDIT_FILE_SIZE) return { result: false, ... }

  // 7. 文件存在性检查
  // ...

  // 8. 读取状态验证
  const readTimestamp = toolUseContext.readFileState.get(fullFilePath)
  if (!readTimestamp || readTimestamp.isPartialView) {
    return { result: false, message: 'File has not been read yet' }
  }

  // 9. 修改时间验证
  // ...

  // 10. 字符串匹配验证
  const actualOldString = findActualString(file, old_string)
  if (!actualOldString) return { result: false, ... }

  // 11. 多重匹配检查
  const matches = file.split(actualOldString).length - 1
  if (matches > 1 && !replace_all) return { result: false, ... }

  return { result: true }
}
```

### 4.4.3 权限检查流程

权限检查在输入验证之后、工具执行之前进行：

```typescript
// src/utils/permissions/filesystem.ts
export async function checkWritePermissionForTool(
  tool: Tool,
  input: any,
  permissionContext: ToolPermissionContext,
): Promise<PermissionDecision> {
  const { file_path } = input
  const fullPath = expandPath(file_path)

  // 1. AlwaysDeny 检查（最高优先级）
  const denyRule = matchingRuleForInput(fullPath, permissionContext, 'edit', 'deny')
  if (denyRule !== null) {
    return {
      behavior: 'deny',
      reason: 'Path matches a deny rule',
      updatedInput: input,
    }
  }

  // 2. AlwaysAllow 检查
  const allowRule = matchingRuleForInput(fullPath, permissionContext, 'edit', 'allow')
  if (allowRule !== null) {
    return { behavior: 'allow', updatedInput: input }
  }

  // 3. AlwaysAsk 检查
  const askRule = matchingRuleForInput(fullPath, permissionContext, 'edit', 'ask')
  if (askRule !== null) {
    return { behavior: 'ask', updatedInput: input }
  }

  // 4. 危险文件/目录检查
  const dangerousMatch = checkDangerousPaths(fullPath)
  if (dangerousMatch !== null) {
    return {
      behavior: 'ask',
      reason: `Path is in protected directory: ${dangerousMatch}`,
      updatedInput: input,
    }
  }

  // 5. 返回需要询问
  return { behavior: 'ask', updatedInput: input }
}
```

### 4.4.4 结果处理与渲染

工具执行完成后，结果通过多个渲染器进行后处理：

```typescript
// 结果渲染器接口
export interface ToolRenderer {
  // React UI 渲染
  renderToolResultMessage?(
    content: Output,
    progressMessagesForMessage: ProgressMessage<P>[],
    options: { style?: 'condensed'; theme: ThemeName; tools: Tools; verbose: boolean },
  ): React.ReactNode

  // API 消息块渲染
  mapToolResultToToolResultBlockParam(
    content: Output,
    toolUseID: string,
  ): ToolResultBlockParam

  // 转录本搜索文本提取
  extractSearchText?(out: Output): string

  // 截断检测
  isResultTruncated?(output: Output): boolean
}
```

**FileEditTool 结果渲染**：

```typescript
// src/tools/FileEditTool/FileEditTool.ts
mapToolResultToToolResultBlockParam(data: FileEditOutput, toolUseID) {
  const { filePath, userModified, replaceAll } = data
  const modifiedNote = userModified
    ? '. The user modified your proposed changes before accepting them.'
    : ''

  if (replaceAll) {
    return {
      tool_use_id: toolUseID,
      type: 'tool_result',
      content: `The file ${filePath} has been updated${modifiedNote}. All occurrences were successfully replaced.`,
    }
  }

  return {
    tool_use_id: toolUseID,
    type: 'tool_result',
    content: `The file ${filePath} has been updated successfully${modifiedNote}.`,
  }
}
```

## 4.5 高级特性

### 4.5.1 工具延迟加载 (Tool Deferral)

当工具数量众多时，Claude Code 支持延迟加载部分工具：

```typescript
// src/Tool.ts
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 延迟加载标记
  readonly shouldDefer?: boolean
  // 强制加载标记（即使延迟加载）
  readonly alwaysLoad?: boolean
}
```

工具搜索（ToolSearch）允许模型在需要时查找和启用延迟的工具。

### 4.5.2 并发安全

工具通过 `isConcurrencySafe` 方法声明其并发安全性：

```typescript
// src/Tool.ts
isConcurrencySafe(input: z.infer<Input>): boolean
```

对于非并发安全的工具，系统会序列化执行，防止竞态条件。

### 4.5.3 进度报告

部分工具支持实时进度报告：

```typescript
// 进度回调签名
export type ToolCallProgress<P extends ToolProgressData = ToolProgressData> = (
  progress: ToolProgress<P>,
) => void

export type ToolProgress<P extends ToolProgressData> = {
  toolUseID: string
  data: P
}
```

**BashProgress 示例**：

```typescript
// src/types/tools.js
export type BashProgress = {
  type: 'bash_progress'
  data: {
    running: boolean
    restart?: boolean
    shell_pid?: number
    timeout_seconds?: number
    truncated?: boolean
    partial_line?: string
  }
}
```

### 4.5.4 工具分组显示

对于 AgentTool 等支持并行调用的工具，系统提供分组渲染：

```typescript
// src/Tool.ts
renderGroupedToolUse?(
  toolUses: Array<{
    param: ToolUseBlockParam
    isResolved: boolean
    isError: boolean
    isInProgress: boolean
    progressMessages: ProgressMessage<P>[]
    result?: { param: ToolResultBlockParam; output: unknown }
  }>,
  options: { shouldAnimate: boolean; tools: Tools },
): React.ReactNode | null
```

## 4.6 小结

本章深入分析了 Claude Code 的工具系统架构，包括：

1. **工具基类设计**：基于 TypeScript 泛型和 Zod 的类型安全设计
2. **工具注册机制**：条件注册与死码消除策略
3. **工具调度流程**：从选择到执行的完整生命周期
4. **权限系统**：多层次的路径和命令安全检查
5. **内置工具**：BashTool、FileReadTool、FileEditTool、GrepTool、AgentTool、MCPTool

Claude Code 的工具系统通过精心设计的安全层、类型系统和灵活的可扩展性，为 AI Agent 提供了强大而安全的环境交互能力。