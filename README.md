# Claude Code Agent 设计教材

基于 Claude Code 泄露源码的深度架构分析

## 事件背景

2026年3月31日，安全研究者 [Chaofan Shou](https://x.com/Fried_rice) 发现 Anthropic 发布到 npm 的 Claude Code 包中未删除 source map 文件，导致完整 TypeScript 源码泄露（共 1902 个源文件，513,237 行代码）。

## 在线文档

**阅读完整教材**：[https://a576378368.github.io/cc-analysis/](https://a576378368.github.io/cc-analysis/)

文档使用 MkDocs Material 主题构建，支持深色模式、代码高亮、Mermaid 图表。

## 教材结构

| 章节 | 主题 | 说明 |
|:-----|:-----|:-----|
| 第一章 | [架构总览](docs/01-architecture/) | 六层架构设计、数据流总览 |
| 第二章 | [程序入口与初始化](docs/02-entrypoint/) | CLI 分流、主程序入口 |
| 第三章 | [Query/Agent 执行内核](docs/03-query-engine/) | 查询引擎、上下文管理 |
| 第四章 | [工具系统设计与实现](docs/04-tools/) | 工具基类、权限系统 |
| 第五章 | [记忆系统架构](docs/05-memory/) | Auto/Session/Agent/Team 四层记忆 |
| 第六章 | [MCP 协议实现](docs/06-mcp/) | MCP 客户端、传输层、OAuth |
| 第七章 | [Skills 机制](docs/07-skills/) | 技能加载、Shell 执行 |
| 第八章 | [Sandbox 安全机制](docs/08-sandbox/) | 沙箱隔离、权限验证 |
| 第九章 | [UI 组件架构](docs/09-ui-components/) | React 组件设计 |
| 第十章 | [多Agent系统](docs/10-multi-agent/) | 子Agent、Team协作 |

### 附录

- [A: Context 上下文管理](docs/appendix-a-context-management/)
- [B: Session Storage 持久化](docs/appendix-b-session-storage/)
- [C: 隐私保护与数据处理](docs/appendix-c-privacy/)
- [D: Prompt 管理机制](docs/appendix-d-prompt-management/)
- [E: 竞品对比分析](docs/appendix-e-competitive-comparison/)
- [G: 源码文件树详解](docs/appendix-g-src-file-tree/)

## 目录结构

```
cc-analysis/
├── README.md           # 本文件
├── cc_src/            # Claude Code 泄露源码（35MB）
├── docs/              # MkDocs 文档源文件
│   ├── 01-architecture/
│   ├── 02-entrypoint/
│   ├── 03-query-engine/
│   ├── ...（十章 + 附录）
│   ├── src/           # 源码副本（已弃用，请使用 cc_src/）
│   └── mkdocs.yml   # MkDocs 配置
└── docs/site/        # GitHub Pages 构建输出
```

## 源码

源码位于 `cc_src/` 目录，共 1902 个 TypeScript 文件。

## 本地预览

```bash
cd docs
pip install mkdocs-material mkdocstrings mkdocs-glightbox
mkdocs serve
# 访问 http://localhost:8000
```

---

## 声明

> **本项目仅供学术研究与技术学习使用。**

本仓库所有内容均为对公开信息的二次整理与分析。Claude Code 的所有权利归 [Anthropic](https://www.anthropic.com) 所有。

1. **无侵权意图**：本分析文档基于已在公共互联网上广泛流传的信息整理撰写，目的在于帮助开发者了解 AI Coding Agent 的安全边界、隐私设计与工程架构，属于正当的技术研究行为。
2. **禁止商业使用**：禁止将本仓库内容用于任何商业目的，或以此绕过、破坏 Claude Code 的安全机制与用户协议。
3. **免责声明**：本仓库作者不对因参考本文档而产生的任何直接或间接损失负责。如有任何合规疑虑，请以 Anthropic 官方文档与用户协议为准。
4. **如需删除**：若 Anthropic 认为本仓库内容侵犯其合法权益，请通过 Issue 联系，我们将在核实后第一时间进行删除处理。

---

*最后更新：2026年5月*
