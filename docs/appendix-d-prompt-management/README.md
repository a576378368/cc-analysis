# 附录D：Prompt 管理机制

## 概述

Claude Code 的 Prompt 管理机制是一个多层次的系统，负责构建、协调和动态注入各种类型的提示内容。该系统确保 AI 代理能够在不同的上下文环境中正确理解和响应用户请求。

本附录详细分析 Prompt 架构的核心组件：System Prompt 构建、动态 Prompt 注入、Agent Prompt 模板以及版本管理机制。

---

## D.1 Prompt 架构概述

### D.1.1 架构层次

Claude Code 的 Prompt 架构分为以下几个层次：

```
┌─────────────────────────────────────────────────────────────┐
│                   Override System Prompt                    │
│              (最高优先级，完全替换其他所有 Prompt)            │
├─────────────────────────────────────────────────────────────┤
│                   Coordinator System Prompt                 │
│              (协调者模式专用，当启用协调者模式时)            │
├─────────────────────────────────────────────────────────────┤
│                   Agent System Prompt                       │
│         (Agent 模式专用，替换默认 Prompt 或追加到默认)        │
├─────────────────────────────────────────────────────────────┤
│                   Custom System Prompt                      │
│             (用户通过 --system-prompt 指定)                  │
├─────────────────────────────────────────────────────────────┤
│                   Default System Prompt                     │
│                    (标准 Claude Code Prompt)                 │
├─────────────────────────────────────────────────────────────┤
│                   Append System Prompt                      │
│              (始终追加在最后，通过 appendSystemPrompt)        │
└─────────────────────────────────────────────────────────────┘
```

### D.1.2 核心类型定义

**SystemPrompt 类型** (`src/utils/systemPromptType.ts`)

```typescript
export type SystemPrompt = readonly string[] & {
  readonly __brand: 'SystemPrompt'
}

export function asSystemPrompt(value: readonly string[]): SystemPrompt {
  return value as SystemPrompt
}
```

SystemPrompt 是一个品牌化类型（Branded Type），使用 TypeScript 的唯铭型（nominal typing）来区分普通数组和系统提示数组。`asSystemPrompt` 函数将普通数组转换为品牌化类型，确保类型安全。

---

## D.2 System Prompt 构建

### D.2.1 构建策略

`buildEffectiveSystemPrompt` 函数 (`src/utils/systemPrompt.ts`) 实现了 Prompt 构建的核心逻辑，采用优先级分层策略：

**优先级顺序：**

1. **Override System Prompt**（优先级 0）
   - 当设置时，完全替换所有其他 Prompt
   - 主要用于 Loop 模式

2. **Coordinator System Prompt**（优先级 1）
   - 当 `COORDINATOR_MODE` 功能启用且 `CLAUDE_CODE_COORDINATOR_MODE` 环境变量为真时
   - 使用 `getCoordinatorSystemPrompt()` 获取专用协调者提示

3. **Agent System Prompt**（优先级 2）
   - 当 `mainThreadAgentDefinition` 存在时
   - 内置 Agent 使用 `getSystemPrompt({ toolUseContext })` 方法
   - 自定义 Agent 使用 `getSystemPrompt()` 方法

4. **Custom System Prompt**（优先级 3）
   - 用户通过 `--system-prompt` 命令行选项指定

5. **Default System Prompt**（优先级 4）
   - 标准 Claude Code 提示

6. **Append System Prompt**
   - 始终追加在最后（除非 Override 已设置）

### D.2.2 Proactive 模式特殊处理

在 Proactive 模式或 KAIROS 模式下，Agent 指令会**追加**到默认 Prompt 后面，而不是替换默认 Prompt：

```typescript
if (
  agentSystemPrompt &&
  (feature('PROACTIVE') || feature('KAIROS')) &&
  isProactiveActive_SAFE_TO_CALL_ANYWHERE()
) {
  return asSystemPrompt([
    ...defaultSystemPrompt,
    `\n# Custom Agent Instructions\n${agentSystemPrompt}`,
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])
}
```

这种设计确保 Proactive Agent 能够继承基础的自主 Agent 能力，同时在其上添加领域特定的指令。

### D.2.3 记忆体集成

当 Agent 启用了 memory 功能时，系统会记录记忆体加载事件：

```typescript
if (mainThreadAgentDefinition?.memory) {
  logEvent('tengu_agent_memory_loaded', {
    agent_type: mainThreadAgentDefinition.agentType,
    scope: mainThreadAgentDefinition.memory,
    source: 'main-thread',
  })
}
```

---

## D.3 动态 Prompt 注入

### D.3.1 Agent 列表动态注入

Claude Code 支持将 Agent 列表通过消息附件（attachment）注入，而不是直接嵌入工具描述中：

```typescript
export function shouldInjectAgentListInMessages(): boolean {
  if (isEnvTruthy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES)) return true
  if (isEnvDefinedFalsy(process.env.CLAUDE_CODE_AGENT_LIST_IN_MESSAGES))
    return false
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_agent_list_attach', false)
}
```

这种设计的原因是：动态 Agent 列表在 MCP 异步连接、`/reload-plugins` 或权限模式变更时会导致工具描述变化，从而破坏工具 Schema 缓存。

### D.3.2 工具描述格式化

`getToolsDescription` 函数格式化 Agent 的工具描述：

```typescript
function getToolsDescription(agent: AgentDefinition): string {
  const { tools, disallowedTools } = agent
  const hasAllowlist = tools && tools.length > 0
  const hasDenylist = disallowedTools && disallowedTools.length > 0

  if (hasAllowlist && hasDenylist) {
    const denySet = new Set(disallowedTools)
    const effectiveTools = tools.filter(t => !denySet.has(t))
    if (effectiveTools.length === 0) return 'None'
    return effectiveTools.join(', ')
  } else if (hasAllowlist) {
    return tools.join(', ')
  } else if (hasDenylist) {
    return `All tools except ${disallowedTools.join(', ')}`
  }
  return 'All tools'
}
```

### D.3.3 Fork Subagent 机制

当启用 Fork Subagent 功能时，会注入特殊的 Fork 指令和示例：

```typescript
const whenToForkSection = forkEnabled
  ? `

## When to fork

Fork yourself (omit \`subagent_type\`) when the intermediate tool output isn't worth keeping in your context. The criterion is qualitative — "will I need this output again" — not task size.
- **Research**: fork open-ended questions. If research can be broken into independent questions, launch parallel forks in one message. A fork beats a fresh subagent for this — it inherits context and shares your cache.
...
`
  : ''
```

---

## D.4 Agent Prompt 模板

### D.4.1 Agent 定义类型

Claude Code 使用统一的 `AgentDefinition` 类型来定义所有 Agent：

```typescript
export type BaseAgentDefinition = {
  agentType: string           // Agent 类型标识符
  whenToUse: string           // 何时使用此 Agent 的描述
  tools?: string[]           // 允许的工具列表
  disallowedTools?: string[]  // 禁止的工具列表
  skills?: string[]           // 预加载的技能
  mcpServers?: AgentMcpServerSpec[]  // Agent 专用的 MCP 服务器
  hooks?: HooksSettings       // 会话级钩子
  color?: AgentColorName      // Agent 颜色
  model?: string              // 指定模型
  effort?: EffortValue         // 努力级别
  permissionMode?: PermissionMode  // 权限模式
  maxTurns?: number           // 最大轮次
  memory?: AgentMemoryScope   // 持久记忆范围
  isolation?: 'worktree' | 'remote'  // 隔离模式
  omitClaudeMd?: boolean      // 是否省略 CLAUDE.md
  initialPrompt?: string      // 首次用户消息前追加的内容
}
```

### D.4.2 内置 Agent

Claude Code 提供以下内置 Agent：

| Agent 类型 | 用途 | 工具限制 | 模型 |
|-----------|------|---------|------|
| `general-purpose` | 通用任务执行 | 所有工具 | 继承默认 |
| `Explore` | 快速代码库探索 | 禁止写入工具 | haiku (外部) / inherit (内部) |
| `Plan` | 软件架构和计划设计 | 禁止写入工具 | inherit |
| `verification` | 实现验证和测试 | 禁止写入工具 | inherit |
| `statusline-setup` | 状态行配置 | Read, Edit | sonnet |
| `claude-code-guide` | Claude Code 使用指南 | 动态 | haiku |

### D.4.3 Explore Agent

`Explore` Agent 是专为快速文件搜索和分析设计的只读 Agent：

```typescript
export const EXPLORE_AGENT: BuiltInAgentDefinition = {
  agentType: 'Explore',
  whenToUse: EXPLORE_WHEN_TO_USE,
  disallowedTools: [
    AGENT_TOOL_NAME,
    EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  source: 'built-in',
  baseDir: 'built-in',
  model: process.env.USER_TYPE === 'ant' ? 'inherit' : 'haiku',
  omitClaudeMd: true,  // 不需要 CLAUDE.md 上下文
  getSystemPrompt: () => getExploreSystemPrompt(),
}
```

关键特性：
- **只读模式**：严格禁止任何文件修改操作
- **快速响应**：通过并行工具调用最大化效率
- **省略 CLAUDE.md**：不需要项目的 CLAUDE.md 上下文

### D.4.4 Verification Agent

`Verification` Agent 专注于验证实现工作的正确性，采用对抗性测试策略：

```typescript
export const VERIFICATION_AGENT: BuiltInAgentDefinition = {
  agentType: 'verification',
  whenToUse: VERIFICATION_WHEN_TO_USE,
  color: 'red',
  background: true,
  disallowedTools: [
    AGENT_TOOL_NAME,
    EXIT_PLAN_MODE_TOOL_NAME,
    FILE_EDIT_TOOL_NAME,
    FILE_WRITE_TOOL_NAME,
    NOTEBOOK_EDIT_TOOL_NAME,
  ],
  source: 'built-in',
  baseDir: 'built-in',
  model: 'inherit',
  getSystemPrompt: () => VERIFICATION_SYSTEM_PROMPT,
  criticalSystemReminder_EXPERIMENTAL:
    'CRITICAL: This is a VERIFICATION-ONLY task. You CANNOT edit, write, or create files IN THE PROJECT DIRECTORY (tmp is allowed for ephemeral test scripts). You MUST end with VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL.',
}
```

### D.4.5 自定义 Agent 解析

自定义 Agent 从 Markdown 文件或 JSON 配置中解析：

```typescript
export function parseAgentFromMarkdown(
  filePath: string,
  baseDir: string,
  frontmatter: Record<string, unknown>,
  content: string,
  source: SettingSource,
): CustomAgentDefinition | null {
  // 解析 frontmatter 中的各种配置
  // content 作为 systemPrompt
  const agentDef: CustomAgentDefinition = {
    baseDir,
    agentType: agentType,
    whenToUse: whenToUse,
    // ... 其他字段
    getSystemPrompt: () => {
      if (isAutoMemoryEnabled() && memory) {
        const memoryPrompt = loadAgentMemoryPrompt(agentType, memory)
        return systemPrompt + '\n\n' + memoryPrompt
      }
      return systemPrompt
    },
    source,
    filename,
    // ...
  }
  return agentDef
}
```

---

## D.5 Prompt 版本管理

### D.5.1 Agent 定义缓存

`getAgentDefinitionsWithOverrides` 函数使用 memoization 来缓存 Agent 定义：

```typescript
export const getAgentDefinitionsWithOverrides = memoize(
  async (cwd: string): Promise<AgentDefinitionsResult> => {
    // 加载内置 Agent
    // 加载插件 Agent
    // 加载自定义 Agent（从 Markdown 文件）
    // 合并所有 Agent 定义
    // 返回活跃 Agent 列表和完整列表
  },
)
```

缓存可以通过 `clearAgentDefinitionsCache` 函数清除：

```typescript
export function clearAgentDefinitionsCache(): void {
  getAgentDefinitionsWithOverrides.cache.clear?.()
  clearPluginAgentCache()
}
```

### D.5.2 功能开关集成

Claude Code 使用 `feature` 函数和 GrowthBook 进行功能开关管理：

```typescript
// 协调者模式
if (feature('COORDINATOR_MODE')) {
  if (isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)) {
    const { getCoordinatorSystemPrompt } = require('../coordinator/coordinatorMode.js')
    return asSystemPrompt([getCoordinatorSystemPrompt(), ...])
  }
}

// 内置 Explore/Plan Agent
if (feature('BUILTIN_EXPLORE_PLAN_AGENTS')) {
  return getFeatureValue_CACHED_MAY_BE_STALE('tengu_amber_stoat', true)
}

// 验证 Agent
if (feature('VERIFICATION_AGENT')) {
  if (getFeatureValue_CACHED_MAY_BE_STALE('tengu_hive_evidence', false)) {
    agents.push(VERIFICATION_AGENT)
  }
}
```

### D.5.3 Zod Schema 验证

Agent 定义使用 Zod 进行运行时验证：

```typescript
const AgentJsonSchema = lazySchema(() =>
  z.object({
    description: z.string().min(1),
    tools: z.array(z.string()).optional(),
    disallowedTools: z.array(z.string()).optional(),
    prompt: z.string().min(1),
    model: z.string().trim().min(1).transform(m => m.toLowerCase() === 'inherit' ? 'inherit' : m).optional(),
    effort: z.union([z.enum(EFFORT_LEVELS), z.number().int()]).optional(),
    permissionMode: z.enum(PERMISSION_MODES).optional(),
    mcpServers: z.array(AgentMcpServerSpecSchema()).optional(),
    hooks: HooksSchema().optional(),
    maxTurns: z.number().int().positive().optional(),
    skills: z.array(z.string()).optional(),
    initialPrompt: z.string().optional(),
    memory: z.enum(['user', 'project', 'local']).optional(),
    background: z.boolean().optional(),
    isolation: z.enum(['worktree', 'remote']).optional(),
  }),
)
```

`lazySchema` 函数用于打破循环依赖链：`AppState -> loadAgentsDir -> settings/types -> HooksSchema -> AppState`

---

## D.6 协调者模式 Prompt

### D.6.1 协调者系统提示

`getCoordinatorSystemPrompt` 函数 (`src/coordinator/coordinatorMode.ts`) 生成协调者专用的系统提示：

```typescript
export function getCoordinatorSystemPrompt(): string {
  return `You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

## 1. Your Role

You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
...
`
}
```

### D.6.2 工作流阶段

协调者将任务分解为以下阶段：

| 阶段 | 执行者 | 目的 |
|------|--------|------|
| Research | Workers (并行) | 调查代码库，查找文件，理解问题 |
| Synthesis | Coordinator | 阅读发现，理解问题，制定实现规格 |
| Implementation | Workers | 根据规格进行针对性修改，提交 |
| Verification | Workers | 测试变更是否有效 |

### D.6.3 续接 vs 启动决策

协调者根据上下文重叠度决定是继续现有 Worker 还是启动新的：

| 情况 | 机制 | 原因 |
|------|------|------|
| 研究恰好覆盖需要编辑的文件 | **继续** (SendMessage) | Worker 已在上下文中且现在有明确计划 |
| 研究范围广但实现范围窄 | **启动新** | 避免引入探索噪音 |
| 纠正失败或扩展近期工作 | **继续** | Worker 有错误上下文 |
| 验证另一个 Worker 刚写的代码 | **启动新** | 验证者应独立审视 |
| 第一次尝试完全错误 | **启动新** | 错误上下文会污染重试 |
| 完全无关的任务 | **启动新** | 无有用上下文可复用 |

---

## D.7 Prompt 优化策略

### D.7.1 缓存优化

Claude Code 通过多种机制优化 Prompt 缓存：

1. **Agent 列表外置**：将 Agent 列表从工具描述移到附件消息，避免 MCP/插件变更时缓存失效
2. **静态工具描述**：工具 Schema 变化不会导致所有 Agent 的系统提示失效
3. **按需懒加载**：使用 `lazySchema` 延迟 Schema 加载，避免循环依赖

### D.7.2 上下文缩减

部分 Agent 可以省略 CLAUDE.md 上下文以节省 Token：

```typescript
// Explore 是快速只读搜索 Agent，不需要 CLAUDE.md 中的提交/PR/lint 规则
omitClaudeMd: true,

// Plan 是只读的，可以直接读取 CLAUDE.md 如果需要约定
omitClaudeMd: true,
```

### D.7.3 环境适配

内置 Agent 根据构建目标调整工具和模型：

```typescript
// Ant 原生构建：find/grep 替换为嵌入式 bfs/ugrep
const embedded = hasEmbeddedSearchTools()
const globGuidance = embedded
  ? `- Use \`find\` via ${BASH_TOOL_NAME} for broad file pattern matching`
  : `- Use ${GLOB_TOOL_NAME} for broad file pattern matching`
```

---

## D.8 总结

Claude Code 的 Prompt 管理机制通过以下设计实现灵活性和可维护性：

1. **分层优先级**：Override > Coordinator > Agent > Custom > Default > Append
2. **品牌化类型**：使用 `SystemPrompt` 唯铭类型确保类型安全
3. **动态注入**：Agent 列表通过附件消息注入以优化缓存
4. **统一 Agent 模型**：内置和自定义 Agent 使用统一的 `AgentDefinition` 类型
5. **功能开关集成**：通过 feature flags 和 GrowthBook 实现灰度发布
6. **Schema 验证**：使用 Zod 进行运行时验证和懒加载

这套机制使得 Claude Code 能够支持多种运行模式（普通、协调者、Proactive），同时保持代码的模块化和可测试性。