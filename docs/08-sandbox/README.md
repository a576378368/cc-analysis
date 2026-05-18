# 第八章：Sandbox 安全机制

## 目录

1. [Sandbox 概述](#1-sandbox-概述)
2. [Sandbox 架构](#2-sandbox-架构)
3. [沙箱决策 shouldUseSandbox()](#3-沙箱决策-shouldusesandbox)
4. [权限配置转换](#4-权限配置转换)
5. [路径验证](#5-路径验证)
6. [进程隔离](#6-进程隔离)
7. [沙箱逃逸防护](#7-沙箱逃逸防护)

---

## 1. Sandbox 概述

### 1.1 什么是沙箱隔离

沙箱（Sandbox）是一种安全机制，旨在将代码执行限制在受控环境中，防止潜在危险的操作影响系统其他部分。在 Claude Code 中，沙箱通过以下技术实现：

- **Linux**: 使用 `bwrap` (bubblewrap) 进行系统调用过滤和资源限制
- **macOS**: 使用 Apple Silicon 的运行时防护机制
- **Windows**: 通过 WSL2（仅 WSL2，不支持 WSL1）实现隔离

### 1.2 Claude Code 的安全设计理念

Claude Code 采用**纵深防御**（Defense in Depth）策略，部署多层安全控制：

```
┌─────────────────────────────────────────────────────────────┐
│                    Claude Code 安全层次                       │
├─────────────────────────────────────────────────────────────┤
│  第1层: 用户权限配置 (settings.json)                          │
│    - 允许/拒绝规则 (permissions.allow/deny)                 │
│    - 沙箱配置 (sandbox.filesystem, sandbox.network)         │
│    - 模式匹配 (*, ?)                                         │
├─────────────────────────────────────────────────────────────┤
│  第2层: 命令解析与验证 (bashPermissions.ts)                   │
│    - AST 语义分析 (parseForSecurity)                         │
│    - 命令注入检测                                             │
│    - 路径约束检查                                            │
├─────────────────────────────────────────────────────────────┤
│  第3层: 沙箱运行时隔离 (sandbox-adapter.ts)                   │
│    - 文件系统限制 (allowRead, denyWrite, allowWrite)          │
│    - 网络限制 (allowedDomains, deniedDomains)                 │
│    - 进程隔离 (bwrap)                                        │
├─────────────────────────────────────────────────────────────┤
│  第4层: 平台级隔离                                           │
│    - Linux: bwrap 系统调用过滤                                │
│    - macOS: 运行时 sandbox                                   │
│    - Windows: WSL2 虚拟机                                    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Sandbox 架构

### 2.1 四层结构设计

Claude Code 的沙箱架构分为四个核心层级：

```
┌──────────────────────────────────────────────────────────────────┐
│                     Sandbox 架构四层模型                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │              Layer 4: 平台级隔离 (Platform Sandbox)         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │  │
│  │  │  Linux bwrap │  │  macOS Runtime│  │  WSL2        │    │  │
│  │  │  (Bubblewrap)│  │  (Apple Silicon)│  │  (Hyper-V)   │    │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │            Layer 3: 进程隔离与清理 (Process Isolation)       │  │
│  │  - bwrap 进程包装                                            │  │
│  │  - 临时文件清理 (cleanupAfterCommand)                       │  │
│  │  - 裸 Git 仓库文件清理 (scrubBareGitRepoFiles)              │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │         Layer 2: 运行时配置 (Runtime Config)                 │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │  │
│  │  │ Filesystem    │  │ Network      │  │ Process     │    │  │
│  │  │ Restrictions │  │ Restrictions │  │ Limits      │    │  │
│  │  └──────────────┘  └──────────────┘  └──────────────┘    │  │
│  └────────────────────────────────────────────────────────────┘  │
│                              ↓                                   │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │         Layer 1: 用户配置 (Settings)                        │  │
│  │  - sandbox.enabled                                          │  │
│  │  - sandbox.autoAllowBashIfSandboxed                        │  │
│  │  - sandbox.filesystem.*                                     │  │
│  │  - sandbox.network.*                                       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 与权限系统的关系

沙箱与权限系统紧密集成，形成双重保护：

```
┌─────────────────────────────────────────────────────────────────┐
│                    权限系统与沙箱交互流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   用户输入命令                                                    │
│        ↓                                                         │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ Step 1: 权限检查 (bashToolHasPermission)                │    │
│   │   - exact match: 精确命令匹配                             │    │
│   │   - prefix match: 前缀规则匹配                           │    │
│   │   - wildcard match: 通配符规则匹配                        │    │
│   │   - deny > ask > allow (优先级)                          │    │
│   └─────────────────────────────────────────────────────────┘    │
│        ↓                                                         │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ Step 2: 沙箱决策 (shouldUseSandbox)                      │    │
│   │   - sandbox.enabled 检查                                 │    │
│   │   - dangerouslyDisableSandbox 检查                       │    │
│   │   - excludedCommands 检查                                │    │
│   └─────────────────────────────────────────────────────────┘    │
│        ↓                                                         │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ Step 3: 沙箱包装 (wrapWithSandbox)                       │    │
│   │   - 转换 settings 为 runtime config                      │    │
│   │   - 调用 sandbox-runtime 包装命令                         │    │
│   └─────────────────────────────────────────────────────────┘    │
│        ↓                                                         │
│   ┌─────────────────────────────────────────────────────────┐    │
│   │ Step 4: 执行与清理 (exec + cleanupAfterCommand)           │    │
│   │   - 在沙箱中执行命令                                      │    │
│   │   - 清理临时文件                                          │    │
│   │   - 清理裸 Git 仓库文件                                   │    │
│   └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 沙箱决策 shouldUseSandbox()

### 3.1 命令是否进入沙箱的判断

`shouldUseSandbox()` 函数决定命令是否应该在沙箱中执行：

```typescript
// src/tools/BashTool/shouldUseSandbox.ts
export function shouldUseSandbox(input: Partial<SandboxInput>): boolean {
  // 1. 检查沙箱是否启用
  if (!SandboxManager.isSandboxingEnabled()) {
    return false
  }

  // 2. 检查是否显式禁用沙箱
  if (
    input.dangerouslyDisableSandbox &&
    SandboxManager.areUnsandboxedCommandsAllowed()
  ) {
    return false
  }

  // 3. 检查命令是否存在
  if (!input.command) {
    return false
  }

  // 4. 检查是否在 excludedCommands 中
  if (containsExcludedCommand(input.command)) {
    return false
  }

  return true
}
```

### 3.2 自动放行规则 (excludedCommands)

用户可以在设置中配置 `excludedCommands`，将某些命令排除在沙箱之外：

```typescript
function containsExcludedCommand(command: string): boolean {
  // 1. 检查动态配置中的禁用命令 (ANT only)
  if (process.env.USER_TYPE === 'ant') {
    const disabledCommands = getFeatureValue_CACHED_MAY_BE_STALE<{
      commands: string[]
      substrings: string[]
    }>('tengu_sandbox_disabled_commands', { commands: [], substrings: [] })
    // ...
  }

  // 2. 检查用户配置的 excludedCommands
  const settings = getSettings_DEPRECATED()
  const userExcludedCommands = settings.sandbox?.excludedCommands ?? []
  // 对每个子命令进行模式匹配
  // ...
}
```

### 3.3 决策流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    shouldUseSandbox() 决策流程                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  开始                                                            │
│    ↓                                                             │
│  [沙箱是否启用?] ──否──→ 返回 false (不禁用沙箱)                    │
│    │                                                            │
│   是                                                            │
│    ↓                                                             │
│  [dangerouslyDisableSandbox?] ──是──→ [允许非沙箱命令?] ──是──→ 返回 false│
│    │                            │                               │
│   否                            否                               │
│    ↓                            ↓                               │
│  [有 command?] ──否──→ 返回 false                                │
│    │                                                            │
│   是                                                            │
│    ↓                                                             │
│  [在 excludedCommands 中?] ──是──→ 返回 false                     │
│    │                                                            │
│   否                                                            │
│    ↓                                                             │
│  返回 true (使用沙箱)                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 4. 权限配置转换

### 4.1 settings 语义到 runtime config

`sandbox-adapter.ts` 中的 `convertToSandboxRuntimeConfig()` 函数将用户配置转换为沙箱运行时配置：

```typescript
// src/utils/sandbox/sandbox-adapter.ts
export function convertToSandboxRuntimeConfig(
  settings: SettingsJson,
): SandboxRuntimeConfig {
  // 网络配置
  const network = {
    allowedDomains,      // 允许的域名列表
    deniedDomains,       // 拒绝的域名列表
    allowUnixSockets,    // 允许 Unix 套接字
    allowAllUnixSockets, // 允许所有 Unix 套接字
    allowLocalBinding,   // 允许本地绑定
    httpProxyPort,       // HTTP 代理端口
    socksProxyPort,      // SOCKS 代理端口
  }

  // 文件系统配置
  const filesystem = {
    denyRead,   // 拒绝读取的路径
    allowRead,  // 允许读取的路径
    allowWrite, // 允许写入的路径
    denyWrite,  // 拒绝写入的路径
  }

  return { network, filesystem, ignoreViolations, ripgrep }
}
```

### 4.2 文件系统限制

文件系统限制通过多层机制实现：

```
┌─────────────────────────────────────────────────────────────────┐
│                   文件系统限制层级                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 绝对禁止写入的路径 (denyWrite)                                │
│     ├── settings.json 文件                                       │
│     ├── .claude/skills 目录                                      │
│     └── 裸 Git 仓库文件 (HEAD, objects, refs, hooks, config)     │
│                                                                  │
│  2. 始终允许写入的路径 (allowWrite)                                │
│     ├── . (当前工作目录)                                          │
│     ├── Claude 临时目录                                           │
│     ├── Git worktree 主仓库目录                                   │
│     └── --add-dir 添加的目录                                      │
│                                                                  │
│  3. 沙箱配置中的写入允许列表                                       │
│     └── 用户通过 sandbox.filesystem.allowWrite 配置               │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

关键实现代码：

```typescript
// 始终禁止写入 settings.json 文件
const settingsPaths = SETTING_SOURCES.map(source =>
  getSettingsFilePathForSource(source),
).filter((p): p is string => p !== undefined)
denyWrite.push(...settingsPaths)

// 禁止写入 .claude/skills
denyWrite.push(resolve(originalCwd, '.claude', 'skills'))

// 裸 Git 仓库安全处理
const bareGitRepoFiles = ['HEAD', 'objects', 'refs', 'hooks', 'config']
for (const dir of dirs) {
  for (const gitFile of bareGitRepoFiles) {
    try {
      statSync(p)
      denyWrite.push(p)  // 存在则拒绝写入
    } catch {
      bareGitRepoScrubPaths.push(p)  // 不存在则事后清理
    }
  }
}
```

### 4.3 网络限制

网络限制支持域名级别的访问控制：

```typescript
// 域名配置转换
for (const ruleString of permissions.allow || []) {
  const rule = permissionRuleValueFromString(ruleString)
  if (
    rule.toolName === WEB_FETCH_TOOL_NAME &&
    rule.ruleContent?.startsWith('domain:')
  ) {
    allowedDomains.push(rule.ruleContent.substring('domain:'.length))
  }
}

// 托管域名策略
if (shouldAllowManagedSandboxDomainsOnly()) {
  // 只使用 policySettings 中的域名
  const policySettings = getSettingsForSource('policySettings')
  for (const domain of policySettings?.sandbox?.network?.allowedDomains || []) {
    allowedDomains.push(domain)
  }
}
```

---

## 5. 路径验证

### 5.1 路径验证逻辑

路径验证在 `pathValidation.ts` 中实现，确保命令只能访问被授权的路径：

```typescript
// src/utils/permissions/pathValidation.ts
export function isPathAllowed(
  resolvedPath: string,
  context: ToolPermissionContext,
  operationType: FileOperationType,
): PathCheckResult {
  // 1. 检查 deny 规则
  const denyRule = matchingRuleForInput(resolvedPath, context, permissionType, 'deny')
  if (denyRule !== null) {
    return { allowed: false, decisionReason: { type: 'rule', rule: denyRule } }
  }

  // 2. 检查内部可编辑路径
  if (operationType !== 'read') {
    const internalEditResult = checkEditableInternalPath(resolvedPath, {})
    if (internalEditResult.behavior === 'allow') {
      return { allowed: true, decisionReason: internalEditResult.decisionReason }
    }
  }

  // 3. 安全检查
  const safetyCheck = checkPathSafetyForAutoEdit(resolvedPath, precomputedPathsToCheck)
  if (!safetyCheck.safe) {
    return { allowed: false, decisionReason: { type: 'safetyCheck', reason: safetyCheck.message } }
  }

  // 4. 工作目录检查
  const isInWorkingDir = pathInAllowedWorkingPath(resolvedPath, context, precomputedPathsToCheck)
  if (isInWorkingDir) {
    if (operationType === 'read' || context.mode === 'acceptEdits') {
      return { allowed: true }
    }
  }

  // 5. 沙箱写入白名单检查
  if (operationType !== 'read' && isPathInSandboxWriteAllowlist(resolvedPath)) {
    return { allowed: true, decisionReason: { type: 'other', reason: 'Path is in sandbox write allowlist' } }
  }

  // 6. 检查 allow 规则
  const allowRule = matchingRuleForInput(resolvedPath, context, permissionType, 'allow')
  if (allowRule !== null) {
    return { allowed: true, decisionReason: { type: 'rule', rule: allowRule } }
  }

  return { allowed: false }
}
```

### 5.2 模式匹配

路径验证支持多种模式匹配方式：

```typescript
// 路径模式支持
// - /path/to/file      : 精确路径
// - /path/to/*.txt     : 通配符匹配
// - /path/to/**        : 递归匹配
// - //absolute/path    : 绝对路径 (CC 特定语法)

// 解析 CC 特定路径约定
export function resolvePathPatternForSandbox(
  pattern: string,
  source: SettingSource,
): string {
  // //path → /path (绝对路径)
  if (pattern.startsWith('//')) {
    return pattern.slice(1)
  }

  // /path → ${settings_dir}/path (相对于设置文件目录)
  if (pattern.startsWith('/') && !pattern.startsWith('//')) {
    const root = getSettingsRootPathForSource(source)
    return resolve(root, pattern.slice(1))
  }

  // ~/path, ./path, path → 原样传递
  return pattern
}
```

### 5.3 安全边界

路径验证包含多层安全检查：

```
┌─────────────────────────────────────────────────────────────────┐
│                    路径验证安全边界                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. UNC 路径阻止                                                  │
│     - 防止通过网络路径泄露凭证                                     │
│     └── containsVulnerableUncPath()                              │
│                                                                  │
│  2. 波浪号变体阻止                                                │
│     - ~user, ~+, ~- 等在 shell 中有特殊含义                       │
│     - 这些可能绕过 expandTilde() 的保护                           │
│     └── cleanPath.startsWith('~') 检测                          │
│                                                                  │
│  3. Shell 扩展语法阻止                                            │
│     - $VAR, ${VAR}, $(cmd), %VAR%                               │
│     - =cmd (Zsh equals expansion)                               │
│     └── cleanPath.includes('$') || includes('%') || startsWith('=') │
│                                                                  │
│  4. 危险路径阻止                                                  │
│     - /* (根目录递归)                                            │
│     - ~ (主目录)                                                 │
│     - /usr, /tmp, /etc (根目录直接子项)                          │
│     - Windows 驱动器根目录                                        │
│     └── isDangerousRemovalPath()                                 │
│                                                                  │
│  5. Glob 模式限制                                                │
│     - 写入操作禁止使用 glob                                       │
│     - 读取操作验证 glob 基础目录                                   │
│     └── GLOB_PATTERN_REGEX.test(cleanPath)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 进程隔离

### 6.1 bwrap 包装 (Linux)

在 Linux 上，沙箱使用 bubblewrap (bwrap) 实现进程隔离：

```typescript
// src/utils/sandbox/sandbox-adapter.ts
async function wrapWithSandbox(
  command: string,
  binShell?: string,
  customConfig?: Partial<SandboxRuntimeConfig>,
  abortSignal?: AbortSignal,
): Promise<string> {
  // 如果沙箱未启用，直接返回原命令
  if (!isSandboxingEnabled()) {
    return command
  }

  // 调用基础沙箱管理器的包装方法
  return BaseSandboxManager.wrapWithSandbox(
    command,
    binShell,
    customConfig,
    abortSignal,
  )
}
```

bwrap 的典型调用参数包括：

```
bwrap \
  --ro-bind /usr /usr \
  --ro-bind /lib /lib \
  --tmpfs /tmp \
  --proc /proc \
  --dev /dev \
  --unshare-pid \
  --unshare-net \
  --uid 0 \
  --gid 0 \
  <command>
```

### 6.2 macOS Runtime

在 macOS (Apple Silicon) 上，沙箱使用系统运行时隔离机制：

```typescript
// macOS sandbox 使用内置的运行时沙箱
// 通过 sandbox-runtime 包抽象平台差异
await BaseSandboxManager.initialize(runtimeConfig, wrappedCallback)
```

### 6.3 清理机制

命令执行后的清理工作由 `cleanupAfterCommand()` 处理：

```typescript
// src/utils/sandbox/sandbox-adapter.ts
cleanupAfterCommand: (): void => {
  // 1. 调用基础沙箱管理器的清理
  BaseSandboxManager.cleanupAfterCommand()

  // 2. 清理可能存在的裸 Git 仓库文件
  scrubBareGitRepoFiles()
}

// 清理裸 Git 仓库文件的实现
function scrubBareGitRepoFiles(): void {
  for (const p of bareGitRepoScrubPaths) {
    try {
      rmSync(p, { recursive: true })
      logForDebugging(`[Sandbox] scrubbed planted bare-repo file: ${p}`)
    } catch {
      // ENOENT 是预期情况
    }
  }
}
```

在 Shell.ts 中也会在命令完成后触发清理：

```typescript
// src/utils/Shell.ts
void shellCommand.result.then(async result => {
  if (shouldUseSandbox) {
    SandboxManager.cleanupAfterCommand()
  }
  // ...
})
```

---

## 7. 沙箱逃逸防护

### 7.1 已知防护点

Claude Code 实现多层防护来阻止沙箱逃逸：

```
┌─────────────────────────────────────────────────────────────────┐
│                    沙箱逃逸防护机制                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 设置文件保护                                                  │
│     - settings.json 禁止写入                                      │
│     - .claude/skills 禁止写入                                    │
│     防护: 防止修改命令配置或技能来绕过沙箱                         │
│                                                                  │
│  2. 裸 Git 仓库检测                                              │
│     - 检测 cwd 是否为裸仓库                                      │
│     - 在沙箱中创建 .git 文件夹后进行清理                           │
│     防护: 防止通过 core.fsmonitor 机制逃逸                        │
│                                                                  │
│  3. cd + git 组合阻止                                            │
│     - 检测复合命令中的 cd + git 组合                              │
│     - 需要用户确认才允许                                         │
│     防护: 防止 cd 到恶意目录执行 git 命令                         │
│                                                                  │
│  4. 工作目录隔离                                                  │
│     - 验证所有文件操作在允许的工作目录内                          │
│     - 使用 realpath 解析符号链接                                 │
│     防护: 防止通过符号链接绕过目录限制                            │
│                                                                  │
│  5. 命令注入检测                                                  │
│     - Tree-sitter AST 解析验证                                    │
│     - 检测危险模式 (backticks, $(), 等)                          │
│     防护: 防止恶意构造的命令注入                                  │
│                                                                  │
│  6. 环境变量过滤                                                  │
│     - PATH, LD_PRELOAD, DYLD_* 保持不变                          │
│     - 只允许安全的 env var 前缀                                   │
│     防护: 防止通过环境变量劫持二进制                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 异常处理

沙箱异常处理机制确保安全失败：

```typescript
// 检查沙箱是否可用及不可用原因
function getSandboxUnavailableReason(): string | undefined {
  // 用户启用了沙箱但平台不支持
  if (!isSupportedPlatform()) {
    const platform = getPlatform()
    if (platform === 'wsl') {
      return 'sandbox.enabled is set but WSL1 is not supported (requires WSL2)'
    }
    return `sandbox.enabled is set but ${platform} is not supported (requires macOS, Linux, or WSL2)`
  }

  // 当前平台不在 enabledPlatforms 列表中
  if (!isPlatformInEnabledList()) {
    return `sandbox.enabled is set but ${getPlatform()} is not in sandbox.enabledPlatforms`
  }

  // 依赖缺失
  const deps = checkDependencies()
  if (deps.errors.length > 0) {
    return `sandbox.enabled is set but dependencies are missing: ${deps.errors.join(', ')}`
  }

  return undefined
}
```

### 7.3 网络请求处理

当沙箱中的命令尝试访问网络时：

```typescript
// src/components/permissions/SandboxPermissionRequest.tsx
export function SandboxPermissionRequest({
  hostPattern: { host },
  onUserResponse,
}: SandboxPermissionRequestProps) {
  const onSelect = (value: string) => {
    switch (value) {
      case 'yes':
        onUserResponse({ allow: true, persistToSettings: false })
        break
      case 'yes-dont-ask-again':
        onUserResponse({ allow: true, persistToSettings: true })
        break
      case 'no':
        onUserResponse({ allow: false, persistToSettings: false })
        break
    }
  }

  // 托管域名策略阻止
  const managedDomainsOnly = shouldAllowManagedSandboxDomainsOnly()
  // ...
}
```

---

## 附录：关键数据结构

### SandboxRuntimeConfig

```typescript
interface SandboxRuntimeConfig {
  network: {
    allowedDomains: string[]
    deniedDomains: string[]
    allowUnixSockets?: boolean
    allowAllUnixSockets?: boolean
    allowLocalBinding?: boolean
    httpProxyPort?: number
    socksProxyPort?: number
  }
  filesystem: {
    denyRead: string[]
    allowRead: string[]
    allowWrite: string[]
    denyWrite: string[]
  }
  ignoreViolations?: IgnoreViolationsConfig
  enableWeakerNestedSandbox?: boolean
  enableWeakerNetworkIsolation?: boolean
  ripgrep: {
    command: string
    args: string[]
    argv0: string
  }
}
```

### ISandboxManager 接口

```typescript
export interface ISandboxManager {
  initialize(sandboxAskCallback?: SandboxAskCallback): Promise<void>
  isSupportedPlatform(): boolean
  isPlatformInEnabledList(): boolean
  getSandboxUnavailableReason(): string | undefined
  isSandboxingEnabled(): boolean
  isSandboxEnabledInSettings(): boolean
  checkDependencies(): SandboxDependencyCheck
  isAutoAllowBashIfSandboxedEnabled(): boolean
  areUnsandboxedCommandsAllowed(): boolean
  isSandboxRequired(): boolean
  setSandboxSettings(options: {...}): Promise<void>
  getFsReadConfig(): FsReadRestrictionConfig
  getFsWriteConfig(): FsWriteRestrictionConfig
  getNetworkRestrictionConfig(): NetworkRestrictionConfig
  wrapWithSandbox(command: string, binShell?: string, ...): Promise<string>
  cleanupAfterCommand(): void
  getSandboxViolationStore(): SandboxViolationStore
  refreshConfig(): void
  reset(): Promise<void>
}
```

---

*文档版本: 1.0*
*最后更新: 2026/05/18*