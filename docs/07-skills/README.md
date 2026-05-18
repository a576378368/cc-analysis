# 第七章：Skills 机制与 Skills 系统

> Skills 是 Claude Code 平台化扩展的核心机制，通过文件系统、内建打包和 MCP 协议三种方式为 Agent 注入领域特定能力。本章深入分析 Skills 的架构设计、加载机制、执行流程和内建技能实现。

## 1. Skills 系统概述

### 1.1 什么是 Skills

Skills（技能）是 Claude Code Agent 扩展其能力的核心机制。通过 Skills，用户和开发者可以创建自定义的 slash 命令，为 Agent 添加特定领域的知识和行为规范。Skills 本质上是一个带有 YAML frontmatter 的 Markdown 文件，其中定义了技能的元数据（如名称、描述、可用工具等）和具体的 Prompt 内容。

Claude Code 支持三种技能来源：
- **File-based Skills**：通过文件系统加载的技能，存储在项目的 `.claude/skills/`、用户的 `~/.claude/skills/` 或托管配置的 `.claude/skills/` 目录中
- **Bundled Skills**：内建打包的技能，随 Claude Code CLI 一起发布，提供开箱即用的功能
- **MCP Skills**：通过 Model Context Protocol（MCP）协议映射外部工具和服务为技能

### 1.2 Skills 在 Agent 扩展中的作用

Skills 扩展机制使 Agent 能够：

1. **自定义行为**：通过特定的 Prompt 引导 Agent 以特定方式处理任务
2. **工具限定**：使用 `allowed-tools` frontmatter 控制技能可使用的工具集合
3. **条件激活**：通过 `paths` frontmatter 实现基于文件路径的条件激活
4. **上下文注入**：通过 `shell` frontmatter 在技能内容中嵌入 Shell 命令执行
5. **子 Agent 分叉**：使用 `context: fork` 将技能作为独立子 Agent 运行

### 1.3 与传统命令的区别

| 特性 | 传统命令 | Skills |
|------|---------|--------|
| 定义方式 | TypeScript 代码 | Markdown + YAML frontmatter |
| 部署方式 | 编译进 CLI | 文件系统或配置 |
| 灵活性 | 低（需要代码修改） | 高（用户可自定义） |
| 可共享性 | 困难 | 容易（复制文件即可） |
| 工具控制 | 固定 | 可配置 |
| 执行模式 | 仅内联 | 内联或分叉 |

## 2. 技能来源

### 2.1 File-based 技能（文件系统）

File-based 技能是最常见的技能类型，存储在文件系统中。Claude Code 从多个目录并行加载技能：

```
加载优先级（从高到低）：
1. 托管策略目录: <managed-path>/.claude/skills/
2. 用户目录: ~/.claude/skills/
3. 项目目录: <project>/.claude/skills/
4. 额外目录: <additional-dirs>/.claude/skills/
```

**目录结构要求**：File-based 技能必须采用目录格式：
```
skills/
└── <skill-name>/
    └── SKILL.md
```

每个技能必须放在以其名称命名的独立目录中，且必须包含 `SKILL.md` 文件。不支持单个 `.md` 文件格式。

### 2.2 Bundled 技能（内建打包）

Bundled Skills 是编译进 Claude Code CLI 二进制文件的内置技能，通过 `registerBundledSkill()` 函数注册。源码位于 `src/skills/bundled/` 目录。

**已内建技能列表**：

| 技能名称 | 功能描述 | 特性标志 |
|---------|---------|---------|
| `updateConfig` | 配置管理（权限、环境变量、Hooks 等） | - |
| `keybindings` | 键盘快捷键自定义 | - |
| `verify` | 代码验证 | - |
| `debug` | 调试辅助 | - |
| `loremIpsum` | 占位文本生成 | - |
| `skillify` | 将现有 Prompt 转换为 Skills | - |
| `remember` | 记忆管理 | - |
| `simplify` | 代码审查与重构 | - |
| `batch` | 批量处理任务 | - |
| `stuck` | 协助解决阻塞问题 | - |
| `dream` | （KAIROS 特性）梦境生成 | KAIROS/KAIROS_DREAM |
| `hunter` | （REVIEW_ARTIFACT 特性）代码猎手 | REVIEW_ARTIFACT |
| `loop` | （AGENT_TRIGGERS 特性）循环任务 | AGENT_TRIGGERS |
| `scheduleRemoteAgents` | 远程 Agent 调度 | AGENT_TRIGGERS_REMOTE |
| `claudeApi` | Claude API 开发 | BUILDING_CLAUDE_APPS |
| `claudeInChrome` | Chrome 扩展集成 | 自动检测 |
| `runSkillGenerator` | 技能生成器 | RUN_SKILL_GENERATOR |

### 2.3 MCP Skills（协议映射）

MCP（Model Context Protocol）技能通过 MCP 协议将外部工具和服务映射为 Claude Code 的技能。MCP 技能构建器在 `src/skills/mcpSkillBuilders.ts` 中定义。

**关键设计**：`mcpSkillBuilders.ts` 是一个依赖图叶子模块，只导入类型，不导入任何实现，避免了模块循环依赖问题。该模块使用写一次注册模式，通过 `registerMCPSkillBuilders()` 注册 `createSkillCommand` 和 `parseSkillFrontmatterFields` 两个函数供 MCP 技能发现使用。

## 3. 技能发现机制

### 3.1 多目录并行加载

技能加载的核心函数是 `getSkillDirCommands()`，位于 `src/skills/loadSkillsDir.ts`。该函数使用 `memoize` 缓存确保每个工作目录只加载一次。

```typescript
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    // 并行加载所有目录的技能
    const [
      managedSkills,      // 托管策略
      userSkills,         // 用户配置
      projectSkillsNested, // 项目级
      additionalSkillsNested, // 额外目录
      legacyCommands,     // 遗留 /commands/ 目录
    ] = await Promise.all([...])
  }
)
```

**加载流程图**：

```
getSkillDirCommands(cwd)
    │
    ├── 检查 --bare 模式
    │   └── 若为 bare：仅加载 --add-dir 路径，返回
    │
    └── 并行执行五个加载源
        │
        ├── loadSkillsFromSkillsDir(managedSkillsDir, 'policySettings')
        ├── loadSkillsFromSkillsDir(userSkillsDir, 'userSettings')
        ├── Promise.all(projectSkillsDirs.map(dir => 
        │       loadSkillsFromSkillsDir(dir, 'projectSettings')))
        ├── Promise.all(additionalDirs.map(dir => 
        │       loadSkillsFromSkillsDir(join(dir, '.claude', 'skills'), ...)))
        │
        └── loadSkillsFromCommandsDir(cwd)  // 遗留兼容
```

### 3.2 去重机制（inode 级别）

相同文件通过不同路径访问（如 symlink 或重叠的父目录）时，技能系统使用 `realpath()` 解析后的规范路径进行去重：

```typescript
// 获取文件的规范路径（解析 symlink）
async function getFileIdentity(filePath: string): Promise<string | null> {
  try {
    return await realpath(filePath)
  } catch {
    return null
  }
}

// 使用 Map 存储已见文件 ID
const seenFileIds = new Map<string, SettingSource>()

for (const entry of allSkillsWithPaths) {
  const fileId = await getFileIdentity(entry.filePath)
  if (seenFileIds.has(fileId)) {
    // 跳过重复技能
    continue
  }
  seenFileIds.set(fileId, entry.skill.source)
}
```

**为什么使用 realpath 而不是 inode？**

代码注释解释：某些虚拟/容器/NFS 文件系统报告不可靠的 inode 值（如 inode 0 或 ExFAT 上的精度丢失），而 `realpath` 解析 symlink 后得到的规范路径是文件系统无关的。

### 3.3 memoize 缓存

使用 `lodash-es/memoize` 缓存技能加载结果：

```typescript
export const getSkillDirCommands = memoize(
  async (cwd: string): Promise<Command[]> => {
    // 加载逻辑
  }
)

// 清除缓存（测试用）
export function clearSkillCaches() {
  getSkillDirCommands.cache?.clear?.()
  loadMarkdownFilesForSubdir.cache?.clear?.()
  conditionalSkills.clear()
  activatedConditionalSkillNames.clear()
}
```

## 4. 技能解析

### 4.1 Frontmatter 字段

技能文件顶部使用 YAML frontmatter 定义元数据：

```yaml
---
name: My Skill Display Name    # 可选：显示名称（不同于目录名）
description: 技能描述          # 必填：简短描述
when_to_use: 何时使用          # 可选：使用提示
argument-hint: <args>          # 可选：参数格式提示
allowed-tools: [Read, Write]   # 可选：允许的工具列表
model: sonnet                  # 可选：指定模型
disable-model-invocation: false # 可选：禁用模型调用
user-invocable: true          # 可选：用户是否可调用
context: inline                # 可选：inline 或 fork
agent: general-purpose        # 可选：fork 时使用的 Agent 类型
paths:                         # 可选：条件激活路径
  - src/**/*.ts
  - "!**/*.test.ts"
effort: medium                 # 可选：努力级别
shell: bash                    # 可选：bash 或 powershell
hooks:                         # 可选：Hooks 配置
  PreToolUse:
    - match:
        tool: Write
      async handler: ...
---
```

**完整 Frontmatter 字段列表**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | string | 显示名称 |
| `description` | string | 技能描述 |
| `when_to_use` | string | 使用提示 |
| `argument-hint` | string | 参数格式提示 |
| `allowed-tools` | string[] | 允许的工具列表 |
| `model` | string | 指定模型 |
| `user-invocable` | boolean | 用户是否可调用 |
| `context` | inline/fork | 执行上下文 |
| `agent` | string | fork 时使用的 Agent |
| `paths` | string[] | 条件激活路径 |
| `effort` | string | 努力级别 |
| `shell` | bash/powershell | Shell 类型 |
| `hooks` | object | Hooks 配置 |
| `arguments` | string[] | 参数名列表 |
| `version` | string | 版本号 |

### 4.2 SKILL.md 格式

`SKILL.md` 文件格式示例：

```markdown
---
name: Simplify Code Review
description: Review code for reuse, quality, and efficiency
allowed-tools: [Read, Edit, Bash, Agent]
effort: medium
when_to_use: When you want to improve code quality
---

# Simplify: Code Review and Cleanup

Review all changed files for reuse, quality, and efficiency.

## Phase 1: Identify Changes

Run `git diff` to see what changed...

## Phase 2: Launch Review Agents

Use the Agent tool to launch concurrent review agents...
```

### 4.3 条件技能（path-filtered）

条件技能通过 `paths` frontmatter 定义，当模型操作匹配的文件时自动激活：

```typescript
// 条件技能存储
const conditionalSkills = new Map<string, Command>()
const activatedConditionalSkillNames = new Set<string>()

// 当文件被访问时，激活匹配的条件技能
export function activateConditionalSkillsForPaths(
  filePaths: string[],
  cwd: string
): string[] {
  for (const [name, skill] of conditionalSkills) {
    const skillIgnore = ignore().add(skill.paths)
    for (const filePath of filePaths) {
      const relativePath = relative(cwd, filePath)
      if (skillIgnore.ignores(relativePath)) {
        dynamicSkills.set(name, skill)
        conditionalSkills.delete(name)
        activatedConditionalSkillNames.add(name)
        break
      }
    }
  }
}
```

**动态技能发现**：当访问项目中的文件时，系统会从文件路径向上遍历，动态发现 `.claude/skills/` 目录：

```typescript
export async function discoverSkillDirsForPaths(
  filePaths: string[],
  cwd: string
): Promise<string[]> {
  for (const filePath of filePaths) {
    let currentDir = dirname(filePath)
    while (currentDir.startsWith(resolvedCwd + pathSep)) {
      const skillDir = join(currentDir, '.claude', 'skills')
      if (!dynamicSkillDirs.has(skillDir)) {
        dynamicSkillDirs.add(skillDir)
        try {
          await fs.stat(skillDir)
          // 检查是否被 gitignore
          if (await isPathGitignored(currentDir, resolvedCwd)) continue
          newDirs.push(skillDir)
        } catch {}
      }
      currentDir = dirname(currentDir)
    }
  }
}
```

## 5. Shell 执行集成

### 5.1 prompt 内嵌 Shell

Skills 支持在 Markdown 内容中嵌入 Shell 命令，使用两种语法：

**代码块语法**：
````markdown
```!
git status
ls -la
```
````

**内联语法**：
```markdown
The current directory is !`pwd`
```

### 5.2 命令执行流程

Shell 命令执行由 `executeShellCommandsInPrompt()` 函数处理（`src/utils/promptShellExecution.ts`）：

```typescript
export async function executeShellCommandsInPrompt(
  text: string,
  context: ToolUseContext,
  slashCommandName: string,
  shell?: FrontmatterShell
): Promise<string> {
  // 解析 Shell 类型（bash 或 powershell）
  const shellTool = shell === 'powershell' && isPowerShellToolEnabled()
    ? getPowerShellTool()
    : BashTool

  // 提取命令
  const blockMatches = text.matchAll(BLOCK_PATTERN)  // ```! ... ```
  const inlineMatches = text.includes('!`') 
    ? text.matchAll(INLINE_PATTERN)  // !`...`
    : []

  // 并行执行所有命令
  await Promise.all([...blockMatches, ...inlineMatches].map(async match => {
    const command = match[1]?.trim()
    
    // 权限检查
    const permissionResult = await hasPermissionsToUseTool(
      shellTool, { command }, context, ...
    )
    
    if (permissionResult.behavior !== 'allow') {
      throw new MalformedCommandError(...)
    }
    
    // 执行命令
    const { data } = await shellTool.call({ command }, context)
    
    // 替换原文中的命令为输出
    result = result.replace(match[0], () => output)
  }))
}
```

**执行流程图**：

```
技能内容文本
    │
    ├── 正则匹配 ```! ... ``` 代码块
    └── 正则匹配 !`...` 内联命令（仅当文本包含 !` 时扫描）
    │
    ├── 权限检查 hasPermissionsToUseTool()
    │   └── 检查 alwaysAllowRules 中的工具列表
    │
    ├── Shell 执行（BashTool 或 PowerShellTool）
    │   └── 使用 processToolResultBlock 处理结果
    │
    └── 结果注入
        └── String.replace() 替换命令为输出
```

### 5.3 结果注入

Shell 命令执行后，结果通过 `processToolResultBlock()` 处理并注入回技能内容：

```typescript
const toolResultBlock = await processToolResultBlock(shellTool, data, randomUUID())
const output = typeof toolResultBlock.content === 'string'
  ? toolResultBlock.content
  : formatBashOutput(data.stdout, data.stderr)

result = result.replace(match[0], () => output)
```

**安全考虑**：

1. **权限隔离**：Shell 命令通过 `hasPermissionsToUseTool()` 检查权限，使用 `alwaysAllowRules.command` 作为允许列表
2. **MCP 禁用**：`loadedFrom === 'mcp'` 时不执行内嵌 Shell 命令（MCP 技能是远程的、不可信的）
3. **变量替换**：`${CLAUDE_SKILL_DIR}` 和 `${CLAUDE_SESSION_ID}` 在执行前被替换为实际值

```typescript
// 替换 ${CLAUDE_SKILL_DIR}
if (baseDir) {
  const skillDir = process.platform === 'win32' 
    ? baseDir.replace(/\\/g, '/') 
    : baseDir
  finalContent = finalContent.replace(/\$\{CLAUDE_SKILL_DIR\}/g, skillDir)
}

// 替换 ${CLAUDE_SESSION_ID}
finalContent = finalContent.replace(
  /\$\{CLAUDE_SESSION_ID\}/g,
  getSessionId()
)
```

## 6. 内建技能一览

### 6.1 技能注册机制

内建技能通过 `registerBundledSkill()` 注册（`src/skills/bundledSkills.ts`）：

```typescript
export type BundledSkillDefinition = {
  name: string
  description: string
  aliases?: string[]
  whenToUse?: string
  argumentHint?: string
  allowedTools?: string[]
  model?: string
  disableModelInvocation?: boolean
  userInvocable?: boolean
  isEnabled?: () => boolean  // 动态启用检查
  hooks?: HooksSettings
  context?: 'inline' | 'fork'
  agent?: string
  files?: Record<string, string>  // 提取到磁盘的参考文件
  getPromptForCommand: (args: string, context: ToolUseContext) 
    => Promise<ContentBlockParam[]>
}
```

### 6.2 内建技能列表

| 技能名 | 源文件 | 功能 | 特性标志 |
|-------|--------|------|---------|
| `updateConfig` | updateConfig.ts | 配置管理 | - |
| `keybindings` | keybindings.ts | 快捷键管理 | - |
| `verify` | verify.ts | 代码验证 | - |
| `debug` | debug.ts | 调试辅助 | - |
| `loremIpsum` | loremIpsum.ts | 占位文本 | - |
| `skillify` | skillify.ts | Prompt 转技能 | - |
| `remember` | remember.ts | 记忆管理 | - |
| `simplify` | simplify.ts | 代码审查重构 | - |
| `batch` | batch.ts | 批量处理 | - |
| `stuck` | stuck.ts | 阻塞协助 | - |
| `dream` | dream.ts | 梦境生成 | KAIROS/KAIROS_DREAM |
| `hunter` | hunter.ts | 代码猎手 | REVIEW_ARTIFACT |
| `loop` | loop.ts | 循环任务 | AGENT_TRIGGERS |
| `scheduleRemoteAgents` | scheduleRemoteAgents.ts | 远程调度 | AGENT_TRIGGERS_REMOTE |
| `claudeApi` | claudeApi.ts | API 开发 | BUILDING_CLAUDE_APPS |
| `claudeInChrome` | claudeInChrome.ts | Chrome 扩展 | 自动检测 |
| `runSkillGenerator` | runSkillGenerator.ts | 技能生成 | RUN_SKILL_GENERATOR |

### 6.3 Bundled Skill 示例：simplify

```typescript
export function registerSimplifySkill(): void {
  registerBundledSkill({
    name: 'simplify',
    description: 'Review changed code for reuse, quality, and efficiency, then fix any issues found.',
    userInvocable: true,
    async getPromptForCommand(args) {
      let prompt = SIMPLIFY_PROMPT
      if (args) {
        prompt += `\n\n## Additional Focus\n\n${args}`
      }
      return [{ type: 'text', text: prompt }]
    },
  })
}
```

**参考文件提取机制**：某些 bundled skills 包含需要在磁盘上提取的参考文件（如模板、配置文件）。使用 `files` 字段定义：

```typescript
// files: Record<string, string> - key 是相对路径，value 是内容
files: {
  'template.json': '{"name": "default"}',
  'config.yml': 'version: 1\n...'
}

// 提取后，技能内容会添加基础目录前缀
const blocks = await inner(args, ctx)
return prependBaseDir(blocks, extractedDir)
```

## 附录：类型定义

### Command 类型（技能）

```typescript
interface Command {
  type: 'prompt'
  name: string
  description: string
  hasUserSpecifiedDescription: boolean
  allowedTools: string[]
  argumentHint?: string
  argNames?: string[]
  whenToUse?: string
  version?: string
  model?: string
  disableModelInvocation: boolean
  userInvocable: boolean
  context?: 'inline' | 'fork'
  agent?: string
  effort?: EffortValue
  paths?: string[]
  contentLength: number
  isHidden: boolean
  progressMessage: string
  source: PromptCommand['source']
  loadedFrom: LoadedFrom
  hooks?: HooksSettings
  skillRoot?: string
  userFacingName(): string
  getPromptForCommand(args: string, context: ToolUseContext): Promise<ContentBlockParam[]>
}
```

### LoadedFrom 类型

```typescript
type LoadedFrom = 
  | 'commands_DEPRECATED'  // 旧版 /commands/ 目录
  | 'skills'               // .claude/skills/ 目录
  | 'plugin'               // 插件
  | 'managed'              // 托管策略
  | 'bundled'              // 内建技能
  | 'mcp'                  // MCP 协议
```