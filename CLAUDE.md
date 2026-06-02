# CLAUDE.md — 项目上下文

## 项目概述

Hugo 静态博客，PaperMod 主题，GitHub Pages 托管，GitHub Actions 自动部署。

## 关键路径

- `hugo.toml` — 站点配置（主题、菜单、参数）
- `content/posts/` — 博客文章（Markdown）
- `content/archives/` — 归档页
- `content/search/` — 搜索页
- `themes/PaperMod/` — 主题（git submodule，不要直接修改）
- `.github/workflows/deploy.yml` — CI/CD 部署工作流
- `docs/setup-guide.md` — 搭建文档
- `.hugo-version` — Hugo 版本号（单一版本源，CI 和本地都从此文件读取）

## 常用命令

```bash
hugo server --buildDrafts   # 本地预览（含草稿）
hugo new content posts/xxx.md  # 新建文章
hugo --minify                # 构建生产版本
```

## 规则

- 文章放在 `content/posts/` 下，使用 Markdown
- `draft: true` 的文章不会被发布
- 修改 `hugo.toml` 后需重启 `hugo server`
- 不要修改 `themes/PaperMod/` 内文件，自定义布局放 `layouts/` 覆盖
- 主题更新：`git submodule update --remote --merge`
- 升级 Hugo 时只需修改 `.hugo-version`，CI 自动读取，无需改 deploy.yml
