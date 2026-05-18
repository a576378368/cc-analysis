# 第五章：记忆系统架构

Claude Code 的记忆系统是一套精心设计的多层持久化架构，旨在让 AI Agent 能够在跨会话的维度上积累关于用户、项目和团队的知识。与传统的单一日志系统不同，Claude Code 采用四层记忆分类体系，每一层服务于不同的认知需求，并在存储粒度、访问频率和生命周期上做了明确区分。

本章将深入剖析该记忆系统的架构设计、实现细节和数据流转机制。

---

## 5.1 记忆系统概述

### 5.1.1 多层记忆架构设计理念

Claude Code 的记忆系统并非一个单一的统一存储，而是由多个在语义上独立、在实现上协同的子系统构成。设计者选择了一种分层策略，将记忆按照"可派生性"（derivability）这一核心原则进行分类：凡是能从当前项目状态（如代码、Git 历史、CLAUDE.md 文件）直接推导出的信息，一律不写入记忆；只有那些"非推导性"的信息——例如用户的角色背景、过往的偏好反馈、项目决策的动机——才值得进入记忆系统。

这一原则体现在 `memoryTypes.ts` 中的 `WHAT_NOT_TO_SAVE_SECTION`：

```typescript
export const WHAT_NOT_TO_SAVE_SECTION: readonly string[] = [
  '## What NOT to save in memory',
  '- Code patterns, conventions, architecture, file paths, or project structure — these can be derived by reading the current project state.',
  '- Git history, recent changes, or who-changed-what — `git log` / `git blame` are authoritative.',
  '- Debugging solutions or fix recipes — the fix is in the code; the commit message has the context.',
  '- Anything already documented in CLAUDE.md files.',
  '- Ephemeral task details: in-progress work, temporary state, current conversation context.',
]
```

这种设计哲学避免了两个常见问题：一是记忆膨胀（memory bloat）——当 Agent 不加区分地记录一切时，记忆文件变得臃肿且难以维护；二是记忆污染（memory pollution）——当记忆内容与现实脱节时，Agent 会基于过时信息做出错误判断。

### 5.1.2 四层记忆类型体系

Claude Code 将记忆严格划分为四种类型，每种类型都有明确的语义范围和使用场景：

| 类型 | 描述 | 作用域 | 典型内容 |
|------|------|--------|----------|
| `user` | 关于用户角色、目标和知识的信息 | 始终私有（private） | "用户是数据科学家"、"用户有10年Go经验但首次接触React" |
| `feedback` | 用户给出的行为指导——既包括"不要做什么"也包括"什么方法有效" | 默认为私有，团队共识升为team | "不要在测试中mock数据库，我们曾因此踩坑" |
| `project` | 关于项目状态、目标和决策的信息 | 私有或team，强烈偏向team | "3月5日后进入merge freeze，因为移动团队要切发布分支" |
| `reference` | 外部系统中信息的指针 | 通常为team | "流水线Bug在Linear的'INGEST'项目中跟踪" |

四种类型通过 `<type>` XML 风格块嵌入到系统提示词中，每种类型都包含 `<name>`、`<scope>`（作用域指导）、`<description>`（语义说明）、`<when_to_save>`（保存时机）、`<how_to_use>`（使用方式）和 `<examples>`（示例）六个子字段。

### 5.1.3 架构总览图

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户会话                                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                     System Prompt                               │
│  ┌─────────────┐  ┌──────────────┐  ┌─────────────────────────┐ │
│  │ Auto Memory │  │Session Memory│  │     Agent Memory        │ │
│  │  (Prompt)   │  │  (Context)   │  │     (Prompt)            │ │
│  └─────────────┘  └──────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
          │                  │                    │
          ▼                  ▼                    ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────┐
│  MEMORY.md      │  │ session-memory/ │  │ agent_memory/       │
│  (Index)        │  │ notes.md        │  │ ├── user/           │
│  +              │  │                 │  │ ├── project/       │
│  memories/*.md  │  │ (会话级摘要)      │  │ └── local/         │
│                 │  │                 │  │     (Agent定义绑定) │
│  (Auto持久层)    │  │ (会话级临时)     │  │                    │
└─────────────────┘  └─────────────────┘  └─────────────────────┘
          │                                    │
          │              ┌─────────────────────┘
          ▼              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Team Memory (共享层)                        │
│  ~/.claude/projects/<project>/memory/team/                 │
│  ├── MEMORY.md (Team索引)                                  │
│  └── *.md (团队共享记忆)                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 5.2 Auto Memory 实现

Auto Memory 是 Claude Code 记忆系统中最核心的子系统，它负责持久化跨会话的、基于文件的记忆存储。所有 Auto Memory 数据最终以 Markdown 文件的形式存在于磁盘上，路径为 `~/.claude/projects/<project>/memory/`。

### 5.2.1 MEMORY.md 索引管理

Auto Memory 采用了"索引 + 主题文件"的双层架构。`MEMORY.md` 作为入口索引文件，每一行包含一个记忆文件的链接和简短描述，格式为：

```markdown
- [用户角色：数据科学家](memories/user_role.md) — 专注可观测性/日志
- [反馈：测试不用mock数据库](memories/feedback_testing.md) — 曾导致生产迁移失败
```

这种设计的优势在于：
- **索引可快速扫描**：Agent 可以在系统提示词中直接加载 MEMORY.md 的内容（受限于 200 行和 25KB 的双上限），快速了解有哪些记忆可用
- **主题文件可深度存储**：具体的记忆内容放在独立的 `.md` 文件中，支持完整的 Markdown 格式和详细描述
- **去重机制**：保存新记忆前先扫描现有 MEMORY.md，如果已存在相关记忆则更新而非创建

`memdir.ts` 中的 `loadMemoryPrompt()` 负责构建系统提示词中的记忆部分：

```typescript
export async function loadMemoryPrompt(): Promise<string | null> {
  const autoEnabled = isAutoMemoryEnabled()

  const skipIndex = getFeatureValue_CACHED_MAY_BE_STALE(
    'tengu_moth_copse',
    false,
  )

  if (feature('KAIROS') && autoEnabled && getKairosActive()) {
    // Assistant 模式：每日日志模式
    return buildAssistantDailyLogPrompt(skipIndex)
  }

  if (feature('TEAMMEM')) {
    if (teamMemPaths!.isTeamMemoryEnabled()) {
      const autoDir = getAutoMemPath()
      const teamDir = teamMemPaths!.getTeamMemPath()
      await ensureMemoryDirExists(teamDir)
      // ... 日志统计
      return teamMemPrompts!.buildCombinedMemoryPrompt(
        extraGuidelines,
        skipIndex,
      )
    }
  }

  if (autoEnabled) {
    const autoDir = getAutoMemPath()
    await ensureMemoryDirExists(autoDir)
    return buildMemoryLines(
      'auto memory',
      autoDir,
      extraGuidelines,
      skipIndex,
    ).join('\n')
  }
  // ...
}
```

### 5.2.2 topic memories/*.md 存储

每个具体的记忆以独立文件存在，采用 frontmatter 元数据格式：

```markdown
---
name: 用户角色：数据科学家
description: 用户是数据科学家，目前专注于可观测性和日志系统
type: user
---

这是一个关于用户角色的记忆。

## 背景
用户曾在对话中提到他们正在调查现有的日志系统。
他们倾向于使用结构化日志而非自然语言日志。
```

文件命名采用语义化方式（如 `user_role.md`、`feedback_testing.md`、`project_auth_middleware.md`），而非时间戳或 UUID，这使得 Agent 在扫描时能够快速推断内容主题。

`memoryScan.ts` 中的 `scanMemoryFiles()` 负责扫描目录并读取每个记忆文件的 frontmatter：

```typescript
export async function scanMemoryFiles(
  memoryDir: string,
  signal: AbortSignal,
): Promise<MemoryHeader[]> {
  try {
    const entries = await readdir(memoryDir, { recursive: true })
    const mdFiles = entries.filter(
      f => f.endsWith('.md') && basename(f) !== 'MEMORY.md',
    )

    const headerResults = await Promise.allSettled(
      mdFiles.map(async (relativePath): Promise<MemoryHeader> => {
        const filePath = join(memoryDir, relativePath)
        const { content, mtimeMs } = await readFileInRange(
          filePath,
          0,
          FRONTMATTER_MAX_LINES,
          undefined,
          signal,
        )
        const { frontmatter } = parseFrontmatter(content, filePath)
        return {
          filename: relativePath,
          filePath,
          mtimeMs,
          description: frontmatter.description || null,
          type: parseMemoryType(frontmatter.type),
        }
      }),
    )

    return headerResults
      .filter((r): r is PromiseFulfilledResult<MemoryHeader> => r.status === 'fulfilled')
      .map(r => r.value)
      .sort((a, b) => b.mtimeMs - a.mtimeMs)  // 按修改时间倒序
      .slice(0, MAX_MEMORY_FILES)
  } catch {
    return []
  }
}
```

`scanMemoryFiles` 的实现采用了"读后排序"策略：一次性读取所有文件的头部（前 30 行），解析出 mtime 和 frontmatter，然后排序。这种方式在常见场景（≤200 个文件）下相比"先 stat 排序再读"的策略可以减少一半的系统调用次数。

### 5.2.3 relevant recall 机制

当用户提出一个问题时，Agent 需要判断哪些已存储的记忆与当前查询相关。`findRelevantMemories.ts` 实现了这一机制：

```typescript
export async function findRelevantMemories(
  query: string,
  memoryDir: string,
  signal: AbortSignal,
  recentTools: readonly string[] = [],
  alreadySurfaced: ReadonlySet<string> = new Set(),
): Promise<RelevantMemory[]> {
  const memories = (await scanMemoryFiles(memoryDir, signal)).filter(
    m => !alreadySurfaced.has(m.filePath),
  )
  if (memories.length === 0) {
    return []
  }

  const selectedFilenames = await selectRelevantMemories(
    query,
    memories,
    signal,
    recentTools,
  )
  // ... 返回 path + mtimeMs
}
```

关键设计点包括：

1. **两阶段选择**：先用 `scanMemoryFiles` 扫描获取所有记忆文件的头部元数据，然后通过 `selectRelevantMemories` 调用 Sonnet 模型进行相关性判断
2. **已有记忆过滤**：`alreadySurfaced` 参数避免在同一个会话中重复选择同一记忆文件
3. **工具上下文感知**：当 Agent 最近使用过某个工具（如 `mcp__X__spawn`）时，selector 会避免选择该工具的参考文档——因为对话中已经有实际使用示例了
4. **Telemetry 支持**：`feature('MEMORY_SHAPE_TELEMETRY')` 控制是否记录选择率数据，用于优化模型

### 5.2.4 路径解析与安全

`paths.ts` 实现了记忆目录路径的解析逻辑，包含多层优先级：

```typescript
export const getAutoMemPath = memoize(
  (): string => {
    const override = getAutoMemPathOverride() ?? getAutoMemPathSetting()
    if (override) {
      return override
    }
    const projectsDir = join(getMemoryBaseDir(), 'projects')
    return (
      join(projectsDir, sanitizePath(getAutoMemBase()), AUTO_MEM_DIRNAME) + sep
    ).normalize('NFC')
  },
  () => getProjectRoot(),
)
```

优先级顺序：
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量（Cowork 使用）
2. `autoMemoryDirectory` 配置项（来自 policySettings/localSettings/userSettings）
3. `<memoryBase>/projects/<sanitized-git-root>/memory/`

安全验证通过 `validateMemoryPath()` 实现，包括：
- 拒绝相对路径（防止 `../foo` 遍历）
- 拒绝根目录和近根目录（`/`、`/a`）
- 拒绝 Windows drive-root（`C:`）
- 拒绝 UNC 路径（`\\server\share`）
- 拒绝 null byte 注入

---

## 5.3 Session Memory 实现

Session Memory 是一种会话级别的记忆机制，它在单个会话的生命周期内维护一份关于"当前正在做什么"的实时摘要。与 Auto Memory 的持久化不同，Session Memory 是临时的——它在会话结束时会被丢弃，但其内容会被 Auto Memory 提取机制所消费。

### 5.3.1 会话摘要提取架构

Session Memory 的核心是 `sessionMemory.ts` 中的 `extractSessionMemory` 函数，它通过 `registerPostSamplingHook` 注册为一个 post-sampling hook：

```typescript
export function initSessionMemory(): void {
  if (getIsRemoteMode()) return
  const autoCompactEnabled = isAutoCompactEnabled()

  if (!autoCompactEnabled) {
    return
  }

  // 注册钩子：门控检查在钩子执行时惰性进行
  registerPostSamplingHook(extractSessionMemory)
}
```

当每次 Assistant 响应结束后（post-sampling 阶段），`extractSessionMemory` 被调用，检查是否需要触发记忆提取。

### 5.3.2 基于 Token 阈值的触发机制

Session Memory 的触发需要同时满足两个条件：

```typescript
export function shouldExtractMemory(messages: Message[]): boolean {
  // 1. 检查初始化阈值（首次需要累积足够的上下文）
  const currentTokenCount = tokenCountWithEstimation(messages)
  if (!isSessionMemoryInitialized()) {
    if (!hasMetInitializationThreshold(currentTokenCount)) {
      return false
    }
    markSessionMemoryInitialized()
  }

  // 2. 检查更新阈值（基于上下文增长而非累积量）
  const hasMetTokenThreshold = hasMetUpdateThreshold(currentTokenCount)

  // 3. 检查工具调用次数阈值
  const toolCallsSinceLastUpdate = countToolCallsSince(messages, lastMemoryMessageUuid)
  const hasMetToolCallThreshold =
    toolCallsSinceLastUpdate >= getToolCallsBetweenUpdates()

  // 4. 检查最后一个 Assistant turn 是否有工具调用（确保在自然断点提取）
  const hasToolCallsInLastTurn = hasToolCallsInLastAssistantTurn(messages)

  // 触发条件：
  // - 两个阈值都满足（token AND tool calls），或者
  // - token阈值满足且最后一个turn没有工具调用
  const shouldExtract =
    (hasMetTokenThreshold && hasMetToolCallThreshold) ||
    (hasMetTokenThreshold && !hasToolCallsInLastTurn)

  // ...
}
```

默认配置值定义在 `sessionMemoryUtils.ts`：

```typescript
export const DEFAULT_SESSION_MEMORY_CONFIG: SessionMemoryConfig = {
  minimumMessageTokensToInit: 10000,    // 初始化需要至少10000 tokens
  minimumTokensBetweenUpdate: 5000,    // 更新需要至少增长5000 tokens
  toolCallsBetweenUpdates: 3,          // 至少3次工具调用
}
```

这种设计确保：
- **不会过早触发**：首次需要累积足够的上下文（10000 tokens）才能初始化
- **不会过于频繁**：即使工具调用次数足够，也需要上下文增长达到阈值
- **在自然断点提取**：当最后一个 Assistant turn 没有工具调用时，表明可能是一个自然的交流暂停点，此时触发提取不会打断工作流

### 5.3.3 后台子Agent提取机制

Session Memory 的提取通过 `runForkedAgent` 在后台异步执行：

```typescript
await runForkedAgent({
  promptMessages: [createUserMessage({ content: userPrompt })],
  cacheSafeParams: createCacheSafeParams(context),
  canUseTool: createMemoryFileCanUseTool(memoryPath),
  querySource: 'session_memory',
  forkLabel: 'session_memory',
  overrides: { readFileState: setupContext.readFileState },
})
```

`createMemoryFileCanUseTool` 是一个严格的工具权限工厂：

```typescript
export function createMemoryFileCanUseTool(memoryPath: string): CanUseToolFn {
  return async (tool: Tool, input: unknown) => {
    if (
      tool.name === FILE_EDIT_TOOL_NAME &&
      typeof input === 'object' &&
      input !== null &&
      'file_path' in input
    ) {
      const filePath = input.file_path
      if (typeof filePath === 'string' && filePath === memoryPath) {
        return { behavior: 'allow' as const, updatedInput: input }
      }
    }
    return {
      behavior: 'deny' as const,
      message: `only ${FILE_EDIT_TOOL_NAME} on ${memoryPath} is allowed`,
      decisionReason: {
        type: 'other' as const,
        reason: `only ${FILE_EDIT_TOOL_NAME} on ${memoryPath} is allowed`,
      },
    }
  }
}
```

这确保了后台子Agent **只能编辑会话记忆文件**，无法访问项目代码或其他敏感文件。

### 5.3.4 会话记忆文件格式

会话记忆使用预定义的 Markdown 模板结构化存储：

```markdown
# Session Title
_A short and distinctive 5-10 word descriptive title for the session. Super info dense, no filler_

# Current State
_What is actively being worked on right now? Pending tasks not yet completed. Immediate next steps._

# Task specification
_What did the user ask to build? Any design decisions or other explanatory context_

# Files and Functions
_What are the important files? In short, what do they contain and why are they relevant?_

# Workflow
_What bash commands are usually run and in what order? How to interpret their output if not obvious?_

# Errors & Corrections
_Errors encountered and how they were fixed. What did the user correct? What approaches failed and should not be tried again?_

# Codebase and System Documentation
_What are the important system components? How do they work/fit together?_

# Learnings
_What has worked well? What has not? What to avoid? Do not duplicate items from other sections_

# Key results
_If the user asked a specific output such as an answer to a question, a table, or other document, repeat the exact result here_

# Worklog
_Step by step, what was attempted, done? Very terse summary for each step_
```

每个section都包含一个**斜体描述行**（以 `_` 包裹），这是模板指令而非内容，提取Agent被指示**绝不能修改这些行**：

```typescript
// sessionMemoryUtils.ts
const basePrompt = substituteVariables(promptTemplate, variables)

// "绝不能修改、删除或添加 section headers"
// "绝不能修改或删除斜体描述行"
```

### 5.3.5 更新提示词构建

`prompts.ts` 中的 `buildSessionMemoryUpdatePrompt` 构建更新提示词：

```typescript
export async function buildSessionMemoryUpdatePrompt(
  currentNotes: string,
  notesPath: string,
): Promise<string> {
  const promptTemplate = await loadSessionMemoryPrompt()

  // 分析section大小，生成提醒
  const sectionSizes = analyzeSectionSizes(currentNotes)
  const totalTokens = roughTokenCountEstimation(currentNotes)
  const sectionReminders = generateSectionReminders(sectionSizes, totalTokens)

  // 变量替换
  const variables = {
    currentNotes,
    notesPath,
  }

  const basePrompt = substituteVariables(promptTemplate, variables)
  return basePrompt + sectionReminders
}
```

关键机制包括：
- **Section大小分析**：`analyzeSectionSizes` 计算每个section的token估计值
- **超限警告**：当某个section超过 2000 tokens 或总量超过 12000 tokens 时，在提示词末尾追加警告
- **自动截断**：`truncateSessionMemoryForCompact` 用于在compact消息中插入session memory时截断过长section

---

## 5.4 Agent Memory 实现

Agent Memory 是与特定 Agent 定义绑定的记忆系统。与 Auto Memory 的项目级持久化不同，Agent Memory 绑定到 Agent 本身，并在 Agent 被调用时自动加载到上下文中。

### 5.4.1 三层作用域设计

Agent Memory 按照作用域分为三层：

1. **User Scope**（用户级）：跨所有项目，属于特定用户
2. **Project Scope**（项目级）：属于特定项目下的所有 Agent
3. **Local Scope**（本地级）：仅属于特定 Agent 实例

这种分层设计使得记忆可以在不同粒度上复用：用户级偏好（如"使用Zig而不是Rust"）可以被所有项目继承；项目级约定（如"测试必须用真实数据库"）在该项目下所有 Agent 间共享；Local 级记忆则记录该特定 Agent 的工作状态。

### 5.4.2 与 Agent 定义绑定

Agent Memory 的路径与 Agent 定义文件所在目录相关。当 Agent 定义文件位于 `~/.claude/agents/<agent>.md` 时，其关联的 Agent Memory 位于 `~/.claude/agents/<agent>/memory/`。

### 5.4.3 Snapshot 快照机制

Agent Memory 支持"快照"机制，可以在特定时刻创建记忆的不可变副本。这一机制主要用于：
- **实验性变更前的备份**：在对记忆进行可能有害的修改前，创建快照以便回滚
- **跨分支隔离**：不同分支可以拥有独立的记忆快照，避免相互干扰
- **共享模板**：管理员可以创建预置的记忆模板，新 Agent 可以基于模板初始化

快照文件命名格式：`YYYY-MM-DD_HH-MM-SS_snapshot.md`

---

## 5.5 Team Memory 实现

Team Memory 是 Auto Memory 的扩展，它为团队协作场景提供共享的记忆层。与 Auto Memory 的私有性不同，Team Memory 的内容对项目内所有用户可见和可贡献。

### 5.5.1 团队共享记忆路径

`teamMemPaths.ts` 定义了 Team Memory 的路径：

```typescript
export function getTeamMemPath(): string {
  return (join(getAutoMemPath(), 'team') + sep).normalize('NFC')
}

export function getTeamMemEntrypoint(): string {
  return join(getAutoMemPath(), 'team', 'MEMORY.md')
}
```

Team Memory 作为 Auto Memory 的子目录存在，路径结构为：
```
~/.claude/projects/<project>/memory/team/
```

### 5.5.2 安全验证机制

`teamMemPaths.ts` 实现了两层安全验证来防止符号链接逃逸攻击（PSR M22186）：

```typescript
async function realpathDeepestExisting(absolutePath: string): Promise<string> {
  const tail: string[] = []
  let current = absolutePath
  for (
    let parent = dirname(current);
    current !== parent;
    parent = dirname(current)
  ) {
    try {
      const realCurrent = await realpath(current)
      return tail.length === 0
        ? realCurrent
        : join(realCurrent, ...tail.reverse())
    } catch (e: unknown) {
      const code = getErrnoCode(e)
      if (code === 'ENOENT') {
        // 区分真正不存在和悬空符号链接
        try {
          const st = await lstat(current)
          if (st.isSymbolicLink()) {
            throw new PathTraversalError(
              `Dangling symlink detected (target does not exist): "${current}"`,
            )
          }
        } catch {}
      } else if (code === 'ELOOP') {
        throw new PathTraversalError(`Symlink loop detected in path: "${current}"`)
      }
      // ...
      tail.push(current.slice(parent.length + sep.length))
      current = parent
    }
  }
  return absolutePath
}
```

关键设计：

1. **深度优先遍历**：从目标路径向上遍历到最近的已存在祖先
2. **符号链接检测**：使用 `lstat` 而非 `stat` 来区分悬空符号链接和真实不存在的路径
3. **循环检测**：识别符号链接循环（ELOOP错误）
4. **实时路径解析**：`realpath()` 将符号链接递归解析为真实路径，与 `path.resolve()` 的静态解析形成对比

`validateTeamMemWritePath` 执行两阶段验证：

```typescript
export async function validateTeamMemWritePath(
  filePath: string,
): Promise<string> {
  // 第一阶段：字符串级验证
  const resolvedPath = resolve(filePath)
  const teamDir = getTeamMemPath()
  if (!resolvedPath.startsWith(teamDir)) {
    throw new PathTraversalError(`Path escapes team memory directory`)
  }

  // 第二阶段：符号链接解析验证
  const realPath = await realpathDeepestExisting(resolvedPath)
  if (!(await isRealPathWithinTeamDir(realPath))) {
    throw new PathTraversalError(
      `Path escapes team memory directory via symlink: "${filePath}"`,
    )
  }
  return resolvedPath
}
```

### 5.5.3 作用域选择指导

Team Memory 的记忆类型与 Auto Memory 类似，但增加了 `<scope>` 指导：

- **user 类型**：始终为 private，不存在 team scope
- **feedback 类型**：默认为 private；仅当指导是项目级约定（而非个人偏好）时才升为 team
- **project 类型**：强烈偏向 team
- **reference 类型**：通常为 team

系统提示词中的 `buildCombinedMemoryPrompt` 包含了 Team Memory 的完整提示词：

```typescript
export function buildCombinedMemoryPrompt(
  extraGuidelines?: string[],
  skipIndex = false,
): string {
  const autoDir = getAutoMemPath()
  const teamDir = getTeamMemPath()
  // ... 构建包含 auto + team 两个目录的提示词
}
```

### 5.5.4 同步机制

Team Memory 的同步通过以下机制实现：

1. **会话启动时同步**：`loadMemoryPrompt` 在每次会话开始时加载 Team Memory 的 MEMORY.md 和索引
2. **提取时写入**：`extractMemories` 在后台子Agent中同时处理 auto 和 team 记忆的提取
3. **冲突处理**：当多个用户同时修改同一记忆时，后写入的版本覆盖先前的（基于 mtime）

---

## 5.6 记忆文件结构

### 5.6.1 目录布局

完整的记忆系统目录结构：

```
~/.claude/
├── projects/
│   └── <sanitized-project-root>/
│       └── memory/
│           ├── MEMORY.md                    # Auto Memory 索引
│           ├── memories/                    # Auto Memory 主题文件目录
│           │   ├── user_role.md
│           │   ├── feedback_testing.md
│           │   └── project_auth_middleware.md
│           └── team/
│               ├── MEMORY.md                # Team Memory 索引
│               └── *.md                     # Team Memory 文件
├── agents/
│   └── <agent-name>/
│       └── memory/
│           ├── MEMORY.md                    # Agent Memory 索引
│           └── *.md                         # Agent Memory 文件
└── session-memory/
    └── notes.md                             # Session Memory 文件
```

### 5.6.2 文件格式规范

**主题记忆文件格式**（Auto/Team Memory）：

```markdown
---
name: <记忆名称>
description: <一句话描述，用于相关性判断>
type: <user | feedback | project | reference>
---

<详细记忆内容>
```

**索引文件格式**（MEMORY.md）：

```markdown
- [记忆名称](memories/file.md) — 一句话hook
- [另一个记忆](memories/another.md) — 另一句话hook
```

**Session Memory 文件格式**：使用预定义的结构化模板（见 5.3.4 节）。

### 5.6.3 更新策略

| 操作 | 策略 |
|------|------|
| 新增记忆 | 先扫描 MEMORY.md，如有相关主题记忆则更新，否则创建新文件 |
| 更新记忆 | 直接编辑对应文件，更新 frontmatter 中的 `name`、`description` 和内容 |
| 删除记忆 | 从 MEMORY.md 中移除索引行，删除对应的 `.md` 文件 |
| 索引截断 | 当 MEMORY.md 超过 200 行或 25KB 时截断，并追加警告 |

---

## 5.7 数据流图

### 5.7.1 记忆写入数据流

```
用户输入 → Agent → 决策：保存记忆？
                              │
                              ├── 是 ──→ 检查记忆类型（user/feedback/project/reference）
                              │                   │
                              │                   ▼
                              │            选择作用域（private/team）
                              │                   │
                              │                   ▼
                              │            创建/更新主题文件
                              │                   │
                              │                   ▼
                              │            更新 MEMORY.md 索引
                              │                   │
                              │                   ▼
                              │            extractMemories 后台Agent 确认
                              │
                              └── 否 ──→ 继续正常对话

```

### 5.7.2 记忆读取数据流

```
用户查询 → Agent → 检查是否需要访问记忆
                        │
                        ├── /remember 或明确请求 ──→ 读取 MEMORY.md 索引
                        │                                    │
                        │                                    ▼
                        │                          scanMemoryFiles 扫描所有主题文件
                        │                                    │
                        │                                    ▼
                        │                          findRelevantMemories 选择相关记忆
                        │                                    │
                        │                                    ▼
                        │                          加载选中的记忆文件内容
                        │
                        └── 隐式相关 ──→ findRelevantMemories 自动选择
                                               │
                                               ▼
                                      加载选中的记忆文件内容
```

### 5.7.3 Session Memory 提取数据流

```
Post-Sampling Hook 触发
        │
        ▼
shouldExtractMemory 检查
├── token阈值 ≥ 10000？ ──→ 否 ──→ 返回（不提取）
│                              是
├── token增量 ≥ 5000？ ──→ 否 ──→ 返回（不提取）
│                              是
├── 工具调用 ≥ 3？ ──→ 否 ──→ 检查最后一个turn是否无工具调用
│                              是                      否 ──→ 返回
│                              是
└── 触发提取
        │
        ▼
runForkedAgent（后台子Agent）
        │
        ├── 只允许 Edit 工具访问 notes.md
        │
        ▼
buildSessionMemoryUpdatePrompt
        │
        ├── 读取当前 notes.md
        ├── 分析 section 大小
        └── 生成更新提示词
        │
        ▼
子Agent 执行 Edit 操作
        │
        ▼
更新 lastMemoryMessageUuid
        │
        ▼
提取完成，继续主对话流程
```

---

## 5.8 实现细节汇总

### 5.8.1 核心模块映射

| 功能 | 源文件 |
|------|--------|
| Auto Memory 路径解析 | `src/memdir/paths.ts` |
| 记忆类型定义与提示词 | `src/memdir/memoryTypes.ts` |
| MEMORY.md 索引构建 | `src/memdir/memdir.ts` |
| 记忆文件扫描 | `src/memdir/memoryScan.ts` |
| 相关记忆选择 | `src/memdir/findRelevantMemories.ts` |
| Team Memory 路径与验证 | `src/memdir/teamMemPaths.ts` |
| Team Memory 提示词 | `src/memdir/teamMemPrompts.ts` |
| Session Memory 核心逻辑 | `src/services/SessionMemory/sessionMemory.ts` |
| Session Memory 工具函数 | `src/services/SessionMemory/sessionMemoryUtils.ts` |
| Session Memory 提示词模板 | `src/services/SessionMemory/prompts.ts` |
| Auto Memory 提取服务 | `src/services/extractMemories/extractMemories.ts` |
| 提取服务提示词 | `src/services/extractMemories/prompts.ts` |

### 5.8.2 Feature Flag 对照

| Flag | 功能 |
|------|------|
| `TEAMMEM` | Team Memory 功能开关 |
| `KAIROS` | Assistant 模式每日日志开关 |
| `tengu_passport_quail` | extractMemories 后台提取总开关 |
| `tengu_slate_thimble` | 非交互模式下提取开关 |
| `tengu_herring_clock` | Team Memory 功能开关（备用） |
| `tengu_moth_copse` | 跳过 MEMORY.md 索引更新步骤 |
| `tengu_bramble_lintel` | 提取频率控制（每N个turn） |
| `tengu_coral_fern` | 搜索过去上下文功能 |
| `MEMORY_SHAPE_TELEMETRY` | 记忆选择率遥测 |

### 5.8.3 安全机制

Claude Code 的记忆系统实现了多层安全防护：

1. **路径验证**：所有记忆路径都经过 `validateMemoryPath` 和 `validateTeamMemWritePath` 验证
2. **符号链接防护**：Team Memory 使用 `realpathDeepestExisting` 检测符号链接逃逸
3. **工具权限隔离**：后台提取子Agent 只能访问记忆目录，无法访问项目代码
4. **敏感数据保护**：Team Memory 提示词中明确禁止保存 API keys 或用户凭证
5. **配置隔离**：`projectSettings`（提交到代码库的设置）无法覆盖 `autoMemoryDirectory`

---

本章详细阐述了 Claude Code 记忆系统从架构设计到具体实现的完整面貌。这套系统的核心价值在于：通过将记忆严格分类（四类型）、分层（Auto/Session/Agent/Team）、并以文件为中心持久化，Claude Code 能够在多会话维度上积累知识和上下文，同时保持对"记忆污染"和"记忆膨胀"的有效防范。