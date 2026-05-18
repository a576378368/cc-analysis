# Claude Code Agent 设计教材

> 基于 Claude Code 泄露源码的深度架构分析

![Claude Code](https://img.shields.io/badge/Claude-Code-blue)
![TypeScript](https://img.shields.io/badge/TypeScript-5.x-green)
![License](https://img.shields.io/badge/License-MIT-yellow)

## 关于本教材

本教材基于 2026年3月31日泄露的 Claude Code 完整 TypeScript 源码（共1902个源文件，513,237行代码）进行深度分析，系统性地讲解现代 AI Agent 的核心设计模式与工程实现。

## 源码来源

2026年3月31日，安全研究者 [Chaofan Shou](https://x.com/Fried_rice) 发现 Anthropic 发布到 npm 的 Claude Code 包中未删除 source map 文件，导致完整 TypeScript 源码泄露。该源码已在公共互联网广泛流传。

## 教材结构

本教材共分 **十章**，从架构设计到具体实现进行全方位讲解：

### 第一部分：架构基础

| 章节 | 主题 | 核心内容 |
|:-----|:-----|:---------|
| 第一章 | [架构总览](01-architecture/) | 六层架构设计、数据流总览、模块关系 |
| 第二章 | [程序入口与初始化](02-entrypoint/) | CLI 分流、主程序入口、环境初始化 |

### 第二部分：核心机制

| 章节 | 主题 | 核心内容 |
|:-----|:-----|:---------|
| 第三章 | [Query/Agent 执行内核](03-query-engine/) | 查询引擎、上下文管理、Agent SDK |
| 第四章 | [工具系统设计与实现](04-tools/) | 工具基类、权限系统、工具调度 |
| 第五章 | [记忆系统架构](05-memory/) | Auto/Session/Agent/Team 四层记忆 |

### 第三部分：扩展与安全

| 章节 | 主题 | 核心内容 |
|:-----|:-----|:---------|
| 第六章 | [MCP 协议实现](06-mcp/) | MCP 客户端、传输层、OAuth 认证 |
| 第七章 | [Skills 机制](07-skills/) | 技能加载、Shell 执行、内建技能 |
| 第八章 | [Sandbox 安全机制](08-sandbox/) | 沙箱隔离、权限验证、路径验证 |

### 第四部分：交互与协作

| 章节 | 主题 | 核心内容 |
|:-----|:-----|:---------|
| 第九章 | [UI 组件架构](09-ui-components/) | React 组件设计、控制面组件 |
| 第十章 | [多Agent系统](10-multi-agent/) | 子Agent、Team协作、Swarm架构 |

### 附录

- [附录 A: Context 上下文管理](appendix-a-context-management/)
- [附录 B: Session Storage 持久化](appendix-b-session-storage/)
- [附录 C: 隐私保护与数据处理](appendix-c-privacy/)
- [附录 D: Prompt 管理机制](appendix-d-prompt-management/)
- [附录 E: 竞品对比分析](appendix-e-competitive-comparison/)
- [附录 G: 源码文件树详解](appendix-g-src-file-tree/)

## 核心设计模式

### 1. 分层架构

```
CLI 引导层 → TUI/REPL 交互层 → Query/Agent 执行内核 → Tool/Permission 层 → Memory/Persistence 层 → MCP/Remote/Swarm 扩展层
```

### 2. 多层记忆系统

- **Auto Memory**: 长期记忆，跨session
- **Session Memory**: 当前会话摘要
- **Agent Memory**: Agent 角色绑定记忆
- **Team Memory**: 团队共享记忆

### 3. 安全设计

- 沙箱进程隔离
- 工具权限控制
- 路径验证与限制
- OAuth 认证

## 技术栈

| 层级 | 技术 |
|:-----|:-----|
| 语言 | TypeScript |
| UI框架 | React + Ink |
| 协议 | MCP (Model Context Protocol) |
| 隔离 | bwrap (Linux), macOS Runtime |
| 构建 | Bun |

## 如何使用本教材

1. **入门**: 从第一章架构总览开始，了解整体设计
2. **深入**: 根据兴趣选择特定章节深入研究
3. **实践**: 结合源码目录 `src/` 进行对照学习

## 本地预览

```bash
# 安装依赖
pip install mkdocs-material mkdocstrings mkdocs-glightbox

# 启动本地服务器
mkdocs serve

# 访问 http://localhost:8000
```

## 部署到 GitHub Pages

```bash
mkdocs gh-deploy
```

## 贡献与反馈

本教材基于公开源码的学术研究。如有发现分析错误或需要补充的内容，欢迎提交 Issue。

## 免责声明

> 本教材仅供学术研究与技术学习使用。Claude Code 的所有权利归 Anthropic 所有。本分析不构成任何侵权意图，一切内容基于公开信息整理。

---

*最后更新: 2026年5月*