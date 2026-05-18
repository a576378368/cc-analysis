# 附录G：源码文件树详解

本附录详细说明 Claude Code 源码目录结构、各级目录功能概述、核心文件职责、文件命名规范、模块依赖关系以及关键文件速查表。本文档基于 `/home/yang/workspace/claude-code-analysis/src/` 目录分析编写，旨在帮助开发者快速定位源码、了解架构全貌。

---

## 1. 源码目录结构总览

Claude Code 源码位于 `/src` 目录，采用多级目录结构组织，包含 37 个一级目录和子目录，共计约 1884 个 TypeScript/TSX 源文件。整体结构遵循功能模块化原则，按职责划分为以下几个大类：

| 类别 | 一级目录 | 说明 |
|------|---------|------|
| 核心运行入口 | `main.tsx`、`Tool.ts`、`Task.ts` | 应用主入口、工具基类、任务基类 |
| 命令系统 | `commands/`、`commands.ts` | 所有 slash commands 实现 |
| 工具系统 | `tools/`、`tools.ts` | 工具定义与实现（45+工具） |
| 任务系统 | `tasks/` | 任务类型定义与执行 |
| 查询引擎 | `query/`、`QueryEngine.ts` | 对话管理、Token 预算管理 |
| UI 渲染 | `ink/`、`components/` | Ink 终端渲染引擎 + React 组件 |
| 状态管理 | `state/` | 全局状态管理 |
| 钩子系统 | `hooks/` | React hooks（110+ 钩子） |
| 服务层 | `services/` | API、Analytics、MCP、OAuth 等服务 |
| 插件系统 | `plugins/`、`skills/` | 插件加载、技能管理 |
| 网关/桥接 | `bridge/` | 远程会话桥接、REPL 桥接 |
| 工具函数 | `utils/` | 工具函数库（180+ 文件） |
| 配置常亮 | `constants/`、`types/`、`schemas/` | 类型定义、Schema、常量 |
| 迁移脚本 | `migrations/` | 版本迁移脚本 |
| 其他 | `assistant/`、`bootstrap/`、`coordinator/`、`screen/`、`server/`、`voice/`、`vim/` | 辅助功能模块 |

---

## 2. 一级目录详解（按字母排序）

### 2.1 `assistant/` 目录

**功能概述**：负责 Claude Code 的 Assistant（AI 助手）会话管理功能，支持会话历史检索、Agentic 搜索等。

**文件结构**：
- `sessionHistory.ts` — Assistant 会话历史管理

**核心职责**：
- 管理用户与 Assistant 之间的对话历史
- 提供基于上下文的会话检索能力
- 支持 Agentic 模式下的会话组织

**关键依赖**：
- `src/context/` — 上下文管理
- `src/state/` — 状态管理

---

### 2.2 `bootstrap/` 目录

**功能概述**：应用启动时的初始化状态管理，处理命令行参数解析和全局状态初始化。

**文件结构**：
- `state.ts` — 引导状态定义与初始化逻辑

**核心职责**：
- 应用启动时的状态设置
- 命令行参数处理（cwd、remote mode、main loop model 等）
- 初始化前的环境准备

**关键依赖**：
- `src/main.tsx` — 主入口
- `src/state/` — 状态管理

---

### 2.3 `bridge/` 目录

**功能概述**：远程会话桥接功能，支持 Claude Code 远程控制、REPL 桥接等核心桥接能力。

**文件结构**（主要文件）：
- `bridgeMain.ts` — 桥接主逻辑（115571 行）
- `replBridge.ts` — REPL 桥接实现（100537 行）
- `remoteBridgeCore.ts` — 远程桥接核心
- `bridgeMessaging.ts` — 桥接消息传递
- `bridgeUI.ts` — 桥接 UI 组件
- `initReplBridge.ts` — REPL 桥接初始化
- `bridgeApi.ts` — 桥接 API 定义
- `bridgeConfig.ts` — 桥接配置
- `bridgeDebug.ts` — 桥接调试工具
- `bridgeEnabled.ts` — 桥接启用状态
- `bridgePointer.ts` — 桥接指针管理
- `bridgeStatusUtil.ts` — 桥接状态工具
- `codeSessionApi.ts` — 代码会话 API
- `createSession.ts` — 创建会话
- `debugUtils.ts` — 调试工具
- `envLessBridgeConfig.ts` — 无环境变量桥接配置
- `flushGate.ts` — 刷新门控
- `inboundAttachments.ts` — 入站附件处理
- `inboundMessages.ts` — 入站消息处理
- `jwtUtils.ts` — JWT 工具
- `pollConfigDefaults.ts` — 轮询配置默认值
- `pollConfig.ts` — 轮询配置
- `sessionIdCompat.ts` — 会话 ID 兼容性
- `sessionRunner.ts` — 会话运行器
- `trustedDevice.ts` — 信任设备管理
- `types.ts` — 类型定义
- `workSecret.ts` — 工作密钥
- `replBridgeHandle.ts` — REPL 桥接句柄
- `replBridgeTransport.ts` — REPL 传输层

**核心职责**：
- 支持远程 Claude Code 实例控制
- REPL（Read-Eval-Print Loop）桥接通信
- 远程会话创建与管理
- 桥接消息编解码与传输

**关键依赖**：
- `src/services/api/` — API 服务
- `src/utils/sessionStorage/` — 会话存储
- `src/utils/ssh/` — SSH 会话

---

### 2.4 `buddy/` 目录

**功能概述**：Buddy 功能模块（与 Agent 协作相关）。

**文件结构**：此目录结构较简单，包含协作相关的辅助功能。

**核心职责**：
- Agent 间协作支持
- Buddy 模式下的通信机制

---

### 2.5 `cli/` 目录

**功能概述**：命令行接口核心实现，处理用户输入、命令解析和输出格式化。

**文件结构**：
- `command.ts` — 命令定义
- `handlers/` — 命令处理器
- `transports/` — 传输层（ndjsonSafeStringify、structuredIO、remoteIO）
- `print.ts` — 打印输出（212735 行，最大文件之一）
- `exit.ts` — 退出处理
- `update.ts` — 更新处理
- `structuredIO.ts` — 结构化 IO

**核心职责**：
- 命令行参数解析
- 命令处理器注册与分发
- 输出格式化与打印
- 远程 IO 处理

**关键依赖**：
- `src/commands/` — 具体命令实现
- `src/ink/` — 终端渲染

---

### 2.6 `commands/` 目录

**功能概述**：所有 slash commands（斜杠命令）的实现目录，88 个子目录，包含 100+ 命令实现文件。

**文件结构**（主要命令目录）：
| 命令目录 | 说明 |
|---------|------|
| `init.ts` | 初始化命令 |
| `install.tsx` | 安装命令 |
| `insights.ts` | 分析命令（115949 行） |
| `ultraplan.tsx` | 规划命令（66629 行） |
| `commit.ts`、`commit-push-pr.ts` | Git 提交相关 |
| `diff/` | Diff 命令相关 |
| `mcp/` | MCP 服务器管理 |
| `model/` | 模型切换 |
| `session/` | 会话管理 |
| `tasks/` | 任务管理 |
| `teleport/` | 远程传输 |
| `review.ts` | 代码审查 |
| `security-review.ts` | 安全审查 |
| `brief.ts` | 摘要生成 |
| `help/` | 帮助命令 |
| `config/` | 配置管理 |
| `feedback/` | 反馈 |
| `stats/` | 统计信息 |
| `status/` | 状态显示 |
| `theme/` | 主题设置 |
| `voice/` | 语音控制 |
| `vim/` | Vim 模式 |
| `agent/` | Agent 管理 |
| `permissions/` | 权限管理 |

**核心职责**：
- 实现所有用户可调用的 slash commands
- 命令参数验证与执行
- 命令输出格式化

**关键依赖**：
- `src/utils/` — 工具函数
- `src/services/` — 服务层
- `src/ink/` — UI 渲染

---

### 2.7 `components/` 目录

**功能概述**：React UI 组件库，包含 33 个子目录，200+ 组件文件。

**文件结构**（主要组件）：
| 组件 | 说明 |
|------|------|
| `App.tsx` | 应用根组件 |
| `Messages.tsx` | 消息列表（147457 行） |
| `Message.tsx` | 单条消息（79113 行） |
| `MessageRow.tsx` | 消息行组件 |
| `MessageSelector.tsx` | 消息选择器 |
| `LogSelector.tsx` | 日志选择器（200487 行，最大文件） |
| `Stats.tsx` | 统计面板（152782 行） |
| `ScrollKeybindingHandler.tsx` | 滚动键绑定（149202 行） |
| `VirtualMessageList.tsx` | 虚拟消息列表 |
| `FullscreenLayout.tsx` | 全屏布局 |
| `StatusLine.tsx` | 状态栏 |
| `Spinner.tsx` | 加载动画 |
| `Markdown.tsx` | Markdown 渲染 |
| `ThemePicker.tsx` | 主题选择器 |
| `ModelPicker.tsx` | 模型选择器 |
| `GlobalSearchDialog.tsx` | 全局搜索 |
| `HistorySearchDialog.tsx` | 历史搜索 |
| `ContextVisualization.tsx` | 上下文可视化 |
| `OutputStylePicker.tsx` | 输出样式选择器 |
| `BridgeDialog.tsx` | 桥接对话框 |
| `Onboarding.tsx` | 引导流程 |
| `ResumeTask.tsx` | 恢复任务 |
| `AutoUpdater.tsx` | 自动更新 |
| `Feedback.tsx` | 反馈组件 |
| `FileEditToolDiff.tsx` | 文件编辑 Diff |
| `MCPServerApprovalDialog.tsx` | MCP 审批对话框 |

**子目录结构**：
- `permissions/` — 权限对话框（17 个子目录）
- `agents/` — Agent 相关组件
- `design-system/` — 设计系统组件
- `diff/` — Diff 展示组件
- `mcp/` — MCP 相关组件
- `memory/` — 记忆组件
- `messages/` — 消息子组件
- `PromptInput/` — 提示输入组件
- `Settings/` — 设置面板
- `shell/` — Shell 组件
- `skills/` — 技能相关组件
- `tasks/` — 任务组件
- `teams/` — 团队组件
- `TrustDialog/` — 信任对话框
- `ui/` — 通用 UI 组件
- `wizard/` — 向导组件

**核心职责**：
- 所有 UI 组件的实现
- 用户交互界面渲染
- 对话框、面板、按钮等 UI 元素

**关键依赖**：
- `src/ink/` — Ink 渲染引擎
- `src/state/` — 状态管理
- `src/hooks/` — React hooks

---

### 2.8 `constants/` 目录

**功能概述**：全局常量定义，包括 OAuth 配置、产品常量、查询来源等。

**文件结构**：
- `oauth.ts` — OAuth 配置
- `product.ts` — 产品常量
- `querySource.ts` — 查询来源定义

**核心职责**：
- 全局常量集中管理
- 产品级别配置常量

---

### 2.9 `context/` 目录

**功能概述**：全局上下文管理，包括系统上下文和用户上下文。

**文件结构**：
- `context.ts` — 上下文主文件

**核心职责**：
- 系统上下文（System Context）管理
- 用户上下文（User Context）管理
- 上下文信息的整合与传递

**关键依赖**：
- `src/types/message.ts` — 消息类型
- `src/utils/messages.ts` — 消息工具

---

### 2.10 `coordinator/` 目录

**功能概述**：协调器模式支持，用于多 Agent 协调工作。

**文件结构**：
- 支持 COORDINATOR_MODE 功能标志

**核心职责**：
- 多 Agent 协调调度
- 协调器模式初始化

---

### 2.11 `entrypoints/` 目录

**功能概述**：应用入口点定义，包括主入口和 Agent SDK 入口。

**文件结构**：
- `init.ts` — 主入口初始化
- `agentSdkTypes.ts` — Agent SDK 类型
- `sdk/` — SDK 入口点

**核心职责**：
- 应用启动初始化
- SDK 模式入口定义

---

### 2.12 `hooks/` 目录

**功能概述**：React Hooks 库，包含 110+ 个自定义 Hooks。

**文件结构**（主要 Hooks）：
| Hook | 说明 |
|------|------|
| `useGlobalKeybindings.tsx` | 全局快捷键绑定 |
| `useTypeahead.tsx` | 自动补全（212610 行） |
| `useVoiceIntegration.tsx` | 语音集成（99464 行） |
| `useReplBridge.tsx` | REPL 桥接（115652 行） |
| `useVoice.ts` | 语音控制（45802 行） |
| `useVirtualScroll.ts` | 虚拟滚动 |
| `useTextInput.ts` | 文本输入 |
| `useVimInput.ts` | Vim 输入 |
| `useInputBuffer.ts` | 输入缓冲 |
| `useCommandKeybindings.tsx` | 命令快捷键 |
| `useSearchInput.ts` | 搜索输入 |
| `usePromptSuggestion.ts` | 提示建议 |
| `useMergedTools.ts` | 工具合并 |
| `useMergedCommands.ts` | 命令合并 |
| `useMergedClients.ts` | 客户端合并 |
| `useCanUseTool.tsx` | 工具可用性检查 |
| `useSettings.ts` | 设置管理 |
| `useSettingsChange.ts` | 设置变更 |
| `useSkillsChange.ts` | 技能变更 |
| `useHistorySearch.ts` | 历史搜索 |
| `useTaskListWatcher.ts` | 任务列表监视 |
| `useTasksV2.ts` | 任务管理 V2 |
| `useBackgroundTaskNavigation.ts` | 后台任务导航 |
| `useDiffInIDE.ts` | IDE Diff |
| `useDiffData.ts` | Diff 数据 |
| `useTurnDiffs.ts` | 对话轮次 Diff |
| `useCancelRequest.ts` | 取消请求 |
| `useExitOnCtrlCD.ts` | Ctrl+C 退出 |
| `useExitOnCtrlCDWithKeybindings.ts` | 带快捷键的 Ctrl+C |
| `useClipboardImageHint.ts` | 剪贴板图片提示 |
| `usePasteHandler.ts` | 粘贴处理 |
| `useMailboxBridge.ts` | 邮箱桥接 |
| `useRemoteSession.ts` | 远程会话 |
| `useSSHSession.ts` | SSH 会话 |
| `useSwarmInitialization.ts` | Swarm 初始化 |
| `useSwarmPermissionPoller.ts` | Swarm 权限轮询 |
| `useTeleportResume.tsx` | 远程恢复 |
| `useIDEIntegration.tsx` | IDE 集成 |
| `useIDEIntegration.tsx` | IDE 集成 |
| `useLspPluginRecommendation.tsx` | LSP 插件推荐 |
| `useInboxPoller.ts` | 收件箱轮询 |
| `useOfficialMarketplaceNotification.tsx` | 官方市场通知 |
| `useClaudeCodeHintRecommendation.tsx` | Claude Code 提示推荐 |
| `useChromeExtensionNotification.tsx` | Chrome 扩展通知 |
| `usePrStatus.ts` | PR 状态 |
| `useMemoryUsage.ts` | 内存使用 |
| `useIdeSelection.ts` | IDE 选择 |
| `useIdeAtMentioned.ts` | IDE @提及 |
| `useIdeLogging.ts` | IDE 日志 |
| `useIdeConnectionStatus.ts` | IDE 连接状态 |
| `useAwaySummary.ts` | 离开摘要 |
| `useScheduledTasks.ts` | 计划任务 |
| `useElapsedTime.ts` | 耗时统计 |
| `useDoublePress.ts` | 双击 |
| `useBlink.ts` | 闪烁 |
| `useTimeout.ts` | 超时 |
| `useMinDisplayTime.ts` | 最小显示时间 |
| `useApiKeyVerification.ts` | API 密钥验证 |
| `useCopyOnSelect.ts` | 选择复制 |
| `useVoiceEnabled.ts` | 语音启用状态 |
| `useSessionBackgrounding.ts` | 会话后台化 |
| `useFileHistorySnapshotInit.ts` | 文件历史快照 |
| `useManagePlugins.ts` | 插件管理 |
| `useSkillImprovementSurvey.ts` | 技能改进调查 |
| `useQueueProcessor.ts` | 队列处理 |
| `useNotifyAfterTimeout.ts` | 超时通知 |
| `useDirectConnect.ts` | 直接连接 |
| `useUpdateNotification.ts` | 更新通知 |
| `useTeammateViewAutoExit.ts` | 队友视图自动退出 |
| `useTerminalSize.ts` | 终端尺寸 |
| `useLogMessages.ts` | 日志消息 |
| `useDeferredHookMessages.ts` | 延迟钩子消息 |
| `useCommandQueue.ts` | 命令队列 |
| `useAfterFirstRender.ts` | 首次渲染后 |
| `useExitOnCtrlCDWithKeybindings.ts` | 带快捷键退出 |
| `useDynamicConfig.ts` | 动态配置 |
| `useMainLoopModel.ts` | 主循环模型 |
| `usePromptSuggestion.ts` | 提示建议 |
| `fileSuggestions.ts` | 文件建议 |
| `unifiedSuggestions.ts` | 统一建议 |
| `renderPlaceholder.ts` | 渲染占位符 |

**子目录**：
- `notifs/` — 通知相关
- `toolPermission/` — 工具权限处理

**核心职责**：
- 所有 UI 交互逻辑的 Hook 封装
- 状态管理副作用处理
- 用户输入处理
- 快捷键绑定
- IDE 集成逻辑

---

### 2.13 `ink/` 目录

**功能概述**：Ink 渲染引擎核心，终端 UI 渲染的基础设施。

**文件结构**（主要文件，共 49 个文件）：
| 文件 | 说明 |
|------|------|
| `ink.tsx` | Ink 渲染主文件（251886 行，最大文件之一） |
| `screen.ts` | 屏幕管理（49323 行） |
| `selection.ts` | 选择管理（34933 行） |
| `render-node-to-output.ts` | 渲染节点到输出（63281 行） |
| `output.ts` | 输出管理（26183 行） |
| `log-update.ts` | 日志更新 |
| `parse-keypress.ts` | 按键解析 |
| `reconciler.ts` | 协调器 |
| `renderer.ts` | 渲染器 |
| `render-border.ts` | 边框渲染 |
| `render-to-screen.ts` | 渲染到屏幕 |
| `dom.ts` | DOM 管理 |
| `styles.ts` | 样式定义 |
| `Ansi.tsx` | ANSI 颜色解析 |
| `colorize.ts` | 颜色处理 |
| `wrap-text.ts` | 文本换行 |
| `wrapAnsi.ts` | ANSI 换行 |
| `stringWidth.ts` | 字符串宽度 |
| `measure-text.ts` | 文本测量 |
| `measure-element.ts` | 元素测量 |
| `line-width-cache.ts` | 行宽缓存 |
| `node-cache.ts` | 节点缓存 |
| `optimizer.ts` | 优化器 |
| `focus.ts` | 焦点管理 |
| `hit-test.ts` | 点击测试 |
| `terminal.ts` | 终端管理 |
| `terminal-focus-state.ts` | 终端焦点状态 |
| `terminal-querier.ts` | 终端查询 |
| `termio.ts` | 终端 IO |
| `useTerminalNotification.ts` | 终端通知 |
| `searchHighlight.ts` | 搜索高亮 |
| `bidi.ts` | 双向文本支持 |
| `squash-text-nodes.ts` | 文本节点压缩 |
| `clearTerminal.ts` | 清除终端 |
| `supports-hyperlinks.ts` | 超链接支持 |
| `widest-line.ts` | 最宽行 |
| `get-max-width.ts` | 获取最大宽度 |
| `tabstops.ts` | 制表符 |
| `warn.ts` | 警告 |
| `constants.ts` | 常量 |
| `instances.ts` | 实例 |
| `root.ts` | 根节点 |

**子目录**：
- `components/` — Ink 内部组件
- `events/` — 事件处理
- `hooks/` — Ink 钩子
- `layout/` — 布局
- `termio/` — 终端 IO

**核心职责**：
- 终端 UI 渲染引擎
- ANSI 颜色解析与渲染
- 文本布局与换行
- 用户输入（按键）处理
- 焦点管理
- 超链接支持

**关键依赖**：
- React（用于组件化）
- `src/utils/` — 工具函数

---

### 2.14 `keybindings/` 目录

**功能概述**：键盘快捷键管理。

**文件结构**：
- 快捷键定义与配置

**核心职责**：
- 全局快捷键注册
- 快捷键与命令映射

---

### 2.15 `memdir/` 目录

**功能概述**：内存目录（Memdir）管理，团队记忆功能。

**文件结构**：
- `memdir.ts` — 内存目录主文件
- `memoryAge.ts` — 记忆老化
- `memoryScan.ts` — 记忆扫描
- `memoryTypes.ts` — 记忆类型
- `findRelevantMemories.ts` — 查找相关记忆
- `paths.ts` — 路径管理
- `teamMemPaths.ts` — 团队记忆路径
- `teamMemPrompts.ts` — 团队记忆提示
- `memoryAge.ts` — 记忆年龄

**核心职责**：
- 团队共享记忆管理
- 记忆检索与匹配
- 记忆持久化

---

### 2.16 `migrations/` 目录

**功能概述**：数据迁移脚本，处理配置格式变更、模型升级等。

**文件结构**：
| 迁移文件 | 说明 |
|---------|------|
| `migrateBypassPermissionsAcceptedToSettings.ts` | 权限迁移 |
| `migrateFennecToOpus.ts` | Fennec 到 Opus |
| `migrateReplBridgeEnabledToRemoteControlAtStartup.ts` | REPL 桥接迁移 |
| `resetAutoModeOptInForDefaultOffer.ts` | 自动模式迁移 |
| `migrateEnableAllProjectMcpServersToSettings.ts` | MCP 设置迁移 |
| `migrateSonnet1mToSonnet45.ts` | Sonnet 模型迁移 |
| `resetProToOpusDefault.ts` | Pro 默认值迁移 |
| `migrateLegacyOpusToCurrent.ts` | Opus 遗留迁移 |
| `migrateSonnet45ToSonnet46.ts` | Sonnet 模型迁移 |
| `migrateAutoUpdatesToSettings.ts` | 自动更新迁移 |
| `migrateOpusToOpus1m.ts` | Opus 模型迁移 |

**核心职责**：
- 配置数据版本迁移
- 数据格式升级
- 默认值调整

---

### 2.17 `moreright/` 目录

**功能概述**：权限扩展模块，提供更细粒度的权限控制。

**核心职责**：
- 扩展权限管理
- 权限规则定义

---

### 2.18 `native-ts/` 目录

**功能概述**：原生 TypeScript 功能，包含文件索引、颜色 diff、Yoga 布局等。

**子目录**：
- `file-index/` — 文件索引
- `color-diff/` — 颜色 diff
- `yoga-layout/` — Yoga 布局

**核心职责**：
- 高性能原生实现
- 布局计算

---

### 2.19 `outputStyles/` 目录

**功能概述**：输出样式管理，控制终端输出格式。

**文件结构**：
- `loadOutputStylesDir.ts` — 加载输出样式目录

**核心职责**：
- 输出样式加载
- 样式配置管理

---

### 2.20 `plugins/` 目录

**功能概述**：插件系统管理，支持内置插件和第三方插件。

**文件结构**：
- `builtinPlugins.ts` — 内置插件定义
- `bundled/` — 捆绑插件
- `bundled/index.ts` — 捆绑插件索引

**核心职责**：
- 插件加载与初始化
- 插件生命周期管理
- 插件 API 提供

**关键依赖**：
- `src/services/plugins/` — 插件服务
- `src/types/plugin.ts` — 插件类型

---

### 2.21 `query/` 目录

**功能概述**：查询引擎核心模块，对话管理和 Token 预算管理。

**文件结构**：
| 文件 | 说明 |
|------|------|
| `query.ts` | 查询主文件（68683 行） |
| `QueryEngine.ts` | 查询引擎（46630 行） |
| `tokenBudget.ts` | Token 预算管理 |
| `deps.ts` | 依赖管理 |
| `config.ts` | 查询配置 |
| `stopHooks.ts` | 停止钩子 |

**核心职责**：
- 对话上下文管理
- Token 预算计算与控制
- 查询配置
- 对话轮次管理

**关键依赖**：
- `@anthropic-ai/sdk` — Anthropic SDK
- `src/tools/` — 工具系统
- `src/utils/` — 工具函数

---

### 2.22 `remote/` 目录

**功能概述**：远程连接相关功能。

**核心职责**：
- 远程会话连接
- 远程 IO 处理

---

### 2.23 `schemas/` 目录

**功能概述**：JSON Schema 定义，用于配置验证和类型生成。

**核心职责**：
- 配置 Schema 定义
- Schema 验证

---

### 2.24 `screens/` 目录

**功能概述**：全屏屏幕组件，类似页面级别的 UI 组织。

**核心职责**：
- 全屏视图渲染
- 页面级 UI 组织

---

### 2.25 `server/` 目录

**功能概述**：本地服务器功能，支持后台服务运行。

**核心职责**：
- 本地服务启动
- 后台进程管理

---

### 2.26 `services/` 目录

**功能概述**：服务层，包含 API、Analytics、MCP、OAuth 等核心服务。

**文件结构**（主要服务）：
| 服务目录 | 说明 |
|---------|------|
| `analytics/` | 分析服务 |
| `api/` | API 请求 |
| `autoDream/` | 自动 Dream |
| `compact/` | 压缩服务 |
| `extractMemories/` | 记忆提取 |
| `lsp/` | Language Server Protocol |
| `MagicDocs/` | Magic 文档 |
| `mcp/` | Model Context Protocol |
| `oauth/` | OAuth 认证 |
| `plugins/` | 插件服务 |
| `policyLimits/` | 策略限制 |
| `PromptSuggestion/` | 提示建议 |
| `remoteManagedSettings/` | 远程托管设置 |
| `SessionMemory/` | 会话记忆 |
| `settingsSync/` | 设置同步 |
| `teamMemorySync/` | 团队记忆同步 |
| `tips/` | 提示服务 |
| `tools/` | 工具服务 |
| `toolUseSummary/` | 工具使用摘要 |
| `AgentSummary/` | Agent 摘要 |

**核心职责**：
- API 请求封装
- 分析数据收集
- MCP 服务器管理
- OAuth 认证流程
- 插件服务
- 设置同步

---

### 2.27 `skills/` 目录

**功能概述**：技能（Skills）系统，支持内置技能和第三方技能加载。

**文件结构**：
- `bundledSkills.ts` — 内置技能
- `loadSkillsDir.ts` — 技能目录加载（34415 行）
- `mcpSkillBuilders.ts` — MCP 技能构建器
- `bundled/` — 捆绑技能

**核心职责**：
- 技能加载与管理
- MCP 技能集成
- 技能执行环境

---

### 2.28 `state/` 目录

**功能概述**：全局状态管理，React 状态和 Store 管理。

**文件结构**：
| 文件 | 说明 |
|------|------|
| `AppState.tsx` | 应用状态（23480 行） |
| `AppStateStore.ts` | 状态存储（21847 行） |
| `onChangeAppState.ts` | 状态变更处理 |
| `store.ts` | Store 定义 |
| `selectors.ts` | 状态选择器 |
| `teammateViewHelpers.ts` | 队友视图辅助 |

**核心职责**：
- 全局状态定义
- 状态变更通知
- 状态选择器

---

### 2.29 `tasks/` 目录

**功能概述**：任务系统，支持多种任务类型的定义和执行。

**文件结构**：
| 文件 | 说明 |
|------|------|
| `LocalMainSessionTask.ts` | 本地主会话任务 |
| `types.ts` | 任务类型定义 |
| `pillLabel.ts` | 任务标签 |
| `stopTask.ts` | 停止任务 |
| `LocalAgentTask/` | 本地 Agent 任务 |
| `LocalShellTask/` | 本地 Shell 任务 |
| `RemoteAgentTask/` | 远程 Agent 任务 |
| `InProcessTeammateTask/` | 进程内队友任务 |
| `DreamTask/` | Dream 任务 |

**核心职责**：
- 任务类型定义
- 任务状态管理
- 任务执行

**关键依赖**：
- `src/Task.ts` — 任务基类
- `src/state/` — 状态管理

---

### 2.30 `tools/` 目录

**功能概述**：工具系统核心，包含 45+ 内置工具的实现。

**文件结构**（主要工具）：
| 工具 | 说明 |
|------|------|
| `BashTool/` | Bash 执行工具 |
| `FileReadTool/` | 文件读取 |
| `FileEditTool/` | 文件编辑 |
| `FileWriteTool/` | 文件写入 |
| `GlobTool/` | 文件匹配 |
| `GrepTool/` | 文本搜索 |
| `WebSearchTool/` | 网页搜索 |
| `WebFetchTool/` | 网页获取 |
| `MCPTool/` | MCP 工具 |
| `McpAuthTool/` | MCP 认证 |
| `ReadMcpResourceTool/` | MCP 资源读取 |
| `ListMcpResourcesTool/` | MCP 资源列表 |
| `LSPTool/` | Language Server Protocol |
| `TaskCreateTool/` | 任务创建 |
| `TaskListTool/` | 任务列表 |
| `TaskGetTool/` | 任务获取 |
| `TaskUpdateTool/` | 任务更新 |
| `TaskStopTool/` | 任务停止 |
| `TaskOutputTool/` | 任务输出 |
| `EnterPlanModeTool/` | 进入计划模式 |
| `ExitPlanModeTool/` | 退出计划模式 |
| `EnterWorktreeTool/` | 进入工作树 |
| `ExitWorktreeTool/` | 退出工作树 |
| `AgentTool/` | Agent 工具 |
| `SkillTool/` | 技能工具 |
| `REPLTool/` | REPL 执行 |
| `PowerShellTool/` | PowerShell |
| `NotebookEditTool/` | Notebook 编辑 |
| `ConfigTool/` | 配置工具 |
| `AskUserQuestionTool/` | 提问工具 |
| `BriefTool/` | 摘要工具 |
| `SendMessageTool/` | 发送消息 |
| `ScheduleCronTool/` | 定时任务 |
| `SleepTool/` | 延迟工具 |
| `SyntheticOutputTool/` | 合成输出 |
| `TodoWriteTool/` | 待办事项 |
| `ToolSearchTool/` | 工具搜索 |
| `RemoteTriggerTool/` | 远程触发 |
| `TeamCreateTool/` | 团队创建 |
| `TeamDeleteTool/` | 团队删除 |
| `shared/` — 共享工具模块 |
| `testing/` — 测试工具 |

**核心职责**：
- 工具定义与实现
- 工具参数验证
- 工具执行与结果处理

**关键依赖**：
- `src/Tool.ts` — 工具基类
- `src/services/mcp/` — MCP 服务
- `src/permissions/` — 权限系统

---

### 2.31 `types/` 目录

**功能概述**：全局类型定义，包括消息类型、权限类型、插件类型等。

**文件结构**：
| 文件 | 说明 |
|------|------|
| `message.ts` | 消息类型定义 |
| `permissions.ts` | 权限类型（13145 行） |
| `plugin.ts` | 插件类型（11308 行） |
| `hooks.ts` | 钩子类型（9138 行） |
| `logs.ts` | 日志类型（11291 行） |
| `command.ts` | 命令类型 |
| `textInputTypes.ts` | 文本输入类型 |
| `ids.ts` | ID 类型 |
| `generated/` | 生成的类型 |

**核心职责**：
- 类型安全保证
- 接口定义
- 类型导出与再导出

---

### 2.32 `upstreamproxy/` 目录

**功能概述**：上游代理支持，用于网络请求代理。

**核心职责**：
- 代理配置
- 网络请求转发

---

### 2.33 `utils/` 目录

**功能概述**：工具函数库，包含 180+ 个工具文件，是代码量最大的目录之一。

**文件结构**（分类整理）：

**会话管理类**：
| 文件 | 说明 |
|------|------|
| `sessionStorage.ts` | 会话存储（180620 行） |
| `sessionStoragePortable.ts` | 便携式会话存储 |
| `sessionRestore.ts` | 会话恢复 |
| `sessionStart.ts` | 会话启动 |
| `sessionState.ts` | 会话状态 |
| `sessionTitle.ts` | 会话标题 |
| `sessionFileAccessHooks.ts` | 文件访问钩子 |
| `sessionEnvironment.ts` | 会话环境 |
| `sessionIngressAuth.ts` | 会话入口认证 |

**消息处理类**：
| 文件 | 说明 |
|------|------|
| `messages.ts` | 消息处理（193203 行，最大文件） |
| `messageQueueManager.ts` | 消息队列管理 |
| `collapseReadSearch.ts` | 压缩搜索结果 |
| `collapseTeammateShutdowns.ts` | 压缩队友关闭 |
| `collapseHookSummaries.ts` | 压缩钩子摘要 |
| `messagePredicates.ts` | 消息谓词 |

**IDE 集成类**：
| 文件 | 说明 |
|------|------|
| `ide.ts` | IDE 集成（46585 行） |
| `idePathConversion.ts` | IDE 路径转换 |
| `jumpcloud.ts` | JumpCloud 集成 |
| `jetbrains.ts` | JetBrains 集成 |
| `claudeDesktop.ts` | Claude Desktop 集成 |

**Git 操作类**：
| 文件 | 说明 |
|------|------|
| `git.ts` | Git 操作（30270 行） |
| `gitDiff.ts` | Git Diff |
| `gitSettings.ts` | Git 设置 |
| `detectRepository.ts` | 仓库检测 |
| `githubRepoPathMapping.ts` | GitHub 路径映射 |

**Shell 相关**：
| 文件 | 说明 |
|------|------|
| `Shell.ts` | Shell 管理（16929 行） |
| `ShellCommand.ts` | Shell 命令 |
| `shellConfig.ts` | Shell 配置 |
| `bash/` — Bash 相关工具 |
| `shell/` — Shell 工具 |

**权限相关**：
| 文件 | 说明 |
|------|------|
| `permissions/` — 权限工具 |
| `privacyLevel.ts` — 隐私级别 |

**MCP 相关**：
| 文件 | 说明 |
|------|------|
| `mcp/` — MCP 工具 |
| `mcpInstructionsDelta.ts` — MCP 指令增量 |
| `mcpOutputStorage.ts` — MCP 输出存储 |
| `mcpValidation.ts` — MCP 验证 |
| `mcpWebSocketTransport.ts` — MCP WebSocket 传输 |

**文件操作类**：
| 文件 | 说明 |
|------|------|
| `file.ts` | 文件操作（18256 行） |
| `fileHistory.ts` | 文件历史（34660 行） |
| `fileReadCache.ts` | 文件读取缓存 |
| `fileRead.ts` | 文件读取 |
| `fileStateCache.ts` | 文件状态缓存 |
| `fsOperations.ts` | 文件系统操作 |
| `readFileInRange.ts` | 范围读取 |
| `filePersistence/` — 文件持久化 |

**配置管理类**：
| 文件 | 说明 |
|------|------|
| `config.ts` | 配置管理（63496 行） |
| `markdownConfigLoader.ts` | Markdown 配置加载 |
| `envDynamic.ts` | 动态环境变量 |
| `env.ts` | 环境变量 |
| `envUtils.ts` | 环境工具 |
| `envValidation.ts` | 环境验证 |
| `managedEnv.ts` | 托管环境 |
| `settings/` — 设置工具 |

**状态管理类**：
| 文件 | 说明 |
|------|------|
| `status.tsx` | 状态管理（48635 行） |
| `statusNoticeDefinitions.tsx` | 状态通知定义 |
| `apponta.ts` | Aponta 状态 |

**其他工具类**：
| 文件 | 说明 |
|------|------|
| `tokens.ts` | Token 计算 |
| `tokenBudget.ts` | Token 预算 |
| `markdown.ts` | Markdown 处理 |
| `markdownConfigLoader.ts` | Markdown 配置 |
| `json.ts` | JSON 工具 |
| `jsonRead.ts` | JSON 读取 |
| `yaml.ts` | YAML 工具 |
| `xml.ts` | XML 工具 |
| `pdf.ts` | PDF 处理 |
| `imageResizer.ts` | 图片调整 |
| `imageStore.ts` | 图片存储 |
| `imagePaste.ts` | 图片粘贴 |
| `imageValidation.ts` | 图片验证 |

**调试/诊断类**：
| 文件 | 说明 |
|------|------|
| `debug.ts` | 调试工具 |
| `debugFilter.ts` | 调试过滤器 |
| `diagLogs.ts` | 诊断日志 |
| `errorLogSink.ts` | 错误日志接收 |
| `log.ts` | 日志工具 |
| `heapDumpService.ts` | 堆转储服务 |
| `startupProfiler.ts` | 启动分析 |

**后台任务类**：
| 文件 | 说明 |
|------|------|
| `background/` — 后台任务 |
| `cron.ts` | Cron 调度 |
| `cronScheduler.ts` | Cron 调度器 |
| `cronTasks.ts` | Cron 任务 |
| `cronTasksLock.ts` | Cron 任务锁 |
| `cronJitterConfig.ts` | Cron 抖动配置 |

**Teleporter/Remote**：
| 文件 | 说明 |
|------|------|
| `teleport.tsx` | 远程传输（175779 行） |
| `teleport/` — 传输工具 |
| `tmuxSocket.ts` | Tmux 套接字 |

**Swarm/Team**：
| 文件 | 说明 |
|------|------|
| `swarm/` — Swarm 工具 |
| `teammate.ts` | 队友管理 |
| `teammateMailbox.ts` | 队友邮箱 |
| `teammateContext.ts` | 队友上下文 |
| `teamDiscovery.ts` | 团队发现 |
| `teamMemoryOps.ts` | 团队记忆操作 |

**主题/样式类**：
| 文件 | 说明 |
|------|------|
| `theme.ts` | 主题（26830 行） |
| `themeUtils.ts` | 主题工具 |
| `systemTheme.ts` | 系统主题 |
| `format.ts` | 格式化 |

**hooks 工具类**：
| 文件 | 说明 |
|------|------|
| `hooks.ts` | Hooks 工具（159458 行，第二大文件） |

**工具池/结果存储**：
| 文件 | 说明 |
|------|------|
| `toolPool.ts` | 工具池 |
| `toolResultStorage.ts` | 工具结果存储 |
| `toolErrors.ts` | 工具错误 |
| `toolSchemaCache.ts` | 工具 Schema 缓存 |

**其他**：
- `analytics/` — 分析工具
- `computer_use/` — 计算机使用
- `deepLink/` — 深度链接
- `dxt/` — DXT 工具
- `filePersistence/` — 文件持久化
- `git/` — Git 工具
- `github/` — GitHub 工具
- `hooks/` — Hook 工具
- `memory/` — 记忆工具
- `model/` — 模型工具
- `nativeInstaller/` — 原生安装
- `plugins/` — 插件工具
- `powershell/` — PowerShell 工具
- `processUserInput/` — 输入处理
- `sandbox/` — 沙箱工具
- `secureStorage/` — 安全存储
- `suggestions/` — 建议工具
- `swarm/backends/` — Swarm 后端
- `telemetry/` — 遥测
- `teleport/` — 传输工具
- `todo/` — 待办工具
- `ultraplan/` — UltraPlan

**核心职责**：
- 工具函数集合
- 业务逻辑封装
- 配置管理
- 文件操作
- 网络请求

---

### 2.34 `vim/` 目录

**功能概述**：Vim 模式支持，提供类似 Vim 的文本操作体验。

**文件结构**：
| 文件 | 说明 |
|------|------|
| `textObjects.ts` | 文本对象 |
| `operators.ts` | 操作符 |
| `motions.ts` | 动作 |
| `transitions.ts` | 状态转换 |
| `types.ts` | 类型定义 |

**核心职责**：
- Vim 模式模拟
- 文本对象定义
- 操作符与动作
- 状态机转换

---

### 2.35 `voice/` 目录

**功能概述**：语音输入/输出支持。

**核心职责**：
- 语音识别集成
- 语音合成
- 语音控制

---

## 3. 核心文件索引

以下列出最核心的 30 个文件，按重要性排序：

| 排名 | 文件 | 行数 | 职责 |
|------|------|------|------|
| 1 | `src/main.tsx` | 803924 | 应用主入口，所有初始化逻辑 |
| 2 | `src/utils/messages.ts` | 193203 | 消息处理核心 |
| 3 | `src/components/LogSelector.tsx` | 200487 | 日志选择组件 |
| 4 | `src/hooks/useTypeahead.tsx` | 212610 | 自动补全 |
| 5 | `src/hooks/useReplBridge.tsx` | 115652 | REPL 桥接 |
| 6 | `src/commands/insights.ts` | 115949 | 分析命令 |
| 7 | `src/components/Messages.tsx` | 147457 | 消息列表 |
| 8 | `src/components/ScrollKeybindingHandler.tsx` | 149202 | 滚动处理 |
| 9 | `src/components/Stats.tsx` | 152782 | 统计面板 |
| 10 | `src/hooks/useVoiceIntegration.tsx` | 99464 | 语音集成 |
| 11 | `src/bridge/replBridge.ts` | 100537 | REPL 桥接 |
| 12 | `src/tasks/LocalMainSessionTask.ts` | 15136 | 本地任务 |
| 13 | `src/bridge/bridgeMain.ts` | 115571 | 桥接主逻辑 |
| 14 | `src/commands/ultraplan.tsx` | 66629 | 规划命令 |
| 15 | `src/query/query.ts` | 68683 | 查询主文件 |
| 16 | `src/components/Message.tsx` | 79113 | 消息组件 |
| 17 | `src/utils/status.tsx` | 48635 | 状态管理 |
| 18 | `src/query/QueryEngine.ts` | 46630 | 查询引擎 |
| 19 | `src/utils/ide.ts` | 46585 | IDE 集成 |
| 20 | `src/ink/ink.tsx` | 251886 | Ink 渲染 |
| 21 | `src/Tool.ts` | 29516 | 工具基类 |
| 22 | `src/Task.ts` | 3152 | 任务基类 |
| 23 | `src/commands.ts` | 25185 | 命令注册 |
| 24 | `src/dialogLaunchers.tsx` | 22948 | 对话框启动器 |
| 25 | `src/query.ts` | 68683 | 查询管理 |
| 26 | `src/ink/screen.ts` | 49323 | 屏幕管理 |
| 27 | `src/ink/selection.ts` | 34933 | 选择管理 |
| 28 | `src/utils/sessionStorage.ts` | 180620 | 会话存储 |
| 29 | `src/utils/hooks.ts` | 159458 | Hooks 工具 |
| 30 | `src/services/tools.ts` | N/A | 工具服务 |

---

## 4. 文件命名规范

Claude Code 源码遵循统一的文件命名规范：

### 4.1 文件类型后缀

| 后缀 | 类型 | 示例 |
|------|------|------|
| `.ts` | TypeScript 源文件 | `Tool.ts`、`Task.ts` |
| `.tsx` | React TSX 组件 | `App.tsx`、`Messages.tsx` |
| `.js` | JavaScript 文件（少量） | 工具脚本 |

### 4.2 命名约定

**组件命名**：
- React 组件：PascalCase（帕斯卡命名）
  - `App.tsx`、`MessageRow.tsx`、`StatusLine.tsx`
- 非组件工具：camelCase（驼峰命名）
  - `sessionStorage.ts`、`gitDiff.ts`

**目录命名**：
- 目录名：kebab-case（短横线命名）或 camelCase
  - `file-index/`、`color-diff/`、`native-ts/`

**工具 Hooks**：
- Hooks 以 `use` 前缀开头
  - `useTextInput.ts`、`useVimInput.ts`、`useHistorySearch.ts`

**类型定义**：
- 类型文件通常与功能模块同名
  - `types/permissions.ts` → 定义 `PermissionMode` 等类型
  - `types/message.ts` → 定义消息相关类型

**常量文件**：
- 常量目录 `constants/` 存放配置常量
  - `constants/oauth.ts`、`constants/product.ts`

### 4.3 文件组织原则

1. **共址原则**：相关功能放在同一目录
2. **单一职责**：每个文件尽量单一职责
3. **按类型分目录**：组件在 `components/`、工具在 `tools/`、服务在 `services/`
4. **索引文件**：`index.ts` 用于目录导出

---

## 5. 模块依赖关系

### 5.1 核心依赖流向图

```
main.tsx (入口)
  ├─► bootstrap/state.ts (引导状态)
  ├─► commands.ts (命令注册)
  ├─► state/AppState.tsx (全局状态)
  │     └─► hooks/ (各种Hooks)
  │           └─► components/ (UI组件)
  │                 └─► ink/ (渲染引擎)
  │
  ├─► query/ (查询引擎)
  │     └─► tools.ts (工具系统)
  │           └─► tools/* (具体工具)
  │
  ├─► dialogLaunchers.tsx (对话框)
  │     └─► components/ (UI组件)
  │
  └─► services/ (服务层)
        ├─► api/ (API服务)
        ├─► mcp/ (MCP服务)
        ├─► analytics/ (分析服务)
        └─► plugins/ (插件服务)
```

### 5.2 主要模块依赖关系

**状态管理依赖**：
```
state/AppState.tsx
  ├─► state/AppStateStore.ts
  ├─► state/onChangeAppState.ts
  └─► hooks/useCanUseTool.tsx
```

**工具系统依赖**：
```
tools.ts
  ├─► Tool.ts
  ├─► tools/BashTool/
  ├─► tools/FileReadTool/
  ├─► tools/FileEditTool/
  └─► tools/WebSearchTool/
```

**查询引擎依赖**：
```
QueryEngine.ts
  ├─► query.ts
  ├─► query/tokenBudget.ts
  └─► utils/tokens.ts
```

**桥接系统依赖**：
```
bridge/bridgeMain.ts
  ├─► bridge/replBridge.ts
  ├─► bridge/bridgeMessaging.ts
  └─► bridge/sessionRunner.ts
```

**UI 渲染依赖**：
```
components/App.tsx
  ├─► components/Messages.tsx
  ├─► components/MessageRow.tsx
  ├─► ink/ink.tsx
  └─► hooks/ (useVirtualScroll, etc.)
```

### 5.3 循环依赖处理

代码库中存在一些循环依赖，通过以下方式解决：

1. **Lazy Require**：在 `main.tsx` 中使用 `require()` 延迟加载
2. **类型再导出**：在 `types/` 中集中导出类型
3. **接口拆分**：将共同类型提取到独立文件

```typescript
// 延迟加载示例 (main.tsx)
const getTeammateUtils = () => require('./utils/teammate.js');
```

---

## 6. 关键文件速查表

### 6.1 入口与初始化

| 用途 | 文件路径 |
|------|----------|
| 主入口 | `/src/main.tsx` |
| 命令行入口 | `/src/cli/print.ts` |
| 初始化 | `/src/entrypoints/init.ts` |
| 状态初始化 | `/src/bootstrap/state.ts` |

### 6.2 状态管理

| 用途 | 文件路径 |
|------|----------|
| 应用状态定义 | `/src/state/AppState.tsx` |
| 状态存储 | `/src/state/AppStateStore.ts` |
| 状态变更处理 | `/src/state/onChangeAppState.ts` |
| 状态选择器 | `/src/state/selectors.ts` |

### 6.3 消息与对话

| 用途 | 文件路径 |
|------|----------|
| 消息处理核心 | `/src/utils/messages.ts` |
| 消息类型定义 | `/src/types/message.ts` |
| 消息列表组件 | `/src/components/Messages.tsx` |
| 单条消息组件 | `/src/components/Message.tsx` |
| 消息行组件 | `/src/components/MessageRow.tsx` |

### 6.4 工具系统

| 用途 | 文件路径 |
|------|----------|
| 工具基类 | `/src/Tool.ts` |
| 工具注册 | `/src/tools.ts` |
| Bash 工具 | `/src/tools/BashTool/index.ts` |
| 文件读取 | `/src/tools/FileReadTool/index.ts` |
| 文件编辑 | `/src/tools/FileEditTool/index.ts` |
| Web 搜索 | `/src/tools/WebSearchTool/index.ts` |
| MCP 工具 | `/src/tools/MCPTool/index.ts` |

### 6.5 任务系统

| 用途 | 文件路径 |
|------|----------|
| 任务基类 | `/src/Task.ts` |
| 任务类型定义 | `/src/tasks/types.ts` |
| 本地 Shell 任务 | `/src/tasks/LocalShellTask/index.ts` |
| 本地 Agent 任务 | `/src/tasks/LocalAgentTask/index.ts` |
| 远程 Agent 任务 | `/src/tasks/RemoteAgentTask/index.ts` |

### 6.6 查询引擎

| 用途 | 文件路径 |
|------|----------|
| 查询引擎核心 | `/src/query/QueryEngine.ts` |
| 查询主文件 | `/src/query/query.ts` |
| Token 预算 | `/src/query/tokenBudget.ts` |
| 查询配置 | `/src/query/config.ts` |

### 6.7 UI 渲染

| 用途 | 文件路径 |
|------|----------|
| Ink 渲染引擎 | `/src/ink/ink.tsx` |
| 屏幕管理 | `/src/ink/screen.ts` |
| 样式定义 | `/src/ink/styles.ts` |
| 选择管理 | `/src/ink/selection.ts` |
| ANSI 解析 | `/src/ink/Ansi.tsx` |
| 文本换行 | `/src/ink/wrap-text.ts` |

### 6.8 桥接与远程

| 用途 | 文件路径 |
|------|----------|
| 桥接主逻辑 | `/src/bridge/bridgeMain.ts` |
| REPL 桥接 | `/src/bridge/replBridge.ts` |
| 远程桥接核心 | `/src/bridge/remoteBridgeCore.ts` |
| 桥接消息 | `/src/bridge/bridgeMessaging.ts` |
| 会话运行器 | `/src/bridge/sessionRunner.ts` |

### 6.9 服务层

| 用途 | 文件路径 |
|------|----------|
| API 服务 | `/src/services/api/` |
| MCP 服务 | `/src/services/mcp/` |
| 分析服务 | `/src/services/analytics/` |
| OAuth 服务 | `/src/services/oauth/` |
| 插件服务 | `/src/services/plugins/` |
| 设置同步 | `/src/services/settingsSync/` |

### 6.10 配置与常量

| 用途 | 文件路径 |
|------|----------|
| 全局配置 | `/src/utils/config.ts` |
| 配置管理 | `/src/utils/config.ts` |
| OAuth 配置 | `/src/constants/oauth.ts` |
| 产品常量 | `/src/constants/product.ts` |
| 权限类型 | `/src/types/permissions.ts` |
| 插件类型 | `/src/types/plugin.ts` |

### 6.11 Hooks

| 用途 | 文件路径 |
|------|----------|
| 全局快捷键 | `/src/hooks/useGlobalKeybindings.tsx` |
| 文本输入 | `/src/hooks/useTextInput.ts` |
| Vim 输入 | `/src/hooks/useVimInput.ts` |
| 虚拟滚动 | `/src/hooks/useVirtualScroll.ts` |
| 设置管理 | `/src/hooks/useSettings.ts` |
| 历史搜索 | `/src/hooks/useHistorySearch.ts` |
| 任务监视 | `/src/hooks/useTaskListWatcher.ts` |
| IDE Diff | `/src/hooks/useDiffInIDE.ts` |

### 6.12 Skills 与 Plugins

| 用途 | 文件路径 |
|------|----------|
| 技能加载 | `/src/skills/loadSkillsDir.ts` |
| 内置技能 | `/src/skills/bundledSkills.ts` |
| MCP 技能构建 | `/src/skills/mcpSkillBuilders.ts` |
| 内置插件 | `/src/plugins/builtinPlugins.ts` |
| 插件服务 | `/src/services/plugins/` |

---

## 7. 目录快速索引

### 7.1 按功能快速定位

**想要修改 UI 组件？**
→ `/src/components/` 及子目录

**想要修改命令实现？**
→ `/src/commands/` 目录

**想要修改工具定义？**
→ `/src/tools/` 目录

**想要修改终端渲染？**
→ `/src/ink/` 目录

**想要修改状态管理？**
→ `/src/state/` 目录

**想要添加新的 Hook？**
→ `/src/hooks/` 目录

**想要修改 API 调用？**
→ `/src/services/api/` 目录

**想要修改桥接逻辑？**
→ `/src/bridge/` 目录

**想要修改会话存储？**
→ `/src/utils/sessionStorage.ts`

**想要修改会话消息处理？**
→ `/src/utils/messages.ts`

**想要修改 IDE 集成？**
→ `/src/utils/ide.ts`

**想要修改 Git 操作？**
→ `/src/utils/git.ts`

### 7.2 按文件大小快速定位问题

**最大文件 TOP 5**：
1. `src/main.tsx` — 803KB
2. `src/utils/messages.ts` — 193KB
3. `src/components/LogSelector.tsx` — 200KB
4. `src/hooks/useTypeahead.tsx` — 212KB
5. `src/hooks/useReplBridge.tsx` — 115KB

**大型服务文件**：
- `src/commands/insights.ts` — 115KB
- `src/bridge/bridgeMain.ts` — 115KB
- `src/bridge/replBridge.ts` — 100KB
- `src/query/query.ts` — 68KB
- `src/commands/ultraplan.tsx` — 66KB

---

## 8. 附录：源码统计信息

| 指标 | 数值 |
|------|------|
| 一级目录数 | 37 |
| TypeScript/TSX 文件总数 | ~1884 |
| `commands/` 子目录数 | 88 |
| `tools/` 工具数 | 45+ |
| `hooks/` Hooks 数 | 110+ |
| `components/` 组件目录数 | 33 |
| `services/` 服务目录数 | 22 |
| 最大源文件 | `src/main.tsx` (803KB) |
| 第二大文件 | `src/utils/messages.ts` (193KB) |

---

*本文档基于 Claude Code 源码分析生成，最后更新于 2026/05/18。*