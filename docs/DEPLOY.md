# 部署说明

## GitHub Pages 部署

本项目使用 GitHub Pages 部署，采用 Jekyll 静态站点生成。

### 部署步骤

1. ** Fork 本仓库**

2. **启用 GitHub Pages**
   - 进入 Settings → Pages
   - Source 选择 `main` 分支和 `/docs` 目录

3. **等待部署**
   - GitHub Pages 会自动构建
   - 约1-2分钟后即可访问

### 本地预览

```bash
# 安装 Jekyll
gem install jekyll bundler

# 安装依赖
cd docs
bundle install

# 启动本地服务器
bundle exec jekyll serve

# 访问 http://localhost:4000
```

### 目录结构

```
docs/
├── _config.yml          # Jekyll 配置
├── index.md             # 首页
├── 01-architecture/     # 第一章：架构总览
├── 02-entrypoint/       # 第二章：程序入口
├── 03-query-engine/     # 第三章：查询引擎
├── 04-tools/            # 第四章：工具系统
├── 05-memory/           # 第五章：记忆系统
├── 06-mcp/              # 第六章：MCP协议
├── 07-skills/           # 第七章：Skills
├── 08-sandbox/          # 第八章：沙箱安全
├── 09-ui-components/    # 第九章：UI组件
└── 10-multi-agent/      # 第十章：多Agent
```

### 注意事项

- 章节文档在对应目录的 `README.md`
- 图片资源放在 `assets/` 目录
- 使用 Markdown 格式编写，支持 GFM

## CI/CD

GitHub Actions 自动配置在 `.github/workflows/jekyll.yml`（如需要可添加）。