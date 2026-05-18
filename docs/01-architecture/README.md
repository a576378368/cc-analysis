# 第一章：架构总览

> 本章深入剖析 Claude Code 的整体架构设计，从六层架构到核心模块的职责划分，帮助读者建立对整个系统的全局认知。

## 1. 项目概述

Claude Code 是一个基于终端的 AI 编程助手，通过 CLI/TUI 界面与用户交互。它是 Claude AI 模型在命令行环境中的实现，提供了强大的代码编辑、文件操作、搜索和多代理协作能力。

**核心定位**：
- 终端优先的 AI 编程工具
- 支持多代理协作（Swarm/Team）
- 内置 MCP（Model Context Protocol）支持
- 会话记忆与持久化
- 安全沙箱执行环境

**技术亮点**：
- **全量 TypeScript 实现**：约 1902 个源文件，513,237 行代码
- **响应式流式架构**：基于 AsyncGenerator 的异步流式处理
- **多层次记忆系统**：Auto/Session/Agent/Team 四层记忆体系
- **纵深安全防御**：沙箱隔离 + 权限控制 + 路径验证三层防护
- **平台化扩展**：MCP 协议 + Skills 机制 + Plugin 系统

**源码规模统计**：
| 指标 | 数值 |
|------|------|
| 源文件数量 | ~1902 个 |
| 代码行数 | ~513,237 行 |
| 工具类型 | 44+ 种 |
| UI 组件目录 | 33 个 |
| 核心模块 | 12 个 |

---

## 2. 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Claude Code 六层架构                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                   第1层：CLI 引导层 (Bootstrap)                   │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐       │    │
│  │  │ --version │ │ --daemon  │ │  remote   │ │   bg/ps   │       │    │
│  │  │  快速路径  │ │  后台守护  │ │  远程控制  │ │  会话管理  │       │    │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘       │    │
│  │  文件：src/entrypoints/cli.tsx, src/main.tsx                    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                  第2层：TUI/REPL 交互层                           │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐       │    │
│  │  │  Messages │ │  Input    │ │  Status   │ │  Dialog   │       │    │
│  │  │  消息渲染  │ │  命令行    │ │  状态行   │ │  对话框   │       │    │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘       │    │
│  │  文件：src/components/Messages.tsx, src/ink.ts, src/replLauncher.tsx│    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                 第3层：Query/Agent 执行内核                       │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐       │    │
│  │  │  Query    │ │ QueryEngine│ │  Compact  │ │  Tools    │       │    │
│  │  │  查询循环  │ │  查询引擎  │ │  上下文   │ │  工具调度  │       │    │
│  │  │           │ │           │ │  压缩    │ │           │       │    │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘       │    │
│  │  文件：src/query.ts, src/QueryEngine.ts, src/services/compact/    │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                   第4层：Tool/Permission 层                      │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐       │    │
│  │  │ FileTool  │ │ BashTool   │ │  MCPTool  │ │ Permission│       │    │
│  │  │  文件操作  │ │  命令执行  │ │  MCP工具  │ │  权限管理  │       │    │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘       │    │
│  │  文件：src/tools/*.ts, src/hooks/toolPermission/, src/types/permissions.ts│    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                  第5层：Memory/Persistence 层                   │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐       │    │
│  │  │  Memdir   │ │ SessionMem│ │  History  │ │  FileCache│       │    │
│  │  │  记忆目录  │ │  会话记忆  │ │  历史记录  │ │  文件缓存  │       │    │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘       │    │
│  │  文件：src/memdir/*.ts, src/services/SessionMemory/, src/history.ts│    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                 │                                       │
│                                 ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                 第6层：MCP/Remote/Swarm 扩展层                    │    │
│  │  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐       │    │
│  │  │  MCP      │ │  Remote   │ │  Swarm    │ │  Skills   │       │    │
│  │  │  协议连接  │ │  远程模式  │ │  多代理   │ │  技能扩展  │       │    │
│  │  └───────────┘ └───────────┘ └───────────┘ └───────────┘       │    │
│  │  文件：src/services/mcp/, src/remote/, src/skills/, src/coordinator/│    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 各层职责详解

### 3.1 CLI 引导层 (Bootstrap Layer)

**职责**：程序入口点，处理命令行参数解析，快速路径分流

**关键文件**：
- `src/entrypoints/cli.tsx` - CLI 入口，早期参数解析和快速路径分流
- `src/main.tsx` - 主入口，完整的 CLI 初始化和 REPL 启动

**核心功能**：
- 快速路径处理（`--version`, `--daemon-worker`, `remote-control`）
- 环境变量预加载（MDM、Keychain 预取）
- 配置初始化（GrowthBook, Analytics）
- 会话管理（`bg`, `ps`, `logs`, `attach`, `kill`）

```typescript
// src/entrypoints/cli.tsx 核心结构
async function main(): Promise<void> {
  // 1. 快速路径：--version 无需加载任何模块
  if (args[0] === '--version') {
    console.log(`${MACRO.VERSION}`);
    return;
  }
  // 2. daemon worker 快速路径
  if (feature('DAEMON') && args[0] === '--daemon-worker') {
    await runDaemonWorker(args[1]);
    return;
  }
  // 3. remote-control 桥接模式
  if (feature('BRIDGE_MODE') && args[0] === 'remote-control') {
    await bridgeMain(args.slice(1));
    return;
  }
  // 4. 后台会话管理
  if (feature('BG_SESSIONS') && (args[0] === 'ps' || args[0] === 'logs' ...)) {
    await bg.psHandler(args.slice(1));
    return;
  }
  // 5. 完整 CLI 加载
  const { main: cliMain } = await import('../main.js');
  await cliMain();
}
```

---

### 3.2 TUI/REPL 交互层 (Interaction Layer)

**职责**：用户界面渲染，消息显示，输入处理，状态管理

**关键文件**：
- `src/components/Messages.tsx` - 消息列表渲染
- `src/components/MessageRow.tsx` - 单条消息渲染
- `src/components/StatusLine.tsx` - 状态行显示
- `src/ink.ts` - Ink TUI 根组件
- `src/replLauncher.tsx` - REPL 启动器

**组件结构**：
```
src/components/
├── Messages.tsx          # 消息列表主组件
├── MessageRow.tsx        # 单条消息行
├── Message.tsx           # 消息内容组件
├── VirtualMessageList.tsx # 虚拟化消息列表（性能优化）
├── StatusLine.tsx        # 底部状态栏
├── PromptInput/          # 命令行输入
│   └── index.tsx
├── diff/                 # 差异显示组件
├── mcp/                  # MCP 服务器管理 UI
├── permissions/          # 权限请求 UI
├── skills/               # 技能管理 UI
└── tasks/                # 任务管理 UI
```

**核心渲染流程**：
1. 用户输入 → `handleUserInput()`
2. 输入处理 → `processUserInput()`
3. 查询引擎执行 → `QueryEngine.submitMessage()`
4. 流式输出 → `yield* query()` → 渲染到 TUI

---

### 3.3 Query/Agent 执行内核 (Query/Agent Core)

**职责**：AI 模型交互、工具调度、上下文管理、多轮对话

**关键文件**：
- `src/query.ts` - 查询循环核心实现（~68683 行）
- `src/QueryEngine.ts` - 查询引擎类（~46630 行）
- `src/services/compact/` - 上下文压缩服务

**核心组件**：

#### QueryEngine 类
```typescript
// src/QueryEngine.ts
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]      // 对话消息历史
  private abortController: AbortController // 中断控制器
  private permissionDenials: SDKPermissionDenial[] // 权限拒绝记录
  private totalUsage: NonNullableUsage     // Token 使用统计
  private readFileState: FileStateCache    // 文件状态缓存

  async *submitMessage(prompt, options?): AsyncGenerator<SDKMessage> {
    // 核心：消息提交 → AI 查询 → 工具执行循环
  }
}
```

#### Query 循环结构
```typescript
// src/query.ts
export async function* query(params: QueryParams): AsyncGenerator<StreamEvent | Message> {
  // queryLoop 包含：
  // 1. 消息预处理（附件、上下文）
  // 2. AI 模型调用（流式响应）
  // 3. 工具调用处理
  // 4. 响应后处理（压缩、总结）
  // 5. 继续循环或结束
}
```

**关键状态机转换**：
```
user_input → query_fn_entry → api_call → stream → tool_use →
tool_result → (compact?) → assistant_message → continue || terminal
```

**子目录**：
- `src/query/` - 查询相关子模块
  - `config.ts` - 查询配置构建
  - `deps.ts` - 查询依赖注入
  - `transitions.ts` - 状态转换
  - `tokenBudget.ts` - Token 预算管理
  - `stopHooks.ts` - 停止钩子

---

### 3.4 Tool/Permission 层

**职责**：工具注册、权限检查、沙箱执行、工具结果处理

**关键文件**：
- `src/Tool.ts` - 工具基类定义
- `src/tools.ts` - 工具注册表
- `src/hooks/toolPermission/` - 权限检查钩子
- `src/types/permissions.ts` - 权限类型定义

**工具目录结构**：
```
src/tools/
├── AgentTool/           # 代理工具（多代理协作核心）
├── BashTool/            # Bash 命令执行
├── FileEditTool/        # 文件编辑
├── FileReadTool/        # 文件读取
├── FileWriteTool/       # 文件写入
├── GlobTool/            # 文件 glob 搜索
├── GrepTool/            # 内容搜索
├── MCPTool/             # MCP 协议工具
├── SkillTool/           # 技能调用
├── WebFetchTool/        # 网页获取
├── WebSearchTool/       # 网页搜索
├── TaskCreateTool/      # 任务创建
├── TaskStopTool/        # 任务停止
└── shared/              # 共享工具代码
```

**权限管理架构**：
```typescript
// src/hooks/toolPermission/index.ts
// 权限检查流程：
// 1. canUseTool() 检查权限模式
// 2. 不同模式：ask（询问）、auto（自动允许）、no（拒绝）
// 3. 沙箱检查：sandbox filesystem access
// 4. 拒绝跟踪：记录权限拒绝用于分析
```

**工具执行流程**：
```
query → tool_use block → canUseTool() → 
  sandbox check → execute tool → 
  result → tool_result block → continue
```

---

### 3.5 Memory/Persistence 层

**职责**：会话记忆、上下文持久化、文件变化跟踪、历史管理

**关键文件**：
- `src/memdir/memdir.ts` - 记忆目录管理
- `src/services/SessionMemory/sessionMemory.ts` - 会话记忆服务
- `src/history.ts` - 对话历史管理
- `src/utils/fileStateCache.ts` - 文件状态缓存

**Memdir 目录结构**：
```
src/memdir/
├── memdir.ts          # 记忆目录主逻辑（~21174 行）
├── memoryTypes.ts     # 记忆类型定义
├── memoryScan.ts      # 记忆扫描
├── memoryAge.ts       # 记忆老化
├── paths.ts           # 路径管理
├── teamMemPaths.ts    # 团队记忆路径
└── teamMemPrompts.ts  # 团队记忆提示
```

**SessionMemory 服务结构**：
```
src/services/SessionMemory/
├── sessionMemory.ts        # 会话记忆主逻辑（~16561 行）
├── sessionMemoryUtils.ts   # 工具函数
├── prompts.ts             # 记忆提取提示词
└── bundled/               # 内置记忆模板
```

**数据持久化路径**：
```
~/.claude/
├── sessions/           # 会话存储
├── memories/           # 记忆目录
├── skills/             # 技能配置
├── settings.json       # 用户设置
└── analytics/         # 分析数据
```

---

### 3.6 MCP/Remote/Swarm 扩展层

**职责**：外部协议集成、远程协作、多代理编排

**关键文件**：
- `src/services/mcp/` - MCP 协议客户端
- `src/services/mcp/client.ts` - MCP 客户端主逻辑
- `src/remote/` - 远程模式支持
- `src/coordinator/` - 协调器模式（多代理）
- `src/skills/` - 技能系统

**MCP 服务结构**：
```
src/services/mcp/
├── client.ts            # MCP 客户端（~119060 行）
├── config.ts            # MCP 配置
├── types.ts             # 类型定义
├── auth.ts              # MCP 认证
├── useManageMCPConnections.tsx # 连接管理
├── MCPConnectionManager.tsx    # 连接管理器
└── officialRegistry.ts  # 官方 MCP 注册表
```

**Remote 模式架构**：
```
src/remote/
├── bridgeMain.ts        # 桥接主逻辑
├── bridgeEnabled.ts     # 桥接启用检查
└── types.ts             # 类型定义
```

**Swarm/Coordinator 多代理架构**：
```
src/coordinator/
└── coordinatorMode.ts    # 协调器模式

src/tools/AgentTool/
├── loadAgentsDir.ts     # 代理加载
└── agentColorManager.ts # 代理颜色管理
```

**Skills 目录结构**：
```
src/skills/
├── bundled/             # 内置技能
│   └── index.ts
├── loadSkillsDir.ts     # 技能目录加载（~34415 行）
└── bundledSkills.ts     # 技能定义
```

---

## 4. 数据流总览

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          完整数据流                                       │
└──────────────────────────────────────────────────────────────────────────┘

用户输入 (Terminal)
     │
     ▼
┌─────────────────┐
│  CLI 解析       │  ← src/entrypoints/cli.tsx
│  args = process │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  快速路径分流   │
│  (version/daemon│
│   /remote/bg)   │
└────────┬────────┘
         │ (无快速路径)
         ▼
┌─────────────────┐
│  Main.tsx       │  ← src/main.tsx
│  初始化         │
│  - configs      │
│  - telemetry    │
│  - migrations   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  launchRepl()   │  ← src/replLauncher.tsx
│  REPL 启动      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  App.tsx        │  ← src/components/App.tsx
│  Ink TUI 渲染   │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Messages.tsx   │  ← 用户看到 TUI
│  消息列表渲染   │
└────────┬────────┘
         │
         │ (用户输入命令)
         ▼
┌─────────────────┐
│  processUserInput() │ ← src/utils/processUserInput/
│  输入处理        │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  QueryEngine    │  ← src/QueryEngine.ts
│  .submitMessage │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  query()        │  ← src/query.ts
│  查询循环       │
│  ┌───────────┐  │
│  │ 1. 准备   │  │  - 加载记忆
│  │   消息    │  │  - 检查权限
│  │ 2. API    │  │  - 准备上下文
│  │   调用    │  │  - token 预算
│  │ 3. 工具   │  │     │
│  │   执行    │  │     │
│  │ 4. 压缩   │  │     │
│  │   整理    │  │     │
│  └─────┬─────┘  │     │
└────────┼────────┘     │
         │              │
         ▼              │
┌─────────────────┐     │
│  流式响应       │     │
│  yield* query() │     │
└────────┬────────┘     │
         │              │
         ▼              │
┌─────────────────┐     │
│  工具执行       │  ← ─┘
│  runTools()     │
│  (Sandbox)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  工具结果       │
│  tool_result    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  继续循环或    │
│  结束          │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  响应输出      │
│  → TUI 渲染    │
│  → 保存历史    │
│  → 更新记忆    │
└─────────────────┘
```

---

## 5. 核心模块关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           模块依赖关系图                                  │
└─────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────┐
                              │  cli.tsx    │  ← 程序入口
                              └──────┬──────┘
                                     │
                                     ▼
                              ┌─────────────┐
                              │  main.tsx   │  ← 主入口
                              └──────┬──────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
            ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
            │  replLauncher│ │  App.tsx     │ │  setup.ts    │
            │  (REPL启动)   │ │  (TUI渲染)   │ │  (设置流程)   │
            └──────┬───────┘ └──────┬───────┘ └──────────────┘
                   │                │
                   │    ┌───────────┴───────────┐
                   │    │                       │
                   ▼    ▼                       ▼
            ┌──────────────┐          ┌──────────────┐
            │ Messages.tsx │          │  StatusLine  │
            │ (消息列表)    │          │  (状态栏)    │
            └──────┬───────┘          └──────────────┘
                   │
                   │ 用户输入
                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Query/Agent 执行内核                             │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐    │
│  │  QueryEngine   │───▶│   query.ts     │───▶│  services/     │    │
│  │ (查询引擎类)    │    │  (查询循环)    │    │  compact/      │    │
│  └───────┬────────┘    └────────────────┘    │  (上下文压缩)  │    │
│          │                                   └───────┬────────┘    │
│          │                                           │             │
│          ▼                                           ▼             │
│  ┌────────────────┐                    ┌────────────────┐      │
│  │   Tool.ts      │◀────────────────────│  tools.ts      │      │
│  │  (工具基类)     │                     │  (工具注册表)   │      │
│  └───────┬────────┘                    └───────┬────────┘      │
│          │                                      │               │
│          ▼                                      ▼               │
│  ┌────────────────┐          ┌────────────────────────────┐     │
│  │  tools/*.ts    │          │  hooks/toolPermission/     │     │
│  │  (具体工具)     │          │  (权限检查钩子)            │     │
│  └────────────────┘          └────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         Memory/Persistence 层                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐│
│  │  memdir/    │  │ SessionMem/ │  │  history.ts │  │ fileState   ││
│  │  (记忆目录)  │  │ (会话记忆)   │  │  (历史管理)  │  │ (文件缓存)  ││
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘│
└──────────────────────────────────────────────────────────────────────┘
                   │
                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      MCP/Remote/Swarm 扩展层                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐│
│  │ services/   │  │   remote/   │  │ coordinator/│  │  skills/    ││
│  │ mcp/        │  │  (远程模式)  │  │ (协调器)    │  │  (技能系统) ││
│  │ (MCP客户端)  │  │             │  │             │  │             ││
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘│
└──────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键源码路径索引

| 模块 | 路径 | 说明 |
|------|------|------|
| CLI 入口 | `src/entrypoints/cli.tsx` | 命令行解析，快速路径 |
| 主入口 | `src/main.tsx` | 完整初始化逻辑 |
| REPL 启动 | `src/replLauncher.tsx` | TUI REPL 启动 |
| TUI 根组件 | `src/components/App.tsx` | Ink 根组件 |
| 查询引擎 | `src/QueryEngine.ts` | 查询生命周期管理 |
| 查询循环 | `src/query.ts` | 核心查询逻辑实现 |
| 工具注册 | `src/tools.ts` | 工具定义与注册 |
| 工具基类 | `src/Tool.ts` | 工具抽象基类 |
| 记忆目录 | `src/memdir/memdir.ts` | 记忆系统核心 |
| 会话记忆 | `src/services/SessionMemory/sessionMemory.ts` | 会话记忆服务 |
| MCP 客户端 | `src/services/mcp/client.ts` | MCP 协议客户端 |
| 权限管理 | `src/hooks/toolPermission/` | 权限检查逻辑 |
| 历史管理 | `src/history.ts` | 对话历史 |
| 上下文压缩 | `src/services/compact/` | 上下文压缩服务 |
| 协调器模式 | `src/coordinator/coordinatorMode.ts` | 多代理协调 |
| 技能系统 | `src/skills/loadSkillsDir.ts` | 技能加载 |

---

## 7. 核心设计原则总结

Claude Code 的架构设计体现了以下核心原则：

### 7.1 性能优先原则

**零导入快路径**：`cli.tsx` 对 `--version` 等常用命令实现零模块加载，确保即时响应。

**动态 Import 策略**：只有在需要时才加载完整模块，避免不必要的初始化开销。

**流式处理架构**：基于 AsyncGenerator 的流式响应，边处理边输出，减少等待时间。

**虚拟化渲染**：仅渲染可见区域的 UI 组件，优化大量消息时的性能。

### 7.2 安全纵深防御原则

**三层安全机制**：
1. **权限层**：工具使用前必须经过权限检查（ask/auto/bypass）
2. **沙箱层**：Bash 命令根据 `shouldUseSandbox()` 判断是否进入沙箱
3. **路径验证层**：验证操作路径是否在允许范围内

**危险命令拦截**：模式匹配检测 `rm -rf`、`dd`、`mkfs` 等危险操作。

### 7.3 可扩展性原则

**MCP 协议**：通过 Model Context Protocol 支持外部服务扩展。

**Skills 系统**：支持文件系统、内建、MCP 三种技能来源。

**Agent 协作**：支持子 Agent 创建和 Team 协作模式。

### 7.4 记忆层次化原则

**四层记忆架构**：Auto/Session/Agent/Team 各司其职，兼顾长期记忆和会话上下文。

**记忆压缩机制**：多种压缩级别（Snip/Micro/Auto/Reactive/Full）确保 Token 高效利用。

---

## 8. 架构决策记录

### 8.1 为什么选择六层架构？

Claude Code 采用六层分层架构，而非更简单的单层或两层架构，主要基于以下考虑：

| 设计选择 | 替代方案 | 优势 |
|---------|---------|------|
| 分层解耦 | 单层架构 | 便于独立演进、测试、维护 |
| 独立 Tool 层 | Tool 混合在 Query 层 | 权限控制更精细、安全性更高 |
| 独立 Memory 层 | Memory 混合在 Query 层 | 支持多层次记忆策略 |
| 独立扩展层 | 扩展直接集成 | 支持 MCP/Skills/Remote 多样化扩展 |

### 8.2 关键架构决策

1. **AsyncGenerator 作为核心模式**
   - 决策：使用 AsyncGenerator 实现流式处理
   - 原因：边处理边输出，提升用户体验
   - 影响：所有主要函数都返回 AsyncGenerator

2. **文件化记忆系统**
   - 决策：使用文件系统而非数据库存储记忆
   - 原因：简单、跨平台、可版本控制
   - 影响：记忆检索需要额外处理

3. **权限模式分离**
   - 决策：权限检查与工具执行分离
   - 原因：支持多种权限策略（ask/auto/bypass）
   - 影响：权限拒绝可被跟踪和分析

### 8.3 技术债务与演进

| 技术债务 | 当前处理 | 演进方向 |
|---------|---------|---------|
| 大文件（main.tsx 803KB） | 模块化拆分 | 持续拆分独立模块 |
| 44+ 工具类 | 共享基类 | 工具生成器模式 |
| 200+ UI 组件 | 设计系统 | 组件库抽象 |

---

## 9. 参考资料

### 9.1 核心源码文件

| 文件 | 规模 | 职责 |
|------|------|------|
| `src/main.tsx` | ~803KB | 主入口，初始化逻辑 |
| `src/query.ts` | ~68KB | 查询循环核心 |
| `src/QueryEngine.ts` | ~46KB | 查询引擎类 |
| `src/services/mcp/client.ts` | ~119KB | MCP 客户端 |
| `src/components/Messages.tsx` | ~147KB | 消息渲染 |
| `src/tools/BashTool/BashTool.tsx` | ~70KB | Bash 工具实现 |

### 9.2 相关文档

- [第二章：程序入口与初始化](../02-entrypoint/)
- [第三章：Query/Agent 执行内核](../03-query-engine/)
- [第四章：工具系统设计与实现](../04-tools/)
- [第五章：记忆系统架构](../05-memory/)
- [第六章：MCP 协议实现](../06-mcp/)

---

## 10. 常见问题

### Q1: 为什么 Claude Code 启动很快？

A1: 通过 `cli.tsx` 实现快路径分流，`--version` 等命令无需加载完整模块。

### Q2: 工具权限是如何工作的？

A2: 工具执行前通过 `canUseTool()` 检查权限模式，根据模式决定是否提示用户。

### Q3: 记忆系统如何避免 Token 溢出？

A3: 通过多级压缩机制（Snip → Micro → Auto → Reactive → Full）动态管理上下文长度。

### Q4: MCP 和 Skills 有什么区别？

A4: MCP 是外部服务协议（工具/资源），Skills 是可执行的能力单元（Markdown + Shell）。

### Q5: 如何实现多 Agent 协作？

A5: 通过 AgentTool 创建子 Agent，通过 SendMessageTool 进行 Agent 间通信，通过 Team 共享上下文。

---

*最后更新：2026年5月 | 基于 Claude Code 源码 v2026.03 分析*