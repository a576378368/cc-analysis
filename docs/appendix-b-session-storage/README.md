# 附录B：Session Storage 与 Resume 持久化机制

## B.1 Session Storage 概述

Session Storage 是 Claude Code 的核心持久化组件，负责管理对话 transcript 的存储与恢复。它采用纯 Node.js 实现，无内部依赖（无日志、无实验特性标志、无功能开关），可在 CLI 和 VS Code 扩展之间共享。

### B.1.1 核心设计目标

1. **跨平台兼容性**：路径清理（sanitization）替换所有非字母数字字符为连字符，支持 Windows 等文件系统限制
2. **大文件高效读取**：使用 head/tail 缓冲区读取元数据，避免解析完整 JSONL 文件
3. **工作树支持**：自动处理 Git worktree 场景下的会话查找
4. **UUID 验证**：会话 ID 格式校验，确保数据完整性

### B.1.2 存储结构

```
~/.claude/projects/
  └── {sanitized_project_path}/
      └── {sessionId}.jsonl
```

- **JSONL 格式**：每行一个 JSON 对象，便于流式追加和增量读取
- **会话元数据**：嵌入在首条用户消息中（customTitle、tag 等字段）

### B.1.3 核心文件

| 文件 | 职责 |
|------|------|
| `src/utils/sessionStorage.ts` | CLI 会话存储实现 |
| `src/utils/sessionStoragePortable.ts` | 跨平台共享工具库 |

## B.2 Transcript 管理

### B.2.1 消息格式

Transcript 中的每条消息都是包含 `type`、`message`、`timestamp` 等字段的 JSON 对象：

```typescript
interface Message {
  type: 'user' | 'assistant' | 'system' | 'attachment' | 'hook_result'
  message: {
    id?: string
    content: string | ContentBlock[]
    role?: 'user' | 'assistant'
  }
  timestamp: string
  uuid: string
  // ... 其他字段
}
```

### B.2.2 轻量级读取（Lite Read）

对于仅需元数据（如 customTitle、tag）的场景，使用 `readSessionLite()` 仅读取文件首尾各 64KB：

```typescript
export async function readSessionLite(
  filePath: string,
): Promise<LiteSessionFile | null> {
  const buf = Buffer.allocUnsafe(LITE_READ_BUF_SIZE)
  const headResult = await fh.read(buf, 0, LITE_READ_BUF_SIZE, 0)
  const tailOffset = Math.max(0, stat.size - LITE_READ_BUF_SIZE)
  // ...
}
```

这种设计优势：
- **零解析**：通过正则表达式提取字段，无需完整解析 JSON
- **内存高效**：固定 64KB 缓冲区，支持任意大小文件
- **快速定位**：使用 `extractJsonStringField()` / `extractLastJsonStringField()` 提取字段

### B.2.3 首条 Prompt 提取

`extractFirstPromptFromHead()` 从文件头提取首个有意义的用户提示：

```typescript
const SKIP_FIRST_PROMPT_PATTERN =
  /^(?:\s*<[a-z][\w-]*[\s>]|\[Request interrupted by user[^\]]*\])/
```

跳过规则：
- `isMeta: true` 的系统消息
- `isCompactSummary: true` 的压缩摘要
- `<command-name>` 包装的命令
- XML 标签开头的 IDE 上下文消息

### B.2.4 路径解析与工作树支持

```typescript
export async function resolveSessionFilePath(
  sessionId: string,
  dir?: string,
): Promise<{ filePath: string; projectPath: string | undefined; fileSize: number } | undefined>
```

查找顺序：
1. 规范化项目路径 → `findProjectDir()` 精确匹配
2. 工作树回退：扫描兄弟工作树目录
3. 全局扫描：无 `dir` 时遍历 `~/.claude/projects/` 下所有项目

**哈希冲突处理**：当路径超过 200 字符时，Bun.hash 与 Node.js 的 djb2Hash 产生不同后缀。`findProjectDir()` 通过前缀匹配处理：

```typescript
const prefix = sanitized.slice(0, MAX_SANITIZED_LENGTH)
const match = dirents.find(d => d.name.startsWith(prefix + '-'))
```

## B.3 Resume 恢复机制

### B.3.1 压缩边界（Compact Boundary）

Resume 时需要识别最后一次完整压缩的位置。系统通过 `compact_boundary` 标记消息定位：

```typescript
function parseBoundaryLine(line: string): { hasPreservedSegment: boolean } | null {
  const parsed = JSON.parse(line)
  if (parsed.type !== 'system' || parsed.subtype !== 'compact_boundary') {
    return null
  }
  return { hasPreservedSegment: Boolean(parsed.compactMetadata?.preservedSegment) }
}
```

### B.3.2 分块读取协议（Chunked Read Protocol）

`readTranscriptForLoad()` 实现高效的边界定位：

```
1. 初始化 8MB 输出缓冲区（最小化 grow 操作）
2. 逐 1MB 分块读取文件
3. 行扫描：检测 compact_boundary 标记
4. 截断输出：遇到 boundary 时清空缓冲区，从 boundary 位置重新开始
5. 保留最后一个 attribution-snapshot（附加到 EOF）
```

关键数据结构：

```typescript
type LoadState = {
  out: Sink                    // 输出缓冲区
  boundaryStartOffset: number  // 压缩边界文件偏移
  hasPreservedSegment: boolean // 是否保留片段
  lastSnapSrc: Buffer | null   // 最后 attr-snap 的内容
  lastSnapLen: number         // 最后 attr-snap 的长度
  carryBuf: Buffer            // 跨块边界的行片段
}
```

### B.3.3 跨块行处理（Carry Buffer）

当 `compact_boundary` 标记跨越分块边界时，使用 `carryBuf` 暂存不完整行：

```typescript
function processStraddle(s: LoadState, chunk: Buffer, bytesRead: number): number {
  if (s.carryLen === 0) return 0
  // 合并 carryBuf + chunk 首行
  const combined = cb.toString('utf-8', 0, s.carryLen) + chunk.toString('utf-8', 0, firstNl)
  // 检测是否是 compact_boundary
}
```

### B.3.4 归因快照（Attribution Snapshot）

压缩后保留最后一个 attribution-snapshot，追加到输出文件末尾：

```typescript
function finalizeOutput(s: LoadState): void {
  if (s.carryLen > 0) {
    sinkWrite(s.out, cb, 0, s.carryLen)
  }
  if (s.lastSnapSrc) {
    sinkWrite(s.out, s.lastSnapSrc, 0, s.lastSnapLen)
  }
}
```

## B.4 自动压缩机制

### B.4.1 压缩触发阈值

```typescript
export const AUTOCOMPACT_BUFFER_TOKENS = 13_000
export const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
export const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
```

阈值计算：

```typescript
export function getAutoCompactThreshold(model: string): number {
  const effectiveContextWindow = getEffectiveContextWindowSize(model)
  return effectiveContextWindow - AUTOCOMPACT_BUFFER_TOKENS
}
```

### B.4.2 压缩决策流程

```
shouldAutoCompact()
  ├── 递归守卫：session_memory / compact 子代理
  ├── 上下文折叠模式：CONTEXT_COLLAPSE 启用时跳过
  ├── 自动压缩开关检查：isAutoCompactEnabled()
  └── Token 阈值检查：>= autoCompactThreshold
```

**电路断路器**：连续 3 次压缩失败后停止自动压缩，防止无限重试。

### B.4.3 压缩执行路径

```
autoCompactIfNeeded()
  ├── 尝试 Session Memory 压缩（优先）
  │   └── trySessionMemoryCompaction()
  └── 传统压缩
      └── compactConversation()
```

### B.4.4 Micro Compact（微压缩）

微压缩是轻量级压缩，在每个 API 调用前运行：

**时间触发微压缩**（`timeBasedMCConfig.ts`）：
- 当距上次助手消息的时间超过 60 分钟（服务器端缓存 TTL 到期）
- 保留最近 N 个可压缩工具结果，清除旧结果
- 减少 prompt 重写量

```typescript
export type TimeBasedMCConfig = {
  enabled: boolean
  gapThresholdMinutes: number  // 默认 60 分钟
  keepRecent: number         // 默认保留 5 个
}
```

**缓存编辑微压缩**（`CACHED_MICROCOMPACT`）：
- 使用 Anthropic 的 `cache_edits` API
- 不修改本地消息内容，仅通过 API 层指示删除哪些工具结果
- 保持服务器端缓存的缓存前缀

### B.4.5 压缩提示词

压缩摘要提示词支持两种模式：

**基础压缩**（完整摘要）：
```typescript
const BASE_COMPACT_PROMPT = `Your task is to create a detailed summary of the conversation...
1. Primary Request and Intent
2. Key Technical Concepts
3. Files and Code Sections
4. Errors and fixes
5. Problem Solving
6. All user messages
7. Pending Tasks
8. Current Work
9. Optional Next Step
```

**部分压缩**（仅摘要近期消息）：
- 用于保留前缀 + 压缩中间部分的场景
- "earlier messages are being kept intact"

### B.4.6 压缩后清理

`runPostCompactCleanup()` 在每次压缩后执行：

```typescript
export function runPostCompactCleanup(querySource?: QuerySource): void {
  resetMicrocompactState()
  if (isMainThreadCompact) {
    getUserContext.cache.clear?.()
    resetGetMemoryFilesCache('compact')
  }
  clearSystemPromptSections()
  clearClassifierApprovals()
  clearSpeculativeChecks()
  clearBetaTracingState()
  clearSessionMessagesCache()
}
```

## B.5 清理策略

### B.5.1 警告状态管理

压缩警告抑制状态通过 React 外部存储实现：

```typescript
export const compactWarningStore = createStore<boolean>(false)

export function suppressCompactWarning(): void {
  compactWarningStore.setState(() => true)
}

export function clearCompactWarningSuppression(): void {
  compactWarningStore.setState(() => false)
}
```

- **抑制**：压缩成功后抑制警告（此时无准确 token 计数）
- **清除**：新压缩尝试开始时清除抑制状态

### B.5.2 API 轮次分组

`groupMessagesByApiRound()` 将消息按 API 往返分组：

```typescript
export function groupMessagesByApiRound(messages: Message[]): Message[][] {
  let lastAssistantId: string | undefined
  for (const msg of messages) {
    if (msg.type === 'assistant' && msg.message.id !== lastAssistantId && current.length > 0) {
      groups.push(current)
      current = [msg]
    }
    // ...
  }
}
```

边界触发条件：新的 assistant 消息 ID（不同 API 响应）

### B.5.3 图像剥离

压缩请求前剥离图像以避免触发 `prompt_too_long`：

```typescript
export function stripImagesFromMessages(messages: Message[]): Message[] {
  return messages.map(message => {
    if (message.type !== 'user') return message
    return {
      ...message,
      message: {
        ...message.message,
        content: content.flatMap(block => {
          if (block.type === 'image') return [{ type: 'text', text: '[image]' }]
          if (block.type === 'document') return [{ type: 'text', text: '[document]' }]
          return [block]
        }),
      },
    }
  })
}
```

### B.5.4 重新注入附件过滤

压缩摘要不应包含重复注入的附件类型：

```typescript
export function stripReinjectedAttachments(messages: Message[]): Message[] {
  return messages.filter(m =>
    !(m.type === 'attachment' &&
      (m.attachment.type === 'skill_discovery' || m.attachment.type === 'skill_listing'))
  )
}
```

### B.5.5 PTL 重试截断

当压缩请求本身触发 `prompt_too_long` 时，使用 `truncateHeadForPTLRetry()` 回退：

```typescript
export function truncateHeadForPTLRetry(
  messages: Message[],
  ptlResponse: AssistantMessage,
): Message[] | null {
  const groups = groupMessagesByApiRound(input)
  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  // 按需丢弃最早的 20% 分组
}
```

## B.6 关键数据结构

### B.6.1 CompactionResult

```typescript
export interface CompactionResult {
  boundaryMarker: SystemMessage           // 压缩边界标记
  summaryMessages: UserMessage[]        // 摘要消息
  attachments: AttachmentMessage[]      // 附件
  hookResults: HookResultMessage[]      // Hook 结果
  messagesToKeep?: Message[]            // 保留的消息
  userDisplayMessage?: string          // 用户显示消息
  preCompactTokenCount?: number        // 压缩前 token 数
  postCompactTokenCount?: number       // 压缩后 token 数
  truePostCompactTokenCount?: number  // 实际压缩后 token 数
  compactionUsage?: ReturnType<typeof getTokenUsage>
}
```

### B.6.2 RecompactionInfo

```typescript
export type RecompactionInfo = {
  isRecompactionInChain: boolean        // 是否在压缩链中
  turnsSincePreviousCompact: number    // 距上次压缩的轮次
  previousCompactTurnId?: string       // 上次压缩的 turn ID
  autoCompactThreshold: number         // 自动压缩阈值
  querySource?: QuerySource            // 查询来源
}
```

### B.6.3 SessionMemoryCompactConfig

```typescript
export type SessionMemoryCompactConfig = {
  minTokens: number           // 最少保留 token 数（默认 10,000）
  minTextBlockMessages: number // 最少文本块消息数（默认 5）
  maxTokens: number           // 最大保留 token 数（默认 40,000）
}
```

## B.7 相关文件索引

| 文件路径 | 描述 |
|---------|------|
| `src/utils/sessionStorage.ts` | CLI 会话存储主实现 |
| `src/utils/sessionStoragePortable.ts` | 跨平台共享工具（UUID、路径、JSON 提取） |
| `src/services/compact/compact.ts` | 核心压缩逻辑 |
| `src/services/compact/autoCompact.ts` | 自动压缩触发器 |
| `src/services/compact/microCompact.ts` | 微压缩实现 |
| `src/services/compact/sessionMemoryCompact.ts` | Session Memory 压缩 |
| `src/services/compact/postCompactCleanup.ts` | 压缩后清理 |
| `src/services/compact/grouping.ts` | API 轮次分组 |
| `src/services/compact/prompt.ts` | 压缩提示词模板 |
| `src/services/compact/timeBasedMCConfig.ts` | 时间触发微压缩配置 |
| `src/services/compact/compactWarningState.ts` | 警告状态管理 |
| `src/services/compact/compactWarningHook.ts` | React Hook 封装 |