---
title: "博客搭建记录"
date: 2026-06-02
tags: ["博客", "Hugo"]
categories: ["技术"]
summary: "使用 Hugo + GitHub Pages 搭建个人博客的完整过程记录"
draft: false
---

## 为什么选择 Hugo

Hugo 是目前最快的静态站点生成器，构建速度极快，主题生态丰富，非常适合以内容为主的个人博客。

## 技术栈

- **Hugo** — 静态站点生成器
- **PaperMod** — 简洁现代的博客主题
- **GitHub Pages** — 免费托管
- **GitHub Actions** — 自动部署

## 开始写作

使用以下命令创建新文章：

```bash
hugo new content posts/my-first-post.md
```

在文章头部添加 front matter：

```yaml
---
title: "文章标题"
date: 2026-06-02
tags: ["标签"]
categories: ["分类"]
---
```

本地预览：

```bash
hugo server --buildDrafts
```

访问 `http://localhost:1313` 即可预览。
