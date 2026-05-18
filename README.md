# Claude Code Agent 设计教材

> 基于 Claude Code 泄露源码的深度架构分析

![TypeScript](https://img.shields.io/badge/TypeScript-5.x-blue)
![Source Files](https://img.shields.io/badge/Source%20Files-1902-green)

---

## 事件背景

2026年3月31日，安全研究者 [Chaofan Shou](https://x.com/Fried_rice) 发现 Anthropic 发布到 npm 的 Claude Code 包中**未删除 source map 文件**，导致完整 TypeScript 源码泄露：

- **1902 个** 源文件
- **513,237 行** 代码

本教材基于泄露源码进行深度分析，系统讲解现代 AI Agent 的核心设计模式与工程实现。

## 在线阅读

**访问地址**：[https://a576378368.github.io/cc-analysis/](https://a576378368.github.io/cc-analysis/)

> 文档使用 MkDocs Material 主题构建，支持深色模式、代码高亮、Mermaid 图表。

---

## 教材目录

### 核心章节

| 章节 | 主题 | 说明 |
|:-----|:-----|:-----|
| [第一章](01-architecture/) | 架构总览 | 六层架构设计、数据流总览、模块关系 |
| [第二章](02-entrypoint/) | 程序入口与初始化 | CLI 分流、主程序入口、环境初始化 |
| [第三章](03-query-engine/) | Query/Agent 执行内核 | 查询引擎、上下文管理、Agent SDK |
| [第四章](04-tools/) | 工具系统设计与实现 | 工具基类、权限系统、工具调度 |
| [第五章](05-memory/) | 记忆系统架构 | Auto/Session/Agent/Team 四层记忆 |
| [第六章](06-mcp/) | MCP 协议实现 | MCP 客户端、传输层、OAuth 认证 |
| [第七章](07-skills/) | Skills 机制 | 技能加载、Shell 执行，内建技能 |
| [第八章](08-sandbox/) | Sandbox 安全机制 | 沙箱隔离、权限验证、路径验证 |
| [第九章](09-ui-components/) | UI 组件架构 | React 组件设计、控制面组件 |
| [第十章](10-multi-agent/) | 多Agent系统 | 子Agent、Team协作、Swarm架构 |

### 附录

| 附录 | 主题 |
|:-----|:-----|
| [A](appendix-a-context-management/) | Context 上下文管理 |
| [B](appendix-b-session-storage/) | Session Storage 持久化 |
| [C](appendix-c-privacy/) | 隐私保护与数据处理 |
| [D](appendix-d-prompt-management/) | Prompt 管理机制 |
| [E](appendix-e-competitive-comparison/) | 竞品对比分析 |
| [G](appendix-g-src-file-tree/) | 源码文件树详解 |

---

## 核心设计亮点

### 分层架构

```
CLI 引导层 → TUI/REPL 交互层 → Query/Agent 执行内核 → Tool/Permission 层 → Memory/Persistence 层 → MCP/Remote/Swarm 扩展层
```

### 多层记忆系统

- **Auto Memory** - 长期记忆，跨 session
- **Session Memory** - 当前会话摘要
- **Agent Memory** - Agent 角色绑定记忆
- **Team Memory** - 团队共享记忆

### 安全设计

- 沙箱进程隔离（bwrap/macOS Runtime）
- 工具权限控制（ask/auto/bypass）
- 路径验证与限制
- OAuth 认证

---

## 技术栈

| 层级 | 技术 |
|:-----|:-----|
| 文档构建 | MkDocs + Material |
| 文档格式 | Markdown |
| 图表 | Mermaid |
| 源码语言 | TypeScript |

---

## 快速开始

### 在线阅读

直接访问：[https://a576378368.github.io/cc-analysis/](https://a576378368.github.io/cc-analysis/)

### 本地预览

```bash
# 克隆仓库
git clone https://github.com/a576378368/cc-analysis.git
cd cc-analysis/docs

# 安装依赖
pip install mkdocs-material mkdocstrings mkdocs-glightbox

# 启动本地服务器
mkdocs serve

# 访问 http://localhost:8000
```

---

## 目录结构

```
cc-analysis/
├── README.md              # 本文件
├── cc_src/               # Claude Code 泄露源码（35MB）
└── docs/                 # 文档源文件
    ├── 01-architecture/  # 第一章
    ├── 02-entrypoint/    # 第二章
    ├── ...
    ├── 10-multi-agent/    # 第十章
    ├── appendix-*/       # 附录
    ├── index.md          # MkDocs 首页
    └── mkdocs.yml        # MkDocs 配置
```

---

## 声明

本项目仅供学术研究与技术学习使用。
本仓库所有内容均为对公开信息的二次整理与分析。Claude Code 的所有权利归 [Anthropic](https://www.anthropic.com) 所有。

- **无侵权意图**：本分析文档基于已在公共互联网上广泛流传的信息整理撰写，目的在于帮助开发者了解 AI Coding Agent 的安全边界、隐私设计与工程架构，属于正当的技术研究行为。
- **禁止商业使用**：禁止将本仓库内容用于任何商业目的，或以此绕过、破坏 Claude Code 的安全机制与用户协议。
- **免责声明**：本仓库作者不对因参考本文档而产生的任何直接或间接损失负责。如有任何合规疑虑，请以 Anthropic 官方文档与用户协议为准。
- **如需删除**：若 Anthropic 认为本仓库内容侵犯其合法权益，请通过 Issue 联系，我们将在核实后第一时间进行删除处理。

---

*最后更新：2026年5月*
