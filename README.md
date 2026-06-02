# 胤澄的博客

基于 Hugo + PaperMod 主题搭建的个人博客，托管于 GitHub Pages。

## 技术栈

- **Hugo** v0.162.1 — 静态站点生成器
- **PaperMod** — 简洁现代的博客主题
- **GitHub Pages** — 免费托管
- **GitHub Actions** — 自动部署

## 本地开发

```bash
# 安装 Hugo (macOS)
brew install hugo

# 本地预览
hugo server --buildDrafts

# 构建
hugo --minify
```

预览地址：`http://localhost:1313`:1313`:1313`

## 写作

```bash
# 新建文章
hugo new content posts/my-article.md
```

文章 Front Matter：

```yaml
---
title: "文章标题"
date: 2026-06-02
tags: ["标签"]
categories: ["分类"]
summary: "文章摘要"
draft: false
---
```

## 部署

推送到 `main` 分支后，GitHub Actions 自动构建并部署到 GitHub Pages。

## 文档

- [搭建指南](docs/setup-guide.md) — 完整的搭建过程记录

## License

MIT
