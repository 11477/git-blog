# Hugo + GitHub Pages 个人博客搭建指南

## 概述

本指南记录了使用 **Hugo** 静态站点生成器 + **PaperMod** 主题 + **GitHub Pages** 托管 + **GitHub Actions** 自动部署的完整搭建过程。

## 技术栈

| 组件 | 版本/说明 |
|------|----------|
| Hugo | v0.162.1 (extended) |
| 主题 | PaperMod |
| 托管 | GitHub Pages |
| CI/CD | GitHub Actions |
| 域名 | `11477.github.io` |

---

## 1. 环境准备

### 1.1 安装 Hugo

macOS 使用 Homebrew 安装：

```bash
brew install hugo
```

验证安装：

```bash
hugo version
# hugo v0.162.1+extended
```

> **注意**：需要安装 `extended` 版本以支持 SCSS/Sass，Homebrew 默认安装的就是 extended 版本。

### 1.2 确认 Git 环境

```bash
git --version
```

---

## 2. 初始化项目

### 2.1 创建站点

```bash
mkdir git-blog && cd git-blog
hugo new site . --force
git init
```

生成的目录结构：

```
.
├── archetypes/     # 文章模板
│   └── default.md
├── content/        # 博客内容（Markdown）
├── data/           # 数据文件
├── layouts/        # 自定义布局模板
├── static/         # 静态资源（图片、CSS等）
├── themes/         # 主题
└── hugo.toml       # 站点配置
```

### 2.2 安装 PaperMod 主题

使用 Git Submodule 引入主题（推荐方式，便于后续更新）：

```bash
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

更新主题：

```bash
git submodule update --remote --merge
```

---

## 3. 站点配置

编辑 `hugo.toml`：

```toml
baseURL = 'https://11477.github.io/'
locale = 'zh-cn'
title = 'cyw777的博客'
theme = 'PaperMod'

[pagination]
  pagerSize = 10

[params]
  defaultTheme = 'auto'
  ShowReadingTime = true
  ShowShareButtons = false
  ShowPostNavLinks = true
  ShowBreadCrumbs = true
  ShowCodeCopyButtons = true
  ShowRssButtonInSectionTermList = true
  UseHugoToc = true

  [params.homeInfoParams]
    Title = 'cyw777的博客'
    Content = '技术 / 思考 / 记录'

  [[params.socialIcons]]
    name = 'github'
    url = 'https://github.com/11477'

[params.editPost]
  URL = 'https://github.com/11477/git-blog/tree/main/content'
  Text = '建议修改'
  appendFilePath = true

[outputs]
  home = ['HTML', 'RSS']

[markup]
  [markup.highlight]
    style = 'dracula'
    lineNos = false
    codeFences = true
    guessSyntax = true

[markup.goldmark]
  [markup.goldmark.renderer]
    unsafe = true

[[menu.main]]
  identifier = 'archives'
  name = '归档'
  url = '/archives/'
  weight = 5

[[menu.main]]
  identifier = 'categories'
  name = '分类'
  url = '/categories/'
  weight = 10

[[menu.main]]
  identifier = 'tags'
  name = '标签'
  url = '/tags/'
  weight = 15

[[menu.main]]
  identifier = 'search'
  name = '搜索'
  url = '/search/'
  weight = 20
```

### 3.1 关键配置说明

| 配置项 | 说明 |
|--------|------|
| `baseURL` | 站点根 URL，部署时必须正确设置 |
| `locale` | 语言区域（Hugo v0.158+ 替代 `languageCode`） |
| `defaultTheme` | `auto` 跟随系统深色/浅色模式 |
| `ShowCodeCopyButtons` | 代码块显示复制按钮 |
| `UseHugoToc` | 使用 Hugo 内置目录生成 |

### 3.2 创建归档和搜索页

`content/archives/_index.md`：

```yaml
---
title: "归档"
layout: "archives"
summary: "archives"
---
```

`content/search/_index.md`：

```yaml
---
title: "搜索"
layout: "search"
placeholder: "搜索文章..."
---
```

---

## 4. 写作工作流

### 4.1 创建新文章

```bash
hugo new content posts/my-article.md
```

### 4.2 Front Matter 模板

```yaml
---
title: "文章标题"
date: 2026-06-02
tags: ["标签1", "标签2"]
categories: ["分类"]
summary: "文章摘要"
draft: false        # false 才会被发布
---
```

### 4.3 本地预览

```bash
# 预览草稿
hugo server --buildDrafts

# 仅预览已发布内容
hugo server
```

访问 `http://localhost:1313`:1313` 查看效果。

### 4.4 构建静态文件

```bash
hugo --minify
```

输出到 `public/` 目录。

---

## 5. GitHub Pages 部署

### 5.1 创建 GitHub 仓库

在 GitHub 上创建仓库，建议命名为 `git-blog`（或 `<username>.github.io` 以使用根域名）。

### 5.2 配置 GitHub Pages

1. 进入仓库 **Settings → Pages**
2. Source 选择 **GitHub Actions**

### 5.3 GitHub Actions 工作流

创建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy Hugo to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

defaults:
  run:
    shell: bash

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.162.1
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i ${{ runner.temp }}/hugo.deb

      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0

      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5

      - name: Build with Hugo
        env:
          HUGO_CACHEDIR: ${{ runner.temp }}/hugo_cache
          HUGO_ENVIRONMENT: production
        run: hugo --minify --baseURL "${{ steps.pages.outputs.base_url }}/"

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 5.4 推送代码触发部署

```bash
git remote add origin https://github.com/11477/git-blog.git
git add .
git commit -m "init: Hugo blog with PaperMod theme"
git branch -M main
git push -u origin main
```

推送后 GitHub Actions 自动构建并部署。

---

## 6. 自定义域名（可选）

### 6.1 配置域名

1. 在域名服务商添加 CNAME 记录指向 `11477.github.io`
2. 在仓库 **Settings → Pages → Custom domain** 填入域名
3. 勾选 **Enforce HTTPS**

### 6.2 Hugo 配置

修改 `hugo.toml` 中的 `baseURL`：

```toml
baseURL = 'https://your-domain.com/'
```

---

## 7. .gitignore

```
public/
resources/
.hugo_build.lock
```

---

## 8. 常用命令速查

| 命令 | 说明 |
|------|------|
| `hugo new content posts/xxx.md` | 新建文章 |
| `hugo server --buildDrafts` | 本地预览（含草稿） |
| `hugo --minify` | 构建生产版本 |
| `hugo list all` | 列出所有文章 |
| `hugo mod update` | 更新模块依赖 |

---

## 9. 注意事项

1. **Hugo 版本**：CI 中的 `HUGO_VERSION` 需与本地一致，否则可能构建失败
2. **Submodule 更新**：更新主题后需 `git submodule update --remote --merge`
3. **Draft 文章**：`draft: true` 的文章不会被发布，本地预览需加 `--buildDrafts`
4. **baseURL**：部署时必须正确设置，否则 CSS/JS 等资源路径会出错
5. **PaperMod 兼容性**：主题的部分废弃警告不影响使用，等主题更新即可
