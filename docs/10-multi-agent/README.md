---
layout: default
title: 第十章：多Agent系统与协作机制
nav_order: 10
---

# 第十章：多Agent系统与协作机制

> 本章基于 Claude Code 源码中 `src/tools/AgentTool/`、`src/tools/TeamCreateTool/`、`src/tools/SendMessageTool/`、`src/coordinator/`、`src/utils/swarm/` 等核心模块的深度分析，讲解多Agent系统的架构设计与工程实现。

## 10.1 多Agent系统概述

Claude Code 的多Agent系统（Multi-Agent System）是其分布式协作能力的基础。与传统的单Agent架构不同，Claude Code 支持创建由一个**团队主导者（Team Lead）**和多个**工作Agent（Worker）**组成的协作网络。

### 10.1.1 系统角色划分

```
┌─────────────────────────────────────────────────────────────────┐
│                      Claude Code Session                         │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                     Team Lead                            │  │
│   │  - 主会话Agent（人类交互入口）                             │  │
│   │  - 负责任务分发与结果汇总                                  │  │
│   │  - 持有 SendMessageTool、TaskStopTool                    │  │
│   │  - 可以是 Coordinator 模式或 Normal 模式                   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                              │                                   │
│              ┌───────────────┼───────────────┐                  │
│              ▼               ▼               ▼                  │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │
│   │    Worker A   │  │    Worker B   │  │    Worker C   │      │
│   │  (独立工作目录)│  │  (独立工作目录)│  │  (独立工作目录)│      │
│   │  - 执行任务    │  │  - 执行任务    │  │  - 执行任务    │      │
│   │  - 报告结果    │  │  - 报告结果    │  │  - 报告结果    │      │
│   └───────────────┘  └───────────────┘  └───────────────┘      │
└─────────────────────────────────────────────────────────────────┘
```

### 10.1.2 Agent 类型体系

Claude Code 定义了三种主要的 Agent 类型（定义于 `loadAgentsDir.ts`）：

| 类型 | 来源 | 特点 |
|------|------|------|
| **BuiltInAgent** | 内置（`built-in/`目录） | 动态 prompt，无静态 systemPrompt |
| **CustomAgent** | 用户/项目/策略设置 | 从 `.md` 文件解析，支持 memory |
| **PluginAgent** | 插件加载 | 由插件系统提供 |

```typescript
// src/tools/AgentTool/loadAgentsDir.ts
export type AgentDefinition =
  | BuiltInAgentDefinition
  | CustomAgentDefinition
  | PluginAgentDefinition

export type BaseAgentDefinition = {
  agentType: string
  whenToUse: string
  tools?: string[]
  disallowedTools?: string[]
  skills?: string[]
  mcpServers?: AgentMcpServerSpec[]
  hooks?: HooksSettings
  color?: AgentColorName
  model?: string
  effort?: EffortValue
  permissionMode?: PermissionMode
  maxTurns?: number
  memory?: AgentMemoryScope  // 'user' | 'project' | 'local'
  background?: boolean
  isolation?: 'worktree' | 'remote'
}
```

### 10.1.3 核心模块清单

| 模块路径 | 职责 |
|---------|------|
| `src/tools/AgentTool/` | Agent 创建、运行、恢复 |
| `src/tools/TeamCreateTool/` | 团队创建与管理 |
| `src/tools/SendMessageTool/` | Agent 间通信 |
| `src/coordinator/coordinatorMode.ts` | 协调者模式 |
| `src/utils/swarm/` | Swarm 基础设施 |
| `src/utils/teammateMailbox.ts` | 消息邮箱 |
| `src/bridge/` | 远程桥接 |

---

## 10.2 子Agent创建机制

Claude Code 提供了两种创建子Agent的机制：**显式创建**（通过 AgentTool）和**隐式分叉**（Fork Subagent）。

### 10.2.1 AgentTool 显式创建

`AgentTool` 是创建子Agent的主要工具，定义于 `src/tools/AgentTool/AgentTool.tsx`。

```typescript
// AgentTool 输入参数
interface AgentToolInput {
  description?: string      // Agent 描述
  subagent_type?: string     // Agent 类型（如 'worker'）
  prompt: string            // 初始 prompt
  model?: string            // 模型（可选，默认继承）
  max_turns?: number        // 最大轮次
  isolation?: 'worktree'    // 工作树隔离
  background?: boolean       // 后台运行
}
```

#### 子Agent类型选择

```typescript
// src/tools/AgentTool/loadAgentsDir.ts - 内置 Agent 类型
export function getActiveAgentsFromList(allAgents: AgentDefinition[]): AgentDefinition[] {
  const builtInAgents = allAgents.filter(a => a.source === 'built-in')
  const pluginAgents = allAgents.filter(a => a.source === 'plugin')
  const userAgents = allAgents.filter(a => a.source === 'userSettings')
  const projectAgents = allAgents.filter(a => a.source === 'projectSettings')
  // ...
}
```

### 10.2.2 Fork Subagent 隐式分叉

Fork Subagent 是一个实验性功能，当 `subagent_type` 被省略时触发：

```typescript
// src/tools/AgentTool/forkSubagent.ts
export function isForkSubagentEnabled(): boolean {
  if (feature('FORK_SUBAGENT')) {
    if (isCoordinatorMode()) return false
    if (getIsNonInteractiveSession()) return false
    return true
  }
  return false
}

export const FORK_AGENT = {
  agentType: FORK_SUBAGENT_TYPE,
  tools: ['*'],           // 继承父工具池
  maxTurns: 200,
  model: 'inherit',       // 继承父模型
  permissionMode: 'bubble'
} satisfies BuiltInAgentDefinition
```

**Fork 的关键特性**：

1. **完整上下文继承**：子Agent继承父Agent的完整对话历史
2. **字节级缓存优化**：通过相同的 API 请求前缀实现 prompt 缓存共享
3. **独立工作目录**：可运行在隔离的 git worktree 中

```typescript
// 构建分叉消息 - 保持缓存友好的前缀
export function buildForkedMessages(
  directive: string,
  assistantMessage: AssistantMessage,
): MessageType[] {
  // 保留完整的 assistant 消息（包括所有 tool_use 块）
  // 构建单个 user 消息，包含所有 tool_result 占位符 + 每子Agent指令
  // 结果：[...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
}
```

### 10.2.3 Agent 生命周期

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent 生命周期                                 │
│                                                                  │
│  1. 创建 (AgentTool.call)                                        │
│     ├── 解析 agent 定义                                           │
│     ├── 构建 system prompt                                       │
│     └── 注册到 AppState.tasks                                     │
│                                                                  │
│  2. 运行 (runAgent)                                              │
│     ├── 加载工具池                                                │
│     ├── 执行 Tool Call 循环                                       │
│     └── 处理 tool 结果                                            │
│                                                                  │
│  3. 恢复 (resumeAgent)                                           │
│     ├── 从磁盘读取 transcript                                     │
│     ├── 重建 context                                             │
│     └── 继续执行                                                 │
│                                                                  │
│  4. 完成 (task-notification)                                     │
│     └── 发送 <task-notification> 给父Agent                        │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2.4 后台任务管理

AgentTool 支持后台执行，通过 `background: true` 参数：

```typescript
// src/tools/AgentTool/agentToolUtils.ts
export async function runAsyncAgentLifecycle({
  taskId,
  abortController,
  makeStream,
  metadata,
  description,
  toolUseContext,
  rootSetAppState,
}: {
  taskId: string
  abortController: AbortController
  makeStream: (params: OnCacheSafeParams) => Promise<StreamResult>
  metadata: AgentMetadata
  description: string
  toolUseContext: ToolUseContext
  rootSetAppState: SetAppStateFn
}): Promise<void>
```

后台任务结果通过 `<task-notification>` XML 标签返回给父Agent：

```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable status summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

---

## 10.3 Agent Memory 协作

Claude Code 的多Agent系统中，每个 Agent 可以拥有独立的持久化记忆（Agent Memory），支持三种作用域。

### 10.3.1 Memory 作用域类型

```typescript
// src/tools/AgentTool/loadAgentsDir.ts
export type AgentMemoryScope = 'user' | 'project' | 'local'

// 作用域层级
//
// ┌─────────────────────────────────────────────────────┐
// │ User Scope (~/.claude/agent-memory/<agent>/)         │
// │   - 跨项目共享                                        │
// │   - 用户级别学习                                       │
// ├─────────────────────────────────────────────────────┤
// │ Project Scope (.claude/agent-memory/<agent>/)        │
// │   - 项目内共享，可版本控制                            │
// │   - 团队协作                                          │
// ├─────────────────────────────────────────────────────┤
// │ Local Scope (.claude/agent-memory-local/<agent>/)    │
// │   - 本机专用，不入版本控制                            │
// │   - 机器特定学习                                      │
// └─────────────────────────────────────────────────────┘
```

### 10.3.2 AgentMemory 实现

```typescript
// src/tools/AgentTool/agentMemory.ts
export function getAgentMemoryDir(
  agentType: string,
  scope: AgentMemoryScope,
): string {
  const dirName = sanitizeAgentTypeForPath(agentType)
  switch (scope) {
    case 'project':
      return join(getCwd(), '.claude', 'agent-memory', dirName)
    case 'local':
      return getLocalAgentMemoryDir(dirName)
    case 'user':
      return join(getMemoryBaseDir(), 'agent-memory', dirName)
  }
}

// 加载 Agent Memory Prompt
export function loadAgentMemoryPrompt(
  agentType: string,
  scope: AgentMemoryScope,
): string {
  const memoryDir = getAgentMemoryDir(agentType, scope)
  return buildMemoryPrompt({
    displayName: 'Persistent Agent Memory',
    memoryDir,
    extraGuidelines: [scopeNote]
  })
}
```

### 10.3.3 Memory 快照机制

Agent Memory 快照（Snapshot）用于在团队成员间同步记忆状态：

```typescript
// src/tools/AgentTool/agentMemorySnapshot.ts
export async function checkAgentMemorySnapshot(
  agentType: string,
  scope: AgentMemoryScope,
): Promise<{
  action: 'none' | 'initialize' | 'prompt-update'
  snapshotTimestamp?: string
}> {
  // 检查快照是否存在
  // 比较本地记忆与快照时间戳
  // 返回需要执行的操作
}

export async function initializeFromSnapshot(
  agentType: string,
  scope: AgentMemoryScope,
  snapshotTimestamp: string,
): Promise<void> {
  // 从项目快照初始化本地记忆
  await copySnapshotToLocal(agentType, scope)
  await saveSyncedMeta(agentType, scope, snapshotTimestamp)
}
```

**快照文件结构**：

```
.claude/
└── agent-memory-snapshots/
    └── <agentType>/
        └── snapshot.json       # 元数据（updatedAt）
        ├── memory-1.md        # 记忆文件
        └── memory-2.md
```

### 10.3.4 Team Memory 协作

Team Memory 通过 `src/memdir/teamMemPrompts.ts` 提供：

```typescript
// src/memdir/teamMemPrompts.ts
export function buildTeamMemoryPrompt(params: {
  teamId: string
  teammateName: string
  teammateMode: 'leader' | 'teammate'
  teamAllowedPaths?: TeamAllowedPath[]
  memberColor?: string
}): string {
  // 构建团队记忆上下文
  // 包含团队成员、共享路径等信息
}
```

---

## 10.4 Team 协作模式

Team 是 Claude Code 多Agent系统的核心协作单元，由 TeamCreateTool 创建。

### 10.4.1 Team 创建

```typescript
// src/tools/TeamCreateTool/TeamCreateTool.ts
export const TeamCreateTool: Tool<InputSchema, Output> = buildTool({
  name: 'TeamCreate',
  async call(input, context) {
    const { team_name, description, agent_type } = input

    // 生成唯一团队名
    const finalTeamName = generateUniqueTeamName(team_name)

    // 创建团队文件
    const teamFile: TeamFile = {
      name: finalTeamName,
      description,
      createdAt: Date.now(),
      leadAgentId: formatAgentId(TEAM_LEAD_NAME, finalTeamName),
      leadSessionId: getSessionId(),
      members: [{
        agentId: leadAgentId,
        name: TEAM_LEAD_NAME,
        agentType: agent_type || TEAM_LEAD_NAME,
        model: leadModel,
        joinedAt: Date.now(),
        tmuxPaneId: '',
        cwd: getCwd(),
        subscriptions: [],
      }],
    }

    // 写入团队配置
    await writeTeamFileAsync(finalTeamName, teamFile)

    // 更新 AppState
    setAppState(prev => ({
      ...prev,
      teamContext: { teamName, ... }
    }))
  }
})
```

### 10.4.2 Team 文件结构

```typescript
// src/utils/swarm/teamHelpers.ts
export type TeamFile = {
  name: string
  description?: string
  createdAt: number
  leadAgentId: string
  leadSessionId?: string
  hiddenPaneIds?: string[]
  teamAllowedPaths?: TeamAllowedPath[]
  members: Array<{
    agentId: string
    name: string
    agentType?: string
    model?: string
    color?: string
    planModeRequired?: boolean
    joinedAt: number
    tmuxPaneId: string
    cwd: string
    worktreePath?: string
    sessionId?: string
    subscriptions: string[]
    backendType?: BackendType
    isActive?: boolean
    mode?: PermissionMode
  }>
}
```

**团队目录结构**：

```
~/.claude/teams/<team-name>/
├── config.json           # 团队配置
└── inboxes/
    ├── team-lead.json    # 主导者收件箱
    ├── worker-a.json     # 工作Agent A 收件箱
    └── worker-b.json     # 工作Agent B 收件箱
```

### 10.4.3 协调者模式（Coordinator Mode）

协调者模式是 Claude Code 的多Agent编排模式，启用后主Agent 变为**协调者**：

```typescript
// src/coordinator/coordinatorMode.ts
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}

export function getCoordinatorSystemPrompt(): string {
  return `You are a **coordinator**. Your job is to:
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible`

// 协调者工具集
const INTERNAL_WORKER_TOOLS = new Set([
  TEAM_CREATE_TOOL_NAME,
  TEAM_DELETE_TOOL_NAME,
  SEND_MESSAGE_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
])
```

**协调者工作流**：

```
用户请求
    │
    ▼
┌─────────────────────────────────────────┐
│           Coordinator Agent             │
│                                         │
│  1. 分解任务 ──────────────────────────  │
│     研究任务 ──→ Worker A (并行)          │
│     研究任务 ──→ Worker B (并行)          │
│                                         │
│  2. 综合结果                             │
│     读取 Worker findings                │
│     理解问题                             │
│     制定实现规格                         │
│                                         │
│  3. 分发实现 ──────────────────────────  │
│     SendMessage(Worker A) → 执行修复      │
│     SendMessage(Worker B) → 执行测试     │
│                                         │
│  4. 验证汇总                             │
│     Worker 结果 → task-notification      │
│     验证 → 报告用户                      │
└─────────────────────────────────────────┘
```

### 10.4.4 工作树隔离（Worktree Isolation）

Team 成员可以在独立的 git worktree 中运行，避免相互干扰：

```typescript
// src/utils/swarm/teamHelpers.ts
async function destroyWorktree(worktreePath: string): Promise<void> {
  // 使用 git worktree remove 命令
  // 或回退到 rm -rf
}

// 团队清理时销毁所有工作树
export async function cleanupTeamDirectories(teamName: string): Promise<void> {
  // 读取 team file 获取 worktree paths
  // 销毁每个 worktree
  // 清理团队目录和任务目录
}
```

---

## 10.5 Agent 间通信

Claude Code 的 Agent 间通信通过 `SendMessageTool` 和基于文件的邮箱系统实现。

### 10.5.1 SendMessageTool 消息类型

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts
const StructuredMessage = lazySchema(() =>
  z.discriminatedUnion('type', [
    z.object({
      type: z.literal('shutdown_request'),
      reason: z.string().optional(),
    }),
    z.object({
      type: z.literal('shutdown_response'),
      request_id: z.string(),
      approve: semanticBoolean(),
      reason: z.string().optional(),
    }),
    z.object({
      type: z.literal('plan_approval_response'),
      request_id: z.string(),
      approve: semanticBoolean(),
      feedback: z.string().optional(),
    }),
  ]),
)
```

### 10.5.2 TeammateMailbox 文件系统

```typescript
// src/utils/teammateMailbox.ts
export async function writeToMailbox(
  recipientName: string,
  message: Omit<TeammateMessage, 'read'>,
  teamName?: string,
): Promise<void> {
  await ensureInboxDir(teamName)
  const inboxPath = getInboxPath(recipientName, teamName)

  // 使用文件锁防止并发写入
  const release = await lockfile.lock(inboxPath, LOCK_OPTIONS)
  try {
    const messages = await readMailbox(recipientName, teamName)
    messages.push({ ...message, read: false })
    await writeFile(inboxPath, jsonStringify(messages, null, 2))
  } finally {
    await release()
  }
}
```

**邮箱文件格式**：

```json
// ~/.claude/teams/<team>/inboxes/<agent>.json
[
  {
    "from": "team-lead",
    "text": "Fix the null pointer in src/auth/validate.ts:42",
    "timestamp": "2026-05-18T10:30:00.000Z",
    "read": false,
    "color": "blue",
    "summary": "Fix null pointer bug"
  }
]
```

### 10.5.3 消息路由

```
┌─────────────────────────────────────────────────────────────────┐
│                    消息路由流程                                   │
│                                                                  │
│  SendMessageTool.call(input)                                     │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐    直接消息     ┌─────────────┐                  │
│  │ In-Process  │ ─────────────→ │  邮箱写入   │                  │
│  │  Teammate   │                └─────────────┘                  │
│  └─────────────┘                      │                           │
│         │                             ▼                           │
│         ▼                    ┌─────────────┐                     │
│  ┌─────────────┐              │  useInbox   │                     │
│  │  队列 Pending │              │  Poller     │                     │
│  │  Message     │              └─────────────┘                     │
│  └─────────────┘                      │                           │
│                                      ▼                           │
│                               作为 Turn 提交                     │
└─────────────────────────────────────────────────────────────────┘
```

### 10.5.4 协议消息类型

Claude Code 定义了丰富的协议消息类型：

| 消息类型 | 方向 | 用途 |
|---------|------|------|
| `shutdown_request` | Leader → Worker | 请求关闭 |
| `shutdown_response` | Worker → Leader | 同意/拒绝关闭 |
| `plan_approval_request` | Worker → Leader | 请求计划批准 |
| `plan_approval_response` | Leader → Worker | 批准/拒绝计划 |
| `permission_request` | Worker → Leader | 请求权限 |
| `permission_response` | Leader → Worker | 权限响应 |
| `idle_notification` | Worker → Leader | 空闲通知 |
| `task_assignment` | Leader → Worker | 任务分配 |

---

## 10.6 Multi-Agent 执行流程

### 10.6.1 完整执行流程图

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Multi-Agent 执行完整流程                            │
│                                                                      │
│  1. 团队创建                                                          │
│     User ──→ TeamCreate ──→ TeamFile ──→ AppState.teamContext         │
│                           │                                          │
│                           ▼                                          │
│                      resetTaskList                                   │
│                                                                      │
│  2. Worker 启动                                                       │
│     AgentTool ──→ AgentTool.call ──→ registerAsyncAgent               │
│                                   │                                  │
│                                   ▼                                  │
│                              spawnInProcess / spawnPane               │
│                                   │                                  │
│                                   ▼                                  │
│                              inProcessRunner / tmux pane              │
│                                                                      │
│  3. 任务执行                                                          │
│     Worker ──→ runAgent ──→ Tool Calls ──→ Results                    │
│                   │                                                   │
│                   ▼                                                   │
│              完成 ──→ task-notification                              │
│                                                                      │
│  4. 消息通信                                                          │
│     SendMessage ──→ writeToMailbox ──→ inbox poll                    │
│                         │                        │                    │
│                         ▼                        ▼                    │
│                    Inbox File              useInboxPoller            │
│                                              │                        │
│                                              ▼                        │
│                                         readMailbox                   │
│                                              │                        │
│                                              ▼                        │
│                                         提交为 Turn                   │
│                                                                      │
│  5. 结果汇总                                                          │
│     task-notification ──→ Coordinator ──→ 综合 ──→ User               │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 10.6.2 Inbox Poller 机制

```typescript
// src/hooks/useInboxPoller.ts
export function useInboxPoller({
  enabled,
  isLoading,
  onSubmitMessage,
}: Props): void {
  const poll = useCallback(async () => {
    const unread = await readUnreadMessages(
      agentName,
      currentAppState.teamContext?.teamName,
    )

    for (const message of unread) {
      // 处理不同类型的消息
      if (isPermissionRequest(message.text)) {
        // 权限请求 → 路由到权限处理
      } else if (isShutdownRequest(message.text)) {
        // 关闭请求 → 处理关闭
      } else if (isPlanApprovalRequest(message.text)) {
        // 计划批准请求 → 显示 UI
      } else {
        // 普通消息 → 提交为 Turn
        onSubmitMessage(formatTeammateMessages([message]))
      }

      // 标记为已读
      await markMessagesAsRead(agentName, teamName)
    }
  }, [enabled, isLoading])

  // 每秒轮询
  useInterval(poll, INBOX_POLL_INTERVAL_MS)
}
```

### 10.6.3 权限同步

当 Worker 需要权限时，通过邮箱系统与 Leader 通信：

```
Worker                          Leader
   │                              │
   │  permission_request ────────→│
   │                              │
   │                              │ 显示 ToolUseConfirm UI
   │                              │
   │←───── permission_response ──┤
   │                              │
   │  执行 Tool                    │
   ▼                              ▼
```

### 10.6.4 后端类型支持

Claude Code 支持多种 Worker 后端类型：

```typescript
// src/utils/swarm/backends/types.ts
export type BackendType =
  | 'tmux'         // tmux 会话
  | 'iterm2'       // iTerm2 窗格
  | 'screen'       // screen 会话
  | 'in-process'   // 同进程内运行

export type PaneBackendType = 'tmux' | 'iterm2' | 'screen'
```

**In-Process Runner** 用于同进程内的 Worker，执行效率更高：

```typescript
// src/utils/swarm/inProcessRunner.ts
export async function runInProcessTeammate({
  identity,
  abortController,
  canUseTool,
  ...params
}): Promise<void> {
  // 使用 AsyncLocalStorage 隔离上下文
  await runWithTeammateContext(identity, async () => {
    // 执行 agent
    await runAgent({
      ...params,
      override: {
        agentId: identity.agentId,
        abortController,
      }
    })
  })

  // 完成时发送 idle notification
  await writeToMailbox(TEAM_LEAD_NAME, {
    from: identity.name,
    text: jsonStringify(createIdleNotification(identity.agentId, {...})),
    timestamp: new Date().toISOString(),
  })
}
```

---

## 10.7 远程桥接与跨会话通信

### 10.7.1 Bridge 架构

Claude Code 支持通过 Bridge 与远程会话通信：

```
Local Session                        Remote Session
┌──────────────────┐                ┌──────────────────┐
│    BridgeMain    │ ←── WebSocket ──→│    BridgeMain    │
│                  │                  │                  │
│  peerSessions    │                  │  peerSessions    │
│  bridgeMessaging │                  │  bridgeMessaging │
└──────────────────┘                └──────────────────┘
```

### 10.7.2 跨会话消息发送

```typescript
// src/tools/SendMessageTool/SendMessageTool.ts
if (feature('UDS_INBOX') && typeof input.message === 'string') {
  const addr = parseAddress(input.to)

  if (addr.scheme === 'bridge') {
    // 通过 Bridge 发送跨会话消息
    const { postInterClaudeMessage } = require('../../bridge/peerSessions.js')
    await postInterClaudeMessage(addr.target, input.message)
  }

  if (addr.scheme === 'uds') {
    // 通过 Unix Domain Socket 发送
    const { sendToUdsSocket } = require('../../utils/udsClient.js')
    await sendToUdsSocket(addr.target, input.message)
  }
}
```

---

## 10.8 最佳实践与设计模式

### 10.8.1 团队协作设计模式

**1. 任务分解模式**

```typescript
// 协调者分解任务
AgentTool({
  description: "研究认证模块",
  subagent_type: "worker",
  prompt: "研究 src/auth/ 模块，找出可能的 null pointer 异常..."
})

AgentTool({
  description: "研究测试覆盖",
  subagent_type: "worker",
  prompt: "查找所有与 auth 相关的测试文件..."
})
```

**2. 继续 vs 重新创建模式**

| 场景 | 策略 |
|------|------|
| 研究探索了需要编辑的文件 | 继续（SendMessage） |
| 研究广泛但实现狭窄 | 重新创建 |
| 修正失败或扩展近期工作 | 继续 |
| 验证其他 Worker 刚写的代码 | 重新创建 |
| 第一次尝试方向完全错误 | 重新创建 |
| 完全无关的任务 | 重新创建 |

### 10.8.2 消息设计模式

**1. 自包含 Prompt**

Worker 无法看到对话历史，每个 prompt 必须包含所有必要信息：

```typescript
// 好的 prompt（自包含）
"Fix the null pointer in src/auth/validate.ts:42. The user field on Session (src/auth/types.ts:15) is undefined when sessions expire but the token remains cached..."

// 不好的 prompt（依赖上下文）
"Based on your findings, fix the auth bug"
```

**2. 精确的 Git 操作**

```typescript
// 精确指定
"Create a new branch from main called 'fix/session-expiry'. Cherry-pick only commit abc123 onto it..."
```

### 10.8.3 错误处理模式

```typescript
// Worker 失败时继续同一 Worker（保留上下文）
if (workerResult.status === 'failed') {
  SendMessageTool({
    to: workerId,
    message: "修复测试失败：assertEquals 期望 'Invalid session' 但得到 'Session expired'"
  })
}
```

---

## 10.9 小结

本章深入分析了 Claude Code 的多Agent系统架构：

| 组件 | 核心功能 |
|------|----------|
| **AgentTool** | 子Agent创建、恢复、生命周期管理 |
| **TeamCreateTool** | 团队创建、成员管理 |
| **SendMessageTool** | Agent 间通信、协议消息 |
| **teammateMailbox** | 基于文件的异步消息系统 |
| **inProcessRunner** | 同进程内 Worker 执行 |
| **coordinatorMode** | 协调者编排模式 |

多Agent系统的核心设计理念：

1. **隔离与协作平衡**：通过 worktree 隔离避免干扰，通过邮箱系统实现通信
2. **异步优先**：Worker 后台执行，通过 `<task-notification>` 异步通知
3. **权限分层**：Leader 集中权限决策，Worker 执行具体任务
4. **可恢复性**：完整的 transcript 保存支持 Agent 恢复

下一章我们将探讨 Claude Code 的 UI 组件架构，理解其终端交互界面的设计实现。

---

*参考源码版本：Claude Code npm package (2026年3月31日泄露版)*